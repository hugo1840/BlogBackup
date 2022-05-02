@[TOC](Service统一入口访问应用)

# Service存在的意义
Service的主要作用在于：

-	防止Pod失联（服务发现，增删Pod都能感知到IP）；
-	定义一组Pod的访问策略（负载均衡）；
-	支持ClusterIP、NodePort以及LoadBalancer三种类型。

# Pod与Service的关系
Service与Pod通过label-selector相关联。

```bash
# svc.yaml
apiVersion: v1
kind: Service
metadata:
   labels:
     app: java-demo
   name: java-demo
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:  # 通过此处标签关联到具体的Pod
    app: java-demo
  type: NodePort
```

Service向Pod转发请求是通过 **iptables**（基于Linux Netfilter的IP包过滤）和 **IPVS** 实现的。IPVS则是一个基于传输层的负载均衡和数据转发的Linux内核模块。Service的功能由 **kube-proxy** 组件提供。

# Service的类型
Service有三种常用的类型：ClusterIP、NodePort和LoadBalancer。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210214203008340.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)

 
**ClusterIP**：分配一个集群内部IP地址，只能在集群内部访问（同命名空间内的Pod），默认的ServiceType。ClusterIP模式的Service提供的是Pod的一个稳定的vip地址。

**NodePort**：分配一个集群内部的IP地址，并在每个节点上启用一个端口来暴露服务，可以从集群外部访问。访问地址为`节点IP:PORT`。

**LoadBalancer**：分配一个集群内部IP地址，并在每个节点上启用一个端口来暴露服务。除此之外，Kubernetes会请求底层云平台上的负载均衡器，将每个节点（节点IP:PORT）作为后端添加进去。


# 使用NodePort对外暴露应用
使用`kubectl expose`命令发布已经创建好的deployment控制器管理的Pod。

```bash
$ kubectl expose deployment java-demo --port=80 --target-port=8080 --type=NodePort
```

