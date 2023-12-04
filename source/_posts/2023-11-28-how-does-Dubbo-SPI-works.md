---
title: Dubbo SPI 的工作原理
date: 2023-11-28 16:09:28
tags: [java, dubbo, spi]
---

`SPI` 作为一种服务发现机制，允许程序在运行时动态地加载具体实现类。因其强大的可拓展性，`SPI` 被广泛应用于各类技术框架中，例如 `JDBC` 驱动、`Spring` 和 `Dubbo` 等等。`Dubbo` 并未使用原生的 `Java SPI`，而是重新实现了一套更加强大的 **`Dubbo SPI`**。本文将简单介绍 `SPI` 的设计理念，通过示例带你体会 `SPI` 的作用，通过 **`Dubbo` 获取拓展的流程图**和**源码分析**带你理解 `Dubbo SPI` 的工作原理。深入了解 `Dubbo SPI`，你将能更好地利用这一机制为你的程序提供灵活的拓展功能。

<!-- more -->

## SPI 简介

`SPI` 的全称是 `Service Provider Interface`，是一种服务发现机制。一般情况下，一项服务的接口和具体实现，都是服务提供者编写的。在 `SPI` 机制中，一项服务的接口是服务使用者编写的，不同的服务提供者编写不同的具体实现。在程序运行时，服务加载器动态地为接口加载具体实现类。因为 `SPI` 具备“动态加载”的特性，我们很容易通过它为程序提供拓展功能。以 `Java` 的 `JDBC` 驱动为例，JDK 提供了 `java.sql.Driver` 接口，各个数据库厂商，例如 `MySQL`、`Oracle` 提供具体的实现。

{% asset_img "Pasted image 20231129010505.png" SPI 机制 %}

目前 **SPI 的实现方式**大多是将接口实现类的全限定名配置在文件中，并由服务加载器读取配置文件，加载实现类：

- Java SPI：`META-INF/services/full.qualified.interface.name`
- Dubbo SPI：`META-INF/dubbo/full.qualified.interface.name`（还有其他目录可供选择）
- Spring SPI: `META-INF/spring.factories`

## SPI 示例

### Java SPI 示例

定义一个接口 `Animal`。

```java
public interface Animal {
    void bark();
}
```

定义两个实现类 `Dog` 和 `Cat`。

```java
public class Dog implements Animal {
    @Override
    public void bark() {
        System.out.println("Dog bark...");
    }
}

public class Cat implements Animal {
    @Override
    public void bark() {
        System.out.println("Cat bark...");
    }
}
```

在 **`META-INF/services`** 文件夹下创建一个文件，名称为 `Animal` 的全限定名 `com.moralok.dubbo.spi.test.Animal`，文件内容为实现类的全限定名，实现类的全限定名之间用换行符分隔。

```file
com.moralok.dubbo.spi.test.Dog
com.moralok.dubbo.spi.test.Cat
```

进行测试。

```java
public class JavaSPITest {
    @Test
    void bark() {
        System.out.println("Java SPI");
        System.out.println("============");
        ServiceLoader<Animal> serviceLoader = ServiceLoader.load(Animal.class);
        serviceLoader.forEach(Animal::bark);
    }
}
```

测试结果

```console
Java SPI
============
Dog bark...
Cat bark...
```

### Dubbo SPI 示例

`Dubbo` 并未使用原生的 `Java SPI`，而是重新实现了一套功能更加强大的 `SPI` 机制。`Dubbo SPI` 的配置文件放在 **`META-INF/dubbo`** 文件夹下，名称仍然是接口的全限定名，但是内容是“名称->实现类的全限定名”的键值对，另外接口需要标注 `SPI` 注解。

```file
dog = com.moralok.dubbo.spi.test.Dog
cat = com.moralok.dubbo.spi.test.Cat
```

进行测试。

```java
public class DubboSPITest {

    @Test
    void bark() {
        System.out.println("Dubbo SPI");
        System.out.println("============");
        ExtensionLoader<Animal> extensionLoader = ExtensionLoader.getExtensionLoader(Animal.class);
        Animal dog = extensionLoader.getExtension("dog");
        dog.bark();
        Animal cat = extensionLoader.getExtension("cat");
        cat.bark();
    }
}
```

测试结果

```console
Dubbo SPI
============
Dog bark...
Cat bark...
```

## Dubbo 获取扩展流程图

