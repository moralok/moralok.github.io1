---
title: Spring 中 @Import 注解的使用和源码分析
date: 2023-12-04 08:36:21
tags: [java, spring]
---

`Import` 注解是 `Spring` 基于 `Java` 注解配置的重要组成部分，处理 `Import` 注解是处理 `Configuration` 注解的子过程之一，本文将介绍 `Import` 注解的 `3` 种使用方式，然后通过分析源码和处理过程示意图解释它是如何导入（注册） `BeanDefinition` 的。

<!-- more -->

- 本文的写作动机继承自{% post_link 'source-code-analysis-of-Spring-Configuration-annotation' Spring @Configuration 注解的源码分析 %}，处理 `@Import` 是处理 `@Configuration` 过程的一部分。

## 使用方式

`Import` 注解有 `3` 种导入（注册） `BeanDefinition` 的方式：

1. 使用 `Import` 将目标类的 `Class` 对象，解析为 `BeanDefinition` 并注册。
2. 使用 `Import` 配合 `ImportSelector` 的实现类，将 `selectImports` 方法返回的所有全限定类名字符串，解析为 `BeanDefinition` 并注册。
3. 使用 `Import` 配合 `ImportBeanDefinitionRegistra`r 的实现类，在 `registerBeanDefinitions` 方法中，直接向 `BeanDefinitionRegistry` 中注册 `BeanDefinition`。

## 测试用例

测试了使用 `Import` 注解的 `3` 种方式：

1. 使用 `Import` 直接导入（注册） `Red`。
2. 配合 `ImportBeanDefinitionRegistrar` 间接注册 `Color`。
3. 配合 `ImportSelector` 间接导入（注册） `Blue`。

用例中的特别地测试了以下两种情况：

1. 使用 `Import` 直接导入和配合 `ImportSelector` 间接导入相同的类 `Red` 只会注册一个 `BeanDefinition`。
2. 尽管 `MyImportSelector` 书面顺序在 `MyImportBeanDefinitionRegistrar` 之后，但是 `MyImportBeanDefinitionRegistrar` 判断 `registry` 是否包含在 `MyImportSelector` 导入的类 `Blue` 时，不受顺序影响。

```java
@Configuration
@Import({Red.class, MyImportBeanDefinitionRegistrar.class, MyImportSelector.class,})
public class ImportConfig {
}

public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        boolean hasRed = registry.containsBeanDefinition("com.moralok.bean.Red");
        boolean hasBlue = registry.containsBeanDefinition("com.moralok.bean.Blue");
        if (hasRed && hasBlue) {
            BeanDefinition beanDefinition = new RootBeanDefinition(Color.class);
            registry.registerBeanDefinition("color", beanDefinition);
        }
    }
}

public class MyImportSelector implements ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        return new String[] {"com.moralok.bean.Blue", "com.moralok.bean.Red"};
    }
}

public class Color {
}

public class Red {
}

public class Blue {
}

public class IocTest {
    @Test
    public void importTest() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(ImportConfig.class);
        String[] beanDefinitionNames = ac.getBeanDefinitionNames();
        for (String name : beanDefinitionNames) {
            System.out.println("beanDefinitionName.........." + name);
        }
    }
}
```

测试结果

```console
......
beanDefinitionName..........importConfig
beanDefinitionName..........com.moralok.bean.Red
beanDefinitionName..........com.moralok.bean.Blue
beanDefinitionName..........color
```

## 源码分析

关于 `Import` 注解的源码分析需要建立在对关于 `Configuration` 注解的源码的了解基础上，因为前者是 `Spring` 解析配置类处理过程的一部分，可以参考文章:

- {% post_link 'source-code-analysis-of-Spring-Configuration-annotation' 'Spring @Configuration 注解的源码分析' %}

### 获取要导入的目标

在 `doProcessConfigurationClass` 方法中处理配置类构建配置模型时，会调用 `processImports` 方法处理 `Import` 注解。在进入方法前，会调用 `getImports` 方法从 `sourceClass` 获取要导入的目标。

> 注意：目标不仅仅来自直接标注在 `sourceClass` 上的 `Import` 注解，因为 `sourceClass` 上可能还有其他的注解，这些注解自身可能标注了 `Import` 注解，因此需要递归地遍历所有注解，找到所有的 `Import` 注解。

