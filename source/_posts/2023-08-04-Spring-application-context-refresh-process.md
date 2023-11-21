---
title: Spring 应用 context 刷新流程
date: 2023-08-04 09:24:48
tags:
    - java
    - spring
---

### context 刷新流程简单图解

#### 刷新流程

{% asset_img "Pasted image 20231122005901.png" Spring context 刷新流程 %}

#### 刷新流程中的组件

{% asset_img "Pasted image 20231122021817.png" Spring context 刷新流程中的组件 %}

### 上下文刷新 AbstractApplicationContext#refresh
```java
public void refresh() throws BeansException, IllegalStateException {
    // 刷新和销毁的同步监视器。
    synchronized (this.startupShutdownMonitor) {
        // 1. 准备 context 以供刷新。
        prepareRefresh();
        // 2. 告诉 context 子类刷新内部 beanFactory 并返回。
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        // 3. 准备 beanFactory 以供在此 context 中使用。
        prepareBeanFactory(beanFactory);
        try {
            // 4. 允许 context 子类对 bean 工厂进行后处理。
            postProcessBeanFactory(beanFactory);
            // 5. 调用在 context 中注册为 bean 的工厂后处理器。
            invokeBeanFactoryPostProcessors(beanFactory);
            // 6. 注册拦截 Bean 创建的 Bean 后处理器。
            registerBeanPostProcessors(beanFactory);
            // 7. 初始化 context 的消息源。
            initMessageSource();
            // 8. 为 context 初始化事件多播器。
            initApplicationEventMulticaster();
            // 9. 在特定 context 子类中初始化其他特殊 bean。
            onRefresh();
            // 10. 检查监听器 beans 并注册。
            registerListeners();
            // 11. 实例化所有剩余的（非惰性初始化）单例。
            finishBeanFactoryInitialization(beanFactory);
            // 12. 最后一步：发布相应的事件。
            finishRefresh();
        }
        catch (BeansException ex) {
            // ...
            // 销毁已经创建的单例，以防止资源未正常释放。
            destroyBeans();
            // 重置 'active' flag.
            cancelRefresh(ex);
            throw ex;
        }
        finally {
            resetCommonCaches();
        }
    }
}
```

### 准备 context 以供刷新 prepareRefresh

准备此 context 以供刷新，设置其启动日期和活动标志以及执行属性源的初始化。

> 编写管理资源的容器时，可以参考。

```java
protected void prepareRefresh() {
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);
    if (logger.isInfoEnabled()) {
        logger.info("Refreshing " + this);
    }
    // 初始化 PropertySource，默认什么都不做。
    initPropertySources();
    // 校验所有被标记为 required 的 properties 都可以被解析。
    getEnvironment().validateRequiredProperties();
    // 允许收集早期的 ApplicationEvents，当事件多播器可用就发送。
    this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
}
```

### obtainFreshBeanFactory

告诉子类刷新内部 bean 工厂并返回，返回的实例类型为 **DefaultListableBeanFactory**。在这里完成了配置文件的读取，初步注册了 bean 定义。

