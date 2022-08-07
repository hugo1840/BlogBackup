@[TOC](Ingress：比Service更强大的服务暴露与负载均衡)

根据OSI七层模型：
>物理层$\rightarrow$数据链路层$\rightarrow$网络层$\rightarrow$传输层$\rightarrow$会话层$\rightarrow$表示层$\rightarrow$应用层

负载均衡可以分为4层LB和7层LB两种类型：
- **四层负载均衡（传输层）**：基于IP和端口实现转发。支持四层转发的有Nginx、Haproxy、LVS、IPVS、iptables等。
- **七层负载均衡（应用层）**：基于应用层协议实现代理，例如HTTP协议（域名、URL）。支持七层代理的有Nginx、Haproxy等。

使用Service NodePort对外暴露服务存在的不足：
- 一个端口只能给一个服务使用，需要提前规划。服务很多时，端口管理简直灾难；
- 只支持四层负载均衡。


# Ingress是什么
Ingress是k8s中的一个抽象资源，提供了一个暴露应用的入口定义方法。Ingress控制器会提供一个统一的访问入口，通常是80端口（HTTP）或者443端口（HTTPS）。Ingress Controller负责路由流量，根据Ingress生成具体的路由规则，并对Pod负载均衡。Ingress控制器管理的负载均衡器，为集群提供全局的负载均衡能力。

>官方维护的Nginx控制器的项目为：
https://github.com/kubernetes/ingress-nginx


# 部署Ingress Controller
>部署方法参考：`https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal-clusters`
版本兼容参考：`https://github.com/kubernetes/ingress-nginx#support-versions-table`
国内镜像地址：`registry.cn-hangzhou.aliyuncs.com/google_containers/`

查看k8s版本：
```bash
kubectl version
```

下载对应版本兼容的部署文件：
```bash
wget -O ingress-deploy.yaml https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.2/deploy/static/provider/baremetal/deploy.yaml
```

官方的镜像地址由于网络原因无法下载，可以替换为阿里云的镜像地址：
```
image: k8s.gcr.io/ingress-nginx/controller:v1.1.2@sha256:28b11ce69e57843de44e3db6413e98d09de0f6688e33d4bd384002a44f78405c
...
image: k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1@sha256:64d8c73dca984af206adf9d6d7e46aa550362b1d7a01f3a0a91b20cc67868660
#分别替换为
image: registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:v1.1.2
...
image: registry.cn-hangzhou.aliyuncs.com/google_containers/kube-webhook-certgen:v1.1.1
```

部署Ingress控制器：
```bash
kubectl apply -f ingress-deploy.yaml
```

查看部署结果：
```bash
[root@k8s-master ~]# kubectl get deploy,po,svc -n ingress-nginx
NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           2m34s

NAME                                           READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-4ffbq       0/1     Completed   0          2m34s
pod/ingress-nginx-admission-patch-svtjc        0/1     Completed   0          2m34s
pod/ingress-nginx-controller-5558df4df-jk7xp   1/1     Running     0          2m34s

NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.111.4.120     <none>        80:31924/TCP,443:32021/TCP   2m34s
service/ingress-nginx-controller-admission   ClusterIP   10.100.224.180   <none>        443/TCP                      2m34s
```

上面的状态表示部署成功。


# 创建Ingress rules
使用Ingress之前需要创建规则。

通过命令行创建Ingress规则：
```bash
kubectl create ingress 
```

通过配置文件创建Ingress规则：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress-playroom
spec:
  ingressClassName: nginx     #指定ingress控制器
  rules:
  - host: web.myingressplayroom.com  #指定域名
    http:
      paths:
      - path: /               #转发路径
	pathType: Prefix
        backend:
          service:
	    name: nginx-web   #指定Service名称
	    port:
	      number: 80      #指定Service端口
