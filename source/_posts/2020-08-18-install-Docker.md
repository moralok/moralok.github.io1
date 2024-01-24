---
title: 安装 Docker
date: 2020-08-18 15:57:49
tags: [docker]
---

记录安装 `Docker` 的过程用于备忘，主要在新建虚拟机或重装云服务器系统时使用。

<!-- more -->

## 在 Ubuntu 上安装 Docker

官方文档：[Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

1. 卸载旧版本
```shell
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```
2. 设置 `apt` 仓库
```shell
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
3. 安装最新版本
```shell
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
4. 通过运行 `hello-world` 镜像验证安装成功
```shell
sudo docker run hello-world
```

## 在 Linux 上安装 Docker 后的步骤

官方文档：[Linux post-installation steps for Docker Engine](https://docs.docker.com/engine/install/linux-postinstall/)

### 以非 root 用户身份管理 Docker

1. 创建 `docker` 群组
```shell
sudo groupadd docker
```
2. 将当前用户添加到 `docker` 群组
```shell
sudo usermod -aG docker $USER
```
3. 登出再登录，或者通过以下命令切换群组登录
```shell
newgrp docker
```
4. 验证可以不通过 `sudo` 执行 `docker` 命令
```shell
docker run hello-world
```

### 配置 Docker 通过 systemd 启动

1. 设置开机自动启动 `Docker` 
```shell
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```
2. 关闭开机自动启动
```shell
sudo systemctl disable docker.service
sudo systemctl disable containerd.service
```