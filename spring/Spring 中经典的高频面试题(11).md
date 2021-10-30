# 11 . Spring 中经典的高频面试题



### 1、什么是 Spring 框架？Spring 框架有哪些主要模块？
Spring 框架是一个为 Java 应用程序的开发提供了综合、广泛的基础性支持的 Java 平台。Spring 帮助
开发者解决了开发中基础性的问题，使得开发人员可以专注于应用程序的开发。Spring 框架本身亦是按
照设计模式精心打造，这使得我们可以在开发环境中安心的集成 Spring 框架，不必担心 Spring 是如何
在后台进行工作的。

Spring 框架至今已集成了 20 多个模块。这些模块主要被分如下图所示的核心容器、数据访问/集成,、 Web、AOP（面向切面编程）、工具、消息和测试模块。

### 2、使用 Spring 框架能带来哪些好处？
下面列举了一些使用 Spring 框架带来的主要好处：
1. Dependency Injection(DI) 方法使得构造器和 JavaBean properties 文件中的依赖关系一目了然。

2. 与 EJB 容器相比较，IOC 容器更加趋向于轻量级。这样一来 IOC 容器在有限的内存和 CPU 资源的
情况下进行应用程序的开发和发布就变得十分有利。
3. Spring 并没有闭门造车，Spring 利用了已有的技术比如 ORM 框架、logging 框架、J2EE、Quartz
和 JDK Timer，以及其他视图技术。
4. Spring 框架是按照模块的形式来组织的。由包和类的编号就可以看出其所属的模块，开发者仅仅需
要选用他们需要的模块即可。
5. 要测试一项用 Spring 开发的应用程序十分简单，因为测试相关的环境代码都已经囊括在框架中了。
更加简单的是，利用 JavaBean 形式的 POJO 类，可以很方便的利用依赖注入来写入测试数据。
6. Spring 的 Web 框架亦是一个精心设计的 Web MVC 框架，为开发者们在 web 框架的选择上提供
了一个除了主流框架比如 Struts、过度设计的、不流行 web 框架的以外的有力选项。
7. Spring 提供了一个便捷的事务管理接口，适用于小型的本地事务处理（比如在单 DB 的环境下）和
复杂的共同事务处理（比如利用 JTA 的复杂 DB 环境）。

### 3、什么是控制反转(IOC)？什么是依赖注入？
1. 控制反转是应用于软件工程领域中的，在运行时被装配器对象来绑定耦合对象的一种编程技巧，对
象之间耦合关系在编译时通常是未知的。在传统的编程方式中，业务逻辑的流程是由应用程序中的早已
被设定好关联关系的对象来决定的。在使用控制反转的情况下，业务逻辑的流程是由对象关系图来决定
的，该对象关系图由装配器负责实例化，这种实现方式还可以将对象之间的关联关系的定义抽象化。而
绑定的过程是通过“依赖注入”实现的。
2. 控制反转是一种以给予应用程序中目标组件更多控制为目的设计范式，并在我们的实际工作中起到
了有效的作用。
3. 依赖注入是在编译阶段尚未知所需的功能是来自哪个的类的情况下，将其他对象所依赖的功能对象
实例化的模式。这就需要一种机制用来激活相应的组件以提供特定的功能，所以依赖注入是控制反转的
基础。否则如果在组件不受框架控制的情况下，框架又怎么知道要创建哪个组件？

### 4、在 Java 中依赖注入有哪些方式？
1. 构造器注入
2. Setter 方法注入
3. 接口注入

### 5 BeanFactory 和 ApplicationContext 有什么区别？
BeanFactory 可以理解为含有 bean 集合的工厂类。BeanFactory 包含了种 bean 的定义，以便在接
收到客户端请求时将对应的 bean 实例化。

BeanFactory 还能在实例化对象的时生成协作类之间的关系。此举将 bean 自身与 bean 客户端的配置
中解放出来。BeanFactory 还包含了 bean 生命周期的控制，调用客户端的初始化方法（initialization
Methods）和销毁方法（destruction Methods）。

从表面上看，ApplicationContext 如同 bean factory 一样具有 bean 定义、bean 关联关系的设置，
根据请求分发 bean 的功能。但 ApplicationContext 在此基础上还提供了其他的功能。

1. 提供了支持国际化的文本消息
2. 统一的资源文件读取方式
3. 已在监听器中注册的 bean 的事件

以下是三种较常见的 ApplicationContext 实现方式：

