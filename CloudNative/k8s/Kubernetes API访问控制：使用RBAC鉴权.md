---
tags: [k8s]
title: Kubernetes API访问控制：使用RBAC鉴权
created: '2022-12-27T10:15:06.544Z'
modified: '2022-12-28T09:22:41.773Z'
---

Kubernetes API访问控制：使用RBAC鉴权


基于角色的访问控制（Role-Based Access Control, RBAC）是一种基于组织中用户的角色来调节控制对计算机或网络资源的访问的方法。RBAC鉴权机制使用rbac.authorization.k8s.io API组来驱动鉴权决定，允许通过Kubernetes API动态配置策略。

要启用RBAC，在启动API服务器时将`--authorization-mode`参数设置为一个逗号分隔的列表并确保其中包含RBAC。
```bash
kube-apiserver --authorization-mode=Example,RBAC --<其他选项> --<其他选项>
```

# API对象
RBAC API声明了四种Kubernetes对象：Role、ClusterRole、RoleBinding和ClusterRoleBinding。

## Role和ClusterRole
RBAC的Role或ClusterRole中包含一组代表相关权限的规则。这些权限是纯粹累加的（不存在拒绝某操作的规则）。

Role总是用来在某个名字空间内设置访问权限；在创建Role时，必须指定该Role所属的**名字空间**。与之相对，ClusterRole则是一个**集群作用域**的资源。这两种资源的名字不同（Role和ClusterRole） 是因为Kubernetes对象要么是名字空间作用域的，要么是集群作用域的，不可两者兼具。

ClusterRole有若干用法，可以用它来：

- 定义对某名字空间域对象的访问权限，并将在个别名字空间内被授予访问权限；
- 为名字空间作用域的对象设置访问权限，并被授予跨所有名字空间的访问权限；
- 为集群作用域的资源定义访问权限。

### Role示例
下面是一个位于default名字空间的Role的示例，可用来授予对Pod的读访问权限：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]   # ""表示core API组
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

### ClusterRole示例
ClusterRole同样可以用于授予Role能够授予的权限。因为ClusterRole属于集群范围，所以它也可以为以下资源授予访问权限：

- 集群范围资源（比如Node）；
- 非资源端点（比如`/healthz`）；
- 跨名字空间访问的名字空间作用域的资源（如Pod）：比如，可以使用ClusterRole来允许某特定用户执行`kubectl get pods --all-namespaces`。

下面是一个ClusterRole的示例，可用来为任一特定名字空间中的Secret授予读访问权限，或者跨名字空间的访问权限（取决于该角色是如何绑定的）：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace"被忽略，因为ClusterRoles不受名字空间限制
  name: secret-reader
rules:
- apiGroups: [""]
  # 在HTTP层面，用来访问Secret资源的名称为"secrets"
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

## RoleBinding和ClusterRoleBinding
角色绑定（Role Binding）是将角色中定义的权限赋予一个或者一组用户。它包含若干主体（用户、组或服务账户）的列表和对这些主体所获得的角色的引用。RoleBinding在指定的名字空间中执行授权，而ClusterRoleBinding在集群范围执行授权。

一个RoleBinding可以引用**同一名字空间中的任何Role**。RoleBinding也可以**引用某个ClusterRole并将该ClusterRole绑定到RoleBinding所在的名字空间**。如果希望将某ClusterRole绑定到集群中所有名字空间，要使用ClusterRoleBinding。

### RoleBinding示例
下面的例子中的RoleBinding将"pod-reader" Role授予在"default"名字空间中的用户"jane"。这样，用户"jane"就具有了读取"default"名字空间中所有Pod的权限。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# 此角色绑定允许 "jane" 读取 "default" 名字空间中的Pod
# 需要在该命名空间中有一个名为 “pod-reader” 的Role
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# 可以指定不止一个“subject（主体）”
- kind: User
  name: jane # "name" 是区分大小写的
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" 指定与某Role或ClusterRole的绑定关系
  kind: Role         # 此字段必须是Role或ClusterRole
  name: pod-reader   # 此字段必须与要绑定的Role或ClusterRole的名称匹配
  apiGroup: rbac.authorization.k8s.io
