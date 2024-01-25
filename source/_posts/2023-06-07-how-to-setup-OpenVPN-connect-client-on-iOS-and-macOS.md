---
title: 在 iOS 和 macOS 上安装 OpenVPN 客户端
date: 2023-06-07 17:04:23
tags: [openvpn, proxy]
---

本文记录了如何在 `iOS` 和 `macOS` 上安装和配置 `OpenVPN` 客户端，主要介绍如何编写客户端配置文件 `client.ovpn` 并导入。

<!-- more -->

- 服务端的安装：{% post_link 'how-to-setup-OpenVPN-server-on-windows-10' 在 Windows 10 上安装 OpenVPN 服务端 %}
- 主要目的：{% post_link 'how-to-use-OpenVPN-to-access-home-network' 使用 OpenVPN 访问家庭内网 %}


## 安装 OpenVPN Connect

- `macOS` 访问[官网下载](https://openvpn.net/client-connect-vpn-for-mac-os/)。
- `iOS` 访问 [AppStore](https://itunes.apple.com/us/app/openvpn-connect/id590379981?mt=8)，需要登录外区 `Apple ID`。


## 配置 OpenVPN Connect

> 客户端配置文件模板 `client.ovpn` 以及 `ca.crt`（`ca`），`client.crt`（`cert`），`client.key`（`key`）等文件均在安装 `OpenVPN` 服务端时获得。

客户端提供了两种方式导入配置文件：

1. 通过 `URL`，建议 `URL` 仅限在私有网络内访问。
2. 通过其他方式例如邮件（安全性降低），下载为本地文件再导入。本人使用 `OneDrive` 共享到 `iPhone`。

对于客户端配置而言，`iOS` 的困难点在于其文件系统封闭，`ca.crt`（`ca`），`client.crt`（`cert`），`client.key`（`key`）不能放置到指定位置。因此配置文件分为两种形式：

1. 将 `CA` 根证书 `ca.crt`，客户端证书 `client.crt`，客户端密钥 `client.key` 的内容复制粘贴到 `client.ovpn` 中，形成一个联合配置文件。这种方式简单方便，推荐！。
2. 使用 `openssl` 将 `CA` 根证书 `ca.crt`，客户端证书 `client.crt`，客户端密钥 `client.key` 转换为 `PKCS#12` 文件，先导入 `client.ovpn12` 再导入 `client.ovpn`。**不推荐的原因在于本人导入失败**，最终放弃。

### 单一 client.ovpn

1. 从目录 `C：\Program Files\OpenVPN\sample-config` 复制客户端配置文件模板 `client.ovpn`，修改配置
2. 将 `remote your-server 1194` 中的地址和端口替换成你的 `OpenVPN` 服务端对外的地址和端口
3. 将 `ca ca.crt`，`cert client.crt`，`key client.key`，`tls-auth ta.key 1` 注释掉，再将各自文件中的内容以类 `XML` 的形式粘贴到 `client.ovpn` 中
4. 将修改好的客户端配置文件导入到客户端中

```ini
remote your-server 1194

;ca ca.crt
;cert client.crt
;key client.key

;tls-auth ta.key 1

<ca>
-----BEGIN CERTIFICATE-----
paste contents of ca.crt
-----END CERTIFICATE-----
</ca>
<cert>
-----BEGIN CERTIFICATE-----
paste contents of client.crt
-----END CERTIFICATE-----
</cert>
<key>
-----BEGIN PRIVATE KEY-----
paste contents of client.key
-----END PRIVATE KEY-----
</key>
```

### client.ovpn + client.opvn12（失败）

1. 使用 `openssl` 命令将客户端的证书和密钥文件转换为 `PKCS#12` 形式的文件。该命令会提示 `Enter Export Password`，可以为空，但为了安全建议设置密码。
```shell
openssl pkcs12 -export -in cert -inkey key -certfile ca -name MyClient -out client.ovpn12
```
2. 由于在 `iOS` 中导入 `PKCS#12` 文件到 `Keychain` 中时只导入了客户端证书和密钥，`CA` 根证书并没有导入，`client.ovpn` 文件中必须要保留 `CA`根证书的配置。
既可以用传统的引用文件的方式：
```ini
ca ca.crt
```
也可以用类 XML 的形式粘贴 ca.crt 内容到 client.ovpn 中：
```ini
<ca>
paste contents of ca.crt here
</ca>
```
3. 先导入 `client.ovpn12`（需要输入转换时的密码），再导入 `client.ovpn`。

> 但是我失败了……导入 `client.ovpn12` 时密码一直错误，没有解决。推荐使用第一种方式。


## NAT 和 DDNS

本人的设置如下，仅供参考：

- `OpenVPN` 服务端配置的端口号为默认的 `1194`。
- 在路由器管理后台的 `NAT` 设置中，配置一个自定义的高位端口号如 `49999` 作为对外端口号映射到 `Windows 10` 主机的 `1194` 端口号。这既是为了安全，也是为了避免不必要的检测风险（有没有用我也不知道啊 =_= ）。
- 由于公网 `IP` 是动态的，一旦 `IP` 发生变化，就需要修改配置文件。因此使用 `ddns-go` 配合 `Cloudflare` 实现动态域名解析。


## 参考文章

- [iOS 使用 OpenVPN 的 FAQ](https://openvpn.net/vpn-server-resources/faq-regarding-openvpn-connect-ios/)
- [如何通过 iOS Keychain 使用客户端证书和密钥](https://openvpn.net/faq/how-do-i-use-a-client-certificate-and-private-key-from-the-ios-keychain/)
- [如何配置 iOS OpenVPN 客户端的证书认证](https://help.endian.com/hc/en-us/articles/360008350974-How-to-configure-iOS-OpenVPN-client-with-certificate-authentication)