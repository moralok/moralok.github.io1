---
title: 使用 logrotate 滚动 Docker 容器内的 Nginx 的日志
date: 2023-12-02 13:26:20
tags: [logrotate, docker, nginx]
---

`Nginx` 没有提供开箱即用的日志滚动功能，而是将其交给使用者自己实现。你既可以按照官方文档的建议通过编写脚本实现，也可以使用 `logrotate` 管理日志。但是和在普通场景下不同，在使用 `Docker` 运行 `Nginx` 时，你可能需要额外考虑一点细节。本文记录了在为 `Docker` 中的 `Nginx` 的日志文件配置滚动功能过程中遇到的一些问题和思考。

<!-- more -->

## Nginx 滚动日志

### 官方文档

> In order to rotate log files, they need to be renamed first. After that USR1 signal should be sent to the master process. The master process will then re-open all currently open log files and assign them an unprivileged user under which the worker processes are running, as an owner. After successful re-opening, the master process closes all open files and sends the message to worker process to ask them to re-open files. Worker processes also open new files and close old files right away. As a result, old files are almost immediately available for post processing, such as compression.

根据官方文档的解释，滚动日志文件的流程应如下，你可以自己编写 `Shell` 脚本配合 `crontab` 实现定时滚动功能。

1. 首先重命名日志文件
2. 之后发送 `USR1` 信号给 `Nginx` 主进程，`Nginx` 将重新打开日志文件
3. 对日志文件进行后处理，比如压缩（可选）

```shell
$ mv access.log access.log.0
$ kill -USR1 `cat master.nginx.pid`
$ sleep 1
$ gzip access.log.0    # do something with access.log.0
```

#### 说明

1. 在没有执行 `kill` 命令前，即便已经重命名了日志文件，`Nginx` 还是会向重命名后的文件写入日志。因为在 `Linux` 系统中，系统内核是根据文件描述符定位文件的。
2. `USR1` 是自定义信号，软件的作者自己确定收到该信号后做什么。在 `Nginx` 中，主进程收到信号后，会重新打开所有当前打开的日志文件并将它们分配给一个非特权用户作为所有者，工作进程就是在该所有者下运行的。成功重新打开后，主进程关闭所有打开的文件并向工作进程发送消息，要求它们重新打开文件。工作进程也打开新文件并立即关闭旧文件。

### 使用 logrotate

> logrotate is designed to ease administration of systems that generate large numbers of log files. It allows automatic rotation, compression, removal, and mailing of log files. Each log file may be handled daily, weekly, monthly, or when it grows too large.

`logrotate` 旨在简化生成大量日志文件的系统的管理。它允许自动滚动、压缩、删除和邮寄日志文件。每个日志文件可以在每天、每周、每月或当它变得太大时处理。`Linux` 一般默认安装了 `logrotate`。

#### 默认配置文件

查看默认配置文件：`cat /etc/logrotate.conf`。

```conf
# see "man logrotate" for details
# 每周滚动日志文件
weekly

# 默认使用 adm group，因为这是 /var/log/syslog 的所属组
su root adm

# 保留 4 周的备份（其实是保留 4 个备份，对应 weekly 的设置，就是保留 4 周）
rotate 4

# 在滚动旧日志文件后，创建新的空日志文件
create

# 使用日期作为滚动日志文件的后缀
#dateext

# 如果你希望压缩日志文件，请取消注释
#compress

# 软件包将日志滚动的配置信息放入此目录中
include /etc/logrotate.d

# system-specific logs may be also be configured here.
```

#### 配置信息所在目录

查看日志滚动的配置信息所在的目录：`ls /etc/logrotate.d/`。

```console
alternatives  apport  apt  bootlog  btmp  dpkg  rsyslog  ubuntu-advantage-tools  ufw  unattended-upgrades  wtmp
```

#### 为 Nginx 新增配置

为 `Nginx` 新增日志滚动配置，`vim /etc/logrotate.d/nginx`。

