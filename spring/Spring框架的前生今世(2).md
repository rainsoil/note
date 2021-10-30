#  2 . Spring的前世今生

&emsp;&emsp;相信经历过不使用框架开发Web项目的70后、80后都会有如此感触，如今的程序员开发项目太轻松
了，基本只需要关心业务如何实现，通用技术问题只需要集成框架便可。早在2007年，一个基于Java
语言的开源框架正式发布，取了一个非常有活力且美好的名字，叫做 Spring。它是一个开源的轻量级
Java SE（Java标准版本）/Java EE（Java企业版本）开发应用框架，其目的是用于简化企业级应用程
序开发。应用程序是由一组相互协作的对象组成。而在传统应用程序开发中，一个完整的应用是由一组
相互协作的对象组成。所以开发一个应用除了要开发业务逻辑之外，最多的是关注如何使这些对象协作
来完成所需功能，而且要低耦合、高聚合。业务逻辑开发是不可避免的，那如果有个框架出来帮我们来
创建对象及管理这些对象之间的依赖关系。可能有人说了，比如“抽象工厂、工厂方法模式”不也可以

帮我们创建对象，“生成器模式”帮我们处理对象间的依赖关系，不也能完成这些功能吗？可是这些又
需要我们创建另一些工厂类、生成器类，我们又要而外管理这些类，增加了我们的负担，如果能有种通
过配置方式来创建对象，管理对象之间依赖关系，我们不需要通过工厂和生成器来创建及管理对象之间
的依赖关系，这样我们是不是减少了许多工作，加速了开发，能节省出很多时间来干其他事。Spring
框架刚出来时主要就是来完成这个功能。

&emsp;&emsp;Spring框架除了帮我们管理对象及其依赖关系，还提供像通用日志记录、性能统计、安全控制、异常处
理等面向切面的能力，还能帮我管理最头疼的数据库事务，本身提供了一套简单的JDBC访问实现，提
供与第三方数据访问框架集成（如Hibernate、JPA），与各种 JavaEE技术整合（如JavaMail、任务
调度等等），提供一套自己的Web层框架SpringMVC、而且还能非常简单的与第三方Web框架集成。
从这里我们可以认为 Spring 是一个超级粘合大平台，除了自己提供功能外，还提供粘合其他技术和框
架的能力，从而使我们可以更自由的选择到底使用什么技术进行开发。而且不管是JAVASE（C/S架构）
应用程序还是 JAVA EE（B/S架构）应用程序都可以使用这个平台进行开发。如今的 Spring 已经不再
是一个框架，早已成为了一种生态。SpringBoot的便捷式开发实现了零配置，SpringCloud全家桶，
提供了非常方便的解决方案。接下来，让我们来深入探讨Spring到底能给我们带来什么？

## 一切从Bean开始

&emsp;&emsp;说到 Bean这个概念，还得从 Java的起源说起。早在 1996 年，Java还只是一个新兴的、初出茅庐的
编程语言。人们之所以关注她仅仅是因为，可以使用Java的Applet来开发Web应用，作为浏览器组
件。但开发者们很快就发现这个新兴的语言还能做更多的事情。与之前的所有语言不同，Java让模块化
构建复杂的系统成为可能（当时的软件行业虽然在业务上突飞猛进，但当时开发用的是传统的面向过程
开发思想，软件的开发效率一直踟蹰不前。伴随着业务复杂性的不断加深，开发也变得越发困难。其实，
当时也是OOP 思想飞速发展的时期，她在80年代末被提出，成熟于90年代，现今大多数编程语言都
已经是面向对象的）。

&emsp;&emsp;同年12月，Sun公司发布了当时还名不见经传但后来人尽皆知的 JavaBean 1.00-A规范。早期的
JavaBean规范针对于Java，她定义了软件组件模型。这个规范规定了一整套编码策略，使简单的Java
对象不仅可以被重用，而且还可以轻松地构建更为复杂的应用。尽管 JavaBean最初是为重用应用组件
而设计的，但当时他们却是主要用作构建窗体控件，毕竟在PC 时代那才是主流。但相比于当时正如日
中天的Delphi、VB和C++，它看起来还是太简易了，以至于无法胜任任何"实际的"工作需要。

