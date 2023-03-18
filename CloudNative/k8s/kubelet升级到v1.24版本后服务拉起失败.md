---
tags: [k8s]
title: kubelet升级到v1.24版本后服务拉起失败
created: '2023-03-18T07:13:32.699Z'
modified: '2023-03-18T07:29:59.255Z'
---

kubelet升级到v1.24版本后服务拉起失败

# 现象
k8s集群从1.23版本升级到1.24版本，升级完kubelet组件后，发现kubelet systemd服务拉起失败。

节点状态：
```bash
[root@k8s-master1 ~]# kubectl get nodes
NAME          STATUS     ROLES           AGE     VERSION
k8s-master1   NotReady   control-plane   3d17h   v1.23.0
k8s-master2   Ready      control-plane   3d15h   v1.23.0
k8s-master3   Ready      control-plane   3d15h   v1.23.0
k8s-worker1   Ready      <none>          3d16h   v1.23.0
k8s-worker2   Ready      <none>          3d16h   v1.23.0
```

kubelet服务状态：
```bash
[root@k8s-master1 ~]# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: activating (auto-restart) (Result: exit-code) since Sat 2023-03-18 14:31:18 CST; 9s ago
     Docs: https://kubernetes.io/docs/
  Process: 128871 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=1/FAILURE)
 Main PID: 128871 (code=exited, status=1/FAILURE)

Mar 18 14:31:18 k8s-master1 kubelet[128871]: --tls-min-version string                                   Minimum TLS version supported. Possible values: Ver...
Mar 18 14:31:18 k8s-master1 kubelet[128871]: --tls-private-key-file string                              File containing x509 private key matching --tls-cer...
Mar 18 14:31:18 k8s-master1 kubelet[128871]: --topology-manager-policy string                           Topology Manager policy to use. Possible values: 'n...
Mar 18 14:31:18 k8s-master1 kubelet[128871]: --topology-manager-scope string                            Scope to which topology hints applied. Topology Man...
Mar 18 14:31:18 k8s-master1 kubelet[128871]: -v, --v Level                                                  number for the log level verbosity
Mar 18 14:31:18 k8s-master1 kubelet[128871]: --version version[=true]                                   Print version information and quit
Mar 18 14:31:18 k8s-master1 kubelet[128871]: --vmodule pattern=N,...                                    comma-separated list of pattern=N settings...g format)
Mar 18 14:31:18 k8s-master1 kubelet[128871]: --volume-plugin-dir string                                 The full path of the directory in which to search f...
Mar 18 14:31:18 k8s-master1 kubelet[128871]: --volume-stats-agg-period duration                         Specifies interval for kubelet to calculate and cac...
Mar 18 14:31:18 k8s-master1 kubelet[128871]: Error: failed to parse kubelet flag: unknown flag: --network-plugin
Hint: Some lines were ellipsized, use -l to show in full.
```

# 解决办法
从kubelet服务状态信息中注意到以下报错：
```bash
kubelet[128871]: Error: failed to parse kubelet flag: unknown flag: --network-plugin
```

修改下面的配置文件，去掉`--network-plugin=cni`：
```bash
[root@k8s-master1 ~]# cat /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.6"

[root@k8s-master1 ~]# vi /var/lib/kubelet/kubeadm-flags.env
[root@k8s-master1 ~]# cat /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.6"
```

最后重启服务即可：
```bash
systemctl restart kubelet
```

节点状态：
```bash
[root@k8s-master1 ~]# kubectl get nodes
NAME          STATUS     ROLES           AGE     VERSION
k8s-master1   Ready      control-plane   3d17h   v1.24.11
k8s-master2   Ready      control-plane   3d15h   v1.23.0
k8s-master3   Ready      control-plane   3d15h   v1.23.0
k8s-worker1   Ready      <none>          3d16h   v1.23.0
k8s-worker2   Ready      <none>          3d16h   v1.23.0
```




