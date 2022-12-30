---
tags: [k8s]
title: Kubernetes API访问控制：用户认证
created: '2022-12-18T06:13:52.991Z'
modified: '2022-12-26T09:04:37.275Z'
---

Kubernetes API访问控制：用户认证

k8s安全控制框架主要由下面三个阶段控制。每一个阶段都支持插件方式，通过API Server配置来启用插件。

1. **Authentication**（认证）
2. **Authorization**（鉴权）
3. **Admission Control**（准入控制）

k8s API Server提供三种客户端认证方式：
- HTTPS证书认证：基于CA证书签名的数字证书认证（kubeconfig）
- HTTP Token认证：通过一个token来识别用户（serviceaccount）
- HTTP Base认证：用户名+密码的方式认证（1.19版本弃用）

建立TLS后，HTTP请求将进入认证步骤。集群创建脚本或者集群管理员配置API服务器，使之运行一个或多个身份认证组件。认证步骤的输入整个HTTP请求；但是，通常组件只检查**头部或/和客户端证书**。认证模块包含客户端证书、密码、普通令牌、引导令牌和JSON Web令牌（JWT，用于服务账户）。

可以指定多个认证模块，在这种情况下，服务器依次尝试每个验证模块，**直到其中一个成功**。如果请求认证不通过，服务器将以HTTP状态码**401**拒绝该请求。 反之，该用户被认证为特定的username，并且该用户名可用于后续步骤以在其决策中使用。部分验证器还提供用户的组成员身份，其他则不提供。

# Kubernetes中的用户
所有Kubernetes集群都有两类用户：**由Kubernetes管理的服务账号**和**普通用户**。Kubernetes假定普通用户是由一个与集群无关的服务通过以下方式之一进行管理的：

- 负责分发私钥的管理员；
- 类似Keystone或者Google Accounts这类用户数据库；
- 包含用户名和密码列表的文件。

鉴于此，Kubernetes并不包含用来代表普通用户账号的对象。普通用户的信息无法通过API调用添加到集群中。尽管无法通过API调用来添加普通用户，Kubernetes仍然认为能够提供由集群的证书机构签名的合法证书的用户是通过身份认证的用户。Kubernetes使用证书中的`subject`的Common Name字段（例如`"/CN=bob"`）来确定用户名。

与普通用户不同，服务账号是Kubernetes API所管理的用户。它们被绑定到特定的名字空间，或者由API服务器自动创建，或者通过API调用创建。服务账号与一组以Secret保存的凭据相关，这些凭据会被挂载到Pod中，从而允许集群内的进程访问Kubernetes API。

API请求或者与某普通用户相关联，或者与某服务账号相关联，亦或者被视作**匿名请求**。这意味着**集群内外的每个进程**在向API服务器发起请求时都必须通过身份认证，否则会被视作匿名用户。这里的进程可以是在某工作站上输入kubectl命令的操作人员，也可以是节点上的kubelet组件，还可以是控制面的成员。

# 身份认证策略
Kubernetes通过身份认证插件利用客户端证书、持有者令牌（Bearer Token）或身份认证代理（Proxy）来认证API请求的身份。HTTP请求发给API服务器时，插件会将以下属性关联到请求本身：

- **用户名**：用来辩识最终用户的字符串。常见的值可以是`kube-admin`或`john@example.com`。
- **用户ID**：用来辩识最终用户的字符串，旨在比用户名有更好的一致性和唯一性。
- **用户组**：取值为一组字符串，其中各个字符串用来标明用户是某个命名的用户逻辑集合的成员。常见的值可能是`system:masters`或者`devops-team`等。
- **附加字段**：一组额外的键-值映射，键是字符串，值是一组字符串；用来保存一些鉴权组件可能觉得有用的额外信息。

所有（属性）值对于身份认证系统而言都是不透明的，只有被鉴权组件解释过之后才有意义。

我们可以同时启用多种身份认证方法，并且通常会至少使用两种方法：
- 针对服务账号使用服务账号令牌；
- 至少另外一种方法对用户的身份进行认证。

当集群中启用了多个身份认证模块时，第一个成功地对请求完成身份认证的模块会直接做出评估决定。API服务器并不保证身份认证模块的运行顺序。对于所有通过身份认证的用户，`system:authenticated`组都会被添加到其组列表中。

