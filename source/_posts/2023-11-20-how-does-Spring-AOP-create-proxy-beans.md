---
title: Spring AOP 如何创建代理 beans
date: 2023-11-20 07:23:15
tags:
    - java
    - spring
    - aop
---

Spring AOP 是基于代理实现的，它既支持 JDK 动态代理也支持 CGLib。

- 在什么时候创建代理对象的？
- 怎么创建代理对象的？

## 过程简单图解

{% asset_img "Pasted image 20231120223506.png" Spring AOP 创建代理的过程 %}

## 准备工作

- 引入依赖
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>4.3.12.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
    <version>4.3.12.RELEASE</version>
</dependency>
```
- 目标对象类
```java
public class MathCalculator {

    public int div(int i, int j) {
        return i / j;
    }
}
```
- 切面类
```java
@Aspect
public class LogAspects {

    @Pointcut("execution(public int com.moralok.aop.MathCalculator.*(..))")
    public void pointCut() {

    }

    @Before("pointCut()")
    public void logStart(JoinPoint joinPoint) {
        System.out.println(joinPoint.getSignature().getName() + "除法运行@Before。。。参数列表为 " + Arrays.asList(joinPoint.getArgs()) + "");
    }

    @After("pointCut()")
    public void logEnd(JoinPoint joinPoint) {
        System.out.println(joinPoint.getSignature().getName() + "除法结束@After。。。");
    }

    @AfterReturning(value = "pointCut()", returning = "result")
    public void logReturn(JoinPoint joinPoint, Object result) {
        System.out.println(joinPoint.getSignature().getName() + "除法正常返回@AfterReturning。。。运行结果 " + result);
    }

    @AfterThrowing(value = "pointCut()", throwing = "e")
    public void logException(JoinPoint joinPoint, Exception e) {
        System.out.println(joinPoint.getSignature().getName() + "除法异常@AfterThrowing。。。异常信息 " + e.getMessage());
    }

    @Around(value = "execution(public String com.moralok.bean.Car.getName(..))")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println(joinPoint.getSignature().getName() + " @Around开始");
        Object proceed = joinPoint.proceed();
        System.out.println(joinPoint.getSignature().getName() + " @Around结束");
        return proceed;
    }
}
```
- 配置类
```java
@Configuration
@EnableAspectJAutoProxy
public class AopConfig {

    @Bean
    public MathCalculator mathCalculator() {
        return new MathCalculator();
    }

    @Bean
    public LogAspects logAspects() {
        return new LogAspects();
    }
}
```
- 测试类
```java
public class AopTest {

    @Test
    public void aopTest() {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(AopConfig.class);
        MathCalculator mathCalculator = ac.getBean(MathCalculator.class);
        mathCalculator.div(1, 1);
        mathCalculator.div(1, 0);
        ac.close();
    }
}
```
- Debug 断点的判断条件（可选）
```console
beanName.equals("mathCalculator")
```

## 创建代理 Bean 和创建普通 Bean 的区别

其实创建代理 Bean 的过程和{% post_link how-does-Spring-load-beans '创建普通 Bean 的过程' %}直到进行初始化处理（initializeBean）前都是一样的。更具体地说，如很多资料所言，Spring 创建代理对象的工作，是在应用后置处理器阶段完成的。

### 常规的入口 getBean

mathCalculator 以 getBean 方法为起点，开始创建的过程。

```java
@Override
public void preInstantiateSingletons() throws BeansException {
    // ...（mathCalculator）
    getBean(beanName);
    // ...
}
```

### 应用后置处理器

在正常地实例化 Bean 后，初始化 Bean 时，会对 Bean 实例应用后置处理器。

可是，**究竟是哪一个后置处理器做的呢**？

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
    // ...
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    // ...
    invokeInitMethods(beanName, wrappedBean, mbd);
    // ...
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    return wrappedBean;
}
```

### AnnotationAwareAspectJAutoProxyCreator

