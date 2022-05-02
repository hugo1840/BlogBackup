@[TOC](Helm包管理器)

之前提到的应用部署流程主要包括：创建项目镜像、配置deployment控制器、以及通过Service和Ingress发布应用等几个重要步骤。对于较为复杂的项目，可能会需要配置很多的yaml文件，管理起来较为复杂。为了将这些yaml文件作为一个整体来管理，同时支持yaml文件的高效复用、以及应用级别的版本管理，就需要用到Kubernetes的包管理器Helm。

Helm的v3版本在v2版本的基础上对架构进行了较大的改动：一是移除了Tiller组件，直接通过kubeconfig连接apiserver；二是release名称可以在不同的命名空间中重用；三是chart支持放到docker镜像仓库。

# Helm安装部署
## 先决条件
安装和使用Helm之前必须满以下条件：
-	一个Kubernetes群集；
-	确定安装版本的安全配置。

Helm有两种主要的安装方式，即二进制版本安装和使用脚本安装（参考 https://helm.sh/zh/docs/intro/install/）。

## 二进制安装
首先，前往 `https://github.com/helm/helm/releases` 下载需要的版本。解压下载的文件`tar -zxvf helm-v3.0.0-linux-amd64.tar.gz`。然后在解压目录中找到helm程序，移动到需要的目录中`mv linux-amd64/helm /usr/local/bin/helm`。

## 脚本安装
获取脚本并在本地执行安装
```bash
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```
也可以直接执行安装：`curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash`

## Helm初始化
Helm安装后以后需要配置一个chart仓库地址。添加Helm官方仓库的命令为`helm repo add stable https://charts.helm.sh/stable`

```bash
# 查看配置的仓库列表
$ helm repo list

# 查看安装的charts列表
$ helm search repo stable
NAME                                    CHART VERSION   APP VERSION                     DESCRIPTION
stable/acs-engine-autoscaler            2.2.2           2.1.1                           DEPRECATED Scales worker nodes within agent pools
stable/aerospike                        0.2.8           v4.5.0.5                        A Helm chart for Aerospike in Kubernetes
stable/airflow                          4.1.0           1.10.4                          Airflow is a platform to programmatically autho...
stable/ambassador                       4.1.0           0.81.0                          A Helm chart for Datawire Ambassador
# ... and many more
```

# 使用Chart发布应用版本
Helm使用的包格式称为 charts。 chart就是一个描述Kubernetes相关资源的文件集合。单个chart可以用来部署一些简单的， 类似于memcache pod，或者某些复杂的HTTP服务器以及web全栈应用、数据库、缓存等等。Chart是作为特定目录布局的文件被创建的。它们可以打包到要部署的版本存档中。

## 使用官方的Chart
```bash
# 更新charts列表
$ helm repo update 
# 在Charts仓库中搜索名字包含weave的chart
$ helm search repo weave
# 展示chart的相关信息
$ helm show chart stable/weave
$ helm show all stable/weave

# 安装指定的Chart并自动生成此次发布版本的名字
$ helm install stable/weave --generate-name
# 安装指定的Chart并指定此次发布版本的名字
$ helm install myName stable/weave

# 查看部署的所有版本
$ helm list

# 查看部署的Pod和Service
$ kubectl get pods,svc
```
如果Service使用的不是NodePort，可使用 `kubectl edit svc serviceName` 命令动态修改服务的yaml配置文件。

## 创建自己的Chart
创建chart的命令为 `helm create chartName`。

```bash
$ helm create mychart
$ cd mychart && ls
Chart.yaml   charts   templates   values.yaml
```

创建的chart目录下的重要文件包括：
-	**Chart.yaml**：包含了创建Chart的相关属性信息；
-	**templates**：模板目录， 当和values 结合时，可生成有效的Kubernetes manifest文件；
-	**values.yaml**：包含了chart默认的配置值；
-	**charts**：包含了创建chart依赖的其他charts。

添加或修改templates文件夹下的yaml文件，然后使用`helm install`部署创建的chart即可。

# 变量引用、动态更新与回滚
当使用一套yaml部署多个应用时，通常可能需要修改配置文件中的资源名称、镜像、标签、副本数、端口等多个属性。如果一个一个地修改这些参数，会非常耗时耗力。Helm支持在templates中的yaml文件中引用values.yaml中定义的变量，能够有效提升应用部署的效率。引用格式为`{{ .Values.变量名 }}`。

动态更新values.yaml中的变量：`helm upgrade 发布版本名称 --set 变量=值 chart名称`

```bash
# 动态更新变量
$ helm upgrade web01 --set replicas=3 mychart/
# 查看历史部署版本
$ helm history web01
# 回滚到第一次部署的版本
$ helm rollback web01 1
```


References
[1\] https://helm.sh/zh/docs/intro/quickstart/
[2\] https://helm.sh/zh/docs/topics/charts/
