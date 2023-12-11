---
title: ComponentScan 扫描路径覆盖的真相
date: 2023-12-11 10:11:22
tags: [java, spring, spring boot]
---

`@ComponentScan` 注解是 `Spring` 中很常用的注解，用于扫描并加载指定类路径下的 `Bean`，而 `Spring Boot` 为了便捷使用 `@SpringBootApplication` 组合注解集成了 `@ComponentScan` 的能力。也许你听说过使用后者会覆盖前者中关于包扫描的设置，但你是否质疑过这个“不合常理”的结论？是否好奇过为什么它们不像其他注解在嵌套使用时可以同时生效？又是否好奇过 `@SpringBootApplication` 可以间接设置 `@ComponentScan` 属性的原因？本文从源码角度分析 `@ComponentScan` 的工作原理，揭示它独特的检索算法和注解层次结构中的属性覆盖机制。

<!--more -->

- 本文的写作动机继承自{% post_link 'source-code-analysis-of-Spring-Configuration-annotation' Spring @Configuration 注解的源码分析 %}，处理 `@ComponentScan` 是处理 `@Configuration` 过程的一部分。

## 入口

对于标注了 `@ComponentScan` 注解的配置类，处理过程如下：

- 获取 `@ComponentScan` 的注解属性
- 遍历注解属性集合，依次根据其中的信息进行扫描，获取 `Bean` 定义
- 如果获取到的 `Bean` 定义中有任何其他配置类，将递归解析（处理配置类）

> 这里和处理 `@Import` 的过程很像，都出现了递归解析新获得的配置类。

```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {
    // ...
    // 处理任何 @ComponentScan 注解
    // 获取 @ComponentScan 的注解属性，该注解是可重复的
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
    if (!componentScans.isEmpty() &&
            !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
        // 遍历
        for (AnnotationAttributes componentScan : componentScans) {
            // 如果配置类被标注了 @ComponentScan -> 立即扫描
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // 检测被扫描到的 Bean 定义中是否有任何其他配置类，如有需要递归解析
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(
                        holder.getBeanDefinition(), this.metadataReaderFactory)) {
                    // 递归解析配置类
                    parse(holder.getBeanDefinition().getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }
    // ...
}
```

## 扫描获取 Bean 定义

我们先跳过“获取 `@ComponentScan` 的注解属性”的过程，来看“扫描获取 `Bean` 定义”的过程。扫描是通过 `ComponentScanAnnotationParser` 的 `parse` 方法完成的，这个方法很长，但逻辑并不复杂，主要是为 `ClassPathBeanDefinitionScanner` 设置一些来自 `@ComponentScan` 的注解属性值，最终执行扫描。`ClassPathBeanDefinitionScanner` 顾名思义是基于类路径的 `Bean` 定义扫描器，真正的扫描工作全部委托给了它。在这些设置过程中，我们需要关注 `basePackages` 的设置：

- 使用 Set 存储合并结果，用于去重
- 获取设置的 `basePackages` 值并添加
- 获取设置的 `basePackageClasses` 值，转换为它们所在的包名并添加
- 如果结果集现在还是空的，获取被标注的配置类所在的包名并添加

> 最后一条规则就是“默认情况下扫描配置类所在的包”的说法由来，并且根据代码可知，如果主动设置了值，这条规则就不起作用了。

