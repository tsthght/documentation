### Dockerfile实战

#### 1 Dockerfile介绍

Dockerfile是Docker用来**构建镜像**的文本文件，包含自定义的**指令和格式**，通过docker build 命令构建镜像。



#### 2 使用Dockerfile部署MySQL5.7环境
##### 2.1 准备工作

需要：a 创建一个工作目录；b 在工作目录中生成一个 mysql5.7的配置文件my.cnf


```
# 创建 工作文件夹
mkdir docker_mysql57_env
cd docker_mysql57_env

```

my.cnf 内容如下：

```
# cat my.cnf
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
symbolic-links=0
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
server-id=1001
log-slave-updates=true
binlog-format=ROW
gtid-mode=on
enforce-gtid-consistency=true
log_bin=mysql-bin
```

##### 2.2 编写Dockerfile

```
# 指令不区分大消息，通常建议大写
# 基于centos6.6镜像构建我们的镜像
FROM centos:6.6
# 镜像制作者的信息。可以通过docker inspect 命令查看相应字段
MAINTAINER tsthght "781181214@qq.com"
# 安装软件，比如需要安装wget
RUN yum install -y wget
# 将mysql5.7 yum源 下载到 /tmp目录，并安装
WORKDIR /tmp
RUN wget https://repo.mysql.com/mysql57-community-release-el6.rpm
RUN rpm -ivh mysql57-community-release-el6.rpm
# 安装mysql,这一步耗时比较长（当前网络情况下，大概等待2-3分钟）
RUN yum install -y mysql-community-server.x86_64
# 设置环境变量（仅仅为了演示，在当前示例中无意义）
ENV APP_PATH /tmp
# 将mysql配置文件拷贝到/etc/目录
RUN rm -rf /etc/my.cnf
ADD ./my.cnf /etc/
# 容器被创建时，自动启动MySQL
RUN service mysqld start
```

##### 2.3 构建镜像

```
>docker build -t mysql57_env:1.0 .
```

##### 2.4 查看构建的镜像

```
>docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
mysql57_env             1.0                 07787d48dc5f        19 seconds ago      267MB
```

##### 2.5 使用镜像创建容器

```
docker run -v ~/docker-data:/data -it -d --name mysql57_2.1 mysql57_env:1.0 /bin/bash
docker start mysql57_2.1
docker exec -it mysql57_2.1 /bin/bash
```

##### 2.6 简单验证

```
# 验证ENV设置
>echo $APP_PATH
# 验证安装的mysql
>rpm -qa|grep mysql
# 验证mysql是否已经启动
>ps -ef|grep mysqld
# 验证拷贝的my.cnf
>cat /etc/my.cnf
# 验证wget下载的rpm包在/tmp目录
> ls -alh /tmp
```

##### 2.7 相关命令

```
# 查看镜像
docker images
# 删除镜像
docker rmi [image-id]
# 启动容器
docker start [NAMES]
# 停止容器
docker stop [NAMES]
# 删除容器
docker rm [NAMES]
# 查看容器异常启动的日志
docker logs [CONTAINER ID]
```

##### 2.8 上传镜像

```
# 1 在https://hub.docker.com/上注册账号(我的账号是 tsthght)
# 2 登陆到docker hub
>docker login
# 3 修改镜像文件标签
>docker tag mysql57_env:1.0 tsthght/mysql57:1.0
# 4 上传镜像
>docker push tsthght/mysql57:1.0
```

#### 3 Dockerfile需要注意

- 谨慎选择基础镜像，尽量选择官方镜像，不同镜像的大小不同，通常：busybox < debian < centos < ubuntu。

- 充分利用缓存，尽量将所有的Dockerfile文件中相同的部分都放到最前面，不同的部分往后放。

- 尽量不要在Dockerfile中使用EXPOSE指令做端口映射，会影响移植性。

- 使用Dockerfile共享Docker镜像，方便版本管理/追踪变化，清楚构建过程，确定性强。