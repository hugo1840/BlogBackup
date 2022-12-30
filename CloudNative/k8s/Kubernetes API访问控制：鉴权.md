---
tags: [k8s]
title: Kubernetes API访问控制：鉴权
created: '2022-12-26T05:27:01.756Z'
modified: '2022-12-27T10:16:22.483Z'
---

Kubernetes API访问控制：鉴权

将请求验证为来自特定的用户后，请求必须被鉴权（Authorization）。请求必须包含请求者的用户名、请求的行为以及受该操作影响的对象。如果现有策略声明用户有权完成请求的操作，那么该请求被鉴权通过。

Kubernetes支持多种鉴权模块，例如ABAC模式、RBAC模式和Webhook模式等。管理员创建集群时，他们配置应在API服务器中使用的鉴权模块。如果配置了多个鉴权模块，则**Kubernetes会检查每个模块，任意一个模块鉴权该请求，请求即可继续**；如果所有模块拒绝了该请求，请求将会被拒绝（HTTP状态码**403**）。

# 审查请求属性
Kubernetes仅审查以下API请求属性：

- **用户**：身份验证期间提供的`user`字符串。
- **组**：经过身份验证的用户所属的组名列表。
- **额外信息**：由身份验证层提供的任意字符串键到字符串值的映射。
- **API**：指示请求是否针对API资源。
- **请求路径**：各种非资源端点的路径，如`/api`或`/healthz`。
- **API请求动词**：API动词get、list、create、update、patch、watch、 proxy、redirect、delete和deletecollection用于**资源请求**。
- **HTTP请求动词**：HTTP动词get、post、put和delete用于**非资源请求**。
- **资源**：正在访问的资源的ID或名称（仅限资源请求）- 对于使用get、update、patch和delete 动词的资源请求，必须提供**资源名称**。
- **子资源**：正在访问的子资源（仅限资源请求）。
- **名字空间**：正在访问的对象的名称空间（仅适用于名字空间资源请求）。
- **API组**：正在访问的API组（仅限资源请求）。空字符串表示**核心API组**。


# 确定请求动词
## 非资源请求
对于`/api/v1/...`或`/apis/<group>/<version>/...`之外的端点（endpoints）的请求被视为非资源请求（Non-Resource Requests），并使用该请求的HTTP方法的小写形式作为其请求动词。

例如，对`/api`或`/healthz`这类端点的GET请求将使用**get**作为其动词。

## 资源请求
要确定对资源API端点的请求动词，需要查看所使用的HTTP动词以及该请求是针对单个资源还是一组资源：

| HTTP动词 | 请求动词 |
| :--: | :--: |
| POST | create |
| GET, HEAD | get（针对单个资源）、list（针对集合） |
| PUT | update |
| PATCH | patch |
| DELETE | delete（针对单个资源）、deletecollection（针对集合） |

Kubernetes有时使用专门的动词以对额外的权限进行鉴权。例如：

- RBAC：对`rbac.authorization.k8s.io` API组中`roles`和`clusterroles`资源的`bind`和`escalate`动词。
- 身份认证：对核心API组中`users`、`groups`和`serviceaccounts`以及`authentication.k8s.io` API组中的`userextras`所使用的`impersonate`动词。


# 鉴权模块
Kuberbetes支持以下几种鉴权模块：
- Node：节点鉴权，一个专用鉴权模式，根据调度到kubelet上运行的Pod为kubelet授予权限。 
- ABAC：基于属性的访问控制（ABAC）定义了一种访问控制范型，通过使用将属性组合在一起的策略， 将访问权限授予用户。策略可以使用任何类型的属性（用户属性、资源属性、对象，环境属性等）。
- RBAC：基于角色的访问控制（RBAC） 是一种基于企业内个人用户的角色来管理对计算机或网络资源的访问的方法。在此上下文中，权限是单个用户执行特定任务的能力，例如查看、创建或修改文件。被启用之后，RBAC使用`rbac.authorization.k8s.io` API组来驱动鉴权决策，从而允许管理员通过Kubernetes API动态配置权限策略。要启用RBAC，需要先使用`--authorization-mode=RBAC`启动API服务器。
- Webhook：WebHook是一个HTTP回调：发生某些事情时调用的HTTP POST；通过HTTP POST进行简单的事件通知。实现WebHook的Web应用程序会在发生某些事情时将消息发布到 URL。


