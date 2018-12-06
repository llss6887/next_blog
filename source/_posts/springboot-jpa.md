---
title: SpringBoot 集成jpa
date: 2017-05-21 20:36:47
urlname: SpringBoot-jpa
tags:
  - jpa
  - SpringBoot
categories: SpringBoot
---
## 添加依赖
### 添加jar包
```xml
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
```
<!-- more -->
### 添加配置文件

```yaml
server:
  port: 8080
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://182.61.34.13:3306/jpa_test
    username: root
    password: root
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
  jooq:
    sql-dialect: org.hibernate.dialect.MySQL5InnoDBDialect

```
### 创建 表
```sql
create table `t_user` (
	`id` int (20),
	`real_name` varchar (96)
); 
```

### 创建dao
```java
public interface UserDao extends JpaRepository<User, Integer> {

    @Override
    void delete(Integer id);

    @Override
    User findOne(Integer id);

    @Override
    User save(User u);
}
```
继承JpaRepository，JpaRepository定义了一些增删改查的方法。
根据方法名来自动生成SQL，主要的语法是findXXBy,readAXXBy,queryXXBy,countXXBy, getXXBy后面跟属性名称：
`User findByUserName(String userName);`
也使用一些加一些关键字And、 Or
`User findByUserNameOrEmail(String username, String email);
`修改、删除、统计也是类似语法

`Long deleteById(Long id);
Long countByUserName(String userName)`
基本上SQL体系中的关键词都可以使用，例如：LIKE、 IgnoreCase、 OrderBy。
`
List<User> findByEmailLike(String email);
User findByUserNameIgnoreCase(String userName);
List<User> findByUserNameOrderByEmailDesc(String email);`

### 创建controller

```java
@RestController
public class JpaController {

    @Autowired
    private UserDao userDao;

    @GetMapping("/save/{name}")
    public void save(@PathVariable String name){
        User u = new User();
        u.setReal_name(name);
        userDao.save(u);
    }
    @GetMapping("/find/{id}")
    public User findonne(@PathVariable int id){
        return userDao.findOne(id);
    }
}
```
### 启动项目
访问 localhost:8080/save/lisi  查看数据库：
![](ec3a6ec54aec48ef92646f390024a874.png)
localhost:8080/save/1  查看页面：
![](2eacf09c6bed4545a13abfb380082c10.png)

此时，jpa搭建成功。

## 自定义SQL查询

Spring data 支持大部分的根据名称定义的查询，对于自定义查询，spring data 也是支持的，在方法的上面增加`@Query`，修改删除，需要增加`@Modifying`,也可以根据需要增加`@Transactional`

```java
@Query("select u from User u where u.real_name = ?1")
public User queryByReal_name(String name);

@Modifying
@Query("update User u set u.real_name = ?1 where u.id = ?2")
public User updateById(String name, int id);
```
## 分页查询
编写 带pageable参数的查询，一般放到最后一个参数
```java
@Override
Page<User> findAll(Pageable pageable);
```
编写controller
```java
@GetMapping("/findAll")
public Page<User> findAll(@RequestParam() int page, int size){
	//排序方式  和字段  可以多個字段排序
	Sort sort = new Sort(Sort.Direction.DESC, "id");
	Pageable p = (Pageable) new PageRequest(page, size, sort);
	Page<User> all = userDao.findAll(p);
	return all;
}
```

## 多表查询
和hibernate的hql多表查询差别不大，有两种方式，一种是hibernate的级联查询，一种是将查询结果映射到一个结果类中。

首先需要定义一个结果集的接口类。

```java
public interface HotelSummary {

	City getCity();

	String getName();

	Double getAverageRating();

	default Integer getAverageRatingRounded() {
		return getAverageRating() == null ? null : (int) Math.round(getAverageRating());
	}

}
```
查询的方法返回类型设置为新创建的接口

```java
@Query("select h.city as city, h.name as name, avg(r.rating) as averageRating "
		- "from Hotel h left outer join h.reviews r where h.city = ?1 group by h")
Page<HotelSummary> findByCity(City city, Pageable pageable);
```

```java
@Query("select h.name as name, avg(r.rating) as averageRating "
		- "from Hotel h left outer join h.reviews r  group by h")
Page<HotelSummary> findByCity(Pageable pageable);

```
使用

```java
Page<HotelSummary> hotels = this.hotelRepository.findByCity(new PageRequest(0, 10, Direction.ASC, "name"));
for(HotelSummary summay:hotels){
		System.out.println("Name" +summay.getName());
	}
```
在运行中 spring会将产生一个代理类来接受结果，用getxxx的方式来获取。

源码地址：[**戳这里**](https://github.com/llss6887/springboot/tree/master/springbootjpa "**戳这里**")













































