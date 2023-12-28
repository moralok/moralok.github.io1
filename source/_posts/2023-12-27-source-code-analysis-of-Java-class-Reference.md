---
title: Java 类 Reference 的源码分析
date: 2023-12-27 09:43:28
tags: [java]
---

我们知道 `Java` 扩充了“引用”的概念，引入了软引用、弱引用和虚引用，它们都属于 `Reference` 类型，也都可以配合 `ReferenceQueue` 使用。你是否好奇常常被一笔带过的“`引用对象`的处理过程”？你是否在探究 `NIO` 堆外内存的自动释放时看到了 `Cleaner` 的关键代码但不太能梳理整个过程？你是否好奇在研究 `JVM` 时偶尔看到的 `Reference Handler` 线程？本文将分析 `Reference` 和 `ReferenceQueue` 的源码带你理解`引用对象`的工作机制。

<!-- more -->

> 事实上，个人感觉在无相关前置知识的情况下，单纯看 `JDK` 的 `Java` 代码是没办法很好地理解`引用对象`是如何被添加到`引用队列`中的。因为 `Reference` 的 `pending` 字段的含义和赋值操作是隐藏在 `JVM` 的 `C++` 代码中，本文搁置了其中的细节，仅分析 `JDK` 中相关的 `Java` 代码。

## Reference

`Reference` 是`引用对象`的抽象基类。此类定义了所有引用对象通用的操作。由于引用对象是与垃圾收集器密切合作实现的，因此该类可能无法直接子类化。

<div style="width:70%;margin:auto">{% asset_img "Pasted image 20231227230915.png" Reference 相关类图 %}</div>

### 构造函数

- `referent`: `引用对象`关联的对象
- `queue`: `引用对象`准备注册到的`引用队列`

`Reference` 提供了两个构造函数，一个需要传入`引用队列`（`ReferenceQueue`），一个不需要。如果一个`引用对象`（`Reference`）注册到一个`引用队列`，在检测到关联对象有适当的可达性变化后，垃圾收集器将把该`引用对象`添加到该引用队列。

> “关联对象有适当的可达性变化”并不容易理解，在很多表述中它很容易被简化为“可以被回收”，但是同时我们又拥有另一条规则，即“一个对象是否可回收的判断依据是是否从 `Root` 对象可达”。在面对 `Reference` 的子类时，我们有种割裂感，好像一条和谐的规则出现了特殊条例。{% post_link explore-the-Java-classes-Cleaner-and-Finalizer '探索 Java 类 Cleaner 和 Finalizer' %}

```java
Reference(T referent) {
    this(referent, null);
}

Reference(T referent, ReferenceQueue<? super T> queue) {
    this.referent = referent;
    // ReferenceQueue.NULL 表示没有注册到引用队列
    this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
}
```

### 属性

#### 成员变量

- `referent`: `引用对象`关联的对象，**该对象将被垃圾收集器特殊对待**。我们很难直观地感受何谓“被垃圾收集器特殊对待”，它对应着“在检测到关联对象有适当的可达性变化后，垃圾收集器将把`引用对象`添加到该引用队列”。
- `queue`: `引用对象`注册到的`引用队列`
- `next`: 用于指向下一个`引用对象`，当`引用对象`已经添加到`引用队列`中，`next` 指向`引用队列`中的下一个`引用对象`
- `discovered`: 用于指向下一个`引用对象`，用于在全局的 `pending` 链表中，指向下一个待添加到`引用队列`的`引用对象`

<div style="width:60%;margin:auto">{% asset_img "Pasted image 20231227192404.png" 引用对象 %}</div>

#### 静态变量

> 注意：`lock` 和 `pending` 是全局共享的。

- `lock`: 用于与垃圾收集器同步的对象，**垃圾收集器必须在每个收集周期开始时获取此锁**。因此至关重要的是持有此锁的任何代码必须尽快运行完，不分配新对象并避免调用用户代码。
- `pending`: 等待加入`引用队列`的`引用对象`链表。垃圾收集器将`引用对象`添加到 `pending` 链表中，而 `Reference-Handler` 线程将删除它们，并做清理或入队操作。`pending` 链表受上述 `lock` 对象的保护，并使用 `discovered` 字段来链接下一个元素。