## 鉴权模块设置参数
我们必须在策略中包含一个参数标志，以指明策略包含哪个鉴权模块。可以使用的参数有：

- `--authorization-mode=ABAC`：基于属性的访问控制（ABAC）模式允许使用本地文件配置策略。
- `--authorization-mode=RBAC`：基于角色的访问控制（RBAC）模式允许使用Kubernetes API创建和存储策略。
- `--authorization-mode=Webhook`：WebHook是一种 HTTP回调模式，允许使用远程REST端点管理鉴权。
- `--authorization-mode=Node`：节点鉴权是一种特殊用途的鉴权模式，专门对kubelet发出的API请求执行鉴权。
- `--authorization-mode=AlwaysDeny`：该标志阻止所有请求。仅将此标志用于测试。
- `--authorization-mode=AlwaysAllow`：此标志允许所有请求。仅在不需要API请求的鉴权时才使用此标志。

我们可以选择多个鉴权模块。模块**按顺序**检查，以便较靠前的模块具有更高的优先级来允许或拒绝请求。

## 检查API访问
kubectl提供`auth can-i`子命令，用于快速查询API鉴权。该命令使用`SelfSubjectAccessReview` API来确定当前用户是否可以执行给定操作，无论使用何种鉴权模式该命令都可以工作。

```bash
kubectl auth can-i create deployments --namespace dev
#输出为yes或no
```

管理员可以将此与用户伪装（User Impersonation）结合使用，以确定其他用户可以执行的操作。

```bash
kubectl auth can-i list secrets --namespace dev --as dave
```

检查名字空间dev里的dev-sa服务账户是否可以列举名字空间target里的Pod：

```bash
kubectl auth can-i list pods --namespace target \
	--as system:serviceaccount:dev:dev-sa
```

`SelfSubjectAccessReview`是authorization.k8s.io API组的一部分，它将API服务器鉴权公开给外部服务。该组中的其他资源包括：

- `SubjectAccessReview`：对任意用户的访问进行评估，而不仅仅是当前用户。当鉴权决策被委派给API服务器时很有用。例如，kubelet和扩展API服务器使用它来确定用户对自己的API的访问权限。
- `LocalSubjectAccessReview`：与SubjectAccessReview类似，但仅限于特定的名字空间。
- `SelfSubjectRulesReview`：返回用户可在名字空间内执行的操作集的审阅。用户可以快速汇总自己的访问权限，或者用于UI中的隐藏/显示动作。

可以通过创建普通的Kubernetes资源来查询这些API，其中返回对象的响应`"status"`字段是查询的结果。

```bash
kubectl create -f - -o yaml << EOF
apiVersion: authorization.k8s.io/v1
kind: SelfSubjectAccessReview
spec:
  resourceAttributes:
    group: apps
    name: deployments
    verb: create
    namespace: dev
EOF
```

生成的SelfSubjectAccessReview为：

```yaml
apiVersion: authorization.k8s.io/v1
kind: SelfSubjectAccessReview
metadata:
  creationTimestamp: null
spec:
  resourceAttributes:
    group: apps
    name: deployments
    namespace: dev
    verb: create
status:
  allowed: true
  denied: false
```

# 特权提升途径
能够在名字空间中创建或者编辑Pod的用户，无论是直接操作还是通过控制器（例如Operator）来操作，都可以提升他们在该名字空间内的权限。

- 挂载该名字空间内的任意Secret
  - 可以用来访问其他工作负载专用的Secret。
  - 可以用来获取权限更高的服务账号的令牌。
- 使用该名字空间内的任意服务账号
  - 可以用另一个工作负载的身份来访问Kubernetes API（伪装）。
  - 可以执行该服务账号的任意特权操作。
- 挂载该名字空间里其他工作负载专用的ConfigMap
  - 可以用来获取其他工作负载专用的信息，例如数据库主机名。
- 挂载该名字空间里其他工作负载的卷
  - 可以用来获取其他工作负载专用的信息，并且更改它。



**References**
【1】https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authorization/
【2】https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/