Kubernetes支持的身份认证策略包括：X509客户证书、静态令牌文件、启动引导令牌（Bootstrap Tokens）、服务账号令牌（Service Account Tokens）、OpenID Connect（OIDC）令牌、Webhook令牌、身份认证代理等。

## X509客户证书
通过给API服务器传递`--client-ca-file=...`选项，就可以启动客户端证书身份认证。所引用的文件必须包含一个或者多个证书机构，用来验证向API服务器提供的客户端证书。如果提供了客户端证书并且证书被验证通过，则subject中的`Common Name`就被作为请求的用户名。自Kubernetes 1.4开始，客户端证书还可以通过证书的`organization`字段标明用户的组成员信息。

在使用客户端证书认证的场景下，我们可以通过easyrsa、openssl或cfssl等工具以手工方式生成证书。

使用cfssl手动生成证书的步骤如下：

1. 下载对应版本和平台的cfssl工具。

```bash
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl_1.5.0_linux_amd64 -o cfssl
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssljson_1.5.0_linux_amd64 -o cfssljson
curl -L https://github.com/cloudflare/cfssl/releases/download/v1.5.0/cfssl-certinfo_1.5.0_linux_amd64 -o cfssl-certinfo

chmod +x cfssl
chmod +x cfssljson
chmod +x cfssl-certinfo
```

2. 创建一个目录用于保存所生成的构件和初始化cfssl。

```bash
mkdir cert
cd cert
../cfssl print-defaults config > config.json
../cfssl print-defaults csr > csr.json
```

3. 创建一个JSON配置文件来生成CA文件，例如`ca-config.json`。

```json
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "8760h"
      }
    }
  }
}
```

4. 创建一个JSON配置文件，用于CA证书签名请求（CSR），例如`ca-csr.json`。用需要的值替换掉尖括号中的值。

```json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "<country>",
    "ST": "<state>",
    "L": "<city>",
    "O": "<organization>",
    "OU": "<organization unit>"
  }]
}
```

5. 生成CA秘钥文件`ca-key.pem`和证书文件`ca.pem`。

```bash
../cfssl gencert -initca ca-csr.json | ../cfssljson -bare ca
```

6. 创建一个JSON配置文件，用来为API服务器生成秘钥和证书，例如`server-csr.json`。用需要的值替换掉尖括号中的值。`MASTER_CLUSTER_IP`是为API服务器指定的服务集群IP。以下示例假定默认DNS域名为`cluster.local`。

```json
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "<MASTER_IP>",
    "<MASTER_CLUSTER_IP>",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [{
    "C": "<country>",
    "ST": "<state>",
    "L": "<city>",
    "O": "<organization>",
    "OU": "<organization unit>"
  }]
}
```

7. 为API服务器生成秘钥和证书，默认会分别存储为`server-key.pem`和`server.pem`两个文件。

```bash
../cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
     --config=ca-config.json -profile=kubernetes \
     server-csr.json | ../cfssljson -bare server
```

客户端节点可能不认可自签名CA证书的有效性。对于非生产环境，或者运行在防火墙后的环境，我们可以分发自签名的CA证书到所有客户节点，并刷新本地列表以使证书生效。

在每一个客户节点，执行以下操作：
```bash
sudo cp ca.crt /usr/local/share/ca-certificates/kubernetes.crt
sudo update-ca-certificates
```

## 静态令牌文件
当API服务器的命令行设置了`--token-auth-file=...`选项时，会从文件中读取持有者令牌。目前，令牌会长期有效，并且在不重启API服务器的情况下无法更改令牌列表。

令牌文件是一个CSV文件，包含至少3个列：令牌、用户名和用户的UID。其余列被视为可选的组名。如果要设置的组名不止一个，则对应的列必须用双引号括起来。

当使用持有者令牌来对某HTTP客户端执行身份认证时，API服务器希望看到一个名为Authorization的HTTP头，其值格式为`Bearer <token>`。持有者令牌必须是一个可以放入HTTP头部值字段的字符序列。
```bash
# Authorization: Bearer <token>
Authorization: Bearer 31ada4fd-adec-460c-809a-9e56ceb75269
```

## 启动引导令牌
为了支持平滑地启动引导新的集群，Kubernetes引入了一种动态管理的持有者令牌类型，称作启动引导令牌（Bootstrap Token）。这些令牌以Secret的形式保存在`kube-system`名字空间中，可以被动态管理和创建。控制器管理器包含的TokenCleaner控制器能够在启动引导令牌过期时将其删除。

