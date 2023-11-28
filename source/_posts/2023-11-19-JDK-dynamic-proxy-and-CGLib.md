---
title: JDK 动态代理和 CGLib
date: 2023-11-19 14:20:39
tags:
    - java
    - jdk proxy
    - cglib
---

## 介绍

### JDK 动态代理

Java 标准库提供了动态代理功能，允许程序在运行期动态创建指定接口的实例。

### CGLib 动态代理

使用 ASM 框架，加载代理对象的 Class 文件，通过修改其字节码生成子类。

[cglib Github 仓库](https://github.com/cglib/cglib)

### 适用场景

- JDK 动态代理适用于实现接口的类，对未实现接口的类无能为力。
- CGLib 不要求类实现接口，但对 final 方法无能为力。

### 性能比较

- 在 JDK 8 以前，CGLib 性能更好
- 从 JDK 8 开始，JDK 动态代理性能更好

> 根据 README.md 的提醒，cglib 已经不再维护，且在较新版本的 JDK 尤其是 JDK 17+ 中表现不佳，官方推荐可以考虑迁移到 [ByteBuddy](https://bytebuddy.net/)。在如今越来越多的项目迁移到 JDK 17 的背景下，值得注意。

## 使用

### 代理对象的类和接口

代理对象的类和实现的接口：

- HelloService.java
```java
public interface HelloService {
    void sayHello(String name);
}
```

- HelloService.java
```java
public class HelloServiceImpl implements HelloService {
    @Override
    public void sayHello(String name) {
        if (name == null) {
            throw new IllegalArgumentException("name can not be null");
        }
        System.out.println("Hello " + name);
    }
}
```

### JDK 动态代理示例

- 自定义 InvocationHandler
```java
public class UserServiceInvocationHandler implements InvocationHandler {

    private Object target;

    public UserServiceInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            System.out.println("do sth. before invocation");
            Object ret = method.invoke(target, args);
            System.out.println("do sth. after invocation");
            return ret;
        } catch (Exception e) {
            System.out.println("do sth. when exception occurs");
            throw e;
        } finally {
            System.out.println("do sth. finally");
        }
    }
}
```
- 测试类
```java
public class JdkProxyTest {

    public static void main(String[] args) {
        HelloService target = new HelloServiceImpl();
        ClassLoader classLoader = target.getClass().getClassLoader();
        Class<?>[] interfaces = target.getClass().getInterfaces();
        UserServiceInvocationHandler invocationHandler = new UserServiceInvocationHandler(target);

        HelloService proxy = (HelloService) Proxy.newProxyInstance(classLoader, interfaces, invocationHandler);
        proxy.sayHello("Tom");
        System.out.println("=================");
        proxy.sayHello(null);
    }
}
```
- 结果
```console
do sth. before invocation
Hello Tom
do sth. after invocation
do sth. finally
=================
do sth. before invocation
do sth. when exception occurs
do sth. finally
Exception in thread "main" java.lang.reflect.UndeclaredThrowableException
	at com.sun.proxy.$Proxy0.sayHello(Unknown Source)
	at com.moralok.proxy.jdk.JdkProxyTest.main(JdkProxyTest.java:19)
Caused by: java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.moralok.proxy.jdk.UserServiceInvocationHandler.invoke(UserServiceInvocationHandler.java:18)
	... 2 more
Caused by: java.lang.IllegalArgumentException: name can not be null
	at com.moralok.proxy.HelloServiceImpl.sayHello(HelloServiceImpl.java:8)
```

### CGLib 动态代理示例

- 引入依赖
```xml
<dependencies>
    <dependency>
        <groupId>cglib</groupId>
        <artifactId>cglib</artifactId>
        <version>3.3.0</version>
    </dependency>
</dependencies>
```
- 自定义 MethodInterceptor
```java
public class UserServiceMethodInterceptor implements MethodInterceptor {

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        try {
            System.out.println("do sth. before invocation");
            Object ret = proxy.invokeSuper(obj, args);
            System.out.println("do sth. after invocation");
            return ret;
        } catch (Exception e) {
            System.out.println("do sth. when exception occurs");
            throw e;
        } finally {
            System.out.println("do sth. finally");
        }
    }
}
```
- 测试类
```java
public class CglibTest {

    public static void main(String[] args) {
        UserServiceMethodInterceptor methodInterceptor = new UserServiceMethodInterceptor();
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(HelloServiceImpl.class);
        enhancer.setCallback(methodInterceptor);

        HelloService proxy = (HelloService) enhancer.create();
        proxy.sayHello("Tom");
        System.out.println("=================");
        proxy.sayHello(null);
    }
}
```
- 结果
```console
do sth. before invocation
Hello Tom
do sth. after invocation
do sth. finally
=================
do sth. before invocation
do sth. when exception occurs
do sth. finally
Exception in thread "main" java.lang.IllegalArgumentException: name can not be null
	at com.moralok.proxy.HelloServiceImpl.sayHello(HelloServiceImpl.java:8)
	at com.moralok.proxy.HelloServiceImpl$$EnhancerByCGLIB$$c51b2c31.CGLIB$sayHello$0(<generated>)
	at com.moralok.proxy.HelloServiceImpl$$EnhancerByCGLIB$$c51b2c31$$FastClassByCGLIB$$c068b511.invoke(<generated>)
	at net.sf.cglib.proxy.MethodProxy.invokeSuper(MethodProxy.java:228)
	at com.moralok.proxy.cglib.UserServiceMethodInterceptor.intercept(UserServiceMethodInterceptor.java:14)
	at com.moralok.proxy.HelloServiceImpl$$EnhancerByCGLIB$$c51b2c31.sayHello(<generated>)
	at com.moralok.proxy.cglib.CglibTest.main(CglibTest.java:18)
```

## 查看 JDK 生成的代理类

使用以下语句，将在工作目录下生成代理类的 Class 文件。

```java
System.setProperty("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
```

```java
public final class $Proxy0 extends Proxy implements HelloService {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        // 将 InvocationHandler 传递给父类 Proxy
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    // 代理方法调用 InvocationHandler 的 invoke 方法
    public final void sayHello(String var1) throws  {
        try {
            super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    // 静态代码块，初始化 Method 属性。
    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("com.moralok.proxy.HelloService").getMethod("sayHello", Class.forName("java.lang.String"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

## 查看 CGLib 生成的子类

使用以下语句，将 CGLib 生成的子类的 Class 文件输出到指定目录，会发现出现了 3 个 Class 文件。

```java
System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "C:\\Users\\username\\Class");
```

- HelloServiceImpl$$EnhancerByCGLIB$$c51b2c31.class，代理类
- HelloServiceImpl$$FastClassByCGLIB$$a5654167.class，被代理类的 FastClass
- HelloServiceImpl$$EnhancerByCGLIB$$c51b2c31$$FastClassByCGLIB$$c068b511.class，代理类的 FastClass

{% asset_img "Snipaste_2023-11-28_02-01-40.png" CGLib 生成的子类的 Class 文件 %}

### 代理类定义

继承了被代理类。

```java
public class HelloServiceImpl$$EnhancerByCGLIB$$c51b2c31 extends HelloServiceImpl implements Factory {
}
```

### 静态代码块

```java
static {
    // 调用静态钩子方法
    CGLIB$STATICHOOK1();
}

static void CGLIB$STATICHOOK1() {
    CGLIB$THREAD_CALLBACKS = new ThreadLocal();
    CGLIB$emptyArgs = new Object[0];
    Class var0 = Class.forName("com.moralok.proxy.HelloServiceImpl$$EnhancerByCGLIB$$c51b2c31");
    Class var1;
    // 获取 Object 类的 equals、toString、hashCode、clone 这几个特定方法的 Method 对象
    Method[] var10000 = ReflectUtils.findMethods(new String[]{"equals", "(Ljava/lang/Object;)Z", "toString", "()Ljava/lang/String;", "hashCode", "()I", "clone", "()Ljava/lang/Object;"}, (var1 = Class.forName("java.lang.Object")).getDeclaredMethods());
    // 还生成了相对应的 Method 属性保存（为了减少一次寻址吗？）
    CGLIB$equals$1$Method = var10000[0];
    // 为每一个 Method 创建一个 MethodProxy
    CGLIB$equals$1$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/Object;)Z", "equals", "CGLIB$equals$1");
    CGLIB$toString$2$Method = var10000[1];
    CGLIB$toString$2$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/String;", "toString", "CGLIB$toString$2");
    CGLIB$hashCode$3$Method = var10000[2];
    CGLIB$hashCode$3$Proxy = MethodProxy.create(var1, var0, "()I", "hashCode", "CGLIB$hashCode$3");
    CGLIB$clone$4$Method = var10000[3];
    CGLIB$clone$4$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/Object;", "clone", "CGLIB$clone$4");
    // 被代理类的方法也做相同处理
    CGLIB$sayHello$0$Method = ReflectUtils.findMethods(new String[]{"sayHello", "(Ljava/lang/String;)V"}, (var1 = Class.forName("com.moralok.proxy.HelloServiceImpl")).getDeclaredMethods())[0];
    CGLIB$sayHello$0$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/String;)V", "sayHello", "CGLIB$sayHello$0");
}
```

MethodProxy 稍后再做介绍。

### 构造器方法

构造器方法内，调用了绑定回调（Callbacks）方法。

```java
public HelloServiceImpl$$EnhancerByCGLIB$$c51b2c31() {
    CGLIB$BIND_CALLBACKS(this);
}

