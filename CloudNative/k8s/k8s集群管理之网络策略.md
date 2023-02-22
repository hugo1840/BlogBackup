---
tags: [k8s]
title: k8s集群管理之网络策略
created: '2023-02-11T12:17:46.234Z'
modified: '2023-02-22T13:31:06.853Z'
---

k8s集群管理之网络策略

如果你希望在IP地址或端口层面（OSI第3层或第4层）控制网络流量，则可以考虑为集群中特定应用使用Kubernetes网络策略（**NetworkPolicy**）。NetworkPolicy是一种以应用为中心的结构，允许设置如何允许Pod与网络上的各类网络“实体”通信。

可以与Pod通信的网络实体是通过如下三个标识符来辩识的：

1. 其他被允许的Pods（例外，Pod无法阻塞对自身的访问）；
2. 被允许的名字空间；
3. IP组块（例外：与Pod运行所在的节点的通信总是被允许的，无论Pod或节点的IP地址）。

在定义基于Pod或名字空间的NetworkPolicy时，可以使用选择器（Selector）来设定哪些流量可以进入或离开与该选择器匹配的Pod。另外，当创建基于IP的NetworkPolicy时，我们基于IP组块（CIDR范围）来定义策略。

# 前置条件
网络策略通过**网络插件**来实现。要使用网络策略，必须使用支持NetworkPolicy的网络解决方案。创建一个NetworkPolicy资源对象而没有控制器来使它生效的话，是没有任何作用的。

# Pod隔离类型
Pod有两种隔离: **出口隔离**和**入口隔离**。它们涉及到可以建立哪些连接。这里的“隔离”不是绝对的，而是意味着“有一些限制”。另外，“非隔离方向”意味着在所述方向上没有限制。这两种隔离（或不隔离）是独立声明的，并且都与从一个Pod到另一个Pod的连接有关。

默认情况下，一个Pod的出口是非隔离的，即所有外向连接都是被允许的。如果有任何的NetworkPolicy选择该Pod并在其`policyTypes`中包含**Egress**，则该Pod是出口隔离的。当一个Pod的出口被隔离时，唯一允许的来自Pod的连接是适用于出口的Pod的某个NetworkPolicy的`egress`列表所允许的连接。这些egress列表的效果是相加的。

默认情况下，一个Pod对入口是非隔离的，即所有入站连接都是被允许的。如果有任何的NetworkPolicy选择该Pod并在其`policyTypes`中包含**Ingress**，则该Pod被隔离入口。当一个Pod的入口被隔离时，唯一允许进入该Pod的连接是来自该Pod节点的连接、以及适用于入口的Pod的某个NetworkPolicy的`ingress`列表所允许的连接。这些ingress列表的效果是相加的。

网络策略是相加的，所以不会产生冲突。如果策略适用于Pod某一特定方向的流量，Pod在对应方向所允许的连接是适用的网络策略所允许的集合。因此，评估的顺序不影响策略的结果。

要允许从源Pod到目的Pod的连接，源Pod的出口策略和目的Pod的入口策略都需要允许连接。如果任何一方不允许连接，建立连接将会失败。

# NetworkPolicy资源
下面是一个NetworkPolicy的示例:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - ipBlock:
            cidr: 172.17.0.0/16
            except:
              - 172.17.1.0/24
        - namespaceSelector:
            matchLabels:
              project: myproject
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 6379
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
      ports:
        - protocol: TCP
          port: 5978
