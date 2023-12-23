---
title: synchronized 锁机制的分析和验证
date: 2023-12-19 12:09:07
tags: [java, lock, synchronized]
---

本文详细介绍了 `Java` 中 `synchronized` 锁的机制、存储结构、优化措施以及升级过程，并通过 `jol-core` 演示 `Mark Word` 的变化来验证锁升级的多个 `case`。

<!-- more -->

> 待完善

利用 `synchronized` 实现同步的基础：`Java` 中的每一个对象都可以作为锁。具体表现为以下 `3` 种形式。

- 对于普通同步方法，锁是当前实例对象。
- 对于静态同步方法，锁是当前类的 `Class` 对象。
- 对于同步方法块，锁是 `synchronized` 括号里配置的对象。

当一个线程试图访问同步代码块时，它首先必须得到锁，退出或抛出异常时必须释放锁。

- 在 `JVM` 层面，`synchronized` 锁是基于进入和退出 `Monitor` 来实现的，每一个对象都有一个 `Monitor` 与之相关联。
- 在字节码层面，同步方法块是使用 `monitorenter` 和 `monitorexit` 指令实现的，前者在编译后插入到同步方法块的开始位置，后者插入到同步方法块的结束位置和异常位置。

## 存储结构

> 锁存在哪里呢？锁里面又会存储什么信息呢？

### 对象头

`synchronized` 用的锁是存在 `Java` 对象头（`object header`）里的。如果对象是数组类型，则虚拟机用 `3` 字宽（`Word`）存储对象头，如果对象是非数组类型，则用 `2` 字宽存储对象头。在 `32` 位虚拟机中，`1` 字宽等于 `4` 字节，即 `32bit`。在 `64` 位虚拟机中，`1` 字宽等于 `8` 字节，即 `64bit`。

`Java` 对象头的组成结构如下：

|长度|内容|说明|
|--|--|--|
|`32/64bit`|`Mark Word`|**存储对象的 `hashCode` 或锁信息**|
|`32/64bit`|`Class Metadata Address`|存储指向对象类型数据的指针|
|`32/64bit`|`Array length`|数组的长度（如果当前对象是数组）|

### Mark Word

`Java` 对象头里的 `Mark Word` 里默认存储对象的 `HashCode`，分代年龄和锁标记位。在运行期间，`Mark Word` 里存储的数据会随着锁标志位的变化而变化。`Mark Word` 可能变化为另外 `4` 种数据。

以 `32` 位虚拟机为例：

<table>
  <tr>
    <th rowspan="2">锁状态</th>
    <th colspan="2">25bit</th>
    <th rowspan="2">4bit</th>
    <th>1bit</th>
    <th>2bit</th>
  </tr>
  <tr>
    <td>23bit</td>
    <td>2bit</td>
    <td>是否是偏向锁</td>
    <td>锁标志位</td>
  </tr>
  <tr>
    <td>无锁状态</td>
    <td colspan="2">对象的 hashCode</td>
    <td>对象分代年龄</td>
    <td>0</td>
    <td>01</td>
  </tr>
  <tr>
    <td>偏向锁</td>
    <td>线程 ID</td>
    <td>Epoch</td>
    <td>对象分代年龄</td>
    <td>1</td>
    <td>01</td>
  </tr>
  <tr>
    <td>轻量级锁</td>
    <td colspan="4">指向栈中锁记录的指针</td>
    <td>00</td>
  </tr>
  <tr>
    <td>重量级锁</td>
    <td colspan="4">指向互斥量（重量级锁）的指针</td>
    <td>10</td>
  </tr>
  <tr>
    <td>GC 标记</td>
    <td colspan="4">空</td>
    <td>11</td>
  </tr>
</table>

以 `64` 位虚拟机为例：

<table>
  <tr>
    <th rowspan="2">锁状态</th>
    <th colspan="2">56bit</th>
    <th>1bit</th>
    <th>4bit</th>
    <th>1bit</th>
    <th>2bit</th>
  </tr>
  <tr>
    <td>25bit</td>
    <td>31bit</td>
    <td>-</td>
    <td>-</td>
    <td>是否是偏向锁</td>
    <td>锁标志位</td>
  </tr>
  <tr>
    <td>无锁状态</td>
    <td>unused</td>
    <td>对象的 hashCode</td>
    <td>cms_free</td>
    <td>对象分代年龄</td>
    <td>0</td>
    <td>01</td>
  </tr>
  <tr>
    <td>偏向锁</td>
    <td  colspan="2">线程 ID(54bit) | Epoch(2bit)</td>
    <td>cms_free</td>
    <td>对象分代年龄</td>
    <td>1</td>
    <td>01</td>
  </tr>
  <tr>
    <td>轻量级锁</td>
    <td colspan="5">指向栈中锁记录的指针</td>
    <td>00</td>
  </tr>
  <tr>
    <td>重量级锁</td>
    <td colspan="5">指向互斥量（重量级锁）的指针</td>
    <td>10</td>
  </tr>
  <tr>
    <td>GC 标记</td>
    <td colspan="5">空</td>
    <td>11</td>
  </tr>
