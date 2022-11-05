---
tags: [k8s]
title: k8s敏感数据存储：Secret
created: '2022-11-03T08:31:22.173Z'
modified: '2022-11-05T06:56:23.125Z'
---

k8s敏感数据存储：Secret

# Secret是什么
Secret与ConfigMap类似，区别在于Secret用于存储敏感数据，例如密码、令牌或密钥对象。所有的数据都要经过base64编码。

>:warning:**注意**
>默认情况下，Kubernetes Secret未加密地存储在API服务器的底层数据存储（etcd）中。任何拥有API访问权限的人都可以检索或修改Secret，任何有权访问etcd的人也可以。此外，任何有权限在命名空间中创建Pod的人都可以读取该命名空间中的任何Secret，包括间接访问，例如创建Deployment。
>
>为了安全地使用Secret，请至少执行以下步骤：
>- 为Secret启用静态加密；
>- 以最小特权访问Secret并启用或配置RBAC规则；
>- 限制Secret对特定容器的访问；
>- 考虑使用外部Secret存储驱动。
>
>关于提升Secret安全性，请参考：https://kubernetes.io/zh-cn/docs/concepts/security/secrets-good-practices/


命令`kubectl create secret`支持三种Secret数据类型：
- `docker registry`：存储镜像仓库认证信息；
- `generic`：存储用户名、密码；
- `tls`：存储证书，例如从文件、目录或字符串创建。

```bash
[root@k8s-master ~]# kubectl create secret --help
Create a secret using specified subcommand.

Available Commands:
  docker-registry Create a secret for use with a Docker registry
  generic         Create a secret from a local file, directory, or literal value
  tls             Create a TLS secret

Usage:
  kubectl create secret [flags] [options]
```

# Secret的类型
创建Secret时，可以使用Secret资源的type字段，或者与其等价的kubectl命令行参数（如果有的话）为其设置类型。 Secret类型有助于对Secret数据进行编程处理。

Kubernetes提供若干种内置的类型，用于一些常见的使用场景。 

| 内置类型 | 用法 |
| :--: | :--: |
| Opaque | 用户定义的任意数据 |
| kubernetes.io/service-account-token | 服务账号令牌 |
| kubernetes.io/dockercfg | ~/.dockercfg文件的序列化形式 |
| kubernetes.io/dockerconfigjson | ~/.docker/config.json文件的序列化形式 |
| kubernetes.io/basic-auth | 用于基本身份认证的凭据 |
| kubernetes.io/ssh-auth | 用于SSH身份认证的凭据 |
| kubernetes.io/tls | 用于TLS客户端或者服务端的数据 |
| bootstrap.kubernetes.io/token | 启动引导令牌数据 |

当Secret配置文件中未作显式设定时，**默认**的Secret类型是Opaque。使用`generic`子命令来标明要创建的是一个Opaque类型Secret。
```bash
kubectl create secret generic <Secret名称>
```

# Secret使用方法

Pod可以通过以下三种方式来使用Secret：
- 作为挂载到一个或多个容器中的存储卷中的文件；
- 作为容器的环境变量；
- 由kubelet在拉取容器镜像时使用。

## 创建Secret
### 创建Secret的一些限制

1. Secret对象的名称必须是合法的DNS子域名：长度不能超过253个字符，只能包含小写字母、数字、`-`或者`.`，必须以字母或数字开头，并且以字母或数字结尾。

2. 在为创建Secret编写配置文件时，可以设置`data`与（或）`stringData`字段。data和stringData字段都是可选的。**data字段中所有键值都必须是`base64`编码的字符串**。stringData字段中可以使用任何字符串作为其取值。

3. data和stringData中的键名只能包含字母、数字、`-`、`_`或`.`字符。stringData字段中的所有键值对都会在内部被合并到data字段中。如果某个主键同时出现在data和stringData字段中，**stringData所指定的键值具有更高的优先级**。

4. 每个Secret的大小最多为**1MiB**。这一限制是为了避免用户创建非常大的Secret，进而导致API 服务器和kubelet内存耗尽。不过创建很多小的Secret也可能耗尽内存。可以使用**资源配额**来约束每个名字空间中Secret的个数。

可以通过以下三种方法来创建Secret：

### 使用kubectl命令创建Secret

1. 创建Secret

