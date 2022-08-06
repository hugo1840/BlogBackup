@[TOC](Service：服务发现与负载均衡)

# Service存在的意义
Service是k8s中将运行在一组Pods上的应用程序暴露为网络服务的抽象方法。

- 服务发现：找到提供同一服务的一组Pod；
- 负载均衡：定义一组Pod的访问策略。

举个例子，考虑一个图片处理后端，它运行了3个副本。这些副本是可互换的，前端不需要关心它们调用了哪个后端副本。 然而组成这一组后端程序的Pod实际上可能会发生变化（被销毁或重建）， 前端客户端不应该也没必要知道，而且也不需要跟踪这一组后端的状态。Service定义的抽象能够**解耦**这种关联。

Service与Pod的关系：
- Service通过标签关联一组Pod；
- Service为一组Pod提供负载均衡能力。

# Service的定义与创建
## Service的创建
**方法一**：为已有的deployment创建service。
```bash
kubectl expose deployment
```

**方法二**：直接创建Service。

生成模板文件：
```bash
kubectl create service clusterip my-service --tcp=80:9376 --dry-run=client -o yaml > service.yaml
```

编辑yaml文件：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

修改后创建服务：
```bash
kubectl apply -f service.yaml
```

查看Service状态：
```bash
kubectl get service
```

查看Service关联的Pod地址：
```bash
kubectl get ep
```

## 多端口Servcie定义
对于某些服务，需要公开多个端口。相应地，Service也要配置多个端口定义，通过端口名称区分。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - name: http
	  protocol: TCP
      port: 80
      targetPort: 80
	- name: https
	  protocol: TCP
      port: 443
      targetPort: 443
```

# Service的常用类型
可以通过ServiceTypes指定Service的类型，有以下几种选择：

- `ClusterIP`：通过集群的内部IP暴露服务，选择该值时服务只能够在集群内部访问。是默认的类型；
- `NodePort`：通过每个节点上的IP和静态端口（NodePort）暴露服务。NodePort服务会路由到自动创建的ClusterIP服务。 通过请求`<节点IP>:<节点端口>`，可以从集群外部访问一个NodePort服务。
- `LoadBalancer`：使用云提供商的负载均衡器向外部暴露服务。外部负载均衡器可以将流量路由到自动创建的NodePort服务和ClusterIP服务上。
- `ExternalName`：通过返回CNAME和对应值，可以将服务映射到externalName字段的内容（例如`foo.bar.example.com`）。无需创建任何类型代理。

**注**：需要使用kube-dns 1.7及以上版本或者CoreDNS 0.0.8及以上版本才能使用`ExternalName`类型。

## ClusterIP
分配一个稳定的IP地址，即vip，只能在集群内部访问（在Pod中或者Node上都能访问）。

```yaml
spec:
  type: ClusterIP
  ports:
    - port: 80
	  protocol: TCP
	  targetPort: 80
  selector:
    app: web
```

## NodePort
在每个节点上启用一个端口来暴露服务，可以在集群外部访问。也会分配一个稳定的内部集群IP地址。
- 访问地址：`<NodeIP>:<NodePort>`
- 端口范围：**30000-32767**

```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
	targetPort: 80
	nodePort: 30009
  selector:
    app: web
```

## LoadBalancer
与NodePort类似，在每个节点上启用一个端口来暴露服务。除此之外，k8s会请求底层云平台（比如AWS、阿里云、腾讯云）上的负载均衡器，将每个`NodeIP:NodePort`作为后端加进去。


# Service代理模式
Service底层实现主要有iptables和ipvs两种网络模式，决定了如何转发流量。默认为**iptables**模式。

查看使用的Service代理模式：
```bash
#查看kube-proxy的Pod名称
[root@k8s-master ~]# kubectl get pod -n kube-system
#查看Pod日志
[root@k8s-master ~]# kubectl logs kube-proxy-58kfs -n kube-system
I0806 00:16:02.180484       1 server_others.go:561] "Unknown proxy mode, assuming iptables proxy" proxyMode=""
I0806 00:16:02.305394       1 server_others.go:206] "Using iptables Proxier"
```

## iptables
这种模式下，kube-proxy会监视Kubernetes控制节点对Service对象和Endpoints对象的添加和移除。 对每个Service，它会配置iptables规则，从而捕获到达该Service的clusterIP和端口的请求，进而将请求重定向到Service的一组后端中的某个Pod上面。 对于每个Endpoints对象，它也会配置iptables规则，这个规则会选择一个后端组合。使用iptables处理流量具有较低的系统开销，因为流量由Linux netfilter处理， 而无需在用户空间和内核空间之间切换。

查看负载均衡规则：
```bash
#iptables-save命令用于将linux内核中的iptables表导出到标准输出
iptables-save | grep <SERVICE-NAME>
```

**实验1**
部署Nginx前端web服务：
```bash
[root@k8s-master ~]# cat nginx-deploy.yaml
apiVersion: apps/v1   #api版本
kind: Deployment      #资源类型
metadata:             #资源的元数据
  name: nginx-web
  namespace: default
