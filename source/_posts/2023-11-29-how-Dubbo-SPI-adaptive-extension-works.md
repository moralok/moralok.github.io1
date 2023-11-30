---
title: Dubbo SPI 自适应拓展的工作原理
date: 2023-11-29 13:40:07
tags: [java, dubbo, spi]
---

直接展示一个具体的 `Dubbo SPI` 自适应拓展是什么样子，是一种非常好的表现其作用的方式。正如官方博客中所说的，它让人对自适应拓展有更加感性的认识，避免读者一开始就陷入复杂的代码生成逻辑。本文在此基础上，从更原始的使用方式上展现“动态加载”技术对“按需加载”的天然倾向，从更普遍的角度解释自适应拓展的本质目的，在介绍 `Dubbo` 的具体实现是如何约束自身从而规避缺点之后，详细梳理了 `Dubbo SPI` 自适应拓展的相关源码和工作原理。

<!-- more -->

> 站在现有设计回头看的视角更偏向于展现为什么这样设计很好，却并不好展现如果不这样设计会有什么问题，以至于有时候会有种这个设计很妙，但妙在哪里体会不够深的感觉。思考一项技术如何从最初发展到现在，解决以及试图解决哪些问题，因此可能引入哪些问题，也许脑补的并不完全符合历史事实，但仍然会让人更加深刻地认识这项技术本身，体会设计中的巧思，并避免一直陷在庞杂的细节处理中。

## 原理

在 `Dubbo` 中，很多拓展都是通过 `SPI` 机制动态加载的，比如 `Protocol`、`Cluster` 和 `LoadBalance` 等。有些拓展我们并不想在框架启动阶段被加载，而是希望在拓展方法被调用时，根据运行时参数进行加载。为了让大家对自适应拓展有一个感性的认识，下面我们通过一个实例进行演示。

### 示例

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

在运行时根据参数动态地加载拓展。

```java
public void bark(String type) {
    if (type == null) {
        throw new IllegalArgumentException("type == null");
    }
    // 通过 SPI 动态地加载具体的 Animal
    Animal animal = ExtensionLoader.getExtensionLoader(Animal.class).getExtension(type);
    // 调用目标方法
    animal.bark();
}
```

### 改进

是不是感觉平平无奇？没错，当你拥有动态加载的能力后，按需加载是自然而然会产生的想法，并不是什么高大上的设计。两者甚至不仅仅是天性相合，可能更像是你中有我，我中有你。在正常场景中，这样一段代码也并不需要进一步被抽象和重构，它本身就很简洁。现在设想一下，你的应用中，有大量的拓展需要动态加载，你可能需要在很多地方写很多根据运行时参数动态加载拓展并调用方法的代码，就像下面这样：

```java
Animal animal = ExtensionLoader.getExtensionLoader(Animal.class).getExtension(type);
animal.bark();

WeelMaker weelMaker = ExtensionLoader.getExtensionLoader(WeelMaker.class).getExtension(weelMakerName);
weelMaker.makeWeel();

LoadBalance loadBalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invocation.getLoadBalanceType());
loadBalance.select();

// ...
```

这会带来一些小问题，总是需要写 `ExtensionLoader.getExtensionLoader(XXX.class).getExtension(parameter)` 这样重复的代码；引入了 `ExtensionLoader` 这个“中介”，不能直面拓展本身。后者可能有点难以体会，以动物园 `Zoo` 和 动物 `Animal` 举例。

在非动态加载情况下，我们可能会这样写：

```java
public class Zoo {
    private List<Animal> animals;

    public void bark(String type) {
        for (Animal animal : animals) {
            if (type.equals(animal.name)) {
                animal.bark();
            }
        }
    }
}
```

在动态加载情况下，我们可能会这样写。在这种情况下，`Zoo` 没有合适的方式直接持有 `Animal`，而是通过 `ExtensionLoader` 间接地持有。

```java
public class Zoo {
    private ExtensionLoader<Animal> extensionLoader = ExtensionLoader.getExtensionLoader(Animal.class);

    public void bark(String type) {
        Animal animal = extensionLoader.getExtension(type);
        animal.bark();
    }
}
```

