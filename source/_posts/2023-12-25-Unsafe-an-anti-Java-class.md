---
title: Unsafe，一个“反 Java”的 class
date: 2023-12-25 08:27:04
tags: [java]
---

`Unsafe` 类位于 `sun.misc` 包中，它提供了一组用于执行低级别、不安全操作的方法。尽管 `Unsafe` 类及其所有方法都是公共的，但它的使用受到限制，因为只有受信任的代码才能获取其实例。这个类通常被用于一些底层的、对性能敏感的操作，比如直接内存访问、`CAS`（`Compare and Swap`）操作等。本文将介绍这个“反 `Java`”的类及其方法的典型使用场景。

<!-- more -->

> 由于 `Unsafe` 类涉及到直接内存访问和其他底层操作，使用它需要极大的谨慎，因为它可以绕过 `Java` 语言的一些安全性和健壮性检查。在正常的应用程序代码中，最好避免直接使用 `Unsafe` 类，以确保代码的可读性和可维护性。在一些特殊情况下，比如一些高性能库的实现，可能会使用 `Unsafe` 类来进行一些性能优化。

> 尽管在生产中需要谨慎使用 `Unsafe`，但是可以在测试中使用它来更真实地接触 `Java` 对象在内存中的存储结构，验证自己的理论知识。

## 获取 Unsafe 实例

> 在 `Java 9` 及之后的版本中，`Unsafe` 类中的 `getUnsafe()` 方法被标记为不安全（`Unsafe`），不再允许普通的 `Java` 应用程序代码通过此方法获取 `Unsafe` 实例。这是为了提高 `Java` 的安全性，防止滥用 `Unsafe` 类的功能。

在正常的 `Java` 应用程序中，获取 `Unsafe` 实例是不被推荐的，因为它违反了 `Java` 语言的安全性和封装原则。`Unsafe` 类的设计本意是为了 `Java` 库和虚拟机的实现使用，而不是为了普通应用程序开发者使用。`Unsafe` 对象为调用者提供了执行不安全操作的能力，它可用于在任意内存地址读取和写入数据，因此返回的 `Unsafe` 对象应由调用者仔细保护。它绝不能传递给不受信任的代码。此类中的大多数方法都是非常低级的，并且对应于少量硬件指令。

获取 `Unsafe` 实例的静态方法如下：

```java
@CallerSensitive
public static Unsafe getUnsafe() {
    Class<?> caller = Reflection.getCallerClass();
    // 检查调用方法的类是被引导类加载器所加载
    if (!VM.isSystemDomainLoader(caller.getClassLoader()))
        throw new SecurityException("Unsafe");
    return theUnsafe;
}
```

