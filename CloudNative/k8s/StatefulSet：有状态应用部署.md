---
tags: [k8s]
title: StatefulSet：有状态应用部署
created: '2022-11-01T08:11:15.366Z'
modified: '2022-11-02T15:03:42.830Z'
---

StatefulSet：有状态应用部署

# 为什么要使用StatefulSet
## 有状态应用
Deployment控制器被设计成用来部署和管理无状态应用，例如Web服务。无状态应用中，所有Pod都一模一样，提供同一个服务，也不考虑在哪个Node上运行。可以随意扩缩容。

在实际场景中，Deployment控制器并不能满足所有应用的要求，尤其是分布式应用。分布式应用往往会部署多个实例，这些实例之间往往彼此依赖，存在主从关系、主备关系，例如MySQL主从、etcd集群，等等。类似这种应用被称为**有状态应用**。

以MySQL主从集群为例，有状态应用一般具有以下三个特点：
1. 每个Pod角色不相同（主库、从库）；
2. Pod之间存在连接关系（主从复制进程）；
3. 每个Pod使用独立的存储（非共享存储）。

StatefulSet被设计用来部署有状态应用，比如有以下一个或多个需求：
- Pod具有稳定且唯一的网络标识符；
- Pod需要分配稳定的、持久化的存储；
- 有序地部署、扩缩容、删除和停止Pod；
- 自动且有序地实现Pod的滚动更新。

## StatefulSet的使用限制
- 为Pod分配的存储必须由PersistentVolume Provisioner基于所请求的**存储类**（storage class）来制备，或者由集群管理员预先制备。
- 删除或者扩缩容StatefulSet并**不会删除它关联的存储卷**，这样做是为了保证数据安全。
- 需要创建**无头服务**（Headless Service）来负责StatefulSet中Pod的网络标识。
- 当删除一个StatefulSet时，该StatefulSet不提供任何终止Pod的保证。为了实现StatefulSet中的Pod可以有序且优雅地终止，可以在删除之前将StatefulSet缩容到0。
- 在默认Pod管理策略（OrderedReady）下进行滚动更新时，可能会进入需要人工干预才能修复的损坏状态。


# StatefulSet应用部署实践

对官网上给出的StatefulSet例子（`statefulset-demo.yaml`）的yaml文件稍作修改如下：

```yaml
# 无头服务，用于控制网络域名
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None       # 指定集群内部IP为None
  selector:
    app: nginx

---
apiVersion: apps/v1
kind: StatefulSet       # 类型为StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx        # 必须与.spec.template.metadata.labels匹配
  serviceName: "nginx"  # 必须与上面Service中.metadata.name匹配
  replicas: 3 
  minReadySeconds: 10   # 默认是0
  template:
    metadata:
      labels:
        app: nginx      # 必须与.spec.selector.matchLabels匹配
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  # volumeClaimTemplates是StatefulSet独有（Deployment中不支持），通过存储类来提供稳定的存储
  volumeClaimTemplates:  
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "managed-nfs-storage"   # 必须使用已经配置好的存储类
      resources:
        requests:
          storage: 1Gi
```

## 创建无头服务
有时候应用不需要负载均衡和单独的Service IP。这种情况下，可以通过指定`spec.clusterIP`的值为**None**来创建无头服务（Headless Service）。可以将一个无头服务与其他服务发现机制配置接口，而不必与Kubernetes的实现捆绑在一起。无头服务并不会分配Cluster IP，kube-proxy也不会处理它们，而且k8s控制平台也不会为它们进行负载均衡和代理。DNS如何实现自动配置，依赖于Service是否定义了selector。

在使用StatefulSet部署有状态应用时，为了保证稳定的网络ID（域名），需要使用Headless Service来维护Pod网络身份，并且在StatefulSet的yaml文件中添加`serviceName："无头服务名称"`字段指定StatefulSet控制器要使用对应名称的Headless Service。


## 配置Pod Selector
必须设置StatefulSet的`.spec.selector.matchLabels`字段，使之匹配其在`.spec.template.metadata.labels`中设置的标签。未指定匹配的Pod selector将在创建StatefulSet期间导致验证错误。


