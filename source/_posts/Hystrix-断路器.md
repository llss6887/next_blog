---
title: Hystrix-断路器
date: 2017-08-03 23:59:47
urlname: Hystrix
tags:
  - Hystrix
  - SpringCloud
categories: SpringCloud
---

## 断路器：Hystrix客户端
Netflix的创造了一个调用的库Hystrix实现了断路器图案。在微服务架构中，通常有多层服务调用。
断路器就比如家里面的保险丝，一但出现故障，就自动断开，避免造成火灾啥的。
![](1.png)
微服务图
<!--more-->

较低级别的服务中的服务故障可能导致用户级联故障。当对特定服务的呼叫达到一定阈值时（Hystrix中的默认值为5秒内的20次故障），电路打开，不进行通话。在错误和开路的情况下，开发人员可以提供后备。
![](2.png)
Hystrix回退防止级联故障

开放式电路会停止级联故障，并允许不必要的或失败的服务时间来愈合。回退可以是另一个Hystrix保护的调用，静态数据或一个正常的空值。回退可能被链接，所以第一个回退使得一些其他业务电话又回到静态数据。

### 实战

#### 目录结构
![](3.png)

#### pom
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-hystrix</artifactId>
    </dependency>
</dependencies>
```


#### 启动类
在启动类上加上@EnableHystrix或者@EnableCircuitBreaker开启熔断
```java
@SpringBootApplication
@EnableHystrix //开启熔断器
public class HystrixApplication {
    public static void main(String[] args) {
        SpringApplication.run(HystrixApplication.class, args);
    }
}
```

#### 服务类

**重要的是要记住，HystrixCommand和fallback应该放在同一个类中，并且具有相同的方法参数。**
```java
@Service
//全局的降级操作，fallback的方法不能有参数
@DefaultProperties(defaultFallback = "failback")
public class HystrixService {

    //回调函数的参数和原的一致
    @HystrixCommand(fallbackMethod = "failHystrix")
    public String getHystrix(String name){
        return name + ": is ok";
    }

    public String failHystrix(String name){
        return name + ": is fail";
    }
    public String failback(){
        return ": is fail";
    }
}
```

#### 调用入口
```java
@RestController
public class HystrixController {
    @Autowired
    private HystrixService service;

    @GetMapping("/hystrix/{name}")
    public String getHystrix(@PathVariable String name){
        return service.getHystrix(name);
    }
}
```

#### 熔断测试
正常调用
![](4.png)
将service方法加入异常在次调用
```java
@Service
//全局的降级操作，fallback的方法不能有参数
@DefaultProperties(defaultFallback = "failback")
public class HystrixService {

    //回调函数的参数和原的一致
    @HystrixCommand(fallbackMethod = "failHystrix")
    public String getHystrix(String name){
		int i = 1/0;
        return name + ": is ok";
    }

    public String failHystrix(String name){
        return name + ": is fail";
    }
    public String failback(){
        return ": is fail";
    }
}
```
![](5.png)
已经走了降级返回的静态返回值。

全局测试：
全局是 不写fallbackMethod，会在@DefaultProperties的defaultFallback找对应的降级方法。
全局的回退方法不应该有任何参数，除了额外的参数以获得执行异常，不应抛出任何异常。下面按优先顺序降序列出的回退：

命令回退使用fallbackMethod属性定义@HystrixCommand
使用defaultFallback属性定义的命令默认回退@HystrixCommand
使用defaultFallback属性定义的类默认回退@DefaultProperties
```java
@Service
//全局的降级操作，fallback的方法不能有参数
@DefaultProperties(defaultFallback = "failback")
public class HystrixService {

    //回调函数的参数和原的一致
    //@HystrixCommand
    public String getHystrix(String name){
		int i = 1/0;
        return name + ": is ok";
    }

    public String failHystrix(String name){
        return name + ": is fail";
    }
    public String failback(){
        return ": is fail";
    }
}
```

调用结果：
![](6.png)
**
如果将回调方法加入@HystrixCommand，则回退方法也可以作为单独的Hystrix命令运行**
```java
@HystrixCommand(fallbackMethod = "failHystrix")
public String getHystrix(String name){
    int i = 1 / 0;
    return name + ": is ok";
}

@HystrixCommand
public String failHystrix(String name){
    return name + ": is fail";
}
```

### Hystrix 参数详解
```java
hystrix.command.default和hystrix.threadpool.default中的default为默认CommandKey

Command Properties
Execution相关的属性的配置：
hystrix.command.default.execution.isolation.strategy 隔离策略，默认是Thread, 可选Thread｜Semaphore

hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds 命令执行超时时间，默认1000ms