这些令牌的格式为`[a-z0-9]{6}.[a-z0-9]{16}`。第一个部分是令牌的ID；第二个部分是令牌的Secret。可以用如下所示的方式来在HTTP头部设置令牌：

```
Authorization: Bearer 781292.db7bc3a58fc5f07e
```

我们必须在API服务器上设置`--enable-bootstrap-token-auth`标志来启用基于启动引导令牌的身份认证组件。必须通过控制器管理器的`--controllers`标志来启用TokenCleaner控制器；这可以通过类似`--controllers=*,tokencleaner`这种设置来做到。如果使用kubeadm来启动引导新的集群，该工具会帮我们完成这些设置。

身份认证组件的认证结果为`system:bootstrap:<Token ID>`，该用户属于`system:bootstrappers`用户组。这里的用户名和组设置都是有意设计成这样，其目的是阻止用户在启动引导集群之后继续使用这些令牌。这里的用户名和组名可以用来（并且已经被kubeadm用来）构造合适的鉴权策略，以完成启动引导新集群的工作。

## 服务账号令牌
服务账号（Service Account）是一种自动被启用的用户认证机制，使用经过签名的持有者令牌来验证请求。该插件可接受两个可选参数：

- `--service-account-key-file`：文件包含PEM编码的x509 RSA或ECDSA私钥或公钥，用于验证ServiceAccount令牌。这样指定的文件可以包含多个密钥，并且可以使用不同的文件多次指定此参数。若未指定，则使用`--tls-private-key-file`参数。
- `--service-account-lookup`：如果启用，则从API删除的令牌会被回收。

服务账号通常由API服务器自动创建并通过**ServiceAccount准入控制器**关联到集群中运行的Pod上。持有者令牌会挂载到Pod中可预知的位置，允许集群内进程与API服务器通信。服务账号也可以使用Pod规约的`serviceAccountName`字段显式地关联到Pod上。

```yaml
apiVersion: apps/v1  # 该apiVersion从Kubernetes 1.9开始可用
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
spec:
  replicas: 3
  template:
    metadata:
    # ...
    spec:
      serviceAccountName: bob-the-bot
      containers:
      - name: nginx
        image: nginx:1.14.2
```

在集群外部使用服务账号持有者令牌也是完全合法的，且可用来为长时间运行的、需要与Kubernetes API服务器通信的任务创建标识。要手动创建服务账号，可以使用`kubectl create serviceaccount <名称>`命令。此命令会在当前的名字空间中生成一个服务账号。

```bash
# 创建服务账号
kubectl create serviceaccount jenkins
# 创建相关联的令牌
kubectl create token jenkins
```

所创建的令牌是一个已签名的JWT令牌。已签名的JWT可以用作持有者令牌，并将被认证为所给的服务账号。服务账号被身份认证后，所确定的用户名为`system:serviceaccount:<名字空间>:<服务账号>`， 并被分配到用户组`system:serviceaccounts`和`system:serviceaccounts:<名字空间>`。

由于服务账号令牌也可以保存在Secret API对象中，任何能够写入这些Secret的用户都可以请求一个令牌，且任何能够读取这些Secret的用户都可以被认证为对应的服务账号。因此，在为用户授予访问服务账号的权限以及对Secret的读取或写入权能时，要格外小心。


# 匿名请求
启用匿名请求支持之后，如果请求没有被已配置的其他身份认证方法拒绝， 则被视作匿名请求（Anonymous Requests）。这类请求获得用户名`system:anonymous`和对应的用户组`system:unauthenticated`。

例如，在一个配置了令牌身份认证且启用了匿名访问的服务器上，如果请求提供了非法的持有者令牌，则会返回**401 Unauthorized**错误。如果请求没有提供持有者令牌，则被视为匿名请求。

在**1.5.1-1.5.x**版本中，匿名访问默认情况下是被**禁用**的，可以通过为API服务器设定`--anonymous-auth=true`来启用。

在**1.6及之后**版本中，如果所使用的鉴权模式不是`AlwaysAllow`，则匿名访问默认是被**启用**的。从1.6版本开始，ABAC和RBAC鉴权模块要求对`system:anonymous`用户或者`system:unauthenticated`用户组执行显式的权限判定，所以之前的为用户`*`或用户组`*`赋予访问权限的策略规则都不再包含匿名用户。