## 卷申请模板
为了保证稳定的存储，StatefulSet的存储卷使用`volumeClaimTemplates`创建，称为卷申请模板。当StatefulSet通过`volumeClaimTemplates`中定义的存储类创建一个`PersistentVolume`时，同样也会为每个Pod分配并创建一个带编号的PVC。不同Pod的PV存储**互不共享**，彼此独立。在Pod被意外删除并重建后，其对应的PV存储中的数据不会丢失。

> 关于存储类（StorageClass）的配置，参考文章：https://blog.csdn.net/Sebastien23/article/details/126276294

配置好存储类后，运行上面的yaml文件进行部署：

```bash
[root@k8s-master ~]# kubectl apply -f statefulset-demo.yaml
service/nginx created
statefulset.apps/web created
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get pods
NAME                                      READY   STATUS              RESTARTS      AGE
bs2                                       1/1     Running             5 (12h ago)   88d
nfs-client-provisioner-5d5775b9bb-5657c   1/1     Running             2 (12h ago)   80d
pod-sc-nfs                                1/1     Running             1 (12h ago)   13h
web-0                                     0/1     ContainerCreating   0             5s
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get pods
NAME                                      READY   STATUS              RESTARTS      AGE
bs2                                       1/1     Running             5 (12h ago)   88d
nfs-client-provisioner-5d5775b9bb-5657c   1/1     Running             2 (12h ago)   80d
pod-sc-nfs                                1/1     Running             1 (12h ago)   13h
web-0                                     1/1     Running             0             36s
web-1                                     0/1     ContainerCreating   0             5s
[root@k8s-master ~]#
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get pods
NAME                                      READY   STATUS    RESTARTS      AGE
bs2                                       1/1     Running   5 (12h ago)   88d
nfs-client-provisioner-5d5775b9bb-5657c   1/1     Running   2 (12h ago)   80d
pod-sc-nfs                                1/1     Running   1 (12h ago)   13h
web-0                                     1/1     Running   0             79s
web-1                                     1/1     Running   0             48s
web-2                                     1/1     Running   0             8s
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get sts
NAME   READY   AGE
web    3/3     5h9m
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get svc
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   102d
service/nginx        ClusterIP   None         <none>        80/TCP    68m
```

可以看到，上面的StatefulSet部署过程中，三个Pod副本不是同时创建的，而是依次先后顺序创建的。Pod的命名规则为`<statefulset.metadata.name>-编号`，其中编号从0开始。


# Pod标识
StatefulSet与Deployment控制器的主要区别在于：使用StatefulSet部署的Pod是有身份的，主要体现在主机名、稳定的网络标识、稳定的PV存储这三个方面。

## 有序编号
对于具有N个副本的StatefulSet，该StatefulSet中的每个Pod将被分配一个从0到N-1的整数序号，该序号在此StatefulSet中是唯一的。

Pod的命名规则为`<StatefulSet.metadata.name>-编号`，例如web-0、web-1、web-2。Pod名称与Pod内部执行hostname命令返回的主机名相同。

```bash
[root@k8s-master ~]# kubectl exec -it web-0 -- sh
# hostname
web-0
# exit
[root@k8s-master ~]#
```

## 稳定的网络ID
StatefulSet可以使用无头服务控制它的Pod的网络域。管理域的这个服务的格式为： 
```
<headlessService.metadata.name>.<namespace>.svc.cluster.local
```
其中`cluster.local`是集群域。

一旦每个Pod创建成功，就会得到一个匹配的DNS子域，格式为
```
<StatefulSet.metadata.name-index>.<headlessService.metadata.name>.<namespace>.svc.cluster.local
```

对于前面部署的StatefulSet，尝试在busybox的Pod中查看域名解析记录：
```bash
[root@k8s-master ~]# kubectl run bs --image=busybox:1.28.4 -- sleep 24h
[root@k8s-master ~]# kubectl exec -it bs2 -- sh
/ # nslookup nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx
Address 1: 10.244.169.130 web-1.nginx.default.svc.cluster.local
Address 2: 10.244.36.113 web-2.nginx.default.svc.cluster.local
Address 3: 10.244.36.112 web-0.nginx.default.svc.cluster.local
/ #
/ # ping web-1.nginx.default.svc.cluster.local
PING web-1.nginx.default.svc.cluster.local (10.244.169.130): 56 data bytes
64 bytes from 10.244.169.130: seq=0 ttl=63 time=0.169 ms
64 bytes from 10.244.169.130: seq=1 ttl=63 time=0.120 ms
64 bytes from 10.244.169.130: seq=2 ttl=63 time=0.125 ms
^C
--- web-1.nginx.default.svc.cluster.local ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.120/0.138/0.169 ms
/ # exit
```