&emsp;&emsp;复杂的应用通常需要事务、安全、分布式等服务的支持，但JavaBean并未直接提供。所以到了 1998
年 3 月，Sun 公司发布了EJB 1.0 规范，该规范把 Java组件的设计理念延伸到了服务器端，并提供了
许多必须的企业级服务，但他也不再像早期的 JavaBean 那么简单了。实际上，除了名字叫 EJB Bean
以外，其他的和JavaBean关系不大了。

&emsp;&emsp;尽管现实中有很多系统是基于EJB 构建的，但EJB从来没有实现它最初的设想：简化开发。EJB 的
声明式编程模型的确简化了很多基础架构层面的开发，例如事务和安全；但另一方面EJB在部署描述符
和配套代码实现等方面变得异常复杂。随着时间的推移，很多开发者对EJB已经不再抱有幻想，开始寻
求更简洁的方法。

&emsp;&emsp;现在Java组件开发理念重新回归正轨。新的编程技术AOP和 DI的不断出现，他们为JavaBean提
供了之前EJB才能拥有的强大功能。这些技术为POJO提供了类似EJB 的声明式编程模型，而没有引入
任何EJB的复杂性。当简单的JavaBean足以胜任时，人们便不愿编写笨重的EJB组件了。

&emsp;&emsp;客观地讲，EJB 的发展甚至促进了基于POJO的编程模型。引入新的理念，最新的 EJB规范相比之
前的规范有了前所未有的简化，但对很多开发者而言，这一切的一切都来得太迟了。到了EJB 3规范发
布时，其他基于 POJO的开发架构已经成为事实的标准了，而 Spring 框架也就是在这样的大环境下出
现的。

## Spring的设计初衷
&emsp;&emsp;Spring是为解决企业级应用开发的复杂性而设计，她可以做很多事。但归根到底支撑 Spring的仅
仅是少许的基本理念，而所有的这些基本理念都能可以追溯到一个最根本的使命：简化开发。这是一个
郑重的承诺，其实许多框架都声称在某些方面做了简化。而Spring 则立志于全方面的简化Java开发。
对此，她主要采取了4个关键策略：
1. 基于POJO的轻量级和最小侵入性编程；
2. 通过依赖注入和面向接口松耦合；
3. 基于切面和惯性进行声明式编程；
4. 通过切面和模板减少样板式代码；
5. 
而他主要是通过：面向Bean(BOP)、依赖注入（DI）以及面向切面（AOP）这三种方式来达成的。

## BOP编程伊始
&emsp;&emsp;Spring 是面向Bean的编程（Bean Oriented Programming, BOP），Bean在 Spring中才是真正的
主角。Bean在Spring中作用就像 Object对 OOP 的意义一样，Spring中没有Bean也就没有Spring
存在的意义。Spring提供了IOC 容器通过配置文件或者注解的方式来管理对象之间的依赖关系。

&emsp;&emsp;控制反转(其中最常见的实现方式叫做依赖注入（Dependency Injection，DI），还有一种方式叫
“依赖查找”（DependencyLookup，DL），她在 C++、Java、PHP 以及.NET中都运用。在最早的
Spring中是包含有依赖注入方法和依赖查询的，但因为依赖查询使用频率过低，不久就被Spring移除
了，所以在Spring中控制反转也被直接称作依赖注入)，她的基本概念是：不创建对象，但是描述创建
它们的方式。在代码中不直接与对象和服务连接，但在配置文件中描述哪一个组件需要哪一项服务。容
器 （在Spring 框架中是 IOC 容器）负责将这些联系在一起。

&emsp;&emsp; 在典型的IOC 场景中，容器创建了所有对象，并设置必要的属性将它们连接在一起，决定什么时间
调用方法。

## 依赖注入的基本概念
&emsp;&emsp;Spring 设计的核心 org.springframework.beans 包（架构核心是 org.springframework.core
包），它的设计目标是与JavaBean组件一起使用。这个包通常不是由用户直接使用，而是由服务器将
其用作其他多数功能的底层中介。下一个最高级抽象是BeanFactory 接口，它是工厂设计模式的实现，
允许通过名称创建和检索对象。BeanFactory 也可以管理对象之间的关系。

BeanFactory 最底层支持两个对象模型。
1. 单例：提供了具有特定名称的全局共享实例对象，可以在查询时对其进行检索。Singleton 是默
认的也是最常用的对象模型。
2. 原型：确保每次检索都会创建单独的实例对象。在每个用户都需要自己的对象时，采用原型模式。
Bean工厂的概念是Spring作为IOC 容器的基础。 IOC 则将处理事情的责任从应用程序代码转移到
框架。