1. ClassPathXmlApplicationContext：从 classpath 的 XML 配置文件中读取上下文，并生成上下文
定义。应用程序上下文从程序环境变量中取得。
ApplicationContext context = new ClassPathXmlApplicationContext(“application.xml”);

2. FileSystemXmlApplicationContext ：由文件系统中的 XML 配置文件读取上下文。
ApplicationContext context = new FileSystemXmlApplicationContext(“application.xml”);
3. XmlWebApplicationContext：由 Web 应用的 XML 文件读取上下文。

### 6、Spring 提供几种配置方式来设置元数据？

将 Spring 配置到应用开发中有以下三种方式：

1. 基于 XML 的配置
2. 基于注解的配置
3. 基于 Java 的配置


### 7、如何使用 XML 配置的方式配置 Spring？ 

在 Spring 框架中，依赖和服务需要在专门的配置文件来实现，我常用的 XML 格式的配置文件。这些配
置文件的格式通常用开头，然后一系列的 

SpringXML 配置的主要目的时候是使所有的 Spring 组件都可以用 xml 文件的形式来进行配置。这意味
着不会出现其他的 Spring 配置类型（比如声明的方式或基于 Java Class 的配置方式）

Spring 的 XML 配置方式是使用被 Spring 命名空间的所支持的一系列的 XML 标签来实现的。Spring
有以下主要的命名空间：context、beans、jdbc、tx、aop、mvc 和 aso。

```xml
<beans>
<!-- JSON Support -->
<bean name="viewResolver"
class="org.springframework.web.servlet.view.BeanNameViewResolver"/>
<bean name="jsonTemplate"
class="org.springframework.web.servlet.view.json.MappingJackson2JsonView"/>
<bean id="restTemplate" class="org.springframework.web.client.RestTemplate"/>
</beans>
```
下面这个 web.xml 仅仅配置了 DispatcherServlet，这件最简单的配置便能满足应用程序配置运行时组
件的需求。

```xml
<web-app>
<display-name>Archetype Created Web Application</display-name>
<servlet>
<servlet-name>spring</servlet-name>
<servlet-class>
org.springframework.web.servlet.DispatcherServlet
</servlet-class>
<load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
<servlet-name>spring</servlet-name>
<url-pattern>/</url-pattern>
</servlet-mapping>
</web-app>
```


### 8、Spring 提供哪些配置形式？
Spring 对 Java 配置的支持是由@Configuration 注解和@Bean 注解来实现的。由@Bean 注解的方法
将会实例化、配置和初始化一个新对象，这个对象将由 Spring 的 IOC 容器来管理。@Bean 声明所起
到的作用与 元素类似。被@Configuration 所注解的类则表示这个类的主要目的是作为 bean 定义的资
源。被@Configuration 声明的类可以通过在同一个类的内部调用@bean 方法来设置嵌入 bean 的依
赖关系。

最简单的@Configuration 声明类请参考下面的代码：

```java
@Configuration
public class AppConfig{
@Bean
public MyService myService() {
return new MyServiceImpl();
}
}
```

对于上面的@Beans 配置文件相同的 XML 配置文件如下：

```xml
<beans>
<bean id="myService" class="com.services.MyServiceImpl"/>
</beans>
```
上述配置方式的实例化方式如下：利用 AnnotationConfigApplicationContext 类进行实例化

```java
public static void main(String[] args) {
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
MyService myService = ctx.getBean(MyService.class);
myService.doStuff();
}
```


要使用组件组建扫描，仅需用@Configuration 进行注解即可：
```java
@Configuration
@ComponentScan(basePackages = "com")
public class AppConfig {
}
```

在上面的例子中，com.gupaoedu 包首先会被扫到，然后再容器内查找被@Component 声明的类，
找到后将这些类按照 Spring bean 定义进行注册。

如 果 你 要 在 你 的 web 应 用 开 发 中 选 用 上 述 的 配 置 的 方 式 的 话 ， 需 要 用
AnnotationConfigWebApplicationContext 类来读取配置文件，可以用来配置 Spring 的 Servlet 监
听器 ContrextLoaderListener 或者 Spring MVC 的 DispatcherServlet。