```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {
    // 前后省略 @PropertySource、@ComponentScan、@ImportSource、@Bean 等注解的处理
    // 处理 Import 注解
    processImports(configClass, sourceClass, getImports(sourceClass), true);
}
```

`collectImports` 方法是一种常见的递归写法（深度优先遍历）。`imports` 存放要导入的目标，`visited` 存放已经访问过的 `sourceClass`。`sourceClass` 在入口处包装了一个普通的 `Class`，在递归的过程中包装的都是一个注解 `Class`。

> 注意：这里还没有检测循环导入的情况并抛出异常，但 `visited` 保证了只会遍历一次。

```java
// 获取 Import 注解 value 中的 Class 对象，并包装为 SourceClass 返回
private Set<SourceClass> getImports(SourceClass sourceClass) throws IOException {
    Set<SourceClass> imports = new LinkedHashSet<SourceClass>();
    Set<SourceClass> visited = new LinkedHashSet<SourceClass>();
    collectImports(sourceClass, imports, visited);
    return imports;
}

// 递归地收集要导入的目标（包装为 SourceClass）
private void collectImports(SourceClass sourceClass, Set<SourceClass> imports, Set<SourceClass> visited)
        throws IOException {

    // 如果 sourceClass 尚未访问过
    if (visited.add(sourceClass)) {
        // 遍历 sourceClass 上的注解
        for (SourceClass annotation : sourceClass.getAnnotations()) {
            String annName = annotation.getMetadata().getClassName();
            // 只要注解的名称不是 java 开头或者不是 Import 注解
            if (!annName.startsWith("java") && !annName.equals(Import.class.getName())) {
                // 将该注解作为 sourceClass 递归地调用
                collectImports(annotation, imports, visited);
            }
        }
        // 将 Import 注解的 value 的值转换为 sourceClass 加入 imports
        imports.addAll(sourceClass.getAnnotationAttributes(Import.class.getName(), "value"));
    }
}
```

这时候，并不区分要导入的目标的 `Class` 有什么特别之处，`Import` 注解的语义，此时宽泛地说就是：“将 `value` 中的类导入”。但是显而易见，这样的方式不够灵活，因此才有了另外两种更有灵活性的导入方式：`ImportSelector` 和 `ImportBeanDefinitionRegistrar`，`Spring` 最终不会真的注册这两种类，而是注册它们“介绍”的类，相当于把确定导入什么类的工作委托给它们。

### 处理要导入的目标

`processImports` 方法是处理 `Import` 注解的核心方法，这里的处理逻辑就对应着 `Import` 注解的三种使用方式。主要步骤如下：

- 检测要导入的候选者不为空
- 判断是否要检测循环导入以及是否存在循环导入
- 处理要导入的候选者
    - 如果是 `ImportSelector` 类型，调用 `selectImports` 方法获取新的要导入的目标，递归调用 `processImports` 处理
    - 如果是 `ImportBeanDefinitionRegistrar` 类型，添加到配置模型 `configClass`（出口 `1`）
    - 如果是其他剩余情况，作为配置类处理（出口 `2`）

