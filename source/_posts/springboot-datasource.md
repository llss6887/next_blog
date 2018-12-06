---
title: SpringBoot 多数据源
date: 2017-05-22 20:36:47
urlname: springboot-datasource
tags:
  - datasource
  - springboot
categories: SpringBoot
---
## **概述**
在开发过程中可能需要用到多个数据源，比如一个项目（MySQL）就是和（SQL Server）混合使用，就需要使用多数据源；

## 开始

### 创建多数据源配置

```yaml
master:
  datasource:
    url: jdbc:mysql://182.61.34.12:3306/test?useUnicode=true&characterEncoding=utf8
    username: root
    password: root
    driverClassName: com.mysql.jdbc.Driver


slave:
  datasource:
    url: jdbc:mysql://182.61.34.12:3306/test?useUnicode=true&characterEncoding=utf8
    username: root
    password: root
    driverClassName: com.mysql.jdbc.Driver
```
<!-- more -->

### 创建数据源配置类

#### 数据源一
```java
@Configuration
@MapperScan(basePackages = {"com.llss.master.dao"}, sqlSessionFactoryRef = "masterSqlSessionFactory")
public class MasterDataSouceConf {

    @Value("${master.datasource.url}")
    private String url;

    @Value("${master.datasource.username}")
    private String user;

    @Value("${master.datasource.password}")
    private String password;

    @Value("${master.datasource.driverClassName}")
    private String driverClass;

    @Bean(name = "masterDatasource")
    @Primary
    public DataSource masterDataSource(){
        DruidDataSource ds = new DruidDataSource();
        ds.setDriverClassName(driverClass);
        ds.setUrl(url);
        ds.setUsername(user);
        ds.setPassword(password);
        return ds;
    }

    @Bean(name = "masterSqlSessionFactory")
    @Primary
    public SqlSessionFactory masterSqlSessionFactory(@Qualifier("masterDatasource") DataSource dataSource) throws Exception {
        SqlSessionFactoryBean ssfb = new SqlSessionFactoryBean();
        ssfb.setDataSource(dataSource);
        return ssfb.getObject();
    }

    @Bean(name = "masterDataSourceTransactionManager")
    @Primary
    public DataSourceTransactionManager masterDataSourceTransactionManager(){
        DataSourceTransactionManager dstm = new DataSourceTransactionManager(masterDataSource());
        return dstm;
    }
}
```
#### 数据源二
```java
@Configuration
@MapperScan(basePackages = {"com.llss.slave.dao"}, sqlSessionFactoryRef = "slaveSqlSessionFactory")
public class SalveDataSouceConf {

    @Value("${slave.datasource.url}")
    private String url;

    @Value("${slave.datasource.username}")
    private String user;

    @Value("${slave.datasource.password}")
    private String password;

    @Value("${slave.datasource.driverClassName}")
    private String driverClass;

    @Bean(name = "slaveDatasource")
    public DataSource slaveDataSource(){
        DruidDataSource ds = new DruidDataSource();
        ds.setDriverClassName(driverClass);
        ds.setUrl(url);
        ds.setUsername(user);
        ds.setPassword(password);
        return ds;
    }

    @Bean(name = "slaveSqlSessionFactory")
    public SqlSessionFactory slaveSqlSessionFactory(@Qualifier("slaveDatasource") DataSource dataSource) throws Exception {
        SqlSessionFactoryBean ssfb = new SqlSessionFactoryBean();
        ssfb.setDataSource(dataSource);
        return ssfb.getObject();
    }

    @Bean(name = "slaveDataSourceTransactionManager")
    public DataSourceTransactionManager slaveDataSourceTransactionManager(){
        DataSourceTransactionManager dstm = new DataSourceTransactionManager(slaveDataSource());
        return dstm;
    }
}
```
### 创建Dao

#### 数据源一
```java
@MapperScan
public interface RoleDao {

    @Select("select * from role where id = #{id}")
    public Role getRole(@Param("id") int id);

    @Insert("insert into role(name) values(#{name})")
    public void addRole(Role role);

    @Delete("delete from role where id = #{id}")
    public void delete(@Param("id") int id);

    @Update("update role set name = #{name} where id = #{id}")
    public void Update(@Param("id") int id, @Param("name") String name);
}
```

#### 数据源二

```java
@MapperScan
public interface UserDao {

    @Select("select * from user where id = #{id}")
    public User getUser(@Param("id") int id);

    @Insert("insert into user(name) values(#{name})")
    public void addUser(User role);

    @Delete("delete from user where id = #{id}")
    public void delete(@Param("id") int id);

    @Update("update user set name = #{name} where id = #{id}")
    public void Update(@Param("id") int id, @Param("name") String name);
}
```

### 实体类

```java
public class User {
    private int id;
    private String name;
}
public class Role {
    private int id;
    private String name;
}
```
### 启动类及测试
```java
@SpringBootApplication
@RestController
public class DataSourceServer {

    @Autowired
    private UserDao userDao;

    @Autowired
    private RoleDao roleDao;

    public static void main(String[] args) {
        SpringApplication.run(DataSourceServer.class, args);
    }

    @GetMapping("/master")
    public User getUser(){
        return userDao.getUser(2);
    }

    @GetMapping("/slave")
    public Role getRole(){
        return roleDao.getRole(2);
    }

    @GetMapping("/addMaster/{name}")
    public void addMaster(@PathVariable("name") String name){
        User u = new User();
        u.setName(name);
        userDao.addUser(u);
    }

    @GetMapping("/addSlave/{name}")
    public void addSlave(@PathVariable("name") String name){
        Role r = new Role();
        r.setName(name);
        roleDao.addRole(r);
    }
}
```
## 总结
上述只是整合了注解版本的mybatis多数据源，如果使用xml版本的，只需要设置SqlSessionFactory的MapperLocations和setTypeAliasesPackage即可。

源码地址：[**戳这里**](https://github.com/llss6887/springboot/tree/master/moredatasource "**戳这里**")













