{% asset_img "Pasted image 20231129204920.png" Dubbo 获取扩展流程图 %}

## Dubbo SPI 源码分析

### 获取 ExtensionLoader

```java
private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<>(64);

public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null) {
        throw new IllegalArgumentException("Extension type == null");
    }
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
    }
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type (" + type +
                ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
    }

    // 从缓存中获取，如果缓存未命中，则创建，保存到缓存并返回
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```

这个方法包含了如下步骤：

1. 参数校验。
2. 从缓存 `EXTENSION_LOADERS` 中获取与拓展类对应的 `ExtensionLoader`，如果缓存未命中，则创建一个新的实例，保存到缓存并返回。

> “**从缓存中获取，如果缓存未命中，则创建，保存到缓存并返回**”，类似的 `getOrCreate` 的处理模式在 `Dubbo` 的源码中经常出现。

`EXTENSION_LOADERS` 是 `ExtensionLoader` 的静态变量，保存了“拓展类->`ExtensionLoader`”的映射关系。

### 根据 name 获取 Extension

```java
public T getExtension(String name) {
    return getExtension(name, true);
}

public T getExtension(String name, boolean wrap) {
    if (StringUtils.isEmpty(name)) {
        throw new IllegalArgumentException("Extension name == null");
    }
    if ("true".equals(name)) {
        // 获取默认的拓展实现
        return getDefaultExtension();
    }
    // Holder，用于持有目标对象
    final Holder<Object> holder = getOrCreateHolder(name);
    // 双重检查
    Object instance = holder.get();
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                // 创建拓展实例，设置到 holder 中。
                instance = createExtension(name, wrap);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}

private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<>();

private Holder<Object> getOrCreateHolder(String name) {
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<>());
        holder = cachedInstances.get(name);
    }
    return holder;
}
```

这个方法中获取 `Holder` 和获取拓展实例都是使用 `getOrCreate` 的模式。

`Holder` 用于持有拓展实例。`cachedInstances` 是 `ExtensionLoader` 的成员变量，保存了“`name->Holder`(拓展实例)”的映射关系。

### 创建 Extension

```java
private T createExtension(String name, boolean wrap) {
    // 从配置文件中加载所有的拓展类，可得到“name->拓展实现类”的映射关系表
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null || unacceptableExceptions.contains(name)) {
        throw findException(name);
    }
    try {
        // 使用 getOrCreate 模式获取拓展实例
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.getDeclaredConstructor().newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // 向拓展实例中注入依赖
        injectExtension(instance);

        // 是否包装默认为 true
        if (wrap) {

            List<Class<?>> wrapperClassesList = new ArrayList<>();
            if (cachedWrapperClasses != null) {
                wrapperClassesList.addAll(cachedWrapperClasses);
                wrapperClassesList.sort(WrapperComparator.COMPARATOR);
                Collections.reverse(wrapperClassesList);
            }

            if (CollectionUtils.isNotEmpty(wrapperClassesList)) {
                // 遍历包装类
                for (Class<?> wrapperClass : wrapperClassesList) {
                    Wrapper wrapper = wrapperClass.getAnnotation(Wrapper.class);
                    // 区别于旧版本：支持使用 Wrapper 注解进行匹配
                    if (wrapper == null
                            || (ArrayUtils.contains(wrapper.matches(), name) && !ArrayUtils.contains(wrapper.mismatches(), name))) {
                        // 如果有匹配的包装类，包装拓展实例
                        instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                    }
                }
            }
        }

        // 初始化拓展实例
        initExtension(instance);
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                type + ") couldn't be instantiated: " + t.getMessage(), t);
    }
}
```

这个方法包含如下步骤：

1. 通过 `getExtensionClasses` 获取所有拓展类
2. 通过反射创建拓展实例
3. 向拓展实例中注入依赖
4. 将拓展实例包装在适配的 `Wrapper` 对象中
5. 初始化拓展实例

第一步是加载拓展类的关键，第三步和第四步是 **`Dubbo IOC`** 和 **`AOP`** 的具体实现。

最后拓展实例的结构如下图。

{% asset_img "Pasted image 20231129024723.png" 拓展实例被包装后的结构图 %}

### 加载 Extension Class

