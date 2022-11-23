---
tags: [ELK]
title: Elasticsearch集群部署
created: '2022-11-23T05:05:52.716Z'
modified: '2022-11-23T05:52:28.663Z'
---

Elasticsearch集群部署

# ES单机部署
ES单机部署，即部署了只包含单个节点的cluster。此时所有的主分片都位于同一节点上，且无法分配副本分片，因此存在故障时数据丢失风险。
```bash
# 解压安装包
cd /opt
mkdir elasticsearch
cd elasticsearch
cp /root/elasticsearch-7.9.3-linux-x86_64.tar.gz .
tar -xzf elasticsearch-7.9.3-linux-x86_64.tar.gz

# 创建es用户
groupadd es
useradd es -d /home/es -g es -s /bin/bash
chown es:es -R /opt/elasticsearch/

# 运行ES实例
su - es
cd /opt/elasticsearch/elasticsearch-7.9.3
./bin/elasticsearch
```

检查集群状态：
```bash
[root@es-node1 ~]# curl -XGET 'http://localhost:9200/_cluster/stats?pretty'
{
  "_nodes" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "oz075eUtSdK2aYbVHycQfw",
  "timestamp" : 1669021423695,
  "status" : "green",
  "indices" : {
    "count" : 0,
    "shards" : { },
    "docs" : {
      "count" : 0,
      "deleted" : 0
    },
    "store" : {
      "size_in_bytes" : 0,
      "reserved_in_bytes" : 0
    },
    "fielddata" : {
      "memory_size_in_bytes" : 0,
      "evictions" : 0
    },
    "query_cache" : {
      "memory_size_in_bytes" : 0,
      "total_count" : 0,
      "hit_count" : 0,
      "miss_count" : 0,
      "cache_size" : 0,
      "cache_count" : 0,
      "evictions" : 0
    },
    "completion" : {
      "size_in_bytes" : 0
    },
    "segments" : {
      "count" : 0,
      "memory_in_bytes" : 0,
      "terms_memory_in_bytes" : 0,
      "stored_fields_memory_in_bytes" : 0,
      "term_vectors_memory_in_bytes" : 0,
      "norms_memory_in_bytes" : 0,
      "points_memory_in_bytes" : 0,
      "doc_values_memory_in_bytes" : 0,
      "index_writer_memory_in_bytes" : 0,
      "version_map_memory_in_bytes" : 0,
      "fixed_bit_set_memory_in_bytes" : 0,
      "max_unsafe_auto_id_timestamp" : -9223372036854775808,
      "file_sizes" : { }
    },
    "mappings" : {
      "field_types" : [ ]
    },
    "analysis" : {
      "char_filter_types" : [ ],
      "tokenizer_types" : [ ],
      "filter_types" : [ ],
      "analyzer_types" : [ ],
      "built_in_char_filters" : [ ],
      "built_in_tokenizers" : [ ],
      "built_in_filters" : [ ],
      "built_in_analyzers" : [ ]
    }
  },
  "nodes" : {
    "count" : {
      "total" : 1,
      "coordinating_only" : 0,
      "data" : 1,
      "ingest" : 1,
      "master" : 1,
      "ml" : 1,
      "remote_cluster_client" : 1,
      "transform" : 1,
      "voting_only" : 0
    },
    "versions" : [
      "7.9.3"
    ],
    "os" : {
      "available_processors" : 2,
      "allocated_processors" : 2,
      "names" : [
        {
          "name" : "Linux",
          "count" : 1
        }
      ],
      "pretty_names" : [
        {
          "pretty_name" : "CentOS Linux 7 (Core)",
          "count" : 1
        }
      ],
      "mem" : {
        "total_in_bytes" : 3953963008,
        "free_in_bytes" : 555470848,
        "used_in_bytes" : 3398492160,
        "free_percent" : 14,
        "used_percent" : 86
      }
    },
    "process" : {
      "cpu" : {
        "percent" : 1
      },
      "open_file_descriptors" : {
        "min" : 256,
        "max" : 256,
        "avg" : 256
      }
    },
    "jvm" : {
      "max_uptime_in_millis" : 596622,
      "versions" : [
        {
          "version" : "15",
          "vm_name" : "OpenJDK 64-Bit Server VM",
          "vm_version" : "15+36-1562",
          "vm_vendor" : "Oracle Corporation",
          "bundled_jdk" : true,
          "using_bundled_jdk" : true,
          "count" : 1
        }
      ],
      "mem" : {
        "heap_used_in_bytes" : 235726336,
        "heap_max_in_bytes" : 1073741824
      },
      "threads" : 37
    },
    "fs" : {
      "total_in_bytes" : 18238930944,
      "free_in_bytes" : 15545573376,
      "available_in_bytes" : 15545573376
    },
    "plugins" : [ ],
    "network_types" : {
      "transport_types" : {
        "security4" : 1
      },
      "http_types" : {
        "security4" : 1
      }
    },
    "discovery_types" : {
      "zen" : 1
    },
    "packaging_types" : [
      {
        "flavor" : "default",
        "type" : "tar",
        "count" : 1
      }
    ],
    "ingest" : {
      "number_of_pipelines" : 1,
      "processor_stats" : {
        "gsub" : {
          "count" : 0,
          "failed" : 0,
          "current" : 0,
          "time_in_millis" : 0
        },
        "script" : {
          "count" : 0,
          "failed" : 0,
          "current" : 0,
          "time_in_millis" : 0
        }
      }
    }
  }
}
```

