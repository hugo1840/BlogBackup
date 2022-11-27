---
tags: [ELK]
title: Logstash重要配置参数
created: '2022-11-25T09:13:39.013Z'
modified: '2022-11-27T05:12:36.398Z'
---

Logstash重要配置参数

Logstash有两种类型的配置文件：pipeline配置文件，用于定义Logstash处理管道；以及Logstash本身的配置文件，用于指定控制Logstash启动和运行的参数。

Logstash本身的配置文件位于安装路径的**config**目录下，包括以下几个：
- `logstash.yml`：用于控制Logstash本身的启动和运行。在通过命令行启动Logstash时手动指定的参数值会覆盖此文件中的同名参数值。
- `pipelines.yml`：用于指定在一个Logstash实例中运行多个管道的框架和指令配置。
- `jvm.options`：JVM配置。
- `log4j2.properties`：log4j2库的默认配置。
- `startup.options`：Logstash本身不会读取该配置文件。但是在通过Debian包或者RPM包安装Logstash时，`$LS_HOME/bin/system-install`程序会读取该文件中的配置来为Logstash创建systemd（或者upstart）启动脚本。如果修改了该配置文件，需要重新运行`system-install`来使新的配置生效。


# Logstash配置文件
下面是`$LS_HOME/config/logstash.yml`文件中的一些重要配置项。这里的`$LS_HOME`为Logstash的安装路径。

## 节点配置

1. `node.name`

节点名称，默认为服务器主机名。

2. `path.data`

Logstash及其插件用于数据持久化需求的路径，默认为`$LS_HOME/data`。

## Pipeline主要配置

1. `pipeline.id`

管道的ID。

2. `pipeline.workers`

在管道的Filter和Output阶段中可以并行的worker数量。默认值等于`java.lang.Runtime.getRuntime.availableProcessors`，即主机的CPU核数。可以被`pipelines.yml`中的同名参数值覆盖。

3. `pipeline.batch.size`

单个worker线程在执行Filter和Output之前，从Input收集到的事件（Events）的最大数量。默认大小为**125**。更大的batch size意味着更高的处理效率，但是同时也会增加内存开销。需要同时调整`jvm.options`中的堆内存大小配置。

4. `pipeline.ordered`

管道中的事件排序设置。可取的值有auto、true和false。默认值为**auto**。
- 如果`pipeline.ordered: auto`并且`pipeline.workers: 1`，会自动启用事件排序。
- 如果`pipeline.ordered: true`，表示强制进行排序。如果`pipeline.workers`不等于1，会阻止Logstash启动。
- 如果`pipeline.ordered: false`，则会禁止排序处理，可以节省排序成本。


## Pipeline其他配置

1. `path.config`

主管道的Logstash配置路径。如果指定了目录或通配符，将按字母顺序从该目录读取管道配置文件。例如`/etc/logstash/conf.d/*.conf`。

2. `config.reload.automatic`

当设置为true时，会定期检查配置是否已更改，并在配置发生变化时重新加载配置。也可以通过SIGHUP信号手动触发。默认为**false**。

3. `config.reload.interval`

Logstash检查配置文件更改的频率(秒)。默认为`3s`。

4. `config.support_escapes`

是否支持转义字符。当设置为true时，加引号的字符串将处理转义序列，例如`\n`、`\r`、`\t`、`\\`、`\"`、`\'`。默认为**false**。


## 队列主要配置

1. `queue.type`

用于事件缓冲的内部队列模型。内存队列对应的类型为`memory`，磁盘ACKed持久队列对应的类型为`persisted`。默认为**memory**。

2. `path.queue`

启用持久队列（`queue.type: persisted`）时用于存储数据文件的目录路径。默认路径为`path.data/queue`。

3. `queue.max_events`

当启用持久队列（`queue.type: persisted`）时，队列中未读事件的最大数目。默认值为0，表示无限制。

4. `queue.max_bytes`

队列的总容量（以字节数表示）。默认值为1024mb。请确保磁盘驱动容量大于此处指定的值。如果同时指定了`queue.max_events`和`queue.max_bytes`，Logstash将以两者中最先达到的限制为准。

## 死信队列配置

1. `dead_letter_queue.enable`

是否启用插件支持的死信队列特性。默认为**false**。

2. `dead_letter_queue.max_bytes`

每个死信队列的最大大小。默认值为1024mb。如果新的条目会导致死信队列大小超出此限制，该条目将被删除。

3. `path.dead_letter_queue`

死信队列存储数据文件的路径，默认为`path.data/dead_letter_queue`。

## 其他配置

1. `log.level`

告警日志级别，可选值有fatal、error、warn、info、debug、以及trace。默认级别为**info**。

2. `path.logs`

Logstash日志目录，默认为`LS_HOME/logs`。

3. `path.plugins`

自定义插件的存储路径。可以多次指定此设置以包含多个路径。插件应该存放在一个特定的目录层次结构中，例如`$LS_HOME/plugins/TYPE/NAME`。其中，TYPE是`inputs`、`filters`、`outputs`或`codecs`，NAME是插件的名称。


# Pipeline管道配置

