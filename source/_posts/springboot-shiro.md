---
title: SpringBoot 集成shiro
date: 2017-07-24 20:36:47
urlname: springboot-shiro
tags:
  - shiro
  - SpringBoot
categories: SpringBoot
---

## shiro简介
Apache Shiro是Java的一个安全框架。功能强大，使用简单的Java安全框架，它为开发人员提供一个直观而全面的认证，授权，加密及会话管理的解决方案。 

实际上，Shiro的主要功能是管理应用程序中与安全相关的全部，同时尽可能支持多种实现方法。Shiro是建立在完善的接口驱动设计和面向对象原则之上的，支持各种自定义行为。Shiro提供的默认实现，使其能完成与其他安全框架同样的功能，这不也是我们一直努力想要得到的吗！

Apache Shiro相当简单，对比Spring Security，可能没有Spring Security做的功能强大，但是在实际工作时可能并不需要那么复杂的东西，所以使用小而简单的Shiro就足够了。对于它俩到底哪个好，这个不必纠结，能更简单的解决项目问题就好了。
<!--more-->
Shiro可以非常容易的开发出足够好的应用，其不仅可以用在JavaSE环境，也可以用在JavaEE环境。Shiro可以帮助我们完成：认证、授权、加密、会话管理、与Web集成、缓存等。这不就是我们想要的嘛，而且Shiro的API也是非常简单；其基本功能点如下图所示：
![](shiro.png)

Authentication：身份认证/登录，验证用户是不是拥有相应的身份；

Authorization：授权，即权限验证，验证某个已认证的用户是否拥有某个权限；即判断用户是否能做事情，常见的如：验证某个用户是否拥有某个角色。或者细粒度的验证某个用户对某个资源是否具有某个权限；

Session Manager：会话管理，即用户登录后就是一次会话，在没有退出之前，它的所有信息都在会话中；会话可以是普通JavaSE环境的，也可以是如Web环境的；

Cryptography：加密，保护数据的安全性，如密码加密存储到数据库，而不是明文存储；

Web Support：Web支持，可以非常容易的集成到Web环境；

Caching：缓存，比如用户登录后，其用户信息、拥有的角色/权限不必每次去查，这样可以提高效率；

Concurrency：shiro支持多线程应用的并发验证，即如在一个线程中开启另一个线程，能把权限自动传播过去；

Testing：提供测试支持；

Run As：允许一个用户假装为另一个用户（如果他们允许）的身份进行访问；

Remember Me：记住我，这个是非常常见的功能，即一次登录后，下次再来的话不用登录了。


## shiro是如何工作的

![](shiro_nei.png)

应用代码通过Subject来进行认证和授权，而Subject又委托给SecurityManager；

我们需要给Shiro的SecurityManager注入Realm，从而让SecurityManager能得到合法的用户及其权限进行判断。

## spring security 与apache shiro 差别

shiro配置更加容易理解，容易上手；security配置相对比较难懂。

在spring的环境下，security整合性更好。Shiro对很多其他的框架兼容性更好，号称是无缝集成。
shiro 不仅仅可以使用在web中，它可以工作在任何应用环境中。

在集群会话时Shiro最重要的一个好处或许就是它的会话是独立于容器的。

Shiro提供的密码加密使用起来非常方便。



## 与SpringBoot的集成

### 项目结构
![](1.png)

### maven依赖

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

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>

        <dependency>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-spring</artifactId>
            <version>1.3.2</version>
        </dependency>

        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.1.4</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>

    </dependencies>
```

### 配置文件

```YAML
server:
  port: 8080
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    username: root
    password: root
    url: jdbc:mysql://182.61.34.12:3306/shiro?useUnicode=true&amp;characterEncoding=UTF-8
    type: com.alibaba.druid.pool.DruidDataSource
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: update
```
### realm的认证和授权

```java
public class MyShiroRealm extends AuthorizingRealm {