## 稳定的存储
对于StatefulSet中定义的每个VolumeClaimTemplate，每个Pod都会对应地创建一个PersistentVolumeClaim（PVC）。在上面的nginx示例中，每个Pod将会得到基于存储类`managed-nfs-storage`制备的1 Gib的PersistentVolume（PV）。如果没有声明StorageClass，就会使用默认的存储类。 

当一个Pod被调度（重新调度）到节点上时，它的`volumeMounts`会挂载与其PVC相关联的PV。当Pod或者StatefulSet被删除时，与PVC相关联的PV并不会被删除。要删除它必须通过手动方式来完成。

检查上面的StatefulSet部署示例中三个Pod是否已经通过指定的存储类分配了PV存储：
```bash
[root@k8s-master ~]# kubectl get sc,pv,pvc
NAME                                              PROVISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
storageclass.storage.k8s.io/managed-nfs-storage   k8s-sigs.io/nfs-subdir-external-provisioner   Delete          Immediate           false                  80d

# RWO表示可以被一个节点以读写方式挂载
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                      STORAGECLASS          REASON   AGE
persistentvolume/pvc-376d3ffc-e893-4550-a204-ed5df076d8b3   1Gi        RWX            Delete           Bound    default/test-nfs-scclaim   managed-nfs-storage            14h
persistentvolume/pvc-43f414bb-bc3b-40fa-bf1b-74ef7597435c   1Gi        RWO            Delete           Bound    default/www-web-2          managed-nfs-storage            84m
persistentvolume/pvc-86e62634-1ef6-4f76-9102-aecc67046054   1Gi        RWO            Delete           Bound    default/www-web-0          managed-nfs-storage            85m
persistentvolume/pvc-edc6fb5a-a55b-465c-ac1b-661d5cd45982   1Gi        RWO            Delete           Bound    default/www-web-1          managed-nfs-storage            85m

NAME                                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
persistentvolumeclaim/test-nfs-scclaim   Bound    pvc-376d3ffc-e893-4550-a204-ed5df076d8b3   1Gi        RWX            managed-nfs-storage   14h
persistentvolumeclaim/www-web-0          Bound    pvc-86e62634-1ef6-4f76-9102-aecc67046054   1Gi        RWO            managed-nfs-storage   85m
persistentvolumeclaim/www-web-1          Bound    pvc-edc6fb5a-a55b-465c-ac1b-661d5cd45982   1Gi        RWO            managed-nfs-storage   85m
persistentvolumeclaim/www-web-2          Bound    pvc-43f414bb-bc3b-40fa-bf1b-74ef7597435c   1Gi        RWO            managed-nfs-storage   84m
[root@k8s-master ~]#
```

