@[TOC](Prometheus+Grafana监控Docker容器和Linux主机)

# Prometheus概述
Prometheus发布于2016年，比zabbix晚了4年。虽然其社区支持暂时不如后者完善，但是对容器的支持更好，不仅支持swarm原生群集，还支持kubernetes容器集群的监控。

Prometheus的特点包括：
-	多维数据模型：由度量名称和键值对标识的时间序列数据；
-	PromQL：一种灵活的查询语言，可以利用多维数据完成复杂查询；
-	不依赖分布式存储，单个服务器节点可直接工作；
-	基于HTTP的pull方式采集时间序列数据；
-	支持通过PushGateway组件推送时间序列数据；
-	通过服务发现（service discovery）或静态配置发现目标；
-	多种图形模式及仪表盘支持（grafana）。

**架构**
Prometheus+Grafana的架构如下：

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210204145105217.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)

 

**组件**
| 主要组件 | 功能 |
| -- | -- |
| Prometheus Server | 收集指标和存储时间序列数据，并提供查询接口 |
| ClientLibrary | 客户端库 |
| Push Gateway | 短期存储指标数据，主要用于临时性任务 |
| Exporters | 采集已有的第三方服务监控指标并暴露metrics |
| Alertmanager | 处理告警 |
| Web UI | 简单的Web控制台 |

**实例和作业**
实例（Instance）和作业（Job）是Prometheus中的两个重要概念。实例是指可以抓取的目标（即被监控端，通常是单个的进程），而作业则是指具有相同目标的实例的集合。 

# Prometheus部署
Prometheus有四种常见的部署方法：
-	使用预编译的二进制文件（pre-compiled binaries）安装；
-	通过源码编译（from source）；
-	使用docker部署；
-	通过配置管理系统安装：比如Ansible、Chef、Puppet和SaltStack。

下面主要介绍第三种方法（最简单最方便），其它方法参见 `https://prometheus.io/docs/prometheus/latest/installation/` 。

**使用docker部署**
需要准备配置文件prometheus.yml（见后面的内容）。
```bash
$ docker run -d -p 9090:9090 prom/prometheus  # 最简单的运行命令
# 运行并绑定挂载主机上的配置文件prometheus.yml
$ docker run -d --name=prometheus -p 9090:9090 \
-v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \  
prom/prometheus
# 访问http://主机IP:9090
```

# Prometheus配置文件
## prometheus.yml
Prometheus的配置语言用YAML语言编写，其格式一般如下：
```bash
# global关键字用于定义全局参数
global:
  # 定义从被监控目标采集数据的频率，默认为1分钟
  [ scrape_interval: <duration> | default = 1m ]

  # 定义数据采集请求的超时时间，默认为10秒
  [ scrape_timeout: <duration> | default = 10s ]

  # 评估规则（告警检测）的频率，默认为1分钟
  [ evaluation_interval: <duration> | default = 1m ]

  # 跟外部系统（federation, remote storage, Alertmanager）通信时向任意时间序列或告警添加的标签
  external_labels:
    [ <labelname>: <labelvalue> ... ]

  # 记录PromQL查询语句的文件。重新加载配置会重新打开此文件
  [ query_log_file: <string> ]

# rule_files定义了告警规则文件列表
rule_files:
  [ - <filepath_glob> ... ]

# scrape_configs定义了监控数据抓取的配置列表
scrape_configs:
  [ - <scrape_config> ... ]

# alerting定义了与Alertmanager相关的告警配置
alerting:
  alert_relabel_configs:
    [ - <relabel_config> ... ]
  alertmanagers:
    [ - <alertmanager_config> ... ]

# remote_write和remote_read定义了远程读写的相关配置
remote_write:
  [ - <remote_write> ... ]

remote_read:
  [ - <remote_read> ... ]
```

