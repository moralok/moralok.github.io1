---
title: Spring @Configuration 注解的源码分析
date: 2023-11-24 05:43:52
tags:
    - java
    - spring
---

注解 @Configuration 是 Spring 中常用的注解，在一般应用场景中，它用于标识一个类作为配置类，搭配注解 @Bean 将创建的 bean 交给 Spring 容器管理。神奇的是，被 @Bean 注解的方法，只会被真正调用一次。这种方法调用被拦截的情况很容易让人联想到代理，如果你在 Debug 时注意过配置类的实例，你会发现配置类的 Class 名称中携带 EnhancerBySpringCGLIB。

本文将从源码角度，分析 @Configuration 注解是如何工作的。通过本文，你将收获：

1. 基于注解的 Spring 应用上下文是如何加载配置类的？
2. 配置类是如何被解析成为 bean 定义并注册到 beanFactory 中的？
3. 被 @Bean 注解的方法（在一般情况下）为何只会被调用一次？

```java
@Configuration
public class BeanConfig {

    @Bean
    public Person lisi() {
        return new Person("lisi", 20);
    }

    @Bean(value = "customName")
    public Person person() {
        // 测试 lisi() 在配置类拥有注解 @Configuration 时只会真正执行一次
        lisi();
        return new Person("wangwu", 30);
    }
}
```

```java
@Test
public void annotationConfigTest() {
    ApplicationContext ac = new AnnotationConfigApplicationContext(BeanConfig.class);
    BeanConfig beanConfig = (BeanConfig) ac.getBean("beanConfig");
    // 即使通过 beanConfig 调用，也不会执行第二次
    Person lisi = beanConfig.lisi();
}
```

## 解析配置类

### 什么是配置类？

通常情况下，我们称被 @Configuration 注解的类为配置类。事实上，配置类的范围比这个定义稍微广一些，在解析配置类时，我们再进一步说明。

```java
ApplicationContext ac = new AnnotationConfigApplicationContext(BeanConfig.class);

public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
    this();
    // 注册类，几乎可以说无条件地注册 annotatedClasses 的 bean 定义
    register(annotatedClasses);
    refresh();
}
```

当 BeanConfig 被传递给 AnnotationConfigApplicationContext，自身会先被解析为 BeanDefinition 注册到 beanFactory 中。有两点需要注意：

1. annotatedClasses 可以传入多个，意味着一开始**静态**指定的配置类可以有多个
2. annotatedClasses 除了在命名上提示用户应传入被注解的类外，`register(annotatedClasses)` 将它们作为普通的 Bean 注册到 beanFactory 中。它们是从外界传入的**首批** BeanDefinition。

之后 Spring 进入 refresh 流程。注意此时的 beanDefinitionMap，除了 beanConfig 外，AnnotationConfigApplicationContext 在创建时，已经自动注册了 6 个 bean 定义，其中一个就是我们今天的主角 `org.springframework.context.annotation.internalConfigurationAnnotationProcessor -> org.springframework.context.annotation.ConfigurationClassPostProcessor`。

{% asset_img "Snipaste_2023-11-24_15-26-08.png" AnnotationConfigApplicationContext 刷新前的 bean 定义 %}

### ConfigurationClassPostProcessor 介绍

ConfigurationClassPostProcessor 实现了接口 BeanDefinitionRegistryPostProcessor，也因此同时实现了接口 BeanFactoryPostProcessor。在{% post_link Spring-application-context-refresh-process 'Spring 应用 context 刷新流程' %}中，我们介绍过这两个接口，它们作为工厂后处理器，被用于 refresh 的 `invokeBeanFactoryPostProcessors(beanFactory)` 阶段。工厂后处理器的作用，一言以蔽之，允许自定义修改应用上下文的 bean 定义。

ConfigurationClassPostProcessor 的具体作用可以概括为两点：

1. 解析配置类中配置的 Bean，将它们的 bean 定义注册到 Bean Factory 中
2. （如有必要）增强配置类为代理配置类

### 核心 processConfigBeanDefinitions

进入 `invokeBeanFactoryPostProcessors(beanFactory)`，ConfigurationClassPostProcessor 先作为 BeanDefinitionRegistryPostProcessor 被调用。

```java
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    // ...
    // 处理配置的 BeanDefinition
    processConfigBeanDefinitions(registry);
}
```

