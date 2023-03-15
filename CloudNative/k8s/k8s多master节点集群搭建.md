---
tags: [k8s]
title: k8s多master节点集群搭建
created: '2023-03-15T13:41:37.296Z'
modified: '2023-03-15T14:13:21.850Z'
---

k8s多master节点集群搭建

# 单master节点集群搭建
首先搭建一个单master节点的集群，具体可以参考博文[k8s单master节点集群搭建：使用kubeadm](https://blog.csdn.net/Sebastien23/article/details/112976697)和[kubernetes群集部署与测试](https://blog.csdn.net/Sebastien23/article/details/113757356)、以及[K8s集群部署中的变化和注意事项](https://blog.csdn.net/Sebastien23/article/details/125958860)。

# 添加多个master节点
参照集群中的第一个master节点，对后续要加入集群的master节点做好`kubeadm init`命令之前的所有环境配置工作（操作系统参数、网络配置、镜像仓库、yum源等）。无需单独在这些后续加入的master节点上安装k8s网络插件。

在还未加入集群的master节点上创建目录：
```bash
mkdir -p /etc/kubernetes/pki/etcd
```

将第一个master节点上的下列文件拷贝到还未加入集群的master节点上：
```bash
scp /etc/kubernetes/admin.conf root@{k8s_master02_IP}:/etc/kubernetes/
scp /etc/kubernetes/pki/ca.* root@{k8s_master02_IP}:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.* root@{k8s_master02_IP}:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.* root@{k8s_master02_IP}:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/etcd/ca.* root@{k8s_master02_IP}:/etc/kubernetes/pki/etcd/
```

注意不能多拷贝其他文件，否则在加入集群时可能会收到下面的报错：
```bash
error execution phase control-plane-prepare/certs: 
error creating PKI assets: failed to write or validate certificate "etcd-peer": 
certificate etcd/peer is invalid: x509: certificate is valid for k8s-master01, localhost, not k8s-master02
```

在第一个master节点上打印出加入集群的命令：
```bash
kubeadm token create --print-join-command
```

在还未加入集群的master节点上执行上面打印出来的命令来加入集群，注意加上`--control-plane`来表明是以管理节点的身份加入。
```bash
kubeadm join {k8s_master01_IP}:6443 --token xxxxxx --discovery-token-ca-cert-hash sha256:xxxxxx --control-plane
```

最后检查加入集群的master节点状态是否READY。
```bash
kubectl get nodes
```

# 可能遇到的错误
如果收到下面的报错信息：
```bash
error execution phase preflight: [preflight] Some fatal errors occurred:
        [ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
```

修改对应的操作系统参数，再重新加入master节点即可。
```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

如果收到下面的报错信息：
```bash
unable to add a new control plane instance a cluster that doesn't have a stable controlPlaneEndpoint address
```

需要执行下面的命令来修改kubeadm配置文件：
```bash
kubectl edit cm kubeadm-config -n kube-system
```
在`kubernetesVersion`同一级添加controlPlaneEndpoint配置，再重新加入master节点即可。
```bash
...
  kubernetesVersion: v1.23.0
  controlPlaneEndpoint: {k8s_master01_IP_OR_LoadBalance_IP}:6443
...
```


**References**
[1] https://blog.csdn.net/Sebastien23/article/details/112976697
[2] https://blog.csdn.net/Sebastien23/article/details/113757356
[3] https://blog.csdn.net/Sebastien23/article/details/125958860
[4] https://blog.csdn.net/wangy_0228/article/details/128157888
[5] https://blog.csdn.net/hedao0515/article/details/126342939?spm=1001.2014.3001.5506