```java
private Map<String, Class<?>> getExtensionClasses() {
    // 使用 getOrCreate 模式获取所有拓展类
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}

private Map<String, Class<?>> loadExtensionClasses() {
    // 1. 缓存默认拓展名
    cacheDefaultExtensionName();

    Map<String, Class<?>> extensionClasses = new HashMap<>();

    // 遍历加载策略，加载各个策略的目录下的配置文件，获取拓展类
    for (LoadingStrategy strategy : strategies) {
        loadDirectory(extensionClasses, strategy.directory(), type.getName(), strategy.preferExtensionClassLoader(),
                strategy.overridden(), strategy.excludedPackages());
        loadDirectory(extensionClasses, strategy.directory(), type.getName().replace("org.apache", "com.alibaba"),
                strategy.preferExtensionClassLoader(), strategy.overridden(), strategy.excludedPackages());
    }

    return extensionClasses;
}

// 如果存在默认拓展名，提取并缓存
private void cacheDefaultExtensionName() {
    // 获取 SPI 注解
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if (defaultAnnotation == null) {
        return;
    }

    String value = defaultAnnotation.value();
    if ((value = value.trim()).length() > 0) {
        // 对 SPI 的 value 内容进行切分
        String[] names = NAME_SEPARATOR.split(value);
        // 检测 name 是否合法
        if (names.length > 1) {
            throw new IllegalStateException("More than 1 default extension name on extension " + type.getName()
                    + ": " + Arrays.toString(names));
        }
        if (names.length == 1) {
            // 缓存默认拓展名，用于 getDefaultExtension
            cachedDefaultName = names[0];
        }
    }
}
```

#### 依次处理特定目录

代码参考旧版本更容易理解。处理过程在本质上就是依次加载 `META-INF/dubbo/internal/`、`META-INF/dubbo/`、`META-INF/services/` 三个目录下的配置文件，获取拓展类。

```java
loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
loadDirectory(extensionClasses, DUBBO_DIRECTORY);
loadDirectory(extensionClasses, SERVICES_DIRECTORY);
```

新版本使用原生的 `Java SPI` 加载 `LoadingStrategy`，允许用户自定义加载策略。

1. `DubboInternalLoadingStrategy`，目录 `META-INF/dubbo/internal/`，优先级最高
2. `DubboLoadingStrategy`，目录 `META-INF/dubbo/`，优先级普通
3. `ServicesLoadingStrategy`，目录 `META-INF/services/`，优先级最低

```java
private static volatile LoadingStrategy[] strategies = loadLoadingStrategies();

private static LoadingStrategy[] loadLoadingStrategies() {
    // 通过 Java SPI 加载 LoadingStrategy 
    return stream(load(LoadingStrategy.class).spliterator(), false)
            .sorted()
            .toArray(LoadingStrategy[]::new);
}
```

`LoadingStrategy` 的 `Java SPI` 配置文件

{% asset_img "Snipaste_2023-11-29_03-25-54.png" LoadingStrategy 的 Java SPI 配置文件 %}

#### loadDirectory 方法

`loadDirectory` 方法先通过 `classLoader` 获取所有的资源链接，然后再通过 `loadResource` 方法加载资源。

新版本中 `extensionLoaderClassLoaderFirst` 可以设置是否优先使用 `ExtensionLoader's ClassLoader` 获取资源链接。

```java
private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir, String type,
                            boolean extensionLoaderClassLoaderFirst, boolean overridden, String... excludedPackages) {
    // filename = 文件夹路径 + type 的全限定名
    String fileName = dir + type;
    try {
        Enumeration<java.net.URL> urls = null;
        ClassLoader classLoader = findClassLoader();

        // 区别于旧版本：先从 ExtensionLoader's ClassLoader 获取资源链接，默认为 false
        if (extensionLoaderClassLoaderFirst) {
            ClassLoader extensionLoaderClassLoader = ExtensionLoader.class.getClassLoader();
            if (ClassLoader.getSystemClassLoader() != extensionLoaderClassLoader) {
                urls = extensionLoaderClassLoader.getResources(fileName);
            }
        }

        // 根据文件名加载所有同名文件
        if (urls == null || !urls.hasMoreElements()) {
            if (classLoader != null) {
                urls = classLoader.getResources(fileName);
            } else {
                urls = ClassLoader.getSystemResources(fileName);
            }
        }

        if (urls != null) {
            while (urls.hasMoreElements()) {
                java.net.URL resourceURL = urls.nextElement();
                // 加载资源
                loadResource(extensionClasses, classLoader, resourceURL, overridden, excludedPackages);
            }
        }
    } catch (Throwable t) {
        logger.error("Exception occurred when loading extension class (interface: " +
                type + ", description file: " + fileName + ").", t);
    }
}
```

