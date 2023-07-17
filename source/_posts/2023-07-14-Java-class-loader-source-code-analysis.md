---
title: Java 类加载器源码分析
date: 2023-07-14 04:08:11
tags:
    - Java
    - ClassLoader
---

### 组织类加载工作：loadClass
当 `Java` 程序启动的时候，`Java` 虚拟机会调用 `java.lang.ClassLoader#loadClass(java.lang.String)` 加载 `main` 方法所在的类。

```java
public Class<?> loadClass(String name) throws ClassNotFoundException {
    return loadClass(name, false);
}
```

根据注释可知，此方法加载具有指定二进制名称的类，它由 `Java` 虚拟机调用来解析类引用，调用它等同于调用 `loadClass(name, false)`。

```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    // 以二进制名称获取类加载的锁进行同步
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        // 首先检查类是否已加载，根据该方法注释可知：
        // 如果当前类加载器已经被 Java 虚拟机记录为具有该二进制名称的类的加载器（initiating loader），Java 虚拟机可以直接返回 Class 对象。
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                // 如果类还未加载，先委派给父·类加载器进行加载，如果父·类加载器为 null，则使用虚拟机内建的类加载器进行加载
                if (parent != null) {
                    // 递归调用
                    c = parent.loadClass(name, false);
                } else {
                    // 递归调用的终结点
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
                // 当父·类加载器长尝试加载但是失败，捕获异常但是什么都不做，因为接下来，当前类加载器需要自己也尝试加载。
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                // 父·类加载器未找到类，当前类加载器自己找。
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

根据注释可知，`java.lang.ClassLoader#loadClass(java.lang.String, boolean)` 同样是加载“具有指定二进制名称的类”，此方法的实现按以下顺序搜索类：
1. 调用 `findLoadedClass(String)` 以检查该类是否已加载。
2. 在父·类加载器上调用 `loadClass` 方法。如果父·类加载器为空，则使用虚拟机内置的类加载器。
3. 调用 `findClass(String)` 方法来查找该类。

如果使用上述步骤找到了该类（找到并定义类），并且解析标志为 `true`，则此方法将对生成的 `Class` 对象调用 `resolveClass(Class)` 方法。鼓励 `ClassLoader` 的子类重写 `findClass(String)`，而不是此方法。除非被重写，否则此方法在整个类加载过程中以 `getClassLoadingLock` 方法的结果进行同步。

> 注意：**父·类加载器**并非**父类·类加载器**（当前类加载器的父类），而是当前的类加载器的 `parent` 属性被赋值另外一个类加载器实例，其含义更接近于“可以委派类加载工作的另一个类加载器（一个帮忙干活的上级）”。虽然绝大多数说法中，当一个类加载器的 `parent` 值为 `null` 时，它的父·类加载器是引导类加载器（`bootstrap class loader`），但是当看到 `findBootstrapClassOrNull` 方法时，我有点困惑，因为我以为会看到语义类似于 `loadClassByBootstrapClassLoader` 这样的方法名。从注释和代码的语义上看，`bootstrap class loader` 不像是任何一个类加载器的父·类加载器，但是从类加载的机制设计上说，它是，只是因为它并非由 Java 语言编写而成，不能实例化并赋值给 `parent` 属性。`findBootstrapClassOrNull` 方法的语义更接近于：当一个类加载器的父·类加载器为 `null` 时，将准备加载的目标类先当作启动类（`Bootstrap Class`）尝试查找，如果找不到就返回 `null`。

#### 怎么并行地加载类 getClassLoadingLock
需要加载的类可能很多很多，我们很容易想到如果可以并行地加载类就好了。显然，`JDK` 的编写者考虑到了这一点。
此方法返回类加载操作的锁对象。为了向后兼容，此方法的默认实现的行为如下。如果此 `ClassLoader` 对象注册为具备并行能力，则该方法返回与指定类名关联的专用对象。 否则，该方法返回此 `ClassLoader` 对象。
简单地说，如果 `ClassLoader` 对象注册为具备并行能力，那么一个 `name` 一个锁对象，已创建的锁对象保存在 `ConcurrentHashMap` 类型的 `parallelLockMap` 中，这样类加载工作可以并行；否则所有类加载工作共用一个锁对象，就是 `ClassLoader` 对象本身。
这个方案意味着非同名的目标类可以认为在加载时没有冲突？

```java
protected Object getClassLoadingLock(String className) {
    Object lock = this;
    if (parallelLockMap != null) {
        Object newLock = new Object();
        lock = parallelLockMap.putIfAbsent(className, newLock);
        if (lock == null) {
            lock = newLock;
        }
    }
    return lock;
}
```

##### 什么是 "`ClassLoader` 对象注册为具有并行能力"呢？
`AppClassLoader` 中有一段 `static` 代码。事实上 `java.lang.ClassLoader#registerAsParallelCapable` 是将 `ClassLoader` 对象注册为具有并行能力唯一的入口。因此，所有想要注册为具有并行能力的 `ClassLoader` 都需要调用一次该方法。
```java
static {
    ClassLoader.registerAsParallelCapable();
}
```

`java.lang.ClassLoader#registerAsParallelCapable` 方法有一个注解 `@CallerSensitive`，这是因为它的代码中调用的 `native` 方法 `sun.reflect.Reflection#getCallerClass()` 方法。由注释可知，当且仅当以下所有条件全部满足时才注册成功：
1. 尚未创建调用者的实例（类加载器尚未实例化）
2. 调用者的所有超类（`Object` 类除外）都注册为具有并行能力。

