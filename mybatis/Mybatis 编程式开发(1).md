----------------
#  1. Mybatis 编程式开发

---------------

### 编程式使用
&nbsp;&nbsp;&nbsp;大部分时候，我们都是在 Spring 里面去集成 MyBatis。因为 Spring 对 MyBatis 的
一些操作进行的封装，我们不能直接看到它的本质，所以先看下不使用容器的时候，也
就是编程的方式，MyBatis 怎么使用。

&nbsp;&nbsp;先引入 mybatis jar 包。
&nbsp;&nbsp;&nbsp;&nbsp;首先我们要创建一个全局配置文件，这里面是对 MyBatis 的核心行为的控制，比如
mybatis-config.xml。

&nbsp;&nbsp;&nbsp;&nbsp;第二个就是我们的映射器文件，Mapper.xml，通常来说一张表对应一个，我们会在
这个里面配置我们增删改查的 SQL 语句，以及参数和返回的结果集的映射关系。

&nbsp;&nbsp;&nbsp;&nbsp;跟 JDBC 的代码一样，我们要执行对数据库的操作，必须创建一个会话，这个在
MyBatis 里面就是 SqlSession。SqlSession 又是工厂类根据全局配置文件创建的。所以
整个的流程就是这样的（如下代码）。最后我们通过 SqlSession 接口上的方法，传入我
们的 Statement ID 来执行 SQL。这是第一种方式。

&nbsp;&nbsp;&nbsp;&nbsp;这种方式有一个明显的缺点，就是会对 Statement ID 硬编码，而且不能在编译时进
行类型检查，所以通常我们会使用第二种方式，就是定义一个 Mapper 接口的方式。这
个接口全路径必须跟 Mapper.xml 里面的 namespace 对应起来，方法也要跟 Statement
ID 一一对应。
```java
public void testMapper() throws IOException {
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new
SqlSessionFactoryBuilder().build(inputStream);
SqlSession session = sqlSessionFactory.openSession();
try {
BlogMapper mapper = session.getMapper(BlogMapper.class);
Blog blog = mapper.selectBlogById(1);
System.out.println(blog);
} finally {
session.close();
}
}
```
这个就是我们单独使用 MyBatis 的全部流程。

这个案例非常重要，后面我们讲源码还是基于它。
#### 核心对象的生命周期
&nbsp;&nbsp;在编程式使用的这个 demo 里面，我们看到了 MyBatis 里面的几个核心对象：
SqlSessionFactoryBuiler、SqlSessionFactory、SqlSession 和 Mapper 对象。这几个
核心对象在 MyBatis 的整个工作流程里面的不同环节发挥作用。如果说我们不用容器，自己去管理这些对象的话，我们必须思考一个问题：什么时候创建和销毁这些对象？

在一些分布式的应用里面，多线程高并发的场景中，如果要写出高效的代码，必须
了解这四个对象的生命周期。这四个对象的声明周期的描述在官网上面也可以找到。

