---
title: 如何使用 SSH 连接 Github 和服务器
date: 2023-06-29 04:36:35
tags:
    - ssh
---

## 使用 SSH 连接 Github

### 检查现有 SSH 密钥
打开终端，输入 `ls -al ~/.ssh` 以查看是否存在现有的 SSH 密钥。
```shell
$ ls -al ~/.ssh
total 16
drwx------ 2 wrmao wrmao 4096 Jun 28 20:19 .
drwxr-xr-x 6 wrmao wrmao 4096 Jun 28 20:13 ..
-rw------- 1 wrmao wrmao  106 Jun 28 20:07 authorized_keys
-rw-r--r-- 1 wrmao wrmao  444 Jun 28 20:19 known_hosts
```
检查目录列表以查看是否已经有 SSH 公钥。 默认情况下，GitHub 的一个支持的公钥的文件名是以下之一。
- id_rsa.pub
- id_ecdsa.pub
- id_ed25519.pub

### 生成 SSH 密钥
如果没有密钥，就需要生成新的 SSH 密钥；如果已有，跳到上传已有密钥环节。
打开终端，粘贴下面的文本（替换为你的 GitHub 电子邮件地址），这将以提供的电子邮件地址为标签创建新 SSH 密钥。
一直 yes 确定选择默认即可。
```shell
$ ssh-keygen -t ed25519 -C "your_email@example.com"
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/your-user/.ssh/id_ed25519):
```
### 上传 SSH 密钥
将 SSH 公钥复制到剪贴板，在 Github 上的 `Settings` - `Access` - `SSH and GPG keys` - `New SSH key`，粘贴即可。
```shell
$ cat ~/.ssh/id_ed25519.pub
```

### 配置 Git
```shell
$ git config --global user.name "moralok"
$ git config --global user.email "wrmao.public@outlook.com"
```

## SSH 的原理
### 关于 SSH
使用 SSH 协议可以连接远程服务器和服务并向它们验证，而无需在每次访问时都提供用户名和密码，Github 还可以使用 SSH 密钥对提交进行签名。

### 公钥和私钥
SSH 的使用（非对称加密）需要生成公钥 public key 和私钥 private key。常用的算法有 `rsa`、`ecdsa` 和 `ed25519`，相对应的公钥默认文件名即id_XXX.pub。`ed25519` 的安全性介于 `rsa 2048` 和 `rsa 4096` 之间，但性能却提升数十倍。
在生成密钥时，会要求你 `Enter passphrase (empty for no passphrase):`，可以输入一个口令保护私钥的使用。不为空的情况下，正常使用是需要输入这个口令的，很多人认为麻烦，因此留空。
公钥的权限必须是 644，私钥的权限必须是 600，否则 SSH 认为其不可靠。
私钥是要安全保管在客户端不能泄露的，公钥则要提供给远程服务器或服务。服务端的 `~/.ssh/authorized_keys` 里面存储着可以登录的客户端的公钥。我们将公钥粘贴到 Github 的过程就是对应于此。
```shell
$ ssh-keygen -t rsa -b 4096 -f my_id -C "email@example.com"
```
- `-t` 表示算法，如 `rsa`。
- `-b` 表示 `rsa` 密钥长度，默认 2048 bit，`ed25519` 不需要指定。
- `-f` 表示文件名。
- `-C` 表示在公钥文件中添加注释，可修改

### SSH 公钥登录过程
1. Client 将自己的公钥存放到服务端，追加到 authorized_keys 文件。
2. Server 收到 Client 的连接请求后，会在 authorized_keys 文件中匹配到 Client 传过来的公钥，并生成随机数 R，用公钥对随机数加密得到 pubKey(R)。
3. Client 收到后通过私钥解密得到随机数 R，然后对随机数 R 和本次会话的 sessionKey 使用 MD5 生成摘要 Digest1，发送给服务端。
4. Server 会对随机数 R 和会话的 sessionKey 同样使用 MD5 生成摘要 Digest2，对比相同即完成认证过程。

### 避免中间人攻击
SSH 通过口令确认避免中间人攻击，如果用户第一次登录 Server，系统会提示：
```shell
$ ssh -T git@github.com
The authenticity of host 'github.com (20.205.243.166)' can't be established.
ECDSA key fingerprint is SHA256:p2QAMXNIC1TJYWeIOttrVc98/R1BUFWu3/LiyKgUfQM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'github.com,20.205.243.166' (ECDSA) to the list of known hosts.
Hi ${username}! You've successfully authenticated, but GitHub does not provide shell access.
```
Server 需要在其网站上公示其公钥的指纹，Github 的公钥指纹[在这里](https://docs.github.com/zh/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)。
确认匹配后，客户端会在 `~/.ssh/known_hosts` 中记录，下次登录不再警告。

## 使用 SSH 免密登录服务器
使用现成的密钥，将 `~/.ssh/id_ed25519.pub` 的内容追加到自己服务端的 `~/.ssh/authorized_keys` 中，使用 `ssh user@host` 成功免密登录。


## 参考链接
[使用 SSH 进行连接 Github](https://docs.github.com/zh/authentication/connecting-to-github-with-ssh/about-ssh)
[Git 多台电脑共用SSH Key](https://blog.csdn.net/u012408797/article/details/116196831)
[SSH协议登录过程详解](https://blog.csdn.net/neo949332116/article/details/102926051)
[GitHub 的 SSH 密钥指纹](https://docs.github.com/zh/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)
[使用 Ed25519 算法生成你的 SSH 密钥](https://zhuanlan.zhihu.com/p/110413836)

