@[TOC](构建docker基础镜像并部署LNMP网站平台)

编写dockerfile时，尽量使用&&连接shell命令。尽量少使用 **RUN**、**COPY** 和 **ADD** 指令，因为这三个指令每次使用都会在镜像中增加分层（**layers**），导致最终生成的镜像变大。使用其他指令生成的是临时过渡层，不会增加镜像的大小。

# 构建nginx基础镜像
```bash
# Dockerfile-nginx 写入以下内容
FROM centos:7
RUN yum install -y gcc gcc-c++ make \
     openssl-devel pcre-devel gd-devel \
     iproute net-tools telnet wget curl && \
     yum clean all && \
     rm -rf /var/cache/yum/*  # 清理yum缓存以减小镜像
     
RUN wget http://nginx.org/download/nginx-1.15.5.tar.gz && \
     tar -zxf nginx-1.15.5.tar.gz && \
     cd nginx-1.15.5 && \
     ./configure --prefix=/usr/local/nginx \
     --with-http_ssl_module --with-http_stub_status_module && \
     make -j 4 && make install && \
     rm -rf /usr/local/nginx/html/* && \
     echo "ok" >> /usr/local/nginx/html/status.html && \
     cd / && rm -rf nginx-1.12.2* && \
     ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
     
ENV PATH $PATH:/usr/local/nginx/sbin
COPY nginx.conf /usr/local/nginx/conf/nginx.conf
WORKDIR /usr/local/nginx
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

```bash
# 创建镜像
$ docker build -t nginx:v1 -f Dockerfile-nginx .  # .表示将当前目录中的文件打包为镜像
$ docker images  # 查看镜像
$ docker run -d --name nginx01 -p 88:80 nginx:v1  # 使用创建的镜像运行容器
# 访问http://主机IP:88/status.html，显示ok
# 
# 利用已有镜像创建新的镜像
$ echo "Good day, mate" >> index.html
$ vim Dockerfile-index  # 写入以下内容
FROM nginx:v1
COPY index.html /usr/local/nginx/html
#
$ docker build -t nginx:v2 -f Dockerfile-index .
$ docker images  # 查看镜像
$ docker run -d --name nginx02 -p 89:80 nginx:v2
# 访问http://主机IP:89，显示Good day, mate
```

如果镜像拉取速度太慢，修改`/etc/docker/daemon.json`配置文件如下。

```bash
$ vi /etc/docker/daemon.json
{
	"registry-mirrors": ["https://registry.dockercn.com","https://mj9kvemk.mirror.aliyuncs.com"]
}
```


# 构建tomcat基础镜像
```bash
# Dockerfile-tomcat 写入以下内容
FROM centos:7
ENV VERSION=8.0.46

