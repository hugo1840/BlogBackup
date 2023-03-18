---
tags: [k8s]
title: K8S容器运行时从Docker切换为Containerd
created: '2023-03-18T01:07:09.534Z'
modified: '2023-03-18T02:00:15.831Z'
---

K8S容器运行时从Docker切换为Containerd

K8S从1.24版本起不再支持docker容器引擎，可选的替代品有`containerd`、`cri-o`、`podman`。下面演示将单个node节点的容器引擎从docker切换为containerd的过程。

# 检查内核参数与模块
## overlay和br_netfilter
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

## 内核网络参数
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

# containerd安装配置
## 安装containerd
安装containerd相关依赖包：
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

安装containerd：
```bash
ls /etc | grep containerd

# 如果没有安装
yum install -y containerd.io 
```

修改containerd为独立运行时的默认配置：
```bash
containerd config default > /etc/containerd/config.toml
```

## 修改镜像仓库地址
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
```

保存退出后，重启服务生效：
```bash
systemctl restart containerd

# 启用开机自启
systemctl enable containerd
```

# 切换容器运行时
修改kubelet配置，将当前节点的默认容器运行时修改为containerd：
```bash
vim /etc/sysconfig/kubelet 
KUBELET_EXTRA_ARGS=--container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock --cgroup-driver=systemd 

# 保存退出，重启生效
systemctl restart kubelet
```

在k8s-master检查：
```bash
[root@k8s-master1 ~]# kubectl get nodes -o wide
NAME          STATUS   ROLES                  AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
k8s-master1   Ready    control-plane,master   3d11h   v1.23.0   192.168.x.x   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   containerd://1.6.18
k8s-master2   Ready    control-plane,master   3d10h   v1.23.0   192.168.x.x   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   containerd://1.6.18
k8s-master3   Ready    control-plane,master   3d10h   v1.23.0   192.168.x.x   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   containerd://1.6.18
k8s-worker1   Ready    <none>                 3d10h   v1.23.0   192.168.x.x   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   containerd://1.6.18
k8s-worker2   Ready    <none>                 3d10h   v1.23.0   192.168.x.x   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   containerd://1.6.18

```
输出中，`CONTAINER-RUNTIME`一列，k8s-node2的容器运行时已变成containerd，其余节点还是docker。

最后停用docker服务：
```bash
systemctl disable docker && systemctl stop docker
```

# crictl管理工具
Containerd可以通过crictl命令来管理容器。

配置crictl管理containerd：
```bash
vi /etc/crictl.yaml 
runtime-endpoint: unix:///run/containerd/containerd.sock 
image-endpoint: unix:///run/containerd/containerd.sock 
timeout: 10 
debug: false
```

查看crictl常用命令：
```bash
crictl --help
```




