---
tags: [nginx]
title: Nginx安装与配置
created: '2023-07-09T05:53:07.685Z'
modified: '2023-07-09T06:43:53.112Z'
---

Nginx安装与配置

# 准备工作
创建nginx用户和安装目录：
```bash
[root@webserver ~]# groupadd nginx
[root@webserver ~]# useradd -g nginx -s /bin/bash nginx

[root@webserver ~]# mkdir /nginx
[root@webserver ~]# mv nginx-1.24.0.tar.gz /nginx
[root@webserver ~]# chown -R nginx:nginx /nginx
```

安装依赖包：
```bash
[root@webserver ~]# yum install -y gcc-c++
[root@webserver ~]# yum install -y pcre pcre-devel
[root@webserver ~]# yum install -y zlib zlib-devel
[root@webserver ~]# yum install -y openssl openssl-devel
[root@webserver ~]# yum install -y perl
```

# 编译安装nginx
解压安装包：
```bash
[nginx@webserver ~]$ su - nginx
[nginx@webserver nginx]$ cd /nginx
[nginx@webserver nginx]$ tar -zxvf nginx-1.24.0.tar.gz
```

编译安装configure：
```bash
# --prefix指定安装目录前缀
# --with-stream表示编译 TCP/UDP 代理模块
[nginx@webserver nginx]$ cd nginx-1.24.0/
[nginx@webserver nginx-1.24.0]$ ./configure --prefix=/nginx --with-stream 

checking for OS
 + Linux 3.10.0-1160.90.1.el7.x86_64 x86_64
checking for C compiler ... found
 + using GNU C compiler
 + gcc version: 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) 
checking for gcc -pipe switch ... found
checking for -Wl,-E switch ... found
checking for gcc builtin atomic operations ... found
...
...
Configuration summary
  + using system PCRE library
  + OpenSSL library is not used
  + using system zlib library

  nginx path prefix: "/nginx"
  nginx binary file: "/nginx/sbin/nginx"
  nginx modules path: "/nginx/modules"
  nginx configuration prefix: "/nginx/conf"
  nginx configuration file: "/nginx/conf/nginx.conf"
  nginx pid file: "/nginx/logs/nginx.pid"
  nginx error log file: "/nginx/logs/error.log"
  nginx http access log file: "/nginx/logs/access.log"
  nginx http client request body temporary files: "client_body_temp"
  nginx http proxy temporary files: "proxy_temp"
  nginx http fastcgi temporary files: "fastcgi_temp"
  nginx http uwsgi temporary files: "uwsgi_temp"
  nginx http scgi temporary files: "scgi_temp"
```

>:coffee:更多模块编译请参考：http://nginx.org/en/docs/configure.html

  
编译安装make：
```bash
[nginx@webserver nginx-1.24.0]$ make && make install
```

检查安装目录：
```bash
[nginx@webserver nginx-1.24.0]$ cd ..
[nginx@webserver nginx]$ ls
conf  html  logs  nginx-1.24.0  nginx-1.24.0.tar.gz  sbin

# 主配置文件为nginx.conf
[nginx@webserver nginx]$ ls conf/
fastcgi.conf          fastcgi_params          koi-utf  mime.types          nginx.conf          scgi_params          uwsgi_params          win-utf
fastcgi.conf.default  fastcgi_params.default  koi-win  mime.types.default  nginx.conf.default  scgi_params.default  uwsgi_params.default

[nginx@webserver nginx]$ ls html
50x.html  index.html

# error.log和access.log需要启动nginx后才会生成
[nginx@webserver nginx]$ ls logs

# 可执行文件
[nginx@webserver nginx]$ ls sbin/
nginx
```

# 启停nginx
配置root用户环境变量：
```bash
[root@webserver ~]# echo 'export PATH=$PATH:/nginx/sbin' >> .bash_profile
[root@webserver ~]# source .bash_profile
```

测试nginx（需要root用户）：
```bash
[root@webserver ~]# nginx -t
nginx: the configuration file /nginx/conf/nginx.conf syntax is ok
nginx: configuration file /nginx/conf/nginx.conf test is successful
```

启动nginx：
```bash
nginx -c /nginx/conf/nginx.conf
```

重载nginx（用于修改nginx配置文件后）：
```bash
nginx -s reload
```

停止nginx：
```bash
nginx -s stop   # 从OS层面直接暴力kill进程
nginx -s quit   # 等待nginx进程处理完任务再退出
```

# 配置系统服务
创建系统服务配置文件：
```bash
[root@webserver ~]# cat > /usr/lib/systemd/system/nginx.service << EOF
[Unit]
Description=nginx - high performance web server
Documentation=http://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/nginx/logs/nginx.pid
ExecStart=/nginx/sbin/nginx -c /nginx/conf/nginx.conf
ExecReload=/nginx/sbin/nginx -s reload
ExecStop=/nginx/sbin/nginx -s stop

[Install]
WantedBy=multi-user.target
EOF
```

加入开机自启：
```bash
[root@webserver ~]# systemctl daemon-reload
[root@webserver ~]# systemctl enable nginx
```

启停nginx：
```bash
[root@webserver ~]# systemctl start nginx
[root@webserver ~]# systemctl stop nginx
[root@webserver ~]# systemctl status nginx
```


**References**
【1】http://nginx.org/en/docs/configure.html