```bash
# 方法一
[root@k8s-master ~]# echo -n 'admin' > ./username.txt
[root@k8s-master ~]# echo -n '1f2d1e2e67df' > ./password.txt
[root@k8s-master ~]# kubectl create secret generic db-user-pass \
> --from-file=username=./username.txt --from-file=password=./password.txt
secret/db-user-pass created

# 方法二
[root@k8s-master ~]# kubectl create secret generic db-user-pass \
> --from-literal=username=devuser --from-literal=password='S!B\*d$zDsb='
secret/db-user-pass created
```

方法一通过`--from-file`参数指定要读取Secret数据的文件，将文件名作为键值对的Key，并将文件内容作为键值对的Value。可以使用`--from-file=[key=]source`来设置密钥的键名称。我们不需要对文件中包含的密码字符串中的特殊字符进行转义。

echo命令后的标志`-n`确保生成的文件在文本末尾不包含额外的换行符。因为当kubectl读取文件并将内容编码为base64字符串时，多余的换行符也会被编码。`kubectl create secret`命令将这些文件打包成一个Secret并在API服务器上创建对象。

方法二使用`--from-literal=<key>=<value>`标签提供Secret数据。可以多次使用此标签，提供多个键值对。特殊字符（例如`$`，`\`，`*`，`=`和`!`）由shell解释执行，而且需要转义。在大多数shell中，转义密码最简便的方法是用单引号括起来。

2. 验证Secret

检查Secret是否已被成功创建：
```bash
[root@k8s-master ~]# kubectl get secrets
NAME                                 TYPE                                  DATA   AGE
db-user-pass                         Opaque                                2      13s
default-token-jwvc6                  kubernetes.io/service-account-token   3      103d
my-ingress-playroom                  kubernetes.io/tls                     2      87d
nfs-client-provisioner-token-dg4kp   kubernetes.io/service-account-token   3      82d
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl describe secrets/db-user-pass
Name:         db-user-pass
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password.txt:  12 bytes
username.txt:  5 bytes
```
kubectl get和kubectl describe命令默认不显示Secret的内容。这是为了防止Secret被意外暴露或存储在终端日志中。

3. 解码Secret

```bash
[root@k8s-master ~]# kubectl get secret db-user-pass -o jsonpath='{.data}'
{"password":"MWYyZDFlMmU2N2Rm","username":"YWRtaW4="}[root@k8s-master ~]#

# 为了避免在shell历史记录中存储Secret的编码值，建议执行如下命令
[root@k8s-master ~]# kubectl get secret db-user-pass -o jsonpath='{.data.username}' | base64 --decode
admin[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get secret db-user-pass -o jsonpath='{.data.password}' | base64 --decode
1f2d1e2e67df[root@k8s-master ~]#
```

4. 删除Secret

```bash
[root@k8s-master ~]# kubectl delete secret db-user-pass
secret "db-user-pass" deleted
```

### 基于配置文件来创建Secret
可以先用JSON或YAML格式在一个清单文件中定义Secret对象，然后创建该对象。Secret资源包含2个键值对：data和stringData。data字段用来存储base64编码的任意数据。stringData允许Secret使用未编码的字符串。

>:bear: Secret数据的JSON和YAML序列化结果是以base64编码的。换行符在这些字符串中无效，必须省略。Linux用户应该在base64命令中添加`-w 0`选项，或者在`-w`选项不可用的情况下，输入`base64 | tr -d '\n'`。

将目标字符串转换为base64编码：
```bash
[root@k8s-master ~]# echo -n 'admin' | base64
YWRtaW4=
[root@k8s-master ~]#
[root@k8s-master ~]# echo -n '1f2d1e2e67df' | base64
MWYyZDFlMmU2N2Rm
```

编写yaml文件并创建Secret对象：
```bash
[root@k8s-master ~]# cat mysecret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl apply -f mysecret.yaml
secret/mysecret created
```

使用stringData字段可以将一个非base64编码的字符串直接放入Secret中，当创建或更新该Secret时，此字段将被编码。

```bash
[root@k8s-master ~]# cat mysecret-stringdata.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret-stringdata
type: Opaque
stringData:
  config.yaml: |
    apiUrl: "https://my.api.com/api/v1"
    username: devsuer
    password: Nsjhudt75&
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl apply -f mysecret-stringdata.yaml
secret/mysecret-stringdata created
```

检查Secret对象可以发现，stringData中的键值对数据实际上被编码后存储到了data字段中。
```bash
[root@k8s-master ~]# kubectl get secret mysecret-stringdata -o yaml
apiVersion: v1
data:
  config.yaml: YXBpVXJsOiAiaHR0cHM6Ly9teS5hcGkuY29tL2FwaS92MSIKdXNlcm5hbWU6IGRldnN1ZXIKcGFzc3dvcmQ6IE5zamh1ZHQ3NSYgCg==
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Secret","metadata":{"annotations":{},"name":"mysecret-stringdata","namespace":"default"},"stringData":{"config.yaml":"apiUrl: \"https://my.api.com/api/v1\"\nusername: devsuer\npassword: Nsjhudt75\u0026 \n"},"type":"Opaque"}
  creationTimestamp: "2022-11-04T02:47:07Z"
  name: mysecret-stringdata
  namespace: default
  resourceVersion: "384810"
  uid: e326b027-b929-4798-8f91-9a399e0f7bc9
