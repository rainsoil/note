# 9. Spring 事务详谈

-----


-----
### 从Spring事务配置说起
先看看Spring事务的基础配置
```xml
<aop:aspectj-autoproxy proxy-target-class="true"/>
<bean id="transactionManager"
class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
<property name="dataSource" ref="dataSource"/>
</bean>
<tx:annotation-driven transaction-manager="transactionManager"/>
<!-- 配置事务传播特性 -->
<tx:advice id="transactionAdvice" transaction-manager="transactionManager">
<tx:attributes>
<tx:method name="add*" propagation="REQUIRED"
rollback-for="Exception,RuntimeException,SQLException"/>
<tx:method name="remove*" propagation="REQUIRED"
rollback-for="Exception,RuntimeException,SQLException"/>
<tx:method name="modify*" propagation="REQUIRED"
rollback-for="Exception,RuntimeException,SQLException"/>
<tx:method name="login" propagation="NOT_SUPPORTED"/>
<tx:method name="query*" read-only="true"/>
</tx:attributes>
</tx:advice>
<aop:config>
<aop:pointcut expression="execution(public * com.gupaoedu.vip..*.service..*Service.*(..))"
id="transactionPointcut"/>
<aop:advisor pointcut-ref="transactionPointcut" advice-ref="transactionAdvice"/>
</aop:config>

```

### 数据库事务原理详解
#### 1. 事务基本概念
事务(Transaction)是访问并可能更新数据库中各种数据项的一个程序执行单元(unit)。
特点：事务是恢复和并发控制的基本单位。事务应该具有 4 个属性：原子性、一致性、
隔离性、持久性。这四个属性通常称为 ACID 特性。

原子性（Automicity）。一个事务是一个不可分割的工作单位，事务中包括的诸操作要
么都做，要么都不做。

一致性（Consistency）。事务必须是使数据库从一个一致性状态变到另一个一致性状态。
一致性与原子性是密切相关的。

隔离性（Isolation）。一个事务的执行不能被其他事务干扰。即一个事务内部的操作及
使用的数据对并发的其他事务是隔离的，并发执行的各个事务之间不能互相干扰。

持久性（Durability）。持久性也称永久性（Permanence），指一个事务一旦提交，它
对数据库中数据的改变就应该是永久性的。接下来的其他操作或故障不应该对其有任何
影响。

#### 2、事务的基本原理
Spring 事务的本质其实就是数据库对事务的支持，没有数据库的事务支持，Spring 是无
法提供事务功能的。对于纯 JDBC 操作数据库，想要用到事务，可以按照以下步骤进行：

1. 获取连接 Connection con = DriverManager.getConnection()
2. 开启事务 con.setAutoCommit(true/false);
3. 执行 CRUD
4. 提交事务/回滚事务 con.commit() / con.rollback();
5. 关闭连接 conn.close();

使用 Spring 的事务管理功能后，我们可以不再写步骤 2 和 4 的代码，而是由 Spirng 自
动完成。 那么 Spring 是如何在我们书写的 CRUD 之前和之后开启事务和关闭事务的
呢？解决这个问题，也就可以从整体上理解 Spring 的事务管理实现原理了。下面简单地
介绍下，注解方式为例子

配置文件开启注解驱动，在相关的类和方法上通过注解@Transactional 标识。
Spring 在启动的时候会去解析生成相关的 bean，这时候会查看拥有相关注解的类和方
法，并且为这些类和方法生成代理，并根据@Transaction 的相关参数进行相关配置注入，
这样就在代理中为我们把相关的事务处理掉了（开启正常提交事务，异常回滚事务）。
真正的数据库层的事务提交和回滚是通过 binlog 或者 redo log 实现的。
#### 3、Spring 事务的传播属性