```java
public abstract class Reference<T> {
    private T referent;         /* Treated specially by GC */

    volatile ReferenceQueue<? super T> queue;
    @SuppressWarnings("rawtypes")
    volatile Reference next;

    transient private Reference<T> discovered;  /* used by VM */

    static private class Lock { }
    private static Lock lock = new Lock();

    private static Reference<Object> pending = null;
}
```

> `Reference` 其实可以理解为单链表中的一个节点，除了核心的 `referent` 和 `queue`，`next` 和 `discovered` 都用于指向下一个`引用对象`，只是分别用于两条不同的单链表上。

`pending` 链表：

<div style="width:70%;margin:auto">{% asset_img "Pasted image 20231227191324.png" pending 链表 %}</div>

`ReferenceQueue`：

<div style="width:70%;margin:auto">{% asset_img "Pasted image 20231227191330.png" ReferenceQueue %}</div>

### ReferenceHandler 线程

启动任意一个非常简单的 `Java` 程序，通过 `JVM` 相关的工具，比如 `JConsole`，你都能看到一个名为 `Reference Handler` 的线程。

<div style="width:60%;margin:auto">{% asset_img "Snipaste_2023-12-27_19-41-19.png" Reference-Handler 线程 %}</div>

`ReferenceHandler` 类本身的代码并不复杂。

```java
private static class ReferenceHandler extends Thread {
    // 确保类已经初始化
    private static void ensureClassInitialized(Class<?> clazz) {
        try {
            Class.forName(clazz.getName(), true, clazz.getClassLoader());
        } catch (ClassNotFoundException e) {
            throw (Error) new NoClassDefFoundError(e.getMessage()).initCause(e);
        }
    }

    static {
        // 预加载和初始化 InterruptedException 和 Cleaner，以避免在 run 方法中懒加载发生内存不足时陷入麻烦（咱也不知道具体啥麻烦）
        ensureClassInitialized(InterruptedException.class);
        ensureClassInitialized(Cleaner.class);
    }

    ReferenceHandler(ThreadGroup g, String name) {
        super(g, name);
    }

    public void run() {
        // run 方法循环调用 tryHandlePending
        while (true) {
            tryHandlePending(true);
        }
    }
}
```

#### 创建线程并启动

`Reference-Handler` 线程是通过静态代码块创建并启动的。

```java
static {
    // 不断获取父线程组，直到最高的系统线程组
    ThreadGroup tg = Thread.currentThread().getThreadGroup();
    for (ThreadGroup tgn = tg;
            tgn != null;
            tg = tgn, tgn = tg.getParent());
    Thread handler = new ReferenceHandler(tg, "Reference Handler");
    // 设置为最高优先级
    handler.setPriority(Thread.MAX_PRIORITY);
    // 设置为守护线程
    handler.setDaemon(true);
    handler.start();

    // provide access in SharedSecrets
    // 不懂，看到一个说法覆盖 JVM 的默认处理方式
    SharedSecrets.setJavaLangRefAccess(new JavaLangRefAccess() {
        @Override
        public boolean tryHandlePendingReference() {
            return tryHandlePending(false);
        }
    });
}
```

#### run 处理逻辑

`run` 方法的核心处理逻辑。本质上，`ReferenceHandler` 线程将 `pending` 链表上的`引用对象`分发到各自注册的`引用队列`中。如果理解了 `Reference` 作为单链表节点的一面，这部分代码不难理解，反而是其中应对 `OOME` 的处理很值得关注，但更多的可能是看了个寂寞，不好重现问题并验证。