</table>

> 在上述表述中，很容易让人产生困惑的地方是 `hashCode` 和分代年龄是对象的固有属性，当 `Mark Word` 中存储的数据发生变化时，这些重要的数据去哪了？

### 内部结构可视化

“百闻不如一见”，`jol-core` 提供了打印对象内部结构的能力。

1. 添加依赖，新版本比旧版本打印结果的可读性更好
```xml
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>${org.openjdk.jol.version}</version>
</dependency>
```
2. 使用 `ClassLayout.parseInstance(objectExample).toPrintable()` 打印
```java
@Data
@Slf4j
public class ObjectInternalTest {

    private byte aByte;
    private int aInt;

    public static void main(String[] args) {
        ObjectInternalTest objectInternalTest = new ObjectInternalTest();
        log.info(ClassLayout.parseInstance(objectInternalTest).toPrintable());
    }
}
```
3. 打印结果：`mark|class|fields|alignment`。这样我们就能通过查看 `Mark Word` 的值更直观地确定当前锁的状态。
```text
2023-12-23 20:21:02 - com.moralok.concurrency.ch2.ObjectExample object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0x00060828
 12   4    int ObjectExample.aInt        0
 16   1   byte ObjectExample.aByte       0
 17   7        (object alignment gap)    
Instance size: 24 bytes
Space losses: 0 bytes internal + 7 bytes external = 7 bytes total
```

### 指针压缩和 cms_free

注意到指向对象类型数据的指针仅 `4` 个字节，这是因为默认情况下 `JVM` 参数 `UseCompressedOops` 是启用的。

```text
|--------------------------------------------------------------------------------------------------------------|--------------------|
|                                            Object Header (96 bits)                                           |        State       |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                  Mark Word (64 bits)                           |    Klass Word (32 bits)     |                    |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
| unused:25 | identity_hashcode:31 | cms_free:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Normal       |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
| thread:54 |       epoch:2        | cms_free:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Biased       |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                         ptr_to_lock_record                            | lock:2 |    OOP to metadata object   | Lightweight Locked |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                     ptr_to_heavyweight_monitor                        | lock:2 |    OOP to metadata object   | Heavyweight Locked |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                                                       | lock:2 |    OOP to metadata object   |    Marked for GC   |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
```

使用 `-XX:-UseCompressedOops` 关闭指针压缩，指向对象类型数据的指针才会变回 `8` 个字节

```text
|------------------------------------------------------------------------------------------------------------|--------------------|
|                                            Object Header (128 bits)                                        |        State       |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                  Mark Word (64 bits)                         |    Klass Word (64 bits)     |                    |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
| unused:25 | identity_hashcode:31 | unused:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Normal       |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
| thread:54 |       epoch:2        | unused:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Biased       |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                       ptr_to_lock_record:62                         | lock:2 |    OOP to metadata object   | Lightweight Locked |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                     ptr_to_heavyweight_monitor:62                   | lock:2 |    OOP to metadata object   | Heavyweight Locked |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                                                     | lock:2 |    OOP to metadata object   |    Marked for GC   |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
```

你可能还会注意到开启和关闭指针压缩时，还有一个 `bit` 从 `cms_free` 变成 `unused`。这个 `cms_free` 是做什么用的呢？在未开启指针压缩的情况下，指针的低位因为内存对齐的缘故往往是 `0`，我们可以给这些 `bit` 设置 `1` 用于标记特殊状态。`CMS` 将 `Klass` 指针的最低位设置为 `1` 用于表示特定的内存块不是一个对象，而是空闲的内存。在开启指针压缩后，`JVM` 通过右移移除指针中没用到的低位，因此 `CMS` 需要一个地方存储这个表示是否为空闲内存的 `bit`，就是 `cms_free`。

> 这在一定程度上解决了我心中的一个问题：`JVM` 是怎么判断一个空闲的内存块的？

