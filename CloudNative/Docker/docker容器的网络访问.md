@[TOC](docker容器的网络访问)
# 网络访问模式
容器的子网络基于驱动，是可插拔的（pluggable）。默认情况下，存在以下几种驱动程序，它们提供核心的联网功能：
-	**Bridge**：网桥（bridge）是默认的网络驱动。当应用程序在单机模式（**standalone**）的容器中运行且需要通信时，通常会使用网桥驱动。
-	**Host**：对于单机模式的容器，容器与docker主机之间没有网络隔离，容器可以直接使用主机的网络。
-	**Overlay**：overlay网络驱动将多个docker守护进程（daemon）连接在一起，使得swarm服务之间能够互相通信。也适用于swarm服务和单机容器、或者两个不同的单机容器之间的通信。
-	**Macvlan**：macvlan网络驱动允许用户给每个容器分配一个MAC地址，使得这些容器在网络中可以被看作物理设备。
-	**None**：禁用容器内所有网络。通常与自定义网络驱动一起使用。
-	**第三方网络插件**（third-party network plugins）：从Docker Hub下载的第三方网络驱动程序。

## Bridge networks
当用户启动docker后，会自动创建一个默认的bridge网桥（名字也是bridge）。==除非特别注明使用其他网络，所有新运行的容器都会连接到此bridge网络==。用户也可以自定义一个bridge网络，且用户自定义的bridge优于docker自动创建的默认bridge网络。默认创建的bridge网络不建议用于生产环境。

用户自定义bridge网络与容器默认bridge网络的主要区别包括：
-	用户自定义bridge网络可以提供容器之间的**DNS自动解析**（automatic DNS resolution）。默认bridge网络仅允许容器之间通过IP地址访问彼此；
-	用户自定义bridge网络隔离性更好。所有未使用`--network`关键字的容器都会被自动加入默认的bridge网络，因此存在一定的风险；
-	容器在运行中（on the fly）可以随机加入或退出用户自定义网络。但如果要从默认bridge网络中移除容器，需要先停止运行该容器；
-	用户自定义网络支持更加灵活的配置。对于默认bridge网络，其中的所有容器都只能使用完全相同的网络配置（比如MTU和iptables配置），并且修改配置后需要重启Docker本身。用户自定义网络可以通过使用`docker network create`命令，为不同的应用分组分别进行网络配置。
-	默认bridge网络中的容器可以通过使`--link`标志（flag）共享环境变量。用户自定义网络不支`--link`标志，但是可以使用volumes、docker-compose、swarm服务等更高级的方式共享变量和数据。

==连接到同一个用户自定义网络的容器彼此之间暴露所有端口==。使用`-p`或`--publish`标志发布容器端口使其可以被不同网络中的容器和非docker主机访问。

**管理用户自定义网桥**
```bash
$ docker network create my-net  # 创建自定义bridge
$ docker network rm my-net  # 移除网络
# 运行容器并连接到自定义网络，将容器的80端口发布到docker主机的
# 99端口（外部客户端可以通过主机IP:99访问容器的80端口）
$ docker create --name my-nginx --network my-net --publish 99:80 nginx
$ docker ps -l  # 查看容器端口映射
# 将正在运行的容器连接到已经存在的用户自定义网络
$ docker network connect my-net my-nginx
$ docker network disconnect my-net my-nginx  # 断开连接
```
## Host networking
主机模式下，容器共享主机的网络命名空间（networking namespace）。容器自己不会被分配IP地址，且容器端口映射（port mapping）不会生效，因此`-p`、`--publish`、`-P`、`--publish-all`选项会被忽略。Host网络模式仅在Linux操作系统中支持。
```bash
# 运行容器并连接到host网络，-rm表示如果容器退出或停止就移除容器
$ docker run --rm -d --network host --name my_nginx nginx 
```
打开`http://主机IP:80/`访问Nginx。需要docker主机的80端口可用（参见 https://hub.docker.com/_/nginx/）。

检查网络接口：`ip addr show`
查看80端口绑定的进程：`sudo netstat -tulpn | grep :80`
停止容器：`docker container stop my_nginx`

## Overlay networks
Overlay网络驱动在不同的docker守护进程主机之间创建了一个分布式的网络，特别适用于不同docker主机上的容器进行通信、或者多个使用swarm服务的应用之间进行通信。

具体使用方法参见 https://docs.docker.com/network/overlay/。

## Macvlan networks
有些应用程序，比如监控网络流量的程序，希望直接连到物理网络。在这种情况下，可以使用macvlan网络驱动给容器的虚拟网络接口（virtual network interface）分配一个物理地址（MAC address）。

具体使用方法参见 https://docs.docker.com/network/macvlan/。

## None: Disable networking
在运行容器时使用`--network none`标志可以禁用容器内部网络。此时，在容器内部仅创建了回环接口（loopback device）。

## 网络测试
使用busybox镜像测试以上网络模式。
```bash
$ docker pull busybox  # 拉取镜像
# 测试一：默认bridge网络
$ docker run -it --name bs01 busybox
/ # ifconfig  # 会输出eth0和lo两块网络设备
/ # exit
# 测试二：host网络
$ docker run -it --name bs02 --network host busybox
/ # ifconfig  # 与在docker主机中的输出一致，包括docker0、ens33和lo
/ # exit
# 测试三：禁用容器网络
$ docker run -it --name bs03 --network none busybox
/ # ifconfig  # 仅输出lo回环设备
/ # exit
# 测试四：自定义bridge网络
$ docker network create test-net
$ docker run -it --name bs04 --network test-net busybox
$ docker run -it --name bs05 --network test-net busybox
$ docker run -it --name bs05 --net container:bs04 busybox  # 与前一句命令效果相同，表示使bs05加入bs04的网络
/ # ping bs04  # 自定义网络支持DNS自动解析
/ # ping $HOSTNAME_of_bs04  # DNS自动解析
/ # ip addr  # 会输出eth0和lo两块网络设备
/ # exit
$ docker rm -f $(docker ps -a | awk '{print $1}')  # 移除所有容器
```

# 网络访问原理
在docker主机上有容器在运行时，在主机中执行`ifconfig`或`ip addr`命令的输出一般会包括以下内容：
-	回环设备**lo**
-	主机网卡**ens33**
-	**docker0**网桥：连接到bridge网络的容器的网关。
-	以 **veth** 开头的网络设备：其数量等于连接到bridge网络的容器数目，且与每个容器内的eth0网卡一一配对。
-	以 **br-** 开头的网桥：代表用户自定义的bridge网络。

当bridge网络中的容器将数据发送到外网时，数据包先通过容器内eth0和主机上veth网卡组成的接口转发到主机上的docker0网桥，然后通过源地址转换（Source Network Address Translation, **SNAT**）转发到主机的物理网卡eth0，最终发送到外网。容器接收外网传回的数据包时，使用的则是目的地址转换（Destination Network Address Translation, **DNAT**）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210130214014508.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210130214045712.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)


Sources:
\[1] https://docs.docker.com/network/
\[2] https://docs.docker.com/engine/reference/commandline/network/

