---
title: Spring AutowiredAnnotationBeanPostProcessor 的源码分析
date: 2023-12-08 12:00:30
tags: [java, spring]
---

在 `Spring` 中，`AutowiredAnnotationBeanPostProcessor` 是一个非常重要的后处理器，它可以自动装配标注注解的字段和方法，默认使用 `@Autowired` 和 `@Value` 注解，可以支持 `JSR-330` 的 `@Inject` 注解。本文通过分析源码介绍它的调用时机和工作原理。

<!-- more -->

### 介绍

`AutowiredAnnotationBeanPostProcessor` 顾名思义，是自动装配注解的 `BeanPostProcessor`，但是它处理的不仅仅是 `@Autowired` 这一个注解。个人认为 `Autowired Annotation` 的意思更接近“用于标注**目标**是**被自动装配**的**注解**”。使用“目标”是为了表达注解标注的目标不仅仅限于字段，更是包括构造函数、方法、方法参数以及注解；使用“被自动装配”是为了表达注解描述的是目标的特征或者被处理的结果，体现出被动的语义更准确；使用“注解”是为了表达注解的种类不仅仅限于 `@Autowired`，还包括 `@Value` 和 `@Inject`，它们都指示目标需要被自动装配处理。

通过 `AutowiredAnnotationBeanPostProcessor` 的构造函数可以看到 `@Inject` 注解的特别之处，为了使用它，需要在 `Maven` 配置中额外引入 `javax.inject` 依赖。

```java
public AutowiredAnnotationBeanPostProcessor() {
    this.autowiredAnnotationTypes.add(Autowired.class);
    this.autowiredAnnotationTypes.add(Value.class);
    try {
        this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
                ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
        logger.info("JSR-330 'javax.inject.Inject' annotation found and supported for autowiring");
    }
    catch (ClassNotFoundException ex) {
        // JSR-330 API not available - simply skip.
    }
}
```

### 入口：`populateBean` 方法

我们在{% post_link how-does-Spring-load-beans 'Spring Bean 加载过程' %}中介绍过**为 `bean` 填充属性值**发生在 `populateBean` 方法中。我们也将直接从这里开始跟踪代码的处理过程。

