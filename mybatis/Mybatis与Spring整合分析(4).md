# 4. Mybatis于Spring 整合分析

---------------------
#### 与 Spring 整合分析

http://www.mybatis.org/spring/zh/index.html
这里我们以传统的Spring为例,因为配置更加直观,在Spring中使用配置注册是一样的.

在前面,我们基于编程式的工程已经弄清楚了Mybatis的工作流程,核心模块和底层原理。编程式的工作,也就是Mybatis的原声API 里面有三个核心对象: SqlSessionFacyory,SqlSession,MapperProxy.

大部分我们不会在项目中单独使用Mybatis的工程,而是继承到Spring、中使用,但是却没有看到这三个对象在代理中出现.我们都是直接注入了一个Mapper接口,调用它的方法.

所以有几个关键的对象,我们要弄清楚:
1. SqlSessionFactory 是什么时候创建的?
2. SqlSession 去哪里了?为什么不用他来getMapper ？
3. 为什么@Autowired 注入一个接口,在使用的时候却变成了代理对象?在IOC 的容器中我们注入的是什么?注入的时候发生了什么事情?
#### 关键配置

我们先看一下把Mybatis集成到Spring 要做的几件事情。

为了让大家看起来更加直观,我们使用传统的XML 配置给大家做讲解,当然使用配置类@Configuration 效果是一样的,对于Spring来说只是解析的方式的差异.

除了Mybatis的依赖之外,我们还需要在pom.xml 里面加入Mybatis和Spring整合的jar(注意版本:mybatis 的版本和 mybatis-spring 的版本有兼容关系)
```xml
<dependency>
<groupId>org.mybatis</groupId>
<artifactId>mybatis-spring</artifactId>
<version>2.0.0</version>
</dependency>
```
然后在Spring的ApplicationContext.xml里面配置SqlSessionFactoryBean,它是用来帮助我们创建会话的,其中还要指定全局配置文件和Mapper映射器文件的路径.

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
<property name="configLocation" value="classpath:mybatis-config.xml"></property>
<property name="mapperLocations" value="classpath:mapper/*.xml"></property>
<property name="dataSource" ref="dataSource"/>
</bean>
```
然后在appcalitionContext.xml 配置需要扫描Mapper接口的路径

在Mybatis中有几种配置方式.一种是配置一个MapperScannerConfigurer
```xml
<bean id="mapperScanner" class="org.mybatis.spring.mapper.MapperScannerConfigurer">
<property name="basePackage" value="com.crud.dao"/>
</bean>

```

另一种是配置一个<scan>标签
```xml
<mybatis-spring:scan base-package="com.crud.dao"/>
```
还有一种是直接用@MapperScan 注解,比如我们在Spring Boot的主启动类上 添加上一个注解
```java
@SpringBootApplication
@MapperScan("com.crud.dao")
public class MybaitsApp {
public static void main(String[] args) {
SpringApplication.run(MybaitsApp.class, args);
}
}
```

这三种方式的实现效果是一样的.

### 1. 创建会话工厂
Spring对Mybatis的对象进行了管理,但是并不会直接替换Mybatis的核心对象,也就是意味着: Mybatis包里的 SqlSessionFactory,SqlSession,MapperProxy 这些都会用到,而mybatis-spring.jar 里面的类只是做了一些包装和桥梁的工作.

所以 第一步,我们看一下在Spring中,工厂类是怎么创建的?

我们在Spring的配置文件里面配置了一个SqlSessionFactoryBean,我们来看一下这个类
![image](http://files.luyanan.com/47afadcb-a962-477e-add9-13d6cb117cce.jpg)


它实现了InitialzingBean接口,所以要实现afterPropertiesSet() 方法,这个方法会在Bean属性值设置完成 的时候调用.

另外它实现了FactoryBean 接口,所以它初始化的时候,实际上是调用getObject() 方法,它里面调用的也是afterPropertiesSet() 方法.

在afterPropertiesSet() 方法里面

第一步是一些标签属性的检查,接下来是调用 buildSqlSessionFactory() 方法.

然后定义了一个Configuration, 叫做targetConfiguration 

426行,判断Configuration 对象是否存在,也就是是否已经解析过,如果已经有对象,那就覆盖一下属性.

433行,如果Configuration 不存在,但是配置了 configLocation属性,就会根据mybatis-config.xml 的文件路径,构建一个xmlConfiuration对象.

436行,否则,Configuration 对象不存在,configLocation 路径也没有,只能使用默认的属性去构建给configurationProperties 赋值.

  后面就是基于当前factory 对象里面的已有的属性,对targetConfiguration 对象里面的属性赋值.
```java
Optional.ofNullable(this.objectFactory).ifPresent(targetConfiguration::setObjectFactory);