在本示例中，创建代理的后置处理器就是 AnnotationAwareAspectJAutoProxyCreator，它继承自 AbstractAutoProxyCreator，AbstractAutoProxyCreator 实现了 BeanPostProcessor 接口。

那么，**它是什么时候，怎么加入到 beanFactory 中呢**？

PS: 显然，还有其他继承自 AbstractAutoProxyCreator 的后置处理器，暂时不谈。

#### BeanPostProcessor 的方法

postProcessBeforeInitialization 和 postProcessAfterInitialization 方法，前者什么都没做，后者在必要时对 Bean 进行包装。

- **`AbstractAutoProxyCreator#postProcessAfterInitialization` 就是创建代理对象的入口。**
- **wrapIfNecessary 就是将 Bean 包装成代理 Bean 的入口方法**。

```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
    implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
    // ...
    @Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // 什么都没做
        return bean;
    }
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean != null) {
            Object cacheKey = getCacheKey(bean.getClass(), beanName);
            if (!this.earlyProxyReferences.contains(cacheKey)) {
                // 如有必要，将 bean 包装成代理对象
                return wrapIfNecessary(bean, beanName, cacheKey);
            }
        }
        return bean;
    }
    // ...
}
```

## 创建代理 Bean 的过程

### 按需包装成代理 wrapIfNecessary

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    // 判断是否直接返回 bean
    if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // 如果有适用于当前 bean 的 advise 则为其创建代理
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

### AbstractAutoProxyCreator 视角，创建代理

AbstractAutoProxyCreator#createProxy，创建一个 ProxyFactory，将工作交给它处理。

1. 创建一个代理工厂 ProxyFactory
2. 设置相关信息
3. 通过 ProxyFactory 获取代理

```java
protected Object createProxy(
        Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
    }

    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);

    if (!proxyFactory.isProxyTargetClass()) {
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        }
        else {
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }

    return proxyFactory.getProxy(getProxyClassLoader());
}
```

### ProxyFactory 视角，获取代理

ProxyFactory#getProxy，创建一个 AopProxy 并委托它实现 getProxy。

> AopProxy 的含义与职责从字面上有点不好理解。

```java
public Object getProxy(ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
}
```

#### ProxyFactor视角，创建 AopProxy

ProxyFactory#createAopProxy，获取一个 AopProxyFactory 创建 AopProxy。

```java
protected final synchronized AopProxy createAopProxy() {
    if (!this.active) {
        activate();
    }
    // 获取 AopProxy 工厂并创建一个 AopProxy
    return getAopProxyFactory().createAopProxy(this);
}
```

#### AopProxyFactory视角，创建 AopProxy

AopProxyFactory#createAopProxy。

- AopProxyFactory 有且仅有一个默认实现 DefaultAopProxyFactory。
- createAopProxy 方法会根据配置信息，返回具体实现：开箱即用的有 JdkDynamicAopProxy 或者 ObjenesisCglibAopProxy。

