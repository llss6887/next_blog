---
title: Spring Cloud-zuul
date: 2017-08-10 23:59:47
urlname: SpringCloud-zuul
tags:
  - SpringCloud
categories: SpringCloud
---

## Spring Cloud-zuul
服务网关是微服务架构中一个不可或缺的部分。通过服务网关统一向外系统提供REST API的过程中，除了具备服务路由、均衡负载功能之外，它还具备了权限控制等功能。Spring Cloud Netflix中的Zuul就担任了这样的一个角色，为微服务架构提供了前门保护的作用，同时将权限控制这些较重的非业务逻辑内容迁移到服务路由层面，使得服务集群主体能够具备更高的可复用性和可测试性。

在Spring Cloud体系中， Spring Cloud Zuul就是提供负载均衡、反向代理、权限认证的一个API gateway。是微服务架构的不可或缺的一部分，提供动态路由，监控，弹性，安全等的边缘服务。Zuul是Netflix出品的一个基于JVM路由和服务端的负载均衡器。
<!--more-->
### 简单实用

#### POM依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zuul</artifactId>
</dependency> 
```

#### 启动类
```java
@EnableZuulProxy
@SpringBootApplication
public class ZuulSimpleApplication {
    public static void main(String[] args) {
        SpringApplication.run(ZuulSimpleApplication.class, args);
    }
}
```
加入`@EnableZuulProxy`注解开启网关路由

#### YML配置
```xml
server:
  port: 8080
zuul:
  routes:
    user:
      path: /demo/**  #配置/demo 当访问localhost:8080/demo时重定向到下面的url地址
      url: http://heimamba.com
```

**路径匹配：**
在Zuul中，路由匹配的路径表达式采用了Ant风格定义。Ant风格的路径表达式使用起来非常简单，它一共有下面这三种通配符：

符号|说明|
:--:|:--| 
？|匹配任意的单个字符
*|匹配任意数量的字符
**|匹配任意数量的字符，支持多级目录

#### 启动访问
页面输入 localhost:8080/demo 出现以下：
![](1.png) 
说明配置生效，控制台打印: `Mapped URL path [/demo/**] onto handler of type [class org.springframework.cloud.netflix.zuul.web.ZuulController]`

当访问不存在的地址时会出现：`No route found for uri: /lib/fancybox/source/jquery.fancybox.css`

### 服务化
在微服务架构中，服务名和服务实例的地址关系已经在eureka中存在了，只需要将zuul注册到eureka中就可以发现服务了。无需其他配置，zuul就可以自动寻找eureka上的服务了。
默认的访问地址为：http://zuul的ip:zuul的端口/eureka的服务名称/**

#### 启动eureka
![](2.png)

#### 启动producer注册到eureka
```java
@RestController
public class MessageController {

    @RequestMapping("/message/{name}")
    public String getMessage(@PathVariable String name){
        return name + "  测试！";
    }
}
```
![](3.png)

服务已经注册到eureka上

#### zuul的服务化
**POM：**
增加eureka的依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```
**YML配置：**
增加eureka的地址
```xml
server:
  port: 8002
eureka:
  client:
    service-url:
      defaultZone: http://localhost:9001/eureka/,http://localhost:9002/eureka/,http://localhost:9003/eureka/
```
**启动类：**
增加`@EnableDiscoveryClient`注解，注册到eureka上
```java
@SpringBootApplication
@EnableZuulProxy
@EnableDiscoveryClient
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
**启动测试**
![](4.png)
已经注册成功。
访问localhost:8002/message-server/message/lisi
![](5.png)
当将zuul注册到eureka上的时候，zuul会自动发现服务，无需我们配置。
Zuul的路由规则如下：

http://ZUUL_HOST:ZUUL_PORT/微服务在Eureka上的serviceId/**会被转发到serviceId对应的微服务

### zuul的过滤器
Filter是Zuul的核心，用来实现对外服务的控制。Filter的生命周期有4个，分别是“PRE”、“ROUTING”、“POST”、“ERROR”，整个生命周期可以用下图来表示。
![](6.png)
Zuul大部分功能都是通过过滤器来实现的，这些过滤器类型对应于请求的典型生命周期。

- **PRE：** 这种过滤器在请求被路由之前调用。我们可利用这种过滤器实现身份验证、在集群中选择请求的微服务、记录调试信息等。
- **ROUTING：**这种过滤器将请求路由到微服务。这种过滤器用于构建发送给微服务的请求，并使用Apache HttpClient或Netfilx Ribbon请求微服务。
- **POST：**这种过滤器在路由到微服务以后执行。这种过滤器可用来为响应添加标准的HTTP Header、收集统计信息和指标、将响应从微服务发送给客户端等。
-** ERROR：**在其他阶段发生错误时执行该过滤器。 除了默认的过滤器类型，Zuul还允许我们创建自定义的过滤器类型。例如，我们可以定制一种STATIC类型的过滤器，直接在Zuul中生成响应，而不将请求转发到后端的微服务。

#### 自定义的Filter
实现自定义Filter，需要继承ZuulFilter的类，并覆盖其中的4个方法。
```java
public class MyFilter extends ZuulFilter {
    @Override
    String filterType() {
        return "pre"; //定义filter的类型，有pre、route、post、error四种
    }

    @Override
    int filterOrder() {
        return 10; //定义filter的顺序，数字越小表示顺序越高，越先执行
    }

    @Override
    boolean shouldFilter() {
        return true; //表示是否需要执行该filter，true表示执行，false表示不执行
    }

    @Override
    Object run() {
        return null; //filter需要执行的具体操作
    }
}
```
#### 实例
```java
public class MyFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 10;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        RequestContext context = RequestContext.getCurrentContext();
        HttpServletRequest request = context.getRequest();
		//当请求有token时 放行，否则 不放行
        String token = request.getParameter("token");
        if(StringUtils.isNotBlank(token)){
            context.setSendZuulResponse(true);
            context.setResponseStatusCode(200);
            context.set("data", true);
            return null;
        }else{
            context.setSendZuulResponse(false);
            context.setResponseStatusCode(400);
            context.set("data", false);
            return null;
        }
    }
}
```
**将自定义filter加入拦截队列中：**
```java
@Configuration
public class FilterConfig {
    @Bean
    public MyFilter getMyFilter(){
        return new MyFilter();
    }
}
```
**pom配置：**
```xml
server:
  port: 8080
zuul:
  routes:
    user:
      path: /demo/**
      url: http://heimamba.com
```
**启动项目：**
访问：http://localhost:8080/demo,页面无法正常访问
![](7.png)

访问：http://localhost:8080/demo&token=xxx，页面可以正常访问
![](8.png)

使用“PRE”类型的Filter做很多的验证工作，在实际使用中我们可以结合shiro、oauth2.0等技术去做鉴权、验证。

#### 路由熔断
当后端服务出现故障时候，我们希望降级操作，而不是传递给最外层，zuul提供了熔断功能。
```java
public interface ZuulFallbackProvider {
   /**
	 * The route this fallback will be used for.
	 * @return The route the fallback will be used for.
	 */
	public String getRoute();

	/**
	 * Provides a fallback response.
	 * @return The fallback response.
	 */
	public ClientHttpResponse fallbackResponse();
}
```
实现类通过实现getRoute方法，告诉Zuul它是负责哪个route定义的熔断。而fallbackResponse方法则是告诉 Zuul 断路出现时，它会提供一个什么返回值来处理请求。

**自定义熔断:**
```java
@Component
public class ZuulFallBack implements ZuulFallbackProvider {
    @Override
    public String getRoute() {
        return "message-server";
    }

