---
title: 在 Windows 10 上安装 OpenVPN 服务器
date: 2023-06-07 15:18:02
tags: 
    - openvpn
---

## 安装 OpenVPN server
从 [OpenVPN 社区](https://openvpn.net/community-downloads/) 下载 Windows 64-bit MSI installer。本次安装的版本为 OpenVPN 2.6.4。

> 注意事项：在选择安装类型时选择 Customize 而不要选择 Install Now。额外勾选 OpenVPN -> OpenVPN Service -> Entire feature will be installed on local hard drive 和 OpenSSL Utilities -> EasyRSA 3 Certificate Management Scripts -> Entire feature will be installed on local hard drive。

安装完毕后，会弹出一条消息提示未找到可读的连接配置文件，暂时忽略。
此时在 控制面板\网络和 Internet\网络连接 中可以看到创建了两个新的网络适配器 OpenVPN TAP-Windows6 和 OpenVPN Wintun。

## 配置 OpenVPN server
打开 Windows 10 终端程序。
进入 OpenVPN 默认安装目录中的 easy-rsa 目录。
```shell
cd 'C:\Program Files\OpenVPN\easy-rsa'
```
执行命令进入 Easy-RSA 3 Shell
```shell
.\EasyRSA-Start.bat
```
初始化公钥基础设施目录 pki
```shell
./easyrsa init-pki
```
构建证书颁发机构（CA）密钥，CA 根证书文件将在后续用于对其他证书和密钥进行签名。该命令要求输入 Common Name，输入主机名即可。创建的 ca.crt 保存在目录 `C：\Program Files\OpenVPN\easy-rsa\pki` 中，ca.key 保存在目录 `C：\Program Files\OpenVPN\easy-rsa\pki\private` 中。
```shell
./easyrsa build-ca nopass
```
构建服务器证书和密钥。创建的 server.crt 保存在目录 `C：\Program Files\OpenVPN\easy-rsa\pki\issued` 中，server.key 保存在目录 `C：\Program Files\OpenVPN\easy-rsa\pki\private` 中。
```shell
./easyrsa build-server-full server nopass
```
构建客户端证书和密钥。创建的 client.crt 保存在目录 `C：\Program Files\OpenVPN\easy-rsa\pki\issued` 中，client.key 保存在目录 `C：\Program Files\OpenVPN\easy-rsa\pki\private` 中。
```shell
./easyrsa build-client-full client nopass
```
生成 Diffie-Hellman 参数
```shell
./easyrsa gen-dh
```
从目录 `C：\Program Files\OpenVPN\sample-config` 复制服务端配置文件模板 server.ovpn 到目录 `C：\Program Files\OpenVPN\config` 中，修改以下配置：
```ini
port 1194
dh dh.pem
duplicate-cn
;tls-auth ta.key 0
```
端口号按需修改，默认为1194，需要保证 OpenVPN 的网络流量可以通过防火墙，设置 Windows 10 Defender 允许 OpenVPN 通过即可。dh2048.pem 修改为生成的文件名 dh.pem。取消注释 duplicate-cn，让多个客户端使用同一个客户端证书。注释掉 tls-auth ta.key 0。复制 ca.crt，dh.pem，server.crt 和 server.key 到目录 C：\Program Files\OpenVPN\config 中。

## 启动与连接
启动 OpenVPN，点击连接，系统提示分配 IP 10.8.0.1。按配置，每次 OpenVPN server 都将为自己分配 10.8.0.1。

## 参考链接
[openvpn安装配置说明(windows系统)](https://blog.eyyyye.com/article/39#menu_3)
[如何在Windows 10上安装和配置OpenVPN](https://zhuanlan.zhihu.com/p/525197398)