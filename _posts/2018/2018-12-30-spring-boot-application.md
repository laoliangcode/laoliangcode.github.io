---
layout: post
title: Spring Boot application启动过程
category: springboot
tags: [springboot]
keywords: Spring Boot,application
---
​
   spring boot中的 `SpringApplication`类可以方便的通过main方法启动spring应用程序。spring boot的主要启动过程是在 `SpringApplication`类中的静态方法run中完成的。

## 1 SpringApplication基本使用
### 1.1 SpringApplication启动
  spring应用程序启动非常简单，直接调用`SpringApplication`类中的run方法就行。其中第一个参数是spring的配置源信息，是`@Configuration`注解的类。
``` java
@SpringBootApplication
public class SpringApplicationBootstrap {
    public static void main(String[] args) {
        SpringApplication.run(SpringApplicationBootstrap.class, args);
    }
}
```
### 1.2 自定义SpringApplication
#### 1.2.1 SpringApplication API
  SpringApplication除了可以直接调用静态方法启动，也可以new一个SpringApplication对象，并对这个对象进行一些自定义的设置，比如可以取消打印Banner、自定义配置源。这里的配置源信息，可以是类名，也可以是包名和XML文件地址。
``` java
@SpringBootApplication
public class SpringApplicationCustomizeBootstrap {
    public static void main(String[] args) {
        SpringApplication springApplication = new SpringApplication();
        Set<String> sources = new HashSet<>();
        sources.add(SpringApplicationCustomizeBootstrap.class.getName());
        springApplication.setSources(sources);
        springApplication.setBannerMode(Banner.Mode.OFF);
        springApplication.run(args);
    }
}
```
#### 1.2.2 SpringApplicationBuilder流式API
  SpringApplicationBuilder类提供了一套流式API来设置SpringApplication。还可以用来构建具有父子关系的应用上下文。
``` java
public class SpringApplicationBuilderCustomizeBootstrap {
    public static void main(String[] args) {
        new SpringApplicationBuilder(SpringApplicationBuilderCustomizeBootstrap.class)
                .web(WebApplicationType.NONE)
                .bannerMode(Banner.Mode.OFF)
                .run(args);
    }
}
```
## 2 SpringApplication准备阶段
### 2.1 推断web应用类型
  spring boot是根据类路径中知否存在相关类来推断web应用类型的，包括三种应用类型：
* REACTIVE：`WebApplicationType.REACTIVE`
* SERVLET: `WebApplicationType.SERVLET`
* NONE: `WebApplicationType.NONE`

推断过程是这样的：先推断是否是存在REACTIVE相关类，如果存在并且不存在webmvc相关类，则是REACTIVE类型；再判断是否存在SERVLET。下面是具体的推断过程，WebApplicationType#deduceFromClasspath：
``` java
static WebApplicationType deduceFromClasspath() {
		if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
				&& !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
				&& !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
			return WebApplicationType.REACTIVE;
		}
		for (String className : SERVLET_INDICATOR_CLASSES) {
			if (!ClassUtils.isPresent(className, null)) {
				return WebApplicationType.NONE;
			}
		}
		return WebApplicationType.SERVLET;
	}
```
### 2.2 加载应用上下文初始化器
 spring boot是利用spring的工厂加载机制，来加载`ApplicationContextInitializer`类，并对加载的类进行排序。具体加载类路径下的META-INF/spring.factories文件。加载过程如下：
``` java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = getClassLoader();
		// Use names and ensure unique to protect against duplicates
		Set<String> names = new LinkedHashSet<>(
				SpringFactoriesLoader.loadFactoryNames(type, classLoader));
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
				classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}
```
### 2.3 加载应用事件监听器
 加载过程跟加载上下文初始化器原理一样，是加载`ApplicationListener`类。
### 2.4 推断主类
  spring boot是根据main线程的执行堆栈来推断主类的。是通过递归一个的异常的堆栈信息，找到main方法，从而推断出主类，具体过程如下：
``` java
private Class<?> deduceMainApplicationClass() {
		try {
			StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
			for (StackTraceElement stackTraceElement : stackTrace) {
				if ("main".equals(stackTraceElement.getMethodName())) {
					return Class.forName(stackTraceElement.getClassName());
				}
			}
		}
		catch (ClassNotFoundException ex) {
			// Swallow and continue
		}
		return null;
	}
```
## 3 SpringApplication运行阶段
### 3.1 加载运行监听器
  加载原理跟加载应用事件监听器一样，加载的是`SpringApplicationRunListener`类。spring boot是通过`SpringApplicationRunListeners`类来运行运行监听器`SpringApplicationRunListener`，运行监听器的作用就是发布具体的事件。运行监听器中执行的方法以及发布的事件类型如下：
* starting：发布的是`ApplicationStartingEvent`事件，这个事件是在应用刚启动的发布的。这个方法和事件是spring boot1.5有的。
* environmentPrepared：发布的是`ApplicationEnvironmentPreparedEvent`事件，这个时候`Environment`对象刚创建好还没开始使用，并且应用上下文还没创建。这个方法和事件是spring boot1.0开始就有了。
* 