# ES集群部署
下面我们搭建一个三节点的ES集群。我们可以将三个Node部署在同一台服务器上，无需任何网络配置，Elasticsearch将绑定到可用的环回地址，并将扫描本地端口**9300到9305**，以尝试连接到运行在同一服务器上的其他Node。

如果新增ES节点部署在不同的服务器上，则需要在`elasticsearch.yml`文件中配置`discovery.seed_hosts`参数，以便新增节点可以发现集群中的其他节点。

## 准备工作
在两个新增节点上创建好ES用户和安装目录：
```bash
# 解压安装包
cd /opt
mkdir elasticsearch
cd elasticsearch
cp /root/elasticsearch-7.9.3-linux-x86_64.tar.gz .
tar -xzf elasticsearch-7.9.3-linux-x86_64.tar.gz

# 创建es用户
groupadd es
useradd es -d /home/es -g es -s /bin/bash
chown es:es -R /opt/elasticsearch/
```

修改各节点配置文件`elasticsearch.yml`，增加以下配置参数：
```yml
# 自定义集群名称
cluster.name: rockstar   
# 根据节点分别修改为node-1/2/3 
node.name: node-1     
# 数据存放目录
path.data: /opt/elasticsearch/elasticsearch-7.9.3/data
# 日志存放目录
path.logs: /opt/elasticsearch/elasticsearch-7.9.3/logs
# 不同节点所服务器IP，端口默认为9300
discovery.seed_hosts: ["192.168.169.144:9300", "192.168.169.142:9300", "192.168.169.143:9300"]
```

## 启动Node1
在节点1上，以守护进程模式运行ES，并将PID记录到文件中：
```bash
[es@es-node1 elasticsearch-7.9.3]$ ./bin/elasticsearch -d -p pid
[es@es-node1 elasticsearch-7.9.3]$ cat pid
4710[es@es-node1 elasticsearch-7.9.3]$
```

检查集群健康状态：
```bash
[es@es-node1 ~]$  curl -XGET 'http://localhost:9200/_cat/health?v'
epoch      timestamp cluster  status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1669174627 03:37:07  rockstar green           1         1      0   0    0    0        0             0                  -                100.0%
[es@es-node1 ~]$
```

ES集群有三种状态：
- 集群状态为`green`，表示集群健康且副本分片可用；
- 集群状态为`yellow`，表示集群功能上可用、但是无法创建副本分片到其他节点，存在数据丢失风险；
- 集群状态为`red`，表示有数据不可用。


守护进程运行模式下，可以通过杀掉ES进程来停止ES：
```bash
[es@es-node1 elasticsearch-7.9.3]$ pkill -F pid
[es@es-node1 elasticsearch-7.9.3]$
```

## 添加Node2
在新增节点2上运行ES：
```bash
[es@es-node2 ~]$ cd /opt/elasticsearch/elasticsearch-7.9.3
[es@es-node2 elasticsearch-7.9.3]$ ./bin/elasticsearch -d -p pid
```

检查集群健康状态：
```bash
[es@es-node1 ~]$  curl -XGET 'http://localhost:9200/_cat/health?v'
epoch      timestamp cluster  status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1669174627 03:37:07  rockstar green           1         1      0   0    0    0        0             0                  -                100.0%
[es@es-node1 ~]$

[es@es-node2 elasticsearch-7.9.3]$ curl -XGET 'http://localhost:9200/_cat/health?v'

^C
[es@es-node2 elasticsearch-7.9.3]$ cat pid
2183[es@es-node2 elasticsearch-7.9.3]$ ps -ef | grep 2183
```
看上去像是node1和node2的ES进程都起来了，但是新增节点node2并没有自动加入集群。

