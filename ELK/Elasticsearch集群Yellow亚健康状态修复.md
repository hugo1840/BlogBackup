---
tags: [ELK]
title: Elasticsearch集群Yellow亚健康状态修复
created: '2023-02-23T08:55:36.697Z'
modified: '2023-02-23T12:03:34.464Z'
---

Elasticsearch集群Yellow亚健康状态修复

# 问题背景
Elasticsearch集群健康状态为**Yellow**，涉及到多个索引。

# 排查流程
在浏览器打开Kibana Console进行问题排查，console地址为：
```bash
http://{Kibana_IP}:5601/app/dev_tools#/console
```

在console运行以下API命令来获取基本信息：
```bash
GET _cat/health?v
GET _cat/master?v
GET _cat/nodes?v
GET _cat/indices?v

GET _cat/shards?v
# 输出中各列分别为：
# shard：分片名称；prirep：主分片或副本，
# state：分片状态，可以为 INITIALIZING | RELOCATING | STARTED | UNASSIGNED
# docs：分片中文档的数量；store：分片占用的磁盘空间

GET _cat/allocation?v
# 获取分配到每个节点的分片数量以及所占用的磁盘空间
```

获取健康状态为Yellow的索引信息：
```bash
GET _cat/indices?v&health=yellow
```
输出中包含的列有health、status（索引状态）、index（索引名称）、uuid、pri（主分片数量）、rep（副本数量）、docs.count、docs.deleted、store.size、pro.store.size。

从上面拿到的异常状态索引中，任选一个（假设为`ftimes_infra_migrad_2022-09`）继续查看该索引的分片信息：
```bash
GET _cat/shards/ftimes_infra_migrad_2022-09?v
```
输出的列中包含index、shard（分片名称）、prirep（primary还是replica）、state、docs、store（分片大小）、ip、node（分片所在节点）。

观察目标索引的各个分片的分配情况。Yellow健康状态下一般这里可以看到有replica分片没有被正确分配，即`prirep=r`的行记录，对应的分片状态为`state=UNASSIGNED`。

假设未被正确分配的replica分片名称为`0`，检查该分片分配失败的原因：
```bash
GET _cluster/allocation/explain
{
  "index": "ftimes_infra_migrad_2022-09",
  "shard": 0,
  "primary": false
}
```

检查输出中的`explanation`部分：
```bash
...
"explanation": "shard has exceeded the maximum number of retries [5] on failed
allocation attempts - manually call [/_cluster/reroute?retry_failed=true] to retry,
..."
```

# 解决办法
下面我们尝试手动分配该replica分片。需要确保replica分片要分配的节点上有足够的磁盘空间，并且同一索引的primary分片和replica分片不在同一节点上。
```bash
# 查看分片的大小、主分片所在节点
GET _cat/shards/ftimes_infra_migrad_2022-09?v

# 查看各节点的磁盘空间使用情况
GET _cat/allocation?v

# 将replica分片手动分配到指定节点es_data_21
POST /_cluster/reroute
{
  "command": [
    {
      "allocation_replica": {
        "index": "ftimes_infra_migrad_2022-09",
        "shard": 0,
        "node": "es_data_21"
      }
    }
  ]
}
```

执行后收到下面的报错：
```bash
...
"type": "illegal_argument_exception",
"reason": "[allocation_replica] allocation of [ftimes_infra_migrad_2022-09][0] on
node {es_data_21}{...}{...} is not allowed, reason: [NO(shard has exceeded the 
maximum number of retries [5] on failed allocation attempts - manually call 
[/_cluster/reroute?retry_failed=true] to retry, ... )]"
```

根据错误提示执行以下命令：
```bash
POST /_cluster/reroute?retry_failed=true
```
ES集群就会自动重新分配之前分配出错的replica副本。

过一小段时间后，检查所有索引健康状态：
```bash
GET _cat/indices?v&health=yellow
```


**:fish:MORE ...**

在Kibana的console API命令中，可以使用`s`来对检索结果按指定的列排序，并使用通配符`*`来匹配任意字符串。
```bash
# 获取集群中所有索引信息，并按index列排序
GET _cat/indices?v&s=index

# 获取集群中名称以ftimes开头的所有索引信息，并按index列排序
GET _cat/indices/ftimes*?v&s=index

# 获取集群中名称以gzone开头的索引的所有分片信息
GET _cat/shards/gzone*
```

