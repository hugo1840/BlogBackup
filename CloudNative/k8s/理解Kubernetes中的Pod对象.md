@[TOC](理解Kubernetes中的Pod对象)

# Pod存在的意义
Pod为亲密性应用而存在。亲密性应用场景包括：
-	两个应用之间发生文件交互；
-	两个应用需要通过127.0.0.1或者socket通信；
-	两个应用需要发生频繁的调用。


# Pod实现机制
同一个Pod中的容器共享网络和存储。
## 网络共享
同一个Pod里的容器共享网络命名空间（包括IP、端口号、MAC地址）。网络命名空间的信息存储在Infrastructure容器中（节点上自动生成的Pause容器）。

```bash
$ kubectl get pods 
java-demo-8567655c52-2v567   RUNNING
java-demo-8567655c52-sp59a   RUNNING
java-demo-8567655c52-35atw   RUNNING

# 导出创建Pod的yaml文件
$ kubectl get pods java-demo-8567655c52-2v567 -o yaml > pod.yaml 
# 删除yaml文件中的多余字段，在Pod中运行nginx和java-demo两个容器
$ vi pod.yaml  
apiVersion: v1
kind: Pod
metadata:
   labels:
     app: my-pod
   name: my-pod
   namespace: default
spec:
  containers:
  - image: nginx
    name: nginx
    image: nginx
  - image:
    name: java-demo
    image: dokerhub_account/java-demo
```

使用修改过的yaml文件运行新的Pod。
```bash
$ kubectl apply -f pod.yaml
$ kubectl get pods  # READY 2/2 表示Pod内有两个容器运行
NAME                        READY        STATUS
java-demo-8567655c52-2v567   1/1            RUNNING
java-demo-8567655c52-sp59a   1/1            RUNNING
java-demo-8567655c52-35atw   1/1            RUNNING
my-pod                       2/2            RUNNING

$ kubectl exec -it my-pod -c java-demo bash  # 进入my-pod中的java-demo容器  
[root@my-pod tomcat]$ ls  # 查看容器文件系统
[root@my-pod tomcat]$ ps -ef  # 注意PID=1的进程
[root@my-pod tomcat]$ ifconfig  # 查看内部IP地址
eth0 : inet 10.244.36.68
[root@my-pod tomcat]$ exit

$ kubectl exec -it my-pod -c nginx bash  # 进入my-pod中的nginx容器  
[root@my-pod nginx]$ ifconfig
eth0 : inet 10.244.36.68
```

## 存储共享
同一个Pod的容器之间可以通过Volume数据卷实现存储共享。下面的脚本在my-pod中定义了write和read两个容器。
```bash
$ vi emptyDir.yaml
apiVersion: v1
kind: Pod
metadata:
   name: my-pod
spec:
  containers:
  - name: write
    image: centos
    command: ["bash", "-c", "for i in {1.. 100}; do echo $i >> /data/hello; sleep 1; done"]
   volumeMounts:
     - name: data
       mountPath: /data

  - name: read
    image: centos
    command: ["bash", "-c", "tail -f /data/hello"]
    volumeMounts:
      - name: data
        mountPath: /data
        
 # 在当前Pod存在的节点创建一个名为data的空目录
 # 在两个容器中分别使用mountPath关键字挂载此共享目录 
  volumes:
  - name: data
    emptyDir: {}
```

运行Pod并查看容器共享文件夹。
```bash
$ kubectl delete pod my-pod
$ kubectl apply -f emptyDir.yaml
$ kubectl get pods
$ kubectl exec -it my-pod -c write bash
[root@my-pod /]$ ls /data/
/data/hello
[root@my-pod /]$ exit

$ kubectl exec -it my-pod -c read bash
[root@my-pod /]$ ls /data/
/data/hello
[root@my-pod /]$ exit

$ kubectl logs my-pod -c read  # 查看read容器日志
```

# Pod容器分类与设计模式
容器可以分为：
-	基础容器（Infrastructure Containers）：维护整个Pod网络空间；
-	初始化容器（Init Containers）：初始化容器先于业务容器开始执行；
-	业务容器（Containers）：并行启动。


Pod模板常用功能字段（以product.yaml为例）
```bash
# 定义Deployment控制器相关属性
apiVersion: apps/v1
kind: Deployment
metadata:
   name: product
   namespace: ms
spec:
   replicas: 2
   selector:
     matchLabels:
       project: ms
       app: product
   # 以下是与Pod Template有关的定义
   template: 
     metadata:
       labels:  # 标签
         project: ms
         app: product
     spec:
       imagePollSecrets:  # 拉取镜像策略，包含认证信息
       - name: registry-pull-secret
       containers:  # 业务容器
       - name: product
         image: 192.168.30.201/microservice/product:2020-12-25-21-35-45
         imagePullPolicy: Always  # 镜像拉取策略
         ports:
           - protocol: TCP
             containerPort: 8010
         env:  # 环境变量
           - name: JAVA_OPTS
             value: “-Xmx1g”
         resources:  # 资源限制
           requests:  # 资源分配
             cpu: 0.5
             memory: 256Mi
           limits:  # 限制容器使用的资源上限
             cpu: 1
             memory: 1Gi
         readinessProbe:  # 健康检查，是否准备就绪
             tcpSocket:
               port: 8010
             initialDelaySeconds: 10
             periodSeconds: 10
         # 健康检查，如果Pod状态不健康则Service不会转发流量
          livenessProbe:  
             tcpSocket: 
               port: 8010
             initialDelaySeconds: 60
             periodSeconds: 10
```

使用`kubectl get pods`显示状态为RUNNING并不表示容器内的应用已经成功启动，需要使用`kubectl logs ${podName}`查看日志进一步确认（比如应用使用的端口已经启用）。
