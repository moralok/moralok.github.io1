---
title: 探索 Java 类 Cleaner 和 Finalizer
date: 2023-12-28 12:59:48
tags: [java]
---

`Java` 类 `Cleaner` 和 `Finalizer` 都实现了一种 `finalization` 机制，前者更轻量和强大，你可能在了解 `NIO` 的堆外内存自动释放机制中注意过它；后者为人所诟病，`finalize` 方法被人强烈反对使用。本文想要解析它们的原因不在于它们实现的功能，而在于它们是 `Reference` 的具体子类。
`Reference` 作为和 `GC` 紧密联系的类，你可能从很多文字描述中了解过 `SoftReference`、`WeakReference` 还有 `PhantomReference` 但是却很少从代码层面了解过它们，当你牢记“一个对象是否可以被回收的判断依据是它是否从 `Root` 对象可达”这条规则再面对 `Reference` 的子类时是否产生过割裂感；你是否好奇过 `Finalizer` 如何和重写 `finalize` 方法的类产生联系，本文将从 `Cleaner` 和 `Finalizer` 的源码揭示一些你可能已知的结论背后的朴素原理。

<!-- more -->

> 本文的写作动机继承自 {% post_link source-code-analysis-of-Java-class-Reference 'Java 类 Reference 的源码分析' %}，有时候也会自我怀疑研究一个涉及大家极力劝阻使用的 `finalize` 是否浪费精力，只能说确实如此！要不是半途而废会膈应难受肯定就停了！只能说这个过程确实帮助自己对 `Java` 引用和 `GC` 对其的处理有更加深刻的理解。

## 虚引用之 Cleaner

### 虚引用介绍

`PhantomReference` 对象在垃圾收集器确定其`关联对象`可以被回收时或可以被回收后一段时间，将被入队。“可以被回收”更明确的描述是“虚引用的`关联对象`变成 **`phantom reachable`** ，即只有虚引用引用了它”。但是和软引用和弱引用不同，**当虚引用入队时并不会被垃圾收集器自动清理（其关联对象）**。一个 `phantom reachable` 的对象会一直维持原样直到所有虚引用被清理或者它们自身变得不可达。

`PhantomReference` 的代码非常简单：

1. `PhantomReference` 仅提供了一个 `public` 构造函数，必须提供 `ReferenceQueue` 参数。它不像 `SoftReference` 和 `WeakReference` 可以离开 `ReferenceQueue` 单独使用，尽管 `queue` 可以为 `null`，但是这样做并没有意义。
2. `get()` 返回 `null`，这意味着不能通过 `PhantomReference` 获取其关联的对象 `referent`。

> `get()` 返回 `null` 并不是可以随意忽略的事情，它保证了 `phantom reachable` 对象不会被重新触达和修改（这是为清理工作留出时间吗）。

```java
public class PhantomReference<T> extends Reference<T> {
    public T get() {
        return null;
    }
    public PhantomReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```

通过以下示例验证 `GC` 不会自动清理虚引用的关联对象：

```java
public static void main(String[] args) throws InterruptedException {
    Scanner scanner = new Scanner(System.in);

    byte[] bytes = new byte[100 * 1024 * 1024];
    ReferenceQueue<byte[]> queue = new ReferenceQueue<>();
    PhantomReference<byte[]> phantomReference = new PhantomReference<>(bytes, queue);

    Thread thread = new Thread(() -> {
        for (; ; ) {
            try {
                Reference<? extends byte[]> remove = queue.remove(0);
                System.out.println(remove + " enqueued");
                // 需要调用 clear 主动清理关联对象，可以验证 gc 后总堆内存占用下降
                // remove.clear();
                // System.gc();
            } catch (InterruptedException e) {
                System.out.println(Thread.currentThread().getName() + " interrupt");
                break;
            }
        }
    });
    thread.start();

    System.out.println("暂停查看堆内存占用");
    scanner.next();

    bytes = null;
    System.gc();
    System.out.println("gc 后 sleep 3s，查看总堆内存占用未下降");
    TimeUnit.SECONDS.sleep(3);

    scanner.next();
    thread.interrupt();
}
```

### Cleaner 介绍

虚引用最常用于以比 `finalization` 更灵活的方式安排清理工作，比如其子类 `Cleaner` 就是一种基于虚引用的清理器，它比 `finalization` 更轻量但更强大。`Cleaner` 追踪其`关联对象`并封装任意的清理代码，在 `GC` 检测到其`关联对象`变成 `phantom reachable` 后一段时间，`Reference-Handler` 线程将运行清理代码。同时 `Cleaner` 可以被直接调用，它是线程安全的并且可以保证清理代码最多运行一次。但是 `Cleaner` 不是 `finalization` 的替代品，为了避免阻塞 `Reference-Handler` 线程，清理代码应极其简单和直接。

