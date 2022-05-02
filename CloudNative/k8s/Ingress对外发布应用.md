@[TOC](Ingress对外发布应用)

# NodePort存在的问题
通过Service的NodePort发布应用可能存在以下问题：
-	端口冲突，每创建一个Service就会占用一个端口，因此需要做好端口的分配与管理；
-	NodePort使用的是**四层负载均衡**（即传输层，通过IP+端口号转发流量），无法使用域名或URL进行转发；
-	无法为集群内所有的应用/Pod做统一的代理（统一的访问入口）。

# Ingress对外暴露应用
Ingress即是为了解决NodePort的以上问题应运而生。

## Pod与Ingress的关系
 
Ingress与Pod的关系的几个重要特点如下：
-	Ingress通过 **Service** 关联Pod；
-	基于**域名**访问；
-	通过**Ingress Controller**实现Pod的**负载均衡**（支持 TCP/UDP 四层和HTTP **七层**）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210215212031311.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)


## Ingress Controller部署
Ingress控制器的拓扑图如下。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210215211948346.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)

 
Ingress不是一个负载均衡的实现，而仅仅是一系列定义的规则。Ingress通过Ingress Controller实现负载均衡。Ingress控制器部署在每个Node节点上。Ingress控制器会提供一个统一的访问入口，通常是80端口（HTTP）或者443端口（HTTPS）。


**部署Ingress控制器**

使用yaml文件配置Ingress控制器（官方维护的版本是基于nginx实现的）。
```bash
# ingress-controller.yaml
# 指定ingress控制器所处的命名空间
apiVersion: v1
kind: Namespace
metadata:
   name: ingress-nginx
   labels:
     app.kubernetes.io/name: ingress-nginx
     app.kubernetes.io/part-of: ingress-nginx

- - -
kind: ConfigMap
apiVersion: v1
metadata:
   name: nginx-configuration  # nginx配置文件
   namespace: ingress-nginx
   labels:
     app.kubernetes.io/name: ingress-nginx
     app.kubernetes.io/part-of: ingress-nginx

- - -
kind: ConfigMap
apiVersion: v1
metadata:
   name: tcp-services  # TCP服务配置文件
   namespace: ingress-nginx
   labels:
     app.kubernetes.io/name: ingress-nginx
     app.kubernetes.io/part-of: ingress-nginx

- - -
kind: ConfigMap
apiVersion: v1
metadata:
   name: udp-services  # UDP服务配置文件
   namespace: ingress-nginx
   labels:
     app.kubernetes.io/name: ingress-nginx
     app.kubernetes.io/part-of: ingress-nginx

- - -
apiVersion: v1
kind: ServiceAccount
metadata:
   name: nginx-ingress-serviceaccount
   namespace: ingress-nginx
   labels:
     app.kubernetes.io/name: ingress-nginx
     app.kubernetes.io/part-of: ingress-nginx

- - -
# ingress控制器访问master节点需要的授权rbac
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
   name: nginx-ingress-clusterrole
   labels:
     app.kubernetes.io/name: ingress-nginx
     app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
     - " "
    resources:
     - configmaps
     - endpoints
     - nodes
     - pods
     - secrets
    verbs:
     - list
     - watch
…  # 此处省略了内容

- - -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role  # 角色权限的集合
metadata:
   name: nginx-ingress-clusterrole
   namespace: ingress-nginx
   labels:
     app.kubernetes.io/name: ingress-nginx
     app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
     - " "
    resources:
  …
    verbs:
  …
… # 此处省略了内容

- - -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding  # 角色绑定到指定账户
metadata:
   name: nginx-ingress-role-nisa-binding
   namespace: ingress-nginx
   labels:
     app.kubernetes.io/name: ingress-nginx
     app.kubernetes.io/part-of: ingress-nginx
roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: Role
   name: nginx-ingress-role
subjects:
   - kind: ServiceAccount
     name: nginx-ingress-serviceaccount
     namespace: ingress-nginx
…  # 此处省略了内容

- - -
# 部署Ingress控制器镜像
apiVersion: apps/v1
kind: DaemonSet
metadata:
   name: nginx-ingress-controller
   namespace: ingress-nginx
   labels:
     app.kubernetes.io/name: ingress-nginx
     app.kubernetes.io/part-of: ingress-nginx
spec:
  selector:
    matchLabels:
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
…
… # 此处省略了内容

- - -
apiVersion: v1
kind: Service
metadata:
   name: nginx-ingress
   namespace: ingress-nginx
spec:
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  - name: https
    port: 443
    targetPort: 443
    protocol: TCP
    selector:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
      
```

通过配置文件部署ingress控制器：`kubectl apply -f ingress-controller.yaml`

查看ingress控制器状态：`kubectl get pods -n ingress-nginx`


## Ingress创建规则
```bash
# ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
   name: web
spec:
  rules:
  - host: example.xxx.com  # 为部署的应用指定域名
   http:
     paths:
     - backend:
         serviceName: web  # 指定应用关联的Service
         servicePort: 80  # 指定Service使用的端口
```

通过配置文件创建规则：`kubectl apply -f ingress.yaml`

查看创建的ingress规则：`kubectl get ingress`

在Windows客户端主机的`C:\Windows\System32\drivers\etc\hosts`文件中添加一行：`任一节点IP   example.xxx.com`。然后打开cmd检查是否能直接ping通域名。也可以直接在浏览器输入域名检查是否能访问。

