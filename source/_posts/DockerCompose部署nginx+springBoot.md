---
title: Docker Compose部署nginx+springBoot
date: 2018-09-22 23:59:47
urlname: docker-compose
tags:
  - Docker Compose
  - nginx
  - java
  - SpringBoot
categories: docker
---
![](2cc27fb6da554a7e81c6979ebe7e81ae.jpg)

## **Docker Compose概述**

Compose是一个用于定义和运行多容器Docker应用程序的工具。使用Compose，通过YAML文件来配置应用程序的服务。然后，使用单个命令，可以从配置中创建并启动所有服务。通俗一点就是你通过配置YAML文件，你可以同时构建和启动多个Docker容器，而且不必一个个的启动，通过编排，将所有依赖的Docker一个命令全启动。
Compose适用于所有环境：生产，登台，开发，测试以及CI工作流程。
<!--more-->
### **使用Compose基本上是一个三步过程：**
定义您的应用程序环境，Dockerfile以便可以在任何地方进行复制。
定义构成应用程序的服务，docker-compose.yml 以便它们可以在隔离环境中一起运行。
Run docker-compose up和Compose启动并运行整个应用程序。
`docker-compose.yml` 文件是这样的

```yaml
version: '3'
services:
  web:      #容器的名称
    build: . #构建web这个容器的Dockerfile的目录
    ports:   #端口的映射
    - "5000:5000"  
    volumes:   #文件挂载
    - .:/code
    - logvolume01:/var/log
    links:   #依赖的容器
    - redis
  redis:
    image: redis  #使用的镜像
volumes:
  logvolume01: {}
```
## **常用命令**
docker-compose up -d nginx  构建建启动nignx容器 后台启动
docker-compose exec nginx bash  进入到nginx容器中
docker-compose down   删除所有nginx容器,镜像
docker-compose ps   显示所有正在运行的容器
docker-compose restart nginx   重新启动nginx容器
docker-compose build nginx    构建镜像        
docker-compose build --no-cache nginx   不带缓存的构建。
docker-compose logs  nginx    查看nginx的日志 
docker-compose logs -f nginx   查看nginx的实时日志
docker-compose config  -q   验证（docker-compose.yml）文件配置，当配置正确时，不输出任何内容，当文件配置错误，输出错误信息。 
docker-compose events --json nginx       以json的形式输出nginx的docker日志
docker-compose pause nginx    暂停nignx容器
docker-compose unpause nginx     恢复ningx容器
docker-compose rm nginx           删除容器（删除前必须关闭容器）
docker-compose stop        停止所有容器
docker-compose stop nginx       停止nignx容器
docker-compose start nginx       启动nignx容器

