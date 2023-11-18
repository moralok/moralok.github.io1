---
title: 使用 OpenVPN 访问家庭内网
date: 2023-06-07 22:19:17
tags: 
    - openvpn
---

## 网络概况
宽带是电信宽带，分配了动态的公网 IP。
光猫使用桥接模式，通过路由器拨号上网（PPPoE），路由器局域网为 192.168.3.0/24。
一台 Windows 10 主机，在路由器局域网上的 IP 192.168.3.120。。
Windows 10 主机上运行 Vmware 虚拟机，网络采用 NAT 模式。子网为 192.168.46.0/24。运行了 Linux 主机，IP 192.168.46.128。Windows 10 主机在子网中的 IP 为 192.168.3.1。
|子网|Windows 10 主机|Linux 主机|客户端|
|------|------|------|------|
|路由器 192.168.3.0/24|192.168.3.120|-|-|
|Vmware NAT 192.168.46.0/24|192.168.46.1|192.168.46.128|-|
|VPN 10.8.0.0/24|10.8.0.1|-|10.8.0.6|



## 目标和考虑因素
1. 能从公网访问家庭内网，包括 Windows 10 主机和虚拟机上的 Linux 主机。
2. 不想通过路由器的 NAT 功能直接将路由器局域网上的设备映射到公网。一是为了安全，二是为了避免运营商审查。
3. 已经尝试过使用 ZeroTier，将所需设备组建在一个局域网当中，可以作为备选方案。同时不想每次新增设备都安装 ZeroTier。
4. 想利用公网 IP 以及上行带宽尝尝鲜。
5. 想要能直连虚拟机上的 Linux 主机，而不是通过 Vmware 的 NAT 映射。不想每次新增服务都要设置 NAT，修改 Windows Defender 的规则。

## 实现过程

### 在 Windows 10 上安装 OpenVPN 服务器
见{% post_link how-to-setup-OpenVPN-server-on-windows-10 '在 Windows 10 上安装 OpenVPN 服务器' %}。

### 在 iOS 和 macOS 上安装 OpenVPN 客户端
见{% post_link how-to-setup-OpenVPN-connect-client-on-iOS-and-macOS '在 iOS 和 macOS 上安装 OpenVPN 客户端' %}。

### 客户端访问服务端其他的私有子网
如果在所需的每一个设备上都安装 OpenVPN，将它们连接在 VPN 的子网 10.8.0.0/24 中，也是可以满足需求的，但是这和每个设备都安装 ZeroTier 差不多。

#### server.ovpn 新增配置
在 `server.ovpn` 配置文件中新增一行配置，这个配置的意思是将该路由配置统一推送给客户端，让它们可以访问服务端的其他私有子网。相当于将服务端的其他私有子网的情况告知客户端，这样客户端就知道发往 192.168.46.128 的 Packet 是发向哪里的。
```ini
push "route 192.168.46.0 255.255.255.0"
```

#### 打开 Windows 10 主机的路由转发功能
1. `Win + R` 输入 regedit 打开注册表。
2. 找到 `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters`，修改 `IPEnableRouter` 为 1。
3. 重启

#### 为虚拟机上的 Linux 主机新增路由
在终端中输入命令：
```
route add -net 10.8.0.0/24 gw 192.168.46.1
```
这是为了让 OpenVPN 服务端的其他私有子网上的设备知道来自 10.8.0.0/24 的 IP Packet 应该路由回 OpenVPN 服务端。

## 参考链接
[透过openvpn来访问内网资源](https://blog.51cto.com/richie/389636)
[OpenVPN 路由详解](https://limbo.moe/posts/2018/openvpn-routes)
[OpenVPN中的另一个路由问题 - 在VPN上无法访问本地计算机](https://qastack.cn/superuser/865302/yet-another-routing-issue-in-openvpn-cannot-access-local-machines-while-on-vpn)
[openvpn添加本地路由表](https://www.nixops.me/articles/openvpn-add-local-routing-table.html)
[windows开启路由转发](https://blog.csdn.net/qq_43615820/article/details/113660623)
[linux route命令的使用详解](https://www.cnblogs.com/snake-hand/p/3143041.html)
[Windows命令行route命令使用图解](https://blog.csdn.net/bcbobo21cn/article/details/52548923)