> 个人认为宽松地讲，“填充属性”等于“注入属性”等于“自动装配”，前两者更侧重处理的结果，后者更侧重过程的特征，但请注意在具体的代码上下文中应辨析区别。例如为 `bean` 填充属性是 `Spring` 的重要目标之一，基于 `Autowired Annotation` 进行自动装配某一个后处理器的功能，是 `Spring` 实现目标的其中一个具体方式。

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    // ...
    // 如果存在 InstantiationAwareBeanPostProcessor 或者需要检查依赖
    if (hasInstAwareBpps || needsDepCheck) {
        PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        // 如果存在 InstantiationAwareBeanPostProcessor 
        if (hasInstAwareBpps) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    // AutowiredAnnotationBeanPostProcessor 实现了 InstantiationAwareBeanPostProcessor 接口
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    // 调用 postProcessPropertyValues 方法
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
    // ...
}
```

> 有时候在 `Spring` 中看到 `BeanPostProcessor` 并不能代表将目光转向该接口的方法实现。不同 `BeanPostProcessor` 的子接口存在不同的调用时机。`AutowiredAnnotationBeanPostProcessor` 间接实现了 `InstantiationAwareBeanPostProcessor` 并直接实现了 `MergedBeanDefinitionPostProcessor`，这是我们今天要关注的两个重点接口。

**`AutowiredAnnotationBeanPostProcessor` 是什么时候注册的呢？**

以 `AnnotationConfigApplicationContext` 为例，它在构造函数中创建了 `AnnotatedBeanDefinitionReader`，`AnnotatedBeanDefinitionReader` 又在构造函数中注册了基于注解配置的处理器：

```java
AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
```

其中就包括 `AutowiredAnnotationBeanPostProcessor`。

### 后处理 PropertyValues

`AutowiredAnnotationBeanPostProcessor` 实现了 `InstantiationAwareBeanPostProcessor` 接口，该接口关注 `bean` 的实例化：

- `postProcessBeforeInstantiation`（实例化前）
- `postProcessAfterInstantiation`（实例化后）
- `postProcessPropertyValues`（实例化后）

> `postProcessPropertyValues` 方法在工厂将给定属性值应用到给定 `bean` 之前对给定属性值进行后处理。允许检查是否满足所有依赖关系，例如基于 `bean` 属性 `setters` 上的 `@Required` 注解进行检查。还允许替换要应用的属性值，通常是通过基于原始 `PropertyValues` 创建新的 `MutablePropertyValues` 实例，并添加或删除特定值。

`postProcessPropertyValues` 方法做了两件事情：

- 查找需要自动装配的元数据
- 注入

```java
@Override
public PropertyValues postProcessPropertyValues(
        PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {
    // 查找自动装配元数据
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
    try {
        // 注入
        metadata.inject(bean, beanName, pvs);
    }
    catch (BeanCreationException ex) {
        throw ex;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
    }
    return pvs;
}
```

### 查找自动装配元数据

> 这部分代码体现了注入（Injection）和自动装配（Autowiring）的等价性。`InjectionMetadata` 和 `AutowiringMetadata` 的含义是用于注入（自动装配）的元数据。

`InjectionMetadata` 是用于管理注入元数据的内部类，不适合直接在应用程序中使用。它和 `Class` 是一对一的关系，封装了需要**被注入**的元素 `InjectedElement`。一个 `InjectedElement` 对应着一个字段（`Field`）或一个方法（`Method`），分别对应着两个实现类 `AutowiredFieldElement` 和 `AutowiredMethodElement`。这里再次体现了**被注入、被自动装配**的语义。

查找自动装配元数据的过程如下：

- 先从缓存中获取，如果存在且不需要刷新，则直接返回结果
- 否则构建自动装配元数据并放入缓存

> 注意：在 `postProcessPropertyValues` 第一次调用 `findAutowiringMetadata` 缓存中就已经有结果了。什么时候构建并存入缓存的呢？

```java
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, PropertyValues pvs) {
    // 缓存 key，如果没有指定退化为使用全限定类名
    String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
    // 双重检查
    // 先从缓存中获取
    InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
    // 检测是否需要刷新
    if (InjectionMetadata.needsRefresh(metadata, clazz)) {
        synchronized (this.injectionMetadataCache) {
            metadata = this.injectionMetadataCache.get(cacheKey);
            if (InjectionMetadata.needsRefresh(metadata, clazz)) {
                if (metadata != null) {
                    metadata.clear(pvs);
                }
                try {
                    // 构建自动装配元数据
                    metadata = buildAutowiringMetadata(clazz);
                    // 放入缓存
                    this.injectionMetadataCache.put(cacheKey, metadata);
                }
                catch (NoClassDefFoundError err) {
                    throw new IllegalStateException("Failed to introspect bean class [" + clazz.getName() +
                            "] for autowiring metadata: could not find class that it depends on", err);
                }
            }
        }
    }
    return metadata;
}

