---
title: Spring Bean 加载过程
date: 2023-08-10 14:56:51
tags:
    - java
    - spring
---

## Spring Bean 生命周期

{% asset_img "Pasted image 20231118213120.png" Spring Bean 生命周期 %}

## 获取 Bean

获取指定 Bean 的入口方法是 getBean，在 Spring 上下文刷新过程中，就依次调用 `AbstractBeanFactory#getBean(java.lang.String)` 方法获取 `non-lazy-init` 的 Bean。

```java
public Object getBean(String name) throws BeansException {
    // 具体工作由 doGetBean 完成
    return doGetBean(name, null, null, false);
}
```

### deGetBean

作为公共处理逻辑，由 AbstractBeanFactory 自己实现。

```java
protected <T> T doGetBean(
    final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly) 
    throws BeansException {
    // 转换名称：去除 FactoryBean 的前缀 &，将别名转换为规范名称
    final String beanName = transformedBeanName(name);
    Object bean;

    // 检查单例缓存中是否已存在
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        // ...
        // 如果已存在，直接返回该实例或者使用该实例（FactoryBean）创建并返回对象
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }
    else {
        // 如果当前 Bean 是一个正在创建中的 prototype 类型，表明可能发生循环引用
        // 注意：Spring 并未解决 prototype 类型的循环引用问题，要抛出异常
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
        
        // 如果当前 beanFactory 没有 bean 定义，去 parent beanFactory 中查找
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            String nameToLookup = originalBeanName(name);
            if (args != null) {
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else {
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
        }
        
        if (!typeCheckOnly) {
            // 标记为至少创建过一次
            markBeanAsCreated(beanName);
        }
        
        try {
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // 确保 bean 依赖的 bean（构造器参数） 都已实例化
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    if (isDependent(beanName, dep)) {
                        // 注意：Spring 并未解决构造器方法中的循环引用问题，要抛异常
                    }
                    // 注册依赖关系，确保先销毁被依赖的 bean
                    registerDependentBean(dep, beanName);
                    // 递归，获取依赖的 bean
                    getBean(dep);
                }
            }
        }

        if (mbd.isSingleton()) {
            // 如果是单例类型（绝大多数都是此类型）
            // 再次从缓存中获取，如果仍不存在，则使用传入的 ObjectFactory 创建
            sharedInstance = getSingleton(beanName, new ObjectFactory<Object>(
                {
                    @Override
                    public Object getObject() throws BeansException {
                        try {
                            // 创建 bean
                            return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                            // 由于可能已经提前暴露，需要显示地销毁                
                            destroySingleton(beanName);
                            throw ex;
                        }
                    }
                });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        }
        else if (mbd.isPrototype()) {
            // 如果是原型类型，每次都新创建一个
            // ...
        }
        else {
            // 如果是其他 scope 类型
            // ...
        }
    }
    catch (BeansException ex) {
        cleanupAfterBeanCreationFailure(beanName);
        throw ex;
    }
}
```

### getSingleton

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    // 加锁
    synchronized (this.singletonObjects) {
        // 再次从缓存中获取（和调用前从缓存中获取构成双重校验）
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            if (this.singletonsCurrentlyInDestruction) {
                // 如果正在销毁单例，则抛异常
                // 注意：不要在销毁方法中调用获取 bean 方法
            }
            // 创建前，先注册到正在创建中的集合
            // 在出现循环引用时，第二次进入 doGetBean，用此作为判断标志
            beforeSingletonCreation(beanName);
            boolean newSingleton = false;
            // ...
            try {
                // 使用传入的单例工厂创建对象
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            }
            catch (IllegalStateException ex) {
                // 如果异常的出现是因为 bean 被创建了，就忽略异常，否则抛出异常
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    throw ex;
                }
            }
            catch (BeanCreationException ex) {
                // ...
            }
            finally {
                // ...
                // 创建后，从正在创建中集合移除
                afterSingletonCreation(beanName);
            }
            if (newSingleton) {
                // 添加单例到缓存
                addSingleton(beanName, singletonObject);
            }
        }
        return (singletonObject != NULL_OBJECT ? singletonObject : null);
    }
}
```

## 创建 Bean

createBean 是创建 Bean 的入口方法，由 AbstractBeanFactory 定义，由 AbstractAutowireCapableBeanFactory 实现。

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
    // ...
    try {
        // 给 Bean 后置处理器一个返回代理的机会
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    // ...
    // 常规的创建 Bean
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    return beanInstance;
}
```