## scrape_configs
下面主要介绍scrape_configs的配置参数，它定义了监控目标（target）的集合并描述了从监控目标采集数据的相关参数。监控目标可以通过`static_configs`关键字配置，也可以通过使用服务发现机制来动态地发现。使用`relabel_configs`关键字可以在采集数据前对任意监控目标及其标签（label）进行进一步的修改。
```bash
# 数据采集作业的名称
job_name: <job_name>

# 数据采集的频率，默认为对应的全局配置
[ scrape_interval: <duration> | default = <global_config.scrape_interval> ]

# 数据采集的超时时间，默认为对应的全局配置
[ scrape_timeout: <duration> | default = <global_config.scrape_timeout> ]

# 从监控目标获取指标的HTTP资源路径
[ metrics_path: <path> | default = /metrics ]

# 当采集的监控数据的标签与服务器端配置的不一样时，如果honor_labels设置为false，则以服务器端为准，否则以采集数据的标签为准
[ honor_labels: <boolean> | default = false ]

# honor_timestamps设置为true时，使用采集的监控数据的时间戳
[ honor_timestamps: <boolean> | default = true ]

# 监控数据采集请求使用的协议，默认为http
[ scheme: <scheme> | default = http ]

# 可选的HTTP URL参数
params:
  [ <string>: [<string>, ...] ]

# 使用配置的用户名和密码设置每个数据采集请求的Authorization头部标签。其中password与password_file配置互斥
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]

# 使用bear token设置每个数据采集请求的Authorization头部标签。与bearer_token_file配置互斥
[ bearer_token: <secret> ]

# 使用bear token设置每个数据采集请求的Authorization头部标签。与bearer_token配置互斥
[ bearer_token_file: <filename> ]

# 数据采集请求的TLS配置（安全传输层协议）
tls_config:
  [ <tls_config> ]

# 可选的代理URL.
[ proxy_url: <string> ]

# Azure服务发现的配置清单
azure_sd_configs:
  [ - <azure_sd_config> ... ]

# Consul服务发现的配置清单
consul_sd_configs:
  [ - <consul_sd_config> ... ]

# DigitalOcean服务发现的配置清单
digitalocean_sd_configs:
  [ - <digitalocean_sd_config> ... ]

# Docker Swarm服务发现的配置清单
dockerswarm_sd_configs:
  [ - <dockerswarm_sd_config> ... ]

# DNS服务发现的配置清单
dns_sd_configs:
  [ - <dns_sd_config> ... ]

# EC2服务发现的配置清单
ec2_sd_configs:
  [ - <ec2_sd_config> ... ]

# Eureka服务发现的配置清单
eureka_sd_configs:
  [ - <eureka_sd_config> ... ]

# file服务发现的配置清单
file_sd_configs:
  [ - <file_sd_config> ... ]

# GCE服务发现的配置清单
gce_sd_configs:
  [ - <gce_sd_config> ... ]

# Hetzner服务发现的配置清单
hetzner_sd_configs:
  [ - <hetzner_sd_config> ... ]

# Kubernetes服务发现的配置清单
kubernetes_sd_configs:
  [ - <kubernetes_sd_config> ... ]

# Marathon服务发现的配置清单
marathon_sd_configs:
  [ - <marathon_sd_config> ... ]

# AirBnB's Nerve服务发现的配置清单
nerve_sd_configs:
  [ - <nerve_sd_config> ... ]

# OpenStack服务发现的配置清单
openstack_sd_configs:
  [ - <openstack_sd_config> ... ]

# Zookeeper Serverset服务发现的配置清单
serverset_sd_configs:
  [ - <serverset_sd_config> ... ]

# Triton服务发现的配置清单
triton_sd_configs:
  [ - <triton_sd_config> ... ]

# 为当前作业静态配置的监控目标清单.
static_configs:
  [ - <static_config> ... ]

# 监控目标relabel重新设定标签配置
relabel_configs:
  [ - <relabel_config> ... ]

# 指标relabel配置清单
metric_relabel_configs:
  [ - <relabel_config> ... ]

# 监控数据采样的数目限制，默认为0表示无限制
[ sample_limit: <int> | default = 0 ]

# 数据采集的监控目标的个数限制，默认为0表示无限制
[ target_limit: <int> | default = 0 ]
```

其他参数的配置参见 `https://prometheus.io/docs/prometheus/latest/configuration/configuration/ `。