```xml
<web-app>
<context-param>
<param-name>contextClass</param-name>
<param-value>
org.springframework.web.context.support.AnnotationConfigWebApplicationContext
</param-value>
</context-param>
<context-param>
<param-name>contextConfigLocation</param-name>
<param-value>com.AppConfig</param-value>
</context-param>
<listener>
<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
<servlet>
<servlet-name>dispatcher</servlet-name>
<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
<init-param>
<param-name>contextClass</param-name>
<param-value>
org.springframework.web.context.support.AnnotationConfigWebApplicationContext
</param-value>
</init-param>
<init-param>
<param-name>contextConfigLocation</param-name>
<param-value>com.web.MVCConfig</param-value>
</init-param>

</servlet>
<servlet-mapping>
<servlet-name>dispatcher</servlet-name>
<url-pattern>/web/*</url-pattern>
</servlet-mapping>
</web-app>
```

### 9、怎样用注解的方式配置 Spring？

Spring 在 2.5 版本以后开始支持用注解的方式来配置依赖注入。可以用注解的方式来替代 XML 方式的
bean 描述，可以将 bean 描述转移到组件类的内部，只需要在相关类上、方法上或者字段声明上使用
注解即可。注解注入将会被容器在 XML 注入之前被处理，所以后者会覆盖掉前者对于同一个属性的处
理结果。

注解装配在Spring 中是默认关闭的。所以需要在Spring文件中配置一下才能使用基于注解的装配模式。
如果你想要在你的应用程序中使用关于注解的方法的话，请参考如下的配置。

```xml
<beans>
<context:annotation-config/>
</beans>
```
在标签配置完成以后，就可以用注解的方式在 Spring 中向属性、方法和构造方法中自动装配变量。
下面是几种比较重要的注解类型：

1. @Required：该注解应用于设值方法。
2. @Autowired：该注解应用于有值设值方法、非设值方法、构造方法和变量。
3. @Qualifier：该注解和@Autowired 注解搭配使用，用于消除特定 bean 自动装配的歧义。
4. JSR-250 Annotations：Spring 支持基于 JSR-250 注解的以下注解，@Resource、@PostConstruct
和 @PreDestroy。

### 10、请解释 Spring Bean 的生命周期？
Spring Bean 的生命周期简单易懂。在一个 bean 实例被初始化时，需要执行一系列的初始化操作以达
到可用的状态。同样的，当一个 bean 不在被调用时需要进行相关的析构操作，并从 bean 容器中移除。

Spring bean factory 负责管理在 spring 容器中被创建的 bean 的生命周期。Bean 的生命周期由两组
回调（call back）方法组成
1. 初始化之后调用的回调方法。
2. 销毁之前调用的回调方法。

Spring 框架提供了以下四种方式来管理 bean 的生命周期事件：

1. InitializingBean 和 DisposableBean 回调接口
2. 针对特殊行为的其他 Aware 接口
3. Bean 配置文件中的 Custom init()方法和 destroy()方法
4. @PostConstruct 和@PreDestroy 注解方式
使用 customInit()和 customDestroy()方法管理 bean 生命周期的代码样例如下：
```xml
<beans>
<bean id="demoBean" class="com.task.DemoBean"
init-Method="customInit" destroy-Method="customDestroy">
</bean>
</beans>
```
### 11、Spring Bean 作用域之间的区别？
Spring 容器中的 bean 可以分为 5 个范围。所有范围的名称都是自说明的，但是为了避免混淆，还是让
我们来解释一下：

1. singleton：这种 bean 范围是默认的，这种范围确保不管接受到多少个请求，每个容器中只有一个
bean 的实例，单例的模式由 bean factory 自身来维护。
2.  prototype：原形范围与单例范围相反，为每一个 bean 请求提供一个实例。
3. request：在请求 bean 范围内会每一个来自客户端的网络请求创建一个实例，在请求完成以后，bean
会失效并被垃圾回收器回收。
4. Session：与请求范围类似，确保每个 session 中有一个 bean 的实例，在 session 过期后，bean 会
随之失效。
5. global-session：global-session 和 Portlet 应用相关。当你的应用部署在 Portlet 容器中工作时，它
包含很多 portlet。如果你想要声明让所有的 portlet 共用全局的存储变量的话，那么这全局变量需要存
储在 global-session 中。

全局作用域与 Servlet 中的 session 作用域效果相同。

### 12、什么是 Spring inner beans？
在 Spring 框架中，无论何时 bean 被使用时，当仅被调用了一个属性。一个明智的做法是将这个 bean
声明为内部 bean。内部 bean 可以用 setter 注入“属性”和构造方法注入“构造参数”的方式来实现。
比如，在我们的应用程序中，一个 Customer 类引用了一个 Person 类，我们的要做的是创建一个 Person
的实例，然后在 Customer 内部使用。