### 构造函数

`Cleaner` 的构造函数为 `private`，仅可通过 `create` 方法创建实例。

- `referent`: `关联对象`
- `dummyQueue`: 假队列，需要它仅仅是因为 `PhantomReference` 的构造函数需要一个 `queue` 参数，但是这个 `queue` 完全没用，在 `Reference` 中 `Reference-Handler` 线程会显式调用 `cleaners` 而不会执行入队操作。

```java
public class Cleaner extends PhantomReference<Object> {
    private static final ReferenceQueue<Object> dummyQueue = new ReferenceQueue<>();
    private final Runnable thunk;

    private Cleaner(Object referent, Runnable thunk) {
        super(referent, dummyQueue);
        this.thunk = thunk;
    }

    public static Cleaner create(Object ob, Runnable thunk) {
        if (thunk == null)
            return null;
        // 添加到 Cleaner 自身维护的双链表
        return add(new Cleaner(ob, thunk));
    }
}
```

### 添加 `cleaner`

- 使用 `synchronized` 同步
- `Cleaner` 自身维护一个双向链表存储 `cleaners`，通过静态变量 `first` 存储头节点，以防止 `cleaners` 比其`关联对象`更早被 `GC`。

<div style="width:70%;margin:auto">{% asset_img "Pasted image 20231228220806.png" Cleaner 双向链表 %}</div>

```java
// 头节点
static private Cleaner first = null;
// 双向指针
private Cleaner next = null, prev = null;
private static synchronized Cleaner add(Cleaner cl) {
    // 头插法
    if (first != null) {
        cl.next = first;
        first.prev = cl;
    }
    first = cl;
    return cl;
}
```

### clean 方法

在 `Reference` 中 `Reference-Handler` 线程对于 `Cleaner` 类型的对象，会显式地调用其 `clean` 方法并返回，而不会将其入队。

1. 使用 `synchronized` 同步，从双链表上移除自身
2. 调用 `thunk` 的 `run` 方法

```java
public void clean() {
    if (!remove(this))
        return;
    try {
        thunk.run();
    } catch (final Throwable x) {
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
                if (System.err != null)
                    new Error("Cleaner terminated abnormally", x).printStackTrace();
                System.exit(1);
                return null;
            }
        });
    }
}

private static synchronized boolean remove(Cleaner cl) {
    // next 指针指向自身代表已经移除，可以避免重复移除和执行
    if (cl.next == cl)
        return false;
    // 更新双链表
    if (first == cl) {
        if (cl.next != null)
            first = cl.next;
        else
            first = cl.prev;
    }
    if (cl.next != null)
        cl.next.prev = cl.prev;
    if (cl.prev != null)
        cl.prev.next = cl.next;
    // 通过将 next 指针指向自身表示已经被移除
    cl.next = cl;
    cl.prev = cl;
    return true;
}
```

### Cleaner 处理流程

- 创建的 `Cleaner` 对象被 `Cleaner` 类的双链表直接或间接引用（强引用），因此不会被垃圾回收
- 一切的起点仍然是 `GC` 特殊地对待虚引用的关联对象，当关联对象从 `reachable` 变成 `phantom reachable`，`GC` 将 `Cleaner` 对象将加入 `pending-list`
- `Reference-Handler` 线程又将其移除并调用 `clean` 方法
- 在调用完毕后，`Cleaner` 对象变成 `unreachable` 并最终被垃圾回收，其关联对象也被垃圾回收

> 注意，Cleaner 对象本身在被调用完毕之前始终是被静态变量引用，是 `reachable` 的，我们讨论的被判定为可回收的、变成 `phantom reachable` 状态的是关联对象。

<div style="width:70%;margin:auto">{% asset_img "Pasted image 20231229012917.png" Cleaner 处理流程 %}</div>

> 事实上，个人猜测“虚引用的关联对象不像软引用和弱引用会被自动清理”描述的仅仅是一个表象，判断是否要被垃圾回收的根本法则仍然是“对象是否从 `Root` 对象可达”，软引用和弱引用的`关联对象`之所以会被垃圾回收是因为它们在加入 `pending-list` 时被从`引用对象`断开，否则当`引用对象`被添加到`引用队列`时，`引用队列`如果从 `Root` 对象可达，将导致`关联对象`也从 `Root` 对象可达。在 `Reference` 的 `clear()` 的注释中提及该方法只被 `Java` 代码调用，`GC` 不需要调用该方法就可以直接清理，肯定是 `GC` 有直接清理`关联对象`的场景。同时 `Reference` 类有一句注释“`GC` 在检测到`关联对象`有特定的可达性变化后，将把`引用对象`添加到`引用队列`”，它并未将特定的可达性变化直接描述为`关联对象`变为不可达。目前尚未从 `JVM` 源代码验证该猜测。