```java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
			Collection<SourceClass> importCandidates, boolean checkForCircularImports) throws IOException {
    // 如果要导入的目标为空，直接返回
    if (importCandidates.isEmpty()) {
        return;
    }

    if (checkForCircularImports && isChainedImportOnStack(configClass)) {
        // 如果要检查循环导入，且确实存在循环导入，则抛出异常
        this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
    }
    else {
        // 将配置模型放入 importStack，用于检查循环导入
        this.importStack.push(configClass);
        try {
            // 遍历每一个准备导入的目标
            for (SourceClass candidate : importCandidates) {
                // 如果是 ImportSelector 类型，委托给它确定导入目标
                if (candidate.isAssignable(ImportSelector.class)) {
                    // 加载类
                    Class<?> candidateClass = candidate.loadClass();
                    // 实例化得到 ImportSelector 实例
                    ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                    // 调用其 Aware 接口
                    ParserStrategyUtils.invokeAwareMethods(
                            selector, this.environment, this.resourceLoader, this.registry);
                    if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
                        // 如果是 DeferredImportSelector 类型，存入 deferredImportSelectors 推迟调用
                        this.deferredImportSelectors.add(
                                new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
                    }
                    else {
                        // 调用 selectImports 方法，返回要导入的目标的全限定类名
                        String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                        // 包装为 SourceClass
                        Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                        // 递归调用 processImports
                        // 从这里看，ImportSelector 本质上是更加灵活的 Import
                        processImports(configClass, currentSourceClass, importSourceClasses, false);
                    }
                }
                // 如果是 ImportBeanDefinitionRegistrar 类型，委托给它注册额外的 BeanDefinitions
                else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                    // 加载类
                    Class<?> candidateClass = candidate.loadClass();
                    // 实例化得到 ImportBeanDefinitionRegistrar 实例
                    ImportBeanDefinitionRegistrar registrar =
                            BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                    // 调用其 Aware 接口
                    ParserStrategyUtils.invokeAwareMethods(
                            registrar, this.environment, this.resourceLoader, this.registry);
                    // 添加到配置模型
                    configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                }
                // 既不是 ImportSelector，也不是 ImportBeanDefinitionRegistrar 的其他剩余情况，将其视为被 Configuration 注解标注的配置类进行处理
                else {
                    this.importStack.registerImport(
                            currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                    // asConfigClass 方法建立了 candidate importBy configClass 的关系
                    processConfigurationClass(candidate.asConfigClass(configClass));
                }
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to process import candidates for configuration class [" +
                    configClass.getMetadata().getClassName() + "]", ex);
        }
        finally {
            // pop 配置模型
            this.importStack.pop();
        }
    }
}
```

#### 类型一：ImportSelector

如果要导入的目标是 `ImportSelector` 类型，那么 `Spring` 将确定真正导入什么目标的工作委托给它，不导入目标本身，实际上只导入目标“介绍”的类。具体步骤是：

1. 先获取 `Class` 对象
2. 再实例化得到一个 `ImportSelector` 实例
3. 调用 `selectImports` 方法，该方法返回的是类的全限定名，这样就得到了真正要导入的目标
4. 再次递归调用 `processImports`

`ImportSelector` 就像它名字的含义一样，本质上是一种导入选择器，是一种更加灵活的 `getImports` 方法。由于返回的目标可能属于三种情形中的任意一种，所以对这些目标的处理还是要回到 `processImports` 方法。可以说 `ImportSelector` 类型本身不是 `processImports` 方法的出口，它最终会转换为 `ImportBeanDefinitionRegistrar` 或其他剩余情况。

`ImportSelector` 灵活性的来源：

- `selectImports` 的 `AnnotationMetadata` 参数，为它提供了根据注解信息返回要导入的目标的能力
- `ImportSelector` 可以实现 `Aware` 接口，用以感知到一些容器级别的资源，如 `BeanFactory`，这为它提供了根据这些资源中的信息返回要导入的目标的能力

#### 类型二：ImportBeanDefinitionRegistrar

如果要导入的目标是 `ImportBeanDefinitionRegistrar`，它会和 `ImportSelector` 有些相似却又有所不同。`Spring` 同样将确定真正导入什么目标的工作委托给它，不导入目标本身，实际上只导入目标“介绍”的类。

1. 先获取 `Class` 对象
2. 再实例化得到一个 `ImportBeanDefinitionRegistrar` 实例
3. 添加到配置模型 `configClass` 的 `importBeanDefinitionRegistrars` 属性

`ImportBeanDefinitionRegistrar` 不像 `ImportSelector` 需要进一步处理，它本身就代表着一个返回出口，成为了配置模型的一部分。但是请注意，`registerBeanDefinitions` 方法此时并没有被调用。

`ImportBeanDefinitionRegistrar` 灵活性的来源：

- `registerBeanDefinitions` 的 `AnnotationMetadata` 参数，为它提供了根据注解信息决定注册 `BeanDefinition` 的能力
- `registerBeanDefinitions` 的 `BeanDefinitionRegistry` 参数，为它提供了根据 `BeanDefinitionRegistry` 中的信息决定注册 `BeanDefinition` 的能力
- `ImportBeanDefinitionRegistrar` 可以实现 `Aware` 接口，用以感知到一些容器级别的资源，如 `BeanFactory`，这为它提供了根据这些资源中的信息返回要导入的目标的能力

#### 类型三：其他剩余情况