```

RoleBinding也可以引用ClusterRole，以将对应ClusterRole中定义的访问权限授予RoleBinding所在名字空间的资源。这种引用使得我们可以跨整个集群定义一组通用的角色，之后在多个名字空间中复用。

例如，尽管下面的RoleBinding引用的是一个ClusterRole，"dave"（这里的主体，区分大小写）只能访问 "development" 名字空间中的Secrets对象，因为RoleBinding所在的名字空间（由其metadata决定）是 "development"。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# 此角色绑定使得用户 "dave" 能够读取 "development" 名字空间中的Secrets
# 需要一个名为 "secret-reader" 的ClusterRole
kind: RoleBinding
metadata:
  name: read-secrets
  # RoleBinding的名字空间决定了访问权限的授予范围
  # 这里隐含授权仅在 "development" 名字空间内的访问权限
  namespace: development
subjects:
- kind: User
  name: dave     # name是区分大小写的
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding示例
要跨整个集群完成访问权限的授予，可以使用一个ClusterRoleBinding。下面的ClusterRoleBinding允许 "manager" 组内的所有用户访问任何名字空间中的Secret。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# 此集群角色绑定允许 “manager” 组中的任何人访问任何名字空间中的Secret资源
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager       # name是区分大小写的
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

**创建了绑定之后，不能再修改绑定对象所引用的Role或ClusterRole**。试图改变绑定对象的roleRef将导致合法性检查错误。如果想要改变现有绑定对象中`roleRef`字段的内容，**必须删除重新创建绑定对象**。

这种限制有两个主要原因：
- 将roleRef设置为不可变更，从而可以为用户授予对现有绑定对象的update权限，这样可以让他们管理主体列表，同时不能更改这些主体被授予的角色。
- 针对不同角色的绑定是完全不一样的绑定。要求通过删除/重建绑定来更改roleRef，这样可以确保要赋予绑定的所有主体会被授予新的角色（而不是在允许或者不小心修改了roleRef的情况下导致所有现有主体未经验证即被授予新角色对应的权限）。

命令`kubectl auth reconcile`可以创建或者更新包含RBAC对象的清单文件，并且在必要的情况下删除和重新创建绑定对象，以改变所引用的角色。

## 对资源的引用
在Kubernetes API中，大多数资源都是使用对象名称的字符串表示来呈现与访问的。例如，对于Pod应使用"pods"。RBAC使用对应API端点的URL中呈现的名字来引用资源。有一些Kubernetes API涉及子资源（subresource），例如Pod的日志。对Pod日志的请求看起来像这样：

```
GET /api/v1/namespaces/{namespace}/pods/{name}/log
```

在这里，pods对应名字空间作用域的Pod资源，而log是pods的子资源。在RBAC角色表达子资源时，使用**斜线**（`/`）来分隔资源和子资源。要允许某主体读取pods同时访问这些Pod的log子资源，可以这样写：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-and-pod-logs-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]
```

对于某些请求，也可以通过`resourceNames`列表按**名称**引用资源。在指定时，可以将请求限定为资源的单个实例。下面的例子中限制可以get和update一个名为my-configmap的ConfigMap：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: configmap-updater
rules:
- apiGroups: [""]
  # 在HTTP层面，用来访问ConfigMap资源的名称为 "configmaps"
  resources: ["configmaps"]
  resourceNames: ["my-configmap"]
  verbs: ["update", "get"]
```

不能使用资源名字来限制`create`或者`deletecollection`请求。对于create请求而言，这是因为在鉴权时可能还不知道新对象的名字。如果使用resourceName来限制list或者watch请求，客户端必须在它们的list或者watch请求里包含一个与指定的resourceName匹配的`metadata.name`字段选择器。例如：
`kubectl get configmaps --field-selector=metadata.name=my-configmap`

使用通配符`*`可以批量引用所有的`resources`和`verbs`对象，无需逐一引用。对于`nonResourceURLs`，可以将通配符`*`作为后缀实现全局通配，对于`apiGroups`和`resourceNames`，**空集**表示没有任何限制。下面的示例允许对所有当前和未来资源执行所有动作（注意，这类似于内置的 cluster-admin）。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: example.com-superuser  # 此角色仅作示范，请勿使用
rules:
- apiGroups: ["example.com"]
  resources: ["*"]
  verbs: ["*"]
```

注意：在resources和verbs条目中使用通配符会为敏感资源授予过多的访问权限。例如，如果添加了新的资源类型、新的子资源或新的自定义动词，通配符条目会自动授予访问权限。用户可能不希望出现这种情况。应该执行最小特权原则，使用具体的resources和verbs确保仅赋予工作负载正常运行所需的权限。