##### 怎么保证这两个条件成立呢？
1. 对于第一个条件，可以通过将调用的代码写在 `static` 代码块中来实现。如果写在构造器方法里，并且通过单例模式保证只实例化一次可以吗？答案是不行的，后续会解释这个“注册”行为在构造器方法中是如何被使用以及为何不能写在构造器方法里。
2. 对于第二个条件，由于 `Java` 虚拟机加载类时，总是会先尝试加载其父类，又因为加载类时会先调用 `static` 代码块，因此父类的 `static` 代码块总是先于子类的 `static` 代码块。

你可以看到 `AppClassLoader->URLClassLoader->SecureClassLoader->ClassLoader` 均在 `static` 代码块实现注册，以保证满足以上两个条件。

##### 注册工作做了什么？
简单地说就是保存了类加载器所属 `Class` 的 `Set`。

```java
@CallerSensitive
protected static boolean registerAsParallelCapable() {
    // 获得此方法的调用者的 Class 实例，asSubClass 可以将 Class<?> 类型的 Class 转换为代表指定类的子类的 Class<? extends U> 类型的 Class。
    Class<? extends ClassLoader> callerClass =
        Reflection.getCallerClass().asSubclass(ClassLoader.class);
    // 注册调用者的 Class 为具有并行能力
    return ParallelLoaders.register(callerClass);
}
```

方法 `java.lang.ClassLoader.ParallelLoaders#register`。`ParallelLoaders` 封装了一组具有并行能力的加载器类型。就是持有 `ClassLoader` 的 `Class` 实例的集合，并保证添加时加同步锁。

```java
// private 修饰，只有其外部类 ClassLoader 才可以使用
// static 修饰，内部类如果需要定义 static 方法或者 static 变量，必须用 static 修饰
private static class ParallelLoaders {
    // private 修饰构造器方法，不希望这个类被实例化，只想要使用它的静态变量和方法。
    private ParallelLoaders() {}

    // the set of parallel capable loader types
    // 使用 loaderTypes 时通过 synchronized 加同步锁
    private static final Set<Class<? extends ClassLoader>> loaderTypes =
        Collections.newSetFromMap(
            // todo: 为什么使用弱引用来实现？为了卸载类时的垃圾回收？
            new WeakHashMap<Class<? extends ClassLoader>, Boolean>());
    static {
        // 将 ClassLoader 本身注册为具有并行能力
        synchronized (loaderTypes) { loaderTypes.add(ClassLoader.class); }
    }

    /**
     * Registers the given class loader type as parallel capabale.
     * Returns {@code true} is successfully registered; {@code false} if
     * loader's super class is not registered.
     */
    static boolean register(Class<? extends ClassLoader> c) {
        synchronized (loaderTypes) {
            if (loaderTypes.contains(c.getSuperclass())) {
                // register the class loader as parallel capable
                // if and only if all of its super classes are.
                // Note: given current classloading sequence, if
                // the immediate super class is parallel capable,
                // all the super classes higher up must be too.
                // 当且仅当其所有超类都具有并行能力时，才将类加载器注册为具有并行能力。
                // 注意：给定当前的类加载顺序（加载类时，Java 虚拟机总是先尝试加载其父类），如果直接超类具有并行能力，则所有更高的超类也必然具有并行能力。
                loaderTypes.add(c);
                return true;
            } else {
                return false;
            }
        }
    }

    /**
     * Returns {@code true} if the given class loader type is
     * registered as parallel capable.
     */
    static boolean isRegistered(Class<? extends ClassLoader> c) {
        synchronized (loaderTypes) {
            return loaderTypes.contains(c);
        }
    }
}
```

##### “注册”怎么和锁产生联系？
但是以上的注册过程只是起到一个“标记”作用，没有涉及和锁相关的代码，那么这个“标记”是怎么和真正的锁产生联系呢？`ClassLoader` 提供了三个构造器方法：
```java
private ClassLoader(Void unused, ClassLoader parent) {
    // 由 private 修饰，不允许子类重写
    this.parent = parent;
    if (ParallelLoaders.isRegistered(this.getClass())) {
        // 如果类加载器已经注册为具有并行能力，则做一些赋值操作
        parallelLockMap = new ConcurrentHashMap<>();
        // 保存 package->certs 的 map 映射，相关的工作也可以并行进行
        package2certs = new ConcurrentHashMap<>();
        assertionLock = new Object();
    } else {
        // no finer-grained lock; lock on the classloader instance
        parallelLockMap = null;
        package2certs = new Hashtable<>();
        assertionLock = this;
    }
}

// 由 protect 修饰，允许子类重写，传递了父·类加载器。
protected ClassLoader(ClassLoader parent) {
    this(checkCreateClassLoader(), parent);
}

// 由 protect 修饰，允许子类重写，父·类加载器使用 getSystemClassLoader 方法返回的系统类加载器。
protected ClassLoader() {
    this(checkCreateClassLoader(), getSystemClassLoader());
}
```

`ClassLoader` 的构造器方法最终都调用 `private` 修饰的 `java.lang.ClassLoader#ClassLoader(java.lang.Void, java.lang.ClassLoader)`，又因为父类的构造器方法总是先于子类的构造器方法被执行，这样一来，所有继承 `ClassLoader` 的类加载器在创建的时候都会根据在创建实例之前是否注册为具有并行能力而做不同的操作。

##### 为什么注册的代码不能写在构造器方法里？
使用“注册”的代码也解释了 `java.lang.ClassLoader#registerAsParallelCapable` 为了满足调用成功的第一个条件为什么不能写在构造器方法中，因为使用这个机制的代码先于你在子类构造器方法里编写的代码被执行。
同时，不论是 `loadLoader` 还是 `getClassLoadingLock` 都是由 `protect` 修饰，允许子类重写，来自定义并行加载类的能力。

> todo: 讨论自定义类加载器的时候，印象里似乎对并行加载类的提及比较少，之后留意一下。

#### 检查目标类是否已加载
加载类之前显然需要检查目标类是否已加载，这项工作最终是交给 `native` 方法，在虚拟机中执行，就像在黑盒中一样。
todo: 不同类加载器同一个类名会如何判定？