```

Ingress控制器基于域名做分流；创建Ingress规则时如果不指定域名，集群中只会有一个Ingress规则生效。

Ingress通过Service关联Pod；创建Ingress规则时需要指定Service名称，Service配置为ClusterIP即可。


使Ingress规则生效：
```bash
kubectl apply -f ingress-rules.yaml
```

查看创建的Ingress规则：
```bash
[root@k8s-master ~]# kubectl get ingress
NAME                  CLASS   HOSTS                       ADDRESS           PORTS   AGE
my-ingress-playroom   nginx   web.myingressplayroom.com   192.168.x.y       80      7s
```


# 暴露Ingress Controller的方式

## 通过Service NodePort
通过Service NodePort对外暴露Ingress Controller的访问流程为：
>用户访问域名$\rightarrow$公网LB$\rightarrow$Service NodePort$\rightarrow$Ingress Controller所在Pod$\rightarrow$基于域名分流$\rightarrow$应用所在的Pod

接上一小节配置完Ingress规则后，在本地配置域名解析：
```
#在C:\Windows\System32\drivers\etc\hosts中添加域名解析
#因为使用了NodePort，这里IP为任意Node节点都可
192.168.x.y  web.myingressplayroom.com
```

在浏览器中访问`<域名>:<Ingress-Controller-NodePort>`：
```
http://web.myingressplayroom.com:31924
https://web.myingressplayroom.com:32021
#两者都会显示Welcome to Nginx!
```

**实验**
再次创建一个apache httpd应用：
```bash
kubectl create deployment web-httpd --image=httpd
kubectl expose deployment web-httpd --port=80 --target-port=80
kubectl get pods,svc
```

配置Ingress规则：
```bash
[root@k8s-master ~]# cp ingress-rules1.yaml ingress-rules2.yaml
[root@k8s-master ~]#
[root@k8s-master ~]# vim ingress-rules2.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: httpd-ingress-playroom
spec:
  ingressClassName: nginx     #指定ingress控制器
  rules:
  - host: httpd.ingressplayroom.com  #指定域名
    http:
      paths:
      - path: /               #转发路径
        pathType: Prefix
        backend:
          service:
            name: web-httpd   #指定Service名称
            port:
              number: 80      #指定Service端口
```

配置生效：
```bash
kubectl apply -f ingress-rules2.yaml
```

配置域名解析：
```
192.168.x.y  web.myingressplayroom.com  httpd.ingressplayroom.com
```

在浏览器中访问`<域名>:<Ingress-Controller-NodePort>`：
```
http://httpd.ingressplayroom.com:31924/ 
https://httpd.ingressplayroom.com:32021/
#两者都会显示It works!
```

查看Ingress资源的具体描述：
```bash
[root@k8s-master ~]# kubectl describe ingress httpd-ingress-playroom
Name:             httpd-ingress-playroom
Namespace:        default
Address:          192.168.136.120
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host                       Path  Backends
  ----                       ----  --------
  httpd.ingressplayroom.com
                             /   web-httpd:80 (10.244.169.170:80)
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    23m (x2 over 24m)  nginx-ingress-controller  Scheduled for sync

[root@k8s-master ~]# kubectl describe ingress my-ingress-playroom
Name:             my-ingress-playroom
Namespace:        default
Address:          192.168.136.120
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host                       Path  Backends
  ----                       ----  --------
  web.myingressplayroom.com
                             /   nginx-web:80 (10.244.169.165:80,10.244.36.87:80,10.244.36.88:80)
Events:                      <none>
```


## 通过共享主机网络
通过共享主机网络对外暴露Ingress Controller，需要事先配置`hostNetwork: True`。访问流程为：
>用户访问域名$\rightarrow$公网LB$\rightarrow$宿主机80/443端口$\rightarrow$Ingress Controller所在Pod$\rightarrow$基于域名分流$\rightarrow$应用所在的Pod

修改Ingress Controller部署文件，为Ingress控制器的容器配置`hostNetwork: True`。
```bash
[root@k8s-master ~]# vim ingress-deploy.yaml
spec:
      hostNetwork: True   #启用共享宿主机网络
      containers:
      - args:
        - /nginx-ingress-controller
	    ...