```java
public class Customer{
private Person person;
}
public class Person{
private String name;
private String address;
private int age;
}
```
内部 bean 的声明方式如下：

```xml
<bean id="CustomerBean" class="com.common.Customer">
<property name="person">
<bean class="com.common.Person">
<property name="name" value="lokesh" />
<property name="address" value="India" />
<property name="age" value="34" />
</bean>
</property>
</bean>
```

### 13、Spring 框架中的单例 Beans 是线程安全的么？
Spring 框架并没有对单例 bean 进行任何多线程的封装处理。关于单例 bean 的线程安全和并发问题需
要开发者自行去搞定。但实际上，大部分的 Spring bean 并没有可变的状态(比如 Serview 类和 DAO
类)，所以在某种程度上说 Spring 的单例 bean 是线程安全的。如果你的 bean 有多种状态的话（比如
View Model 对象），就需要自行保证线程安全。

最浅显的解决办法就是将多态 bean 的作用域由“singleton”变更为“prototype”。

### 14、请举例说明如何在 Spring 中注入一个 Java 集合？

Spring 提供了以下四种集合类的配置元素：
1.  : 该标签用来装配可重复的 list 值。
2.  : 该标签用来装配没有重复的 set 值。
3. : 该标签可用来注入键和值可以为任何类型的键值对。
4.  : 该标签支持注入键和值都是字符串类型的键值对。

下面看一下具体的例子：
```xml
<beans>
<bean id="javaCollection" class="com.JavaCollection">
<property name="customList">
<list>
<value>INDIA</value>
<value>Pakistan</value>
<value>USA</value>
<value>UK</value>
</list>
</property>
<property name="customSet">
<set>
<value>INDIA</value>
<value>Pakistan</value>
<value>USA</value>
<value>UK</value>
</set>
</property>
<property name="customMap">
<map>
<entry key="1" value="INDIA"/>
<entry key="2" value="Pakistan"/>
<entry key="3" value="USA"/>
<entry key="4" value="UK"/>

</map>
</property>
<property name="customProperies">
<props>
<prop key="admin">admin@111.com</prop>
<prop key="support">support@111.com</prop>
</props>
</property>
</bean>
</beans>
```

### 15、如何向 Spring Bean 中注入 java.util.Properties？
第一种方法是使用如下面代码所示的 标签：

```xml
<bean id="adminUser" class="com.common.Customer">
<property name="emails">
<props>
<prop key="admin">admin@aaa.com</prop>
<prop key="support">support@aaa.com</prop>
</props>
</property>
</bean>

```
也可用”util:”命名空间来从 properties 文件中创建出一个 propertiesbean，然后利用 setter 方法注
入 bean 的引用。

### 16、请解释 Spring Bean 的自动装配？
在 Spring 框架中，在配置文件中设定 bean 的依赖关系是一个很好的机制，Spring 容器还可以自动装
配合作关系 bean 之间的关联关系。这意味着 Spring 可以通过向 Bean Factory 中注入的方式自动搞定
bean 之间的依赖关系。自动装配可以设置在每个 bean 上，也可以设定在特定的 bean 上。
下面的 XML 配置文件表明了如何根据名称将一个 bean 设置为自动装配：
```xml
<bean id="employeeDAO" class="com.EmployeeDAOImpl" autowire="byName" />

```

除了 bean 配置文件中提供的自动装配模式，还可以使用@Autowired 注解来自动装配指定的 bean。
在使用@Autowired 注解之前需要在按照如下的配置方式在 Spring 配置文件进行配置才可以使用。
```xml
<context:annotation-config />

```
也可以通过在配置文件中配置 AutowiredAnnotationBeanPostProcessor 达到相同的效果。
```xml
<bean class ="org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor"/>

```
配置好以后就可以使用@Autowired 来标注了。

```xml
@Autowired
public EmployeeDAOImpl ( EmployeeManager manager ) {
this.manager = manager;
}
```

### 17、自动装配有哪些局限性？
自动装配有如下局限性：

重写：你仍然需要使用 和< property>设置指明依赖，这意味着总要重写自动装配。

原生数据类型:你不能自动装配简单的属性，如原生类型、字符串和类。

模糊特性：自动装配总是没有自定义装配精确，因此，如果可能尽量使用自定义装配。