核心方法 `processConfigBeanDefinitions(registry)` 冗长，不应过度关注细节（但确实反复读加深了理解）。它基于配置类的 BeanDefinition Registry（也就是 BeanFactory），构建和校验配置模型。作用可以简单地概括：

1. 从 BeanDefinition Registry（即 Bean Factory）中查找配置类
2. 解析配置类得到配置模型，从模型中读取 BeanDefinitions 注册到 BeanDefinition Registry（回到 1 重新循环直到不再引入新的配置类）

举例来说，静态添加的配置类只有 BeanConfig，假如 BeanConfig 不仅被 @Configuration 注解，还被 @ComponentScan 注解，因为后者 Spring 又扫描到并添加了新的一个配置类，这个新的配置类就需要继续被解析。

> 应正视配置模型这个概念。最初我先入为主，带着`解析得到 BeanDefinitions` 的观念，非常不理解 `processConfigBeanDefinitions` 方法上 `Build and validate a configuration model based on the registry of Configuration classes` 这句注释。注意：处理得到 bean 定义分为两个阶段，解析配置类得到配置模型，从配置模型中读取 bean 定义。

```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    List<BeanDefinitionHolder> configCandidates = new ArrayList<BeanDefinitionHolder>();
    // 刚开始，获取全部的 BeanDefinitions 作为候选
    String[] candidateNames = registry.getBeanDefinitionNames();

    for (String beanName : candidateNames) {
        BeanDefinition beanDef = registry.getBeanDefinition(beanName);
        if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
                ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
            // 根据 beanDef 的 attributes 判断是 Full 还是 Lite 的配置类
            // 已经处理过的配置类，会在 attributes 中添加标识
            if (logger.isDebugEnabled()) {
                logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
            }
        }
        else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
            // 未处理过的候选者，检查是否是配置类。configCandidates 的命名让人有些困惑，感觉这里指代的就是配置类
            configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
        }
    }

    // Return immediately if no @Configuration classes were found
    // 和注释略有不符，配置类的判定没有这么单一，不仅限于 @Configuration 注解的 Full 配置类
    if (configCandidates.isEmpty()) {
        return;
    }

    // 根据 attributes 中的 order 排序（来源于 @Order，可能不存在）
    Collections.sort(configCandidates, new Comparator<BeanDefinitionHolder>() {
        @Override
        public int compare(BeanDefinitionHolder bd1, BeanDefinitionHolder bd2) {
            int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
            int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
            return (i1 < i2) ? -1 : (i1 > i2) ? 1 : 0;
        }
    });

    // 检测应用上下文中是否配置了自定义 bean 名称生成策略
    SingletonBeanRegistry sbr = null;
    if (registry instanceof SingletonBeanRegistry) {
        sbr = (SingletonBeanRegistry) registry;
        if (!this.localBeanNameGeneratorSet && sbr.containsSingleton(CONFIGURATION_BEAN_NAME_GENERATOR)) {
            // 如果 localBeanNameGenerator 未设置，且 registry 中有配置，就获取并使用
            BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
            this.componentScanBeanNameGenerator = generator;
            this.importBeanNameGenerator = generator;
        }
    }

    // 创建 ConfigurationClassParser，解析每一个配置类（不限于 Full 类型）
    ConfigurationClassParser parser = new ConfigurationClassParser(
            this.metadataReaderFactory, this.problemReporter, this.environment,
            this.resourceLoader, this.componentScanBeanNameGenerator, registry);


    Set<BeanDefinitionHolder> candidates = new LinkedHashSet<BeanDefinitionHolder>(configCandidates);
    Set<ConfigurationClass> alreadyParsed = new HashSet<ConfigurationClass>(configCandidates.size());
    do {
        // 解析
        parser.parse(candidates);
        parser.validate();

        // 从 parser 中获取 ConfigurationClass，这就是配置模型（忍不住吐槽一下这个命名和配置类好容易搞混）
        Set<ConfigurationClass> configClasses = new LinkedHashSet<ConfigurationClass>(parser.getConfigurationClasses());
        // 排除已经解析过的 ConfigurationClass（candidates 是不重复的，为什么会得到的 ConfigurationClass 会重复啊）
        configClasses.removeAll(alreadyParsed);

        // Read the model and create bean definitions based on its content
        // 读取模型并根据它的内容创建 BeanDefinitions
        if (this.reader == null) {
            this.reader = new ConfigurationClassBeanDefinitionReader(
                    registry, this.sourceExtractor, this.resourceLoader, this.environment,
                    this.importBeanNameGenerator, parser.getImportRegistry());
        }
        // 加载 BeanDefinitions
        this.reader.loadBeanDefinitions(configClasses);
        // 添加到已经解析的集合中。上面的 configClasses.removeAll(alreadyParsed) 造成的干扰好大
        alreadyParsed.addAll(configClasses);

        // 清空这一轮处理的配置类集合
        candidates.clear();
        // 通过 registry 中的 Beandefinition 数量判断是否有新增的 BeanDefinition
        if (registry.getBeanDefinitionCount() > candidateNames.length) {
            // 重新获取全部 BeanDefinition 作为候选
            String[] newCandidateNames = registry.getBeanDefinitionNames();
            // 保留旧的候选集合用于筛选
            Set<String> oldCandidateNames = new HashSet<String>(Arrays.asList(candidateNames));
            // 为什么 alreadyParsedClasses 不定义在循环外，需要每次动态地从 alreadyParsed 获取？难道 configurationClass.getMetadata().getClassName() 的结果会变化吗？
            Set<String> alreadyParsedClasses = new HashSet<String>();
            for (ConfigurationClass configurationClass : alreadyParsed) {
                alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
            }
            for (String candidateName : newCandidateNames) {
                // 快速地排除旧候选
                if (!oldCandidateNames.contains(candidateName)) {
                    BeanDefinition bd = registry.getBeanDefinition(candidateName);
                    // 检查是否是配置类并且未被解析过
                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
                            !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                        // 添加到新的配置类集合
                        candidates.add(new BeanDefinitionHolder(bd, candidateName));
                    }
                }
            }
            candidateNames = newCandidateNames;
        }
    }
    while (!candidates.isEmpty());

    // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
    if (sbr != null) {
        if (!sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
            sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
        }
    }

    if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
        ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
    }
}
```

