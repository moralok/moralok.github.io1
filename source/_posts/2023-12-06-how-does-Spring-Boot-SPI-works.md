---
title: Spring Boot SPI 的工作原理
date: 2023-12-06 17:27:55
tags: [spring, spring boot, spi]
---

在分析 `Spring Boot` 自动配置的工作原理时，我们并没有深入“如何获得配置在 `spring.factories` 中的自动配置类”。本文将从图解和源码两个角度分析 `Spring Boot SPI` 机制，了解 `spring.factories` 中的配置是如何被加载和解析成为缓存中的“接口-实现”键值对。

<!-- more -->

## 介绍

`Spring Boot SPI` 是一个框架内部使用的工厂加载机制。

`SpringFactoriesLoader` 从 “`META-INF/spring.factories`” 文件加载并实例化给定类型的工厂，这些文件可能存在于类路径下的多个 `JAR`q 文件中。

`spring.factories` 文件必须采用 `Properties` 格式，其中键是接口或抽象类的全限定名称，值是逗号分隔的实现类的全限定名称列表。例如：

```properties
example.MyService=example.MyServiceImpl1,example.MyServiceImpl2
```

## 图解

`loadSpringFactories` 方法加载以及缓存所有工厂是以 `ClassLoader` 为单位的，过程如下：

<div style="width:70%;margin:auto">{% asset_img "Pasted image 20231207024143.png" 加载所有工厂 %}</div>

资源从一个位置名称字符串转换为缓存中的键值对，经历了以下过程。

<div style="width:70%;margin:auto">{% asset_img "Pasted image 20231207024151.png" 从资源位置的名称转换为键值对 %}</div>


缓存的结构示意图

<div style="width:70%;margin:auto">{% asset_img "Pasted image 20231207024134.png" 从资源位置的名称转换为键值对 %}</div>

## 源码分析

```java
public abstract class SpringFactoriesLoader {
    
    /**
	 * 寻找工厂的位置
	 * <p>可以存在于多个 JAR 文件中
	 */
    public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
    
    private static final Log logger = LogFactory.getLog(SpringFactoriesLoader.class);
    
    private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap<>();
    
    /**
     * 使用给定的类加载器从“META-INF/spring.factories”文件加载并实例化给定类型的工厂实现。
     * <p>返回的工厂通过 AnnotationAwareOrderComparator 进行排序。
     * <p>如果需要自定义实例化策略，可以使用 loadFactoryNames 方法获取所有注册的工厂名称。
     */
    public static <T> List<T> loadFactories(Class<T> factoryClass, @Nullable ClassLoader classLoader) {
        Assert.notNull(factoryClass, "'factoryClass' must not be null");
        // 确定使用的 class loader
        ClassLoader classLoaderToUse = classLoader;
        if (classLoaderToUse == null) {
            classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
        }
        // 使用给定的类加载器从“META-INF/spring.factories”文件加载给定类型的工厂实现的全限定名称。
        List<String> factoryNames = loadFactoryNames(factoryClass, classLoaderToUse);
        if (logger.isTraceEnabled()) {
            logger.trace("Loaded [" + factoryClass.getName() + "] names: " + factoryNames);
        }
        List<T> result = new ArrayList<>(factoryNames.size());
        // 遍历工厂实现的全限定名
        for (String factoryName : factoryNames) {
            // 实例化工厂实现
            result.add(instantiateFactory(factoryName, factoryClass, classLoaderToUse));
        }
        // 使用 AnnotationAwareOrderComparator 对结果进行排序
        AnnotationAwareOrderComparator.sort(result);
        return result;
    }
    
    /**
     * 使用给定的类加载器从“META-INF/spring.factories”文件加载给定类型的工厂实现的全限定名称。
     */
    public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
        String factoryClassName = factoryClass.getName();
        return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
    }
    
    /**
     * 使用给定的类加载器从“META-INF/spring.factories”文件加载所有的工厂。
     */
    private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
        // 从缓存中获取
        MultiValueMap<String, String> result = cache.get(classLoader);
        // 如果缓存中已存在，返回结果
        if (result != null)
            return result;
        // 如果缓存中不存在
        try {
            // 从类路径下查找给定名称的所有资源
            Enumeration<URL> urls = (classLoader != null ?
                    classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                    ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
            result = new LinkedMultiValueMap<>();
            // 遍历资源
            while (urls.hasMoreElements()) {
                URL url = urls.nextElement();
                UrlResource resource = new UrlResource(url);
                // 从给定资源中加载属性
                Properties properties = PropertiesLoaderUtils.loadPropertie(resource);
                for (Map.Entry<?, ?> entry : properties.entrySet()) {
                    // 将值按逗号分割成列表
                    List<String> factoryClassNames = Arrays.asList(
                            StringUtils.commaDelimitedListToStringArray((String) entry.getValue()));
                    result.addAll((String) entry.getKey(), factoryClassNames);
                }
            }
            // 放入缓存
            cache.put(classLoader, result);
            return result;
        }
        catch (IOException ex) {
            throw new IllegalArgumentException("Unable to load factories from location [" +
                    FACTORIES_RESOURCE_LOCATION + "]", ex);
        }
    }
    
    @SuppressWarnings("unchecked")
    private static <T> T instantiateFactory(String instanceClassName, Class<T> factoryClass, ClassLoader classLoader) {
        try {
            // 加载类
            Class<?> instanceClass = ClassUtils.forName(instanceClassName, classLoader);
            // 检测 factoryClass 是否和 instanceClass 相同，或者是其超类或超接口
            if (!factoryClass.isAssignableFrom(instanceClass)) {
                throw new IllegalArgumentException(
                        "Class [" + instanceClassName + "] is not assignable to [" + factoryClass.getName() + "]");
            }
            // 通过反射进行实例化
            return (T) ReflectionUtils.accessibleConstructor(instanceClass).newInstance();
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException("Unable to instantiate factory class: " + factoryClass.getName(), ex);
        }
    }
}
```

`spring.factories` 中的键值对并非严格的接口-实现关系，比如 `Spring Boot` 自动配置机制中，`EnableAutoConfiguration` 的值是标准的配置类（被 Configuration 注解标注的类）。因此在实例化方法中，需要检测是否可赋值。

`loadFactories` 方法默认实例化给定类型的所有工厂，如果需要自定义实例化策略，可以通过 loadFactoryNames 方法获取所有注册的工厂。

> `SpringFactoriesLoader` 的源码既简单又不简单，代码较少，逻辑也清晰，但是查找资源并加载部分用到的 `API`，如 `UrlResource`、`MultiMap`，并不大熟悉，感觉很多框架里加载和解析资源部份的代码都不大好学以致用。