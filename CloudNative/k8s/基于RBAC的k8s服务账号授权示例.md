---
tags: [k8s]
title: 基于RBAC的k8s服务账号授权示例
created: '2023-01-01T08:50:45.579Z'
modified: '2023-01-01T08:59:29.627Z'
---

基于RBAC的k8s服务账号授权示例


服务账号（ServiceAccount）是Kubernetes API所管理的用户。它们被绑定到特定的名字空间，或者由API服务器自动创建，或者通过API调用创建。服务账号与一组以Secret保存的凭据相关，这些凭据会被挂载到Pod中，从而允许集群内的进程访问Kubernetes API。


# 使用kubectl命令
下面的例子展示了如何为服务账号分配只能创建deployment、statefulset和daemonset的权限。
```bash
# 创建命名空间
kubectl create ns app-team1

# 创建服务账号
kubectl create sa cicd-token -n app-team1

# 创建集群角色
kubectl create clusterrole deployer-clus --verb=create --resource=deployments,daemonsets,statefulsets

# 服务账号绑定角色
kubectl create rolebinding rb-cicd-token --serviceaccount=app-team1:cicd-token --clusterrole=deployer-clus -n app-team1

# 测试服务账号权限
kubectl create deploy webserver --image=nginx --as=system:serviceaccount:app-team1:cicd-token -n app-team1
kubectl get deploy,pods -n app-team1
```

# 使用yaml文件
上面的流程写成对应的yaml文件为：
```yaml
apiVersion:v1
kind: ServiceAccount
metadata:
  name: cicd-token
  namespace: app-team1

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployer-clus
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets", "statefulsets"]
  verbs: ["create"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rb-cicd-token
  namespace: app-team1
subjects:
- kind: ServiceAccount
  name: cicd-token
  namespace: app-team1
roleRef:
  kind: ClusterRole
  name: deployer-clus
  apiGroup: rbac.authorization.k8s.io
```

# 修改服务账号权限
绑定了deployer-clus角色后，服务账号`cicd-token`只有create权限，但是无法查看资源：
```bash
[root@k8s-master ~]# kubectl get deploy,pods --as=system:serviceaccount:app-team1:cicd-token -n app-team1
Error from server (Forbidden): deployments.apps is forbidden: User "system:serviceaccount:app-team1:cicd-token" cannot list resource "deployments" in API group "apps" in the namespace "app-team1"
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:app-team1:cicd-token" cannot list resource "pods" in API group "" in the namespace "app-team1"
```

使用`kubectl edit`命令修改deployer-clus角色来为服务账号添加读权限：
```bash
[root@k8s-master ~]# kubectl edit clusterrole/deployer-clus
clusterrole.rbac.authorization.k8s.io/deployer-clus edited
```

在打开的yaml文件中添加对deployments和pods的读权限：
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  creationTimestamp: "2023-01-01T07:59:26Z"
  name: deployer-clus
  resourceVersion: "547284"
  uid: 6dea3a32-3bc1-4f85-b3c8-1261ba31d632
rules:
- apiGroups:
  - apps
  resources:
  - deployments
  - daemonsets
  - statefulsets
  verbs:
  - create
- apiGroups:
  - apps
  resources:
  - deployments
  - daemonsets
  - statefulsets
  verbs:
  - get
  - list
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
```

测试服务账号的权限修改是否生效：
```bash
[root@k8s-master ~]# kubectl get deploy,pods --as=system:serviceaccount:app-team1:cicd-token -n app-team1
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/webserver   0/1     1            0           26m

NAME                             READY   STATUS             RESTARTS   AGE
pod/webserver-7c4f9bf7bf-c4qjp   0/1     ImagePullBackOff   0          26m
```

**References**
【1】https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authentication/
【2】https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/
【3】https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authorization/#determine-the-request-verb


