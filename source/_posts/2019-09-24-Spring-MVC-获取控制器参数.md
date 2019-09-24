---
title: Spring MVC 获取控制器参数
date: 2019-09-24 16:13:49
tags:
---

## Spring MVC 获取控制器参数

在使用 Spring MVC 进行 Web 开发时，当前端发起请求，传递参数到后端时，后端的开发人员如何获取到这些参数呢？

### 如何考虑

这件事需要考虑两个方面：
1. 参数是采用什么方式传递到后端的
2. 参数如何正确地进行类型转换和格式化

#### 参数采用什么方式传递到后端
参数采用什么方式传递到后端，将影响我们采用什么方式获取对应的参数。在 Spring MVC 中，表现为我们使用不同的注解。参数的传递方式一般可以分为三种：
1. HTTP 参数（`@RequestParam`）
2. HTTP 请求体（`@RequestBody`）
3. URL（`@PathVariable`）

#### 参数如何正确的进行转换和格式化
在大部分的情况下，我们不关心也不会自己去开发这一步，因为 Spring MVC 已经提供了大量的转换规则，通过这些规则就能非常简易地获取大部分的参数。

### 常用方式

#### 在无注解下获取参数

在没有注解的情况下，Spring MVC 也可以获取参数，且参数允许为空，唯一的要求是参数名称和 HTTP 请求的参数名称保持一致。

**代码示例**：

```java
package com.moralok.springmvc

/** imports **/

@RequestMapping("/my")
@RestController
public class MyController {

    @RequestMapping("/no/annotation")
    @ResponseBody
    public Map<String, Object> noAnnotation(Integer intVal, Long longVal, String str) {
        Map<String, Object> paramsMap = new HashMap<>();
        paramsMap.put("intVal", intVal);
        paramsMap.put("longVal", longVal);
        paramsMap.put("str", str);
        return paramsMap;
    }

}
```

#### 使用 `@RequestParam` 获取参数

在前后端分离的趋势下，前端的命名规则可能与后端的规则不同，这是需要把前端的参数与后端对应起来。Spring MVC 提供了注解 `@RequestParam` 来确定前后端参数名称的映射关系。

默认情况下，`@RequestParam` 标注的参数是不能为空的，为了让它能够为空，可以配置其属性 required 为 false。

**代码示例**：

```java
package com.moralok.springmvc

/** imports **/

@RequestMapping("/my")
@RestController
public class MyController {

    @GetMapping("annotation")
    @ResponseBody
    public Map<String, Object> requestParam(
            @RequestParam("int_val") Integer intVal,
            @RequestParam("long_val") Long longVal,
            @RequestParam("str_val") String strVal
    ) {
        Map<String, Object> paramsMap = new HashMap<>();
        paramsMap.put("intVal", intVal);
        paramsMap.put("longVal", longVal);
        paramsMap.put("str", strVal);
        return paramsMap;
    }

}
```

#### 传递数组

Spring MVC 支持传递数组。Spring MVC 内部已经能够支持用逗号分隔的数组参数。需要传递数组参数时，每个参数的数组元素只需要通过逗号分隔即可。

**代码示例**：

```java
package com.moralok.springmvc

/** imports **/

@RequestMapping("/my")
@RestController
public class MyController {

    @GetMapping("/requestArray")
    @ResponseBody
    public Map<String, Object> requestArray(int[] intArr, long[] longArr, String[] strArr) {
        Map<String, Object> paramsMap = new HashMap<>();
        paramsMap.put("intArr", intArr);
        paramsMap.put("longArr", longArr);
        paramsMap.put("strArr", strArr);
        return paramsMap;
    }
}
```


#### 传递 Json

在前后端分离的趋势下，使用 _JSON_ 已经十分普遍。有时前端需要提交较为复杂的数据到后端，为了更好组织和提高代码的可读性，可以将数据转换为 JSON 数据集，通过 HTTP 请求体提交给后端。

当方法的参数标注为 `@RequestBody`，意味着它将接收前端提交的 JSON 请求体，当 JSON 请求体和某一个 POJO 之间的属性名称是保持一致时，Spring MVC 会通过这层映射关系将 JSON 请求体转化为对应的 POJO 对象。

**代码示例**：

```java
package com.moralok.springmvc

/** imports **/

@RequestMapping("/my")
@RestController
public class MyController {

	@PostMapping("insert")
	@ResponseBody
	public User insert(@RequestBody User user) {
		userService.insertUser(user);
		return user;
	}
}
```

#### 通过 URL 传递参数

在采用 RESTful 风格的网站中，参数往往是通过 URL 进行传递。Spring MVC 可以通过处理器映射和注解 `@PathVariable` 的组合获取参数。

**代码示例**：

```java
package com.moralok.springmvc

/** imports **/

@RequestMapping("/my")
@RestController
public class MyController {

    @GetMapping("/{id}")
    @ResponseBody
    public User get(@PathVariable("id") Long id) {
        return userService.getUser(id);
    }
}
```


#### 获取格式化参数

在一些应用中，往往需要格式化数据，其中最为典型的当属日期和货币。Spring MVC 分别提供了 `@DateTimeFormat` 和 `@NumberFormat`。

**代码示例**：

```java
package com.moralok.springmvc

/** imports **/

@RequestMapping("/my")
@RestController
public class MyController {

    @PostMapping("/format/commit")
    @ResponseBody
    public Map<String, Object> format(
            @DateTimeFormat(iso= DateTimeFormat.ISO.DATE) Date date,
            @NumberFormat(pattern = "#,###.##") Double number
            ) {
        Map<String, Object> dataMap = new HashMap<>();
        dataMap.put("date", date);
        dataMap.put("number", number);
        return dataMap;
    }
}
```

