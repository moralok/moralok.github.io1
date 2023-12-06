---
title: Spring Boot 自定义 starter 和自动配置的工作原理
date: 2023-12-06 08:02:11
tags: [spring, spring boot, auto configuration]
---

如果你正在参与一个共享库的开发，你可能会想为使用方提供自动配置的支持，以帮助对方快速地接入和使用。自动配置机制往往和 `starter` 联系在一起，本文将介绍如何创建一个自定义的 `starter` 并从源码角度分析 `Spring Boot` 自动配置的工作原理。

<!-- more -->

## 自定义 starter

一个 library 的完整 `Spring Boot starter` 可能包含以下组件：

- 自动配置模块：包含自动配置的代码。
- 启动模块：提供“自动配置模块、library 以及其他有用的依赖项”的依赖项。简而言之，添加 `starter` 之后应该足以开始使用这个 library。

> **如果你不需要将自动配置的代码和依赖项管理分开，你可以将它们合并到一个模块中**。

### 命名规范

- 不要以 `spring-boot` 开头命名模块，即使你使用的是不同的 Maven groupId，因为 `Spring` 可能在将来提供官方的自动配置支持。自定义 `starter` 约定俗成的命名方式是 `xxx-spring-boot-starter`。
- 如果你的 `starter` 提供了配置属性的定义，请选择适当的命名空间，避免使用 `Spring Boot` 的命名空间，否则他们未来的修改可能破坏你的配置。