```conf
/path/to/your/nginx/logs/*.log {
    # 切换用户
    su moralok moralok
    # 每天滚动日志文件
    daily
    # 使用日期作为滚动日志文件的后缀
    dateext
    # 如果日志丢失，不报错继续滚动下一个日志
    missingok
    # 保留 31 个备份
    rotate 31
    # 不压缩
    nocompress
    # 整个日志组运行一次的脚本
    sharedscripts
    # 滚动后的处理
    postrotate
        # 重新打开日志文件        
        docker exec nginx sh -c "[ ! -f /var/run/nginx.pid ] || (kill -USR1 `docker exec nginx cat /var/run/nginx.pid`; echo 'Successfully rotating nginx logs.')"
    endscript
}
```

#### 验证配置和测试

测试配置文件是否有错误，`logrotate -d /etc/logrotate.d/nginx`。

强制滚动：`logrotate -f /etc/logrotate.d/nginx`

### 一些坑

#### /var/lib/logrotate/ 权限问题

当你使用校验过配置文件的正确性后，尝试强制滚动时，可能会遇到报错。

```console
$ logrotate -f /etc/logrotate.d/nginx 
error: error creating output file /var/lib/logrotate/status.tmp: Permission denied
```

这是因为 logrotate 会在 `/var/lib/logrotate/` 目录下创建 `status` 文件。查看目录权限可知，需要以 `root` 身份运行 `logrotate`。

```console
$ ll /var/lib/logrotate
total 12
drwxr-xr-x  2 root root 4096 Dec  2 08:35 ./
drwxr-xr-x 44 root root 4096 Jun 26 09:02 ../
-rw-r--r--  1 root root 1395 Dec  2 08:35 status
```

事实上，`logrotate -d /etc/logrotate.d/nginx` 命令也会读取 `/var/lib/logrotate/status`，但是 `other` 对该目录也有 `r` 读取权限，所以没有报错。

```console
$ logrotate -d /etc/logrotate.d/nginx 
WARNING: logrotate in debug mode does nothing except printing debug messages!  Consider using verbose mode (-v) instead if this is not what you want.

reading config file /etc/logrotate.d/nginx
Reading state from file: /var/lib/logrotate/status
...
```

#### 日志文件夹的权限

即使你使用 `root` 身份运行 `logrotate`，你可能还会遇到以下报错

```console
$ logrotate -f /etc/logrotate.d/nginx 
error: skipping "/path/to/your/nginx/logs/*.log" because parent directory has insecure permissions (It's world writable or writable by group which is not "root") Set "su" directive in config file to tell logrotate which user/group should be used for rotation.
```

你需要在配置文件中，使用 `su <user> <group>` 指定日志所在文件夹所属的用户和组，`logrotate` 才能正确读写。

#### 由宿主机还是容器主导

首先 `Nginx` 的日志文件夹通过挂载映射到宿主机，日志滚动既可以由宿主机主导，也可以由容器主导，不过不论如何我们都需要向 `Docker` 容器内的 `Nginx` 发送 `USR1` 信号。有人倾向于在容器内完成所有工作，和宿主机几乎完全隔离；我个人更青睐于由宿主机主导，因为容器内的环境并不总是拥有你想要使用的软件（除非你总是定制自己使用的镜像），甚至标准镜像往往非常精简。

在 `logrotate` 配置中的 `postrotate` 部分添加脚本，使用 `docker exec` 在容器内执行命令，完成向 `Nginx` 发送信号的工作。脚本的处理逻辑大概是“如果存在 `/var/run/nginx.pid`，就执行 `` kill -USR1 \`cat /var/run/nginx.pid\` `` 命令，并打印成功的消息”。但是我看到很多文章中分享的配置类似下面这样：

```shell
docker exec nginx sh -c "if [ -f /var/run/nginx.pid ]; then kill -USR1 $(docker exec nginx cat /var/run/nginx.pid); fi"