[http://www.mybatis.org/mybatis-3/zh/getting-started.html](http://www.mybatis.org/mybatis-3/zh/getting-started.html)

我们从每个对象的作用的角度来理解一下，只有理解了它们是干什么的，才知道什
么时候应该创建，什么时候应该销毁。

##### 1）SqlSessionFactoryBuiler
首 先 是 SqlSessionFactoryBuiler 。 它 是 用 来 构 建 SqlSessionFactory 的 ， 而
SqlSessionFactory 只需要一个，所以只要构建了这一个 SqlSessionFactory，它的使命
就完成了，也就没有存在的意义了。所以它的生命周期只存在于==方法的局部==。
##### 2）SqlSessionFactory
SqlSessionFactory 是用来创建 SqlSession 的，每次应用程序访问数据库，都需要
创建一个会话。因为我们一直有创建会话的需要，所以 SqlSessionFactory 应该存在于
应用的整个生命周期中（==作用域是应用作用域==）。创建 SqlSession 只需要一个实例来做
这件事就行了，否则会产生很多的混乱，和浪费资源。所以我们要采用单例模式
##### 3）SqlSession
SqlSession 是一个会话，因为它不是线程安全的，不能在线程间共享。所以我们在
请求开始的时候创建一个 SqlSession 对象，在请求结束或者说方法执行完毕的时候要及
时关闭它（==一次请求或者操作中==）。
##### 4）Mapper
Mapper（实际上是一个代理对象）是从 SqlSession 中获取的。
```java
BlogMapper mapper = session.getMapper(BlogMapper.class);
```
它的作用是发送 SQL 来操作数据库的数据。它应该在一个 SqlSession 事务方法之
内。

最后总结如下：

| 对象                      | 生命周期                     |
| ------------------------- | ---------------------------- |
| SqlSessionFactoryBuiler   | 方法局部（method）           |
| SqlSessionFactory（单例） | 应用级别（application）      |
| SqlSession                | 请求和操作（request/method） |
| Mapper                    | 方法（method）               |
这个就是我们在编程式的使用里面看到的四个对象的生命周期的总结。

#### 核心配置解读
第一个是 config 文件。大部分时候我们只需要很少的配置就可以让 MyBatis 运行起
来。其实 MyBatis 里面提供的配置项非常多，我们没有配置的时候使用的是系统的默认
值。
mybatis-3 的 源 码 托 管 在 github 上 。 源 码 地 址
https://github.com/mybatis/mybatis-3/releases

目前最新的版本是 3.5.1，大家可以从官方上下载到最新的源码。

第一个是 jar 包和文档。第二个第三个是源码。

在这个压缩包里面，解压出来有一个 mybatis-3.5.1.pdf，是英文版本的。如果阅读
英文困难，可以基于 3.5.1 的中文版本学习
http://www.mybatis.org/mybatis-3/zh/index.html
##### 一级标签
###### configuration
configuration 是整个配置文件的根标签，实际上也对应着 MyBatis 里面最重要的
配置类 Configuration。它贯穿 MyBatis 执行流程的每一个环节。我们打开这个类看一
下，这里面有很多的属性，跟其他的子标签也能对应上。

注意：MyBatis 全局配置文件顺序是固定的，否则启动的时候会报错。
###### properties
第一个是 properties 标签，用来配置参数信息，比如最常见的数据库连接信息。

为了避免直接把参数写死在 xml 配置文件中，我们可以把这些参数单独放在
properties 文件中，用 properties 标签引入进来，然后在 xml 配置文件中用${}引用就
可以了。

可以用 resource 引用应用里面的相对路径，也可以用 url 指定本地服务器或者网络的绝对路径。

我们为什么要把这些配置独立出来？有什么好处？或者说，公司的项目在打包的时
候，有没有把 properties 文件打包进去？
1.  提取，利于多处引用，维护简单；
2.  把配置文件放在外部，避免修改后重新编译打包，只需要重启应用；
3.  程序和配置分离，提升数据的安全性，比如生产环境的密码只有运维人员掌握。
###### settings
setttings 里面是 MyBatis 的一些核心配置，我们最后再看，先看下其他的以及标签。
###### typeAliases
TypeAlias 是类型的别名，跟 Linux 系统里面的 alias 一样，主要用来简化全路径类
名的拼写。比如我们的参数类型和返回值类型都可能会用到我们的 Bean，如果每个地方
都配置全路径的话，那么内容就比较多，还可能会写错。

我们可以为自己的 Bean 创建别名，既可以指定单个类，也可以指定一个 package，
自动转换。配置了别名以后，只需要写别名就可以了，比如 com.domain.Blog
都只要写 blog 就可以了。

MyBatis 里面有系统预先定义好的类型别名，在 TypeAliasRegistry 中。

###### typeHandlers【重点】
由于 Java 类型和数据库的 JDBC 类型不是一一对应的（比如 String 与 varchar），
所以我们把 Java 对象转换为数据库的值，和把数据库的值转换成 Java 对象，需要经过
一定的转换，这两个方向的转换就要用到 TypeHandler。

有的同学可能会有疑问，我没有做任何的配置，为什么实体类对象里面的一个 String
属性，可以保存成数据库里面的 varchar 字段，或者保存成 char 字段？

这是因为 MyBatis 已经内置了很多 TypeHandler（在 type 包下），它们全部全部
注册在 TypeHandlerRegistry 中，他们都继承了抽象类 BaseTypeHandler，泛型就是要处理的 Java 数据类型。

当我们做数据类型转换的时候，就会自动调用对应的 TypeHandler 的方法。

如果我们需要自定义一些类型转换规则，或者要在处理类型的时候做一些特殊的动
作，就可以编写自己的 TypeHandler，跟系统自定义的 TypeHandler 一样，继承抽象类
BaseTypeHandler<T>。有 4 个抽象方法必须实现，我们把它分成两类：

set 方法从 Java 类型转换成 JDBC 类型的，get 方法是从 JDBC 类型转换成 Java 类
型的。
| 从java类型到JDBC类型             | 从JDBC类型到JAVA类型                                         |
| -------------------------------- | ------------------------------------------------------------ |
| setNonNullParameter:设置非空参数 | getNullableResult：获取空结果集（根据列名），一般都是调用这个getNullableResult：获取空结果集（根据下标值）getNullableResult：存储过程用的 |

比如我们想要在获取或者设置 String 类型的时候做一些特殊处理，我们可以写一个
String 类型的 TypeHandler
```java
public class MyTypeHandler extends BaseTypeHandler<String> {
public void setNonNullParameter(PreparedStatement ps, int i, String parameter,
JdbcType jdbcType)
throws SQLException {
// 设置 String 类型的参数的时候调用，Java 类型到 JDBC 类型
System.out.println("---------------setNonNullParameter1："+parameter);
ps.setString(i, parameter);
}
public String getNullableResult(ResultSet rs, String columnName) throws SQLException
{
// 根据列名获取 String 类型的参数的时候调用，JDBC 类型到 java 类型
System.out.println("---------------getNullableResult1："+columnName);
return rs.getString(columnName);
}
// 后面两个方法省略…………
}
```

第二步，在 mybatis-config.xml 文件中注册：

```java
<typeHandlers>
<typeHandler handler="com.type.MyTypeHandler"></typeHandler>
</typeHandlers>
```
第三步，在我们需要使用的字段上指定，比如：
插入值的时候，从 Java 类型到 JDBC 类型，在字段属性中指定 typehandler：
```java
<insert id="insertBlog" parameterType="com.domain.Blog">
insert into blog (bid, name, author_id)
values (#{bid,jdbcType=INTEGER},
#{name,jdbcType=VARCHAR,typeHandler=com.type.MyTypeHandler},
#{authorId,jdbcType=INTEGER})
</insert>
```
返回值的时候，从 JDBC 类型到 Java 类型，在 resultMap 的列上指定 typehandler：
```java
<result column="name" property="name" jdbcType="VARCHAR"
typeHandler="com.type.MyTypeHandler"/>
```
###### objectFactory【重点】
当我们把数据库返回的结果集转换为实体类的时候，需要创建对象的实例，由于我
们不知道需要处理的类型是什么，有哪些属性，所以不能用 new 的方式去创建。在
MyBatis 里面，它提供了一个工厂类的接口，叫做 ObjectFactory，专门用来创建对象的
实例，里面定义了 4 个方法。
| 方法                                                         | 作用                           |
| ------------------------------------------------------------ | ------------------------------ |
| void setProperties(Properties properties);                   | 设置参数时调用                 |
| <T> T create(Class<T> type);                                 | 创建对象（调用无参构造函数）   |
| <T> T create(Class<T> type, List<Class<?>> constructorArgTypes, List<Object>constructorArgs); | 创建对象（调用带参数构造函数） |
| <T> boolean isCollection(Class<T> type)                      | 判断是否集合                   |

ObjectFactory 有一个默认的实现类 DefaultObjectFactory，创建对象的方法最终
都调用了 instantiateClass()，是通过反射来实现的。

如果想要修改对象工厂在初始化实体类的时候的行为，就可以通过创建自己的对象
工厂，继承 DefaultObjectFactory 来实现（不需要再实现 ObjectFactory 接口）。


例如：
```java
public class MyObjectFactory extends DefaultObjectFactory {

@Override
public Object create(Class type) {
if (type.equals(Blog.class)) {
Blog blog = (Blog) super.create(type);
blog.setName("by object factory");
blog.setBid(1111);
blog.setAuthorId(2222);
return blog;
}
Object result = super.create(type);
return result;
}
}
```
我们可以直接用自定义的工厂类来创建对象
```java
public class ObjectFactoryTest {
public static void main(String[] args) {
MyObjectFactory factory = new MyObjectFactory();
Blog myBlog = (Blog) factory.create(Blog.class);
System.out.println(myBlog);
}
}
```
这样我们就直接拿到了一个对象
如果在 config 文件里面注册，在创建对象的时候会被自动调用：
如果在 config 文件里面注册，在创建对象的时候会被自动调用：

```xml
<objectFactory type="org.mybatis.example.MyObjectFactory">
<!-- 对象工厂注入的参数 -->
<property name="name" value="666"/>
</objectFactory>
```

这样，就可以让 MyBatis 的创建实体类的时候使用我们自己的对象工厂。
###### plugins

插件是 MyBatis 的一个很强大的机制，跟很多其他的框架一样，MyBatis 预留了插
件的接口，让 MyBatis 更容易扩展。

根据官方的定义，插件可以拦截这四个对象的这些方法，我们把这四个对象称作
MyBatis 的四大对象。我们会在带大家阅读源码，知道了这 4 大对象的作用之后，再来
分析自定义插件的开发和插件运行的原理。
http://www.mybatis.org/mybatis-3/zh/configuration.html#plugins

| 类（或接口）     | 方法                                                         |
| ---------------- | ------------------------------------------------------------ |
| Executor         | update, query, flushStatements, commit, rollback, getTransaction, close, isClosed |
| ParameterHandler | getParameterObject, setParameters                            |

ResultSetHandler| handleResultSets, handleOutputParameters
StatementHandler |prepare, parameterize, batch, update, query
###### environments、environment
environments 标签用来管理数据库的环境，比如我们可以有开发环境、测试环境、
生产环境的数据库。可以在不同的环境中使用不同的数据库地址或者类型。
```xml
<environments default="development">
<environment id="development">
<transactionManager type="JDBC"/>
<dataSource type="POOLED">
<property name="driver" value="com.mysql.jdbc.Driver"/>
<property name="url"
value="jdbc:mysql://127.0.0.1:3306/-mybatis?useUnicode=true"/>
<property name="username" value="root"/>
<property name="password" value="123456"/>
</dataSource>
</environment>
</environments>
```
一个 environment 标签就是一个数据源，代表一个数据库。这里面有两个关键的标
签，一个是事务管理器，一个是数据源。

###### transactionManager
如果配置的是 JDBC，则会使用 Connection 对象的 commit()、rollback()、close()管理事务

如果配置成 MANAGED，会把事务交给容器来管理，比如 JBOSS，Weblogic。因
为我们跑的是本地程序，如果配置成 MANAGE 不会有任何事务。

如 果 是 Spring + MyBatis ， 则 没 有 必 要 配 置 ， 因 为 我 们 会 直 接 在
applicationContext.xml 里面配置数据源，覆盖 MyBatis 的配置。

###### dataSource
将在下一节（settings）详细分析。在跟 Spring 集成的时候，事务和数据源都会交
给 Spring 来管理。
###### mappers
<mappers>标签配置的是我们的映射器，也就是 Mapper.xml 的路径。这里配置的
目的是让 MyBatis 在启动的时候去扫描这些映射器，创建映射关系。

我们有四种指定 Mapper 文件的方式：

http://www.mybatis.org/mybatis-3/zh/configuration.html#mappers

1. 使用相对于类路径的资源引用（resource）
2. 使用完全限定资源定位符（绝对路径）（URL）
3. 使用映射器接口实现类的完全限定类名
4. 将包内的映射器接口实现全部注册为映射器（最常用）

###### settings

最后 settings 我们来单独说一下，因为 MyBatis 的一些最关键的配置都在这个标签
里面（只讲解一些主要的）。

| 属性名                           | 含义                                                         | 简介                                                         | 有效值                                                       | 默认值                                                |
| -------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ----------------------------------------------------- |
| cacheEnabled                     | 是否使用缓存                                                 | 是整个工程中所有映射器配置缓存的开关，即是一个全局缓存开关   | true/false                                                   | true                                                  |
| lazyLoadingEnabled               | 是否开启延迟加载                                             | 控制全局是否使用延迟加载（association、collection）。当有特殊关联关系需要单独配置时，可以使用 fetchType 属性来覆盖此配置 | true/false                                                   | false                                                 |
| aggressiveLazyLoading            | 是否需要侵入式延迟加载                                       | 开启时，无论调用什么方法加载某个对象，都会加载该对象的所有属性，关闭之后只会按需加载 | true/false                                                   | false                                                 |
| defaultExecutorType              | 设置默认的执行器                                             | 有三种执行器：SIMPLE 为普通执行器；REUSE 执行器会重用与处理语句；BATCH 执行器将重用语句并执行批量更新 | SIMPLE/REUSE/BATCH                                           | SIMPLE                                                |
| lazyLoadTriggerMethods           | 指定哪个对象的方法触发一次延迟加载                           | 配置需要触发延迟加载的方法的名字，该方法就会触发一次延迟加载 | 一个逗号分隔的方法名称列表                                   | equals，clone，hashCode，toString                     |
| localCacheScope                  | MyBatis 利用本地缓存机制（LocalCache）防止循环引用（circularreferences）和加速重复嵌套查询 | 默认值为 SESSION，这种情况下会缓存一个会话中执行的所有查询。若设置值为STATEMENT，本地会话仅用在语句执行上，对相同 SqlSession 的不同调用将不会共享数据 | SESSION/STATEMEN T                                           | SESSION                                               |
| logImpl                          | 日志实现                                                     | 指定 MyBatis 所用日志的具体实现，未指定时将自动查找          | SLF4J、LOG4J、LOG4J2、JDK_LOGGING、COMMONS_LOGGING、STDOUT_LOGGING、NO_LOGGING | 无                                                    |
| multipleResultSetsEnabled        | 是否允许单一语句返回多结果集                                 | 即 Mapper 配置中一个单一的 SQL 配置是否能返回多个结果集      | true/false                                                   | true                                                  |
| useColumnLabel                   | 使用列标签代替列名                                           | 设置是否使用列标签代替列名                                   | true/false                                                   | true                                                  |
| useGeneratedKeys                 | 是否支持 JDBC 自动生成主键                                   | 设置之后，将会强制使用自动生成主键                           |                                                              |                                                       |
| 的策略                           | true/false                                                   | false                                                        |                                                              |                                                       |
| autoMappingBehavior              | 指定 MyBatis 自动 映射字段或属性的方式                       | 有三种方式，NONE时将取消自动映射；PARTIAL时只会自动映射没有定义结果集的结果映射；FULL 时会映射任意复杂的结果集 | NONE/PARTIAL/FULL PARTIAL                                    |                                                       |
| autoMappingUnknownColumnBehavior | 设置当自动映射时发现未知列的动作                             | 有三种动作：NONE 时不做任何操作；WARNING 时会输出提醒日志；FAILING时会抛出 SqlSessionException 异常表示映射失败 | NONE/WARNING/FAILING                                         | NONE                                                  |
| defaultStatementTimeout          | 设置超时时间                                                 | 该超时时间即数据库驱动连接数据库时，等待数据库回应的最大秒数 | 任意正整数                                                   | 无                                                    |
| defaultFetchSize                 | 设置驱动的结果集获取数量（fetchSize）的提示值                | 为了防止从数据库查询出来的结果过多，而导致内存溢出，可以通过设置fetchSize 参数来控制结果集的数量 | 任意正整数                                                   | 无                                                    |
| safeRowBoundsEnabled             | 允许在嵌套语句中使用分页（RowBound，即行内嵌套语句）         | 如果允许在 SQL 的行内嵌套语句中使用分页，就设置该值为 false  | true/false                                                   | false                                                 |
| safeResultHandlerEnabled         | 允许在嵌套语句中使用分页（ResultHandler，即结果集处理）      | 如果允许在 SQL 的结果集使用分页，就设置该值为 false          | true/false                                                   | true                                                  |
| mapUnderscoreToCamelCase         | 是否开启驼峰命名规则（camel case）映射                       | 表明数据库中的字段名称与工程中Java 实体类的映射是否采用驼峰命名规则校验 | true/false                                                   | false                                                 |
| jdbcTypeForNull                  | JDBC类型的默认设置                                           | 当没有为参数提供特定的 JDBC 类型时，为空值指定 JDBC 类型。某些驱动需要指定列的 JDBC 类型，多数情况直接用一般类型即可，比如 NULL、VARCHAR 或 OTHER | 常用 NULL、VARCHAR、OTHER                                    | OTHER                                                 |
| defaultScriptingLanguage         | 动态 SQL 默认语言                                            | 指定动态 SQL 生成的默认语言                                  | 一个类型别名或者一个类的全路径名                             | org.apache.ibatis.scripting.xmltags.XMLLanguageDriver |
| callSettersOnNulls               | 是否在控制情况下调用 Set 方法                                | 指定当结果集中值为 null 时是否调用映射对象的 sette（r map 对象时为 put）方法，这对于有 Map.keySet()依赖或null 值初始化时是有用的。注意基本类型是不能设置成 null 的 | true/false                                                   | false                                                 |
| returnInstanceForEmptyRow        | 返回空实体集对象                                             | 当返回行的所有列都是空时，MyBatis默认返回 null。当开启这个设置时，MyBatis 会返回一个空实例。请注意，它也适用于嵌套的结果集（从MyBatis3.4.2 版本开始） | true/false                                                   | false                                                 |
| logPrefix                        | 日志前缀                                                     | 指定 MyBatis 所用日志的具体实现，未指定时将自动查找          | 任意字符串                                                   | 无                                                    |
| vfsImpl                          | vfs 实现                                                     | 指定 vfs 的实现                                              | 自定义 VFS 的实现的类的全限定名，以逗号分隔                  | 无                                                    |
| useActualParamName               | 使用方法签名 允许使用方法签名中的名称作为语句参数名称。要使用该特性，工程必须采用 Java8 编译，并且加上-parameters选项（从 MyBatis3.4.1 版本开始） | 自定义 VFS 的实现的类的全限定名，以逗号分隔                  | 无                                                           |                                                       |
| configurationFactory             | 配置工厂                                                     | 指定提供配置示例的类。返回的配置实例用于加载反序列化的懒加载参数。这个类必须有一个签名的静态配置getconfiguration()方法（从MyBatis3.2.3 版本开始） | 一个类型别名或者一个类型的全路径名                           | 无                                                    |

### Mapper.xml 映射配置文件
http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html

&nbsp;&nbsp;映射器里面最主要的是配置了 SQL语句，也解决了我们的参数映射和结果集映射的问题。一共有 8 个标签:
- cache- 给定命名空间的缓存配置(是否开启二级缓存)
- cache-ref-其他命名空间缓存配置的引用.
- resultMap- 是最复杂也是最强大的元素,用来描述如何从数据库结果集中加载对象
```xml
<resultMap id="BaseResultMap" type="Employee">
<id column="emp_id" jdbcType="INTEGER" property="empId"/>
<result column="emp_name" jdbcType="VARCHAR" property="empName"/>
<result column="gender" jdbcType="CHAR" property="gender"/>
<result column="email" jdbcType="VARCHAR" property="email"/>
<result column="d_id" jdbcType="INTEGER" property="dId"/>
</resultMap>
```
- sql- 可被其他语句引用的可重用语句块
```xml
<sql id="Base_Column_List">
emp_id, emp_name, gender, email, d_id
</sql>
```
- 增删改查标签:
- - insert: 映射插入语句
- - update: 映射修改语句
- - delete: 映射删除语句
- - select: 映射查询语句
### 总结
| 配置名称           | 配置含义           | 配置简介                                                     |
| ------------------ | ------------------ | ------------------------------------------------------------ |
| configuration      | 包裹所有的配置标签 | 整个配置文件的顶级标签                                       |
| properties         | 属性               | 该标签可以引入外部配置的属性,也可以自己配置.该配置标签所在的同一个配置的其他配置均可以引用此配置的属性 |
| setting            | 全局配置参数       | 用来配置一些改变运行时行为的信息,例如 是否使用缓存机制,是否使用延迟加载,是否使用错误处理机制等. |
| typeAliases        | 类型别名           | 用来设置一些别名来代码java的长类型声明(如java.lang.int变为int),减少配置编码的冗余 |
| typeHandlers       | 类型处理器         | 将数据库获取的值以合适的方法转换为java类型,或者将java类型的参数转换为数据库对应的类型 |
| objectFactory      | 对象工厂           | 实例化目标类的工厂类配置                                     |
| plugins            | 插件               | 可以通过插件修改Mybatis的核心行为,例如对语句执行的某一点进行拦截调用. |
| environments       | 环境集合属性对象   | 数据库环境信息的集合,在同一个配置文件中,可以有多个数据库环境即可,这样可以使Mybatis将sql同时映射到多个数据库 |
| environment        | 环境子属性对象     | 数据库环境配置的详细配置                                     |
| transactionManager | 事务管理           | 指定Mybatis的事务管理器                                      |
| dataSource         | 数据源             | 使用其中的type指定数据源的链接类型,在标签对中可以使用property属性指定数据库连接池的其他信息 |
| mappers            | 映射器             | 配置SQL映射文件的位置,告知Mybatis去哪里加载SQL映射文件       |

#### Mybatis最佳实践
##### 为什么需要动态sql？
由于前台传入的查询参数不同,所以写了很多的if else,还需要非常注意SQL里面的and,空格,逗号和转义的单引号这些,拼接和调试SQL就是一件非常耗时的工作.
 Mybatis的动态SQL就帮助我们解决了这个问题,他是基于OGNL表达式的
##### 动态标签有哪些?
按照官网的分类,Mybatis的动态标签主要有四类:if,choose(when,otherwise),trim(where,set),foreaach.
######  if 需要判断的时候,条件写在test中
以下语句可以用<where> 改写
```xml
<select id="selectDept" parameterType="int"
resultType="com.crud.bean.Department">
select * from tbl_dept where 1=1
<if test="deptId != null">
and dept_id = #{deptId,jdbcType=INTEGER}
</if>
</select>
```
#####  choose(when,otherwise) --需要选择一个条件的时候
```xml
<select id="getEmpList_choose" resultMap="empResultMap"
parameterType="com.crud.bean.Employee">
SELECT * FROM tbl_emp e
<where>
<choose>
<when test="empId !=null">
e.emp_id = #{emp_id, jdbcType=INTEGER}
</when>
<when test="empName != null and empName != ''">
AND e.emp_name LIKE CONCAT(CONCAT('%', #{emp_name,
jdbcType=VARCHAR}),'%')
</when>
<when test="email != null ">
AND e.email = #{email, jdbcType=VARCHAR}
</when>
<otherwise>
</otherwise>
</choose>
</where>
</select>
```
######  trim(where,set)--需要去掉where,and,逗号之类的符号的时候
主要最后一个条件 did多个一个逗号,就是用trim去掉的
```xml
<update id="updateByPrimaryKeySelective"
parameterType="com.crud.bean.Employee">
update tbl_emp

<set>
<if test="empName != null">
emp_name = #{empName,jdbcType=VARCHAR},
</if>
<if test="gender != null">
gender = #{gender,jdbcType=CHAR},
</if>
<if test="email != null">
email = #{email,jdbcType=VARCHAR},
</if>
<if test="dId != null">
d_id = #{dId,jdbcType=INTEGER},
</if>
</set>
where emp_id = #{empId,jdbcType=INTEGER}
</update>
```
######  trim用来指定或者去掉前缀或者后缀
```xml
<insert id="insertSelective" parameterType="com.crud.bean.Employee">
insert into tbl_emp
<trim prefix="(" suffix=")" suffixOverrides=",">
<if test="empId != null">
emp_id,
</if>
<if test="empName != null">
emp_name,
</if>
<if test="dId != null">
d_id,
</if>
</trim>
<trim prefix="values (" suffix=")" suffixOverrides=",">
<if test="empId != null">
#{empId,jdbcType=INTEGER},
</if>
<if test="empName != null">
#{empName,jdbcType=VARCHAR},
</if>
<if test="dId != null">
#{dId,jdbcType=INTEGER},
</if>
</trim>
</insert>
```

###### foreach---需要遍历集合的时候
```xml
<delete id="deleteByList" parameterType="java.util.List">
delete from tbl_emp where emp_id in
<foreach collection="list" item="item" open="(" separator="," close=")">
#{item.empId,jdbcType=VARCHAR}
</forea
```
动态SQL主要是用来解决SQL语句生成的问题
#####  批量操作
   我们在生产的项目中会有一个批量操作的场景,比如导入文件批量处理数据的情况(批量新增商户,批量修改商户信息),当数据量非常大,比如超过几万条的时候,在java代码中循环发送SQL到数据库执行肯定是不现实的,因为这意味着要跟数据库创建几万次绘画,即使我们使用了数据库连接池技术,对于数据库服务器来说也是不堪重负的

   在Mybatis里面支持批量操作的,包括批量的插入,更新,删除.我们可以直接传入一个List,Set,Map或者数组,配置动态SQL的标签,Mybatis会自动帮我们生成语法正确的SQL语句
###### 批量插入
批量插入的语法是这样的,只要在value后面增加插入的值就可以了
```mysql
insert into tbl_emp (emp_id, emp_name, gender,email, d_id) values ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , ( ?,?,?,?,? ) , 
```
在Mapper文件里面,我们可以使用foreach标签拼接values部分的语句
```xml
<!-- 批量插入 -->
<insert id="batchInsert" parameterType="java.util.List" useGeneratedKeys="true">
<selectKey resultType="long" keyProperty="id" order="AFTER">
SELECT LAST_INSERT_ID()
</selectKey>
insert into tbl_emp (emp_id, emp_name, gender,email, d_id)
values
<foreach collection="list" item="emps" index="index" separator=",">
( #{emps.empId},#{emps.empName},#{emps.gender},#{emps.email},#{emps.dId} )
</foreach>
</insert>
```
java代码里面,直接传入一个List类型的参数

  我们来测试一下.效率要比循环发送SQL执行的要高的多,最关键的地方就在于减少了跟数据库交互的次数,并且避免了开启和结束事务的时间消耗。
######  批量更新
批量更新的语法是这样的,通过case when,来匹配id相关的字段值
```mysql
update tbl_emp set
emp_name =
case emp_id
when ? then ?
when ? then ?
when ? then ? end ,
gender =
case emp_id
when ? then ?
when ? then ?
when ? then ? end ,
email =
case emp_id
when ? then ?
when ? then ?
when ? then ? end
where emp_id in ( ? , ? , ? )
```
所以在Mapper文件里面最关键的就是case when和where的配置

需要注意一下open属性和separator属性
```xml
<update id="updateBatch">
update tbl_emp set
emp_name =
<foreach collection="list" item="emps" index="index" separator=" " open="case emp_id"
close="end">
when #{emps.empId} then #{emps.empName}
</foreach>
,gender =
<foreach collection="list" item="emps" index="index" separator=" " open="case emp_id"
close="end">
when #{emps.empId} then #{emps.gender}
</foreach>
,email =
<foreach collection="list" item="emps" index="index" separator=" " open="case emp_id"
close="end">
when #{emps.empId} then #{emps.email}
</foreach>
where emp_id in
<foreach collection="list" item="emps" index="index" separator="," open="("
close=")">
#{emps.empId}
</foreach>
</update>
```
批量删除也是类似的
###### Batch Executor
当然Mybatis的动态标签的批量操作也是存在一定的缺点的,比如数据量比较大的时候,拼接出来的SQL语句过大.

MySql的服务端对于接受的数据包有大小限制,max_allowed_packet 默认是4M，需要修改默认配置才可以解决这个问题
```
Caused by: com.mysql.jdbc.PacketTooBigException: Packet for query is too large (7188967 >
4194304). You can change this value on the server by setting the max_allowed_packet' variable.
```
在我们的全局配置文件中,可以配置默认的Executor的类型,其中有一种BatchExecutor
> <setting name="defaultExecutorType" value="BATCH" />
> 也可以在创建会话的时候指定执行器的类型
> SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH);

