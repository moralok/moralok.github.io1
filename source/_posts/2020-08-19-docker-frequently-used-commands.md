---
title: Docker 常用命令
date: 2020-08-19 22:33:24
tags: [docker]
---

根据个人使用经验从 `Docker` 官方命令行参考中选取常用的命令作为备忘清单。

<!-- more -->

## 容器相关

| 命令 | 补充 |描述|
|------|------|------|
| `docker run -d --name=CONTAINER -p 1080:80 IMAGE:TAG` | `-e` `-v` `--network` `--restart` `-it` | 创建容器 |
| `docker ps \| grep CONTAINER` | `-a` `-q` | 列出容器 |
| `docker stop CONTAINER` |  | 停止容器 |
| `docker rm CONTAINER` |  | 删除容器 |
| `docker start CONTAINER` |  | 启动已停止容器 |
| `docker restart CONTAINER` |  | 重启容器 |
| `docker logs -f CONTAINER` | `-n` `-t` `--since` `--until` | 获取容器日志 |
| `docker exec CONTAINER ls` | `-d` `-e` `-w` | 在容器内执行命令 |
| `docker exec -it CONTAINER /bin/bash` |  | 进入容器 |
| `docker inspect CONTAINER` | `-f` | 查看容器信息 |
| `docker top CONTAINER` |  | 显示容器中的运行进程 |
| `docker cp CONTAINER:/app/source /app/target` | 双向 目录或文件 | 在宿主机和容器间拷贝文件 |
| `docker attach CONTAINER` | 使用 `CTRL-p CTRL-q` 退出 | 附着到容器 |
| `docker port CONTAINER` | `7890/tcp` | 列出容器的端口映射 |
| `docker rename CONTAINER NEW_NAME` |  | 重命名容器 |


## 镜像相关

| 命令 | 补充 |描述|
|------|------|------|
| `docker images` | `-a` | 列出镜像 |
| `docker pull IMAGE:TAG` |  | 拉取镜像 |
| `docker rmi IMAGE:TAG` |  | 删除镜像 |
| `docker build -t IMAGE:TAG .` |  | 构建镜像 |
| `docker search IMAGE` | 去官网方便 | 搜索镜像 |


## 网络相关

| 命令 | 补充 |描述|
|------|------|------|
| `docker network ls` |  | 列出网络 |
| `docker network create NETWORK` |  | 创建网络 |
| `docker network rm NETWORK` |  | 删除网络 |
| `docker network inspect NETWORK` |  | 查看网络信息 |
| `docker network connect NETWORK CONTAINER` |  | 将容器连接到网络 |
| `docker network disconnect NETWORK CONTAINER` |  | 将容器从网络断开 |


## 卷相关

| 命令 | 补充 |描述|
|------|------|------|
| `docker volume ls` |  | 列出卷 |
| `docker volume create VOLUME` |  | 创建卷 |
| `docker volume rm VOLUME` |  | 删除卷 |
| `docker volume inspect VOLUME` |  | 查看卷信息 |


## Docker 相关

| 命令 | 补充 |描述|
|------|------|------|
| `docker stats` | `-a` | 显示容器资源使用统计 |
| `docker info` | `-f` | 显示 Docker 信息 |
| `docker version` |  | 显示 Docker 版本 |

## systemd

| 命令 | 补充 |描述|
|------|------|------|
| `systemctl start docker` |  | 启动 Docker 服务 |
| `systemctl stop docker` |  | 停止 Docker 服务 |
| `systemctl restart docker` |  | 重启 Docker 服务 |
| `systemctl status docker` |  | 查看 Docker 服务状态 |
| `systemctl enable docker` |  | 设置 Docker 服务开启自启动 |


## Docker Compose

Define and run multi-container applications with Docker.

Usage:  `docker compose [OPTIONS] COMMAND`

| 命令 | 补充 |描述|
|------|------|------|
| `docker compose ls` |  | List running compose projects |
| `docker compose up` | `-d` | Create and start containers |
| `docker compose down` |  | Stop and remove containers, networks |
| `docker compose start` |  | Start services |
| `docker compose stop` |  | Stop services |
| `docker compose restart` |  | Restart service containers |
| `docker compose logs` | `-f` | View output from containers |
| `docker compose ps` |  | List containers |
| `docker compose images` |  | List images used by the created containers |
| `docker compose top` |  | Display the running processes |
| `docker compose version` |  | Show the Docker Compose version information |


## 参考链接

[Docker 命令行参考](https://docs.docker.com/engine/reference/run/)
[深入探究docker attach的退出方式](https://www.cnblogs.com/doujianli/p/10048707.html)