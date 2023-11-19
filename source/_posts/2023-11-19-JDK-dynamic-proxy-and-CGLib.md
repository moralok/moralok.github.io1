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

## 总结和思考

两者在使用上是相仿的。

||JDK 动态代理|CGLib 动态代理|
|--|--|--|
|差异|必须实现接口|子类继承，不适用 final 方法|
|拦截器|InvocationHandler|MethodInterceptor|
|额外信息|ClassLoader、interfaces|Class|
|创建代理|Proxy.newProxyInstance|Enhancer.create|

- 对于两者的源码，读得不多。有时候会感慨，看别人这么多年前的代码，还是感觉吃力，一下子会失去动力和信息。
- 但也提醒自己，不要太在意，用得本就不多，涉及源码的机会更是没有，如果方方面面都要细究，人生太短，智商不够，等涉足相关问题再回头研究。
- 基础的用法和概念应该了解，不然看到 Spring AOP 源码时，分不清 Spring 封装的边界在哪里。
