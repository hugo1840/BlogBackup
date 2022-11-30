---
tags: [ELK]
title: ELK+Filebeat经典架构部署
created: '2022-11-21T06:28:44.099Z'
modified: '2022-11-30T04:53:05.169Z'
---

ELK+Filebeat经典架构部署

| 服务器角色 | IP地址 |
| :--: | :--: |
| Elasticsearch | 192.168.69.142-144 |
| Kibana | 192.168.69.144 |
| Filebeat | 192.168.69.143 |
| Logstash | 192.168.69.145 |


# 安装JAVA开发环境
根据需要在所有服务器上安装Java开发环境：
```bash
yum install java-1.8.0-openjdk-devel -y

vim /etc/profile
# 在末尾添加保存
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.352.b08-2.el7_9.x86_64
export JRE_HOME=$JAVA_HOME/jre
export CLASSPATH=$JAVA_HOME/lib:$JRE_HOME/lib:$CLASSPATH
export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH

# 使环境配置生效
source /etc/profile

# 检查Java版本
java -version
whereis java
```

# ES和Kibana部署

## ES部署

>ES部署参考：https://blog.csdn.net/Sebastien23/article/details/127998410

## XPack安全配置
ES内置了以下用户用于特殊用途：
- `elastic`：ES超级用户，拥有ES集群的所有权限；
- `kibana_system`：Kibana用于连接ES的用户；
- `logstash_system`：Logstash用于连接ES并存储监控数据的用户；
- `beats_system`：Beats用于连接ES并存储监控数据的用户；
- `apm_system`：APM Server用于连接ES并存储监控数据的用户；
- `remote_monitoring_user`：Metricbeat用于在ES中采集和存储监控数据的用户。

其中除了`elastic`以外都是最小权限用户。

要使用这些用户，必须先开启X-Pack安全特性。当我们的ES是Basic或者试用License时，ES安全特性默认是禁用的。

1. **开启安全特性**

修改每个ES节点的`elasticsearch.yml`配置文件以开启安全特性：
```yml
# 在当前节点开启安全特性
xpack.security.enabled: true
```

2. **生成证书**

使用`elasticsearch-certutil cert --ca`命令来为每个ES节点生成密钥文件：
```bash
cd /opt/elasticsearch/elasticsearch-7.9.3
# 为ES集群创建一个证书颁发机构（会生成一个elastic-stack-ca.p12的Keystore文件）
./bin/elasticsearch-certutil ca
# 生成证书（不用设置证书密码，会生成一个elastic-certificates.p12证书）
./bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12

# 把Keystore文件拷贝到ES配置文件目录下
mv elastic-stack-ca.p12 config/
# 拷贝证书到配置目录下的certs子目录下
mkdir config/certs
mv elastic-certificates.p12 config/certs/
# 最后把Keystore文件和证书拷贝到其他节点
```

`elasticsearch-certutil`命令生成的证书中不包含主机名信息，我们可以对集群中的每个节点使用该证书，但是不能启用主机名验证。


3. **开启节点间SSL安全传输**

修改每个ES节点的`elasticsearch.yml`配置文件以开启安全特性：
```yml
# 开启TLS/SSL安全传输协议
xpack.security.transport.ssl.enabled: true
# TLS/SSL认证级别设置为只验证证书
xpack.security.transport.ssl.verification_mode: certificate
# 密钥文件和证书的存放路径（与elasticsearch.yml同一级目录）
xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12
# 证书存放路径（与elasticsearch.yml同一级目录）
xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12
```

4. **重启ES**

```bash
cd /opt/elasticsearch/elasticsearch-7.9.3
pkill -F pid
./bin/elasticsearch -d -p pid
```

5. **为内置用户设置密码**

通过执行以下命令为上面的内置用户设置密码：
```bash
./bin/elasticsearch-setup-passwords interactive
```


## Kibana部署
准备工作：
```bash
# 解压安装包
cd /opt
mkdir kibana
cd kibana
mv /root/kibana-7.9.3-linux-x86_64.tar.gz .
tar -xzf kibana-7.9.3-linux-x86_64.tar.gz

# 使用ES用户部署
chown es:es -R /opt/kibana
```