```java
protected final Class<?> findLoadedClass(String name) {
    if (!checkName(name))
        return null;
    return findLoadedClass0(name);
}

private native final Class<?> findLoadedClass0(String name);
```

#### 保证核心类库的安全性：双亲委派模型
正如在代码和注释中所看到的，正常情况下，类的加载工作先委派给自己的父·类加载器，即 `parent` 属性的值——另一个类加载器实例。一层一层向上委派直到 `parent` 为 `null`，代表类加载工作会尝试先委派给虚拟机内建的 `bootstrap class loader` 处理，然后由 `bootstrap class loader` 首先尝试加载。如果被委派方加载失败，委派方会自己再尝试加载。
正常加载类的是应用类加载器 `AppClassLoader`，它的 `parent` 为 `ExtClassLoader`，`ExtClassLoader` 的 `parent` 为 `null`。

> 在网上也能看到有人提到以前大家称之为“父·类加载器委派机制”，“双亲”一词易引人误解。

##### 为什么要用这套奇怪的机制
这样设计很明显的一个目的就是保证核心类库的类加载安全性。比如 `Object` 类，设计者不希望编写代码的人重新写一个 `Object` 类并加载到 `Java` 虚拟机中，但是加载类的本质就是读取字节数据传递给 `Java` 虚拟机创建一个 `Class` 实例，使用这套机制的目的之一就是为了让核心类库先加载，同时先加载的类不会再次被加载。

通常流程如下：
1. `AppClassLoader` 调用 `loadClass` 方法，先委派给 `ExtClassLoader`。
2. `ExtClassLoader` 调用 `loadClass` 方法，先委派给 `bootstrap class loader`。
3. `bootstrap class loader` 在其设置的类路径中无法找到 `BananaTest` 类，抛出 `ClassNotFoundException` 异常。
4. `ExtClassLoader` 捕获异常，然后自己调用 `findClass` 方法尝试进行加载。
5. `ExtClassLoader` 在其设置的类路径中无法找到 `BananaTest` 类，抛出 `ClassNotFoundException` 异常。
6. `AppClassLoader` 捕获异常，然后自己调用 `findClass` 方法尝试进行加载。

注释中提到鼓励重写 `findClass` 方法而不是 `loadClass`，因为正是该方法实现了所谓的“双亲委派模型”，`java.lang.ClassLoader#findClass` 实现了如何查找加载类。如果不是专门为了破坏这个类加载模型，应该选择重写 `findClass`；其次是因为该方法中涉及并行加载类的机制。

### 查找类资源：findClass
默认情况下，类加载器在自己尝试进行加载时，会调用 `java.lang.ClassLoader#findClass` 方法，该方法由子类重写。`AppClassLoader` 和 `ExtClassLoader` 都是继承 `URLClassLoader`，而 `URLClassLoader` 重写了 `findClass` 方法。根据注释可知，该方法会从 `URL` 搜索路径查找并加载具有指定名称的类。任何引用 `Jar` 文件的 `URL` 都会根据需要加载并打开，直到找到该类。

过程如下：
1. 将 `name` 转换为 `path`，比如 `com.example.BananaTest` 转换为 `com/example/BananaTest.class`。
2. 使用 `URL` 搜索路径 `URLClassPath` 和 `path` 中获取 `Resource`，本质上就是轮流将可能存放的目录列表拼接上文件路径进行查找。
3. 调用 `URLClassLoader` 的私有方法 `defineClass`，该方法调用父类 `SecureClassLoader` 的 `defineClass` 方法。

```java
protected Class<?> findClass(final String name)
    throws ClassNotFoundException
{
    final Class<?> result;
    try {
        // todo:
        result = AccessController.doPrivileged(
            new PrivilegedExceptionAction<Class<?>>() {
                public Class<?> run() throws ClassNotFoundException {
                    // 将 name 转换为 path
                    String path = name.replace('.', '/').concat(".class");
                    // 从 URLClassPath 中查找 Resource
                    Resource res = ucp.getResource(path, false);
                    if (res != null) {
                        try {
                            return defineClass(name, res);
                        } catch (IOException e) {
                            throw new ClassNotFoundException(name, e);
                        } catch (ClassFormatError e2) {
                            if (res.getDataError() != null) {
                                e2.addSuppressed(res.getDataError());
                            }
                            throw e2;
                        }
                    } else {
                        return null;
                    }
                }
            }, acc);
    } catch (java.security.PrivilegedActionException pae) {
        throw (ClassNotFoundException) pae.getException();
    }
    if (result == null) {
        throw new ClassNotFoundException(name);
    }
    return result;
}
```

#### 查找类的目录列表：URLClassPath
`URLClassLoader` 拥有一个 `URLClassPath` 类型的属性 `ucp`。由注释可知，`URLClassPath` 类用于维护一个 `URL` 的搜索路径，以便从 `Jar` 文件和目录中加载类和资源。
`URLClassPath` 的核心构造器方法：

```java
public URLClassPath(URL[] urls,
                    URLStreamHandlerFactory factory,
                    AccessControlContext acc) {
    // 将 urls 保存到 ArrayList 类型的属性 path 中，根据注释，path 的含义为 URL 的原始搜索路径。
    for (int i = 0; i < urls.length; i++) {
        path.add(urls[i]);
    }
    // 将 urls 保存到 Stack 类型的属性 urls 中，根据注释，urls 的含义为未打开的 URL 列表。
    push(urls);
    if (factory != null) {
        // 如果 factory 不为 null，使用它创建一个 URLStreamHandler 实例处理 Jar 文件。
        jarHandler = factory.createURLStreamHandler("jar");
    }
    if (DISABLE_ACC_CHECKING)
        this.acc = null;
    else
        this.acc = acc;
}
```

