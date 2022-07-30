@[TOC](k8s部署metrics server资源监控及日志查看)

# 查看资源的常用命令
## kubectl get
查看资源信息
```bash
kubectl get <资源类型> <资源名称>
kubectl get <资源类型> <资源名称> -o wide  #显示详细信息
kubectl get <资源类型> <资源名称> -o yaml  #导出yaml文件配置
```
例如
```bash
kubectl get node
kubectl get node k8s-master 
```

查看节点标签信息
```bash
#给节点打标签
[root@k8s-master ~]# kubectl label node k8s-node1 node-role.kubernetes.io/node1=

[root@k8s-master ~]# kubectl get node
NAME         STATUS   ROLES                  AGE   VERSION
k8s-master   Ready    control-plane,master   33h   v1.23.0
k8s-node1    Ready    node1                  33h   v1.23.0
k8s-node2    Ready    <none>                 33h   v1.23.0

[root@k8s-master ~]# kubectl get node k8s-master --show-labels
```

几个常用缩写
```bash
kubectl get cs # 查看control-manager和scheduler组件状态
kubectl get po  # 相当于kubectl get pods
kubectl get svc  # 相当于kubectl get service
```

查看集群中所有API资源信息
```bash
kubectl api-resources
```

## kubectl describe
查看资源详细描述
```bash
kubectl describe <资源类型> <资源名称>
```

可以把`kubectl get`与`kubectl describe`结合使用：
```bash
[root@k8s-master ~]# kubectl get po
NAME                         READY   STATUS    RESTARTS   AGE
web-demo1-5ff6d576bb-5292c   1/1     Running   0          4h4m
web-demo1-5ff6d576bb-92gqk   1/1     Running   0          4h4m
web-demo1-5ff6d576bb-d26cf   1/1     Running   0          4h4m
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get pod web-demo1-5ff6d576bb-5292c -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP               NODE        NOMINATED NODE   READINESS GATES
web-demo1-5ff6d576bb-5292c   1/1     Running   0          4h10m   10.244.169.134   k8s-node2   <none>           <none>
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl describe pod web-demo1-5ff6d576bb-5292c
Name: ...
Namespace: ...
Node: ...
Start Time: ...
Labels: ...
IP: ...
COntainers: ...
Conditions: ...
Volumes: ...
Tolerations: ...


[root@k8s-master ~]# kubectl get node
NAME         STATUS   ROLES                  AGE   VERSION
k8s-master   Ready    control-plane,master   34h   v1.23.0
k8s-node1    Ready    node1                  34h   v1.23.0
k8s-node2    Ready    <none>                 34h   v1.23.0
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get node k8s-node2 -o wide
NAME        STATUS   ROLES    AGE   VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
k8s-node2   Ready    <none>   34h   v1.23.0   192.168.136.98    <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://20.10.17
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl describe node k8s-node2
Name: ...
Roles: ...
Labels: ...
Taints: ...
Conditions: ...
Addresses: ...
Capacity:
  cpu: ...
  memory: ...
System Info: ...
PodCIDR: ...
Events: ...
```

使用`kubectl top`命令可以监控集群资源利用率，但是需要先安装**Metrics Server**，否则会报错。
```bash
[root@k8s-master ~]# kubectl top node
error: Metrics API not available
[root@k8s-master ~]# kubectl top pod
error: Metrics API not available
```

# Metrics Server部署
## 通过yaml文件部署metrics-server
下载yaml配置文件：
```bash
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

修改配置文件`components.yaml`，添加`--kubelet-insecure-tls`参数，告诉metrics server不验证kubelet提供的https证书。
```yaml
containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --kubelet-insecure-tls
```

部署Metrics Server
```bash
kubectl apply -f components.yaml
```

检查是否部署成功
```bash
[root@k8s-master ~]# kubectl get apiservices | grep metrics
v1beta1.metrics.k8s.io                 kube-system/metrics-server   False (MissingEndpoints)   8m27s
[root@k8s-master ~]# kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
Error from server (ServiceUnavailable): the server is currently unable to handle the request
```
如果状态为True且能够返回数据，说明Metrics Server运行正常。可以看到，本次部署失败了。

## 部署报错排查
查看镜像部署状态：
```bash
kubectl get po -n kube-system
NAME                                       READY   STATUS             RESTARTS       AGE
...
metrics-server-574849569f-svt2v            0/1     ImagePullBackOff   0              8m8s
```

查看Pod日志：
```bash
[root@k8s-master ~]# kubectl logs metrics-server-574849569f-svt2v -n kube-system
Error from server (BadRequest): container "metrics-server" in pod "metrics-server-574849569f-svt2v" is waiting to start: trying and failing to pull image
```

>结合上面的输出分析，应该是镜像拉取失败了，需要将yaml文件中的镜像下载地址替换为国内的镜像仓库地址。

删除当前出错的Metrics Server部署：
```bash
kubectl delete -f components.yaml
```

### 替换镜像下载地址
替换镜像下载地址为
```yaml
image: registry.cn-shenzhen.aliyuncs.com/zengfengjin/metrics-server:v0.5.0
```

重新部署
```bash
kubectl apply -f components.yaml
```

安装成功，Pod状态变为**Running**，但是一直没有**Ready**。
```bash
[root@k8s-master ~]# kubectl get deployment,po -n kube-system
NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/calico-kube-controllers   1/1     1            1           35h
deployment.apps/coredns                   2/2     2            2           35h
deployment.apps/metrics-server            0/1     1            0           87s

NAME                                           READY   STATUS    RESTARTS        AGE
...
pod/metrics-server-798c598bb8-rv827            0/1     Running   0               87s
```

检查Metrics Server的Pod日志：
```bash
[root@k8s-master ~]# kubectl logs metrics-server-798c598bb8-rv827 -n kube-system
E0724 14:26:19.149545       1 scraper.go:139] "Failed to scrape node" err="GET \"https://192.168.x.x:10250/stats/summary?only_cpu_and_memory=true\": bad status code \"403 Forbidden\"" node="k8s-node1"

