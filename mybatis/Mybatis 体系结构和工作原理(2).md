# 2. Mybatis 体系结构和工作原理



-----------------
### Mybatis的工作流程分析
  首先,在Mybatis启动的时候解析配置文件,包括全局配置文件和映射器文件,这里面包含了我们怎么去控制Mybatis的行为,和我们要对数据库下达的命令,也就是我们的SQL信息,我们会把他们解析为一个configuration对象
    
接下来就是我们操作数据库的接口,他在应用程序和数据库的中间,代表我们和数据库之间的一次连接,这个就是sqlSession对象.

我们要获得一个会话,必须有个会话工厂sqlSessionFactory.SqiSessionFactory 里面又必须包含我们的所有的配置信息,所以我们会通过一个Builder来创建工厂类.

我们知道,Mybatis是对JDBC的封装,也就是意味着底层一定会出现JDBC的一些核心对象,比如执行SQL的Statement,结果集ResultSet.在Mybatis中SqlSession 只是提供了应用一个接口,还不是SQL的真正执行对象.

在SqlSession中有一个Executor对象,用来封装对数据库的操作.在执行器Executor执行query或者update操作的时候,我们创建了一系列的对象来处理参数,执行SQL，处理结果集.
Mybatis的执行流程如下:
![image](http://files.luyanan.com/f1489826-9f17-4042-a2e9-281d5f7dfaf2.png)

### Mybatis架构分层和模块划分
![image](http://files.luyanan.com/06061f78-afa2-494e-85a9-d8d8e938f36c.jpg)

#### 接口层
首先接口层是我们打交道最多的.核心对象是SqlSession,它是上层应用和Mybatis打交道的桥梁,SqlSession 上定义了非常多的对数据库的操作方法,接口层在接收到调用请求的时候M回调用核心处理层的相应模块来完成具体的数据库操作.

#### 核心处理层
 接下来是核心处理层.既然叫核心处理层,也就是跟数据库操作相关的动作都是在这一层完成

 核心处理层主要做了这几件事情:
 1. 把接口中传入的参数解析并且映射成JDBC类型
 2. 解析XML文件中的SQL语句,包括插入参数,和动态SQL的生成
 3. 执行SQL语句
 4. 处理结果集,并且映射成java对象
 > 插件层也是属于核心层,这是由他的工作方式和拦截的对象决定的 
#### 基础支持层

最后一个就是基础支持层.基础支持层主要是一些抽取出来的通用的功能(实现复用),用来支持核心处理层的功能.比如数据源,缓存,日志,XML解析,反射,IO,事务等这些功能.

### Mybatis  缓存详解
#### cache 缓存
 缓存是一般的ORM框架都是提供的功能,目的就是为为了提升查询的效率和减少数据库的压力.跟Hibernate一样,Mybatis 也有一级缓存和二级缓存,并且预留了继承第三方缓存的接口.
 #### 缓存体系结构
 Mybatis跟缓存相关的类都在cache包里面,其中有一个Cache接口,只有一个默认的实现 PerpetualCache,它是用HashMap实现的.

 除此之外,还有很多的装饰器,通过这些装饰器可以额外实现很多的功能:回收策略,日志记录,定时刷新等.
 ![image](http://files.luyanan.com/94eaa2f7-756f-4b8c-888e-5cf6b2136f72.jpg)
 但是无论怎么装饰,经过多少装饰,最后使用的还是基础的实现类(默认的实现类PerpetualCache)
 所有的缓存实现类总体上可分为三类:基础缓存,淘汰算法缓存,装饰器缓存
| 缓存实现类          | 描述             | 作用                                                         | 装饰条件                                |
| ------------------- | ---------------- | ------------------------------------------------------------ | --------------------------------------- |
| 基本缓存            | 缓存基本实现类   | 默认是PerpetualCche,也可以自定义比如RedisCache,EhCache等,据类基础功能的缓存类 | 无                                      |
| LruCache            | LRU策略的缓存    | 当缓存到达上限的时候,删除最近使用最少的缓存(Least Recently Use) | eviction="LRU"(默认)                    |
| FifoCache           | FIFO策略的缓存   | 当缓存达到上限的时候,删除最先入队的缓存                      | eviction="FIFO"                         |
| SoftCache WeakCache | 带清理策略的缓存 | 通过JVM的软引用和弱引用来实现缓存,当JVM内存不足的时候,会自动清理到这些缓存,基于SoftReference和WeakReference | eviction="SOFT" eviction="WEAK"         |
| LoggingCache        | 带日志功能的缓存 | 比如输出缓存命中率                                           | 基本                                    |
| SynchronizedCache   | 同步缓存         | 基于Synchronized关键字实现,解决并发问题                      | 基本                                    |
| BlockingCache       | 阻塞缓存         | 通过在get/put方式中加锁,保证只有一个线程操作缓存,基于java重入锁实现 | blocking=true                           |
| SerializedCache     | 支持序列化的缓存 | 将对象序列化后放入缓存中,取出是反序列化                      | readOnly=faise(默认)                    |
| ScheduledCache      | 定时调度的缓存   | 在进行get/put/remove/getSize等操作前,判断缓存时间是否超过了设置的最长缓存时间(默认是一个小时),如果是则清空缓存--即每隔一段时间清空一次缓存 | flushInterval不为空                     |
| TransactionalCache  | 事务缓存         | 在二级缓存中使用,可一次存入多个缓存,移除多个缓存             | 在TransactionalCache中用Map维护对应关系 |

 ####  一级缓存
 ##### 一级缓存(本地缓存)介绍
  一级缓存也叫本地缓存,Mybatis的一级缓存默认的在会话(SqlSession)层面进行缓存的,Mybatis的一级缓存默认是开启的,不需要任何的配置.

  首先我们必须去弄清楚一个问题,在Mybatis的执行流程里面,设计到那么多的对象,那么缓存PerpetualCache应该放到哪个对象里面去维护?如果要在同一个会话中共享一级缓存,那么对象肯定是在SqlSession里面创建的,作为SqlSession的一个属性.

  DefaultSqlSession 里面只有两个属性,Configuration 是全局的,所有缓存只可能放到Executor 里面维护-- SimpleExecutor/ReuseExecutor/BatchExecutor  的父类BaseExecutor 的构造函数里面持有了PerpetualCache

  在同一个会话中,多次执行相同的SQL语句,会直接从内存中取到缓存的结果,不会再发送SQL到数据库.但是不同的会话里面,即使执行的SQL是一模一样的(通过一个Mapper的同一个方法的相同参数调用) ,也不能使用到一级缓存

  ![image](http://files.luyanan.com/2f186b8f-f96e-46d8-b6c8-1a2f6776dd6f.png)

  接下来我们验证一下,Mybatis 的一级缓存到底是不是只能在一个会话中共享,以及跨会话(不同的session)操作相同的数据会产生什么问题.
  ##### 一级缓存验证
  判断是否命中缓存: 如果再次发送SQL到数据执行,说明没有命中缓存;如果直接打印对象,则说明是从内存缓存中获取到数据

  1. 在同一个session中共享
  ```java
 BlogMapper mapper = session.getMapper(BlogMapper.class);
System.out.println(mapper.selectBlog(1));
System.out.println(mapper.selectBlog(1));

  ```
2. 不同session 不能共享
```java
SqlSession session1 = sqlSessionFactory.openSession();
BlogMapper mapper1 = session1.getMapper(BlogMapper.class);
System.out.println(mapper.selectBlog(1));
```
> PS:一级缓存在BaseExecutor 的query()---queryFromDatabase()中存入,在queryFromDatabase()之前会get()

3.在 同一个会话中,update(包括delete) 会导致一级缓存被清空
```java
mapper.updateByPrimaryKey(blog);
session.commit();
System.out.println(mapper.selectBlogById(1));
```

一级缓存是在BaseExecutor 中的update()方法调用 clearLocalCache()清空的(无条件),query中会判断

如果出现了跨会话,会出现什么问题呢?
4. 其他会话更新了数据,导致读取到脏数据(一级缓存不能跨会话共享)
```java
// 会话 2 更新了数据，会话 2 的一级缓存更新
BlogMapper mapper2 = session2.getMapper(BlogMapper.class);
mapper2.updateByPrimaryKey(blog);
session2.commit();
// 会话 1 读取到脏数据，因为一级缓存不能跨会话共享
System.out.println(mapper1.selectBlog(1));
```
##### 一级缓存的不足
使用一级缓存的时候,因为缓存不能跨会话共享,不同的会话之间对于相同的数据可能会有不一样的缓存.再有多个会话或者分布式的缓存下,会存在脏数据的问题,如果要解决这个问题,就要使用二级缓存.
#### 二级缓存介绍
二级缓存是用来解决一级缓存不能跨会话共享的问题的,范围是namespace 级别的,可以被多个SqlSession共享(只要是同一个接口里里面的相同的方法,都可以共享),生命周期和应用同步.

思考一个问题:如果开启了二级缓存,二级缓存应该是工作在一级缓存之前还是一级缓存之后呢?二级缓存是在哪里维护的?

作为一个作用范围更广的缓存,他肯定是在SqlSession 的外层,否则不可能被多个SqlSession 共享.而一级缓存是在SqlSession的内部,所以第一个问题,肯定是工作在一级缓存之前,也就是只有取不到二级缓存的情况下才会到一个会话中取一级缓存.

第二个问题,二级缓存存放到哪个对象中维护呢?要跨会话共享的话,SqlSession 本身和它里面的BaseExecutor 已经满足不了需求了,那我们应该在BaseExecutor 外创建一个对象.

实际上,Mybatis是用了一个装饰类来维护,就是CachingExecutor.如果启动了二级缓存,Mybatis 就会在创建Executor 对象的时候就会对Executor 进行装饰.

CacheExecutor 对于查询请求,会判断二级缓存是否有缓存结果,如果有就直接返回,如果没有委派交给真正的查询器Executor 实现类,比如SimpleExecutor来执行查询,再走到一级缓存的流程,最后会把结果缓存起来,并且返回给用户.

![image](http://files.luyanan.com/08ece13d-de7a-49f3-9e61-3d092695d95d.png)
一级缓存是默认开启的,那么二级缓存呢？

##### 开启二级缓存的方法
第一步:在 mybatis-config.xml 中配置了(可以不配置,默认为true)
> <setting name="cacheEnabled" value="true"/>

只要没有显示的设置 cacheEnable=false, 都会用CachingExecutor 装饰基本的执行器.

第二步: 在Mapper.xml文件中配置<cache/> 标签：
```xml
<!-- 声明这个 namespace 使用二级缓存 -->
<cache type="org.apache.ibatis.cache.impl.PerpetualCache"
size="1024" <!—最多缓存对象个数，默认 1024-->
eviction="LRU" <!—回收策略-->
flushInterval="120000" <!—自动刷新时间 ms，未配置时只有调用时刷新-->
readOnly="false"/> <!—默认是 false（安全），改为 true 可读写时，对象必须支持序列
化 -->
```
cache属性讲解
| 属性          | 含义                               | 取值                                                         |
| ------------- | ---------------------------------- | ------------------------------------------------------------ |
| type          | 缓存实现类                         | 需要实现Cache接口,默认是PerpetualCache                       |
| size          | 最多缓存对象个数                   | 默认1024                                                     |
| eviction      | 回收策略(缓存淘汰算法)             | LRU - 最近最少使用的;移除最长时间不被使用的对象(默认) FIFO - 先进先出,按照对象进入缓存的顺序来移除他们; SOFT - 软引用;移除基于垃圾回收器状态和软引用规则的对象; WEAK - 弱引用;更积极的移除基于垃圾回收器状态和弱引用规则的对象 |
| flushInterval | 定时自动清理缓存间隔               | 自动刷新时间,时间 ms,未配置时只有调用时刷新                  |
| readOnly      | 是否只读                           | true:只读缓存,会给所有调用者返回缓存对象的相同实例.因此这些实例不能被修改,这提供了很重要的性能优势.false:读写缓存,会返回对象的拷贝(通过序列化),不会共享,这会慢一些,但是安全,因为默认为false.改为false读写的时候,对象必须支持序列化. |
| blocking      | 是否使用可重入锁实现缓存的并发控制 | true,会使用BlockingCache对Cache进行装饰,默认为false          |

Mapper.xml 配置了<cache>之后,select()会被缓存.update(),delete()和insert()会刷新缓存.

思考:如果cacheEnable=true,Mapper.xml没有配置标签,还有二级缓存吗?还会出现CacheExecutor包装对象吗?

只要 cacheEnable=true 基本执行器就会被装饰.有没有配置<cache>,决定了在启动的时候会不会创建这个mapper对象的Cache对象,最终会影响到CachingExecutor query 方法的判断.

```java
if (cache != null) {

```
如果某些查询方法对数据的实时性要求很高,不需要二级缓存,怎么办?

我们可以在单个statement ID 上 显式关闭二级缓存(默认为true)
```xml
<select id="selectBlog" resultMap="BaseResultMap" useCache="false">

```

了解了二级缓存的工作位置和开启关闭的方式之后,我们来验证一下二级缓存.

##### 二级缓存验证

> 验证二级缓存需要先开启二级缓存

1. 事务不提交,二级缓存不存在
```java
BlogMapper mapper1 = session1.getMapper(BlogMapper.class);
System.out.println(mapper1.selectBlogById(1));
// 事务不提交的情况下，二级缓存不会写入
// session1.commit();
BlogMapper mapper2 = session2.getMapper(BlogMapper.class);
System.out.println(mapper2.selectBlogById(1));
```
思考:为什么事务不提交,二级缓存不生效?

因为二级缓存使用transactionCacheManage(TCM) 来管理,最后又调用了TransactionCache的getObject(),putObject(),和commit()方法,TransactionCache里面又持有了真正的Cache对象,比如是经过层层包装的PerpetualCache对象

在putObject的时候,只是添加到了entriesToAddOnCommit里面,只有它的commit()方法被调用的时候才会调用flushPendingEntries()真正写入缓存,.它就是在DefaultSqlSession调用 commit()的时候被调用.

2. 使用不同的session 和mapper,验证二级缓存可以跨session存在,取消以上 commit()注释
3. 在其他的session中执行增删改查操作,验证缓存会被刷新
```java
Blog blog = new Blog();
blog.setBid(1);
blog.setName("357");
mapper3.updateByPrimaryKey(blog);
session3.commit();
// 执行了更新操作，二级缓存失效，再次发送 SQL 查询
System.out.println(mapper2.selectBlogById(1));
```

**思考:为什么增删改操作会清空缓存?** 

在 CacheingExecutor的 update()方法里面调用 flushCacheIfRequired(ms),isFlushCacheRequired 就是从标签里面取到的 flushCache的值。而增删改操作的flushCache 属性默认为true

#####  什么时候开启二级缓存

  一级缓存默认是打开的,二级缓存需要配置才可以开启.那么我们必须思考一个问题,在什么情况下才有必要去开启二级缓存?

 1. 因为所有的增删改 都会刷新二级缓存,导致二级缓存失效,所有适合在查询喂猪的应用中使用,比如历史交易,历史订单等查询.否则缓存就失去了意义.
 2. 如果多个 namespace中有针对于同一个表的操作,比如Blog 表,如果在一个namespace 中刷新了缓存,另一个namespace 中没有刷新,就会出现脏数据的情况.所以,推荐在一个Mapper里面只操作单表的情况下使用.

**思考:如果要让多个namespace 共享一个二级缓存,应该怎么做**
跨namespace 的缓存共享问题,可以使用<cache-ref>来解决
```xml
<cache-ref namespace="com.crud.dao.DepartmentMapper" />

```

cache-ref 代表引用别的命名空间的Cache配置,两个命令空间的操作使用的是同一个Cache.在关联表比较少,或者按照业务可以对表进行分组的时候可以使用.

注意:在这种情况下,多个Mappper的操作都会引用缓存刷新,缓存的意思已经不大了.

**第三方缓存做二级缓存**
除了Mybatis 自带的二级缓存之外,我们也可以通过实现Cache 接口来自定义二级缓存.

Mybatis官方提供了一些第三方缓存集成方式,比如 ehcache和redis：

 https://github.com/mybatis/redis-cache

 pom文件中引入依赖
 ```xml
 <dependency>
<groupId>org.mybatis.caches</groupId>
<artifactId>mybatis-redis</artifactId>
<version>1.0.0-beta2</version>
</dependency>
 
 ```
 Mapper.xml配置,type 使用RedisCache

```xml
<cache type="org.mybatis.caches.redis.RedisCache"
eviction="FIFO" flushInterval="60000" size="512" readOnly="true"/>
```


redis.properties 配置
```properties
host=localhost
port=6379
connectionTimeout=5000
soTimeout=5000
database=0
```
### Mybatis 源码解读

分析源码,我们从编程式的demo 入手
```java
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
SqlSession session = sqlSessionFactory.openSession();
BlogMapper mapper = session.getMapper(BlogMapper.class);
Blog blog = mapper.selectBlogById(1);
```

把文件读取成流的这一步我们就省略了,接下来我们分成四步解析.

第一步,我们通过建造者模式创建一个工厂类,配置文件的解析就是在这一步完成的,包括mybatis-config.xml和Mapper 适配器文件

第二步,通过sqlSessionFactory创建SqlSession

第三步,获取一个Mapper对象

第四步,调用接口方法.

#### 配置解析过程

首先我们要清楚的是配置解析的过程全部直接洗了两种文件,一个是mybatis-config.xml全部配置文件,另外就是可能有很多个的Mapper.xml文件,也包括在Mapper接口类上定义的注解.

我们从mybatis-config.xml 开始,这里我们具体看一下这里面的标签都是怎么解析的,解析的时候都做了什么？
>SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

首先 我们new了一个SqlSessionFactoryBuilder,非常明显的建造者模式,它里面定义了很多个build 方法的重载,最终返回的是一个SqlSessionFactory 对象(单例模式).我们点进去build 方法.

这里面创建了一个XMLConfigBuilder 对象(Configuration对象也是这个时候创建的).

##### XMLConfigBuilder

XMLConfigBuilder 是抽象类BaseBuilder 的一个子类,专门用来解析全局配置文件,针对不同的构建目标还有其他的一些子类,比如:
- XMLMapperBuider: 解析Mapper映射器
- XMLStateementBuilder: 解析增删改查标签

根据我们解析的文件流,这里后面两个参数都是空的,创建了一个parser.

这里有两部,第一步是调用parser的parser()方法,它会返回一个Configuration 类.

之前我们说过,也就是配置文件里面所有的信息都会放在Configuration里面,Configuration 类里面有很多的属性,有很多是跟config里面的标签直接对应的.

我们先看一下parser()犯法:

首先会检查是不是已经解析过,也就是说在应用的生命周期里面, config配置文件只需要解析一次,生成的Configuration 对象也会存在应用的整个生命周期中,接下来就是parserConfiguration()方法:
```java
parseConfiguration(parser.evalNode("/configuration"));

```
这下面有十几个方法,对应着config文件里面的所有一级标签.

##### propertiesElement()

第一个解析的是<properties> 标签,读取我们引入的外部配置文件.这里面又有两种类型,一种是放在resource目录下的,是相对路径,一种是写得绝对路径.解析的最终结果就是我们会把所有的配置信息方法名为defaults的Properties 对象里面,最后把XPathParser和Configuration 的Properties属性都设置为我们填充后的Properties对象.

##### settingsAsProperties()

第二个,我们把<settings> 标签也解析成了一个Properties 对象,对于<settings>标签的子标签处理在后面.

在早期的版本里面解析和设置都是在后面一起的,这里先解析成Properties 对象是因为后面的两个方法要用到.

##### loadCustomVfs(settings)
loadCustomVfs是 获取Vitual File System 的自定义实现类,比如我们要读取本地文件或者FTP远程文件 时候,就可以用到自定义的VFS类,我们根据<settings> 标签里面的<vfsImpl> 标签,生成了一个抽象类VFS的子类,并且赋值到Configuration 中.

##### loadCustomLogImpl(settings)
loadCustomLogImpl 是根据<logImpl> 标签获取日志的实现类,我们可以用到很多的日志的方案,包括LOG4J,Log4J2,slf4j等,这里生成了一个Log接口的实现类,并且赋值到Configuration中.

##### typeAliasesElement()
接下来,我们解析<typeAliases> 标签,它有两种定义方法,一种是直接定义一个类的别名,一种就是指定一个包,那么这个package下的所有的类的名字都会成为这个类全路径的别名.

类的别名和类的关系,我们都放在一个TypeAliasRegistry 对象里面.

##### pluginElement()
接下来解析<plugins> 标签,比如 PageHelper   的翻页插件,或者我们自定义的插件.<plugins> 标签里面只有<plugin> 标签,<plugin>标签里面只有<property> 标签.

标签解析完之后,会生成一个Interceptor 对象,并且添加到Configuration 的InterceptorChin属性里面,这是一个List

##### objectFactoryElement()、objectWrapperFactoryElement()

接下来的两个标签是用来实例化对象用的.<objectFactory>和<objectWrapperFactory> 这两个标签,分别生成 ObjectFactory和ObjectWrapperFactory 对象,同样设置到Configuration 的属性里面.

#####  reflectorFactoryElement()

解析 reflectorFactory 标签,生成 ReflectorFactory对象


##### settingsElement(settings)

这里就是对<settings> 标签里面所有子标签的处理了,前面我们已经把子标签全部转换成了Properties 对象，所以我们这里只处理Properties 对象就可以了.

二级标签里面有很多的配置,比如二级缓存,延迟加载,自动生成主键这些.需要注意的是,我们之前提到的所有的默认值,都是在这里赋值的.

所有的值,都会赋值到Configuration 的属性里面去.

#####  environmentsElement()
 这一步是解析<environments> 标签.

 我们之前讲过,一个environments 就是对应一个数据源,所以在这里会根据配置的 的<transactionManager>创建一个事务工厂,根据<dataSource> 标签创建一个数据源,最后把这两个对象设置成 Environment 对象的属性,放到Configuration 里面.


##### databaseIdProviderElement()
解析 databaseIdProvider 标签,生成 DatabaseIdProvider对象(用来支持不同厂商的数据库)

##### typeHandlerElement()
跟TypeAlias  一样, typeHandler 有两种配置方式, 一种是单独配置一个类,一种是指定一个 package.我们最后得到的是 javaType和jdbcType,以及用来做相互映射的 TypeHandler之间的映射关系.

最后存放到TypeHandlerRegistry 对象里面.

#####  mapperElement()

http://www.mybatis.org/mybatis-3/zh/configuration.html#mappers

###### 1. 判断
最后就是<Mappers>标签的解析
| 扫描类型 | 含义     |
| -------- | -------- |
| resource | 相对路径 |
| url      | 绝对路径 |
| package  | 包       |
| class    | 单个接口 |

首先会判断是不是接口,只有接口才会解析,然后判断是不是注册了,单个Mapper 重复注册就会抛出异常.

###### 2. 注册
XMLMapperBuilder.parser() 方法,是对Mapper 映射器的解析.里面有两个方法: 

configurationElement() -- 解析所有的子标签,其中 buildStatementFromContext() 最终获得 MappedStatement 对象.

bindMapperForNameSpace() --  把namespace(接口类型)和工厂类绑定起来

无论是按照 package 扫描还是按照接口扫描,租后都会调用到MapperRegistry 的addMapper() 方法.

MapperRegistry 里面维护的其实是一个Map 容器,存储接口和代理工厂的映射关系.

###### 3. 处理注解.

除了映射文件,这里也会去解析Mapper接口方法的注解.在addMapper() 方法里面创建了一个MapperAnnotationBuilder ,我们点进去看一下parser()方法.

parserCach()和parserCacgeRef()方法其实是对 @CacheNamespace和@CacheNamespaceRef 这两个注解的处理.

parserStatement()方法里面的各种getAnnotation() 都是对注解的解析,比如@Options,@SelectKey,@ResultMap等等.

最后同样会解析成MappedStatement对象,也就是是说在XML中配置,和使用注解配置,最后起到一样的效果.,

###### 4. 收尾

如果注解没有完成,还要从Map中remove掉
```java
// MapperRegistry.java
finally {
if (!loadCompleted) {
knownMappers.remove(type);
}
```
最后,MapperRegistry 也会放到Configuration 中

第二步是调用另一个build() 方法,返回DefaultSqlSesionFactory.

###### 总结

在这一步,我们主要完成了 config 配置文件,Mapper文件,Mapper接口上的注解的解析.

我们得到了一个最重要的对象Configuration,这里面存放了全部的配置信息,它在属性里面还有各种各样的容器.

最后,返回了一个DefaultSqlSessionFactory ,里面持有了Configuratrion 的实例.

### 会话创建过程

这是第二步,我们跟数据库的每一次链接,都需要创建一个会话,我们用openSession()方法来创建.

DefaultSqlSessionFactory -- openSessionFromDataSource()

这个会话里面，需要包含一个Executor 用来执行SQL.Executor 又要指定事务类型和执行器的类型。

所以我们会先从 Configuration 里面拿到Enviroment,Enviroment 里面就有事务工厂.

#### 1. 创建Transaction

| 属性    | 产生工厂类                | 产生事务           |
| ------- | ------------------------- | ------------------ |
| JDBC    | JdbcTransactionFactory    | JdbcTransaction    |
| MANAGED | ManagedTransactionFactory | ManagedTransaction |

如果配置的是JDBC，则会使用Connection对象的 commit().rollback(),close()管理事务.

如果配置成MANAGED,就会把事务交给容器来管理,比如JBOSS,Weblogic.以为我们跑的是本地程序,如果配置成MANAGED,不会有任务事务.

如果是Spring+Mybatis,则没有必要配置,因为我们会直接在applicationContext.xml 里面配置数据源和事务管理器,覆盖Mybatis的配置。

#### 2. 创建Executor
我们知道,Executor 的基本类型有三种,SIMPLE,BATCH,REUSE,默认是SIMPLE(settingsElement()读取默认值),他们都继承了抽象类BaseExecutor

为什么要让抽象类实现接口,然后让具体实现类继承抽象类(模板方法模式)
```
“定义一个算法的骨架，并允许子类为一个或者多个步骤提供实现。
模板方法使得子类可以在不改变算法结构的情况下，重新定义算法的某些步骤。”
```

![image](http://files.luyanan.com/92272b7e-c792-4ff5-a27e-17094f9f84ed.jpg)

**问题:三种类型的区别?(通过update 方法对比)**

*SimpleExecutor*：每执行一次 update或者 select,就开启一个Statement对象,用完立即关闭Statement

*ReuseExecutor*: 执行update或者select,以sql作为key查找Statement对象,存在就使用,不存在就创建,用完之后,不关闭Statement对象,而是放置于Map中,供下一次使用.简言之,就是重复使用Statement

*BatchExecutor*:执行update(没有select,批处理不支持select),将所有sql都添加到批处理中(addBatch(),等待统一执行(executeBatch()),它缓存了多个Statement对象,每个Statement对象都是addBatch()完毕后,等待逐一执行executeBatch()批处理,与JDBC批处理相同.

### 获得Mapper对象

现在我们已经有了 DefaultSqlSession了,必须找到Mapper.xml里面定义的Statement ID,才能执行对应的SQL。

找到Statement ID 有两种方式:一种是直接调用session的方法,在参数中传入Statement ID,这种方式属于硬编码,我们没办法知道有多少地方调用,修改起来很麻烦.

另一个问题是如果参数传入错误,在编译阶段是不会报错的,不利于预先发现问题.

```java
Blog blog = (Blog) session.selectOne("com.gupaoedu.mapper.BlogMapper.selectBlogById
", 1);
```
所以在Mybatis后期的版本中提供了第二种方式,就是定义一个接口,然后再调用Mapper接口的方法.

由于我们的接口名称跟Mapper.xml的namespace是对应的,接口的方法跟statement ID 也是对应的,所以根据方法就能找到对应的SQL
```java
BlogMapper mapper = session.getMapper(BlogMapper.class);
```

在这里,我们主要研究一下Mapper 对象是怎么获得的,它的本质是什么?

DefaultSqlSession的 getMapper()方法,调用Configuration 的getMapper()方法

```java
configuration.<T>getMapper()

```

Configuration 的getMapper(),又调用了MapperRegistry的getMapper() 方法
```java
mapperRegistry.getMapper()

```

我们知道,在解析mapper 标签和Mapper.xml 标签的时候已经把接口类型和类型对应的MapperProxyFactory 放到了Map中.获取Mapper代理对象,实际上是从Map中获取了对应的工厂类,调用以下方法创建对象
```java
MapperProxyFactory.newInstance()

```
最后通过代理模式返回代理对象
```java
return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[]
{ mapperInterface }, mapperProxy);
```


**JDK动态代理和Mybatis使用的JDBC动态代理有什么区别?**

JDK动态代理
![image](http://files.luyanan.com/4ef4eb8e-3262-4d2b-a658-47eb39f13ad8.jpg)

JDK动态代理在实现了 InvocationHandler的代理类里面,需要传入一个被代理对象的实现类.

Mybatis 的动态代理
![image](http://files.luyanan.com/e9f7054d-6097-4f4a-ac29-4c20934e3b4b.jpg)

不需要实现类的原因:我们只需要根据接口类型+方法的名字,就可以找到Staement ID了,而唯一要做的一件事情也是这样的,所以不需要实现类,在MapperProxy 里面直接执行逻辑(也就是执行sql) 就可以了.

**总结**
 获取Mapper对象的过程,实质上是获取了一个MapperProxy 对象,MapperProxy中有sqlSession,MapperInterface,methodCache.

 ###  4.执行SQL

 ```java
Blog blog = mapper.selectBlog(1); 
 ```
 由于所有的Mapper 都是MapperProxy 代理对象,所以任意的方法都是执行MapperProxy 的invoke()方法

 我们看一下 invoke()方法
 ###### 1. MapperProxy.invoke()方法
1. 首先判断是否需要去执行SQL,还是直接执行方法
Object 本身的方法和JAVA8 中接口的默认的方法是不需要去执行SQL的

2. 获取缓存
这里加入缓存是为了提升MapperMethod的获取速度
```java
// 获取缓存，保存了方法签名和接口方法的关系
final MapperMethod mapperMethod = cachedMapperMethod(method);
```

Map的 computeIfAbsent()方法:只有key不存在或者value为null的时候才调用 mapperFunction().


###### 2. MapperMethod.execute()
接下来有调用了mapperMethod的execute方法
```java
mapperMethod.execute(sqlSession, args);

```
MapperMethod 里面主要有两个属性,一个是sqlCommand,一个是 MethodSignature, 这两个都是MapperMethod的内部类.

另外定义了多个execute()方法.

在这一步,根据不同的type 和返回类型:

调用 conbvertArgsToSqlCommandParam() 将参数换换为SQL的参数

调用sqlSession 的insert(),update(),delete(),selectOne() 方法,我们以查询为例,会走到selectOne() 方法.

###### 3. DefaultSqlSession.selectOne()
selectOne()方法最终也是调用了selectList() 方法.

在 sleectList() 中,我们根据 command name(Statement ID) 从Configuration 中拿到 MappedStatement ,这个 ms 上面有我们在xml 中配置的所有属性,包括id,statementType,sqlSource,usCache,入参,出参等等.

然后执行 Execute.query()方法.

前面我们说到了Executor 有三种基本类型, SIMPLE/REUSE/BATCH,还有一种包装类型CacheExecute,那么在这里会选择哪一种执行器呢?

我们要回过头看看DefaultSqlSession 在初始化的时候是怎么赋值的,这个就是我们的会话创建过程.

如果我们启动了二级缓存,就会先调用CacheExecute的query()方法,里面有缓存相关的操作,然后才是调用基本类型的执行器,比如默认的SimpleExecutor.

在没有开启二级缓存的情况下,先回走到BaseExecutor 的query()方法,(否则就会先走到CachingExecutor)

![image](http://files.luyanan.com/beadc1aa-7733-4034-8078-72fdfb22e4ee.jpg)

###### 4. BaseExecutor.query() 
1. 创建CacheKey
从Configuration 中获取MappedStatemet,然后从BoundSql 中获取SQL信息,创建CacheKey.这个CacheKey 就是缓存的key

然后在调用另一个query() 方法.

2. 清空本地缓存

queryStack 用于记录查询栈,防止递归查询重复处理缓存。

flushCache=true 的时候,会先清理本地缓存(一级缓存):clearLocalCache();

如果没有清理缓存,会从数据库查询:queryFromDatabase()

如果 LocalCacheScope== STATEMENT,会清理本地缓存。

3.  从数据库查询。
 - 缓存 

             先在缓存用占位符占位.执行查询后，移出占位符,放入数据
 - 查询

             执行Execitor的 doQuery();默认是 SimpleExecutor

######  5.  SimpleExecutor.doQuery() 
1. 创建 StatementHandler  
 在configuration.newStatementHandler() 中,new 一个StatementHandler ,先得到RoutingStatementHandler.

 RoutingStatementHandler 里面没有任何的实现,是用来创建基本的StatementHandler 的.这里会根据MappedStatement 里面的 statementType 决定了StatementHanlder 的类型,默认是 PREPARED(STATEMENT,PREPARD,CALLABLE)
 ```java
 switch (ms.getStatementType()) {
case STATEMENT:
delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler,
boundSql);
break;
case PREPARED:
delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds,
resultHandler, boundSql);
break;
case CALLABLE:
delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds,
resultHandler, boundSql);
break;
default:
throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
}
 ```

StatementHandler 里面包含了处理参数的 ParameterHandler 和处理结果集的 ResultSetHandler 

这两个对象都是上面new的时候创建的
```java
this.parameterHandler = configuration.newParameterHandler(mappedStatement,
parameterObject, boundSql);
this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement,
rowBounds, parameterHandler, resultHandler, boundSql);
```

这三个对象都是可以被插件拦截的四大对象之一,所以在创建之后都要用拦截器进行包装的方法.

```java
statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);

```
```java
parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);

```

```java
resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);

```

2. 创建Statement
用new 出来的StatementHandler 创建Statement 对象 --prepareStatement() 方法对语句进行预编译,处理参数

handler.para,terize(stmt)

3. 执行的 StatementHandler的 query()方法

RoutingStatementHandler 的query() 方法.

delegate委派,最终执行PreparedStatementHandler的query() 方法.

4. 执行 PreparedStatement的execute() 方法
后面的JDBC 包中的PreparedStatement的执行了

5. ResultSetHandler 处理结果集
```java
 return resultSetHandler.handleResultSets(ps);
```

附时序图
01-SQLSessionFactory.
![image](http://files.luyanan.com/caf880fc-5024-45f4-b062-c9831109c08f.jpg)
02-DefaultSqlSession.
![image](http://files.luyanan.com/4919f92e-beef-4174-bca3-001f2078eb0a.jpg)
03-getMapper
![image](http://files.luyanan.com/c5ebe546-3da2-4cda-890b-ed15ea8ce640.jpg)
04-MapperProxy
![image](http://files.luyanan.com/5ed515bd-1471-4cd9-b4f1-0606276df649.jpg)

####  Mybatis 核心对象
| 对象             | 相关对象                                                     | 作用                                                         |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Configuration    | MapperRegistry <br>TypeAliasRegistry <br> TypeHandlerRegistry | 包含了Mybatis的所有的配置信息                                |
| SqlSession       | SqlSessionFactory<br> DefaultSqlSession                      | 对数据库的增删改查的操作进行了封装,提供给应用层使用.         |
| Executor         | BaseExecutor<br>SimpleExecutor<br>BatchExecutor<br>ReuseExecutor | Mybatis 执行器,是Mybatis调度的核心,负责SQL语句的生成和查询缓存的维护. |
| StatementHandler | BaseStatementHandler<br>SimpleStatementHandler<br>PrepardStatementHandler | CallableStatementHandler                                     |
| PreameterHandler | DefaultParmeterHandler                                       | 把用户传递的参数转换给JDBC Statement 所需的参数              |
| ResultSetHandler | DefaultResultHandler                                         | 把JDBC 返回的ResultSet 结果集对象转换为List 类型的集合       |
| MapperProxy      | MapperProxyFactory                                           | 代理对象,用于代理Mapper 接口方法.                            |
| MappedStatement  | sqlSource<br> BoundSql                                       | MappedStatement 维护了 一条<select                           |