##### URLClassPath#getResource
`URLClassLoader` 调用 `sun.misc.URLClassPath#getResource(java.lang.String, boolean)` 方法获取指定名称对应的资源。根据注释，该方法会查找 `URL` 搜索路径上的第一个资源，如果找不到资源，则返回 `null`。
显然，这里的 `Loader` 不是我们前面提到的类加载器。`Loader` 是 `URLClassPath` 的内部类，用于表示根据一个基本 `URL` 创建的资源和类的加载器。也就是说一个基本 `URL` 对应一个 `Loader`。

```java
public Resource getResource(String name, boolean check) {
    if (DEBUG) {
        System.err.println("URLClassPath.getResource(\"" + name + "\")");
    }

    Loader loader;
    // 获取缓存（默认没有用）
    int[] cache = getLookupCache(name);
    // 不断获取下一个 Loader 来获取 Resource，直到获取到或者没有下一个 Loader
    for (int i = 0; (loader = getNextLoader(cache, i)) != null; i++) {
        Resource res = loader.getResource(name, check);
        if (res != null) {
            return res;
        }
    }
    return null;
}
```

##### URLClassPath#getNextLoader
获取下一个 `Loader`，其实根据 `index` 从一个存放已创建 `Loader` 的 `ArrayList` 中获取。

```java
private synchronized Loader getNextLoader(int[] cache, int index) {
    if (closed) {
        return null;
    }
    if (cache != null) {
        if (index < cache.length) {
            Loader loader = loaders.get(cache[index]);
            if (DEBUG_LOOKUP_CACHE) {
                System.out.println("HASCACHE: Loading from : " + cache[index]
                                    + " = " + loader.getBaseURL());
            }
            return loader;
        } else {
            return null; // finished iterating over cache[]
        }
    } else {
        // 获取 Loader
        return getLoader(index);
    }
}
```

##### URLClassPath#getLoader(int)
1. 用 `index` 到存放已创建 `Loader` 的列表中去获取（调用方传入的 `index` 从 `0` 开始不断递增直到超过范围）。
2. 如果 `index` 超过范围，说明已有的 `Loader` 都找不到目标 `Resource`，需要到未打开的 `URL` 中查找。
3. 从未打开的 `URL` 中取出（`pop`）一个来创建 `Loader`，如果 `urls` 已经为空，则返回 `null`。

```java
private synchronized Loader getLoader(int index) {
    if (closed) {
        return null;
    }
    // Expand URL search path until the request can be satisfied
    // or the URL stack is empty.
    while (loaders.size() < index + 1) {
        // Pop the next URL from the URL stack
        // 如果 index 超过数组范围，需要从未打开的 URL 中取出一个，创建 Loader 并返回
        URL url;
        synchronized (urls) {
            if (urls.empty()) {
                return null;
            } else {
                url = urls.pop();
            }
        }
        // Skip this URL if it already has a Loader. (Loader
        // may be null in the case where URL has not been opened
        // but is referenced by a JAR index.)
        String urlNoFragString = URLUtil.urlNoFragString(url);
        if (lmap.containsKey(urlNoFragString)) {
            continue;
        }
        // Otherwise, create a new Loader for the URL.
        Loader loader;
        try {
            // 根据 URL 创建 Loader
            loader = getLoader(url);
            // If the loader defines a local class path then add the
            // URLs to the list of URLs to be opened.
            URL[] urls = loader.getClassPath();
            if (urls != null) {
                push(urls);
            }
        } catch (IOException e) {
            // Silently ignore for now...
            continue;
        } catch (SecurityException se) {
            // Always silently ignore. The context, if there is one, that
            // this URLClassPath was given during construction will never
            // have permission to access the URL.
            if (DEBUG) {
                System.err.println("Failed to access " + url + ", " + se );
            }
            continue;
        }
        // Finally, add the Loader to the search path.
        validateLookupCache(loaders.size(), urlNoFragString);
        loaders.add(loader);
        lmap.put(urlNoFragString, loader);
    }
    if (DEBUG_LOOKUP_CACHE) {
        System.out.println("NOCACHE: Loading from : " + index );
    }
    return loaders.get(index);
}
```

##### URLClassPath#getLoader(java.net.URL)
根据指定的 `URL` 创建 `Loader`，不同类型的 `URL` 会返回不同具体实现的 `Loader`。
1. 如果 `URL` 不是以 `/` 结尾，认为是 `Jar` 文件，则返回 `JarLoader` 类型，比如 `file:/C:/Users/xxx/.jdks/corretto-1.8.0_342/jre/lib/rt.jar`。
2. 如果 `URL` 以 `/` 结尾，且协议为 `file`，则返回 `FileLoader` 类型，比如 `file:/C:/Users/xxx/IdeaProjects/java-test/target/classes/`。
3. 如果 `URL` 以 `/` 结尾，且协议不会 `file`，则返回 `Loader` 类型。

```java
private Loader getLoader(final URL url) throws IOException {
    try {
        return java.security.AccessController.doPrivileged(
            new java.security.PrivilegedExceptionAction<Loader>() {
            public Loader run() throws IOException {
                String file = url.getFile();
                if (file != null && file.endsWith("/")) {
                    if ("file".equals(url.getProtocol())) {
                        return new FileLoader(url);
                    } else {
                        return new Loader(url);
                    }
                } else {
                    return new JarLoader(url, jarHandler, lmap, acc);
                }
            }
        }, acc);
    } catch (java.security.PrivilegedActionException pae) {
        throw (IOException)pae.getException();
    }
}
```