## 终结引用之 Finalizer

`FinalReference` 用于实现 `finalization`，其代码很简单。

```java
class FinalReference<T> extends Reference<T> {
    public FinalReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```

其子类 `Finalizer` 继承自 `FinalReference`，`Cleaner` 在代码设计上和它非常相似。

### 构造函数

`Finalizer` 的构造函数为 `private`，仅可通过 `register` 方法创建实例。

- `finalizee`: `关联对象`，即重写了 `finalize` 方法的类的实例
- `queue`: 引用队列

> 根据注释 `register` 由 `VM` 调用，我们可以合理猜测，这里就是重写了 `finalize` 方法的类的实例和 `Finalizer` 对象关联的起点。

```java
final class Finalizer extends FinalReference<Object> {
    private static ReferenceQueue<Object> queue = new ReferenceQueue<>();

    private Finalizer(Object finalizee) {
        super(finalizee, queue);
        add();
    }
    // 由 VM 调用
    static void register(Object finalizee) {
        new Finalizer(finalizee);
    }
}
```

### 添加 `Finalizer`

- 使用 `synchronized` 同步
- `Finalizer` 自身维护一个双向链表存储 `finalizers`，通过静态变量 `unfinalized` 存储头节点

<div style="width:70%;margin:auto">{% asset_img "Pasted image 20231228230444.png" Finalizer 双向链表 %}</div>

```java
private static Finalizer unfinalized = null;
private static final Object lock = new Object();
private Finalizer next = null, prev = null;

private void add() {
    synchronized (lock) {
        if (unfinalized != null) {
            this.next = unfinalized;
            unfinalized.prev = this;
        }
        unfinalized = this;
    }
}
```

### `Finalizer` 线程

`finalizers` 的清理通常是由一条名为 `Finalizer` 的线程处理。启动任意一个非常简单的 `Java` 程序，通过 `JVM` 相关的工具，比如 `JConsole`，你都能看到一个名为 `Finalizer` 的线程。

<div style="width:70%;margin:auto">{% asset_img "Snipaste_2023-12-28_23-09-21.png" Finalizer 线程 %}</div>

#### run 方法

```java
private static class FinalizerThread extends Thread {
    private volatile boolean running;
    FinalizerThread(ThreadGroup g) {
        super(g, "Finalizer");
    }
    public void run() {
        // 防止递归调用 run（什么场景？）
        if (running)
            return;
        // Finalizer thread 在 System.initializeSystemClass 被调用前启动，等待 JavaLangAccess 可用
        while (!VM.isBooted()) {
            // 推迟直到 VM 初始化完成
            try {
                VM.awaitBooted();
            } catch (InterruptedException x) {
                // 忽略并继续
            }
        }
        final JavaLangAccess jla = SharedSecrets.getJavaLangAccess();
        // 标记为运行中
        running = true;
        for (;;) {
            try {
                // 从队列中移除
                Finalizer f = (Finalizer)queue.remove();
                // 调用 runFinalizer
                f.runFinalizer(jla);
            } catch (InterruptedException x) {
                // 忽略并继续
            }
        }
    }
}
```

#### 创建和启动

`Finalizer` 线程是通过静态代码块创建和启动的。

```java
static {
    // 向上获取父线程组，直到系统线程组
    ThreadGroup tg = Thread.currentThread().getThreadGroup();
    for (ThreadGroup tgn = tg;
            tgn != null;
            tg = tgn, tgn = tg.getParent());
    // 创建 FinalizerThread 并启动
    Thread finalizer = new FinalizerThread(tg);
    // 设置优先级为最高减 2
    finalizer.setPriority(Thread.MAX_PRIORITY - 2);
    finalizer.setDaemon(true);
    finalizer.start();
}
```

#### 获取 Finalizer 并调用