所谓 spring 事务的传播属性，就是定义在存在多个事务同时存在的时候，spring 应该如
何处理这些事务的行为。这些属性在 TransactionDefinition 中定义，具体常量的解释见
下表：
| 常量名称                  | 常量解释                                                     |      |
| ------------------------- | ------------------------------------------------------------ | ---- |
| PROPAGATION_REQUIRED      | 支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择，也是 Spring默认的事务的传播。 |      |
| PROPAGATION_REQUIRES_NEW  | 新建事务,如果当前存在事务,把当前事务挂起。新建的事务将和被挂起的事务没有任何关系两个独立的事务，外层事务失败回滚之后，不能回滚内层事务执行的结果，内层事务失败抛出异常，外层事务捕获，也可以不处理回滚操作 |      |
| PROPAGATION_SUPPORTS      | 支持当前事务，如果当前没有事务，就以非事务方式执行。         |      |
| PROPAGATION_MANDATORY     | 支持当前事务，如果当前没有事务，就抛出异常。                 |      |
| PROPAGATION_NOT_SUPPORTED | 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。   |      |
| PROPAGATION_NEVER         | 以非事务方式执行，如果当前存在事务，则抛出异常。             |      |
| PROPAGATION_NESTED        | 如果一个活动的事务存在，则运行在一个嵌套的事务中。如果没有活动事务，则按REQUIRED 属性执行。它使用了一个单独的事务，这个事务拥有多个可以回滚的保存点。内部事务的回滚不会对外部事务造成影响。它只对DataSourceTransactionManager 事务管理器起效。 |      |

#### 4、数据库隔离级别
| 隔离级别         | 隔离级别的值 | 导致的问题                                                   |
| ---------------- | ------------ | ------------------------------------------------------------ |
| Read-Uncommitted | 0            | 导致脏读                                                     |
| Read-Committed   | 1            | 避免脏读，允许不可重复读和幻读                               |
| Repeatable-Read  | 2            | 避免脏读，不可重复读，允许幻读                               |
| Serializable     | 3            | 串行化读，事务只能一个一个执行，避免了脏读、不可重复读、幻读。执行效率慢，使用时慎重 |

脏读：一事务对数据进行了增删改，但未提交，另一事务可以读取到未提交的数据。如
果第一个事务这时候回滚了，那么第二个事务就读到了脏数据。

不可重复读：一个事务中发生了两次读操作，第一次读操作和第二次操作之间，另外一
个事务对数据进行了修改，这时候两次读取的数据是不一致的。

幻读：第一个事务对一定范围的数据进行批量修改，第二个事务在这个范围增加一条数
据，这时候第一个事务就会丢失对新增数据的修改。

总结：

隔离级别越高，越能保证数据的完整性和一致性，但是对并发性能的影响也越大。
大多数的数据库默认隔离级别为 Read Commited，比如 SqlServer、Oracle
少数数据库默认隔离级别为：Repeatable Read 比如： MySQL InnoDB


#### 5、Spring 中的隔离级别
| 常量                       | 解释                                                         |
| -------------------------- | ------------------------------------------------------------ |
| ISOLATION_DEFAULT          | 这是个 PlatfromTransactionManager 默认的隔离级别，使用数据库默认的事务隔离级别。另外四个与 JDBC 的隔离级别相对应。 |
| ISOLATION_READ_UNCOMMITTED | 这是事务最低的隔离级别，它允许另外一个事务可以看到这个事务未提交的数据。这种隔离级别会产生脏读，不可重复读和幻像读。 |
| ISOLATION_READ_COMMITTED   | 保证一个事务修改的数据提交后才能被另外一个事务读取。另外一个事务不能读取该事务未提交的数据。 |
| ISOLATION_REPEATABLE_READ  | 这种事务隔离级别可以防止脏读，不可重复读。但是可能出现幻像读。 |
| ISOLATION_SERIALIZABLE     | 这是花费最高代价但是最可靠的事务隔离级别。事务被处理为顺序执行。 |

#### 6、事务的嵌套
通过上面的理论知识的铺垫，我们大致知道了数据库事务和 Spring 事务的一些属性和特
点，接下来我们通过分析一些嵌套事务的场景，来深入理解 Spring 事务传播的机制。

假设外层事务 Service A 的 Method A() 调用 内层 Service B 的 Method B()
**PROPAGATION_REQUIRED(Spring 默认)**

