@[TOC](Pod环境变量和initContainer)

# Pod环境变量
Pod中的环境变量主要有以下几种应用场景：
- 容器内应用程序获取Pod信息；
- 容器内应用程序通过用户定义的变量改变默认行为。

Pod环境变量可以按照以下方式定义：
- 自定义变量值；
- 变量值从Pod属性获取；
- 变量值从Secret、ConfigMap获取。

---
**测试**
生成测试用的yaml文件：
```bash
[root@k8s-master ~]# kubectl run pod1 --image=busybox --dry-run=client -o yaml -- sleep 24
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pod1
  name: pod1
spec:
  containers:
  - args:
    - sleep
    - "24"
    image: busybox
    name: pod1
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

通过`env`属性手动定义变量、从Pod属性获取变量值：
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pod1
  name: pod1
spec:
  containers:
  - args:
    - sleep
    - "24h"
    image: busybox
    name: pod1
    env:      #自定义变量
	  - name: testvar1
	    value: 42
	  - name: testvar2
	    value: KillerQueen
      - name: THIS_NODE_NAME   #从Pod属性获取变量值
	    valueFrom:
		  fieldRef:
		    fieldPath: spec.nodeName
	  - name: THIS_POD_NAME
	    valueFrom:
		  fieldRef:
		    fieldPath: metadata.name
	  - name: THIS_POD_NAMESPACE
	    valueFrom:
		  fieldRef:
		    fieldPath: metadata.namespace
	  - name: THIS_POD_IP
	    valueFrom:
		  fieldRef:
		    fieldPath: status.podIP
	  - name: THIS_POD_SRV_ACCOUNT
	    valueFrom:
		  fieldRef:
		    fieldPath: spec.serviceAccountName
```

部署Pod：
```bash
[root@k8s-master ~]# kubectl apply -f pod1.yaml
Error from server (BadRequest): error when creating "pod1.yaml": Pod in version "v1" cannot be handled as a Pod: json: cannot unmarshal number into Go struct field EnvVar.spec.containers.env.value of type string
```

将变量值为数字的加上引号，变为字符串：
```yaml
env:      #自定义变量
  - name: testvar1
    value: "42"
```

重新部署后，检查环境变量：
```bash
[root@k8s-master ~]# kubectl exec -it pod1 -- sh
/ # echo $testvar2
KillerQueen
/ # echo $THIS_NODE_NAME
k8s-node2
/ # echo $THIS_POD_NAME
pod1
/ # echo $THIS_POD_NAMESPACE
default
/ # echo $THIS_POD_SRV_ACCOUNT
default
```


# initContainer
初始化容器（**initContainer**）仅用于同一个Pod中的业务容器运行前的初始化工作，执行完就结束。

- 优先于应用容器执行；
- 支持大部分应用容器配置，但不支持健康检查（lifecycle/livenessProbe/readinessProbe/startupProbe）。

主要应用场景包括：

- 环境检查：比如确保应用容器依赖的服务启动后再启动应用容器；
- 初始化配置：比如给应用准备配置文件。

---
**测试**
定义一个Pod，其中有两个初始化容器，一个等待`myservice`服务，另一个等待`mydb`服务。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app.kubernetes.io/name: MyApp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

运行Pod：
```bash
kubectl apply -f initConainer.yaml
```

查看Pod创建状态：
```bash
[root@k8s-master ~]# kubectl get -f initContainer.yaml
NAME        READY   STATUS     RESTARTS   AGE
myapp-pod   0/1     Init:0/2   0          28s
```

