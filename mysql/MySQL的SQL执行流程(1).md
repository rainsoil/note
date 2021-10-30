#  MySQL的执行流程

MySQL的发展历史和版本分支

| 时间       | 里程碑                                                       |
| ---------- | ------------------------------------------------------------ |
| 1996年     | MySQL 1.0发布,它的历史可以追溯到1979年,作者Monty用BASIC设计的一个报表工具 |
| 1996年10月 | 3.11.1 发布, MySQL没有2.0版本                                |
| 2000年     | ISAM升级为MyISAM引擎,MySQL开源                               |
| 2003年     | MySQL4.0 发布,集成了InnoDB存储引擎                           |
| 2005年     | MySQL5.0版本发布,提供了视图、存储过程等功能                  |
| 2008年     | MySQL AB公司被Sun公司收购,进入Sun MySQL时代                  |
| 2009年     | Oracle公司收购Sun公司, 进入Oracle MySQL时代                  |
| 2010年     | MySQL5.5发布,InnoDB 成为默认的存储引擎                       |
| 2016年     | MySQL发布8.0.0版本.为什么没有6、7呢?5.6可以当成6.x, 5.7 可以当成7.x |

因为Mysql 是开源的(也有收费版本)，所以在MySQL 稳定版本的基础上也发展出来了很多的分支,就想`Linux` 一样, 有`Ubuntu`、`RedHat`、`ContOS`、`Fedora [fɪ'dɔrə]`、 `Debian[Deb'-ee-en]`等等. 

大家最熟悉的应该是`MariaDB`,因为`Contos7` 里面自带了一个`MariaDB`,它是怎么来的呢? Oracle 收购MySQL了之后, MySQL创始人之一`Monty`担心MySQL数据库的发展的未来(开发缓慢,封闭,可能会被闭源),就创建了一个分支`MariaDB`,默认使用全新的`Maria`存储引擎,它是原`MyISAM` 存储引擎的升级版本. 

其他流行分支:

`Percona Server`是MySQL最重要的分支之一,它基于`InnoDB`存储引擎的基础上,提高了性能和易管理性,最后形成了增强版的`XtraDB` 引擎, 可以用来更好的发挥服务器硬件上的性能. 

国内有一些MySQL的分支或者自研的存储引擎,比如网易的`InnoSQL`、极数云舟的`ArkDB`.

我们操作数据库有各种各样的方式,比如`Linux` 系统中的命令行,比如数据库工具`Navicat`、比如程序,例如Java语言中的`JDBC API` 或者`ORM`框架. 

大家有没有思考过,当我们的工具或者程序连接上数据库之后,实际上发生了什么事情呢? 它的内部是怎么工作的呢? 

以一条查询语句为例,我们来看下MySQL的工作流程是怎么样的? 

 一条查询SQL语句是怎么执行的?