以下将通过一款基于 `Redis` 实现的分布式锁 [**redis-lock**](https://github.com/moralok/redis-lock) 的 `starter` 介绍如何创建一个自定义的 `Spring Boot starter`。**注意：实际上项目中的的 `redis-lock-spring-boot-starter` 合并了自动配置模块和启动模块**。

### 自动配置模块

自动配置模块包含开始使用 library 所需要的一切配置。它还可能包含配置键定义（`@ConfigurationProperties`）和任何其他可用于进一步自定义组件初始化方式的回调接口。

按照惯例，模块命名为 `redis-lock-spring-boot-autoconfigure`。

#### 依赖项

自动配置模块需要添加以下依赖。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure</artifactId>
    <version>${spring-boot.version}</version>
</dependency>
```

#### 配置类

和平常在 `Spring` 中使用一个 library 时一样，创建配置类并配置好使用它所需要的 `Bean`。

- `Configuration` 注解，标识为配置类
- `Bean` 注解，配置所需要的 `Bean`
- `EnableConfigurationProperties` 注解，启用配置属性（可选）

```java
@Configuration
@EnableConfigurationProperties(RedisLockProperties.class)
public class RedisLockAutoConfiguration {

    @Autowired
    private RedisLockProperties redisLockProperties;

    @Bean
    @ConditionalOnMissingBean(RedisClient.class)
    public RedisClient redisClient() {
        RedisURI redisURI = new RedisURI();
        redisURI.setHost(redisLockProperties.getHost());
        redisURI.setPort(redisLockProperties.getPort());
        redisURI.setDatabase(redisLockProperties.getDatabase());
        if (redisLockProperties.getUsername() != null) {
            redisURI.setUsername(redisLockProperties.getUsername());
        }
        if (redisLockProperties.getPassword() != null) {
            redisURI.setUsername(redisLockProperties.getPassword());
        }
        return RedisClient.create(redisURI);
    }

    @Bean
    @ConditionalOnMissingBean(RedisLockManager.class)
    public RedisLockManager redisLockManager(RedisClient redisClient) {
        return new RedisLockManager(redisClient);
    }
}
```

#### 配置属性

你可能需要定义一些配置属性来设置使用 library 所需要的属性。

```java
@ConfigurationProperties(prefix = RedisLockProperties.PREFIX)
public class RedisLockProperties {

    public static final String PREFIX = "redis-lock";
    private String host = "localhost";
    private int port = 6379;
    private int database = 0;
    private String username;
    private String password;
    private long waitTimeMillis;
    private long leaseTimeMillis;
    // 省略 setter 和 getter 方法
}
```

#### spring.factories 文件

在 `src/main/resources/META-INF` 目录中添加一个 `spring.factories` 文件，文件内容如下。键为 `EnableAutoConfiguration` 的全限定名，值为配置类的全限定名，如果需要配置多个配置类，可以用逗号分隔。

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.moralok.redislock.autoconfigure.RedisLockAutoConfiguration
```

### 启动模块

`starter` 实际上是一个空的 `jar`，它唯一的目的就是提供使用 `library` 所需要的依赖项。

按照惯例，模块命名为 `redis-lock-spring-boot-starter`。

需要引入以下依赖：

```xml
<!-- library 的依赖项 -->
<dependency>
    <groupId>com.moralok.redis-lock</groupId>
    <artifactId>core</artifactId>
    <version>${redis-lock.version}</version>
</dependency>
<!-- 自动配置模块的依赖项 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>redis-lock-spring-boot-autoconfigure</artifactId>
    <version>${redis-lock.version}</version>
</dependency>
<!-- 其他需要的依赖的依赖项，比如日志相关的 -->
```

### 使用

这样就创建了一个自定义 `starter`。在项目中引入 `starter` 后，无需进一步配置，即可使用 `RedisLockManager` 和 `RedisClient`。

```xml
<dependency>
    <groupId>com.moralok.redis-lock</groupId>
    <artifactId>redis-lock-spring-boot-starter</artifactId>
    <version>${redis-lock.version}</version>
</dependency>
```

## 自动配置的工作原理

从自定义 `starter` 的过程来看，使用 `library` 所需要的配置类和依赖项并没有“凭空消失”，而是由 `starter` 的编写者提供。然而在正常情况下，第三方的 `jar` 中的配置类并不在 `Spring` 扫描 `Bean` 的范围内，那么 `starter` 中的配置类是如何被注册到 `Spring` 容器中呢？我们做的事情中，看起来比较特别的一件事情是添加了 `spring.factories` 文件。

### SpringBootApplication 注解

在 `Spring Boot` 的启动类（也是 `Spring context` 的最初配置类）上，标注了 `SpringBootApplication` 注解。该注解上标注了 `EnableAutoConfiguration` 注解，它的全限定名正是 `spring.factories` 文件中配置的键。注解的名字表明它用于**启用自动配置**功能。

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

### 启用自动配置

`EnableAutoConfiguration` 注解用于启用自动配置功能。该注解上标注了 `Import` 注解，导入了 `AutoConfigurationImportSelector`。很多形似 `EnableXXX` 的注解都是通过 `Import` 注解导入（注册）一些配置类，达到启用 `XXX` 功能的目的。`Import` 注解的功能详见之前的文章：

- {%post_link 'use-and-analysis-of-Import-annotation-in-Spring' 'Spring 中 @Import 注解的使用和源码分析' %}

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	Class<?>[] exclude() default {};

	String[] excludeName() default {};
}
```

### 自动配置导入选择器

导入选择器 `ImportSelector` 的 `selectImports` 方法返回要导入的类的全限定名。`AutoConfigurationImportSelector` 的名字含义是自动配置导入选择器，顾名思义它返回的应该是要导入的**自动配置类**。自动配置类这个说法有点容易让人误解，好像这个配置类本身具备“自动”的特性，实际上它就是一个普通的配置类。自动配置描述的是一种机制，想象一下，如果我们在 `selectImports` 方法中返回 `starter` 中的配置类 `RedisLockAutoConfiguration`，是不是就为 `redis-lock` 完成了自动配置。事实上，`selectImports` 方法的作用就是找到并返回那些需要被自动配置的配置类。

```java
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    // 检测是否启用自动配置
    if (!isEnabled(annotationMetadata)) {
        // 如果未启用，返回空数组
        return NO_IMPORTS;
    }
    try {
        // 加载自动配置的元数据
        AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
                .loadMetadata(this.beanClassLoader);
        // 获取注解中配置的 exclude 和 excludeName
        AnnotationAttributes attributes = getAttributes(annotationMetadata);
        // 核心方法：获取候选的配置类
        List<String> configurations = getCandidateConfigurations(annotationMetadata,
                attributes);
        // 移除重复的
        configurations = removeDuplicates(configurations);
        // 排序
        configurations = sort(configurations, autoConfigurationMetadata);
        // 从注解的配置中获取需要排除的
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        // 检查排除的类，如果已加载且不在 configurations 中，抛出异常（不理解原因）
        checkExcludedClasses(configurations, exclusions);
        // 移除需要排除的
        configurations.removeAll(exclusions);
        // 过滤，获取 spring.factories 中的 AutoConfigurationImportFilter 执行过滤
        configurations = filter(configurations, autoConfigurationMetadata);
        // 触发自动配置类导入事件
        fireAutoConfigurationImportEvents(configurations, exclusions);
        return StringUtils.toStringArray(configurations);
    }
    catch (IOException ex) {
        throw new IllegalStateException(ex);
    }
}
```

可以通过环境变量 `spring.boot.enableautoconfiguration` 覆盖是否启用自动配置功能。

```java
protected boolean isEnabled(AnnotationMetadata metadata) {
    if (getClass() == AutoConfigurationImportSelector.class) {
        return getEnvironment().getProperty(
                EnableAutoConfiguration.ENABLED_OVERRIDE_PROPERTY, Boolean.class,
                true);
    }
    return true;
}
```

### 获取候选的配置类

我们前面提到过，自动配置类本身只是普通的配置类，那么有什么标记或特征表明目标是一个自动配置类吗？有的，凡是配置在 `spring.factories` 文件中 `EnableAutoConfiguration`（`org.springframework.boot.autoconfigure.EnableAutoConfiguration`） 键下的类，就是候选的自动配置类。
`getCandidateConfigurations` 方法用于获取候选的配置类。该方法运用了 **`Spring` 的 `SPI`** 机制，通过 `SpringFactoriesLoader` 获得所有配置在 `spring.factories` 文件中，`org.springframework.boot.autoconfigure.EnableAutoConfiguration` 键下的类，其中就包括了 `RedisLockAutoConfiguration`。这样就完成了自动配置。

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
        AnnotationAttributes attributes) {
    // 通过 SpringFactoriesLoader 加载候选的配置
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
            getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
    Assert.notEmpty(configurations,
            "No auto configuration classes found in META-INF/spring.factories. If you "
                    + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}

protected Class<?> getSpringFactoriesLoaderFactoryClass() {
    return EnableAutoConfiguration.class;
}
```

