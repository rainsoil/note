# 深入浅出SpringBoot核心原理

## 1. SpringBoot 的基本认识

不管是不是Spring Cloud Netflix  还是Spring Cloud Alibaba, 都是基于SpringBoot 这个框架来构建的.

### 1. 什么是SpringBoot

对于Sping框架,我们接触的比较多的应该是Spring MVC和Spring. 而Spring 的核心在于IOC(控制反转)和ID(依赖注入). 而这些框架在使用的过程中需要配置大量的XML，或者需要做很多繁琐的配置. 

 SpringBoot 框架是为了能够帮助使用Spring框架的开发者快速高效的构建一个基于Spring 框架以及Spring生态体系的应用解决方案. 它是对"约定优于配置" 这个理念下的一个最佳实践. 因为它是一个服务于框架的框架, 服务的范围是简化配置文件. 

#### 约定优于配置的体现

约定优于配置的体现主要是:

1. Maven 的目录结构
   1. 默认有resources 文件夹存放配置文件
   2. 默认打包方式为jar
2. spring-boot-starter-web 中默认包含Spring MVC 相关的依赖以及内置的tomcat 容器,使得构建一个web应用更加简单
3. 默认提供 `application.properties/yml` 配置文件
4. 默认通过` spring.profiles.active ` 属性来解决运行环境时读取的配置文件
5.  EnableAutoConfiguration  默认对于依赖的starter 进行自动装载.

####  从 SpringBootApplication 注解入手

为了解开SpringBoot 的奥秘,我们直接从Annotation 入手, 看看@SpringBootApplication 里面, 做了什么呢? 打开SpringBootApplication 这个注解, 可以看到它实际上是一个复合注解. 

```java

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
@ConfigurationPropertiesScan
public @interface SpringBootApplication {
    @AliasFor(
        annotation = EnableAutoConfiguration.class
    )
    Class<?>[] exclude() default {};

    @AliasFor(
        annotation = EnableAutoConfiguration.class
    )
    String[] excludeName() default {};

    @AliasFor(
        annotation = ComponentScan.class,
        attribute = "basePackages"
    )
    String[] scanBasePackages() default {};

    @AliasFor(
        annotation = ComponentScan.class,
        attribute = "basePackageClasses"
    )
    Class<?>[] scanBasePackageClasses() default {};

    @AliasFor(
        annotation = Configuration.class
    )
    boolean proxyBeanMethods() default true;
}

```

 SpringBootApplication  本质上是由三个注解组成, 分别是:

- @Configuration

- @EnableAutoConfiguration

-  @ComponentScan 

  我们可以直接使用这三个注解也可以启动SpringBoot 应用,只是每次配置这三个注解比较繁琐, 所以直接用一个复合注解更方便些. 

  然后仔细观察这三个注解, 除了 @EnableAutoConfiguration 可以稍微陌生些, 其他两个注解使用的都比较多. 

#####  简单分析@Configuration

@Configuration 这个注解大家应该有用到过, 它是JavaConfig 形式的基于Spring IOC 容器的配置类使用的一种注解. 因为SpringBoot 本质上就是一个Spring 应用, 所以通过这个注解来加载IOC 容器的配置是很正常的. 所以在启动类里面标注了@Configuration,意味着它其实也是一个IOC容器里面的配置类. 

传统意义上的Spring 应用 都是基于XML 形式来配置bean 的依赖关系, 然后通过Spring 应用在启动的时候, 把Bean 进行初始化并且如果bean 还存在依赖关系,则分析这些已经在IOC容器中的bean 根据依赖关系进行组装. 直到Java 5中, 引入了Annotation 这个特性, Spring 框架也进随大流并且推出了基于Java 代码和Annotation 元信息的依赖关系绑定描述的方式. 也就是JavaConfig. 从Spring3开始,Spring 就支持了两种Bean 的配置方式, 一种下基于XML的方式, 另一种就是JavaConfig. 任何一个标注了@Configuration 的Java 类定义都是一个JavaConfig 配置类. 而在这个配置类中, 任何标注了Bean 注解的方法, 他的返回值都会作为Bean 定义注册到Spring 的IOC容器中, 方法名默认为这个bean 的id