RUN yum install java-1.8.0-openjdk wget curl unzip iproute net-tools -y && \
     yum clean all && rm -rf /var/cache/yum/*
     
RUN wget https://archive.apache.org/dist/tomcat/tomcat-8/v8.0.46/bin/apache-tomcat-${VERSION}.tar.gz && \
     tar -zxf apache-tomcat-${VERSION}.tar.gz && \
     mv apache-tomcat-${VERSION} /usr/local/tomcat && \
     rm -rf apache-tomcat-${VERSION}.tar.gz /usr/local/tomcat/webapps/* && \
     mkdir /usr/local/tomcat/webapps/test && \
     echo "ok" > /usr/local/tomcat/webapps/test/status.html && \
     sed -i '1a JAVA_OPTS="-Djava.security.egd=file:/dev/./urandom"' usr/local/tomcat/bin/catalina.sh && \
     ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
     
ENV PATH $PATH:/usr/local/tomcat/bin
WORKDIR /usr/local/tomcat
EXPOSE 8080
CMD ["catalina.sh", "run"]  # 前台运行tomcat
```

```bash
# 创建镜像
$ docker build -t tomcat:v1 -f Dockerfile-tomcat .
$ docker run -d --name tomcat01 -p 8089:8080 tomcat:v1  # tomcat端口是8080
# 访问http://主机IP:8089/test/status.html
# 
# 创建新的镜像并部署war包（需要先去jenkins.io下载jenkins.war包）
$ vim Dockerfile  # 写入以下内容
FROM tomcat:v1
COPY jenkins.war /usr/local/tomcat/webapps/ROOT.war

# 创建镜像并运行tomcat容器
$ docker build -t tomcat:v2 -f Dockerfile .
$ docker run -d --name tomcat02 -p 8088:8080 tomcat:v2
# 检查容器中部署的war包
$ docker exec -it tomcat02 bash
# ls webapps/
ROOT   ROOT.war   test
# 访问http://主机IP:8088，可以查看部署的jenkins
```

# 构建php基础镜像
```bash
# Dockerfile-php 写入以下内容
FROM centos:7
RUN yum install epel-release -y && \
     yum install -y gcc gcc-c++ make gd-devel libxml2-devel \
     libcurl-devel libjpeg-devel libpng-devel openssl-devel \
     libmcrypt-devel libxslt-devel libtidy-devel autoconf \
     iproute net-tools telnet wget curl && \
     yum clean all && \
     rm -rf /var/cache/yum/*
     
RUN wget http://docs.php.net/distributions/php-5.6.36.tar.gz && \
     tar -zxf php-5.6.36.tar.gz && \
     cd php-5.6.36 && \
     ./configure --prefix=/usr/local/php \
     --with-config-file-path=/usr/local/php/etc \
     --enable-fpm --enable-opcache \
     --with-mysql --with-mysqli  --with-pdo-mysql \
     --with-openssl --with-zlib --with-curl --with-gd \
     --with-jpeg-dir --with-png-dir --with-freetype-dir \
     --enable-mbstring --with-mcrypt --enable-hash && \
     make -j 4 && make install && \
     cp php.ini-production /usr/local/php/etc/php.ini && \
     cp sapi/fpm/php-fpm.conf /usr/local/php/etc/php-fpm.conf && \
     sed -i "90a \daemonize = no" /usr/local/php/etc/php-fpm.conf && \
     mkdir /usr/local/php/log && \
     cd / && rm -rf php* && \
     ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
     
# 需要准备php.ini和php-fpm.conf配置文件
ENV PATH $PATH:/usr/local/php/sbin:/usr/local/php/bin
COPY php.ini /usr/local/php/etc/
COPY php-fpm.conf /usr/local/php/etc/
WORKDIR /usr/local/php
EXPOSE 9000
CMD ["php-fpm"]
```

```bash
# 创建镜像
$ docker build -t php:v1 -f Dockerfile-php .
$ docker images
# 运行容器并检查是否正常运行
$ docker run -d --name php01 php:v1
$ docker ps -l
$ docker exec -it php01 bash
# pwd
# ps -ef
# ./bin/php -v  # 查看php版本
```


# 快速部署LNMP网站平台
下面利用前面构建的基础镜像来搭建一个LNMP网站平台（Linux + nginx + mysql + php）。

```bash
# 创建自定义网络
$ docker network create lnmp
# 创建mysql容器
$ docker run -d --name lnmp_mysql --net lnmp \
--mount src=mysql-vol,dst=/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_DATABASE=wordpress mysql:5.7 --character-set-server=utf8
# 创建php容器
$ docker run -d --name lnmp_php --net lnmp \
--mount src=wwwroot,dst=/wwwroot php:v1
# 创建nginx容器（docker主机上需准备nginx.conf配置文件）
$ docker run -d --name lnmp_nginx --net lnmp -p 8000:80 \
--mount type=bind,src=$(pwd)/nginx.conf,dst=/usr/local/nginx/conf/nginx.conf \
--mount src=wwwroot,dst=/wwwroot nginx:v1
# 测试
$ cd /var/lib/docker/volumes/wwwroot/_data/
$ vi test.php  # 写入
<?php phpinfo();? >
# 访问http://主机IP:8000/test.php

# 部署wordpress博客
$ wget https://cn.wordpress.org/wordpress-4.9.4-zh_CN.tar.gz
$ tar -zxf wordpress-4.9.4-zh_CN.tar.gz  # 解压后得到wordpress文件夹
# 部署wordpress
$ cp -r wordpress /var/lib/docker/volumes/wwwroot/_data/
# 访问http://主机IP:8000/wordpress/
# 利用之前创建mysql容器时的信息配置wordpress
# 数据库名：wordpress，用户名：root，密码：123456，数据库主机：lnmp_mysql
```

如果不能自动创建`wp-config.php`配置文件，可能需要复制网站给出的配置文件内容，并在`/var/lib/docker/volumes/wwwroot/_data/wordpress/`目录下创建该配置文件。配置好的wordpress网站如下。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210206120718857.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NlYmFzdGllbjIz,size_16,color_FFFFFF,t_70#pic_center)