    @Autowired
    private UserService userService;
    /**
     * 授权-------------------判断是否有权限     PrincipalCollection参数保存了当前的用户对象
     * @param principalCollection
     * @return
     */

    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        String userName = (String)principalCollection.getPrimaryPrincipal();
        User user = userService.findUserByName(userName);
        SimpleAuthorizationInfo simpleAuthorizationInfo = new SimpleAuthorizationInfo();
        user.getRoles().forEach(role -> {
            simpleAuthorizationInfo.addRole(role.getName());
            role.getPers().forEach(permission -> {
                simpleAuthorizationInfo.addStringPermission(permission.getName());
            });
        });
        return simpleAuthorizationInfo;
    }

    /**
     * 用户认证
     * 认证-----------登录-----------根据用户名，查到相应的用户，这样就可以得到在数据*	库中保存密码
	 * 下一步交给<property name="credentialsMatcher" ref="passwordMatcher"/>所对应的加密码的类进行比较
	 * 程序会将登录时用户输入的明文通过加密算法加密，再与数据库中密码进行比较，这个比较工作就由CustomCredentialsMatcher.java类进行处理
	 * 
     * @param authenticationToken
     * @return
     * @throws AuthenticationException
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
        if(authenticationToken.getPrincipal() == null){
            return null;
        }
		//1.得参数token转化为它的子类，再得到用户名
        String username = authenticationToken.getPrincipal().toString();
		//2.调用业务逻辑，根据用户名查询一个用户对象
        User user = userService.findUserByName(username);
		//3.判断是否成功
        if(user != null){
            SimpleAuthenticationInfo simpleAuthenticationInfo = new SimpleAuthenticationInfo(username, user.getPassword(), user.getName());
            return simpleAuthenticationInfo;
        }
        return null;
    }
}
```

### 自定义MD5加密
```java
/**
 * 自定义密码加密
 */
public class Encrypt {
    public static String md5(String password, String salt){
        return new Md5Hash(password,salt,2).toString();
    }

    public static void main(String[] args) {
        String pass = md5("123456", "wangsu");
        System.out.printf(pass);
    }
}
```
### 对比认证

```java

/**
 * 加密后对比是否一致
 */
public class CustomCredentialsMatcher extends SimpleCredentialsMatcher {
    @Override
    public boolean doCredentialsMatch(AuthenticationToken token, AuthenticationInfo info) {
        UsernamePasswordToken usernamePasswordToken = (UsernamePasswordToken) token;
        Object tokentials = Encrypt.md5(String.valueOf(usernamePasswordToken.getPassword()), usernamePasswordToken.getUsername());
        Object credentials = getCredentials(info);
        return equals(tokentials, credentials);
    }
}
```
### 安全管理器以及过滤器

```java
@Configuration
public class ShiroConfiguration {

    /**
     * 配置权限管理的认证和授权
     * @return
     */
    @Bean
    public MyShiroRealm getRealm(){
        MyShiroRealm realm = new MyShiroRealm();
        realm.setCredentialsMatcher(customCredentialsMatcher());
        return realm;
    }

    /**
     * 认证对比
     * @return
     */
    @Bean
    public CustomCredentialsMatcher customCredentialsMatcher(){
        return new CustomCredentialsMatcher();
    }

    /**
     * 安全管理器
     * @return
     */
    @Bean
    public DefaultSecurityManager securityManager(){
        DefaultSecurityManager dsn = new DefaultWebSecurityManager();
        dsn.setRealm(getRealm());
        return dsn;
    }