更多命令  [**参考这里**](https://docs.docker.com/compose/compose-file/#context "**参考这里**")

## **第一个Docker Compose的例子：**

这个例子，用Docker Compose 来编排启动两个springBoot的app。
app.jar还是用上次Docker那篇文章的app， 文章地址  **[##点击这里##](https://llss6887.github.io/2018/09/22/docker-springboot/ "##点击这里##")**

### **安装**

**1.从github上下载docker-compose二进制文件安装**
下载最新版的docker-compose文件 

$ sudo curl -L
https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o/usr/local/bin/docker-compose

添加可执行权限 
$ sudo chmod +x /usr/local/bin/docker-compose

测试安装结果 
$ docker-compose --version 
docker-compose version 1.16.1, build 1719ceb

**2.pip安装**

$ sudo pip install docker-compose
![](e38ab15b591a4f9c9dbabecdb8793052.png)
安装完毕！

在/home/docker-compose/文件夹下创建 docker-compose.yml文件

先生成app的容器，Dockerfile内容如下：
```yaml
FROM java:8
MAINTAINER "liss"<168337573@qq.com>
WORKDIR  /tmp
RUN mkdir /home/apps/
EXPOSE 8080
CMD java -jar /home/apps/app.jar  #启动jar的命令
```
执行 build -t app:v1 . 生成一个app:v1的容器
![](c5b13dd8f05c4cceae93e980a24eb8cc.png)

开始编写docker-compose.yml文件，内容如下：

```yaml
version: '2'  #版本名称  必须写
services:
  app-1:  #定义名称
    container_name: "app-1"  #容器名称
    restart: always  #重启策略
    image: "app:v1"  #用的哪个镜像
    working_dir: /tmp  #工作目录
    volumes:   #文件挂载
     - /home/apps/:/home/apps/
    ports:  #映射端口
      - "9001:8080"
  app-2:
    container_name: "app-2"
    restart: always
    image: "app:v1"
    working_dir: /tmp
    volumes:
     - /home/apps/:/home/apps/
    ports:
      - "9002:8080"
```
执行 `docker-compose up `启动编排，此模式为前端启动。

后台启动命令为 `docker-compose up -d`

![![](2caa80527dcb4f05962a9345562cb3f8.png)](d37fc6dcd08549408c7b4fc68f7ee240.png)
启动完毕！ 查看docker正在运行的容器： docker ps 或者docker-compose ps

![](c56b40409b844923959b540069c96e23.png)
一切正常！ 我们来访问下这个服务器的9001端口和9002端口
![](1eb4d903910e4fccb8629f29987c2778.png)
![](434c760dbe5846b8bbac38721a29213f.png)
我们看到两个都没有问题，本次编排成功。

#### **下面我们看用nginx如何做负载均衡**
首先安装nginx的容器 并配置nginx.conf的负载均衡，为了能直观看到效果，我们将app1的html内容改为 `this is my app1`, app2改为 `this is app2 ` app的容器还用之前的app:v1这个容器

下载nginx安装包，地址为：[http://nginx.org/en/download.html](http://nginx.org/en/download.html "http://nginx.org/en/download.html")

创建nginx的Dockerfile文件：
```yaml
FROM centos:6
MAINTAINER "liss"<168337573@qq.com>
RUN yum install -y gcc gcc-c++ make openssl-devel pcre-devel  #安装nginx必须的库
ADD nginx-1.12.2.tar.gz /tmp #将下载的nginx放到Dockerfile一个目录 并添加到容器中
RUN cd /tmp/nginx-1.12.2 && ./configure --prefix=/usr/local/nginx && make -j 2 && make install #执行nginx的安装


RUN rm -f /usr/local/nginx/conf/nginx.conf #删除安装的nginx配置文件
EXPOSE 80 #对外端口
CMD ["/usr/local/nginx/sbin/nginx", "-g", "daemon off;"] #启动命令
```
执行 `docker build -t nginx:app .` 创建nginx容器。
修改一个nginx.conf 
```html
upstream app {
      server app-1:8080;  #compose中的名称和端口
      server app-2:8080;
    }

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  /home/logs/nginx/access/host.access.log  main;

        location / {
            proxy_pass http://app; #负载均衡配置
            proxy_set_header Host $host:$server_port;
            proxy_set_header X-Forwarded-Host $server_name;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header REMOTE-HOST $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header HTTP_X_FORWARDED_FOR $remote_addr;
           #proxy_set_header Forwarded $proxy_add_forwarded;

        }
```

docker-compose.yml文件如下：
```yaml
version: '2'
services:
  nginx:
     container_name: "nginx-app"
     image: "nginx:app"
     restart: always
     ports:
      - "8008:80"  #我的服务80端口被占着 就映射到了8008端口
     volumes:
      - ./nginx/nginx.conf:/usr/local/nginx/conf/nginx.conf
  app-1:
    container_name: "app-1"
    restart: always
    image: "app:v1"
    working_dir: /tmp
    volumes:
     - ./app1/:/home/apps/
    expose:  #容器暴露的端口
      - "8080"
    depends_on:  #添加的依赖
     - "nginx"
  app-2:
    container_name: "app-2"
    restart: always
    image: "app:v1"
    working_dir: /tmp
    volumes:
     - ./app2/:/home/apps/
    expose:
      - "8080"
    depends_on:
     - "nginx"
```
现在，我们来启动我们的编排任务：

`docker-compose up -d `  后台启动
`docker-compose logs -f` 实时查看启动的容器日志

![](e24f0716370444c8afbc30f43924655a.png)
![](8a261bf3e7b34dd989b3d06ce0bc54da.png)

启动完毕，我们来访问下8008是否成功
![](0ed272b56fad4a0c9bda9656f7f1bff1.gif)

到这里就成功了！ 

## **结语**
这里只是一个简单的例子。实际生产中可能需要与tomcat、mysql、redis等等这些一起编排，写法都是一个写法，只不过根据自己的需求，编写自己的YAML就可以了。  学无止境   每一天都是崭新的最后一天。




源码地址： [**戳这里**](https://github.com/llss6887/docker.git "**戳这里**")


博客地址：[ **https://llss6887.github.io**]( https://llss6887.github.io " https://llss6887.github.io")

