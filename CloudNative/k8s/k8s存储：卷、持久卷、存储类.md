@[TOC](k8s存储：卷、持久卷、存储类)

容器中的文件是临时存储在磁盘上的，数据卷的引入主要是为了解决以下几个问题：
- 当容器升级或者崩溃时，kubelet会重建容器，容器内原有的文件会丢失；
- 一个Pod中运行的多个容器需要共享文件。

常用的数据卷（**Volume**）有：
- 本地：例如hostPath、emptyDir；
- 网络：例如NFS、Ceph、GlusterFS；
- 公有云：例如AWS EBS；
- k8s资源：例如configmap、secret。


# emptyDir：临时数据卷
emptyDir卷是一种临时存储卷，与Pod的生命周期绑定。在Pod运行期间，emptyDir一直存在；如果Pod被删除了，emptyDir中的数据也会被永久删除。

>**注**：容器崩溃**不**会导致Pod被删除，因此容器崩溃期间emptyDir中的数据是安全的。

emptyDir可以实现Pod中容器之间的**数据共享**。尽管Pod中的容器挂载emptyDir卷的路径可能不同，这些容器都可以读写emptyDir卷中相同的文件。

emptyDir的配置示例如下：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - image: centos
    name: writer
	command: ["bash","-c","for i in {1..100}; do echo $i >> /data/tmpfile; sleep 1; done"]
    volumeMounts:
    - mountPath: /data
      name: data
	  
  - image: centos
    name: reader
	command: ["bash", "-c", "tail -f /data/tmpfile"]
	volumeMounts:
	  - mountPath: /data
	 
  volumes:
  - name: data
    emptyDir: {}
```


emptyDir挂在的容器目录位于Pod所在节点的以下路径：
```
/var/lib/kubelet/pods/${container_ID}/volumes/kubernetes.io~empty-dir
```

# hostPath：节点数据卷
hostPath卷能将Pod所在宿主机节点文件系统上的文件或目录挂载到Pod中。具有相同配置（例如基于同一PodTemplate创建）的多个Pod，会由于节点上文件的不同，而在不同节点上有不同的行为。

>**注**：HostPath卷存在许多安全风险，最佳做法是尽可能避免使用HostPath。 当必须使用HostPath卷时，它的范围应仅限于所需的文件或目录，并以只读方式挂载。HostPath卷可能会暴露特权系统凭据（例如Kubelet）或特权API（例如容器运行时套接字），可用于容器逃逸或攻击集群的其他部分。


HostPath配置示例如下：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /mydata
      name: test-volume1
	- mountPath: /mylog
      name: test-volume2
  volumes:
  - name: test-volume1
    hostPath:
      path: /data       # 宿主上目录位置
      type: Directory 
  - name: test-volume2
    hostPath:
      path: /log        # 宿主上目录位置
      type: Directory 	  
```

# nfs：网络数据卷
提供对NFS挂载支持，可以自动将NFS共享路径挂载到Pod中。

搭建NFS文件共享服务器：
```bash
[root@k8s-node1 ~]# yum install nfs-utils -y
[root@k8s-node1 ~]# vi /etc/exports
/ifs/kubernetes *(rw,sync,no_root_squash)
[root@k8s-node1 ~]# mkdir -p /ifs/kubernetes
[root@k8s-node1 ~]# systemctl start nfs
[root@k8s-node1 ~]# systemctl enable nfs
```

所有Node节点上都要安装nfs-utils包。在其他节点挂载NFS路径：
```bash
[root@k8s-node2 ~]# yum install nfs-utils -y
[root@k8s-node2 ~]# mkdir -p /ifs/kubernetes 
#手动挂载
[root@k8s-node2 ~]# mount -t nfs 192.168.136.120:/ifs/kubernetes /ifs/kubernetes
#开机自动挂载
[root@k8s-node2 ~]# echo '192.168.136.120:/ifs/kubernetes  /ifs/kubernetes  nfs4  defaults  0 0' >> /etc/fstab
[root@k8s-node2 ~]# mount -a
#移除挂载
[root@k8s-node2 ~]# umount -v /ifs/kubernetes/
```

将nginx网站程序根目录持久化到NFS存储，为多个Pod提供网站程序文件。
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver
spec:
  selector:
    matchLabels:
	  app: nginx
  replicas: 3
  template:
    metadata:
	  labels:
	    app: nginx
	spec:
	  containers:
	  - name: nginx
	    image: nginx
		volumeMounts:
		- name: wwwroot
		  mountPath: /usr/share/nginx/html
	  volumes:
	  - name: wwwroot
	    nfs:
		  server: 192.168.136.120
		  path: /ifs/kubernetes
