@[TOC](Dockerfile的介绍与使用)

Docker可以通过读取dockerfile自动构建（build）定制化镜像。

# Dockerfile格式
Dockerfile包含若干行从上至下按顺序排列的指令。其格式为`INSTRUCTION args`。Dockerfile的内容必须以FROM指令开始（但是前面可以有parser directives语句、注释语句或者ARG指令）。
```bash
# directive=value  # parser directive，非注释
# escape=\  # parser directive，定义转义字符
# 这里可以写注释（注释不能出现在parser directives之前）
FROM ubuntu:18.04
COPY . /app
RUN make /app
CMD python /app/app.py
```

# Dockerfile指令
Dockerfile中支持很多指令，比较重要的有FROM、RUN、CMD、ENTRYPOINT、EXPOSE、COPY、USER、WORKDIR。

**FROM**
表示构建的新镜像是基于哪个镜像。格式为

`FROM [--platform=<platform>] <image>[:<tag>] [AS <name>]`

其中，platform可以为linux/amd64、linux/arm64、windows/amd64，name可用于指代正在被创建的镜像。

FROM指令在一个dockerfile中可以出现多次，用于创建多个镜像、或者作为创建其他镜像的依赖镜像。FROM指令可以使用ARG指令中声明的值。
```bash
ARG CODE_VERSION=latest
FROM base:${CODE_VERSION}
```

**RUN** 
构建镜像时运行的shell命令。

`RUN <command>` （shell格式）
`RUN ["executable", "param1", "param2"]` （exec格式）

RUN指令后面接的shell命令可以使用“\”标志换行。
```bash
RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'  # shell格式
RUN ["/bin/bash", "-c", "echo Hello"]  # exec格式
```

**CMD** 
CMD用于指定在容器运行时执行的命令。每个dockerfile里只能有一个CMD指令，如果有多个，==只有最后一个会生效==。CMD指令也可以为ENTRYPOINT指令提供默认参数。

`CMD ["executable", "param1", "param2"]`（exec格式）
`CMD command param1 param2`（shell格式）
`CMD ["param1", "param2"]`（与ENTRYPOINT指令配合使用）

```bash
# shell格式中shell命令通过/bin/sh -c执行
FROM Ubuntu
CMD echo "This is a test." | wc -
# exec格式
FROM Ubuntu
CMD ["/usr/bin/wc", "--help"]
```

**LABEL**
向镜像中添加元数据（metadata）。原有的MAINTAINER指令已被LABEL取代。

`LABEL <key>=<value> <key>=<value> …`

包含空格的值需要用双引号包含起来。可以使用“\”换行。

```bash
LABEL "com.example.vendor"="ACME Incorporated"
LABEL version="1.0"
LABEL description="This text illustrates \
That label values can span multiple lines."
```
查看镜像的LABELs：`docker image inspect --format='' my_image`


**EXPOSE** 
声明容器运行的服务端口（默认为TCP端口）。

`EXPOSE <port> [<port>/<protocol>…]`

EXPOSE只是声明端口，并不会发布端口。因此在运行容器时，需要使用`docker run`的`-p`或`-P`选项手动发布端口。
```bash
# Dockerfile文件里
EXPOSE 80/tcp
EXPOSE 80/udp
# 运行容器时
Docker run -p 80 :80/tcp -p 80:80/udp …
```

**ENV** 
设置环境变量。后续指令都可以使用定义的环境变量的值，且该环境变量会留存于使用最终生成镜像运行的容器中。如果只是在镜像创建过程中需要某个变量，建议使用ARG指令。

`ENV <key>=<value>`

变量值中的空格可以使用引号或反向斜杠转义。
```bash
ENV key1="John Wick"
ENV key2=John\ Snow
ENV key3=Schumacher
```
查看环境变量：`docker inspect `
修改环境变量：`docker run --env <key>=value`

**ADD**
从src处拷贝新文件、目录或者远程文件URL，并将它们添加到镜像文件系统的dest路径中。

`ADD [--chown=<user>:<group>] <src>… <dest>`
`ADD [--chown=<user>:<group>] ["<src>", …, "<dest>"]`

其中，加上引号是为了转义路径中的空格字符。源路径src中可以使用通配符。
```bash
ADD hom* /absoluteDir/
ADD test.txt relativeDir/
ADD --chown=55:mygroup files* /somedir/
ADD --chown=bin files* /somedir/  # 使用相同的用户名和组名bin
```
其他注意事项：
-	源路径src必须位于创建镜像的上下文中（工作目录及其子目录），而不能是形如`ADD ../somedir /somedir`；
-	如果源路径src是一个目录，该目录下的所有内容（包括文件系统元数据）都会被拷贝；
-	如果源路径src是一个形如`http://example.com/foobar`的URL文件，且目标路径dest以“/”结尾，下载的URL文件会被存放在`dest/foobar`目录下；
-	如果源路径src是一个本地压缩文件，拷贝后会被自动解压缩为一个目录；
-	如果目标路径dest不以“/”结尾，则会被当成一个常规文件，源路径中的内容会被写入该文件；
-	如果目标路径dest不存在，则会被自动创建。