```

>**注**：除非选择支持网络策略的网络解决方案，否则将上述示例发送到API服务器没有任何效果。

- 必需字段：与所有其他的Kubernetes配置一样，NetworkPolicy需要`apiVersion`、`kind`和`metadata`字段。
- **spec**：`spec`字段中包含了在一个名字空间中定义特定网络策略所需的所有信息。
- **podSelector**：每个NetworkPolicy都包括一个`podSelector`，它对该策略所适用的一组Pod进行选择。示例中的策略选择带有`"role=db"`标签的Pod。**空**的podSelector选择名字空间下的**所有**Pod。
- **policyTypes**：每个NetworkPolicy都包含一个`policyTypes`列表，其中包含Ingress或Egress，或两者兼具。policyTypes字段表示给定的策略是应用于进入所选Pod的入站流量还是来自所选Pod的出站流量，或两者兼有。如果NetworkPolicy未指定policyTypes，则**默认情况下始终设置Ingress**。
- **ingress**：每个NetworkPolicy可包含一个`ingress`规则的白名单列表。每个规则都允许同时匹配`from`和`ports`部分的流量。示例策略中包含一条简单的规则：它匹配三个进站流量来源的某个特定端口，第一个通过`ipBlock`指定，第二个通过`namespaceSelector`指定，第三个通过`podSelector`指定。
- **egress**：每个NetworkPolicy可包含一个`egress`规则的白名单列表。每个规则都允许匹配`to`和`port`部分的流量。该示例策略包含一条规则，该规则将指定端口上的流量匹配到`10.0.0.0/24`中的任何目的地。

所以，该网络策略示例：
- 隔离default名字空间下`role=db`的Pod。

- Ingress规则允许以下Pod连接到default名字空间下的带有`role=db`标签的所有Pod的6379 TCP端口：
  - default名字空间下带有`role=frontend`标签的所有Pod；
  - 带有`project=myproject`标签的所有名字空间中的Pod；
  - IP地址范围为`172.17.0.0–172.17.0.255`和`172.17.2.0–172.17.255.255`（即，除了`172.17.1.0/24`之外的所有`172.17.0.0/16`）。

- Egress规则允许default名字空间中任何带有标签`role=db`的Pod的5978 TCP端口到CIDR `10.0.0.0/24`的连接。


# 选择器TO和FROM
可以在ingress的`from`部分或egress的`to`部分中指定四种选择器：

1. **podSelector**

此选择器将在与NetworkPolicy**相同的名字空间**中选择特定的Pod，应将其允许作为入站流量来源或出站流量目的地。

2. **namespaceSelector**

此选择器将选择特定的名字空间，将满足条件的名字空间中**所有Pod**用作其入站流量来源或出站流量目的地。

3. **namespaceSelector**和**podSelector**

一个同时指定`namespaceSelector`和`podSelector`的`to/from`条目选择特定名字空间中的特定Pod。

注意使用正确的YAML语法。下面的策略：
```yaml
...
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          user: alice
      podSelector:
        matchLabels:
          role: client
  ...
```
此策略在`from`数组中仅包含一个元素，只允许来自标有`role=client`的Pod**且**该Pod所在的名字空间中标有`user=alice`的连接。

但是下面这项策略：
```yaml
 ...
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          user: alice
    - podSelector:
        matchLabels:
          role: client
  ...
```
它在`from`数组中包含两个元素，允许来自本地名字空间中标有`role=client`的Pod的连接，**或**来自任何名字空间中标有`user=alice`的任何Pod的连接。

4. **ipBlock**

此选择器将选择特定的IP CIDR范围以用作入站流量来源或出站流量目的地。这些应该是**集群外部IP**，因为Pod IP存在时间短暂的且随机产生。

集群的入站和出站机制通常需要重写数据包的源IP或目标IP。在发生这种情况时，不确定在NetworkPolicy处理之前还是之后发生，并且对于网络插件、云提供商、Service实现等的不同组合，其行为可能会有所不同。

对入站流量而言，这意味着在某些情况下，可以根据实际的原始源IP过滤传入的数据包，而在其他情况下，NetworkPolicy所作用的源IP则可能是LoadBalancer或Pod的节点等。对于出站流量而言，这意味着从Pod到被重写为集群外部IP的Service IP的连接可能会或可能不会受到基于ipBlock的策略的约束。

# 默认策略
默认情况下，如果名字空间中不存在任何策略，则所有进出该名字空间中Pod的流量都被允许。以下示例可以更改该名字空间中的默认行为。

**示例1**：默认拒绝所有入站流量

可以通过创建选择所有容器但不允许任何进入这些容器的入站流量的NetworkPolicy来为名字空间创建default隔离策略。
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}    # 空的podSelector选择名字空间下的所有Pod
  policyTypes:
  - Ingress
```

