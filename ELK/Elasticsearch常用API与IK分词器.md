---
tags: [ELK]
title: Elasticsearch常用API与IK分词器
created: '2022-11-29T06:37:17.802Z'
modified: '2022-12-01T06:58:13.506Z'
---

Elasticsearch常用API与IK分词器

**ES版本： 7.9**

# 常用管理查询API命令
通过cat API可以获取方便人眼阅读的信息。

cat API命令支持指定参数`v`（verbose）来显示详细信息。
```bash
# 获取master节点信息
curl -uelastic -XGET 'http://${Elasticsearch_IP}:9200/_cat/master'
curl -uelastic -XGET 'http://${Elasticsearch_IP}:9200/_cat/master?v'
```

cat API命令支持指定参数`help`来显示输出中包含哪些列，通过`h=colName1,colName2,...`来控制输出中只显示特定的列。
```bash
[es@es-node1 ~]$ curl -uelastic -XGET 'http://${Elasticsearch_IP}:9200/_cat/master?help'
Enter host password for user 'elastic':
id   |   | node id
host | h | host name
ip   |   | ip address
node | n | node name
[es@es-node1 ~]$ curl -uelastic -XGET 'http://${Elasticsearch_IP}:9200/_cat/master?h=ip,node'
Enter host password for user 'elastic':
192.168.xx.xx node-2
```

****
以下示例中，假设ES节点IP为192.168.12.142-144。如果未开启安全特性，curl命令中可以省略`-uelastic`。


## 集群和节点状态
查看集群健康状态：
```bash
$ curl -uelastic -XGET 'http://192.168.12.144:9200/_cat/health?v'
Enter host password for user 'elastic':
epoch      timestamp cluster  status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1669798978 09:02:58  rockstar green           3         3     18   9    0    0        0             0                  -                100.0%
```

获取指定节点的简要信息：
```bash
$ curl -uelastic 'http://192.168.12.144:9200'
Enter host password for user 'elastic':
{
  "name" : "node-1",
  "cluster_name" : "rockstar",
  "cluster_uuid" : "oz075eUtSdK2bVHycQfw",
  "version" : {
    "number" : "7.9.3",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "c4138e51121ef06a640866cddc601906fe5c868",
    "build_date" : "2020-10-16T10:36:16.141335Z",
    "build_snapshot" : false,
    "lucene_version" : "8.6.2",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

获取集群中指定节点信息：
```bash
# 全部信息，包括http/ingest/os/jvm/process/plugins/settings等
$ curl -uelastic -XGET 'http://192.168.12.144:9200/_nodes/node-1,node-2/all?pretty'

