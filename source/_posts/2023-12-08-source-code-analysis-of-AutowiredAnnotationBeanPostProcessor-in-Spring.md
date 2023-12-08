---
title: Spring AutowiredAnnotationBeanPostProcessor 的源码分析 
date: 2023-12-08 12:00:30
tags: [java, spring]
---

在 `Spring` 中，`AutowiredAnnotationBeanPostProcessor` 是一个非常重要的后处理器，它可以自动装配标注注解的字段和方法，默认使用 `@Autowired` 和 `@Value` 注解，可以支持 `JSR-330` 的 `@Inject` 注解。本文通过分析源码介绍它的工作原理。

<!-- more -->

### 入口：`populateBean` 方法

我们在{% post_link Spring-application-context-refresh-process 'Spring 应用 context 刷新流程' %}中介绍过属性注入发生在 `populateBean` 方法中。

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    // ...
    // 如果存在 InstantiationAwareBeanPostProcessor 或者需要检查依赖
    if (hasInstAwareBpps || needsDepCheck) {
        PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
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

### 后处理 PropertyValues

`AutowiredAnnotationBeanPostProcessor` 实现了 `InstantiationAwareBeanPostProcessor` 接口，在 `postProcessPropertyValues` 方法中，它做了两件事情：

- 查找需要自动装配的元数据
- 注入

> `postProcessPropertyValues` 方法在工厂将给定属性值应用到给定 `bean` 之前对给定属性值进行后处理。允许检查是否满足所有依赖关系，例如基于 `bean` 属性 `setters` 上的 `@Required` 注解进行检查。还允许替换要应用的属性值，通常是通过基于原始 `PropertyValues` 创建新的 `MutablePropertyValues` 实例，并添加或删除特定值。

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

`InjectionMetadata` 是用于管理注入元数据的内部类，不适合直接在应用程序中使用。它和 `Class` 是一对一的关系，封装了需要注入的元素 `InjectedElement`。一个 `InjectedElement` 对应着一个字段（`Field`）或一个方法（`Method`），分别对应着两个实现类 `AutowiredFieldElement` 和 `AutowiredMethodElement`。

查找自动装配元数据的过程如下：

- 先从缓存中获取，如果存在且不需要刷新，则直接返回结果
- 否则构建自动装配元数据并放入缓存

```java
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, PropertyValues pvs) {
    // 缓存 key
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
                // 查找代表自动装配的注解：@Autowired、@Value、@Inject（可选）
                AnnotationAttributes ann = findAutowiredAnnotation(field);
                if (ann != null) {
                    if (Modifier.isStatic(field.getModifiers())) {
                        if (logger.isWarnEnabled()) {
                            logger.warn("Autowired annotation is not supported on static fields: " + field);
                        }
                        return;
                    }
                    // 确定 required
                    boolean required = determineRequiredStatus(ann);
                    // 创建 AutowiredFieldElement 并添加
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
                // 查找代表自动装配的注解：@Autowired、@Value、@Inject（可选）
                AnnotationAttributes ann = findAutowiredAnnotation(bridgedMethod);
                if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
                    if (Modifier.isStatic(method.getModifiers())) {
                        if (logger.isWarnEnabled()) {
                            logger.warn("Autowired annotation is not supported on static methods: " + method);
                        }
                        return;
                    }
                    if (method.getParameterTypes().length == 0) {
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
    // 封装为 InjectionMetadata
    return new InjectionMetadata(clazz, elements);
}
```

### 注入

`InjectionMetadata` 的 `inject` 方法比较简单，内部会遍历并调用 `InjectedElement` 的 `inject` 方法，`AutowiredFieldElement` 和 `AutowiredMethodElement` 各自实现了 `inject` 方法。对于字段来说，注入意味着将一个解析得到的 `value` 通过反射设置到字段中；对于方法来说，注入意味着解析得到方法的参数的 `value`，然后通过反射调用方法。

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
- 都会记录自动装配的 `bean`，用于判断是否使用 `DependencyDescriptor` 的变体优化缓存
- 都通过 `beanFactory.resolveDependency` 解析依赖

> 不理解缓存 `DependencyDescriptor` 代码上的注释

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
                // beanFactory 解析依赖
                value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
            }
            catch (BeansException ex) {
                throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(field), ex);
            }
            synchronized (this) {
                // 如果未缓存，则缓存
                if (!this.cached) {
                    if (value != null || this.required) {
                        // 缓存 desc
                        this.cachedFieldValue = desc;
                        // 注册依赖关系，用于控制销毁顺序
                        registerDependentBeans(beanName, autowiredBeanNames);
                        // 如果自动装配的 bean 刚好只有一个
                        if (autowiredBeanNames.size() == 1) {
                            String autowiredBeanName = autowiredBeanNames.iterator().next();
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
            for (int i = 0; i < arguments.length; i++) {
                MethodParameter methodParam = new MethodParameter(method, i);
                // 创建依赖描述符
                DependencyDescriptor currDesc = new DependencyDescriptor(methodParam, this.required);
                currDesc.setContainingClass(bean.getClass());
                descriptors[i] = currDesc;
                try {
                    // beanFactory 解析依赖
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
                        for (int i = 0; i < arguments.length; i++) {
                            this.cachedMethodArguments[i] = descriptors[i];
                        }
                        // 注册依赖关系
                        registerDependentBeans(beanName, autowiredBeans);
                        // 如果自动装配的 bean 数量等于参数的数量
                        if (autowiredBeans.size() == paramTypes.length) {
                            Iterator<String> it = autowiredBeans.iterator();
                            for (int i = 0; i < paramTypes.length; i++) {
                                String autowiredBeanName = it.next();
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

> 为什么在这里 `beanFactory.resolveDependency` 需要的参数和未缓存时不一样啊？

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