---
tags: [k8s]
title: K8S + GitLab + Jenkins自动化发布项目实践（二）
created: '2023-03-25T06:02:50.788Z'
modified: '2023-03-26T10:58:15.833Z'
---

K8S + GitLab + Jenkins自动化发布项目实践（二）

前置工作：已部署5节点k8s集群，并搭建了代码仓库和镜像仓库（GitLab + Harbor）。

| 主机名 | IP | 角色 |
| :--: | :--: | :--: | 
| k8s-master1 | 192.168.124.a | k8s控制平面 |
| k8s-master2 | 192.168.124.b | k8s控制平面 |
| k8s-master3 | 192.168.124.c | k8s控制平面 |
| k8s-worker1 | 192.168.124.d | k8s工作节点 |
| k8s-worker2 | 192.168.124.e | k8s工作节点 |
| harborgit | 192.168.124.f | 代码仓库 & 镜像仓库 |

# Jenkins容器化部署
Jenkins是一款开源的、用于自动化构建、测试和部署的CI-CD工具。

## 部署NFS PV存储
>:fire:配置参考：https://blog.csdn.net/Sebastien23/article/details/126276294

在工作节点k8s-worker1上部署NFS服务器：
```bash
[root@k8s-worker1 ~]# mkdir -p /ifs/kubernetes
[root@k8s-worker1 ~]# yum install nfs-utils -y
[root@k8s-worker1 ~]# vi /etc/exports
/ifs/kubernetes *(rw,sync,no_root_squash)

[root@k8s-worker1 ~]# systemctl start nfs
[root@k8s-worker1 ~]# systemctl enable nfs
```

在其他工作节点挂载NFS共享目录：
```bash
[root@k8s-worker2 ~]# mkdir -p /ifs/kubernetes
[root@k8s-worker2 ~]# yum install nfs-utils -y
[root@k8s-worker2 ~]# echo '192.168.124.d:/ifs/kubernetes  /ifs/kubernetes  nfs4  defaults  0 0' >> /etc/fstab
[root@k8s-worker2 ~]# mount -a
```

修改目录权限：
```bash
[root@k8s-worker1 ~]# mkdir /ifs/kubernetes/jenkins
[root@k8s-worker1 ~]# chmod -R 777 /ifs/kubernetes
```

定义一个nfs类型的PV资源：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-jenkins
spec:
  capacity:
    storage: 5Gi       
  accessModes:
    - ReadWriteMany     
  nfs:
    path: /ifs/kubernetes/jenkins
  server: 192.168.124.d
```

创建PV资源：
```bash
[root@k8s-master1 jenkins]# kubectl apply -f pv-nfs-jenkins.yml
persistentvolume/pv-nfs-jenkins created
[root@k8s-master1 jenkins]# kubectl get pv
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-nfs-jenkins   5Gi        RWX            Retain           Available                                   6s
```

## Jenkins部署
通过Deployment控制器来部署Jenkins官方镜像，会对外暴露**8080**（Web访问端口）和**50000**（Slave通信端口）两个端口。Jenkins容器的数据存储默认在`/var/jenkins_home`目录下，需要对该目录进行PV持久化。

>:train:官方镜像地址：https://hub.docker.com/r/jenkins/jenkins
>:train:部署配置：https://www.jenkins.io/doc/book/installing/kubernetes/

我们使用`jenkins.yml`来部署Jenkins容器，通过PVC来请求PV资源，同时定义好Service访问和RBAC服务账号权限。
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      name: jenkins
  template:
    metadata:
      name: jenkins
      labels:
        name: jenkins
    spec:
      serviceAccountName: jenkins
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts
          ports:
            - containerPort: 8080
            - containerPort: 50000
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
      securityContext:
        fsGroup: 1000
      volumes:
      - name: jenkins-home
        persistentVolumeClaim:
          claimName: jenkins

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi

---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  selector:
    name: jenkins
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
      nodePort: 30008
    - name: agent
      port: 50000
      protocol: TCP

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins

---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jenkins
rules:
- apiGroups: [""]
  resources: ["pods","events"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["secrets","events"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins
subjects:
- kind: ServiceAccount
  name: jenkins
```

部署Jenkins容器：
```bash
[root@k8s-master1 jenkins]# kubectl apply -f jenkins.yml
deployment.apps/jenkins created
persistentvolumeclaim/jenkins created
service/jenkins created
serviceaccount/jenkins created
role.rbac.authorization.k8s.io/jenkins created
rolebinding.rbac.authorization.k8s.io/jenkins created
```

部署完成后，通过查看Pod日志来获取Jenkins初始化的管理员口令：
```bash
kubectl get pods 
kubectl logs <jenkins-pod-name> 

# 也可以在宿主机的以下路径找到
cat /ifs/kubernetes/jenkins/secrets/initialAdminPassword
```