可以验证，StatefulSet中各个Pod的PV存储都是各自独立的。
```bash
# --往各个Pod的同一路径下写入不同的文件
[root@k8s-master ~]# kubectl exec -it web-0 -- sh
# cd /usr/share/nginx/html
# touch 0.html
# exit
[root@k8s-master ~]# kubectl exec -it web-1 -- sh
# cd /usr/share/nginx/html
# touch 1.html
# exit
[root@k8s-master ~]# kubectl exec -it web-2 -- sh
# cd /usr/share/nginx/html
# touch 2.html
# exit

# --在NFS共享存储服务器上观察
[root@k8s-node1 kubernetes]# df -Th | grep ifs
192.168.124.140:/ifs/kubernetes                                                                   nfs4       37G  3.7G   34G  10% /var/lib/kubelet/pods/93221f31-c910-49c0-b58c-5704577f1209/volumes/kubernetes.io~nfs/nfs-client-root
192.168.124.140:/ifs/kubernetes/default-test-nfs-scclaim-pvc-376d3ffc-e893-4550-a204-ed5df076d8b3 nfs4       37G  3.7G   34G  10% /var/lib/kubelet/pods/d0ae0b1e-c8f9-4223-aa45-ba159979b4a4/volumes/kubernetes.io~nfs/pvc-376d3ffc-e893-4550-a204-ed5df076d8b3
192.168.124.140:/ifs/kubernetes/default-www-web-0-pvc-86e62634-1ef6-4f76-9102-aecc67046054        nfs4       37G  3.7G   34G  10% /var/lib/kubelet/pods/97efa9d7-8ca2-420f-b6f7-8f0d1e79fb32/volumes/kubernetes.io~nfs/pvc-86e62634-1ef6-4f76-9102-aecc67046054
192.168.124.140:/ifs/kubernetes/default-www-web-2-pvc-43f414bb-bc3b-40fa-bf1b-74ef7597435c        nfs4       37G  3.7G   34G  10% /var/lib/kubelet/pods/5dbb574a-160b-47e3-a967-e10c9708a3b6/volumes/kubernetes.io~nfs/pvc-43f414bb-bc3b-40fa-bf1b-74ef7597435c
[root@k8s-node1 kubernetes]#
[root@k8s-node1 kubernetes]# cd /ifs/kubernetes/
[root@k8s-node1 kubernetes]#
[root@k8s-node1 kubernetes]# ll
total 4
-rw-r--r--. 1 root root  0 Aug 12 23:26 a.txt
-rw-r--r--. 1 root root  0 Aug 12 23:29 b.txt
drwxrwxrwx. 2 root root  6 Nov  1 10:54 default-test-nfs-scclaim-pvc-376d3ffc-e893-4550-a204-ed5df076d8b3
drwxrwxrwx. 2 root root 20 Nov  2 01:42 default-www-web-0-pvc-86e62634-1ef6-4f76-9102-aecc67046054
drwxrwxrwx. 2 root root 20 Nov  2 01:43 default-www-web-1-pvc-edc6fb5a-a55b-465c-ac1b-661d5cd45982
drwxrwxrwx. 2 root root 20 Nov  2 01:44 default-www-web-2-pvc-43f414bb-bc3b-40fa-bf1b-74ef7597435c
-rw-r--r--. 1 root root 20 Aug 12 23:49 index.html
[root@k8s-node1 kubernetes]#
[root@k8s-node1 kubernetes]# ls default-www-web-0-pvc-86e62634-1ef6-4f76-9102-aecc67046054
0.html
[root@k8s-node1 kubernetes]# ls default-www-web-1-pvc-edc6fb5a-a55b-465c-ac1b-661d5cd45982/
1.html
[root@k8s-node1 kubernetes]# ls default-www-web-2-pvc-43f414bb-bc3b-40fa-bf1b-74ef7597435c/
2.html
[root@k8s-node1 kubernetes]#
```

删除并自动重建StatefulSet中的某个Pod后，对应PV存储中的文件依然会被保留，不会丢失。
```bash
[root@k8s-master ~]# kubectl delete pod web-2
pod "web-2" deleted
[root@k8s-master ~]# kubectl get pods
NAME                                      READY   STATUS              RESTARTS      AGE
bs2                                       1/1     Running             5 (14h ago)   88d
nfs-client-provisioner-5d5775b9bb-5657c   1/1     Running             2 (14h ago)   80d
pod-sc-nfs                                1/1     Running             1 (14h ago)   14h
web-0                                     1/1     Running             0             115m
web-1                                     1/1     Running             0             114m
web-2                                     0/1     ContainerCreating   0             3s
[root@k8s-master ~]# kubectl exec -it web-2 -- sh
# ls /usr/share/nginx/html
2.html
# exit
[root@k8s-master ~]#
```

## Pod名称标签
当通过StatefulSet控制器创建Pod时，它会添加一个标签`statefulset.kubernetes.io/pod-name`，该标签值设置为Pod名称。这个标签允许我们给StatefulSet中的特定Pod绑定一个Service。

```bash
[root@k8s-master ~]# kubectl get pods -l statefulset.kubernetes.io/pod-name
NAME    READY   STATUS    RESTARTS        AGE
web-0   1/1     Running   1 (7h13m ago)   9h
web-1   1/1     Running   1 (7h13m ago)   9h
web-2   1/1     Running   1 (7h13m ago)   7h20m
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get pods --show-labels
NAME                                      READY   STATUS    RESTARTS        AGE     LABELS
bs2                                       1/1     Running   6 (7h12m ago)   88d     run=bs2
nfs-client-provisioner-5d5775b9bb-5657c   1/1     Running   3 (7h13m ago)   81d     app=nfs-client-provisioner,pod-template-hash=5d5775b9bb
pod-sc-nfs                                1/1     Running   2 (7h13m ago)   22h     <none>
web-0                                     1/1     Running   1 (7h13m ago)   9h      app=nginx,controller-revision-hash=web-67bb74dc,statefulset.kubernetes.io/pod-name=web-0
web-1                                     1/1     Running   1 (7h13m ago)   9h      app=nginx,controller-revision-hash=web-67bb74dc,statefulset.kubernetes.io/pod-name=web-1
web-2                                     1/1     Running   1 (7h13m ago)   7h21m   app=nginx,controller-revision-hash=web-67bb74dc,statefulset.kubernetes.io/pod-name=web-2
[root@k8s-master ~]#

```

