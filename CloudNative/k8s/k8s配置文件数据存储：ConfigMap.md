---
tags: [k8s]
title: k8s配置文件数据存储：ConfigMap
created: '2022-11-03T03:41:59.575Z'
modified: '2022-11-03T11:04:30.807Z'
---

k8s配置文件数据存储：ConfigMap

ConfigMap是一种可以将非机密性的数据保存到键值对中的API对象。创建ConfigMap后，数据实际上会存储在etcd中，在创建Pod时会引用该数据。Pod可以将ConfigMap作为环境变量、命令行参数、或者存储卷中的配置文件来使用。

ConfigMap可用于将环境配置信息和容器镜像解耦，便于应用配置的修改。

# ConfigMap使用场景
使用ConfigMap可以将应用的配置数据和应用程序代码分开。

ConfigMap在设计上不是用来保存大量数据的。在ConfigMap中保存的数据**不可超过1MiB**。如果需要保存超出此尺寸限制的数据，可以考虑挂载存储卷或者使用独立的数据库或者文件服务。

# ConfigMap对象
和其他Kubernetes对象都有一个`spec`字段不同，ConfigMap使用`data`和`binaryData`字段。这些字段能够接收键值对作为其取值。data和binaryData字段都是可选的。data字段设计用来保存`UTF-8`字符串，而binaryData则被设计用来保存二进制数据为`base64`编码的字符串。

ConfigMap的名字必须是一个合法的DNS子域名，即：
- 不能超过253个字符；
- 只能包含小写字母、数字、`-`或者`.`；
- 以字母或数字开头，并且以字母或数字结尾。

data或binaryData字段下面每个键的名称都必须由字母数字、`-`、`_`或`.`组成。在data下保存的键名不可以与在binaryData下出现的键名有重叠。

# ConfigMap使用方法
可以事先定义一个包含键值对数据的ConfigMap，并在目标Pod中的`spec`字段中引用ConfigMap中的数据来配置容器。该Pod和ConfigMap必须要在**同一个namespace**中。

>:eagle: **静态Pod**中的spec字段**不**能引用ConfigMap或任何其他API对象。

可以使用四种方式来使用ConfigMap配置Pod中的容器：
- 在容器命令和参数内；
- 容器的环境变量；
- 在只读卷里面添加一个文件，让应用来读取；
- 编写代码在Pod中运行，使用Kubernetes API来读取ConfigMap。

这些不同的方法适用于不同的数据使用方式。对前三种方法，kubelet使用ConfigMap中的数据在Pod中启动容器。

第四种方法意味着我们必须编写代码才能读取ConfigMap和它的数据。然而，由于是直接使用Kubernetes API，因此只要ConfigMap发生更改，应用就能够通过订阅来获取更新，并且做出响应。通过直接进入Kubernetes API，这个技术也可以让我们能够获取到**不同的namespace里**的ConfigMap。

## 定义ConfigMap
下面的文件`configmap-demo.yaml`定义了一个ConfigMap，其中包含了两种键值对数据：类属性键和类文件键。
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # 类文件键，|表示有多行数据
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true 
```

创建ConfigMap并检查：
```bash
[root@k8s-master ~]# kubectl apply -f configmap-demo.yaml
configmap/game-demo created
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get configmap
NAME               DATA   AGE
game-demo          4      7s
kube-root-ca.crt   1      103d
[root@k8s-master ~]#
```


## 在Pod中引用ConfigMap
下面的文件`configmap-demo-pod.yaml`定义了一个Pod，其中spec字段中通过容器环境变量和挂载configMap只读卷两种方式，引用了上面定义的ConfigMap中存储的键值对数据。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # 通过容器环境变量来引用ConfigMap中的数据
        - name: PLAYER_INITIAL_LIVES     # 自定义的环境变量名称
          valueFrom:
            configMapKeyRef:
              name: game-demo            # 要获取数据来源的ConfigMap名称
              key: player_initial_lives  # 需要从ConfigMap中取值的键
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"   # ConfigMap中的数据在容器中的挂载路径
        readOnly: true         # 必须为只读卷
  volumes:
    # 通过挂载类型为configMap的只读卷来引用ConfigMap中的数据
    - name: config
      configMap:
        name: game-demo     # 要获取数据来源的ConfigMap名称
        # 来自ConfigMap的一组键，将被创建为文件
        items:
        - key: "game.properties"     # 需要从ConfigMap中取值的键
          path: "game.properties"    # 在volumeMounts.mountPath下挂载的路径名
        - key: "user-interface.properties"
          path: "user-interface.properties"
```

