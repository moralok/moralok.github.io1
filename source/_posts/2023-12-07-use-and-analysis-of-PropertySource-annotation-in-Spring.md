---
title: Spring 中 @PropertySource 注解的使用和源码分析
date: 2023-12-07 11:55:49
tags: [java, spring]
---

`@PropertySource` 注解提供了一种方便的声明性机制，用于将 `PropertySource` 添加到 `Spring` 容器的 `Environment` 环境中。该注解通常搭配 `@Configuration` 注解一起使用。本文将介绍如何使用 `@PropertySource` 注解，并通过分析源码解释外部配置文件是如何被解析进入 `Spring` 的 `Environment` 中。

<!-- more -->

## 使用方式

`@Configuration` 注解表示这是一个配置类，`Spring` 在处理配置类时，会解析并处理配置类上的 `@PropertySource` 注解，将对应的配置文件解析为 `PropertySource`，添加到 `Spring` 容器的 `Environment` 环境中。这样就可以在其他的 `Bean` 中，使用 `@Value` 注解使用这些配置

```java
@Configuration
@PropertySource(value = "classpath:/player.properties", encoding = "UTF-8")
public class PropertySourceConfig {

    @Bean
    public Player player() {
        return new Player();
    }
}

public class Player {
    private String name;
    private Integer age;
    @Value("${player.nickname}")
    private String nickname;
    // 省略 setter 和 getter 方法
}
```

配置文件

```properties
player.nickname=Tom
```

测试类

```java
public class PropertySourceTest {

    @Test
    public void test() {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(PropertySourceConfig.class);

        Player player = (Player) ac.getBean("player");
        System.out.println(player);

        ConfigurableEnvironment environment = ac.getEnvironment();
        String property = environment.getProperty("player.nickname");
        System.out.println(property);

        ac.close();
    }
}
```

测试结果

```console
Player{name='null', age=null, nickname='Tom'}
Tom
```

## 源码分析

关于 `Spring` 是如何处理配置类的请参见之前的文章：

- {% post_link 'source-code-analysis-of-Spring-Configuration-annotation' 'Spring @Configuration 注解的源码分析' %}

### 获取 @PropertySource 注解属性

`Spring` 在解析配置类构建配置模型时，会对配置类上的 `@PropertySource` 注解进行处理。`Spring` 将获取所有的 `@PropertySource` 注解属性，并遍历进行处理。

- `@PropertySource` 注解是可重复的，一个类上可以标注多个
- `@PropertySources` 注解包含 `@PropertySource` 注解

```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {
    // ...
    // 处理 @PropertySource 注解
    // 获取所有 @PropertySource 注解的属性并遍历。注意该注解为可重复的。
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), PropertySources.class,
            org.springframework.context.annotation.PropertySource.class)) {
        // 如果 environment 是 ConfigurableEnvironment 的一个实例，目前恒为 true
        if (this.environment instanceof ConfigurableEnvironment) {
            // 处理单个 @PropertySource 注解的属性
            processPropertySource(propertySource);
        }
        else {
            logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                    "]. Reason: Environment must implement ConfigurableEnvironment");
        }
    }
    // ...
}
```

使用 `IDEA` 查看 `AnnotationAttributes`：

<div style="width:70%;margin:auto">{% asset_img "Snipaste_2023-12-07_21-12-20.png" IDEA 查看 AnnotationAttributes %}</div>

### 处理 @PropertySource 注解属性

- 读取 `@PropertySource` 注解属性的信息，如名称、编码和位置等等
- 遍历 `location` 查找资源
- 通过 `PropertySourceFactory` 使用资源创建属性源 `PropertySource`
- 将属性源添加到 `Environment`

> 注意属性源 `PropertySource` 不是 `@PropertySource` 注解，而是表示 `name/value` 属性对的源的抽象基类。

