---
layout: post
title: ribbon执行原理源码初探
category: springcloud
tags: [springcloud]
keywords: springcloud,ribbon
---

    本文从源码层面介绍下原生ribbon以及与springcloud集成后的执行流程。

## 1 原生ribbon执行流程解析
### 1.1 ribbon关键组件
#### 1.1.1 ILoadBalancer
ribbon负载均衡核心组件，管理服务列表`Server`的组件，并从中选择一个`Server`发起服务调用。
接口类关系如下：
![](http://image.laoliangcode.com/springcloud/ribbon/ILoadBalancer.jpg?x-oss-process=style/laoliangcodew)
#### 1.1.2 IRule
`ILoadBalancer`选择服务时所采用的规则组件，定义使用何种方式从服务列表中选择一个服务。
接口类关系如下：
![](http://image.laoliangcode.com/springcloud/ribbon/IRule.jpg?x-oss-process=style/laoliangcodew
)
#### 1.1.3 ServerList
获取服务列表的组件，定义服务列表的获取方式以及更新策略。
接口类关系如下：
![](http://image.laoliangcode.com/springcloud/ribbon/ServerList.jpg?x-oss-process=style/laoliangcodew)
#### 1.1.4 ServerListFilter
服务列表过滤组件，采用`ServerList`获取服务列表后，还可以对服务列表进行过滤，`ServerListFilter`就是定义过滤方式的组件。
接口类关系如下，`ZonePreferenceServerListFilter`是springcloud加的类：
![](http://image.laoliangcode.com/springcloud/ribbon/ServerListFilter.jpg?x-oss-process=style/laoliangcodew)
#### 1.1.5 ServerListUpdater
服务列表更新组件，服务列表并不是一成不变的，当更新的时候，`ServerListUpdater`的职责就是刷新服务列表，并将服务列表更新到`ILoadBalancer`中。
接口类关系如下：
![](http://image.laoliangcode.com/springcloud/ribbon/ServerListUpdater.jpg?x-oss-process=style/laoliangcodew)
#### 1.1.6 IPing
探测服务是否存活的组件
接口类关系如下：
![](http://image.laoliangcode.com/springcloud/ribbon/IPing.jpg?x-oss-process=style/laoliangcodew)
#### 1.1.7 IClientConfig
配置组件，ribbon用到的配置都是从这个组件中获取，内部使用`ConfigurationManager`
使用管理配置。
接口类关系如下,`FeignOptionsClientConfig`是springcloud加的类：
![](http://image.laoliangcode.com/springcloud/ribbon/IClientConfig.jpg?x-oss-process=style/laoliangcodew)
#### 1.1.8 IClient
客户端调用组件，原生ribbon用的客户端调用组件，内部采用jersey实现，跟springcloud集成后，一般使用spring的`RestTemplate`。
接口类关系如下，除了`RestClient`，都是springcloud中类：
![](http://image.laoliangcode.com/springcloud/ribbon/IClient.jpg?x-oss-process=style/laoliangcodew)
### 1.2 原生API使用
下面使用`ZoneAwareLoadBalancer`演示，采用ribbon2.2.5版本，使用使用轮询`RoundRobinRule`的规则选取`Server`。
```
public static void main(String[] args) {
        ZoneAwareLoadBalancer<Server> balancer = new ZoneAwareLoadBalancer<Server>();
        IRule globalRule = new RoundRobinRule();
        balancer.setRule(globalRule);
        List<Server> servers = new ArrayList<Server>();

        servers.add(createServer(1, "a"));
        servers.add(createServer(2, "a"));
        servers.add(createServer(3, "a"));
        balancer.setServersList(servers);
        testChooseServer(balancer);
    }

    private static void testChooseServer(ZoneAwareLoadBalancer<Server> balancer) {
        for (int i = 0; i < 100; i++) {
            Server server = balancer.chooseServer(null);
            String zone = server.getZone().toLowerCase();
            System.out.println(server + ":" + zone);
        }
    }
 }
```
### 1.3 原生ribbon执行流程分析
#### 1.3.1 示例代码
```
    public static void main(String[] args) throws Exception {
        ConfigurationManager.loadPropertiesFromResources("ribbon-demo.properties");
        RestClient client = (RestClient) ClientFactory.getNamedClient("demo");
        HttpRequest request = HttpRequest.newBuilder() .uri("/zl").build();
        for(int i = 0; i < 10; i++) {
            HttpResponse httpResponse = client.executeWithLoadBalancer(request);
            String result = httpResponse.getEntity(String.class);
            System.out.println("RibbonDemo.main:" + result);
        }
    }
```
properties配置文件ribbon-demo.properties
``` properties
demo.ribbon.listOfServers=localhost:8080,localhost:8088
```
#### 1.3.2 代码执行流程分析
上述代码中的ribbon组件都是采用默认的配置，通过`IClientConfig`组件获取，默认的组件类名可以从这个类`DefaultClientConfigImpl`中找到。配置key在配置文件中，需要加上<client-name>.ribbon前缀，其中client-name是变化的，一般是服务名，如实例中的demo。这些组件是使用`ClientFactory`类初始化的，同时这些组件都实现了`IClientConfigAware`接口，初始化的时候会调用`initWithNiwsConfig`方法，从`IClientConfig`中初始化数据。
| 组件接口 | 默认实现类 | 配置key |
| --- | --- | --- |
| `IClient` | `RestClient` | ClientClassName |
| `ILoadBalancer` | `ZoneAwareLoadBalancer` | NFLoadBalancerClassName |
| `IRule` | `AvailabilityFilteringRule` | NFLoadBalancerRuleClassName |
| `ServerList` | `ConfigurationBasedServerList` | NIWSServerListClassName |
| `ServerListFilter` | `ZoneAffinityServerListFilter` | NIWSServerListFilterClassName |
| `ServerListUpdater` | `PollingServerListUpdater` | ServerListUpdaterClassName |
| `IPing` | `DummyPing` | NFLoadBalancerPingClassName |

整体流程如下图：
![](http://image.laoliangcode.com/springcloud/ribbon/ribbon%E5%8E%9F%E7%94%9F%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.jpg?x-oss-process=style/laoliangcode)
## 2 springcloud集成ribbon执行流程分析
### 2.1 ribbon到springcloud的组件迁移
如上所述，原生ribbon各组件的类都是同步`IClientConfig`获取，默认实现类的类名配置在`DefaultClientConfigImpl`。
迁移到springcloud后，为了跟springboot集成，都是把这些组件类配置成了bean。默认情况下，配置类是`RibbonClientConfiguration`。对于客户端调用组件，springcloud默认是使用spring-web工程中的`RestTemplate`，而不是ribbon的`IClient`。
springcloud默认使用的ribbon组件如下：
| 组件接口 | 默认实现类 |
| --- | --- |
| `ILoadBalancer` | `ZoneAwareLoadBalancer` |
| `IRule` | `ZoneAvoidanceRule` |
| `ServerList` | `ConfigurationBasedServerList` |
| `ServerListFilter` | `ZonePreferenceServerListFilter` |
| `ServerListUpdater` | `PollingServerListUpdater` |
| `IPing` | `DummyPing` |

如果需要集成eureka，会从`EurekaRibbonClientConfiguration`配置类加载bean，覆盖默认的bean配置。在这个配置中，`ServerList`和`IPing`有变化，分别使用了`DiscoveryEnabledNIWSServerList`和`NIWSDiscoveryPing`。
### 2.2 springcloud到ribbon的关联
上面说到，springcloud客户端调用使用`RestTemplate`，那`RestTemplate`又是如何调用到ribbon的组件的呢？答案是使用了`RestTemplate`是通过的拦截器功能。
`RestTemplate`是通过`LoadBalancerAutoConfiguration`自动化装配组件装配的。关于springboot的自动化装配，可以参考[Spring Boot核心特性之组件自动装配](https://www.laoliangcode.com/springboot/2018/12/15/spring-boot-auto-configuration.html)。给`RestTemplate`装配的拦截器是`LoadBalancerInterceptor`。在这个拦截器中，会关联一个`LoadBalancerClient`组件`RibbonLoadBalancerClient`，在这个组件中，实现了跟ribbon的关联。
spring cloud自动配置ribbon相关类如下表：
| 配置类名 | 主要功能 | 所在springcloud工程
| --- | --- | --- |
| `LoadBalancerAutoConfiguration` | 主要装配`RestTemplate` 及`LoadBalancerInterceptor` | spring-cloud-commons | 
| `RibbonAutoConfiguration` | 主要装配`LoadBalancerClient` 及`SpringClientFactory` | spring-cloud-netflix-core |
| `RibbonEurekaAutoConfiguration` | 装配eureka跟ribbon集成相关类 | spring-cloud-netflix-eureka-client |
| `RibbonClientConfiguration` | ribbon默认组件配置类 | spring-cloud-netflix-core |
| `EurekaRibbonClientConfiguration` | eureka跟ribbon集成组件配置类 | spring-cloud-netflix-eureka-client |
### 2.3 ribbon集成到springcloud后的流程分析
#### 2.3.1 示例代码
本文使用的springcloud版本是Edgware.SR3。
```
    @Autowired
    private RestTemplate restTemplate;
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    @RequestMapping("B/{name}")
    public String sayHello(@PathVariable("name")  String name) {
        return restTemplate.getForObject("http://serviceA/sayHello/" + name, String.class);
    }
```
#### 2.3.2 执行流程分析
当`RestTemplate`加上`@LoadBalanced`注解后，请求就会由`LoadBalancerInterceptor`将请求引导到ribbon上。
整体流程如下图所示：
![](http://image.laoliangcode.com/springcloud/ribbon/springcloud%2Bribbon%E6%95%B4%E4%BD%93%E6%B5%81%E7%A8%8B.jpg?x-oss-process=style/laoliangcode)
详细执行流程如下图：
![](http://image.laoliangcode.com/springcloud/ribbon/spring%20cloud%2Bribbon%E6%89%A7%E8%A1%8C%E8%AF%A6%E7%BB%86%E6%B5%81%E7%A8%8B.jpg?x-oss-process=style/laoliangcode)
#### 2.3.3 服务列表更新流程
服务列表更新是通过组件`ServerListUpdater`定时拉取，也是使用`PollingServerListUpdater`，与原生的区别在于`ServerList`的不同。springcloud集成eureka后，`ServerList`组件有变化，会通过`EurekaClient`从eurekaServer拉取服务列表。
具体流程如下：
![](http://image.laoliangcode.com/springcloud/ribbon/ribbon%2Beureka%E6%9C%8D%E5%8A%A1%E5%88%97%E8%A1%A8%E6%9B%B4%E6%96%B0%E6%B5%81%E7%A8%8B.jpg?x-oss-process=style/laoliangcode)
#### 2.3.4 IPing组件执行流程
在`ILoadBalancer`的初始化过程中，会启动定时器，定时的检查`Server`是否还存活。原生ribbon默认的是不执行参照，springcloud集成eureka后，会从`EurekaClient`中获取服务实例的状态，从而判断是否存活。
流程如下：
![](http://image.laoliangcode.com/springcloud/ribbon/ribbon%20ping%E6%9C%BA%E5%88%B6.jpg?x-oss-process=style/laoliangcode)
## springcloud自定义ribbon组件
### 自定义Configuration
原生ribbon组件是通过`ClientFactory`初始化，在springcloud中，各种bean要从`ApplicationContext`中获取，而`SpringClientFactory`扮演的就是中间者的角色，负责从spring容器中获取ribbon相关组件。
`SpringClientFactory`会为每个客户端创建一个spring容器，这个容器都会默认加载`RibbonClientConfiguration`配置中的bean，另外自定义的配置需要通过`@RibbonClients`或`@RibbonClient`注解标注，其中`@RibbonClients`的`defaultConfiguration`配置的是默认的配置，对所以client都起作用，但可以覆盖`RibbonClientConfiguration`配置中的bean，比如`RibbonEurekaAutoConfiguration`就是通过这种方式将`EurekaRibbonClientConfiguration`加载为配置类。如果需要为单个client单独配置bean，可以使用`@RibbonClient`注解标注Configuration，这种方式优先级最高，覆盖前面两种配置方式。
`@RibbonClients`和`@RibbonClient`注解是通过`RibbonClientConfigurationRegistrar`类将配置注册为`RibbonClientSpecification`类型的bean，并且会注入到`SpringClientFactory`中，供创建client的spring容器时使用。
### 外部化配置ribbon组件
springcloud为ribbon的组件提供的外部化的配置的方式，即在properties文件中配置所需类名。

* <clientName>.ribbon.NFLoadBalancerClassName: 配置`ILoadBalancer`组件类名
* <clientName>.ribbon.NFLoadBalancerRuleClassName: 配置`IRule`组件类名
* <clientName>.ribbon.NFLoadBalancerPingClassName: 配置`IPing`组件类名
* <clientName>.ribbon.NIWSServerListClassName: 配置`ServerList`组件类名
* <clientName>.ribbon.NIWSServerListFilterClassName: 配置`ServerListFilter`组件类名

`PropertiesFactory`类提供这种配置的实现。在这几种组件初始化之前，都会使用如下代码，优先从配置文件中读取类名，并初始化组件：
```
	if (this.propertiesFactory.isSet(IRule.class, name)) {
		return this.propertiesFactory.get(IRule.class, config, name);
	}
```
本文从源码层面介绍下原生ribbon以及与springcloud集成后的执行流程。
## 1 原生ribbon执行流程解析
### 1.1 ribbon关键组件
#### 1.1.1 ILoadBalancer
ribbon负载均衡核心组件，管理服务列表`Server`的组件，并从中选择一个`Server`发起服务调用。
接口类关系如下：
![](http://image.laoliangcode.com/springcloud/ribbon/ILoadBalancer.jpg?x-oss-process=style/laoliangcodew)
#### 1.1.2 IRule
`ILoadBalancer`选择服务时所采用的规则组件，定义使用何种方式从服务列表中选择一个服务。
接口类关系如下：
![](http://image.laoliangcode.com/springcloud/ribbon/IRule.jpg?x-oss-process=style/laoliangcodew
)
#### 1.1.3 ServerList
获取服务列表的组件，定义服务列表的获取方式以及更新策略。
接口类关系如下：
![](http://image.laoliangcode.com/springcloud/ribbon/ServerList.jpg?x-oss-process=style/laoliangcodew)
#### 1.1.4 ServerListFilter
服务列表过滤组件，采用`ServerList`获取服务列表后，还可以对服务列表进行过滤，`ServerListFilter`就是定义过滤方式的组件。
接口类关系如下，`ZonePreferenceServerListFilter`是springcloud加的类：
![](http://image.laoliangcode.com/springcloud/ribbon/ServerListFilter.jpg?x-oss-process=style/laoliangcodew)
#### 1.1.5 ServerListUpdater
服务列表更新组件，服务列表并不是一成不变的，当更新的时候，`ServerListUpdater`的职责就是刷新服务列表，并将服务列表更新到`ILoadBalancer`中。
接口类关系如下：
![](http://image.laoliangcode.com/springcloud/ribbon/ServerListUpdater.jpg?x-oss-process=style/laoliangcodew)
#### 1.1.6 IPing
探测服务是否存活的组件
接口类关系如下：
![](http://image.laoliangcode.com/springcloud/ribbon/IPing.jpg?x-oss-process=style/laoliangcodew)
#### 1.1.7 IClientConfig
配置组件，ribbon用到的配置都是从这个组件中获取，内部使用`ConfigurationManager`
使用管理配置。
接口类关系如下,`FeignOptionsClientConfig`是springcloud加的类：
![](http://image.laoliangcode.com/springcloud/ribbon/IClientConfig.jpg?x-oss-process=style/laoliangcodew)
#### 1.1.8 IClient
客户端调用组件，原生ribbon用的客户端调用组件，内部采用jersey实现，跟springcloud集成后，一般使用spring的`RestTemplate`。
接口类关系如下，除了`RestClient`，都是springcloud中类：
![](http://image.laoliangcode.com/springcloud/ribbon/IClient.jpg?x-oss-process=style/laoliangcodew)
### 1.2 原生API使用
下面使用`ZoneAwareLoadBalancer`演示，采用ribbon2.2.5版本，使用使用轮询`RoundRobinRule`的规则选取`Server`。
``` java 
public static void main(String[] args) {
        ZoneAwareLoadBalancer<Server> balancer = new ZoneAwareLoadBalancer<Server>();
        IRule globalRule = new RoundRobinRule();
        balancer.setRule(globalRule);
        List<Server> servers = new ArrayList<Server>();

        servers.add(createServer(1, "a"));
        servers.add(createServer(2, "a"));
        servers.add(createServer(3, "a"));
        balancer.setServersList(servers);
        testChooseServer(balancer);
    }

    private static void testChooseServer(ZoneAwareLoadBalancer<Server> balancer) {
        for (int i = 0; i < 100; i++) {
            Server server = balancer.chooseServer(null);
            String zone = server.getZone().toLowerCase();
            System.out.println(server + ":" + zone);
        }
    }
 }
```
### 1.3 原生ribbon执行流程分析
#### 1.3.1 示例代码
``` java 
    public static void main(String[] args) throws Exception {
        ConfigurationManager.loadPropertiesFromResources("ribbon-demo.properties");
        RestClient client = (RestClient) ClientFactory.getNamedClient("demo");
        HttpRequest request = HttpRequest.newBuilder() .uri("/zl").build();
        for(int i = 0; i < 10; i++) {
            HttpResponse httpResponse = client.executeWithLoadBalancer(request);
            String result = httpResponse.getEntity(String.class);
            System.out.println("RibbonDemo.main:" + result);
        }
    }
```
properties配置文件ribbon-demo.properties
``` properties
demo.ribbon.listOfServers=localhost:8080,localhost:8088
```
#### 1.3.2 代码执行流程分析
上述代码中的ribbon组件都是采用默认的配置，通过`IClientConfig`组件获取，默认的组件类名可以从这个类`DefaultClientConfigImpl`中找到。配置key在配置文件中，需要加上<client-name>.ribbon前缀，其中client-name是变化的，一般是服务名，如实例中的demo。这些组件是使用`ClientFactory`类初始化的，同时这些组件都实现了`IClientConfigAware`接口，初始化的时候会调用`initWithNiwsConfig`方法，从`IClientConfig`中初始化数据。

| 组件接口 | 默认实现类 | 配置key |
| --- | --- | --- |
| `IClient` | `RestClient` | ClientClassName |
| `ILoadBalancer` | `ZoneAwareLoadBalancer` | NFLoadBalancerClassName |
| `IRule` | `AvailabilityFilteringRule` | NFLoadBalancerRuleClassName |
| `ServerList` | `ConfigurationBasedServerList` | NIWSServerListClassName |
| `ServerListFilter` | `ZoneAffinityServerListFilter` | NIWSServerListFilterClassName |
| `ServerListUpdater` | `PollingServerListUpdater` | ServerListUpdaterClassName |
| `IPing` | `DummyPing` | NFLoadBalancerPingClassName |

整体流程如下图：
![](http://image.laoliangcode.com/springcloud/ribbon/ribbon%E5%8E%9F%E7%94%9F%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.jpg?x-oss-process=style/laoliangcode)
## 2 springcloud集成ribbon执行流程分析
### 2.1 ribbon到springcloud的组件迁移
如上所述，原生ribbon各组件的类都是同步`IClientConfig`获取，默认实现类的类名配置在`DefaultClientConfigImpl`。
迁移到springcloud后，为了跟springboot集成，都是把这些组件类配置成了bean。默认情况下，配置类是`RibbonClientConfiguration`。对于客户端调用组件，springcloud默认是使用spring-web工程中的`RestTemplate`，而不是ribbon的`IClient`。
springcloud默认使用的ribbon组件如下：

| 组件接口 | 默认实现类 |
| --- | --- |
| `ILoadBalancer` | `ZoneAwareLoadBalancer` |
| `IRule` | `ZoneAvoidanceRule` |
| `ServerList` | `ConfigurationBasedServerList` |
| `ServerListFilter` | `ZonePreferenceServerListFilter` |
| `ServerListUpdater` | `PollingServerListUpdater` |
| `IPing` | `DummyPing` |

如果需要集成eureka，会从`EurekaRibbonClientConfiguration`配置类加载bean，覆盖默认的bean配置。在这个配置中，`ServerList`和`IPing`有变化，分别使用了`DiscoveryEnabledNIWSServerList`和`NIWSDiscoveryPing`。
### 2.2 springcloud到ribbon的关联
上面说到，springcloud客户端调用使用`RestTemplate`，那`RestTemplate`又是如何调用到ribbon的组件的呢？答案是使用了`RestTemplate`是通过的拦截器功能。
`RestTemplate`是通过`LoadBalancerAutoConfiguration`自动化装配组件装配的。关于springboot的自动化装配，可以参考[Spring Boot核心特性之组件自动装配](https://www.laoliangcode.com/springboot/2018/12/15/spring-boot-auto-configuration.html)。给`RestTemplate`装配的拦截器是`LoadBalancerInterceptor`。在这个拦截器中，会关联一个`LoadBalancerClient`组件`RibbonLoadBalancerClient`，在这个组件中，实现了跟ribbon的关联。
spring cloud自动配置ribbon相关类如下表：

| 配置类名 | 主要功能 | 所在springcloud工程
| --- | --- | --- |
| `LoadBalancerAutoConfiguration` | 主要装配`RestTemplate` 及`LoadBalancerInterceptor` | spring-cloud-commons | 
| `RibbonAutoConfiguration` | 主要装配`LoadBalancerClient` 及`SpringClientFactory` | spring-cloud-netflix-core |
| `RibbonEurekaAutoConfiguration` | 装配eureka跟ribbon集成相关类 | spring-cloud-netflix-eureka-client |
| `RibbonClientConfiguration` | ribbon默认组件配置类 | spring-cloud-netflix-core |
| `EurekaRibbonClientConfiguration` | eureka跟ribbon集成组件配置类 | spring-cloud-netflix-eureka-client |

### 2.3 ribbon集成到springcloud后的流程分析
#### 2.3.1 示例代码
本文使用的springcloud版本是Edgware.SR3。
``` java 
    @Autowired
    private RestTemplate restTemplate;
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    @RequestMapping("B/{name}")
    public String sayHello(@PathVariable("name")  String name) {
        return restTemplate.getForObject("http://serviceA/sayHello/" + name, String.class);
    }
```
#### 2.3.2 执行流程分析
当`RestTemplate`加上`@LoadBalanced`注解后，请求就会由`LoadBalancerInterceptor`将请求引导到ribbon上。
整体流程如下图所示：
![](http://image.laoliangcode.com/springcloud/ribbon/springcloud%2Bribbon%E6%95%B4%E4%BD%93%E6%B5%81%E7%A8%8B.jpg?x-oss-process=style/laoliangcode)
详细执行流程如下图：
![](http://image.laoliangcode.com/springcloud/ribbon/spring%20cloud%2Bribbon%E6%89%A7%E8%A1%8C%E8%AF%A6%E7%BB%86%E6%B5%81%E7%A8%8B.jpg?x-oss-process=style/laoliangcode)
#### 2.3.3 服务列表更新流程
服务列表更新是通过组件`ServerListUpdater`定时拉取，也是使用`PollingServerListUpdater`，与原生的区别在于`ServerList`的不同。springcloud集成eureka后，`ServerList`组件有变化，会通过`EurekaClient`从eurekaServer拉取服务列表。
具体流程如下：
![](http://image.laoliangcode.com/springcloud/ribbon/ribbon%2Beureka%E6%9C%8D%E5%8A%A1%E5%88%97%E8%A1%A8%E6%9B%B4%E6%96%B0%E6%B5%81%E7%A8%8B.jpg?x-oss-process=style/laoliangcode)
#### 2.3.4 IPing组件执行流程
在`ILoadBalancer`的初始化过程中，会启动定时器，定时的检查`Server`是否还存活。原生ribbon默认的是不执行参照，springcloud集成eureka后，会从`EurekaClient`中获取服务实例的状态，从而判断是否存活。
流程如下：
![](http://image.laoliangcode.com/springcloud/ribbon/ribbon%20ping%E6%9C%BA%E5%88%B6.jpg?x-oss-process=style/laoliangcode)
## 3 springcloud自定义ribbon组件
### 3.1 自定义Configuration
原生ribbon组件是通过`ClientFactory`初始化，在springcloud中，各种bean要从`ApplicationContext`中获取，而`SpringClientFactory`扮演的就是中间者的角色，负责从spring容器中获取ribbon相关组件。
`SpringClientFactory`会为每个客户端创建一个spring容器，这个容器都会默认加载`RibbonClientConfiguration`配置中的bean，另外自定义的配置需要通过`@RibbonClients`或`@RibbonClient`注解标注，其中`@RibbonClients`的`defaultConfiguration`配置的是默认的配置，对所以client都起作用，但可以覆盖`RibbonClientConfiguration`配置中的bean，比如`RibbonEurekaAutoConfiguration`就是通过这种方式将`EurekaRibbonClientConfiguration`加载为配置类。如果需要为单个client单独配置bean，可以使用`@RibbonClient`注解标注Configuration，这种方式优先级最高，覆盖前面两种配置方式。
`@RibbonClients`和`@RibbonClient`注解是通过`RibbonClientConfigurationRegistrar`类将配置注册为`RibbonClientSpecification`类型的bean，并且会注入到`SpringClientFactory`中，供创建client的spring容器时使用。
### 3.2 外部化配置ribbon组件
springcloud为ribbon的组件提供的外部化的配置的方式，即在properties文件中配置所需类名。

* <clientName>.ribbon.NFLoadBalancerClassName: 配置`ILoadBalancer`组件类名
* <clientName>.ribbon.NFLoadBalancerRuleClassName: 配置`IRule`组件类名
* <clientName>.ribbon.NFLoadBalancerPingClassName: 配置`IPing`组件类名
* <clientName>.ribbon.NIWSServerListClassName: 配置`ServerList`组件类名
* <clientName>.ribbon.NIWSServerListFilterClassName: 配置`ServerListFilter`组件类名

`PropertiesFactory`类提供这种配置的实现。在这几种组件初始化之前，都会使用如下代码，优先从配置文件中读取类名，并初始化组件：
``` java 
	if (this.propertiesFactory.isSet(IRule.class, name)) {
		return this.propertiesFactory.get(IRule.class, config, name);
	}
```