type: Opaque
```

### 使用kustomize来创建Secret
可以在kustomization.yaml文件中定义`secreteGenerator`字段，并在定义中引用其它本地文件、`.env`文件或文字值生成Secret。在所有情况下，都不需要对取值作base64编码。YAML文件的名称**必须**是`kustomization.yaml`或`kustomization.yml`。

若要创建Secret，应用包含kustomization文件的目录。

```bash
[root@k8s-master ~]# cat kustomization/kustomization.yaml
secretGenerator:
- name: database-creds
  literals:
  - username=admin
  - password=1f2d1e2e67df

# -k选项后必须是kustomization.yml文件所在的目录名称
[root@k8s-master ~]# kubectl apply -k kustomization/
secret/database-creds-bkbd782d2c created
```

生成Secret时，Secret的名称最终是由name字段和数据的哈希值拼接而成。这将保证每次修改数据时生成一个新的Secret。
```bash
[root@k8s-master ~]# kubectl get secrets
NAME                                 TYPE                                  DATA   AGE
database-creds-bkbd782d2c            Opaque                                2      7s
default-token-jwvc6                  kubernetes.io/service-account-token   3      104d
my-ingress-playroom                  kubernetes.io/tls                     2      88d
mysecret                             Opaque                                2      18m
mysecret-stringdata                  Opaque                                1      17m
nfs-client-provisioner-token-dg4kp   kubernetes.io/service-account-token   3      82d
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get secret database-creds-bkbd782d2c -o yaml
apiVersion: v1
data:
  password: MWYyZDFlMmU2N2Rm
  username: YWRtaW4=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"password":"MWYyZDFlMmU2N2Rm","username":"YWRtaW4="},"kind":"Secret","metadata":{"annotations":{},"name":"database-creds-bkbd782d2c","namespace":"default"},"type":"Opaque"}
  creationTimestamp: "2022-11-04T03:04:09Z"
  name: database-creds-bkbd782d2c
  namespace: default
  resourceVersion: "386886"
  uid: ee90b6d8-9091-4664-88e8-b550688924ae
type: Opaque
```

## 编辑Secret
使用kubectl来编辑一个已有的Secret：
```bash
[root@k8s-master ~]# kubectl get secret mysecret  -o jsonpath='{.data.password}' | base64 --decode
1f2d1e2e67df[root@k8s-master ~]#
[root@k8s-master ~]#
[root@k8s-master ~]# echo -n 'newpass)911' | base64
bmV3cGFzcyk5MTE=

