---
tags: [k8s]
title: 基于RBAC的k8s用户授权示例
created: '2022-12-31T12:05:09.336Z'
modified: '2023-01-01T08:52:37.509Z'
---

基于RBAC的k8s用户授权示例


# Role与RoleBinding创建示例
## Role示例
Role是一组权限的集合。
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]   # 空值表示核心API组，包括namespace、pod、service、pv、pvc等
  resources: ["pods"]  # 资源名称（复数），例如pods、deployments、services
  verbs: ["get", "watch", "list"]   # 对资源的操作动词
```

查看API resource是否属于核心API组:
```bash
[root@k8s-master rbac]# kubectl api-resources
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
configmaps                        cm           v1                                     true         ConfigMap
endpoints                         ep           v1                                     true         Endpoints
namespaces                        ns           v1                                     false        Namespace
nodes                             no           v1                                     false        Node
persistentvolumeclaims            pvc          v1                                     true         PersistentVolumeClaim
persistentvolumes                 pv           v1                                     false        PersistentVolume
pods                              po           v1                                     true         Pod
secrets                                        v1                                     true         Secret
serviceaccounts                   sa           v1                                     true         ServiceAccount
services                          svc          v1                                     true         Service
daemonsets                        ds           apps/v1                                true         DaemonSet
deployments                       deploy       apps/v1                                true         Deployment
replicasets                       rs           apps/v1                                true         ReplicaSet
statefulsets                      sts          apps/v1                                true         StatefulSet
```
其中，**APIVERSION**列的值为`v1`的资源属于核心API组。例如上面的configmap、namespaces、pods、services就属于核心API组；反之，deployments、replicasets、statefulsets就不属于核心API组。


## RoleBinding示例
RoleBinding可以将Role中定义的权限赋予一个用户或者一组用户。
```yaml
apiVersion: rbac.authorization.k8s.io/v1
# 此角色绑定允许 "jane" 读取 "default" 名字空间中的Pod
# 需要在该命名空间中有一个名为 “pod-reader” 的Role
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:   # 可以指定不止一个subject（主体）
- kind: User   # 主体的类型
  name: jane   # 主体的名称，区分大小写
  apiGroup: rbac.authorization.k8s.io
roleRef:   # roleRef指定与某Role或ClusterRole的绑定关系
  kind: Role         # 此字段必须是Role或ClusterRole
  name: pod-reader   # 此字段必须与要绑定的Role或ClusterRole的名称匹配
  apiGroup: rbac.authorization.k8s.io
```


# 基于RBAC的用户授权示例
## 用k8s CA签发客户端证书

创建Kubernetes集群时自动生成的根证书位于`/etc/kubernetes/pki/ca.crt`。

准备生成CA证书JSON配置文件和证书签名请求（**CSR**）的脚本。其中，`ca-config.json`用于生成证书颁发机构（Certificate Authority, **CA**）文件，可以指定证书过期时间；`mckinsey.csr.json`则用于生成用户mckinsey的CA证书签名请求，其中**CN**字段对应用户名（CommonName）。

```bash
[root@k8s-master rbac]# cat cert.sh
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF

cat > mckinsey-csr.json <<EOF
{
  "CN": "mckinsey",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "USA",
      "ST": "California",
      "L": "Sacramento",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

# 指定根证书、JSON配置文件，生成以用户名mckinsey为前缀的密钥和证书文件
cfssl gencert -ca=/etc/kubernetes/pki/ca.crt -ca-key=/etc/kubernetes/pki/ca.key -config=ca-config.json -profile=kubernetes mckinsey-csr.json | cfssljson -bare mckinsey
```

运行脚本签发客户端证书：
```bash
[root@k8s-master rbac]# ./cert.sh
2022/12/31 05:00:06 [INFO] generate received request
2022/12/31 05:00:06 [INFO] received CSR
2022/12/31 05:00:06 [INFO] generating key: rsa-2048
2022/12/31 05:00:06 [INFO] encoded CSR
2022/12/31 05:00:06 [INFO] signed certificate with serial number 695268160968997598742877842743429519856567946785
2022/12/31 05:00:06 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
[root@k8s-master rbac]#
[root@k8s-master rbac]# ls
ca-config.json  cert.sh  mckinsey.csr  mckinsey-csr.json  mckinsey-key.pem  mckinsey.pem 
```
其中，`mckinsey-key.pem`为私钥，`mckinsey.pem`为数字证书。


##  生成kubeconfig授权证书

查看k8s管理员的kubeconfig文件：
```bash
[root@k8s-master rbac]# cat /root/.kube/config | grep server
    server: https://192.168.124.139:6443
```

准备为用户mckinsey生成kubeconfig授权证书的脚本：
```bash
[root@k8s-master rbac]# cat kubeconfig.sh

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://192.168.124.139:6443 \
  --kubeconfig=mckinsey.kubeconfig

# 设置客户端认证
kubectl config set-credentials mckinsey \
  --client-key=mckinsey-key.pem \
  --client-certificate=mckinsey.pem \
  --embed-certs=true \
  --kubeconfig=mckinsey.kubeconfig

# 设置默认上下文
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=mckinsey \
  --kubeconfig=mckinsey.kubeconfig

# 设置当前使用配置
kubectl config use-context kubernetes --kubeconfig=mckinsey.kubeconfig
```

运行脚本生成授权证书`mckinsey.kubeconfig`：
```bash
[root@k8s-master rbac]# ./kubeconfig.sh
Cluster "kubernetes" set.
User "mckinsey" set.
Context "kubernetes" created.
Switched to context "kubernetes".
[root@k8s-master rbac]# ls
ca-config.json  cert.sh  kubeconfig.sh  mckinsey.csr  mckinsey-csr.json  mckinsey-key.pem  mckinsey.kubeconfig  mckinsey.pem  
```


## 创建RBAC权限策略：Pod读权限

在创建RBAC权限之前，前面的用户mckinsey是没有权限访问集群资源的：
```bash
[root@k8s-master rbac]# kubectl get pods --kubeconfig=./mckinsey.kubeconfig
Error from server (Forbidden): pods is forbidden: User "mckinsey" cannot list resource "pods" in API group "" in the namespace "default"
```

为mckinsey用户创建并绑定RBAC权限策略，这里仅授予对Pod的读权限。
```bash
[root@k8s-master rbac]# cat rbac.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

---

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: mckinsey
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

运行脚本授予RBAC权限：
```bash
[root@k8s-master rbac]# kubectl apply -f rbac.yaml
role.rbac.authorization.k8s.io/pod-reader created
rolebinding.rbac.authorization.k8s.io/read-pods created
```

##  指定kubeconfig文件测试权限

```bash
[root@k8s-master rbac]# kubectl get pods --kubeconfig=./mckinsey.kubeconfig
NAME                                      READY   STATUS             RESTARTS       AGE
bs2                                       1/1     Running            11 (12d ago)   147d
nfs-client-provisioner-5d5775b9bb-5657c   1/1     Running            9 (12d ago)    140d
web-0                                     0/1     ImagePullBackOff   4 (56d ago)    59d
web-1                                     0/1     ImagePullBackOff   4 (56d ago)    59d
web-2                                     0/1     ImagePullBackOff   4 (56d ago)    59d
[root@k8s-master rbac]#
[root@k8s-master rbac]# kubectl get deploy --kubeconfig=./mckinsey.kubeconfig
Error from server (Forbidden): deployments.apps is forbidden: User "mckinsey" cannot list resource "deployments" in API group "apps" in the namespace "default"
```
可以看出，mckinsey用户对Pod资源有读权限，对其他资源（例如Deployment）没有读权限。

## 添加RBAC权限策略：Service读权限
Service与Pod同属于核心API组。修改rbac.yaml为mckinsey用户添加对Service资源的读权限：

```yaml 
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "services"]   # 添加"services"资源
  verbs: ["get", "watch", "list"]

---

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: mckinsey
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

更新RBAC策略并验证mckinsey用户权限：
```bash
[root@k8s-master rbac]# kubectl apply -f rbac.yaml
role.rbac.authorization.k8s.io/pod-reader configured
rolebinding.rbac.authorization.k8s.io/read-pods unchanged
[root@k8s-master rbac]#
[root@k8s-master rbac]# kubectl get svc --kubeconfig=./mckinsey.kubeconfig
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   161d
nginx        ClusterIP   None         <none>        80/TCP    59d
```

## 添加RBAC权限策略：Deployment读权限

查看Deploments资源所属的API组：
```bash
[root@k8s-master rbac]# kubectl api-resources | grep deploy
deployments                       deploy       apps/v1                                true         Deployment
```
可以看到，deployments资源属于apps API组。

修改rbac.yaml为mckinsey用户添加对Service资源的读权限：

```yaml 
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: ["", "apps"]   # 添加"apps" API组
  resources: ["pods", "services", "deployments"]   # 添加"deployments"资源
  verbs: ["get", "watch", "list"]

---

kind: RoleBinding
# 此处内容不变，已省略
```

更新RBAC策略并验证mckinsey用户权限：
```bash
[root@k8s-master rbac]# kubectl apply -f rbac.yaml
role.rbac.authorization.k8s.io/pod-reader configured
rolebinding.rbac.authorization.k8s.io/read-pods unchanged
[root@k8s-master rbac]#
[root@k8s-master rbac]# kubectl get deploy --kubeconfig=./mckinsey.kubeconfig
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
nfs-client-provisioner   1/1     1            1           140d
```

## 添加RBAC权限策略：Pod写权限

目前mckinsey用户对Pod只有读权限。如果尝试用该用户删除Pod，会收到报错：
```bash
[root@k8s-master rbac]# kubectl delete pod web-1 --kubeconfig=./mckinsey.kubeconfig
Error from server (Forbidden): pods "web-1" is forbidden: User "mckinsey" cannot delete resource "pods" in API group "" in the namespace "default"
```

为mckinsey用户添加删除Pod的权限：
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "services", "deployments"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]   # 应当在rules下另起一个子项
  resources: ["pods"]
  verbs: ["delete"]   # 删除单个资源对应的动词为"delete"

---

kind: RoleBinding
# 此处内容不变，已省略
```

更新RBAC策略并验证mckinsey用户权限：
```bash
[root@k8s-master rbac]# kubectl apply -f rbac.yaml
role.rbac.authorization.k8s.io/pod-reader configured
rolebinding.rbac.authorization.k8s.io/read-pods unchanged
[root@k8s-master rbac]#
[root@k8s-master rbac]# kubectl delete pod web-1 --kubeconfig=./mckinsey.kubeconfig
pod "web-1" deleted
```

我们可以把`mckinsey.kubeconfig`拷贝到任意一台服务器上，只要该服务器能够连接到`mckinsey.kubeconfig`中的server，就可以通过mckinsey用户的权限来访问Kubernetes集群资源。

如果想在执行kubectl命令时省略`--kubeconfig=./mckinsey.kubeconfig`，可以将`mckinsey.kubeconfig`文件重命名为系统用户家目录下的`.kube/config`。



**References**
【1】https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authentication/
【2】https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/certificates/
【3】https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/certificate-signing-requests/#add-to-kubeconfig
【4】https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/
【5】https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authorization/#determine-the-request-verb