## ClusterRole聚合
可以将若 ClusterRole聚合（Aggregate）起来，形成一个复合的ClusterRole。作为集群控制面的一部分，控制器会监视带有`aggregationRule`的ClusterRole对象集合。`aggregationRule`为控制器定义一个标签选择算符供后者匹配应该组合到当前ClusterRole的`roles`字段中的ClusterRole对象。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: []   # 控制面自动填充这里的规则
```

如果你创建一个与某个已存在的聚合ClusterRole的标签选择算符匹配的ClusterRole，这一变化会触发新的规则被添加到聚合ClusterRole的操作。下面的例子中，通过创建一个标签同样为`rbac.example.com/aggregate-to-monitoring: true`的ClusterRole，新的规则可被添加到上面定义的"monitoring" ClusterRole中。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
# 当创建"monitoring-endpoints" ClusterRole时，下面的规则会被添加到"monitoring" ClusterRole中
rules:
- apiGroups: [""]
  resources: ["services", "endpointslices", "pods"]
  verbs: ["get", "list", "watch"]
```

### Role示例
以下示例均为从Role或ClusterRole对象中截取出来，这里仅展示其rules部分。

允许读取在核心API组下的"pods"：
```yaml
rules:
- apiGroups: [""]
  # 在HTTP层面，用来访问Pod资源的名称为 "pods"
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

允许在"apps" API组中读/写Deployment（在HTTP层面，对应URL中资源部分为"deployments"）：
```yaml
rules:
- apiGroups: ["apps"]
  # 在HTTP层面，用来访问Deployment资源的名称为 "deployments"
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

允许读取核心API组中的Pod和读/写"batch" API组中的Job资源：
```yaml
rules:
- apiGroups: [""]
  # 在 HTTP 层面，用来访问 Pod 资源的名称为 "pods"
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  # 在 HTTP 层面，用来访问 Job 资源的名称为 "jobs"
  resources: ["jobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

允许读取名称为"my-config"的ConfigMap（需要通过RoleBinding绑定以限制为某名字空间中特定的ConfigMap）：
```yaml
rules:
- apiGroups: [""]
  # 在 HTTP 层面，用来访问 ConfigMap 资源的名称为 "configmaps"
  resources: ["configmaps"]
  resourceNames: ["my-config"]
  verbs: ["get"]
```

允许读取在核心组中的"nodes"资源（因为Node是集群作用域的，所以需要ClusterRole绑定到ClusterRoleBinding才生效）：
```yaml
rules:
- apiGroups: [""]
  # 在 HTTP 层面，用来访问 Node 资源的名称为 "nodes"
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

允许针对非资源端点`/healthz`和其子路径上发起GET和POST请求（必须在ClusterRole绑定ClusterRoleBinding才生效）：
```yaml
rules:
- nonResourceURLs: ["/healthz", "/healthz/*"]   # nonResourceURL 中的'*'是一个全局通配符
  verbs: ["get", "post"]
```

## 对主体的引用
RoleBinding或ClusterRoleBinding可绑定角色到某主体（Subject）上。主体可以是组、用户或者服务账户。

Kubernetes用字符串来表示用户名。用户名可以是普通的用户名，像`"alice"`；或者是邮件风格的名称，如`"bob@example.com"`，或者是以字符串形式表达的数字ID。

前缀`system:`是Kubernetes系统保留的，所以要确保所配置的用户名或者组名不能出现上述`system:`前缀。除了对前缀的限制之外，RBAC鉴权系统不对用户名格式作任何要求。

`system:serviceaccount:`（单数）是用于服务账户（ServiceAccount）**用户名**的前缀；`system:serviceaccounts:`（复数）是用于服务账户**组名**的前缀。

### RoleBinding示例
下面示例是RoleBinding中的片段，仅展示其subjects的部分。

对于名称为alice@example.com的用户：
```yaml
subjects:
- kind: User
  name: "alice@example.com"
  apiGroup: rbac.authorization.k8s.io
```

对于名称为frontend-admins的用户组：
```yaml
subjects:
- kind: Group
  name: "frontend-admins"
  apiGroup: rbac.authorization.k8s.io
```