# 获取节点1和节点2的JVM信息
$ curl -uelastic -XGET 'http://192.168.12.144:9200/_nodes/node-1,node-2/jvm?pretty'
Enter host password for user 'elastic':
{
  "_nodes" : {
    "total" : 2,
    "successful" : 2,
    "failed" : 0
  },
  "cluster_name" : "rockstar",
  "nodes" : {
    "rYGEaADYRBSAEQPWNkyY5g" : {
      "name" : "node-1",
      "transport_address" : "192.168.12.144:9300",
      "host" : "192.168.12.144",
      "ip" : "192.168.12.144",
      "version" : "7.9.3",
      "build_flavor" : "default",
      "build_type" : "tar",
      "build_hash" : "c4138e51121ef06a6404866cddc601906fe5c868",
      "roles" : [
        "data",
        "ingest",
        "master",
        "ml",
        "remote_cluster_client",
        "transform"
      ],
      "jvm" : {
        "pid" : 2219,
        "version" : "15",
        "vm_name" : "OpenJDK 64-Bit Server VM",
        "vm_version" : "15+36-1562",
        "vm_vendor" : "Oracle Corporation",
        "bundled_jdk" : true,
        "using_bundled_jdk" : true,
        "start_time_in_millis" : 1669795711311,
        "mem" : {
          "heap_init_in_bytes" : 1073741824,
          "heap_max_in_bytes" : 1073741824,
          "non_heap_init_in_bytes" : 7667712,
          "non_heap_max_in_bytes" : 0,
          "direct_max_in_bytes" : 0
        },
        "gc_collectors" : [
          "G1 Young Generation",
          "G1 Old Generation"
        ],
        "memory_pools" : [
        ...
```

查看集群详细状态：
```bash
$ curl -uelastic -XGET 'http://192.168.12.144:9200/_cluster/stats?pretty'
Enter host password for user 'elastic':
{
  "_nodes" : {
    "total" : 3,
    "successful" : 3,
    "failed" : 0
  },
  "cluster_name" : "rockstar",
  "cluster_uuid" : "oz075eUtSdK2aYbVHycQfw",
  "timestamp" : 1669799218973,
  "status" : "green",
  "indices" : {
    "count" : 9,
    "shards" : {
      ...
```

获取集群配置：
```bash
$ curl -uelastic -XGET 'http://192.168.12.144:9200/_cluster/settings'
Enter host password for user 'elastic':
{"persistent":{},"transient":{}}
```

## 索引和分片信息
获取集群的Index信息：
```bash
$ curl -uelastic -XGET 'http://192.168.12.144:9200/_cat/indices?v'
Enter host password for user 'elastic':
health status index                            uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .security-7                      ZTMNHmkmTZqtNIWPraQWtA   1   1         49            0    273.4kb        136.7kb
green  open   .kibana-event-log-7.9.3-000001   s2kg5nJtRn2uUDk42CaphA   1   1          1            0     11.1kb          5.5kb
green  open   logstash-2022.11.30-000001       Dew3Q7p7Q228zmm26RLQIQ   1   1        338            0    289.6kb        144.4kb
green  open   .apm-custom-link                 RJ1jXnaCRqOsGaIPSsR27w   1   1          0            0       416b           208b
green  open   .kibana_task_manager_1           sFtwt3f2SieW9V0CfxHOBw   1   1          6          233    266.5kb        133.2kb
green  open   .apm-agent-configuration         4NhRxRoERtaqYGKPTRjiiA   1   1          0            0       416b           208b
green  open   filebeat-7.9.3-2022.11.28-000001 W5HfRyURSeabIcqjfOkgFw   1   1          0            0       416b           208b
green  open   .kibana_1                        NKNsET5_Qei_QP5Cxl1yzw   1   1         13            4     20.8mb         10.4mb
```

查看指定索引的信息：
```bash
# 查看索引logstash-2022.11.30-000001的详细信息
$ curl -uelastic -XGET 'http://192.168.12.144:9200/logstash-2022.11.30-000001?pretty'
Enter host password for user 'elastic':
{
  "logstash-2022.11.30-000001" : {
    "aliases" : {
      "logstash" : {
        "is_write_index" : true
      }
    },
    "mappings" : {
      "dynamic_templates" : [
        {
          "message_field" : {
            "path_match" : "message",
            "match_mapping_type" : "string",
            "mapping" : {
              "norms" : false,
              "type" : "text"
              ...
```

获取集群分片信息：
```bash
$ curl -uelastic -XGET 'http://192.168.12.143:9200/_cat/shards?v'
Enter host password for user 'elastic':
index                            shard prirep state   docs   store ip              node
.security-7                      0     p      STARTED   49 136.7kb 192.168.12.144 node-1
.security-7                      0     r      STARTED   49 136.7kb 192.168.12.143 node-3
.kibana_task_manager_1           0     p      STARTED    6 133.2kb 192.168.12.142 node-2
.kibana_task_manager_1           0     r      STARTED    6 133.2kb 192.168.12.143 node-3
filebeat-7.9.3-2022.11.28-000001 0     r      STARTED    0    208b 192.168.12.144 node-1
filebeat-7.9.3-2022.11.28-000001 0     p      STARTED    0    208b 192.168.12.142 node-2
.apm-agent-configuration         0     r      STARTED    0    208b 192.168.12.144 node-1
.apm-agent-configuration         0     p      STARTED    0    208b 192.168.12.142 node-2
.kibana_1                        0     p      STARTED   13  10.4mb 192.168.12.142 node-2
.kibana_1                        0     r      STARTED   13  10.4mb 192.168.12.143 node-3
logstash-2022.11.30-000001       0     p      STARTED  338 144.4kb 192.168.12.144 node-1
logstash-2022.11.30-000001       0     r      STARTED  338 145.1kb 192.168.12.143 node-3
ilm-history-2-000001             0     p      STARTED              192.168.12.144 node-1
ilm-history-2-000001             0     r      STARTED              192.168.12.142 node-2
.kibana-event-log-7.9.3-000001   0     p      STARTED    1   5.5kb 192.168.12.142 node-2
.kibana-event-log-7.9.3-000001   0     r      STARTED    1   5.5kb 192.168.12.143 node-3
.apm-custom-link                 0     p      STARTED    0    208b 192.168.12.144 node-1
.apm-custom-link                 0     r      STARTED    0    208b 192.168.12.143 node-3
```

# IK中文分词插件
到`https://github.com/medcl/elasticsearch-analysis-ik/releases/tag/v7.9.3`下载支持中文分词的IK插件（与ES版本一一对应）。

安装IK中文分词插件：
```bash
# 上传IK安装包到各个ES节点
chown es:es elasticsearch-analysis-ik-7.9.3.zip
mv elasticsearch-analysis-ik-7.9.3.zip /opt/elasticsearch/elasticsearch-7.9.3/plugins/

# 解压IK安装包到plugins/ik路径下
su - es
cd /opt/elasticsearch/elasticsearch-7.9.3/plugins
mkdir ik
mv elasticsearch-analysis-ik-7.9.3.zip ik/
cd ik && unzip elasticsearch-analysis-ik-7.9.3.zip
rm elasticsearch-analysis-ik-7.9.3.zip

# 重启ES
cd ../..
pkill -F pid
./bin/elasticsearch -d -p pid
```

通过Analyze API可以查看分词插件的效果。先测试一下ES自带的标准分词插件对中文的分词效果：
```bash
$ curl -uelastic -XGET 'http://192.168.12.144:9200/_analyze?pretty' -H 'content-Type:application/json' -d '
{
"analyzer": "standard",
"text": "我是东北人"
}'
Enter host password for user 'elastic':
{
  "tokens" : [
    {
      "token" : "我",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "<IDEOGRAPHIC>",
      "position" : 0
    },
    {
      "token" : "是",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "<IDEOGRAPHIC>",
      "position" : 1
    },
    {
      "token" : "东",
      "start_offset" : 2,
      "end_offset" : 3,
      "type" : "<IDEOGRAPHIC>",
      "position" : 2
    },
    {
      "token" : "北",
      "start_offset" : 3,
      "end_offset" : 4,
      "type" : "<IDEOGRAPHIC>",
      "position" : 3
    },
    {
      "token" : "人",
      "start_offset" : 4,
      "end_offset" : 5,
      "type" : "<IDEOGRAPHIC>",
      "position" : 4
    }
  ]
}
```

再测试一下IK分词插件对中文的分词效果：
```bash
$ curl -uelastic -XGET 'http://192.168.12.144:9200/_analyze?pretty' -H 'content-Type:application/json' -d '
{
"analyzer": "ik_max_word",
"text": "我是东北人"
}'
Enter host password for user 'elastic':
{
  "tokens" : [
    {
      "token" : "我",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "CN_CHAR",
      "position" : 0
    },
    {
      "token" : "是",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "CN_CHAR",
      "position" : 1
    },
    {
      "token" : "东北人",
      "start_offset" : 2,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "东北",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 3
    },
    {
      "token" : "北人",
      "start_offset" : 3,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 4
    }
  ]
}
```

可以看到，比起ES自带的standard analyzer的逐字拆分，IK分词器的拆分结果明显更贴合中文语法习惯，能够提升ES检索中文关键字的效率。


# 常用数据变更API命令
## Index操作
### 创建索引
创建一个名为`myindex-test-001`的索引：
```bash
$ curl -uelastic -XPUT 'http://${ES_IP}:9200/myindex-test-001'
Enter host password for user 'elastic':
{"acknowledged":true,"shards_acknowledged":true,"index":"myindex-test-001"}
```

### 删除索引
删除指定名称的索引：
```bash
$ curl -uelastic -XDELETE 'http://${ES_IP}:9200/myindex-test-001'
Enter host password for user 'elastic':
{"acknowledged":true}
```

**:apple:示例**
创建一个名为`accounts-test-001`的索引：
```bash
$ curl -uelastic -XPUT 'http://192.168.12.143:9200/accounts-test-001' 
```

查看该索引的信息：
```bash
$ curl -uelastic -XGET 'http://192.168.12.143:9200/accounts-test-001?pretty'
Enter host password for user 'elastic':
{
  "accounts-test-001" : {
    "aliases" : { },
    "mappings" : { },
    "settings" : {
      "index" : {
        "creation_date" : "1669866666291",
        "number_of_shards" : "1",
        "number_of_replicas" : "1",
        "uuid" : "xPj6R4xwSg20SPkPISk2kw",
        "version" : {
          "created" : "7090399"
        },
        "provided_name" : "accounts-test-001"
      }
    }
  }
}
```

### 添加字段
通过**PUT mapping** API向该索引添加三个新的Field：name、title、description，同时指定分词插件：
```bash
# 只有text类型的属性才支持analyzer
$ curl -uelastic -XPUT 'http://192.168.12.143:9200/accounts-test-001/_mapping' -H 'content-Type:application/json' -d '
{
  "properties": {
    "name": {
      "type": "keyword"
    },
    "title": {
      "type": "text",
      "analyzer": "ik_max_word",
      "search_analyzer": "ik_max_word"
    },
    "description" : {
      "type": "text",
      "analyzer": "ik_max_word",
      "search_analyzer": "ik_max_word"
    }
  }
}'
Enter host password for user 'elastic':
{"acknowledged":true}
```

再次查看该索引的信息：
```bash
$ curl -uelastic -XGET 'http://192.168.12.143:9200/accounts-test-001?pretty'
Enter host password for user 'elastic':
{
  "accounts-test-001" : {
    "aliases" : { },
    "mappings" : {
      "properties" : {
        "description" : {
          "type" : "text",
          "analyzer" : "ik_max_word"
        },
        "name" : {
          "type" : "keyword"
        },
        "title" : {
          "type" : "text",
          "analyzer" : "ik_max_word"
        }
      }
    },
    "settings" : {
      "index" : {
        "creation_date" : "1669866666291",
        "number_of_shards" : "1",
        "number_of_replicas" : "1",
        "uuid" : "xPj6R4xwSg20SPkPISk2kw",
        "version" : {
          "created" : "7090399"
        },
        "provided_name" : "accounts-test-001"
      }
    }
  }
}
```
可以看到，mappings中出现了新增的字段属性。


## Document操作
### 插入记录
通过**PUT**请求向索引中插入一条记录（**需要指定Id**）：
```bash
curl -XPUT 'http://${ES_IP}:9200/${index_name}/_doc/Id'
```

示例：
```bash
# 插入一条document记录
$ curl -uelastic -XPUT 'http://192.168.12.143:9200/accounts-test-001/_doc/1' -H 'content-Type:application/json' -d '
{
  "name": "John Maston",
  "title": "牛仔",
  "description": "美国西部神枪手"
}'
Enter host password for user 'elastic':
{"_index":"accounts-test-001","_type":"_doc","_id":"1","_version":1,"result":"created","_shards":{"total":2,"successful":2,"failed":0},"_seq_no":0,"_primary_term":1}

# 查看插入的记录
$ curl -uelastic 'http://192.168.12.143:9200/accounts-test-001/_doc/1?pretty'
Enter host password for user 'elastic':
{
  "_index" : "accounts-test-001",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "name" : "John Maston",
    "title" : "牛仔",
    "description" : "美国西部神枪手"
  }
}
```

通过**POST**请求向索引中插入一条记录（不需要指定Id，ES会生成一个随机的Id）：
```bash
curl -XPOST 'http://${ES_IP}:9200/${index_name}/_doc/'
```

示例：
```bash
# Id由ES自动生成
$ curl -uelastic -XPOST 'http://192.168.12.143:9200/accounts-test-001/_doc/' -H 'content-Type:application/json' -d '
{
  "name": "Taylor Swift",
  "title": "歌手",
  "description": "美国乡村民谣流行乐女艺人"
}'
Enter host password for user 'elastic':
{"_index":"accounts-test-001","_type":"_doc","_id":"taYbzIQBn__-53X17Y8l","_version":1,"result":"created","_shards":{"total":2,"successful":2,"failed":0},"_seq_no":1,"_primary_term":1}

# 查看插入的记录
$ curl -uelastic 'http://192.168.12.143:9200/accounts-test-001/_doc/taYbzIQBn__-53X17Y8l?pretty'
Enter host password for user 'elastic':
{
  "_index" : "accounts-test-001",
  "_type" : "_doc",
  "_id" : "taYbzIQBn__-53X17Y8l",
  "_version" : 1,
  "_seq_no" : 1,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "name" : "Taylor Swift",
    "title" : "歌手",
    "description" : "美国乡村民谣流行乐女艺人"
  }
}

```

### 删除记录
通过**DELETE**请求删除索引中指定Id的记录：
```bash
curl -XDELETE 'http://${ES_IP}:9200/${index_name}/_doc/Id'
```

如果要删除符合特殊条件的记录，可以使用`_delete_by_query`接口。


### 更新记录
通过**PUT**请求和`_doc`接口插入记录时，如果指定的Id已经存在，就会更新该行文档：
```bash
$ curl -uelastic -XPUT 'http://192.168.12.143:9200/accounts-test-001/_doc/1' -H 'content-Type:application/json' -d '
{
  "name": "Arthur Morgan",
  "title": "牛仔",
  "description": "美国西部神枪手"
}'
Enter host password for user 'elastic':
{"_index":"accounts-test-001","_type":"_doc","_id":"1","_version":2,"result":"updated","_shards":{"total":2,"successful":2,"failed":0},"_seq_no":2,"_primary_term":1}
```
可以看到，返回结果中版本`_version`从1变成了2，同时`result`从created变成了updated，说明该行数据被更新了。这里的更新其实是“**完全替换**”，需要给出所有的字段。如果新的PUT命令中只给出了要修改的字段（比如description），那么其他未明确给出的字段（name和title）都会被舍弃！！！

如果只需要更新document中的个别字段，可以使用**POST**请求和`_update`接口。
```bash
$ curl -uelastic -XPOST 'http://192.168.12.143:9200/accounts-test-001/_update/1' -H 'content-Type:application/json' -d '
{
 "doc": {
  "description": "美国东部神枪手"
 }
}'
Enter host password for user 'elastic':
{"_index":"accounts-test-001","_type":"_doc","_id":"1","_version":5,"result":"updated","_shards":{"total":2,"successful":2,"failed":0},"_seq_no":5,"_primary_term":1}
```


# 常用数据查询API命令
使用**GET**方法和`_search`接口直接返回指定索引中的所有记录：
```bash
$ curl -uelastic 'http://192.168.12.143:9200/accounts-test-001/_search'
Enter host password for user 'elastic':
{"took":7,"timed_out":false,"_shards":{"total":1,"successful":1,"skipped":0,"failed":0},"hits":{"total":{"value":2,"relation":"eq"},"max_score":1.0,"hits":[{"_index":"accounts-test-001","_type":"_doc","_id":"1","_score":1.0,"_source":{"name":"Arthur Morgan","title":"牛仔","description":"美国东部神枪手"}},{"_index":"accounts-test-001","_type":"_doc","_id":"taYbzIQBn__-53X17Y8l","_score":1.0,"_source":
{
"name": "Taylor Swift",
"title": "歌手",
"description": "美国乡村民谣流行乐女艺人"
}}]}}
```

## match查询
通过`_search`接口的match查询可以检索指定字段中的关键字：
```bash
# 查询accounts-test-001索引中description字段包含“美国”的文档
$ curl -uelastic 'http://192.168.12.143:9200/accounts-test-001/_search?pretty=true' -H 'content-Type:application/json' -d '
{
  "query": {
    "match": {
      "description": "美国"
    }
  }
}'
Enter host password for user 'elastic':
{
  "took" : 5,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.21110919,
    "hits" : [
      {
        "_index" : "accounts-test-001",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.21110919,
        "_source" : {
          "name" : "Arthur Morgan",
          "title" : "牛仔",
          "description" : "美国东部神枪手"
        }
      },
      {
        "_index" : "accounts-test-001",
        "_type" : "_doc",
        "_id" : "taYbzIQBn__-53X17Y8l",
        "_score" : 0.160443,
        "_source" : {
          "name" : "Taylor Swift",
          "title" : "歌手",
          "description" : "美国乡村民谣流行乐女艺人"
        }
      }
    ]
  }
}
```

如果给定了多个关键字，match查询会认为它们是**逻辑或**的关系。
```bash
# match查询中给定“神枪手”和“女艺人”两个关键字
$ curl -uelastic 'http://192.168.12.143:9200/accounts-test-001/_search?pretty=true' -H 'content-Type:application/json' -d '
{
  "query": {
    "match": {
      "description": "神枪手 女艺人"
    }
  }
}'
Enter host password for user 'elastic':
{
  "took" : 20,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 1.605183,
    "hits" : [
      {
        "_index" : "accounts-test-001",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.605183,
        "_source" : {
          "name" : "Arthur Morgan",
          "title" : "牛仔",
          "description" : "美国东部神枪手"
        }
      },
      {
        "_index" : "accounts-test-001",
        "_type" : "_doc",
        "_id" : "taYbzIQBn__-53X17Y8l",
        "_score" : 1.2199391,
        "_source" : {
          "name" : "Taylor Swift",
          "title" : "歌手",
          "description" : "美国乡村民谣流行乐女艺人"
        }
      }
    ]
  }
}
```

## bool查询
Bool查询支持多个关键字的**逻辑与**检索。
```bash
# bool查询中给定“美国”和“神枪手”两个关键字
$ curl -uelastic 'http://192.168.12.143:9200/accounts-test-001/_search?pretty=true' -H 'content-Type:application/json' -d '
{
  "query": {
    "bool": {
      "must": [
        { "match": {"description": "美国"} },
        { "match": {"description": "神枪手"} }
      ]
    }
  }
}'
Enter host password for user 'elastic':
{
  "took" : 16,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.8162922,
    "hits" : [
      {
        "_index" : "accounts-test-001",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.8162922,
        "_source" : {
          "name" : "Arthur Morgan",
          "title" : "牛仔",
          "description" : "美国东部神枪手"
        }
      }
    ]
  }
}
```


**References**
【1】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/query-dsl-bool-query.html
【2】https://www.ruanyifeng.com/blog/2017/08/elasticsearch.html
【3】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/cat-shards.html
【4】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/indices-create-index.html
【5】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/cluster-health.html
【6】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/indices-analyze.html
【7】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/docs-delete-by-query.html
【8】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/indices-put-mapping.html
【9】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/search-search.html
【10】https://www.elastic.co/guide/en/elasticsearch/reference/7.9/query-dsl-match-query.html






