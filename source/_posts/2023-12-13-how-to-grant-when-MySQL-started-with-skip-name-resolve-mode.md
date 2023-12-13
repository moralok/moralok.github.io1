---
title: 当 MySQL 以 skip-name-resolve 模式启动时如何使用 grant 命令
date: 2023-12-13 12:16:17
tags: [mysql]
---

本文介绍了 `MySQL` 中 `skip-name-resolve` 参数**对连接的优化作用**，随之而来的权限表**仅可使用 `IP` 的限制**，以及如何在无法提前确定 `IP` 的情况下使用 `grant` 命令**搭配通配符 `%` 进行授权**。

<!-- more -->

## 背景介绍

在 `MySQL` 新建一个用户 `exporter` 给 `mysqld-exporter` 使用，然后通过 `grant` 命令授予该用户部分权限。执行命令如下：

```sql
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'XXXXXXXX' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
```

在执行时提示“这次 `grant` 需要关闭 `skip-name-resolve` 参数然后重启 `MySQL` 才能生效”。

```console
0 row(s) affected, 1 warning(s): 1285 MySQL is started in --skip-name-resolve mode; you must restart it without this switch for this grant to work
```

本着默认的参数不随便修改的想法，我们来看一看 `skip-name-resolve` 参数究竟有什么用。

## skip-name-resolve 的作用

`MySQL` 在启动时默认使用 `skip-name-resolve` 参数。顾名思义，该参数用于跳过 `MySQL` 的 `DNS` 解析。关于 `MySQL` 是如何使用 `DNS` 的，官方文档是这样介绍的：

当一个新线程连接到 `mysqld`，`mysqld` 会生成一个新线程去处理该请求。该线程首先检查主机名是否在主机名缓存中，如果没有那么该线程会调用 `gethostbyaddr_r()` 和 `gethostbyname_r()` 来解析主机名。如果操作系统不支持上述线程安全的调用，那么该线程会加锁然后调用 `gethostbyaddr()` 和 `gethostbyname()`。这种情况下，在第一个线程准备好之前，没有其他线程可以解析不在主机名缓存中的其他主机名。`MySQL` 允许通过 `skip-name-resolve` 参数禁用 `DNS` 解析，但是如果禁用的话，在 `MySQL` 的权限表中就只能使用 `IP`。

> 获取 `IP` 和主机名可以用于在权限表中匹配权限，如果禁用 `DNS` 解析，在权限表中只能使用 `IP` 也就理所当然了。

官方建议如果你的 `DNS` 解析非常慢或者你有非常多的主机，可以使用 `skip-name-resolve` 参数禁用 `DNS` 解析或者增加 `HOST_CACHE_SIZE`（默认值：`128`）并重新编译 `mysqld` 来获得更高的性能。以下文章的作者介绍了他们因为 `MySQL` `DNS` 解析遇到的连接慢的问题：

- [mysql中的 skip-name-resolve 问题](https://blog.csdn.net/hwhua1986/article/details/78188231)
- [skip-name-resolve to speed up MySQL and avoid problems](https://www.vionblog.com/skip-name-resolve-to-speed-up-mysql-and-avoid-problems/)

> 因为缺乏对具体的 `MySQL` 代码实现的了解，所以没有办法对上述过程进行深入和准确的描述。在官方的表述中，根据 `IP` 查询主机名和根据主机名查询 `IP` 似乎都存在；但在查阅到的资料中，部分文章提到的一般是“反向解析”，也就是根据 `IP` 查询主机名；但是更多的文章对这部分比较含糊其辞。

## 怎么使用 grant 命令

如果你并不考虑对特定用户多加限制，那么使用通配符 `%` 即可，它将允许从任意 `IP` 连接 `MySQL`。

```sql
CREATE USER 'exporter'@'%' IDENTIFIED BY 'XXXXXXXX' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';
```

但是在我的情况中，新建的用户是给 `mysqld-exporter` 获取 `MySQL` 的监控数据的，显然**限制该用户的 `IP`、连接数以及权限**很有必要。可是如果使用的是 `Docker` 或者其他无法提前确定 `IP` 的环境怎么办呢？还是可以**使用通配符 `%` 限制为仅某一 `IP` 范围可以访问**。

```sql
CREATE USER 'exporter'@'172.19.%.%' IDENTIFIED BY 'XXXXXXXX' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'172.19.%.%';
```