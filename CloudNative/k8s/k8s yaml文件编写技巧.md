@[TOC](k8s yaml文件编写技巧)

# yaml文件格式
 - 缩进表示层级关系；
 - 不支持Tab缩进，只能使用空格缩进；
 - 通常开头缩进2个空格；
 - 字符后缩进1个空格，例如冒号、逗号后面；
 - 使用`#`注释；
 - `---`表示一个文件的开始，在同一个yaml文件中可以分隔不同的资源。
 
 
## deployment.yaml
编写一个应用部署文件`deployment.yaml`。

```yaml
---
#控制器定义
apiVersion: apps/v1   #api版本
kind: Deployment      #资源类型
metadata:             #资源的元数据
  name: web           
  namespace: default
spec:                 #资源规格
  replicas: 3         #副本数量
  selector:           #标签选择器
    matchLabels:      #与下面Pod的labels保持一致
	  app: web
  #被控制对象
  template:           #Pod模板
    metadata:
	  labels:         #Pod的标签
	    app: web
	spec:
	  containers:     #容器配置
	  - name: web
	    image: nginx:1.20  #容器使用的镜像
```

执行上面的yaml文件
```bash
kubectl apply -f deployment.yaml
```
相当于
```bash
kubectl create deployment web --image=nginx:1.20 --replicas=3 -n default
```

## service.yaml
编写一个对外暴露服务文件`service.yaml`。
```yaml
---
apiVersion: v1
kind: Service
metadata: 
  name: web
  namespace: default
spec:
  ports:
  - port: 80          #Service端口，通过ClusterIP访问
    protocol: TCP
	targetPort: 8080  #Pod内部署应用的服务端口，比如nginx是80
  selector:           #标签选择器，与deployment中保持一致
    app: web
  type: NodePort      #Service类型
```

执行上面的yaml文件
```bash
kubectl apply -f service.yaml
```
相当于
```bash
kubectl expose deployment web --port=80 --target-port=8080 --type=NodePort -n default
```

Service和Deployment通过**Labels**标签关联，因此：
- 同一个应用的Deployment、Pod、Service应该使用**相同的**标签；
- 不同应用应该使用**不同的**标签，否则可能会导致同一个Service关联到多个应用。

删除已经部署的应用
```bash
kubectl delete -f deploymeny.yaml
```
删除服务
```bash
kubectl delete -f service.yaml
```

# 使用kubectl create部署应用
部署应用
```bash
kubectl create deployment web --image=xhliang/java-app
kubectl get deployment,pods
```

对外发布服务
```bash
kubectl expose deployment web --port=80 --target-port=8080 --type=NodePort --name=web
kubectl get pods,svc
```

查看Service关联的Pod
```bash
kubectl get endpoints   #缩写为 kubectl get ep
kubectl get pods -o wide
```

# 自动生成yaml文件
在日常运维中，从零开始写一个yaml文件的效率太低，而且容易出错。一个很好的方法就是利用kubectl命令来自动生成部署用的yaml文件，然后根据需求进行修改即可。

方法一：利用`kubectl create ... --dry-run -o yaml > xxx.yaml`生成一个部署文件。

`--dry-run=client`表示在本地尝试运行，但是不会实际部署。

```bash
kubectl create deployment nginx --image=nginx:1.16 -o yaml --dry-run=client > deployment.yaml
```

方法二：利用`kubectl get ... -o yaml > xxx.yaml`命令导出已有部署的yaml文件。
```bash
kubectl get deployment nginx -o yaml > deployment.yaml
```

查询资源支持的属性字段：
```bash
kubectl explain pods.spec.containers
kubectl explain deployment
```