[root@k8s-master ~]# kubectl describe pod metrics-server-798c598bb8-rv827 -n kube-system
...
Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  6m25s                 default-scheduler  Successfully assigned kube-system/metrics-server-798c598bb8-rv827 to k8s-node2
  Normal   Pulling    6m24s                 kubelet            Pulling image "registry.cn-shenzhen.aliyuncs.com/zengfengjin/metrics-server:v0.5.0"
  Normal   Pulled     5m51s                 kubelet            Successfully pulled image "registry.cn-shenzhen.aliyuncs.com/zengfengjin/metrics-server:v0.5.0" in 32.739844006s
  Normal   Created    5m51s                 kubelet            Created container metrics-server
  Normal   Started    5m51s                 kubelet            Started container metrics-server
  Warning  Unhealthy  75s (x29 over 5m25s)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 500
```

注意到以下GET请求被拒绝（**403 Forbidden**）
```bash
GET \"https://192.168.x.x:10250/stats
```

>原因是我们下载的yaml文件是最新的，是基于**0.6.x**写的，而镜像下载地址被手动改成了**0.5.x**。在Metrics Server中，**0.5.x**的配置文件中需要的权限配置与**0.6.x**不一样。**0.5.x**中需要对`nodes/stats`的访问权限，但是**0.6.x**中改成了`nodes/metrics`。

删除当前出错的Metrics Server部署：
```bash
kubectl delete -f components.yaml
```

### metrics-server资源访问权限修改
检查配置文件中metrics-server角色权限：
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
```

增加访问`nodes/stats`的权限，修改为
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes/stats
  - pods
  - nodes
  verbs:
  - get
  - list
  - watch
```

重新部署
```bash
kubectl apply -f components.yaml

[root@k8s-master ~]# kubectl get deployment,po -n kube-system
NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/calico-kube-controllers   1/1     1            1           35h
deployment.apps/coredns                   2/2     2            2           36h
deployment.apps/metrics-server            1/1     1            1           37s

NAME                                           READY   STATUS             RESTARTS        AGE
pod/metrics-server-798c598bb8-j69cc            1/1     Running            0               36s
```

检查部署是否成功
```bash
[root@k8s-master ~]# kubectl get apiservices | grep metrics
v1beta1.metrics.k8s.io                 kube-system/metrics-server   True        80s
[root@k8s-master ~]# kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
{"kind":"NodeMetricsList","apiVersion":"metrics.k8s.io/v1beta1","metadata":{},"items":[{"metadata":{"name":"k8s-master","creationTimestamp":"2022-07-24T15:06:31Z","labels":{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/os":"linux","kubernetes.io/arch":"amd64","kubernetes.io/hostname":"k8s-master","kubernetes.io/os":"linux","node-role.kubernetes.io/control-plane":"","node-role.kubernetes.io/master":"","node.kubernetes.io/exclude-from-external-load-balancers":""}},"timestamp":"2022-07-24T15:06:11Z","window":"10s","usage":{"cpu":"201981233n","memory":"1244120Ki"}},{"metadata":{"name":"k8s-node1","creationTimestamp":"2022-07-24T15:06:31Z","labels":{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/os":"linux","kubernetes.io/arch":"amd64","kubernetes.io/hostname":"k8s-node1","kubernetes.io/os":"linux","node-role.kubernetes.io/node1":""}},"timestamp":"2022-07-24T15:06:14Z","window":"20s","usage":{"cpu":"86504224n","memory":"846880Ki"}},{"metadata":{"name":"k8s-node2","creationTimestamp":"2022-07-24T15:06:31Z","labels":{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/os":"linux","kubernetes.io/arch":"amd64","kubernetes.io/hostname":"k8s-node2","kubernetes.io/os":"linux"}},"timestamp":"2022-07-24T15:06:10Z","window":"10s","usage":{"cpu":"77667531n","memory":"883624Ki"}}]}
```

查看资源监控
```bash
[root@k8s-master ~]# kubectl top node
NAME         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-master   208m         10%    1217Mi          70%
k8s-node1    83m          4%     829Mi           48%
k8s-node2    80m          4%     864Mi           50%
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl top pod
NAME                         CPU(cores)   MEMORY(bytes)
web-demo1-5ff6d576bb-5292c   1m           5Mi
web-demo1-5ff6d576bb-92gqk   1m           7Mi
web-demo1-5ff6d576bb-d26cf   1m           3Mi
```

# 查看日志的常用命令

## kubelet日志
kubelet组件使用systemd管理服务，查看日志的命令为
```bash
journalctl -u kubelet -f
```
或者查看系统日志
```bash
tail -f /var/log/messages
```

## Pod日志
其他k8s组件采用容器部署，查看日志的命令为
```bash
kubectl get po -n <命名空间>
kubectl logs <Pod名称> -n <命名空间>
kubectl logs <Pod名称> -n <命名空间> -f #实时查看
```

标准输出在宿主机的路径为
```bash
/var/lib/docker/containers/<container-id>/<container-id>-json.log
```

查看容器ID的方法为
```bash
kubectl get pods -o wide
#到Pod所在节点查看容器ID
docker ps | grep web-demo1-5ff6d576bb-92gqk
```

也可以进入容器终端日志目录查看日志
```bash
kubectl exec -it <Pod名称> --bash
```

**参考文章**
【1】https://stackoverflow.com/questions/70362216/getting-error-while-implementing-metric-server-inside-the-kubernetes
【2】https://github.com/kubernetes-sigs/metrics-server/releases