这里的处理，决定了 Spring AOP 会使用哪一种动态代理实现。比如 Spring AOP 默认使用 JDK 动态代理，如果目标对象实现了接口 Spring 会使用 JDK 动态代理，这些结论的依据就在于此。

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
}
```

### 获取代理 AopProxy#getProxy

AopProxy 视角，获取代理。

#### JDK 动态代理

JdkDynamicAopProxy。

```java
@Override
public Object getProxy(ClassLoader classLoader) {
    // ...
    // JDK 动态代理，已经和 Spring 无关
    return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

##### InvocationHandler 的 invoke 方法

根据 `Proxy.newProxyInstance(classLoader, proxiedInterfaces, this)` 可知，this 也就是 JdkDynamicAopProxy 同时也是一个 InvocationHandler，它必然实现了 invoke 方法，当代理对象调用方法时，就会进入到 invoke 方法中。

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // ...
}
```

#### CGLib 动态代理

ObjenesisCglibAopProxy。

```java
@Override
public Object getProxy(ClassLoader classLoader) {
    // ...
    // CGLib 动态代理，已经和 Spring 无关
    Enhancer enhancer = createEnhancer();
    // ...
    return createProxyClassAndInstance(enhancer, callbacks);
}
```

##### 为什么 Spring 中没有依赖 CGLib

你可能会注意到 Spring 中并没有直接依赖 CGLib，像 Enhancer 所在的包是 `org.springframework.cglib.proxy`。根据文档：

> 从 spring 3.2 开始，不再需要将 cglib 添加到类路径中，因为 cglib 类在 org.springframework 下重新打包并分布在 spring-core jar 中。 这样做既是为了方便，也是为了避免与使用不同版本 cglib 的其他项目发生潜在冲突。

## 创建代理前的准备

在前面预留了一些问题，当初我在看网上的资料时就有这些困惑。

### Bean 后置处理器 AspectJAwareAdvisorAutoProxyCreator 在什么时候，怎么加入到 beanFactory 中的？

Debug 停留在 Spring 上下文刷新方法中的 finishBeanFactoryInitialization。

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    // ...
    invokeBeanFactoryPostProcessors(beanFactory);
    // ...
    finishBeanFactoryInitialization(beanFactory);
    // ...
}
```

从 beanFatory 的 beanDefinitionMap 可以观察到，配置类 AopConfig 中的 MathCalculator 和 LogAspect 的信息已经就位。

{% asset_img "Snipaste_2023-11-20_15-51-48.png" 初始化 beanFactory 时的 beanDefinitionMap %}

从 beanFactory 的 beanProcessor 可以观察到，AnnotationAwareAspectJAutoProxyCreator 已经就位。

{% asset_img "Snipaste_2023-11-20_15-56-15.png" 初始化 beanFactory 时的 beanPostProcessor %}

#### @EnableXXX 的魔法

注解 @EnableXXX 往往伴随着注解 @Import，在 invokeBeanFactoryPostProcessors(beanFactory) 中，工厂后置处理器 ConfigurationClassPostProcessor 会处理它。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
    boolean proxyTargetClass() default false;
    boolean exposeProxy() default false;
}
```

在 ConfigurationClassPostProcessor 的处理中，因为 AspectJAutoProxyRegistrar 实现了 ImportBeanDefinitionRegistrar，registerBeanDefinitions 方法会被调用，AnnotationAwareAspectJAutoProxyCreator 的 beanDefinition 随之被注册到 beanFactory，因 AnnotationAwareAspectJAutoProxyCreator 实现了 BeanPostProcessor 被提前创建。

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(
            AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // 如有必要注册 AspectJAnnotationAutoProxyCreator
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
        // 根据配置设置一些属性
        AnnotationAttributes enableAspectJAutoProxy =
                AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
        if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
        }
        if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
            AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
        }
    }
}
```

```java
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, Object source) {
    return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}
```

### 切面类 LogAspect 的解析是在什么时候？

进入创建 Bean 的方法 createBean 后，除了 doCreateBean，应额外留意 resolveBeforeInstantiation 方法。

1. `Object bean = resolveBeforeInstantiation(beanName, mbdToUse)`，在实例化前进行解析。
2. `Object beanInstance = doCreateBean(beanName, mbdToUse, args)`，创建 Bean 的具体过程。

```java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException { 
    // ...
    try {
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    // ...

    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    // ...
    return beanInstance;
}
```

#### 入口方法 resolveBeforeInstantiation

根据注释，该方法给 BeanPostProcessors 一个机会提前返回一个代理对象。在本示例中，返回 null，但是方法在第一次执行后已经提前解析得到 advisors 并缓存。

```java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    Object bean = null;
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            Class<?> targetType = determineTargetType(beanName, mbd);
            if (targetType != null) {
                // 注意，应用的是实例化前的处理
                bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                if (bean != null) {
                    // 注意，应用的是初始化后的处理
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                }
            }
        }
        mbd.beforeInstantiationResolved = (bean != null);
    }
    return bean;
}
```