修改配置文件`kibana.yml`：
```yml
# 配置ES实例IP、用户名及密码
# kibana_system用户密码为前面通过elasticsearch-setup-passwords命令配置
elasticsearch.hosts: ["http://192.168.69.142:9200","http://192.168.69.143:9200","http://192.168.69.144:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "yourPassword"

# Kibana服务端口，默认5601
server.port: 5601	  
# Kibana服务IP，默认localhost，仅允许本地访问；
# 修改为Kibana服务器IP地址或DNS域名以允许远程访问       
server.host: "192.168.69.144"     
```

启动Kibana：
```bash
./bin/kibana &
```

>**注**：kibana对应的进程为`./bin/../node/bin/node ./bin/../src/cli`。重启kibana时执行`lsof -i:5601`或者`ps -ef | grep node`查看占用端口的进程PID，并手动杀掉该进程。

在浏览器中通过以下地址访问Kibana前端：
```
http://192.168.69.144:5601/status
```


# Filebeat部署
Filebeat部署在要采集日志的服务器上，并且需要有对日志采集目录的读权限。

## 准备工作
```bash
# 解压安装包
cd /opt
mkdir filebeat
cd filebeat
mv /root/filebeat-7.9.3-linux-x86_64.tar.gz .
tar -xzf filebeat-7.9.3-linux-x86_64.tar.gz
cd filebeat-7.9.3-linux-x86_64

# 创建filebeat用户
groupadd filebeat
useradd filebeat -d /home/filebeat -g filebeat -s /bin/bash
chown filebeat:filebeat -R /opt/filebeat/

# 关闭防火墙
systemctl stop firewalld && systemctl disable firewalld
```

## Logstash配置
修改配置文件`filebeat.yml`中的Logstash output访问地址：
```yml
# 配置logstash输出地址
output.logstash:
  hosts: ["192.168.69.145:5044"]
```

## 数据采集源配置
日志采集源配置有两种方法：

方法一是直接修改`filebeat.yml`中的日志采集路径配置：
```yml
# 每个-代表一个数据采集源
filebeat.inputs:
- type: log
  enabled: true          # 是否开启
  paths:
    - /var/log/*.log     # 采集路径
```

方法二是使用Filebeat自带的模块采集配置。
```bash
# 检查可用的内置模块
./filebeat modules list

# 开启需要使用到的模块（这里启用了system/nginx/mysql三个模块）
./filebeat modules enable system nginx mysql
```

在Filebeat安装路径的`modules.d`目录下，配置相应模块的日志采集路径（以`modules.d/nginx.yml`为例）：
```yml
- module: nginx
  access:
    var.paths: ["/var/log/nginx/access.log*"] 
```

然后授予filebeat用户采集路径的读权限：
```bash
setfacl -R -m u:filebeat:r-x /var/log/
setfacl -R -d -m u:filebeat:r-x /var/log/
```

## 手动加载索引模板
Elasticsearch使用索引模板（index templates）定义:
- 控制索引行为的设置，包括用于管理索引的生命周期策略。
- 决定如何分析字段的映射。每个映射设置用于特定数据字段的ES数据类型。

如果采用`filebeat.yml`中的默认配置，Filebeat在成功连接ES后会自动加载索引模板。如果我们配置了`output.logstash`等其他输出，就必须手动将索引模板加载到ES中。

手动加载索引模板需要连接到Elasticsearch。如果启用了另一个输出配置（例如`output.logstash`），则需要临时禁用该输出，并使用`-E`选项启用`output.elasticsearch`。如果Elasticsearch输出已经启用，则可以省略`-E`标志。

在`filebeat.yml`中临时启用elasticsearch输出：
```yml
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["192.168.69.142:9200", "192.168.69.143:9200", "192.168.69.144:9200"]
  # 这里用于加载索引模板的必须是elastic超级用户，如果使用beats_system会报错unauthorized
  username: "elastic"
  password: "yourPassword"
```

