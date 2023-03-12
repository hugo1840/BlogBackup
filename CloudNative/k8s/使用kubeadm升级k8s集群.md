---
tags: [k8s]
title: 使用kubeadm升级k8s集群
created: '2023-03-12T06:34:21.939Z'
modified: '2023-03-12T10:31:57.180Z'
---

使用kubeadm升级k8s集群

升级k8s集群的方式取决于集群的部署方式、以及后续更改它的方式。一般的升级流程是：
1. 升级主控制平面节点；
2. 升级其他控制平面节点；
3. 升级集群中的工作节点；
4. 升级kubectl之类的客户端；
5. 根据新Kubernetes版本的API变化，调整**manifests**清单文件和其他资源。

# 准备工作
需要注意的事项主要有：
1. 必须禁用交换分区；
2. 升级之前必须备份所有组件和数据，例如etcd；
3. 不要跨多个小版本升级，例如直接从1.19升级到1.24（要考虑版本之间的兼容性）；
4. 在测试或演练环境多次实操后，才能上生产环境；
5. 由于`1.24`版本将移除dockershim，如果当前集群使用的容器运行时是docker，升级kubelet时可能会报错，可以将docker替换为containerd。

禁用交换分区的办法如下：
```bash
# 检查交换分区使用的设备
cat /proc/swaps

# 关闭交换分区
swapoff -a

# 永久禁用交换分区
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

查看当前集群的k8s版本：
```bash
[root@k8s-master ~]# kubectl version
Client Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.0", GitCommit:"ab69524f795c42094a6630298ff53f3c3ebab7f4", GitTreeState:"clean", BuildDate:"2021-12-07T18:16:20Z", GoVersion:"go1.17.3", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.0", GitCommit:"ab69524f795c42094a6630298ff53f3c3ebab7f4", GitTreeState:"clean", BuildDate:"2021-12-07T18:09:57Z", GoVersion:"go1.17.3", Compiler:"gc", Platform:"linux/amd64"}
```

查看最新的补丁版本，确定要升级到的k8s版本：
```bash
yum list --showduplicates kubeadm --disableexcludes=kubernetes
```

下面我们尝试使用kubeadm将k8s集群从`1.23.0`版本升级到`1.24.11`版本。

# 升级控制平面节点
## 升级kubeadm
升级的第一个控制平面节点上必须有`/etc/kubernetes/admin.conf`文件。

- 升级kubeadm：
```bash
yum install -y kubeadm-1.24.11-0 --disableexcludes=kubernetes
```

- 验证kubeadm版本：
```bash
kubeadm version
```

- 验证升级计划。检查当前集群是否可被升级，并获取要升级的目标版本。
```bash
kubeadm upgrade plan
```
`kubeadm upgrade`命令也会自动对kubeadm在节点上所管理的证书执行续约操作。如果需要略过证书续约操作，可以使用标志`--certificate-renewal=false`。

- 升级到指定的目标版本：
```bash
sudo kubeadm upgrade apply v1.24.11
```

- 手动升级CNI插件。

容器网络接口（CNI）驱动提供了程序自身的升级说明。如果CNI插件作为DaemonSet运行，则在其他控制平面节点上不需要此步骤。

>:coffee:**kubeadm upgrade apply**做了以下工作：
>- 检查集群是否处于可升级状态:
>   - API服务器是可访问的；
>   - 所有节点处于Ready状态；
>   - 控制面是健康的。
>- 强制执行版本偏差策略。
>- 确保控制面的镜像是可用的或可拉取到服务器上。
>- 如果组件配置要求版本升级，则生成替代配置与（或）使用用户提供的覆盖版本配置。
>- 升级控制面组件或回滚（如果其中任何一个组件无法启动）。
>- 应用新的CoreDNS和kube-proxy清单，并强制创建所有必需的RBAC规则。
>- 如果旧文件在180天后过期，将创建API服务器的新证书和密钥文件并备份旧文件。


升级其他控制平面节点与第一个控制面节点相同，但是使用下面的命令，而不是`kubeadm upgrade apply`命令。
```bash
sudo kubeadm upgrade node
```
并且不需要执行`kubeadm upgrade plan`和更新CNI驱动插件的操作。

>:coffee:**kubeadm upgrade node**在其他控制平节点上执行以下操作：
>- 从集群中获取kubeadm的ClusterConfiguration。
>- （可选操作）备份kube-apiserver证书。
>- 升级控制平面组件的静态Pod清单。
>- 为本节点升级kubelet配置。


## 暂停节点上的调度
将当前节点标记为不可调度，并驱逐节点上的Pod：
```bash
kubectl drain <节点名称> --ignore-daemonsets
```

## 升级kubelet和kubectl
升级kubelet和kubectl组件：
```bash
yum install -y kubelet-1.24.11-0 kubectl-1.24.11-0 --disableexcludes=kubernetes
```

重启kubelet：
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

## 恢复节点上的调度
恢复当前节点上的Pod调度，使其上线：
```bash
kubectl uncordon <节点名称>
```

# 升级工作节点
工作节点上的升级过程应该一次执行一个节点，或者一次执行几个节点，以不影响运行工作负载所需的最小容量。

## 升级kubeadm
```bash
yum install -y kubeadm-1.24.11-0 --disableexcludes=kubernetes
```

在工作节点上，执行下面的命令来升级本地的kubelet配置：
```bash
sudo kubeadm upgrade node
```

>:coffee:**kubeadm upgrade node**在工作节点上完成以下工作：
>- 从集群获取kubeadm的ClusterConfiguration。
>- 为本节点升级kubelet配置。


## 暂停节点上的调度
将节点标记为不可调度并驱逐所有负载：
```bash
kubectl drain <节点名称> --ignore-daemonsets
```

## 升级kubelet和kubectl
升级kubelet和kubectl组件：
```bash
yum install -y kubelet-1.24.11-0 kubectl-1.24.11-0 --disableexcludes=kubernetes
```

重启kubelet：
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

## 恢复节点上的调度
将节点标记为可调度，让节点重新上线：
```bash
kubectl uncordon <节点名称>
```

# 验证集群状态
在所有节点上升级kubelet后，在控制平面节点运行以下命令，验证所有节点是否可用：
```bash
kubectl get nodes
```
**STATUS**应显示所有节点为**Ready**状态，并且版本号已经被更新。


# 从故障状态恢复
如果`kubeadm upgrade`失败并且没有回滚，例如由于执行期间节点意外关闭，我们可以再次运行`kubeadm upgrade`。该命令是**幂等**的，并最终确保实际状态是声明的期望状态。

要从故障状态恢复，还可以运行`kubeadm upgrade apply --force`而无需更改集群正在运行的版本。

在升级期间，kubeadm向`/etc/kubernetes/tmp`目录下的如下备份文件夹写入数据：
```
kubeadm-backup-etcd-<date>-<time>
kubeadm-backup-manifests-<date>-<time>
```

- **kubeadm-backup-etcd**包含当前控制面节点本地etcd成员数据的备份。如果etcd升级失败并且自动回滚也无法修复，则可以将此文件夹中的内容复制到`/var/lib/etcd`进行手工修复。如果使用的是外部的etcd，则此备份文件夹为空。

- **kubeadm-backup-manifests**包含当前控制面节点的静态Pod清单文件的备份版本。如果升级失败并且无法自动回滚，则此文件夹中的内容可以复制到`/etc/kubernetes/manifests`目录实现手工恢复。如果由于某些原因，在升级前后某个组件的清单未发生变化，则kubeadm也不会为之生成备份版本。




**References**
[1] https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/cluster-upgrade/
[2] https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/
[3] https://v1-24.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/
[4] https://serverfault.com/questions/684771/best-way-to-disable-swap-in-linux