# StatefulSet扩缩容与更新
## 扩容与缩容
**OrderedReady**是StatefulSet默认的Pod管理策略。在对StatefulSet进行部署和扩缩容时，遵循以下策略：

- 对于包含N个副本的StatefulSet，当部署Pod时，它们是依次**顺序创建**的，顺序为`0..N-1`。
- 当删除Pod时，它们是**逆序终止**的，顺序为`N-1..0`。
- 在将扩缩操作应用到Pod之前，它**前面的所有Pod必须是Running和Ready状态**。
- 在一个Pod终止之前，它**后面所有的Pod必须完全关闭**。

StatefulSet扩容/缩容副本的命令为`kubectl scale`或者`kubectl patch`：
```
kubeclt scale sts <StatefulSet名称> --replicas=<副本数>
kubectl patch sts <StatefulSet名称> -p '{"spec":{"replicas":<副本数>}}'
```

示例：
```sql
[root@k8s-master ~]# kubectl get pods -l app=nginx
NAME    READY   STATUS    RESTARTS        AGE
web-0   1/1     Running   1 (7h51m ago)   9h
web-1   1/1     Running   1 (7h51m ago)   9h
web-2   1/1     Running   1 (7h51m ago)   7h59m
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl scale sts web  --replicas=5   # 扩容
statefulset.apps/web scaled
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get pods -l app=nginx
NAME    READY   STATUS              RESTARTS        AGE
web-0   1/1     Running             1 (7h52m ago)   9h
web-1   1/1     Running             1 (7h52m ago)   9h
web-2   1/1     Running             1 (7h52m ago)   8h
web-3   0/1     ContainerCreating   0               3s
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get pods -l app=nginx
NAME    READY   STATUS              RESTARTS        AGE
web-0   1/1     Running             1 (7h53m ago)   9h
web-1   1/1     Running             1 (7h52m ago)   9h
web-2   1/1     Running             1 (7h53m ago)   8h
web-3   1/1     Running             0               23s
web-4   0/1     ContainerCreating   0               3s
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl scale sts web  --replicas=3   # 缩容
statefulset.apps/web scaled
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get pods -l app=nginx
NAME    READY   STATUS    RESTARTS     AGE
web-0   1/1     Running   1 (8h ago)   10h
web-1   1/1     Running   1 (8h ago)   10h
web-2   1/1     Running   1 (8h ago)   8h
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                      STORAGECLASS          REASON   AGE
persistentvolume/pvc-376d3ffc-e893-4550-a204-ed5df076d8b3   1Gi        RWX            Delete           Bound    default/test-nfs-scclaim   managed-nfs-storage            23h
persistentvolume/pvc-43f414bb-bc3b-40fa-bf1b-74ef7597435c   1Gi        RWO            Delete           Bound    default/www-web-2          managed-nfs-storage            10h
persistentvolume/pvc-465c74c9-2132-4173-9e17-8abb28eba570   1Gi        RWO            Delete           Bound    default/www-web-4          managed-nfs-storage            23m
persistentvolume/pvc-86e62634-1ef6-4f76-9102-aecc67046054   1Gi        RWO            Delete           Bound    default/www-web-0          managed-nfs-storage            10h
persistentvolume/pvc-9392451d-3e19-4782-afa4-eaeb011aa0c1   1Gi        RWO            Delete           Bound    default/www-web-3          managed-nfs-storage            23m
persistentvolume/pvc-edc6fb5a-a55b-465c-ac1b-661d5cd45982   1Gi        RWO            Delete           Bound    default/www-web-1          managed-nfs-storage            10h

NAME                                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
persistentvolumeclaim/test-nfs-scclaim   Bound    pvc-376d3ffc-e893-4550-a204-ed5df076d8b3   1Gi        RWX            managed-nfs-storage   23h
persistentvolumeclaim/www-web-0          Bound    pvc-86e62634-1ef6-4f76-9102-aecc67046054   1Gi        RWO            managed-nfs-storage   10h
persistentvolumeclaim/www-web-1          Bound    pvc-edc6fb5a-a55b-465c-ac1b-661d5cd45982   1Gi        RWO            managed-nfs-storage   10h
persistentvolumeclaim/www-web-2          Bound    pvc-43f414bb-bc3b-40fa-bf1b-74ef7597435c   1Gi        RWO            managed-nfs-storage   10h
persistentvolumeclaim/www-web-3          Bound    pvc-9392451d-3e19-4782-afa4-eaeb011aa0c1   1Gi        RWO            managed-nfs-storage   23m
persistentvolumeclaim/www-web-4          Bound    pvc-465c74c9-2132-4173-9e17-8abb28eba570   1Gi        RWO            managed-nfs-storage   23m
```
可以看到，缩容后被删除的`web-3`和`web-4`对应的PV存储被保留了，没有被一同清除。

