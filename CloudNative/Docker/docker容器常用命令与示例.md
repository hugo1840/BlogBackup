@[TOC](docker容器常用命令与示例)
# 管理镜像常用命令
| 指令 | 描述 |
|--|--|
| docker image ls | 列出镜像 |
| docker image build | 从dockerfile构建镜像 |
| docker image history 或 docker history | 查看给定镜像的历史 |
| docker image inspect 或 docker inspect | 显示镜像的详细信息 |
| docker image pull 或 docker pull | 从镜像仓库拉取镜像 |
| docker image push 或 docker push | 推送镜像到镜像仓库 |
| docker image rm 或 docker rmi | 移除镜像 |
| docker image prune | 移除未使用的镜像（没有被标记或被任何容器引用） |
| docker image tag 或 docker tag | 创建一个引用源镜像标记目标镜像 |
| docker export | 导出容器文件系统到tar归档文件 |
| docker image import 或 docker import | 从tar归档文件导入容器文件系统 |
| docker image save 或 docker save | 保存一个或多个镜像到tar归档文件 |
| docker image load 或 docker load | 从tar归档文件或者标准输入加载镜像 |

# 创建容器常用选项
以下选项用于`docker run`和`docker container run`命令。
| 选项 | 描述 |
|--|--|
| -i, -\-interactive | 交互式 |
| -t, -\-tty | 分配一个伪终端 |
| -d, -\-detach | 在后台运行容器并打印容器ID |
| -e, -\-env | 设置环境变量 |
| -p, -\-publish | 发布容器端口到主机 |
| -P, -\-publish-all | 发布容器所有exposed的端口到宿主机随机端口 |
| -\-name | 指定容器名称 |
| -h, -\-hostname | 设置容器主机名 |
| -\-ip | 指定容器IPv4地址，只用于自定义网络 |
| -\-network | 连接容器到一个网络 |
| -\-mount | 挂载文件系统到容器 |
| -v, -\-volume | 绑定挂载一个卷 |
| -\-restart | 容器退出时的重启策略，默认no，可选\[always\|on-failure\] |

**举例**

```bash
$ docker run -itd centos  # 在后台运行一个centos容器
$ docker ps -l  # 列出最新创建容器的状态（包括Container ID）
$ docker top $Container_ID  # 查看容器正在运行的进程
```

命令格式：`docker container run [OPTIONS] IMAGE [COMMAND] [ARG …]`

```bash
# 运行容器并设置容器名称、环境变量和端口转发（80端口对应http协议）
$ docker container run -d --name webserver01 -e test_val=1024 -p 88:80 nginx  
$ docker ps -l  # 查看容器进程状态
# 在浏览器中通过宿主机IP:88端口访问容器
$ docker logs webserver1  # 查看容器访问日志
$ docker exec -it webserver01 bash  # 打开容器终端
$ echo $test_val  # 查看环境变量
$ exit
# 选项大写-P会为容器随机选择一个暴露端口发布到主机
$ docker container run -d --name webserver02 -P nginx  
$ docker ps -l  # 查看暴露的容器端口（PORTS栏：0.0.0.0:32769->80/tcp
```


# 容器资源限制
以下选项用于`docker run`和`docker container run`命令。
| 选项 | 描述 |
|--|--|
| -m, -\-memory | 容器可以使用的最大内存量 |
| -\-memory-swap | 交换区的磁盘容量限制（内存+交换区），-1表示无限制 |
| -\-memory-swappiness=\<0-100\> | 容器使用swap分区的百分比：0-100，默认为-1 |
| -\-oom-kill-disable | 禁用Out-Of-Memory Killer，防止系统自动杀死占用内存过高的进程 |
| -\-cpus | 可使用的CPU数量 |
| -\-cpuset-cpus | 限制容器使用特定的CPU核心，如0-3, 0, 1 |
| -c, -\-cpu-shares | CPU共享（相对权重） |

**示例**

```bash
# 允许容器最多使用500M内存和100M的swap分区
$ docker run -d --name nginx01 --memory="500m" --memory-swap="600m" --oom-kill-disable nginx  
$ docker stats --no-stream nginx01  # 查看容器的资源使用状况
```

 
# 管理容器的常用命令
| 选项 | 描述 |
|--|--|
| docker ps | 列出容器 |
| docker inspect | 查看容器详细信息 |
| docker exec | 在运行容器中执行命令 |
| docker commit | 从容器创建一个新镜像 |
| docker cp | 在本地文件系统和容器之间拷贝文件 |
| docker logs | 获取容器日志 |
| docker port | 列出或指定容器端口映射 |
| docker top | 显示容器正在运行的进程 |
| docker stats | 显示容器的资源使用统计 |
| docker stop/start | 停止/启动容器 |
| docker rm | 删除容器 |

**示例**
命令格式：`docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]`

```bash
$ docker commit nginx01 nginx:webserver01  # 提交新镜像
$ docker images
REPOSITORY	   TAG	         IMAGE ID	    …
nginx	       webserver01	 3942abf7a25	…
nginx	       latest	     dgcf2657c39	…
$ docker run -d --name nginx02 nginx:webserver01  # 利用提交的新镜像创建容器
$ docker cp nginx.tar nginx02:/  # 将本地文件拷贝到容器根目录下
$ docker exec -it nginx02 ls /  # 查看容器根目录
```

Source: https://docs.docker.com/engine/reference/commandline/docker/