docker exec nginx sh -c "[ ! -f /var/run/nginx.pid ] || kill -USR1 `cat /var/run/nginx.pid`;"
```

经过测试都会有以下报错，我不清楚是否大多是抓取发布的文章，也不清楚他们是否测试过，对于 `Shell` 脚本写得不多的我来说，半夜测试反复排查错误真是头昏脑胀。在我原先的理解里，`-c` 后面的脚本是整体发到容器内部执行的，后来我才意识到，我对脚本内部的命令在宿主机还是容器里执行的理解是错误的。

```console
cat: /var/run/nginx.pid: No such file or directory
```

## logrotate 备忘

[帮助文档](https://linux.die.net/man/8/logrotate)

### 命令参数

```shell
-d, --debug : 打开调试模式，这意味着不会对日志进行任何更改，并且 logrotate 状态文件不会更新。仅打印调试消息。
-f, --force : 告诉 logrotate 强制滚动，即使它认为这没有必要。有时，在将新条目添加到 logrotate 配置文件后，或者如果已手动删除旧日志文件，这会很有用，因为将创建新文件，并且正确地继续记录日志。
-m, --mail <command> : 告诉 logrotate 在邮寄日志时使用哪个命令。此命令应接受两个参数：消息的主题，收件人。然后，该命令必须读取标准输入上的消息并将其邮寄给收件人。默认邮件命令是 /bin/mail -s。
-s, --state <statefile> : 告诉 logrotate 使用备用状态文件。如果 logrotate 以不同用户身份运行不同的日志文件集，这会非常有用。默认状态文件是 /var/lib/logrotate/status。
--usage : 打印简短的用法信息。
--?, --help : 打印帮助信息。
-v, --verbose : 打开详细模式，例如在滚动期间显示消息。
```

### 常用配置文件参数

|参数|说明|
|--|--|
|daily|周期：每天|
|weekly|周期：每周|
|monthly|周期：每月|
|yearly|周期：每年|
|dateext|使用日期代替数字作为扩展名（YYYYMMDD），如：access.log-20231202|
|dateformat|必须配合 dateext 使用，只支持 %Y %m %d %s 这四个参数|
|compress|使用 gzip 压缩旧版本日志文件|
|nocompress|不压缩|
|delaycompress|延迟到下一次滚动周期再压缩|
|nodelaycompress|不延迟压缩，覆盖 delaycompress|
|create mode owner group|滚动后（运行 postrotate 脚本前），立即创建日志文件，指定权限属性|
|nocreate|不创建新的日志文件，覆盖 create|
|ifempty|即使是空文件也滚动（默认），覆盖 notifempty|
|missingok|如果日志丢失，不报错继续滚动下一个日志|
|notifempty|如果是空文件，不滚动，覆盖 ifempty|
|sharedscripts|在所有日志都滚动后统一执行一次脚本。如果没有配置，每个日志滚动后都会执行一次脚本|
|prerotate/endscript|存放在滚动以前需要执行的脚本，两者必须单独成行|
|postrotate/endscript|存放在滚动以后需要执行的脚本，两者必须单独成行|
|rotate count|日志文件在删除或邮寄前滚动的次数，0 指没有备份|
|size log-size|当日志文件到达指定的大小时才滚动，默认单位 byte，可以使用 k、M、G|

> 文档说默认在运行 `postrotate` 脚本前创建新文件，可是新文件不是 `Nginx` 重新打开文件时创建的吗？之前脚本执行报错，也没有产生新文件啊。

## 参考文章

- [Rotating Log-files](https://nginx.org/en/docs/control.html#logs)
- [Log Rotation](https://www.nginx.com/resources/wiki/start/topics/examples/logrotation/)
- [logrotate(8) - Linux man page](https://linux.die.net/man/8/logrotate)
- [error: error creating output file /var/lib/logrotate.status.tmp: Permission denied](https://serverfault.com/questions/1023555/error-error-creating-output-file-var-lib-logrotate-status-tmp-permission-deni)
- [apache-and-logrotate-configuration](https://stackoverflow.com/questions/26482773/apache-and-logrotate-configuration)
- [docker nginx/openresty容器使用logrotate日志切割](https://blog.slogra.com/post-792.html)