```

更新Ingress Controller配置：
```bash
kubectl apply -f ingress-deploy.yaml
```

查看Ingress控制器更新结果：
```bash
[root@k8s-master ~]# kubectl get pods -n ingress-nginx
NAME                                           READY   STATUS        RESTARTS       AGE
pod/ingress-nginx-admission-create-4ffbq       0/1     Completed     0              6h39m
pod/ingress-nginx-admission-patch-svtjc        0/1     Completed     0              6h39m
pod/ingress-nginx-controller-5558df4df-jk7xp   1/1     Terminating   1 (3h1m ago)   6h39m
pod/ingress-nginx-controller-d8d4889db-7jd7g   1/1     Running       0              12s
```

Ingress Contoller更新完成后，可以发现它所在Pod绑定的IP不再是10开头的集群内部IP，而是变成了node1的宿主机IP。
```bash
[root@k8s-master ~]# kubectl get pods -o wide -n ingress-nginx
NAME                                       READY   STATUS      RESTARTS   AGE     IP                NODE        NOMINATED NODE   READINESS GATES
ingress-nginx-admission-create-4ffbq       0/1     Completed   0          6h40m   10.244.169.169    k8s-node2   <none>           <none>
ingress-nginx-admission-patch-svtjc        0/1     Completed   0          6h40m   10.244.36.91      k8s-node1   <none>           <none>
ingress-nginx-controller-d8d4889db-7jd7g   1/1     Running     0          31s     192.168.136.120   k8s-node1   <none>           <none>
```

配置域名解析：
```
192.168.136.120  web.myingressplayroom.com  httpd.ingressplayroom.com
```
这里配置的IP为ingress controller所在宿主机的IP。由于没有使用到Service，如果配置成其他Node节点的IP将会无法访问。

浏览器直接访问域名（无需指定NodePort端口）：
```
web.myingressplayroom.com   #显示Welcome to Nginx!
httpd.ingressplayroom.com   #显示It works!
```

检查各节点是否监听了80和443端口：
```bash
[root@k8s-node1 ~]#  ss -antp | grep :80
LISTEN     0      128          *:80                       *:*                   users:(("nginx",pid=27992,fd=19),("nginx",pid=27987,fd=19))
LISTEN     0      128          *:80                       *:*                   users:(("nginx",pid=27991,fd=11),("nginx",pid=27987,fd=11))
LISTEN     0      128       [::]:80                    [::]:*                   users:(("nginx",pid=27991,fd=12),("nginx",pid=27987,fd=12))
LISTEN     0      128       [::]:80                    [::]:*                   users:(("nginx",pid=27992,fd=20),("nginx",pid=27987,fd=20))

[root@k8s-node1 ~]#  ss -antp | grep :443
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=27992,fd=21),("nginx",pid=27987,fd=21))
LISTEN     0      128          *:443                      *:*                   users:(("nginx",pid=27991,fd=13),("nginx",pid=27987,fd=13))
ESTAB      0      0      192.168.136.120:43340              10.96.0.1:443                 users:(("nginx-ingress-c",pid=27947,fd=3))
LISTEN     0      128       [::]:443                   [::]:*                   users:(("nginx",pid=27991,fd=14),("nginx",pid=27987,fd=14))
LISTEN     0      128       [::]:443                   [::]:*                   users:(("nginx",pid=27992,fd=22),("nginx",pid=27987,fd=22))
```
可以发现只有ingress controller所在的Node节点监听了自己的80和443端口。因此域名解析时必须使用该节点的IP。

为避免端口冲突，每个Node节点只能运行一个Ingress Controller的副本。如果要使用共享主机网络模式部署多个Ingress Controller副本，可以采用DaemonSet控制器部署。


# Ingress规则：HTTPS证书配置
配置HTTPS步骤：
1. 准备域名证书文件，可以来自openssl/cfssl工具自签、或者权威机构颁发；
2. 将证书文件保存到Secret；
3. Ingress规则配置TLS。

安装cfssl工具：
```bash
[root@k8s-master ~]# tar zvxf cfssl.tar.gz
[root@k8s-master ~]# mv cfssl* /usr/bin/
```

配置自签证书：
```bash
[root@k8s-master ~]# mkdir ssl 
[root@k8s-master ~]# vim ssl/certs.sh
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