如果要导入的目标属于既不是 `ImportSelector` 也不是 `ImportBeanDefinitionRegistrar` 的其他剩余情况，那么 `Spring` 将其视为被 `Configuration` 注解标注的配置类进行处理。这里的处理逻辑是，`Import` 注解导入的类可能不是一个普通的类，而是一个配置类，因此需要回到 `processConfigurationClass` 进行处理。`processConfigurationClass` 方法正是本文开头的 `doProcessConfigurationClass` 方法的调用方，这里有两个地方值得注意：

- `Import` 注解产生的 `ConfigurationClass` 根据不同的情况需要合并或者被抛弃，显式声明比 Import 导入的优先级更高。
- 其他剩余情况下，目标最终会转换为一个配置模型，添加到 `parser` 的 `configurationClasses` 属性。

```java
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
    // 判断是否跳过处理
    if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
        return;
    }

    // 如果配置模型已经存在
    ConfigurationClass existingClass = this.configurationClasses.get(configClass);
    if (existingClass != null) {
        // 如果新的配置模型代表的类，是 Import 导入的
        if (configClass.isImported()) {
            // 如果已存在的配置模型也是 Import 导入的
            if (existingClass.isImported()) {
                // 合并它们的来源
                // 比如一个类 A 既被 Config1 上的 Import 注解导入，也被 Config2 上的 Import 导入
                existingClass.mergeImportedBy(configClass);
            }
            // 否则忽略新的因为 Import 导入而产生的配置模型
            return;
        }
        else {
            // 使用显式定义的代替 Import 导入的（显式定义的和 Import 导入的有什么不同吗）
            this.configurationClasses.remove(configClass);
            for (Iterator<ConfigurationClass> it = this.knownSuperclasses.values().iterator(); it.hasNext();) {
                if (configClass.equals(it.next())) {
                    it.remove();
                }
            }
        }
    }

    // 先递归地处理配置类和它的父类，因为配合各种注解，可能引入更多的类
    SourceClass sourceClass = asSourceClass(configClass);
    do {
        sourceClass = doProcessConfigurationClass(configClass, sourceClass);
    }
    while (sourceClass != null);

    // 一个配置类，本身最终被解析成配置模型（配置模型在后续将会解析出 BeanDefinition）
    this.configurationClasses.put(configClass, configClass);
}
```

### DeferredImportSelector 的调用时机

在解析完每一批（注释中说“全部”）的配置类后，会统一调用 `DeferredImportSelector`。它作为一个标记接口推迟了 `selectImports` 的时机，打破了处理顺序的限制，在方法被调用时，可以得到更加完整的信息。注释中说“在选择导入的目标是 `@Conditional` 时，这个类型的选择器会很有用”，但是我不太理解，因为这个时候，处理配置类得到的信息尚未转换为 `ImportSelector` 可以感知到的信息，不像 `ImportBeanDefinitionRegistrar`，它被调用的时机在最后，也因此可以感知到更多的信息。

```java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
    this.deferredImportSelectors = new LinkedList<DeferredImportSelectorHolder>();

    for (BeanDefinitionHolder holder : configCandidates) {
        BeanDefinition bd = holder.getBeanDefinition();
        try {
            if (bd instanceof AnnotatedBeanDefinition) {
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
    // 调用 DeferredImportSelectors
    processDeferredImportSelectors();
}

private void processDeferredImportSelectors() {
    // 获取处理这一批配置类获得的 DeferredImportSelectors
    List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
    // 清空
    this.deferredImportSelectors = null;
    // 排序
    Collections.sort(deferredImports, DEFERRED_IMPORT_COMPARATOR);
    // 遍历
    for (DeferredImportSelectorHolder deferredImport : deferredImports) {
        ConfigurationClass configClass = deferredImport.getConfigurationClass();
        try {
            // 调用 selectImports 获取要导入的目标
            String[] imports = deferredImport.getImportSelector().selectImports(configClass.getMetadata());
            // 调用 processImports 处理要导入的目标，这里不管循环导入？竟然是任由 StackOverFlow
            processImports(configClass, asSourceClass(configClass), asSourceClasses(imports), false);
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to process import candidates for configuration class [" +
                    configClass.getMetadata().getClassName() + "]", ex);
        }
    }
}
```

### ImportBeanDefinitionRegistrar 的调用时机