## 更新策略
通过StatefulSet的`.spec.updateStrategy`字段可以配置和禁用Pod的容器、标签、资源请求或限制、以及注解的自动滚动更新。该字段有两个允许的值：

- **OnDelete**
当StatefulSet的`.spec.updateStrategy.type`设置为OnDelete时，控制器将不会自动更新StatefulSet中的Pod。用户必须手动删除Pod以便让控制器创建新的Pod，以此来对StatefulSet的`.spec.template`的变动作出反应。

- **RollingUpdate**
RollingUpdate更新策略下，StatefulSet中的Pod执行自动滚动更新。这是**默认**的更新策略。


## 滚动更新
当StatefulSet采用滚动更新策略时，StatefulSet控制器会删除和重建StatefulSet中的每个Pod。它将按照与Pod终止相同的顺序（**从最大序号到最小序号**）进行，每次更新一个Pod。

Kubernetes控制平面会等到被更新的Pod进入Running和Ready状态，然后再更新它前面编号更小的Pod。如果设置了`.spec.minReadySeconds`（**最短就绪秒数**）字段，控制平面在Pod就绪后会额外等待一定的时间再执行下一步。

### 分区滚动更新
通过声明`.spec.updateStrategy.rollingUpdate.partition`字段，RollingUpdate更新策略可以实现分区。如果声明了一个分区，当StatefulSet的`.spec.template`被更新时，所有序号大于等于该分区序号的Pod都会被更新。**所有序号小于该分区序号的Pod都不会被更新，并且即使它们被删除也会依照之前的版本进行重建**。如果StatefulSet的`.spec.updateStrategy.rollingUpdate.partition`大于它的`.spec.replicas`，则对它的`.spec.template`的更新将不会传递到它的Pod。在大多数情况下都不需要使用分区，除非我们希望进行阶段更新、执行金丝雀或执行分阶段上线。

### 最大不可用Pod数
可以通过指定`.spec.updateStrategy.rollingUpdate.maxUnavailable`字段来控制更新期间不可用的Pod的最大数量。该值可以是绝对值（例如5）或者是期望Pod个数的百分比（例如10%）。绝对值是根据百分比值四舍五入计算的。该字段不能为0。默认设置为1。

该字段适用于0到`replicas - 1`范围内的所有Pod。如果在0到`replicas - 1`范围内存在不可用Pod，这类Pod将被计入maxUnavailable值。

### 强制回滚
在默认Pod管理策略（OrderedReady）下使用滚动更新，可能会进入需要人工干预才能修复的损坏状态。

如果更新后Pod模板配置进入无法运行或就绪的状态（例如， 由于错误的二进制文件或应用程序级配置错误），StatefulSet将停止回滚并等待。在这种状态下，仅将Pod模板还原为正确的配置是不够的。StatefulSet将继续等待损坏状态的Pod准备就绪（需要人工干预修复），然后再尝试将其恢复为正常工作配置。恢复模板后，还必须删除StatefulSet曾经尝试使用错误的配置来运行的Pod，以便StatefulSet能够使用修改后的正确模板来重新创建Pod。


**References**
【1】https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
【2】https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#headless-services
【3】https://kubernetes.io/zh-cn/docs/tutorials/stateful-application/basic-stateful-set/





