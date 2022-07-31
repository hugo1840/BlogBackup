
@[TOC](Deployment应用生命周期与Pod健康检查)

通过Deployment部署的应用生命周期的一般过程为：
```mermaid
flowchat
st=>start: 部署
e=>end: 下线
op1=>operation: 升级
op2=>operation: 回退
op3=>operation: 扩容

st(right)->op1(right)->op2(right)->op3(right)->e
```


# Deployment应用生命周期
## 部署
通过yaml文件部署项目：
```bash
kubectl apply -f web.yaml
```
或者命令行临时部署（不推荐，一般只用于测试）：
```bash
kubectl create deployment web --image=nginx:1.16 --replicas=3
```

## 升级
应用部署后通常会经历版本升级。
```bash
#方法一：通过配置文件升级
kubectl apply -f xxx.yaml
#方法二：直接升级应用镜像版本
kubectl set image deployment/web nginx=nginx:1.17
#方法三：修改原来部署使用的yaml文件
kubectl edit deployment/web
```
K8S通过滚动发布的方式升级。**滚动发布**是指每次只升级一个或多个服务，升级完成后依次加入生产环境。不断执行这个过程，直到集群中的全部旧版本升级到新版本。

查看滚动升级过程：
```bash
kubectl describe deployment xxx
kubectl rollout status deployment/xxx
```

K8S通过**ReplicaSet**控制器（**RS**）维护Pod的副本数量。RS会不断对比当前Pod的数量与期望的Pod数量。Deployment每次发布都会创建一个RS作为记录，用于实现**滚动升级**和**版本回退**。

如果我们直接删除Deployment中的一个Pod，RS会生成一个新的Pod，以达到期望的Pod数量。
```bash
kubectl delete pod xxx
kubectl get pods
```

查看RS记录：
```bash
kubectl get rs [-n default]
```

查看版本对应RS记录：
```bash
kubectl rollout history deployment xxx
```

## 回退
应用升级失败后，通常要回退应用版本。

查看历史版本：
```bash
kubectl rollout history deployment/web
```

回退到上一个版本：
```bash
kubectl rollout undo deployment/web
```

回滚到指定历史版本：
```bash
kubectl rollout undo deployment/web --to-revision=2
```

查看镜像对应的版本号：
```bash
kubectl describe rs | grep -E "revision|Image"
kubectl describe rs xxx | grep -E "revision|Image"
kubectl describe $(kubectl get rs -o name | grep "web1-") | grep -E "revision|Image"
```

## 扩容
随着业务量的变化，通常需要对应用进行水平扩容（或者缩容）。

方法一：修改yaml配置文件里的`replicas`值，再重新apply。
```bash
kubectl edit deployment/web
kubectl apply -f xxx.yaml
```

方法二：使用`scale`命令直接扩缩容。
```bash
kubectl scale deployment web --raplicas=10
```

## 下线
项目下线后，需要删除不再使用的Deployment和Service。
```bash
kubectl delete deploy/web
kubectl delete svc/web
```
或者
```bash
kubectl delete -f xxx.yaml
```

# Pod健康检查
Pod是K8S创建和管理的最小单元，由一个或多个容器组成。Pod中的容器共享**网络**和**存储**资源。Pod中的容器始终位于同一个Node上，不会跨Node。

可以单独运行一个Pod：
```bash
kubectl apply -f pod.yaml
kubectl run nginx --image=nginx
```
此时的Pod不属于任何一个Deployment，也没有对应的ReplicaSet，因此被删除后不会再自动生成一个新的。

## Sidecar模式
一个Pod中可以运行多个容器。通过在Pod中定义专门容器，来执行主业务容器需要的辅助工作，这种部署Pod的方法被称为**边车模式**（**Sidecar**）。其好处是将辅助功能（如日志收集、应用监控）同主业务容器**解耦**，实现独立发布和能力重用。

## 重启策略
可以为Pod定义重启策略`restartPolicy`，一般可选以下三种模式：
- `Always`：当容器终止退出后，总是重启容器，为默认策略；
- `OnFailure`：当容器异常退出（退出状态码非0）时，才重启容器；
- `Never`：当容器终止退出时，从不重启容器。

## 健康检查
可以为Pod定义健康检查，一般可选以下三种检查探针：
- `livenessProbe`（存活检查）：如果检查失败，将kill容器，并根据**重启策略**进行操作。
- `readinessProbe`（就绪检查）：如果检查失败，会把Pod从Service endpoints中剔除。
- `startupProbe`（启动检查）：启动检查成功才由存活检查接手，用于保护慢启动容器。

健康检查一般可以通过以下三种检查方式实现：
- `httpGet`：发送HTTP请求，返回200-400范围状态码为成功；
- `exec`：执行shell命令返回状态码是0为成功；
- `tcpSocket`：发起TCP Socket建立成功。