```java
private void processPropertySource(AnnotationAttributes propertySource) throws IOException {
    // 属性源的 name，大部分时候不指定
    String name = propertySource.getString("name");
    if (!StringUtils.hasLength(name)) {
        name = null;
    }
    // 编码
    String encoding = propertySource.getString("encoding");
    if (!StringUtils.hasLength(encoding)) {
        encoding = null;
    }
    // 位置
    String[] locations = propertySource.getStringArray("value");
    Assert.isTrue(locations.length > 0, "At least one @PropertySource(value) location is required");
    // 找不到资源时是否忽略，默认 false
    boolean ignoreResourceNotFound = propertySource.getBoolean("ignoreResourceNotFound");
    // 属性源工厂，默认 DefaultPropertySourceFactory
    Class<? extends PropertySourceFactory> factoryClass = propertySource.getClass("factory");
    PropertySourceFactory factory = (factoryClass == PropertySourceFactory.class ?
            DEFAULT_PROPERTY_SOURCE_FACTORY : BeanUtils.instantiateClass(factoryClass));
    // 遍历位置
    for (String location : locations) {
        try {
            // 解析位置，这代表 location 也可以使用占位符
            String resolvedLocation = this.environment.resolveRequiredPlaceholders(location);
            // 查找资源
            Resource resource = this.resourceLoader.getResource(resolvedLocation);
            // 创建属性源，并添加到 Environment
            addPropertySource(factory.createPropertySource(name, new EncodedResource(resource, encoding)));
        }
        catch (IllegalArgumentException ex) {
            // Placeholders not resolvable
            if (ignoreResourceNotFound) {
                if (logger.isInfoEnabled()) {
                    logger.info("Properties location [" + location + "] not resolvable: " + ex.getMessage());
                }
            }
            else {
                throw ex;
            }
        }
        catch (IOException ex) {
            // Resource not found when trying to open it
            if (ignoreResourceNotFound &&
                    (ex instanceof FileNotFoundException || ex instanceof UnknownHostException)) {
                if (logger.isInfoEnabled()) {
                    logger.info("Properties location [" + location + "] not resolvable: " + ex.getMessage());
                }
            }
            else {
                throw ex;
            }
        }
    }
}
```

### 添加属性源到 Environment

将属性源添加到 `Environment` 中有以下几个规则：

- 所有通过 `@PropertySource` 注解加入的属性源，`name` 都会添加到 `propertySourceNames`
- `propertySourceNames` 为空时，代表这是第一个通过 `@PropertySource` 注解加入的属性源，添加到最后（前面有系统属性源）
- `propertySourceNames` 不为空时，添加到上一个添加到 `propertySourceNames` 中的属性源的前面（后来居上）
- 添加到 `propertySources` 的方法中都是先尝试移除，后添加（代表可能有顺序调整，具体场景不知）
- 如果已存在通过 `@PropertySource` 注解加入的属性源，则扩展为 `CompositePropertySource`，里面包含多个同名属性源（后来居上）

```java
private void addPropertySource(PropertySource<?> propertySource) {
    String name = propertySource.getName();
    // 从 Environment 中取出 MutablePropertySources
    MutablePropertySources propertySources = ((ConfigurableEnvironment) this.environment).getPropertySources();
    // 检查环境中是否已存在该属性源并且 propertySourceNames 中不存在
    if (propertySources.contains(name) && this.propertySourceNames.contains(name)) {
        // 如果已经添加过，则扩展
        PropertySource<?> existing = propertySources.get(name);
        PropertySource<?> newSource = (propertySource instanceof ResourcePropertySource ?
                ((ResourcePropertySource) propertySource).withResourceName() : propertySource);
        if (existing instanceof CompositePropertySource) {
            // 如果已经扩展过，添加（addFirst）
            ((CompositePropertySource) existing).addFirstPropertySource(newSource);
        }
        else {
            // 第一次扩展，替换为 CompositePropertySource
            if (existing instanceof ResourcePropertySource) {
                existing = ((ResourcePropertySource) existing).withResourceName();
            }
            CompositePropertySource composite = new CompositePropertySource(name);
            composite.addPropertySource(newSource);
            composite.addPropertySource(existing);
            propertySources.replace(name, composite);
        }
    }
    else {
        // 如果 propertySourceNames 为空，添加到最后
        if (this.propertySourceNames.isEmpty()) {
            propertySources.addLast(propertySource);
        }
        // 如果 propertySourceNames 不为空，添加到上一次添加的属性源的前面
        else {
            String firstProcessed = this.propertySourceNames.get(this.propertySourceNames.size() - 1);
            propertySources.addBefore(firstProcessed, propertySource);
        }
    }
    // 添加到 propertySourceNames
    this.propertySourceNames.add(name);
}
```

可以适当地将添加属性源和使用属性分开看待，`Environment` 是它们产生联系的枢纽，`@PropertySource` 注解的处理过程是 `@Configuration` 注解的处理过程的一部分，在文件中的配置转换成为 `Environment` 中的 `PropertySource` 后，如何使用它们是独立的一件事情。

<div style="width:70%;margin:auto">{% asset_img "Snipaste_2023-12-07_22-49-22.png" Environment 中的 MutablePropertySources %}</div>