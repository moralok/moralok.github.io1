---
title: 在 Ubuntu 上安装 Clash
date: 2023-05-27 16:07:01
tags: [clash, proxy]
---

本文记录了如何在 `Ubuntu` 上安装 `Clash`，以供各类程序在必要的时候使用代理。
有些时候我们面对特定资源会遇到下载速度极其缓慢甚至无法下载的情况，像是使用 `Github`，下载 `k8s` 相关镜像，还有访问特定网站等等。也许有些 Dirty Hack 的方式可以短暂地绕开限制，比如修改 `hosts` 文件，但这并不总是见效。也许添加国内的镜像仓库地址可以满足很多人的需求，但是说不准就会遇到镜像更新不及时甚至镜像被污染的情况。总之，如果你有一把不错的梯子，使用代理辅助开发肯定会拥有更好的体验。
当然，为 `Git` 设置代理，为 `Terminal` 设置代理，为 `Docker Engine` 设置代理，为 `Docker` 容器设置代理，为 **“还有一些你尚未知道原来这还需要这样设置代理的地方”** 设置代理，仍然是一件痛苦的事情。如果你有条件为全屋设备配置透明代理，肯定能避免踩非常多的坑。

<!-- more -->

> **注意：多个 `Clash` 相关的仓库已经 GG**
> **请谨慎甄别网上搜索到的所谓备份的可靠性和安全性！！！**
> **请谨慎甄别网上搜索到的所谓备份的可靠性和安全性！！！**
> **请谨慎甄别网上搜索到的所谓备份的可靠性和安全性！！！**

---

## 通过 Docker 运行 Clash

> 本人最新的安装是通过直接拷贝旧服务器上的二进制可执行文件以及全球 `IP` 库文件来完成的，保留已有安装的相关文件在短时间内仍然可以满足重新安装的需求。**注意到原作者 `Dreamacro` 的 `Docker` 镜像仓库仍然保留，也许通过 `Docker` 运行 `Clash` 是个更好的选择**。

- `docker-compose.yml`
```yaml
version: "1.0"
services:
  clash:
    image: dreamacro/clash:v1.18.0
    container_name: clash
    ports:
      - 9090:9090
      - 7890:7890
    mem_limit: ${CLASH_MEM_LIMIT:-64m}
    restart: unless-stopped
```
- 配置文件在容器内的位置 `/root/.config/clash/config.yaml`

> 长远来看，使用无法得到更新的旧版本仍然可能在未来导致安全问题的发生，以后再说吧 =_=。

---

## 二进制安装 Clash

> 本方式仅限仍然可获得可靠安全的二进制可执行文件的人使用。

### 安装 Clash

1. ~~从 [GitHub](https://github.com/Dreamacro/clash/releases) 下载预构建的二进制文件。~~（ **仓库已删除** ）
```shell
$ wget https://github.com/Dreamacro/clash/releases/download/v1.16.0/clash-linux-amd64-v1.16.0.gz
```
2. 使用 `gzip` 解压压缩包 `clash-linux-amd64-v1.16.0.gz` 得到 `clash-linux-amd64-v1.16.0`。忽略提示。
```shell
$ gzip -d clash-linux-amd64-v1.16.0.gz 
gzip: clash-linux-amd64-v1.16.0: Value too large for defined data type
```
3. 移动二进制文件到目录 `/usr/local/bin` 并且重命名为 `clash`。
```shell
$ sudo mv clash-linux-amd64-v1.16.0 /usr/local/bin/clash
```
4. 通过 `clash -v` 查看 `Clash` 的版本信息。
```shell
$ clash -v
Clash v1.16.0 linux amd64 with go1.20.4 Fri May 19 13:57:32 UTC 2023
```

### 使用 Clash

1. 使用命令 `clash` 启动，可以看到日志。
```shell
:~$ clash
INFO[0000] Can't find config, create a initial config file 
INFO[0000] Can't find MMDB, start download              
INFO[0003] Mixed(http+socks) proxy listening at: 127.0.0.1:7890
```
2. ~~在第一次启动 `Clash` 时，`Clash` 会在 `~/.config` 下创建目录 `clash`，并在其中创建 `3` 个文件。其中 `config.yaml` 是 `Clash` 的配置文件，`Country.mmdb` 是全球 `IP` 库，可以实现各个国家的 `IP` 信息解析和地理定位。`cache.db` 用于缓存。~~（ **`CDN` 已失效** ）
```shell
:~/.config/clash$ ls
cache.db  config.yaml  Country.mmdb
```
3. 如果梯子的订阅支持 `Clash`，直接订阅；否则需要手动修改配置文件。
4. 可以浏览器访问 http://clash.razord.top/#/proxies 选择代理服务器。（ **注意：使用 `http`** ）

### 设置 Clash 开机自动启动

> ~~参考官方文档 [Clash as a service](https://dreamacro.github.io/clash/introduction/service.html)。~~（ **已 `404`** ）

1. 拷贝配置文件和全球 `IP` 库到 `/etc/clash`
```shell
~/.config/clash$ sudo cp config.yaml /etc/clash/
~/.config/clash$ sudo cp Country.mmdb /etc/clash/
```
2. 在 `/etc/systemd/system/clash.service` 创建 `systemd` 配置文件
```shell
[Unit]
Description=Clash daemon, A rule-based proxy in Go.
After=network-online.target

[Service]
Type=simple
Restart=always 
ExecStart=/usr/local/bin/clash -d /etc/clash

[Install]
WantedBy=multi-user.target
```
3. 重载 `systemd`
```shell
systemctl daemon-reload
```
4. 设置系统开机时自动启动 `Clash`
```shell
systemctl enable clash
```
5. 启动 `Clash`
```shell
systemctl start clash
```
6. 检查 `Clash` 的健康状态和日志
```shell
systemctl status clash
journalctl -xe
```