public static boolean needsRefresh(InjectionMetadata metadata, Class<?> clazz) {
    // metadata.targetClass != clazz 的场景是什么？
    return (metadata == null || metadata.targetClass != clazz);
}
```

#### 构建自动装配元数据

构建自动装配元数据只需要给定一个 `Class`，沿着给定的 `Class` 的父类向上循环查找直到 `Object` 类。在每个循环中，先遍历当前类声明的所有属性，找到标注了自动装配注解的属性，为其创建 `AutowiredFieldElement` 并添加到临时集合，再遍历当前类声明的所有方法，找到标注了自动装配注解的方法，为其创建 `AutowiredMethodElement` 并添加到临时集合。最后汇总 `InjectedElement` 封装到 `InjectionMetadata` 中。

```java
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
    LinkedList<InjectionMetadata.InjectedElement> elements = new LinkedList<InjectionMetadata.InjectedElement>();
    Class<?> targetClass = clazz;

    do {
        final LinkedList<InjectionMetadata.InjectedElement> currElements =
                new LinkedList<InjectionMetadata.InjectedElement>();
        // 处理字段 Field -> AutowiredFieldElement
        ReflectionUtils.doWithLocalFields(targetClass, new ReflectionUtils.FieldCallback() {
            @Override
            public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
                // 查找表示需被自动装配的注解：@Autowired、@Value、@Inject（可选）
                AnnotationAttributes ann = findAutowiredAnnotation(field);
                if (ann != null) {
                    if (Modifier.isStatic(field.getModifiers())) {
                        // 不支持静态字段
                        if (logger.isWarnEnabled()) {
                            logger.warn("Autowired annotation is not supported on static fields: " + field);
                        }
                        return;
                    }
                    // 确定 required
                    boolean required = determineRequiredStatus(ann);
                    // 根据 field 和 required 创建 AutowiredFieldElement 并添加
                    currElements.add(new AutowiredFieldElement(field, required));
                }
            }
        });
        // 处理方法 Method -> AutowiredMethodElement
        ReflectionUtils.doWithLocalMethods(targetClass, new ReflectionUtils.MethodCallback() {
            @Override
            public void doWith(Method method) throws IllegalArgumentException, IllegalAccessException {
                Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
                if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
                    return;
                }
                // 查找表示需被自动装配的注解：@Autowired、@Value、@Inject（可选）
                AnnotationAttributes ann = findAutowiredAnnotation(bridgedMethod);
                if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
                    if (Modifier.isStatic(method.getModifiers())) {
                        // 不支持静态方法
                        if (logger.isWarnEnabled()) {
                            logger.warn("Autowired annotation is not supported on static methods: " + method);
                        }
                        return;
                    }
                    if (method.getParameterTypes().length == 0) {
                        // 不支持无参数的方法
                        if (logger.isWarnEnabled()) {
                            logger.warn("Autowired annotation should only be used on methods with parameters: " +
                                    method);
                        }
                    }
                    // 确定 required
                    boolean required = determineRequiredStatus(ann);
                    PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
                    // 创建 AutowiredMethodElement 并添加
                    currElements.add(new AutowiredMethodElement(method, required, pd));
                }
            }
        });

        elements.addAll(0, currElements);
        // 向父类继续查找
        targetClass = targetClass.getSuperclass();
    }
    while (targetClass != null && targetClass != Object.class);
    // 封装为 InjectionMetadata 返回
    return new InjectionMetadata(clazz, elements);
}
```

### 注入

对于字段来说，注入意味着将一个解析得到的 `value` 通过反射设置到字段中；对于方法来说，注入意味着解析得到方法参数的 `value`，然后通过反射调用方法。

`InjectionMetadata` 的 `inject` 方法比较简单，内部会遍历并调用 `InjectedElement` 的 `inject` 方法，`AutowiredFieldElement` 和 `AutowiredMethodElement` 各自实现了 `inject` 方法。

```java
public void inject(Object target, String beanName, PropertyValues pvs) throws Throwable {
    Collection<InjectedElement> elementsToIterate =
            (this.checkedElements != null ? this.checkedElements : this.injectedElements);
    if (!elementsToIterate.isEmpty()) {
        boolean debug = logger.isDebugEnabled();
        // 遍历（InjectedElement 包装的可能是字段，也可能是方法）
        for (InjectedElement element : elementsToIterate) {
            if (debug) {
                logger.debug("Processing injected element of bean '" + beanName + "': " + element);
            }
            // 注入
            element.inject(target, beanName, pvs);
        }
    }
}
```

不论是 `AutowiredFieldElement` 还是 `AutowiredMethodElement`，`inject` 的过程都比较相似：

- 都使用 `DependencyDescriptor` 描述即将被注入的特定依赖项，`DependencyDescriptor` 包装了构造函数参数、方法参数或者字段，允许以统一的方式访问它们的元数据
- 都会缓存 `DependencyDescriptor`
- 都会记录自动装配的 `bean`，用于判断是否使用 `DependencyDescriptor` 的变体 `ShortcutDependencyDescriptor` 优化缓存
- 都通过 `beanFactory.resolveDependency` 解析依赖

#### 字段注入

```java
private class AutowiredFieldElement extends InjectionMetadata.InjectedElement {
    // 是否必须
    private final boolean required;
    // 是否已缓存
    private volatile boolean cached = false;
    // Field 依赖描述符的缓存
    private volatile Object cachedFieldValue;

    public AutowiredFieldElement(Field field, boolean required) {
        super(field, null);
        this.required = required;
    }