#### loadResource 方法

`loadResource` 方法用于读取和解析配置文件，并通过反射加载类，最后调用 `loadClass` 方法进行其他操作。`loadClass` 方法用于操作缓存。

新版本中 `excludedPackages` 可以设置将指定包内的类都排除。

```java
private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader,
                            java.net.URL resourceURL, boolean overridden, String... excludedPackages) {
    try {
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), StandardCharsets.UTF_8))) {
            String line;
            String clazz = null;
            // 按行读取配置内容
            while ((line = reader.readLine()) != null) {
                // 定位 # 字符，截取 # 字符之前的内容，# 字符之后的内容为注释，需要忽略
                final int ci = line.indexOf('#');
                if (ci >= 0) {
                    line = line.substring(0, ci);
                }
                line = line.trim();
                if (line.length() > 0) {
                    try {
                        String name = null;
                        // 定位 = 字符，以 = 字符为界，截取键值对
                        int i = line.indexOf('=');
                        if (i > 0) {
                            name = line.substring(0, i).trim();
                            clazz = line.substring(i + 1).trim();
                        } else {
                            clazz = line;
                        }
                        if (StringUtils.isNotEmpty(clazz) && !isExcluded(clazz, excludedPackages)) {
                            // 加载类，并通过 loadClass 进行缓存
                            // 区别于旧版本：根据 excludedPackages 判断是否排除
                            loadClass(extensionClasses, resourceURL, Class.forName(clazz, true, classLoader), name, overridden);
                        }
                    } catch (Throwable t) {
                        IllegalStateException e = new IllegalStateException(
                                "Failed to load extension class (interface: " + type + ", class line: " + line + ") in " + resourceURL +
                                        ", cause: " + t.getMessage(), t);
                        exceptions.put(line, e);
                    }
                }
            }
        }
    } catch (Throwable t) {
        logger.error("Exception occurred when loading extension class (interface: " +
                type + ", class file: " + resourceURL + ") in " + resourceURL, t);
    }
}
```

#### loadClass 方法

`loadClass` 方法设置了多个缓存，比如 `cachedAdaptiveClass`、`cachedWrapperClasses`、`cachedNames` 和 `cachedClasses`。

新版本中 `overridden` 可以设置是否覆盖 `cachedAdaptiveClass`、`cachedClasses` 的 `name->clazz`。

```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name,
                        boolean overridden) throws NoSuchMethodException {
    // 检测 clazz 是否合法
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("Error occurred when loading extension class (interface: " +
                type + ", class line: " + clazz.getName() + "), class "
                + clazz.getName() + " is not subtype of interface.");
    }
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        // 检测 clazz 是否有 Adaptive 注解，有则设置 cachedAdaptiveClass 缓存
        // 区别于旧版本：根据 overriden 判断是否可覆盖
        cacheAdaptiveClass(clazz, overridden);
    } else if (isWrapperClass(clazz)) {
        // 检测 clazz 是否是 Wrapper 类型，是则添加到 cachedWrapperClasses 缓存
        cacheWrapperClass(clazz);
    } else {
        // 检测 clazz 是否有默认的构造器方法，如果没有，则抛出异常
        clazz.getConstructor();
        if (StringUtils.isEmpty(name)) {
            // 如果 name 为空，则尝试从 Extension 注解中获取 name，或者使用小写的类名（可能截取 type 后缀）作为 name
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                throw new IllegalStateException(
                        "No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
            }
        }

        // 切分 name
        String[] names = NAME_SEPARATOR.split(name);
        if (ArrayUtils.isNotEmpty(names)) {
            // 如果 clazz 有 Activate 注解，则缓存 names[0]->Activate 的映射关系
            cacheActivateClass(clazz, names[0]);
            for (String n : names) {
                // 缓存 clazz->n 的映射关系
                cacheName(clazz, n);
                // 缓存 n->clazz 的映射关系，传递到最后，终于轮到 extensionClasses
                // 区别于旧版本：根据 overriden 判断是否可覆盖
                saveInExtensionClass(extensionClasses, clazz, n, overridden);
            }
        }
    }
}

private void cacheAdaptiveClass(Class<?> clazz, boolean overridden) {
    if (cachedAdaptiveClass == null || overridden) {
        // 可以设置是否覆盖
        cachedAdaptiveClass = clazz;
    } else if (!cachedAdaptiveClass.equals(clazz)) {
        throw new IllegalStateException("More than 1 adaptive class found: "
                + cachedAdaptiveClass.getName()
                + ", " + clazz.getName());
    }
}

private void cacheWrapperClass(Class<?> clazz) {
    if (cachedWrapperClasses == null) {
        cachedWrapperClasses = new ConcurrentHashSet<>();
    }
    cachedWrapperClasses.add(clazz);
}

private void cacheActivateClass(Class<?> clazz, String name) {
    Activate activate = clazz.getAnnotation(Activate.class);
    if (activate != null) {
        cachedActivates.put(name, activate);
    } else {
        // support com.alibaba.dubbo.common.extension.Activate
        com.alibaba.dubbo.common.extension.Activate oldActivate =
                clazz.getAnnotation(com.alibaba.dubbo.common.extension.Activate.class);
        if (oldActivate != null) {
            cachedActivates.put(name, oldActivate);
        }
    }
}
```

