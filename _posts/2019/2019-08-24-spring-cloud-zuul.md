---
layout: post
title: zuul源码分析
category: springcloud
tags: [springcloud]
keywords: springcloud,zuul
---

本文分析了zuul的执行原理，并介绍了跟spring cloud的集成原理和执行过程。

## 1 zuul原生执行原理
### 1.1 zuul核心组件
#### 1.1.1 ZuulServlet/ZuulServletFilter
这两个组件是zuul的执行入口，是原生的javaWeb servlet和filter。用于顺序执行执行zuul的pre、route、post、error过滤器。
#### 1.1.2 FilterLoader/FilterRegistry
FilterLoader用于zuul的filter加载，将加载的filter按保存到FilterRegistry，后续执行时，从中获取到对应filter，并执行。原生的zuul是加载groovy文件，并编译成class再保存。
#### 1.1.3 ZuulFilter
zuul最核心的组件，ZuulServlet中顺序执行的filter就是ZuulFilter，zuul的各种功能也是通过不同的ZuulFilter实现的。两个主要的方法shouldFilter()和run()，分别表示此filter是否可以执行，以及filter的具体执行逻辑。
#### 1.1.4 RequestContext
用于保存zuul执行过程的上下文信息，这些一般可以用于觉得下一个filter是否可以执行。

### 1.2 原生zuul执行流程
zuul-core包中都是zuul执行流程中用到的类，但并没有核心的filter，所有要用zuul实现想要的功能，还需自定义自己的filter。这里以zuul提供的zuul-simple-webapp包作为示例，介绍一下zuul的执行流程。这个包实际上就是一个web程序，并把ZuulServlet配置在了web.xml中，初始化zuul的入口是StartServer，加载了groovy目录的filter。这里用到的zuul版本是1.3.0，下图是原生zuul的执行流程，并没有画出具体filter的执行过程：
![](http://image.laoliangcode.com/springcloud/zuul/zuul%E5%8E%9F%E7%94%9F%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.jpg)

## 2 spring cloud集成zuul执行流程
spring cloud在ZuulProxyAutoConfiguration和ZuulServerAutoConfiguration中自动装配zuul的组件。通过ZuulController包装ZuulServlet来实现zuul的执行入口。filter都是spring cloud自己定义的，其中PreDecorationFilter是pre类型的filter，用来根据配置找出路由规则，而最核心的是route类型的filter RibbonRoutingFilter，在这个filter中，默认集成了hystrix和ribbon，根据配置的服务名，负载均衡找出一台机器，发出http请求。下图是具体的执行流程：
![](http://image.laoliangcode.com/springcloud/zuul/spring%20cloud%E9%9B%86%E6%88%90zuul%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.jpg)