    /**
     * 过滤器
     * @param securityManager
     * @return
     */
    @Bean
    public ShiroFilterFactoryBean shiroFilterFactoryBean(DefaultSecurityManager securityManager){
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
		//设置管理器
        shiroFilterFactoryBean.setSecurityManager(securityManager);
        Map<String, String> map = new HashMap<>();
        map.put("/logout", "logout");
		//对所有人认证
        map.put("/**", "authc");
		//登录页面
        shiroFilterFactoryBean.setLoginUrl("/login");
		//成功后跳转页面
        shiroFilterFactoryBean.setSuccessUrl("/index");
        shiroFilterFactoryBean.setUnauthorizedUrl("/error");
        shiroFilterFactoryBean.setFilterChainDefinitionMap(map);
        return shiroFilterFactoryBean;
    }
	//注解需要的
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(DefaultSecurityManager securityManager){
        AuthorizationAttributeSourceAdvisor aas = new AuthorizationAttributeSourceAdvisor();
        aas.setSecurityManager(securityManager);
        return aas;
    }

}
```
### 实体类

```java
@Entity(name = "t_user")
@Getter
@Setter
public class User implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private int id;

    @Column(name = "name", unique = true)
    private String name;

    @Column(name = "password")
    private String password;

    @ManyToMany(mappedBy = "users", fetch = FetchType.EAGER)
    private List<Role> roles = new ArrayList<>();

}

@Entity(name = "t_role")
@Getter
@Setter
public class Role implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private int id;

    @Column(name = "name", unique = true)
    private String name;

    @ManyToMany(fetch = FetchType.EAGER, cascade = CascadeType.PERSIST)
    @JoinTable(name = "t_user_mid_role",
    joinColumns = {@JoinColumn(name = "r_id", referencedColumnName = "id")},
    inverseJoinColumns = {@JoinColumn(name = "u_id", referencedColumnName = "id")})
    private Set<User> users = new HashSet<>();


    @ManyToMany(mappedBy = "roles", fetch = FetchType.EAGER)
    private Set<Permission> pers = new HashSet<>();
}

@Entity(name = "t_permission")
@Getter
@Setter
public class Permission {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private int id;

    @Column(name = "name", unique = true)
    private String name;

    @ManyToMany(fetch = FetchType.EAGER, cascade = CascadeType.PERSIST)
    @JoinTable(name = "t_role_mid_per",
            joinColumns = {@JoinColumn(name = "p_id", referencedColumnName = "id")},
            inverseJoinColumns = {@JoinColumn(name = "r_id", referencedColumnName = "id")})
    private Set<Role> roles = new HashSet<>();
}
```
### service类

```java
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserDao userDao;

    public User findUserByName(String name){
        return userDao.findByName(name);
    }
}

```

### 模拟登录的controller

```java
@RestController
public class LoginController {

    @GetMapping("/login")
    public String getUser(User user){
        UsernamePasswordToken upt = new UsernamePasswordToken();
        upt.setUsername(user.getName());
        upt.setPassword(user.getPassword().toCharArray());
        Subject subject = SecurityUtils.getSubject();
        String reValue = "OK";
        try{
            subject.login(upt);
        }catch (Exception e){
            e.printStackTrace();
            reValue = "fail";
        }
        return reValue;
    }
}

```

### 启动类

```java
@SpringBootApplication
public class ShiroApplication {
    public static void main(String[] args) {
        SpringApplication.run(ShiroApplication.class, args);
    }

    @Bean
    public OpenEntityManagerInViewFilter openEntityManagerInViewFilter(){
        return new OpenEntityManagerInViewFilter();
    }
}

```

### 页面标签

通过shiro:hasPermission 判断是否有权限
``` html
<shiro:hasPermission name="sysadmin">
							<span id="topmenu" onclick="toModule('sysadmin');">系统管理</span>
			    		</shiro:hasPermission
```

## 启动应用 模拟测试

访问http://localhost:8080/login?name=lisi&password=123456
![](ceshi.png)
看到已经认证成功。

## 数据库数据

![](db1.png)
![](db2.png)
![](db3.png)
![](db4.png)
![](db5.png)


shiro的使用基本也就这样子了，在实际中，一般配合页面面前或者注解的权限来开发。相对spring security 比较轻量级，主要做登录认证和一些跳转，也比较容易上手。


源码地址：[**戳这里**](https://github.com/llss6887/springboot/tree/master/springbootshiro)