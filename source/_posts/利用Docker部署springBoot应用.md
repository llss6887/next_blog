---
title: 利用Docker部署springBoot应用
date: 2018-09-22 20:36:47
urlname: docker-springboot
tags:
  - Docker
  - SpringBoot
  - java
categories: docker
---

![](82b6146d0e5f4a91886fa68384fe8e26.png)

##  **简介**
Docker是一个开源的引擎，可以轻松的为任何应用创建一个轻量级的、可移植的、自给自足的容器。开发者在笔记本上编译测试通过的容器可以批量地在生产环境中部署，包括VMs（虚拟机）、bare metal、OpenStack 集群和其他的基础应用平台。
<!--more-->
## **Docker通常用于如下场景：**

web应用的自动化打包和发布；
自动化测试和持续集成、发布；
在服务型环境中部署和调整数据库或其他的后台应用；
从头编译或者扩展现有的OpenShift或Cloud Foundry平台来搭建自己的PaaS环境。


## **Docker 的优点**

### 1、简化程序：
Docker 让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的 Linux 机器上，便可以实现虚拟化。Docker改变了虚拟化的方式，使开发者可以直接将自己的成果放入Docker中进行管理。方便快捷已经是 Docker的最大优势，过去需要用数天乃至数周的	任务，在Docker容器的处理下，只需要数秒就能完成。


### 2、避免选择恐惧症：
如果你有选择恐惧症，还是资深患者。Docker 帮你	打包你的纠结！比如 Docker 镜像；Docker 镜像中包含了运行环境和配置，所以 Docker 可以简化部署多种应用实例工作。比如 Web 应用、后台应用、数据库应用、大数据应用比如 Hadoop 集群、消息队列等等都可以打包成一个镜像部署。

### 3、节省开支：
一方面，云计算时代到来，使开发者不必为了追求效果而配置高额的硬件，Docker 改变了高性能必然高价格的思维定势。Docker 与云的结合，让云空间得到更充分的利用。不仅解决了硬件管理的问题，也改变了虚拟化的方式。

## **Docker 架构**
Docker 使用客户端-服务器 (C/S) 架构模式，使用远程API来管理和创建Docker容器。
Docker 容器通过 Docker 镜像来创建。
容器与镜像的关系类似于面向对象编程中的对象与类。

|  Docker |  	面向对象  |
| ------------ | ------------ |
| 容器  | 	对象  |
| 镜像  |   	类|

![](55930b576ef540fdaed3e883383196a3.png)

## **CentOS Docker 安装**
Docker支持以下的CentOS版本：
CentOS 7 (64-bit)
CentOS 6.5 (64-bit) 或更高的版本

### **前提条件**
1. 目前，CentOS 仅发行版本中的内核支持 Docker。
2. Docker 运行在 CentOS 7 上，要求系统为64位、系统内核版本为 3.10 以上。
3. Docker 运行在 CentOS-6.5 或更高的版本的 CentOS 上，要求系统为64位、系统内核版本为 2.6.32-431 或者更高版本。

### **安装 Docker**

安装一些必要的系统工具：

`sudo yum install -y yum-utils device-mapper-persistent-data lvm2`

安装Docker：
`yum install docker-io`

查看Docker版本：
 `docker -v`

启动docker：
`systemctl start docker`

镜像加速：
由于docker默认使用的国外的镜像，所以网速感人，我们替换为国内的镜像，国内镜像，一般有网易，阿里云等等，这里使用网易的镜像。
在   /etc/docker/daemon.json添加一下内容即可：

`{
  "registry-mirrors": ["http://hub-mirror.c.163.com"]
}`

查找镜像：
这里使用hello-world的镜像
`docker search hello-world`

如果要查找nginx的镜像  将hello-world替换成nginx即可。
![](0ef85b2fc4e342868ce0cee3c4a2534b.png)

拉取镜像：
`docker pull docker.io/hello-world`

之后查看镜像是否拉取成功：
`docker images`
![](48ce68768d094135b4ed332b082c6fc6.png)

运行刚才拉取的hello：
`docker run hello-world`![](eadfc943540a4040ac0add5f269c61e0.png)


