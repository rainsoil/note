# 3. MyBatis插件原理



--------------------------

Mybatis 通过提供插件机制,让我们可以根据自己的需要去增强Mybatis 的功能

需要注意的是,如果没有完全理解Mybatis的运行原理和插件的工作方式,最好不要使用插件,因为他会改变底层的工作逻辑,给系统带来很大的影响.

#### 1. 猜想
Mybatis 的插件可以在不修改原来代码的情况下，通过拦截方式改变四大核心对象的行为,比如处理参数,处理SQL,处理结果.
#####  第一个问题:
不修改对象的代码,怎么对对象的行为进行修改,比如说在原来的方法前面做一点事情,在原来的方法后面做一点事情.

答案: 大家很容易想到使用代理模式,这个也确实是Mybatis 插件的原理。

##### 第二个问题:
我们可以定义很多的插件,那么这种所有的插件都会行成一个链路,比如我们提交一个休假申请,先是项目经理审批,然后是部门经理审批,再是HR审批,再到总经理审批,怎么实现层层的拦截.,

答案: 插件是层层拦截的,我们又需要用到另一种设计模式--责任链模式

如果是用代理模式,我们就要解决几个问题？
 ######  有哪些对象允许被代理？有哪些方法可以被拦截?
 我们应该了解Mybatis 允许哪些对象的哪些方法允许被拦截,并不是每一个运行的节点的都是可以被修改的,只有清楚了这些对象的方法的作用,当我们自己编写插件拦截的时候才知道从哪里去拦截.

在Mybatis的官网上有答案,我们来看一下
http://www.mybatis.org/mybatis-3/zh/configuration.html#plugins

![image](http://files.luyanan.com/974b81fa-9dd7-47a4-ba9c-afa995d91b8a.png)


Executor 会拦截CachingExecutor或者BaseExecutor

因为创建Executor时先创建Executor,再拦截

###### 2. 怎么创建代理?
如果我们用JDK代理,要有一个实现InvocationHandler的代理类,用来包装被代理对象,这个类是自己创建还是谁创建?

###### 3. 什么时候创建代理对象?是在Mybatis启动的时候创建,还是调用的时候创建?

##### 4. 被代理后,调用的是什么方法?怎么调用到原代理对象的方法(比如Executor的query 方法)

要解决后面三个问题,我们先看一下别人的插件是如何工作的?

#### 2. 插件编写与注册

我们以PageHelper 为例:
##### 1. 编写自己的插件类
1. 实现Interceptor 接口

 这个是所有插件必须实现的接口

2. 添加@Intercepts({@Signarure()}) ,指定拦截的对象和方法,方法参数,方法名称+参数类型,构成了方法的签名,决定了能拦截到哪个方法.

问题:拦截参数跟参数的顺序有关系吗?

3. 实现接口的三个方法

```java
// 用于覆盖被拦截对象的原有方法（在调用代理对象 Plugin 的 invoke()方法时被调用）
Object intercept(Invocation invocation) throws Throwable;
// target 是被拦截对象，这个方法的作用是给被拦截对象生成一个代理对象，并返回它
Object plugin(Object target);
// 设置参数
void setProperties(Properties properties);
```
##### 2. 插件注册,在mybatis-config.xml中注册插件
```xml
<plugins>
<plugin interceptor="com.github.pagehelper.PageInterceptor">
<property name="offsetAsPageNum" value="true"/> ……后面全部省略……
</plugin>
</plugins>
```
##### 3. 插件登记
Mybatis 启动时扫描<plugins> 标签,注册到Configuration 对象的InterceptorChin 中,Property 里面的参数,会调用setProperty() 处理

以上就是编写和使用自定义插件的全部步骤

#### 3.代理和拦截是怎么实现的?

- 问题1 : 四大对象是什么时候代理的,也就是:代理对象是什么时候创建的?
- 问题2: 多个插件的情况下,代理能不能被代理?代理顺序和调用顺序的关系?
- 问题3: 谁来创建代理对象?
- 问题4: 被代理后,调用的是什么方法?怎么调用到原被代理对象的方法?

**问题1:**

Executor 是openSession()的时候创建的;StatementHandler 是SimpleExecutor.doQuery()的时候创建的;里面包含了处理参数的ParameterHandler和处理结果集ResultSetHandler的创建,创建之后立即调用InterceptorChin.pluginAll(),返回层层代理之后的对象.

**问题2:**

可以被代理,debug看一下.
![image](http://files.luyanan.com/95727d74-69e6-4812-827e-a551dc96577c.jpg)

**问题3:**
Plugin 类,在我们重写的 plugin() 方法里面可以直接调用 return Plugin.wrap(target,this) 返回调用对象.

**问题4:**
因为代理类是Plugin,所以最后调用的是 Plugin的 invoke() 方法,它先调用了定义的拦截器intercept() 方法.可以通过 invocation.proceed() 调用到被代理对象被拦截的方法.

总结流程如下:
![image](http://files.luyanan.com/d933bc11-8774-40d2-ba30-99f1775593f2.jpg)


总结:
| 对象           | 作用                                                     |
| -------------- | -------------------------------------------------------- |
| Interceptor    | 自定义插件需要实现接口,实现三个方法                      |
| InterceptChain | 配置的插件解析后会保存到Configuration的InterceptChain中  |
| Plugin         | 用来创建代理对象,包装四大对象                            |
| Invocation     | 对被代理类进行包装,可以调用proceed() 调用到被拦截的方法. |

#### 4. PageHelper 原理
用法（EmployeeController.getEmpsWithJson()）
```java
PageHelper.startPage(pn, 10);
List<Employee> emps = employeeService.getAll();
PageInfo page = new PageInfo(emps, 10);
```
先看PageHelper jar 中PageInterceptor 的源码,拦截的是Executor的两个query()方法,在这里对SQL 进行了改写
```java
//调用方言获取分页 sql
String pageSql = dialect.getPageSql(ms, boundSql, parameter, rowBounds, pageKey);
```
跟踪到最后,是在MySqlDialect.getPageSql() 对SQL 进行改写,翻页参数是从一个Page对象里面拿到的,那么Page对象是怎么传到这里的呢?

上一步,AbstractHelperDialect.getPageSql()中:
```java
Page page = getLocalPage();
return getPageSql(sql, page, pageKey);
```
Page 对象是从一个ThreadLocal 变量中拿到的,那么他是什么时候赋值的呢?

回到EmployeeController. getEmpsWithJson()中，PageHelper.startPage() 方法,把分页参数放到了ThreadLocal 变量中
```java
protected static void setLocalPage(Page page) {
LOCAL_PAGE.set(page);
}
```
关键类总结:
| 对象            | 作用         |
| --------------- | ------------ |
| PageInterceptor | 自定义拦截器 |
| Page            | 包装分页参数 |
| PageInfo        | 包装结果     |
| PageHelper      | 工具类       |
#### 5. 应用场景分析
| 作用         | 实现方式                                                     |
| ------------ | ------------------------------------------------------------ |
| 水平分表     | 对query,update 方法进行拦截,在接口上添加注解,通过反射获取接口注解,根据注解上配置的参数进行分表,修改原SQL 。 |
| 数据加解密   | update-- 加密,query--解密,获得入参和返回值                   |
| 菜单权限控制 | 对query 方法进行拦截,在方法上添加注解,根据权限配置,以及用户登陆信息,在SQL上加上权限过滤条件. |