### 判断是否是配置类

1. 不能 className 为 null 或由工厂方法创建
2. 被 @Configuration 注解（全配置类）
3. 被 @Component、@ComponentScan、@Import、@ImportResource 注解或者拥有被 @Bean 注解的方法
4. 设置 order 用于排序

```java
public static boolean checkConfigurationClassCandidate(BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {
    String className = beanDef.getBeanClassName();
    if (className == null || beanDef.getFactoryMethodName() != null) {
        // 如果 className 为 null，或者拥有工厂方法，返回 false
        // 举例，被 @Bean 注解的方法返回的 Bean，它的类即使被 @Configuration 注解，因为有工厂方法，就不属于配置类
        return false;
    }

    // 获取 AnnotationMetadata
    // 这些分支对应的场景？
    AnnotationMetadata metadata;
    if (beanDef instanceof AnnotatedBeanDefinition &&
            className.equals(((AnnotatedBeanDefinition) beanDef).getMetadata().getClassName())) {
        // Can reuse the pre-parsed metadata from the given BeanDefinition...
        metadata = ((AnnotatedBeanDefinition) beanDef).getMetadata();
    }
    else if (beanDef instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) beanDef).hasBeanClass()) {
        // Check already loaded Class if present...
        // since we possibly can't even load the class file for this Class.
        Class<?> beanClass = ((AbstractBeanDefinition) beanDef).getBeanClass();
        metadata = new StandardAnnotationMetadata(beanClass, true);
    }
    else {
        try {
            MetadataReader metadataReader = metadataReaderFactory.getMetadataReader(className);
            metadata = metadataReader.getAnnotationMetadata();
        }
        catch (IOException ex) {
            if (logger.isDebugEnabled()) {
                logger.debug("Could not find class file for introspecting configuration annotations: " + className, ex);
            }
            return false;
        }
    }

    if (isFullConfigurationCandidate(metadata)) {
        // 属于 Full 配置类，添加 Attribute 标记，标识已处理过
        beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
    }
    else if (isLiteConfigurationCandidate(metadata)) {
        // 属于 Lite 配置类，添加 Attribute 标记，标识已处理过
        beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
    }
    else {
        return false;
    }

    // 不管是 Full 还是 Lite 配置类，检查是否被 @Order 注解，如果有则设置到 Attribute，用于排序
    Map<String, Object> orderAttributes = metadata.getAnnotationAttributes(Order.class.getName());
    if (orderAttributes != null) {
        beanDef.setAttribute(ORDER_ATTRIBUTE, orderAttributes.get(AnnotationUtils.VALUE));
    }

    return true;
}
```