```java
private void runFinalizer(JavaLangAccess jla) {
    synchronized (this) {
        // 判断是否已经终结过
        if (hasBeenFinalized()) return;
        // 从双链表上移除
        remove();
    }
    try {
        // 获取关联的 finalizee
        Object finalizee = this.get();
        // 如果不为 null 且不是 Enum 类型
        if (finalizee != null && !(finalizee instanceof java.lang.Enum)) {
            // 调用 invokeFinalize
            jla.invokeFinalize(finalizee);
            // 清理栈槽以降低保守 GC 时误保留的可能性
            finalizee = null;
        }
    } catch (Throwable x) { }
    // 清理关联对象
    super.clear();
}

// 和 Cleaner 类似，使用 next 指向自身表示已被移除
private boolean hasBeenFinalized() {
    return (next == this);
}

// 和 Cleaner 类似的处理
private void remove() {
    synchronized (lock) {
        if (unfinalized == this) {
            if (this.next != null) {
                unfinalized = this.next;
            } else {
                unfinalized = this.prev;
            }
        }
        if (this.next != null) {
            this.next.prev = this.prev;
        }
        if (this.prev != null) {
            this.prev.next = this.next;
        }
        this.next = this;
        this.prev = this;
    }
}
```

### finalize 的调用原理

关于如何调用 `finalize` 方法涉及不少平时接触不到的代码。

```java
// 获取 JavaLangAccess
final JavaLangAccess jla = SharedSecrets.getJavaLangAccess();
// 通过 JavaLangAccess 调用 finalizee 的 finalize 方法
jla.invokeFinalize(finalizee);
public static void setJavaLangAccess(JavaLangAccess jla) {
    javaLangAccess = jla;
}
```

`SharedSecrets` 的 `javaLangAccess` 通过 `setJavaLangAccess` 设置

```java
public static void setJavaLangAccess(JavaLangAccess jla) {
    javaLangAccess = jla;
}
public static JavaLangAccess getJavaLangAccess() {
    return javaLangAccess;
}
```

`setJavaLangAccess` 方法在 `System` 中被调用，`javaLangAccess` 被设置为一个匿名类实例，其中 `invokeFinalize` 方法间接调用了传入对象的 `finalize` 方法。

```java
private static void setJavaLangAccess() {
    // Allow privileged classes outside of java.lang
    sun.misc.SharedSecrets.setJavaLangAccess(new sun.misc.JavaLangAccess(){
        // ...
        public void invokeFinalize(Object o) throws Throwable {
            o.finalize();
        }
    });
}
```

`System` 的 `setJavaLangAccess` 方法在 `initializeSystemClass` 方法中被调用。这里正对应着 `FinalizerThread` 的 `run` 方法中等待 `VM` 初始化完成的处理。

```java
// 初始化 System class，在线程初始化之后调用
private static void initializeSystemClass() {
    // ...
    // register shared secrets
    setJavaLangAccess();
    // 通知 wait 的线程
    sun.misc.VM.booted();
}
```

### Finalizer 的注册时机

你是否好奇过 `JVM` 是如何保证 `finalize` 方法最多被调用一次的？如果曾经猜测过 `JVM` 可能在对象中留有标记，那么在我们研究过对象的内部结构之后可以确认其中并没有用于记录对象是否已经 `finalized` 的地方。同时我们注意到 `hasBeenFinalized` 方法通过 `next` 指针是否指向自己表示是否已经 `finalized`。我们可以合理猜测 `register` 的调用时机是在对象创建时，因此最多仅有一次被注册。

通过以下示例可以测试：

- 在创建重写了 `finalize` 方法的类创建对象期间会调用 `register` 创建并注册 `Finalizer`
- 在未重写 `finalize` 方法的类创建对象期间不会调用`register`
- `Finalizer` 不仅可以保证 `finalize` 只会被调用一次，甚至不会第二次被添加到 `pending-list`，因为 `runFinalizer` 最后调用了 `super.clear()`，`JVM` 不会特殊对待复活的对象

```java
public class FinalReferenceTest_1 {

    private static FinalizeObj save = null;

    public static void main(String[] args) throws InterruptedException {
        System.out.println("创建 finalize obj，使用 Debug 强制运行到 Finalizer.register");
        FinalizeObj finalizeObj = new FinalizeObj();

        System.out.println("gc");
        finalizeObj = null;
        System.gc();
        System.out.println("sleep 1s");
        TimeUnit.SECONDS.sleep(1);
        save.echo();

        save = null;
        System.gc();
        System.out.println("sleep 1s");
        TimeUnit.SECONDS.sleep(1);
        System.out.println(save == null);
    }

    static class FinalizeObj {
        FinalizeObj() {
            System.out.println("SaveSelf created");
        }
        @Override
        protected void finalize() throws Throwable {
            System.out.println("finalized");
            save = this;
        }
        public void echo() {
            System.out.println("I am alive.");
        }
    }
}
```




## 参考文章

- [How is an object marked as finalized in Java (so that the finalize method wouldn't be called the second time)?](https://stackoverflow.com/questions/21809922/how-is-an-object-marked-as-finalized-in-java-so-that-the-finalize-method-wouldn)