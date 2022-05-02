@[TOC](k8s单master节点集群搭建：使用kubeadm)
# 架构
Kubernetes单master群集仅包含一个master节点和三个node节点（或者worker节点）。条件允许的话，可以搭建多master节点群集，将master节点的数量增加到三个。

**安装要求**
-	多台已安装CentOS7.x-x86_64操作系统的Linux虚机；
-	硬件配置：每台虚机至少2GB内存，至少2个CPU，master节点20G硬盘或更多，Node节点40G硬盘或更多；
-	集群内所有机器之间网络互通；
-	可以访问外网，以拉取镜像，否则需要预先下载好所需镜像。

# 环境准备（所有节点）
## 配置网卡和静态IP
以VMware为例，创建完四台虚机后，在虚拟机设置里将网络适配器设置为**NAT模式**。在编辑菜单里打开**虚拟网络编辑器**，选择Vmnet8（NAT模式），点击右下角的更改设置，然后打开**NAT设置**，记住**网关IP**和子网掩码，点击确认。然后打开**DHCP设置**，记住起始IP地址，点击确认。最后点击确认关闭虚拟网络编辑器。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210122083000345.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)


依次在master节点和三个node节点中执行以下操作。
-	在终端中执行`dhclient`指令获取IP地址。然后使用`ip addr`指令查看ens33网卡绑定的IP地址（inet后面的地址）；
-	执行`vi /etc/sysconfig/network-scripts/ifcfg-ens33`打开网卡配置文件，将`ONBOOT=no`改为`ONBOOT=yes`（开机启动网卡），`BOOTPROTO=dhcp`改为`BOOTPROTO=static`（静态IP）。
-	在第二步打开的网卡配置文件中加入以下几行内容：
`IPADDR=192.168.30.128`  # 第一步中获取的IP地址
`NETMASK=255.255.255.0`  # NAT设置中的子网掩码
`GATEWAY=192.168.30.2`  # NAT设置中的网关地址
`DNS1=119.29.29.29`  # DNS服务器地址
-	最后执行`systemctl restart network.service`重启网络服务。

 
全部配置完成以后，可以执行`ping www.baidu.com`查看是否能访问外网，然后ping其他虚机的IP地址查看彼此之间网络是否连通。

## 关闭防火墙

```bash
$ systemctl stop firewalld  # 暂时关闭
$ systemctl disable firewalld  # 永久关闭
```

## 关闭selinux

```bash
$ sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久关闭
$ setenforce 0  # 暂时关闭
```

## 关闭swap

```bash
$ swapoff -a  # 暂时关闭
$ sed -ri 's/.*swap.*/#&/' /etc/fstab  # 永久关闭
```

## 配置主机名

```bash
$ hostnamectl set-hostname k8s-master  # 在master节点执行
$ hostnamectl set-hostname k8s-node1  # 在node1节点执行
$ hostnamectl set-hostname k8s-node2  # 在node2节点执行
$ hostnamectl set-hostname k8s-node3  # 在node3节点执行
```

## 修改hosts文件
仅在master节点执行：

```bash
$ cat >> /etc/hosts << EOF
> 192.168.30.128 k8s-master
> 192.168.30.129 k8s-node1
> 192.168.30.130 k8s-node2
> 192.168.30.131 k8s-node3
> EOF
```


## 使用iptables控制网络流量

```bash
$ cat > /etc/sysctl.d/k8s.conf << EOF
> net.bridge.bridge-nf-call-ip6tables = 1
> net.bridge.bridge-nf-call-iptables = 1
> EOF
$ sysctl --system  # 配置生效
```


## 时间同步

```bash
$ yum install ntpdate -y
$ ntpdate time.windows.com  # 与Windows宿主机同步时间
```


至此，群集中所有节点的主机名和IP地址信息如下。
|主机| IP地址 |
|--|--|
| k8s-master | 192.168.30.128 |
| k8s-node1 | 192.168.30.129 |
| k8s-node2 | 192.168.30.130 |
| k8s-node3 | 192.168.30.131 |


# 安装docker和k8s组件（所有节点）
## 安装docker

```bash
$ yum install -y wget
$ wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
$ yum -y install docker-ce-18.06.1.ce-3.el7
$ systemctl enable docker && systemctl start docker
$ docker --version
Docker version 18.06.1-ce, build e68fc7a
$ cat > /etc/docker/daemon.json << EOF
> {
> "registry-mirrors" : ["https://b9pmyelo.mirror.aliyuncs.com"]
> }
> EOF
```


## 添加阿里云yum源

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

## 安装kubeadm/kubelet/kubectl

```bash
$ yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0 
$ systemctl enable kubelet
```

# Master节点部署

```bash
$ kubeadm init \
--apiserver-advertise-address=192.168.30.128 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v.1.18.0 \
--service-cidr=10.96.0.0/12 \  # 不与其他IP冲突即可
--pod-network-cidr=10.244.0.0/16   # 不与其他IP冲突即可
```

该指令的执行时间比较长，会自动从阿里云镜像仓库拉取kube-proxy、kube-apiserver、kube-control-manager、kube-scheduler、coredns、etcd等组件。运行docker image指令可以查看已经拉取的镜像。执行完成并提示successful后，切换到集群用户和执行以下指令（也可以直接在root用户下执行）：

```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


# Node节点部署
上一步中，执行成功后会提示node节点加入集群的指令，直接复制后依次在每个node节点粘贴执行即可。一般格式如下：

```bash
$ kubeadm join 192.168.66.128:6443 --token diz4py.s7dhmaks8e \
--discovery-token-ca-cert-hash sha256:83hjdj8u4fjb30jw2exb76b32wqe
```

该指令的执行时间一般也比较长，如果出错的话可以多试几次。默认token有效期为24小时，过期之后需要重新创建：
`$ kubeadm token create --print-join-command`

在master节点运行kubectl get nodes可以查看所有节点状态，如果状态都为Ready则表示节点加入群集成功。

# 部署CNI网络插件
最后要部署容器网络插件。在master节点运行：

```bash
$ wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

如果获取失败，修改/etc/hosts文件，在末尾添加下面一行内容后重试：
`199.232.68.133 raw.githubusercontent.com`

# k8s群集测试
在master节点执行`kubectl get pods -n kube-system`，确认所有pods的状态都是**Running**。然后执行`kubectl get nodes`，确认所有节点的状态都是**Ready**。则表示集群已经配置成功。

下面开始测试，在集群中新建一个pod，并检查能否正常运行。

```bash
$ kubectl create deployment nginx --image=nginx
$ kubectl expose deployment nginx --port=80 --type=NodePort
$ kubectl get pod,svc
```

查看输出结果中`service/nginx`对应的TCP端口（80:后面接的数字，比如`80:32753`）。在浏览器中访问`http://NodeIP:Port`，其中NodeIP可以是任意一个node节点的IP地址，比如192.168.30.129:32753，192.168.30.130:32753，或者192.168.30.131:32753。若能成功访问则表示测试成功。


