---
title: 如何使用 Spring Security
date: 2019-10-17 14:37:01
tags:
---


### 在 pom.xml 中添加项目依赖
``` xml
<!--SpringSecurity依赖配置-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### 添加 Spring Security 的配置类

SecurityConfig 需要继承自 WebSecurityConfigurerAdapter，通过重写父类的方法，实现自定义的需求。

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled=true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
}
```

通常，我们会关注以下几个方法。

#### 配置用户签名服务

本方法用于配置用户信息，包括为用户配置角色。在 Spring Security 中默认是没有任何用户配置的。Spring Security 会自动地生成一个名称为 user、密码通过随机生成的用户，密码可以在日志中观察得到。显然，默认情况下存在各类的缺点，为了克服这些缺点，我们需要自定义用户签名服务。

##### 使用内存签名服务

```java
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.inMe
}
```

```java
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.userDetailsService(userDetailsService())
            .passwordEncoder(passwordEncoder());
}
```