或者
```bash
[root@k8s-master ~]# kubectl describe -f initContainer.yaml
Name:         myapp-pod
Namespace:    default
Status:       Pending
...
Init Containers:
  init-myservice:
    Container ID:  docker://667aa13dc8a9e8a1ecad54d88d0400c3d101e6946c1a41301ba6d9e66858b008
    Image:         busybox:1.28
    ...
    State:          Running
      Started:      Sun, 31 Jul 2022 04:00:04 -0400
    Ready:          False
    ...
  init-mydb:
    Container ID:
    Image:         busybox:1.28
    Image ID:
    ...
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    ...
Containers:
  myapp-container:
    Container ID:
    Image:         busybox:1.28
    Image ID:
    ...
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    ...
Conditions:
  Type              Status
  Initialized       False
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  ...
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  3m40s  default-scheduler  Successfully assigned default/myapp-pod to k8s-node2
  Normal  Pulling    3m40s  kubelet            Pulling image "busybox:1.28"
  Normal  Pulled     3m9s   kubelet            Successfully pulled image "busybox:1.28" in 30.987078455s
  Normal  Created    3m9s   kubelet            Created container init-myservice
  Normal  Started    3m9s   kubelet            Started container init-myservice
```

查看容器日志：
```bash
[root@k8s-master ~]# kubectl logs myapp-pod -c init-myservice 
nslookup: can't resolve 'myservice.default.svc.cluster.local'
waiting for myservice

[root@k8s-master ~]# kubectl logs myapp-pod -c init-mydb
Error from server (BadRequest): container "init-mydb" in pod "myapp-pod" is waiting to start: PodInitializing
```

由于`myservice`和`mydb`都未创建，`init-myservice`一直没有Ready，所以`init-mydb`一直在等待`init-myservice`就绪，而应用容器一直在等两个初始化容器就绪。


接下来我们创建对应的服务：
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
---
apiVersion: v1
kind: Service
metadata:
  name: mydb
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9377
```

暴露服务：
```bash
kubectl apply -f init-services.yaml
```

再次查看应用状态：
```bash
[root@k8s-master ~]# kubectl get -f initContainer.yaml
NAME        READY   STATUS    RESTARTS   AGE
myapp-pod   1/1     Running   0          15m

[root@k8s-master ~]# kubectl describe -f initContainer.yaml
Name:         myapp-pod
Namespace:    default
...
Status:       Running
...
Init Containers:
  init-myservice:
    Container ID:  docker://667aa13dc8a9e8a1ecad54d88d0400c3d101e6946c1a41301ba6d9e66858b008
    ...
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
    Ready:          True
    ...
  init-mydb:
    Container ID:  docker://be037f56bdf5b2c940de7e0516b9167bf70674005fb64986aaed135c7e94d011
    ...
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
    Ready:          True
    ...
Containers:
  myapp-container:
    Container ID:  docker://0083db021fda9835ac7113c61f5a48403a91257b35c6866a9dfa84b8de57a46f
    ...
    State:          Running
      Started:      Sun, 31 Jul 2022 04:14:40 -0400
    Ready:          True
    Restart Count:  0
    ...
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  ...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  15m   default-scheduler  Successfully assigned default/myapp-pod to k8s-node2
  Normal  Pulling    15m   kubelet            Pulling image "busybox:1.28"
  Normal  Pulled     15m   kubelet            Successfully pulled image "busybox:1.28" in 30.987078455s
  Normal  Created    15m   kubelet            Created container init-myservice
  Normal  Started    15m   kubelet            Started container init-myservice
  Normal  Pulled     30s   kubelet            Container image "busybox:1.28" already present on machine
  Normal  Created    30s   kubelet            Created container init-mydb
  Normal  Started    29s   kubelet            Started container init-mydb
  Normal  Pulled     29s   kubelet            Container image "busybox:1.28" already present on machine
  Normal  Created    29s   kubelet            Created container myapp-container
  Normal  Started    28s   kubelet            Started container myapp-container
```

---
**总结**
Pod中会有这几类容器：
- **基础容器**（Infrastructure Container）：维护整个Pod网络空间；
- **初始化容器**（initContainer）：先于业务容器开始串行执行；
- **业务容器**（Container） ：并行启动（Sidecar容器是从业务容器中分离出来的辅助容器）。