spec:                 #资源规格
  replicas: 3         #副本数量
  selector:           #标签选择器
    matchLabels:      #与下面Pod的labels保持一致
      app: nginx-web
  #被控制对象
  template:           #Pod模板
    metadata:
      labels:         #Pod的标签
        app: nginx-web
    spec:
      containers:     #容器配置
      - name: nginx-web
        image: nginx:1.20  #容器使用的镜像
[root@k8s-master ~]#
[root@k8s-master ~]# cat nginx-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-web
  namespace: default
spec:
  ports:
  - port: 80          #Service端口，通过ClusterIP访问
    protocol: TCP
    targetPort: 80    #Pod内部署应用的服务端口，比如nginx是80
  selector:           #标签选择器，与deployment中保持一致
    app: nginx-web
  type: NodePort      #Service类型
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl apply -f nginx-deploy.yaml
[root@k8s-master ~]# kubectl apply -f nginx-svc.yaml
```

查看iptables转发规则：
```bash
[root@k8s-master ~]# iptables-save | grep nginx
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/nginx-web" -m tcp --dport 32379 -j KUBE-SVC-CJUJYHIKVPA67ZFN
-A KUBE-SERVICES -d 10.111.222.10/32 -p tcp -m comment --comment "default/nginx-web cluster IP" -m tcp --dport 80 -j KUBE-SVC-CJUJYHIKVPA67ZFN
```
上面有两条规则（流量入口）：
- 通过NodePort访问32379端口，命中`KUBE-NODEPORTS`，走`KUBE-SVC-CJUJYHIKVPA67ZFN`规则；
- 通过ClusterIP访问80端口，命中`KUBE-SERVICES`，也会走`KUBE-SVC-CJUJYHIKVPA67ZFN`规则。

继续查看KUBE-SVC-CJUJYHIKVPA67ZFN规则的转发模式：
```bash
# KUBE-SVC-CJUJYHIKVPA67ZFN规则
[root@k8s-master ~]# iptables-save | grep KUBE-SVC-CJUJYHIKVPA67ZFN
...
-A KUBE-SVC-CJUJYHIKVPA67ZFN -m comment --comment "default/nginx-web" -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-7JJLFLA7KUXKZONR
-A KUBE-SVC-CJUJYHIKVPA67ZFN -m comment --comment "default/nginx-web" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-VGBWIIDSI463MCML
-A KUBE-SVC-CJUJYHIKVPA67ZFN -m comment --comment "default/nginx-web" -j KUBE-SEP-VJTMVZQFXALRW5SX
```

可见KUBE-SVC-CJUJYHIKVPA67ZFN规则后面还有三条转发规则（负载均衡）：
- `KUBE-SEP-7JJLFLA7KUXKZONR`，命中的概率为三分之一；
- `KUBE-SEP-VGBWIIDSI463MCML`，命中的概率为`(1-33%)*50%=33%`；
- `KUBE-SEP-VJTMVZQFXALRW5SX`，命中的概率为`1-33%-33%=33%`。
可见，iptables通过`KUBE-SVC-CJUJYHIKVPA67ZFN`规则实现了简单的负载均衡。

继续查看上面三种规则的转发模式：
```bash
[root@k8s-master ~]# iptables-save | grep  KUBE-SEP-7JJLFLA7KUXKZONR
-A KUBE-SEP-7JJLFLA7KUXKZONR -p tcp -m comment --comment "default/nginx-web" -m tcp -j DNAT --to-destination 10.244.169.160:80
-A KUBE-SVC-CJUJYHIKVPA67ZFN -m comment --comment "default/nginx-web" -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-7JJLFLA7KUXKZONR
[root@k8s-master ~]#
[root@k8s-master ~]# iptables-save | grep  KUBE-SEP-VGBWIIDSI463MCML
-A KUBE-SEP-VGBWIIDSI463MCML -p tcp -m comment --comment "default/nginx-web" -m tcp -j DNAT --to-destination 10.244.36.83:80
-A KUBE-SVC-CJUJYHIKVPA67ZFN -m comment --comment "default/nginx-web" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-VGBWIIDSI463MCML
[root@k8s-master ~]#
[root@k8s-master ~]# iptables-save | grep  KUBE-SEP-VJTMVZQFXALRW5SX
-A KUBE-SEP-VJTMVZQFXALRW5SX -p tcp -m comment --comment "default/nginx-web" -m tcp -j DNAT --to-destination 10.244.36.84:80
-A KUBE-SVC-CJUJYHIKVPA67ZFN -m comment --comment "default/nginx-web" -j KUBE-SEP-VJTMVZQFXALRW5SX
```
可见三条规则都做了DNAT（网络目标地址转换），并准发到三个不同的目标IP的80端口。而这三个IP刚好对应了我们部署的nginx-web的三个Pod。

```bash
[root@k8s-master ~]# kubectl get ep
NAME         ENDPOINTS                                           AGE
nginx-web    10.244.169.160:80,10.244.36.83:80,10.244.36.84:80   45m
```


## IPVS
在ipvs模式下，kube-proxy监视Kubernetes服务和端点，调用netlink接口相应地创建IPVS规则，并定期将IPVS规则与Kubernetes服务和端点同步。该控制循环可确保IPVS状态与所需状态匹配。访问服务时，IPVS将流量定向到后端Pod之一。

IPVS代理模式基于类似于iptables模式的netfilter挂钩函数， 但是使用哈希表作为基础数据结构，并且在内核空间中工作。这意味着，与iptables模式下的kube-proxy相比，IPVS模式下的kube-proxy重定向通信的延迟要短，并且在同步代理规则时具有更好的性能。与其他代理模式相比，IPVS模式还支持更高的网络流量吞吐量。

查看负载均衡规则：
```bash
#安装ipvs管理工具
yum install ipvsadm -y
#查看代理规则
ipvsadm -L -n
```

Service默认使用iptables代理模式。通过kubeadm方式可以修改为ipvs模式。
```bash
#将mode配置为"ipvs"
[root@k8s-master ~]# kubectl edit configmap kube-proxy -n kube-system
mode: "ipvs"
```

kube-proxy配置文件以configmap方式存储。如果要让所有节点生效，需要重建所有节点的kube-proxy的Pod。
```bash
#删除Pod后会自动重建，配置才会生效
kubectl delete pod kube-proxy-75z28 -n kube-system
```

**实验2**
```bash
#查看节点1的ipvs转发规则
[root@k8s-node1 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn

#删除node1的kube-proxy Pod
[root@k8s-node1 ~]# kubectl delete pod kube-proxy-75z28 -n kube-system

#查看节点1的ipvs转发规则
[root@k8s-node1 ~]# 
[root@k8s-node1 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
...
TCP  172.17.0.1:32042 rr
  -> 10.244.36.83:80              Masq    1      0          0
  -> 10.244.36.84:80              Masq    1      0          0
  -> 10.244.169.160:80            Masq    1      0          0
TCP  192.168.124.140:32042 rr
  -> 10.244.36.83:80              Masq    1      0          0
  -> 10.244.36.84:80              Masq    1      0          0
  -> 10.244.169.160:80            Masq    1      0          0
...
```

**对比总结**
- iptables
  - 灵活、功能强大；
  - 规则遍历匹配和更新，呈线性时延；
  - 适合节点数较少的情况，一般不超过100个。
- IPVS
  - 工作在内核态，性能更佳；
  - 调度算法丰富，如轮询rr、最少连接lc、目标地址哈希dh、源地址哈希sh等。


## DNS域名
k8s默认部署了CoreDNS组件。CoreDNS服务监视Kubernetes API，为每一个Service创建DNS记录用于域名解析。

ClusterIP A记录格式为：
```
<service-name>.<namespace-name>.svc.cluster.local
示例：my-svc.my-namespace.svc.cluster.local
```

**实验3**
```bash
[root@k8s-master ~]# kubectl run bs2 --image=busybox:1.28.4 -- sleep 24h
pod/bs2 created

[root@k8s-master ~]# kubectl exec -it bs2 -- sh
/ # nslookup nginx-web
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx-web
Address 1: 10.111.222.10 nginx-web.default.svc.cluster.local

```


**References**
【1】https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/
【2】https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies


