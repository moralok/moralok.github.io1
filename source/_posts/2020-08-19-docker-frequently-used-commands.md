---
title: Docker 常用命令列表
date: 2020-08-19 22:33:24
tags:
    - docker
---


## 常用命令

从 Docker 官方命令行参考中按顺序选取常用的命令和用法。

### docker attach

将本地标准输入、输出和错误流附加到一个正在运行的容器。

#### 用法
```shell
docker attach [OPTIONS] CONTAINER
```

#### 描述
使用 `docker attach` 通过容器 ID 或名称将终端的标准输入、输出和错误附加到正在运行的容器。这允许你查看其正在进行的输出或以交互方式控制它，就好像命令直接在你的终端中运行一样。

如果 `docker run` 同时指定 `-it` 选项，使用 `CTRL-p CTRL-q` 退出 `docker attach` 时，不影响容器继续运行。

#### 选项
|名称和缩写|默认值|描述|
|------|------|------|
|`--detach-keys`||覆盖用于从容器分离的键序列|
|`--sig-proxy`|true|代理转发所有收到的信号给进程|

### docker exec

在正在运行中的容器里执行命令。

#### 用法
```shell
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

#### 选项
|名称和缩写|描述|
|------|------|
|`--detach`,`-d`|分离模式：在后台运行命令|
|`--env`,`-e`|设置环境变量|
|`--interactive`,`-i`|即使未附加也保持 STDIN 打开|
|`--tty`,`-t`|分配伪 TTY|
|`--workdir`,`-w`|容器内的工作目录|


### docker images

列出镜像。

#### 用法
```shell
docker images [OPTIONS] [REPOSITORY[:TAG]]
```

#### 选项
|名称和缩写|描述|
|------|------|
|`--all`,`-a`|展示所有镜像|


### docker info

展示系统范围的信息。

#### 用法
```shell
docker info [OPTIONS]
```

#### 选项
|名称和缩写|描述|
|------|------|
|`--format`,`-f`|使用指定模板格式化输出|


### docker inspect

返回有关 Docker 对象的低级信息。

#### 用法
```shell
docker inspect [OPTIONS] NAME|ID [NAME|ID...]
```

#### 选项
|名称和缩写|描述|
|------|------|
|`--format`,`-f`|使用指定模板格式化输出|


### docker logs

获取容器的日志。

#### 用法
```shell
docker logs [OPTIONS] CONTAINER
```

#### 选项
|名称和缩写|描述|
|------|------|
|`--follow`,`-f`|按照日志输出|
|`--since`|显示指定时间戳（2013-01-02T13:23:37Z）或相对时间（42m）后的日志|
|`--tail`,`-n`|从日志末尾开始显示的行数|
|`--timestamps`,`-t`|显示时间戳|
|`--until`|显示指定时间戳（2013-01-02T13:23:37Z）或相对时间（42m）前的日志|


### docker network

管理网络。

#### 用法
```shell
docker network COMMAND
```

#### docker network connect

将容器连接到网络。

##### 用法
```shell
docker network connect [OPTIONS] NETWORK CONTAINER
```

#### docker network create

创建网络。

##### 用法
```shell
docker network create [OPTIONS] NETWORK
```

##### 选项
|名称和缩写|默认值|描述|
|------|------|------|
|`--driver`,`-d`|bridge|按照日志输出|

#### docker network disconnect

断开容器与网络的连接。

##### 用法
```shell
docker network create [OPTIONS] NETWORK
```

#### docker network inspect

显示一个或多个网络的详细信息。

##### 用法
```shell
docker network inspect [OPTIONS] NETWORK [NETWORK...]
```

#### docker network ls

列出网络。

##### 用法
```shell
docker network ls [OPTIONS]
```

##### 选项
|名称和缩写|默认值|描述|
|------|------|------|
|`--filter`,`-f`|提供过滤值（例如 driver=bridge）|

#### docker network rm

删除一个或多个网络。

##### 用法
```shell
docker network rm NETWORK [NETWORK...]
```

##### 选项
|名称和缩写|默认值|描述|
|------|------|------|
|`--filter`,`-f`|提供过滤值（例如 driver=bridge）|


### docker port

列出容器的端口映射或特定映射。

#### 用法
```shell
docker port CONTAINER [PRIVATE_PORT[/PROTO]]
```

#### 例子
```shell
docker ps

CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                                            NAMES
b650456536c7        busybox:latest      top                 54 minutes ago      Up 54 minutes       0.0.0.0:1234->9876/tcp, 0.0.0.0:4321->7890/tcp   test
docker port test

7890/tcp -> 0.0.0.0:4321
9876/tcp -> 0.0.0.0:1234
docker port test 7890/tcp

0.0.0.0:4321
docker port test 7890/udp

2014/06/24 11:53:36 Error: No public port '7890/udp' published for test
docker port test 7890

0.0.0.0:4321
```


### docker ps

列出容器。

#### 用法
```shell
docker ps [OPTIONS]
```

#### 选项
|名称和缩写|描述|
|------|------|
|`--all`,`-a`|显示所有容器（默认只显示运行中的容器）|
|`--filter`,`-f`|根据提供的条件过滤输出|


### docker pull

从仓库下载镜像。

#### 用法
```shell
docker pull [OPTIONS] NAME[:TAG|@DIGEST]
```


### docker rename

重命名容器。

#### 用法
```shell
docker rename CONTAINER NEW_NAME
```


### docker restart

重启一个或多个容器。

#### 用法
```shell
docker restart [OPTIONS] CONTAINER [CONTAINER...]
```


### docker rm

移除一个或多个容器。

#### 用法
```shell
docker rm [OPTIONS] CONTAINER [CONTAINER...]
```

#### 选项
|名称和缩写|描述|
|------|------|
|`--force`,`-f`|强制删除正在运行的容器（使用 SIGKILL）|


### docker rmi

删除一个或多个镜像。

#### 用法
```shell
docker rmi [OPTIONS] IMAGE [IMAGE...]
```

#### 选项
|名称和缩写|描述|
|------|------|
|`--force`,`-f`|强制删除镜像（当其正在被容器使用中）|


### docker run

从镜像创建并运行新容器。

#### 用法
```shell
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

#### 选项
|名称和缩写|默认值|描述|
|------|------|------|
|`--attach`,`-a`||附加到 STDIN、STDOUT 或 STDERR|
|`--detach`,`-d`||在后台运行容器并打印容器 ID|
|`--env`,`-e`||设置环境变量|
|`--expose`||公开一个端口或一系列端口|
|`--interactive`,`-i`||即使未附加，也要保持 STDIN 打开|
|`--name`||为容器命名|
|`--network`||将容器连接到网络|
|`--publish`,`-p`||将容器的端口发布到主机|
|`--restart`|no|容器退出时应用的重启策略|
|`--tty`,`-t`||分配伪 TTY|
|`--volume`,`-v`||绑定挂载卷|


### docker search

在 Docker Hub 中搜索镜像。

#### 用法
```shell
docker search [OPTIONS] TERM
```

#### 选项
|名称和缩写|描述|
|------|------|
|`--filter`,`-f`|根据提供的条件过滤输出|
|`--limit`|最大搜索结果数量|


### docker start

启动一个或多个容器。

#### 用法
```shell
docker start [OPTIONS] CONTAINER [CONTAINER...]
```

#### 选项
|名称和缩写|描述|
|------|------|
|`--attach`,`-a`|附加 STDOUT/STDERR 和转发信号|
|`--interactive`,`-i`|附加容器的 STDIN|


### docker stats

显示容器资源使用统计的实时流。

#### 用法
```shell
docker stats [OPTIONS] [CONTAINER...]
```

#### 选项
|名称和缩写|描述|
|------|------|
|`--all`,`-a`|显示所有容器（默认只显示运行中的容器）|

#### 例子
```shell
docker stats
CONTAINER ID   NAME                 CPU %     MEM USAGE / LIMIT     MEM %     NET I/O           BLOCK I/O         PIDS
3d2180bad26e   zoo1                 0.05%     34.98MiB / 15.59GiB   0.22%     38.6MB / 0B       5.39MB / 656MB    39
828987756ee6   zoo3                 0.04%     80.3MiB / 15.59GiB    0.50%     38.6MB / 0B       20.9MB / 522MB    32
d6e23f304046   zoo2                 0.04%     78.84MiB / 15.59GiB   0.49%     38.6MB / 0B       19.8MB / 883MB    32
```


### docker stop

停止一个或多个正在运行的容器。

#### 用法
```shell
docker stop [OPTIONS] CONTAINER [CONTAINER...]
```


### docker top

显示容器中的运行进程。

#### 用法
```shell
docker top CONTAINER [ps OPTIONS]
```

#### 例子
```shell
docker top jellyfin
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                652954              652845              0                   6月16                ?                   00:00:00            /package/admin/s6/command/s6-svscan -d4 -- /run/service
root                653039              652954              0                   6月16                ?                   00:00:00            s6-supervise s6-linux-init-shutdownd
root                653040              653039              0                   6月16                ?                   00:00:00            /package/admin/s6-linux-init/command/s6-linux-init-shutdownd -c /run/s6/basedir -g 3000 -C -B
root                653071              652954              0                   6月16                ?                   00:00:00            s6-supervise s6rc-fdholder
root                653072              652954              0                   6月16                ?                   00:00:00            s6-supervise svc-jellyfin
root                653073              652954              0                   6月16                ?                   00:00:00            s6-supervise s6rc-oneshot-runner
root                653081              653073              0                   6月16                ?                   00:00:00            /package/admin/s6/command/s6-ipcserverd -1 -- /package/admin/s6/command/s6-ipcserver-access -v0 -E -l0 -i data/rules -- /package/admin/s6/command/s6-sudod -t 30000 -- /package/admin/s6-rc/command/s6-rc-oneshot-run -l ../.. --
user                653197              653072              0                   6月16                ?                   00:11:38            /usr/bin/jellyfin --ffmpeg=/usr/lib/jellyfin-ffmpeg/ffmpeg
```


### docker version

显示 Docker 版本信息。

#### 用法
```shell
docker version [OPTIONS]
```


### docker volume

管理卷。

#### 用法
```shell
docker volume COMMAND
```

#### docker volume create

创建卷。

##### 用法
```shell
docker volume create [OPTIONS] [VOLUME]
```

##### 选项
|名称和缩写|默认值|描述|
|------|------|------|
|`--driver`,`-d`|local|指定卷驱动程序名称|

#### docker volume inspect

显示一个或多个卷的详细信息。

##### 用法
```shell
docker volume inspect [OPTIONS] VOLUME [VOLUME...]
```

#### docker volume ls

列出卷。

##### 用法
```shell
docker volume ls [OPTIONS]
```

##### 选项
|名称和缩写|默认值|描述|
|------|------|------|
|`--filter`,`-f`|local|提供过滤值（例如 dangling=true）|

#### docker volume rm

删除一个或多个卷。

##### 用法
```shell
docker volume rm [OPTIONS] VOLUME [VOLUME...]
```


## 参考链接

[Docker 命令行参考](https://docs.docker.com/engine/reference/run/)
[深入探究docker attach的退出方式](https://www.cnblogs.com/doujianli/p/10048707.html)