@[TOC](在k8s群集中部署Java应用)

# 项目迁移到k8s的流程
项目迁移到k8s的一般流程为：制作镜像、控制器管理Pod、暴露应用、对外发布应用、日志&监控。

一般企业会用到的三种镜像有：基础镜像（OS）、运行环境镜像（基础镜像+JDK/PHP/Tomcat/Nginx等）、项目镜像（运行环境镜像+项目打包）。

# 构建项目镜像
假设有一个打包好的tomcat-java-demo-master.zip压缩包，解压后的文件包括：src、db、pom.xml、Dockerfile和License等。其中，src目录下是项目的Java源代码，pom.xml中描述了Java项目编译环境依赖包，db目录下则是项目使用到的数据库sql文件。
```bash
$ unzip tomcat-java-demo-master.zip
$ cd tomcat-java-demo-master
$ ls
Dockerfile  src  db  pom.xml  LICENSE  README.md
$ scp db/tables_ly_tomcat.sql root@192.168.30.200:~  # 拷贝sql文件到数据库主机
```

部署应用之前需要确保k8s节点能够连接到数据库主机。假设MySQL数据库主机IP为192.168.30.200，连接命令为`mysql -h192.168.30.200 -uroot -p`。使用MySQL的`source`命令将sql文件导入数据库。

## 配置应用数据源
修改`src/main/resources/application.yml`文件，将datasource下的localhost替换为数据库主机IP地址，（如有必要）修改数据库的用户名和密码。
```bash
$ vi src/main/resources/application.yml
server:
  port: 8080
spring:
  datasource:
    # jdbc:mysql://192.168.30.200:3306/test，test为数据库名
    url: jdbc:mysql://localhost:3306/test?characterEncoding=utf-8
    username: root
    password: 12345
driver-class-name: com.mysql.jdbc.Driver
```

## 编译应用war包
为了编译Java源代码，需要在master节点部署jdk和maven环境。
```bash
$ yum install -y java-1.8.0-openjdk maven
$ mvn clean package -Dmaven.test.skip=true  # 项目编译打包并跳过单元测试
```
编译完成后会在当前目录下生成一个target目录，该目录下有一个.war包，可以部署到tomcat中。

## 打包项目镜像
编写Dockerfile将项目打包成镜像。
```bash
# 编写Dcokerfile
$ vi Dockerfile
FROM tomcat:v1  # tomcat:v1为运行环境镜像
LABEL maintainer xxx@xx.com
RUN -rf /usr/local/tomcat/webapps/*  # 删除原有镜像中部署的webapps
ADD target/*.war /usr/local/tomcat/webapps/ROOT.war  # 拷贝编译好的项目war包到tomcat/webapps/目录下

# 打包镜像
$ docker build -t java-demo:v1 -f Dockerfile .  
$ docker images 

# 将创建的镜像推送到私有Harbor仓库
$ docker login 192.168.30.201  # 登录到Harbor仓库
$ docker push 192.168.30.201/project/java-demo:v1

# 将镜像推送到Docker Hub公开仓库
$ docker push dockerhub_account/java-demo:v1
```

# 部署项目镜像到k8s
## 控制器管理Pod
由于部署的是无状态web应用，因此使用Deployment控制器即可。
```bash
# --dry-run -o yaml表示生成部署的yaml配置文件但并不进行实际的部署
$ kubectl create deployment java-demo --image=192.168.30.201/project/java-demo:v1 \
--dry-run -o yaml > java-demo-deploy.yaml
```

修改生成的java-demo-deploy.yaml文件，将其中的replicas设置为3，即部署3个镜像副本。修改保存后使用该文件运行Pod。
```bash
$ kubectl apply -f java-demo-deploy.yaml
$ kubectl get pods  # 查看生成的Pod
java-demo-8567655c52-2v567   RUNNING
java-demo-8567655c52-7w56y   RUNNING
java-demo-8567655c52-35atw   RUNNING

$ kubectl delete pod java-demo-8567655c52-7w56y  # 删除一个Pod
$ kubectl get pods  # 会自动生成一个新的Pod
java-demo-8567655c52-2v567   RUNNING
java-demo-8567655c52-sp59a   RUNNING
java-demo-8567655c52-35atw   RUNNING

$ kubectl logs java-demo-8567655c52-sp59a  # 查看Pod日志
$ kubectl get pods -o wide  # 显示Pod所在的节点名称
```

## Service暴露应用
部署项目镜像以后，就可以配置Service暴露外部访问端口了。
```bash
# 80为k8s集群内部访问端口，8080为tomcat容器服务端口，NodePort随机生成一个供集群外部访问的端口
$ kubectl expose deployment java-demo --port=80 --target-port=8080 \
--type=NodePort --dry-run -o yaml > svc.yaml
# 使生成的yaml文件生效
$ kubectl apply -t svc.yaml
$ kubectl get pods,svc  # 查看Pod和Service暴露的端口
```


Source: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#expose