```java
public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
    Assert.state(this.environment != null, "Environment must not be null");
    Assert.state(this.resourceLoader != null, "ResourceLoader must not be null");
    // 创建 ClassPathBeanDefinitionScanner
    ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
            componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);
    // Bean 名称生成器
    Class<? extends BeanNameGenerator> generatorClass = componentScan.getClass("nameGenerator");
    boolean useInheritedGenerator = (BeanNameGenerator.class == generatorClass);
    scanner.setBeanNameGenerator(useInheritedGenerator ? this.beanNameGenerator :
            BeanUtils.instantiateClass(generatorClass));

    ScopedProxyMode scopedProxyMode = componentScan.getEnum("scopedProxy");
    if (scopedProxyMode != ScopedProxyMode.DEFAULT) {
        scanner.setScopedProxyMode(scopedProxyMode);
    }
    else {
        Class<? extends ScopeMetadataResolver> resolverClass = componentScan.getClass("scopeResolver");
        scanner.setScopeMetadataResolver(BeanUtils.instantiateClass(resolverClass));
    }

    scanner.setResourcePattern(componentScan.getString("resourcePattern"));
    // 设置 Filter
    for (AnnotationAttributes filter : componentScan.getAnnotationArray("includeFilters")) {
        for (TypeFilter typeFilter : typeFiltersFor(filter)) {
            scanner.addIncludeFilter(typeFilter);
        }
    }
    for (AnnotationAttributes filter : componentScan.getAnnotationArray("excludeFilters")) {
        for (TypeFilter typeFilter : typeFiltersFor(filter)) {
            scanner.addExcludeFilter(typeFilter);
        }
    }
    // 是否懒加载
    boolean lazyInit = componentScan.getBoolean("lazyInit");
    if (lazyInit) {
        scanner.getBeanDefinitionDefaults().setLazyInit(true);
    }
    // 设置 basePackages（使用 Set 去重）
    Set<String> basePackages = new LinkedHashSet<String>();
    // 获取设置的 basePackages 值
    String[] basePackagesArray = componentScan.getStringArray("basePackages");
    for (String pkg : basePackagesArray) {
        // 允许占位符
        String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
                ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
        basePackages.addAll(Arrays.asList(tokenized));
    }
    // 获取 basePackageClasses，本质上是为了获取它们所在的包名
    for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
        basePackages.add(ClassUtils.getPackageName(clazz));
    }
    // 如果为空，获取被标注的配置类所在的包名
    if (basePackages.isEmpty()) {
        basePackages.add(ClassUtils.getPackageName(declaringClass));
    }
    // 排除被标注的配置类本身
    scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
        @Override
        protected boolean matchClassName(String className) {
            return declaringClass.equals(className);
        }
    });
    // 执行扫描
    return scanner.doScan(StringUtils.toStringArray(basePackages));
}
```

`parse` 方法与其说是解析，不如说是封装了一些设置并最终调用 `ClassPathBeanDefinitionScanner`，而设置的属性值来源于 `@ComponentScan` 的注解属性。关于获取 `@ComponentScan` 的注解属性的方法 `AnnotationConfigUtils.attributesForRepeatable` 在分析 `@PropertySource` 时也曾经遇到过，顾名思义我们知道它应该是用于获取可重复的注解的属性。可是它和直接获取注解对象有什么区别呢？