#### URLClassPath.FileLoader#getResource
以 `FileLoader` 的 `getResource` 为例，如果文件找到了，就会将文件包装成一个 `FileInputStream`，再将 `FileInputStream` 包装成一个 `Resource` 返回。
```java
Resource getResource(final String name, boolean check) {
    final URL url;
    try {
        URL normalizedBase = new URL(getBaseURL(), ".");
        url = new URL(getBaseURL(), ParseUtil.encodePath(name, false));

        if (url.getFile().startsWith(normalizedBase.getFile()) == false) {
            // requested resource had ../..'s in path
            return null;
        }

        if (check)
            URLClassPath.check(url);

        final File file;
        if (name.indexOf("..") != -1) {
            file = (new File(dir, name.replace('/', File.separatorChar)))
                    .getCanonicalFile();
            if ( !((file.getPath()).startsWith(dir.getPath())) ) {
                /* outside of base dir */
                return null;
            }
        } else {
            file = new File(dir, name.replace('/', File.separatorChar));
        }

        if (file.exists()) {
            return new Resource() {
                public String getName() { return name; };
                public URL getURL() { return url; };
                public URL getCodeSourceURL() { return getBaseURL(); };
                public InputStream getInputStream() throws IOException
                    { return new FileInputStream(file); };
                public int getContentLength() throws IOException
                    { return (int)file.length(); };
            };
        }
    } catch (Exception e) {
        return null;
    }
    return null;
}
```

#### ClassLoader 的搜索路径
从上文可知，`ClassLoader` 调用 `findClass` 方法查找类的时候，并不是漫无目的地查找，而是根据设置的类路径进行查找，不同的 `ClassLoader` 有不同的类路径。

以下是通过 `IDEA` 启动 `Java` 程序时的命令，可以看到其中通过 `-classpath` 指定了应用·类加载器 `AppClassLoader` 的类路径，该类路径除了包含常规的 `JRE` 的文件路径外，还额外添加了当前 `maven` 工程编译生成的 `target\classes` 目录。

```shell
C:\Users\xxx\.jdks\corretto-1.8.0_342\bin\java.exe -agentlib:jdwp=transport=dt_socket,address=127.0.0.1:52959,suspend=y,server=n -javaagent:C:\Users\xxx\AppData\Local\JetBrains\IntelliJIdea2022.3\captureAgent\debugger-agent.jar -Dfile.encoding=UTF-8 -classpath "C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\charsets.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\ext\access-bridge-64.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\ext\cldrdata.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\ext\dnsns.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\ext\jaccess.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\ext\jfxrt.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\ext\localedata.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\ext\nashorn.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\ext\sunec.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\ext\sunjce_provider.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\ext\sunmscapi.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\ext\sunpkcs11.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\ext\zipfs.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\jce.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\jfr.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\jfxswt.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\jsse.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\management-agent.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\resources.jar;C:\Users\xxx\.jdks\corretto-1.8.0_342\jre\lib\rt.jar;C:\Users\xxx\IdeaProjects\java-test\target\classes;C:\Program Files\JetBrains\IntelliJ IDEA 2022.3.3\lib\idea_rt.jar" org.example.BananaTest
```

##### bootstrap class loader
启动·类加载器 `bootstrap class loader`，加载核心类库，即 `<JRE_HOME>/lib` 目录中的部分类库，如 `rt.jar`，只有名字符合要求的 `jar` 才能被识别。 启动 Java 虚拟机时可以通过选项 `-Xbootclasspath` 修改默认的类路径，有三种使用方式：
- `-Xbootclasspath:`：完全覆盖核心类库的类路径，不常用，除非重写核心类库。
- `-Xbootclasspath/a:` 以后缀的方式拼接在原搜索路径后面，常用。
- `-Xbootclasspath/p:` 以前缀的方式拼接再原搜索路径前面.不常用，避免引起不必要的冲突。

在 `IDEA` 中编辑启动配置，添加 `VM` 选项，`-Xbootclasspath:C:\Software`，里面没有类文件，启动虚拟机失败，提示：
```shell
Error occurred during initialization of VM
java/lang/NoClassDefFoundError: java/lang/Object

进程已结束,退出代码1
```

##### ExtClassLoader
扩展·类加载器 `ExtClassLoader`，加载 `<JRE_HOME>/lib/ext/` 目录中的类库。启动 `Java` 虚拟机时可以通过选项 `-Djava.ext.dirs` 修改默认的类路径。显然修改不当同样可能会引起 `Java` 程序的异常。

##### AppClassLoader
应用·类加载器 `AppClassLoader` ，加载应用级别的搜索路径中的类库。可以使用系统的环境变量 `CLASSPATH` 的值，也可以在启动 Java 虚拟机时通过选项 `-classpath` 修改。

`CLASSPATH` 在 `Windows` 中，多个文件路径使用分号 `;` 分隔，而 `Linux` 中则使用冒号 `:` 分隔。以下例子表示当前目录和另一个文件路径拼接而成的类路径。
- Windows：`.;C:\path\to\classes`
- Linux：`.:/path/to/classes`

事实上，`AppClassLoader` 最终的类路径，不仅仅包含 `-classpath` 的值，还会包含 `-javaagent` 指定的值。

### 字节数据转换为 Class 实例：defineClass
方法 `defineClass`，顾名思义，就是定义类，将字节数据转换为 `Class` 实例。在 `ClassLoader` 以及其子类中有很多同名方法，方法内各种处理和包装，最终都是为了使用 `name` 和字节数据等参数，调用 `native` 方法获得一个 `Class` 实例。
以下是定义类时最终可能调用的 `native` 方法。

```java
private native Class<?> defineClass0(String name, byte[] b, int off, int len,
                                     ProtectionDomain pd);

private native Class<?> defineClass1(String name, byte[] b, int off, int len,
                                     ProtectionDomain pd, String source);

private native Class<?> defineClass2(String name, java.nio.ByteBuffer b,
                                     int off, int len, ProtectionDomain pd,
                                     String source);
```