`Unsafe` 使用单例模式，可以通过静态方法 `getUnsafe` 获取 `Unsafe` 实例，并且调用方法的类为启动类加载器所加载才不会抛出异常。获取 `Unsafe` 实例有以下两种可行方案：
1. 通过 `-Xbootclasspath/a:${path}` 把调用方法的类所在的 `jar` 包路径追加到启动类路径中，使该类被启动类加载器加载。关于启动类路径的信息可以参考[Java 类加载器源码分析 | ClassLoader 的搜索路径](https://www.moralok.com/2023/07/13/Java-class-loader-source-code-analysis/#ClassLoader-%E7%9A%84%E6%90%9C%E7%B4%A2%E8%B7%AF%E5%BE%84)
2. 通过反射获取 `Unsafe` 类中的 `Unsafe` 实例
```java
private static Unsafe getUnsafe() {
    try {
        Field f = Unsafe.class.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        return (Unsafe) f.get(null);
    } catch (NoSuchFieldException | IllegalAccessException e) {
        throw new RuntimeException(e);
    }
}
```

## 内存操作

`Unsafe` 类中包含了一些关于内存操作的方法，这些方法通常被认为是不安全的，因为它们可以绕过 `Java` 语言的内置安全性和类型检查。以下是一些常见的 `Unsafe` 类中关于内存操作的方法：

- `allocateMemory`: 分配一个给定大小（以字节为单位）的本地内存块，内容未初始化，通常是垃圾。生成的本地指针永远不会为零，并且将针对所有类型进行对齐。
```java
public native long allocateMemory(long bytes);
```
- `reallocateMemory`: 将本地内存块的大小调整为给定大小（以字节为单位），超过旧内存块大小的内容未初始化，通常是垃圾。当且仅当请求的大小为零时，生成的本地指针才为零。传递给此方法的地址可能为空，在这种情况下将执行分配。
```java
public native long reallocateMemory(long address, long bytes);
```
- `freeMemory`: 释放之前由 `allocateMemory` 或 `reallocateMemory` 分配的内存。
```java
public native void freeMemory(long address);
```
- `setMemory`: 将给定内存块中的所有字节设置为固定值（通常为零）。
```java
public native void setMemory(Object o, long offset, long bytes, byte value);
public void setMemory(long address, long bytes, byte value) {
    setMemory(null, address, bytes, value);
}
```
- `copyMemory`: 复制指定长度的内存块
```java
public native void copyMemory(Object srcBase, long srcOffset, Object destBase, long destOffset, long bytes);
```
- `putXxx`: 将指定偏移量处的内存设置为指定的值，其中 `Xxx` 可以是 `Object`、`int`、`long`、`float` 和 `double` 等。
```java
public native void putObject(Object o, long offset, Object x);
```
- `getXxx`: 从指定偏移量处的内存读取值，其中 `Xxx` 可以是 `Object`、`int`、`long`、`float` 和 `double` 等。
```java
public native Object getObject(Object o, long offset);
```
- `putXxx` 和 `getXxx` 也提供了按绝对基地址操作内存的方法。
```java
public native byte getByte(long address);
public native void putByte(long address, byte x);
```

从内存读取值时，除非满足以下情况之一，否则结果不确定：
1. 偏移量是通过 `objectFieldOffset` 从字段的 `Field` 对象获取的，`o` 指向的对象的类与字段所属的类兼容。
2. 偏移量和 `o` 指向的对象（无论是否为 `null`）分别是通过 `staticFieldOffset` 和 `staticFieldBase` 从 `Field` 对象获得的。
3. `o` 指向的是一个数组，偏移量是一个形式为 `B+N*S` 的整数，其中 `N` 是数组的有效索引，`B` 和 `S` 分别是通过 `arrayBaseOffset` 和 `arrayIndexScale` 获得的值。

> 做一些“不确定”的测试，比如使用 `byte` 相关的方法操作 `int` 所在的内存块，是有意思且有帮助的，了解如何破坏，也可以更好地学习如何保护。

### 分配堆外内存

在 `Java NIO`（`New I/O`）中，分配堆外内存使用了 `Unsafe` 类的 `allocateMemory` 方法。堆外内存是一种在 `Java` 虚拟机之外分配的内存，它不受 `Java` 堆内存管理机制的控制。这种内存分配的主要目的是提高 `I/O` 操作的性能，因为它可以直接与底层操作系统进行交互，而不涉及 `Java` 堆内存的复杂性。Java 虚拟机的垃圾回收器虽然不直接管理这块内存，但是它通过一种称为“引用清理”（`Reference Counting`）的机制来处理。

```java
DirectByteBuffer(int cap) {
    super(-1, 0, cap, cap);
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    Bits.reserveMemory(size, cap);

    long base = 0;
    try {
        // 分配本地内存
        base = unsafe.allocateMemory(size);
    } catch (OutOfMemoryError x) {
        Bits.unreserveMemory(size, cap);
        throw x;
    }
    // 初始化本地内存
    unsafe.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
        // Round up to page boundary
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
    // 使用虚引用 Cleaner 对象跟踪 DirectByteBuffer 对象的垃圾回收
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
}
```

当 `DirectByteBuffer` 对象仅被 `Cleaner` 对象（虚引用）引用时，它可以在任意一次 `GC` 中被垃圾回收。在 `DirectByteBuffer` 对象被垃圾回收后，`Cleaner` 对象会被加入到引用队列，`ReferenceHandler` 线程将调用 `Deallocator` 对象的 `run` 方法，从而实现本地内存的自动释放。

```java
private static class Deallocator implements Runnable {

    private static Unsafe unsafe = Unsafe.getUnsafe();

    private long address;
    private long size;
    private int capacity;

    private Deallocator(long address, long size, int capacity) {
        assert (address != 0);
        this.address = address;
        this.size = size;
        this.capacity = capacity;
    }

    public void run() {
        if (address == 0) {
            // Paranoia
            return;
        }
        // 释放本地内存
        unsafe.freeMemory(address);
        address = 0;
        Bits.unreserveMemory(size, capacity);
    }

}
```

## CAS 相关

`Unsafe` 提供了 `3` 个 `CAS` 相关操作的方法，方法将内存位置的值与预期原值比较，如果相匹配，则 `CPU` 会自动将该位置更新为新值，否则，`CPU` 不做任何操作。这些方法的底层实现对应着 `CPU` 指令 `cmpxchg`。

```java
// 如果 Java 变量当前符合预期，则自动将其更新为 x。
public final native boolean compareAndSwapObject(Object o, long offset, Object expected, Object x);
public final native boolean compareAndSwapInt(Object o, long offset, int expected, int x);
public final native boolean compareAndSwapLong(Object o, long offset, long expected, long x);
```

在 `AtomicInteger` 的实现中，静态字段 `valueOffset` 即为字段 `value` 的内存偏移地址，`valueOffset` 的值在 `AtomicInteger` 初始化时，在静态代码块中通过 `Unsafe` 的 `objectFieldOffset` 方法获取。

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
}
```

CAS 更新变量的值的内存变化如下：

<div style="width:80%;margin:auto">{% asset_img "Pasted image 20231226005107.png" CAS 更新变量的值 %}</div>

配合 `ClassLayout` 打印 `AtomicInteger` 的内部结构更直观地感受 `offset` 的含义：

```text
java.util.concurrent.atomic.AtomicInteger object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0xf8003dbc
 12   4    int AtomicInteger.value       1
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```



## 参考文章

- [Java魔法类：Unsafe应用解析](https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html)