#### InstantiationAwareBeanPostProcessor

应用 InstantiationAwareBeanPostProcessor 的 postProcessBeforeInstantiation。

```java
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            // 循环依次处理
            Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
            if (result != null) {
                return result;
            }
        }
    }
    return null;
}
```

AnnotationAwareAspectJAutoProxyCreator 不仅仅是一个 BeanPostProcessor，它还是一个 InstantiationAwareBeanPostProcessor。

```java
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
    Object cacheKey = getCacheKey(beanClass, beanName);
    
    if (beanName == null || !this.targetSourcedBeans.contains(beanName)) {
        if (this.advisedBeans.containsKey(cacheKey)) {
            return null;
        }
        if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return null;
        }
    }
    
    if (beanName != null) {
        TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
        if (targetSource != null) {
            this.targetSourcedBeans.add(beanName);
            Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
            Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
            this.proxyTypes.put(cacheKey, proxy.getClass());
            return proxy;
        }
    }
    
    return null;
}
```

和 wrapIfNecessary 方法对比，容易发现两者有不少相似的处理。

{% asset_img "Snipaste_2023-11-20_21-33-34.png" 实例化前后创建代理的对比 %}

> **注意：以下方法应注意是否被子类重写**。

org.springframework.aop.aspectj.autoproxy.AspectJAwareAdvisorAutoProxyCreator#shouldSkip

```java
protected boolean shouldSkip(Class<?> beanClass, String beanName) {
    // 查找并缓存 advisors
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    // ...
}
```

org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean

```java
protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, TargetSource targetSource) {
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    // ...
}
```

org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#findEligibleAdvisors

```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    // 查找并缓存 advisors
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    // ...
}
```

容易注意到两者在创建代理前，**都会调用 findCandidateAdvisors 方法查找候选的 advisors**，其实这也是我们想要找的对切面类的解析处理所在。

#### 查找并缓存 advisors

org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors

```java
protected List<Advisor> findCandidateAdvisors() {
    List<Advisor> advisors = super.findCandidateAdvisors();
    advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
    return advisors;
}
```

org.springframework.aop.aspectj.annotation.BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors

```java
public List<Advisor> buildAspectJAdvisors() {
    List<String> aspectNames = this.aspectBeanNames;  
    if (aspectNames == null) {
        // 第一次进入，没有缓存
        synchronized (this) {
            aspectNames = this.aspectBeanNames;
            if (aspectNames == null) {
                List<Advisor> advisors = new LinkedList<Advisor>();
                aspectNames = new LinkedList<String>();
                String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                        this.beanFactory, Object.class, true, false);
                for (String beanName : beanNames) {
                    // ...
                    // 如果是切面，解析得到 advisors
                    if (this.advisorFactory.isAspect(beanType)) {
                        aspectNames.add(beanName);
                        // ...
                        if (this.beanFactory.isSingleton(beanName)) {
                            this.advisorsCache.put(beanName, classAdvisors);
                        }
                        else {
                            this.aspectFactoryCache.put(beanName, factory);
                        }
                    }
                }
                this.aspectBeanNames = aspectNames;
                return advisors;
            }
        }
    }

    if (aspectNames.isEmpty()) {
        return Collections.emptyList();
    }
    // 以后进来读缓存
    List<Advisor> advisors = new LinkedList<Advisor>();
    for (String aspectName : aspectNames) {
        List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
        if (cachedAdvisors != null) {
            advisors.addAll(cachedAdvisors);
        }
        else {
            MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
            advisors.addAll(this.advisorFactory.getAdvisors(factory));
        }
    }
    return advisors;
}
```

可以通过 `beanFactory->beanPostProcessors->aspectJAdvisorsBuilder->advisorsCache` 观察 advisors 的查找情况。

{% asset_img "Snipaste_2023-11-20_20-50-04.png" 观察 advisors %}