`ConfigurationClassPostProcessor` 在每次解析得到新的一批配置模型后，都会调用 `ConfigurationClassBeanDefinitionReader` 的 `loadBeanDefinitions` 方法加载 `BeanDefinition`，在这过程的最后会从 `ImportBeanDefinitionRegistrar` 加载 `BeanDefinition`。这代表在处理同一批配置类时，在 `registerBeanDefinitions` 方法中总是能感知到以其他方式注册到 `BeanDefinitionRegistry` 中的 `BeanDefinition`，不论书面定义的顺序如何。

```java
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
    TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
    // 遍历每一个配置模型
    for (ConfigurationClass configClass : configurationModel) {
        // 从配置模型中加载 BeanDefinistion
        loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
    }
}

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

    // 如果配置模型本身是导入的，为自身注册 BeanDefinition
    if (configClass.isImported()) {
        registerBeanDefinitionForImportedConfigurationClass(configClass);
    }
    // 为 BeanMethod 加载 BeanDefinition（Bean 注解）
    for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        loadBeanDefinitionsForBeanMethod(beanMethod);
    }
    // 为 ImportResources 加载 BeanDefinition（ImportResource 注解）
    loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
    // 从 ImportBeanDefinitionRegistrar 加载 BeanDefinition
    loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}

private void loadBeanDefinitionsFromRegistrars(Map<ImportBeanDefinitionRegistrar, AnnotationMetadata> registrars) {
    // 遍历 ImportBeanDefinitionRegistrar 调用 registerBeanDefinitions 方法注册 BeanDefinition
    for (Map.Entry<ImportBeanDefinitionRegistrar, AnnotationMetadata> entry : registrars.entrySet()) {
        entry.getKey().registerBeanDefinitions(entry.getValue(), this.registry);
    }
}
```

### 循环导入的检测

在处理导入的目标前将配置类放入 `importStack`，处理完毕移除。如果要导入的目标属于其他剩余情况时，注册被导入类->所有导入类集合的映射关系。

```java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
			Collection<SourceClass> importCandidates, boolean checkForCircularImports) throws IOException {
    // ...
    if (checkForCircularImports && isChainedImportOnStack(configClass)) {
        // 如果要检查循环导入，且确实存在循环导入，则抛出异常
        this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
    }
    else {
        // 将配置模型放入 importStack，用于检查循环导入
        this.importStack.push(configClass);
        try {
            // 遍历每一个准备导入的目标
            for (SourceClass candidate : importCandidates) {
                // ...
                else {
                    // 记录了被导入类->所有导入类集合的映射关系
                    this.importStack.registerImport(
                            currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                    // asConfigClass 方法建立了 candidate importBy configClass 的关系
                    processConfigurationClass(candidate.asConfigClass(configClass));
                }
            }
        }
        // ...
        finally {
            // pop 配置模型
            this.importStack.pop();
        }
    }
}
```

检测是否发生循环导入。以当前类开始，循环向上查找最近一个导入自身的类，如果找到自身，说明发生循环导入。

```java
private boolean isChainedImportOnStack(ConfigurationClass configClass) {
    // 如果 importStack 已存在该配置模型
    if (this.importStack.contains(configClass)) {
        String configClassName = configClass.getMetadata().getClassName();
        // 获取最新一个导入 configClass 的类
        AnnotationMetadata importingClass = this.importStack.getImportingClassFor(configClassName);
        // 循环查找导入类的最近一个导入类，如果找到了自身，表示发生循环导入
        while (importingClass != null) {
            if (configClassName.equals(importingClass.getClassName())) {
                return true;
            }
            importingClass = this.importStack.getImportingClassFor(importingClass.getClassName());
        }
    }
    return false;
}
```

## 总结

### 对比

||`ImportSelector`|`ImportBeanDefinitionRegistrar`|其他剩余情况|
|--|--|--|--|
|灵活性|中|高|低|
|处理结果||转换为配置模型的一部分|转换为一个配置模型|
|方法调用时机|立即（或解析配置类的最后）|加载 `BeanDefinition` 的最后||
|方法的结果|获取 `Import` 目标|直接注册 `BeanDefinition`||


### 处理过程示意图

<div style="width:70%;margin:auto">{% asset_img "Pasted image 20231205175232.png" Import 注解的处理过程示意图 %}</div>