对于kube-system名字空间中的默认服务账户：
```yaml
subjects:
- kind: ServiceAccount
  name: default
  namespace: kube-system
```

对于"qa"名称空间中的所有服务账户：
```yaml
subjects:
- kind: Group
  name: system:serviceaccounts:qa
  apiGroup: rbac.authorization.k8s.io
```

对于在任何名字空间中的服务账户：
```yaml
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```

对于所有已经过身份认证的用户：
```yaml
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
```

对于所有未通过身份认证的用户：
```yaml
subjects:
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
```

对于所有用户：
```yaml
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
```


# 默认Role和RoleBinding
API服务器会创建一组默认的ClusterRole和ClusterRoleBinding对象。其中许多是以`system:`为前缀的，用以标识对应资源是直接由集群控制面管理的。所有的默认ClusterRole和ClusterRoleBinding都有`kubernetes.io/bootstrapping=rbac-defaults`标签。

## 自动协商
在每次启动时，API服务器都会更新默认ClusterRole以添加缺失的各种权限，并更新默认的ClusterRoleBinding以增加缺失的各类主体。这种自动协商机制允许集群去修复一些不小心发生的修改，并且有助于保证角色和角色绑定在新的发行版本中有权限或主体变更时仍然保持最新。

如果基于RBAC的鉴权机制被启用，则自动协商功能默认是被启用的。如果要禁止此功能，请将默认ClusterRole以及ClusterRoleBinding的`rbac.authorization.kubernetes.io/autoupdate`注解设置成false。需要注意，缺少默认权限和角色绑定主体可能会导致集群无法正常工作。

## API发现角色
无论是经过身份验证的还是未经过身份验证的用户，默认的角色绑定都授权他们读取被认为是可安全地公开访问的API（包括CustomResourceDefinitions）。如果要禁用匿名的未经过身份验证的用户访问，请在API服务器配置中中添加`--anonymous-auth=false`的配置选项。

通过运行命令kubectl可以查看这些角色的配置信息:
```yaml
kubectl get clusterroles system:discovery -o yaml
```

如果我们编辑该ClusterRole，所作的变更会被API服务器在重启时自动覆盖，这是通过自动协商机制完成的。要避免这类覆盖操作，要么不要手动编辑这些角色，要么禁止自动协商机制。

| 默认ClusterRole | 默认ClusterRoleBinding | 描述 |
| :--: | :--: | :--: |
| system:basic-user | system:authenticated 组 | 允许用户以只读的方式去访问他们自己的基本信息。在 v1.14 版本之前，这个角色在默认情况下也绑定在 system:unauthenticated 上。 |
| system:discovery | system:authenticated 组 | 允许以只读方式访问API发现端点，这些端点用来发现和协商API级别。在 v1.14 版本之前，这个角色在默认情况下绑定在 system:unauthenticated 上。 |
| system:public-info-viewer	 | system:authenticated 和 system:unauthenticated 组	 | 允许对集群的非敏感信息进行只读访问，此角色是在 v1.14 版本中引入的。 |


## 面向用户的角色
一些默认的ClusterRole不是以前缀`system:`开头的。这些是面向用户的角色。它们包括超级用户角色（`cluster-admin`）、使用ClusterRoleBinding在集群范围内完成授权的角色（`cluster-status`）、 以及使用RoleBinding在特定名字空间中授予的角色（`admin`、`edit`、`view`）。

面向用户的ClusterRole使用ClusterRole聚合以允许管理员在这些ClusterRole上添加用于定制资源的规则。如果想要添加规则到admin、edit 或者 view，可以创建带有以下一个或多个标签的ClusterRole：

```yaml
metadata:
  labels:
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
```

| 默认ClusterRole | 默认ClusterRoleBinding | 描述 |
| :--: | :--: | :--: |
| cluster-admin | system:masters 组 | 允许超级用户在平台上的任何资源上执行所有操作。当在ClusterRoleBinding中使用时，可以授权对集群中以及所有名字空间中的全部资源进行完全控制。当在RoleBinding中使用时，可以授权控制角色绑定所在名字空间中的所有资源，包括名字空间本身。 |
| admin | 无 | 允许管理员访问权限，旨在使用RoleBinding在名字空间内执行授权。如果在RoleBinding中使用，则可授予对名字空间中的大多数资源的读/写权限，包括创建角色和角色绑定的能力。此角色不允许对资源配额或者名字空间本身进行写操作。 |
| edit | 无 | 允许对名字空间的大多数对象进行读/写操作。此角色不允许查看或者修改角色或者角色绑定。此角色可以访问Secret，以名字空间中任何ServiceAccount的身份运行Pod，所以可以用来了解名字空间内所有服务账户的API访问级别。 |
| view | 无 | 允许对名字空间的大多数对象有只读权限。它不允许查看角色或角色绑定。此角色不允许查看Secrets，因为读取Secret的内容意味着可以访问名字空间中ServiceAccount的凭据信息，进而允许利用名字空间中任何ServiceAccount的身份访问API（这是一种特权提升）。 |


