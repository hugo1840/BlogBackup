@[TOC](部署和使用Harbor镜像仓库)

# Harbor概述
Harbor是由VMware开源的容器镜像仓库。事实上，Harbor是在Docker Registry上进行了相应的企业级扩展，从而获得了更加广泛的应用。这些新的企业级特性包括：管理用户界面、基于角色的访问控制、AD/LDAP集成及审计日志等。

## 组件与功能
| 组件 | 功能 |
|--|--|
| harbor-adminserver | 配置管理中心 |
| harbor-db | mysql数据库 |
| harbor-jobservice | 负责镜像复制 |
| harbor-log | 记录操作日志 |
| harbor-ui | Web管理里页面和API |
| nginx | 前端代理，负责前端页面和镜像上传/下载/转发 |
| redis | 会话 |
| registry | 镜像存储 |

## 安装要求

**最小硬件要求**
2 CPUs，4GB内存，40GB硬盘。

**软件要求**
Docker engine 17.06.0-ce+或更高版本，Docker Compose 1.18.0或更高版本，Openssl最新版本。

**网络端口要求**
开放443、4443端口（HTTPS协议，用于对外服务）和80端口（HTTP协议，用于对内服务）。

更具体信息详见 [https://goharbor.io/docs/2.1.0/install-config/installation-prereqs/](https://goharbor.io/docs/2.1.0/install-config/installation-prereqs/)。

# Harbor部署
Harbor有在线安装和离线安装两种主要的安装方式。在线安装（Online Installer）是从Docker Hub下载Harbor相关镜像，安装包比较小；离线安装（Offline Installer）适用于没有网络的情况，其安装包包含预创建（pre-built）的镜像，因此安装包比较大。

## 安装docker compose

Docker compose用于定义和运行多个容器，是Docker官方编排（Orchestration）项目之一，负责快速的部署分布式应用。
```bash
# 下载docker compose v1.28.2
$ sudo curl -L "https://github.com/docker/compose/releases\
/download/1.28.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
# 测试安装是否成功
$ docker-compose --version
```

## 安装Harbor
在 [*https://github.com/goharbor/harbor/releases*](https://github.com/goharbor/harbor/releases) 页面下载对应的安装包。
```bash
$ mkdir /install
$ mv harbor-offline-installer-v1.5.1.tgz /install
$ cd /install && tar -zxf harbor-offline-installer-v1.5.1.tgz
$ cd harbor && vi harbor.cfg  # 修改配置文件
hostname = ${Host_IP}  # 改成主机IP
ui_url_protocol = http
harbor_admin_password = Habor12345
$ ./prepare
$ ./install.sh
# 
$ docker-compose ps  # 查看运行的容器
$ cat docker-compse.yml  # 查看容器编排配置文件
$ netstat -antp | grep :80  # 
# 访问http://主机IP查看部署的Harbor，登录：admin, Harbor12345
```

# Harbor配置与使用
## 配置insecure registries
由于Harbor默认的访问端口为443，因此需要配置非安全镜像仓库（http协议，对应80端口）。如果使用的是https协议和443端口，则不用配置非安全镜像仓库，但是需要使用证书。
```bash
# 添加Harbor镜像仓库到insecure-registries
$ vi /etc/docker/daemon.json
{
"registry-mirros" : ["http://f1361db2.m.daocloud.io"],
"insecure-registries": ["Harbor主机IP:80"]
}

$ systemctl restart docker  # 需要重启docker服务
$ docker-compose ps
$ docker-compose up -d  # 在后台启动docker compose运行的容器
$ docker info  # 查看insecure registries是否配置成功
```

## 创建Harbor用户并赋予角色
访问 **http://主机IP**，在Harbor用户管理中创建用户。然后在项目 > library > 成员中点击添加用户，输入已经创建好的用户名，并赋予合适的角色（**项目管理员**、开发人员、访客），这里选择项目管理员的角色。


## 推送和拉取镜像
向公开仓库library中推送镜像，格式为：
`docker tag IMAGE:TAG HOST_IP/library/IMAGE[:TAG]`
`docker push HOST_IP/library/IMAGE[:TAG]`


示例：
```bash
$ docker login 192.168.30.128  # 默认80端口
$ docker tag nginx:v1 192.168.30.128/library/nginx:v1  # 打tag
$ docker push 192.168.30.128/library/nginx:v1
# 访问http://主机IP，在Harbor项目中查看推送的镜像
$ docker pull 192.168.30.128/library/nginx:v1　# 下载镜像
$ docker logout 192.168.30.128  # 退出登录
```
退出登录后将无法推送镜像。如果library是公开项目，那仍然可以拉取镜像；如果library是私有项目，那么将不能拉取镜像。

此后，我们可以直接使用Harbor仓库运行容器：
`docker run -d 192.168.30.128/project/tomcat:v2` 

其中，IP为Harbor仓库所在主机，project为镜像所在的项目名称。


Source：
\[1] https://goharbor.io/docs/2.1.0/
\[2] https://docs.docker.com/compose/install/
\[3] https://goharbor.io/docs/2.1.0/install-config/installation-prereqs/