> 我大概这辈子都不会想理清楚这里面关于 XML 文件的解析过程，但是我知道可以在这里观察到 beanFactory 因为配置文件注册了哪些 bean。

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 刷新 beanFactory
    // 使用 XmlBeanDefinitionReader 加载配置文件中的 bean 定义
    refreshBeanFactory();
    // 返回 beanFactory
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (logger.isDebugEnabled()) {
        logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
    }
    return beanFactory;
}
```

### 准备 beanFactory

配置 BeanFactory 以供在此 context 中使用，例如 context 的类加载器和一些后处理器，手动注册一些单例。

1. 为 beanFactory 配置 context 相关的资源，如类加载器
2. 添加 Bean 后处理器
    - ApplicationContextAwareProcessor，context 回调，注入特定类型时可触发自定义逻辑
    - ApplicationListenerDetector，检测 ApplicationListener
3. 手动注册单例

> ignoreDependencyInterface 和 registerResolvableDependency 在理解之后比单纯地记忆它们有趣许多。

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 告诉内部 beanFactory 使用 context 的类加载器等等。
    beanFactory.setBeanClassLoader(getClassLoader());
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // 为内部 beanFactory 配置 context 回调。
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    // 如果一个 bean 的依赖实现了以下接口，忽略该依赖的检查和自动装配。
    // 例如在 populateBean 时，如果 bena 的依赖存在 set 方法，就会去解析，调用 getBean
    // 被设置 ignoreDependencyInterface 的依赖，仍然可以通过后置处理器进行依赖注入，例如以下的类型会使用上面那个后置处理器的回调方法注入。
    // 因此 @Autowire 这些通过后置处理器实现依赖注入的注解，也不会受影响
    // 这样设计的一个可能是往往注入这些类型时，希望触发某些事件。
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // BeanFactory 之类的接口没有在普通工厂中注册为可解析类型，直接为它们指定 bean。
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // 提前注册后处理器以检测内部 beans 是否是一个 ApplicationListener。
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // 检测是否有 LoadTimeWeaver，如果存在就准备编织。
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // 手动注册默认的环境 beans。
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}
```

> 在创建 Bean 开始前注册的单例，都属于手动注册的单例 manualSingletonNames

```java
public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
    super.registerSingleton(beanName, singletonObject);

    if (hasBeanCreationStarted()) {
        // Cannot modify startup-time collection elements anymore (for stable iteration)
        synchronized (this.beanDefinitionMap) {
            if (!this.beanDefinitionMap.containsKey(beanName)) {
                Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames.size() + 1);
                updatedSingletons.addAll(this.manualSingletonNames);
                updatedSingletons.add(beanName);
                this.manualSingletonNames = updatedSingletons;
            }
        }
    }
    else {
        // Still in startup registration phase
        if (!this.beanDefinitionMap.containsKey(beanName)) {
            this.manualSingletonNames.add(beanName);
        }
    }

    clearByTypeCache();
}
```

### postProcessBeanFactory

在标准初始化后修改内部 beanFactory，默认什么都不做。

### invokeBeanFactoryPostProcessors

