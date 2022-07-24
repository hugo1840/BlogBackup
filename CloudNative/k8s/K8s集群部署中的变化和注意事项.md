@[TOC](K8s集群部署中的变化和注意事项)

详细部署过程参考之前的文章：
>[kubernetes群集部署与测试](https://blog.csdn.net/Sebastien23/article/details/113757356)

本文主要介绍部署过程中发生的一些新的变化、以及一些可能遇到问题的解决办法。


# 内核参数
## net.bridge.bridge-nf-call-iptables
```bash
vi /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
#保存退出
sysctl --system
```

K8S中最常遇到的问题之一就是网络通信问题。关于`net.bridge.bridge-nf-call-iptables`为什么要开启，请参考文章：
>[为什么 kubernetes 环境要求开启 bridge-nf-call-iptables](https://blog.csdn.net/sanwe3333/article/details/122624447?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-122624447-blog-89292974.pc_relevant_multi_platform_whitelistv2_ad_hc&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1-122624447-blog-89292974.pc_relevant_multi_platform_whitelistv2_ad_hc&utm_relevant_index=1)

以及官方文档：
>[Network Plugin Requirements](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#network-plugin-requirements)

# 配置阿里云源
由于众所周知的原因，需要使用国内的源来加速镜像下载。这里我们一般选择配置阿里云的源。

## 使用阿里云源下载docker镜像
```bash
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo \
-O /etc/yum.repos.d/docker-ce.repo

# cgroup驱动使用systemd
cat > /etc/docker/daemon.json << EOF
{
   "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"],
   "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

systemctl restart docker 
```

检查修改后的配置
```bash
docker info
docker info | grep Cgroup
docker info | grep Registry
```

## 为k8s配置阿里云的源
```bash
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enable=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

# kubeadm一键部署集群
```bash
kubeadm init \
--apiserver-advertise-address=192.168.136.96 \  
--image-repository registry.aliyuncs.com/google_containers \ 
--kubernetes-version v1.23.0 \  
--service-cidr=10.96.0.0/12 \  
--pod-network-cidr=10.244.0.0/16 \
--ignore-preflight-errors=all
```

其中：
- `--apiserver-advertise-address`：集群通告地址；
- `--image-repository`：默认镜像仓库地址；
- `--service-cidr`：集群内部虚拟网络，Pod统一访问入口；
- `--pod-network-cidr`：Pod网络，与CNI网络组件yaml文件中保持一致。


在Node1节点使用root用户执行`kubeadm join`命令可以加入集群。如果token失效，执行下面的命令来生成新的token：
```bash
kubeadm token create --print-join-command
```

# 使用网络插件Calico
在master节点部署网络插件**calico**来替代以前的**flannel**插件。

```bash
wget https://docs.projectcalico.org/manifests/calico.yaml
#或
wget https://docs.projectcalico.org/manifests/calico.yaml --no-check-certificate
```
取消`calico.yaml`中下面两行的注释，并将value配置为`kubeadm init`命令初始化集群时`pod-network-cidr`对应的网段。
```yaml
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"
```

使配置文件生效
```bash
kubectl apply -f calico.yaml 
kubectl get pods -n kube-system
```

# 切换容器引擎为containerd
K8S最新版本不再支持docker容器引擎，可选的替代品有`containerd`、`cri-o`、`podman`。下面演示将单个node节点的容器引擎从docker切换为containerd的过程。

## 内核参数与模块
如果安装了docker-ce，可以跳过该小节内容，因为其中的配置在安装docker时已经完成了。

检查是否已经加载内核模块`overlay`和`br_netfilter`。
```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```

如果没有，手动加载内核模块：
```bash
cat <<EOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

# 加载内核模块
modprobe overlay
modprobe br_netfilter
```

检查系统内核参数：
```bash
sysctl -a | grep bridge
sysctl -a | grep ip_forward
```

如果没有开启，手动调整：
```
cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
```

## containerd安装配置
在k8s-node2执行：
```bash
yum install -y yum-utils device-mapper-persistent-data lvm2 
```

配置docker源（如果已配置，跳过此步骤）：
```bash
ls /etc/yum.repos.d | grep docker-ce
yum-config-manager \ 
--add-repo \ 
https://download.docker.com/linux/centos/docker-ce.repo 
```

安装containerd（如果已安装dokcer，那么containerd也已经一起安装好了，可以跳过该步骤）：
```bash
ls /etc | grep containerd
# 如果没有安装containerd
yum install -y containerd.io 
mkdir -p /etc/containerd 
```

修改containerd为独立运行时的默认配置：
```bash
containerd config default > /etc/containerd/config.toml
```

修改containerd配置文件：
```bash
vim /etc/containerd/config.toml

#pause镜像地址修改为阿里云镜像仓库地址
[plugins."io.containerd.grpc.v1.cri"] 
  sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.2"

#cgroups驱动引擎修改为systemd
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options] 
  SystemdCgroup = true

#Dcoker Hub镜像仓库地址修改为阿里云镜像仓库地址（非必须）
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"] 
  endpoint = ["https://b9pmyelo.mirror.aliyuncs.com"]

# 保存退出，重启服务生效
systemctl restart containerd
```

## kubelet配置
在k8s-node2修改kubelet配置，将当前节点的默认容器运行时修改为containerd：
```bash
vim /etc/sysconfig/kubelet 
KUBELET_EXTRA_ARGS=--container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock --cgroup-driver=systemd 

# 保存退出，重启生效
systemctl restart kubelet
```

在k8s-master检查：
```bash
kubectl get node -o wide
```
输出中，`CONTAINER-RUNTIME`一列，k8s-node2的容器运行时已变成containerd，其余节点还是dcoker。

## crictl工具
Containerd等其他兼容CRI的容器引擎可以通过`crictl`命令来管理容器。

配置crictl工具管理containerd：
```bash
vi /etc/crictl.yaml 
runtime-endpoint: unix:///run/containerd/containerd.sock 
image-endpoint: unix:///run/containerd/containerd.sock 
timeout: 10 
debug: false
```

查看本节点运行的容器（类似`docker ps`）:
```
crictl ps
crictl --help
```

## 切回docker引擎
如果要切回使用原来的docker引擎，只需取消kubelet配置，重启服务即可。
```bash
cat /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS=

# 保存退出，重启生效
systemctl restart kubelet
```

# 常见问题排查
查看节点的kubelet日志：
```bash
journalctl -u kubelet -f
```

获取dashboard登录用户token:
```bash
# cluster-admin管理员用户为dashboard-admin
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```

## dashboard配置文件无法获取
```bash
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml
```
执行时报错。需要配置hosts文件：
```bash
echo '199.232.68.133 raw.githubusercontent.com' >> /etc/hosts
```

## 重启服务器后kubectl命令报错
```bash
[root@k8s-master ~]# kubectl get node -o wide
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

原因是环境变量`KUBECONFIG`未生效。临时解决办法为：
```bash
[root@k8s-master ~]# export KUBECONFIG=/etc/kubernetes/admin.conf
[root@k8s-master ~]# kubectl get node -o wide
NAME         STATUS   ROLES                  AGE   VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
k8s-master   Ready    control-plane,master   27h   v1.23.0   192.168.136.96   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://20.10.17
k8s-node1    Ready    <none>                 27h   v1.23.0   192.168.136.97   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://20.10.17
k8s-node2    Ready    <none>                 27h   v1.23.0   192.168.136.98   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   containerd://1.6.6
```

一劳永逸的办法（root用户为例）：
```bash
cd /root
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> .bash_profile
source /root/.bash_profile
```
