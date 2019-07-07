---
layout: post
title: feign执行原理源码初探
category: springcloud
tags: [springcloud]
keywords: springcloud,feign
---

本文从源码层面介绍一下原生feign及spring cloud集成feign的执行流程。

## 1 原生feign执行流程解析
### 1.1 feign关键组件介绍
feign的组件是通过`Feign.Builder`构造的，在这个构造器中有12个属性，基本构成了feign的核心组件，其中比较关键的有下面几个：

#### 1.1.1 Contract接口
feign的主要功能就是将接口中各种注解，通过解析封装成http请求的URL、参数、body、header等，而这个解析的组件就是`Contract`。feign默认的Contract实现是`Contract.Default`。

#### 1.1.2 Client接口
feign获取到请求的URL后，就需要向服务端发送请求，`Client`就是封装http请求，发送请求、获取结果并解析结果的的组件。feing默认的Client实现是`Client.Default`，通过jdk的`HttpURLConnection`发送请求。

#### 1.1.3 Encoder接口
这是一个将对象编码成http请求体body的组件。默认实现是`Encoder.Default`，这个编码器功能有限，只能编码String和byte[]类型的对象。而我们一般需要编码成json数据，这就需要用到其他的编码器了。比如`JacksonEncoder`。

#### 1.1.4 Decoder
`Decoder`是将请求响应解码成一个具体对象的组件，作用跟`Encoder`相反。Decoder只会解析成功的请求，并且返回值不是void。默认的解码器是`Decoder.Default`，功能也比较简单，只能解码String和byte[]类型。如果需要解析成自己需要的对象，比如将json解析成对象，就需要用到`JacksonDecoder`。

#### 1.1.5 Retryer
Retryer是feign执行失败时的重试组件。默认实现是`Retryer.Default`，默认重试5次，初始间隔100ms，每次间隔会逐渐增加。

#### 1.1.6 Logger及Logger.Level
`Logger`是一个抽象类，定义了一组打印日志的模板，具体的日志有日志框架打印。`Logger.Level`是打印日志的级别，有NONE、BASIC、HEADERS、FULL几种级别。

#### 1.1.7 Options
这是一个配置类，里面有两个配置项，connectTimeoutMillis和readTimeoutMillis，分别表示Client发送请求的连接超时时间和响应超时时间。默认是10s和60s。

#### 1.1.8 其他
`Feign.Builder`中还有其他几个组件，分别介绍如下：

| 组件名 | 默认实现 | 介绍 |
| --- | --- | --- |
| `RequestInterceptor` | 空 |  请求拦截器，可以多个，默认为空 |
| `ErrorDecoder` | `ErrorDecoder.Default` |  响应异常时，分装异常信息 |
| `InvocationHandlerFactory` | `InvocationHandlerFactory.Default` |  生成动态代理的InvocationHandler |
| `decode404` | false |  响应404时，是否解码 |

