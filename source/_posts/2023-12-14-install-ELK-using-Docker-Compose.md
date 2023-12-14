---
title: 使用 Docker Compose 安装 ELK
date: 2023-12-14 19:35:00
tags: [docker, elasticsearch, kibana]
---

印象里每次安装 `ElK` 组件的体验都不是很好，或多或少都遇到过奇奇怪怪的问题。本文几乎完全按照官方文档：[Getting started with the Elastic Stack and Docker Compose: Part 1](https://www.elastic.co/cn/blog/getting-started-with-the-elastic-stack-and-docker-compose) 通过 `docker compose` 安装 `elasticsearch`、`kibana`、`metricbeat`、`filebeat` 和 `logstash`，但是移除了 `ssl` 相关的配置。你可以直接按照原文档进行安装，但是对照本文可以帮助你更快速地移除不需要的配置以及绕开可能踩到的坑。
此安装方式尽量**使用环境变量代替编写配置文件**，便于在备份和分享时将敏感信息留存在本地。本次安装时间为 `2023-12-14`，使用官方镜像，版本为 `8.11.2`。
<!-- more -->

> 如果不考虑持久化配置，直接进入容器手动修改配置文件可能也很合适。

**文件结构**

```text
├── .env

├── docker-compose.yml

├── filebeat.yml

├── logstash.conf

└── metricbeat.yml
```

## 环境文件

我们可以在 `.env` 文件中定义一些准备传递给 `docker-compose` 文件的变量，比如密码、版本和端口号等等。

> 将敏感或者重复使用的信息放在 `.env` 文件中，并在 `.gitignore` 文件中忽略该文件，就可以更方便地将 `docker-compose` 文件备份到 `GitHub` 或者分享给他人。

```conf
# Project namespace (defaults to the current folder name if not set)
#COMPOSE_PROJECT_NAME=myproject

# 'elastic' 用户的密码 (至少 6 字符)
ELASTIC_PASSWORD=changeme

# 'kibana_system' 用户的密码 (至少 6 字符)
KIBANA_PASSWORD=changeme

# Elastic 产品的版本
STACK_VERSION=8.11.2

# 设置集群的名称
CLUSTER_NAME=docker-cluster

# Set to 'basic' or 'trial' to automatically start the 30-day trial
LICENSE=basic
#LICENSE=trial

# Elasticsearch HTTP API 暴露的端口
ES_PORT=9200

# Kibana 暴露的端口
KIBANA_PORT=5601

# ES、KIBANA、LOGSTASH 的内存限制（byte），根据主机可用的内存增加或降低。
ES_MEM_LIMIT=1073741824
KB_MEM_LIMIT=1073741824
LS_MEM_LIMIT=1073741824

# 密钥（起先以为是可选的，其实后面会用到）
ENCRYPTION_KEY=c34d38b3a14956121ff2170e5030b471551370178f43e5626eec58b04a30fae2
```

> 在官方文档中，使用了一个名为 `setup` 的容器，用于做一些前置准备工作，比如 `ssl` 相关的配置和账号密码的设置。以下因为不需要配置 `ssl` 所以不使用 `setup` 容器，`Kibana` 的账号密码另行手动设置。

## ElasticSearch

在 `docker-compose.yml` 文件中添加 `ElasticSearch` 部分。

```yaml
version: '3.8'
services:
  elasticsearch:
    image: elasticsearch:8.11.1
    container_name: elasticsearch
    ports:
      - '${ES_PORT}:9200'
    environment:
      - node.name=elasticsearch
      - cluster.name=${CLUSTER_NAME}
      - discovery.type=single-node
      # 设置密码
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - bootstrap.memory_lock=true
      # 启用  X-Pack 必须登录
      - xpack.security.enabled=true
      # 关闭 ssl
      - xpack.security.http.ssl.enabled=false
      - xpack.security.transport.ssl.enabled=false
      - xpack.license.self_generated.type=${LICENSE}
    # 限制内存
    mem_limit: ${ES_MEM_LIMIT}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
      - elasticsearch-plugins:/usr/share/elasticsearch/plugins
    networks:
      - my-network
    restart: unless-stopped
    # 健康检测：
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s http://localhost:9200 | grep -q 'missing authentication credentials'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120

volumes:
  elasticsearch-data:
    driver: local
  elasticsearch-plugins:
    driver: local
  # kibana-data:
  #   driver: local
  # metricbeat-data:
  #   driver: local
  # filebeat-data:
  #   driver: local
  # logstash-data:
  #   driver: local

networks:
  my-network:
    external: true
```

### 健康检测

对 `Elasticsearch` 进行健康检测的方式是对其发起 `HTTP` 请求，在本示例中目标地址为 `http://localhost:9200`，因为启用了密码，因此返回结果里会带有 'missing authentication credentials' 字样，成功匹配。

在官方文档中的示例是携带 `--cacert` 参数发起 `HTTPS` 请求：

```shell
curl -s --cacert config/certs/ca/ca.crt https://localhost:9200
```

如果在后续启动时发现 `elasticsearch` 容器长时间处于 `waiting` 状态，不能转为 `healthy`：

- 检查是不是写成 `https`
- 手动执行命令查看返回结果是否符合期望，像我原先复制了 `Kibana` 中配置的测试命令，最后发现多了一个 `-I` 参数导致启动一直不能完成。

### 限制内存失败

你可能会遇到以下警告：

```text
WARNING: Your kernel does not support cgroup swap limit. WARNING: Your
kernel does not support swap limit capabilities. Limitation discarded.
To prevent these messages, enable memory and swap accounting on your system. To enable these on system using GNU GRUB (GNU GRand Unified Bootloader), do the following.
```

原因是系统默认未开启 `swap` 限制。开启后即使 `Docker` 未运行，内存和交换计算也会产生约 `1%` 的总可用内存的开销和 `10%` 的整体性能下降。开启方式如下：

- 编辑配置文件 `sudo vim /etc/default/grub`
- 设置 `GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1`
- `sudo update-grub`
- 重启 `sudo reboot`（**直接关闭虚拟机再启动似乎不能生效**）

### 验证 Elasticsearch 启动成功

在启动后，访问 `http://localhost:9200/`，会弹出验证登录对话框，代表 `elasticsearch` 启动成功

<div style="width:80%;margin:auto">{% asset_img "Snipaste_2023-12-15_04-35-10.png" ES 登录对话框 %}</div>

输入账号密码后显示 `Elasticsearch` 信息，代表正常运行。

```json
{
  "name" : "elasticsearch",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "MlwsEv3-SYqGkm2B_uqg4Q",
  "version" : {
    "number" : "8.11.1",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "6f9ff581fbcde658e6f69d6ce03050f060d1fd0c",
    "build_date" : "2023-11-11T10:05:59.421038163Z",
    "build_snapshot" : false,
    "lucene_version" : "9.8.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

## Kibana

在 `docker-compose.yml` 文件中添加 `Kibana` 部分。

```yaml
kibana:
  # 依赖于 elasticsearch，检测其健康状态
  depends_on:
    elasticsearch:
      condition: service_healthy
  image: 'kibana:8.11.1'
  container_name: kibana
  ports: 
    - ${KIBANA_PORT}:5601
  environment:
    - SERVERNAME=kibana
    - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    # Kibana 连接 Elasticsearch 的账号密码，不可用于登录
    - ELASTICSEARCH_USERNAME=kibana_system
    - ELASTICSEARCH_PASSWORD=${KIBANA_PASSWORD}
    # 密钥
    - XPACK_SECURITY_ENCRYPTIONKEY=${ENCRYPTION_KEY}
    - XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY=${ENCRYPTION_KEY}
    - XPACK_REPORTING_ENCRYPTIONKEY=${ENCRYPTION_KEY}
    # 切换成中文
    - I18N_LOCALE=zh-CN
  mem_limit: ${KB_MEM_LIMIT}
  volumes:
    - kibana-data:/usr/share/kibana/data
  networks:
    - my-network
  # 健康检测，发起 HTTP 请求，匹配返回结果中有指定字符串
  healthcheck:
    test:
      [
        "CMD-SHELL",
        "curl -s -I http://localhost:5601 | grep -q 'HTTP/1.1 302 Found'",
      ]
    interval: 10s
    timeout: 10s
    retries: 120
  restart: unless-stopped
```

### 设置 Kibana 密码

配置的账号密码是给 `Kibana` 用于连接 `Elasticsearch` 的，并非用于登录 `Kibana` 的，也不可用于登录。`elastic` 的账号密码才是登录 `Kibana` 的账号密码。

在官方文档中，是通过 `setup` 容器完成 `kibana_system` 账号的密码设置，脚本中执行的命令如下：

```shell
curl -s -X POST --cacert config/certs/ca/ca.crt -u "elastic:${ELASTIC_PASSWORD}" -H "Content-Type: application/json" https://es01:9200/_security/user/kibana_system/_password -d "{\"password\":\"${KIBANA_PASSWORD}\"}"
```

仿照脚本手动发起请求，设置 `kibana_system` 的密码。

```shell
curl -s -X POST -u "elastic:${ELASTIC_PASSWORD}" -H "Content-Type: application/json" http://localhost:9200/_security/user/kibana_system/_password -d "{\"password\":\"${KIBANA_PASSWORD}\"}"
```

> 你也可以进入容器使用 `elasticsearch-setup-passwords` 设置密码。

### 设置密钥

密钥部分如果不配置，会影响 `Kibana` 的功能使用：

- [Secure saved objects](https://www.elastic.co/guide/en/kibana/current/xpack-security-secure-saved-objects.html)
- [kibana-encryption-keys](https://www.elastic.co/guide/en/kibana/current/kibana-encryption-keys.html)

查看日志会看到以下报错信息。

```log
[2023-12-14T18:02:17.493+00:00][ERROR][plugins.observabilityAIAssistant] Error: Unable to create actions client because the Encrypted Saved Objects plugin is missing encryption key. Please set xpack.encryptedSavedObjects.encryptionKey in the kibana.yml or use the bin/kibana-encryption-keys command.
kibana         |     at getActionsClientWithRequest (/usr/share/kibana/node_modules/@kbn/actions-plugin/server/plugin.js:329:15)
kibana         |     at Object.secureGetActionsClientWithRequest [as getActionsClientWithRequest] (/usr/share/kibana/node_modules/@kbn/actions-plugin/server/plugin.js:384:58)
kibana         |     at handler (/usr/share/kibana/node_modules/@kbn/observability-ai-assistant-plugin/server/routes/connectors/route.js:25:65)
kibana         |     at processTicksAndRejections (node:internal/process/task_queues:95:5)
kibana         |     at wrappedHandler (/usr/share/kibana/node_modules/@kbn/server-route-repository/src/register_routes.js:57:13)
kibana         |     at Router.handle (/usr/share/kibana/node_modules/@kbn/core-http-router-server-internal/src/router.js:154:30)
kibana         |     at handler (/usr/share/kibana/node_modules/@kbn/core-http-router-server-internal/src/router.js:113:50)
kibana         |     at exports.Manager.execute (/usr/share/kibana/node_modules/@hapi/hapi/lib/toolkit.js:60:28)
kibana         |     at Object.internals.handler (/usr/share/kibana/node_modules/@hapi/hapi/lib/handler.js:46:20)
kibana         |     at exports.execute (/usr/share/kibana/node_modules/@hapi/hapi/lib/handler.js:31:20)
kibana         |     at Request._lifecycle (/usr/share/kibana/node_modules/@hapi/hapi/lib/request.js:371:32)
kibana         |     at Request._execute (/usr/share/kibana/node_modules/@hapi/hapi/lib/request.js:281:9)
```

你可以通过以下方式自己生成：

```shell
# 进入 kibana 容器
docker exec -it kibana bash
# 执行命令
bin/kibana-encryption-keys generate
```

结果如下：

```text
Kibana is currently running with legacy OpenSSL providers enabled! For details and instructions on how to disable see https://www.elastic.co/guide/en/kibana/8.11/production.html#openssl-legacy-provider
## Kibana Encryption Key Generation Utility

The 'generate' command guides you through the process of setting encryption keys for:

xpack.encryptedSavedObjects.encryptionKey
    Used to encrypt stored objects such as dashboards and visualizations
    https://www.elastic.co/guide/en/kibana/current/xpack-security-secure-saved-objects.html#xpack-security-secure-saved-objects

xpack.reporting.encryptionKey
    Used to encrypt saved reports
    https://www.elastic.co/guide/en/kibana/current/reporting-settings-kb.html#general-reporting-settings

xpack.security.encryptionKey
    Used to encrypt session information
    https://www.elastic.co/guide/en/kibana/current/security-settings-kb.html#security-session-and-cookie-settings


Already defined settings are ignored and can be regenerated using the --force flag.  Check the documentation links for instructions on how to rotate encryption keys.
Definitions should be set in the kibana.yml used configure Kibana.

Settings:
xpack.encryptedSavedObjects.encryptionKey: 8bd58432fa60321fc44cd1953b0468b9
xpack.reporting.encryptionKey: eb360c607b1643e1739fcf8d587b2d42
xpack.security.encryptionKey: ffe9139e6955daaa6cff13050978c3e8
```

### Kibana 未准备就绪

你可能访问 `Kibana` 时还会遇到 `Kibana server is not ready yet` 的报错信息，这次我在使用原先的 `bitnami` 的镜像时怎么都解决不了这个问题。可以参考[Kibana 最常见的“启动报错”或“无法连接ES集群服务”的故障原因及解决方案汇总](https://zhuanlan.zhihu.com/p/641274429)，或者像我一样直接换镜像换版本试试。

> 如果成功完成以上安装，几乎就没有硬坑了。**以下部分是完全照搬官方文档**，可以不安装。如果想要快速在视觉上有一些感受，也可以安装体验。

## metricbeat

在 `docker-compose.yml` 文件中添加 `metricbeat` 部分。

```yaml
metricbeat:
  depends_on:
    elasticsearch:
      condition: service_healthy
    kibana:
      condition: service_healthy
  image: elastic/metricbeat:${STACK_VERSION}
  container_name: metricbeat
  user: root
  volumes:
    - metricbeat-data:/usr/share/metricbeat/data
    - "./metricbeat.yml:/usr/share/metricbeat/metricbeat.yml:ro"
    - "/var/run/docker.sock:/var/run/docker.sock:ro"
    - "/sys/fs/cgroup:/hostfs/sys/fs/cgroup:ro"
    - "/proc:/hostfs/proc:ro"
    - "/:/hostfs:ro"
  networks:
    - my-network
  environment:
    - ELASTIC_USER=elastic
    - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
    # 注意：改为 http
    - ELASTIC_HOSTS=http://elasticsearch:9200
    - KIBANA_HOSTS=http://kibana:5601
    - LOGSTASH_HOSTS=http://logstash:9600
```

### metricbeat.yml

和官方文档相比，移除了 `ssl` 相关的配置。

```yaml
metricbeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

metricbeat.modules:
- module: elasticsearch
  xpack.enabled: true
  period: 10s
  hosts: ${ELASTIC_HOSTS}
  username: ${ELASTIC_USER}
  password: ${ELASTIC_PASSWORD}
  ssl.enabled: false

- module: logstash
  xpack.enabled: true
  period: 10s
  hosts: ${LOGSTASH_HOSTS}

- module: kibana
  metricsets:
    - stats
  period: 10s
  hosts: ${KIBANA_HOSTS}
  username: ${ELASTIC_USER}
  password: ${ELASTIC_PASSWORD}
  xpack.enabled: true

- module: docker
  metricsets:
    - "container"
    - "cpu"
    - "diskio"
    - "healthcheck"
    - "info"
    # - "image"
    - "memory"
    - "network"
  hosts: ["unix:///var/run/docker.sock"]
  period: 10s
  enabled: true

processors:
  - add_host_metadata: ~
  - add_docker_metadata: ~

output.elasticsearch:
  hosts: ${ELASTIC_HOSTS}
  username: ${ELASTIC_USER}
  password: ${ELASTIC_PASSWORD}
```

### 容器退出

你可能会遇到 `metricbeat` 容器启动完毕就立即退出的问题，查看容器日志可知，`metricbeat.yml` 文件必须属于 `uid=0` 的用户或者 `root` 用户。

```text
Exiting: error loading config file: config file ("metricbeat.yml") must be owned by the user identifier (uid=0) or root
```

执行命令：

```shell
sudo chown root metricbeat.yml
```

在修改完你可能会发现问题还是没有被解决，再次查看日志可知，`metricbeat.yml` 文件的权限必须为 "`-rw-rw-r--`"。

```text
Exiting: error loading config file: config file ("metricbeat.yml") can only be writable by the owner but the permissions are "-rw-rw-r--" (to fix the permissions use: 'chmod go-w /usr/share/metricbeat/metricbeat.yml')
```

执行命令：

```shell
sudo chmod 0644 metricbeat.yml
```

> **`filebeat.yml` 文件存在相同的问题，可以提前解决**。

前往“左侧菜单栏->Management->堆栈监测” 查看集群概览，`Kibana` 会提示可以使用开箱即用的规则。

<div style="width:80%;margin:auto">{% asset_img "Snipaste_2023-12-15_05-55-46.png" 集群概览 %}</div>

## filebeat

在 `docker-compose.yml` 文件中添加 `filebeat` 部分。

```yaml
filebeat:
  depends_on:
    elasticsearch:
      condition: service_healthy
  image: elastic/filebeat:${STACK_VERSION}
  container_name: filebeat
  user: root
  volumes:
    - filebeat-data:/usr/share/filebeat/data
    - "./filebeat_ingest_data/:/usr/share/filebeat/ingest_data/"
    - "./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro"
    - "/var/lib/docker/containers:/var/lib/docker/containers:ro"
    - "/var/run/docker.sock:/var/run/docker.sock:ro"
  environment:
    - ELASTIC_USER=elastic
    - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
    # 注意：改用 http
    - ELASTIC_HOSTS=http://elasticsearch:9200
    - KIBANA_HOSTS=http://kibana:5601
    - LOGSTASH_HOSTS=http://logstash:9600
  networks:
    - my-network
```

### filebeat.yaml

```yaml
filebeat.inputs:
- type: filestream
  id: default-filestream
  paths:
    - ingest_data/*.log

filebeat.autodiscover:
  providers:
    - type: docker
      hints.enabled: true

processors:
- add_docker_metadata: ~

setup.kibana:
  host: ${KIBANA_HOSTS}
  username: ${ELASTIC_USER}
  password: ${ELASTIC_PASSWORD}

output.elasticsearch:
  hosts: ${ELASTIC_HOSTS}
  username: ${ELASTIC_USER}
  password: ${ELASTIC_PASSWORD}
  ssl.enabled: false
```

前往“左侧菜单栏->Observability->日志->Stream” 查看流式传输。

<div style="width:80%;margin:auto">{% asset_img "Snipaste_2023-12-15_05-59-47.png" 流式传输 %}</div>

## logstash

在 `docker-compose.yml` 文件中添加 `logstash` 部分。

```yaml
logstash:
  depends_on:
    elasticsearch:
      condition: service_healthy
    kibana:
      condition: service_healthy
  image: logstash:${STACK_VERSION}
  container_name: logstash
  user: root
  volumes:
    - logstash-data:/usr/share/logstash/data
    - "./logstash_ingest_data/:/usr/share/logstash/ingest_data/"
    - "./logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro"
  environment:
    - xpack.monitoring.enabled=false
    - ELASTIC_USER=elastic
    - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
    - ELASTIC_HOSTS=http://elasticsearch:9200
  networks:
    - my-network
```

### logstash.conf

```conf
input {
  file {
    mode => "tail"
    path => "/usr/share/logstash/ingest_data/*"
  }
}

filter {
}

output {
  elasticsearch {
    index => "logstash-%{+YYYY.MM.dd}"
    hosts => "${ELASTIC_HOSTS}"
    user => "${ELASTIC_USER}"
    password => "${ELASTIC_PASSWORD}"
  }
}
```

前往“左侧菜单栏->Analytics->Discover”，查看数据视图。

<div style="width:80%;margin:auto">{% asset_img "Snipaste_2023-12-15_06-03-42.png" 数据视图 %}</div>

> 每一次不顺利的安装都是一场折磨。