### 18、请解释各种自动装配模式的区别？
在 Spring 框架中共有 5 种自动装配，让我们逐一分析。
1. no：这是 Spring 框架的默认设置，在该设置下自动装配是关闭的，开发者需要自行在 bean 定义中
用标签明确的设置依赖关系。
2. byName：该选项可以根据 bean 名称设置依赖关系。当向一个 bean 中自动装配一个属性时，容器
将根据 bean 的名称自动在在配置文件中查询一个匹配的 bean。如果找到的话，就装配这个属性，如
果没找到的话就报错。
3. byType：该选项可以根据 bean 类型设置依赖关系。当向一个 bean 中自动装配一个属性时，容器将
根据 bean 的类型自动在在配置文件中查询一个匹配的 bean。如果找到的话，就装配这个属性，如果
没找到的话就报错。
4. constructor：造器的自动装配和 byType 模式类似，但是仅仅适用于与有构造器相同参数的 bean，
如果在容器中没有找到与构造器参数类型一致的 bean，那么将会抛出异常。
5. autodetect：该模式自动探测使用构造器自动装配或者 byType 自动装配。首先，首先会尝试找合适
的带参数的构造器，如果找到的话就是用构造器自动装配，如果在 bean 内部没有找到相应的构造器或
者是无参构造器，容器就会自动选择 byTpe 的自动装配方式。

### 19、请举例解释@Required Annotation？

在产品级别的应用中，IOC 容器可能声明了数十万了 bean，bean 与 bean 之间有着复杂的依赖关系。
设值注解方法的短板之一就是验证所有的属性是否被注解是一项十分困难的操作。可以通过在中设置
“dependency-check”来解决这个问题。

在应用程序的生命周期中，你可能不大愿意花时间在验证所有 bean 的属性是否按照上下文文件正确配
置。或者你宁可验证某个 bean 的特定属性是否被正确的设置。即使是用“dependency-check”属性
也不能很好的解决这个问题，在这种情况下，你需要使用@Required 注解。

需要用如下的方式使用来标明 bean 的设值方法。

```java
public class EmployeeFactoryBean extends AbstractFactoryBean<Object> {
private String designation;
public String getDesignation() {
return designation;
}
@Required
public void setDesignation(String designation) {
this.designation = designation;
}
}
```
RequiredAnnotationBeanPostProcessor 是 Spring 中的后置处理用来验证被@Required 注解的
bean 属性是否被正确的设置了。在使用 RequiredAnnotationBeanPostProcesso 来验证 bean 属性之
前，首先要在 IOC 容器中对其进行注册：

```xml
<bean class="org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor" />

```
但是如果没有属性被用 @Required 注解过的话，后置处理器会抛出一个 BeanInitializationException
异常。
### 20、请举例说明@Qualifier 注解？

@Qualifier注解意味着可以在被标注bean的字段上可以自动装配。Qualifier注解可以用来取消Spring
不能取消的 bean 应用。
### 21、构造方法注入和设值注入有什么区别？
请注意以下明显的区别：

1. 在设值注入方法支持大部分的依赖注入，如果我们仅需要注入 int、string 和 long 型的变量，我们不
要用设值的方法注入。对于基本类型，如果我们没有注入的话，可以为基本类型设置默认值。在构造方
法注入不支持大部分的依赖注入，因为在调用构造方法中必须传入正确的构造参数，否则的话为报错。
2. 设值注入不会重写构造方法的值。如果我们对同一个变量同时使用了构造方法注入又使用了设置方法
注入的话，那么构造方法将不能覆盖由设值方法注入的值。很明显，因为构造方法尽在对象被创建时调
用。
3. 在使用设值注入时有可能还不能保证某种依赖是否已经被注入，也就是说这时对象的依赖关系有可能
是不完整的。而在另一种情况下，构造器注入则不允许生成依赖关系不完整的对象。
4. 在 设 值 注 入 时 如 果 对 象 A 和 对 象 B 互 相 依 赖 ， 在 创 建 对 象 A 时 Spring 会 抛 出
sObjectCurrentlyInCreationException 异常，因为在 B 对象被创建之前 A 对象是不能被创建的，反之
亦然。所以 Spring 用设值注入的方法解决了循环依赖的问题，因对象的设值方法是在对象被创建之前
被调用的。

### 22、Spring 框架中有哪些不同类型的事件？
Spring 的 ApplicationContext 提供了支持事件和代码中监听器的功能。
我们可以创建 bean 用来监听在 ApplicationContext 中发布的事件。ApplicationEvent 类和在
ApplicationContext 接口中处理的事件，如果一个 bean 实现了 ApplicationListener 接口，当一个
ApplicationEvent 被发布以后，bean 会自动被通知。