> 我们知道 `@SpringBootApplication` 拥有和 `@ComponentScan` 具备相似的功能，并且可以使用 `scanBasePackages` 和 `scanBasePackageClasses` 这两个属性设置扫描的包。也许你还知道 `@SpringBootApplication` 之所以如此是因为它被标注了 `@ComponentScan`，`scanBasePackages` 和 `scanBasePackageClasses` 分别是它的元注解 `@ComponentScan` 中 `basePackages` 和 `basePackageClasses` 的别名。你甚至可能知道**如果在配置类上使用 `@ComponentScan` 设置包扫描后会导致 `@SpringBootApplication` 设置的包扫描失效**。
**可是为什么呢？**在 `Spring` 中我们会看到从指定类上直接获取目标注解的代码，我们还会看到递归地从元注解上获取目标注解的代码，我们使用 `@ComponentScan` 的经验告诉我们可重复注解不是覆盖彼此而是共同生效，那么为什么 `@SpringBootApplication` 上的 `@ComponentScan` 就被覆盖了呢？**想当然的认为 `@SpringBootApplication` 上标注了 `@ComponentScan` 是一切的原因是不够的**。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

	@AliasFor(annotation = EnableAutoConfiguration.class)
	Class<?>[] exclude() default {};

	@AliasFor(annotation = EnableAutoConfiguration.class)
	String[] excludeName() default {};

	@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
	String[] scanBasePackages() default {};

	@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
	Class<?>[] scanBasePackageClasses() default {};

}
```

## 获取注解属性

`attributesForRepeatable` 方法有两个重载方法，最终调用的版本如下。先后处理了 `@ComponentScan` 和 `@ComponentScans`。

```java
static Set<AnnotationAttributes> attributesForRepeatable(AnnotationMetadata metadata,
        String containerClassName, String annotationClassName) {
    // Set 用于存储结果
    Set<AnnotationAttributes> result = new LinkedHashSet<AnnotationAttributes>();
    // 处理 @ComponentScan
    addAttributesIfNotNull(result, metadata.getAnnotationAttributes(annotationClassName, false));
    // 处理 @ComponentScans
    Map<String, Object> container = metadata.getAnnotationAttributes(containerClassName, false);
    if (container != null && container.containsKey("value")) {
        for (Map<String, Object> containedAttributes : (Map<String, Object>[]) container.get("value")) {
            addAttributesIfNotNull(result, containedAttributes);
        }
    }
    return Collections.unmodifiableSet(result);
}
```

### 检索注解的规则

根据注释，`getAnnotationAttributes` 方法检索给定类型的注解的属性，**检索的目标可以是直接注解也可以是元注解**，同时考虑**组合注解上的属性覆盖**。

- 元注解指的是标注在其他注解上的注解，用于对被标注的注解进行说明，比如 `@SpringBootApplication` 上的 `@ComponentScan` 就被称为元注解，此时 `@SpringBootApplication` 被称为组合注解
- 组合注解中存在属性覆盖现象

> 其实这两点分别对应了我们想要探究的两个问题：`@ComponentScan` 究竟是如何被检索的？注解属性比如 `basePackages` 又是如何被覆盖的？

```java
public Map<String, Object> getAnnotationAttributes(String annotationName, boolean classValuesAsString) {
    // 获取合并的注解属性
    return (this.annotations.length > 0 ? AnnotatedElementUtils.getMergedAnnotationAttributes(
            getIntrospectedClass(), annotationName, classValuesAsString, this.nestedAnnotationsAsMap) : null);
}
```

根据注释，`getMergedAnnotationAttributes` 方法获取所提供元素上方的注解层次结构中指定的 `annotationName` 的第一个注解，并将该注解的属性与注解层次结构较低级别中的注解中的匹配属性合并。注解层次结构中较低级别的属性会覆盖较高级别中的同名属性，并且完全支持单个注解中或是注解层次结构中的 `@AliasFor` 语义。与 `getAllAnnotationAttributes` 方法相反，一旦找到指定 `annotationName` 的第一个注解，此方法使用的搜索算法将停止搜索注解层次结构。因此，指定的 `annotationName` 的附加注解将被忽略。

> 这注释有点太抽象了，理解代码后再来回味吧。

```java
public static AnnotationAttributes getMergedAnnotationAttributes(AnnotatedElement element,
        String annotationName, boolean classValuesAsString, boolean nestedAnnotationsAsMap) {
    // 以 get 语义进行搜索（是指找到即终止搜索？）
    AnnotationAttributes attributes = searchWithGetSemantics(element, null, annotationName,
            new MergedAnnotationAttributesProcessor(classValuesAsString, nestedAnnotationsAsMap));
    // 后处理注解属性
    AnnotationUtils.postProcessAnnotationAttributes(element, attributes, classValuesAsString, nestedAnnotationsAsMap);
    return attributes;
}
```

`searchWithGetSemantics` 方法有多个重载方法，最终调用的版本如下：

- 先获取 `element` 上的所有注解（包括重复的，不包括继承的），**这意味着可重复注解 `@ComponentScan` 标注了多个就会有多个实例**
- 在注解中搜索
- 如果没找到，就从继承的注解中继续搜索

> 本方法是一个会被递归调用的方法，在第一次调用时 `element` 是配置类，之后就是注解。

```java
private static <T> T searchWithGetSemantics(AnnotatedElement element,
        @Nullable Class<? extends Annotation> annotationType, @Nullable String annotationName,
        @Nullable Class<? extends Annotation> containerType, Processor<T> processor,
        Set<AnnotatedElement> visited, int metaDepth) {
    // 防止无限递归
    if (visited.add(element)) {
        try {
            // 获取 element 上的所有注解（包括重复，不包括继承的）
            List<Annotation> declaredAnnotations = Arrays.asList(element.getDeclaredAnnotations());
            // 在获得的注解中搜索
            T result = searchWithGetSemanticsInAnnotations(element, declaredAnnotations,
                    annotationType, annotationName, containerType, processor, visited, metaDepth);
            if (result != null) {
                return result;
            }
            // 表明在直接声明的注解中没有找到
            // 如果 element 是一个类
            if (element instanceof Class) {
                // 获取所有的注解（包括重复的和继承的）
                List<Annotation> inheritedAnnotations = new ArrayList<>();
                for (Annotation annotation : element.getAnnotations()) {
                    // 排除已经搜索过的，只留下继承的注解
                    if (!declaredAnnotations.contains(annotation)) {
                        inheritedAnnotations.add(annotation);
                    }
                }
                // 继续搜索
                result = searchWithGetSemanticsInAnnotations(element, inheritedAnnotations,
                        annotationType, annotationName, containerType, processor, visited, metaDepth);
                if (result != null) {
                    return result;
                }
            }
        }
        catch (Throwable ex) {
            AnnotationUtils.handleIntrospectionFailure(element, ex);
        }
    }

    return null;
}
```

遍历注解进行搜索。

- 先在注解中搜索，**这意味着如果配置类标注了 `@ComponentScan`，直接就找到了**
- 如果没找到再在元注解中搜索，如果配置类只标注了 `@SpringBootApplication`，就是在这部分找到元注解 `@ComponentScan`

> 严格意义上说，并不是直接标注的 `@ComponentScan` 会覆盖 `@SpringBootApplication` 上间接标注的 `@ComponentScan`，而是搜索在找到第一个注解后终止没有继续查找。这解答了我们的第一个疑问。

```java
private static <T> T searchWithGetSemanticsInAnnotations(@Nullable AnnotatedElement element,
        List<Annotation> annotations, @Nullable Class<? extends Annotation> annotationType,
        @Nullable String annotationName, @Nullable Class<? extends Annotation> containerType,
        Processor<T> processor, Set<AnnotatedElement> visited, int metaDepth) {

    // 遍历注解进行查找，如果同时标注 @SpringBootApplication 和 @ComponentScan，在这部分就会找到 @ComponentScan 就返回了
    for (Annotation annotation : annotations) {
        // 获取注解的 Class
        Class<? extends Annotation> currentAnnotationType = annotation.annotationType();
        // 检测是否属于 Java 语言注解包中（以 java.lang.annotation 开头）的注解，例如 @Documented，是的话跳过
        if (!AnnotationUtils.isInJavaLangAnnotationPackage(currentAnnotationType)) {
            // 检测是否满足条件：等于 annotationType（传入 null），或者和目标的名字（@ComponentScan 全限定类名）相同，或者属于总是处理（默认 false）
            if (currentAnnotationType == annotationType ||
                    currentAnnotationType.getName().equals(annotationName) ||
                    processor.alwaysProcesses()) {
                // 处理注解获得注解属性
                T result = processor.process(element, annotation, metaDepth);
                if (result != null) {
                    // processor.aggregates() 默认返回 false
                    if (processor.aggregates() && metaDepth == 0) {
                        processor.getAggregatedResults().add(result);
                    }
                    else {
                        // 注意：难道标注多个 @ComponentScan 也只找到一个就返回了？
                        return result;
                    }
                }
            }
            // 容器里的可重复注解，因为 containerType 为 null，跳过
            else if (currentAnnotationType == containerType) {
                for (Annotation contained : getRawAnnotationsFromContainer(element, annotation)) {
                    T result = processor.process(element, contained, metaDepth);
                    if (result != null) {
                        // No need to post-process since repeatable annotations within a
                        // container cannot be composed annotations.
                        processor.getAggregatedResults().add(result);
                    }
                }
            }
        }
    }

    // 在元注解中递归的搜索，@SpringBootApplication 中的 @ComponentScan 就是在这找到的
    for (Annotation annotation : annotations) {
        // 获取注解的 Class
        Class<? extends Annotation> currentAnnotationType = annotation.annotationType();
        // 检测是否属于 Java 语言注解包中
        if (!AnnotationUtils.isInJavaLangAnnotationPackage(currentAnnotationType)) {
            // 递归到元注解中搜索，深度加 1
            T result = searchWithGetSemantics(currentAnnotationType, annotationType,
                    annotationName, containerType, processor, visited, metaDepth + 1);
            if (result != null) {
                // 进行后处理，注解层次结构中较低级别的属性会覆盖较高级别中的同名属性就是在这发生的
                processor.postProcess(element, annotation, result);
                if (processor.aggregates() && metaDepth == 0) {
                    processor.getAggregatedResults().add(result);
                }
                else {
                    return result;
                }
            }
        }
    }

    return null;
}
```

处理 `@ComponentScan` 获得 `AnnotationAttributes`。

```java
public AnnotationAttributes process(@Nullable AnnotatedElement annotatedElement, Annotation annotation, int metaDepth) {
    return AnnotationUtils.retrieveAnnotationAttributes(annotatedElement, annotation,
            this.classValuesAsString, this.nestedAnnotationsAsMap);
}
```

以 `AnnotationAttributes` 映射的形式检索给定注解的属性。

```java
static AnnotationAttributes retrieveAnnotationAttributes(@Nullable Object annotatedElement, Annotation annotation,
        boolean classValuesAsString, boolean nestedAnnotationsAsMap) {

    Class<? extends Annotation> annotationType = annotation.annotationType();
    AnnotationAttributes attributes = new AnnotationAttributes(annotationType);
    // 遍历属性方法
    for (Method method : getAttributeMethods(annotationType)) {
        try {
            // 获取属性值
            Object attributeValue = method.invoke(annotation);
            // 获取默认值
            Object defaultValue = method.getDefaultValue();
            // 如果默认值不为 null 且和属性值相同
            if (defaultValue != null && ObjectUtils.nullSafeEquals(attributeValue, defaultValue)) {
                attributeValue = new DefaultValueHolder(defaultValue);
            }
            // 属性名 -> 属性值
            attributes.put(method.getName(),
                    adaptValue(annotatedElement, attributeValue, classValuesAsString, nestedAnnotationsAsMap));
        }
        catch (Throwable ex) {
            if (ex instanceof InvocationTargetException) {
                Throwable targetException = ((InvocationTargetException) ex).getTargetException();
                rethrowAnnotationConfigurationException(targetException);
            }
            throw new IllegalStateException("Could not obtain annotation attribute value for " + method, ex);
        }
    }

    return attributes;
}

