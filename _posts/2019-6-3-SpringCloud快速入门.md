---
layout:     post   				    # 使用的布局（不需要改）
title:      SpringCloud快速入门			# 标题 
subtitle:   以Spring Cloud教程合集为基础的SpringCloud文档学习 #副标题
date:       2019-6-3 				# 时间
author:     Tempo					# 作者
header-img: img/post-bg-map.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - SpringCould
---

因为项目需要，所以本文v1.0只是粗略看一遍，主要关注配置  
[by](https://blog.csdn.net/u012702547/article/details/78717512)

个人总结：
* Spring Cloud  
  一站式分布式框架，提供一组常见模式的工具（例如配置管理，服务发现，断路器，智能路由，微代理，控制总线）。开发人员可以快速地支持实现这些模式的服务和应用程序。
    * eureka 一个服务治理组件，包含服务注册和发现等
    * Ribbon 一个基于HTTP和TCP的客户端负载均衡器，从Eureka注册中心去获取服务端列表，然后进行轮询访问以到达负载均衡的作用
    * Hystrix 一个断路器，用来提供服务崩溃的备胎
    * Feign Feign整合了Ribbon和Hystrix（两个消费者会调用的服务）
    * Zuul Feign升级版，一个网关，所有的外部访问都要先经过API网关，然后API网关来实现请求路由、负载均衡、权限验证等功能
    * Spring Cloud Config 分布式配置中心，是一个独立的微服务应用，用来连接配置仓库(git)并将获取到的配置信息提供给客户端使用
* Spring Cloud Bus
    * RabbitMQ 
    * Kafka 
Spring

# 1.使用Spring Cloud搭建服务注册中心 
本文介绍在Spring Cloud中使用Eureka搭建一个服务注册中心，然后再向其中注册服务。  
Eureka是一个服务治理组件，包含了服务注册中心、服务注册与发现机制。  
示例中，搭建了一个server的springboot工程，一个client的springboot工程。
#### Server配置
```json
server.port=1111
eureka.instance.hostname=localhost
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.client.service-url.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/
```

> 1.server.port=1111表示设置该服务注册中心的端口号  
2.eureka.instance.hostname=localhost表示设置该服务注册中心的hostname  
3.eureka.client.register-with-eureka=false,由于我们目前创建的应用是一个服务注册中心，而不是普通的应用，默认情况下，这个应用会向注册中心（也是它自己）注册它自己，设置为false表示禁止这种默认行为  
4.eureka.client.fetch-registry=false,表示不去检索其他的服务，因为服务注册中心本身的职责就是维护服务实例，它也不需要去检索其他服务

#### Client配置
```json
spring.application.name=hello-service
eureka.client.service-url.defaultZone=http://localhost:1111/eureka
```

# 2.使用Spring Cloud搭建高可用服务注册中心

服务注册中心是一个单节点的服务注册中心，这样一旦发生了故障，那么整个服务就会瘫痪，所以我们需要一个高可用的服务注册中心，在Eureka中，通过集群来解决这个问题。  
Eureka Server的高可用实际上就是将自己作为服务向其他服务注册中心注册自己，这样就会形成一组互相注册的服务注册中心，进而实现服务清单的互相同步，达到高可用的效果。

#### peer配置
向server工程中添加两个配置文件application-peer1.properties和application-peer2.properties：

> application-peer1.properties:
```json
server.port=1111
eureka.instance.hostname=peer1
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.client.service-url.defaultZone=http://peer2:1112/eureka/
```

> application-peer2.properties:
```json
server.port=1112
eureka.instance.hostname=peer2
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.client.service-url.defaultZone=http://peer1:1111/eureka/
```

>1.在peer1的配置文件中，让它的service-url指向peer2，在peer2的配置文件中让它的service-url指向peer1  
2.为了让peer1和peer2能够被正确的访问到，我们需要在C:\Windows\System32\drivers\etc目录下的hosts文件总添加两行配置，如下:  
127.0.0.1 peer1  
127.0.0.1 peer2  
3.由于peer1和peer2互相指向对方，实际上我们构建了一个双节点的服务注册中心集群

[一个猜想]：如果想要提高服务集群的稳定性，可以增加新的server节点。

# 3.Spring Cloud中服务的发现与消费
服务的发现由Eureka客户端来完成，而服务的消费由Ribbon来完成。Ribbo是一个基于HTTP和TCP的客户端负载均衡器，当我们将Ribbon和Eureka一起使用时，Ribbon会从Eureka注册中心去获取服务端列表，然后进行轮询访问以到达负载均衡的作用，服务端是否在线这些问题则交由Eureka去维护。  
在具体实现中，创建一个Spring Boot项目，然后添加Eureka和Ribbon依赖  
创建一个Controller类，并向Controller类中注入RestTemplate对象，同时在Controller中提供一个名为/ribbon-consumer的接口，在该接口中，我们通过刚刚注入的restTemplate来实现对HELLO-SERVICE服务提供的/hello接口进行调用  
最后我们需要在application.properties中配置服务注册中心的位置，如下：
```json
spring.application.name=ribbon-consumer  
server.port=9000  
eureka.client.service-url.defaultZone=http://peer1:1111/eureka
```
调用服务后，Ribbon输出了当前客户端维护的HELLO-SERVICE的服务列表情况，每一个provider的位置都展示出来，Ribbon就是按照这个列表进行轮询，进而实现基于客户端的负载均衡。同时这里的日志还输出了其他信息，比如各个实例的请求总数量，第一次连接信息，上一次连接信息以及总的请求失败数量等。

# 4.Eureka中的核心概念

#### 服务提供者
Eureka服务治理体系支持跨平台，虽然我们前文使用了Spring Boot来作为服务提供者，但是对于其他技术平台只要支持Eureka通信机制，一样也是可以作为服务提供者，服务提供者主要有如下一些功能：

* 服务注册  
服务提供者在启动的时候会通过发送REST请求将自己注册到Eureka Server上，同时还携带了自身服务的一些元数据信息。Eureka Server在接收到这个REST请求之后，将元数据信息存储在一个双层结构的Map集合中，第一层的key是服务名，第二层的key是具体服务的实例名。eureka.client.register-with-eureka，该值默认就为true，表示启动注册操作，如果设置为false则不会启动注册操作。

* 服务同步  
如果两个服务提供者的信息分别被两个服务注册中心所维护，但是由于服务注册中心之间也互相注册为服务，当服务提供者发送请求到一个服务注册中心时，它会将该请求转发给集群中相连的其他注册中心，从而实现注册中心之间的服务同步，通过服务同步，两个服务提供者的服务信息我们就可以通过任意一台注册中心来获取到。

* 服务续约  
在注册完服务之后，服务提供者会维护一个心跳来不停的告诉Eureka Server：“我还在运行”，以防止Eureka Server将该服务实例从服务列表中剔除，这个动作称之为服务续约，和服务续约相关的属性有两个，如下：
```json
eureka.instance.lease-expiration-duration-in-seconds=90  
eureka.instance.lease-renewal-interval-in-seconds=30
```
第一个配置用来定义服务失效时间，默认为90秒，第二个用来定义服务续约的间隔时间，默认为30秒。

#### 服务消费者
消费者主要是从服务注册中心获取服务列表，拿到服务提供者的列表之后，服务消费者就知道去哪里调用它所需要的服务了。

* 获取服务  
当我们启动服务消费者的时候，它会发送一个REST请求给服务注册中心来获取服务注册中心上面的服务提供者列表，而Eureka Server上则会维护一份只读的服务清单来返回给客户端，这个服务清单并不是实时数据，而是一份缓存数据，默认30秒更新一次，如果想要修改清单更新的时间间隔，可以通过eureka.client.registry-fetch-interval-seconds=30来修改，单位为秒(注意这个修改是在eureka-server上来修改)。另一方面，我们的服务消费端要确保具有获取服务提供者的能力，此时要确保eureka.client.fetch-registry=true这个配置为true。

* 服务调用  
服务消费者从服务注册中心拿到服务提供者列表之后，通过服务名就可以获取具体提供服务的实例名和该实例的元数据信息，客户端将根据这些信息来决定调用哪个实例，我们之前采用了Ribbon，Ribbon中默认采用轮询的方式去调用服务提供者，进而实现了客户端的负载均衡。

* 服务下线  
服务提供者在运行的过程中可能会发生关闭或者重启，当服务进行正常关闭时，它会触发一个服务下线的REST请求给Eureka Server，告诉服务注册中心我要下线了，服务注册中心收到请求之后，将该服务状态置为DOWN，表示服务已下线，并将该事件传播出去，这样就可以避免服务消费者调用了一个已经下线的服务提供者了。

#### 服务注册中心
服务注册中心就是Eureka提供的服务端，它提供了服务注册与发现功能。

* 失效剔除  
上面我们说到了服务下线问题，正常的服务下线发生流程有一个前提那就是服务正常关闭,但是在实际运行中服务有可能没有正常关闭，比如系统故障、网络故障等原因导致服务提供者非正常下线，那么这个时候对于已经下线的服务Eureka采用了定时清除：Eureka Server在启动的时候会创建一个定时任务，每隔60秒就去将当前服务提供者列表中超过90秒还没续约的服务剔除出去，通过这种方式来避免服务消费者调用了一个无效的服务。

* 自我保护  
Eureka Server在运行期间会去统计心跳失败比例在15分钟之内是否低于85%，如果低于85%，Eureka Server会将这些实例保护起来，让这些实例不会过期，但是在保护期内如果服务刚好这个服务提供者非正常下线了，此时服务消费者就会拿到一个无效的服务实例，此时会调用失败，对于这个问题需要服务消费者端要有一些容错机制，如重试，断路器等。我们在单机测试的时候很容易满足心跳失败比例在15分钟之内低于85%，这个时候就会触发Eureka的保护机制，一旦开启了保护机制，则服务注册中心维护的服务实例就不是那么准确了，此时我们可以使用eureka.server.enable-self-preservation=false来关闭保护机制，这样可以确保注册中心中不可用的实例被及时的剔除。

# 5.什么是客户端负载均衡

#### 服务端负载均衡
一般情况下我们所说的负载均衡通常都是指服务端负载均衡，服务端负载均衡又分为两种，一种是硬件负载均衡，还有一种是软件负载均衡。

硬件负载均衡主要通过在服务器节点之间安装专门用于负载均衡的设备，常见的如F5。

软件负载均衡则主要是在服务器上安装一些具有负载均衡功能的软件来完成请求分发进而实现负载均衡，常见的就是Nginx。都会维护一个可用的服务端清单，然后通过心跳机制来删除故障的服务端节点以保证清单中都是可以正常访问的服务端节点，此时当客户端的请求到达负载均衡服务器时，负载均衡服务器按照某种配置好的规则从可用服务端清单中选出一台服务器去处理客户端的请求。这就是服务端负载均衡。

#### 客户端负载均衡

客户端负载均衡和服务端负载均衡最大的区别在于服务清单所存储的位置。在客户端负载均衡中，所有的客户端节点都有一份自己要访问的服务端清单，这些清单统统都是从Eureka服务注册中心获取的。在Spring Cloud中我们如果想要使用客户端负载均衡，方法很简单，开启@LoadBalanced注解即可，这样客户端在发起请求的时候会先自行选择一个服务端，向该服务端发起请求，从而实现负载均衡。

# 6.Spring RestTemplate中几种常见的请求方式
# 7.RestTemplate的逆袭之路，从发送请求到负载均衡
# 8.Spring Cloud中负载均衡器概览
# 9.Spring Cloud中的负载均衡策略

# 10.Spring Cloud中的断路器Hystrix
服务提供者被关闭时我们需要断路器，如果请求超时也会触发熔断请求，调用回调方法返回数据。 
#### 具体实践
首先我们需要在服务消费者pom中引入hystrix  
引入hystrix之后，我们需要在入口类上通过@EnableCircuitBreaker开启断路器功能,我们也可以使用一个名为@SpringCloudApplication  
然后我们创建一个HelloService类，如下：
```Java
@Service
public class HelloService {
    @Autowired
    private RestTemplate restTemplate;

    @HystrixCommand(fallbackMethod = "error")
    public String hello() {
        ResponseEntity<String> responseEntity = restTemplate.getForEntity("http://HELLO-SERVICE/hello", String.class);
        return responseEntity.getBody();
    }

    public String error() {
        return "error";
    }
}
```
> 1.RestTemplate执行网络请求的操作我们放在HelloService中来完成。  
2.error方法是一个请求失败时回调的方法。  
3.在hello方法上通过@HystrixCommand注解来指定请求失败时回调的方法。  

最后我们将ConsumerController的逻辑修改成下面这样：
```Java
@RestController
public class ConsumerController {
    @Autowired
    private HelloService helloService;
    @RequestMapping(value = "/ribbon-consumer",method = RequestMethod.GET)
    public String helloController() {
        return helloService.hello();
    }
}
```

# 11.Spring Cloud自定义Hystrix请求命令
# 12.Spring Cloud中Hystrix的服务降级与异常处理
fallbackMethod所描述的函数实际上就是一个备胎，用来实现服务的降级处理，在注解中我们可以通过fallbackMethod属性来指定降级处理的方法名称，在自定义Hystrix请求命令时我们可以通过重写getFallback函数来处理服务降级之后的逻辑。使用注解来定义服务降级逻辑时，服务降级函数和@HystrixCommand注解要处于同一个类中，同时，服务降级函数在执行过程中也有可能发生异常，所以也可以给服务降级函数添加‘备胎’，如下：
```Java
@HystrixCommand(fallbackMethod = "error1")
public Book test2() {
    return restTemplate.getForObject("http://HELLO-SERVICE/getbook1", Book.class);
}

@HystrixCommand(fallbackMethod = "error2")
public Book error1() {
    //发起某个网络请求（可能失败）
    return null;
}
public Book error2() {
    return new Book();
}
```
在实际开发中，并不是所有的请求都要提前预备好服务降级问题，如果我就是要将服务调用失败的信息展示给用户，那么此时就没有必要添加断路器了。  
我们在调用服务提供者时有可能会抛异常，默认情况下方法抛了异常会自动进行服务降级，交给服务降级中的方法去处理，此时，如果有一个异常抛出后我不希望进入到服务降级方法中去处理，而是直接将异常抛给用户，那么我们可以在@HystrixCommand注解中添加忽略异常。

# 13.Spring Cloud中Hystrix的请求缓存 
# 14.Spring Cloud中Hystrix的请求合并 
# 15.Spring Cloud中Hystrix仪表盘与Turbine集群监控 
# 16.Spring Cloud中声明式服务调用Feign
（并没有在微服务中找到相关配置）
# 17.Spring Cloud中Feign配置详解

#### Ribbon配置
ribbon的配置其实非常简单，直接在application.properties中配置即可，如下：

```json
# 设置连接超时时间
ribbon.ConnectTimeout=600
# 设置读取超时时间
ribbon.ReadTimeout=6000
# 对所有操作请求都进行重试
ribbon.OkToRetryOnAllOperations=true
# 切换实例的重试次数
ribbon.MaxAutoRetriesNextServer=2
# 对当前实例的重试次数
ribbon.MaxAutoRetries=1
```
这个参数的测试方式很简单，我们可以在服务提供者的hello方法中睡眠5s，然后调节这个参数就能看到效果。下面的参数是我们配置的超时重试参数，超时之后，首先会继续尝试访问当前实例1次，如果还是失败，则会切换实例访问，切换实例一共可以切换两次，两次之后如果还是没有拿到访问结果，则会报Read timed out executing GET http://hello-service/hello。
但是这种配置是一种全局配置，就是是对所有的请求生效的，如果我想针对不同的服务配置不同的连接超时和读取超时，那么我们可以在属性的前面加上服务的名字，如下：
```json
# 设置针对hello-service服务的连接超时时间
hello-service.ribbon.ConnectTimeout=600
# 设置针对hello-service服务的读取超时时间
hello-service.ribbon.ReadTimeout=6000
# 设置针对hello-service服务所有操作请求都进行重试
hello-service.ribbon.OkToRetryOnAllOperations=true
# 设置针对hello-service服务切换实例的重试次数
hello-service.ribbon.MaxAutoRetriesNextServer=2
# 设置针对hello-service服务的当前实例的重试次数
hello-service.ribbon.MaxAutoRetries=1
```
#### Hystrix配置
Feign中Hystrix的配置和Ribbon有点像，基础配置如下：
```json
# 设置熔断超时时间
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=10000
# 关闭Hystrix功能（不要和上面的配置一起使用）
feign.hystrix.enabled=false
# 关闭熔断功能
hystrix.command.default.execution.timeout.enabled=false
```
这种配置也是全局配置，如果我们想针对某一个接口配置，比如/hello接口，那么可以按照下面这种写法，如下：
```json
# 设置熔断超时时间
hystrix.command.hello.execution.isolation.thread.timeoutInMilliseconds=10000
# 关闭熔断功能
hystrix.command.hello.execution.timeout.enabled=false
```

# 18.Spring Cloud中的API网关服务Zuul
API网关是一个更为智能的应用服务器，它有点类似于我们微服务架构系统的gateway，所有的外部访问都要先经过API网关，然后API网关来实现请求路由、负载均衡、权限验证等功能。  
Spring Cloud中提供的Spring Cloud Zuul实现了API网关的功能，本文我们就先来看看Spring Cloud Zuul的一个基本使用。

#### 具体实例
1. 创建Spring Boot工程并添加依赖  
首先我们创建一个普通的Spring Boot工程名为api-gateway，然后添加相关依赖，这里我们主要添加两个依赖spring-cloud-starter-zuul和spring-cloud-starter-eureka，spring-cloud-starter-zuul依赖中则包含了ribbon、hystrix、actuator等。
2. 添加注解  
然后在入口类上添加@EnableZuulProxy注解表示开启Zuul的API网关服务功能
3. 配置路由规则
application.properties文件中的配置可以分为两部分，一部分是Zuul应用的基础信息，还有一部分则是路由规则，如下：
```json
# 基础信息配置
spring.application.name=api-gateway
server.port=2006
# 路由规则配置
zuul.routes.api-a.path=/api-a/ **
zuul.routes.api-a.serviceId=feign-consumer

# API网关也将作为一个服务注册到eureka-server上
eureka.client.service-url.defaultZone=http://localhost:1111/eureka/
```

>我们在这里配置了路由规则所有符合/api-a/**的请求都将被转发到feign-consumer服务上，至于feign-consumer服务的地址到底是什么则由eureka-server去分析，我们这里只需要写上服务名即可。  
以上面的配置为例，如果我请求http://localhost:2006/api-a/hello1接口则相当于请求http://localhost:2005/hello1(我这里feign-consumer的地址为http://localhost:2005)，我们在路由规则中配置的api-a是路由的名字，可以任意定义，但是一组path和serviceId映射关系的路由名要相同。

#### 请求过滤
构建好了网关，接下来我们就来看看如何利用网关来实现一个简单的权限验证。这里就涉及到了Spring Cloud Zuul中的另外一个核心功能：请求过滤。请求过滤有点类似于Java中Filter过滤器，先将所有的请求拦截下来，然后根据现场情况做出不同的处理。  
首先我们定义一个过滤器继承自ZuulFilter，如下：  
```java

public class PermisFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        String login = request.getParameter("login");
        if (login == null) {
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            ctx.addZuulResponseHeader("content-type","text/html;charset=utf-8");
            ctx.setResponseBody("非法访问");
        }
        return null;
    }
}
```

> 1.filterType方法的返回值为过滤器的类型，过滤器的类型决定了过滤器在哪个生命周期执行，pre表示在路由之前执行过滤器，其他可选值还有post、error、route和static，当然也可以自定义。  
2.filterOrder方法表示过滤器的执行顺序，当过滤器很多时，这个方法会有意义。  
3.shouldFilter方法用来判断过滤器是否执行，true表示执行，false表示不执行，在实际开发中，我们可以根据当前请求地址来决定要不要对该地址进行过滤，这里我直接返回true。  
4.run方法则表示过滤的具体逻辑，假设请求地址中携带了login参数的话，则认为是合法请求，否则就是非法请求，如果是非法请求的话，首先设置ctx.setSendZuulResponse(false);表示不对该请求进行路由，然后设置响应码和响应值。这个run方法的返回值在当前版本(Dalston.SR3)中暂时没有任何意义，可以返回任意值。  

然后在入口类中配置相关的Bean即可，如下：

```java
@Bean
PermisFilter permisFilter() {
    return new PermisFilter();
}
```

# 20.Spring Cloud Zuul中路由配置细节
[还可参考](https://www.jianshu.com/p/4c3bc42640d8)
我们说当我的访问地址符合/api-a/**规则的时候，会被自动定位到feign-consumer服务上去，不过两行代码有点麻烦，我们可以用下面一行代码来代替，如下：

```
zuul.routes.feign-consumer=/api-a/**
```
zuul.routes后面跟着的是服务名，服务名后面跟着的是路径规则，这种配置方式显然更简单。
如果映射规则我们什么都不写，系统也给我们提供了一套默认的配置规则，默认的配置规则如下：

```
zuul.routes.feign-consumer.path=/feign-consumer/**
zuul.routes.feign-consumer.serviceId=feign-consumer
```

默认情况下，Eureka上所有注册的服务都会被Zuul创建映射关系来进行路由，但是对于我这里的例子来说，我希望提供服务的是feign-consumer，hello-service作为服务提供者只对服务消费者提供服务，不对外提供服务，如果使用默认的路由规则，则Zuul也会自动为hello-service创建映射规则，这个时候我们可以采用如下方式来让Zuul跳过hello-service服务，不为其创建路由规则：

```
zuul.ignored-services=hello-service
```
这个时候我们就可以确保先加载feign-consumer-hello的匹配规则，后加载feign-consumer的匹配规则。
```yml
spring:
  application:
    name: api-gateway
server:
  port: 2006
zuul:
  routes:
    feign-consumer-hello:
      path: /feign-consumer/hello/**
      serviceId: feign-consumer-hello
    feign-consumer:
      path: /feign-consumer/**
      serviceId: feign-consumer
eureka:
  client:
    service-url:
      defaultZone: http://localhost:1111/eureka/
```

上文我们说了一个zuul.ignored-services=hello-service属性可以忽略掉一个服务，不给某个服务设置映射规则，这个配置我们可以进一步细化，比如说我不想给/hello接口路由，那我们可以按如下方式配置(后面我都用yaml配置)：

```
zuul:
    ignored-patterns: /**/hello/**
```
此时访问/hello接口就会报404错误  
此外，我们也可以统一的为路由规则增加前缀，设置方式如下：
```
zuul:
  prefix: /myapi
```
此时我们的访问路径就变成了http://localhost:2006/myapi/feign-consumer/hello1。  

# 21.Spring Cloud Zuul中异常处理细节
# 22.分布式配置中心Spring Cloud Config初窥
随着我们的分布式项目越来越大，我们可能需要将配置文件抽取出来单独管理，Spring Cloud Config对这种需求提供了支持。  
Spring Cloud Config为分布式系统中的**外部配置**提供服务器和客户端支持。  
我们可以使用Config Server在所有环境中管理应用程序的外部属性，Config Server也称为分布式配置中心，本质上它就是一个独立的微服务应用，用来连接配置仓库并将获取到的配置信息提供给客户端使用；客户端就是我们的各个微服务应用，我们在客户端上指定配置中心的位置，客户端在启动的时候就会自动去从配置中心获取和加载配置信息。

1. 首先我们来构建一个配置中心，方式很简单，创建一个普通的Spring Boot项目，叫做config-server,创建好之后，添加依赖
2. 然后在入口类上添加@EnableConfigServer注解，表示开启配置中心服务端功能
3. 然后在application.properties中配置一下git仓库的信息
4. 接下来我们需要在github上设置好配置中心，首先我在本地找一个空文件夹，在该文件夹中创建一个文件夹叫config-repo，然后在config-repo中创建四个配置文件
5. 依次执行如下命令将本地文件同步到Github仓库中，如此之后，我们的配置文件就上传到GitHub上了
此时启动我们的配置中心，通过/{application}/{profile}/{label}就能访问到我们的配置文件了  
其中application表示配置文件的名字，对应我们上面的配置文件就是app  
profile表示环境，我们有dev、test、prod还有默认  
label表示分支，默认我们都是放在master分支上

# 23.Spring Cloud Config服务端配置细节(一)
1.首先我们需要一个远程的Git仓库，自己学习可以直接用GitHub，在在实际生产环境中，需要自己搭建一个Git服务器，远程Git仓库的作用主要是用来保存我们的配置文件
2.除了远程Git仓库之外，我们还需要一个本地Git仓库，每当Config Server访问远程Git仓库时，都会保存一份到本地，这样当远程仓库无法连接时，就直接使用本地存储的配置信息
3.至于微服务A、微服务B则是我们具体的应用，这些应用在启动的时候会从Config Server中来加载相应的配置信息
4.当微服务A/B尝试去从Config Server中加载配置信息的时候，Config Server会先通过git clone命令克隆一份配置文件保存到本地
5.由于配置文件是存储在Git仓库中，所以配置文件天然的具备版本管理功能，Git中的Hook功能可以实时监控配置文件的修改

安全保护
开发环境中我们的配置中心肯定是不能随随便便被人访问的，我们可以加上适当的保护机制，由于微服务是构建在Spring Boot之上，所以整合Spring Security是最方便的方式。

首先添加依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
然后在application.properties中配置用户名密码：

```xml
security.user.name=sang
security.user.password=123
```
最后在配置中心的客户端上配置用户名和密码即可，如下：

```xml
spring.cloud.config.username=sang
spring.cloud.config.password=123
```
# 24.Spring Cloud Config服务端配置细节(二)之加密解密
# 25.Spring Cloud Config客户端配置细节
# 26.Spring Cloud Bus之RabbitMQ初窥  
#### background

[by简书](https://www.jianshu.com/p/a5012222d01e)  
介绍一些Rabbitmq的基本概念：

* Broker：可以理解成消息队列服务器的实体，它是一个中间件应用，负责接收消息生产者的消息，然后将消息发送到消息接收者或者其他的Broker。
* Exchange：消息交换机，是消息第一个到达的地方，消息通过它指定的路由规则，分发到不同的消息队列中去。
* Queue：消息队列，消息通过发发送和路由之后最终到达的地方，到达Queue的消息即进入逻辑上等待消费的状态。每个消息都会被发送到一个或多个队列。
* Binding：绑定，它的作用就是把Exchange和Queue按照路由规则绑定起来，也就是Exchange和Queue之间的虚拟连接。
* Routing Key：路由关键字，Exchange根据这个关键字进行消息投递。
* Virtual host：虚拟主机，它是对Broker的虚拟划分，将消费者，生产者和它们的依赖的AMQP相关结构进行隔离，一般都是为了安全考虑。比如，我们可以在一个Broker中设置多个虚拟主机，对不同用户进行权限的分离。
* Connection：连接，代表生产者，消费者，Broker之间进行通信的物理网络。
* Channel：消息通道，用于连接生产者和消费者的逻辑结构。在客户端的每个连接里，可建立多个Channel，每个Channel代表一个会话任务，通过Channel可以隔离同一个连接中的不同交互内容。
* Producer：消息生产者，制造消息并发送消息的程序。
* Consumer：消息消费者，接收消息并处理消息的程序。

消息投递到队列的整个过程大致如下：  
* 1.客户端连接到消息队列服务器，打开一个Channel。
* 2.客户端声明一个Exchange，并设置相关属性。
* 3.客户端声明一个Queue，并设置相关属性。
* 4.客户端使用Routing Key，在Exchange和Queue之间建立好绑定关系。
* 5.客户端投递消息到Exchange。
* Exchange接收到消息后，根据消息的key和已经设置的Binding，进行消息路由，将消息投递到一个或多个Queue里。

Exchange也有几种类型。
* 1.Direct交互机：完全根据Key进行投递。比如，绑定时设置了Routing Key为abc，那么客户端提交的消息，只有设置了key为Routing Key的才会被投递到队列。
* 2.Topic交互机：对Key进行模式匹配后进行投递，可以使用符号#匹配一个或多个词，符号*匹配正好一个词。比如,abc.#匹配abc.def.ghi, abc.*只匹配abc.def.
* 3.Fanout交互机：不需要任何Key，它采用广播的模式，一个消息进来时，投递到与该交互机绑定的所有队列。
Rabbitmq支持消息持久化，也就是将数据写在磁盘上。为了数据安全考虑，大多数情况下都会选择持久化。消息队列持久化包括三个部分：

* Exchange持久化，在声明时指定durable >=1.
* Queue持久化，在声明时指定durable => 1.
* 消息持久化，在投递时指定delivery_mode => 2(1是非持久化）。

如果Exchange和Queue都是持久化，那么它们之间的Binding也是持久化的。如果Exchange和Queue两者之间有一个是持久化的，一个是非持久化的，就不允许建立绑定。

#### 具体实现
在微服务架构的系统中，我们通常会使用轻量级的消息代理来构建一个共用的消息主题让系统中所有微服务实例都能连接上来，由于该主题中产生的消息会被所有实例监听和消费，所以我们称它为消息总线。在总线上的各个实例都可以方便地广播一些需要让其他连接在该主题上的实例都知道的消息，例如配置信息的变更或者其他一些管理操作等。  
Spring Cloud Bus可以将分布式系统的节点与轻量级消息代理链接，然后可以实现广播状态更改（例如配置更改）或广播其他管理指令。Spring Cloud Bus就像一个分布式执行器，用于扩展的Spring Boot应用程序，但也可以用作应用程序之间的通信通道。那么这里就涉及到了消息代理，目前流行的消息代理中间件有不少，Spring Cloud Bus支持RabbitMQ和Kafka

```
spring.application.name=rabbitmq-hello
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=sang
spring.rabbitmq.password=123456
```

# 27.Spring Cloud Bus整合RabbitMQ  

# 28.Spring Cloud Bus整合Kafka
此处从[JHipster官网](https://www.jhipster-cn.tech/using-kafka/)来学习

#### background
JHipster支持Kafka：
* 可以使用JHipster配置SpringCloudSteam
* 在application-*.yml文件中添加必要的topic配置信息，并监控Kafka的healthcheck，展示到admin screen中
* 生成DockerCompose配置文件，易于启动kafka server的docker。
* 在微服务环境中，使用docker的情况下，如果一个微服务或者网关使用kafka，子生成器将会生成特定的配置，所有的微服务和网关将会使用该kafka代理来获取所有信息。

#### Tutorial
1. 生成使用kafka的微服务，启动docker-compose -f src/main/docker/kafka.yml up -d
2. 创建model
3. 创建Channels
4. 更改Configuration
5. Producer and Consumer