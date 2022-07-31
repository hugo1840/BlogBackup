@[TOC](静态Pod、Pod创建流程、容器资源限制)

# 静态Pod
静态容器具有以下特点：
- Pod由特定节点上的kubelet管理；
- 不能使用控制器；
- Pod名称标识当前节点名称。

在kubelet配置文件中启用静态Pod的参数：

```bash
vi /var/lib/kubelet/config.yaml
...
staticPodPath: /etc/kubernetes/manifests
```
将部署的pod.yaml放到该目录下会由kubelet自动创建。

K8S组件中除了kubelet以外都是以容器形式运行的。在K8S集群搭建好之前，只能通过上面的方式由kubelet创建静态Pod。

Master节点部署的组件有`api-server`、`controller-manager`、`scheduler`和`etcd`；Node节点部署的组件有`kube-proxy`、`kubelet`和`docker`。

```bash
[root@k8s-master ~]# kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS          AGE
calico-kube-controllers-5c64b68895-kthkh   1/1     Running   4 (6h16m ago)     8d
calico-node-6mxfm                          1/1     Running   4 (6h16m ago)     8d
calico-node-nzn8p                          1/1     Running   150 (6h16m ago)   8d
calico-node-vg6s9                          1/1     Running   4 (6h16m ago)     8d
coredns-6d8c4cb4d-k9245                    1/1     Running   4 (6h16m ago)     8d
coredns-6d8c4cb4d-xzl49                    1/1     Running   4 (6h16m ago)     8d
etcd-k8s-master                            1/1     Running   4 (6h16m ago)     8d
kube-apiserver-k8s-master                  1/1     Running   4 (6h16m ago)     8d
kube-controller-manager-k8s-master         1/1     Running   4 (6h16m ago)     8d
kube-proxy-58kfs                           1/1     Running   5 (6h16m ago)     8d
kube-proxy-75z28                           1/1     Running   4 (6h16m ago)     8d
kube-proxy-f9522                           1/1     Running   4 (6h16m ago)     8d
kube-scheduler-k8s-master                  1/1     Running   4 (6h16m ago)     8d
metrics-server-798c598bb8-j69cc            1/1     Running   3 (6h16m ago)     6d19h

[root@k8s-master ~]# kubectl get deploy -n kube-system
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
calico-kube-controllers   1/1     1            1           8d
coredns                   2/2     2            2           8d
metrics-server            1/1     1            1           6d19h
```

可以发现，`etcd-k8s-master`、`kube-apiserver-k8s-master`、`kube-controller-manager-k8s-master`、`kube-scheduler-k8s-master`都是以master节点的名称结束的，并且没有对应的控制器。

实际上，master节点的这四个组件都是由本节点kubelet直接管理的静态Pod部署。
```bash
[root@k8s-master ~]# grep staticPod /var/lib/kubelet/config.yaml
staticPodPath: /etc/kubernetes/manifests
[root@k8s-master ~]#
[root@k8s-master ~]# ls /etc/kubernetes/manifests/
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

# Pod创建流程
k8s基于list-watch机制的控制器架构，实现组件间交互的解耦。其他组件监控自己负责的资源，当这些资源发生变化时，`kube-apiserver`会通知这些组件，这个过程类似于**发布**与**订阅**。

从我们发出命令开始，一个Pod被创建的大致流程如下：

1. 所有使用`kubectl`发出的命令都会发送给`api-server`，后者接收后将其写入`etcd`中。
2. `etcd`写入成功即向`api-server`返回写入成功的消息，否则会返回错误信息。
3. `api-server`通知调度器`scheduler`把要创建的Pod分配到合适的节点上。
4. `scheduler`选择好目标节点后，打上标记，并将结果上报给`api-server`。
5. `api-server`收到`scheduler`反馈的Pod-Node绑定信息后，将其写入`etcd`。
6. `api-server`通知目标node节点上的`kubelet`组件创建Pod。
7. 目标节点上的`kubelet`组件调用docker或者其他容器引擎来创建容器，并将创建结果信息上报给`api-server`。
8. `api-server`将收到的Pod创建结果信息写入`etcd`。

由于创建一个单独的Pod时不会使用到控制器，所以上面的过程中没有涉及到`controller-manager`。此外，也没有创建Service，所以也不涉及到`kube-proxy`组件。


# 容器资源限制
通过以下两个属性来限制容器能够使用的CPU和内存资源的**上限**：
- `resources.limits.cpu`
- `resources.limits.memory`

通过以下两个属性来设置容器使用的最小资源需求（**下限**），作为容器调度时资源分配的依据：
- `resources.requests.cpu`
- `resources.requests.memory`

k8s会根据requests的值去寻找有足够可用资源的节点来调度Pod。

注意事项：
- 没有配置资源限制时，容器可以使用所在节点的所有资源。
- limits一般低于节点配置的20%。
- requests的值必须小于limits，一般低于limits的20%到30%。
- 只设置limits限制时，requests默认也会采用limits的值。
- requests是预留性质的资源，容器运行时实际使用的资源可能小于requests。
- 如果requests配置过高，就会导致节点分配的Pod过少，会导致资源空闲。
- 如果requests配置过低，就会导致节点分配的Pod过多，可能会导致资源饱和。
- CPU资源计量单位为毫核，1核CPU=1000m，0.5=500m。

查看当前节点的资源分配情况（Allocated resources）:
```bash
kubectl describe node k8s-node1
```



