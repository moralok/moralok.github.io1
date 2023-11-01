---
title: JVM GC 的测试和分析
date: 2023-11-01 11:42:50
tags: jvm
---

```java
public class JvmGcTest {

    public static final int _512KB = 512 * 1024;
    public static final int _1MB = 1024 * 1024;
    public static final int _6MB = 6 * 1024 * 1024;
    public static final int _7MB = 7 * 1024 * 1024;
    public static final int _8MB = 8 * 1024 * 1024;

    // -Xms20M -Xmx20M -Xmn10M -XX:+UseSerialGC -XX:+PrintGCDetails -verbose:gc
    public static void main(String[] args) {

    }
}
```

```console
Heap
 def new generation   total 9216K, used 2010K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  24% used [0x00000000fec00000, 0x00000000fedf68c8, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
  to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
 tenured generation   total 10240K, used 0K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,   0% used [0x00000000ff600000, 0x00000000ff600000, 0x00000000ff600200, 0x0000000100000000)
 Metaspace       used 3288K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 348K, capacity 388K, committed 512K, reserved 1048576K
```


### 堆的组成

- `Heap`: 堆。
    - `def new generation`: 新生代。
    - `tenured generation`: 老年代。
    - `Metaspace`: 元空间，并不属于堆， `-XX:+PrintGCDetails` 将它的信息一起输出。


### 空间占比
新生代中的空间占比 `eden:from:to` 在默认情况下是 `8:1:1`，与观察到的数据 `8192K:1024K:1024K` 一致。
新生代的空间 `eden + from + to` 为 10240K，符合 `-Xmn10M` 设置的大小。
`total` 显示为 9216K，即 `eden + from` 的大小，是因为 `to` 的空间不计算在内。新生代可用的空间只有 `eden + from`，`to` 空间只是在使用标记-复制算法进行垃圾回收时使用。
老年代的空间为 10240K。
目前仅 `eden` 中已用 2010K，约占 `eden` 空间的 24%。


### 从地址指针分析空间
地址指针为 16 位的 16 进制的数字，64 位机器。
`[0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)` 分别表示地址空间的开始、已用、结束的地址指针。
新生代 `[0x00000000fec00000, 0x00000000ff600000)`，老年代 `[0x00000000ff600000, 0x0000000100000000)`，计算可得空间大小均为 10MB。
`eden` 中已用的空间地址为 `[0x00000000fec00000, 0x00000000fedf68c8)`，空间大小为 2058440 byte，约等于 2010K。

显而易见，新生代和老生代是一片完全连续的地址空间。


```java
public static void main(String[] args) {
    List<byte[]> list = new ArrayList<>();
    list.add(new byte[_7MB]);
}
```

```console
[GC (Allocation Failure) [DefNew: 2013K->721K(9216K), 0.0105099 secs] 2013K->721K(19456K), 0.0105455 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
Heap
 def new generation   total 9216K, used 8135K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  90% used [0x00000000fec00000, 0x00000000ff33d8c0, 0x00000000ff400000)
  from space 1024K,  70% used [0x00000000ff500000, 0x00000000ff5b45f0, 0x00000000ff600000)
  to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
 tenured generation   total 10240K, used 0K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,   0% used [0x00000000ff600000, 0x00000000ff600000, 0x00000000ff600200, 0x0000000100000000)
 Metaspace       used 3354K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 360K, capacity 388K, committed 512K, reserved 1048576K
```


### GC 类型
- GC: minor GC。
- Fulle GC: full GC。

`Allocation Failure`，分配空间失败，`eden` 的剩余空间不足以存放 7M 大小的对象，新生代发生 `minor GC`。
`[DefNew: 2013K->721K(9216K), 0.0105099 secs]`，新生代在垃圾回收前后空间的占用变化和耗时。
`2013K->721K(19456K), 0.0105455 secs`，整个堆在垃圾回收前后空间的占用变化和耗时。


### from 和 to 的角色变换
`from` 的已用空间的地址为 `[0x00000000ff500000, 0x00000000ff5b45f0)`，空间大小为 738800 byte，约 721K，与 GC 后的新生代空间占用大小一致。可见在垃圾回收后，`eden` 区域存活的对象全部转移到了原 `to` 空间，`from` 和 `to` 空间的角色相互转换（从地址空间的信息可以看到此时 `to` 的地址指针比 `from` 的地址指针小）。
`eden` 的已用空间的地址为 `[0x00000000fec00000, 0x00000000ff33d8c0)`，空间大小为 7592128 byte，约 7.24M，比 7M 大不少。此时 `eden` 区域除了 `byte[]` 对象外，还存储了其他对象，比如为了创建 `List<byte[]>` 和 `byte[]` 对象而新加载的类对象。

