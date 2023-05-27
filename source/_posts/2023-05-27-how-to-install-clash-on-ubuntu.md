---
title: 在 Ubuntu 上安装 Clash
date: 2023-05-27 16:07:01
tags:
---

## 安装 Clash
从 [GitHub](https://github.com/Dreamacro/clash/releases) 下载预构建的二进制文件。
```shell
:/mnt/hgfs/share$ wget https://github.com/Dreamacro/clash/releases/download/v1.16.0/clash-linux-amd64-v1.16.0.gz
```

使用 gzip 解压压缩包 clash-linux-amd64-v1.16.0.gz 得到 clash-linux-amd64-v1.16.0。
```shell
:/mnt/hgfs/share$ gzip -d clash-linux-amd64-v1.16.0.gz 
gzip: clash-linux-amd64-v1.16.0: Value too large for defined data type
```
忽略提示。

移动二进制文件到目录 `/usr/local/bin` 并且重命名为 clash。
```shell
:/mnt/hgfs/share$ sudo mv clash-linux-amd64-v1.16.0 /usr/local/bin/clash
[sudo] password for wrmao:
```

现在可以通过 `clash -v` 查看 Clash 的版本信息。
```shell
:~$ clash -v
Clash v1.16.0 linux amd64 with go1.20.4 Fri May 19 13:57:32 UTC 2023
```

## 使用 Clash
使用命令 `clash` 启动，可以看到日志。
```shell
:~$ clash
INFO[0000] Can't find config, create a initial config file 
INFO[0000] Can't find MMDB, start download              
INFO[0003] Mixed(http+socks) proxy listening at: 127.0.0.1:7890
```

在第一次启动 Clash 时，Clash 会在 `~/.config` 下创建目录 clash，并在其中创建 3 个文件。
```shell
:~/.config/clash$ ls
cache.db  config.yaml  Country.mmdb
```
其中 config.yaml 是 Clash 的配置文件，Country.mmdb 是全球 IP 库，可以实现各个国家的 IP 信息解析和地理定位。
config.yaml 的内容直接从已有的配置文件复制过来。

可以浏览器访问 http://clash.razord.top/#/proxies 选择代理服务器。

## 设置 Clash 为后台服务
参考官方文档 [Clash as a service](https://dreamacro.github.io/clash/introduction/service.html)。

拷贝配置文件到 `/etc/clash`。
```shell
:~/.config/clash$ sudo cp config.yaml /etc/clash/
:~/.config/clash$ sudo cp Country.mmdb /etc/clash/
```

在 `/etc/systemd/system/clash.service` 创建 systemd 配置文件：
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

之后重载 systemd：
```shell
systemctl daemon-reload
```

设置系统开机时启动 clashd：
```shell
systemctl enable clash
```

马上启动 clashd：
```shell
systemctl start clash
```

检查 Clash 的健康状态和日志：
```shell
systemctl status clash
journalctl -xe
```