实例化并调用所有在 context 中注册的 beanFactory 后处理器，需遵循顺序规则。具体的处理被委托给 PostProcessorRegistrationDelegate。

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
    // ...
}
```

invokeBeanFactoryPostProcessors 方法堪比裹脚布。

**关于调用顺序的规则**：

1. BeanFactoryPostProcessor 分为 context 添加的和 beanFactory 注册的，前者优于后者
2. BeanFactoryPostProcessor 又可分为常规的和 BeanDefinitionRegistryPostProcessor，后者优于前者
3. PriorityOrdered 优于 Ordered 优于剩余的

可能新增 beanDefinition 的情况：

1. BeanDefinitionRegistryPostProcessor 可能在 beanFactory 中引入新的 beanDefinition


```java
public static void invokeBeanFactoryPostProcessors(
		ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

    // 存储已处理过的后处理器
    Set<String> processedBeans = new HashSet<String>();

    // 第一阶段：BeanDefinitionRegistryPostProcessor
    if (beanFactory instanceof BeanDefinitionRegistry) {
        // 如果 beanFactory 同时是 BeanDefinitionRegistry 类型
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
        // 存储常规的 BeanFactoryPostProcessor
        List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<BeanFactoryPostProcessor>();
        // 存储 BeanDefinitionRegistryPostProcessor
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new LinkedList<BeanDefinitionRegistryPostProcessor>();

        // 第 0 轮，先对 context 注册的 BeanFactoryPostProcessor 进行分类
        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                BeanDefinitionRegistryPostProcessor registryProcessor =
                        (BeanDefinitionRegistryPostProcessor) postProcessor;
                // 分类的同时，直接调用 BeanDefinitionRegistryPostProcessor
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                registryProcessors.add(registryProcessor);
            }
            else {
                regularPostProcessors.add(postProcessor);
            }
        }

        // 将 BeanDefinitionRegistryPostProcessors 按是否实现 PriorityOrdered，Ordered，以及剩余的进行分类
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();

        // 第 1 轮，先处理 beanFactory 中实现了 PriorityOrdered 的 BeanDefinitionRegistryPostProcessor
        String[] postProcessorNames =
                beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                // getBean 并添加到当前的后处理器集合
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        // 排序后添加
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();

        // 第 2 轮，Ordered
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();

        // 第 3 轮， 调用剩余的 BeanDefinitionRegistryPostProcessors 直到没有新的出现。
        // 后出现的 PriorityOrdered 不比前面的 Ordered 更早被处理
        boolean reiterate = true;
        while (reiterate) {
            reiterate = false;
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (!processedBeans.contains(ppName)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    reiterate = true;
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();
        }

        // 调用目前出现的 BeanFactoryPostProcessors
        // 仍遵循 PriorityOrdered、Ordered、Regular（registry）、Regular（context 添加的） 的顺序
        invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
        invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    }

    else {
        // 否则，直接调用 context 注册的 beanFactoryPostProcessors
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }

    // 第二阶段：BeanFactoryPostProcessor
    String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

    // 将 BeanFactoryPostProcessor 按是否实现 PriorityOrdered，Ordered，以及剩余的进行分类
    List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
    List<String> orderedPostProcessorNames = new ArrayList<String>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
    for (String ppName : postProcessorNames) {
        if (processedBeans.contains(ppName)) {
            // 第一阶段已处理，跳过
        }
        else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // 第 1 轮，BeanFactoryPostProcessors that implement PriorityOrdered.
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

    // 第 2 轮，BeanFactoryPostProcessors that implement Ordered.
    List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
    for (String postProcessorName : orderedPostProcessorNames) {
        orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

    // 第 3 轮，剩余的 BeanFactoryPostProcessors.
    List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
    for (String postProcessorName : nonOrderedPostProcessorNames) {
        nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

    // 清理缓存的 merged bean definitions 因为 post-processors 可能已经修改了原来的 metadata
    beanFactory.clearMetadataCache();
}
```

### registerBeanPostProcessors

注册拦截 `bean` 创建的 `bean` 后处理器。具体的处理被委托给 PostProcessorRegistrationDelegate。

```java
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
```

registerBeanPostProcessors 相比之下是一条清新的裹脚布。这里特别区分 3 种类型的 Bean 后处理器：

- MergedBeanDefinitionPostProcessor
- InstantiationAwareBeanPostProcessor 感知实例化
- DestructionAwareBeanPostProcessor 感知销毁

ApplicationListenerDetector 既是 MergedBeanDefinitionPostProcessor，又是 DestructionAwareBeanPostProcessor，在初始化后将 listener 加入，在销毁前将 listener 移除。

```java
public static void registerBeanPostProcessors(
		ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
    // 获取 BeanPostProcessor 的名称
    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

    // 注册 BeanPostProcessorChecker，在 BeanPostProcessor 实例化期间创建 bean 时，记录一条消息。
    // 即，当一个 bean 不能被所有 BeanPostProcessors 处理时，记录。
    int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
    beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

    // 将 BeanPostProcessors 按 implement PriorityOrdered，Ordered，和剩余的进行分类。
    List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
    List<BeanPostProcessor> internalPostProcessors = new ArrayList<BeanPostProcessor>();
    List<String> orderedPostProcessorNames = new ArrayList<String>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
    for (String ppName : postProcessorNames) {
        if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
            priorityOrderedPostProcessors.add(pp);
            if (pp instanceof MergedBeanDefinitionPostProcessor) {
                internalPostProcessors.add(pp);
            }
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // 第 1 轮，注册 BeanPostProcessors that implement PriorityOrdered.
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

    // 第 2 轮，注册 BeanPostProcessors that implement Ordered.
    List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
    for (String ppName : orderedPostProcessorNames) {
        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
        orderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
        }
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, orderedPostProcessors);

    // 第 3 轮，注册剩余的 regular BeanPostProcessors.
    List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
    for (String ppName : nonOrderedPostProcessorNames) {
        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
        nonOrderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
        }
    }
    registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

    // 最后（第 4 轮）, 排序并注册 internal BeanPostProcessors.
    sortPostProcessors(internalPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, internalPostProcessors);

    // Re-register post-processor for detecting inner beans as ApplicationListeners,
    // moving it to the end of the processor chain (for picking up proxies etc).
    // 重新注册用于检测 ApplicationListeners 的 Bean 后处理器，将其移动到处理器链的最后（用于获取代理）。
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```

添加 BeanPostProcessor 时

1. 先移除
2. 再添加
3. 判断类型并记录标记
    - 感知实例化的后处理器
    - 感知销毁的后处理器

```java
public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {
    Assert.notNull(beanPostProcessor, "BeanPostProcessor must not be null");
    this.beanPostProcessors.remove(beanPostProcessor);
    this.beanPostProcessors.add(beanPostProcessor);
    if (beanPostProcessor instanceof InstantiationAwareBeanPostProcessor) {
        this.hasInstantiationAwareBeanPostProcessors = true;
    }
    if (beanPostProcessor instanceof DestructionAwareBeanPostProcessor) {
        this.hasDestructionAwareBeanPostProcessors = true;
    }
}
```

### initMessageSource

初始化消息源。 如果在此 context 中未定义，则使用父级的。

```java
protected void initMessageSource() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
        this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
        // 设置 parent MessageSource.
        if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
            HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
            if (hms.getParentMessageSource() == null) {
                // 只有当 parent MessageSource 尚未注册才将 parent context 设置为 parent MessageSource
                hms.setParentMessageSource(getInternalParentMessageSource());
            }
        }
        if (logger.isDebugEnabled()) {
            logger.debug("Using MessageSource [" + this.messageSource + "]");
        }
    }
    else {
        // 使用代理 messageSource，以此接收 getMessage 调用。
        DelegatingMessageSource dms = new DelegatingMessageSource();
        dms.setParentMessageSource(getInternalParentMessageSource());
        this.messageSource = dms;
        beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
        if (logger.isDebugEnabled()) {
            logger.debug("Unable to locate MessageSource with name '" + MESSAGE_SOURCE_BEAN_NAME +
                    "': using default [" + this.messageSource + "]");
        }
    }
}
```

### initApplicationEventMulticaster

初始化 `ApplicationEventMulticaster`。 如果上下文中未定义，则使用 `SimpleApplicationEventMulticaster`。可以看得出代码的结构和 initMessageSource 是类似的。

```java
protected void initApplicationEventMulticaster() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
        this.applicationEventMulticaster =
                beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
        if (logger.isDebugEnabled()) {
            logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
        }
    }
    else {
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
        if (logger.isDebugEnabled()) {
            logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
                    APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
                    "': using default [" + this.applicationEventMulticaster + "]");
        }
    }
}
```

### onRefresh

可以重写模板方法来添加特定 context 的刷新工作。默认情况下什么都不做。

### registerListeners

获取侦听器 `bean` 并注册。无需初始化即可添加

```java
protected void registerListeners() {
    // 注册静态指定的 ApplicationListener，和 beanFactoryPostProcessor 类似，context 可以提前添加好。
    for (ApplicationListener<?> listener : getApplicationListeners()) {
        getApplicationEventMulticaster().addApplicationListener(listener);
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let post-processors apply to them!
    // 这段注释看到不止一次，但是不太理解，感觉和代码联系不起来？
    String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
    for (String listenerBeanName : listenerBeanNames) {
        getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
    }

    // 现在我们终于有了多播器，发布早期的应用事件。
    Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
    this.earlyApplicationEvents = null;
    if (earlyEventsToProcess != null) {
        for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
            getApplicationEventMulticaster().multicastEvent(earlyEvent);
        }
    }
}
```

添加 ApplicationListener。

> 后处理器 ApplicationListenerDetector 在 processor chain 的最后，最终会将创建的代理添加为监听器。什么情况下会出现代码中预防的情况呢？

```java
public void addApplicationListener(ApplicationListener<?> listener) {
    synchronized (this.retrievalMutex) {
        // 如果已经注册，需要显式地删除代理，以避免同一监听器的双重调用。
        Object singletonTarget = AopProxyUtils.getSingletonTarget(listener);
        if (singletonTarget instanceof ApplicationListener) {
            this.defaultRetriever.applicationListeners.remove(singletonTarget);
        }
        this.defaultRetriever.applicationListeners.add(listener);
        this.retrieverCache.clear();
    }
}
```

### finishBeanFactoryInitialization

实例化所有剩余的（非惰性初始化）单例。以 context 视角，是完成内部 beanFactory 的初始化。

几乎可以只关注最后的 `beanFactory.preInstantiateSingletons()`。

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // 为 context 初始化转换服务
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // 如果之前没有任何 bean 后处理器（例如 PropertyPlaceholderConfigurer）注册，则注册默认的嵌入值解析器，主要用于解析注释属性值。
    // 接口 ConfigurableEnvironment 继承自 ConfigurablePropertyResolver
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
            @Override
            public String resolveStringValue(String strVal) {
                return getEnvironment().resolvePlaceholders(strVal);
            }
        });
    }

    // 尽早初始化 LoadTimeWeaverAware，以便尽早注册其转换器。
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // 停止使用临时类加载器进行类型匹配。
    beanFactory.setTempClassLoader(null);

    // 允许缓存所有 bean 定义的元数据，而不期望进一步更改。
    beanFactory.freezeConfiguration();

    // 实例化所有剩余的（非惰性初始化）单例。
    beanFactory.preInstantiateSingletons();
}
```