### doCreateBean

常规的创建 Bean 的具体工作是由 doCreateBean 完成的。

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) throws BeanCreationException {
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 使用相应的策略创建 bean 实例，例如通过工厂方法或者有参、无参构造器方法
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
    Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
    mbd.resolvedTargetType = beanType;
    
    // ...

    // 使用 ObjectFactory 封装实例并缓存，以解决循环引用问题
    boolean earlySingletonExposure = (mbd.isSingleton() 
        && this.allowCircularReferences 
        && isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                return getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }
    
    Object exposedObject = bean;
    try {
        // 填充属性（包括解析依赖的 bean）
        populateBean(beanName, mbd, instanceWrapper);
        if (exposedObject != null) {
            // 初始化 bean
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
    }
    // ...

    // 如有需要，将 bean 注册为一次性的，以供 beanFactory 在关闭时调用销毁方法
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    // ...

    return exposedObject;
}
```

#### createBeanInstance

创建 Bean 实例，并使用 BeanWrapper 封装。实例化的方式：

1. 工厂方法
2. 构造器方法
	1. 有参
	2. 无参

#### populateBean

为创建出的实例填充属性，包括解析当前 bean 所依赖的 bean。

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    PropertyValues pvs = mbd.getPropertyValues();
    // ...
    
    // 给 InstantiationAwareBeanPostProcessors 一个机会，
    // 在设置 bean 属性前修改 bean 状态，可用于自定义的字段注入
    boolean continueWithPropertyPopulation = true;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    continueWithPropertyPopulation = false;
                    break;
                }
            }
        }
    }
    
    // 是否继续填充属性的流程
    if (!continueWithPropertyPopulation) {
        return;
    }
    
    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME
        || mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        // 根据名称注入
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }
        
        // 根据类型注入
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }
        pvs = newPvs;
    }
    
    // 是否存在 InstantiationAwareBeanPostProcessors
    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    // 是否需要检查依赖
    boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);
    
    if (hasInstAwareBpps || needsDepCheck) {
        PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        if (hasInstAwareBpps) {
            // 后置处理 PropertyValues
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvs == null) {
                        return;
                    }
                }   
            }
        }
        if (needsDepCheck) {
            checkDependencies(beanName, mbd, filteredPds, pvs);
        }
    }
    // 将属性应用到 bean 上（常规情况下，前面的处理都用不上）
    applyPropertyValues(beanName, mbd, bw, pvs);
}
```

#### initializeBean

在填充完属性后，实例就可以进行初始化工作：

1. invokeAwareMethods，让 Bean 通过 xxxAware 接口感知一些信息
2. 调用 BeanPostProcessor 的 postProcessBeforeInitialization 方法
3. invokeInitMethods，调用初始化方法
4. 调用 BeanPostProcessor 的 postProcessAfterInitialization 方法

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
    // 处理 Aware 接口的相应方法
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                invokeAwareMethods(beanName, bean);
                return null;
            }
        }, getAccessControlContext());
    }
    else {
        invokeAwareMethods(beanName, bean);
    }
    
    // 应用 BeanPostProcessor 的 postProcessBeforeInitialization 方法
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }
    
    try {
        // 调用初始化方法
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
            (mbd != null ? mbd.getResourceDescription() : null),
            beanName, "Invocation of init method failed", ex);
    }
    
    if (mbd == null || !mbd.isSynthetic()) {
        // 应用 BeanPostProcessor 的 postProcessAfterInitialization 方法
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```

##### 处理 Aware 接口的相应方法

让 Bean 在初始化中，感知（获知）和自身相关的资源，如 beanName、beanClassLoader 或者 beanFactory。

```java
private void invokeAwareMethods(final String beanName, final Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
```

##### 调用初始化方法

1. 如果 bean 实现 InitializingBean 接口，调用 afterPropertiesSet 方法
2. 如果自定义 init 方法且满足调用条件，同样进行调用

```java
protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd) throws Throwable {
    // 是否实现 InitializingBean 接口，是的话调用 afterPropertiesSet 方法
    // 给 bean 一个感知属性已设置并做出反应的机会
    boolean isInitializingBean = (bean instanceof InitializingBean);
    if (isInitializingBean
        && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        if (System.getSecurityManager() != null) {
            try {
                AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
                    @Override
                    public Object run() throws Exception {
                        ((InitializingBean) bean).afterPropertiesSet();
                        return null;
                    }
                }, getAccessControlContext());
            }
            catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        }
        else {
            ((InitializingBean) bean).afterPropertiesSet();
        }
    }

    // 如果存在自定义的 init 方法且方法名称不是 afterPropertiesSet，判断是否调用
    if (mbd != null) {
        String initMethodName = mbd.getInitMethodName();
        if (initMethodName != null
            && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName))
            && !mbd.isExternallyManagedInitMethod(initMethodName)) {
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```

##### BeanPostProcessor 处理

在调用初始化方法前后，BeanPostProcessor 先后进行两次处理。其实和 BeanPostProcessor 相关的代码都非常相似：

1. 获取 Processor 列表
2. 判断 Processor 类型是否是当前需要的
3. 对 bean 进行处理

```java
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName) throws BeansException {

    Object result = existingBean;
    for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
        result = beanProcessor.postProcessBeforeInitialization(result, beanName);
        if (result == null) {
            return result;
        }
    }
    return result;
}

