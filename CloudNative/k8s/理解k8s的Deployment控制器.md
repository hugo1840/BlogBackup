@[TOC](理解k8s的Deployment控制器)

# Pod与控制器的关系
控制器（Controllers）是在集群上管理和运行容器的对象。控制器通过label-selector关联Pod。Pod通过控制器可以实现应用的运维，如伸缩、滚动升级等。

# Deployment的功能
Deployment的功能包括：
-	部署无状态应用；
-	管理Pod和ReplicaSet；
-	具有上线部署、副本设定、滚动升级、回滚等功能；
-	提供声明式更新，例如只更新一个新的镜像。

主要的应用场景包括Web服务、微服务。

# YAML字段解析
生成yaml配置文件
```bash
$ kubectl create deployment dpname --image=nginx \
--dry-run -o yaml > deploy.yaml  # --dry-run表示测试而不实际运行
```

修改deploy.yaml其中的字段。
```bash
# 控制器定义
apiVersion: apps/v1
kind: Deployment
metadata:
   name: nginx-deployment
   namespace: default
spec:
   # 定义Pod可用副本数
   replicas: 3  
   selector:
     # 至少定义两个标签，分别关联项目和应用
     matchLabels:  
       app: nginx
  # 被控制对象（Pod）
   template:
     metadata:
       labels:
         app: nginx
     spec:
       # 定义Pod管理的容器
       containers:
       - name: nginx
         image: nginx:latest
         ports:
         - containerPort: 80
```

# 使用Deployment部署无状态应用
**创建部署**
```bash
# 使用yaml配置文件部署
$ kubectl create -f deploy.yaml
# 使用create deployment命令部署
$ kubectl create deployment NAME --image=IMAGE -- [COMMAND [args…]]
```

查看已经存创建的deployment部署：`kubectl get deploy`

同时查看多个资源对象：`kubectl get deploy,pods`


**发布应用**
Service服务类型分为：ClusterIP，供集群内部访问；NodePort，供集群外部访问；LoadBalancer，主要对接公有云上的LB；ExternalName，添加访问的外部名称。默认为ClusterIP。发布应用使用命令格式为

`kubectl expose (-f FILENAME | TYPE NAME) [--port=port] [--protocol=TCP|UDP|SCTP] [--target-port=number-or-name] [--name=name] [--external-ip=external-ip-of-service] [--type=type] `


执行如下命令：
```bash
# --port为Service端口，--target-port为Pod中部署应用的端口
# svcName为服务名称，depName为部署控制器名称
$ kubectl expose --name=svcNAME deployment depNAME \
--port=80 --target-port=8080 --type=NodePort
# 查看创建服务的端口
$ kubectl get service
$ kubectl get svc
```

# 升级与回滚
升级pod容器镜像命令格式为
` kubectl set image (-f FILENAME | TYPE NAME) CONTAINER_NAME_1=CONTAINER_IMAGE_1 ... CONTAINER_NAME_N=CONTAINER_IMAGE_N `

使用rollout命令查看升级状态：
`kubectl rollout status (TYPE NAME | TYPE/NAME) [flags]`

使用edit命令可以直接修改资源对象的配置文件：
```bash
$ kubectl edit deployment/depName  #  修改控制器配置文件depName.yaml
$ kubectl edit svc/svcName  # 修改服务配置文件svcName.yaml
```
也可以使用`kubectl patch`命令以打补丁的方式动态修改配置文件参数。


执行如下命令：
```bash
# 升级（把NAME替换为对应的资源名称）
$ kubectl set image deployment/NAME nginx:=nginx:1.15
$ kubectl rollout status deployment/NAME  

# 回滚（把NAME替换为对应的资源名称）
$ kubectl rollout history deployment/NAME  # 查看部署与升级历史
$ kubectl rollout undo deployment/NAME  # 回滚到上一个版本
$ kubectl rollout undo deployment/NAME --reversion=2  # 回滚到指定的版本

# 删除
$ kubectl delete deploy/NAME
$ kubectl delete svc/NAME
```

如果删除的是Pod而不是Deployment，kubernetes会自动创建新的Pod对象。

# 弹性伸缩
Kubernetes资源的弹性伸缩使用scale命令实现，其格式为
`kubectl scale [--resource-version=version] [--current-replicas=count] --replicas=COUNT (-f FILENAME | TYPE NAME) `

**弹性扩容**
```bash
$ kubectl scale deployment/depName --replicas=5  # 修改部署的副本数
$ kubectl scale deployment depName --replicas=5
```

# Deployment与ReplicaSet
创建Deployment时也会生成一个ReplicaSet资源对象。
```bash
$ kubectl get deploy,rs  # 查看deployment与replicaset资源
```
ReplicaSet的作用包括：
-	确保Pod的数量符合Deployment配置文件中replicas参数的要求；
-	记录部署与更新的历史版本（与`kubectl rollout history`有关）。

当使用`kubectl set image`进行滚动更新后，会生成一个新的ReplicaSet。新的ReplicaSet会创建新的Pod，并关联到Service。当有一个新的Pod被创建且准备就绪后，旧的ReplicaSet就会对应地删除一个Pod，以维持总数量不变，如此这样逐步替换掉所有旧的Pods。