## 核心组件角色

| 默认ClusterRole | 默认ClusterRoleBinding | 描述 |
| :--: | :--: | :--: |
| system:kube-scheduler | system:kube-scheduler 用户 | 允许访问scheduler组件所需要的资源。 |
| system:volume-scheduler | system:kube-scheduler 用户 | 允许访问kube-scheduler组件所需要的卷资源。 |
| system:kube-controller-manager | system:kube-controller-manager 用户 | 允许访问控制器管理器组件所需要的资源。 |
| system:node | 无 | 允许访问kubelet所需要的资源，包括对所有Secret的读操作和对所有Pod状态对象的写操作。</br>应该使用Node鉴权组件和NodeRestriction准入插件而不是 system:node 角色。同时基于kubelet上调度执行的Pod来授权kubelet对API的访问。system:node 角色的意义仅是为了与从 v1.8 之前版本升级而来的集群兼容。 |
| system:node-proxier | system:kube-proxy 用户 | 允许访问 kube-proxy 组件所需要的资源。 |


## 其他组件角色

| 默认ClusterRole | 默认ClusterRoleBinding | 描述 |
| :--: | :--: | :--: |
| system:auth-delegator	 | 无 | 允许将身份认证和鉴权检查操作外包出去。这种角色通常用在插件式 API 服务器上，以实现统一的身份认证和鉴权。 |
| system:kube-aggregator | 无 | 为 kube-aggregator 组件定义的角色。 |
| system:kube-dns | 在 kube-system 名字空间中的 kube-dns 服务账户 | 为 kube-dns 组件定义的角色 |
| system:kubelet-api-admin | 无 | 允许 kubelet API 的完全访问权限。 |
| system:node-bootstrapper | 无 | 允许访问执行 kubelet TLS 启动引导所需要的资源。 |
| system:node-problem-detector | 无 | 为 node-problem-detector 组件定义的角色。 |
| system:persistent-volume-provisioner | 无 | 允许访问大部分动态卷驱动所需要的资源。 |
| system:monitoring | system:monitoring 组 | 允许对控制平面监控端点的读取访问（例如：kube-apiserver 存活和就绪端点（`/healthz`、`/livez`、`/readyz`），各个健康检查端点（`/healthz/*`、`/livez/*`、`/readyz/*`和`/metrics`）。 |


## 内置控制器角色
Kubernetes控制器管理器运行内建于Kubernetes控制面的控制器。 当使用`--use-service-account-credentials`参数启动时，kube-controller-manager使用单独的服务账户来启动每个控制器。每个内置控制器都有相应的、前缀为`system:controller:`的角色。如果控制管理器启动时未设置`--use-service-account-credentials`，它使用自己的身份凭据来运行所有的控制器，该身份必须被授予所有相关的角色。


# 预防权限提升
RBAC API会阻止用户通过编辑角色或者角色绑定来提升权限。由于这一点是在API级别实现的，所以在RBAC鉴权组件未启用的状态下依然可以正常工作。

## 对角色创建或更新的限制
只有在符合下列条件之一的情况下，才能创建/更新角色:

- 已经拥有角色中包含的所有权限，且其作用域与正被修改的对象作用域相同。（对ClusterRole而言意味着集群范围，对Role而言意味着相同名字空间或者集群范围）。
- 被显式授权在`rbac.authorization.k8s.io` API组中的`roles`或`clusterroles`资源使用`escalate`动词。

例如，如果`user-1`没有列举集群范围所有Secret的权限，他将不能创建包含该权限的ClusterRole。若要允许用户创建/更新角色：

1. 根据需要赋予他们一个角色，允许他们根据需要创建/更新Role或者ClusterRole对象。