### 1.2 原生feign生成动态代理过程
本文使用的feign版本是9.5.0。所谓生成动态代理，就是把feign定义请求的接口生成一个代理类，根据接口中方法的注解以及`Contract`组件解析http请求，并封装成`MethodMetadata`对象及成员`RequestTemplate`对象，最后将每个方法封装的`MethodMetadata`对象组装成Map<Method, MethodHandler>，传递给`InvocationHandler`，供执行时使用。feign默认是通过jdk的Proxy生成动态代理。
关于原feign的Contract使用方法可参考[https://github.com/OpenFeign/feign](
https://github.com/OpenFeign/feign)。
下面解析具体过程，所用的实例代码如下：

``` java
private static void testHello() {
	UserService userService = connect();
	Map<String, Object> param = new HashMap<>();
	param.put("name", "zl");
	param.put("age", 14);

	User user = userService.sayHello(1L, param);
	System.out.println("FeignDemo.testHello:" + user);
}

private static UserService connect() {
	UserService userService = Feign.builder().decoder(new JacksonDecoder())
			.target(UserService.class, "http://localhost:8092/user");
	return userService;
}

public interface UserService {

    @RequestLine("GET /hello/{id}?name={name}&age={age}")
    User sayHello(@Param("id") Long id, @Param("name") String name,@Param("age") Integer age);
}
```

整体执行流程图如下：
![](http://image.laoliangcode.com/springcloud/feign/%E5%8E%9F%E7%94%9Ffeign%E7%94%9F%E6%88%90%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E8%BF%87%E7%A8%8B.jpg?x-oss-process=style/laoliangcode)

sayHello方法解析成`MethodMetadata`后的数据如下：

![](http://image.laoliangcode.com/springcloud/feign/MethodMetadata1.jpg?x-oss-process=style/laoliangcodew)
![](http://image.laoliangcode.com/springcloud/feign/MethodMetadata2.jpg?x-oss-process=style/laoliangcodew)

### 1.3 原生feign执行流程
 调用userService.sayHello方法后，执行的入口是`ReflectiveFeign.FeignInvocationHandler`的invoke，从Map<Method, MethodHandler>中找到方法对应的MethodHandler，然后用调用方法所传递的参数替换掉RequestTemplate中的占位符，重新组装RequestTemplate，再组装http请求的url、请求头、body等信息，最后发送http请求。收到相应后，需要对结果解码成对应的对象。
 下面是具体执行流程，实例代码跟1.2一样：
 
 ![](http://image.laoliangcode.com/springcloud/feign/%E5%8E%9F%E7%94%9Ffeign%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.jpg)
 
 重新封装后的RequestTemplate对象的数据如下图所示：
 
 ![](http://image.laoliangcode.com/springcloud/feign/RequestTemplate.jpg)
 
 ### 1.4 原生feign集成ribbon执行流程
feign集成ribbon关键是把Client组件替换成了跟ribbon集成的Client，这个Client名叫`
RibbonClient`，这个类在feign的feign-ribbon工程下。在这个Client中，会创建ribbon的Client LBClient，执行原生的ribbon获取服务地址。获取到服务地址后，最后还是调用feign默认的Client，发送http请求。
具体执行流程如下图所示：

![](http://image.laoliangcode.com/springcloud/feign/%E5%8E%9F%E7%94%9Ffeign%E9%9B%86%E6%88%90ribbon%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.jpg)

## 2 spring cloud集成feign执行流程解析
spring cloud集成feign，其实就是通过spring boot的自动装配机制，将feign的那几个组件交由spring容器管理。spring cloud是通过`FeignAutoConfiguration`自动装配组件装配feign。默认的配置类是`FeignClientsConfiguration`。spring cloud把feign的一些组件默认替换成了自己的类。下表是spring cloud默认使用的feign组件的类名：

| 组件名 | 实现类 | 说明 |
| --- | --- | --- |
| Contract | SpringMvcContract | 实现了跟springmvc注解的无缝结合 |
| Encoder | SpringEncoder | 利用spring的HttpMessageConverters实现编码 |
| Decoder | ResponseEntityDecoder | 内部通过SpringDecoder实现，也是利用spring的HttpMessageConverters |
| Retryer | Retryer.NEVER_RETRY | feign自己的不重试Retryer |
| Logger | Slf4jLogger | 通过FeignLoggerFactory创建logger |
| Client | LoadBalancerFeignClient | 实现了跟ribbon的集成 |

### 2.1 生成动态代理过程
本文使用的spring cloud版本是Edgware.SR3。
生成动态代理的入口是`@EnableFeignClients`注解中FeignClientsRegistrar类，这个类通过扫描`@FeignClient`注解的接口，注册成`FeignClientFactoryBean`类型的bean。FeignClientFactoryBean中getObject方法就实现了生成接口的动态代理。
spring cloud包装了一个Targeter组件，内部还是调用了原生feign用来生成动态代理。只是在生成过程中，用SpringMvcContract来解析接口注解，组装MethodMetadata对象。
具体生成过程如下：

![](http://image.laoliangcode.com/springcloud/feign/spring%20cloud%E9%9B%86%E6%88%90feign%E7%94%9F%E6%88%90%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E8%BF%87%E7%A8%8B.jpg)

### 2.2 spring cloud集成feign执行流程
spring cloud跟feign集成，默认是集成了ribbon，跟原生feign集成ribbon的原理几乎一样，所用到的类跟feign-ribbon工程下类差不多，为了跟spring容器结合，spring cloud基本上是把一些关键类复制了一份，改了一下名字，并把里面用到一些组件从spring容器中去获取，下表是这些类的对应关系：

| feign原生类名 | spring cloud类名 | 说明 |
| --- | --- | --- |
| RibbonClient | LoadBalancerFeignClient | feign client组件 |
| LBClient | FeignLoadBalancer | ribbon client组件 |
| LBClientFactory | CachingSpringLoadBalancerFactory | 创建ribbon client组件的工厂类 |

下图是具体执行流程：
![](http://image.laoliangcode.com/springcloud/feign/spring%20cloud%E9%9B%86%E6%88%90feign%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.jpg)

### 2.3 spring cloud配置自定义的feign组件
#### 2.3.1 通过Configuration方式配置
跟spring cloud集成ribbon的机制一样，通过FeignContext从spring容器中获取各种组件，并且为每个客户端都创建了一个单独的ApplicationContext。可以通过`@EnableFeignClients`注解中的defaultConfiguration，配置自己所有客户端的默认配置，同时也可以通过`@FeignClient`注解的configuration，配置每个客户端单独的配置，这些配置也是在FeignClientsRegistrar中，将配置以FeignClientSpecification类型注册到spring容器中。
#### 2.3.1 通过外部化方式配置
spring cloud在通过Feign.Builder配置feign的各种组件的过程中，会读取`FeignClientProperties`中的配置，可以通过这种方式配置这几种组件：Logger.LevelRetryer、ErrorDecoder、Request.Options、RequestInterceptor、decode404。