##### 简单分析 ComponentScan

ComponentScan  这个注解是大家接触的最多的, 相当于xml配置中的 `<context:component-scan>`, 他的主要作用就是扫描指定路径下的标识了需要装配的类, 自动装配到Spring 的IOC容器中。

标识需要装配的类的主要形式是: @Component、@Repositiory、@Service、@Controller 这类的注解标识类. 

ComponentScan 默认会扫描当前 package 下的所有加了相关注解标识的类到IOC 容器中. 

#####  简单分析  EnableAutoConfiguration

###### Enable 并不是新鲜玩意儿

仍然是在Spring 3.1 版本中, 提供了一系列的@Enable 开头的注解，Enable 注解应该是在JavaConfig 框架上的更进一步的完善, 使得用户在使用Spring 相关的框架的时候, 避免配置大量的代码从而降低使用的难度. 

比如常见的一些Enable 注解: 

- EnableWebMvc 这个注解是引入MVC 框架在Spring 应用中需要用到的所有bean
- EnableScheduling 开启计划任务的支持

找到EnableAutoConfiguration , 我们可以看到每一个涉及到Enable 开头的注解,都会带有一个 @Import 的注解, 

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}

```

###### Import 注解

Import 注解是什么意思呢? 联想到XML形式下有一个<import resource/>    形式的注解, 就明白它的作用了. Import  就是把多个分开的容器配置合并在一起，在JavaConfig 中所表达的意义是一样的. 

#####  深入分析EnableAutoConfiguration 

EnableAutoConfiguration  的主要作用其实是帮助SpringBoot 应用所有符合条件的@Configuration 配置都加载到当前SpringBoot 创建并使用的IOC容器中, 再回到EnableAutoConfiguration  这个注解中,我们发现它的Import 是这样

```java
@Import({AutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
```

######  AutoConfigurationImportSelector 是什么呢? 

Enable 注解不仅仅可以实现多个Configuration 的整合, 还可以实现一些复杂的场景，比如可以根据上下文来激活不同类型的bean, @Import 注解可以配置三种不同的class

1. 第一种 基于普通的bean 或者带有@Configuration 的bean 进行注入
2. 实现ImportSelector 接口进行动态注入
3. 实现ImportBeanDefinitionRegistrar 接口进行动态注入

#### @EnableAutoConfiguration  注解的实现原理

了解 ImportSelector  和ImportBeanDefinitionRegistrar  之后, 对于EnableAutoConfiguration   的理解就容易一些了, 他会通过Import 导入第三方提供的Bean 的配置类. 

```java
@Import({AutoConfigurationImportSelector.class})

```

从名字来看, 可以猜到它的基于 ImportSelector   来实现基于Bean 的动态加载功能, SpringBoot 的@Enable* 注解的工作原理. 实现ImportSelector 接口selectImports 返回的数组(类的全名) 都会被纳入到Spring 容器中. 

那么可以猜想到这里的实现原理也是一样的, 定位到AutoConfigurationImportSelector 这个类中的selectImports  方法,本质上来说, 其实EnableAutoConfiguration 会帮助SpringBoot 应用把所有符合@Configuration 配置都加载到当前SpringBoot 创建的IOC 容器, 而这里面借助了Spring 框架提供了一个工具类SpringFactoriesLoader的支持，以及用到了Spring 提供了条件注解@Conditional,选择性的针对需要加载的bean 进行条件过滤. 

#### SpringFactoriesLoader

这里简单分析一下SpringFactoriesLoader 这个工具类的使用,  他其实和Java 中的SPI 机制的原理是一样的. 不过它比SPI更好的点在于不会一次性的加载所有的类, 而是根据key 进行加载. 

首先,SpringFactoriesLoader的作用是从classpath/META-INF/spring.factories 文件夹中根据key来加载对应的类到Spring IOC 容器中. 

#### 深入理解过滤条件

在分析 AutoConfigurationImportSelector 的源码时, 会先扫描spring-autoconfiguration-metadata.properties  文件, 最后再扫描spring.factories 对应的类, 会结合前面的元数据进行过滤. 为什么要进行过滤? 原因是很多的@Configuration  其实是依托其他的框架来加载的, 如果当前的classpath 环境下没有相关联的依赖, 则意味着这些类没必要加载, 所以, 通过这种条件过滤可以有效的减少@configuration 类的数量从而减低SpringBoot 的启动时间

###### Conditional 中的其他注解

| Conditions                   | 描述                                       |
| ---------------------------- | ------------------------------------------ |
| @ConditionalOnBean           | 在存在某个Bean 的时候                      |
| @ConditionalOnMissingBean    | 不存在某个Bean 的时候                      |
| @ConditionalOnClass          | 当前classpath 可以找到某个类型的类时       |
| @ConditionalOnMissingClass   | 当前classpath 不可以找到某个类型的类的时候 |
| @ConditionalOnResource       | 当前classpath 是否存在某个资源文件         |
| @ConditionalOnProperty       | 当前jvm 是否包含某个系统属性为某个值       |
| @ConditionalOnWebApplication | 当前Spring context 是否是web 应用程序      |



###  代码演示

####   使用ComponentScan 注解

我们先准备一个需要托管的类 ComponentScanDemo,加上@Configuration 注解

```java
@Configuration
public class ComponentScanDemo {


}
```

DemoClass 类加上 @Service 注解

```java
@Service
public class DemoClass {


}
```

测试类

```java
@ComponentScan(basePackages = "com.springboot.basic.configuration.componentscan")
public class ComponentScanMain {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(ComponentScanMain.class);
        for (int i = 0; i < applicationContext.getBeanDefinitionNames().length; i++) {
            System.out.println(applicationContext.getBeanDefinitionNames()[i]);
        }
    }

}
```

运行结果:

````java
17:32:50.342 [main] DEBUG org.springframework.context.annotation.AnnotationConfigApplicationContext - Refreshing org.springframework.context.annotation.AnnotationConfigApplicationContext@2a33fae0
17:32:50.374 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'org.springframework.context.annotation.internalConfigurationAnnotationProcessor'
17:32:50.452 [main] DEBUG org.springframework.context.annotation.ClassPathBeanDefinitionScanner - Identified candidate component class: file [D:\workspace\lyn\lyn-project\javaNotes\SpringBoot\spring-boot-basic\target\classes\com\springboot\basic\configuration\componentscan\ComponentScanDemo.class]
17:32:50.455 [main] DEBUG org.springframework.context.annotation.ClassPathBeanDefinitionScanner - Identified candidate component class: file [D:\workspace\lyn\lyn-project\javaNotes\SpringBoot\spring-boot-basic\target\classes\com\springboot\basic\configuration\componentscan\DemoClass.class]
17:32:50.609 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'org.springframework.context.event.internalEventListenerProcessor'
17:32:50.612 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'org.springframework.context.event.internalEventListenerFactory'
17:32:50.616 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'org.springframework.context.annotation.internalAutowiredAnnotationProcessor'
17:32:50.624 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'org.springframework.context.annotation.internalCommonAnnotationProcessor'
17:32:50.638 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'componentScanMain'
17:32:50.645 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'componentScanDemo'
17:32:50.647 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'demoClass'
org.springframework.context.annotation.internalConfigurationAnnotationProcessor
org.springframework.context.annotation.internalAutowiredAnnotationProcessor
org.springframework.context.annotation.internalCommonAnnotationProcessor
org.springframework.context.event.internalEventListenerProcessor
org.springframework.context.event.internalEventListenerFactory
componentScanMain
componentScanDemo
demoClass

````

结果能看到我们想要托管的类

#### 使用Import 注解和Bean注解进行注入

先准备一个需要被托管的类

```java
public class OtherBean {


}

```



再准备一个配置类,使用bean 注解将这个类注入到IOC容器中

```java
@Configuration
public class OtherConfig {


    @Bean
    public OtherBean otherBean() {
        return new OtherBean();
    }
}

```

再准备一个想被托管的类

```java
public class DefaultBean {


}

```

准备一个总的配置文件

```java
@Configuration
@Import(OtherConfig.class)
public class SpringConfig {

    @Bean
    public DefaultBean defaultBean() {
        return new DefaultBean();
    }
}

```

编写测试类

```java
public class SecondMain {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(SpringConfig.class);
        for (int i = 0; i < applicationContext.getBeanDefinitionNames().length; i++) {
            System.out.println(applicationContext.getBeanDefinitionNames()[i]);
        }
    }

}