我们更想要以下这种直接持有 `Animal` 的方式，在运行时 `animal` 可以是 `Dog`，也可以是 `Cat`，还可以是其他的动物。

```java
public class Zoo {
    private Animal animal;

    public void bark(String type) {
        animal.bark();
    }
}
```

`Dubbo` 采用了一种称为“自适应拓展”的巧妙设计，通过代理的方式，将动态加载拓展的代码整合到代理类（具体实现类）中。使用方调用代理对象，代理对象根据参数动态加载拓展并调用。例如 `Animal` 的自适应拓展，就像下面这样：

```java
public class AdaptiveAnimal implements Animal {
    public void bark(String type) {
        if (type == null) {
            throw new IllegalArgumentException("type == null");
        }
        
        Animal animal = ExtensionLoader.getExtensionLoader(Animal.class).getExtension(type);
        animal.bark();
    }
}

Animal animal = new AdaptiveAnimal();
animal.bark(type);
```

当然，我们不希望需要手动地为每一个拓展编写 `Adaptive` 代理类，事实上，我们以往接触到的代理方案，大都是自动生成代理的，应该也不会有人会接受完全手写的方式。然而你可能会注意到一个不够和谐的缺点，`bark` 方法的参数列表中新增了 `type` 类型，这不太符合面向对象的设计原则。想象一个更奇怪的场景，我们要为一个方法引入与它本身格格不入的参数用于获取拓展。另外，我们可能需要通过一些标记或约定来告诉代理生成器，方法参数列表中哪一个参数是用于获取拓展的。事实上，`Dubbo` 的另一个设计规避了这一缺点，`Dubbo` 在[公共契约](https://cn.dubbo.apache.org/zh-cn/docsv2.7/dev/contract/)中提到：**所有扩展点参数都包含 `URL` 参数，`URL` 作为上下文信息贯穿整个扩展点设计体系**。因此围绕着 `Dubbo` 以 `URL` 为中心的拓展体系，你很难设计出 `Animal.bark(URL url)` 这样不和谐的方法签名，也不用担心参数列表千奇百怪的情况。同时 `Dubbo` 并未完全抛弃手工编写自适应拓展的方式，而是予以保留。

### 手工编码的自适应拓展

在在 `Dubbo` 中，尽管很少但仍然存在手工编码的自适应拓展，**这类拓展允许你不使用 `URL` 作为参数**，查看它们的代码可以帮助我们更好地理解自适应拓展是如何在真实的应用场景中发挥作用的。以下是 `ExtensionFactory` 的自适应拓展，当你调用它的 `getExtension` 方法时，它就是将工作全权委托给 `factory.getExtension(type, name)` 完成的，而 `factories` 在创建 `AdaptiveExtensionFactory` 时就已经获取了。

```java
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
        // 获取 ExtensionFactory 的 ExtensionLoader
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        // 获取全部支持的（不包含自适应拓展）拓展名称，依次获取拓展加入 factories
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    @Override
    public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            // 委托给其他 ExtensionFactory 拓展获取，比如 SpiExtensionFactory
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }

}
```

至此，我们提到了按需加载是具备动态加载能力后自然的倾向，介绍了在拥有大量拓展情况下演变而来的自适应拓展设计，它的缺点和 Dubbo 是如何规避的。接下来，我们将进入源码分析部分。

## 源码分析

### Adaptive 注解

`Adaptive` 注解是一个与自适应拓展息息相关的注解，该定义如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Adaptive {
    String[] value() default {};
}
```

根据 `Target` 注解的 `value` 可知，`Adaptive` 注解可标注在类或者方法上。当 `Adaptive` 注解标注在类上时，`Dubbo` 不会为该类生成代理类。当 `Adaptive` 注解标注在接口方法上时，`Dubbo` 则会为该方法生成代理逻辑。`Adaptive` 注解在类上的情况很少，在 `Dubbo` 中，仅有两个类被 `Adaptive` 注解标注，分别是 `AdaptiveCompiler` 和 `AdaptiveExtensionFactory`。在这种情况下，拓展的加载逻辑由人工编码完成。在更多时候，`Adaptive` 注解是标注在接口方法上的，这表示拓展的加载逻辑需由框架自动生成。

### 获取自适应拓展

获取自适应拓展的入口方法是 `getAdaptiveExtension`，使用 `getOrCreate` 的模式获取。

```java
public T getAdaptiveExtension() {
    // 双重检查
    // 从缓存中获取自适应拓展
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        // 创建自适应拓展失败的结果也会被缓存，避免重复尝试
        if (createAdaptiveInstanceError != null) {
            throw new IllegalStateException("Failed to create adaptive instance: " +
                    createAdaptiveInstanceError.toString(),
                    createAdaptiveInstanceError);
        }

        synchronized (cachedAdaptiveInstance) {
            instance = cachedAdaptiveInstance.get();
            if (instance == null) {
                try {
                    // 创建自适应拓展
                    instance = createAdaptiveExtension();
                    // 将自适应拓展设置到缓存中
                    cachedAdaptiveInstance.set(instance);
                } catch (Throwable t) {
                    createAdaptiveInstanceError = t;
                    throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                }
            }
        }
    }

    return (T) instance;
}
```

### 创建自适应拓展

当缓存为空时，就会通过 `createAdaptiveExtension` 方法创建。方法包含以下三个处理逻辑：

1. 调用 `getAdaptiveExtensionClass` 方法获取自适应拓展的 `Class` 对象。
2. 通过反射进行实例化。
3. 调用 `injectExtension` 方法对拓展实例进行依赖注入。

> **手工编码的自适应拓展可能依赖其他拓展，但是框架生成的自适应拓展并不依赖其他拓展**。

```java
private T createAdaptiveExtension() {
    try {
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}
```

### 获取自适应拓展类

获取自适应拓展类的 `getAdaptiveExtensionClass` 方法包含以下三个处理逻辑：

1. 通过 `getExtensionClasses` 方法获取所有拓展类。
2. 检查缓存 `cachedAdaptiveClass`，如果不为 `null`，则返回缓存。
3. 如果缓存为 `null`，则调用 `createAdaptiveExtensionClass` 创建自适应拓展类（代理类）。

在{% post_link 'how-Dubbo-SPI-works' 'Dubbo SPI 的工作原理' %}中我们分析过 `getExtensionClasses` 方法，在获取拓展的所有实现类时，如果某个实现类被 `Adaptive` 注解标注了，那么该类就会被赋值给 `cachedAdaptiveClass` 变量。“原理”部分介绍的 `AdaptiveExtensionFactory` 就属于这种情况，我们不再细谈。按前文所说，在绝大多数情况下，`Adaptive` 注解都是用于标注方法而非标注具体的实现类，因此在大多数情况下程序都会走第三个步骤，由框架自动生成自适应拓展类（代理类）。

```java
private Class<?> getAdaptiveExtensionClass() {
    getExtensionClasses();
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

> 到目前为止，**获取自适应拓展的过程和获取普通拓展的过程是非常相似的**，使用 `getOrCreate` 的模式获取拓展，如果缓存为空则创建，创建的时候会先加载全部的拓展实现类，从中获取目标类，通过反射进行实例化，最后进行依赖注入。区别在于获取目标类时，在自适应拓展情况下，返回的可能是一个生成的代理类。生成的过程非常复杂，是我们接下来关注的重点。

### 生成自适应拓展类

生成自适应拓展类的方式相比于以往接触的生成代理类的方式更加“直观且容易理解”，但是相应的，拼接字符串部分的代码并不容易阅读。

1. 通过拼接字符串得到代理类的源码。
2. 使用编译器编译得到 `Class` 对象。

> 在新版本中，这部分代码的可读性有了非常大的提升，原先冗长的处理逻辑被抽象为多个命名含义清晰的方法。

```java
private Class<?> createAdaptiveExtensionClass() {
    // 区别于旧版本：新版本抽象出一个 AdaptiveClassCodeGenerator 用于生成代码
    String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
    ClassLoader classLoader = findClassLoader();
    // 获取编译器拓展
    org.apache.dubbo.common.compiler.Compiler compiler =
            ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    // 编译代码，生成 Class 对象
    return compiler.compile(code, classLoader);
}
```

*为了更直观地了解代码生成的效果及其实现的功能，以 `Protocol` 为例，生成的完整代码（已经经过格式化）展示如下*。

```java
package org.apache.dubbo.rpc;

import org.apache.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
    public void destroy() {
        throw new UnsupportedOperationException(
                "The method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException(
                "The method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1)
            throws org.apache.dubbo.rpc.RpcException {
        if (arg1 == null)
            throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url ("
                    + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader
                .getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }

    public java.util.List getServers() {
        throw new UnsupportedOperationException(
                "The method public default java.util.List org.apache.dubbo.rpc.Protocol.getServers() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0)
            throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url ("
                    + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader
                .getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }
}
```

生成的代理类需完成以下功能：

1. 非 `adaptive` 方法，直接抛出异常。
2. `adaptive` 方法：
    - 准备工作：在参数判空校验之后，从中获取到 `URL` 对象，结合 `URL` 对象和默认拓展名得到最终的拓展名 `extName`。
    - 核心功能：先获取拓展的 `ExtensionLoader`，再根据拓展名 `extName` 获取拓展，最后调用拓展的同名方法。

以上的功能在表面上看来并不复杂，事实上，想要实现的目标处理逻辑也并不复杂，只在为了提供足够的可扩展性，具体实现变得很复杂。复杂的处理逻辑主要集中在如何为“准备工作”部分生成相应的代码，大概可以总结为：在获取拓展前，`Dubbo` 会直接或间接地从参数列表中查找 `URL` 对象，所谓直接就是 `URL` 对象直接在参数列表中，所谓间接就是 `URL` 对象是其中一个参数的属性。在得到 `URL` 对象后，`Dubbo` 会尝试以 `Adaptive` 注解的 `value` 为 `key`，从 `URL` 中获取值作为拓展名，如果获取不到则使用默认拓展名 `defaultExtName`。实际的实现更加复杂，需要耐心阅读和测试。

### 自适应拓展类代码生成器

新版本将代码生成的逻辑抽象到自适应拓展类代码生成器中，注意参数只有 `type` 和 `defaultExtName`，从这里也可以看出如何确定最终加载的拓展，取决于这两个参数和被调用方法的入参。

```java
public AdaptiveClassCodeGenerator(Class<?> type, String defaultExtName) {
    this.type = type;
    this.defaultExtName = defaultExtName;
}

public String generate() {
    // 检测是否至少存在一个方法标注了 Adaptive 注解
    if (!hasAdaptiveMethod()) {
        throw new IllegalStateException("No adaptive method exist on extension " + type.getName() + ", refuse to create the adaptive class!");
    }

    // 区别于旧版本：抽象为几个命名含义清晰的方法，提升了可读性
    // 生成类：包名、导入、类声明
    StringBuilder code = new StringBuilder();
    code.append(generatePackageInfo());
    code.append(generateImports());
    code.append(generateClassDeclaration());

    // 生成方法
    Method[] methods = type.getMethods();
    for (Method method : methods) {
        code.append(generateMethod(method));
    }
    code.append("}");

    if (logger.isDebugEnabled()) {
        logger.debug(code.toString());
    }
    return code.toString();
}
```

#### 检测 Adaptive 注解

在生成代理类源码之前，`generate` 方法会先通过反射检测接口方法中是否至少有一个标注了 `Adaptive` 注解，若不满足，就会抛出异常。

> 流式编程使用得当的话很有可读性啊。

```java
private boolean hasAdaptiveMethod() {
    return Arrays.stream(type.getMethods()).anyMatch(m -> m.isAnnotationPresent(Adaptive.class));
}
```

#### 生成类

生成代理类源码的顺序和普通 `Java` 类文件中内容的顺序一致：

- package
- import
- 类声明

先忽略“生成方法”的部分，以 `Dubbo` 的 `Protocol` 拓展为例，生成的代码如下：

```java
package org.apache.dubbo.rpc;
import org.apache.dubbo.common.extension.ExtensionLoader;
public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
    // 省略方法代码
}
```

#### 生成方法

生成方法的过程同样被抽象为几个命名含义清晰的方法，包含以下五个部分：

- 返回值
- 方法名
- **方法内容**
- 方法参数
- 方法抛出的异常

```java
private String generateMethod(Method method) {
    String methodReturnType = method.getReturnType().getCanonicalName();
    String methodName = method.getName();
    String methodContent = generateMethodContent(method);
    String methodArgs = generateMethodArguments(method);
    String methodThrows = generateMethodThrows(method);
    return String.format(CODE_METHOD_DECLARATION, methodReturnType, methodName, methodArgs, methodThrows, methodContent);
}
```

除了最重要的“方法内容”部分，其他部分都是复制原方法的信息，并不复杂。生成“方法内容”部分，分为是否被 `Adaptive` 注解标注。

```java
private String generateMethodContent(Method method) {
    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    // 检测方法是否被 Adaptive 注解标注
    if (adaptiveAnnotation == null) {
        return generateUnsupported(method);
    } else {
        // ...
    }
    return code.toString();
}
```

##### 无 Adaptive 注解标注的方法

对于无 `Adaptive` 注解标注的方法，生成逻辑很简单，就是生成抛出异常的代码。

```java
private String generateUnsupported(Method method) {
    return String.format(CODE_UNSUPPORTED, method, type.getName());
}
```

*以 `Protocol` 接口的 `destroy` 方法为例，生成的内容如下*：

```java
throw new UnsupportedOperationException("The method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
```

##### 有 Adaptive 注解标注的方法

对于有 Adaptive 注解标注的方法，

```java
// 查找 URL 类型的参数
int urlTypeIndex = getUrlTypeIndex(method);
if (urlTypeIndex != -1) {
    // 生成 URL 判空检查和赋值的代码
    code.append(generateUrlNullCheck(urlTypeIndex));
} else {
    // 如果参数中没有直接出现 URL 类型，生成间接情况下的 URL 判空和赋值代码
    code.append(generateUrlAssignmentIndirectly(method));
}
// 获取方法的 Adaptive 注解的 value
String[] value = getMethodAdaptiveValue(adaptiveAnnotation);
// 检测是否有 Invocation 类型的参数
boolean hasInvocation = hasInvocationArgument(method);
// 生成 Invocation 判空检查代码
code.append(generateInvocationArgumentNullCheck(method));
// 生成拓展名赋值代码
code.append(generateExtNameAssignment(value, hasInvocation));
// 生成拓展名判空检查代码
code.append(generateExtNameNullCheck(value));
// 生成获取拓展和赋值代码
code.append(generateExtensionAssignment());
// 生成调用和返回代码
code.append(generateReturnAndInvocation(method));
```

##### 查找 URL 类型的参数

**直接**从方法的参数类型列表中查找**第一个** `URL` 类型的参数，返回其索引。

```java
private int getUrlTypeIndex(Method method) {
    int urlTypeIndex = -1;
    // 遍历方法的参数类型列表
    Class<?>[] pts = method.getParameterTypes();
    for (int i = 0; i < pts.length; ++i) {
        // 查找第一个 URL 类型的参数
        if (pts[i].equals(URL.class)) {
            urlTypeIndex = i;
            break;
        }
    }
    return urlTypeIndex;
}

// 生成 URL 参数判空检查和赋值代码
private String generateUrlNullCheck(int index) {
    return String.format(CODE_URL_NULL_CHECK, index, URL.class.getName(), index);
}
```

**间接**从方法的参数类型列表中，查找 `URL` 类型的参数，并生成判空检查和赋值代码。

```java
private String generateUrlAssignmentIndirectly(Method method) {
    Class<?>[] pts = method.getParameterTypes();

    Map<String, Integer> getterReturnUrl = new HashMap<>();
    // 遍历方法的参数类型列表
    for (int i = 0; i < pts.length; ++i) {
        // 遍历某一个参数类型的全部方法，查找可以返回 URL 类型的 “getter” 方法
        for (Method m : pts[i].getMethods()) {
            String name = m.getName();
            // 1. 方法名以 get 开头，或者方法名大于 3 个字符
            // 2. 方法的访问权限为 public
            // 3. 非静态方法
            // 4. 方法参数数量为 0
            // 5. 方法返回值类型为 URL
            if ((name.startsWith("get") || name.length() > 3)
                    && Modifier.isPublic(m.getModifiers())
                    && !Modifier.isStatic(m.getModifiers())
                    && m.getParameterTypes().length == 0
                    && m.getReturnType() == URL.class) {
                // 保存方法名->索引的映射
                getterReturnUrl.put(name, i);
            }
        }
    }

    if (getterReturnUrl.size() <= 0) {
        // 如果没有找到 “getter” 方法，抛出异常
        throw new IllegalStateException("Failed to create adaptive class for interface " + type.getName()
                + ": not found url parameter or url attribute in parameters of method " + method.getName());
    }

    // 优先选择方法名为 getUrl 的方法，如果没有则选第一个
    Integer index = getterReturnUrl.get("getUrl");
    if (index != null) {
        return generateGetUrlNullCheck(index, pts[index], "getUrl");
    } else {
        Map.Entry<String, Integer> entry = getterReturnUrl.entrySet().iterator().next();
        return generateGetUrlNullCheck(entry.getValue(), pts[entry.getValue()], entry.getKey());
    }
}

// 生成 URL 参数判空检查和赋值代码
private String generateGetUrlNullCheck(int index, Class<?> type, String method) {
    StringBuilder code = new StringBuilder();
    code.append(String.format("if (arg%d == null) throw new IllegalArgumentException(\"%s argument == null\");\n",
            index, type.getName()));
    code.append(String.format("if (arg%d.%s() == null) throw new IllegalArgumentException(\"%s argument %s() == null\");\n",
            index, method, type.getName(), method));

    code.append(String.format("%s url = arg%d.%s();\n", URL.class.getName(), index, method));
    return code.toString();
}
```

*以 `Protocol` 的 `refer` 和 `export` 方法为例，生成的内容如下*：

```java
// refer
if (arg1 == null) throw new IllegalArgumentException("url == null");
org.apache.dubbo.common.URL url = arg1;
// export
if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
if (arg0.getUrl() == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
org.apache.dubbo.common.URL url = arg0.getUrl();
```

##### 获取 Adaptive 注解的 value

```java
private String[] getMethodAdaptiveValue(Adaptive adaptiveAnnotation) {
    String[] value = adaptiveAnnotation.value();
    // 如果 value 为空，使用类名生成 value
    // 效果：LoadBalance -> load.balance
    if (value.length == 0) {
        String splitName = StringUtils.camelToSplitName(type.getSimpleName(), ".");
        value = new String[]{splitName};
    }
    return value;
}
```

##### 检测 Invocation 类型的参数

检测是否有 `Invocation` 类型的参数，并生成判空检查代码和赋值代码。从 `Invocation` 可以获得 `methodName`。

```java
private boolean hasInvocationArgument(Method method) {
    Class<?>[] pts = method.getParameterTypes();
    return Arrays.stream(pts).anyMatch(p -> CLASSNAME_INVOCATION.equals(p.getName()));
}

private String generateInvocationArgumentNullCheck(Method method) {
    Class<?>[] pts = method.getParameterTypes();
    return IntStream.range(0, pts.length).filter(i -> CLASSNAME_INVOCATION.equals(pts[i].getName()))
                    .mapToObj(i -> String.format(CODE_INVOCATION_ARGUMENT_NULL_CHECK, i, i))
                    .findFirst().orElse("");
}
```

以 `LoadBalance` 的 `select` 方法为例，生成的内容如下：

```java
if (arg2 == null) throw new IllegalArgumentException("invocation == null");
String methodName = arg2.getMethodName();
```

##### 获取拓展名

本方法用于根据 `SPI` 和 `Adaptive` 注解的 `value` 生成“获取拓展名”的代码，同时生成逻辑还受 `Invocation` 影响，因此相对复杂。总结的规则如下：

1. 正常情况下，使用 url.getParameter(value[i]) 获取
2. 如果默认拓展名非空，使用 url.getParameter(value[i], defaultExtName) 获取
3. 如果存在 Invocation，不论默认拓展名是否为空，总是使用 url.getMethodParameter(methodName, value[i], defaultExtName) 获取
4. 因为 protocol 是 url 的一部分，所以可以直接通过 getProtocol 获取。是否使用默认拓展名的方式就退化为原始的三元表达式。

```java
private String generateExtNameAssignment(String[] value, boolean hasInvocation) {
    // TODO: refactor it
    String getNameCode = null;
    // 逆序遍历 value（Adaptive 的 value）
    for (int i = value.length - 1; i >= 0; --i) {
        // 当 i 为最后一个元素的索引（因为是逆序遍历，第一轮就进入本分支）
        if (i == value.length - 1) {
            // 默认拓展名非空
            if (null != defaultExtName) {
                // protocol 是 url 的一部分，可以通过 getProtocol 方法获取，其他的则必须从 URL 参数中获取
                if (!"protocol".equals(value[i])) {
                    if (hasInvocation) {
                        // 如果有 Invocation，则使用 url.getMethodParameter 获取
                        // url.getMethodParameter(methodName, value[i], defaultExtName)
                        getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                    } else {
                        // url.getParameter(value[i], defaultExtName)
                        getNameCode = String.format("url.getParameter(\"%s\", \"%s\")", value[i], defaultExtName);
                    }
                } else {
                    // ( url.getProtocol() == null ? defaultExtName : url.getProtocol() )
                    getNameCode = String.format("( url.getProtocol() == null ? \"%s\" : url.getProtocol() )", defaultExtName);
                }
            // 默认拓展名为空
            } else {
                if (!"protocol".equals(value[i])) {
                    if (hasInvocation) {
                        // url.getMethodParameter(methodName, value[i], defaultExtName)
                        getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                    } else {
                        // url.getParameter(value[i])
                        getNameCode = String.format("url.getParameter(\"%s\")", value[i]);
                    }
                } else {
                    // url.getProtocol()
                    getNameCode = "url.getProtocol()";
                }
            }
        } else {
            if (!"protocol".equals(value[i])) {
                if (hasInvocation) {
                    // url.getMethodParameter(methodName, value[i], defaultExtName)
                    getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                } else {
                    // url.getParameter(value[i], getNameCode)
                    getNameCode = String.format("url.getParameter(\"%s\", %s)", value[i], getNameCode);
                }
            } else {
                // url.getProtocol() == null ? "dubbo" : url.getProtocol()
                getNameCode = String.format("url.getProtocol() == null ? (%s) : url.getProtocol()", getNameCode);
            }
        }
    }
    return String.format(CODE_EXT_NAME_ASSIGNMENT, getNameCode);
}
```

##### 加载拓展

```java
private String generateExtensionAssignment() {
    return String.format(CODE_EXTENSION_ASSIGNMENT, type.getName(), ExtensionLoader.class.getSimpleName(), type.getName());
}
```

以 `Protocol` 接口的 `refer` 方法为例，生成的内容如下：

```java
org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
```

##### 调用与返回

生成方法调用语句，如有必要，返回结果。

```java
private String generateReturnAndInvocation(Method method) {
    String returnStatement = method.getReturnType().equals(void.class) ? "" : "return ";

    String args = IntStream.range(0, method.getParameters().length)
            .mapToObj(i -> String.format(CODE_EXTENSION_METHOD_INVOKE_ARGUMENT, i))
            .collect(Collectors.joining(", "));

    return returnStatement + String.format("extension.%s(%s);\n", method.getName(), args);
}
```

以 `Protocol` 接口的 `refer` 方法为例，生成的内容如下：

```java
return extension.refer(arg0, arg1);
```

> 新版本通过提炼方法、使用流式编程和使用 `String.format()` 代替 StringBuilder，提供了更好的代码可读性。官方写得源码解析真好。

## 参考文章

- [SPI 自适应拓展](https://cn.dubbo.apache.org/zh-cn/docsv2.7/dev/source/adaptive-extension/)