其方法参数有：
- `name`，目标类的名称。
- `byte[]` 或 `ByteBuffer` 类型的字节数据，`off` 和 `len` 只是为了定位传入的字节数组中关于目标类的字节数据，通常分别是 0 和字节数组的长度，毕竟专门构造一个包含无关数据的字节数组很无聊。
- `ProtectionDomain`，保护域，todo:
- `source`，`CodeSource` 的位置。

`defineClass` 方法的调用过程，其实就是从 `URLClassLoader` 开始，一层一层处理后再调用父类的 `defineClass` 方法，分别经过了 `SecureClassLoader` 和 `ClassLoader`。

#### URLClassLoader#defineClass
此方法是再 `URLClassLoader` 的 `findClass` 方法中，获得正确的 `Resource` 之后调用的，由 `private` 修饰。根据注释，它使用从指定资源获取的类字节来定义类，生成的类必须先解析才能使用。

```java
private Class<?> defineClass(String name, Resource res) throws IOException {
    long t0 = System.nanoTime();
    // 获取最后一个 . 的位置
    int i = name.lastIndexOf('.');
    // 返回资源的 CodeSourceURL
    URL url = res.getCodeSourceURL();
    if (i != -1) {
        // 截取包名 com.example
        String pkgname = name.substring(0, i);
        // Check if package already loaded.
        Manifest man = res.getManifest();
        definePackageInternal(pkgname, man, url);
    }
    // Now read the class bytes and define the class
    // 先尝试以 ByteBuffer 的形式返回字节数据，如果资源的输入流不是在 ByteBuffer 之上实现的，则返回 null
    java.nio.ByteBuffer bb = res.getByteBuffer();
    if (bb != null) {
        // Use (direct) ByteBuffer:
        // 不常用
        CodeSigner[] signers = res.getCodeSigners();
        CodeSource cs = new CodeSource(url, signers);
        sun.misc.PerfCounter.getReadClassBytesTime().addElapsedTimeFrom(t0);
        // 调用 java.security.SecureClassLoader#defineClass(java.lang.String, java.nio.ByteBuffer, java.security.CodeSource)
        return defineClass(name, bb, cs);
    } else {
        // 以字节数组的形式返回资源数据
        byte[] b = res.getBytes();
        // must read certificates AFTER reading bytes.
        // 必须再读取字节数据后读取证书，todo:
        CodeSigner[] signers = res.getCodeSigners();
        // 根据 URL 和签名者创建 CodeSource
        CodeSource cs = new CodeSource(url, signers);
        sun.misc.PerfCounter.getReadClassBytesTime().addElapsedTimeFrom(t0);
        // 调用 java.security.SecureClassLoader#defineClass(java.lang.String, byte[], int, int, java.security.CodeSource)
        return defineClass(name, b, 0, b.length, cs);
    }
}
```

`Resource` 类提供了 `getBytes` 方法，此方法以字节数组的形式返回字节数据。
```java
public byte[] getBytes() throws IOException {
    byte[] b;
    // Get stream before content length so that a FileNotFoundException
    // can propagate upwards without being caught too early
    // 在获取内容长度之前获取流，以便 FileNotFoundException 可以向上传播而不会过早被捕获（todo: 不理解）
    // 获取缓存的 InputStream
    InputStream in = cachedInputStream();

    // This code has been uglified to protect against interrupts.
    // Even if a thread has been interrupted when loading resources,
    // the IO should not abort, so must carefully retry, failing only
    // if the retry leads to some other IO exception.
    // 该代码为了防止中断有点丑陋。即使线程在加载资源时被中断，IO 也不应该中止，因此必须小心重试，只有当重试导致其他 IO 异常时才会失败。
    // 检测当前线程是否收到中断信号，收到的话则返回 true 且清除中断状态，重新变更为未中断状态。
    boolean isInterrupted = Thread.interrupted();
    int len;
    for (;;) {
        try {
            // 获取内容长度，顺利的话就跳出循环
            len = getContentLength();
            break;
        } catch (InterruptedIOException iioe) {
            // 如果获取内容长度时，线程被中断抛出了异常，捕获后清除中断状态
            Thread.interrupted();
            isInterrupted = true;
        }
    }

    try {
        b = new byte[0];
        if (len == -1) len = Integer.MAX_VALUE;
        int pos = 0;
        while (pos < len) {
            int bytesToRead;
            if (pos >= b.length) { // Only expand when there's no room
                // 如果当前读取位置已经大于等于数组长度
                // 本次待读取字节长度 = 剩余未读取长度和 1024 取较小值
                bytesToRead = Math.min(len - pos, b.length + 1024);
                if (b.length < pos + bytesToRead) {
                    // 如果当前读取位置 + 本次待读取字节长度 > 数组长度，则创建新数组并复制数据
                    b = Arrays.copyOf(b, pos + bytesToRead);
                }
            } else {
                // 数组还有空间，待读取字节长度 = 数组剩余空间
                bytesToRead = b.length - pos;
            }
            int cc = 0;
            try {
                // 读取数据
                cc = in.read(b, pos, bytesToRead);
            } catch (InterruptedIOException iioe) {
                // 如果读取时，线程被中断抛出了异常，捕获后清除中断状态
                Thread.interrupted();
                isInterrupted = true;
            }
            if (cc < 0) {
                // 如果读取返回值 < 0
                if (len != Integer.MAX_VALUE) {
                    // 且长度并未无限，表示提前检测到 EOF，抛出异常
                    throw new EOFException("Detect premature EOF");
                } else {
                    // 如果长度无限，表示读到了文件结尾，数组长度大于当前读取位置，创建新数组并复制长度
                    if (b.length != pos) {
                        b = Arrays.copyOf(b, pos);
                    }
                    break;
                }
            }
            pos += cc;
        }
    } finally {
        try {
            in.close();
        } catch (InterruptedIOException iioe) {
            isInterrupted = true;
        } catch (IOException ignore) {}

        if (isInterrupted) {
            // 如果 isInterrupted 为 true，代表中断过，重新将线程状态置为中断。
            Thread.currentThread().interrupt();
        }
    }
    return b;
}
```

