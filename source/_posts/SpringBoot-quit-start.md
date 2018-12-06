---
title: SpringBoot 快速开始
date: 2017-05-20 20:36:47
urlname: SpringBoot-quit-start
tags:
  - SpringBoot
categories: SpringBoot
---
![](df3a3b954ec04941a42d483aa45c1741.jpg)

## 简介
spring Boot是由Pivotal团队提供的全新框架，其设计目的是用来简化新Spring应用的初始搭建以及开发过程。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。相比之前ssm和ssh的一大堆文件，SpringBoot采用的原则是约定优于配置，对于整合有默认值，如果之前使用过Spring的开发人员来说，可以很快的上手SpringBoot。
<!-- more -->

## 为什么要用SpringBoot
简单的编码
简单的配置
简单的部署
简单的监控

之前搭建一个SpringMvc的web服务我们需要以下：
配置web.xml，加载Spring和SpringMvc
配置数据库，事务
配置日志
......
打包，到服务器
......

**现在** 有了SpringBoot,上面的都不需要了，SpringBoot采用JavaConfig来配置，抛弃了原来的XML，我们搭建一个简单的ssm，只需要添加依赖，SpringBoot就已经帮我们做好了一切。

## 上手体验
1. 打开 [https://start.spring.io/](https://start.spring.io/ "https://start.spring.io/") 生成一个demo
 ![](f1c16c8dcfb042ad81f900523cad92f8.png)
 
2. 将项目解压，用Intellij IDEA 打开下载的demo，结构如下：

 ![](dfb967c67aae407788ea26bff56625c7.png)
 
src/main/java 程序开发以及主程序入口
src/main/resources 配置文件
src/test/java 测试程序

启动DemoApplication main方法，一个java项目就搭建好了。

pom里默认有两个模块：
```xml
<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
```
spring-boot-starter： 核心模块，支持自动配置，日志，和加载配置等等。

spring-boot-starter-test： 测试模块

## 搭建web项目
### pom.xml中引入web模块的支持：
```xml
<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
 </dependency>
```
### 创建contorller：

```java
@RestController
public class FirstController {

    @RequestMapping("/demo")
    public String firstApp(){
        return "hello springboot";
    }
}
```
### 启动主程序
打开浏览器访问http://localhost:8080/hello，就可以看到效果了。

## 总结

利用springboot可以很快速的搭建一个java项目，不用关系框架之间的兼容，版本等问题，我们想使用任何的一个框架，只需要添加配置即可，SpringBoot非常适合微服务。

源码地址： [**戳这里**](https://github.com/llss6887/springboot/tree/master/demo)













































