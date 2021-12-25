---
title: 如何在Docker上部署ELK和RabbitMQ（国产芯片+UOS）
excerpt: 本篇教程详细介绍了如何在国产芯片和国产操作系统UOS上安装Docker并部署ELK和RabbitMQ
colorspace: orange
---

## 安装

国产操作系统的 CPU 是 arm 架构的，因此需要下载 arm 版本的 Docker 安装包。

### 安装 Docker

[Docker 下载地址](https://download.docker.com/linux/debian/dists/buster/pool/stable/arm64/)

下载页面中的`containerd.io`、`docker-ce-cli`、`docker-rootless-extras`、`docker-ce`。

安装命令：

```bash
# 安装当前目录下的所有deb文件
dpkg -i *.deb
```

### 安装 Docker-compose

[Docker-compose 下载地址](https://github.com/docker/compose/releases/download/v2.1.1/docker-compose-linux-aarch64)

下载页面中的`docker-compose-linux-aarch64`。

安装命令：

```bash
# 移动到系统目录下并改名
mv ./docker-compose-linux-aarch64 /usr/local/bin/docker-compose
# 给执行权限
chmod +x /usr/local/bin/docker-compose
```

检查一下安装是否成功：

```bash
docker -v
docker-compose -v
```

## 导出导入镜像

### 下载镜像

ELK 默认仓库是 x86 的，可以从`arm64v8`这个仓库里下载，或者从默认仓库下载的时候指定 arm 版本的 digest。

另外 RabbitMQ 需要指定 arm 版本的 digest，也就是命令中`@sha256`及后面的部分。

下载命令：

```bash
docker pull arm64v8/elasticsearch:7.14.2
docker pull arm64v8/logstash:7.14.2
docker pull arm64v8/kibana:7.14.2
docker pull rabbitmq:management@sha256:cbe8554bcb0bebf4ee5bd3dcf7a019848daabc8cce896d0054a77d7a8d529ba5
```

### 导出镜像

从外网下载之后可以导出镜像，拷贝到内网使用。

导出命令：

```bash
docker save -o elasticsearch.tar arm64v8/elasticsearch:7.14.2
docker save -o logstash.tar arm64v8/logstash:7.14.2
docker save -o kibana.tar arm64v8/kibana:7.14.2
docker save -o rabbitmq.tar rabbitmq
```

镜像会以 tar 文件形式保存在当前目录下，拷贝到内网机器上备用。

### 导入镜像

在内网机器上将镜像导入到 Docker。

导入命令：

```bash
docker load -i elasticsearch.tar
docker load -i logstash.tar
docker load -i kibana.tar
docker load -i rabbitmq.tar
```

导入之后可能会遇到镜像没有名称的问题，表现为运行命令`docker images`看到的 name 和 tag 都是<none>。可以用下面的命令给镜像命名。

```bash
docker tag <镜像ID> <镜像名称>:<镜像tag>
```

## 配置文件

`docker-compose.yml`文件配置好了将本地的配置文件共享给容器，根据实际情况修改`docker-compose.yml`中的`volums`中的配置内容，指定对应的配置文件路径。教程最后附录了一份供参考的配置文件。

## 运行和停止

使用 Docker-compose 可以很方便地一次创建并运行多个容器，可以参考附录的`docker-compose.yml`文件。

运行命令：

```bash
docker-compose up -d
```

停止命令：

```bash
docker-compose stop
```

## Elasticsearch 配置密码

暂时没有找到在`docker-compose.yml`文件中配置密码的方法，所以需要在运行一次`docker-compose up`之后，进入到 Elasticsearch 容器中进行配置，配置完成后重启容器即可。

进入容器：

```bash
docker exec -it <容器ID> bash
```

之前的文档里使用了自动生成密码。现在因为线上已经有 ELK 了，所以为了顺利迁移，需要手动设置密码：

```bash
bin/elasticsearch-setup-passwords interactive
```

设置完成后重启容器即可：

```bash
docker container restart <容器ID>
```

## 可能遇到的问题

#### RabbitMQ 报错：.erlang.cookie 没有权限

修改`rabbitmq/data/.erlang.cookie`的权限，改为仅当前用户可修改。

```bash
chmod 600 -R  path/to/.erlang.cookie
```

### RabbitMQ 报错：其他没有权限的错误

修改 RabbitMQ 目录的所有者为当前用户或者非 root 用户并给足够权限。

## 附录

### docker-compose.yml

```yml
version: '3'
services:
  elasticsearch:
    image: arm64v8/elasticsearch:7.14.2
    container_name: elk-elasticsearch
    environment:
      - cluster.name=elasticsearch-cluster
      # 单机模式
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - 'ES_JAVA_OPTS=-Xms512m -Xmx1024m'
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      # 数据
      - /home/tong/elasticsearch/data:/usr/share/elasticsearch/data
      # 配置文件
      - /home/tong/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      # 证书路径
      - /home/tong/elasticsearch/config/cert:/usr/share/elasticsearch/config/cert
    ports:
      - 9200:9200
    networks:
      esnet:
        ipv4_address: 192.168.0.11
    restart: always
  kibana:
    image: arm64v8/kibana:7.14.2
    container_name: elk-kibana
    environment:
      SERVER_NAME: kibana
      # 原本输入IP的地方可以用services名代替
      ELASTICSEARCH_URL: http://elasticsearch:9200
    volumes:
      # 配置文件
      - /home/tong/kibana/config:/usr/share/kibana/config
    ports:
      - 5601:5601
    networks:
      esnet:
        ipv4_address: 192.168.0.21
    restart: always
    depends_on:
      - elasticsearch
  logstash:
    image: arm64v8/logstash:7.14.2
    container_name: elk-logstash
    environment:
      - SERVER_NAME=logstash
    volumes:
      # Pipeline配置文件，这个rabbitmq.conf配置了从RabbitMQ获取消息并转发给Elasticsearch
      - /home/tong/logstash/config/rabbitmq.conf:/usr/share/logstash/pipeline/logstash.conf
    ports:
      - 4560:4560
    networks:
      esnet:
        ipv4_address: 192.168.0.31
    restart: always
    depends_on:
      - es01
  rabbitmq:
    image: rabbitmq:latest
    container_name: rabbitmq
    restart: always
    hostname: rabbitmq_14
    ports:
      - 15672:15672
      - 15671:15671
      - 5672:5672
      - 5671:5671
    privileged: true
    volumes:
      # 日志
      - /home/tong/rabbitmq/log:/var/log/rabbitmq
      # 数据
      - /home/tong/rabbitmq/data:/var/lib/rabbitmq
      # 配置文件
      - /home/tong/rabbitmq/config:/etc/rabbitmq
    environment:
      # 初始化账号密码
      - RABBITMQ_DEFAULT_USER=rabbitmq
      - RABBITMQ_DEFAULT_PASS=password
    networks:
      esnet:
        ipv4_address: 192.168.0.41
networks:
  esnet:
    ipam:
      driver: default
      config:
        - subnet: '192.168.0.0/16'
```

### Logstash Pipeline

`logstash/config/rabbitmq.conf`

下面的配置文件中，`host`和`hosts`里都没有配置具体的 IP，而是用 services 的名字代替了。

```
input{
        rabbitmq {
                host => "rabbitmq"
                user => "rabbitmq"
                password => "password"
                queue => "index"
                codec => "json"
        }
}

output {
        elasticsearch {
                hosts => ["http://elasticsearch:9200"]
                index => "index"
                user => "elastic"
                password => "password"
        }
        stdout{
                codec => rubydebug
         }
}
```

### Elasticsearch

`elascitsearch/config/elasticsearch.yml`

```yml
# 这里列举了一部分配置，基本上这些配了就可以了
network.host: 0.0.0.0
http.host: 0.0.0.0
http.port: 9200
http.cors.enabled: true
http.cors.allow-origin: '*'
# 使用了安全验证，这里配置上开启加密和证书位置。如果没有使用安全验证，则不需要配置，同样的其他配置文件中密码部分也不需要配置。
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: cert/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: cert/elastic-certificates.p12
```

### Kibana

`kibana/config/kibana.yml`

```yml
server.host: '0.0.0.0'
i18n.locale: 'zh-CN'
elasticsearch.username: 'kibana'
elasticsearch.password: 'password'
```
