@[TOC](kubernetes群集部署与测试)

# 基本概念
| 组件 | 作用 |
| -- | --|
| Pod | 最小部署单元； 一组容器的集合； 同一个Pod中的容器共享网络命名空间； Pod的生命周期是短暂的 |
| Controlers | ReplicaSet：确保预期的Pod副本数量； Deployment：无状态应用部署； StatefulSet：有状态应用部署； <br>DaemonSet：确保所有Node运行同一个Pod； Job：一次性任务；Cronjob：定时任务 |
| Service | 防止Pod失联；- 定义一组Pod的访问策略（负载均衡服务发现） |
| Label | 附加到某个资源上的标签，用于关联对象、查询和筛选 |
| Namespaces | 命名空间，将对象从逻辑上隔离 |


# Kubernetes部署
Kubernetes有四种安装部署方式：kubeadm、kops、kubespray和二进制安装。其中，kubeadm将大部分组件都放在容器中启动，因此相比二进制安装方式而言消耗的资源更少，是目前官方推荐的部署方式。Kops主要用于在AWS上部署kubernetes集群；Kubespray则支持在GCE、Azure、OpenStack、AWS、vSphere、Packet、Oracle Cloud Infrastructure等多个云平台上部署。

## Kubeadm部署
### 安装要求
-	操作系统：Ubuntu 16.04+、Debian 9+、CentOS 7+、RHEL 7+、Fedora 25+、HypriotOS v1.0.1+等；
-	内存：至少2GB；
-	CPU：至少2 CPUs；
-	集群内所有机器网络互通；
-	每个节点都有独一无二的hostname、MAC地址和product_uuid；
-	开放某些特定端口（6643*、2379-2380、10250-10252、30000-32767）；
-	禁用swap分区。


检查MAC地址：`ip link` 或 `ifconfig -a`
检查product_uuid：`sudo cat /sys/class/dmi/id/product_uuid`
需要用到的端口及对应组件信息：参见`https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports`


### 环境准备（所有节点）
假设群集中所有节点的主机名和IP地址信息如下。
k8s-master	 192.168.30.128
k8s-node1	 192.168.30.129
k8s-node2	 192.168.30.130
k8s-node3	 192.168.30.131

在所有节点关闭防火墙
```bash
$ systemctl stop firewalld  # 暂时关闭
$ systemctl disable firewalld  # 永久关闭
```

在所有节点关闭selinux
```bash
$ sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久关闭
$ setenforce 0  # 暂时关闭
```

在所有节点关闭swap
```bash
$ swapoff -a  # 暂时关闭
$ sed -ri 's/.*swap.*/#&/' /etc/fstab  # 永久关闭
```

分别配置主机名
```bash
$ hostnamectl set-hostname k8s-master  # 在master节点执行
$ hostnamectl set-hostname k8s-node1  # 在node1节点执行
$ hostnamectl set-hostname k8s-node2  # 在node2节点执行
$ hostnamectl set-hostname k8s-node3  # 在node3节点执行
```

在master节点修改hosts文件
```bash
$ cat >> /etc/hosts << EOF
> 192.168.30.128 k8s-master
> 192.168.30.129 k8s-node1
> 192.168.30.130 k8s-node2
> 192.168.30.131 k8s-node3
> EOF
```

在所有节点使用iptables控制网络流量
```bash
$ cat > /etc/sysctl.d/k8s.conf << EOF
> net.bridge.bridge-nf-call-ip6tables = 1
> net.bridge.bridge-nf-call-iptables = 1
> EOF
$ sysctl --system  # 配置生效
```

在所有节点做时间同步
```bash
$ yum install ntpdate -y
$ ntpdate time.windows.com  # 同步时间
$ cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime  # 设置时区
```

### 安装docker和k8s组件（所有节点）
在所有节点安装docker
```bash
$ yum install -y wget
$ wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo \
-O /etc/yum.repos.d/docker-ce.repo
$ yum -y install docker-ce-18.06.1.ce-3.el7  # 与k8s兼容的版本
$ systemctl enable docker && systemctl start docker
$ docker --version
Docker version 18.06.1-ce, build e68fc7a
```

为Docker配置阿里云镜像仓库（加快镜像拉取速度）
```bash
$ cat > /etc/docker/daemon.json << EOF
> {
> "registry-mirrors" : ["https://b9pmyelo.mirror.aliyuncs.com"]
> }
> EOF
```