```

运行结果就能看到我们想要托管的类


```java
springConfig
com.springboot.basic.configuration.imports.other.OtherConfig
otherBean
defaultBean
```

####  使用Import 注解中使用 ImportSelector接口和ImportBeanDefinitionRegistrar 接口进行动态注入

先准备两个类 

```java
public class CacheService {


}

```

```java
public class LoggerService {


}

```

再创建两个类, 分别实现ImportSelector接口和ImportBeanDefinitionRegistrar 接口返回上面两个类

```java
public class CacheImportSelector implements ImportSelector {


    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {

        Map<String, Object> annotationAttributes = annotationMetadata.getAnnotationAttributes(EnableDefineService.class.getName());


        return new String[]{CacheService.class.getName()};
    }
}
```

```java
public class LoggerDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        Class beanClass = LoggerService.class;
        RootBeanDefinition beanDefinition = new RootBeanDefinition(beanClass);
        String beanName = StringUtils.uncapitalize(beanClass.getSimpleName());
        registry.registerBeanDefinition(beanName, beanDefinition);
    }
}
```

创建Enable 注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import({LoggerDefinitionRegistrar.class, CacheImportSelector.class})
public @interface EnableDefineService {


    //配置一些方法
    Class<?>[] exclude() default {};


}

```

编写测试类