hystrix.command.default.execution.timeout.enabled 执行是否启用超时，默认启用true
hystrix.command.default.execution.isolation.thread.interruptOnTimeout 发生超时是是否中断，默认true
hystrix.command.default.execution.isolation.semaphore.maxConcurrentRequests 最大并发请求数，默认10，该参数当使用ExecutionIsolationStrategy.SEMAPHORE策略时才有效。如果达到最大并发请求数，请求会被拒绝。理论上选择semaphore size的原则和选择thread size一致，但选用semaphore时每次执行的单元要比较小且执行速度快（ms级别），否则的话应该用thread。
semaphore应该占整个容器（tomcat）的线程池的一小部分。
Fallback相关的属性
这些参数可以应用于Hystrix的THREAD和SEMAPHORE策略

hystrix.command.default.fallback.isolation.semaphore.maxConcurrentRequests 如果并发数达到该设置值，请求会被拒绝和抛出异常并且fallback不会被调用。默认10
hystrix.command.default.fallback.enabled 当执行失败或者请求被拒绝，是否会尝试调用hystrixCommand.getFallback() 。默认true
Circuit Breaker相关的属性
hystrix.command.default.circuitBreaker.enabled 用来跟踪circuit的健康性，如果未达标则让request短路。默认true
hystrix.command.default.circuitBreaker.requestVolumeThreshold 一个rolling window内最小的请求数。如果设为20，那么当一个rolling window的时间内（比如说1个rolling window是10秒）收到19个请求，即使19个请求都失败，也不会触发circuit break。默认20
hystrix.command.default.circuitBreaker.sleepWindowInMilliseconds 触发短路的时间值，当该值设为5000时，则当触发circuit break后的5000毫秒内都会拒绝request，也就是5000毫秒后才会关闭circuit。默认5000
hystrix.command.default.circuitBreaker.errorThresholdPercentage错误比率阀值，如果错误率>=该值，circuit会被打开，并短路所有请求触发fallback。默认50
hystrix.command.default.circuitBreaker.forceOpen 强制打开熔断器，如果打开这个开关，那么拒绝所有request，默认false
hystrix.command.default.circuitBreaker.forceClosed 强制关闭熔断器 如果这个开关打开，circuit将一直关闭且忽略circuitBreaker.errorThresholdPercentage
Metrics相关参数
hystrix.command.default.metrics.rollingStats.timeInMilliseconds 设置统计的时间窗口值的，毫秒值，circuit break 的打开会根据1个rolling window的统计来计算。若rolling window被设为10000毫秒，则rolling window会被分成n个buckets，每个bucket包含success，failure，timeout，rejection的次数的统计信息。默认10000
hystrix.command.default.metrics.rollingStats.numBuckets 设置一个rolling window被划分的数量，若numBuckets＝10，rolling window＝10000，那么一个bucket的时间即1秒。必须符合rolling window % numberBuckets == 0。默认10
hystrix.command.default.metrics.rollingPercentile.enabled 执行时是否enable指标的计算和跟踪，默认true
hystrix.command.default.metrics.rollingPercentile.timeInMilliseconds 设置rolling percentile window的时间，默认60000
hystrix.command.default.metrics.rollingPercentile.numBuckets 设置rolling percentile window的numberBuckets。逻辑同上。默认6
hystrix.command.default.metrics.rollingPercentile.bucketSize 如果bucket size＝100，window＝10s，若这10s里有500次执行，只有最后100次执行会被统计到bucket里去。增加该值会增加内存开销以及排序的开销。默认100
hystrix.command.default.metrics.healthSnapshot.intervalInMilliseconds 记录health 快照（用来统计成功和错误绿）的间隔，默认500ms
Request Context 相关参数
hystrix.command.default.requestCache.enabled 默认true，需要重载getCacheKey()，返回null时不缓存
hystrix.command.default.requestLog.enabled 记录日志到HystrixRequestLog，默认true

Collapser Properties 相关参数
hystrix.collapser.default.maxRequestsInBatch 单次批处理的最大请求数，达到该数量触发批处理，默认Integer.MAX_VALUE
hystrix.collapser.default.timerDelayInMilliseconds 触发批处理的延迟，也可以为创建批处理的时间＋该值，默认10
hystrix.collapser.default.requestCache.enabled 是否对HystrixCollapser.execute() and HystrixCollapser.queue()的cache，默认true

ThreadPool 相关参数
线程数默认值10适用于大部分情况（有时可以设置得更小），如果需要设置得更大，那有个基本得公式可以follow：
requests per second at peak when healthy × 99th percentile latency in seconds + some breathing room
每秒最大支撑的请求数 (99%平均响应时间 + 缓存值)
比如：每秒能处理1000个请求，99%的请求响应时间是60ms，那么公式是：
1000 （0.060+0.012）

基本得原则时保持线程池尽可能小，他主要是为了释放压力，防止资源被阻塞。
当一切都是正常的时候，线程池一般仅会有1到2个线程激活来提供服务

