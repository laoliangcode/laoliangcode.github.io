---
layout: post
title: hystrix原理源码解析
category: springcloud
tags: [springcloud]
keywords: springcloud,hystrix
---

本文从源码角度介绍了hystrix的原生执行原理，以及跟spring cloud集成的执行原理。

## 1 原生hystrix执行原理
### 1.1 hystrix核心组件
#### 1.1.1 配置体系
##### 1.1.1.1 HystrixCommandKey
一个HystrixCommand都有一个HystrixCommandKey，hystrix的监控、熔断等功能都是以这个key为基础的，每个key都对应着自己的配置。一般对应一个服务里的一个方法。

##### 1.1.1.2 HystrixCommandGroupKey
一组相关联的HystrixCommand可以组成一个group，用HystrixCommandGroupKey来配置，也是HystrixThreadPoolKey的默认配置。一般把一个服务配置成一个group。

##### 1.1.1.3 HystrixThreadPoolKey
一个key对应着hystrix中的一个线程池HystrixThreadPool。默认取HystrixCommandGroupKey得值。

##### 1.1.1.4 HystrixCommandProperties
读取和设置HystrixCommand配置的配置类，里面包含有各种配置的默认值，通过HystrixCommandProperties.Setter设置各种配置。如果没有设置从配置文件中读取，配置key的格式是hystrix.command.[CommandKey或default].[配置名]。

##### 1.1.1.5 HystrixThreadPoolProperties
读取和设置线程池相关的配置，跟HystrixCommandProperties类似，通过HystrixThreadPoolProperties.Setter设置值，配置key的格式是hystrix.threadpool.[ThreadPoolKey或default].[配置名]。

##### 1.1.1.6 HystrixCommand.Setter
hystrix的总配置入口，包含了上面的所有配置，一般在初始化HystrixCommand的时候，通过往构造器中传入这个Setter来完成初始化。

#### 1.1.2 线程池HystrixThreadPool
封装了JUC的线程池，对线程池进行统一管理，包括任务提交、队列限制等。

#### 1.1.3 熔断器HystrixCircuitBreaker
处理熔断相关的逻辑，判断是否处于熔断状态，如果处于熔断状态，则拒绝请求，达到一定窗口时间，可以让熔断器处于半开状态，一共有三种状态：CLOSED, OPEN, HALF_OPEN。

#### 1.1.4 统计组件HystrixMetrics
用于统计HystrixCommand的调用，包括成功、超时、异常、线程池队列/semaphore满、短路器打开等。包括HystrixCommandMetrics、HystrixThreadPoolMetrics、HystrixCollapserMetrics。

#### 1.1.5 HystrixCommand
hystrix的核心组件，初始化、执行、限流、熔断等都是在这个组件中完成的，在run()方法中执行具体业务逻辑。hystrix还有其他相关的的Command，如下图所示。其中HystrixObservableCommand可以自定义Observable，在construct()方法中写业务逻辑，适应于获取多条数据的情况，RibbonCommand是spring cloud封装zuul相关的command，HttpResourceObservableCommand和CacheObservableCommand是ribbon中的类。
![](http://image.laoliangcode.com/springcloud/hystrix/Hystrix.jpg)

### 1.2 HystrixCommand执行流程
 hystrix依赖rxjava这个框架，不是顺序执行的，依赖创建Observable，并通过onNext()、onCompleted()、onError()的回调来完成整个执行流程。本文的hystrix版本是1.5.12。
 示例代码如下：
 ``` java
private String name;
public FirstDemo(String name) {
//        super(HystrixCommandGroupKey.Factory.asKey("firstDemo"));
super(HystrixCommand.Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("firstDemo")).andCommandPropertiesDefaults(
	HystrixCommandProperties.Setter().withExecutionTimeoutInMilliseconds(3600000)));
	this.name = name;
}
@Override
protected String run() throws Exception {
	System.out.println("FirstDemo.run");
	return "Hello:" + name;
}

public static void main(String[] args) {
	FirstDemo demo = new FirstDemo("zl");
  String ret = demo.execute();
	System.out.println(ret);
}
```

 整个执行流程如下图所示：
 ![](http://image.laoliangcode.com/springcloud/hystrix/%E5%8E%9F%E7%94%9Fhystrix%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.jpg)
 
 ### 1.3 @HystrixCommand注解方式执行流程
 hystrix还支持注解的方式，通过在方法上加上@HystrixCommand注解，aspectj框架就会通过aop来拦截请求，执行入口类是HystrixCommandAspect，这个类在hystrix的hystrix-javanica包中。其中定义两个Pointcut，分别用于两张功能。
 ``` java
@Pointcut("@annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand)")
public void hystrixCommandAnnotationPointcut() {
}
@Pointcut("@annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCollapser)")
public void hystrixCollapserAnnotationPointcut() {
}
```
其中HystrixCommand的入口方式是methodsAnnotatedWithHystrixCommand，通过注解的配置信息，初始化不同的command，然后按command的逻辑继续执行。
整体执行流程如下如所示：
![](http://image.laoliangcode.com/springcloud/hystrix/@HystrixCommand%E6%B3%A8%E8%A7%A3%E6%96%B9%E5%BC%8F%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.jpg)

## 2 feign集成hystrix执行流程
feign集成hystrix依赖feign的feign-hystrix包，这个包通过定义hystrix专属的feign组件，目的是完成HystrixCommand的初始化。主要替换的feign组件有这几个：Builder使用HystrixFeign.Builder、InvocationHandler使用HystrixInvocationHandler、对Contract在外面一层HystrixDelegatingContract。HystrixFeign.Builder中多了一个SetterFactory属性，用于配置hystrix，HystrixInvocationHandler中定义了FallbackFactory<?> fallbackFactory、Map<Method, Method> fallbackMethodMap、Map<Method, Setter> setterMethodMap，分别用于每个方法的降级方法及配置。
整体执行流程如下：
![](http://image.laoliangcode.com/springcloud/hystrix/feign%E9%9B%86%E6%88%90hystrix%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.jpg)

## 3 spring cloud集成hystrix
### 3.1 @HystrixCommand注解方式
跟原生的hystrix原理的执行流程一样，入口在HystrixCircuitBreakerConfiguration的hystrixCommandAspectbean，也是初始了一个HystrixCommandAspect。

### 3.2 通过feign集成hystrix
spring cloud通过feign接触hystrix依赖feign集成hystrix的feign-hystrix包。在FeignClientsConfiguration中，如果feign.hystrix.enabled配置为true，Feign.Builder
就会使用HystrixFeign.builder()，同时如果存在feign.hystrix.HystrixFeign类，FeignAutoConfiguration就会将Targeter初始化为HystrixTargeter，从而让feign集成hystrix功能。
流程如下图所示：
![](http://image.laoliangcode.com/springcloud/hystrix/spring%20cloud%E4%B8%ADfeign%E9%9B%86%E6%88%90hystrix%E6%89%A7%E8%A1%8C%E9%80%BB%E8%BE%91.jpg)

### 3.3 spring cloud外部化配置hystrix
hystrix的配置依赖了netflix的配置框架archaius-core，用于加载、缓存、动态刷新配置，spring cloud中的ArchaiusAutoConfiguration就是用于支持archaius的，定了一个自己的`ConfigurableEnvironmentConfiguration`，用于archaius加载spring cloud环境的配置文件，所有我们在application.yml中配置hystrix的各种配置，key跟hystrix定义的一样。Netflix的其中组件，比如euraka、ribbon也是依赖archaius-core管理的配置的，所以也能在application.yml中配置。