**示例2**：允许所有入站流量

如果你想允许一个名字空间中所有Pod的所有入站连接，你可以创建一个明确允许的策略。
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
spec:
  podSelector: {}
  ingress:
  - {}        # 空的ingress清单表示允许所有入站流量
  policyTypes:
  - Ingress
```

**示例3**：默认拒绝所有出站流量

你可以通过创建选择所有容器但不允许来自这些容器的任何出站流量的NetworkPolicy来为名字空间创建 default隔离策略。
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

**示例4**：允许所有出站流量

如果要允许来自名字空间中所有Pod的所有连接，则可以创建一个明确允许来自该名字空间中Pod的所有出站连接的策略。
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
spec:
  podSelector: {}
  egress:
  - {}
  policyTypes:
  - Egress
```

**示例5**：默认拒绝所有入站和出站流量

你可以为名字空间创建默认策略，以通过在该名字空间中创建以下NetworkPolicy来阻止所有入站和出站流量。
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

# 针对某个端口范围
在编写NetworkPolicy时，你可以针对一个端口范围而不是某个固定端口。这一目的可以通过使用`endPort`字段来实现，如下例所示：
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: multi-port-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
      ports:
        - protocol: TCP
          port: 32000
          endPort: 32768
```
上面的规则允许名字空间default中所有带有标签`role=db`的Pod使用TCP协议与10.0.0.0/24范围内的IP通信，只要目标端口介于32000和32768之间就可以。

使用此字段时存在以下限制：
- endPort字段必须等于或者大于port字段的值。
- 只有在定义了port时才能定义endPort。
- 两个字段的设置值都只能是数字。

>**注**：你的集群所使用的CNI插件必须支持在NetworkPolicy规约中使用endPort字段。如果你的网络插件不支持endPort字段，而你指定了一个包含endPort字段的NetworkPolicy，策略只对单个port字段生效。


# 网络策略尚未支持的功能
到Kubernetes 1.26为止，NetworkPolicy API还不支持以下功能，不过你可能可以使用操作系统组件（如SELinux、OpenVSwitch、IPTables等等）或者第七层技术（Ingress控制器、服务网格实现）或准入控制器来实现一些替代方案。

- 强制集群内部流量经过某公用网关（这种场景最好通过服务网格或其他代理来实现）；
- 与TLS相关的场景（考虑使用服务网格或者Ingress控制器）；
- 特定于节点的策略（你可以使用CIDR来表达这一需求不过你无法使用节点在Kubernetes中的其他标识信息来辩识目标节点）；
- 基于名字来选择服务（不过，你可以使用Label来选择目标Pod或名字空间，这也通常是一种可靠的替代方案）；
- 创建或管理由第三方来实际完成的“策略请求”；
- 实现适用于所有名字空间或Pods的默认策略（某些第三方Kubernetes发行版本或项目可以做到这点）；
- 高级的策略查询或者可达性相关工具；
- 生成网络安全事件日志的能力（例如，被阻塞或接收的连接请求）；
- 显式地拒绝策略的能力（目前，NetworkPolicy的模型默认采用拒绝操作，其唯一的能力是添加允许策略）；
- 禁止本地回路或指向宿主的网络流量（Pod目前无法阻塞localhost访问，它们也无法禁止来自所在节点的访问请求）。



**References**
【1】https://kubernetes.io/zh-cn/docs/concepts/services-networking/network-policies/