```java
static boolean tryHandlePending(boolean waitForNotify) {
    Reference<Object> r;
    Cleaner c;
    try {
        // 加锁（和垃圾回收共用一个锁）
        synchronized (lock) {
            // 如果不为 null
            if (pending != null) {
                // 获取头节点
                r = pending;
                // instanceof 可能抛出 OutOfMemoryError，因此在把 r 从 pending 链表中移除前进行
                // 如果是 Cleaner 类型，进行类型转换，后续有特殊处理
                c = r instanceof Cleaner ? (Cleaner) r : null;
                // 从 pending 链表移除 r
                pending = r.discovered;
                r.discovered = null;
            } else {
                // 等待锁可能抛出 OutOfMemoryError，因为可能需要分配 exception 对象
                if (waitForNotify) {
                    lock.wait();
                }
                // retry if waited
                return waitForNotify;
            }
        }
    } catch (OutOfMemoryError x) {
        // 给其他线程 CPU 时间，以便它们能够丢弃一些存活的引用，然后通过 GC 回收一些空间
        // 还可以防止 CPU 密集运行以至于上面的“r instanceof Cleaner”在一段时间内持续抛出 OOME
        Thread.yield();
        // retry
        return true;
    } catch (InterruptedException x) {
        // retry
        return true;
    }

    // 如果是 Cleaner 类型，快速清理并返回
    if (c != null) {
        c.clean();
        return true;
    }

    // 如果 Reference 对象关联了引用队列，则添加到队列
    ReferenceQueue<? super Object> q = r.queue;
    if (q != ReferenceQueue.NULL) q.enqueue(r);
    return true;
}
```

### 关联对象和队列相关方法

```java
/* -- Referent accessor and setters -- */

// 获取关联对象
public T get() {
    return this.referent;
}

// 清理关联对象，该操作不会导致引用对象入队
public void clear() {
    this.referent = null;
}

/* -- Queue operations -- */

// 判断引用对象是否已入队，如果未关联引用队列，则返回 false
public boolean isEnqueued() {
    return (this.queue == ReferenceQueue.ENQUEUED);
}

// 将引用对象添加到其注册的引用队列中，该方法仅 Java 代码调用，JVM 不需要调用本方法可以直接进行入队操作（什么情况下？）
public boolean enqueue() {
    return this.queue.enqueue(this);
}
```

## ReferenceQueue

`引用队列`，在检测到适当的可达性更改后，垃圾收集器将已注册的`引用对象`添加到该队列。

### 属性

```java
public class ReferenceQueue<T> {

    // 构造函数
    public ReferenceQueue() { }
    
    // 一个不可入队的队列
    private static class Null<S> extends ReferenceQueue<S> {
        boolean enqueue(Reference<? extends S> r) {
            return false;
        }
    }
    // 用于表示一个引用对象没有注册到引用队列
    static ReferenceQueue<Object> NULL = new Null<>();
    // 用于表示一个引用对象已经添加到引用队列
    static ReferenceQueue<Object> ENQUEUED = new Null<>();
    
    // 锁对象
    static private class Lock { };
    private Lock lock = new Lock();
    // 头节点
    private volatile Reference<? extends T> head = null;
    // 队列长度
    private long queueLength = 0;
}
```

### 入队

`enqueue` 只能由 `Reference` 类调用。

`引用对象`的 `queue` 字段可以表达`引用对象`的状态：

- `NULL`：表示没有注册到`引用队列`或者已经从`引用队列`中移除
- `ENQUEUED`：表示已经添加到`引用队列`

```java
boolean enqueue(Reference<? extends T> r) {
    synchronized (lock) {
        // 检查引用对象的状态是否可以入队
        ReferenceQueue<?> queue = r.queue;
        if ((queue == NULL) || (queue == ENQUEUED)) {
            return false;
        }
        // 检查注册的 queue 和调用的 queue 是否相同
        assert queue == this;
        // 标记为已入队
        r.queue = ENQUEUED;
        // 头插法，最后一个节点的 next 指向自身（为什么？）
        r.next = (head == null) ? r : head;
        head = r;
        // 队列长度加一
        queueLength++;
        if (r instanceof FinalReference) {
            sun.misc.VM.addFinalRefCount(1);
        }
        // 通知等待的线程
        lock.notifyAll();
        return true;
    }
}
```