基于 `Spring Boot SPI` 机制获取配置在 `spring.factories` 文件中的自动配置类的过程我们不再分析，可以参见以下文章：

- {%post_link 'how-does-Spring-Boot-SPI-works' 'Spring Boot SPI 的工作原理' %}

## 让 starter 更好用

### 为配置属性生成元数据

在平时开发时你可能会注意到，有时候在配置文件 `application.properties` 或 `application.yml` 中编写配置时，`IDEA` 会自动提示我们存在哪些配置，默认值是什么。

<div style="width:70%;margin:auto">{% asset_img "Snipaste_2023-12-07_00-08-47.png" IDEA 对配置属性的智能提示 %}</div>

只需要添加以下依赖，在编译项目时，就会自动调用该处理器 `spring-boot-configuration-processor` 为你的项目中被 `ConfigurationProperties` 注解标注的类生成配置元数据文件。

```xml
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-configuration-processor</artifactId>  
    <optional>true</optional>  
</dependency>
```

> 注意：不要盲目手打相信智能提示弄错了依赖，谁能想到 `Spring` 有好几个命名这么像的 `processor`，偏偏网上还有各种复制粘贴的文章解答在多模块项目中 `spring-boot-configuration-processor` 出现的问题——来自 `Debug` 到深夜的人的怨念。

<div style="width:70%;margin:auto">{% asset_img "Snipaste_2023-12-06_23-52-47.png" 易弄错的 processor %}</div>

### 配合 Conditional 注解

你几乎总是希望在自动配置类中包含一个或者多个 `Conditional` 注解。`ConditionalOnMissingBean` 是一个常用的注解，允许开发人员在对默认设置不满意时覆盖自动配置。

### 谨慎地提供依赖

不要对添加 `starter` 的项目做出假设，如果你的 `starter` 需要用到别的 `starter`，也请提到它们。为你的 library 的典型用法选择一组适当的默认依赖，避免引入不必要的依赖项，尽管当可选的依赖项很多时这可能有些困难。

## 总结

`Spring Boot` 的自动配置在底层是通过标准的 `Configuration` 注解实现的，配合 `Conditional` 注解限制何时应用自动配置。“自动”的特性是基于两个重要的机制：

- `SPI` 机制，从 `spring.factories` 文件中，获取自动配置类的全限定类名
- `Import` 机制，导入从 `ImportSelector` 返回的类

工作原理的示意图如下：

<div style="width:70%;margin:auto">{% asset_img "Pasted image 20231206230317.png" Spring Boot 自动装配工作原理 %}</div>

## 参考文章

- [Creating your own auto-configuration](https://docs.spring.io/spring-boot/docs/1.5.11.RELEASE/reference/html/boot-features-developing-auto-configuration.html)
- [Generating your own meta-data using the annotation processor](https://docs.spring.io/spring-boot/docs/1.5.11.RELEASE/reference/html/configuration-metadata.html#configuration-metadata-annotation-processor)
- {%post_link 'use-and-analysis-of-Import-annotation-in-Spring' 'Spring 中 @Import 注解的使用和源码分析' %}
- {%post_link 'how-does-Spring-Boot-SPI-works' 'Spring Boot SPI 的工作原理' %}