```java
@SpringBootApplication
@EnableDefineService
public class EnableDemoMain {
    public static void main(String[] args) {
        ConfigurableApplicationContext ca= SpringApplication.run(EnableDemoMain.class,args);

        System.out.println(ca.getBean(CacheService.class));
        System.out.println(ca.getBean(LoggerService.class));
    }


}

```

运行则能打印出来两个类的实例信息

#### EnableAutoConfiguration  注入

新建一个项目springboot-core

创建HelloService

```java
public class HelloService {


    public String say() {
        return "Hello World";
    }

}

```

再创建配置类 HelloConfig

```java

@Configuration
public class HelloConfig {


    @Bean
    public HelloService helloService() {
        return new HelloService();
    }
}

```

在 resources 下创建META-INF 目录

新建spring.factories 配置文件,EnableAutoConfiguration   会自动注入这配置文件里配置类

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.springboot.core.HelloConfig

```

创建 spring-autoconfigure-metadata.properties 配置文件, 注入配置文件的时候  会按照这个配置文件中的配置进行条件筛选

```java
com.springboot.core.GupaoConfig.ConditionalOnClass=com.springboot.core.TestClass

```

在另外一个项目里面加入 springboot-core 项目的依赖

```java
      <dependency>
            <groupId>com.springboot.core</groupId>
            <artifactId>springboot-core</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
```

修改主启动类

```java

@SpringBootApplication
public class SpringBootBasicApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext applicationContext = SpringApplication.run(SpringBootBasicApplication.class, args);
        HelloService helloService = applicationContext.getBean(HelloService.class);
        System.out.println(helloService.say());;
    }

}

