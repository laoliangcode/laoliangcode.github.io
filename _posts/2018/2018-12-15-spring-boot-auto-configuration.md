---
layout: post
title: Spring Boot核心特性之组件自动装配
category: springboot
tags: [springboot]
keywords: Spring Boot,自动装配
---
​	
    spring boot能够根据依赖的jar包自动配置spring boot的应用，例如： 如果类路径中存在`DispatcherServlet`类，就会自动配置springMvc相关的Bean。spring boot的自动装配来源于spring的装配，功能也是随时spring的不断升级不断完善的，spring boot正是在spring的基础上实现的自动装配。
## spring模式注解装配
### 模式注解介绍
​	模式注解是应用程序中用来标注组件的注解，例如：`@Repository`是spring框架中用来标注数据访问对象（DAO）的注解。`@Component`是用来标注被spring管理的通用的组件，`@Component`标注的类都可以被spring容器扫描到。并且任何标注`@Component`元注解的的注解也能被spring扫描到，比如`@Service`
``` java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Service {
	@AliasFor(annotation = Component.class)
	String value() default "";
}
```
### 装配方式
​	spring中通过配置扫描的包 ，就能扫描到注解的组件，有两种配置的方式：
#### XML配置
通过`context:component-scan`标签的`base-package`属性，配置需要扫描的包
``` xml
  <context:component-scan base-package="com.laoliangcode.service,com.laoliangcode.dao"/>
```
#### 注解方式装配
``` java
  @ComponentScan(basePackages = "com.laoliangcode.service,com.laoliangcode.dao")
```
### 自定义模式注解
​	可以通过在自定义注解上加上元注解的方式，自定义模式注解。例如：`@UserRepository`注解上加上元注解`@Repository`，这样`@UserRepository`也是模式注解了。这是由于注解具有派生性的特点，`@UserRepository`派生至`@Repository`,`@Repository`派生至`@Component`。
``` java
@Repository
public @interface UserRepository {
    String value() default "";
}
```
## spring @Enable模块装配
​	spring3.1开始支持`@Enable`模块装配，所谓模块是指，把具有相同功能的组件组合在一起，引入到项目中。比如`@EnableWebMvc`注解，就是把spring MVC相关的配置引入到项目中，而不需要其他配置，方便使用spring MVC。这种装配方式是通过`@Import`注解引入其他配置类来实现的，`@EnableWebMvc`通过引入`DelegatingWebMvcConfiguration`配置类，实现spring MVC的自动配置。
``` java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```
​	引入的配置类有两种实现形式，一种是直接使用模式注解`@Configuration`的类，另一种是实现`ImportSelector`接口的`selectImports`方法，来引入配置类。
### 注解方式
​	`@EnableWebMvc`就是这种实现方式。
​	下面列举`User`模块的装配来具体说明实现的方式。可以看出`EnableUserConfig`是通过直接导入`UserConfiguration`来装配`User`模块的。
#### `UserConfiguration`配置类
``` java
  @Configuration
  public class UserConfiguration {
      @Bean
      public UserService userService(UserDao userDao){
          return new UserService(userDao);
      @Bean
      public UserDao userDao() {
          return new UserDao();
      }
  }
```
#### `EnableUserConfig`注解
``` java
  @Target({ElementType.TYPE})
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Import(UserConfiguration.class)
  public @interface EnableUserConfig {
  }
```
#### 使用启动类
``` java
  @EnableUserConfig
  public class EnableUserConfigBootstrap {
      public static void main(String[] args) {
          AnnotationConfigApplicationContext context = new
                 AnnotationConfigApplicationContext(EnableUserConfigBootstrap.class);
  		UserService userService = context.getBean("userService", UserService.class);
          System.out.println("EnableUserConfigBootstrap.main" + userService.findBId(1));
          context.close();
      }
  }
```
### `ImportSelector`接口方式
​	spring中的`EnableCaching`就是这种实现方式。
``` java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(CachingConfigurationSelector.class)
public @interface EnableCaching {
}
```
​	下面列举`User`模块的装配来具体说明实现的方式。这种方式是通过`UserConfigurationSelector`来引入`User`的配置类`UserConfiguration`。
#### `UserConfigurationSelector`类用来导入`UserConfiguration`配置
``` JAVA
public class UserConfigurationSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[] {UserConfiguration.class.getName()};
    }
}
```
#### `EnableUserSelector`注解
``` java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(UserConfigurationSelector.class)
public @interface EnableUserSelector {
}
```
#### 使用启动类
``` java
@EnableUserSelector
public class EnableUserSelectorBootstrap {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new
                AnnotationConfigApplicationContext(EnableUserSelectorBootstrap.class);
		UserService userService = context.getBean("userService", UserService.class);
        System.out.println("EnableUserSelectorBootstrap.main" + userService.findBId(1));
        context.close();
    }
}
```
## spring条件装配
​   spring3.1开始，spring引入了`@Profile`注解，可以根据环境`Environment`的不同引入不同的配置。spring4.0开始，`Conditional`注解可以更灵活的根据不同条件引入不同的配置。
### `@Profile`注解方式的条件装配
​	使用`User`模块的不能`dao`装配来说明`@Profile`的条件装配。
#### `UserProfileConfiguration`配置
``` java
@Configuration
public class UserProfileConfiguration {
    @Bean
    public UserServiceForProfile userServiceForProfile(IUserDao userDao) {
        return new UserServiceForProfile(userDao);
    }
    @Bean
    @Profile("mysql")
    public IUserDao userMysqlDao() {
        return new UserMysqlDao();
    }
    @Bean
    @Profile("oracle")
    public IUserDao userOracleDao() {
        return new UserOracleDao();
    }
}
```
#### `Environment`激活使用
通过`context.getEnvironment().setActiveProfiles("oracle");`的方式，来激活不同的`Profile`
``` java
public class ProfileBootstrap {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new
                AnnotationConfigApplicationContext();
        context.register(UserProfileConfiguration.class);
        context.getEnvironment().setActiveProfiles("oracle");
        context.refresh();
        UserServiceForProfile userService = context.getBean("userServiceForProfile",
                UserServiceForProfile.class);
        System.out.println("ProfileBootstrap.main" + userService.findBId(1));
        context.close();
    }
}
```
### `@Conditional`注解方式的条件装配
​	`@Conditional`注解方式，通过引入实现`Condition`接口的类，来判断条件是否成立，从而确定是否引入某个组件。这个类是通过实现`matches`方法来判断条件是否成立。spring4.0开始，由于引入了`@Conditional`注解，`Profile`也是通过`@Conditional`来实现的。
``` java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(ProfileCondition.class)
public @interface Profile {
	String[] value();
}
```
​	spring boot中大量使用了`@Conditional`注解的方式，来自动装配不同的组件。`@ConditionalOnClass`用来表示类路径存在某些类时加载；`@ConditionalOnMissingBean`用来判断某些类的实例不存在时加载；`ConditionalOnWebApplication`用来判断某种应用类型时加载。例如`webmvc`的自动加载配置`WebMvcAutoConfiguration`:
``` java
    @Configuration
    @ConditionalOnWebApplication(type = Type.SERVLET)
    @ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
    @ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
    @AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
    @AutoConfigureAfter({ DispatcherServletAutoConfiguration.class,
    		TaskExecutionAutoConfiguration.class, ValidationAutoConfiguration.class })
    public class WebMvcAutoConfiguration {
    }
```
​	下面的例子是根据系统变量的值来决定是否装配`UserDao`
####  `OnSystemPropertyCondition`  实现`Condition`接口的`matches`方法，用来判断是否符合条件
``` java
  public class OnSystemPropertyCondition implements Condition {
      @Override
      public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
          Map<String, Object> annotationAttributes = metadata.getAnnotationAttributes(ConditionalOnSystemProperty.class.getName());
          String propertyName = String.valueOf(annotationAttributes.get("name"));
          String propertyValue = String.valueOf(annotationAttributes.get("value"));
          String systemPropertyValue = System.getProperty(propertyName);
          return propertyValue.equals(systemPropertyValue);
      }
  }
```
#### `ConditionalOnSystemProperty`注解
``` java
  @Target({ElementType.TYPE, ElementType.METHOD})
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Conditional(OnSystemPropertyCondition.class)
  public @interface ConditionalOnSystemProperty {
      String name();
      String value();
  }
```
####  启动类
``` java
  public class ConditionalOnSystemPropertyBootstrap {
      @ConditionalOnSystemProperty(name = "user.name", value = "Administrator")
      @Bean
      public UserDao userDao() {
          return new UserDao();
      }
      public static void main(String[] args) {
          AnnotationConfigApplicationContext context = new
                  AnnotationConfigApplicationContext(ConditionalOnSystemPropertyBootstrap.class);
          UserDao userDao = context.getBean("userDao", UserDao.class);
          System.out.println("ConditionalOnSystemPropertyBootstrap.main" + userDao.findBId(1));
          context.close();
      }
  }
```
####  如果`value`指定错误，就会报错：
``` java
  Exception in thread "main" org.springframework.beans.factory.NoSuchBeanDefinitionException: No bean named 'userDao' available
```
## spring boot自动装配
​	spring boot的自动装配结合了上面介绍的spring模式注解、@Enable模块装配和条件装配。另外，spring boot自身还使用了工厂加载机制，用`SpringFactoriesLoader`来装载配置类。
### 实现方法
1. 实现自动装配的类
2. 在`META-INF/spring.factories`文件中配置第一步中的类
3. `@EnableAutoConfiguration`注解激活配置
下面以`User`模块的自动装配为例，来介绍具体的实现步骤
#### 实现装配类`UserAutoConfiguration`
这里用到了前面介绍的@Enable模块装配和条件装配
``` java
@EnableUserSelector
@ConditionalOnSystemProperty(name = "user.name", value = "Administrator")
public class UserAutoConfiguration {
}
```
#### 在`META-INF/spring.factories`文件中添加配置
``` properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.laoliangcode.configuration.UserAutoConfiguration
```
#### 启动类中添加`@EnableAutoConfiguration`注解激活配置
``` java
@EnableAutoConfiguration
public class AutoConfigurationBootstrap {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new
                AnnotationConfigApplicationContext(AutoConfigurationBootstrap.class);
        UserService userService = context.getBean("userService", UserService.class);
        System.out.println("AutoConfigurationBootstrap.main" + userService.findBId(1));
        context.close();
    }
}
```
如果在第一步中，让条件装配不符合条件，就会报错：
``` java
Exception in thread "main" org.springframework.beans.factory.NoSuchBeanDefinitionException: No bean named 'userService' available
```
文中详细代码在github上
``` 
https://github.com/laoliangcode/springboot-demo/tree/master/autoconfigure
```