**COPY** 
从src处拷贝新文件或目录，并将它们添加到容器文件系统的dest路径中。

`COPY [--chown=<user>:<group>] <src>… <dest>`
`COPY [--chown=<user>:<group>] ["<src>", …, "<dest>"]`

其用法与ADD指令类似。

**ENTRYPOINT** 
ENTRYPOINT用于指定运行容器时执行的命令。如果有多个ENTRYPOINT，==只有最后一个会生效==。ENTRYPOINT指令有两种格式。

`ENTRYPOINT ["executable", "param1", "param2"]`  （exec格式）
`ENTRYPOINT command param1 param2` （shell格式） 

对于 **exec** 格式，`docker run <image>`后的参数都会被当成ENTRYPOINT的参数，并且会覆盖CMD中定义的参数；如果`docker run <image>`后面没有参数，那么CMD中定义的参数就会被当成ENTRYPOINT的参数。如果想要覆盖ENTRYPOINT中的内容，可以使用`docker run --entrypoint`命令。

对于 **shell** 格式，不允许在CMD指令或`docker run`命令中使用任何参数。Shell格式的缺点在于：ENTRYPOINT指令会作为`/bin/sh -c`的子命令执行，因此不支持传递信号。也就是说，`docker stop <container>`命令无法将SIGTERM信号传递给ENTRYPOINT指令。

```bash
# dockerfile
FROM Ubuntu
ENTRYPOINT ["top", "-b"]
CMD ["-c"]
# 运行容器并在容器中执行top –H命令
$ docker run -it -rm --name test01 top -H  
# 可以看到PID 1对应的执行命令为top -b -H，docker run后面的-H参数覆盖了CMD中的-c选项
$ docker exec -it test01 ps aux  
$ docker stop test01  # 停止容器运行
```

总结一下ENTRYPOINT和CMD配合使用的注意事项：
-	一个dockerfile中应当至少有CMD或者ENTRYPOINT两者中的一种指令；
-	如果要把容器当作可执行程序使用，应当在dockerfile中定义ENTRYPOINT指令；
-	使用ENTRYPOINT指令时，应当使用CMD指令为ENTRYPOINT指令定义默认参数；
-	使用`docker run`运行容器时，如果为容器定义了运行参数，将会覆盖CMD中为ENTRYPOINT指令定义的参数。


**VOLUME**
VOLUME指令根据给定的名称创建挂载点。
```bash
FROM Ubuntu
RUN mkdir /myvol
RUN echo "hello world" > /myvol/greeting
VOLUME /myvol
```

**USER** 
为RUN、CMD和ENTRYPOINT执行命令指定运行用户。 
格式：`USER <user>[:<group>]` 或 `USER <UID>[:<GID>]`

**WORKDIR** 
为RUN、CMD、ENTRYPOINT、COPY和ADD指定工作目录。

**ARG** 
ARG指令用于定义在创建镜像时可以使用的变量。与环境变量不同，ARG定义的变量不会留存于使用最终生成镜像运行的容器中。ARG定义的变量可以在`docker build`命令中使用。

`ARG <name>[=<default value>]`
`docker build --build-arg <varname>=<value>`

==ENV定义的环境变量会覆盖ARG定义的同名变量的值==。使用示例如下。

```bash
# dockerfile
FROM Ubuntu
ARG CONT_IMG_VER
ENV CONT_IMG_VER=v1.0.0
RUN echo $CONT_IMG_VER
# build镜像
$ docker build --build-arg CONT_IMG_VER=v2.0.1  # 实际使用的是v1.0.0
```

**ONBUILD**
ONBUILD指令用于向镜像中添加一个稍后执行的触发指令（trigger instruction）。该触发指令会在当前镜像被用于创建另一个镜像时被执行（会在其dockerfile中的FROM指令之后立即执行）。

**STOPSIGNAL**
STOPSIGNAL指令用于定义发送给容器的退出信号，比如SIGKILL或者数字9。

**HEALTHCHECK** 
HEALTHCHECK指令为docker定义容器的健康检查。 

**SHELL**
SHELL指令可以覆盖dockerfile指令的shell格式中使用的默认shell。Linux系统中的默认shell为`[“/bin/sh", "-c"]`，Windows系统中默认为`["cmd", "/S", "/C"]`。

```bash
FROM Microsoft/windowsservercore
SHELL ["powershell", "-command"]
RUN Write-Host hello
SHELL ["cmd", "/S", "/C"]
RUN echo hello
```

# Build镜像
`docker build`命令可以利用dockerfile或者上下文（context）创建镜像。指令格式为`docker build [OPTIONS] PATH | URL | -`。常用的命令选项为`--file | -f`（后面接dockerfile的名称）和`--tag | -t`（后面接镜像的名称`name:tag`）。



Source: https://docs.docker.com/engine/reference/builder/ 