public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) throws BeansException {

    Object result = existingBean;
    for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
        result = beanProcessor.postProcessAfterInitialization(result, beanName);
        if (result == null) {
            return result;
        }
    }
    return result;
}
```

#### 再思 Bean 的初始化

以下代码片段一度让我困惑，从注释看，初始化 Bean 实例的工作包括了 populateBean 和 initializeBean，但是 initializeBean 方法的含义就是初始化 Bean。在 initializeBean 方法中，调用了 invokeInitMethods 方法，其含义仍然是调用初始化方法。
在更熟悉代码后，我有一种微妙的、个人性的体会，在 Spring 源码中，有时候视角的变化是很快的，痕迹是很小的。如果不加以理解和区分，很容易迷失在相似的描述中。以此处为例，“初始化 Bean 和 Bean 的初始化”扩展开来是 “Bean 工厂初始化一个 Bean 和 Bean 自身进行初始化”。


```java
// Initialize the bean instance.
Object exposedObject = bean;
try {
    populateBean(beanName, mbd, instanceWrapper);
    if (exposedObject != null) {
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
}
```

在注释这一行，视角是属于 BeanFactory（AbstractAutowireCapableBeanFactory）。从工厂的视角，面对一个刚刚创建出来的 Bean 实例，需要完成两方面的工作：

1. 为 Bean 实例填充属性，包括解析依赖，为 Bean 自身的初始化做好准备。
2. Bean 自身的初始化。

在万事俱备之后，就是 Bean 自身的初始化工作。由于 Spring 的高度扩展性，这部分并不只是单纯地调用初始化方法，还包含 Aware 接口和 BeanPostProcessor 的相关处理，前者偏属于 Java 对象层面，后者偏属于 Spring Bean 层面。
在认同 BeanPostProcessor 的处理属于 Bean 自身初始化工作的一部分后，@PostConstruct 注解的方法被称为 Bean 的初始化方法也就不那么违和了，因为它的实现原理正是 BeanPostProcessor，这仍然在 initializeBean 的范围内。