## AOP编程理念
&emsp;&emsp; 面向切面编程，即AOP，是一种编程思想，它允许程序员对横切关注点或横切典型的职责分界线的
行为（例如日志和事务管理）进行模块化。AOP 的核心构造是方面（切面），它将那些影响多个类的行
为封装到可重用的模块中。

&emsp;&emsp; AOP 和 IOC 是补充性的技术，它们都运用模块化方式解决企业应用程序开发中的复杂问题。在典
型的面向对象开发方式中，可能要将日志记录语句放在所有方法和Java类中才能实现日志功能。在AOP
方式中，可以反过来将日志服务模块化，并以声明的方式将它们应用到需要日志的组件上。当然，优势
就是 Java类不需要知道日志服务的存在，也不需要考虑相关的代码。所以，用 Spring AOP 编写的应
用程序代码是松散耦合的。

AOP的功能完全集成到了Spring 事务管理、日志和其他各种特性的上下文中。

AOP 编程的常用场景有： Authentication（权限认证）、 AutoCaching（自动缓存处理）、 
ErrorHandling
（统一错误处理）、Debugging（调试信息输出）、Logging（日志记录）、Transactions（事务处理）
等。

## Spring5系统架构
&emsp;&emsp; Spring总共大约有 20个模块，由1300多个不同的文件构成。而这些组件被分别整合在核心容器（Core
Container）、AOP（Aspect Oriented Programming）和设备支持（Instrmentation）、数据访问
及集成（Data Access/Integeration）、Web、报文发送（Messaging）、Test，6个模块集合中。以
下是 Spring 5 的模块结构图：
![image](http://files.luyanan.com/3e875382-9827-40b5-ba39-8d3a11eb16b8.png)

组成Spring框架的每个模块集合或者模块都可以单独存在，也可以一个或多个模块联合实现。每个模
块的组成和功能如下：


### 核心容器
由spring-beans、 spring-core、 spring-context和spring-expression（SpringExpressionLanguage,
SpEL） 4个模块组成。

&emsp;&emsp; spring-core和spring-beans模块是Spring框架的核心模块，包含了控制反转（Inversion of
Control, IOC）和依赖注入（Dependency Injection, DI）。BeanFactory 接口是Spring 框架中的核
心接口，它是工厂模式的具体实现。BeanFactory 使用控制反转对应用程序的配置和依赖性规范与实际
的应用程序代码进行了分离。但BeanFactory 容器实例化后并不会自动实例化Bean，只有当Bean 被
使用时 BeanFactory 容器才会对该 Bean 进行实例化与依赖关系的装配。

&emsp;&emsp; spring-context模块构架于核心模块之上，他扩展了BeanFactory，为她添加了Bean生命周期控
制、框架事件体系以及资源加载透明化等功能。此外该模块还提供了许多企业级支持，如邮件访问、远
程访问、任务调度等，ApplicationContext是该模块的核心接口，她的超类是BeanFactory。与
BeanFactory 不同，ApplicationContext容器实例化后会自动对所有的单实例Bean进行实例化与依
赖关系的装配，使之处于待用状态。

&emsp;&emsp; spring-context-support 模块是对Spring IOC 容器的扩展支持，以及IOC子容器。

&emsp;&emsp; spring-context-indexer模块是Spring的类管理组件和Classpath扫描。

&emsp;&emsp;spring-expression模块是统一表达式语言（EL）的扩展模块，可以查询、管理运行中的对象，同
时也方便的可以调用对象方法、操作数组、集合等。它的语法类似于传统EL，但提供了额外的功能，最
出色的要数函数调用和简单字符串的模板函数。这种语言的特性是基于Spring产品的需求而设计，他
可以非常方便地同Spring IOC 进行交互。

### AOP和设备支持
由spring-aop、spring-aspects 和spring-instrument 3个模块组成。


&emsp;&emsp;spring-aop是Spring 的另一个核心模块，是AOP 主要的实现模块。作为继OOP 后，对程序员影
响最大的编程思想之一，AOP极大地开拓了人们对于编程的思路。在Spring 中，他是以JVM的动态
代理技术为基础，然后设计出了一系列的AOP横切实现，比如前置通知、返回通知、异常通知等，同
时，Pointcut接口来匹配切入点，可以使用现有的切入点来设计横切面，也可以扩展相关方法根据需求
进行切入。

&emsp;&emsp; spring-aspects 模块集成自AspectJ框架，主要是为Spring AOP提供多种AOP 实现方法。

&emsp;&emsp; spring-instrument模块是基于JAVA SE中的"java.lang.instrument"进行设计的，应该算是AOP
的一个支援模块，主要作用是在JVM启用时，生成一个代理类，程序员通过代理类在运行时修改类的
字节，从而改变一个类的功能，实现AOP 的功能。在分类里，我把他分在了AOP 模块下，在Spring 官
方文档里对这个地方也有点含糊不清，这里是纯个人观点。

### 数据访问与集成
由spring-jdbc、spring-tx、spring-orm、spring-jms和spring-oxm 5个模块组成。

&emsp;&emsp; spring-jdbc模块是Spring 提供的JDBC抽象框架的主要实现模块，用于简化SpringJDBC操作 。
主要是提供JDBC模板方式、关系数据库对象化方式、SimpleJdbc方式、事务管理来简化JDBC编程，
主要实现类是JdbcTemplate、SimpleJdbcTemplate以及NamedParameterJdbcTemplate。

&emsp;&emsp; spring-tx模块是Spring JDBC事务控制实现模块。使用Spring框架，它对事务做了很好的封装，
通过它的AOP配置，可以灵活的配置在任何一层；但是在很多的需求和应用，直接使用JDBC事务控
制还是有其优势的。其实，事务是以业务逻辑为基础的；一个完整的业务应该对应业务层里的一个方法；
如果业务操作失败，则整个事务回滚；所以，事务控制是绝对应该放在业务层的；但是，持久层的设计
则应该遵循一个很重要的原则：保证操作的原子性，即持久层里的每个方法都应该是不可以分割的。所
以，在使用Spring JDBC事务控制时，应该注意其特殊性。

&emsp;&emsp; spring-orm模块是ORM 框架支持模块，主要集成 Hibernate, Java Persistence API (JPA) 和
Java Data Objects (JDO) 用于资源管理、数据访问对象(DAO)的实现和事务策略。
spring-oxm模块主要提供一个抽象层以支撑OXM（OXM是Object-to-XML-Mapping的缩写，
它是一个O/M-mapper，将java对象映射成XML数据，或者将XML数据映射成java对象），例如：
JAXB, Castor, XMLBeans, JiBX 和 XStream等。

&emsp;&emsp; spring-jms模块（JavaMessagingService）能够发送和接收信息，自SpringFramework4.1以
后，他还提供了对spring-messaging模块的支撑。

### Web组件
&emsp;&emsp; 由spring-web、spring-webmvc、spring-websocket 和spring-webflux 4个模块组成。

&emsp;&emsp; spring-web模块为Spring提供了最基础Web支持，主要建立于核心容器之上，通过Servlet或
者Listeners 来初始化IOC 容器，也包含一些与Web相关的支持。

&emsp;&emsp;spring-webmvc模块众所周知是一个的Web-Servlet模块，实现了Spring MVC
（model-view-Controller）的Web应用。

&emsp;&emsp; spring-websocket模块主要是与Web前端的全双工通讯的协议。

&emsp;&emsp; spring-webflux是一个新的非堵塞函数式 Reactive Web 框架，可以用来建立异步的，非阻塞，
事件驱动的服务，并且扩展性非常好。
### 通信报文
&emsp;&emsp; 即spring-messaging模块，是从Spring4开始新加入的一个模块，主要职责是为Spring 框架集
成一些基础的报文传送应用。
### 集成测试

&emsp;&emsp; 即spring-test 模块，主要为测试提供支持的，毕竟在不需要发布（程序）到你的应用服务器或者连接
到其他企业设施的情况下能够执行一些集成测试或者其他测试对于任何企业都是非常重要的。
### 集成兼容
&emsp;&emsp; 即spring-framework-bom模块，Bill of Materials.解决Spring的不同模块依赖版本不同问题。
## 各模块之间的依赖关系
Spring官网对Spring5各模块之间的关系也做了详细说明：
![image](http://files.luyanan.com/443dee0d-670d-4dbb-83c8-52997df67c7b.jpg)

我本人也对Spring5 各模块做了一次系统的总结，描述模块之间的依赖关系，希望能对小伙伴们有所
帮助。

![image](http://files.luyanan.com/db64b38a-4999-414e-b828-abfe2fad88f1.png)