确保所有非惰性初始化单例都已实例化，同时还要考虑 `FactoryBeans`。 如果需要，通常在工厂设置结束时调用。

{% post_link how-does-Spring-load-beans '加载 Bean 的流程分析在此' %}。

> 先对集合进行 Copy 再迭代是很常见的处理方式，可以有效保证迭代时不受原集合影响，也不会影响到原集合。

```java
@Override
public void preInstantiateSingletons() throws BeansException {
    if (this.logger.isDebugEnabled()) {
        this.logger.debug("Pre-instantiating singletons in " + this);
    }

    // 拷贝一份 beanDefinitionNames
    List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

    // 触发所有非惰性初始化单例的实例化
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                // FactoryBean
                final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
                boolean isEagerInit;
                if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                    isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
                        @Override
                        public Boolean run() {
                            return ((SmartFactoryBean<?>) factory).isEagerInit();
                        }
                    }, getAccessControlContext());
                }
                else {
                    isEagerInit = (factory instanceof SmartFactoryBean &&
                            ((SmartFactoryBean<?>) factory).isEagerInit());
                }
                if (isEagerInit) {
                    // 是否立即初始化
                    getBean(beanName);
                }
            }
            else {
                // 常规 Bean（重要）
                getBean(beanName);
            }
        }
    }

    // 触发所有适用 bean 的初始化后回调
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged(new PrivilegedAction<Object>() {
                    @Override
                    public Object run() {
                        smartSingleton.afterSingletonsInstantiated();
                        return null;
                    }
                }, getAccessControlContext());
            }
            else {
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}
```

### finishRefresh

最后一步，完成 context 刷新，比如发布相应的事件。

```java
protected void finishRefresh() {
    // 1. 为此 context 初始化生命周期处理器。
    initLifecycleProcessor();
    // 2. 将刷新传播到生命周期处理器。
    getLifecycleProcessor().onRefresh();
    // 3. 发布 ContextRefreshedEvent。
    publishEvent(new ContextRefreshedEvent(this));
    // 4. 参与 LiveBeansView MBean（如果处于活动状态）。
    LiveBeansView.registerApplicationContext(this);
}
```