# 修改password字段
[root@k8s-master ~]# kubectl edit secret mysecret
secret/mysecret edited
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get secret mysecret  -o jsonpath='{.data.password}' | base64 --decode
newpass)911[root@k8s-master ~]#
```

## 使用Secret
Secret可以以数据卷的形式挂载，也可以作为环境变量暴露给Pod中的容器使用。Secret也可用于系统中的其他部分，而不是一定要直接暴露给Pod。Kubernetes会检查Secret的卷数据源，确保所指定的对象引用确实指向类型为Secret的对象。因此，如果Pod依赖于某Secret，该Secret必须先于Pod被创建。

如果Secret内容无法取回（可能因为Secret尚不存在或者临时性地出现API服务器网络连接问题），kubelet会周期性地重试运行Pod。kubelet也会为该Pod报告Event事件，给出读取Secret时遇到的问题细节。

定义一个基于Secret的环境变量时，可以将其标记为可选。默认情况下，所引用的Secret都是必需的。只有所有非可选的Secret都可用时，Pod中的容器才能启动运行。如果Pod引用了Secret中的特定主键，而虽然Secret本身存在，对应的主键不存在，Pod启动也会失败。

### 在Pod中以文件形式使用Secret
Kubernetes可以将Secret以 Pod中一个或多个容器的文件系统中的文件的形式呈现出来。

下面的例子中展示了如何把一个Secret中的所有数据都挂载到指定容器中。Secret的data映射中的每个键值对的Key都成为`mountPath`下面的文件名。如果Pod中包含多个容器，则每个容器需要自己的volumeMounts块，不过针对每个Secret而言，只需要一份`.spec.volumes`设置。

```bash
[root@k8s-master ~]# cat mysecret-pod1.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysecret-pod1
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:   # 每个使用Secret的容器都必须添加该字段
    - name: foo
      mountPath: "/etc/foo"   # Secret在容器中存放的路径
      readOnly: true   # 必须为只读卷
  volumes:
  - name: foo
    secret:
      secretName: mysecret   # 设置为目标Secret对象的名称
      optional: false   # 默认设置，意味着mysecret必须已经存在
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl apply -f mysecret-pod1.yaml
pod/mysecret-pod1 created
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl exec -it mysecret-pod1 -- sh -c 'cat /etc/foo/username'
admin[root@k8s-master ~]#
[root@k8s-master ~]# kubectl exec -it mysecret-pod1 -- sh -c 'cat /etc/foo/password'
newpass)911[root@k8s-master ~]#
```

我们也可以使用`.spec.volumes[].secret.items`字段来挂载Secret中的部分数据，并且指定每个键值对数据的目标路径：

```bash
[root@k8s-master ~]# cat mysecret-pod2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysecret-pod2
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      items:
      - key: username
        path: my-group/my-username
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl apply -f mysecret-pod2.yaml
pod/mysecret-pod2 created
[root@k8s-master ~]# kubectl exec -it mysecret-pod2 -- sh -c 'ls /etc/foo/'
my-group
[root@k8s-master ~]# kubectl exec -it mysecret-pod2 -- sh -c 'cat /etc/foo/my-group/my-username'
admin[root@k8s-master ~]#
```

可以为某个Secret主键设置POSIX文件访问权限位。如果不指定访问权限，默认会使用`0644`。也可以为整个Secret卷设置默认的访问模式，然后再根据需要在主键层面重载。

```bash
[root@k8s-master ~]# kubectl get pod mysecret-pod1 -o yaml
apiVersion: v1
kind: Pod
metadata:
...
volumes:
  - name: foo
    secret:
      defaultMode: 420     # 十进制420对应八进制0644
      optional: false
      secretName: mysecret
...
```

挂载的Secret是会自动更新的。当卷中包含来自Secret的数据，而对应的Secret被更新，Kubernetes会跟踪到这一操作并更新卷中的数据。更新的方式是保证**最终一致性**。

### 以环境变量的方式使用Secret
下面是一个通过环境变量来使用Secret的示例Pod：

```bash
[root@k8s-master ~]# cat mysecret-pod3.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysecret-pod3
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_USERNAME   # 自定义的环境变量名称
        valueFrom:
          secretKeyRef:
            name: mysecret   # 引用的Secret名称
            key: username    # 引用的Secret中键值对数据的Key
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
  restartPolicy: Never
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl apply -f mysecret-pod3.yaml
pod/mysecret-pod3 created
[root@k8s-master ~]# kubectl exec -it  mysecret-pod3 -- sh -c 'echo $SECRET_USERNAME'
admin
[root@k8s-master ~]# kubectl exec -it  mysecret-pod3 -- sh -c 'echo $SECRET_PASSWORD'
newpass)911
```

对于通过`envFrom`字段来填充环境变量的Secret而言，如果其中包含的Key不能被当做合法的环境变量名，这些主键会被忽略掉。Pod仍然可以启动。如果定义的Pod中包含非法的变量名称，则Pod可能启动失败，会形成reason为`InvalidVariableNames`的事件，以及列举被略过的非法主键的消息。

### 容器镜像拉取Secret
如果我们尝试从私有仓库拉取容器镜像，就需要一种方式让每个节点上的kubelet能够完成与镜像库的身份认证。可以通过配置镜像拉取Secret来实现。Secret是在Pod层面来配置的。

Pod的`imagePullSecrets`字段是一个对Pod所在的名字空间中的Secret的引用列表。可以使用imagePullSecrets来将镜像仓库访问凭据传递给kubelet。kubelet使用这个访问凭据信息来替Pod拉取私有镜像。

>:elephant: 关于如何使用Secret从私有镜像仓库或代码仓库拉取镜像来创建Pod，请参考：https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/pull-image-private-registry/


### 在静态Pod中使用Secret
不可以在静态Pod中使用ConfigMap或Secret。

# Secret使用场景
## 作为容器环境变量
使用`envFrom`来将Secret的所有数据定义为容器的环境变量。来自Secret的主键成为Pod中的环境变量名称：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
spec:
  containers:
    - name: test-container
      image: busybox
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:
      - secretRef:
          name: mysecret     # Secret名称
  restartPolicy: Never
```