```java
public static void main(String[] args) {
    List<byte[]> list = new ArrayList<>();
    list.add(new byte[_7MB]);
    list.add(new byte[_512KB]);
}
```

```console
[GC (Allocation Failure) [DefNew: 2013K->721K(9216K), 0.0011172 secs] 2013K->721K(19456K), 0.0011443 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 def new generation   total 9216K, used 8647K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  96% used [0x00000000fec00000, 0x00000000ff3bd8d0, 0x00000000ff400000)
  from space 1024K,  70% used [0x00000000ff500000, 0x00000000ff5b45f0, 0x00000000ff600000)
  to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
 tenured generation   total 10240K, used 0K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,   0% used [0x00000000ff600000, 0x00000000ff600000, 0x00000000ff600200, 0x0000000100000000)
 Metaspace       used 3354K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 360K, capacity 388K, committed 512K, reserved 1048576K
```


### eden 空间足够时不发生 GC
由于 `eden` 区域还能放下 512K 的对象，所以仍然只会发生一次垃圾回收。
`eden` 区域的已用空间比例上升到 96%，已用空间的地址为 `[0x00000000fec00000, 0x00000000ff3bd8d0)`，空间大小为 8116432 byte，约 7.74M，比上一次增加了 524304 byte，即 `512 * 1024 + 16`。显然第二次添加时，不再因为创建 `List<byte[]>` 和 `byte[]` 对象而创建额外的对象，只有创建对象所需的 512K 和 16 字节的对象头。


```java
public static void main(String[] args) {
    List<byte[]> list = new ArrayList<>();
    list.add(new byte[_7MB]);
    list.add(new byte[_512KB]);
    list.add(new byte[_512KB]);
}
```

```console
[GC (Allocation Failure) [DefNew: 2013K->721K(9216K), 0.0013580 secs] 2013K->721K(19456K), 0.0013932 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [DefNew: 8565K->512K(9216K), 0.0046378 secs] 8565K->8396K(19456K), 0.0046540 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 def new generation   total 9216K, used 1350K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  10% used [0x00000000fec00000, 0x00000000fecd1a20, 0x00000000ff400000)
  from space 1024K,  50% used [0x00000000ff400000, 0x00000000ff480048, 0x00000000ff500000)
  to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
 tenured generation   total 10240K, used 7884K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,  77% used [0x00000000ff600000, 0x00000000ffdb33a0, 0x00000000ffdb3400, 0x0000000100000000)
 Metaspace       used 3354K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 360K, capacity 388K, committed 512K, reserved 1048576K
```


### 新生代空间不足，部分对象提前晋升到老年代
在第三次添加时，由于 `eden` 空间不足，因此又发生了第二次垃圾回收。
`[DefNew: 8565K->512K(9216K), 0.0046378 secs]`，新生代的空间占用下降到了 512K。
在第二次添加完成后，`eden` `[0x00000000fec00000, 0x00000000ff3bd8d0)` 和 `from` `[0x00000000ff500000, 0x00000000ff5b45f0)` 占用的空间为 `8116432 + 738800 = 8855232` 约 8647.7K，略大于 8565K，在垃圾回收后新生代剩下 512K，应该是在 from 中留下了第二次添加时的 512K。
`8565K->8396K(19456K), 0.0046540 secs`，堆的占用空间并未发生明显下降，部分对象是因为新生代空间不足，提前晋升到了老年代中。
优先将大对象先晋升到老年代应该是一种晋升策略。
`eden` 中加入了第三次添加时的对象，但远大于 512K。此时 `eden`、`from`、`tenured` 中均有不好确认成分的空间占用。


```java
public static void main(String[] args) {
    List<byte[]> list = new ArrayList<>();
    list.add(new byte[_8MB]);
}
```

```java
Heap
 def new generation   total 9216K, used 2177K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  26% used [0x00000000fec00000, 0x00000000fee20730, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
  to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
 tenured generation   total 10240K, used 8192K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,  80% used [0x00000000ff600000, 0x00000000ffe00010, 0x00000000ffe00200, 0x0000000100000000)
 Metaspace       used 3353K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 360K, capacity 388K, committed 512K, reserved 1048576K
```


