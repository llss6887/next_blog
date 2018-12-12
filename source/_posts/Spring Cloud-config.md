---
title: Spring Cloud-config
date: 2017-08-07 23:59:47
urlname: SpringCloud-config
tags:
  - SpringCloud
categories: SpringCloud
---

## Spring Cloud Config

Spring Cloud Config为分布式系统中的外部配置提供服务器和客户端支持。使用Config Server，您可以在所有环境中管理应用程序的外部属性。客户端和服务器上的概念映射与Spring Environment和PropertySource抽象相同，因此它们与Spring应用程序非常契合，但可以与任何以任何语言运行的应用程序一起使用。随着应用程序通过从开发人员到测试和生产的部署流程，您可以管理这些环境之间的配置，并确定应用程序具有迁移时需要运行的一切。服务器存储后端的默认实现使用git，因此它轻松支持标签版本的配置环境，以及可以访问用于管理内容的各种工具。可以轻松添加替代实现，并使用Spring配置将其插入。

就是分为客户端和服务端，服务端将配置文件发布成REST接口，客户端负责从服务端REST接口中获取配置。
<!--more-->

## 使用

### 使用git

#### server端

首先在github上创建一个仓库，叫springcloud-config，在仓库里面创建一个config-repo文件夹，里面有两个配置文件，

    repo-config-dev.properties	
    repo-config-pro.properties


##### 添加依赖
```java
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

##### 配置文件
```xml
server:
  port: 8080
spring:
  application:
    name: springcloud-config-server
  cloud:
    config:
      enabled: true
      server:
        git:
          uri: https://github.com/llss6887/springcloud-config #刚才新建的仓库地址
          search-paths: config-repo  #搜索的目录
          username: 				 #git账号
          password: 				 #git密码
```
Spring Cloud Config 提供本地的配置 `spring.profiles.active=native` 会在src/main/resource中搜索，也可以spring.cloud.config.server.native.searchLocations=file:E:/properties/属性来指定配置文件的位置。

##### 启动类
增加@EnableConfigServer注解，开启服务。
```java
@SpringBootApplication
@EnableConfigServer
public class ConfigApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigApplication.class, args);
    }
}
```

##### 启动测试
HTTP服务具有以下格式的资源：

    /{application}/{profile}[/{label}]
    /{application}-{profile}.yml
    /{label}/{application}-{profile}.yml
    /{application}-{profile}.properties
    /{label}/{application}-{profile}.properties

其中“应用程序”作为SpringApplication中的spring.config.name注入（即常规Spring Boot应用程序中通常为“应用程序”），“配置文件”是活动配置文件（或逗号分隔列表）的属性），“label”是可选的git标签（默认为“master”）。
我们的配置文件叫做 `repo-config-pro.properties` 按照上面的格式，我们的访问地址应该是：http://localhost:8080/repo-config/pro
访问结果如下：
![](1.png)
server端的就完成了。

#### client端的使用
##### 依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

##### 启动类
```java
@SpringBootApplication
public class ConfigClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigClientApplication.class, args);
    }
}
```

##### 服务访问
```java
@RestController
public class ConfigClientController {

    @Value("${name}")
    private String name;

    @RequestMapping("/config_test")
    public String getName(){
        return name;
    }
}
```
##### yml配置文件
yml配置文件，需要配置两个，application.yml和bootstarp.yml
application.yml的配置：
```xml
server:
  port: 8081
```
bootstarp.yml的配置：
**下面的配置必须写到 bootstrap中，因为必须保证config的配置优先于application的配置。**
```xml
spring:
  application:
    name: spring-cloud-client
  cloud:
    config:
      name: repo-config #server端访问的名称
      uri: http://localhost:8080/  #server端的地址
      label: master
      profile: dev  #指定文件
```
spring.application.name：对应{application}部分
spring.cloud.config.profile：对应{profile}部分
spring.cloud.config.label：对应git的分支。如果配置中心使用的是本地存储，则该参数无用
spring.cloud.config.uri：配置中心的具体地址
spring.cloud.config.discovery.service-id：指定配置中心的service-id，便于扩展为高可用配置集群。

##### 启动测试

项目启动后，访问localhost:8081/config_test
结果如下：
![](2.png)
获取到git上的配置属性。

client端只有在第一次启动的时候会获取配置文件的值，当我们手工更改配置文件后，client获取的还是旧的参数。

### svn使用
和git的配置差不多，只是执行active为subversion，启动类和git的没有差别，client端也和git的没有任何差别。
```xml
server:
  port: 8080
spring:
  application:
    name: springcloud-config-svn
  cloud:
    config:
      enabled: true
      server:
        svn:
          uri:  http://192.168.25.7/svn/repo/config-repo
          username:
          password:
          default-label: trunk
      profiles:
        active: subversion
```

## 配置中心的服务化和高可用
高可用就是将配置的服务注册到eureka

### 服务端
#### pom依赖
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka</artifactId>
    </dependency>
</dependencies>
```

#### yml配置文件
```xml
server:
  port: 8081
spring:
  application:
    name: springcloud-config-server
  cloud:
    config:
      enabled: true
      server:
        git:
          uri: https://github.com/llss6887/springcloud-config
          search-paths: config-repo
          username:
          password:
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8001/eureka/

```

#### 启动类
```java
@EnableConfigServer
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaConfigApp {
    public static void main(String[] args) {
        SpringApplication.run(EurekaConfigApp.class, args);
    }
}
```
启动项目，会发现已经注册到服务：
![](20.png)
访问localhost:8081/repo-config/dev,返回结果为:`this is dev update` 服务端搭建完成。

### 客户端

客户端和之前的客户端几乎没啥区别，

#### POM 文件
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka</artifactId>
    </dependency>
</dependencies>
```

#### yml配置文件
还是需要application.yml和bootstrap.yml，下面内容需要写到bootstrap.yml中。
```xml
spring:
  application:
    name: springcloud-config-eureka-client
  cloud:
    config:
      name: repo-config
      profile: dev
      label: master
      discovery:
        service-id: springcloud-config-server #配置service端的name，
        enabled: true						  #开启服务
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8001/eureka/  #配置中心地址，多个逗号分隔
```

#### 启动类
```java
@EnableDiscoveryClient
@SpringBootApplication
@RestController
public class ClientApplication {

    @Value("${name}")
    private String name;

    public static void main(String[] args) {
        SpringApplication.run(ClientApplication.class, args);
    }

    @RequestMapping("/eureka_config")
    public String getConfig(){
        return name;
    }
}
```

#### 测试
输入localhost:8082/eureka_config, 返回值为：`this is dev update`，到此客户端也搭建完毕。


将多个配置中心的服务端注册到eureka，即使有一个宕机，客户端调用的时候依旧是可用的。