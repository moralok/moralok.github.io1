---
title: 如何为终端、docker 和容器设置代理
date: 2023-06-13 22:56:11
tags: 
    - docker
    - proxy
---

在 {% post_link how-to-install-clash-on-ubuntu 'Ubuntu 上安装 Clash' %} 后，Clash 通过监听本地的 7890 端口，提供代理服务。但是不同程序设置代理的方式不尽相同，并不是启动了 Clash 以及在某一处设置后，整个系统发出的 HTTP 请求都能经过代理。本文将介绍如何为终端、docker 和容器添加代理。

## 为终端设置代理
有时候，我们需要在终端通过执行命令的方式访问网络和下载资源，比如使用 `wget` 和 `curl`。

### 设置 Shell 环境变量
这一类软件都是可以通过为 Shell 设置环境变量的方式来设置代理，涉及到的环境变量有 `http_proxy`、`https_proxy` 和 `no_proxy`。
仅为当前会话设置，执行命令：
```shell
export http_proxy=http://proxyAddress:port
export https_proxy=http://proxyAddress:port
export no_proxy=localhost,127.0.0.1
```
永久设置代理，在设置 Shell 环境变量的脚本中（不同 Shell 的配置文件不同，比如 `~/.bashrc` 或 `~/.zshrc`）添加：
```shell
export http_proxy=http://proxyAddress:port
export https_proxy=http://proxyAddress:port
export no_proxy=localhost,127.0.0.1
```
重新启动一个会话或者执行命令 `source ~/.bashrc` 使其在当前会话立即生效。

### 修改 wget 配置文件
在搜索过程中发现还可以在 `wget` 的配置文件 `~/.wgetrc` 中添加:
```shell
use_proxy = on

http_proxy = http://proxyAddress:port
https_proxy = http://proxyAddress:port
```

## 为 docker 设置代理
如果你以为为终端设置代理后 docker 就会使用代理，那你就错了。在从官方的镜像仓库 pull 镜像反复出错后并收到类似 `Error response from daemon: Get "https://registry-1.docker.io/v2/": read tcp 192.168.3.140:59460->44.205.64.79:443: read: connection reset by peer` 这样的报错信息后，我才开始怀疑我并没有真正给 docker 设置好代理。
在执行 `docker pull` 命令时，实际上命令是由守护进程 `docker daemon` 执行的。

### 通过 systemd 设置
如果你的 `docker daemon` 是通过 `systemd` 管理的，那么你可以通过设置 `docker.service` 服务的环境变量来设置代理。
执行命令查看 `docker.service` 信息，得知配置文件位置 `/lib/systemd/system/docker.service`。
```shell
~$ systemctl status docker.service 
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2023-06-13 00:52:54 CST; 22h ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 387690 (dockerd)
      Tasks: 139
     Memory: 89.6M
        CPU: 1min 26.512s
     CGroup: /system.slice/docker.service
```
在 `docker.service` 的 `[Service]` 模块添加：
```shell
Environment=HTTP_PROXY=http://proxyAddress:port
Environment=HTTPS_PROXY=http://proxyAddress:port
Environment=NO_PROXY=localhost,127.0.0.1
```
重新加载配置文件并重启服务：
```shell
systemctl daemon-reload
systemctl restart docker.service
```

### 修改 dockerd 配置文件
还可以修改 `dockerd` 配置文件，添加：
```shell
export http_proxy="http://proxyAddress:port"
```
然后重启 `docker daemon` 即可。

> 国内的镜像仓库在绝大多数时候都可以满足条件，但是存在个别镜像同步不及时的情况，如果使用 latest 标签拉取到的镜像并非近期的镜像，因此有时候需要直接从官方镜像仓库拉取镜像。

## 为 docker 容器设置代理
为 `docker daemon` 进程设置代理和为 docker 容器设置代理是有区别的。比如使用 docker 启动媒体服务器 jellyfin 后，jellyfin 的刮削功能就需要代理才能正常使用，这时候不要因为在很多地方设置过代理就以为容器内部已经在使用代理了。

### 修改配置文件
创建或修改 `~/.docker/config.json`，添加：
```json
{
    "proxies":
    {
        "default":
        {
            "httpProxy": "http://proxyAddress:port",
            "httpsProxy": "http://proxyAddress:port",
            "noProxy": "localhost,127.0.0.1"
        }
    }
}
```
此后创建的新容器，会自动设置环境变量来使用代理。
### 为指定容器添加环境变量
在启动容器时使用 `-e` 手动注入环境变量 `http_proxy`。这意味着进入容器使用 `export` 设置环境变量的方式也是可行的。

> 注意：如果代理是使用宿主机的代理，当网络为 `bridge` 模式，proxyAddress 需要填写宿主机的 IP；如果使用 `host` 模式，proxyAddress 可以填写 127.0.0.1。

## 总结
不要因为在很多地方设置过代理，就想当然地以为当前的访问也是经过代理的。每个软件设置代理的方式不尽相同，但是大体上可以归结为：
1. 使用系统的环境变量
2. 修改软件的配置文件
3. 执行时注入参数

举一反三，像 `apt` 和 `git` 这类软件也是有其设置代理的方法。当你的代理稳定但是相应的访问失败时，大胆假设你的代理没有设置成功。要理清楚，当前的访问是谁发起的，才能正确地使用关键词搜索到正确的设置方式。

> 原本我在 docker 相关的使用中，有关代理的设置方式是通过修改配置文件，实现永久、全局的代理配置。但是在后续的使用中，发现代理在一些场景（比如使用 cloudflare tunnel）中会引起不易排查的问题，决定采用临时、局部的配置方式。

## 参考链接
[Linux 让终端走代理的几种方法](https://zhuanlan.zhihu.com/p/46973701)
[Linux ❀ wget设置代理](https://blog.51cto.com/u_14814563/5415887)
[配置Docker使用代理](https://cloud-atlas.readthedocs.io/zh_CN/latest/docker/network/docker_proxy.html#docker-server-proxy)
[Docker的三种网络代理配置](https://note.qidong.name/2020/05/docker-proxy/)
[docker 设置代理，以及国内加速镜像设置](https://neucrack.com/p/286)