```


这个方法是JAVA8里面判空的方法,如果不为空的话,就会调用对象里面的对象的方法,做赋值的处理。

在498行,如果xmlConfigurationBuilder 不为空,也就是上面的第二种情况了,调用了xmlConfigurationBuilder.parse()去解析配置文件,最终会返回解析好的Configuration 对象.

在第507行,如果没有明确的指定事务工厂,默认使用SpringManagedTransactionFactory,它创建的SpringManagedTransaction 也有getConnection 和close 方法.
```xml
<property name="transactionFactory" value="" />

```
在502行,调用 xmlMapperBuilder.parse(),这个步骤我们之前了解过,它的作用的把接口和对用的MapperProxyFactory 注册到MapperRegistry 中.

最后调用的是sqlSessionFactoryBuilder.build()  返回了一个 DefaultSqlSessionFactory.

OK.我们在这里完成了编程式的案例中的第一步,根据配置文件获取工厂类,它是单例的,会在后来用来创建SqlSession 

用到的Spring 扩展点总结



| 接口                                 | 方法                                | 作用                            |
| ------------------------------------ | ----------------------------------- | ------------------------------- |
| FactoryBean                          | getObject()                         | 返回由FactoryBean创建的Bean实例 |
| InitializingBean                     | afterPropertiesSet()                | bean 属性初始化完成后添加操作   |
| BeanDefinitionRegistryPostProicessor | postProcessBeanDefinitionRegistry() | 注入BeanFinition时添加操作.     |


### 2.  创建SqlSession

####    Q1: 可以直接使用DefaultSqlSession吗？
 我们现在已经有了一个DefaultSqpSessionFactory,按照编程式的开发过程,我们接下来就会创建一个SqlSession的实现类,但是在Spring中,我们不是直接使用DefaultSqlSession的,而是对她进行了一个封装,这个SqlSession的实现类就是SqlSessionTemplate.这个跟spring封装其他的组件也是一样的.比如jdbcTemplate,RedisTemplate等等,也是Spring跟Mybatis最关键的类.

 为什么不用DefaultSqlSession? 它是线程不安全的,注意查看类的注解?
 ```
 Note that this class is not Thread-Safe. 而 SqlSessionTemplate 
 ```
 而SqlSessionTemplate 是线程安全的.
 ```
 * Thread safe, Spring managed, {@code SqlSession} that works with Spring

 ```

回顾一下SqlSession的生命周期



| 对象                     | 生命周期                   |
| ------------------------ | -------------------------- |
| SqlSessionFactoryBuilder | 方法局部(method)           |
| SqlSessionFactory(单例)  | 应用级别(application)      |
| SqlSession               | 请求和操作(request/method) |
| Mapper                   | 方法(method)               |

在编程式开发中,SqlSession在每次请求的时候创建一个,但是Spring里面只有一个SqlSessionTenplate(默认是单例),多个线程同时调用的时候是怎么保证线程安全的？

思考: 为什么SqlSessionTemplate 是线程安全的?

思考:在编程式开发中,有什么方式可以保证SqlSession的线程安全?

SqlSessionTemplate 里面有DefaultSqlSession的所有方法: selectOne(),selectList(),insert(),update(),delete(),不过它都是通过一个代理对象实现的,这个代理对象在构造方法里面通过一个代理类创建.

```java
this.sqlSessionProxy = (SqlSession) newProxyInstance(
SqlSessionFactory.class.getClassLoader(),
new Class[] { SqlSession.class },
new SqlSessionInterceptor());
```
所有的方法都会先走到内部类SqlSessionTenplate 的invoke() 方法.
```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
SqlSession sqlSession = getSqlSession(
SqlSessionTemplate.this.sqlSessionFactory,
SqlSessionTemplate.this.executorType,
SqlSessionTemplate.this.exceptionTranslator);
try {
Object result = method.invoke(sqlSession, args);
```
 首先会使用工厂类,执行器类型,异常解析器创建一个SqlSession,然后调用SqlSession的实现类,实际上就是在这里调用了DefaultSqlSession的方法.

 #### Q2:怎么拿到一个SqlSessionTenplate?
 我们直到在Spring中会使用SqlSessionTemplate 替换DefaultSqlSession.那么接下来看一下怎么在DAO层拿到一个SqlSessionTemplate

  不知道用过Hibernate 的同学还记不记得?如果不使用注入的方式,我们在DAO层注入一个HibernateTemplate的一种方法是什么?

-- 让我们DAO的实现类去继承HibernateDaoSupport

Mybatis 也一样,它提供了一个SqlSessionSupport,里面持有了一个SqlSessionSupport,里面持有了一个SqlSessionTemplate 对象,并且提供了一个getSqlSession() 方法,让我们获取一个SqlSessionTemplate

```java
public abstract class SqlSessionDaoSupport extends DaoSupport {
private SqlSessionTemplate sqlSessionTemplate;
public SqlSession getSqlSession() {
return this.sqlSessionTemplate;
}
前面和后面省略…………
```
 也就是说让我们的DAO 层的实现去继承SqlSessionDaoSupport, 就可以获得SqlSessionTemplate,然后在里面封装SqlSessionTemplate 方法.

 当然,为了减少重复的代码,我们通常不会让我们的实现类去直接继承SqlSessionDaoSupport, 而是先创建一个BaseDao 去继承SqlSessionDaoSupport, 在BaseDao 里面封装对数据库的操作,包括 selectOne(),selectList(),insert(),update(),delete()等这些方法,子类就可以直接调用.
 ```java
 public class BaseDao extends SqlSessionDaoSupport {
//使用 sqlSessionFactory
@Autowired
private SqlSessionFactory sqlSessionFactory;
@Autowired
public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
super.setSqlSessionFactory(sqlSessionFactory);
}
public Object selectOne(String statement, Object parameter) {
return getSqlSession().selectOne(statement, parameter);
}
后面省略…………
 ```
 然后让我们的实现类去继承BaseDao,并且实现我们的Dao接口,这里就是我们 Mapper接口，实现类上需要加上@Repositor注解

 在实现类的方法里面,我们可以直接调用父类(BaseDao)封装的selectOne() 方法,那么它最终会调用 sqlSessionTemplate的 selectOne() 方法.

 ```java
 @Repository
public class EmployeeDaoImpl extends BaseDao implements EmployeeMapper {
@Override
public Employee selectByPrimaryKey(Integer empId) {
Employee emp = (Employee)
this.selectOne("com.crud.dao.EmployeeMapper.selectByPrimaryKey",empId);
return emp;
}
后面省略…………
 ```

 然后在需要的地方,比如service层,注入我们的实现类,调用实现类的方法就行了,
 ```java
 @Autowired
EmployeeDaoImpl employeeDao;
@Test
public void EmployeeDaoSupportTest() {
System.out.println(employeeDao.selectByPrimaryKey(1));
}
 ```
 最终会调用到DefaultSqlSession 的方法

#### Q3: 有没有更好的拿到SqlSessionTemplate 的方法?
这么做有一个问题,我们的每一个DAO 层的接口,如果要拿到一个SqlSesionTemplate 去操作数据库,都要创建实现一个实现类,加上@Repository 注解,j继承BaseDao 这个工作量也不小.

另外一个,我们去直接调用selectOne 方法,还是出现了Statement ID的硬编码,MapperProxy 我们这里根本没用上.

我们可以通过什么方式,不创建任何的实现类,就可以Mapper注入到别的地方使用,并且可以拿到SqlSessionTemplate 操作数据库.

这样也确实我们在Spring中的用法. 那么我们就需要弄清楚,我们只是注入了一个接口,在对象实例化的时候,是怎么拿到SqlSessionTemplate的? 当我们调用方法的时候,还是不是用的MapperProxy ？

### 3. 接口扫描注册
在Service层可以使用@Autowired 自动注入的Mapper接口,需要保存到BeanFactory (比如XmlWebApplicationContext) 中,也就是说接口肯定在Spring启动的时候被扫描注册过了.

1. 什么时候扫描的?
2. 注册的时候,注册的是什么?这个决定了我们拿到的是什么实际对象?
回顾一下,我们在 applicationContext.xml 里面配置了一个MapperScannerConfigurer.

MapperScannerConfigurer 实现了BeanDefinitionRegistryPostProcesser 接口,BeanDefinitionRegistryProcessor是BeanFactoryPostProcessor的子类,可以通过编码的方式修改,新增或者删除某些bean的定义。
![image](http://files.luyanan.com/076e53b4-cc47-4824-a960-87823be614ee.jpg)

我们只需要重写postProcessBeanDefinitionRegistry 方法,在这里面操作Bean就可以了.

在这个方法中,scanner.scan() 方法是ClassPathBeanDefinitionScanner中的,而它的子类ClassPathMapperScanner 覆盖了doScan() 方法,而在doScan() 方法中调用了 processBeanDefinitions:
```java
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);
if (beanDefinitions.isEmpty()) {
LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages) +
"' package. Please check your configuration.");
} else {
processBeanDefinitions(beanDefinitions);
}
return beanDefinitions;
}
```
它先调用父类的soScan() 扫描所有的接口。

processBeanDefinitions 方法里面,在注册beanDefinitions 的时候,BeanClass被改为MapperFactoryBean
```java
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
GenericBeanDefinition definition;
for (BeanDefinitionHolder holder : beanDefinitions) {
definition = (GenericBeanDefinition) holder.getBeanDefinition();
String beanClassName = definition.getBeanClassName();
LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName()
+ "' and '" + beanClassName + "' mapperInterface");
// the mapper interface is the original class of the bean
// but, the actual class of the bean is MapperFactoryBean
definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); //
issue #59
definition.setBeanClass(this.mapperFactoryBean.getClass());
```

问题: 为什么要把BeanClass  修改成MapperFactoryBean,这个类有什么用？

MapperFactoryBean 继承了SqlSessionDaoSupport,可以拿到sqlSessionTemplate

### 4. 接口注入使用
我们使用Mapper 的时候,只需要在加了service 注解的类里面使用@Autowired 注入Mapper接口就可以了.

```java
@Service
public class EmployeeService {
@Autowired
EmployeeMapper employeeMapper;
public List<Employee> getAll() {
return employeeMapper.selectByMap(null);
}
}
```

Spring在启动的时候需要取实例化EmployeeService

EmployeeService 依赖了EmployeeMapper 接口(是EmployeeService 的一个属性)

Spring会根据Mapper的名字从BeanFactory 中获取它的BeanDefinition ,再从BeanDefinition 中获取BeanClass,EmployeeMapper 对应的BeanClass 是MapperFactoryBean.

接下来就是创建MapperFactoryBean,因为实现了FactoryBean 接口,同样是调用getObject() 方法
```java
// MapperFactoryBean.java
public T getObject() throws Exception {
return getSqlSession().getMapper(this.mapperInterface);
}
```

因为MapperFactoryBean 继承了SqlSessionDaoSupport,所以这个 getSqlSession() 方法是调用父类的方法,返回SqlSessionTemplate
```java
// SqlSessionDaoSupport.java
public SqlSession getSqlSession() {
return this.sqlSessionTemplate;
}
```

第二步, SqlSessionTemplate 的getMapper() 方法,里面又有两个方法。
```java
// SqlSessionTemplate.java
public <T> T getMapper(Class<T> type) {
return getConfiguration().getMapper(type, this);
}
```

第一步: SqlSessionoTemplate的 getConfiguration() 方法
```java
// SqlSessionTemplate.java
public Configuration getConfiguration() {
return this.sqlSessionFactory.getConfiguration();
}
```
进入方法,通过 DefaultSqlSessionFactory ,返回全部配置Configuration
```java
// DefaultSqlSessionFactory.java
public Configuration getConfiguration() {
return configuration;
}
```

第二步: Configuration的 getMapper() 方法
```java
// Configuration.java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
return mapperRegistry.getMapper(type, sqlSession);
}
```
这一步我们很熟悉,跟编程式里面的 getMapper() 一样,通过工厂类MapperProxyFactory 获取一个MapperProxy 代理对象

也就是说我们注入Service 里面的接口,实际上还是一个MapperProxy 对象,所以最后调用Mapper 的接口的方法,也就是执行了 MapperProxy.invoke() 方法,后面的流程就跟编程式里面一模一样了.

总结:

| 对象                          | 生命周期                                                     |
| ----------------------------- | ------------------------------------------------------------ |
| SqlSessionTemplate            | spring中SqlSession的替代品,是线程安全的,通过代理模式调用DefaultSqlSession的方法, |
| SqlSessionInterceptor(内部类) | 代理对象,用来代理DefaultSqlSession,在SqlSessioonTemplate  中使用 |
| SqlSessionDapSupport          | 用于获取SqlSessionTemplate,只要继承它就可以了                |
| MapperFactoryBean             | 注册到IOC容器中替换接口类,继承了SqlSessionDaoSupport 用来获取SqlSessionTemplate,因为注入接口的时候,就会调用它的getObject 方法 |
| SqlSessionHolder              | 控制SqlSession和事务                                         |