```

运行成功后打印出`Hello World`

## 2. 什么是starter

Starter 是SpringBoot 中的一个非常重要的概念, Starter相当于模块,它能将模块所需的依赖整个起来并对模块内的Bean 根据环境(条件) 进行自动装配.使用者还需要依赖相应功能的Statrer ,无需做过多的配置和依赖, SpringBoot 就能自动扫描并加载相应的模块.

SpringBoot中存在很多开箱即用的Starter 依赖,使得我们在开发业务代码时能够非常方便、不需要过多关注框架的配置,只需要关注业务即可.

### 	spring-boot-starter-logger

在实际应用中, 日志其实是最重要的一个组件.

1. 他可以作为系统提供错误以及日常 的定位.
2. 也可以对访问的记录进行追踪
3. 当然,在很多大型的互联网应用中, 基于日志的手机以及分析可以了解用户的用户画像,比如兴趣爱好、点击行为.

#### 常见的日志框架

可能是太常见了,所以使得大家很少关注,只是要用的时候复制粘贴一份就可以了,甚至连配置文件中的配置都不清楚.另一方面,Java 中提供了日志组件太多了,一会儿log4j,一会儿logback,一会儿又是log4j4,不清楚其中的关联.

Java 中常用的日志框架: Log4j、Log4j2、Commons Logging、Slf4j、Logback、JUL(Java Util Logging).

#### 简单介绍日志的发展历史

最早的日志组件是Apache 基金会提供的Log4j, Log4j 能够通过配置文件轻松的实现日志系统的管理和多样化配置, 所以很快被广泛运用. 也就我们接触的比较早和比较多的日志组件. 它几乎成了Java 社区的日志标准.

据说Apache 基金会还曾经建议SUN 引入Log4j 到Java 的标准库中, 但SUN 拒绝了. 所以SUN 公司 在Java 1.4 版本中,增加了日志库(Java Util Logger). 其实现基本模仿了Log4j的技术. 在JUL 出来之前, Log4j 就已经成为一项成熟的技术, 使得Log4j 在选择上占据了一定的优势.

Apache 推出JUL 之后, 有一些项目使用JUL, 也有一些项目使用Log4j.  这样就造成了开发者的混乱, 因为这两个日志组件没有关联, 所以想实现统一管理或者替换就非常困难. 怎么办呢?

这个时候又轮到Apache 出手了, 他推出一个Apache Commons Logging 组件, 只是定义了一套日志接口(其内部也提供了一个Simple Log的简易实现),支持运行时动态加载日志组件的实现,也就是说, 在你的应用代码中, 只需要调用Common Logger  的接口, 底层实现可以是Log4j,  也可以是Java Util Logging.

 由于它很出色的完成了主流日志的兼容, 所以基本上在后面的很长一段时间, 是无敌的存在. 连Spring 也都是依赖JCL 进行日志管理.

但是故事还没有结束.

原Log4j 的作者, 它觉得 Apache Commons Logging  不够优秀, 所以它也行搞一套更优雅的方案, 于是slf4j 日志体系诞生了, slf4j  实际上及就是一个日志门面接口,它的作用类似于Commons Logging. 并且它还为slf4j 提供了一个日志的实现 logback.

因为大家可以发现Java 的日志领域被划分为两个大营:Commons Logging 和slf4j

另外还有一个log4j2 是怎么回事? 因为 slf4j 以及它的实现 logback 出来以后,很快就超赶了原本的 apache 的 log4j 体系,所以 apache 在2012 年重写了 log4j, 成立了新的项目 log4j2.

 总的来说,日志的整个体系分为日志框架和日志系统:

- 日志框架: JCL/ slf4j
- 日志系统: Log4j、Log4j2、Logback、JUL。

而我们现在的应用中,绝大部分都是使用 slf4j 作为门面, 然后搭配 logback 或者 log4j 日志系统的. 

##  3.SpringBoot 的另一大神器 - Actuator

微服务应用开发完成后, 最终目的是为了发布到生产环境上给用户使用,开发结束并不意味着研发的生命周期结束,很多的时候只是一个开始, 因为服务在本地测试完成以后, 并不一定能够非常完善的考虑到各种场景. 所以需要通过运维来保障服务的稳定.

在以前的传统应用中,我们可以靠人工来监控. 但是微服务中,几千上万个服务, 我们需要了解各个服务的健康状态, 就必须依赖监控平台来实现.

所以在SpringBoot 框架中提供了 ` spring-boot-starter-actuator ` 自动配置模块来支持对于SpringBoot 应用的监控.

### Actuator

SpringBoot Acyuator 的关键特性是在应用程序里提供众多WEB 端点, 通过他们了解应用程序运行时的内部状态. 有了Actuator ,你可以知道Bean 在Spring 应用程序上下文里是如何组装在一起的, 掌握应用程序可以获取的环境属性信息.

在SpringBoot 项目中, 添加 actuator 的一个starter

```java
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