## 配置`cluster.initial_master_nodes`
检查日志发现如下报错：
```bash
[es@es-node2 elasticsearch-7.9.3]$ tail -f logs/rockstar.log
[2022-11-22T22:53:27,759][WARN ][o.e.c.c.ClusterFormationFailureHelper] [node-2] master not discovered 
yet, this node has not previously joined a bootstrapped (v7+) cluster, and 
[cluster.initial_master_nodes] is empty on this node: have discovered [{node-2}
{LpPlF8IBSpa_aNb5UQ2-Eg}{sa3ZJOCjS6qyJnOfacxe4Q}{127.0.0.1}{127.0.0.1:9300}{dilmrt}
{ml.machine_memory=3953963008, xpack.installed=true, transform.node=true, ml.max_open_jobs=20}]; 
discovery will continue using [192.168.169.144:9300, 192.168.169.142:9300, 192.168.169.143:9300] 
from hosts providers and [{node-2}{LpPlF8IBSpa_aNb5UQ2-Eg}{sa3ZJOCjS6qyJnOfacxe4Q}{127.0.0.1}{127.0.0.1:9300}
{dilmrt}{ml.machine_memory=3953963008, xpack.installed=true, transform.node=true, ml.max_open_jobs=20}] 
from last-known cluster state; node term 0, last-accepted version 0 in term 0
```

关闭掉所有节点的防火墙：
```bash
systemctl stop firewalld  && systemctl disable firewalld
```

在最先启动的**Node1**的`elasticsearch.yml`文件中添加以下配置（新增节点无需配置）：
```ym
# 集群启动时初始选举中可以参与投票的节点，与node.name一致
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
```

修改`elasticsearch.yml`中各节点绑定的IP：
```yml
network.host: 192.168.169.142     # 根据节点调整IP
```

尝试重新启动节点1，收到以下报错：
```bash
[es@es-node1 elasticsearch-7.9.3]$ ERROR: [2] bootstrap checks failed
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65535]
[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
ERROR: Elasticsearch did not exit normally - check the logs at /opt/elasticsearch/elasticsearch-7.9.3/logs/rockstar.log
```
看上去是ES进程被分配的系统资源不足，导致启动失败了。

## 修改系统资源配额
修改所有节点es用户的**max file descriptors**：
```bash
[es@es-node1 elasticsearch-7.9.3]$ ulimit -Hn
4096
[es@es-node1 elasticsearch-7.9.3]$ exit
logout
[root@es-node1 elasticsearch-7.9.3]# vim /etc/security/limits.conf
[root@es-node1 elasticsearch-7.9.3]#
[root@es-node1 elasticsearch-7.9.3]# grep -v '#' /etc/security/limits.conf

es   soft   nofile   65535
es   hard   nofile   65535

[root@es-node1 elasticsearch-7.9.3]# su - es
[es@es-node1 ~]$ ulimit -Hn
65535
```

修改所有节点的**max virtula memory**：
```bash
[root@es-node1 elasticsearch-7.9.3]# vim /etc/sysctl.conf
[root@es-node1 elasticsearch-7.9.3]#
[root@es-node1 elasticsearch-7.9.3]# grep -v '#' /etc/sysctl.conf
vm.max_map_count = 262144
[root@es-node1 elasticsearch-7.9.3]#
[root@es-node1 elasticsearch-7.9.3]# sysctl -p
vm.max_map_count = 262144
```

## 启动集群
重新启动节点1：
```bash
[es@es-node1 elasticsearch-7.9.3]$ ./bin/elasticsearch -d -p pid
[es@es-node1 elasticsearch-7.9.3]$ curl -XGET 'http://192.168.169.144:9200/_cat/health?v'
epoch      timestamp cluster  status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1669179514 04:58:34  rockstar green           1         1      0   0    0    0        0             0                  -                100.0%
[es@es-node1 elasticsearch-7.9.3]$
```

重新启动节点2：
```bash
[es@es-node2 elasticsearch-7.9.3]$ ./bin/elasticsearch -d -p pid
[es@es-node2 elasticsearch-7.9.3]$ curl -XGET 'http://192.168.169.144:9200/_cat/health?v'
epoch      timestamp cluster  status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1669179642 05:00:42  rockstar green           2         2      0   0    0    0        0             0                  -                100.0%
```

重新启动节点3：
```bash
[es@es-node3 elasticsearch-7.9.3]$  ./bin/elasticsearch -d -p pid
[es@es-node3 elasticsearch-7.9.3]$ curl -XGET 'http://192.168.169.144:9200/_cat/health?v'
epoch      timestamp cluster  status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1669179758 05:02:38  rockstar green           3         3      0   0    0    0        0             0                  -                100.0%
[es@mysql-node2 elasticsearch-7.9.3]$
```



**References**
【1】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/targz.html
【2】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/add-elasticsearch-nodes.html
【3】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/discovery-settings.html#unicast.hosts
【4】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/discovery-settings.html#initial_master_nodes