详细配置可以参考示例：[https://github.com/prometheus/prometheus/blob/release-2.24/documentation/examples/prometheus-dockerswarm.yml](https://github.com/prometheus/prometheus/blob/release-2.24/documentation/examples/prometheus-dockerswarm.yml)。


# 使用Prometheus监控docker容器
常用的监控指标有内存、CPU、磁盘、网络。一般使用Google开源的cAdvisor (Container Advisor)来收集正在运行的容器资源信息和性能信息（参见 `https://github.com/google/cadvisor`）。

## 使用cAdvisor采集监控数据（被监控端）
使用Docker部署cadvisor。
```bash
$ docker run -d --volume=/:/rootfs:ro \
--volume=/var/run:/var/run:ro --volume=/sys:/sys:ro \
--volume=/var/lib/docker/:/var/lib/docker:ro \
--volume=/dev/disk/:/dev/disk:ro \
--publish=8080:8080 --detach=true \
--name=cadvisor google/cadvisor:latest
# 访问http://主机IP:8080
```
cAdvisor不负责存储采集到的监控数据，因此只能查看实时数据。cAdvisor为Prometheus提供的数据采集接口为`http://被监控主机IP:8080/metrics`。

## 修改prometheus配置文件
在prometheus.yml的scrape_configs中添加监控作业。
```bash
- job_name: "my_docker"
  static_configs:
  - targets: ['被监控主机IP:8080']
```
然后重启容器：`docker restart prometheus`。可以在`http://主机IP:9090/targets` 页面可以查看prometheus监控的目标。在`http://主机IP:9090/graph` 页面可以查看监控参数的简单图形化展示。


## 使用Grafana可视化监控数据
Grafana则是一个开源的度量分析和可视化系统（https://grafana.com/docs/）。它比Prometheus自带的图形化展示功能更强大。

**Docker部署Grafana**
```bash
$ docker run -d --name=grafana -p 3000:3000 grafana/grafana
# 访问http://主机IP:3000，默认用户名和密码都是admin，登录并修改密码
```

**添加数据源**
在添加数据源中选择Prometheus，在HTTP下的URL栏中粘贴`http://Prometheus主机IP:9090`并保存。


**创建仪表盘（dashboard）**
在新建仪表盘（New dashboard）中点击右侧的导入仪表盘（Import dashboard），输入并搜索193（对应仪表盘名称为docker monitoring），在显示的仪表盘选项（Options）中选择数据源为Prometheus，最后点击导入即可。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210206125124940.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)


在 `https://grafana.com/grafana/dashboards` 上可以发现更多官方和社区提供的仪表盘。


# 使用Prometheus监控Linux主机
Prometheus监控Linux主机需要在被监控目标上安装Node Exporter。在目标主机上执行以下脚本来部署node exporter。
（参见 `https://prometheus.io/docs/guides/node-exporter/#monitoring-linux-host-metrics-with-the-node-exporter`）

## 安装node exporter（被监控端）
```bash
#!/bin/bash

wget https://github.com/prometheus/node_exporter/releases/download/v0.17.0/node_exporter-0.17.0.linux-amd64.tar.gz

tar zxf node_exporter-0.17.0.linux-amd64.tar.gz
mv node_exporter-0.17.0.linux-amd64 /usr/local/node_exporter

cat << EOF > /usr/lib/systemd/system/node_exporter.service
[Unit]
Description=https://prometheus.io

[Service]
Restart=on-failure
ExecStart=/usr/local/node_exporter/node_exporter

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable node_exporter
systemctl restart node_exporter
```
安装完成以后使用`ps -ef | grep node_exporter`查看是否正常运行。Node exporter默认会使用9100端口，为Prometheus提供的数据采集接口为`http://被监控主机IP:9100/metrics`。

## 修改prometheus配置文件
在prometheus.yml的scrape_configs中添加监控作业。
```bash
- job_name: "my_linux"
  static_configs:
  - targets: ['被监控主机IP:9100']
```
然后重启容器：`docker restart prometheus`。可以在`http://主机IP:9090/targets` 页面可以查看prometheus监控的目标。在`http://主机IP:9090/graph` 页面可以查看监控参数的简单图形化展示。

## 使用Grafana可视化监控数据
Grafana提供的9276仪表盘可以很好地可视化Linux主机基础监控信息。参照上文中Grafana可视化容器监控的仪表盘导入操作。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210206125144884.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)