hystrix.threadpool.default.coreSize 并发执行的最大线程数，默认10
hystrix.threadpool.default.maxQueueSize BlockingQueue的最大队列数，当设为－1，会使用SynchronousQueue，值为正时使用LinkedBlcokingQueue。该设置只会在初始化时有效，之后不能修改threadpool的queue size，除非reinitialising thread executor。默认－1。
hystrix.threadpool.default.queueSizeRejectionThreshold 即使maxQueueSize没有达到，达到queueSizeRejectionThreshold该值后，请求也会被拒绝。因为maxQueueSize不能被动态修改，这个参数将允许我们动态设置该值。if maxQueueSize == -1，该字段将不起作用
hystrix.threadpool.default.keepAliveTimeMinutes 如果corePoolSize和maxPoolSize设成一样（默认实现）该设置无效。如果通过plugin（https://github.com/Netflix/Hystrix/wiki/Plugins）使用自定义实现，该设置才有用，默认1.
hystrix.threadpool.default.metrics.rollingStats.timeInMilliseconds 线程池统计指标的时间，默认10000
hystrix.threadpool.default.metrics.rollingStats.numBuckets 将rolling window划分为n个buckets，默认10
```

例如：
```java
@RequestMapping(value = "/getOrderPageList", method = RequestMethod.POST)
@HystrixCommand(
    fallbackMethod = "getOrderPageListFallback",
    threadPoolProperties = {  //10个核心线程池,超过20个的队列外的请求被拒绝; 当一切都是正常的时候，线程池一般仅会有1到2个线程激活来提供服务
        @HystrixProperty(name = "coreSize", value = "10"),
        @HystrixProperty(name = "maxQueueSize", value = "100"),
        @HystrixProperty(name = "queueSizeRejectionThreshold", value = "20")},
    commandProperties = {
        @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "10000"), //命令执行超时时间
        @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "2"), //若干10s一个窗口内失败三次, 则达到触发熔断的最少请求量
        @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "30000") //断路30s后尝试执行, 默认为5s
    })
public String getDemo(String name){
    //do ..
    return "this is ok";
}

public String fallBack(String name){
    System.out.println("执行降级策略");
    return "this is fail";
}
```

### Hystrix的监控
Hystrix的主要优点之一是它收集关于每个HystrixCommand的一套指标。Hystrix仪表板以有效的方式显示每个断路器的运行状况。

#### 依赖文件
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
</dependency>
```

#### 启动类
加入@EnableHystrixDashboard注解
```java
@SpringBootApplication
@EnableHystrix 
@EnableHystrixDashboard
public class HystrixApplication {
    public static void main(String[] args) {
        SpringApplication.run(HystrixApplication.class, args);
    }
}
```

#### 启动项目
上面截图是HystrixBoard的监控首页，该页面并没有什么监控信息。从1,2,3标号中可以知道HystrixBoard提供了三种不同的监控方式。

默认的集群方式：通过URL http://turbine-hostname:port/turbine.stream开启，实现对默认集群的监控。

指定的集群监控，通过URL http://turbine-hostname:port/turbine.stream?cluster=[clusterName]开启对clusterName的集群监控。

单体应用的监控，通过URL http://hystrix-app:port/hystrix.stream开启，实现对具体某个服务实例的监控。

Delay:改参数用来控制服务器上轮训监控信息的延迟时间，默认为2000毫秒，可以通过配置该属性来降低客户端的网络和CPU消耗。

Title：该参数对应了上图头补标题Hystrix Stream之后的内容，默认会使用具体监控实例的URL,可以通过该信息来展示更合适的标题。

输入 localhost:port/hystrix
![](9.png)
输入熔断器所在的应用的host:ip/hystrix.stream 可以进入到监控页面，没有请求都是loading
![](10.png)
发起一次请求hystrix
![](11.png)
仪表盘显示了请求的监控，一个单应用的熔断监控就完成了。
下面是访问的时候的截图
![](12.png)


## Turbine
从个人实例看，Hystrix数据在系统整体健康方面不是非常有用。Turbine是将所有相关/hystrix.stream端点聚合到Hystrix仪表板中使用的/turbine.stream的应用程序。个人实例位于Eureka。运行Turbine就像使用@EnableTurbine注释（例如使用spring-cloud-starter-turbine设置类路径）注释主类一样简单。
```java
@SpringBootApplication
@EnableHystrixDashboard
@EnableTurbine
public class TurbineApp {
    public static void main(String[] args) {
        SpringApplication.run(TurbineApp.class, args);
    }
}


spring.application.name=hystrix-dashboard-turbine
server.port=8001
turbine.appConfig=node01,node02  #配置eureka的 哪些服务 表示监控eureka的哪些服务
turbine.aggregator.clusterConfig= default #聚合哪些集群 默认为default。可使用http://.../turbine.stream?cluster={clusterConfig之一}访问
turbine.clusterNameExpression= new String("default")

eureka.client.serviceUrl.defaultZone=http://localhost:8000/eureka/
```
启动查看
访问 http://localhost:8001/turbine.stream
并且会不断刷新以获取实时的监控数据，说明和单个的监控类似，返回监控项目的信息。进行图形化监控查看，输入：http://localhost:8001/hystrix，输入： http://localhost:8001/turbine.stream，然后点击 Monitor Stream ,可以看到出现了俩个监控列表

![](13.jpg)