    @Override
    protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {
        // 获取要注入的目标（Field 对象）
        Field field = (Field) this.member;
        // value
        Object value;
        if (this.cached) {
            // 如果已缓存，解析已缓存的参数
            value = resolvedCachedArgument(beanName, this.cachedFieldValue);
        }
        else {
            // 创建依赖描述符
            DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
            desc.setContainingClass(bean.getClass());
            Set<String> autowiredBeanNames = new LinkedHashSet<String>(1);
            TypeConverter typeConverter = beanFactory.getTypeConverter();
            try {
                // 通过 beanFactory 解析依赖得到 value
                value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
            }
            catch (BeansException ex) {
                throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(field), ex);
            }
            synchronized (this) {
                // 如果未缓存，则缓存
                if (!this.cached) {
                    if (value != null || this.required) {
                        // 缓存 DependencyDescriptor
                        this.cachedFieldValue = desc;
                        // 注册依赖关系，用于控制销毁顺序
                        registerDependentBeans(beanName, autowiredBeanNames);
                        // 如果自动装配的 bean 刚好只有一个
                        if (autowiredBeanNames.size() == 1) {
                            String autowiredBeanName = autowiredBeanNames.iterator().next();
                            // 检测工厂里存在 bean
                            if (beanFactory.containsBean(autowiredBeanName)) {
                                if (beanFactory.isTypeMatch(autowiredBeanName, field.getType())) {
                                    // 替换为具有预先解析的目标 bean 名称的 DependencyDescriptor 变体
                                    this.cachedFieldValue = new ShortcutDependencyDescriptor(
                                            desc, autowiredBeanName, field.getType());
                                }
                            }
                        }
                    }
                    else {
                        this.cachedFieldValue = null;
                    }
                    this.cached = true;
                }
            }
        }
        if (value != null) {
            // 最后，通过反射将 value 设置到 field
            ReflectionUtils.makeAccessible(field);
            field.set(bean, value);
        }
    }
}
```

#### 方法注入

> 不理解缓存 `DependencyDescriptor` 代码上的注释：Shortcut for avoiding synchronization...

```java
private class AutowiredMethodElement extends InjectionMetadata.InjectedElement {
    // 是否必须
    private final boolean required;
    // 是否已缓存
    private volatile boolean cached = false;
    // Method 参数依赖描述符的缓存
    private volatile Object[] cachedMethodArguments;

    public AutowiredMethodElement(Method method, boolean required, PropertyDescriptor pd) {
        super(method, pd);
        this.required = required;
    }