    @Override
    public ClientHttpResponse fallbackResponse() {
        return new ClientHttpResponse() {
            @Override
            public HttpStatus getStatusCode() throws IOException {
                return HttpStatus.OK;
            }

            @Override
            public int getRawStatusCode() throws IOException {
                return 200;
            }

            @Override
            public String getStatusText() throws IOException {
                return "sucess";
            }

            @Override
            public void close() {

            }

            @Override
            public InputStream getBody() throws IOException {
                //出现异常时候
                return new ByteArrayInputStream("this is bad request".getBytes());
            }

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);
                return headers;
            }
        };
    }
}
```
**yml配置：**
```xml
server:
  port: 8004
eureka:
  client:
    service-url:
      defaultZone: http://localhost:9001/eureka/,http://localhost:9002/eureka/
```
继续用上面的eureka的producer：
![](9.png)

**启动项目：**
访问：http://localhost:8004/message-server/message/liss
![](10.png)
可以正常访问。
关闭message-server服务，再次访问：http://localhost:8004/message-server/message/liss
![](11.png)
熔断说明已经起到作用。

**Zuul 目前只支持服务级别的熔断，不支持具体到某个URL进行熔断。**

### 路由重试

有时候因为网络或者其它原因，服务可能会暂时的不可用，这个时候我们希望可以再次对服务进行重试，Zuul也帮我们实现了此功能，需要结合Spring Retry 一起来实现.

#### pom依赖
```xml
<dependency>
	<groupId>org.springframework.retry</groupId>
	<artifactId>spring-retry</artifactId>
</dependency>
```

#### 开启重试
```xml
zuul:
  retryable: true
ribbon:
  MaxAutoRetries: 2
  MaxAutoRetriesNextServer: 0
```

#### 修改 producer
```java
@RestController
public class MessageController {
    Logger logger = (Logger) LoggerFactory.getLogger(MessageController.class);
    @RequestMapping("/message/{name}")
    public String getMessage(@PathVariable String name){
        logger.info("request name is "+name);
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return name + "  测试！";
    }
}
```

#### 启动项目

访问： http://localhost:8004/message-server/message/liss
2017-08-10 19:50:32.401  INFO 19488 --- [io-9001-exec-14] o.s.c.n.z.f.route.FallbackProvider   : request name is liss

2017-08-10 19:50:33.402  INFO 19488 --- [io-9001-exec-15] o.s.c.n.z.f.route.FallbackProvider   : request name is liss

2017-08-10 19:50:34.404  INFO 19488 --- [io-9001-exec-16] o.s.c.n.z.f.route.FallbackProvider   : request name is liss

说明进行了三次的请求，也就是进行了两次的重试。这样也就验证了我们的配置信息，完成了Zuul的重试功能。

### zuul的高可用
![](13.png)

不同的客户端使用不同的负载将请求分发到后端的Zuul，Zuul在通过Eureka调用后端服务，最后对外输出。因此为了保证Zuul的高可用性，前端可以同时启动多个Zuul实例进行负载，在Zuul的前端使用Nginx或者F5进行负载转发以达到高可用性。