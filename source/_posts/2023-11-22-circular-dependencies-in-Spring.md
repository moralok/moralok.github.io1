---
title: Spring 中的循环依赖
date: 2023-11-22 08:28:26
tags:
    - java
    - spring
---

## 循环依赖的介绍和讨论

### 什么是循环依赖？

当 bean A 依赖另一个 bean B，bean B 也依赖了 bean A，我们称之为循环依赖：

```console
Bean A -> bean B -> bean A
```

首先，我们应该将循环依赖和 Spring 中的循环依赖问题分开看待。循环依赖是一个正常的现象，一个 employee 依赖他的 department，department 拥有许多 employee。先实例化 employee 后实例化 department，然后为它们设置依赖，或者颠倒顺序进行操作，并不会发生什么问题。

### Spring 中的循环依赖问题

当 Spring 加载所有的 beans 时，会进行**依赖注入**处理。Spring 并不是先将所有的 bean 实例化，再去进行依赖注入，而是实例化一个 bean 后，立即对它进行依赖注入，为此它会递归地实例化 bean 的依赖。
实际上，以上的过程同样并不会产生什么大问题，在**实例化和依赖注入分成两个阶段**的情况下，你可以**轻而易举地保存和获取已经实例化的 bean**。唯一的问题是，获取的已经实例化的 bean 可能尚未初始化完毕（比如依赖尚未全部注入），那么你只需要确保它在初始化完毕前不被使用即可。
这样你通过两个 map，一个保存已经初始化完毕、可用的完成品 bean，一个保存尚未初始化完毕、不可用的半成品 bean，就能解决问题了。

> 在一些资料中，你会看到有人强调如果只是想要解决常规的循环引用，那么只需要两个缓存。

{% asset_img "Pasted image 20231122210105.png" 循环引用-常规情况的解决方案 %}

但是问题并不总是那么简单，如果实例化和依赖注入不能分为两个阶段，如果 B 依赖的不再是简单的 A 对象，而是 A 的代理，那么上述方案就不再适用了。

#### 构造器方法形成的循环依赖

如果 A 的构造器方法需要 B，B 的构造器方法需要 A，那么在 A 的实例化阶段就需要 B 的实例，B 的实例化阶段又需要 A，这就陷入了死循环。

> 虽然我们常说 Spring 解决了循环依赖问题，但实际上，Spring 并没有解决所有情形的循环依赖问题。
在构造器方法形成的循环依赖中，需要人为地介入，使用 @Lazy 注解告诉 Spring，延迟初始化 Bean 来解决。
如果是 prototype 类型的 bean 发生循环依赖，Spring 也会抛出异常。显然每次都创建新的 bean 必然会导致无限循环。

#### 循环依赖中出现代理

Spring 鼎鼎大名的核心功能，除了 IOC，还有 AOP。在 AOP 的场景中，bean A 的完成品不是简单的 A 对象，而是一个 A 的代理。

思路其实并不难，既然需要 A 的代理，那么在获取 B 依赖的 A 时，直接根据已有的半成品 A 创建代理就好了。

#### 解决方案的思路小结

当我们脱离 Spring 具体的代码讨论循环依赖问题，我们会发现解决的思路是简单、清晰和理所当然的。事实上 Spring 的解决方案也是如此。

1. 循环依赖本身是普通的，手动可解决的。
2. Spring 依赖注入时，虽然 bean B 依赖的 bean A 尚未初始化完毕，但是已经实例化，可以用来赋值
3. Spring AOP 中，既然 bean B 依赖的 bean A 需要是 A 对象的代理，那么就创建代理，再用来赋值

## 测试用例和流程图预览

{% asset_img "Pasted image 20231122233750.png" 循环引用-流程图 %}

- CircularA
```java
public class CircularA {
    private CircularB circularB;
    public CircularB getCircularB() {
        return circularB;
    }
    public void setCircularB(CircularB circularB) {
        this.circularB = circularB;
    }
}
```
- CircularB
```java
public class CircularB {
    private CircularA circularA;
    public CircularA getCircularA() {
        return circularA;
    }
    public void setCircularA(CircularA circularA) {
        this.circularA = circularA;
    }
}
```
- circular-reference-test.xml
```xml
<beans>
    <bean id="circularA" class="com.moralok.bean.CircularA">
        <property name="circularB" ref="circularB"/>
    </bean>
    <bean id="circularB" class="com.moralok.bean.CircularB">
        <property name="circularA" ref="circularA"/>
    </bean>
</beans>
```
- 测试类
```java
public class CircularReferenceTest {
    @Test
    public void testRegular() {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("circular-reference-test.xml");
        CircularA circularA = (CircularA) applicationContext.getBean("circularA");
    }
}
```