    @Override
    protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {
        if (checkPropertySkipping(pvs)) {
            return;
        }
        // 获取要注入的目标（Method）
        Method method = (Method) this.member;
        // 方法的参数
        Object[] arguments;
        if (this.cached) {
            // Shortcut for avoiding synchronization...
            // 不理解这个注释
            arguments = resolveCachedArguments(beanName);
        }
        else {
            // 获取方法的参数类型数组
            Class<?>[] paramTypes = method.getParameterTypes();
            arguments = new Object[paramTypes.length];
            DependencyDescriptor[] descriptors = new DependencyDescriptor[paramTypes.length];
            Set<String> autowiredBeans = new LinkedHashSet<String>(paramTypes.length);
            TypeConverter typeConverter = beanFactory.getTypeConverter();
            // 遍历
            for (int i = 0; i < arguments.length; i++) {
                MethodParameter methodParam = new MethodParameter(method, i);
                // 为每个方法参数创建依赖描述符
                DependencyDescriptor currDesc = new DependencyDescriptor(methodParam, this.required);
                currDesc.setContainingClass(bean.getClass());
                descriptors[i] = currDesc;
                try {
                    // 通过 beanFactory 解析依赖得到 value
                    Object arg = beanFactory.resolveDependency(currDesc, beanName, autowiredBeans, typeConverter);
                    if (arg == null && !this.required) {
                        arguments = null;
                        break;
                    }
                    // 赋值
                    arguments[i] = arg;
                }
                catch (BeansException ex) {
                    throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(methodParam), ex);
                }
            }
            synchronized (this) {
                // 如果未缓存，则缓存
                if (!this.cached) {
                    if (arguments != null) {
                        this.cachedMethodArguments = new Object[paramTypes.length];
                        // 缓存 DependencyDescriptor
                        for (int i = 0; i < arguments.length; i++) {
                            this.cachedMethodArguments[i] = descriptors[i];
                        }
                        // 注册依赖关系
                        registerDependentBeans(beanName, autowiredBeans);
                        // 如果自动装配的 bean 数量等于参数的数量
                        if (autowiredBeans.size() == paramTypes.length) {
                            Iterator<String> it = autowiredBeans.iterator();
                            // 遍历
                            for (int i = 0; i < paramTypes.length; i++) {
                                String autowiredBeanName = it.next();
                                // 检测工厂里存在 bean
                                if (beanFactory.containsBean(autowiredBeanName)) {
                                    if (beanFactory.isTypeMatch(autowiredBeanName, paramTypes[i])) {
                                        // 替换为具有预先解析的目标 bean 名称的 DependencyDescriptor 变体
                                        this.cachedMethodArguments[i] = new ShortcutDependencyDescriptor(
                                                descriptors[i], autowiredBeanName, paramTypes[i]);
                                    }
                                }
                            }
                        }
                    }
                    else {
                        this.cachedMethodArguments = null;
                    }
                    this.cached = true;
                }
            }
        }
        if (arguments != null) {
            // 通过反射调用方法
            try {
                ReflectionUtils.makeAccessible(method);
                method.invoke(bean, arguments);
            }
            catch (InvocationTargetException ex){
                throw ex.getTargetException();
            }
        }
    }

    private Object[] resolveCachedArguments(String beanName) {
        if (this.cachedMethodArguments == null) {
            return null;
        }
        Object[] arguments = new Object[this.cachedMethodArguments.length];
        // 遍历已缓存的方法参数
        for (int i = 0; i < arguments.length; i++) {
            // 解析已缓存的参数
            arguments[i] = resolvedCachedArgument(beanName, this.cachedMethodArguments[i]);
        }
        return arguments;
    }
}
```

#### 解析已缓存的方法参数或字段

> 为什么在这里 `beanFactory.resolveDependency` 需要的参数和未缓存时不一样啊？虽然内部会通过相同的方式获得 `typeConverter`，但是很奇怪啊。

```java
private Object resolvedCachedArgument(String beanName, Object cachedArgument) {
    if (cachedArgument instanceof DependencyDescriptor) {
        DependencyDescriptor descriptor = (DependencyDescriptor) cachedArgument;
        return this.beanFactory.resolveDependency(descriptor, beanName, null, null);
    }
    else {
        return cachedArgument;
    }
}
```

### 解析依赖

解析依赖的过程暂不深入。

```java
public Object resolveDependency(DependencyDescriptor descriptor, String requestingBeanName,
        Set<String> autowiredBeanNames, TypeConverter typeConverter) throws BeansException {

    descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
    if (javaUtilOptionalClass == descriptor.getDependencyType()) {
        return new OptionalDependencyFactory().createOptionalDependency(descriptor, requestingBeanName);
    }
    else if (ObjectFactory.class == descriptor.getDependencyType() ||
            ObjectProvider.class == descriptor.getDependencyType()) {
        return new DependencyObjectProvider(descriptor, requestingBeanName);
    }
    else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
        return new Jsr330ProviderFactory().createDependencyProvider(descriptor, requestingBeanName);
    }
    else {
        // 如果依赖是懒加载，创建一个代理对象
        Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
                descriptor, requestingBeanName);
        if (result == null) {
            // 一般情况
            result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
        }
        return result;
    }
}
```

```java
public Object doResolveDependency(DependencyDescriptor descriptor, String beanName,
        Set<String> autowiredBeanNames, TypeConverter typeConverter) throws BeansException {

    InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
    try {
        Object shortcut = descriptor.resolveShortcut(this);
        if (shortcut != null) {
            return shortcut;
        }

        Class<?> type = descriptor.getDependencyType();
        Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
        if (value != null) {
            // 如果 value 是 String 类型
            if (value instanceof String) {
                // 解析给定的嵌入值，例如替换占位符 ${}，但不解析 SpEL 表达式
                String strVal = resolveEmbeddedValue((String) value);
                BeanDefinition bd = (beanName != null && containsBean(beanName) ? getMergedBeanDefinition(beanName) : null);
                // 解析 SpEL 表达式
                value = evaluateBeanDefinitionString(strVal, bd);
            }
            TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
            return (descriptor.getField() != null ?
                    converter.convertIfNecessary(value, type, descriptor.getField()) :
                    converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
        }

        Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
        if (multipleBeans != null) {
            return multipleBeans;
        }

        Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
        if (matchingBeans.isEmpty()) {
            if (isRequired(descriptor)) {
                raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
            }
            return null;
        }

        String autowiredBeanName;
        Object instanceCandidate;

        if (matchingBeans.size() > 1) {
            autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
            if (autowiredBeanName == null) {
                if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
                    return descriptor.resolveNotUnique(type, matchingBeans);
                }
                else {
                    // In case of an optional Collection/Map, silently ignore a non-unique case:
                    // possibly it was meant to be an empty collection of multiple regular beans
                    // (before 4.3 in particular when we didn't even look for collection beans).
                    return null;
                }
            }
            instanceCandidate = matchingBeans.get(autowiredBeanName);
        }
        else {
            // We have exactly one match.
            Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
            autowiredBeanName = entry.getKey();
            instanceCandidate = entry.getValue();
        }

        if (autowiredBeanNames != null) {
            autowiredBeanNames.add(autowiredBeanName);
        }
        return (instanceCandidate instanceof Class ?
                descriptor.resolveCandidate(autowiredBeanName, type, this) : instanceCandidate);
    }
    finally {
        ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
    }
}
```

### 构建自动装配元数据的时机

你在 `Debug` 的时候也许会注意到，在第一次进入 `postProcessPropertyValues` 方法，查找自动装配元数据时，就已经是从缓存中获取的了。那么究竟是**什么时候构建自动装配元数据并放入缓存的**呢？这就需要我们目前一直没有讲到的 `MergedBeanDefinitionPostProcessor` 派上用场了。在 `postProcessMergedBeanDefinition` 方法中，也调用了 `findAutowiringMetadata` 方法，这才是真正的第一次查找自动装配元数据。

```java
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
    if (beanType != null) {
        // 查找自动装配元数据
        InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
        // 检查配置成员
        metadata.checkConfigMembers(beanDefinition);
    }
}
```

那么 **`MergedBeanDefinitionPostProcessor` 又是什么时候被调用的**呢？在 `doCreateBean` 方法中，创建实例后，填充属性前。

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
        throws BeanCreationException {
    // 创建实例
    // 允许 post-processors 修改合并过的 bean definition
    synchronized (mbd.postProcessingLock) {
        // 如果尚未被 MergedBeanDefinitionPostProcessor 应用过
        if (!mbd.postProcessed) {
            try {
                // 应用
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Post-processing of merged bean definition failed", ex);
            }
            // 修改为被应用过
            mbd.postProcessed = true;
        }
    }
    // 为 bean 填充属性值
}
```