在 `getByteBuffer` 之后会缓存 `InputStream` 以便调用 `getBytes` 时使用，方法由 `synchronized` 修饰。
```java
private synchronized InputStream cachedInputStream() throws IOException {
    if (cis == null) {
        cis = getInputStream();
    }
    return cis;
}
```

在这个例子中，`Resource` 的实例是 `URLClassPath` 中的匿名类 `FileLoader` 以 `Resource` 的匿名类的方式创建的。
```java
public InputStream getInputStream() throws IOException
{
    // 在该匿名类中，getInputStream 的实现就是简单地根据 FileLoader 中保存的 File 实例创建 FileInputStream 并返回。
    return new FileInputStream(file);
}

public int getContentLength() throws IOException
{
    // 在该匿名类中，getContentLength 的实现就是简单地根据 FileLoader 中保存的 File 实例获取长度。
    return (int)file.length();
};
```

#### SecureClassLoader#defineClass
`URLClassLoader` 继承自 `SecureClassLoader`，`SecureClassLoader` 提供并重载了 `defineClass` 方法，两个方法的注释均比代码长得多。
由注释可知，方法的作用是将字节数据（`byte[]` 类型或者 `ByteBuffer` 类型）转换为 `Class` 类型的实例，有一个可选的 `CodeSource` 类型的参数。

```java
protected final Class<?> defineClass(String name,
                                     byte[] b, int off, int len,
                                     CodeSource cs)
{
    return defineClass(name, b, off, len, getProtectionDomain(cs));
}

protected final Class<?> defineClass(String name, java.nio.ByteBuffer b,
                                     CodeSource cs)
{
    return defineClass(name, b, getProtectionDomain(cs));
}
```

方法中只是简单地将 `CodeSource` 类型的参数转换成 `ProtectionDomain` 类型，就调用 `ClassLoader` 的 `defineClass` 方法。
```java
private ProtectionDomain getProtectionDomain(CodeSource cs) {
    // 如果 CodeSource 为 null，直接返回 null
    if (cs == null)
        return null;

    ProtectionDomain pd = null;
    synchronized (pdcache) {
        // 先从 Map 缓存中获取 ProtectionDomain
        pd = pdcache.get(cs);
        if (pd == null) {
            // 从 CodeSource 中获取 PermissionCollection
            PermissionCollection perms = getPermissions(cs);
            // 缓存中没有，则创建一个 ProtectionDomain 并放入缓存
            pd = new ProtectionDomain(cs, perms, this, null);
            pdcache.put(cs, pd);
            if (debug != null) {
                debug.println(" getPermissions "+ pd);
                debug.println("");
            }
        }
    }
    return pd;
}
```

##### getPermissions
根据注释可知，此方法会返回给定 `CodeSource` 对象的权限。此方法由 `protect` 修饰，`AppClassLoader` 和 `URLClassLoader` 都有重写。当前 `ClassLoader` 是 `AppClassLoader`。

`AppClassLoader#getPermissions`，添加允许从类路径加载的任何类退出 VM的权限。
```java
protected PermissionCollection getPermissions(CodeSource codesource)
{
    // 调用父类 URLClassLoader 的 getPermissions
    PermissionCollection perms = super.getPermissions(codesource);
    // 允许从类路径加载的任何类退出 VM的权限。
    // todo: 这是否自定义的类加载器加载的类，可能不能退出 VM。
    perms.add(new RuntimePermission("exitVM"));
    return perms;
}
```

`SecureClassLoader#getPermissions`，添加一个读文件或读目录的权限。
```java
protected PermissionCollection getPermissions(CodeSource codesource)
{
    // 调用父类 SecureClassLoader 的 getPermissions
    PermissionCollection perms = super.getPermissions(codesource);

    URL url = codesource.getLocation();

    Permission p;
    URLConnection urlConnection;

    try {
        // FileURLConnection 实例
        urlConnection = url.openConnection();
        // 允许 read 的 FilePermission 实例
        p = urlConnection.getPermission();
    } catch (java.io.IOException ioe) {
        p = null;
        urlConnection = null;
    }

    if (p instanceof FilePermission) {
        // if the permission has a separator char on the end,
        // it means the codebase is a directory, and we need
        // to add an additional permission to read recursively
        // 如果文件路径以文件分隔符结尾，表示目录，需要在末尾添加"-"改为递归读的权限
        String path = p.getName();
        if (path.endsWith(File.separator)) {
            path += "-";
            p = new FilePermission(path, SecurityConstants.FILE_READ_ACTION);
        }
    } else if ((p == null) && (url.getProtocol().equals("file"))) {
        String path = url.getFile().replace('/', File.separatorChar);
        path = ParseUtil.decode(path);
        if (path.endsWith(File.separator))
            path += "-";
        p =  new FilePermission(path, SecurityConstants.FILE_READ_ACTION);
    } else {
        /**
         * Not loading from a 'file:' URL so we want to give the class
         * permission to connect to and accept from the remote host
         * after we've made sure the host is the correct one and is valid.
         */
        URL locUrl = url;
        if (urlConnection instanceof JarURLConnection) {
            locUrl = ((JarURLConnection)urlConnection).getJarFileURL();
        }
        String host = locUrl.getHost();
        if (host != null && (host.length() > 0))
            p = new SocketPermission(host,
                                        SecurityConstants.SOCKET_CONNECT_ACCEPT_ACTION);
    }

    // make sure the person that created this class loader
    // would have this permission

    if (p != null) {
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            final Permission fp = p;
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() throws SecurityException {
                    sm.checkPermission(fp);
                    return null;
                }
            }, acc);
        }
        perms.add(p);
    }
    return perms;
}
```