配置阿里云yum软件源
```bash
$ cat > /etc/yum.repos.d/kubernetes.repo << EOF
> [kubernetes]
> name=Kubernetes
> baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
> enable=1
> gpgcheck=0
> repo_gpgcheck=0
> gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
> https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
> EOF
```

在所有节点安装kubeadm、kubectl和kubelet
```bash
$ yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0 
$ systemctl enable kubelet
```

### 部署master节点（master节点）
使用kubeadm引导master节点
```bash
$ kubeadm init \
--apiserver-advertise-address=192.168.30.128 \  # API 服务器所公布的其正在监听的 IP 地址
--image-repository registry.aliyuncs.com/google_containers \  # 选择用于拉取控制平面镜像的容器仓库
--kubernetes-version v.1.18.0 \  # 为控制平面选择一个特定的 Kubernetes 版本
--service-cidr=10.96.0.0/12 \  # 为服务的虚拟 IP 地址另外指定 IP 地址段
--pod-network-cidr=10.244.0.0/16   # 指明 pod 网络可以使用的 IP 地址段
```

其他选项参见：https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/

执行完成并提示successful后，切换到集群用户和执行以下指令（也可以直接在root用户下执行）：
```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 安装Pod网络插件（master节点）
为了不同节点上的Pod能够相互通信，需要安装CNI网络插件。
```bash
$ wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
如果获取失败，修改/etc/hosts文件，在末尾添加下面一行内容后重试：
`199.232.68.133 raw.githubusercontent.com`

### Node加入集群（worker节点）
Master节点上kubeadm init执行成功后会提示node节点加入集群的指令，直接复制后依次在每个node节点粘贴执行即可。一般格式如下：
```bash
$ kubeadm join 192.168.66.128:6443 --token diz4py.s7dhmaks8e \
--discovery-token-ca-cert-hash sha256:83hjdj8u4fjb30jw2exb76b32wqe
```

查看节点信息：`kubectl get node`
查看Pod信息：`kubectl get pods`
查看系统组件：`kubectl get pods -n kube-system`

### 部署Dashboard（master节点）
在master节点配置集群管理页面
```bash
$ kubectl apply -f \
https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommend.yaml
```
执行`kubectl get pods -n kubernetes-dashboard`查看命名空间中运行的容器。

默认dashboard只能集群内部访问，修改`dashboard.yaml`配置文件中的Service为NodePort类型，暴露到外部。
```bash
kind: Service
apiVersion: v1
metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard
   namespace: kubernetes-dashboard
spec:
  type: NodePort  # 添加这一行
  ports:
- port: 443
  targetPort: 8443
  nodePort: 30001  # 添加这一行
  selector:
    k8s-app: kubernetes-dashboard
```
运行`kubectl apply -f dashboard.yaml`使修改的配置文件生效。

创建Service account并绑定默认cluster-admin群集管理员角色。
```bash
$ kubectl create serviceaccount dashboard-admin -n kube-system
$ kubectl create clusterrolebinding dashboard-admin \
--clusterrole=cluster-admin \
--serviceaccount=kube-system:dashboard-admin
$ kubectl describe secrets \
-n kebu-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
# 会输出一串token
```

访问`https://NodeIP:30001`查看kubernetes的Web UI（注意是HTTPS，由于证书无效可能需要添加安全例外）。使用上一步中输出的token登录dashboard。在Web UI中可以查看Pod运行状态和日志、进入Shell执行命令。

## 测试k8s群集
在master节点执行`kubectl get pods -n kube-system`，确认所有pods的状态都是Running。然后执行`kubectl get nodes`，确认所有节点的状态都是Ready。则表示集群已经配置成功。

通过部署nginx Pod测试集群，在master节点执行
```bash
$ kubectl create deployment nginx --image=nginx
$ kubectl expose deployment nginx --port=80 --type=NodePort
$ kubectl get pod,svc
```
查看输出结果中service/nginx对应的TCP端口（80:后面接的数字，比如80:32753）。在浏览器中访问`http://NodeIP:Port`，其中NodeIP可以是任意一个node节点的IP地址，比如192.168.30.129:32753，192.168.30.130:32753，或者192.168.30.131:32753。也可以在dashboard中查看Pod的状态，参考上一小节内容。

