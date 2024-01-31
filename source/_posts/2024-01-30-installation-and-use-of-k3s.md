---
title: k3s 的安装和使用
date: 2024-01-30 19:45:23
tags: [k3s, k8s]
---

本文记录了 `k3s` 的安装和使用，相较于 `minikube`，前者是一个完全兼容的 `Kubernetes` 发行版，安装和使用的体验更佳。

<!-- more -->

## 安装

> 参考官方文档-[快速入门指南](https://docs.k3s.io/zh/quick-start)，使用默认选项启动集群非常简单方便！！！

### 步骤

1. 获取并运行 `k3s` 安装脚本。官方为中国用户提供了镜像加速支持。
```text
$ curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -                                
[sudo] password for moralok:                                                                                                                        
[INFO]  Finding release for channel stable                                                                                                        
[INFO]  Using v1.28.5+k3s1 as release                                                                                                             
[INFO]  Downloading hash rancher-mirror.rancher.cn/k3s/v1.28.5-k3s1/sha256sum-amd64.txt                                                           
[INFO]  Downloading binary rancher-mirror.rancher.cn/k3s/v1.28.5-k3s1/k3s                                                                         
[INFO]  Verifying binary download                                                                                                                 
[INFO]  Installing k3s to /usr/local/bin/k3s                                                                                                      
[INFO]  Skipping installation of SELinux RPM                                                                                                      
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s                                                                                            
[INFO]  Creating /usr/local/bin/crictl symlink to k3s                                                                                             
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr                                                        
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh                                                                                     
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh                                                                                 
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env                                                                        
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service                                                                            
sh: 1014: restorecon: not found                                                                                                                   
sh: 1015: restorecon: not found                                                                                                                   
[INFO]  systemd: Enabling k3s unit                                                                                                                
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.                                        
[INFO]  systemd: Starting k3s
```
2. 可以通过使用 `kubectl` 确认安装成功。刚安装的时候使用 `kubectl` 需要 `root` 权限。
```text
$ sudo kubectl get node
NAME            STATUS   ROLES                  AGE   VERSION
ubuntu-server   Ready    control-plane,master   52m   v1.28.5+k3s1
```
3. 实际上安装的就是一个 `k3s` 可执行文件，`kubectl` 和 `crictl` 只是软链接，指向 `k3s`。
```text
$ ls /usr/local/bin/
crictl  k3s  k3s-killall.sh  k3s-uninstall.sh  kubectl
```

> 安装的信息中显示了 `k3s` 的 `service file` 和 `environment file` 的路径，后续修改启动参数和环境变量需要用到。

### 配置文件权限问题

在刚安装完 `k3s` 的时候，使用 `kubectl` 需要 `root` 权限。根据报错信息可知，是因为非 `root` 用户无法读取配置文件 `/etc/rancher/k3s/k3s.yaml`。

```text
$ kubectl get node                                                                                                                                            
WARN[0000] Unable to read /etc/rancher/k3s/k3s.yaml, please start server with --write-kubeconfig-mode to modify kube config permissions           
error: error loading config file "/etc/rancher/k3s/k3s.yaml": open /etc/rancher/k3s/k3s.yaml: permission denied
```

查看配置文件的信息可知其权限配置为 `600`，只有 `root` 用户具有读写权限。

```text
$ ll /etc/rancher/k3s/k3s.yaml                                                                                               
-rw------- 1 root root 2961 Jan 30 18:58 /etc/rancher/k3s/k3s.yaml
```

一般来说，我们希望能够通过非 `root` 用户使用 `kubectl`，避免通过 `root` 用户或者通过 `sudo` 加输入密码的形式来使用 `kubectl`。那么如何解决这个问题呢？本质上这是一个 `Linux` 的文件权限问题，似乎修改文件的权限配置就可以解决。但是提示信息给出的解决方案并不是那么直接，它告诉我们通过修改 `k3s server` 的启动参数来达到修改配置文件权限的目的。这是因为 `k3s` 服务在每次重启时会根据启动参数和环境变量重置配置文件 `/etc/rancher/k3s/k3s.yaml`，手动修改文件的权限配置并不能优雅地解决这个问题，一旦服务重启，修改就会丢失。

> [k3s 的 Github Discussions](https://github.com/k3s-io/k3s/discussions/7278) 中讨论了这个问题，并链接了文档 [管理 Kubeconfig 选项](https://docs.k3s.io/zh/cli/server#%E7%AE%A1%E7%90%86-kubeconfig-%E9%80%89%E9%A1%B9)，文档介绍了通过修改启动参数和环境变量达到修改配置文件权限的目的。

#### 修改启动参数

第一种方式是修改启动参数。

1. `sudo vim /etc/systemd/system/k3s.service` 添加 `k3s` 启动参数 `--write-kubeconfig-mode 644`
```ini
ExecStart=/usr/local/bin/k3s \
    server --write-kubeconfig-mode 644 \
```
2. `systemctl daemon-reload` 重新加载 `systemd` 配置
3. `systemctl restart k3s.service` 重启服务
4. 验证修改生效
```text
$ ll /etc/rancher/k3s/k3s.yaml
-rw-r--r-- 1 root root 2961 Jan 30 20:13 /etc/rancher/k3s/k3s.yaml
```

#### 修改环境变量

第二种方式是修改环境变量。

1. `sudo vim /etc/systemd/system/k3s.service.env` 添加环境变量 `K3S_KUBECONFIG_MODE=644`
```ini
K3S_KUBECONFIG_MODE=644
```
2. `systemctl restart k3s.service` 重启服务

#### 修改配置文件路径

第三种方式是复制配置信息到当前用户目录下，并使用其作为配置文件的路径。

1. 设置环境变量 `export KUBECONFIG=~/.kube/config`
2. 创建文件夹 `mkdir ~/.kube 2> /dev/null`
3. 复制配置信息 `sudo k3s kubectl config view --raw > "$KUBECONFIG"`
4. 修改配置文件的权限 `chmod 600 "$KUBECONFIG"`

### 配置代理

涉及 `k8s`，难免需要使用代理，否则在拉取镜像时将寸步难行。官方文档 [配置 HTTP 代理](https://docs.k3s.io/zh/advanced#%E9%85%8D%E7%BD%AE-http-%E4%BB%A3%E7%90%86) 介绍了如何配置代理。其中提及 `k3s` 安装脚本会自动使用当前 `shell` 中的 `HTTP_PROXY`、`HTTPS_PROXY` 和 `NO_PROXY`，以及 `CONTAINERD_HTTP_PROXY`、`CONTAINERD_HTTPS_PROXY` 和 `CONTAINERD_NO_PROXY` 变量（如果存在），并将它们写入 `systemd` 服务的环境文件。比如我设置过 `shell` 变量 `HTTP_PROXY`、`HTTPS_PROXY` 和 `NO_PROXY`，`/etc/systemd/system/k3s.service.env` 如下，你也可以自行编辑修改。

```ini
http_proxy='http://127.0.0.1:7890'
https_proxy='http://127.0.0.1:7890'
no_proxy='localhost,127.0.0.1'
K3S_KUBECONFIG_MODE=644
```

## 使用

> k8s 基础教程可参考官方文档 [Kubernetes 基础](https://kubernetes.io/zh-cn/docs/tutorials/kubernetes-basics/)。

### 创建 Deployment

1. 使用 `kubectl create` 命令创建管理 `Pod` 的 `Deployment`。该 `Pod` 根据提供的 `Docker` 镜像运行容器。
```shell
kubectl create deployment hello-node --image=registry.k8s.io/e2e-test-images/agnhost:2.39 -- /agnhost netexec --http-port=8080
```
2. 查看 `Deployment`：
```text
$ kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
hello-node-ccf4b9788-d8k9b   1/1     Running   0          15h
```
3. 查看 `Pod` 中容器的应用程序日志。
```text
$ kubectl logs hello-node-ccf4b9788-d8k9b
I0130 19:26:57.751131       1 log.go:195] Started HTTP server on port 8080
I0130 19:26:57.751350       1 log.go:195] Started UDP server on port  8081
```

### 创建 Service

1. 使用 `kubectl expose` 命令将 `Pod` 暴露给公网：
```shell
kubectl expose deployment hello-node --type=LoadBalancer --port=8080
```
2. 查看你创建的 `Service`：
```text
$ kubectl get services
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
kubernetes   ClusterIP      10.43.0.1      <none>           443/TCP          15h
hello-node   LoadBalancer   10.43.37.170   192.168.46.128   8080:32117/TCP   15h
```
3. 使用 `curl` 发起请求：
```text
$ curl http://localhost:8080
NOW: 2024-01-31 10:55:14.228709273 +0000 UTC m=+25932.159732511
```
4. 再次查看 `Pod` 中容器的应用程序日志。
```text
$ kubectl logs hello-node-ccf4b9788-d8k9b
I0130 19:26:57.751131       1 log.go:195] Started HTTP server on port 8080
I0130 19:26:57.751350       1 log.go:195] Started UDP server on port  8081
I0130 19:32:21.074992       1 log.go:195] GET /
```

### 清理

1. 删除 `Service`：
```shell
kubectl delete service hello-node
```
2. 删除 `Deployment`：
```shell
kubectl delete deployment hello-node
```

## 参考文章

- [快速入门指南](https://docs.k3s.io/zh/quick-start)
- [管理 Kubeconfig 选项](https://docs.k3s.io/zh/cli/server#%E7%AE%A1%E7%90%86-kubeconfig-%E9%80%89%E9%A1%B9)
- [配置 HTTP 代理](https://docs.k3s.io/zh/advanced#%E9%85%8D%E7%BD%AE-http-%E4%BB%A3%E7%90%86)
- [你好，Minikube](https://kubernetes.io/zh-cn/docs/tutorials/hello-minikube/)
- [Permission denied on non-existing /etc/rancher/k3s/config.yaml after fresh install](https://github.com/k3s-io/k3s/issues/7272)
- [error: error loading config file "/etc/rancher/k3s/k3s.yaml": open /etc/rancher/k3s/k3s.yaml: permission denied](https://devops.stackexchange.com/questions/16043/error-error-loading-config-file-etc-rancher-k3s-k3s-yaml-open-etc-rancher)
- [/etc/rancher/k3s/k3s.yaml is world readable #389](https://github.com/k3s-io/k3s/issues/389)