2. 授予他们在所创建/更新角色中包含特殊权限的权限:

- 隐式地为他们授权（如果它们试图创建或者更改Role或ClusterRole的权限，但自身没有被授予相应权限，API请求将被禁止）。
- 通过允许他们在Role或ClusterRole资源上执行`escalate`动作显式完成授权。这里的roles和clusterroles资源包含在rbac.authorization.k8s.io API组中。


## 对角色绑定创建或更新的限制
只有你已经具有了所引用的角色中包含的全部权限时，**或者**你被授权在所引用的角色上执行`bind`动词时，才可以创建或更新角色绑定。这里的权限与角色绑定的作用域相同。例如，如果用户`user-1`没有列举集群范围所有Secret的能力，则他不可以创建ClusterRoleBinding引用授予该许可权限的角色。如要允许用户创建或更新角色绑定：

1. 赋予他们一个角色，使得他们能够根据需要创建或更新RoleBinding或ClusterRoleBinding对象。

2. 授予他们绑定某特定角色所需要的许可权限：

- 隐式授权下，可以将角色中包含的许可权限授予他们；
- 显式授权下，可以授权他们在特定Role（或ClusterRole）上执行`bind`动词的权限。

例如，下面的ClusterRole和RoleBinding将允许用户`user-1`把名字空间`user-1-namespace`中的admin、edit和view角色赋予其他用户：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: role-grantor
rules:
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["rolebindings"]
  verbs: ["create"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles"]
  verbs: ["bind"]
  # 忽略 resourceNames 意味着允许绑定任何 ClusterRole
  resourceNames: ["admin","edit","view"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: role-grantor-binding
  namespace: user-1-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: role-grantor
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: user-1
```

当启动引导第一个角色和角色绑定时，需要为初始用户授予他们尚未拥有的权限。对初始角色和角色绑定进行初始化时需要使用用户组为`system:masters`的凭据，该用户组由默认绑定关联到`cluster-admin`这个超级用户角色。


# 命令行工具
## kubectl create role
创建Role对象，定义在某一名字空间中的权限。例如:

- 创建名称为 “pod-reader” 的Role对象，允许用户对Pods执行get、watch和list操作：
```bash
kubectl create role pod-reader --verb=get --verb=list --verb=watch --resource=pods
```

- 创建名称为 “pod-reader” 的Role对象并指定resourceNames：
```bash
kubectl create role pod-reader --verb=get --resource=pods --resource-name=readablepod --resource-name=anotherpod
```

- 创建名为 “foo” 的Role对象并指定apiGroups：
```bash
kubectl create role foo --verb=get,list,watch --resource=replicasets.apps
```

- 创建名为 “foo” 的Role对象并指定子资源权限:
```bash
kubectl create role foo --verb=get,list,watch --resource=pods,pods/status
```

- 创建名为 “my-component-lease-holder” 的Role对象，使其具有对特定名称的资源执行`get/update`的权限：
```bash
kubectl create role my-component-lease-holder --verb=get,list,watch,update --resource=lease --resource-name=my-component
```

## kubectl create clusterrole
创建ClusterRole对象。用法与`kubectl create role`类似。例如：

- 创建名称为 “pod-reader” 的ClusterRole对象，允许用户对Pods对象执行get、 watch和list操作：
```bash
kubectl create clusterrole pod-reader --verb=get,list,watch --resource=pods
```

- 创建名为 “foo” 的ClusterRole对象并指定nonResourceURL：
```bash
kubectl create clusterrole "foo" --verb=get --non-resource-url=/logs/*
```

- 创建名为 “monitoring” 的ClusterRole对象并指定aggregationRule：
```bash
kubectl create clusterrole monitoring --aggregation-rule="rbac.example.com/aggregate-to-monitoring=true"
```

## kubectl create rolebinding
在特定的名字空间中对Role或ClusterRole授权。例如：

- 在名字空间 “acme” 中，将名为admin的ClusterRole中的权限授予名称 “bob” 的用户:
```bash
kubectl create rolebinding bob-admin-binding --clusterrole=admin --user=bob --namespace=acme
```

- 在名字空间 “acme” 中，将名为view的ClusterRole中的权限授予名字空间 “acme” 中名为myapp的服务账户：
```bash
kubectl create rolebinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp --namespace=acme
```

## kubectl create clusterrolebinding
在整个集群（所有名字空间）中用ClusterRole授权。例如：

- 在整个集群范围，将名为cluster-admin的ClusterRole中定义的权限授予名为 “root” 用户：
```bash
kubectl create clusterrolebinding root-cluster-admin-binding --clusterrole=cluster-admin --user=root
```

- 在整个集群范围内，将名为`system:node-proxier`的ClusterRole的权限授予名为`system:kube-proxy`的用户：
```bash
kubectl create clusterrolebinding kube-proxy-binding --clusterrole=system:node-proxier --user=system:kube-proxy
```

- 在整个集群范围内，将名为view的ClusterRole中定义的权限授予 “acme” 名字空间中名为 “myapp” 的服务账户：
```bash
kubectl create clusterrolebinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp
```

## kubectl auth reconcile
使用清单文件来创建或者更新rbac.authorization.k8s.io/v1 API对象。

尚不存在的对象会被创建，如果对应的名字空间也不存在，必要的话也会被创建。已经存在的角色会被更新，使之包含输入对象中所给的权限。如果指定了`--remove-extra-permissions`，可以删除额外的权限。

已经存在的绑定也会被更新，使之包含输入对象中所给的主体。如果指定了`--remove-extra-permissions`，则可以删除多余的主体。

例如:

- **测试**应用RBAC对象的清单文件，显示将要进行的更改：
```bash
kubectl auth reconcile -f my-rbac-rules.yaml --dry-run=client
```

- 应用RBAC对象的清单文件，默认**保留**角色（roles）中的额外权限和绑定（bindings）中的其他主体：
```bash
kubectl auth reconcile -f my-rbac-rules.yaml
```

- 应用RBAC对象的清单文件，删除角色（roles）中的额外权限和绑定中的其他主体：
```bash
kubectl auth reconcile -f my-rbac-rules.yaml --remove-extra-subjects --remove-extra-permissions
```

# 服务账户权限
默认的RBAC策略为控制面组件、节点和控制器授予权限。但是不会对`kube-system`名字空间之外的服务账户授予权限（除了授予所有已认证用户的发现权限）。

这使得我们可以根据需要向特定ServiceAccount授予特定权限。细粒度的角色绑定可带来更好的安全性，但需要更多精力管理。粗粒度的授权可能导致ServiceAccount被授予不必要的API访问权限（甚至导致潜在的权限提升），但更易于管理。

按从最安全到最不安全的顺序，存在以下方法：

1. 为特定应用的服务账户授予角色（**最佳实践**）

这要求应用在其Pod规约中指定serviceAccountName，并额外创建服务账户（包括通过API、应用程序清单、`kubectl create serviceaccount`等）。

例如，在名字空间 “my-namespace” 中授予服务账户 “my-sa” 只读权限：
```bash
kubectl create rolebinding my-sa-view \
  --clusterrole=view \
  --serviceaccount=my-namespace:my-sa \
  --namespace=my-namespace
```

2. 将角色授予某名字空间中的`default`服务账户

如果某应用没有指定serviceAccountName，那么它将使用default服务账户。default服务账户所具有的权限会被授予给名字空间中所有未指定serviceAccountName的Pod。

许多插件组件在kube-system名字空间以default服务账户运行。要允许这些插件组件以超级用户权限运行，需要将集群的cluster-admin权限授予kube-system名字空间中的default服务账户。启用这一配置意味着在kube-system名字空间中包含以超级用户账号来访问集群API的Secret。

3. 将角色授予名字空间中所有服务账户

如果你想要名字空间中所有应用都具有某角色，无论它们使用的什么服务账户，可以将角色授予该名字空间的服务账户组。例如，在名字空间 “my-namespace” 中的只读权限授予该名字空间中的所有服务账户。

4. 在集群范围内为所有服务账户授予一个受限角色（不建议）

如果不想管理每一个名字空间的权限，你可以向所有的服务账户授予集群范围的角色。例如，为集群范围的所有服务账户授予跨所有名字空间的只读权限。

5. 授予超级用户访问权限给集群范围内的所有服务帐户（强烈不建议）

如果不在乎如何区分权限，你可以将超级用户访问权限授予所有服务账户。这样做会允许所有应用都对你的集群拥有完全的访问权限，并将允许所有能够读取Secret（或创建Pod）的用户对你的集群有完全的访问权限。


**References**
【1】https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authorization/
【2】https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/