// 标识是否已经绑定过回调
private boolean CGLIB$BOUND;

private static final void CGLIB$BIND_CALLBACKS(Object var0) {
    HelloServiceImpl$$EnhancerByCGLIB$$c51b2c31 var1 = (HelloServiceImpl$$EnhancerByCGLIB$$c51b2c31)var0;
    if (!var1.CGLIB$BOUND) {
        // 未绑定过回调则进行绑定，更新标识
        var1.CGLIB$BOUND = true;
        // 先获取 THREAD_CALLBACKS
        Object var10000 = CGLIB$THREAD_CALLBACKS.get();
        if (var10000 == null) {
            // 如果为 null，再获取 STATIC_CALLBACKS
            var10000 = CGLIB$STATIC_CALLBACKS;
            if (var10000 == null) {
                // 如果仍然为 null，直接返回
                return;
            }
        }

        // 每一个 Callback （像之前的 Method 一样）都有专门的属性保存
        var1.CGLIB$CALLBACK_0 = (MethodInterceptor)((Callback[])var10000)[0];
    }

}
```

### 生成的代理方法

CGLib 会为每一个代理方法生成两个对应的方法，一个直接调用父类方法，一个则调用回调（拦截器）的 intercept 方法。

```java
final void CGLIB$sayHello$0(String var1) {
    super.sayHello(var1);
}