### 出队

轮询队列以查看是否有引用对象可用，如果存在可用的引用对象则将其从队列中删除并返回，否则该方法立即返回 `null`。

```java
public Reference<? extends T> poll() {
    // 缩小锁的范围
    if (head == null)
        return null;
    synchronized (lock) {
        return reallyPoll();
    }
}

private Reference<? extends T> reallyPoll() {
    Reference<? extends T> r = head;
    if (r != null) {
        @SuppressWarnings("unchecked")
        Reference<? extends T> rn = r.next;
        // 因为尾节点的 next 指向自身
        head = (rn == r) ? null : rn;
        // 标记为 NULL，避免再次入队
        r.queue = NULL;
        // next 指向自己
        r.next = r;
        // 队列长度减一
        queueLength--;
        if (r instanceof FinalReference) {
            sun.misc.VM.addFinalRefCount(-1);
        }
        return r;
    }
    return null;
}
```

出队操作提供了等待的选项。

```java
// 从队列中移除下一个元素，阻塞直到有元素可用。
public Reference<? extends T> remove() throws InterruptedException {
    return remove(0);
}

// 从队列中移除下一个元素，阻塞直到超时或有元素可用，timeout 以毫秒为单位。
public Reference<? extends T> remove(long timeout)
    throws IllegalArgumentException, InterruptedException
{
    if (timeout < 0) {
        throw new IllegalArgumentException("Negative timeout value");
    }
    synchronized (lock) {
        Reference<? extends T> r = reallyPoll();
        if (r != null) return r;
        long start = (timeout == 0) ? 0 : System.nanoTime();
        for (;;) {
            lock.wait(timeout);
            r = reallyPoll();
            if (r != null) return r;
            // 如果 timeout 大于 0
            if (timeout != 0) {
                long end = System.nanoTime();
                // 计算下一轮等待时间
                timeout -= (end - start) / 1000_000;
                // 到时间直接返回 null
                if (timeout <= 0) return null;
                // 更新开始时间
                start = end;
            }
        }
    }
}
```

### 状态变化

`Reference` 实例（引用对象）可能处于四种内部状态之一：

- `Active`: 新创建的实例处于 `Active` 状态，受到垃圾收集器的特殊处理。收集器在检测到`关联对象`的可达性变为适当状态后的一段时间，会将实例的状态更改为 `Pending` 或 `Inactive`，具体取决于实例在创建时是否注册到`引用队列`中。在前一种情况下，它还会将实例添加到待 `pending-Reference` 列表中。
- `Pending`: 实例处在 `pending-Reference` 列表中，等待 `Reference-Handler` 线程将其加入`引用队列`。未注册到`引用队列`的实例永远不会处于这种状态。
- `Enqueued`: 处在创建实例时注册到的`引用队列`中。当实例从引用队列中删除时，该实例将变为 `Inactive` 状态。未注册到`引用队列`的实例永远不会处于这种状态。
- `Inactive`: 没有进一步的操作。一旦实例变为 `Inactive` 状态，其状态将永远不会再改变。

`Reference` 实例（引用对象）的状态由 `queue` 和 `next` 字段共同表达：

- `Active`: `(queue == ReferenceQueue || queue == ReferenceQueue.NULL) && next == null`
- `Pending`: `queue == ReferenceQueue && next == this`
- `Enqueued`: `queue == ReferenceQueue.ENQUEUED && (next == Following || this)`（在队列末尾时，`next` 指向自身，目前没有体现出这么设计的必要性啊？）
- `Inactive`: `queue == ReferenceQueue.NULL && next == this`

<div style="width:70%;margin:auto">{% asset_img "Pasted image 20231227235157.png" Reference 相关类图 %}</div>

## Reference 的子类

## 参考文章

- [你不可不知的Java引用类型之——Reference源码解析](https://cloud.tencent.com/developer/article/1366147)
- [Java引用类型之：Reference源码解析](https://www.jianshu.com/p/9fd68714c366)
- [JVM之Reference源码分析](https://juejin.cn/post/6942026483489734693)