```

# PV：持久卷
持久卷（**PersistentVolume, PV**）是对存储资源创建和使用的抽象，使得存储可以作为集群中的资源被管理。

## 持久卷的类型
持久卷是通过插件的形式实现的。目前k8s支持的插件包括：

- awsElasticBlockStore
- azureDisk
- cephfs
- csi 容器存储接口
- hostPath卷
- iscsi（iSCSI存储）
- local 节点上挂载的本地存储设备
- nfs 网络文件系统存储
- vsphereVolume（vSphere VMDK卷）

等等。

## volumeMode：卷模式
k8s支持两种卷模式（volumeModes）：`Filesystem`（文件系统）和`Block`（块存储）。volumeMode是一个可选的API参数。如果该参数被省略，默认的卷模式是Filesystem。

volumeMode属性设置为Filesystem的卷会被Pod挂载到某个目录。如果卷的存储来自某块设备而该设备目前为空，k8s会在第一次挂载卷之前在设备上创建文件系统。

也可以将volumeMode设置为Block，以便将卷作为原始块设备来使用。这类卷以块设备的方式交给Pod使用，其上没有任何文件系统，使用前需要格式化。


## accessModes：访问模式
PV支持的访问模式（accessModes）有：

- `ReadWriteOnce`（RWO）：卷可以被一个节点以读写方式挂载。ReadWriteOnce访问模式也允许运行在同一节点上的多个 Pod 访问卷。

- `ReadOnlyMany`（ROX）：卷可以被多个节点以只读方式挂载。

- `ReadWriteMany`（RWX）：卷可以被多个节点以读写方式挂载。

- `ReadWriteOncePod`：卷可以被单个Pod以读写方式挂载。 
如果想确保整个集群中只有一个Pod可以读取或写入该PVC， 要使用ReadWriteOncePod访问模式。这只支持CSI卷以及需要Kubernetes 1.22以上版本。

>参考：https://kubernetes.io/blog/2021/09/13/read-write-once-pod-access-mode-alpha/


## Phase：状态阶段
每个卷会处于以下阶段（Phase）之一：

- `Available`（可用）：卷是一个空闲资源，尚未绑定到任何申领；
- `Bound`（已绑定）：该卷已经绑定到某申领；
- `Released`（已释放）：所绑定的申领已被删除，但是资源尚未被集群回收；
- `Failed`（失败）：卷的自动回收操作失败。

## Reclaim Policy：回收策略
目前的回收策略有：

- `Retain`（手动回收）：默认策略，保留数据，需要手动回收；
- `Recycle`（回收）：保留PV，但是会清除PV中的数据，相当于`rm -rf /thevolume/*`；
- `Delete`（删除）：删除PV和数据，与PV相关联的后端存储同时被删除。诸如AWS EBS、GCE PD、Azure Disk或OpenStack Cinder卷这类关联存储资源也被删除。


# PVC：持久卷申领
持久卷申领（**PersistentVolumeClaim, PVC**）是用户对集群中PV资源的请求。

Pod申请PVC作为卷来使用，k8s通过PVC查找绑定的PV，并在Pod中挂载。创建PVC之后，k8s控制平面将通过匹配访问模式`accessModes`和容量`storage`查找满足要求的PV。如果控制平面找到具有合适的PV，则会将PVC绑定到该PV上。

PVC和PV的一般匹配原则如下：
- 首先，`PV.spec.accessModes`要与`PVC.spec.accessModes`相同；
- 其次，`PV.spec.capacity.storage`不能小于`PVC.spec.resources.requests.storage`；
- 如果有多个PV满足以上要求，则优先选择容量最接近的PV；
- 如果没有找到合适的PV，则容器和PVC会一直处于Pending状态。

PVC中的需求容量字段`storage`只能用于匹配PV，并**不能**起到资源限制作用。容器实际能够使用的容量大小取决于后端存储。也就是说，假设PVC中定义的资源需求为10G，PV后端使用的存储的剩余空间为100G，那么绑定该PV后，容器实际可以使用的存储大小也是100G。


## 示例：创建一个nfs类型的PV
首先确保所有Node节点操作系统已经挂载NFS。

容器应用中使用PVC来申请PV资源（my-pod-pvc.yaml）：
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: webserver
    image: nginx
	ports:
	- containerPort: 80
	volumeMounts:
	- name: wwwroot
	  mountPath: /usr/share/nginx/html
  volumes:
  - name: wwwroot           #与volumeMounts.name保持一致
    persistentVolumeClaim:
	  claimName: my-pvc
	  
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: 
  name: my-pvc       #与claimName保持一致
spec:
  accessModes:       #访问模式
    - ReadWriteMany
  resources:
    requests:
	  storage: 8Gi   #容量
```

查看Pods和PVC状态：
```bash
[root@k8s-master ~]# kubectl apply -f my-pod-pvc.yaml
[root@k8s-master ~]# kubectl get pods,pvc
NAME                             READY   STATUS    RESTARTS        AGE
pod/my-pod                       0/1     Pending   0               23s

NAME                           STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/my-pvc   Pending                                                     23s
```
由于尚未创建PV，my-pod和my-pvc的STATUS都应该为**Pending**。


接下来定义一个nfs类型的PV资源（my-pv.yaml）：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi       #容量
  accessModes:
    - ReadWriteMany     #访问模式
  nfs:
    path: /if/kubernetes
	server: 192.168.136.120
```

查看Pods、PVC和PV状态：
```bash
[root@k8s-master ~]# kubectl apply -f my-pv.yaml

#PV和PVC未绑定前
[root@k8s-master ~]# kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
my-pv   10Gi       RWX            Retain           Available                                   2m20s

#PV和PVC绑定后
[root@k8s-master ~]# kubectl get pods,pvc,pv
NAME                             READY   STATUS    RESTARTS        AGE
pod/my-pod                       1/1     Running   0               3m8s

NAME                           STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/my-pvc   Bound    my-pv    1Gi        RWX                           3m8s

NAME                     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   REASON   AGE
persistentvolume/my-pv   10Gi       RWX            Retain           Bound    default/my-pvc                           3m2s

```

正常情况下，my-pvc和my-pv的STATUS都会变成**Bound**，Pods的状态也会变成Running。

进入容器测试一下：
```bash
[root@k8s-node1 ~]# ls /ifs/kubernetes/
a.txt  b.txt

[root@k8s-master ~]# kubectl exec -it my-pod -- bash
root@my-pod:/# cd /usr/share/nginx/html
root@my-pod:/usr/share/nginx/html# ls
a.txt  b.txt
root@my-pod:/usr/share/nginx/html# echo '<p>Hello Nginx!</p>' > index.html
root@my-pod:/usr/share/nginx/html# ls
a.txt  b.txt  index.html
```

# StorageClass：存储类
PV的供给有两种方式：静态供给和动态供给。像前面那样，集群管理员手动创建若干PV卷，供容器消费使用，即是**静态供给**的方式。静态供给的缺点是维护成本很高。

集群管理员也可以使用`StorageClass`实现PV资源的**动态供给**。每个StorageClass都包含provisioner、parameters和reclaimPolicy字段， 这些字段会在StorageClass需要动态分配PV时会使用到。

## Provisioner：存储类制备器
每个存储类都有一个provisioner，用来决定使用哪个卷插件制备PV。 

目前，NFS没有内部制备器，需要使用外部制备器插件来实现PV的动态供给。

>官方内部支持的制备器参见：https://kubernetes.io/zh-cn/docs/concepts/storage/storage-classes/#provisioner
NFS外部制备器插件参考：https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner


NFS provisioner插件部署：
```bash
[root@k8s-master nfs-storageClass]# ls
class.yaml  deployment.yaml  rbac.yaml
#修改NFS服务器地址和共享文件路径
[root@k8s-master nfs-storageClass]# grep NFS deployment.yaml
            - name: NFS_SERVER
            - name: NFS_PATH

#授权访问api-server
[root@k8s-master nfs-storageClass]# kubectl apply -f rbac.yaml
#部署nfs provisioner插件
[root@k8s-master nfs-storageClass]# kubectl apply -f deployment.yaml
#创建存储类
[root@k8s-master nfs-storageClass]# kubectl apply -f class.yaml

#查看存储类
[root@k8s-master nfs-storageClass]# kubectl get sc
NAME                  PROVISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage   k8s-sigs.io/nfs-subdir-external-provisioner   Delete          Immediate           false                  2m3s
```

**注**：部署配置文件下载地址为 https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/deploy


## NFS StorageClass动态供给PV
修改前面的示例，使用nfs存储类来实现动态供给PV卷（pod-scnfs.yaml）。
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-sc-nfs
spec:
  containers:
  - name: pod-sc-nfs
    image: nginx
	ports:
	- containerPort: 80
	volumeMounts:
	- name: sc-nfs-pvc
	  mountPath: /usr/share/nginx/html
  volumes:
  - name: sc-nfs-pvc           #与volumeMounts.name保持一致
    persistentVolumeClaim:
	  claimName: test-nfs-scclaim
	  
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: 
  name: test-nfs-scclaim       #与claimName保持一致
spec:
  storageClassName: "managed-nfs-storage"   #与部署nfs provisioner插件时class.yaml中metadata.name保持一致
  accessModes:       #访问模式
    - ReadWriteMany
  resources:
    requests:
	  storage: 1Gi   #容量
```

无需手动创建PV，检查部署后状态：
```bash
[root@k8s-master ~]# kubectl apply -f pod-scnfs.yaml
[root@k8s-master ~]# kubectl get pods
NAME                                      READY   STATUS    RESTARTS        AGE
my-pod                                    1/1     Running   0               5h40m
nfs-client-provisioner-5d5775b9bb-5657c   1/1     Running   0               21m
pod-sc-nfs                                1/1     Running   0               24s
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get pvc,pv
NAME                                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
persistentvolumeclaim/my-pvc             Bound    my-pv                                      1Gi        RWX                                  5h41m
persistentvolumeclaim/test-nfs-scclaim   Bound    pvc-14c4221b-f3af-465c-b431-eddc1191a864   1Gi        RWX            managed-nfs-storage   33s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                      STORAGECLASS          REASON   AGE
persistentvolume/my-pv                                      1Gi        RWX            Retain           Bound    default/my-pvc                                            5h41m
persistentvolume/pvc-14c4221b-f3af-465c-b431-eddc1191a864   1Gi        RWX            Delete           Bound    default/test-nfs-scclaim   managed-nfs-storage            33s
[root@k8s-master ~]#
```

可以看到，动态分配的PV的STORAGECLASS一栏为`managed-nfs-storage`，回收策略为`Delete`。

查看NFS Server上的共享文件路径：
```bash
[root@k8s-master ~]# kubectl exec -it pod-sc-nfs -- bash
root@pod-sc-nfs:/# df -Th
Filesystem                                                                                        Type     Size  Used Avail Use% Mounted on
...
192.168.136.120:/ifs/kubernetes/default-test-nfs-scclaim-pvc-14c4221b-f3af-465c-b431-eddc1191a864 nfs4      37G  3.7G   34G  10% /usr/share/nginx/html
...
root@pod-sc-nfs:/# cd /usr/share/nginx/html
root@pod-sc-nfs:/usr/share/nginx/html# ls
root@pod-sc-nfs:/usr/share/nginx/html# touch test-nfs-file-001
root@pod-sc-nfs:/usr/share/nginx/html# exit

[root@k8s-node1 ~]# ll /ifs/kubernetes/
total 4
-rw-r--r--. 1 root root  0 Aug 12 23:26 a.txt
-rw-r--r--. 1 root root  0 Aug 12 23:29 b.txt
drwxrwxrwx. 2 root root  6 Aug 13 05:23 default-test-nfs-scclaim-pvc-14c4221b-f3af-465c-b431-eddc1191a864
-rw-r--r--. 1 root root 20 Aug 12 23:49 index.html
[root@k8s-node1 ~]# ls /ifs/kubernetes/default-test-nfs-scclaim-pvc-14c4221b-f3af-465c-b431-eddc1191a864/
test-nfs-file-001
```

测试Delete回收策略是否会生效：
```bash
[root@k8s-master ~]# kubectl delete -f pod-scnfs.yaml
pod "pod-sc-nfs" deleted
persistentvolumeclaim "test-nfs-scclaim" deleted

#查看绑定的PV是否一并被删除
[root@k8s-master ~]# kubectl get pvc,pv,sc
NAME                           STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/my-pvc   Bound    my-pv    1Gi        RWX                           5h55m

NAME                     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   REASON   AGE
persistentvolume/my-pv   1Gi        RWX            Retain           Bound    default/my-pvc                           5h55m

NAME                                              PROVISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
storageclass.storage.k8s.io/managed-nfs-storage   k8s-sigs.io/nfs-subdir-external-provisioner   Delete          Immediate           false                  36m

#查看NFS Server上的共享文件路径是否同时被删除
[root@k8s-node1 ~]# ls /ifs/kubernetes/
a.txt  b.txt  index.html
```

如果想在删除PVC时，对PV中的数据进行归档，可以将nfs provisioner插件部署配置文件`class.yaml`中的参数
```
archiveOnDelete: "false"
```
修改为`true`后重新部署。这样NFS Server上共享文件路径中的数据就会留存一份归档。


**References**
【1】https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/
【2】https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/
【3】https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-persistent-volume-storage/
【4】https://kubernetes.io/zh-cn/docs/concepts/storage/storage-classes/