BatchExecutor 底层的对JDBC ps.addBatch()的封装,原理是攒一批SQL以后再发送

######  嵌套(关联)查询/N+1/延迟加载
我们在查询业务数据的时候，经常会遇到跨表关联的情况,比如查询员工就会关联部门,查询成绩就会关联课程,查询订单就会关联商品等等

我们映射结果有两个标签,一个是resultType,一个是resultMap.

resultType是select标签的一个属性,适用于返回JDK类型(如Integer,String等等)和实体类.这种情况下结果集的列和实体类的属性可以直接映射,如果返回的字段无法直接映射,就要欧阳那个resultMap来建立映射关系.

  对于关联查询的这种情况,通常不能用resultType来映射,要么就是修改dto,在里面增加字段,这个会导致增加很多无关的字段,要么就是引用关联的对象,比如Blog里面包含一个Author对象，这种情况下就要用到关联查询(assocation,或者嵌套查询),Mybatis可以帮我们自动做结果的映射.

###### 一对一的关联查询有两种配置方式:
1. 嵌套结果
```xml
<!-- 根据文章查询作者，一对一查询的结果，嵌套查询 -->
<resultMap id="BlogWithAuthorResultMap"
type="com.domain.associate.BlogAndAuthor">
<id column="bid" property="bid" jdbcType="INTEGER"/>
<result column="name" property="name" jdbcType="VARCHAR"/>
<!-- 联合查询，将 author 的属性映射到 ResultMap -->
<association property="author" javaType="com.gupaoedu.domain.Author">
<id column="author_id" property="authorId"/>
<result column="author_name" property="authorName"/>
</association>
</resultMap>
```
2. 嵌套查询:
```xml
<!-- 另一种联合查询 (一对一)的实现，但是这种方式有“N+1”的问题 -->
<resultMap id="BlogWithAuthorQueryMap" type="com.domain.associate.BlogAndAuthor">
<id column="bid" property="bid" jdbcType="INTEGER"/>
<result column="name" property="name" jdbcType="VARCHAR"/>
<association property="author" javaType="com.gupaoedu.domain.Author"
column="author_id" select="selectAuthor"/> <!-- selectAuthor 定义在下面-->
</resultMap>
<!-- 嵌套查询 -->
<select id="selectAuthor" parameterType="int" resultType="com.domain.Author">
select author_id authorId, author_name authorName
from author where author_id = #{authorId}
</select>
```
其中第二种查询方式:嵌套查询,由于是分两次查询,当我们查询了员工信息之后,会再发送一条SQL到数据库查询部门信息