### Dubbo IOC

`Dubbo IOC` 是通过 `setter` 方法注入依赖。`Dubbo` 首先通过反射获取目标类的所有方法，然后遍历方法列表，检测方法名是否具有 `setter` 方法特征并满足条件，若有，则通过 `objectFactory` 获取依赖对象，最后通过反射调用 `setter` 方法将依赖设置到目标对象中。

> 与 `Spring IOC` 相比，`Dubbo IOC` 实现的依赖注入功能更加简单，代码也更加容易理解。

```java
private T injectExtension(T instance) {
    // 检测是否有 objectFactory
    if (objectFactory == null) {
        return instance;
    }
    try {
        // 遍历目标类的所有方法
        // 区别于旧版本：增加了对 DisbaleInject、Inject 注解的处理
        for (Method method : instance.getClass().getMethods()) {
            // 检测是否是 setter 方法
            if (!isSetter(method)) {
                continue;
            }

            // 检测是否标注 DisableInject 注解
            if (method.getAnnotation(DisableInject.class) != null) {
                continue;
            }

            // 检测参数类型是否是原始类型
            Class<?> pt = method.getParameterTypes()[0];
            if (ReflectUtils.isPrimitives(pt)) {
                continue;
            }

            // 获取属性名
            String property = getSetterProperty(method);
            // 检测是否标注 Inject 注解
            Inject inject = method.getAnnotation(Inject.class);
            if (inject == null) {
                injectValue(instance, method, pt, property);
            } else {
                // 检测 Inject 是否启动、是否按照类型注入
                if (!inject.enable()) {
                    continue;
                }

                if (inject.type() == Inject.InjectType.ByType) {
                    injectValue(instance, method, pt, null);
                } else {
                    injectValue(instance, method, pt, property);
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}

// 获取属性名
private String getSetterProperty(Method method) {
    return method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
}

// public、set 开头、只有一个参数
private boolean isSetter(Method method) {
    return method.getName().startsWith("set")
            && method.getParameterTypes().length == 1
            && Modifier.isPublic(method.getModifiers());
}

private void injectValue(T instance, Method method, Class<?> pt, String property) {
    try {
        // 从 ObjectFactory 中获取依赖对象
        Object object = objectFactory.getExtension(pt, property);
        if (object != null) {
            // 通过反射调用 setter 方法设置依赖
            method.invoke(instance, object);
        }
    } catch (Exception e) {
        logger.error("Failed to inject via method " + method.getName()
                + " of interface " + type.getName() + ": " + e.getMessage(), e);
    }
}
```

`objectFactory` 是 `ExtensionFactory` 的自适应拓展，通过它获取依赖对象，本质上是根据目标拓展类获取 `ExtensionLoader`，然后获取其自适应拓展，过程代码如下。具体我们不再深入分析，可以参考{% post_link 'how-does-Dubbo-SPI-adaptive-extension-works' 'Dubbo SPI 自适应拓展的工作原理' %}。

```java
public <T> T getExtension(Class<T> type, String name) {
    if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
        ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
        if (!loader.getSupportedExtensions().isEmpty()) {
            return loader.getAdaptiveExtension();
        }
    }
    return null;
}
```

## 参考文章

- [Dubbo SPI](https://cn.dubbo.apache.org/zh-cn/docsv2.7/dev/source/dubbo-spi/)