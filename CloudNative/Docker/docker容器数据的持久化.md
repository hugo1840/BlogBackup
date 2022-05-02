@[TOC](docker容器数据的持久化)

在容器内创建的所有文件默认被存储在一个容器**可写层**（*writable container layer*），也就是意味着：
-	当容器被删除时其内部的数据不会被**持久化**（*persist*），且容器外的进程也难以获取容器内部的数据；
-	容器的可写层与容器运行所在的宿主机紧密耦合，容器内部的数据很难**迁移**到其他机器上；
-	向容器的可写层写入数据需要一个**存储驱动**（*storage driver*）来管理文件系统。存储驱动利用Linux内核来提供一个联合的文件系统。与使用**数据卷**（*data volumes*）直接将数据写入宿主机文件系统相比，这种使用存储驱动的方式降低了数据写入的性能。

# 容器的数据存储方法概览
对于Linux操作系统，docker提供了三种方式供容器向宿主机中存储数据：volumes、bind mounts 和 tmpfs。

 1. **Volumes**
由docker管理宿主机文件系统的一部分（`var/lib/docker/volumes/`）。非docker进程不应当被允许修改这部分文件系统。Volumes是docker数据持久化的最佳方式。

 2. **Bind mounts**
将宿主机中的任意文件或目录挂载到容器中。非docker进程可以在任何时候修改这些文件或目录。

3. **tmpfs**
容器数据仅挂载于宿主机的内存中，而不会写入其文件系统。即容器内部数据不会被持久化地存储到宿主机的磁盘上。
 
# Volume
## 创建和管理卷

```bash
$ docker volume create nginx-vol  # 创建卷
$ docker volume ls  # 列出卷
$ docker volume inspect nginx-vol  # 检查卷的详细信息（挂载点）
```

## 创建容器并挂载卷
挂载卷可以使用`--mount`或`-v|--volume`关键字。一般建议使用前者。两者的区别在于：在使用服务发现（service）时，仅支持`--mount`关键字。

```bash
# 使用--mount关键字挂载卷（src|source, dst|destination|target都可）
$ docker run -d --name=nginx03 --mount src=nginx-vol,dst=/usr/share/nginx/html nginx:latest
# 使用-v关键字挂载卷
$ docker run -d --name=nginx03 -v nginx-vol:/usr/share/nginx/html nginx:latest
$ docker inspect nginx03  # 查看”Mounts”部分中的挂载点信息
```

其中，source为宿主机`/var/lib/docker/volumes/`目录下的文件，destination或target为容器中的文件路径。创建容器时，挂载的卷如果不存在则会被docker自动创建。

## 移除容器和卷
```bash
$ docker stop nginx03
$ docker rm nginx03
$ docker volume rm nginx-vol
$ docker rm -f $(docker ps -a | awk '{print $1}')  # 移除所有容器（慎用）
```

# Bind Mounts
## 创建容器并绑定挂载
绑定挂载可以使用`--mount`或`-v|--volume`关键字。一般建议使用前者。两者的区别在于：使用`--mount`关键字时，如果docker主机上的源文件或源目录（source）不存在，不会自动创建，会抛出一个错误；反之，使用`-v`关键字时，docker会自动为你创建一个目录。

```bash
$ docker run -d -it --name=nginx04 \
--mount type=bind,src="$(pwd)"/app,dst=/usr/share/nginx/html nginx:latest
$ docker run -d --name=nginx04 -v "$(pwd)"/app:/usr/share/nginx/html nginx:latest
$ docker inspect nginx04  # 查看”Mounts”部分中的挂载点信息
```

## 绑定挂载非空目录
如果挂载目标路径（destination）在容器中是非空目录，该目录已有的内容将被隐藏。

```bash
$ docker run -d -it --name broken-container --mount type=bind,source=/tmp,target=/usr nginx:latest
# 因为容器中/usr目录下原有的数据被隐藏，导致容器运行异常所以会报错
docker: Error response from daemon: oci runtime error: container_linux.go:262:
starting container process caused "exec: \"nginx\": executable file not found in $PATH".
```

## 移除容器和绑定挂载

```bash
$ dokcer stop nginx04
$ docker rm nginx04
```

# Volumes vs Bind Mounts
Volumes支持：
-	多个容器之间==共享==数据；
-	多个容器可以==同时挂载相同的卷==；
-	当容器停止或被移除时，卷依然存在，除非明确删除卷；
-	将容器的数据存储在==远程主机或者其他存储上==；
-	将数据从一台docker主机==迁移==到另一台时，先停止容器，然后备份卷目录。

Bind Mounts支持：
-	从主机共享配置文件到容器。默认情况下，挂载主机/etc/resolv.conf文件到每个容器，提供DNS解析；
-	在docker主机上的开发环境和容器之间共享源代码。例如，将Maven target目录挂载到容器中。每次在docker主机上构建Maven项目时，容器都可以访问构建的项目包；
-	当docker主机的文件或目录结构保证与容器所需的绑定挂载一致时。


Source: https://docs.docker.com/storage/
