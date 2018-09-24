---
layout: post
title: Spring Boot(1)： 快速入门
category: springboot
tags: [springboot]
keywords: Spring Boot
---

spring boot快速入门

## Spring Boot 简介

Spring Boot非常简单易用，我们只需要很少的配置，就可以使用Spring Boot快速的搭建基于Spring的生产级程序。Spring Boot提供了一种特殊的方式管理第三方依赖包，当我们需要使用别的框架时，只需要使用Maven依赖进来，就自动添加了需要的功能。
我们使用Spring Boot创建的程序可以直接`java -jar`启动，也可以部署war包启动，也可以通过提供的命令行工具运行`spring scripts`来启动。

## 第一个Spring Boot程序
Spring Boot跟别的java标准程序一样，只需要在classpath下提供springboot的jar包，就可以使用，我们不需要依赖特殊的工具集成，可以通过IDE，也可以通过文本工具直接开发。我们可以直接拷贝springboot的jar包来创建程序，但有更先进的工具Maven可以使用，可以帮助我们管理依赖。
Spring Boot需要Maven 3.2及以上版本，如果还没有安装的，可以参考[官方网站](http://maven.apache.org/).

Spring Boot使用的Maven groupId是`org.springframework.boot`，为了更方便的使用springboot，通常使用springboot的parent项目`spring-boot-starter-parent`，因为这个parent为我们定义很多dependencies以及springboot众多的Starters，还定义了包括springboot本身和第三方依赖的版本号，我们只需要在parent指定springboot的版本，就不要再去关心springboot所依赖的其他各种jar包的版本了。
springboot还提供了一个Maven插件，可以用来生成可执行的jar包。

接下来正式开始我们得第一个springboot，在一个目录下创建一个pom.xml文件，内容如下：

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.laoliangcode</groupId>
    <artifactId>springboot-demo-first</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <!-- 使用springboot的parent依赖 -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.16.RELEASE</version>
    </parent>
    <!-- 添加springboot的web starter依赖 -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
    <!-- 生成可执行jar包的maven插件 -->
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

接下来创建一个springboot需要的入口程序，
- 创建Maven项目的结构目录`src/main/java/com/laoliangcode`
- 创建一个java文件`Application.java`
代码如下：

``` java
package com.laoliangcode;

import org.springframework.boot.*;
import org.springframework.boot.autoconfigure.*;
import org.springframework.stereotype.*;
import org.springframework.web.bind.annotation.*;

@RestController
@EnableAutoConfiguration
public class Application {
    @RequestMapping("/")
    String home() {
        return "Hello World!";
    }
    public static void main(String[] args) throws Exception {
        SpringApplication.run(Application.class, args);
    }
}
```

接下来使用maven打包，`cd 你创建的目录`，输入命令：

```
mvn package
```

这样会在`target`目录下生成一个jar包`springboot-demo-first-0.0.1-SNAPSHOT.jar`
使用命令直接运行程序

```
java -jar springboot-demo-first-0.0.1-SNAPSHOT.jar
```

在浏览器中输入网址http://localhost:8080/，就可以访问到我们得第一个springboot程序了，是不是超级简单。