## 第一次获取 circularA

### AbstractBeanFactory#doGetBean(circularA)

1. 从缓存中获取 circularA（先不看具体代码）
2. 因缓存中不存在，就创建 circularA

```java
protected <T> T doGetBean(
        final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
        throws BeansException {
    // 尝试从缓存中获取 circularA，结果为 null
    Object sharedInstance = getSingleton(beanName);
    // ...
    // 再次从缓存中获取 circularA，如果为 null，就创建
    sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
        @Override
        public Object getObject() throws BeansException {
            try {
                return createBean(beanName, mbd, args);
            }
        }
    });
    // ...
    return (T) bean;
}
```

### 标记 bean 是否正在创建中

在创建 bean circularA 之前，会再次调用 getSingleton(String, ObjectFactory) 尝试获取，这个方法包裹住了 createBean 方法，也在一前一后处理 bean 是否正在创建中的标志。在后续，circularB 怎么判断 circularA 是否属于正在创建中就是依靠于此。

```java
isSingletonCurrentlyInCreation(beanName)
```

> 这里的是否正在创建中，并不是狭义地指代 bean 对象是否已经实例化，而是指代 bean 是否已经实例化和初始化完毕。circular A 在初始化阶段，去获取 circularB，在 circularB 视角中，circular A 就处于正在创建中。

{% asset_img "Pasted image 20231122210026.png" 循环引用-是否正在创建中 %}

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    // ...
    // 标记为正在创建中
    beforeSingletonCreation(beanName);
    boolean newSingleton = false;
    // ...
    try {
        singletonObject = singletonFactory.getObject();
        newSingleton = true;
    }
    // ...
    // 从正在创建中的集合移除
    afterSingletonCreation(beanName);
    if (newSingleton) {
        addSingleton(beanName, singletonObject);
    }
    return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

## 创建 circularA

1. 实例化 circularA
2. 初始化 circularA
    - 对 circularA 进行依赖注入时：getBean(circularB)

很重要的是，在实例化 circularA 之后，尚未进行初始化工作之前，如果 circularA 允许早期暴露，将会被包装为 ObjectFactory 缓存到 singletonFactory（三级缓存） 中。

值得注意的是：

- 如果 circularA 不需要早期暴露，那么这个 ObjectFactory 是不会被使用到的
- 如果 circularA 需要早期暴露，比如它依赖的 circularB 同时依赖它，circularB 将回调 getEarlyBeanReference 方法获得 circular A 的早期 bean 引用。

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
        throws BeanCreationException {
    // ...
    // 实例化 bean
    instanceWrapper = createBeanInstance(beanName, mbd, args);
    // ...
    // 如果单例允许提前暴露的话，就将实例包装为 ObjectFactory 保存在 map singletonFactories 中
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                return getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }
    // 初始化 bean，对 bean 的属性进行依赖注入
    Object exposedObject = bean;
    try {
        populateBean(beanName, mbd, instanceWrapper);
        if (exposedObject != null) {
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
    }
    // ...
    return exposedObject;
}
```

## 依赖注入时获取 circularB

### 填充 Bean 属性

填充 Bean 属性的 populateBean 方法很复杂，这里只关注它对 circularA 的依赖注入将间接地调用 getBean(circularB)。

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    // ...
    // 应用属性值
    applyPropertyValues(beanName, mbd, bw, pvs);
}
```

```java
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
    // ...
    // 如果有必要解析 value
    Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
    // ...
    bw.setPropertyValues(new MutablePropertyValues(deepCopy));
    // ...
}
```

```java
public Object resolveValueIfNecessary(Object argName, Object value) {
    // ...
    // 如果是 RuntimeBeanReference 类型，就解析引用
    if (value instanceof RuntimeBeanReference) {
        RuntimeBeanReference ref = (RuntimeBeanReference) value;
        return resolveReference(argName, ref);
    }
    // ...
}
```

```java
private Object resolveReference(Object argName, RuntimeBeanReference ref) {
    // ...
    // 解析引用将调用 getBean
    Object bean = this.beanFactory.getBean(refName);
    this.beanFactory.registerDependentBean(refName, this.beanName);
    return bean;
    // ...
}
```

### 获取 circularB

AbstractBeanFactory#doGetBean(circularB)，获取 circularB 将经过和 circularA 一样的流程，进入 populateBean(circularB) 对其进行依赖注入，进而再次去获取 circularA。

## 第二次获取 circularA