#### 判断是否属于 Full 配置类

1. 被 @Configuration 注解

```java
public static boolean isFullConfigurationCandidate(AnnotationMetadata metadata) {
    return metadata.isAnnotated(Configuration.class.getName());
}
```

#### 判断是否属于 Lite 配置类

1. 被 @Component、@ComponentScan、@Import、@ImportResource 注解
2. 拥有被 @Bean 注解的方法

```java
public static boolean isLiteConfigurationCandidate(AnnotationMetadata metadata) {
    // 接口和注解，直接返回 false
    if (metadata.isInterface()) {
        return false;
    }

    // 如果被 @Component、@ComponentScan、@Import、@ImportResource 注解
    for (String indicator : candidateIndicators) {
        if (metadata.isAnnotated(indicator)) {
            return true;
        }
    }

    // 拥有被 @Bean 注解的方法
    try {
        return metadata.hasAnnotatedMethods(Bean.class.getName());
    }
    catch (Throwable ex) {
        if (logger.isDebugEnabled()) {
            logger.debug("Failed to introspect @Bean methods on class [" + metadata.getClassName() + "]: " + ex);
        }
        return false;
    }
}
```

### 循环处理直至没有新增配置类

循环处理部分的代码有点难阅读。

- 首先是因为命名比较相似，需要理清各个变量的含义和作用
    - candidateNames 就是普普通通的纯候选者（全部 BeanDefinitions），每次循环在解析完，加载 BeanDefinitions 后可能会新增
    - configCandidates（candidates） 就是符合配置类条件的配置类。虽然命名带 Candidates，其实已经是正牌，并非候选。感觉 configClass 更容易理解，但该变量名另作他用。有新增就要继续循环
    - configClass（ConfigurationClass 类） 是经过解析的配置模型，不要和配置类搞混了。
- 其次是因为对黑盒 parser 的作用不了解，个人经验如果完全将 parser 当作黑盒对待，不了解解析过程、解析的返回结果以及如何处理返回结果，理解循环解析的过程时会有点困难
    - `parser.parse(candidates)` 解析配置类构建得到配置模型（ConfigurationClass）
    - `this.reader.loadBeanDefinitions(configClasses)` 从配置模型（ConfigurationClass）中加载 BeanDefinitions

> 再次提示：**正视配置模型的概念，ConfigurationClassParser 使用配置类构建配置模型并校验；ConfigurationClassBeanDefinitionReader 从配置模型中读取 bean 定义。**

### 解析配置类构建配置模型

`parser.parse(candidates)` 正式进入解析过程，ConfigurationClassParser 负责将配置类转换为配置模型（ConfigurationClass）。

```java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
    this.deferredImportSelectors = new LinkedList<DeferredImportSelectorHolder>();

    // 循环依次处理配置类
    for (BeanDefinitionHolder holder : configCandidates) {
        BeanDefinition bd = holder.getBeanDefinition();
        try {
            if (bd instanceof AnnotatedBeanDefinition) {
                // 拥有注解的 BeanDefinition
                parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
            }
            else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
            }
            else {
                parse(bd.getBeanClassName(), holder.getBeanName());
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
        }
    }

    processDeferredImportSelectors();
}
```

```java
protected final void parse(AnnotationMetadata metadata, String beanName) throws IOException {
    // 每次都 new 一个 ConfigurationClass
    processConfigurationClass(new ConfigurationClass(metadata, beanName));
}
```

最终处理配置模型的方法 processConfigurationClass。

1. 检查配置模型是否曾经处理过
2. 处理配置模型

SourceClass 是一个简单的包装器，允许以统一的方式处理带注解的源类，无论它们是如何被加载的。