## 管道配置文件示例
Logstash的管道配置文件命名以`.conf`结尾，文件中可以指定Input、Filter（可选）、Output插件的相关配置。

一个示例`logstash-simple.conf`：
```bash
# 输入插件类型为stdin
input { stdin { } }

# 指定了grok和date两个过滤插件
filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
  date {
    match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
}

# 指定了elasticsearch和stdout两个输出插件
output {
  elasticsearch { hosts => ["localhost:9200"] }
  stdout { codec => rubydebug }
}
```

Logstash管道通常包含三个部分：输入→过滤器→输出。输入插件生成事件（Event），过滤器修改事件，输出插件将事件发送到其他地方。

如果指定了多个Filter插件，Logstash会按照它们在管道配置文件中出现的顺序依次应用。Logstash插件的配置中支持多种数据类型，包括数组、列表、布尔值、Codec、Hash、数字、密码、URI、路径、字符串、转义字符、注释（`#`）。

>:eagle: **Logstash管道插件**
>- 常见的Input插件及其用法：https://www.elastic.co/guide/en/logstash/7.9/input-plugins.html
>- 常见的Filter插件及其用法：https://www.elastic.co/guide/en/logstash/7.9/filter-plugins.html
>- 常见的Output插件及其用法：https://www.elastic.co/guide/en/logstash/7.9/output-plugins.html
>- 常见的Codecs插件及其用法：https://www.elastic.co/guide/en/logstash/7.9/codec-plugins.html


在启动Logstash时，通过`-f`选项指定管道配置文件：
```bash
./bin/logstash -f logstash-simple.conf
```

如果我们在终端输入以下内容并回车：
```
127.0.0.1 - - [11/Dec/2013:00:01:45 -0800] "GET /xampp/status.php HTTP/1.1" 200 3891 "http://cadenza/xampp/navi.php" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:25.0) Gecko/20100101 Firefox/25.0"
```

可以在标准输出中看到Logstash pipeline处理后的结果形似如下：
```
{
        "message" => "127.0.0.1 - - [11/Dec/2013:00:01:45 -0800] \"GET /xampp/status.php HTTP/1.1\" 200 3891 \"http://cadenza/xampp/navi.php\" \"Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:25.0) Gecko/20100101 Firefox/25.0\"",
     "@timestamp" => "2013-12-11T08:01:45.000Z",
       "@version" => "1",
           "host" => "cadenza",
       "clientip" => "127.0.0.1",
          "ident" => "-",
           "auth" => "-",
      "timestamp" => "11/Dec/2013:00:01:45 -0800",
           "verb" => "GET",
        "request" => "/xampp/status.php",
    "httpversion" => "1.1",
       "response" => "200",
          "bytes" => "3891",
       "referrer" => "\"http://cadenza/xampp/navi.php\"",
          "agent" => "\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10.9; rv:25.0) Gecko/20100101 Firefox/25.0\""
}
```

可以看到，Logstash能够通过Filter插件解析日志行文本，并将其分解为许多不同的离散数据（属性）。这些解析后得到的离散数据（在存入Elasticsearch后）可以用于高效的日志数据查询和分析。

## 获取事件数据和属性
所有事件（Event）都有属性。例如，apache访问日志可能包含状态代码`(200,404)`、请求路径`("/"，"index.html")`、HTTP谓词`(GET, POST)`、客户端IP地址等。Logstash将这些属性称为字段（**Fields**）。

Logstash中的一些配置选项需要字段才能发挥作用。输入插件负责生成事件，所以在Input中还不存在要计算的字段。由于依赖于事件和字段，下面介绍的配置选项只能在过滤器和输出插件中起作用。

1. **字段引用**

一般通过名称来引用字段（Fields）。访问字段的基本语法是`[fieldname]`。如果引用的是最高层级的字段，可以省略`[]`，只使用字段名。如果要引用一个嵌套字段，需要指定该字段的完整路径：`[顶层字段][嵌套字段]`。

例如，下面的事件有五个顶级字段：`(agent、ip、request、response、ua)`和三个嵌套字段`(status、bytes、os)`。指定`[ua][os]`来引用os字段。要引用request字段，只需指定字段名。

```
{
  "agent": "Mozilla/5.0 (compatible; MSIE 9.0)",
  "ip": "192.168.24.44",
  "request": "/index.html"
  "response": {
    "status": 200,
    "bytes": 52353
  },
  "ua": {
    "os": "Windows 7"
  }
}
```

2. **sprintf格式**

字段引用格式也被Logstash用于sprintf格式。sprintf格式支持在字符串中引用字段值。例如，statsd输出插件有一个increment设置，支持按状态码对apache日志计数:

```
output {
  statsd {
    increment => "apache.%{[response][status]}"
  }
}
```

3. **IF条件语句**

Logstash中的条件语句的使用方式和作用与编程语言中的相同。条件语句支持if、else if和else语句，并且可以嵌套。

```
if EXPRESSION {
  ...
} else if EXPRESSION {
  ...
} else {
  ...
}
```