## Jenkins初始化
通过暴露的NodePort访问Jenkins首页，我这里是`http://192.168.124.a:30008`。
```bash
[root@k8s-master1 jenkins]# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                        AGE
jenkins      NodePort    10.106.152.176   <none>        80:30008/TCP,50000:30282/TCP   22m
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP                        11d
```

在自定义Jenkins界面，选择插件来安装（不要安装推荐的插件），选择“无”来取消自动勾选的所有插件。

![jenkins-01](https://img-blog.csdnimg.cn/d7340aeb0fc54702ac1953fef7206cf2.png#pic_center)

定义好管理员用户和实例信息后，来到Jenkins首页。

![jenkins-02](https://img-blog.csdnimg.cn/89bec9e1479a4adabc6b9b602a51f6a9.png#pic_center)

## 安装Jenkins插件
在**Manage Jenkins**$\rightarrow$**System Configuration**$\rightarrow$**Manage Plugins**$\rightarrow$**Available plugins**中分别搜索安装以下插件（install without restart）：
- Git：用于拉取代码。
- Git Parameter：Git参数化构建。
- Pipeline：流水线。
- kubernetes：连接k8s动态创建Slave代理。
- Config File Provider：存储配置文件。
- Extended Choice Parameter：扩展选择框参数。

![jenkins-03](https://img-blog.csdnimg.cn/bdfe1b79f5854c53b19076e93e18bd99.png#pic_center)

Jenkins下载插件默认服务器在国外，如果下载速度很慢，可以修改为国内源：
```bash
cd /ifs/kubernetes/jenkins/updates
sed -i 's/https:\/\/updates.jenkins.io\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json
```

然后通过浏览器重启Jenkins：
```
http://NodeIP:30008/restart
```

**注**：如果重启Jenkins Pod所在服务器后，Jenkins容器状态一直为Unknown，可以直接删除Pod，Deployment控制器会重新再创建一个新的Pod。而且由于`/var/jenkins_home`被挂载在NFS存储中，也不用担心丢失Jenkins数据。
```bash
[root@k8s-master1 jenkins]# kubectl get pods,deploy
NAME                            READY   STATUS              RESTARTS   AGE
pod/jenkins-5bddd99684-w9glp    0/1     Unknown             5          11h
pod/jenkins-slave-nhfbm-0xd6f   0/1     ContainerCreating   0          58m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/jenkins   0/1     1            0           11h

[root@k8s-master1 jenkins]# kubectl delete pod/jenkins-5bddd99684-w9glp
pod "jenkins-5bddd99684-w9glp" deleted
```


# Jenkins主从架构配置
Master（Jenkins本身）提供Web页面来让用户管理项目和Slave（从节点）。项目任务可以运行在Master本机，也可以分配到从节点运行。一个Master可以关联多个Slave。Slave节点起到了分担工作任务和隔离构建环境的作用。

当触发Jenkins任务时，Jenkins会调用Kubernetes API创建Slave Pod。Slave Pod启动后会连接Jenkins，接受任务处理。

## Kubernetes插件配置
Kubernetes插件用于Jenkins在k8s集群中运行动态代理。

>:sunflower:插件介绍：https://github.com/jenkinsci/kubernetes-plugin

配置插件：**Manage Jenkins**$\rightarrow$**Manage Nodes and Clouds**$\rightarrow$**Configure Clouds**中添加Kubernetes。

点击打开Kubernetes Cloud Details，输入Kubernetes地址：
```
https://kubernetes.default
```
表示default命名空间中的kubernetes服务，并点击“连接测试”，出现`Connected to Kubernetes v1.x.x`即表示连接成功。

最后输入Jenkins地址并保存。
```
http://jenkins.default
```

## 安装nerdctl工具 
Jenkins Slave工作需要用到OpenJDK、Maven、git命令、kubectl命令、以及docker build和docker push命令。由于我们的k8s集群使用的容器运行时为containerd，自带的crictl工具不支持build和push操作，因此需要安装nerdctl工具。

>:bear:Nerdctl官方下载地址：https://github.com/containerd/nerdctl/releases

查看containerd版本：
```bash
kubectl get nodes -o wide
#CONTAINER-RUNTIME中的版本信息为containerd://1.6.18
```

下载兼容的nerdctl版本，并解压到`/usr/local`目录下：
```bash
tar Cxzvvf /usr/local nerdctl-full-1.2.1-linux-amd64.tar.gz
```

启动nerdctl和buildkit服务：
```bash
nerdctl run -d --name nginx -p 80:80 nginx:alpine

# nerdctl使用buildkit服务来build镜像
systemctl enable buildkit.service --now
```

## 自定义Jenkins Slave镜像
用于构建Jenkins Slave镜像的Dockerfile如下：
```
FROM centos:7
LABEL maintainer kratos

RUN yum install -y java-1.8.0-openjdk maven git libtool-ltdl-devel && \
    yum clean all && \
    rm -rf /var/cache/yum/* && \
    mkdir -p /usr/share/jenkins

COPY slave.jar /usr/share/jenkins/slave.jar
COPY jenkins-slave /usr/bin/jenkins-slave
COPY settings.xml /etc/maven/settings.xml
RUN chmod +x /usr/bin/jenkins-slave
COPY kubectl /usr/bin/

ENTRYPOINT ["jenkins-slave"]
```

其中：
- `slave.jar`为agent程序，用于接收master下发的任务。
- `jenkins-slave`是用于启动`slave.jar`的Shell脚本。
- `settings.xml`配置中需要将Maven官方源修改为阿里云源。
- 另外还需要`kubectl`客户端工具。

在Dockerfile同级目录下准备好要打包进镜像的文件。
```bash
[root@k8s-master1 jenkins]# ls
Dockerfile  jenkins-slave  kubectl  settings.xml  slave.jar
```

构建镜像并推送到Harbor镜像仓库：
```bash
# 注意命令最后面的'.'
nerdctl build -t 192.168.124.f/library/jenkins-slave-jdk:1.8 .
nerdctl push 192.168.124.f/library/jenkins-slave-jdk:1.8
```
推送成功后在Harbor仓库中可以看到上传的镜像。

如果推送失败，报错类似如下：
```bash
ERRO[0000] server "192.168.124.f" does not seem to support HTTPS  error="failed to do request: ...
... dial tcp 192.168.124.f:443: connect: connection refused
... unexpected status from GET request to https://auth.docker.io/token?offline_token=true&service=registry.docker.io: 401 Unauthorized
```

则需要配置容器运行时的HTTP传输。如果是docker，只需要在`/etc/docker/daemon.json`中增加`insecure-registries`并配置Harbor地址，然后重启docker服务即可。Containerd则需要修改`/etc/containerd/config.toml`中的多项配置。

```bash
# 备份containerd配置文件
[root@k8s-master1 jenkins]# cp /etc/containerd/config.toml /etc/containerd/config.toml.bak

# 修改containerd配置文件
[root@k8s-master1 jenkins]# vi /etc/containerd/config.toml
...
      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.harborgit".tls]
          insecure_skip_verify = true     # 跳过TLS认证
        [plugins."io.containerd.grpc.v1.cri".registry.configs."harbor.harborgit".auth]
          username = "admin"      # 配置Harbor登陆用户和密码
          password = "XXXXXX"

      [plugins."io.containerd.grpc.v1.cri".registry.headers]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."harbor.haborgit"]
          endpoint = ["http://192.168.124.f"]     # 配置Harbor访问地址
...

# 重启containerd
[root@k8s-master1 jenkins]# systemctl restart containerd
```

登录后重新推送即可（要加`--insecure-registry`）：
```bash
[root@k8s-master1 jenkins]# nerdctl login http://192.168.124.f --insecure-registry
[root@k8s-master1 jenkins]# nerdctl push 192.168.124.f/library/jenkins-slave-jdk:1.8 --insecure-registry
```

>:coffee:参考：https://goharbor.io/docs/2.0.0/install-config/run-installer-script/#connect-http


## 测试主从架构是否正常
点击**New Item**输入名称`java-demo`，选中**Pipeline**再点击OK确认。创建完成后，在**java-demo**$\rightarrow$**Configure**$\rightarrow$**Advanced Project Options**$\rightarrow$**Pipeline**中选择“Pipeline Script”。

将下面的测试脚本粘贴进去并保存。
```
pipeline {
  agent {
    kubernetes {
      label "jenkins-slave"
      yaml '''
apiVersion: v1
kind: Pod
metadata:
  name: jenkins-slave
spec:
  containers:
  - name: jnlp
    image: "192.168.124.f/library/jenkins-slave-jdk:1.8"
'''
    }
  }
  stages {
    stage('Main'){
      steps {
        sh 'hostname'
      }
    }
  }
}
```
保存Pipeline脚本后，在新建项目中点击**Build Now**，然后在Build History中查看Console Output日志，直到输出Build成功的信息。

可以通过`kubectl get pods`命令检查k8s集群中是否会自动创建和销毁jenkins-slave Pod。

如果Console Output中收到以下报错，不一定是服务账号权限的原因。
```bash
Failure executing: 
POST at: https://kubernetes.default/api/v1/namespaces/default/pods. 
Message: Unauthorized! Configured service account doesn't have access. 
Service account may have been revoked. Unauthorized.
```
具体原因我们下次进一步排查。








