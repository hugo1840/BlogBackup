---
tags: [k8s]
title: Kubernetes之etcd数据库备份与恢复
created: '2023-03-11T10:41:33.384Z'
modified: '2023-03-12T06:30:23.407Z'
---

Kubernetes之etcd数据库备份与恢复

将etcd容器中的etcdctl命令拷贝到宿主机：
```bash
kubectl cp kube-system/etcd-k8s-master:/usr/local/etcdctl /root/
```

或者直接yum安装etcd包：
```bash
yum install etcd -y
```

etcdctl 3.3的默认API版本为2，与版本3的差别较大。在命令前加上`ETCDCTL_API=3`来调用版本3接口。
```bash
[root@k8s-master ~]# etcdctl --version
etcdctl version: 3.3.11
API version: 2
```

# kubeadm部署方式
## 备份
```bash
ETCDCTL_API=3 etcdctl snapshot save snap_230312.db \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key
```


## 恢复
1. 暂停kube-apiserver和etcd容器
```bash
mv /etc/kubernetes/manifests /etc/kubernetes/manifests.bak
mv /var/lib/etcd/ /var/lib/etcd.bak
```
etcd容器所在Pod是kubelet管理的静态Pod，删除后会自动拉起。移除`/etc/kubernetes/manifests`下的yaml配置文件后，k8s集群会挂掉，etcd容器也不会自动拉起了。

2. 恢复数据库
```bash
ETCDCTL_API=3 etcdctl snapshot restore snap_230312.db --data-dir=/var/lib/etcd
```

3. 启动kube-apiserver和etcd容器
```bash
mv /etc/kubernetes/manifests.bak /etc/kubernetes/manifests
```


# 二进制部署方式
## 备份
```bash
ETCDCTL_API=3 etcdctl snapshot save snap_230313.db \
--endpoints=https://192.168.33.x:2379 \
--cacert=/opt/etcd/ssl/ca.pem \
--cert=/opt/etcd/ssl/server.pem \
--key=/opt/etcd/ssl/server-key.pem
```

## 恢复
1. 暂停kube-apiserver和etcd容器
```bash
systemctl stop kube-apiserver
systemctl stop etcd
mv /var/lib/etcd/default.etcd /var/lib/etcd/default.etcd.bak
```

2. 在所有节点上恢复
```bash
ETCDCTL_API=3 etcdctl snapshot restore snap_230313.db --name etcd-1 \
--initial-cluster="etcd-1=https://192.168.33.a:2380, etcd-2=https://192.168.33.b:2380, etcd-3=https://192.168.33.c:2380" \
--initial-cluster-token=etcd-cluster \
--initial-advertise-peer-urls=https://192.168.33.a:2380 \
--data-dir=/var/lib/etcd/default.etcd
```

3. 启动kube-apiserver和etcd容器
```bash
systemctl start kube-apiserver
systemctl start etcd
```


**References**
【1】https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/configure-upgrade-etcd/
【2】https://etcd.io/docs/v3.6/op-guide/recovery/#restoring-a-cluster