cat > web.myingressplayroom.com-csr.json <<EOF
{
  "CN": "web.myingressplayroom.com",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes web.myingressplayroom.com-csr.json | cfssljson -bare web.myingressplayroom.com
```

生成自签证书：
```bash
[root@k8s-master ssl]# chmod +x certs.sh
[root@k8s-master ssl]# ./certs.sh
[root@k8s-master ssl]# ls
ca-config.json  ca-csr.json  ca.pem    web.myingressplayroom.com.csr       web.myingressplayroom.com-key.pem
ca.csr          ca-key.pem   certs.sh  web.myingressplayroom.com-csr.json  web.myingressplayroom.com.pem
```

将证书保存到k8s Secret中：
```bash
[root@k8s-master ssl]# kubectl create secret tls my-ingress-playroom --cert=web.myingressplayroom.com.pem --key=web.myingressplayroom.com-key.pem
secret/my-ingress-playroom created
[root@k8s-master ssl]# kubectl get secret
NAME                  TYPE                                  DATA   AGE
default-token-jwvc6   kubernetes.io/service-account-token   3      15d
my-ingress-playroom   kubernetes.io/tls                     2      3s
```

在Ingress规则文件中配置Secret：
```
[root@k8s-master ~]# vim ingress-rules1.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress-playroom
spec:
  ingressClassName: nginx     #指定ingress控制器
  tls:
  - hosts:
      - web.myingressplayroom.com    #已生成证书的域名
    secretName: my-ingress-playroom  #已创建的secret资源
  rules:
  - host: web.myingressplayroom.com  #指定域名
    http:
      paths:
      - path: /               #转发路径
        pathType: Prefix
        backend:
          service:
            name: nginx-web   #指定Service名称
            port:
              number: 80      #指定Service端口
```

启用Secret配置：
```bash
[root@k8s-master ~]# kubectl apply -f ingress-rules1.yaml
ingress.networking.k8s.io/my-ingress-playroom configured
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl get ingress
NAME                     CLASS   HOSTS                       ADDRESS           PORTS     AGE
httpd-ingress-playroom   nginx   httpd.ingressplayroom.com   192.168.136.121   80        6h4m
my-ingress-playroom      nginx   web.myingressplayroom.com   192.168.136.121   80, 443   7h52m
```

浏览器https访问，可以查看颁发的自签证书。
```
#NodePort模式
https://web.myingressplayroom.com:32021/
#共享主机网络模式
web.myingressplayroom.com
```


**总结**
Ingress Controller的Pod中运行有两种应用：
- `nginx-ingress-controller`：调用k8s的api动态地获取创建的ingress资源，生成nginx server配置，并应用到管理的nginx服务，然后热加载生效；
- `nginx`：实现代理转发和负载均衡。

```bash
[root@k8s-master ~]# kubectl exec -it ingress-nginx-controller-7c95f9b589-qfpr8 -n ingress-nginx -- ps -ef | grep nginx
    1 www-data  0:00 /usr/bin/dumb-init -- /nginx-ingress-controller --election
    6 www-data  0:05 /nginx-ingress-controller --election-id=ingress-controller
   25 www-data  0:00 nginx: master process /usr/local/nginx/sbin/nginx -c /etc/
   30 www-data  0:00 nginx: worker process
   31 www-data  0:01 nginx: worker process
   32 www-data  0:00 nginx: cache manager process
[root@k8s-master ~]#
[root@k8s-master ~]# kubectl exec -it ingress-nginx-controller-7c95f9b589-qfpr8 -n ingress-nginx -- cat /etc/nginx/nginx.conf | grep ingressplayroom
        ## start server httpd.ingressplayroom.com
                server_name httpd.ingressplayroom.com ;
        ## end server httpd.ingressplayroom.com
        ## start server web.myingressplayroom.com
                server_name web.myingressplayroom.com ;
        ## end server web.myingressplayroom.com
```


**References**
【1】https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress-controllers/
【2】https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress/


