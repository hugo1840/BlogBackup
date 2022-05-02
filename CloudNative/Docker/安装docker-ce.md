@[TOC](Docker-ce安装（CentOS7）)

# 准备工作
对于CentOS操作系统，要求其版本必须为CentOS7或者CentOS8。安装前需要关闭防火墙和selinux。

```bash
$ systemctl stop firewalld && systemctl disable fiewalld
$ sed -i 's/enforcing/disabled/' /etc/selinux/config
```

# 卸载旧版本
在安装社区版（docker-ce）或者企业版（docker-ee）容器之前，如果有安装旧版本的docker或者docker-engine，需要先将其卸载。

```bash
$ sudo yum remove docker \
docker-client docker-client-latest \
docker-common docker-latest \
docker-latest-logrotate docker-logrotate \
docker-engine
```

安装docker-ce有三种方法：一是通过创建docker repository仓库来安装（最常见也最推荐的方法，安装和更新比较简单），二是通过RPM手动安装和更新（适用于没用网络连接的情况），三是使用docker提供的自动化脚本安装（适用于测试和开发环境，生产环境中不推荐）。

# 通过docker repository安装
## 创建repository
```bash
$ sudo yum install -y yum-utils  # 安装依赖包
$ sudo yum-config-manager --add-repo \
https://download.docker.com/linux/centos/docker-ce.repo  # 添加docker软件包源
```

配置软件包源时如果网络不通，可以替换为阿里云的源或者使用离线安装方法。配置完成后，系统中会生成`/etc/yum.repos.d/docker-ce.repo`文件  。

## 安装docker engine
**安装docker engine最新版本**
```bash
$ sudo yum install docker-ce docker-ce-cli containerd.io
```

如果提示接受GPG key，确认与`060A 61C5 1B55 8A7F 742B 77AA C52F EB6B 621E 9F35`匹配，然后点击接受确认。

**安装docker engine指定版本**
查看仓库中可用的docker-ce版本：

```bash
$ yum list docker-ce --showduplicates | sort -r
docker-ce.x86_64	3:18.09.1-3.el7	docker-ce-stable
docker-ce.x86_64	3:18.09.0-3.el7	docker-ce-stable
docker-ce.x86_64	18.06.1.ce-3.el7	docker-ce-stable
```

安装选定的版本：
```bash
$ sudo yum install docker-ce-18.09.1 \
docker-ce-cli-18.09.1 containerd.io
```

**启动docker**
```bash
$ sudo systemctl start docker
$ sudo systemctl enable docker  # 设定开机启动
```

**测试是否安装成功**
```bash
$ sudo docker run hello-world
$ docker info  # 查看安装的docker信息
$ docker version  # 查看docker版本
```

# 创建容器测试

```bash
$ docker run -it nginx  # 创建nginx容器并在前台以交互式运行
# 在xshell中打开一个新的终端
$ docker ps  # 查看运行的容器
$ docker inspect $CONTAINER_ID  # 查看容器ID对应的IP_ADDRESS
$ curl $IP_ADDRESS  # 访问IP对应的容器
$ docker exec -it $CONTAINER_ID bash  # 在容器中打开一个交互式终端
$ ls  # 查看容器内的文件目录
$ exit  # 退出容器终端
```

# 升级docker engine
参考“安装docker engine指定版本”。

# 卸载docker engine
卸载docker engine和containerd包。
```bash
$ sudo yum remove docker-ce docker-ce-cli containerd.io
```

手动 删除镜像、容器、存储卷和个性化配置文件。
```bash
$ sudo rm -rf /var/lib/docker
```


Source: https://docs.docker.com/engine/install/centos/