到这里 一个完整的Docker就搭建完成了，如果想更深入了解 Docker的其他东西， **[点击这里](https://www.docker.com/ "点击这里")**  自行学习。



### **下面来看Docker如何部署一个springboot应用：**
**具体步骤如下：**
1.创建一个springboot应用
2.创建一个java的docker容器
3.将springboot应用和docker容器进行关联
4.启动这个docker容器

springboot应用代码：

```java
@Controller
@RequestMapping("/demo")
public class FirstDocker {

    @RequestMapping("/index")
    public String firstPage(HttpServletRequest request){
        return "demo/index";
    }  
}
```
```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
public class AppServer {

    public static void main(String[] args) {
        SpringApplication.run(AppServer.class, args);
    }

}
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<h1>this is my firstDocker</h1>
</body>
</html>
```
```xml
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.1.RELEASE</version>
        <relativePath/>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.2</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
    </dependencies>
	<build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <!-- 1、设置jar的入口类 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>2.6</version>
                <configuration>
                    <archive>
                        <manifest>
                            <addClasspath>true</addClasspath>
                            <classpathPrefix>lib/</classpathPrefix>
                            <mainClass>com.kjlink.AppServer</mainClass>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>


            <!--2、把附属的jar打到jar内部的lib目录中 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-dependency-plugin</artifactId>
                <version>2.10</version>
                <executions>
                    <execution>
                        <id>copy-dependencies</id>
                        <phase>package</phase>
                        <goals>
                            <goal>copy-dependencies</goal>
                        </goals>
                        <configuration>
                            <outputDirectory>${project.build.directory}/lib</outputDirectory>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```
访问 localhost:8080/demo/index
![](a6474bd045454144b799568098555fe3.png)

在服务器创建docker文件夹
通过maven打包成jar包并上传到服务器
在docker文件夹下创建Dockerfile文件
Docker文件内容如下:

    FROM java:8   #带边拉取java的镜像 必须写
    MAINTAINER "liss"<168337573@qq.com> #作者名称 必须写
    WORKDIR  /tmp  #工作的目录
    ADD ./app.jar app.jar #将本地的jar包添加在docker容器中
    EXPOSE 8080  #容器启动的端口
    CMD java -jar app.jar  #启动jar的命令
	
之后  执行一下命令：

`docker build -t firstdocker:v1 .`

- build：代表构建docker容器
- -t:指定容器的名称
- .：说明Dockerfile文件在当前目录

![](c9b7eff9a4d84444bf49577b2882cfd5.png)
说明这个镜像已经构建完毕。下面我们启动。

`docker run -d -p 8080:8080 firstdocker:v1`

-d:表示后台运行
-p:表示宿主机和容器的端口映射，第一个8080为宿主机对外的端口，第二个8080为容器启动后的端口

`docker ps`
来查看是否启动成功。

![](9777d8876a544b3b960e4a69eb8bb147.png)

我们访问一下是否成功：
![](35463eb5370e43c4bde011769577e3c4.png)

成功访问！  我们的第一个docker部署springboot就完成了。

停止容器
`docker stop aff048d793b7`

aff048d793b7：为 容器的启动id



上面的方式虽然可以部署成功，但不是最佳的方式，因为我们每次需要部署的时候，都要构建，很麻烦，因此docker提供了挂载的功能，我们只要构建好容器，将jar包的路径，日志，等等通过宿主机挂载到docker容器，这样，我们每次部署的时候，只需要替换宿主机的jar包，和重新启动我们的docker容器就可以了。

具体怎么做呢？很简单，修改我们的Dockerfile文件

    FROM java:8
    MAINTAINER "liss"<168337573@qq.com>
    #ADD ./app.jar app.jar
    RUN mkdir -p /home/apps
    EXPOSE 8082
    CMD java -jar /home/apps/app.jar
	
将Dockerfile修改成以上，我们在进行构建容器。
`docker build -t firstdocker:v2 .`

构建完毕之后，执行命令：
`docker run -d -v /home/docker/:/home/apps/ -p 8081:8080 firstdocker:v2 `

-v:表示挂载目录，第一个为宿主机的目录，第二个为容器里面的目录
![](e0a06dd557ba4fe49d36b3895d052bee.png)

也是没有任何问题的，之后我们每次更新app.jar的时候 ，只需要将/home/docker目录下的app.jar替换掉重新启动容器就可以了，不用每次都构建了。


## **结语**
docker真的很强大，这篇文章只是介绍的docker的基本使用，docker还提供的任务编排 Docker Compose,集群管理工具Docker Swarm,快速在多种平台上构建环境的Docker Machine,学无止境 每一天都是崭新的最后一天， 努力、加油 。



源码地址： [**戳这里**](https://github.com/llss6887/docker.git "**戳这里**")


博客地址：[ **https://llss6887.github.io**]( https://llss6887.github.io " https://llss6887.github.io")