public final void sayHello(String var1) {
    // 获取回调（拦截器）
    MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
    if (var10000 == null) {
        // 如果为 null，先进行回调绑定
        CGLIB$BIND_CALLBACKS(this);
        var10000 = this.CGLIB$CALLBACK_0;
    }

    if (var10000 != null) {
        // 如果回调（拦截器）不为 null，则调用 intercept 方法
        var10000.intercept(this, CGLIB$sayHello$0$Method, new Object[]{var1}, CGLIB$sayHello$0$Proxy);
    } else {
        // 否则直接调用父类方法
        super.sayHello(var1);
    }
}
```

CGLib 通过继承实现动态代理的过程，在查看生成的子类的 Class 后，是非常容易理解的。拦截器的参数有代理对象、Method、方法参数和 MethodProxy 对象。

## 分析 MethodProxy

如何在拦截器中调用被代理的方法呢？就是通过 MethodProxy 实现的。

### 创建 MethodProxy

MethodProxy 是 CGLib 为每一个代理方法创建的方法代理，当调用拦截的方法时，它被传递给 MethodInterceptor 对象的 intercept 方法。它可以用于调用原始方法，或对同一类型的不同对象调用相同方法。

```java
CGLIB$sayHello$0$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/String;)V", "sayHello", "CGLIB$sayHello$0");

public static MethodProxy create(Class c1, Class c2, String desc, String name1, String name2) {
    MethodProxy proxy = new MethodProxy();
    // sayHello 方法签名
    proxy.sig1 = new Signature(name1, desc);
    // CGLIB$sayHello$0 方法签名
    proxy.sig2 = new Signature(name2, desc);
    // 被代理类和代理类
    proxy.createInfo = new CreateInfo(c1, c2);
    return proxy;
}
```

CreateInfo 静态内部类，保存被代理类和代理类以及其他一些信息。

```java
private static class CreateInfo
{
    // 被代理类
    Class c1;
    // 代理类
    Class c2;
    NamingPolicy namingPolicy;
    GeneratorStrategy strategy;
    boolean attemptLoad;
    
