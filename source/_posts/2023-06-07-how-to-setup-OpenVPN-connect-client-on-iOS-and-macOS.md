---
title: 在 iOS 和 macOS 上安装 OpenVPN 客户端
date: 2023-06-07 17:04:23
tags: OpenVPN
---

## 安装 OpenVPN Connect
macOS 访问[官网下载](https://openvpn.net/client-connect-vpn-for-mac-os/)。
iOS 访问 [AppStore](https://itunes.apple.com/us/app/openvpn-connect/id590379981?mt=8)，需要登录外区 Apple ID。

## 配置 OpenVPN Connect
客户端提供了两种方式导入配置文件，一是通过 URL，建议 URL 仅限在私有网络内访问，二是通过其他方式例如邮件，下载为本地文件再导入。

配置文件的组织方式又分为两种形式，一种是将 CA 根证书 ca.crt，客户端证书 client.crt，客户端密钥 client.key 的内容复制粘贴到 client.ovpn 中，形成一个联合配置文件；另一种是使用 openssl 将 CA 根证书 ca.crt，客户端证书 client.crt，客户端密钥 client.key 转换为 PKCS#12 文件，先后导入 client.ovpn12 和 client.ovpn。

### 单一 client.ovpn
从目录 `C：\Program Files\OpenVPN\sample-config` 复制客户端配置文件模板 client.ovpn，修改以下配置：
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
将 `remote your-server 1194` 中的地址和端口替换成你的 OpenVPN server 的地址和端口。将 `ca ca.crt`， `cert client.crt`， `key client.key`， `tls-auth ta.key 1` 注释掉，将各自文件中的内容以上述类 XML 的形式粘贴到 client.ovpn 中。

将修改好的客户端配置文件导入到客户端中即可。

### client.ovpn + client.opvn12
使用 openssl 命令将客户端的证书和私钥文件转换为 PKCS#12 形式的文件。该命令会提示 `Enter Export Password`，可以为空，但为了安全建议设置密码。
```shell
openssl pkcs12 -export -in cert -inkey key -certfile ca -name MyClient -out client.ovpn12
```
由于在 iOS 中导入 PKCS#12 文件到 Keychain 中时只导入了客户端证书和密钥，CA 根证书并没有导入，client.ovpn 文件中必须要保留 CA 根证书的配置。
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
先导入 client.ovpn12（需要输入转换时的密码），再导入 client.ovpn。

> 但是我失败了……导入 client.ovpn12 时密码一直错误，搜索到类似的案例，但是没有找到解决方案。不确定是不是 openssl 版本引起的。

## 路由器 NAT
在路由器管理后台的 NAT 设置功能里，配置好对外端口号和 Windows 10 主机上 OpenVPN 端口号的映射关系。

## 参考链接
[iOS 使用 OpenVPN 的 FAQ](https://openvpn.net/vpn-server-resources/faq-regarding-openvpn-connect-ios/)
[如何通过 iOS Keychain 使用客户端证书和密钥](https://openvpn.net/faq/how-do-i-use-a-client-certificate-and-private-key-from-the-ios-keychain/)
[如何配置 iOS OpenVPN 客户端的证书认证](https://help.endian.com/hc/en-us/articles/360008350974-How-to-configure-iOS-OpenVPN-client-with-certificate-authentication)