[concurrentMarkSweepGeneration.cpp](https://hg.openjdk.org/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/gc_implementation/concurrentMarkSweep/concurrentMarkSweepGeneration.cpp)

```c++
// A block of storage in the CMS generation is always in
// one of three states. A free block (FREE), an allocated
// object (OBJECT) whose size() method reports the correct size,
// and an intermediate state (TRANSIENT) in which its size cannot
// be accurately determined.
// STATE IDENTIFICATION:   (32 bit and 64 bit w/o COOPS)
// -----------------------------------------------------
// FREE:      klass_word & 1 == 1; mark_word holds block size
//
// OBJECT:    klass_word installed; klass_word != 0 && klass_word & 1 == 0;
//            obj->size() computes correct size
//
// TRANSIENT: klass_word == 0; size is indeterminate until we become an OBJECT
//
// STATE IDENTIFICATION: (64 bit+COOPS)
// ------------------------------------
// FREE:      mark_word & CMS_FREE_BIT == 1; mark_word & ~CMS_FREE_BIT gives block_size
//
// OBJECT:    klass_word installed; klass_word != 0;
//            obj->size() computes correct size
//
// TRANSIENT: klass_word == 0; size is indeterminate until we become an OBJECT
```

使用 `java -cp .\sa-jdi.jar sun.jvm.hotspot.HSDB` 开启 `HotSpot Debugger`，比对 `ClassLayout` 打印的 `Klass` 指针和 `Class Browser` 中的指针。

|指针压缩|关闭|开启|
|--|--|--|
|ClassLayout|0xf800c105|0x00000245eb873d20|
|二进制表达|11111000000000001100000100000101|00100100010111101011100001110011110100100000|
|HotSpot Debugger|0x00000007c0060828|0x00000245EB873D20|
|二进制表达|011111000000000001100000100000101000|00100100010111101011100001110011110100100000|

### 对象分代年龄

通过以下示例可以测试和验证对象分代年龄的变化。

```java
public static void main(String[] args) throws InterruptedException {
    log.info("测试 Mark Word 中的分代年龄");

    Object lock = new Object();
    log.info("Mark Word 初始为 =====> 无锁状态，age: 0");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
    System.gc();
    TimeUnit.SECONDS.sleep(1);
    log.info("GC 后 =====> 无锁状态，age: 1");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
}
```

## 重量级锁

## 锁优化

`Java 6` 为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”，在 `Java 6` 中，锁一共有 `4` 种状态，级别从低到高依次是：**无锁状态**、**偏向锁状态**、**轻量级锁状态**和**重量级锁状态**，锁的状态会随着竞争的激化逐渐升级。**锁状态可以升级但不能降级**，举例来说偏向锁状态升级成轻量级锁状态后不能降级成偏向锁状态。这种锁升级却不能降级的策略，目的是为了提高获得锁和释放锁的效率。

> 上述的表述并不容易理解，甚至容易让人产生误解。锁状态描述的是锁本身的状态，和是否处于加锁状态无关。以下列表格举例说明，一个偏向锁状态的对象，即使未加锁，也是偏向锁状态，而非无锁状态。

|层次|未加锁|加锁|
|--|--|--|
|1|匿名偏向锁状态 or 偏向锁状态|偏向锁状态|
|2|无锁状态|轻量级锁状态|
|3|重要级锁状态|重要级锁状态|

> 在查阅的众多资料中，关于锁升级过程的介绍并不详尽和准确，虽然大体上大家的观点是比较一致的，但是在一些细节的描述上却有些模糊不清，有些观点自相矛盾，有些观点互相矛盾，有些观点和我的知识或者测试结果矛盾，甚至有些逻辑不通顺以至于不能相互联系形成和谐的整体。以下内容尽可能结合相对权威和详细的资料，补充个人的思考和猜想作为缝合剂，并通过一些测试用例验证部分猜想，试图建立更加连续平滑以及可信服的知识面。

### 锁升级变化图

提前放出锁升级变化图，用于在后续分析和测试过程中对照查看。重点关注以下可能引起锁状态变化的事件：

1. 获取锁和释放锁
2. 竞争，其中弱竞争是指线程交替进入同步块，没有发生直接冲突；强竞争是指线程在同步块内的时候有其他线程想要进入同步块
3. 调用特殊方法，比如计算 `hashCode`（非自定义）或者 `wait` 方法

<div style="width:80%;margin:auto">{% asset_img "Pasted image 20231224002101.png" 锁升级变化图 %}</div>

### 偏向锁

`HotSpot` 的作者经过研究发现，大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。

偏向锁在 `Java 6` 之后是默认开启的，可以通过 `JVM` 参数 `-XX:-UseBiasedLocking` 关闭偏向锁。尽管偏向锁是默认开启的，但是它在应用程序启动几秒钟之后才激活，延迟时间可以通过 `JVM` 参数 `-XX:BiasedLockingStartupDelay` 设置，默认情况下是 `4000ms`。

#### 测试偏向锁配置

##### 延迟偏向

通过以下示例测试并验证延迟偏向。

```java
public static void main(String[] args) throws IOException, InterruptedException {
    log.info("测试：偏向锁是延迟激活的");

    Object lock = new Object();
    log.info("Mark Word 初始为 =====> 无锁状态（非可偏向的）");
    log.info(ClassLayout.parseInstance(lock).toPrintable());

    // 默认情况下偏向延迟的设置为 -XX:BiasedLockingStartupDelay=4000
    log.info("sleep 4000ms，等待偏向锁激活");
    TimeUnit.MILLISECONDS.sleep(4000);

    log.info("偏向锁激活之后，新创建的对象的对象头的 Mark Word 是 =====> 匿名偏向锁");
    Object biasedLock = new Object();
    log.info(ClassLayout.parseInstance(biasedLock).toPrintable());

    log.info("偏向锁激活之前创建的对象的对象头的 Mark Word 仍然是 =====> 无锁状态");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
}
```

测试结果如下：

```text
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)

  0   8        (object header: mark)     0x0000000000000005 (biasable; age: 0)

  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
```

- `JVM` 启动后，**偏向锁尚未激活前**，创建的对象的 `Mark Word` 的末尾 `3` 位为 `0|01`，`non-biasable`，表示无锁状态（非可偏向的）。
- 在 `4000` 毫秒后，**新创建**的对象的 `Mark Word` 的末尾 `3` 位为 `1|01`，`biasable`，表示匿名偏向锁（可偏向的）。
- **偏向锁尚未激活前**创建的对象的对象头的 `Mark Word` 的末尾 `3` 位**仍然**是 `0|01`。

> 在虚拟机启动后，偏向锁激活前，创建的对象的锁标记位为 `1|01`，此时记录线程 `ID` 的 `bit` 全是 `0`（代表指向 `null`），没有偏向任何一个线程，该状态称之为匿名偏向锁。

##### 关闭偏向延迟

通过以下示例测试关闭偏向延迟：

```java
// JVM 参数设置为 -XX:BiasedLockingStartupDelay=0
public static void main(String[] args) throws IOException, InterruptedException {
    log.info("测试：关闭偏向锁的延迟偏向");

    Object lock = new Object();
    log.info("在虚拟机一启动，新创建的对象的对象头的 Mark Word 就是 =====> 匿名偏向锁");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
}
```

##### 关闭偏向锁

通过以下示例测试关闭偏向锁：

```java
// JVM 参数设置为 -XX:-UseBiasedLocking
public static void main(String[] args) throws InterruptedException {
    log.info("测试：关闭偏向锁");

    Object lock = new Object();
    log.info("Mark Word 初始为 =====> 无锁状态（非可偏向的）");
    log.info(ClassLayout.parseInstance(lock).toPrintable());

    log.info("sleep 4000ms");
    TimeUnit.MILLISECONDS.sleep(4000);

    log.info("即使过了偏向延迟时间，创建的对象的对象头的 Mark Word 仍然是 =====> 无锁状态（非可偏向的）");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
}
```

《Java 并发编程的艺术》中写的是 `-XX:-UseBiasedLocking=false`，测试中报错：

```text
Error: Could not create the Java Virtual Machine.
Error: A fatal exception has occurred. Program will exit.
Improperly specified VM option 'UseBiasedLocking=false'
```

另外书中说“在关闭偏向锁后程序默认会进入轻量级锁状态”，个人认为可能会让人产生误解，默认在未获取锁时为无锁状态，获取锁将变为轻量级锁状态。

#### 偏向锁加锁

当一个线程访问同步块时，先测试 `Mark Word` 里是否存储着当前线程 `ID`：
- 如果否，则再测试 `Mark Word` 中偏向锁的标识是否设置成 `1`
    - 如果为 `0`，则说明不是偏向锁状态 =====> *获取偏向锁失败后续处理一*
    - 如果为 `1`，则说明是偏向锁状态，**通过 `CAS` 操作设置偏向锁**
        - 如果成功，说明获得偏向锁
        - 如果失败，说明发生竞争 =====> *获取偏向锁失败后续处理二*
- 如果是，则说明当前线程就是之前获得偏向锁的线程，此刻再次获得锁

> 在**通过 `CAS` 操作设置偏向锁**中，`Compare` 操作是“测试 `Mark Word` 存储线程 `ID` 的 `bit` 是否全部为 `0`，代表偏向的线程 `ID` 为 `null`”，`Swap` 操作是将当前线程 `ID` 设置到 `Mark Word` 的相应位置。

补充思考：

- “通过 `CAS` 操作将当前线程 `ID` 设置到 `Mark Word`”在偏向锁状态下是有且仅有一次的“偏向”动作。（此观点存疑，在《Java 并发编程的艺术》一书中有“重新偏向于其他线程”这样的描述，但是关于竞争偏向锁部分的原理难以理解。个人在测试中，不论是持有偏向锁的线程仍存活但已离开同步块，还是已死亡，后续线程都无法再获取到偏向锁，唯一一种不同线程获取到同一个偏向锁的情况是两个线程可以复用同一个局部变量表槽位，它们的 `tid` 相同，这代表着本质上 `Mark Word` 并无变化）
- 当获得偏向锁的线程离开同步块时，没有“解锁操作”，`Mark Word` 维持不变。个人也不知道如何更准确地描述这个现象，从 `synchronized` 的语义来说，进出同步块代表着获取锁和释放锁；但是从偏向锁的实现来说，即便离开同步方法块，它仍然偏向原先获得锁的线程，甚至在讨论偏向锁发生竞争时，书中提到“检查持有偏向锁的线程是否存活”。个人更倾向于使用“撤销锁”一词描述偏向锁面临竞争时的处理，使用“释放锁”描述线程离开同步块时的处理。
- 当获得偏向锁的线程再次访问同步块时，简单测试 `Mark Word` 里存储着当前线程 `ID`，如果成功即可进入同步块。
- 计算过 `hashCode` 后偏向锁状态会变为其他状态，比如无锁状态，或者升级为轻量级锁甚至重量级锁，这符合 `CAS` 操作的判断条件。

#### 偏向锁撤销

偏向锁使用了一种等到竞争出现才撤销的机制，当获得偏向锁的线程离开同步块时，并没有“解锁操作”，`Mark Word` 将维持不变。当竞争出现时，从现象上说，如果持有偏向锁的线程已经离开同步块，则锁升级为轻量级锁；如果持有锁的线程尚未离开同步块，则锁直接升级为重量级锁。

> 关于偏向锁的撤销，其原理晦涩难懂，个人仍有很多疑问：锁记录中存储偏向的线程 `ID` 的作用，检查持有偏向锁的线程是否存活的作用不符合测试结果，重新偏向于其他线程的复现条件。因为理解有限，不多赘述。

#### 测试偏向锁升级

##### 匿名偏向锁->偏向锁

> 在一个匿名偏向锁状态的对象第一次被作为锁获取时，`Mark Word` 就会从匿名偏向锁变成偏向锁，并且再也不会变回到匿名偏向锁。

测试在匿名偏向锁状态下获取锁将变成偏向锁状态：

```java
public static void main(String[] args) throws IOException, InterruptedException {
    log.info("偏向锁基础测试：匿名偏向锁 -> 偏向锁");

    log.info("sleep 4000ms，等待偏向锁激活");
    TimeUnit.MILLISECONDS.sleep(4000);

    Object lock = new Object();
    log.info("Mark Word 初始为 =====> 匿名偏向锁");
    log.info(ClassLayout.parseInstance(lock).toPrintable());

    synchronized (lock) {
        log.info("{} 获取锁 =====> 偏向锁", Thread.currentThread().getName());
        log.info(ClassLayout.parseInstance(lock).toPrintable());

        log.info("暂停，输入任意字符回车继续，可以使用 jstack 查看线程 tid 和 Mark Word 进行对比");
        scanner.next();
    }

    log.info("偏向锁等到竞争出现才释放锁，因此离开同步方法块后，Mark Word 仍然不变");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
}
```

测试结果如下：

```text
2023-12-21 00:34:39 - 偏向锁基础测试：匿名偏向锁 -> 偏向锁
2023-12-21 00:34:39 - sleep 4000ms，等待偏向锁激活
2023-12-21 00:34:43 - Mark Word 初始为 =====> 匿名偏向锁
2023-12-21 00:34:45 - java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000005 (biasable; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

2023-12-21 00:34:45 - main 获取锁 =====> 偏向锁
2023-12-21 00:34:45 - java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000028761af3005 (biased: 0x00000000a1d86bcc; epoch: 0; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

2023-12-21 00:34:45 - 暂停，输入任意字符回车继续，可以使用 jstack 查看线程 tid 和 Mark Word 进行对比
2023-12-21 00:34:55 - 偏向锁等到竞争出现才释放锁，因此离开同步方法块后，Mark Word 仍然不变
2023-12-21 00:34:55 - java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000028761af3005 (biased: 0x00000000a1d86bcc; epoch: 0; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

通过 `jstack` 获取线程 `tid`（以 `Windows` 为例）：

```shell
jps | findstr "BiasedLockingBaseTest" | ForEach-Object { jstack $_.Split()[0]} | findstr "main"
"main" #1 prio=5 os_prio=0 tid=0x0000028761af3000 nid=0x8668 waiting on condition [0x000000ff7b8ff000]
        at com.moralok.concurrency.ch2.BiasedLockingBaseTest.main(BiasedLockingBaseTest.java:27)
```

关注 `Mark Word` 并转换为二进制表达：

||二进制表达|
|--|--|
|匿名偏向锁 `Mark Word`|00000000000000000000000000000000000000000101|
|偏向锁状态 `Mark Word`|00101000011101100001101011110011000000000101|
|`biased`|00000000000010100001110110000110101111001100|
|`main` 线程 `tid`|00101000011101100001101011110011000000000000|

- **注意**：存储的所谓“线程 `ID`”并非平时所说的线程 `ID`，该值左移可以得到 `jstack` 的返回结果中的 `tid`，`jol-core` 打印了一个名为 `biased` 的值与之相同
- 在离开同步方法块后，`Mark Word` 不变

##### 偏向锁->轻量级锁

测试当拥有偏向锁的线程已经离开同步块，其他线程尝试获取偏向锁（弱竞争），锁将升级为轻量级锁。

```java
public static void main(String[] args) throws InterruptedException {
    Scanner scanner = new Scanner(System.in);
    log.info("测试：当持有偏向锁的线程已经离开同步块，其他线程尝试获取偏向锁时，将获得轻量级锁");

    log.info("sleep 4000ms，等待偏向锁激活");
    TimeUnit.SECONDS.sleep(4);

    Object lock = new Object();
    log.info("Mark Word 初始为 =====> 匿名偏向锁");
    log.info(ClassLayout.parseInstance(lock).toPrintable());

    synchronized (lock) {
        log.info("第一个线程 {} 获取锁 =====> 偏向锁", Thread.currentThread().getName());
        log.info(ClassLayout.parseInstance(lock).toPrintable());
    }

    Thread thread = new Thread(() -> {
        log.info("第二个线程 {} 尝试获取锁", Thread.currentThread().getName());
        log.info(ClassLayout.parseInstance(lock).toPrintable());
        synchronized (lock) {
            log.info("第二个线程 {} 获取锁 =====> 轻量级锁", Thread.currentThread().getName());
            log.info(ClassLayout.parseInstance(lock).toPrintable());
        }
    }, "thread1");
    thread.start();
    thread.join();

    log.info("离开同步块后轻量级锁释放 =====> 无锁状态（可偏向的）");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
}
```

有相关资料提到在拥有偏向锁的线程死亡后，锁可以偏向新的线程，但是验证失败。

```java
public static void main(String[] args) throws IOException, InterruptedException {
    log.info("测试：之前获得偏向锁的线程已死时，新线程获得的仍然是偏向锁");

    log.info("sleep 4000ms，等待偏向锁激活");
    TimeUnit.MILLISECONDS.sleep(4000);

    Object lock = new Object();
    log.info("Mark Word 初始为 =====> 匿名偏向锁");
    log.info(ClassLayout.parseInstance(lock).toPrintable());

    Thread thread1 = new Thread(() -> {
        synchronized (lock) {
            log.info("第一个线程 {} 获取锁 =====> 偏向锁", Thread.currentThread().getName());
            log.info(ClassLayout.parseInstance(lock).toPrintable());
        }
    }, "thread1");
    thread1.start();

    Thread thread2 = new Thread(() -> {
        try {
            thread1.join();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        boolean alive = thread1.isAlive();
        log.info("第一个线程 {} 是否存活 {}", thread1.getName(), alive);
        log.info(ClassLayout.parseInstance(lock).toPrintable());
        synchronized (lock) {
            log.info("即使第一个线程已死亡，第二个线程 {} 获取锁 =====> 轻量级锁", Thread.currentThread().getName());
            log.info(ClassLayout.parseInstance(lock).toPrintable());
        }
    }, "thread2");
    thread2.start();
    thread2.join();

    log.info("离开同步块后轻量级锁释放 =====> 无锁状态（可偏向的）");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
}
```

##### 偏向锁->重量级锁

测试当拥有偏向锁的线程尚未离开同步块，其他线程尝试获取偏向锁（强竞争），锁将升级为重量级锁。

```java
public static void main(String[] args) throws InterruptedException {
    Scanner scanner = new Scanner(System.in);
    log.info("测试：当持有偏向锁的线程尚未离开同步块，其他线程尝试获取偏向锁时，将升级为重量级锁");

    log.info("sleep 4000ms，等待偏向锁激活");
    TimeUnit.SECONDS.sleep(4);

    Object lock = new Object();
    log.info("Mark Word 初始为 =====> 匿名偏向锁");
    log.info(ClassLayout.parseInstance(lock).toPrintable());

    Thread thread1 = new Thread(() -> {
        synchronized (lock) {
            log.info("第一个线程 {} 获取锁 =====> 偏向锁", Thread.currentThread().getName());
            log.info(ClassLayout.parseInstance(lock).toPrintable());

            log.info("暂停，输入任意字符回车继续");
            scanner.next();

            log.info("第一个线程 {} 持有偏向锁，在同步块内发生竞争 =====> 升级为重量级锁", Thread.currentThread().getName());
            log.info(ClassLayout.parseInstance(lock).toPrintable());
        }
        log.info("第一个线程 {} 结束", Thread.currentThread().getName());
    }, "thread1");
    thread1.start();

    TimeUnit.SECONDS.sleep(1);

    Thread thread2 = new Thread(() -> {
        log.info("第二个线程 {} 尝试获取偏向锁失败", Thread.currentThread().getName());
        synchronized (lock) {
            log.info("第二个线程 {} 获取锁 =====> 重量级锁", Thread.currentThread().getName());
            log.info(ClassLayout.parseInstance(lock).toPrintable());
        }
    }, "thread2");
    thread2.start();
    thread2.join();

    log.info("即使离开同步块后 =====> 重量级锁");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
}
```

##### 特例：偏向锁->偏向锁

这是一个很奇怪的测试用例，它是在测试中唯一发生不同线程对同一个锁获得偏向锁的情况。但是排查过程中发现两个线程的 `tid` 相同，猜测是局部变量表槽位复用时有什么优化机制。

> 卡了我好久，也没有探究到实质的新信息。

```java
public static void main(String[] args) throws IOException, InterruptedException {
    log.info("测试：之前获得偏向锁的线程已死时，新线程获得的仍然是偏向锁");

    log.info("sleep 4000ms，等待偏向锁激活");
    TimeUnit.MILLISECONDS.sleep(4000);

    Object lock = new Object();
    log.info("Mark Word 初始为 =====> 匿名偏向锁");
    log.info(ClassLayout.parseInstance(lock).toPrintable());

    Thread thread1 = new Thread(() -> {
        synchronized (lock) {
            log.info("第一个线程 {} 获取锁 =====> 偏向锁", Thread.currentThread().getName());
            log.info(ClassLayout.parseInstance(lock).toPrintable());
        }
    }, "thread1");
    thread1.start();
    thread1.join();

    Thread thread2 = new Thread(() -> {
        synchronized (lock) {
            log.info("第二个线程 {} 获取锁，=====> 偏向锁", Thread.currentThread().getName());
            log.info("震惊！！！为什么两个 tid 相同啊，有什么复用机制吗");
            log.info(ClassLayout.parseInstance(lock).toPrintable());
        }
    }, "thread2");
    thread2.start();
    thread2.join();

    log.info("偏向锁等到竞争出现才释放锁，因此离开同步方法块后，Mark Word 不变");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
}
```

##### 匿名偏向锁状态计算 hashCode

在匿名偏向锁状态计算 `hashCode`，锁将变为无锁状态。

```java
public static void main(String[] args) throws InterruptedException {
    log.info("测试：在匿名偏向锁状态计算 hashCode");

    log.info("sleep 4000ms，等待偏向锁激活");
    TimeUnit.MILLISECONDS.sleep(4000);

    Object lock = new Object();
    log.info("Mark Word 初始为 =====> 匿名偏向锁");
    log.info(ClassLayout.parseInstance(lock).toPrintable());

    int hashCode = lock.hashCode();
    log.info("在计算 hashCode 后：Mark Word =====> 无锁状态（hash|age|0|01）");
    log.info(ClassLayout.parseInstance(lock).toPrintable());

    synchronized (lock) {
        log.info("获取锁 =====> 轻量级锁");
        log.info(ClassLayout.parseInstance(lock).toPrintable());
    }

    log.info("离开同步块后轻量级锁释放 =====> 无锁状态（hash|age|0|01）");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
}
```

##### 偏向锁状态无锁时计算 hashCode

在偏向锁状态无锁时计算 `hashCode`，锁将变为无锁状态。

```java
public static void main(String[] args) throws InterruptedException {
    log.info("测试：在偏向锁状态无锁时计算 hashCode");

    log.info("sleep 4000ms，等待偏向锁激活");
    TimeUnit.MILLISECONDS.sleep(4000);

    Object lock = new Object();
    log.info("Mark Word 初始为 =====> 匿名偏向锁");
    log.info(ClassLayout.parseInstance(lock).toPrintable());

    synchronized (lock) {
        log.info("获取锁 =====> 偏向锁");
        log.info(ClassLayout.parseInstance(lock).toPrintable());
    }

    int hashCode = lock.hashCode();
    log.info("在计算 hashCode 后：Mark Word =====> 无锁状态");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
}
```

##### 偏向锁状态加锁时计算 hashCode

在偏向锁状态加锁时计算 `hashCode`，锁将升级为重量级锁状态。

```java
public static void main(String[] args) throws InterruptedException {
    log.info("测试：在偏向锁状态计算 hashCode");

    log.info("sleep 4000ms，等待偏向锁激活");
    TimeUnit.MILLISECONDS.sleep(4000);

    Object lock = new Object();
    log.info("Mark Word 初始为 =====> 匿名偏向锁");
    log.info(ClassLayout.parseInstance(lock).toPrintable());

    synchronized (lock) {
        log.info("获取锁 =====> 偏向锁");
        log.info(ClassLayout.parseInstance(lock).toPrintable());

        int hashCode = lock.hashCode();
        log.info("在计算 hashCode 后：Mark Word =====> 重量级锁");
        log.info(ClassLayout.parseInstance(lock).toPrintable());
    }

    log.info("即使离开同步块后 =====> 重量级锁");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
}
```

### 轻量级锁

#### 轻量级锁加锁

获取偏向锁失败后续处理一（是否是偏向锁为 `0`）：
- 检测锁标志位是否为 `01` 或者 `00`
    - 如果否，则说明是重量级锁状态 =====> *获取轻量级锁失败后续处理一*
    - 如果是，则说明是无锁状态或者轻量级锁状态，尝试**通过 `CAS` 操作设置轻量级锁**
        - 如果成功，说明获得轻量级锁
        - 如果失败，说明发生竞争 =====> *获取轻量级锁失败后续处理二*

> 在**通过 `CAS` 操作设置轻量级锁**中，`Compare` 操作是“测试 `Mark Word` 的锁标志位是否为 `01`，代表处于无锁状态”，`Swap` 操作是将 `Mark Word` 复制到栈中锁记录，并将指向栈中锁记录的指针设置到 `Mark Word` 的相应位置以及修改锁标志位。所谓“栈中锁记录”又称为 `Displaced Mark Word`，`JVM` 会在当前线程的栈帧中创建用于存储锁记录的空间，用于在轻量级锁状态下临时存放 `Mark Word`。

> 在轻量级锁状态下，明确提及了锁记录的作用，但偏向锁状态下，提及锁记录却并未加以解释。

获取偏向锁失败后续处理一：
- 已经升级为重量级锁

获取偏向锁失败后续处理二（通过 `CAS` 加偏向锁失败）：
- 获取锁失败的线程将锁升级为重量级锁，修改 `Mark Word` 为`指向互斥量（重量级锁）的指针|10`（这个操作将影响到持有轻量级锁的线程的解锁）
- 线程阻塞，等待唤醒

补充思考：

- 有相关资料提及偏向锁并非直接升级到重量级锁，无法验证是否总是有轻量级锁作为中间状态
- 轻量级锁面临竞争时升级为重量级锁的过程相比于偏向锁面临竞争时的升级过程，更加容易理解，后者好多细节没有找到令人信服的答案。

#### 轻量级锁解锁

轻量级锁解锁时，会通过 `CAS` 操作解锁，`Compare` 操作是“测试 `Mark Word` 的锁标志位是否为 `00`，代表处于轻量级锁状态，`Swap` 操作是将栈中锁记录 `Dispaced Mark Word` 替换回对象头的 `Mark Word` 以及修改锁标志位。
如果 `Compare` 操作失败，则代表发生竞争，此时锁已经被其他线程升级为重量级锁以及 `Mark Word` 被修改为`指向互斥量（重量级锁）的指针|10`。持有轻量级锁的线程会释放锁（直接将 `Dispaced Mark Word` 替换回 `Mark Word`？）并唤醒等待的线程，开启新的一轮争抢。

#### 测试轻量级锁升级

##### 无锁->轻量级锁

测试在无锁状态下获取锁，锁将变成轻量级锁状态。

```java
public static void main(String[] args) throws IOException, InterruptedException {
    Scanner scanner = new Scanner(System.in);
    log.info("轻量级锁基础测试：无锁状态 -> 轻量级锁");

    Object lock = new Object();
    log.info("在偏向锁激活之前创建的对象为 =====> 无锁状态（可偏向额）");
    log.info(ClassLayout.parseInstance(lock).toPrintable());

    synchronized (lock) {
        log.info("即使是单线程无竞争获取锁，=====> 轻量级锁");
        log.info(ClassLayout.parseInstance(lock).toPrintable());
        log.info("暂停，回车继续");
        scanner.nextLine();
    }

    log.info("离开同步块后，-> 无锁状态（可偏向的）");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
}
```

##### 无锁状态计算 hashCode

在无锁状态计算 `hashCode`，仍然是无锁状态。

```java
public static void main(String[] args) throws InterruptedException {
    log.info("测试：在无锁状态计算 hashCode");

    Object lock = new Object();
    log.info("Mark Word 初始为 =====> 无锁状态");
    log.info(ClassLayout.parseInstance(lock).toPrintable());

    int hashCode = lock.hashCode();
    log.info("在计算 hashCode 后：Mark Word =====> 无锁状态（hash|age|0|01）");
    log.info(ClassLayout.parseInstance(lock).toPrintable());

    synchronized (lock) {
        log.info("获取锁 =====> 轻量级锁");
        log.info(ClassLayout.parseInstance(lock).toPrintable());
    }

    log.info("离开同步块后轻量级锁释放 =====> 无锁状态（hash|age|0|01）");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
}
```

##### 轻量级锁加锁时计算 hashCode

在轻量级锁状态加锁时计算 `hashCode`，锁将升级为重量级锁状态。

```java
public static void main(String[] args) throws InterruptedException {
    log.info("测试：在轻量级锁状态计算 hashCode");

    Object lock = new Object();
    log.info("Mark Word 初始为 =====> 无锁状态");
    log.info(ClassLayout.parseInstance(lock).toPrintable());

    synchronized (lock) {
        log.info("获取锁 =====> 轻量级锁");
        log.info(ClassLayout.parseInstance(lock).toPrintable());

        int hashCode = lock.hashCode();
        log.info("在计算 hashCode 后：Mark Word =====> 重量级锁");
        log.info(ClassLayout.parseInstance(lock).toPrintable());
    }

    log.info("即使离开同步块后 =====> 重量级锁");
    log.info(ClassLayout.parseInstance(lock).toPrintable());
}
```

## 参考文章

- 《Java 并发编程的艺术》
- [难搞的偏向锁终于被 Java 移除了](https://www.cnblogs.com/FraserYu/p/15743542.html)
- [Details about mark word of java object header](https://stackoverflow.com/questions/60985782/details-about-mark-word-of-java-object-header)