### 新生代空间不足，大对象直接在老年代创建
在新生代的空间肯定不足而老年代空间足够的情况下，大对象会直接在老年代中创建，此时不会发生 GC。


```java
public static void main(String[] args) {
    List<byte[]> list = new ArrayList<>();
    list.add(new byte[_8MB - 16]);
}
```

```console
[GC (Allocation Failure) [DefNew: 2013K->693K(9216K), 0.0015517 secs] 2013K->693K(19456K), 0.0015828 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [DefNew: 8885K->0K(9216K), 0.0048110 secs] 8885K->8885K(19456K), 0.0048264 secs] [Times: user=0.00 sys=0.02, real=0.00 secs] 
Heap
 def new generation   total 9216K, used 410K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,   5% used [0x00000000fec00000, 0x00000000fec66958, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
  to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
 tenured generation   total 10240K, used 8885K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,  86% used [0x00000000ff600000, 0x00000000ffead580, 0x00000000ffead600, 0x0000000100000000)
 Metaspace       used 3321K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 354K, capacity 388K, committed 512K, reserved 1048576K
```


### 大对象的划分指标
当创建的大对象 + 对象头的容量小于等于 `eden`，如果 GC 后的存活对象可以放入 `to`，那么还是会先在 `eden` 中创建大对象。
在本案例中，又会马上发生一次 GC，大对象提前晋升到老年代中。


```java
public static void main(String[] args) {
    List<byte[]> list = new ArrayList<>();
    list.add(new byte[_8MB]);
    list.add(new byte[_8MB]);
}
```

```console
[GC (Allocation Failure) [DefNew: 2013K->721K(9216K), 0.0012274 secs][Tenured: 8192K->8912K(10240K), 0.0113036 secs] 10205K->8912K(19456K), [Metaspace: 3345K->3345K(1056768K)], 0.0125751 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (Allocation Failure) [Tenured: 8912K->8895K(10240K), 0.0011880 secs] 8912K->8895K(19456K), [Metaspace: 3345K->3345K(1056768K)], 0.0012009 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 def new generation   total 9216K, used 246K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,   3% used [0x00000000fec00000, 0x00000000fec3d890, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
  to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
 tenured generation   total 10240K, used 8895K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,  86% used [0x00000000ff600000, 0x00000000ffeafce0, 0x00000000ffeafe00, 0x0000000100000000)
 Metaspace       used 3380K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 363K, capacity 388K, committed 512K, reserved 1048576K
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at com.moralok.jvm.gc.JvmGcTest.main(JvmGcTest.java:21)
```


### 内存不足
当新生代和老年代的空间均不足时，在尝试 GC 和 Full GC 后仍不能成功分配对象，就会发生 `OutOfMemoryError`。


```java
public static void main(String[] args) {
    List<byte[]> list = new ArrayList<>();

    new Thread(() -> {
        list.add(new byte[_8MB]);
        list.add(new byte[_8MB]);
    }).start();

    System.out.println("waiting...");
    try {
        System.in.read();
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

```console
waiting...
[GC (Allocation Failure) [DefNew: 4711K->928K(9216K), 0.0017245 secs][Tenured: 8192K->9117K(10240K), 0.0021690 secs] 12903K->9117K(19456K), [Metaspace: 4267K->4267K(1056768K)], 0.0039336 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Allocation Failure) [Tenured: 9117K->9063K(10240K), 0.0014352 secs] 9117K->9063K(19456K), [Metaspace: 4267K->4267K(1056768K)], 0.0014614 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Exception in thread "Thread-0" java.lang.OutOfMemoryError: Java heap space
	at com.moralok.jvm.gc.JvmGcTest.lambda$main$0(JvmGcTest.java:27)
	at com.moralok.jvm.gc.JvmGcTest$$Lambda$1/2003749087.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:750)

Heap
 def new generation   total 9216K, used 1502K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  18% used [0x00000000fec00000, 0x00000000fed77a00, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
  to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
 tenured generation   total 10240K, used 9063K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,  88% used [0x00000000ff600000, 0x00000000ffed9c50, 0x00000000ffed9e00, 0x0000000100000000)
 Metaspace       used 4787K, capacity 4884K, committed 4992K, reserved 1056768K
  class space    used 522K, capacity 558K, committed 640K, reserved 1048576K
```

### 线程中发生内存不足，不会影响其他线程
当 `Thread-0` 发生 `OutOfMemoryError` 后，`main` 线程仍然正常运行。