// 获取在所提供的 annotationType 中声明的与 Java 对注释属性的要求相匹配的所有方法
static List<Method> getAttributeMethods(Class<? extends Annotation> annotationType) {
    // 先从缓存中获取
    List<Method> methods = attributeMethodsCache.get(annotationType);
    if (methods != null) {
        return methods;
    }
    // 遍历方法筛选
    methods = new ArrayList<>();
    for (Method method : annotationType.getDeclaredMethods()) {
        if (isAttributeMethod(method)) {
            ReflectionUtils.makeAccessible(method);
            methods.add(method);
        }
    }
    // 存入缓存
    attributeMethodsCache.put(annotationType, methods);
    return methods;
}

// 确定提供的方法是否是注解的属性方法。
static boolean isAttributeMethod(@Nullable Method method) {
    // 无参数 && 返回值非 void
    return (method != null && method.getParameterCount() == 0 && method.getReturnType() != void.class);
}
```

### 组合注解的属性覆盖

在获得注解属性后还要进行后处理，使用注解层次结构中较低级别的属性覆盖较高级别中的同名（包括 `@AliasFor` 指定的）属性。比如使用 `@SpringBootApplication` 中的 `scanBasePackages` 的值覆盖 `@ComponentScan` 中的 `basePackages` 的值。

```java
public void postProcess(@Nullable AnnotatedElement element, Annotation annotation, AnnotationAttributes attributes) {
    annotation = AnnotationUtils.synthesizeAnnotation(annotation, element);
    // 获取 AnnotationAttributes 的注解类型（@ComponentScan）
    Class<? extends Annotation> targetAnnotationType = attributes.annotationType();

    // Track which attribute values have already been replaced so that we can short
    // circuit the search algorithms.
    Set<String> valuesAlreadyReplaced = new HashSet<>();
    // 获取注解的属性方法（SpringBootApplication）
    for (Method attributeMethod : AnnotationUtils.getAttributeMethods(annotation.annotationType())) {
        String attributeName = attributeMethod.getName();
        // 获取被覆盖的别名
        String attributeOverrideName = AnnotationUtils.getAttributeOverrideName(attributeMethod, targetAnnotationType);

        // Explicit annotation attribute override declared via @AliasFor
        if (attributeOverrideName != null) {
            // 被覆盖的属性的值是否已经被替换
            if (valuesAlreadyReplaced.contains(attributeOverrideName)) {
                continue;
            }

            List<String> targetAttributeNames = new ArrayList<>();
            targetAttributeNames.add(attributeOverrideName);
            valuesAlreadyReplaced.add(attributeOverrideName);

            // 确保覆盖目标注解中的所有别名属性。 (SPR-14069)
            List<String> aliases = AnnotationUtils.getAttributeAliasMap(targetAnnotationType).get(attributeOverrideName);
            if (aliases != null) {
                for (String alias : aliases) {
                    if (!valuesAlreadyReplaced.contains(alias)) {
                        targetAttributeNames.add(alias);
                        valuesAlreadyReplaced.add(alias);
                    }
                }
            }

            overrideAttributes(element, annotation, attributes, attributeName, targetAttributeNames);
        }
        // Implicit annotation attribute override based on convention
        else if (!AnnotationUtils.VALUE.equals(attributeName) && attributes.containsKey(attributeName)) {
            overrideAttribute(element, annotation, attributes, attributeName, attributeName);
        }
    }
}