### Actuator 提供的 endpoint

 启动服务后, 可以通过下面这个地址看到 actuator 提供的所有endoint 地址

 http://localhost:8080/actuator 

可以看到非常多的endpoint. 有一些Endpiont 是不能访问的, 涉及到安全问题. 

如果想开启访问的那些安全相关的url, 可以在 appliaction.propertie/yml 中配置, 开启所有的 endpoint

> ```
> management.endpoints.web.exposure.include=*
> ```

####  health

 针对当前SpringBoot 的健康检查, 默认情况下, 会通过 "up" 或者"down" , 可以基于下面这个配置, 来打印health 更加详细的信息

> ```
> ## 打印health 更信息的监控信息
> management.endpoint.health.show-details=always
> ```

####  loggers



显示当前SpringBoot 应用中的日志配置信心, 针对每个package 对应的日志级别

#### beans

获取当前SpringBoot 应用中IOC 容器中的所有的bean

#### Dump

获取活动线程的快照.

#### Mappings

返回全部的url 路径, 以及和控制器的映射关系.

#### conditions

显示当前所有的条件注解, 提供一份自动配置生效的条件情况, 记录哪些自动配置条件通过了, 哪些没通过. 

#### shuwdown

关闭应用程序,需要添加以下这个配置

> ```
> ## 使用 shutdowm 关闭应用程序
> management.endpoint.shutdown.enabled=true
> ```
> 

这个Endpoint 是比较危险的, 如果没有一定的安全保证,不要开启. 

#### Env

获取全部的环境信息

###  关于Health 的原理

应用健康状态检查应该是监控系统中最基本的要求, 所以我们基于health 来分析一下它是如何实现的.SpringBoot 预先通过org.springframework.boot.actuate.autoconfigure.health.HealthIndicatorAutoConfiguration   这个就是基于Springboot 的自动装配来载入的. 所以可以在spring-boot-actuator-autoconfigure 这个包下找到spring.factories 

![](http://files.luyanan.com//img/20191102114815.png)

Actuator 中提供了非常多的扩展点, 默认情况下提供了一些常见的服务的监控检查和支持。

-  DataSourceHealthIndicator 
-  DiskSpaceHealthIndicator 
-  RedisHealthIndicator 
- ...

其中, 有一些服务的检查, 需要依赖于当前应用是否集成了对应的组件, 比如redis,如果没有集成, 那么 RedisHealthIndicatorAutoConfiguration  就不会被装载. 因为它有condition 的条件判断. 

###  Actuator 对于JMX 支持

除了REST 方式发布的Endpoint,Actuator  还把它的端点以JMX MBean 的方式发布出来, 可以通过JMX 来查看和管理

#### 操作步骤

在CMD中输入 jconsole,连接到springboot  应用

![](http://files.luyanan.com//img/20191102115501.png)

就可以看到JBean 的信息以及相应的操作,  比如可以在操作菜单中访问shutdown 的endpiont 来关闭服务.

![](http://files.luyanan.com//img/20191102115715.png)

#### 什么是JMX

JMX 全称是 Java Management Extensions , Java 管理扩展,它提供了对java 应用程序和JVM 的监控和管理功能. 通过JMX，我们可以监控:

1. 服务器中的各种资源的使用情况, 比如CPU、内存
2. JVM 的内存的使用情况
3. JVM线程的使用情况

比如前面讲的Actuator ,就是基于JMX技术来实现对endpoint 的访问的. 

