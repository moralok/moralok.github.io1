---
title: Nginx 反向代理在家庭网络中的应用
date: 2023-12-01 10:31:05
tags: [nginx, reverse proxy]
---

原先在使用 `Cloudflare Tunnel` 访问家庭网络中的服务时，是直接将域名解析到相应服务。尽管 `Cloudflare` 已经提供相关的请求统计和安全防护功能，部分服务自身也有访问日志，但是为了更好地监控和跟踪对外服务的使用情况，采集 `Cloudlfare` 统计中缺少的新，决定使用 `Nginx` 反向代理内部服务，统一内部服务的访问入口。简而言之就是，又折腾一些有的没的。以上修改带来的一个附加好处是在局域网内访问服务时，通过在 `hosts` 文件中添加域名映射，可以用更加容易记忆的域名代替 `IP + port` 的形式去访问。

<!-- more -->

> `Cloudflare Tunnel` 相较于 `Zerotier` 和 `OpenVPN`，尽管它们三者都能避免直接开放家庭网络，但前者可以让用户直接使用域名访问到局域网中的服务，便于分享。但它的速度和延迟并不理想，还有人反馈存在网络不稳定的现象，但作为个人玩具还是够用的。有朋友使用公网服务器配合打洞软件和家庭网络中的服务器组网，实现相同目标的同时效果更好。

## 网络结构示意图

客户端发起请求，请求经 `Cloudflare` 转发到局域网中的 `Tunnel`。原先，`Tunnel` 如虚线箭头所示，直接将请求发向目标服务，如今改为发向 `Nginx`，由 `Nginx` 反向代理，发向目标服务。

{% asset_img "Pasted image 20231201202859.png" 网络结构示意图 %}

## 配置

### docker-compose.yml

`Nginx` 和 `Tunnel` 还有其他内部服务应处于同一个网络中。

```yaml
version: "1.0"
services:
  nginx:
    image: nginx:1.25.3
    container_name: nginx
    ports:
      - 80:80
    volumes:
      - $PWD/etc/nginx/conf.d:/etc/nginx/conf.d
      - $PWD/etc/nginx/nginx.conf:/etc/nginx/nginx.conf
      - $PWD/log:/var/log/nginx
    networks:
      - my-network
    restart: unless-stopped

networks:
  my-network:
    external: true
```

### /etc/nginx/nginx.conf

在最后新增了拒绝未匹配成功的域名，在 `Cloudflare Tunnel` 的使用场景中，其实用处不大，因为未经配置的域名也无法解析到 `Nginx` 服务。

```conf
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;

    #以上是默认的配置内容，新增拒绝未匹配成功的域名
    server {
        listen 80 default_server;
        server_name _;

        return 404;
    }
}
```

### /etc/nginx/conf.d

本目录下，配置 server 块。

```conf
server {
    listen       80;
    listen  [::]:80;
    server_name  your.domain.com;

    location / {
        #转发请求
        proxy_pass http://your-service;
    }
}
```

## 正向代理和反向代理

### 代理（正向代理）

**代理**（Proxy）也称为网络代理，是一种特殊的网络服务，允许一个终端通过这个服务与另一个终端进行非直接的连接。一般认为代理服务有利于保障网络终端的隐私或安全，在一定程度上能够阻止网络攻击。

{% asset_img "Pasted image 20231201202645.png" 正向代理 %}

#### 功能

- 提高访问速度
- 隐藏真实IP
- 突破网站的区域限制
- 突破网络审查
- ……

### 反向代理

反向代理（Reverse Proxy）在电脑网络中是代理服务器的一种。服务器根据客户端的请求，从其关联的一组或多组后端服务器上获取资源，然后再将这些资源返回给客户端，客户端只会得知反向代理的 `IP` 地址，而不知道在代理服务器后面的服务器集群的存在。

{% asset_img "Pasted image 20231201202654.png" 反向代理 %}

#### 功能

- 对客户端隐藏服务器（集群）的 IP 地址
- 安全，可作为应用层防火墙
- **负载均衡**
- 缓存服务，缓存静态内容和短时间内大量访问请求的动态内容
- 压缩内容，节省带宽
- ……

### 对比

|正向代理|反向代理|
|--|--|
|客户端的代理|服务端的代理|
|客户端一般需要特别设置|客户端不用做任何设置|
|请求发向目标服务器|请求发向代理|
|服务端知道代理不知道客户端|客户端知道代理不知道服务器|








## 参考文章

- [代理服务器](https://zh.wikipedia.org/wiki/%E4%BB%A3%E7%90%86%E6%9C%8D%E5%8A%A1%E5%99%A8)
- [反向代理](https://zh.wikipedia.org/wiki/%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86)
