---
title: SpringBoot 集成mybatis
date: 2017-05-21 20:36:47
urlname: SpringBoot-mybatis
tags:
  - SpringBoot
  - mybatis
categories: SpringBoot
---

## 添加环境依赖
### 添加mybatis依赖：
```xml
<dependency>
	<groupId>org.mybatis.spring.boot</groupId>
	<artifactId>mybatis-spring-boot-starter</artifactId>
	<version>1.1.1</version>
</dependency>
```
### 添加jdbc
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```
<!-- more -->
### 添加mysql依赖
```xml
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <version>5.1.35</version>
</dependency>
```
## 创建数据源
### 在src/main/resources下创建application.yml，内容如下：
```yaml
server:
  port: 8080
spring:
  datasource:
      driver-class-name: com.mysql.jdbc.Driver
      url: jdbc:mysql://182.61.34.13:3306/springboot?useUnicode=true&characterEncoding=UTF-8&allowMultiQueries=true
      username: root
      password: root
```
### 初始化数据库

```sql
CREATE DATABASE /*!32312 IF NOT EXISTS*/`springboot` /*!40100 DEFAULT CHARACTER SET utf8 */;
 
USE `springboot_db`;
 
DROP TABLE IF EXISTS `t_user`;
 
CREATE TABLE `t_user` (
  `id` int(20) auto increment,
  `real_name` VARCHAR(32) NOT NULL, 
  PRIMARY KEY (`id`)
) 
```

## 整合Mybatis
### 方式一: 注解
#### 创建实体对象
```java
@Getter
@Setter
public class User {

    private int id;
    private String realName;
}
```
#### 创建dao
```java
@Mapper
public interface UserDao {

    @Insert("insert into t_user (real_name) values(#{realName})")
    public void addUser(User user);

    @Select("select * from t_user where id = #{id}")
    public User selectUser(@Param("id") Integer id);

    @Update("update t_user set real_name = #{real_name} where id = #{id}")
    public void updateUser(@Param("real_name") String name, @Param("id") Integer id);

    @Delete("delete from t_user where id = #{id}")
    public void deleteIUser(@Param("id") Integer id);
}
```
#### 创建service
```java
@Service
public class UserService {

    @Autowired
    private UserDao userDao;

    public void addUser(User user){
        userDao.addUser(user);
    }

    public User selectUser(Integer id){
        return userDao.selectUser(id);
    }

    public void updateUser(String name, Integer id){
        userDao.updateUser(name, id);
    }

    public void deleteIUser(Integer id){
        userDao.deleteIUser(id);
    }

}
```
#### 创建controller
```java
@RestController
@RequestMapping("/spring_mybatis")
public class SpringBootMybatis {

    @Autowired
    private UserService userService;


    @GetMapping("/addUser/{name}")
    public void addUser(@PathVariable("name") String name){
        User user = new User();
        user.setRealName(name);
        userService.addUser(user);
    }

    @GetMapping("/findUser/{id}")
    public User findUser(@PathVariable("id") String id){
        return userService.selectUser(Integer.parseInt(id));
    }

    @GetMapping("/updateUser/{id}/{name}")
    public void updateUser(@PathVariable("id") String id, @PathVariable("name") String name){
        userService.updateUser(name, Integer.parseInt(id));
    }

    @GetMapping("/deleteUser/{id}")
    public void deleteUser(@PathVariable("id") String id){
        userService.deleteIUser(Integer.parseInt(id));
    }
}
```
#### 启动类增加MapperScan

```java
@SpringBootApplication
@MapperScan(basePackages = {"com.llss.dao"})
public class AppServerStart {
    public static void main(String[] args) {
        SpringApplication.run(AppServerStart.class, args);
    }
}
```

#### 启动项目
此时，一个RESTful 风格的数据库增删改查就完成了。

### xml配置方式
#### 实体类，service类和controller类不变，增加路径及实体类配置：
```yaml
mybatis:
  type-aliases-package: com.lls.domain
  mapper-locations: classpath:/mybatis/*Mapper.xml

```
#### 在resources目录下创建mybatis目录 并创建UserDaoMapper.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.llss.dao.UserDao">
    <insert id="addUser" parameterType="com.llss.domain.User">
        insert into t_user
        (real_name) values(#{realName})
    </insert>
</mapper>
```
#### 去掉Dao的注释
```java
@Mapper
public interface UserDao {
    public void addUser(User user);
}
```
#### 启动项目
访问 localhost:8080/spring-mybatis/addUser/lisi

## 总结
在实际生产中遇到较为复杂的架构，一般选用成熟的插件，比如com.github.pagehelper分页插件.


源码地址：[**戳这里**](https://github.com/llss6887/springboot/tree/master/springboot-mybatis "**戳这里**")













