```java
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
    if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
        return;
    }

    // ConfigurationClass 重写了 equals 和 hashCode 方法
    ConfigurationClass existingClass = this.configurationClasses.get(configClass);
    // 判断是否已经存在
    if (existingClass != null) {
        // 检查是否是导入的
        if (configClass.isImported()) {
            if (existingClass.isImported()) {
                // 如果新的旧的 isImported 均为 true，合并 ImportedBy 集合
                existingClass.mergeImportedBy(configClass);
            }
            // Otherwise ignore new imported config class; existing non-imported class overrides it.
            return;
        }
        else {
            // 可能找到了一个显式定义的配置类，用于代替导入的，直接移除旧的
            this.configurationClasses.remove(configClass);
            for (Iterator<ConfigurationClass> it = this.knownSuperclasses.values().iterator(); it.hasNext();) {
                if (configClass.equals(it.next())) {
                    it.remove();
                }
            }
        }
    }

    // Recursively process the configuration class and its superclass hierarchy.
    // 递归处理配置类和它的父类
    SourceClass sourceClass = asSourceClass(configClass);
    do {
        sourceClass = doProcessConfigurationClass(configClass, sourceClass);
    }
    while (sourceClass != null);

    // 保存处理过的配置模型
    this.configurationClasses.put(configClass, configClass);
}
```

注意 ConfigurationClass 重写了 equals 和 hashCode 方法。

```java
public boolean equals(Object other) {
    return (this == other || (other instanceof ConfigurationClass &&
            getMetadata().getClassName().equals(((ConfigurationClass) other).getMetadata().getClassName())));
}

public int hashCode() {
    return getMetadata().getClassName().hashCode();
}
```

真正处理配置模型的方法，根据注释很容易知道，如果配置类携带了 @PropertySource、@ComponentScan、@Import、@ImportResource、@Bean 等注解，就是在这里被处理的，比如示例中被 @Bean 注解的方法。

```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {

    // Recursively process any member (nested) classes first
    processMemberClasses(configClass, sourceClass);

    // Process any @PropertySource annotations
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), PropertySources.class,
            org.springframework.context.annotation.PropertySource.class)) {
        if (this.environment instanceof ConfigurableEnvironment) {
            processPropertySource(propertySource);
        }
        else {
            logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                    "]. Reason: Environment must implement ConfigurableEnvironment");
        }
    }

    // Process any @ComponentScan annotations
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
    if (!componentScans.isEmpty() &&
            !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(
                        holder.getBeanDefinition(), this.metadataReaderFactory)) {
                    parse(holder.getBeanDefinition().getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }

    // Process any @Import annotations
    processImports(configClass, sourceClass, getImports(sourceClass), true);

    // Process any @ImportResource annotations
    if (sourceClass.getMetadata().isAnnotated(ImportResource.class.getName())) {
        AnnotationAttributes importResource =
                AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
        String[] resources = importResource.getStringArray("locations");
        Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
        for (String resource : resources) {
            String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
            configClass.addImportedResource(resolvedResource, readerClass);
        }
    }

    // Process individual @Bean methods
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }

    // Process default methods on interfaces
    processInterfaces(configClass, sourceClass);

    // 如果有父类，且之前没有处理过，返回父类继续处理
    if (sourceClass.getMetadata().hasSuperClass()) {
        String superclass = sourceClass.getMetadata().getSuperClassName();
        if (!superclass.startsWith("java") && !this.knownSuperclasses.containsKey(superclass)) {
            this.knownSuperclasses.put(superclass, configClass);
            // Superclass found, return its annotation metadata and recurse
            return sourceClass.getSuperClass();
        }
    }

    // No superclass -> processing is complete
    return null;
}
```

### 读取配置模型，注册 Bean 定义

`this.reader.loadBeanDefinitions(configClasses)` 读取配置模型的内容，注册 BeanDefinitions 到 Registry（也就是 BeanFactory）。

```java
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
    TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
    // 叫 ConfigurationModel 多好啊，ConfigurationClass 真让人容易误解
    for (ConfigurationClass configClass : configurationModel) {
        loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
    }
}
```

处理单独的一个配置模型。

```java
private void loadBeanDefinitionsForConfigurationClass(ConfigurationClass configClass,
        TrackedConditionEvaluator trackedConditionEvaluator) {

    if (trackedConditionEvaluator.shouldSkip(configClass)) {
        String beanName = configClass.getBeanName();
        if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
            this.registry.removeBeanDefinition(beanName);
        }
        this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
        return;
    }

    if (configClass.isImported()) {
        registerBeanDefinitionForImportedConfigurationClass(configClass);
    }
    for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        loadBeanDefinitionsForBeanMethod(beanMethod);
    }
    loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
    loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```

