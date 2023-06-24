---
title: 如何在 Ubuntu 20.04 上安装 Minikube
date: 2023-06-23 21:04:20
tags:
    - minikube
    - kubernetes
---

## 环境搭建

### 安装 Ubuntu 20.04
使用 Vmware Workstation 通过 `ubuntu-20.04.6-live-server-amd64.iso` 安装。需要满足条件如下：
- 2 个或更多 CPU（2 CPU）。
- 2GB 可用内存（4GB）。
- 20GB 可用磁盘空间（30GB）。
- 网络连接
- 容器或虚拟机管理器（Docker）。

### 安装 Docker
参考官方文档：[Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)。

### 安装 Minikube
参考官方文档：[minikube start](https://minikube.sigs.k8s.io/docs/start/)。

### 安装 kubectl 并启动 kubectl 自动补全功能
参考官方文档：[在 Linux 系统中安装并设置 kubectl](https://kubernetes.io/zh-cn/docs/tasks/tools/install-kubectl-linux/)。

### 开始使用
创建集群：
```shell
minikube start
```

#### 创建集群时的权限问题

##### 不加 sudo
不加 `sudo` 的时候，创建集群失败，提示无法选择默认 driver。可能是 docker 处于不健康状态或者用户权限不足。
```
$ minikube start
* minikube v1.30.1 on Ubuntu 20.04
* Unable to pick a default driver. Here is what was considered, in preference order:
  - docker: Not healthy: "docker version --format {{.Server.Os}}-{{.Server.Version}}:{{.Server.Platform.Name}}" exit status 1: permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/version": dial unix /var/run/docker.sock: connect: permission denied
  - docker: Suggestion: Add your user to the 'docker' group: 'sudo usermod -aG docker $USER && newgrp docker' <https://docs.docker.com/engine/install/linux-postinstall/>
* Alternatively you could install one of these drivers:
  - kvm2: Not installed: exec: "virsh": executable file not found in $PATH
  - podman: Not installed: exec: "podman": executable file not found in $PATH
  - qemu2: Not installed: exec: "qemu-system-x86_64": executable file not found in $PATH
  - virtualbox: Not installed: unable to find VBoxManage in $PATH

X Exiting due to DRV_NOT_HEALTHY: Found driver(s) but none were healthy. See above for suggestions how to fix installed drivers.
```

##### 使用 sudo
使用 `sudo` 的时候，会提示不建议通过 `root` 权限使用 `docker`，如果还是想要继续，可以使用选项 `--force`。
```shell
$ sudo minikube start
* minikube v1.30.1 on Ubuntu 20.04
* Automatically selected the docker driver. Other choices: none, ssh
* The "docker" driver should not be used with root privileges. If you wish to continue as root, use --force.
* If you are running minikube within a VM, consider using --driver=none:
*   https://minikube.sigs.k8s.io/docs/reference/drivers/none/

X Exiting due to DRV_AS_ROOT: The "docker" driver should not be used with root privileges.
```

##### 使用选项 --force
考虑到仅用于测试，尝试通过 `sudo minikube start --force` 启动集群，成功启动集群但是提示使用该选项可能会引发未知行为。
```shell
$ sudo minikube start --force
* minikube v1.30.1 on Ubuntu 20.04
! minikube skips various validations when --force is supplied; this may lead to unexpected behavior
* Automatically selected the docker driver. Other choices: ssh, none
* The "docker" driver should not be used with root privileges. If you wish to continue as root, use --force.
* If you are running minikube within a VM, consider using --driver=none:
*   https://minikube.sigs.k8s.io/docs/reference/drivers/none/
* Using Docker driver with root privileges
* Starting control plane node minikube in cluster minikube
* Pulling base image ...
* Downloading Kubernetes v1.26.3 preload ...
    > preloaded-images-k8s-v18-v1...:  397.02 MiB / 397.02 MiB  100.00% 25.89 M
    > index.docker.io/kicbase/sta...:  373.53 MiB / 373.53 MiB  100.00% 7.17 Mi
! minikube was unable to download gcr.io/k8s-minikube/kicbase:v0.0.39, but successfully downloaded docker.io/kicbase/stable:v0.0.39 as a fallback image
* Creating docker container (CPUs=2, Memory=2200MB) ...
* Preparing Kubernetes v1.26.3 on Docker 23.0.2 ...
  - Generating certificates and keys ...
  - Booting up control plane ...
  - Configuring RBAC rules ...
* Configuring bridge CNI (Container Networking Interface) ...
  - Using image gcr.io/k8s-minikube/storage-provisioner:v5
* Verifying Kubernetes components...
* Enabled addons: default-storageclass, storage-provisioner
* Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

成功启动集群后，使用 `kubectl get pod` 测试，提示连接被拒绝。
```shell
$ kubectl get pod
E0622 21:55:27.400754   18561 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
E0622 21:55:27.401000   18561 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
E0622 21:55:27.410464   18561 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
E0622 21:55:27.410951   18561 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
E0622 21:55:27.412076   18561 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

使用 `minikube dashboard` 启动控制台，在访问时同样提示连接被拒绝：`dial tcp 127.0.0.1:8080: connect: connection refused`。

考虑到可能还有别的问题，决定采用官方建议将用户添加到 `docker` 用户组。

##### 将用户添加到 docker 用户组
使用 `sudo usermod -aG docker $USER && newgrp docker` 将当前用户添加到 `docker` 用户组并切换当前用户组到 `docker` 用户组后，正常启动集群。
```shell
$ minikube start
* minikube v1.30.1 on Ubuntu 20.04
* Automatically selected the docker driver. Other choices: ssh, none
* Using Docker driver with root privileges
* Starting control plane node minikube in cluster minikube
* Pulling base image ...
* Downloading Kubernetes v1.26.3 preload ...
    > preloaded-images-k8s-v18-v1...:  393.36 MiB / 397.02 MiB  99.08% 19.35 Mi! minikube was unable to download gcr.io/k8s-minikube/kicbase:v0.0.39, but successfully downloaded docker.io/kicbase/stable:v0.0.39 as a fallback image
    > preloaded-images-k8s-v18-v1...:  397.02 MiB / 397.02 MiB  100.00% 21.44 M
* Creating docker container (CPUs=2, Memory=2200MB) ...
* Preparing Kubernetes v1.26.3 on Docker 23.0.2 ...
  - Generating certificates and keys ...
  - Booting up control plane ...
  - Configuring RBAC rules ...
* Configuring bridge CNI (Container Networking Interface) ...
  - Using image gcr.io/k8s-minikube/storage-provisioner:v5
* Verifying Kubernetes components...
* Enabled addons: storage-provisioner, default-storageclass
* Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

#### 创建集群时下载镜像失败
在解决问题的过程中，发现有人存在下载镜像失败的情况。从启动日志可以看到，由于 minikube 下载 `gcr.io/k8s-minikube/kicbase:v0.0.39` 镜像失败，自动下载 `docker.io/kicbase/stable:v0.0.39` 镜像作为备选。如果从 `docker.io` 下载镜像也很困难，还可以通过指定镜像仓库启动集群。可以通过查看帮助内关于仓库的信息，获取官方建议中国大陆用户使用的镜像仓库地址。
```shell
$ minikube start --help | grep repo
    --image-repository='':
	Alternative image repository to pull docker images from. This can be used when you have limited access to gcr.io. Set it to "auto" to let minikube decide one for you. For Chinese mainland users, you may use local gcr.io mirrors such as registry.cn-hangzhou.aliyuncs.com/google_containers

$ minikube start --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'
```

#### 如何通过宿主机进行访问 minikube 控制台
启动控制台：
```shell
$ minikube dashboard
* Enabling dashboard ...
  - Using image docker.io/kubernetesui/dashboard:v2.7.0
  - Using image docker.io/kubernetesui/metrics-scraper:v1.0.8
* Some dashboard features require the metrics-server addon. To enable all features please run:

	minikube addons enable metrics-server	


* Verifying dashboard health ...
* Launching proxy ...
* Verifying proxy health ...
* Opening http://127.0.0.1:35967/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/ in your default browser...
  http://127.0.0.1:35967/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/

```

如果虚拟机安装的 Ubuntu 是 Desktop 版本，那么你可以在 Ubuntu 里直接通过浏览器访问。但是如果你安装的 Ubuntu 是 server 版本，除了使用 `curl` 访问 url 外，你也许想要在宿主机的浏览器访问：
```shell
kubectl proxy --port=your-port --address='your-virtual-machine-ip' --accept-hosts='^.*' &
```

#### 使用过程中下载镜像失败

##### 从其他镜像仓库下载代替
一般是在需要从 `gcr.io` 镜像仓库下载时发生，比如官方教程中需要执行以下命令，会发现 `deployment` 和 `pod` 迟迟不能达到目标状态。
```shell
$ kubectl create deployment hello-node --image=registry.k8s.io/e2e-test-images/agnhost:2.39 -- /agnhost netexec --http-port=8080

$ kubectl get deployments
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hello-node   0/1     1            0           19s

$ kubectl get pods
NAME                          READY   STATUS             RESTARTS   AGE
hello-node-7b87cd5f68-zwd4g   0/1     ImagePullBackOff   0          38s
```
仅作为测试用途，可以从 Docker 官方仓库搜索镜像，找到排名靠前，版本相同或相近的可靠镜像代替。

##### 配置代理
对于这类网络连接不通的情况，配置代理是通用的解决方案。
刚开始我以为给 Ubuntu 上的 `dockerd` 配置代理帮助加速 `docker pull` 即可，后来发现仍然下载失败。即使我通过 `docker pull` 先下载镜像到本地，配置 `imagePullPolicy` 为 `Never` 或者 `IfNotPresent`，minikube 还是不能识别到本地已有的镜像。猜测 minikube 的机制和我想象的是不同的，需要直接为 minikube 容器配置代理。搜索到以下命令满足需求：
```shell
minikube start --docker-env http_proxy=http://proxyAddress:port --docker-env https_proxy=http://proxyAddress:port --docker-env no_proxy=localhost,127.0.0.1,your-virtual-machine-ip/24
```


## 参考链接
[Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
[minikube start](https://minikube.sigs.k8s.io/docs/start/)
[k8s的迷你版——minikube+docker的安装](https://zhuanlan.zhihu.com/p/429690423)
[minikube - Why The "docker" driver should not be used with root privileges](https://stackoverflow.com/questions/68984450/minikube-why-the-docker-driver-should-not-be-used-with-root-privileges)
[安装Minikube无法访问k8s.gcr.io的简单解决办法](https://www.cnblogs.com/pack27/p/12202687.html)
[让其他电脑访问minikube dashboard](https://www.cnblogs.com/liyuanhong/p/13799404.html)
[【问题解决】This container is having trouble accessing https://k8s.gcr.io | 如何解决从k8s.gcr.io拉取镜像失败问题？](https://blog.csdn.net/qq_43762191/article/details/122709763)
[K8S(kubernetes)镜像源](https://www.cnblogs.com/Leo_wl/p/15775077.html)

