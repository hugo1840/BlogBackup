@[TOC](Pod调度策略：亲和性、污点与污点容忍)

# 节点亲和性
## nodeSelector
nodeSelector用于将Pod调度到匹配Label的节点上，如果没有匹配的标签会调度失败。

作用：
- 约束Pod到特定节点运行；
- 完全匹配节点标签。

应用场景：
- 专用节点：根据业务线将节点分组管理；
- 匹配特殊硬件：部分节点配置有SSD硬盘、GPU。
---
**示例：确保Pod分配到具有SSD硬盘的节点上**
1. 给节点打标签
```bash
#kubectl label nodes <node名称> <label-key>=<label-value>
kubectl label nodes k8s-node1 disktype=ssd
#验证
kubectl get nodes --show-labels
```

2. 添加nodeSelector字段到Pod配置中
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

3. 验证
```bash
kubectl get pods -o wide
```

删除节点标签：
```bash
kubectl label node k8s-node1 <label-key>-
#验证
kubectl get pods -o wide
```


## nodeAffinity
节点亲和性，类似于nodeSelector，可以根据节点上的标签来约束Pod可以调度到哪些节点。

相比nodeSelector：
- 匹配有更多的逻辑组合，不只是字符串的完全相等，支持`in`、`notin`、`exists`、`doesnotexist`、`gt`、`lt`操作符。
- 调度分为软策略和硬策略，而不是硬性要求：
  - `requiredDuringSchedulingIgnoredDuringExecution`：硬策略，必须满足；
  - `preferredDuringSchedulingIgnoredDuringExecution`：软策略，尝试满足，但是不保证。

---
**举例：**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```

**注意**：
- 如果同时使用了`nodeSelector`和`nodeAffinity`，那么k8s在调度Pod时必须同时满足两者的条件。
- 如果在使用`nodeAffinity`时指定了多个`nodeSelectorTerms`，那么节点只要满足其中一个`nodeSelectorTerms`，就可以被调度分配Pod。
- 如果在一个`nodeSelectorTerms`下指定了多个`matchExpressions`，那么节点必须满足所有`matchExpressions`，才能被调度分配Pod。


# 污点与污点容忍
基于节点标签分配是站在Pod的角度上，通过在Pod上添加属性，来确定Pod是否要调度到指定的节点上。相反地，我们也可以在Node节点上添加污点属性（Taints），来避免Pod被分配到不合适的节点上。

- `taints`：避免Pod调度到特定节点上；
- `tolerations`：允许Pod调度到有Taints的节点上。

---
**举例：**
1. 给节点添加污点
```bash
#kubectl taint node <node名称> key=value:[effect]
kubectl taint node k8s-node2 gpu=yes:NoSchedule
```

其中，effect可以取值为：
- `NoSchedule`：一定不能被调度；
- `PreferNoSchedule`：尽量不要被调度，非必须；
- `NoExecute`：不仅不会调度，还会驱逐节点上已有的Pod。

2. 验证
```bash
kubectl describe node k8s-node2 | grep Taint
```

3. 如果希望Pod可以被分配到有污点的节点上，要在Pod配置中添加污点容忍。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "gpu"
    operator: "Equal"
	value: "yes"
    effect: "NoSchedule"
```

删除污点：
```bash
#kubectl taint node <node名称> key:[effect]-
kubectl taint node k8s-node2 gpu:NoSchedule-
```

**注意**：
Taints和Tolerations匹配的原则是`key`相同、`effect`相同，并且满足：
- 运算符是`Exists`（即没有`value`）；
- 运算符是`Equal`，且`value`相等。

两个特例：
- 一个空的`key`和运算符`Exists`，会匹配所有的`key`、`value`和`effect`，意味著容忍所有污点；
- 一个`key`和一个空的`effect`匹配此`key`的所有`effect`。


# nodeName
通过nodeName指定节点名称，可以将Pod调度到指定节点上，**不经过调度器**。由于不经过调度器，使用nodeName后，nodeSelector、nodeAffinity和Taints都**不会**起作用。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: k8s-node2
```

如果指定的节点不存在、或者没有足够的资源，那么该Pod不会运行。如果Pod所在的节点挂了，也不会自动漂移到其他节点。


**References**
【1】https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
【2】https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