通过`setup`命令来手动加载索引模板：
```bash
./filebeat setup --index-management -E output.logstash.enabled=false 
```

最后在`filebeat.yml`中通过注释禁用elasticsearch输出：
```yml
#output.elasticsearch:
  # Array of hosts to connect to.
  #hosts: ["192.168.69.142:9200", "192.168.69.143:9200", "192.168.69.144:9200"]
  # 这里用于加载索引模板的必须是elastic超级用户，如果使用beats_system会报错unauthorized
  #username: "elastic"
  #password: "yourPassword"
```


## 启动和检查
启动Filebeat：
```bash 
cd /opt/filebeat/filebeat-7.9.3-linux-x86_64
./filebeat -e &
```

停止Filebeat：
```bash
kill -15 PID   # SIGTERM
```


# Logstash部署
## 准备工作
```bash
# 解压logstash安装包
cd /opt
mkdir logstash
cd logstash
mv /root/logstash-7.9.3.tar.gz .
tar -xzf logstash-7.9.3.tar.gz
cd logstash-7.9.3

# 创建logstash用户
groupadd logstash
useradd logstash -d /home/logstash -g logstash -s /bin/bash
chown logstash:logstash -R /opt/logstash/

# 关闭防火墙
systemctl stop firewalld && systemctl disable firewalld
```

## 配置文件
根据需要修改配置文件`$LS_HOME/config/logstash.yml`，这里我们保持默认配置。


## Pipeline流程配置
修改管道配置文件`$LS_HOME/conf.d/logstash-sample.conf`，使得Logstash pipeline从指定的Beats获取输入，并将处理过后的数据发送给指定的ES实例。
```bash
# 输入插件类型为beats，配置Beats监听端口
input {
  beats {
    port => 5044
  }
}

filter {
}

# 指定elasticsearch和stdout两个输出插件，并配置ES访问地址
output {
  elasticsearch {
    hosts => ["192.168.69.144","192.168.69.142","192.168.69.143"]
    # 这里用logstash_system用户会报错，可能是权限不足
    #user => "logstash_system"
    user => "elastic"
    password => "yourPassword"
  }
  stdout { codec => rubydebug }
}
```

>:cat:**注**：如果在上面的conf文件的输出插件中使用logstash_system用户，启动时会输到如下报错：
>```
>[ERROR][logstash.outputs.elasticsearch][main] Failed to install template. 
>LogStash::Outputs::ElasticSearch::HttpClient::Pool::BadResponseCodeError: Got response code '403' contacting Elasticsearch
>```


## 启动和检查
启动Logstash：
```bash
su - logstash
cd /opt/logstash/logstash-7.9.3
./bin/logstash -f conf.d/logstash-sample.conf
```

检查Logstash运行日志：
```bash
tail -f logs/logstash-plain.log
```

停止Logstash：
```bash
kill -15 PID   # SIGTERM
```


**References**
【1】https://www.elastic.co/guide/en/logstash/7.9/deploying-and-scaling.html
【2】https://blog.csdn.net/Sebastien23/article/details/127998410
【3】https://blog.csdn.net/Sebastien23/article/details/128046123
【4】https://www.elastic.co/guide/en/beats/filebeat/7.9/filebeat-installation-configuration.html
【5】https://www.elastic.co/guide/en/logstash/7.9/plugins-inputs-beats.html
【6】https://www.elastic.co/guide/en/logstash/7.9/plugins-outputs-elasticsearch.html
【7】https://www.elastic.co/guide/en/beats/filebeat/7.9/logstash-output.html
【8】https://www.elastic.co/guide/en/beats/filebeat/7.9/filebeat-template.html#load-template-manually
【9】https://www.elastic.co/guide/en/beats/filebeat/7.9/configuration-filebeat-options.html
【10】https://www.elastic.co/guide/en/kibana/7.9/settings.html
【11】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/built-in-users.html
【12】https://www.elastic.co/guide/en/logstash/7.9/ls-security.html#ls-monitoring-user
【13】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/configuring-security.html
【14】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/security-settings.html
【15】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/configuring-tls.html












