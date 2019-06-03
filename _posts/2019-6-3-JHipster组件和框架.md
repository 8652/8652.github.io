---
layout:     post   				    # 使用的布局（不需要改）
title:      JHipster组件和框架			# 标题 
subtitle:   简单介绍、配置、用法 #副标题
date:       2019-6-3 				# 时间
author:     Tempo					# 作者
header-img: img/post-bg-map.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - JHipster
---

# JHipster组件和框架

服务注册和发现  
* Eureka
* Consul

统一配置  
* Spring Cloud Config
* Disconf
* Apollo

统一登录  
* UAA
* JWT

API网关  
* Zuul
* Traefik

日志监控  
* ELK

服务跟踪  
* Zipkin

服务降级  
* Hystrix

缓存
* Hazelcast
* EHcache
* Redis

数据库连接池
* Hikari

数据库版本管理
* Liquibase

持久层框架
* Spring Data JPA
* MyBatis

消息队列
* Kafka
* RabbitMQ

## 服务注册和发现  

#### Eureka
Eureka Server 作为服务注册功能的服务器，它是服务注册中心。而系统中的其他微服务，使用 Eureka 的客户端连接到 Eureka Server，并维持心跳连接。这样系统的维护人员就可以通过 Eureka Server 来监控系统中各个微服务是否正常运行。Spring Cloud 的一些其他模块（比如Zuul）就可以通过 Eureka Server 来发现系统中的其他微服务，并执行相关的逻辑。  

[对eureka服务加密](https://cloud.tencent.com/developer/article/1432515)
#### Consul
[springboot中的consul](https://blog.csdn.net/hxpjava1/article/details/78302700)
[jhipster consul](https://www.jhipster.tech/cn/consul/)

## 统一配置

#### Spring Cloud Config
Spring Cloud Config为分布式系统中的外部配置提供服务器和客户端支持。使用Config Server，您可以在所有环境中管理应用程序的外部属性。客户端和服务器上的概念映射与Spring Environment和PropertySource抽象相同，因此它们与Spring应用程序非常契合，但可以与任何以任何语言运行的应用程序一起使用。随着应用程序通过从开发人员到测试和生产的部署流程，您可以管理这些环境之间的配置，并确定应用程序具有迁移时需要运行的一切。服务器存储后端的默认实现使用git，因此它轻松支持标签版本的配置环境，以及可以访问用于管理内容的各种工具。可以轻松添加替代实现，并使用Spring配置将其插入。
[官方wiki](https://springcloud.cc/spring-cloud-config.html)
[科普SpringCloudConfig](https://blog.coding.net/blog/spring-cloud-config)

#### Disconf

没有搜索到在JHipster中使用Disconf的实例
[Disconf官方wiki](https://disconf.readthedocs.io/zh_CN/latest/install/index.html)
[Disconf开发实例](https://blog.csdn.net/u011116672/article/details/73412242)

## 统一登录  
#### UAA
[Jhipster UAA](https://www.jhipster.tech/cn/using-uaa/)
使用 Eureka, Ribbon, Hystrix 和 Feign
#### JWT
无需特别配置，只要在JDL中声明即可

## API网关  
#### Zuul

#### Traefik

日志监控  
* ELK

服务跟踪  
* Zipkin

服务降级  
* Hystrix

缓存
* Hazelcast
* EHcache
* Redis

数据库连接池
* Hikari

数据库版本管理
* Liquibase

持久层框架
* Spring Data JPA
* MyBatis

消息队列
* Kafka
* RabbitMQ

# Spring中的具体配置文件

## bootstrap.yml

Bootstrap.yml（bootstrap.properties）在application.yml（application.properties）之前加载，就像application.yml一样，但是用于应用程序上下文的引导阶段。它通常用于“使用Spring Cloud Config Server时，应在bootstrap.yml中指定spring.application.name和spring.cloud.config.server.git.uri”以及一些加密/解密信息。技术上，bootstrap.yml由父Spring ApplicationContext加载。父ApplicationContext被加载到使用application.yml的之前。

例如，当使用Spring Cloud时，通常从服务器加载“real”配置数据。为了获取URL（和其他连接配置，如密码等），您需要一个较早的或“bootstrap”配置。因此，您将配置服务器属性放在bootstrap.yml中，该属性用于加载实际配置数据（通常覆盖application.yml [如果存在]中的内容）。

当然，在一些情况上不用那么区分这两个文件，你只需要使用application文件即可，把全部选项都写在这里，效果基本是一致的，在不考虑上面的加载顺序覆盖的问题上。

bootstrap.yml 或者 bootstrap.properties
如果你使用的是Spring Cloud，并且能的应用程序的配置存储在远程配置服务器（例如Spring Cloud Config Server）中，则仅使用/需要它。

从文档：

Spring Cloud应用程序通过创建“引导”上下文来运行，该上下文是主应用程序的父上下文。开箱即用，它负责从外部源加载配置属性，并且还解密本地外部配置文件中的属性。

请注意，bootstrap.ymlor bootstrap.properties 可以包含其他配置（例如默认值），但通常您只需将引导配置放在此处即可。

通常它包含两个属性：

配置服务器的位置（spring.cloud.config.uri）
应用程序的名称（spring.application.name）
启动后，Spring Cloud使用应用程序的名称对配置服务器进行HTTP调用，并检索该应用程序的配置。

application.yml 或者 application.properties
包含标准应用程序配置 - 通常是默认配置，因为在引导过程中检索到的任何配置将覆盖此处定义的配置。

首先，bootstrap.yml作为配置文件，是在springcloud中实现的，而不是springboot

在springcloud中，使用bootstrap首先加载一些配置，这部分是高优先级不会被后续覆盖的。

通常是用做加载配置中心配置。不要在这里配置其他属性，会出现很诡异的事。(我测了，配置0000，得出0)

所以就乖乖的按照springcloud的建议来。

最后，bootstrap.yml作为配置文件，是springcloud中的定义

********
个人疑惑
看到大部分配置文件都是-dev结尾（开发模式），需不需要注明部署阶段的配置，还是注明默认sc config？
需要统一配置标准吗，比如eureka和候选consul，是统一制定使用eureka还是两种方案都提供（感觉会好一点）
看起来像是写给公司内部的实践流程（而不是一体机合作伙伴，那么为什么不写部署？）
不同职责的microservice，需要定义的配置不同。比如gateway和普通microservice
uaa的定义是必须的吗，它是对用户端的user管理，还是对服务调用之间的管理
实践实例都是，连接XXXX，无考虑到部署XXXX（是只针对一个普通微服务写的吗？）
之后可以先写一些，比如spring security的配置，然后给导师看一下，同时提出疑问

可以写出md的部分
连接到consul
一小部分部署，如gateway uaa
连接到uaa（spring security）