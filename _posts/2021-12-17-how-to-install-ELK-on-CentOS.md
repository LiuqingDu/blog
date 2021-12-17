---
title: 如何在Linux上部署ELK（CentOS）
excerpt: 本篇教程详细介绍了如何在CentOS上安装ELK
colorspace: green
---

> 本篇教程写于 2019 年，虽然时间较早，但内容没有过时，依然可以用来参考。

## 环境准备

### JDK

检查 Java 版本。

```bash
[root@localhost etc]# java -version
openjdk version "1.8.0_201"
OpenJDK Runtime Environment (build 1.8.0_201-b09)
OpenJDK 64-Bit Server VM (build 25.201-b09, mixed mode)
```

## 安装 ELK 准备

### 下载安装包

安装包下载下来拷到 CentOS 里。

[Elasticsearch 7.1.1](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.1.1-linux-x86_64.tar.gz)

[Logstash 7.1.1](https://artifacts.elastic.co/downloads/logstash/logstash-7.1.1.tar.gz)

[Kibana 7.1.1](https://artifacts.elastic.co/downloads/kibana/kibana-7.1.1-linux-x86_64.tar.gz)

## 安装 ElasticSearch

### 解压安装包

切换到对应目录解压。

```bash
tar -zxf elasticsearch-7.1.1-linux-x86_64.tar.gz
cd elasticsearch-7.1.1/
```

### 配置 ElashticSearch

打开配置文件。

```bash
vi elasticsearch.yml
```

增加如下配置：

```yml
network.host: 0.0.0.0
http.port: 9200
http.cors.enabled: true
http.cors.allow-origin: '*'
node.name: node-1
cluster.initial_master_nodes: ['node-1']
```

### 启动 ElasticSearch

启动 ElasticSearch 为后台进程，并将进程 ID 输出。

```bash
./bin/elasticsearch -d -p pid
```

此时使用浏览器访问 IP:9200 或者使用 curl 命令可以看到返回 JSON：

```bash
curl -X GET "localhost:9200"
```

返回内容：

```json
{
  "name": "node-1",
  "cluster_name": "elasticsearch",
  "cluster_uuid": "2b6bMfBuQWOOBiZJA8tOXw",
  "version": {
    "number": "7.0.0",
    "build_flavor": "default",
    "build_type": "rpm",
    "build_hash": "b7e28a7",
    "build_date": "2019-04-05T22:55:32.697037Z",
    "build_snapshot": false,
    "lucene_version": "8.0.0",
    "minimum_wire_compatibility_version": "6.7.0",
    "minimum_index_compatibility_version": "6.0.0-beta1"
  },
  "tagline": "You Know, for Search"
}
```

### 停止 ElasticSearch

根据进程 ID 结束 ElasticSearch：

```bash
pkill -F pid
```

### 配置密码

配置密码前先停止 ElasticSearch。

#### 生成证书

执行如下命令，生成证书

```bash
bin/elasticsearch-certutil cert -out config/elastic-certificates.p12 -pass ""
```

证书生成在`config`目录下。

#### 启用证书

配置 Elasticsearch 目录下的`elasticsearch.yml`

```yml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

#### 生成密码

启动 Elasticsearch。

执行如下命令，自动生成密码，将屏幕上显示的密码保存下来。

```bash
bin/elasticsearch-setup-passwords auto
```

此时再访问 9200 端口就会要求输入账号密码。

**注意这里会生成多个账号和密码，一般会用到 elastic 和 kibana 两个账号。**

## 安装 Kibana

### 解压安装包

切换到安装包目录安装。

```bash
tar -xzf kibana-7.1.1-linux-x86_64.tar.gz
cd kibana-7.1.1-linux-x86_64/
```

### 配置 Kibana

打开配置文件。

```bash
vi kibana.yml
```

增加如下配置：

```yml
server.host: '0.0.0.0'
```

### 启动 Kibana

启动 Kiban。

```bash
./bin/kibana
```

使用浏览器访问`http://localhost:5601/`可以看到 Kibana 界面。

### 停止 Kibana

```bash
ps | grep kibana
kill -9 进程ID
```

### 启用官方中文界面

关闭 Kibana。

打开配置文件。

```bash
vi kibana.yml
```

增加如下配置。

```yml
i18n.locale: 'zh-CN'
```

启动 Kibana。

### 配置密码

关闭 Kibana。

打开配置文件。

```bash
vi kibana.yml
```

增加如下配置：

```bash
elasticsearch.username: "kibana"
elasticsearch.password: "上一节在ElasticSearch生成的对应的密码"
```

启动 Kibana，浏览器访问时会要求输入账号，此时输入账号为`elastic`的账号密码。

**注意这里用到了两个账号密码，配置文件里的账号是 kibana，浏览器里登录使用 elastic。**

## 安装 Logstash

### 解压安装包

切换到安装包目录安装。

```bash
tar -zxf logstash-7.1.1.tar.gz
cd logstash-7.1.1-linux-x86_64/
```

### 配置 Logstash

打开配置文件。

```bash
vi logstash.yml
```

确认如下配置：

```yml
path.data: /var/lib/logstash
path.config: /etc/logstash/conf.d
path.logs: /var/log/logstash
```

### 启动 Logstash

启动 Logstash。

```bash
./bin/logstash
```

### 停止 Logstash

停止 Logstash。

```bash
ps | grep logstash
kill -9 进程ID
```

### 配置密码

在`config`目录下创建一个`.conf`配置文件，其中输出部分如下：

```bash
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "elklog"
    user => "elastic"
    password => "password"
  }
  stdout{
    codec => rubydebug
  }
}
```

完整的配置内容参考使用 ELK 部分。

## 使用 ELK

### 配置 Logstash

配置 Logstash 用于从 RabbitMQ 消息队列收集日志并发送给 Elasticsearch。

在`config`目录下创建配置文件`rabbitmq.conf`：

```bash
input{
  rabbitmq {
    host => "localhost"
    user => "guest"
    password => "guest"
    queue => "elklog"
    codec => "json"
  }

}
filter {
  if ([message]== "") {
    drop {}
  }
}
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "elklog"
    user => "elastic"
    password => "password"
  }
  stdout{
    codec => rubydebug
  }
}
```

其中`input`指定了从 RabbitMQ 中读取，配置了 RabbitMQ 的地址、用户名、密码、队列名，指定消息的格式为 JSON。

`filter`可以不要，这里是抛弃了 message 字段为空的消息。

`output`配置了输出的 ElasticSearch 地址、索引名称、用户名、密码。输出的编码程序也可以配置上。

### 启动 Logstash

在 Logstash 安装根目录下创建一个`.sh`文件，方便以后通过指定配置文件启动。编写内容如下：

```bash
./bin/logstash -f ./config/rabbitmq.conf
```

赋予执行权限，通过运行该文件启动 Logstash 即可。

### 使用 Kibana 查看日志

使用浏览器访问 IP:9200 打开 Kibana。

要求登录时输入 elastic 的账号，具体看 ElasticSearch 的配置密码部分。

#### 创建索引

当一条数据被发送到 ElasticSearch 的时候，如果没有对应的索引，则会自动创建。如果该数据使用 ElasticSearch 的 jar 包发送，则通常自动创建的索引可以正常使用，无需手动创建。

如果该数据使用 MQ 或者其他非官方的方式发送，则可能涉及到数据类型的问题，比如 MQ 发送的日期类型数据可能会被 ElasticSearch 当作文本类型处理。如果只是在 Kibana 上查看这条数据可能看不出问题，但当使用日期过滤搜索的时候则会出现无法处理的问题。

因此，当涉及到使用第三方的工具传输数据的时候，建议手动创建索引，而不是自动创建。

#### 创建索引模式

初次使用 Kibana 时，需要创建索引模式才可以索引数据。

在左侧导航点管理，子菜单里的索引模式用于创建索引，索引管理可以查看已创建的索引。

点击索引模式，右侧可以创建索引模式。一般初次安装使用的时候会显示找不到任何 ElasticSearch 数据，此时需要向 RabbitMQ 消息队列发送至少一条日志，Logstash 会将日志发送到 ElasticSearch，之后即可在这里看到索引。

确保有日志发送到 ElasticSearch 后，再次刷新就会显示 Logstash 创建的`elklog`索引，输入匹配条件`elklog*`点击下一步。时间筛选选择`@timestamp`，点击创建索引模式完成创建。

创建索引模式后可以看到已创建的索引模式。

#### 查看日志

至少有一个索引模式之后即可查看日志。点击最左侧 Discover 按钮即可查看。

## ELK 数据迁移

ELK 的数据迁移流程一般如下：

1. 在新节点上创建索引 Mapping
2. 在新节点上配置允许的数据来源白名单
3. 在新节点上从旧节点拉取数据
4. 删除旧节点数据

### 新节点创建索引

这里要注意，数据来源和数据目的地的索引格式要相同，否则无法迁移，因此创建的索引 mapping 需要与旧节点索引相同。

创建索引代码可参考附录。

### 配置白名单

编辑`config/elasticsearch.yml`，增加如下配置

```yml
reindex.remote.whitelist: 旧节点IP:9200
```

重启 Elasticsearch 使得配置生效

### 从旧节点拉取数据

在新节点上执行如下代码，根据需要配置地址、用户名密码、索引名等。

```json
POST _reindex
{
  "source": {
    "remote": {
      "host": "http://旧节点:9200",
      "username": "user",
      "password": "pass"
    },
    "index": "旧节点索引名"
  },
  "dest": {
    "index": "新节点索引名"
  }
}
```

### 删除旧节点数据

根据需要可使用 Kibana 删除旧节点的数据。在索引管理界面勾选索引选择删除。