    public CreateInfo(Class c1, Class c2)
    {
        this.c1 = c1;
        this.c2 = c2;
        AbstractClassGenerator fromEnhancer = AbstractClassGenerator.getCurrent();
        if (fromEnhancer != null) {
            namingPolicy = fromEnhancer.getNamingPolicy();
            strategy = fromEnhancer.getStrategy();
            attemptLoad = fromEnhancer.getAttemptLoad();
        }
    }
}
```

### FastClass 和方法索引对

#### 调用原始方法 invokeSuper

MethodProxy 通过 invokeSuper 调用原始方法（父类方法）。

```java
// invoke 方法的代码相似
public Object invokeSuper(Object obj, Object[] args) throws Throwable {
    try {
        // 初始化，生成 FastClassInfo
        init();
        FastClassInfo fci = fastClassInfo;
        // 调用原始（父类）方法
        return fci.f2.invoke(fci.i2, obj, args);
    } catch (InvocationTargetException e) {
        throw e.getTargetException();
    }
}
```

#### 生成 FastClass 信息

```java
private void init()
{
    // 双重校验锁，生成 FastClass 和方法索引对
    if (fastClassInfo == null)
    {
        synchronized (initLock)
        {
            if (fastClassInfo == null)
            {
                CreateInfo ci = createInfo;

                FastClassInfo fci = new FastClassInfo();
                // 生成 FastClass
                fci.f1 = helper(ci, ci.c1);
                fci.f2 = helper(ci, ci.c2);
                // 获取方法索引
                fci.i1 = fci.f1.getIndex(sig1);
                fci.i2 = fci.f2.getIndex(sig2);
                fastClassInfo = fci;
                createInfo = null;
            }
        }
    }
}
```

#### FastClass 信息

- f1 是被代理类的 FastClass 对象，i1 是 `CGLIB$sayHello$0` 方法在生成的 FastClass 中的索引。
- f2 是代理类的 FastClass 对象，i2 是 `sayHello` 方法在生成的 FastClass 中的索引。

invoke 方法根据传入的方法索引，快速定位要调用对象 obj 的哪个方法。

> CGLib 完全有能力获得 `CGLIB$sayHello$0` 的 Method 对象，通过反射实现调用，这样处理逻辑更加清楚。但是早期 Java 反射的性能并不好，通过 FastClass 机制避免使用反射从而提升了性能。

```java
private static class FastClassInfo
{
    FastClass f1;
    FastClass f2;
    int i1;
    int i2;
}
```

### FastClass 的 invoke 方法

以代理类的 FastClass `HelloServiceImpl$$EnhancerByCGLIB$$c51b2c31$$FastClassByCGLIB$$c068b511` 为例，当传入的方法索引为 16 时，就会调用 `CGLIB$sayHello$0` 方法。

1. 获取代理对象
2. 根据传入的方法索引，调用

```java
public Object invoke(int var1, Object var2, Object[] var3) throws InvocationTargetException {
    HelloServiceImpl..EnhancerByCGLIB..c51b2c31 var10000 = (HelloServiceImpl..EnhancerByCGLIB..c51b2c31)var2;
    int var10001 = var1;

    try {
        switch (var10001) {
            case 0:
                return new Boolean(var10000.equals(var3[0]));
            // ...
            case 16:
                var10000.CGLIB$sayHello$0((String)var3[0]);
                return null;
            // ...
        }
    } catch (Throwable var4) {
        throw new InvocationTargetException(var4);
    }

    throw new IllegalArgumentException("Cannot find matching method/constructor");
}
```

#### 获取方法索引

怎么知道方法的索引呢？在初始化 FastClass 信息时，不仅生成了 FastClass，还通过 getIndex 获取方法的索引。

> 在 JDK 7 之后，switch 不仅可以支持 int、enum，还能支持 String，CGLib 这样实现是出于兼容性的考虑还是说有什么性能提升？

```java
public int getIndex(Signature var1) {
    String var10000 = var1.toString();
    switch (var10000.hashCode()) {
        // ...
        case -1721191351:
            if (var10000.equals("CGLIB$sayHello$0(Ljava/lang/String;)V")) {
                return 16;
            }
            break;
        // ...
    }

    return -1;
}
```

## 总结和思考

两者在使用上是相仿的。

- 对于两者的源码，读得不多。有时候会感慨，看这么多年前的代码，还是感觉吃力。有时候想，如果不好好看源码，心里不踏实；如果花很多时间理清楚了，但是发现更多只是知道了一些细节，于整体理解的提升不大，又会感觉不值得。
- 但也提醒自己，不要太在意，用得本就不多，涉及源码的机会更是没有，如果方方面面都要细究，人生太短，智商不够，等涉足相关问题再回头研究。
- 基础的用法和概念应该了解，不然看到 Spring AOP 源码时，分不清 Spring 封装的边界在哪里。

> 借着梳理 Spring 的机会回头再看，又感觉轻松不少。