// 根据提供的注解属性方法的 @AliasFor，获取被覆盖的属性的名称
static String getAttributeOverrideName(Method attribute, @Nullable Class<? extends Annotation> metaAnnotationType) {
    // 获取别名描述符
    AliasDescriptor descriptor = AliasDescriptor.from(attribute);
    // 从元注解中被覆盖的属性名
    return (descriptor != null && metaAnnotationType != null ?
            descriptor.getAttributeOverrideName(metaAnnotationType) : null);
}

// 获取在提供的注解类型中通过 @AliasFor 声明的所有属性别名的映射。该映射由属性名称作为键，每个值代表别名属性的名称列表。空返回值意味着注解没有声明任何属性别名。
static Map<String, List<String>> getAttributeAliasMap(@Nullable Class<? extends Annotation> annotationType) {
    if (annotationType == null) {
        return Collections.emptyMap();
    }
    // 从缓存中获取
    Map<String, List<String>> map = attributeAliasesCache.get(annotationType);
    if (map != null) {
        return map;
    }

    map = new LinkedHashMap<>();
    // 遍历属性方法
    for (Method attribute : getAttributeMethods(annotationType)) {
        // 获取别名列表
        List<String> aliasNames = getAttributeAliasNames(attribute);
        if (!aliasNames.isEmpty()) {
            map.put(attribute.getName(), aliasNames);
        }
    }
    // 存入缓存
    attributeAliasesCache.put(annotationType, map);
    return map;
}