创建上面示例中的Pod，并检查配置数据：
```bash
[root@k8s-master ~]# kubectl apply -f configmap-demo-pod.yaml
pod/configmap-demo-pod created
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl exec -it configmap-demo-pod -- sh -c 'echo $PLAYER_INITIAL_LIVES'
3
[root@k8s-master ~]# kubectl exec -it configmap-demo-pod -- sh -c 'echo $UI_PROPERTIES_FILE_NAME'
user-interface.properties
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl exec -it configmap-demo-pod -- sh -c 'ls /config'
game.properties            user-interface.properties
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl exec -it configmap-demo-pod -- sh -c 'cat /config/game.properties'
enemy.types=aliens,monsters
player.maximum-lives=5
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl exec -it configmap-demo-pod -- sh -c 'cat /config/user-interface.properties'
color.good=purple
color.bad=yellow
allow.textmode=true
```

上面的例子定义了一个卷并将它作为`/config`文件夹挂载到demo容器内，然后创建了两个文件`/config/game.properties`和`/config/user-interface.properties`，尽管ConfigMap中包含了四个键。这是因为Pod定义中在volumes字段下指定了一个`items`数组。如果我们不指定items数组，则ConfigMap中的每个键都会变成一个与该键同名的文件，因此会得到四个文件。

>:coffee: 多个Pod可以引用同一个ConfigMap。如果Pod中有多个容器，则每个容器都需要自己的`volumeMounts`块，但针对每个ConfigMap，只需要设置一个`spec.volumes`块。

# 被挂载的ConfigMap会自动更新
当卷中使用的ConfigMap被更新时，所投射的键最终也会被更新。kubelet组件会在每次周期性同步时检查所挂载的ConfigMap是否为最新。不过，kubelet使用的是其本地的高速缓存（local cache）来获得ConfigMap的当前值。高速缓存的类型可以通过KubeletConfiguration结构的`ConfigMapAndSecretChangeDetectionStrategy`字段来配置。

ConfigMap既可以通过**watch**操作实现内容传播（默认形式），也可以通过基于TTL的方法，还可以直接将所有请求重定向到API服务器。因此，从ConfigMap被更新的那一刻算起，到新的键值被投射到Pod中去，这一时间跨度可能与**kubelet的同步周期**加上**高速缓存的传播延迟**相等。这里的传播延迟取决于所选的高速缓存类型（分别对应watch操作的传播延迟、高速缓存的TTL时长或者0）。

>:snake: 以**环境变量**方式使用的ConfigMap数据**不会被自动更新**。更新这些数据需要重新启动Pod。


# 不可变更的ConfigMap
Kubernetes还提供了一种将ConfigMap设置为不可变更的特性（也支持Secret）。对于大量使用ConfigMap的集群（至少有数万个各不相同的ConfigMap给Pod挂载）而言，禁止更改ConfigMap的数据有以下好处：

- 保护应用，使之免受意外（不需要的）更新所带来的负面影响；
- 大幅降低对kube-apiserver的压力，从而提升集群性能。这是因为系统会关闭对已标记为不可变更的ConfigMap的监控。

我们可以通过将`immutable`字段设置为true创建不可变更的ConfigMap。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  ...
data:
  ...
immutable: true
```

要注意的是，一旦某ConfigMap被标记为不可变更，则**无法逆转**这一变化，也无法更改data或binaryData字段的内容。只能删除并重建ConfigMap。因为现有的Pod会维护一个已被删除的ConfigMap的挂载点，建议重新创建这些Pods。


**References**
【1】https://kubernetes.io/docs/concepts/configuration/configmap/
【2】https://kubernetes.io/docs/concepts/configuration/secret/
【3】https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-subdomain-names