我们只执行了一次查询员工信息的SQL(所谓的1),如果返回了N条记录,就会再发送N条到数据库查询部门信息(所谓的N),这个就是我们所说的N+1的问题,这样就会白白的浪费我们的应用和数据库的性能.

如果我们用了嵌套查询的方式,怎么解决这个问题呢?能不能等到使用部门信息的时候再去查询呢?这个就是我们所说的延迟加载,或者叫懒加载.

在Mybatis里面可以通过开启延迟加载的开关来解决这个问题.

在settings标签里面可以配置
```xml

<!--延迟加载的全局开关。当开启时，所有关联对象都会延迟加载。默认 false -->
<setting name="lazyLoadingEnabled" value="true"/>
<!--当开启时，任何方法的调用都会加载该对象的所有属性。默认 false，可通过 select 标签的
fetchType 来覆盖-->
<setting name="aggressiveLazyLoading" value="false"/>
<!-- Mybatis 创建具有延迟加载能力的对象所用到的代理工具，默认 JAVASSIST -->
<setting name="proxyFactory" value="CGLIB" />
```
lazyLoadingEnabled 决定了是否延迟加载.
aggressiveLazyLoading 决定了是不是对象的所有方法都会触发查询

先来测试一下(也可以修改成查询列表)
1. 没有开启延迟加载的开关,会连续发送两次查询
2. 开启了延迟加载的开关,调用 blog.getAuthor()以及默认的(equals,clone,hshCode,toString)时才会发送第二次查询,其他方法并不会触发查询.比图blog.getName().
3. 如果开启了 aggressiveLazyLoading=true ,其他方法也会触发查询,比如 blog.getName()

问题:为什么可以做到延迟加载呢?blog.getAuthor(),只是一个获取属性的方法,里面并没有链接数据库的代码,为什么会触发对数据库的查询
   将  blog对象打印出来
   > System.out.println(blog.getClass());

   ```
   class com.domain.associate.BlogAndAuthor_$$_jvst70_0

   ```
   这个类的名字的后面有个jvst,是JAVASSIST的缩写,原来到这里带延迟加载的功能的对象blog已经变成了一个代理对象.