`SecureClassLoader#getPermissions`，延迟设置权限，在创建 `ProtectionDomain` 时再设置。
```java
protected PermissionCollection getPermissions(CodeSource codesource)
{
    // 检查以确保类加载器已初始化。在 SecureClassLoader 构造器最后会用一个布尔变量表示加载器初始化成功。
    // 从代码上看，似乎只能保证 SecureClassLoader 的构造器方法已执行完毕？
    check();
    // ProtectionDomain 延迟绑定，Permissions 继承 PermissionCollection 类。
    return new Permissions(); // ProtectionDomain defers the binding
}
```

##### ProtectionDomain
`ProtectionDomain` 的相关构造器参数：
- `CodeSource`
- `PermissionCollection`，如果不为 `null`，会设置权限为只读，表示权限在使用过程中不再修改；同时检查是否需要设置拥有全部权限。
- `ClassLoader`
- `Principal[]`

这样看来，`SecureClassLoader` 为了定义类做的处理，就是简单地创建一些关于权限的对象，并保存了 `CodeSource->ProtectionDomain` 的映射作为缓存。

#### ClassLoader#defineClass
抽象类 `ClassLoader` 中最终用于定义类的 `native` 方法 `define0`，`define1`，`define2` 都是由 `private` 修饰的，`ClassLoader` 提供并重载了 `defineClass` 方法作为使用它们的入口，这些 `defineClass` 方法都由 `protect` `final` 修饰，这意味着这些方法只能被子类使用，并且不能被重写。

```java
protected final Class<?> defineClass(String name, byte[] b, int off, int len)
    throws ClassFormatError
{
    return defineClass(name, b, off, len, null);
}

protected final Class<?> defineClass(String name, byte[] b, int off, int len,
                                     ProtectionDomain protectionDomain)
    throws ClassFormatError
{
    protectionDomain = preDefineClass(name, protectionDomain);
    String source = defineClassSourceLocation(protectionDomain);
    Class<?> c = defineClass1(name, b, off, len, protectionDomain, source);
    postDefineClass(c, protectionDomain);
    return c;
}

protected final Class<?> defineClass(String name, java.nio.ByteBuffer b,
                                     ProtectionDomain protectionDomain)
    throws ClassFormatError
{
    int len = b.remaining();

    // Use byte[] if not a direct ByteBufer:
    if (!b.isDirect()) {
        if (b.hasArray()) {
            return defineClass(name, b.array(),
                                b.position() + b.arrayOffset(), len,
                                protectionDomain);
        } else {
            // no array, or read-only array
            byte[] tb = new byte[len];
            b.get(tb);  // get bytes out of byte buffer.
            return defineClass(name, tb, 0, len, protectionDomain);
        }
    }

    protectionDomain = preDefineClass(name, protectionDomain);
    String source = defineClassSourceLocation(protectionDomain);
    Class<?> c = defineClass2(name, b, b.position(), len, protectionDomain, source);
    postDefineClass(c, protectionDomain);
    return c;
}
```

主要步骤：
1. `preDefineClass` 前置处理
2. `defineClassX`
3. `postDefineClass` 后置处理

##### preDefineClass
确定保护域 `ProtectionDomain`，并检查：
1. 未定义 `java.*` 类
2. 该类的签名者与包（`package`）中其余类的签名者相匹配

```java
private ProtectionDomain preDefineClass(String name,
                                        ProtectionDomain pd)
{
    // 检查 name 为 null 或者有可能是有效的二进制名称
    if (!checkName(name))
        throw new NoClassDefFoundError("IllegalName: " + name);

    // Note:  Checking logic in java.lang.invoke.MemberName.checkForTypeAlias
    // relies on the fact that spoofing is impossible if a class has a name
    // of the form "java.*"
    // 如果 name 以 java. 开头，则抛出异常
    if ((name != null) && name.startsWith("java.")) {
        throw new SecurityException
            ("Prohibited package name: " +
                name.substring(0, name.lastIndexOf('.')));
    }
    if (pd == null) {
        // 如果未传入 ProtectionDomain，取默认的 ProtectionDomain
        pd = defaultDomain;
    }

    // 存放了 package->certs 的 map 映射作为缓存，检查一个包内的 certs 都是一样的
    // todo: certs
    if (name != null) checkCerts(name, pd.getCodeSource());

    return pd;
}
```

##### defineClassSourceLocation
确定 `Class` 的 `CodeSource` 位置。
```java
private String defineClassSourceLocation(ProtectionDomain pd)
{
    CodeSource cs = pd.getCodeSource();
    String source = null;
    if (cs != null && cs.getLocation() != null) {
        source = cs.getLocation().toString();
    }
    return source;
}
```

##### defineClassX 方法
这些 `native` 方法使用了 `name`，字节数据，`ProtectionDomain` 和 `source` 等参数，像黑盒一样，在虚拟机中定义了一个类。

##### postDefineClass
在定义类后使用 `ProtectionDomain` 中的 `certs` 补充 `Class` 实例的 `signer` 信息，猜测在 `native` 方法 `defineClassX` 方法中，对 `ProtectionDomain` 做了一些修改。事实上，从代码上看，将 `CodeSource` 包装为 `ProtectionDomain` 传入后，除了 `defineClassX` 方法外，其他地方都是取出 `CodeSource` 使用。
```java
private void postDefineClass(Class<?> c, ProtectionDomain pd)
{
    if (pd.getCodeSource() != null) {
        // 获取证书
        Certificate certs[] = pd.getCodeSource().getCertificates();
        if (certs != null)
            setSigners(c, certs);
    }
}
```