如果 ServiceB.MethodB() 的事务级别定义为 PROPAGATION_REQUIRED，那么执行
ServiceA.MethodA() 的时候 Spring 已经起了事务，这时调用 ServiceB.MethodB()，
ServiceB.MethodB() 看到自己已经运行在 ServiceA.MethodA() 的事务内部，就不再
起新的事务。

假如 ServiceB.MethodB() 运行的时候发现自己没有在事务中，他就会为自己分配一个
事务。

这样，在 ServiceA.MethodA() 或者在 ServiceB.MethodB() 内的任何地方出现异常，
事务都会被回滚。

**PROPAGATION_REQUIRES_NEW**

比如我们设计 ServiceA.MethodA() 的事务级别为 PROPAGATION_REQUIRED，
ServiceB.MethodB() 的事务级别为 PROPAGATION_REQUIRES_NEW。

那么当执行到 ServiceB.MethodB() 的时候，ServiceA.MethodA() 所在的事务就会挂
起，ServiceB.MethodB() 会起一个新的事务，等待 ServiceB.MethodB() 的事务完成
以后，它才继续执行。

他 与 PROPAGATION_REQUIRED 的 事 务 区 别 在 于 事 务 的 回 滚 程 度 了 。 因 为
ServiceB.MethodB() 是 新 起 一 个 事 务 ， 那 么 就 是 存 在 两 个 不 同 的 事 务 。 如 果
ServiceB.MethodB() 已 经 提 交 ， 那 么 ServiceA.MethodA() 失 败 回 滚 ，
ServiceB.MethodB() 是不会回滚的。如果 ServiceB.MethodB() 失败回滚，如果他抛

出的异常被 ServiceA.MethodA() 捕获，ServiceA.MethodA() 事务仍然可能提交(主要
看 B 抛出的异常是不是 A 会回滚的异常)。

**PROPAGATION_SUPPORTS**

&nbsp;&nbsp;假设 ServiceB.MethodB() 的事务级别为 PROPAGATION_SUPPORTS，那么当执行到
ServiceB.MethodB()时，如果发现 ServiceA.MethodA()已经开启了一个事务，则加入
当前的事务，如果发现 ServiceA.MethodA()没有开启事务，则自己也不开启事务。这种
时候，内部方法的事务性完全依赖于最外层的事务。

**PROPAGATION_NESTED**

&nbsp;&nbsp;现 在 的 情 况 就 变 得 比 较 复 杂 了 , ServiceB.MethodB() 的 事 务 属 性 被 配 置 为
PROPAGATION_NESTED, 此时两者之间又将如何协作呢? ServiceB.MethodB() 如
果 rollback, 那么内部事务(即 ServiceB.MethodB()) 将回滚到它执行前的 SavePoint
而外部事务(即 ServiceA.MethodA()) 可以有以下两种处理方式:
捕获异常，执行异常分支逻辑
```java
void MethodA() {
try {
ServiceB.MethodB();
} catch (SomeException) {
// 执行其他业务, 如 ServiceC.MethodC();
}
}
```
这 种 方 式 也 是 嵌 套 事 务 最 有 价 值 的 地 方 , 它 起 到 了 分 支 执 行 的 效 果 , 如 果
ServiceB.MethodB()失败, 那么执行 ServiceC.MethodC(), 而 ServiceB.MethodB()
已经回滚到它执行之前的 SavePoint, 所以不会产生脏数据(相当于此方法从未执行过), 这 种 特 性 可 以 用 在 某 些 特 殊 的 业 务 中 , 而 PROPAGATION_REQUIRED 和
PROPAGATION_REQUIRES_NEW 都没有办法做到这一点。

外部事务回滚/提交 代码不做任何修改, 那么如果内部事务(ServiceB.MethodB())
rollback, 那么首先 ServiceB.MethodB() 回滚到它执行之前的 SavePoint(在任何情况
下都会如此), 外部事务(即 ServiceA.MethodA()) 将根据具体的配置决定自己是
commit 还是 rollback。