#### 检测配置成员

> `configMember` 这个命名不太理解，指的是通过配置实现注入的 `Member` （`Filed` 和 `Method` 的父类）吗？

检测配置成员，如果不是外部管理的配置成员，则注册为外部管理的配置成员。在合并后的 `bean` 定义中，`externallyManagedConfigMembers` 保存了外部管理的配置成员，用于标记一个配置成员是外部管理的。例如当一个字段同时标注了 `@Resource` 和 `@Autowired` 注解，当 `@Resouce` 注解被处理后，该字段已经被标记，当 `@Autowired` 注解被处理时，就会跳过该字段，避免重复注入造成冲突。

> 这里的外部管理感觉有点指向依赖注入的控制反转思想。

```java
public void checkConfigMembers(RootBeanDefinition beanDefinition) {
    Set<InjectedElement> checkedElements = new LinkedHashSet<InjectedElement>(this.injectedElements.size());
    // 遍历需要被注入的元素
    for (InjectedElement element : this.injectedElements) {
        Member member = element.getMember();
        // 如果不是外部管理的配置成员
        if (!beanDefinition.isExternallyManagedConfigMember(member)) {
            // 注册为外部管理的配置成员
            beanDefinition.registerExternallyManagedConfigMember(member);
            checkedElements.add(element);
            if (logger.isDebugEnabled()) {
                logger.debug("Registered injected element on class [" + this.targetClass.getName() + "]: " + element);
            }
        }
    }
    // 在 `InjectionMetadata#inject` 方法中，迭代的集合将会是它
    this.checkedElements = checkedElements;
}
```

> 如果不了解具体的场景，可能会比较难想象这个标记的用处是什么。