## 带有SSH密钥的Pod
创建包含SSH密钥的Secret：
```bash
kubectl create secret generic ssh-key-secret \
--from-file=ssh-privatekey=/path/to/.ssh/id_rsa \
--from-file=ssh-publickey=/path/to/.ssh/id_rsa.pub
```

在容器中通过存储卷挂载后，容器就可以随便使用Secret数据来建立SSH连接。

## 在Secret卷中带句点的文件
通过在Secret中定义以句点`.`开头的主键，可以隐藏数据。这些以句点开始的Key代表的是以句点开头的隐藏文件，列举目录内容时必须使用`ls -la`才能看到。
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dotfile-secret
data:
  .secret-file: dmFsdWUtMg0KDQo=
```

## 仅对Pod中一个容器可见的Secret
考虑一个需要处理HTTP请求、执行某些复杂的业务逻辑，之后使用HMAC来对某些消息进行签名的程序。因为这一程序的应用逻辑很复杂，其中可能包含未被注意到的远程服务器文件读取漏洞，这种漏洞可能会把私钥暴露给攻击者。

这一程序可以分隔成两个容器中的两个进程：前端容器要处理用户交互和业务逻辑，但无法看到私钥；签名容器可以看到私钥，并对来自前端的简单签名请求作出响应（例如，通过本地主机网络）。


# 不可变更的Secret
Kubernetes允许将特定的Secret（和 ConfigMap）标记为不可更改（Immutable）。禁止更改现有Secret的数据有下列好处：

- 防止意外（或非预期的）更新导致应用程序中断；
- 对于大量使用Secret的集群而言（至少数万个不同的Secret供Pod挂载），通过将Secret标记为不可变，可以极大降低kube-apiserver的负载，提升集群性能。kubelet不需要监视那些被标记为不可更改的Secret。

通过将Secret的`immutable`字段设置为true来创建不可更改的Secret：
```yaml
apiVersion: v1
kind: Secret
metadata:
  ...
data:
  ...
immutable: true
```

一旦一个Secret或ConfigMap被标记为不可更改，撤销此操作或者更改data字段的内容都是不可能的。只能删除并重新创建这个Secret。现有的Pod将维持对已删除Secret的挂载点，因此建议重建这些Pod。


# Secret的信息安全问题
Secret通常保存重要性各异的数值，其中很多都可能会导致Kubernetes中（例如服务账号令牌）或对外部系统的特权提升。 尽管ConfigMap和Secret的工作方式类似，但Kubernetes对Secret有一些额外的保护。

只有当某个节点上的Pod需要某Secret时，对应的Secret才会被发送到该节点上。如果将Secret挂载到Pod中，kubelet会将数据的副本保存在`tmpfs`中，这样机密的数据不会被写入到持久性存储中。一旦依赖于该Secret的Pod被删除，kubelet会删除来自于该Secret的机密数据的本地副本。

同一个Pod中可能包含多个容器。默认情况下，容器只能访问默认ServiceAccount及其相关Secret。必须显式地定义环境变量或者将卷映射到容器中，才能为容器提供对其他Secret的访问。

针对同一节点上的多个Pod可能有多个Secret。不过，只有某个Pod所请求的Secret才有可能对Pod中的容器可见。因此，一个Pod不会获得访问其他Pod的Secret的权限。

>:skull: 在一个节点上以`privileged: true`运行的所有容器可以访问该节点上使用的所有Secret。


**References**
【1】https://kubernetes.io/docs/concepts/configuration/secret/
【2】https://kubernetes.io/docs/concepts/security/secrets-good-practices/
【3】https://kubernetes.io/zh-cn/docs/tasks/configmap-secret/managing-secret-using-kubectl/
【4】https://kubernetes.io/zh-cn/docs/tasks/configmap-secret/managing-secret-using-config-file/
【5】https://kubernetes.io/zh-cn/docs/tasks/configmap-secret/managing-secret-using-kustomize/
【6】https://kubernetes.io/zh-cn/docs/concepts/policy/resource-quotas/
【7】https://kubernetes.io/zh-cn/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod
【8】https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/pull-image-private-registry/