### AbstractBeanFactory#doGetBean(circularA)

1. 第二次获取 circularA 时，仍然先尝试从缓存中获取
```java
protected <T> T doGetBean(
        final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
        throws BeansException {
    // ...
    // 尝试从缓存中获取 circularA，结果为 null
    Object sharedInstance = getSingleton(beanName);
    // ...
}
```
2. 这次将从缓存中得到先前创建的 circularA。
```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    // 如果 singletonObjects 中不存在，且 bean 正在创建过程中
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            // 从 earlySingletonObjects 获取
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                // 如果 earlySingletonObjects 中不存在，且允许早期引用，就从 singletonFactories 中获取
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

在第一次获取 circularA 时，我们跳过了查看 getSingleton(String) 的代码。其实代码的逻辑并不复杂，但是要理解为什么这么做，却需要回过头来反复品味和思考。

## 从缓存中获取 circularA

1. 从 singletonObjects（一级缓存） 获取 circularA，不存在
2. circularA 属于正在创建中，从 earlySingletonObjects（二级缓存） 获取，仍然不存在
3. allowEarlyReference 为真，从 singletonFactories（三级缓存） 获取（circularA 实例化后被包装为 ObjectFactory 保存在这里）
4. 调用 getObject 获得对象

```java
addSingletonFactory(beanName, new ObjectFactory<Object>() {
    @Override
    public Object getObject() throws BeansException {
        return getEarlyBeanReference(beanName, mbd, bean);
    }
});
```

### 回调 getEarlyBeanReference

如果存在 SmartInstantiationAwareBeanPostProcessor，将使用这些后处理器获得早期 bean 引用。

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (bean != null && !mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        // 如果存在 InstantiationAwareBeanPostProcessors
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                // 如果 bp 是 SmartInstantiationAwareBeanPostProcessor 类型
                SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
                // 尝试创建代理
                exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
                if (exposedObject == null) {
                    return null;
                }
            }
        }
    }
    return exposedObject;
}
```

## 获取早期 bean 引用

了解{% post_link how-does-Spring-AOP-create-proxy-beans 'Spring AOP 如何创建代理 beans' %}，才能更好地理解这部分内容。

通过后处理器 getEarlyBeanReference 获取早期 bean 引用时，可能返回的就是 circularA 对象，但是如果 circularA 属于需要创建代理的 bean，就会在这时候立即为它创建代理，而在之后 BeanPostProcessor 处理时就不会再创建代理了。

以 AbstractAutoProxyCreator 为例，它是自动代理创建者的抽象类，同时实现了 SmartInstantiationAwareBeanPostProcessor 和 BeanPostProcessor 接口。

### SmartInstantiationAwareBeanPostProcessor 的方法实现

SmartInstantiationAwareBeanPostProcessor 接口允许在获取 circularA 的早期引用时，就尝试将 circular A 包装为代理。

```java
public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    if (!this.earlyProxyReferences.contains(cacheKey)) {
        this.earlyProxyReferences.add(cacheKey);
    }
    // 如有必要，包装为代理
    return wrapIfNecessary(bean, beanName, cacheKey);
}
```

### BeanPostProcessor 的方法实现

BeanPostProcessor 接口允许在初始化时，调用初始化方法后，尝试将 circular A 包装为代理。如果已经在获取 circularA 的早期引用时就将其包装为代理，则不再进行包装。

```java
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        // 判断早期代理引用中是否存在
        if (!this.earlyProxyReferences.contains(cacheKey)) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
```

## 再思 Spring bean 的三级缓存

著名的“三级缓存”，实际上就是三个存放 Bean 的 map：

- singletonObjects
- earlySingletonObjects
- singletonFactories

在很多网上的资料中，都称 Spring 通过使用三级缓存的设计解决了循环引用问题。
同时我也看到有人反思，这样翻译对学习者造成了很大的困扰，代码中并没有多级 cache 的意味，称之为“三个缓存”比“三级缓存”更合理也更容易理解。
三个存放 Bean 的 map 事实上是相互独立的，甚至说它们是互斥的，**一个 Bean 在同一时间最多只能存在于其中一个 map 中**。

对我个人而言，我对反对者的观点深有同感，如果我没有看过面经，即时我熟读并理解代码，我可能都无法回答 Spring 中的三级缓存是什么。甚至我会被三级缓存这个名词所震慑，在了解它之前在心里放大它的复杂性。
但是在不断阅读的过程中（可能也有已有记忆的加持），我也会感受到称之为“三级缓存”的合理性。这里的分级含义更多体现的是 Bean 的晋升过程。