// 获取通过提供的注解属性的 @AliasFor 配置的别名属性的名称列表
static List<String> getAttributeAliasNames(Method attribute) {
    AliasDescriptor descriptor = AliasDescriptor.from(attribute);
    return (descriptor != null ? descriptor.getAttributeAliasNames() : Collections.<String> emptyList());
}

// 覆盖属性
private void overrideAttributes(@Nullable AnnotatedElement element, Annotation annotation,
        AnnotationAttributes attributes, String sourceAttributeName, List<String> targetAttributeNames) {

    Object adaptedValue = getAdaptedValue(element, annotation, sourceAttributeName);
    // 遍历目标属性中的所有应被覆盖的属性（本尊+别名）
    for (String targetAttributeName : targetAttributeNames) {
        attributes.put(targetAttributeName, adaptedValue);
    }
}
```

在代码的注释中我们留下过一个疑问，如果找到了第一个注解就立即返回，那么标注了多个 `@ComponentScan` 呢？当你 `Debug` 时，你会发现并没有走出现直接标注了 `@ComponentScan` 的处理，其实看到反编译后的代码你就知道了，多个 `@ComponentScan` 被合成了一个 `@ComponentScans`，甚至此时设置的三个 `basePackages` 都是生效的。在 `JDK 8` 引入的重复注解机制，并非一个语言层面上的改动，而是编译器层面的改动。在编译后，多个可重复注解 `@ComponentScan` 会被合并到一个容器注解 `@ComponentScans` 中。

> 因此，“`@ComponentScan` 的配置会覆盖 `@SpringBootApplication` 关于包扫描的配置”这句话既对又不对，它在一个常见的个例上表现出的现象是对的，在更普遍的情况中以及本质上是错误的。你也许可以再根据一些情况罗列出类似的“`@ComponentScan` 使用规则”，但是如果你不明白背后的本质，那么这些只是一些死记硬背的陈述，甚至会带给你错误的认知。

```java
// 标注了两个 `@ComponentScan`，对编译后的字节码进行反编译
@SpringBootApplication(
    scanBasePackages = {"com.example"}
)
@ComponentScans({@ComponentScan(
    basePackages = {"com.example.demo"}
), @ComponentScan({"com"})})
public class DemoApplication {
    public DemoApplication() {
    }

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

### 注解内的别名属性

`postProcess` 方法完成了组合注解的属性覆盖，可是对于 `@ComponentScan` 注解而言，它没有被 `postProcess` 方法处理，它又是如何做到设置 `basePackages` 等于设置 `value` 呢？其实这发生在后处理注解属性方法中，该方法会对注解中标注了 `@AliasFor` 的属性强制执行别名语义。通俗地讲，就是**统一**或**校验**互为别名的属性值，要么只设置了其中一个属性的值，其他别名属性会被赋值为相同的值，要么设置为相同的值，否则会报错。

```java
public static AnnotationAttributes getMergedAnnotationAttributes(AnnotatedElement element,
        String annotationName, boolean classValuesAsString, boolean nestedAnnotationsAsMap) {
    // 以 get 语义进行搜索（是指找到即终止搜索？）
    AnnotationAttributes attributes = searchWithGetSemantics(element, null, annotationName,
            new MergedAnnotationAttributesProcessor(classValuesAsString, nestedAnnotationsAsMap));
    // 后处理注解属性
    AnnotationUtils.postProcessAnnotationAttributes(element, attributes, classValuesAsString, nestedAnnotationsAsMap);
    return attributes;
}

static void postProcessAnnotationAttributes(@Nullable Object annotatedElement,
        @Nullable AnnotationAttributes attributes, boolean classValuesAsString, boolean nestedAnnotationsAsMap) {

    if (attributes == null) {
        return;
    }
    // 获取 AnnotationAttributes 的注解类型（@ComponentScan）
    Class<? extends Annotation> annotationType = attributes.annotationType();

    // Track which attribute values have already been replaced so that we can short
    // circuit the search algorithms.
    Set<String> valuesAlreadyReplaced = new HashSet<>();

    if (!attributes.validated) {
        // 校验 @AliasFor 配置
        // 获取别名映射
        Map<String, List<String>> aliasMap = getAttributeAliasMap(annotationType);
        // 遍历
        for (String attributeName : aliasMap.keySet()) {
            // 跳过已处理的
            if (valuesAlreadyReplaced.contains(attributeName)) {
                continue;
            }
            Object value = attributes.get(attributeName);
            // 属性是否已有值
            boolean valuePresent = (value != null && !(value instanceof DefaultValueHolder));
            // 遍历属性的别名列表
            for (String aliasedAttributeName : aliasMap.get(attributeName)) {
                // 跳过已处理的
                if (valuesAlreadyReplaced.contains(aliasedAttributeName)) {
                    continue;
                }
                // 获取别名属性的值
                Object aliasedValue = attributes.get(aliasedAttributeName);
                // 别名属性是否已有值
                boolean aliasPresent = (aliasedValue != null && !(aliasedValue instanceof DefaultValueHolder));

                // Something to validate or replace with an alias?
                if (valuePresent || aliasPresent) {
                    // 如果属性已有值且别名属性也有值，校验是否相等
                    if (valuePresent && aliasPresent) {
                        // Since annotation attributes can be arrays, we must use ObjectUtils.nullSafeEquals().
                        if (!ObjectUtils.nullSafeEquals(value, aliasedValue)) {
                            String elementAsString =
                                    (annotatedElement != null ? annotatedElement.toString() : "unknown element");
                            throw new AnnotationConfigurationException(String.format(
                                    "In AnnotationAttributes for annotation [%s] declared on %s, " +
                                    "attribute '%s' and its alias '%s' are declared with values of [%s] and [%s], " +
                                    "but only one is permitted.", attributes.displayName, elementAsString,
                                    attributeName, aliasedAttributeName, ObjectUtils.nullSafeToString(value),
                                    ObjectUtils.nullSafeToString(aliasedValue)));
                        }
                    }
                    else if (aliasPresent) {
                        // 复制别名属性的值给属性
                        attributes.put(attributeName,
                                adaptValue(annotatedElement, aliasedValue, classValuesAsString, nestedAnnotationsAsMap));
                        valuesAlreadyReplaced.add(attributeName);
                    }
                    else {
                        // 复制属性的值给别名属性
                        attributes.put(aliasedAttributeName,
                                adaptValue(annotatedElement, value, classValuesAsString, nestedAnnotationsAsMap));
                        valuesAlreadyReplaced.add(aliasedAttributeName);
                    }
                }
            }
        }
        // 校验完毕
        attributes.validated = true;
    }

    // 将 `value` 从 `DefaultValueHolder` 替换为原始的 `value`
    for (String attributeName : attributes.keySet()) {
        if (valuesAlreadyReplaced.contains(attributeName)) {
            continue;
        }
        Object value = attributes.get(attributeName);
        if (value instanceof DefaultValueHolder) {
            value = ((DefaultValueHolder) value).defaultValue;
            attributes.put(attributeName,
                    adaptValue(annotatedElement, value, classValuesAsString, nestedAnnotationsAsMap));
        }
    }
}
```

## 总结

> 又是一篇在写之前自认心里有数，以为可以很快总结完，却不知不觉写了很久，也收获了很多的文章。在刚开始，我只是想接续分析 `@Configuration` 的思路补充关于 `@ComponentScan` 的内容，但是渐渐地我又想要回应心里的疑问，`@ComponentScan` 和 `@SpringBootApplication` 一起使用的问题的本质原因是什么？`Spring` 框架真的很好用，好用到你不用太关心背后的原理，好用到你有时候用一个本质上不太正确的结论“走遍天下却几乎不会遇到问题”。说实话，研究完也有点索然无味，尤其是花了这么多时间看自己很讨厌的关于解析的代码，只能说解开了一个卡点也算疏通了一口气，但是时间成本好大啊，得多看点能“面试”的技术啊！！！

综上分析，`@SpringBootApplication` 的包扫描功能本质上还是 `@ComponentScan` 提供的，但是和常见的嵌套注解不同，检索 `@ComponentScan` 有一套独特的算法，导致 `@SpringBootApplication` 和 `@ComponentScan` 并非简单的叠加效果。

- `Spring` 会先获取 `@ComponentScan` 的注解属性再获取 `@ComponentScans` 的注解属性
- 以 `@ComponentScan` 为例，只获取给定配置类上的注解层次结构中的**第一个** `@ComponentScan`
- 先从直接标注的注解开始，再递归地搜索元注解，这一点决定了 `@ComponentScan` 优先级高于 `@SpringBootApplication`
- 使用注解层次结构中较低级别的属性覆盖较高级别的同名（支持 `@AliasFor`）属性，这一点决定了 `@SpringBootApplication` 可以设置扫描路径
- 多个 `@ComponentScan` 在编译后隐式生成 `@ComponentScans`，这一点决定多个 `@ComponentScan` 彼此之间以及和 `@SpringBootApplication` 互不冲突