其中**EXPRESSION**可以是比较表达式或者布尔算子。例如：
- 比较运算符：`==`、`!=`、`<`、`>`、`<=`、`>=`；
- 正则表达式：`=~`、`!~`（根据左侧的字符串值检查右侧的匹配模式）；
- 包含关系：`in`、`not in`；
- 布尔运算：`and`、`or`、`nand`、`xor`；
- 否定运算符：`!`。

示例：
```
output {
  # Send production errors to pagerduty
  if [loglevel] == "ERROR" and [deployment] == "production" {
    pagerduty {
    ...
    }
  }
}
```

4. **@metadata字段**

在Logstash 1.5以及更高版本中，有一个特殊的字段叫做`@metadata`。`@metadata`的内容在输出时不会出现在任何事件中。它非常适合用于条件句，或者使用字段引用和sprintf格式扩展和构建事件字段。

下面的配置文件将利用标准输入来生成事件。输入的内容会变成事件中的message字段。过滤器中的mutate插件会添加向事件添加一些字段，其中一些嵌套在`@metadata`字段中。

```
input { stdin { } }

filter {
  mutate { add_field => { "show" => "This data will be in the output" } }
  mutate { add_field => { "[@metadata][test]" => "Hello" } }
  mutate { add_field => { "[@metadata][no_show]" => "This data will not be in the output" } }
}

output {
  if [@metadata][test] == "Hello" {
    stdout { codec => rubydebug }
  }
}
```

Logstash管道的对应输出为：
```bash
$ bin/logstash -f ../test.conf
Pipeline main started
asdf
{
    "@timestamp" => 2016-06-30T02:42:51.496Z,
      "@version" => "1",
          "host" => "example.com",
          "show" => "This data will be in the output",
       "message" => "asdf"
}
```

输入的asdf类型成为message字段的内容，IF条件语句成功地评估了嵌套在`@metadata`字段中的test字段的内容，但是输出没有显示名为`@metadata`的字段或其内容。

rubydebug编解码器可以显示`@metadata`字段的内容，只需要增加一个配置标志：`metadata => true`。
```
 stdout { codec => rubydebug { metadata => true } }
```

## 使用环境变量

1. **利用环境变量设置TCP端口**

设置环境变量：
```bash
export TCP_PORT=12345
```

在管道配置文件中使用该环境变量：
```
input {
  tcp {
    port => "${TCP_PORT:54321}"
  }
}
```
其中54321为默认值。如果Logstash pipeline没有读取到对应环境变量，就会使用该默认值；如果没有读取到环境变量，也没有设置默认值，Logstash就会抛出一个配置错误。


2. **利用环境变量设置标签值**

设置环境变量：
```bash
export ENV_TAG="tag2"
```

在管道配置文件中使用该环境变量：
```
filter {
  mutate {
    add_tag => [ "tag1", "${ENV_TAG}" ]
  }
}
```

3. **利用环境变量设置文件路径**

设置环境变量：
```bash
export HOME="/path"
```

在管道配置文件中使用该环境变量：
```
filter {
  mutate {
    add_field => {
      "my_path" => "${HOME}/file.log"
    }
  }
}
```


# 多管道配置
如果要在同一个进程中运行多个管道，Logstash提供了一种方法，通过名为`pipelines.yml`的配置文件来实现。该配置文件必须放在`path.settings`路径（`/etc/logstash`或者安装目录下的config目录）下，并遵循以下结构:

```yml
- pipeline.id: my-pipeline_1
  path.config: "/etc/path/to/p1.config"
  pipeline.workers: 3
- pipeline.id: my-other-pipeline
  path.config: "/etc/different/path/p2.cfg"
  queue.type: persisted
```

该配置文件为YAML格式，并包含一个字典列表。其中每个字典描述一个管道的配置，每个键值对为该管道的一个配置项。上面的示例配置了两个不同的管道，分别对应不同的`pipeline.id`和配置文件路径`path.config`。第一个管道的workers被设置为3，而另一个管道启用了持久队列特性。没有在`pipelines.yml`中显式定义的配置参数值，将采用`logstash.yml`中指定的默认值。

如果启动Logstash时没有指定参数，它将读取`pipelines.yml`文件并实例化文件中的所有管道。反之，如果启动Logstash时使用了`-e`或`-f`选项，Logstash会**忽略**`pipelines.yml`文件并在日志中记录下一条相关警告。


**References**
【1】https://www.elastic.co/guide/en/logstash/7.9/config-setting-files.html
【2】https://www.elastic.co/guide/en/logstash/7.9/logstash-settings-file.html
【3】https://www.elastic.co/guide/en/logstash/7.9/multiple-pipelines.html
【4】https://www.elastic.co/guide/en/logstash/7.9/configuration.html
【5】https://www.elastic.co/guide/en/logstash/7.9/running-logstash-command-line.html#command-line-flags
【6】https://www.elastic.co/guide/en/logstash/7.9/configuration-file-structure.html
【7】https://www.elastic.co/guide/en/logstash/7.9/config-examples.html
【8】https://www.elastic.co/guide/en/logstash/7.9/plugins-inputs-beats.html
【9】https://www.elastic.co/guide/en/logstash/7.9/plugins-outputs-elasticsearch.html