另外三种事务传播属性基本用不到，在此不做分析
#### 7、Spring 事务 API 架构图
![image](http://files.luyanan.com/3cbd6d28-bf3e-4163-82cf-4c90b14bec5f.jpg)

使用 Spring 进行基本的 JDBC 访问数据库有多种选择。Spring 至少提供了三种不同的工
作模式：JdbcTemplate, 一个在 Spring2.5 中新提供的 SimpleJdbc 类能够更好的处理
数据库元数据; 还有一种称之为 RDBMS Object 的风格的面向对象封装方式, 有点类似
于 JDO 的查询设计。 我们在这里简要列举你采取某一种工作方式的主要理由. 不过请注
意, 即使你选择了其中的一种工作模式, 你依然可以在你的代码中混用其他任何一种模
式以获取其带来的好处和优势。 所有的工作模式都必须要求 JDBC 2.0 以上的数据库驱
动的支持, 其中一些高级的功能可能需要 JDBC 3.0 以上的数据库驱动支持。
JdbcTemplate - 这是经典的也是最常用的 Spring 对于 JDBC 访问的方案。这也是最低
级别的封装, 其他的工作模式事实上在底层使用了 JdbcTemplate 作为其底层的实现基
础。JdbcTemplate 在 JDK 1.4 以上的环境上工作得很好。
NamedParameterJdbcTemplate - 对 JdbcTemplate 做了封装，提供了更加便捷的基
于命名参数的使用方式而不是传统的 JDBC 所使用的“?”作为参数的占位符。这种方式
在你需要为某个 SQL 指定许多个参数时，显得更加直观而易用。该特性必须工作在 JDK
1.4 以上。

SimpleJdbcTemplate - 这 个 类 结 合 了 JdbcTemplate 和
NamedParameterJdbcTemplate 的最常用的功能，同时它也利用了一些 Java 5 的特性
所带来的优势，例如泛型、varargs 和 autoboxing 等，从而提供了更加简便的 API 访问
方式。需要工作在 Java 5 以上的环境中。
SimpleJdbcInsert 和 SimpleJdbcCall - 这两个类可以充分利用数据库元数据的特性
来简化配置。通过使用这两个类进行编程，你可以仅仅提供数据库表名或者存储过程的
名称以及一个 Map 作为参数。其中 Map 的 key 需要与数据库表中的字段保持一致。这
两个类通常和 SimpleJdbcTemplate 配合使用。这两个类需要工作在 JDK 5 以上，同时
数据库需要提供足够的元数据信息。

RDBMS 对象包括 MappingSqlQuery, SqlUpdate and StoredProcedure - 这种方式允许你在初始化你的数据访问层时创建可重用并且线程安全的对象。该对象在你定义了
你的查询语句，声明查询参数并编译相应的 Query 之后被模型化。一旦模型化完成，任
何执行函数就可以传入不同的参数对之进行多次调用。这种方式需要工作在 JDK 1.4 以
上。

异常处理
异常结构如下：
![image](http://files.luyanan.com/f2b017de-07cc-4eb3-abbc-47b3eb792493.jpg)
SQLExceptionTranslator 是 一 个 接 口 ， 如 果 你 需 要 在 SQLException 和
org.springframework.dao.DataAccessException 之间作转换，那么必须实现该接口。
转换器类的实现可以采用一般通用的做法(比如使用 JDBC 的 SQLState code)，如果为了
使转换更准确，也可以进行定制（比如使用 Oracle 的 error code）。
SQLErrorCodeSQLExceptionTranslator 是 SQLExceptionTranslator 的默认实现。 该
实现使用指定数据库厂商的 error code，比采用 SQLState 更精确。转换过程基于一个
JavaBean （ 类 型 为 SQLErrorCodes ） 中 的 error code 。 这 个 JavaBean 由
SQLErrorCodesFactory 工厂类创建，其中的内容来自于 “sql-error-codes.xml”配置
文 件 。 该 文 件 中 的 数 据 库 厂 商 代 码 基 于 Database MetaData 信 息 中 的
DatabaseProductName，从而配合当前数据库的使用。

SQLErrorCodeSQLExceptionTranslator 使用以下的匹配规则：

首 先 检 查 是 否 存 在 完 成 定 制 转 换 的 子 类 实 现 。 通 常
SQLErrorCodeSQLExceptionTranslator 这个类可以作为一个具体类使用，不需要进行
定制，那么这个规则将不适用。

接着将 SQLException 的 error code 与错误代码集中的 error code 进行匹配。 默认情
况下错误代码集将从 SQLErrorCodesFactory 取得。 错误代码集来自 classpath 下的
sql-error-codes.xml 文件，它们将与数据库 metadata 信息中的 database name 进行
映射。

使用 fallback 翻译器。SQLStateSQLExceptionTranslator 类是缺省的 fallback 翻译器。
config 模块

NamespaceHandler 接口，DefaultBeanDefinitionDocumentReader 使用该接口来处
理在 spring xml 配置文件中自定义的命名空间。
![image](http://files.luyanan.com/0f8ef8dc-04d5-4438-83f0-c618882078ba.jpg)

在 jdbc 模块，我们使用 JdbcNamespaceHandler 来处理 jdbc 配置的命名空间，其代
码如下：

```java
public class JdbcNamespaceHandler extends NamespaceHandlerSupport {
@Override
public void init() {
registerBeanDefinitionParser("embedded-database", new EmbeddedDatabaseBeanDefinitionParser());
registerBeanDefinitionParser("initialize-database", new InitializeDatabaseBeanDefinitionParser());
}
}
```

其 中 ， EmbeddedDatabaseBeanDefinitionParser 继 承 了
AbstractBeanDefinitionParser ， 解 析 <embedded-database> 元 素 ， 并 使 用
EmbeddedDatabaseFactoryBean 创建一个 BeanDefinition。顺便介绍一下用到的软
件包 org.w3c.dom。

软件包 org.w3c.dom:为文档对象模型 (DOM) 提供接口，该模型是 Java API for XML
Processing 的组件 API。该 Document Object Model Level 2 Core API 允许程序动
态访问和更新文档的内容和结构。

Attr：Attr 接口表示 Element 对象中的属性。

CDATASection： CDATA 节用于转义文本块，该文本块包含的字符如果不转义则会被
视为标记。

CharacterData： CharacterData 接口使用属性集合和用于访问 DOM 中字符数据的
方法扩展节点。

Comment： 此接口继承自 CharacterData 表示注释的内容，即起始 '<!--' 和结束
'-->' 之间的所有字符。

Document： Document 接口表示整个 HTML 或 XML 文档。

DocumentFragment： DocumentFragment 是“轻量级”或“最小”Document 对
象。

DocumentType： 每个 Document 都有 doctype 属性，该属性的值可以为 null，也
可以为 DocumentType 对象。

DOMConfiguration： 该 DOMConfiguration 接口表示文档的配置，并维护一个可识
别的参数表。

DOMError： DOMError 是一个描述错误的接口。

DOMErrorHandler： DOMErrorHandler 是在报告处理 XML 数据时发生的错误或在
进行某些其他处理（如验证文档）时 DOM 实现可以调用的回调接口。

DOMImplementation： DOMImplementation 接口为执行独立于文档对象模型的任
何特定实例的操作提供了许多方法。

DOMImplementationList： DOMImplementationList 接口提供对 DOM 实现的有
序集合的抽象，没有定义或约束如何实现此集合。

DOMImplementationSource： 此接口允许 DOM 实现程序根据请求的功能和版本提
供一个或多个实现，如下所述。

DOMLocator： DOMLocator 是一个描述位置（如发生错误的位置）的接口。

DOMStringList： DOMStringList 接口提供对 DOMString 值的有序集合的抽象，没
有定义或约束此集合是如何实现的。

Element： Element 接口表示 HTML 或 XML 文档中的一个元素。

Entity： 此接口表示在 XML 文档中解析和未解析的已知实体。

EntityReference： EntityReference 节点可以用来在树中表示实体引用。

NamedNodeMap： 实现 NamedNodeMap 接口的对象用于表示可以通过名称访问
的节点的集合。

NameList NameList 接口提供对并行的名称和名称空间值对（可以为 null 值）的有序
集合的抽象，无需定义或约束如何实现此集合。

Node： 该 Node 接口是整个文档对象模型的主要数据类型。

NodeList： NodeList 接口提供对节点的有序集合的抽象，没有定义或约束如何实现此
集合。

Notation： 此接口表示在 DTD 中声明的表示法。

ProcessingInstruction： ProcessingInstruction 接口表示“处理指令”，该指令作为
一种在文档的文本中保持特定于处理器的信息的方法在 XML 中使用。

Text： 该 Text 接口继承自 CharacterData，并且表示 Element 或 Attr 的文本内容
（在 XML 中称为 字符数据）。

TypeInfo： TypeInfo 接口表示从 Element 或 Attr 节点引用的类型，用与文档相关
的模式指定。

UserDataHandler： 当使用 Node.setUserData() 将一个对象与节点上的键相关联时，
当克隆、导入或重命名该对象关联的节点时应用程序可以提供调用的处理程序。

core 模块

1.JdbcTeamplate 对象，其结构如下：
![image](http://files.luyanan.com/6838f36d-f60e-4343-8151-94f0221b0d2c.jpg)

2.RowMapper
![image](http://files.luyanan.com/1730125f-dea5-4997-b3f5-c526fcbf289c.jpg)
3.元数据 metaData 模块
本节中 Spring 应用到工厂模式，结合代码可以更具体了解。
![image](http://files.luyanan.com/3c625dce-0ab3-431c-bade-404901ddbff9.jpg)
CallMetaDataProviderFactory 创建 CallMetaDataProvider 的工厂类，其代码如下：
```java
public static final List<String> supportedDatabaseProductsForProcedures = Arrays.asList(
"Apache Derby",
"DB2",
"MySQL",
"Microsoft SQL Server",
"Oracle",
"PostgreSQL",
"Sybase"
);

/** List of supported database products for function calls */
public static final List<String> supportedDatabaseProductsForFunctions = Arrays.asList(
"MySQL",
"Microsoft SQL Server",
"Oracle",
"PostgreSQL"
);
static public CallMetaDataProvider createMetaDataProvider(DataSource dataSource, final CallMetaDataContext
context) {
try {
CallMetaDataProvider result = (CallMetaDataProvider) JdbcUtils.extractDatabaseMetaData(dataSource,
databaseMetaData -> {
String databaseProductName = JdbcUtils.commonDatabaseName(databaseMetaData.getDatabaseProductName());
boolean accessProcedureColumnMetaData = context.isAccessCallParameterMetaData();
if (context.isFunction()) {
if (!supportedDatabaseProductsForFunctions.contains(databaseProductName)) {
if (logger.isWarnEnabled()) {
logger.warn(databaseProductName + " is not one of the databases fully supported for function calls
" +
"-- supported are: " + supportedDatabaseProductsForFunctions);
}
if (accessProcedureColumnMetaData) {
logger.warn("Metadata processing disabled - you must specify all parameters explicitly");
accessProcedureColumnMetaData = false;
}
}
}
else {
if (!supportedDatabaseProductsForProcedures.contains(databaseProductName)) {
if (logger.isWarnEnabled()) {
logger.warn(databaseProductName + " is not one of the databases fully supported for procedure
calls " +
"-- supported are: " + supportedDatabaseProductsForProcedures);
}
if (accessProcedureColumnMetaData) {
logger.warn("Metadata processing disabled - you must specify all parameters explicitly");
accessProcedureColumnMetaData = false;
}
}
}
CallMetaDataProvider provider;
if ("Oracle".equals(databaseProductName)) {

provider = new OracleCallMetaDataProvider(databaseMetaData);
}
else if ("DB2".equals(databaseProductName)) {
provider = new Db2CallMetaDataProvider((databaseMetaData));
}
else if ("Apache Derby".equals(databaseProductName)) {
provider = new DerbyCallMetaDataProvider((databaseMetaData));
}
else if ("PostgreSQL".equals(databaseProductName)) {
provider = new PostgresCallMetaDataProvider((databaseMetaData));
}
else if ("Sybase".equals(databaseProductName)) {
provider = new SybaseCallMetaDataProvider((databaseMetaData));
}
else if ("Microsoft SQL Server".equals(databaseProductName)) {
provider = new SqlServerCallMetaDataProvider((databaseMetaData));
}
else if ("HDB".equals(databaseProductName)) {
provider = new HanaCallMetaDataProvider((databaseMetaData));
}
else {
provider = new GenericCallMetaDataProvider(databaseMetaData);
}
if (logger.isDebugEnabled()) {
logger.debug("Using " + provider.getClass().getName());
}
provider.initializeWithMetaData(databaseMetaData);
if (accessProcedureColumnMetaData) {
provider.initializeWithProcedureColumnMetaData(databaseMetaData,
context.getCatalogName(), context.getSchemaName(), context.getProcedureName());
}
return provider;
});
return result;
}
catch (MetaDataAccessException ex) {
throw new DataAccessResourceFailureException("Error retrieving database metadata", ex);
}
}
```
TableMetaDataProviderFactory 创建 TableMetaDataProvider 工厂类，其创建过程如
下：
```java
static public CallMetaDataProvider createMetaDataProvider(DataSource dataSource, final CallMetaDataContext
context) {
try {
CallMetaDataProvider result = (CallMetaDataProvider) JdbcUtils.extractDatabaseMetaData(dataSource,
databaseMetaData -> {
String databaseProductName = JdbcUtils.commonDatabaseName(databaseMetaData.getDatabaseProductName());
boolean accessProcedureColumnMetaData = context.isAccessCallParameterMetaData();
if (context.isFunction()) {
if (!supportedDatabaseProductsForFunctions.contains(databaseProductName)) {
if (logger.isWarnEnabled()) {
logger.warn(databaseProductName + " is not one of the databases fully supported for function calls
" +
"-- supported are: " + supportedDatabaseProductsForFunctions);
}
if (accessProcedureColumnMetaData) {
logger.warn("Metadata processing disabled - you must specify all parameters explicitly");
accessProcedureColumnMetaData = false;
}
}
}
else {
if (!supportedDatabaseProductsForProcedures.contains(databaseProductName)) {
if (logger.isWarnEnabled()) {
logger.warn(databaseProductName + " is not one of the databases fully supported for procedure
calls " +
"-- supported are: " + supportedDatabaseProductsForProcedures);
}
if (accessProcedureColumnMetaData) {
logger.warn("Metadata processing disabled - you must specify all parameters explicitly");
accessProcedureColumnMetaData = false;
}
}
}
CallMetaDataProvider provider;
if ("Oracle".equals(databaseProductName)) {
provider = new OracleCallMetaDataProvider(databaseMetaData);
}
else if ("DB2".equals(databaseProductName)) {
provider = new Db2CallMetaDataProvider((databaseMetaData));
}
else if ("Apache Derby".equals(databaseProductName)) {
provider = new DerbyCallMetaDataProvider((databaseMetaData));
}
else if ("PostgreSQL".equals(databaseProductName)) {
provider = new PostgresCallMetaDataProvider((databaseMetaData));

}
else if ("Sybase".equals(databaseProductName)) {
provider = new SybaseCallMetaDataProvider((databaseMetaData));
}
else if ("Microsoft SQL Server".equals(databaseProductName)) {
provider = new SqlServerCallMetaDataProvider((databaseMetaData));
}
else if ("HDB".equals(databaseProductName)) {
provider = new HanaCallMetaDataProvider((databaseMetaData));
}
else {
provider = new GenericCallMetaDataProvider(databaseMetaData);
}
if (logger.isDebugEnabled()) {
logger.debug("Using " + provider.getClass().getName());
}
provider.initializeWithMetaData(databaseMetaData);
if (accessProcedureColumnMetaData) {
provider.initializeWithProcedureColumnMetaData(databaseMetaData,
context.getCatalogName(), context.getSchemaName(), context.getProcedureName());
}
return provider;
});
return result;
}
catch (MetaDataAccessException ex) {
throw new DataAccessResourceFailureException("Error retrieving database metadata", ex);
}
}
```
使用 SqlParameterSource 提供参数值
使用 Map 来指定参数值有时候工作得非常好，但是这并不是最简单的使用方式。Spring
提供了一些其他的 SqlParameterSource 实现类来指定参数值。 我们首先可以看看
BeanPropertySqlParameterSource 类，这是一个非常简便的指定参数的实现类，只要
你有一个符合 JavaBean 规范的类就行了。它将使用其中的 getter 方法来获取参数值。

SqlParameter 封 装 了 定 义 sql 参 数 的 对 象 。 CallableStateMentCallback ，
PrePareStateMentCallback，StateMentCallback，ConnectionCallback 回调类分别
对应 JdbcTemplate 中的不同处理方法。

![image](http://files.luyanan.com/3e9f8220-849a-43af-a56c-cbc362a2b659.jpg)
simple 实现
![image](http://files.luyanan.com/631a58ae-1aa2-4bc4-97c8-bad76d23b904.jpg)
Spring 通过 DataSource 获取数据库的连接。Datasource 是 jdbc 规范的一部分，
它通过 ConnectionFactory 获取。一个容器和框架可以在应用代码层中隐藏连接池和事
务管理。

当使用 spring 的 jdbc 层，你可以通过 JNDI 来获取 DataSource，也可以通过你自
己配置的第三方连接池实现来获取。流行的第三方实现由 apache Jakarta Commons
dbcp 和 c3p0。
![image](http://files.luyanan.com/ab627b50-fe89-4674-8c64-72914c50d8f8.jpg)

TransactionAwareDataSourceProxy 作为目标 DataSource 的一个代理， 在对目标
DataSource 包装的同时，还增加了 Spring 的事务管理能力， 在这一点上，这个类的功
能非常像 J2EE 服务器所提供的事务化的 JNDI DataSource。

该类几乎很少被用到，除非现有代码在被调用的时候需要一个标准的 JDBC DataSource
接口实现作为参数。 这种情况下，这个类可以使现有代码参与 Spring 的事务管理。通
常最好的做法是使用更高层的抽象 来对数据源进行管理，比如 JdbcTemplate 和
DataSourceUtils 等等。

注意：DriverManagerDataSource 仅限于测试使用，因为它没有提供池的功能，这会导
致在多个请求获取连接时性能很差。

object 模块

![image](http://files.luyanan.com/00e60c71-a7c4-46bd-ab47-145254fbe3e8.jpg)

#### JdbcTemplate

JdbcTemplate 是 core 包的核心类。它替我们完成了资源的创建以及释放工作，从而简
化了我们对 JDBC 的使用。 它还可以帮助我们避免一些常见的错误，比如忘记关闭数据
库连接。 JdbcTemplate 将完成 JDBC 核心处理流程，比如 SQL 语句的创建、执行，而
把 SQL 语句的生成以及查询结果的提取工作留给我们的应用代码。 它可以完成 SQL 查
询、更新以及调用存储过程，可以对 ResultSet 进行遍历并加以提取。它还可以捕获 JDBC
异常并将其转换成 org.springframework.dao 包中定义的，通用的，信息更丰富的异常。
使用 JdbcTemplate 进行编码只需要根据明确定义的一组契约来实现回调接口。
PreparedStatementCreator 回调接口通过给定的 Connection 创建一个
PreparedStatement，包含 SQL 和任何相关的参数。 CallableStatementCreateor 实

现同样的处理，只不过它创建的是 CallableStatement。 RowCallbackHandler 接口则
从数据集的每一行中提取值。

我们可以在 DAO 实现类中通过传递一个 DataSource 引用来完成 JdbcTemplate 的实例
化，也可以在 Spring 的 IOC 容器中配置一个 JdbcTemplate 的 bean 并赋予 DAO 实现
类作为一个实例。 需要注意的是 DataSource 在 Spring 的 IOC 容器中总是配制成一个
bean，第一种情况下，DataSource bean 将传递给 service，第二种情况下 DataSource
bean 传递给 JdbcTemplate bean。

NamedParameterJdbcTemplate

NamedParameterJdbcTemplate 类为 JDBC 操作增加了命名参数的特性支持，而不是
传 统 的 使 用 （ '?' ） 作 为 参 数 的 占 位 符 。 NamedParameterJdbcTemplate 类 对
JdbcTemplate 类进行了封装， 在底层，JdbcTemplate 完成了多数的工作。