![image-20200514112627726](http://files.luyanan.com//img/20200514112637.png)

 我们的程序或者工具要操作数据库,第一步要做什么事情呢? 

跟数据库建立连接

##   1 通讯协议

首先,MySQL 必须运行一个服务,监听默认的3306端口. 

在我们开发系统跟第三方对接的时候, 必须要弄清楚的有两件事情.

第一个就是通讯协议, 比如我们使用的是`HTTP` 还是`WebService`还是`TCP`.

第二个是消息格式,比如我们用`XML` 格式还是`JSON`格式, 还是定长格式? 报文头长度多少,包含什么内容,每个字段的详细含义 . 

###  1.1 通讯协议

MySQL 是支持多种通信协议的,可以使用同步/异步的方式, 支持长连接/短连接,这里我们拆分来看,第一个是通讯类型. 

##### **通讯类型:同步或者异步**

**同步通信的特点:**

1. 同步通信依赖于被调用方,受限于被调用方的性能,也就是说,应用操作数据库,线程会阻塞,等待数据库的返回. 
2. 一般只能做到一对一的, 很难做到一对多的通信. 



**异步跟同步相反:**

1. 异步可以避免应用阻塞,但是不能节省SQL执行的时间
2. 如果异步存在并发,每一个SQL的执行都要单独建立一个连接,避免数据混乱.但是这样会给服务器带来巨大的压力(一个连接就会创建一个线程,线程间切换会占用大量的CPU资源).另外异步通信还带来的编码的复杂度,所以一般不建议使用. 如果要使用异步, 必须使用连接池,排队从连接池获取连接而不是创建新连接. 

一般来说, 我们连接数据库都是同步连接. 

##### 连接方式:长连接或者短连接

MySQL既支持短连接,也支持长连接. 短连接就是操作完毕以后, 马上`close`掉. 长连接可以保持打开,减少服务器创建和释放连接的消耗 ,后面的程序访问的时候还可以使用这个连接. 一般我们会在线程池中使用长连接. 

保持长连接会消耗内存,长时间不活动的连接,MySQL服务器会断开. 

```mysql
show global variables like 'wait_timeout'; -- 非交互式超时时间，如 JDBC 程序
show global variables like 'interactive_timeout'; -- 交互式超时时间，如数据库工具
```

![image-20200514114849958](http://files.luyanan.com//img/20200514114851.png)

默认都是28800秒,8个小时. 

那我们怎么看MySQL当前有多少个连接呢? 

可以使用`show status`  命令查看

```bash
show global status like 'Thread%';
```

![image-20200514123727144](http://files.luyanan.com//img/20200514123731.png)

`Threads_cached`: 缓存中线程连接数

`Threads_connected`： 当前打开的连接数

`Threads_created`:为处理连接创建的线程数

`Threads_running`: 非睡眠状态的连接数,通常指并发连接数. 

每产生一个连接或者一个会话,在服务端会创建一个线程来处理. 反过来,如果要杀死会话, 那就`Kill` 线程. 

有了连接数,怎么知道当前连接的状态? 

可以使用`SHOW PROCESSLIST;`(root用户), 查看SQL的执行状态. 

https://dev.mysql.com/doc/refman/5.7/en/show-processlist.html

![image-20200514134806326](http://files.luyanan.com//img/20200514134807.png)

一些常见的状态:

https://dev.mysql.com/doc/refman/5.7/en/thread-commands.html

| 状态                           | 含义                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| `Sleep`                        | 线程正在等待客户端,以向他发送一个新语句                      |
| `Query`                        | 线程正在执行查询或者往客户端发送数据                         |
| `Locked`                       | 该查询被其它查询锁住                                         |
| `Copying to tmp table on disk` | 临时结果集合大于`tmp_table_size`, 线程把临时表从存储器内部格式改为磁盘模式,以节省存储器 |
| `Sending data`                 | 线程正在为`select` 语句处理行,同时正在向客户端发送数据       |
| `Sorting for group`            | 线程正在分类,以满足`GROUP BY`要求                            |
| `Sorting for order`            | 线程正在分类,以满足`ORDER BY` 要求                           |

MySQL服务允许的最大连接数是多少 呢? 

在5.7版本中， 默认是151个,最大可以设置为`16384（2^14）`

```bash
show variables like 'max_connections';

```

![image-20200514142154541](http://files.luyanan.com//img/20200514142156.png)

`show`的参数说明：

1. 级别: 会话`session`级别(默认),全局`global` 级别
2. 动态修改: `set` 重启后失败; 永久生效需要修改配置文件`/etc/my.cnf`

```bash
set global max_connections = 1000;
```

**通讯协议**

MySQL支持哪些通信协议呢? 

第一种是`Unix Socket`

比如我们在`linxu`服务器上,如果没有指定`-h`参数,它就用`socket`方式登录(省略了`-S /var/lib/mysql/mysql.sock`)

![image-20200514142902367](http://files.luyanan.com//img/20200514142903.png)

 它不用通过网络协议,也可以连接到MySQL服务器,它需要用到服务器上的一个物理文件(`/var/lib/mysql/mysql.sock`).

```mysql
select @@socket;
```

如果指定了`-h`参数,就会用第二种方式,`TCP/IP`协议. 

```bash
mysql -h 192.168.56.20 -uroot -pluyanan

```

我们的编程一眼的连接模块都是通过`TCP`协议连接到MySQL服务器的,比如`mysql-connector-java-x.x.xx.jar`

另外还有命名管道(`Named Pipes`)和内存共享(`Share Memory`) 的方式,这两种通讯方式只能在`Window` 上使用,一般用到较少. 



### 1.2 通讯方式

第二个是通讯方式

![image-20200514143436152](http://files.luyanan.com//img/20200514143437.png)

**单工:**

在两台计算机通信的时候,数据的传输是单向的,生活中的类比: 遥控器. 

**半双工:**

在两台计算机之间, 数据传输是双向的,你可以给我发送,我也可以给你发送,但是在这个通讯连接中,同一个时间只能有一台服务器在发送数据,也就是你要给我发送的话, 也必须等我发给你完了之后你才能给我发。 生活中的类比: 对讲机. 

**全双工:**

数据的传输是双向的, 而且可以同时传输. 生活中的类比: 打电话. 

**MySQL使用了半双工的通信方式?**

要么是客户端向服务端发送数据,要么是服务端向客户端发送数据,这两个动作不能同时发生,所以客户端发送SQL给服务端的时候(在一次连接中), 数据是不能分成小块发送的,不管你的SQL语句多大, 都不能一次性发送. 

比如我们用`Mybatis` 动态SQL生成一个批量插入的语句,插入100W条数据,`values` 后面跟了一长串的内容, 或者`where`条件`in` 里面的值太多了,会出现问题. 

这个时候我们必须调整MySQL服务器配置的`max_allowed_packet`参数的值(默认是4M)，把它调大, 否则就会报错. 

另一方面,对于服务端来说,也是一次性发送所有的数据,不能因为你已经取到想要的数据就中断操作,这个时候会对网络和内存产生大量的消耗. 

所以, 我们一定要再程序里面避免不带`limit` 的这种操作,比如一次性把满足条件的数据全部查出来,一定要先`count`一下,如果数据量大的话, 可以分批查询. 

执行一条查询语句, 客户端跟服务端建立连接后,下一步要做什么呢? 

##  2 查询缓存

MySQL 内部自带了一个缓存模块. 

缓存的作用我们知道，把数据以`KV`的形式放到内存中,可以加快数据的读取速度,也可以减少服务器处理的时间.但是MySQL的缓存我们好像比较陌生,从来没有去配置过,也不知道它什么时候生效? 

比如`user_innodb` 有500W条数据, 没有索引,我们在没有索引的字段上执行同样的查询,大家觉得第二次会快吗? 

```mysql
select * from user_innodb where name='张三';
```

发现第二次速度并没有快, 缓存没有生效,为什么呢? MySQL 的缓存默认是关闭的,

```mysql
show variables like 'query_cache%';

```

默认关闭的意思就是不推荐使用,为什么MySQL不推荐使用它自带的缓存呢? 

主要是因为MySQL自带的缓存的应用场景有限,第一个是他要求SQL语句必须一模一样,中间多一个空格,字母大小写不同都被认为是不同的SQL.

第二个就是表里面的任意一条数据发生变化的时候,这种表的所有的缓存都会失效,所以对于有大量数据更新的场景,也不适合. 

所以缓存这块,我们还是交给`ORM`框架(比如`Mybatis` 默认开启了一级缓存), 或者独立的缓存服务,比如`Redis`来处理更合适. 

在MySQL8.0 中，查询缓存已经被移除了. 



## 3 语法解析和预处理(`Parser` & `Preprocessor`)

我们没有使用缓存的话, 就会跳过缓存的模块,那下一步我们要做什么呢? 

这里有一个疑问, 为什么我的一条SQL语句能够被识别呢? 假设我随便执行一个字符串`aaa`, 服务端就会报出一个1064的错. 

![image-20200514150324087](http://files.luyanan.com//img/20200514150325.png)

它是怎么知道我输入的内容是错误的呢? 

这个就是MySQL的`Parser` 解析器和`PreProcessor` 预处理模块. 

这一步主要做的事情就是对语句基于SQL的语法进行词法和语法分析和语义的解析. 

### 3.1 词法解析

词法分析就是把一个完整的SQL 语句打碎成一个个的单词. 

比如一个简单的SQL语句

```mysql
select name from user where id  = 1;
```

它会被打碎成8个符号, 每个符号是什么类型? 从哪里开始到哪里结束. 

###  3.2 语法解析

第二步就是语法分析, 语法分析就会SQL做一些语法检查,比如单引号有没有闭合,然后根据MySQL定义的语法规则,根据SQL生成一个数据结构,这个数据结构我们把它叫做解析数(`select_lex`).

![image-20200514151911435](http://files.luyanan.com//img/20200514151912.png)

任何数据库的中间件,比如`Mycat`、`Sharding-JDBC(用到了Druid Parser)`,都必须要有词法和语法分析功能,在市面上也有很多的开源的词法解析的工具(比如`LEX`、`Yacc`).



### 3.3 预处理器

问题:如果我写了一个语法和词法都正确的SQL,但是表名或者字段都不存在,会在哪里报错呢? 是在数据库的执行层还是解析器? 比如:

```myslq
select * from user2;
```

解析器可以分析语法, 但是它怎么知道数据库里面有什么表? 表里面有什么字段呢? 

实际上还是在解析的时候报错, 解析SQL的环节里面有个预处理器. 

它会检查生成的解析树, 解决解析器无法解析的语义. 比如他会检查表和列表是否存在, 检查名称和别名,保证没有歧义.

预处理之后得到一个新的解析数.

## 4 查询优化(`Query Optimizer`)和查询执行计划

###  4.1 什么是优化器?

得到解析树之后,是不是执行SQL语句了呢? 

这里我们有一个问题, 一条SQL语句是不是只有一种执行方式? 或者说数据库最终执行的SQL是不是就是我们发送的SQL呢? 

这个答案是否定的, 一条SQL 语句是可以有很多种执行方式的,最终返回相同的结果, 他们是等价的,但是如果有那么多的执行方式,这些执行方式最终是怎么得到呢? 最终选择哪一种方式去执行? 根据什么判断标准去选择> 

这个就是MySQL的查询优化器的模块`Query Optimizer`

查询优化器的目的就是根据解析数生成不同的执行计划(`Execution Plan`),然后选出一种最优的执行计划,MySQL里面使用的是基于开销(`cost`)的优化器,那种执行计划开销最小, 就用哪种? 

可以使用这个命令查看查询的开销

```mysql
show status like 'Last_query_cost';

```

https://dev.mysql.com/doc/refman/5.7/en/server-status-variables.html



### 4.2 优化器可以做什么?

 MySQL的优化器能处理那些优化类型呢? 

举两个简单的例子: 

1. 当我们对多张表进行关联查询的时候,以哪个表的数据作为基准表. 
2. 有多个索引可以使用的时候,选择哪个索引. 

实际上,对于每一种数据库来说,优化器的模块都是必不可少的,他们通过复杂的算法尽可能优化查询效率的目标. 

如果对于优化器的细节感兴趣,可以看看《数据库查询优化器的艺术-原理解析与SQL 性能优化》。

![image-20200514154031805](http://files.luyanan.com//img/20200514154032.png)

但是优化器也不是万能的,并不是再垃圾的SQL语句都能自动优化,也不是每次都能选择到最优的执行计划,大家在编写SQL的时候还是要注意的. 

如果我们想知道优化器是怎么工作的,它生成了几种执行计划,每种执行计划是`cost`是多少? 应该怎么做? 

### 4.3 优化器是怎么得到执行计划的?

https://dev.mysql.com/doc/internals/en/optimizer-tracing.html

首先我们要启动优化器的追踪(默认是关闭的)

```mysql
SHOW VARIABLES LIKE 'optimizer_trace';
set optimizer_trace='enabled=on';
```

注意开启这开关是会消耗性能的,因为他要把优化分析的结果写到表里面,所以不要轻易开启,或者查看完之后关闭它(搞改成`off`).

注意:参数分为`session`和`global`级别. 

接着我们执行一个SQL语句,优化器就会生成执行计划 . 

```mysql
select t.tcid from teacher t,teacher_contact tc where t.tcid = tc.tcid;
```

这个时候优化器分析的过程已经记录到系统表里面了, 我们可以查询

```mysql
select * from information_schema.optimizer_trace\G

```

它是一个`JSON`类型的数据，主要分为三部分, 准备阶段、优化阶段、执行阶段. 

![image-20200514154853239](http://files.luyanan.com//img/20200514154854.png)

`expanded_query`是 优化后的SQL语句. 

`considered_execution_plans` 里面列出了所有的执行计划. 

分析玩了之后记得关闭它. 

```mysql
set optimizer_trace="enabled=off";
SHOW VARIABLES LIKE 'optimizer_trace';
```



###  4.4 优化器得到的结果

优化完之后,得到一个什么东西呢? 

优化器最终会把解析树变成一个查询执行计划,查询执行计划是一个数据结构. 

当然, 这个执行计划是不是一定是最优的执行计划呢? 不一定, 因为MySQL也有可能覆盖不到所有的执行计划. 

我们怎么查看MySQL的执行计划呢? 比如多张表关联查询,先查询哪张表? 在执行查询的时候可能用到哪些索引? 实际上用到了哪些索引? 

MySQL 提供了一个执行计划的工具,我们在SQL语句前加上`EXPLAIN`, 就可以看到执行计划的信息. 

```mysql
EXPLAIN select name from user where id=1;
```

> 注意: `Explain`的结果也不一定是最终执行的方式. 



## 5 存储引擎

得到执行计划后,SQL语句是不是就可以执行了呢? 

问题又来了: 

1. 从逻辑的角度来说, 我们的数据是放到哪里? 或者说放到一个什么样的结构里面呢? 
2. 执行计划在哪里执行? 是谁去执行? 

### 5.1 存储引擎基本介绍

我们先回答第一个问题: 在关系型数据库里面,数据是放到什么结构里? 

放到表`Table`里面的. 我们可以把表理解为`Excel` 电子表格的形式,所以我们的表在存储数据的同时,还要组织数据的存储结构,这个存储结构就是由我们的存储引擎决定的,所以我们也可以把存储引擎叫做表类型. 

在MySQL里面,支持多种存储引擎,他们是可以相互替换的,所以叫做插件式的存储引擎,为什么要搞这么多的存储引擎呢? 一种还不够用吗? 

### 5.2 查看存储引擎

比如我们数据库里面已经存在的表,我们怎么查看他们的存储引擎呢?

```mysql
show table status from `库名`;
```

或者通过`DDL` 建表语句查看

在MySQL里面, 我们创建的每一张表都可以指定它的存储引擎,而不是一个数据库只能使用一个存储引擎. 存储引擎的使用是以表为单位的,而且,创建表之后还可以修改存储引擎. 

我们说一张表使用存储引擎决定我们存储数据的结构,那在服务器上他们是怎么储存的呢? 我们先要找到数据库存放数据的路径. 

```mysql
show variables like 'datadir';

```

默认情况下，每个数据库都有自己的一个文件夹, 

任何一个存储引擎都有一个`frm` 文件, 这个是表结构定义文件.,

![image-20200514161008126](http://files.luyanan.com//img/20200514161009.png)

不同的存储引擎存储数据的方式是不一样的,产生的文件也是不一样的,`innodb`是一个,`memory`没有,`myisam`是两个. 

这些存储引擎是差别在哪里呢? 

### 5.3 存储引擎比较

常见的存储引擎: 

`MyISAM` 和`Innodb`是我们用的最多的两个存储引擎,在`MySQL5.5` 版本之前,默认的存储引擎是`MyISAM`,它是MySQL自带的,我们创建表的时候不指定存储引擎,它就会使用`MyISAM`作为存储引擎. 

`MyISAM`的前身是 `ISAM`（`Indexed Sequential Access Method`: 利用索引, 顺序读取数据的方法).

5.5 版本之后默认的存储引擎就改成了`InnoDB`,它是第三方公司为MySQL开发的,为什么要改呢? 最主要的原因是`InnoDB` 支持事务, 支持行级别的锁,对于业务一致性要求高的场景来说更适合. 

这个里面又有 `Oracle` 和 `MySQL` 公司的一段恩怨情仇。 `InnoDB` 本来是 `InnobaseOy` 公司开发的，它和 `MySQL AB` 公司合作开源了 `InnoDB` 的代码。但是没想到 `MySQL` 的竞争对手 `Oracle` 把 `InnobaseOy` 收购了。 后来 08 年 `Sun` 公司（开发 Java 语言的 `Sun`）收购了 `MySQL AB`，09 年 `Sun` 公司 又被 `Oracle` 收购了，所以 `MySQL`，`InnoDB` 又是一家了。有人觉得 `MySQL` 越来越像 `Oracle`，其实也是这个原因。

那么除了这两个我们最熟悉的存储引擎,数据库还支持其他哪些常用的存储引擎呢? 

**数据库支持的存储引擎?**

我们可以用这个命令查看数据库对存储引擎的支持情况. 

```bash
show engines ;

```

![image-20200514162317843](http://files.luyanan.com//img/20200514162319.png)

其中有存储引擎的描述和对事物、`XA`协议和`Savepoints`的支持. 

- `XA` 协议用来实现分布式事务(分为本地资源管理器、事务管理器)
- `Savepoints` 用来实现子事务(嵌套事务), 创建了一个`Savepoints`之后,事务就可以回滚到这个点, 不会影响到创建`Savepoints`之前的操作. 

这些数据库支持的存储引擎,分别有哪些特性? 

https://dev.mysql.com/doc/refman/5.7/en/storage-engines.html

#####  **MyISAM（3 个文件）**

> These tables have a small footprint. Table-level locking limits the performance in read/write workloads, so it is often used in read-only or read-mostly workloads in Web and data warehousing configurations

应用范围比较小,表级别锁定限制了读/写的性能, 因此在web和数据仓库配置中,它通常只用于只读或者以读为主的工作. 

**特点:**

- 支持表级别的锁(插入/更新会锁表), 不支持事务. 

- 拥有较高的插入(`insert`)和查询(`select`)的速度. 

- 存储了表的行数(`count`的速度更快)

   (怎么向数据库快速插入100W条数据呢? 我们可以先用`MyISAM`插入数据, 然后修改存储引擎为`InnoDB`的操作)

- **适合:** 只读之类的数据分析项目. 

#####  `InnoDB（2 个文件）`

https://dev.mysql.com/doc/refman/5.7/en/innodb-storage-engine.html

> The default storage engine in MySQL 5.7. InnoDB is a transaction-safe (ACID compliant) storage engine for MySQL that has commit, rollback, and crash-recovery capabilities to protect user data. InnoDB row-level locking (without escalation to coarser granularity locks) and Oracle-style consistent nonlocking reads increase multi-user concurrency and performance. InnoDB stores user data in clustered indexes to reduce I/O for common queries based on primary keys. To maintain data integrity, InnoDB also supports FOREIGN KEY referential-integrity constraints.

`mysql5.7` 中默认的存储引擎, `InnoDB`是一个事务安全(与`ACID`兼容)的MySQL存储引擎,它具有提交、回滚和崩溃恢复的功能来保护用户数据.`InnoDB` 行级锁(不升级为更粗粒度的锁)和`Oracle`风格的一致非锁读提高了多用户并发性和性能. `innoDB` 将用户数据存储在聚集索引中,以减少基于主键的常见查询的`I/O`, 为了保持数据完整性, `InnoDB`还支持外键引用完整性约束. 

**特点:**

-  支持事务,支持外键,因此数据的完整性、一致性更高, 

- 支持行界别的锁和表级别的锁. 

- 支持读写并发,写不阻塞读(MVCC)，

- 特殊的索引存放方式,可以减少IO,提升查询效率. 

- **适合:** 经常更新的表,存在并发读写或者有事务处理的业务系统. 

  

  

  

#####  `Memory（1 个文件）`

> Stores all data in RAM, for fast access in environments that require quick lookups of non-critical data. This engine was formerly known as the HEAP engine. Its use cases are decreasing; InnoDB with its buffer pool memory area provides a general-purpose and durable way to keep most or all data in memory, and NDBCLUSTER provides fast key-value lookups for huge distributed data sets

将所有数据都存储在`RAM`中,以便在需要快速查找非关键数据的环境中快速访问. 这个引擎以前被称为堆引擎. 其使用案例正在减少,`InnoDB` 及其缓冲池内存区域提供了一种通用、持久的方法来将大部分或所有的数据都保存在内存中,而`ndbcluster` 为大型分布式数据集提供了快速的键值查找. 

**特点**：

- 把数据放在内存中,读写速度很快,但是数据库重启或者崩溃, 数据会全部消失, 只适合做临时表. 
- 将表中的数据存储到内存中. 



#####  `CSV（3 个文件）`

> Its tables are really text files with comma-separated values. CSV tables let you import or dump data in CSV format, to exchange data with scripts and applications that read and write that same format. Because CSV tables are not indexed, you typically keep the data in InnoDB tables during normal operation, and only use CSV tables during the import or export stage.

它的表实际上是带有逗号分隔的文本文件,`csv`表允许以`csv` 格式导入或者转储数据, 以便以读写相同格式的脚本和应用程序交换数据. 因为`csv`表没有索引,所以通常在正常操作期间将数据保存在`innoDB`表中,并且只在导入或者导出阶段使用`csv`., 

**特点:**

- 不允许空行,不支持索引
- 格式通用,可以直接编辑,适合在不同数据库之间导入导出. 





##### `Archive（2 个文件）`

> These compact, unindexed tables are intended for storing and retrieving large amounts of seldom-referenced historical, archived, or security audit information.

这些紧凑的未索引的表用于存储和检索大量很少引用的历史、存档和安全审计信息,

特点： 不支持索引, 不支持`update delete`



这是MySQL里面常见的一些存储引擎,我们看到了, 不同的存储疫情提供的特性不一样,他们有不同的存储机制、索引方式、锁定水平等功能. 

我们在不同的业务场景中对数据操作的要求不同,就可以选择不同的存储引擎来满足我们的需求, 这个就是MySQL支持那么多存储引擎的原因. 



### 5.4 如何选择存储引擎?

如果对数据一致性要求比较高,需要事务支持,可以选择`InnoDB`. 

如果数据查询多更新少,对查询性能要求比较高,可以选择`MyIASM`

如果需要一个用于查询的临时表,可以选择`Memory`.

如果所有的存储引擎都不能满足你的要求,并且技术能力足够,可以根据官网内部手册用C语言开发一个存储引擎. 

https://dev.mysql.com/doc/internals/en/custom-engine.html





##  6 执行引擎(`Query Execution Engine`),返回结果.

OK,存储引擎分析完了,它是我们存储数据的形式,继续第二个问题,是谁使用执行计划去操作执行引擎呢? 

这就是我们的执行引擎, 它利用存储引擎提供的相应的API 来完成操作. 

为什么我们修改了表的存储引擎,操作方式不需要做任何改变? 因为不同功能的存储引擎实现的API是相同的。 

最后将数据返回给客户端,即使没有结果也是要返回的. 






