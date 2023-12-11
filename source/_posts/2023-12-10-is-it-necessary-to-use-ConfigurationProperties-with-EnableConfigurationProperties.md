---
title: ConfigurationProperties 一定要搭配 EnableConfigurationProperties 使用吗
date: 2023-12-10 11:41:00
tags: [java, spring, spring boot]
---

`@ConfigurationProperties` 和 `@EnableConfigurationProperties` 是 `Spring Boot` 中常用的注解，提供了方便和强大的外部化配置支持。尽管它们常常一起出现，但是它们真的必须一起使用吗？`Spring Boot` 的灵活性常常让我们忽略配置背后产生的作用究竟是什么？本文将从源码角度出发分析两个注解的**作用时机**和**工作原理**。

<!-- more -->

- 本文的写作动机继承自{% post_link 'use-and-analysis-of-PropertySource-annotation-in-Spring' Spring 中 @PropertySource 注解的使用和源码分析 %}，两者有点相似并且常被一起提及，都通过外部配置管理运行时的属性值，但实际的工作原理却并不相同。
- 本文没有介绍它们的使用方式，如有需要可以参考 [Guide to @ConfigurationProperties in Spring Boot](https://www.baeldung.com/configuration-properties-in-spring-boot)。
- 理解 `@Import` 的工作原理对阅读本文的源码有非常大的帮助，可以参考{% post_link 'use-and-analysis-of-Import-annotation-in-Spring' Spring 中 @Import 注解的使用和源码分析 %}。

## 注解

`ConfigurationProperties` 是用于**外部化配置**的注解。如果你想**绑定**和**验证**某些外部属性（例如来自 `.properties` 文件），就将其添加到**类定义或 `@Configuration` 类中的 `@Bean` 方法**。请注意，和 `@Value` 相反，`SpEL` 表达式不会被求值，因为属性值是外部化的。查看 `ConfigurationProperties` 注解的源码可知，该注解主要起到标记和存储一些信息的作用。

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ConfigurationProperties {

    // 可有效绑定到此对象的属性的名称前缀
    @AliasFor("prefix")
    String value() default "";

    // 可有效绑定到此对象的属性的名称前缀
    @AliasFor("value")
    String prefix() default "";

    // 绑定到此对象时是否忽略无效字段
    boolean ignoreInvalidFields() default false;

    // 绑定到此对象时是否忽略未知字段
    boolean ignoreUnknownFields() default true;

}
```

查看 `EnableConfigurationProperties` 的源码，我们注意到它通过 `@Import` 导入了 `EnableConfigurationPropertiesImportSelector`。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EnableConfigurationPropertiesImportSelector.class)
public @interface EnableConfigurationProperties {

    // 使用 Spring 快速注册标注了 @ConfigurationProperties 的 bean。无论 value 如何，标准的 Spring Bean 也将被扫描。
    Class<?>[] value() default {};

}
```

## 注解的作用

查看 `EnableConfigurationPropertiesImportSelector` 的源码，关注 `selectImports` 方法。该方法返回了 `ConfigurationPropertiesBeanRegistrar` 和 `ConfigurationPropertiesBindingPostProcessorRegistrar` 的全限定类名，`Spring` 将注册它们。

```java
class EnableConfigurationPropertiesImportSelector implements ImportSelector {

    private static final String[] IMPORTS = {
            ConfigurationPropertiesBeanRegistrar.class.getName(),
            ConfigurationPropertiesBindingPostProcessorRegistrar.class.getName() };

    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        return IMPORTS;
    }
}
```

### 注册目标类

`ConfigurationPropertiesBeanRegistrar` 是一个内部类，查看 `ConfigurationPropertiesBeanRegistrar` 的源码，关注 `registerBeanDefinitions` 方法。注册的目标来自于：

- `@EnableConfigurationProperties` 的 `value` 所指定的类中
- 且标注了 `@ConfigurationProperties` 的类

```java
public static class ConfigurationPropertiesBeanRegistrar
        implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,
            BeanDefinitionRegistry registry) {
        // 1. 获取要注册的 Class
        // 2. 遍历获取的结果进行注册
        getTypes(metadata).forEach((type) -> register(registry,
                (ConfigurableListableBeanFactory) registry, type));
    }


    private List<Class<?>> getTypes(AnnotationMetadata metadata) {
        // 获取所有 @EnableConfigurationProperties 的属性
        MultiValueMap<String, Object> attributes = metadata
                .getAllAnnotationAttributes(
                        EnableConfigurationProperties.class.getName(), false);
        // 获取属性 value 的值，处理后返回
        return collectClasses(attributes == null ? Collections.emptyList()
                : attributes.get("value"));
    }

    // 收集 Class
    private List<Class<?>> collectClasses(List<?> values) {
        return values.stream().flatMap((value) -> Arrays.stream((Object[]) value))
                .map((o) -> (Class<?>) o).filter((type) -> void.class != type)
                .collect(Collectors.toList());
    }

    // 注册 Bean 定义
    private void register(BeanDefinitionRegistry registry,
            ConfigurableListableBeanFactory beanFactory, Class<?> type) {
        String name = getName(type);
        // 检测是否包含 Bean 定义
        if (!containsBeanDefinition(beanFactory, name)) {
            // 如果没找到，则注册
            registerBeanDefinition(registry, name, type);
        }
    }

    // beanName = prefix + 全限定类名 or 全限定类名
    private String getName(Class<?> type) {
        ConfigurationProperties annotation = AnnotationUtils.findAnnotation(type,
                ConfigurationProperties.class);
        String prefix = (annotation != null ? annotation.prefix() : "");
        return (StringUtils.hasText(prefix) ? prefix + "-" + type.getName()
                : type.getName());
    }

    // 检测是否包含 Bean 定义
    private boolean containsBeanDefinition(
            ConfigurableListableBeanFactory beanFactory, String name) {
        // 先检测当前工厂中是否包含
        if (beanFactory.containsBeanDefinition(name)) {
            return true;
        }
        // 如果没找到则检测父工厂是否包含
        BeanFactory parent = beanFactory.getParentBeanFactory();
        if (parent instanceof ConfigurableListableBeanFactory) {
            return containsBeanDefinition((ConfigurableListableBeanFactory) parent,
                    name);
        }
        return false;
    }

    // 注册 Bean 定义
    private void registerBeanDefinition(BeanDefinitionRegistry registry, String name,
            Class<?> type) {
        // 断言目标类标注了 @ConfigurationProperties
        assertHasAnnotation(type);
        GenericBeanDefinition definition = new GenericBeanDefinition();
        definition.setBeanClass(type);
        // registry 注册 Bean 定义
        registry.registerBeanDefinition(name, definition);
    }

    // 断言目标类标注了 @ConfigurationProperties
    private void assertHasAnnotation(Class<?> type) {
        Assert.notNull(
                AnnotationUtils.findAnnotation(type, ConfigurationProperties.class),
                "No " + ConfigurationProperties.class.getSimpleName()
                        + " annotation found on  '" + type.getName() + "'.");
    }

}
```

### 注册后处理器

查看 `ConfigurationPropertiesBindingPostProcessorRegistrar` 的源码，关注 `registerBeanDefinitions` 方法。该方法注册了 `ConfigurationPropertiesBindingPostProcessor` 和 `ConfigurationBeanFactoryMetadata`。

- 前者顾名思义，用于处理 `ConfigurationProperties` 的绑定
- 后者是用于在 `Bean` 工厂初始化期间记住 `@Bean` 定义元数据的实用程序类

```java
public class ConfigurationPropertiesBindingPostProcessorRegistrar
        implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
            BeanDefinitionRegistry registry) {
        if (!registry.containsBeanDefinition(
                ConfigurationPropertiesBindingPostProcessor.BEAN_NAME)) {
            registerConfigurationPropertiesBindingPostProcessor(registry);
            registerConfigurationBeanFactoryMetadata(registry);
        }
    }

    private void registerConfigurationPropertiesBindingPostProcessor(
            BeanDefinitionRegistry registry) {
        GenericBeanDefinition definition = new GenericBeanDefinition();
        definition.setBeanClass(ConfigurationPropertiesBindingPostProcessor.class);
        // Bean 定义的角色为基础设施
        definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        registry.registerBeanDefinition(
                ConfigurationPropertiesBindingPostProcessor.BEAN_NAME, definition);

    }

    private void registerConfigurationBeanFactoryMetadata(
            BeanDefinitionRegistry registry) {
        GenericBeanDefinition definition = new GenericBeanDefinition();
        definition.setBeanClass(ConfigurationBeanFactoryMetadata.class);
        definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        registry.registerBeanDefinition(ConfigurationBeanFactoryMetadata.BEAN_NAME,
                definition);
    }

}
```

## 绑定

`ConfigurationPropertiesBindingPostProcessor` 是用于 `ConfigurationProperties` 绑定的后处理器，关注 `afterPropertiesSet` 方法还有核心方法 `postProcessBeforeInitialization`。

- 在 `afterPropertiesSet` 方法中，它获取到了和自己一起注册的 `ConfigurationBeanFactoryMetadata`。
- 在 `postProcessBeforeInitialization` 方法中，先获取 `@ConfigurationProperties`，再进行绑定。

```java
public class ConfigurationPropertiesBindingPostProcessor implements BeanPostProcessor,
        PriorityOrdered, ApplicationContextAware, InitializingBean {

    public static final String BEAN_NAME = ConfigurationPropertiesBindingPostProcessor.class
            .getName();

    public static final String VALIDATOR_BEAN_NAME = "configurationPropertiesValidator";

    private ConfigurationBeanFactoryMetadata beanFactoryMetadata;

    private ApplicationContext applicationContext;

    private ConfigurationPropertiesBinder configurationPropertiesBinder;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext)
            throws BeansException {
        this.applicationContext = applicationContext;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        // We can't use constructor injection of the application context because
        // it causes eager factory bean initialization
        // 注入了 ConfigurationBeanFactoryMetadata（不理解注解描述的情况？）
        this.beanFactoryMetadata = this.applicationContext.getBean(
                ConfigurationBeanFactoryMetadata.BEAN_NAME,
                ConfigurationBeanFactoryMetadata.class);
        this.configurationPropertiesBinder = new ConfigurationPropertiesBinder(
                this.applicationContext, VALIDATOR_BEAN_NAME);
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE + 1;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)
            throws BeansException {
        // 获取 @ConfigurationProperties
        ConfigurationProperties annotation = getAnnotation(bean, beanName,
                ConfigurationProperties.class);
        if (annotation != null) {
            // 绑定
            bind(bean, beanName, annotation);
        }
        return bean;
    }

    private void bind(Object bean, String beanName, ConfigurationProperties annotation) {
        // 获取 Bean 的类型
        ResolvableType type = getBeanType(bean, beanName);
        // 获取 @Validated 用于校验
        Validated validated = getAnnotation(bean, beanName, Validated.class);
        Annotation[] annotations = (validated == null ? new Annotation[] { annotation }
                : new Annotation[] { annotation, validated });
        // 创建 Bindable 对象
        Bindable<?> target = Bindable.of(type).withExistingValue(bean)
                .withAnnotations(annotations);
        try {
            // 绑定
            this.configurationPropertiesBinder.bind(target);
        }
        catch (Exception ex) {
            throw new ConfigurationPropertiesBindException(beanName, bean, annotation,
                    ex);
        }
    }

    private ResolvableType getBeanType(Object bean, String beanName) {
        // 先查找工厂方法
        Method factoryMethod = this.beanFactoryMetadata.findFactoryMethod(beanName);
        if (factoryMethod != null) {
            // 如果存在，返回工厂方法的返回值类型
            return ResolvableType.forMethodReturnType(factoryMethod);
        }
        // 否则返回 bean 的类型（难道 bean 的类型不是工厂方法的返回值类型吗？）
        return ResolvableType.forClass(bean.getClass());
    }

    private <A extends Annotation> A getAnnotation(Object bean, String beanName,
            Class<A> type) {
        // 先到 @Bean 方法中获取
        A annotation = this.beanFactoryMetadata.findFactoryAnnotation(beanName, type);
        if (annotation == null) {
            // 如果没有，再到类上获取
            annotation = AnnotationUtils.findAnnotation(bean.getClass(), type);
        }
        return annotation;
    }

}
```

`ConfigurationBeanFactoryMetadata` 是用于在 `Bean` 工厂初始化期间记住 `@Bean` 定义元数据的实用程序类。在前面我们介绍过 `@ConfigurationProperties` 不仅可以添加到类定义，还可以用于标注 `@Bean` 方法，`ConfigurationBeanFactoryMetadata` 正是应用于在后者这类情况下获取 `@ConfigurationProperties`。

```java
public class ConfigurationBeanFactoryMetadata implements BeanFactoryPostProcessor {

    private final Map<String, FactoryMetadata> beansFactoryMetadata = new HashMap<>();

    // 后处理 BeanFactory
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
            throws BeansException {
        this.beanFactory = beanFactory;
        // 遍历 Bean 定义
        for (String name : beanFactory.getBeanDefinitionNames()) {
            BeanDefinition definition = beanFactory.getBeanDefinition(name);
            String method = definition.getFactoryMethodName();
            String bean = definition.getFactoryBeanName();、
            // 检测是否是一个 @Bean 方法对应的 Bean 定义
            if (method != null && bean != null) {
                this.beansFactoryMetadata.put(name, new FactoryMetadata(bean, method));
            }
        }
    }

    // 获取工厂方法上的注解
    public <A extends Annotation> A findFactoryAnnotation(String beanName,
			Class<A> type) {
		Method method = findFactoryMethod(beanName);
		return (method == null ? null : AnnotationUtils.findAnnotation(method, type));
	}

    // 获取工厂方法
	public Method findFactoryMethod(String beanName) {
		if (!this.beansFactoryMetadata.containsKey(beanName)) {
			return null;
		}
		AtomicReference<Method> found = new AtomicReference<>(null);
		FactoryMetadata metadata = this.beansFactoryMetadata.get(beanName);
		Class<?> factoryType = this.beanFactory.getType(metadata.getBean());
		String factoryMethod = metadata.getMethod();
        // 如果是代理类，获取其父类
		if (ClassUtils.isCglibProxyClass(factoryType)) {
			factoryType = factoryType.getSuperclass();
		}
        // 遍历声明的方法进行匹配（名字相同即可没问题吗？）
		ReflectionUtils.doWithMethods(factoryType, (method) -> {
			if (method.getName().equals(factoryMethod)) {
				found.compareAndSet(null, method);
			}
		});
		return found.get();
	}
}
```

## 总结

`@EnableConfigurationProperties` 的目的有两个：

- 注册目标
- 注册后处理器用于在目标进行 `Bean` 初始化工作时，介入进行绑定

尽管注册目标时的操作有些巧妙，但是还是要明白 `ConfigurationProperties` 类只是单纯的被注册了而已。对于后处理器而言，无论一个 `ConfigurationProperties` 类是不是通过注解注册，后处理器都会一视同仁地进行绑定。但同时，你又要知道后处理器也是通过 `@EnableConfigurationProperties` 注册的，因此你需要保证至少有一个 `@EnableConfigurationProperties` 标注的类被注册（并被处理了 `@Import`）。
在 `Spring Boot` 中，`@SpringBootApplication` 通过 `@EnableAutoConfiguration` 启用了自动配置，从而注册了 `ConfigurationPropertiesAutoConfiguration`，`ConfigurationPropertiesAutoConfiguration` 标注了 `@EnableConfigurationProperties`。因此，对于 `Spring Boot` 而言，扫描范围内的所有 `ConfigurationProperties` 类，其实都不需要 `@EnableAutoConfiguration`。事实上，由于默认生成的 `beanName` 不同，多余的配置还会重复注册两个 `Bean` 定义。

```java
@Configuration
@EnableConfigurationProperties
public class ConfigurationPropertiesAutoConfiguration {

}
```