```java
public class AllApplicationEventListener implements ApplicationListener<ApplicationEvent> {
@Override

public void onApplicationEvent(ApplicationEvent applicationEvent) {
//process event
}
}
```

Spring 提供了以下 5 中标准的事件：
1. 上下文更新事件（ContextRefreshedEvent）：该事件会在 ApplicationContext 被初始化或者更新
时发布。也可以在调用 ConfigurableApplicationContext 接口中的 refresh()方法时被触发。
2. 上下文开始事件（ContextStartedEvent）：当容器调用 ConfigurableApplicationContext 的 Start()
方法开始/重新开始容器时触发该事件。
3. 上下文停止事件（ContextStoppedEvent）：当容器调用 ConfigurableApplicationContext 的 Stop()
方法停止容器时触发该事件。
4. 上下文关闭事件（ContextClosedEvent）：当 ApplicationContext 被关闭时触发该事件。容器被关
闭时，其管理的所有单例 Bean 都被销毁。
5. 请求处理事件（RequestHandledEvent）：在 Web 应用中，当一个 http 请求（request）结束触发
该事件。

除了上面介绍的事件以外，还可以通过扩展 ApplicationEvent 类来开发自定义的事件。


```java
public class CustomApplicationEvent extends ApplicationEvent {
public CustomApplicationEvent ( Object source, final String msg ){
super(source);
System.out.println("Created a Custom event");
}
}
```
为了监听这个事件，还需要创建一个监听器：

```java
public class CustomEventListener implements ApplicationListener < CustomApplicationEvent >{
@Override
public void onApplicationEvent(CustomApplicationEvent applicationEvent) {
}
}
```

之后通过 applicationContext 接口的 publishEvent()方法来发布自定义事件。
CustomApplicationEvent customEvent = new CustomApplicationEvent(applicationContext, “Test message”);
applicationContext.publishEvent(customEvent);

### 23、FileSystemResource 和 ClassPathResource 有何区别？

在 FileSystemResource 中需要给出 spring-config.xml 文件在你项目中的相对路径或者绝对路径。在
ClassPathResource 中 spring 会在 ClassPath 中自动搜寻配置文件，所以要把 ClassPathResource 文
件放在 ClassPath 下。

如果将 spring-config.xml 保存在了 src 文件夹下的话，只需给出配置文件的名称即可，因为 src 文件
夹是默认。

简而言之，ClassPathResource 在环境变量中读取配置文件，FileSystemResource 在配置文件中读取
配置文件。

### 24、Spring 框架中都用到了哪些设计模式？
Spring 框架中使用到了大量的设计模式，下面列举了比较有代表性的：
1. 代理模式—在 AOP 和 remoting 中被用的比较多。
2. 单例模式：在 spring 配置文件中定义的 bean 默认为单例模式。
3. 模板模式：用来解决代码重复的问题。
比如. RestTemplate, JmsTemplate, JpaTemplate。
4. 委派模式：Srping 提供了 DispatcherServlet 来对请求进行分发。
5. 工厂模式：BeanFactory 用来创建对象的实例，贯穿于 BeanFactory / ApplicationContext 接口的
核心理念。
6. 代理模式：AOP 思想的底层实现技术，Spring 中采用 JDK Proxy 和 CgLib 类库。


### 25、在 Spring 框架中如何更有效的使用 JDBC？
使用 Spring JDBC 框架，资源管理以及错误处理的代价都会减轻。开发人员只需通过 statements 和
queries 语句从数据库中存取数据。Spring 框架中通过使用模板类能更有效的使用 JDBC，也就是所谓
的 JdbcTemplate。

### 26、请解释下 Spring 框架中的 IOC 容器？


Spring 中的 org.springframework.beans 包和 org.springframework.context 包构成了 Spring 框
架 IOC 容器的基础。

BeanFactory 接 口 提 供 了 一 个 先 进 的 配 置 机 制 ， 使 得 任 何 类 型 的 对 象 的 配 置 成 为 可 能 。
ApplicationContex 接口对 BeanFactory（是一个子接口）进行了扩展，在 BeanFactory 的基础上添
加了其他功能，比如与 Spring 的 AOP 更容易集成，也提供了处理 message resource 的机制（用于国
际化）、事件传播以及应用层的特别配置，比如针对 Web 应用的 WebApplicationContext。

### 27、在 Spring 中可以注入 null 或空字符串吗？

完全可以