# 用户伪装
一个用户可以通过伪装（Impersonation）头部字段来以另一个用户的身份执行操作。使用这一能力，可以手动重载请求被身份认证所识别出来的用户信息。例如，管理员可以使用这一功能特性来临时伪装成另一个用户，查看请求是否被拒绝，从而调试鉴权策略中的问题，

带伪装的请求首先会被身份认证识别为发出请求的用户，之后会切换到使用被伪装的用户的用户信息。

- 用户发起API调用时，同时提供自身的凭据和伪装头部字段信息；
- API服务器对用户执行身份认证；
- API服务器确认通过认证的用户具有伪装特权；
- 请求用户的信息被替换成伪装字段的值；
- 评估请求，鉴权组件针对所伪装的用户信息执行操作。

以下HTTP头部字段可用来执行伪装请求：

- `Impersonate-User`：要伪装成的用户名。
- `Impersonate-Group`：要伪装成的用户组名。可以多次指定以设置多个用户组。可选字段；要求"Impersonate-User"必须被设置。
- `Impersonate-Extra-<附加名称>`：一个动态的头部字段，用来设置与用户相关的附加字段。此字段可选；要求"Impersonate-User"被设置。为了能够以一致的形式保留，`<附加名称>`部分必须是小写字符，如果有任何字符不是合法的HTTP头部标签字符，则必须是utf8字符，且转换为百分号编码。
- `Impersonate-Uid`：一个唯一标识符，用来表示所伪装的用户。此头部可选。如果设置，则要求"Impersonate-User"也存在。Kubernetes对此字符串没有格式要求。该字段仅在1.22.0及更高版本中可用。

若要伪装成某个用户、某个组、用户标识符（UID）或者设置附加字段，执行伪装操作的用户必须具有对所伪装的类别（"user"、"group"、"uid"等）执行impersonate动词操作的能力。对于启用了RBAC鉴权插件的集群，下面的ClusterRole封装了设置用户和组伪装字段所需的规则：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: impersonator
rules:
- apiGroups: [""]
  resources: ["users", "groups", "serviceaccounts"]
  verbs: ["impersonate"]
```

为了执行伪装，附加字段和所伪装的UID都位于"authorization.k8s.io" `apiGroup`中。附加字段会被作为`userextras`资源的子资源来执行权限评估。如果要允许用户为附加字段"scopes"和UID设置伪装头部，该用户需要被授予以下角色：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: scopes-and-uid-impersonator
rules:
# 可以设置 "Impersonate-Extra-scopes" 和 "Impersonate-Uid" 头部
- apiGroups: ["authentication.k8s.io"]
  resources: ["userextras/scopes", "uids"]
  verbs: ["impersonate"]
```

也可以通过约束资源可能对应的`resourceNames`限制伪装头部的取值：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: limited-impersonator
rules:
  # 可以伪装成用户 "jane.doe@example.com"
  - apiGroups: [""]
    resources: ["users"]
    verbs: ["impersonate"]
    resourceNames: ["jane.doe@example.com"]
  
  # 可以伪装成用户组 "developers" 和 "admins"
  - apiGroups: [""]
    resources: ["groups"]
    verbs: ["impersonate"]
    resourceNames: ["developers","admins"]
  
  # 可以将附加字段 "scopes" 伪装成 "view" 和 "development"
  - apiGroups: ["authentication.k8s.io"]
    resources: ["userextras/scopes"]
    verbs: ["impersonate"]
    resourceNames: ["view", "development"]
  
  # 可以伪装 UID "06f6ce97-e2c5-4ab8-7ba5-7654dd08d52b"
  - apiGroups: ["authentication.k8s.io"]
    resources: ["uids"]
    verbs: ["impersonate"]
    resourceNames: ["06f6ce97-e2c5-4ab8-7ba5-7654dd08d52b"]
```

基于伪装成一个用户或用户组的能力，你可以执行任何操作，好像你就是那个用户或用户组一样。出于这一原因，**伪装操作是不受名字空间约束的**。如果你希望允许使用Kubernetes RBAC来执行身份伪装，就需要使用ClusterRole和ClusterRoleBinding，而不是Role或RoleBinding。


**References**
【1】https://kubernetes.io/zh-cn/docs/concepts/security/controlling-access/
【2】https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authentication/
【3】https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/certificates/
【4】https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/bootstrap-tokens/
【5】https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/bootstrap-tokens/
【6】https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/certificate-signing-requests/





