#  分库分表之Mycat



官网: http://www.mycat.io/

Mycat 概要介绍： https://github.com/MyCATApache/Mycat-Server

入门指南: https://github.com/MyCATApache/Mycat-doc/tree/master/%E5%85%A5%E9%97%A8%E6%8C%87%E5%8D%97



## 1.Mycat介绍和核心概念

####  1.1 基本介绍

历史:从阿里的`cobar`升级而来, 由开源组织维护, 2.0 正在开发中. 

定位：运行在应用和数据库之间, 可以当作一个mysql 服务器使用,实现对Mysql数据库的分库分表, 也可以通过JDBC 支持其他的数据库. 

Mycat 的关键特性: 

1. 可以当作一个MYSQL 数据库使用
2. 支持Mysql之外的数据库, 通过JDBC实现. 
3. 解决了我们提到的很多问题, 多表`join`、分布式事务、全局序列号、翻页排序等. 
4. 支持ZK配置,带监控`mycat-web`
5. 2.0  正在开发中.

#### 1.2 核心概念

| 概念       | 含义                                                         |
| ---------- | ------------------------------------------------------------ |
| 主机       | 物理主机, 一台服务器、一个数据库、一个3306端口               |
| 物理数据库 | 真实的数据库                                                 |
| 物理表     | 真实存在的表                                                 |
| 分片       | 将原来单个数据库的数据切分后分散在不同的数据库节点           |
| 分片节点   | 分片以后数据存储的节点                                       |
| 分片键     | 分片依据的字段,例如`order` 表中以id为依据分片, id就是分片键, 通常为主键 |
| 分片算法   | 分片的规则, 例如随机、取模、范围、哈希、枚举以及各种组合算法 |
| 逻辑表     | 相当于物理表,是分片表聚合后的结果,对于客户端来说跟真实的表没有区别 |
| 逻辑数据库 | 相当于物理数据库, 是数据节点聚合后的结果,                    |



下载, 解压Mycat(有window版本, 可以在本地数据库测试)

http://dl.mycat.io/

```bash
wget http://dl.mycat.io/1.6.7.3/20190927161129/Mycat-server-1.6.7.3-release-20190927161129-linux.tar.gz
tar -xzvf Mycat-server-1.6.7.3-release-20190927161129-linux.tar.gz
```

`Mycat`解压之后有5个目录, 

| 目录     | 作用      |
| -------- | --------- |
| `bin`    | 启动目录  |
| `catlet` | 空目录    |
| `conf`   | 配置目录  |
| `lib`    | jar包依赖 |
| `logs`   | 日志目录  |



## 2. `Mycat` 配置详解

主要的配置文件有`server.xml`、`schema.xml`、`rule.xml` 和具体的分片规则文件. 

###  2.1 `server.xml`

包含系统配置信息, 

`System`标签: 例如字符集、线程数、心跳、分布式事务开关等. 

`user`标签: 配置登录用户和权限

```xml
<user name="root" defaultAccount="true">
<property name="password">123456</property>
<property name="schemas">catmall</property>
</user>
```

`Mycat`对密码加密

```bash
java -cp Mycat-server-1.6.7.3-release.jar io.mycat.util.DecryptUtil 0:root:123456
```



### 2.2  `schema.xml`

https://dev.mysql.com/doc/refman/5.7/en/glossary.html

`schema` 在Mysql 里面跟数据库是等价的. 

`schema.xml` 里面包含逻辑库、表、分片规则、分片节点和数据源, 可以定义多个`schema`

这里面有三个主要的标签(`table`、`tableNode`、`dataHost`)



####   `</table>`

表名和库名最好小写, 定义了逻辑表以及逻辑表分布的节点和分片规则

```xml
<schema name="catmall" checkSQLschema="false" sqlMaxLimit="100">
    <!-- 范围分片 -->
    <table name="customer" primaryKey="id" dataNode="dn1,dn2,dn3" rule="rang-long-cust" />
    <!-- 取模分片 -->
    <table name="order_info" dataNode="dn1,dn2,dn3" rule="mod-long-order" >
      <!-- ER 表 -->
        <childTable name="order_detail" primaryKey="id" joinKey="order_id" parentKey="order_id"/>
    </table>
    <!-- 全局表 -->
    <table name="student" primaryKey="sid" type="global" dataNode="dn1,dn2,dn3" />
</schema>
```



| 配置            | 作用                                                         |
| --------------- | ------------------------------------------------------------ |
| `primaryKey`    | 执行该逻辑表对应的真实表的主键,`Mycat` 会缓存主键(通过`primaryKey` 属性配置)与具体的`dataNode`的信息<br />当分片规则(rule) 使用非主键进行分片时, 那么在使用主键进行查询时,`Mycat` 就会通过缓存先确定记录在哪个`dataNode`上, 然后再在该`dataNode` 上执行查询. <br />如果没有缓存/缓存并没有命中的话,还是会发送所有语句给`dataNode` |
| `dataNode`      | 数据分片的节点                                               |
| `autoIncrement` | 自增长(全局序列)、true代表主键使用自增长策略                 |
| `type`          | 全局表:`global`, 其他： 不配置                               |



#### `<dataNode>`

```xml
<dataNode name="dn1" dataHost="host1" database="mycat" />

```

数据节点与物理数据库的对应关系.

#### `<dataHost/>`

配置物理主机的信息， `readHost`是从属于`writeHost`的

```xml
<dataHost name="host1" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
    <heartbeat>select user()</heartbeat>
    <!-- can have multi write hosts -->
    <writeHost host="hostM1" url="localhost:3306" user="root" password="123456">
    <!-- can have multi read hosts -->
      <readHost host="hostS2" url="192.168.8.146:3306" user="root" password="xxx"/>
    </writeHost>
    <writeHost host="hostS1" url="localhost:3316" user="root" password="123456"/>
    <!-- <writeHost host="hostM2" url="localhost:3316" user="root" password="123456"/> -->
</dataHost>
```

`balance`: 负载的配置, 决定`select`语句的负载. 

| 值   | 作用                                                         |
| ---- | ------------------------------------------------------------ |
| 0    | 不开启读写分离机制,所有读操作都发送到当前可用的`writeHost` 上 |
| 1    | 所有的读操作都随机的发送到当前的`writeHost` 对应的`readHost`和备用的`writeHost`上. |
| 2    | 所有的读操作都随机发送到所有的`writeHost`和`readHost`上.     |
| 3    | 所有的读操作都只发送到`writeHost`的`readHost`上.             |

`writeType` : 读写分离的配置, 决定`update`、`delete`、`insert` 语句的负载. 

| 值   | 作用                                                         |
| ---- | ------------------------------------------------------------ |
| 0    | 所有的写操作都发送到可用的`writeHost`上(默认第一个,第一个挂了以后发送到第二个) |
| 1    | 所有的写操作都随机的发送到`writeHost`上                      |

`switchType`: 主从切换的配置

| 值   | 作用                                                         |
| ---- | ------------------------------------------------------------ |
| -1   | 表示不自动切换                                               |
| 1    | 默认值, 表示自动切换                                         |
| 2    | 基于Mysql主从切换的状态决定是否切换, 心跳语句`show slave status` |
| 3    | 基于`Mysql  galary cluster` 的切换机制（适合集群）（1.4.1），心跳语句为 `show status like 'wsrep%'。` |



###  2.3 `rule.xml`

定义了分片规则和算法

分片规则: 

```xml
<tableRule name="rang-long-cust">
  <rule>
  <columns>id</columns>
   <algorithm>func-rang-long-cust</algorithm>
  </rule>
</tableRule>
```

分片算法

```xml
<function name="func-rang-long-cust" class="io.mycat.route.function.AutoPartitionByLong">
    <property name="mapFile">rang-long-cust.txt</property>
</function>
```



分片配置:`rang-long-cust.txt`

```text
10001-20000=1
0-10000=0
20001-100000=2
```



### 2.4 ZK配置

https://www.cnblogs.com/leeSmall/p/9551038.html

`Mycat` 也支持zk配置(用于管理配置和生成全局ID), 执行`bin`目录下的`init_zk_data.sh`, 会自动将`zkconf`下的所有配置文件上传到zk(先拷贝到这个目录)

```bash
cd /usr/local/soft/mycat/conf
cp *.txt *.xml *.properties zkconf/
cd /usr/local/soft/mycat/bin
./init_zk_data.sh
```



启用zk配置

`mycat/conf/myid.properties`

```properties
loadZk=true
zkURL=127.0.0.1:2181
clusterId=010
myid=01001
clusterSize=1
clusterNodes=mycat_01
#server booster ; booster install on db same server,will reset all minCon to 2
type=server
boosterDataHosts=dataHost1
```

注意如果指定`init_zk_data.sh` 脚本报错的话, 代表未写入成功, 此时不要启动zk配置并重启,否则本地文件会被覆盖, 

启动的时如果`oadzk=true` 启动时, 会自动从zk下载配置文件并覆盖到本地配置.

在这种情况下如果修改配置, 需要先修改`conf` 目录的配置, `copy`到`zkconf`, 再执行上传. 

![image-20200404214238782](http://files.luyanan.com//img/20200404214256.png)



### 2.5 启动停止

进入`mycat/bin` 目录(主要要先启动物理数据库)

| 操作     | 命令              |
| -------- | ----------------- |
| 启动     | `./mycat start`   |
| 停止     | `./mycat stop`    |
| 重启     | `./mycat restart` |
| 查看状态 | `./mycat status`  |
| 前台运行 | `/mycat console`  |

连接

```bash
mysql -uroot -p123456 -h 192.168.8.151 -P8066 catmall

```



##  3. Mycat 分片验证

`explain` 可以用来看路由结果

在三个数据库中建表

```mysql
CREATE TABLE `customer` (
`id` int(11) DEFAULT NULL,
 `name` varchar(255) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `order_info` (
`order_id` int(11) NOT NULL COMMENT '订单 ID', 
 `uid` int(11) DEFAULT NULL COMMENT '用户 ID',
 `nums` int(11) DEFAULT NULL COMMENT '商品数量',
 `state` int(2) DEFAULT NULL COMMENT '订单状态',
 `create_time` datetime DEFAULT NULL ON UPDATE   CURRENT_TIMESTAMP COMMENT '创建时间',
 `update_time` datetime DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
 PRIMARY KEY (`order_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `order_detail` (
 `order_id` int(11) NOT NULL COMMENT '订单号',
 `id` int(11) NOT NULL COMMENT '订单详情',
 `goods_id` int(11) DEFAULT NULL COMMENT '货品 ID',
 `price` decimal(10,2) DEFAULT NULL COMMENT '价格',
 `is_pay` int(2) DEFAULT NULL COMMENT '支付状态',
 `is_ship` int(2) DEFAULT NULL COMMENT '是否发货',
 `status` int(2) DEFAULT NULL COMMENT '订单详情状态',
 PRIMARY KEY (`order_id`,`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `student` (
`sid` int(8) NOT NULL AUTO_INCREMENT,
 `name` varchar(255) DEFAULT NULL,
 `qq` varchar(255) DEFAULT NULL,
 PRIMARY KEY (`sid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

`schema.xml`

```xml
<table name="customer" dataNode="dn1,dn2,dn3" rule="rang-long-cust" primaryKey="id"/>
<table name="order_info" dataNode="dn1,dn2,dn3" rule="mod-long-order">
<childTable name="order_detail" joinKey="order_id" parentKey="order_id" primaryKey="id"/>
</table>
<table name="student" dataNode="dn1,dn2,dn3" primaryKey="sid" type="global"/>
```



数据节点配置

```xml
<dataNode name="dn1" dataHost="host1" database="mycat"/>
<dataNode name="dn2" dataHost="host2" database="mycat"/>
<dataNode name="dn3" dataHost="host3" database="mycat"/>
<dataHost balance="0" maxCon="1000" minCon="10" name="host1" writeType="0" switchType="1"
slaveThreshold="100" dbType="mysql" dbDriver="native">
<heartbeat>select user()</heartbeat>
<writeHost host="hostM1" url="192.168.8.146:3306" password="123456" user="root"/>
</dataHost>
<dataHost balance="0" maxCon="1000" minCon="10" name="host2" writeType="0" switchType="1"
slaveThreshold="100" dbType="mysql" dbDriver="native">
<heartbeat>select user()</heartbeat>
<writeHost host="hostM1" url="192.168.8.150:3306" password="123456" user="root"/>
</dataHost>
<dataHost balance="0" maxCon="1000" minCon="10" name="host3" writeType="0" switchType="1"
slaveThreshold="100" dbType="mysql" dbDriver="native">
<heartbeat>select user()</heartbeat>
<writeHost host="hostM1" url="192.168.8.151:3306" password="123456" user="root"/>
</dataHost>
```

`schema——rule.xml——分片配置`



####  3.1 范围分片

```xml
<tableRule name="rang-long-cust">
<rule>
<columns>id</columns>
<algorithm>rang-long-cust</algorithm>
</rule>
</tableRule>
```

```xml
<function name="rang-long-cust" class="io.mycat.route.function.AutoPartitionByLong">
<property name="mapFile">rang-long-cust.txt</property>
</function>
```



customer

```mysql
INSERT INTO `customer` (`id`, `name`) VALUES (6666, '赵先生');
INSERT INTO `customer` (`id`, `name`) VALUES (7777, '钱先生');
INSERT INTO `customer` (`id`, `name`) VALUES (16666, '孙先生');
INSERT INTO `customer` (`id`, `name`) VALUES (17777, '李先生');
INSERT INTO `customer` (`id`, `name`) VALUES (26666, '周先生');
INSERT INTO `customer` (`id`, `name`) VALUES (27777, '吴先生');
```



####  3.2 取模分片(ER表)

order_info

```xml
<tableRule name="mod-long-order">
<rule>
<columns>order_id</columns>
<algorithm>mod-long</algorithm>
</rule>
</tableRule>
```

```xml
<function name="mod-long" class="io.mycat.route.function.PartitionByMod">
<property name="count">3</property>
</function>
```

```mysql
INSERT INTO `order_info` (`order_id`, `uid`, `nums`, `state`, `create_time`, `update_time`) VALUES (1, 1000001, 1, 2,
'2019-9-23 14:35:37', '2019-9-23 14:35:37');
INSERT INTO `order_info` (`order_id`, `uid`, `nums`, `state`, `create_time`, `update_time`) VALUES (2, 1000002, 1, 2,
'2019-9-24 14:35:37', '2019-9-24 14:35:37');
INSERT INTO `order_info` (`order_id`, `uid`, `nums`, `state`, `create_time`, `update_time`) VALUES (3, 1000003, 3, 1,
'2019-9-25 11:35:49', '2019-9-25 11:35:49');
```



order_detail

```mysql
INSERT INTO `order_detail` (`order_id`, `id`, `goods_id`, `price`, `is_pay`, `is_ship`, `status`) VALUES (3, 20180001, 85114752, 19.99, 1, 1, 1);
INSERT INTO `order_detail` (`order_id`, `id`, `goods_id`, `price`, `is_pay`, `is_ship`, `status`) VALUES (1, 20180002, 25411251, 1280.00, 1, 1, 0);
INSERT INTO `order_detail` (`order_id`, `id`, `goods_id`, `price`, `is_pay`, `is_ship`, `status`) VALUES (1, 20180003, 62145412, 288.00, 1, 1, 2);
INSERT INTO `order_detail` (`order_id`, `id`, `goods_id`, `price`, `is_pay`, `is_ship`, `status`) VALUES (2, 20180004, 21456985, 399.00, 1, 1, 2);
INSERT INTO `order_detail` (`order_id`, `id`, `goods_id`, `price`, `is_pay`, `is_ship`, `status`) VALUES (2, 20180005, 21457452, 1680.00, 1, 1, 2);
INSERT INTO `order_detail` (`order_id`, `id`, `goods_id`, `price`, `is_pay`, `is_ship`, `status`) VALUES (2, 20180006, 65214789, 9999.00, 1, 1, 3);
```



####  3.3  全局表

student

```xml
<table name="student" dataNode="dn1,dn2,dn3" primaryKey="sid" type="global"/>

```



```mysql
INSERT INTO `student` (`sid`, `name`, `qq`) VALUES (1, '黑白', '166669999');
INSERT INTO `student` (`sid`, `name`, `qq`) VALUES (2, 'AV 哥', '466669999');
INSERT INTO `student` (`sid`, `name`, `qq`) VALUES (3, '最强菜鸟', '368828888');
INSERT INTO `student` (`sid`, `name`, `qq`) VALUES (4, '加载中', '655556666');
INSERT INTO `student` (`sid`, `name`, `qq`) VALUES (5, '猫老公', '265286999');
INSERT INTO `student` (`sid`, `name`, `qq`) VALUES (6, '一个人的精彩', '516895555');
```



####  3.4 Mycat全局id

Mycat 全局序列实现方式主要有4种: 本地文件方式、数据库方式、本地时间戳算法、ZK。 也可以自定义业务序列. 

主要获取全局ID的前缀都是`MYCATSEQ_`



#####  3.4.1 本地文件方式

配置文件`件 server.xml`  `sequnceHandlerType`的值:

0:文件、1:数据库、2:本地时间戳、3: ZK

```xml
<property name="sequnceHandlerType">0</property>

```

文件方式:配置`conf/sequence_conf.properties`

```properties
CUSTOMER.HISIDS= 
CUSTOMER.MINID=10000001
CUSTOMER.MAXID=20000000
CUSTOMER.CURID=10000001
```

语法: `select next value for MYCATSEQ_CUSTOMER`

```mysql
INSERT INTO `customer` (`id`, `name`) VALUES (next value for MYCATSEQ_CUSTOMER, 'zhangsan');
```

优点: 本地加载, 读取速度快. 

缺点: 当Mycat 重新发布后, 配置文件中的`sequence` 需要替换. Mycat 不能做集群部署. 



#####   3.4.2  数据库方式

```xml
<property name="sequnceHandlerType">1</property>

```

配置:  `sequence_db_conf.properties`

把这张表创建在146上, 所以是db1

```properties
#sequence stored in datanode
GLOBAL=dn1
CUSTOMER=dn1
```

在第一个数据库节点上创建`MYCAT_SEQUENCE` 表:

```mysql
DROP TABLE IF EXISTS MYCAT_SEQUENCE;
CREATE TABLE MYCAT_SEQUENCE (
name VARCHAR(50) NOT NULL,
current_value INT NOT NULL,
increment INT NOT NULL DEFAULT 1,
remark varchar(100),
PRIMARY KEY(name))
 ENGINE=InnoDB;
```

注：可以在`schema.xml`  配置文件中配置这张表, 供外部访问

```xml
<table name="mycat_sequence" dataNode="dn1" autoIncrement="true" primaryKey="id"></table>
```



创建存储过程, 获取当前的`sequence`的值

```mysql
DROP FUNCTION IF EXISTS `mycat_seq_currval`;
DELIMITER ;;
CREATE DEFINER=`root`@`%` FUNCTION `mycat_seq_currval`(seq_name VARCHAR(50)) RETURNS varchar(64)
CHARSET latin1
DETERMINISTIC
BEGIN
DECLARE retval VARCHAR(64);
SET retval="-999999999,null";
SELECT concat(CAST(current_value AS CHAR),",",CAST(increment AS CHAR) ) INTO retval FROM
MYCAT_SEQUENCE WHERE name = seq_name;
RETURN retval ;
END
;;
DELIMITER ;
```

创建存储过程, 获取下一个`sequence`

```mysql
DROP FUNCTION IF EXISTS `mycat_seq_nextval`;
DELIMITER ;;
CREATE DEFINER=`root`@`%` FUNCTION `mycat_seq_nextval`(seq_name VARCHAR(50)) RETURNS varchar(64)
CHARSET latin1
DETERMINISTIC
BEGIN
UPDATE MYCAT_SEQUENCE
SET current_value = current_value + increment WHERE name = seq_name;
RETURN mycat_seq_currval(seq_name);
END
;;
DELIMITER ;
```

创建存储过程, 设置`sequence`

```mysql
DROP FUNCTION IF EXISTS `mycat_seq_setval`;
DELIMITER ;;
CREATE DEFINER=`root`@`%` FUNCTION `mycat_seq_setval`(seq_name VARCHAR(50), value INTEGER)
RETURNS varchar(64) CHARSET latin1
DETERMINISTIC
BEGIN
UPDATE MYCAT_SEQUENCE
SET current_value = value
WHERE name = seq_name;
RETURN mycat_seq_currval(seq_name);
END
;;
DELIMITER ;
```



插入记录

```mysql
INSERT INTO MYCAT_SEQUENCE(name,current_value,increment,remark) VALUES ('GLOBAL', 1, 100,'');
INSERT INTO MYCAT_SEQUENCE(name,current_value,increment,remark) VALUES ('ORDERS', 1, 100,'订单表使
用');
```

测试

```mysql
select next value for MYCATSEQ_ORDERS

```

##### 3.4.3 本地时间戳方式

ID= 64 位二进制 (42(毫秒)+5(机器 ID)+5(业务编码)+12(重复累加) ，长度为 18 位

```xml
<property name="sequnceHandlerType">2</property>

```

配置文件`sequence_time_conf.properties`

```properties
#sequence depend on TIME
WORKID=01
DATAACENTERID=01
```

验证: `select next value for MYCATSEQ_GLOBAL`

![image-20200404223523720](http://files.luyanan.com//img/20200404223525.png)



##### 3.4.5 ZK方式

修改`conf/myid.properties`
设置`loadZk=true`（启动时会从zk加载配置, 一定要注意备份配置文件, 并且先用`bin/init_zk_data.sh`,把配置文件写入到zk）

```xml
<property name="sequnceHandlerType">3</property>

```

配置文件: `sequence_distributed_conf.properties`

```properties
# 代表使用 zk
INSTANCEID=ZK
# 与 myid.properties 中的 CLUSTERID 设置的值相同
CLUSTERID=010
```

复制配置文件

```bash
cd /usr/local/soft/mycat/conf
cp *.txt *.xml *.properties zkconf/
chown -R zkconf/
cd /usr/local/soft/mycat/bin
./init_zk_data.sh
```

验证: `select next value for MYCATSEQ_GLOBAL`

![image-20200404223843594](http://files.luyanan.com//img/20200404223845.png)



####   3.5 使用

在`schema.xml` 的	`table`  标签上配置 `autoIncrement="true"`,不需要获取和指定序列的情况下, 就可以使用全局ID



## 4. Mycat 监控和日志监控

### 4.1 监控

####  4.1.1 命令行监控

连接到管理端口9066,注意必须带ip

```bash
mysql -uroot -h127.0.0.1 -p123456 -P9066

```

全部命令

```mysql
mysql>show @@help;

```

| 命令                | 作用                                                         |
| ------------------- | ------------------------------------------------------------ |
| `show @@server`     | 查看服务器状态, 包括占用内存                                 |
| `show @@database`   | 查看数据库                                                   |
| `show @@datanode`   | 查看数据节点                                                 |
| `show @@datasource` | 查看数据源                                                   |
| `show @@connection` | 该命令用于获取Mycat的前端连接状态, 即应用与mycat 的连接      |
| `show @@cache`      | 查看缓存使用情况 <br />SQLRouteCache：sql 路由缓存。 <br />TableID2DataNodeCache ： 缓存表主键与分 片对应关系。 <br />ER_SQL2PARENTID ：缓存 ER 分片中子表与 父表关系 |
| `reload @@config`   | 重新加载配置文件, 使用这个命令时mycat服务不可用              |
| `show @@sysparam`   | 查看参数                                                     |
| `show @@sql.high`   | 执行频率高的sql                                              |
| `show @@sql.slow`   | 慢SQL<br />设置慢SQL的命令:`reload @@sqlslow=5 ;`            |
| `show @@backend`    | 查看后端连接状态                                             |



####  4.1.2 命令行监控 `mycatweb` 监控

https://github.com/MyCATApache/Mycat-download/tree/master/mycat-web-1.0

`mycat-web`是mycat 提供的一个监控工具, 他依赖于zk

本地必须要运行一个zk, 必须先启动zk

下载`mycat-web`

```bash
cd /usr/local/soft
wget http://dl.mycat.io/mycat-web-1.0/Mycat-web-1.0-SNAPSHOT-20170102153329-linux.tar.gz
tar -xzvf Mycat-web-1.0-SNAPSHOT-20170102153329-linux.tar.gz
```

启动`mycat-web`

```bash
cd mycat-web
nohup ./start.sh &
```

停止

`：kill start.jar` 相关的进程

访问端口8082

http://192.168.8.151:8082/mycat/

`mycat` ` server.xml`的配置

```xml
<!-- 1 为开启实时统计、0 为关闭 -->
<property name="useSqlStat">1</property>
```

重启`mycat` 服务生效

###  4.2 日志

`log4j`的`level` 配置要改成`debug`

####  4.2.1  `wrapper.log` 日志

`wrapper.log` 日志: mycat 启动、停止、添加服务等都会被记录到此文件, 如果系统环境配置错误或者缺少配置时, 导致mycat无法启动,可以通过查看`wrapper.log` 定位具体错误原因. 



####  4.2.2  `mycat.log` 日志

`mycat.log` 为mycat的主要日志文件, 记录了启动时分配的相关buffer 信息， 数据源连接信息, 连接池、动态类加载信息等等. 

在`conf/log4j2.xml` 文件中进行相关配置, 如保留个数、大小、字符集、日志文件大小等. 

以`select` 为例

![image-20200404225553807](http://files.luyanan.com//img/20200404225555.png)



##  5. Mycat 高可用

目前Mycat 没有实现对多`Mycat` 集群的支持, 可以暂时使用`HAProxy`来做负载. 

思路：`HAProxy` 对`Mycat` 进行负载. `Keepalived`  实现`VIP`

![image (http://files.luyanan.com//img/20200406112937.png)](C:/Users/luyanan/Downloads/image%20(1).png)



## 6.Mycat注解

###  6.1 注解的作用

当关联的数据不再同一个节点的时候, Mycat是无法实现跨库`join`的. 

举例: 

如果直接在150节点插入主表数据,151插入明细表数据, 此时关联查询无法查询出来数据. 

```mysql
-- 150 节点插入
INSERT INTO `order_info` (`order_id`, `uid`, `nums`, `state`, `create_time`, `update_time`) VALUES (9, 1000003, 2673, 1, '2019-9-25 11:35:49', '2019-9-25 11:35:49'); -- 151 节点插入
INSERT INTO `order_detail` (`order_id`, `id`, `goods_id`, `price`, `is_pay`, `is_ship`, `status`) VALUES (9, 20180001, 2673, 19.99, 1, 1, 1);
```

在mycat 数据库查询, 直接查询没有查到结果

```mysql
select a.order_id,b.price from order_info a, order_detail b where a.nums = b.goods_id;
```

Mycat 作为一个中间件,有很多自身不支持的SQL语句,比如存储过程,但是这些语句在实际的数据库节点上是可以执行的. 有没有办法让Mycat 做一层透明的代理转发,直接找到目标数据节点去执行这些SQL语句呢?

那我们必须有一种方式来告诉Mycat 应该在那些节点上执行. 这个就是Mycat 的注解, 我们在需要执行的SQL 语句前面加上一段代码, 帮助Mycat 找到我们的目标节点. 



###   6.2 注解的用法

注解的形式是: 

```mysql
/*!mycat: sql=注解 SQL 语句*/
```

注解的使用方式: 

```mysql
/*!mycat: sql=注解 SQL 语句*/ 真正执行的 SQL

```

使用时, 将`=` 号后面的"注解SQL语句", 替换为需要的SQL语句即可. 

使用注解有一些限制, 或者注意的地方

| 原始SQL  | 注解SQL                                                      |
| -------- | ------------------------------------------------------------ |
| `select` | 如果需要分片,则使用能确定分片的注解, 比如`/*!mycat: sql=select * from users where user_id=1*/`<br />如果要在所有的分片上执行则可以不加能确定分片的条件 |
| `insert` | 使用`insert`的表作为注解SQL,必须能确定到某个分片<br />原始SQL插入的字段必须包含分片字段<br />非分片字段(只在某个节点上): 需要能确定到某个分片 |
| `delete` | 使用`delete`的表作为注解SQL                                  |
| `update` | 使用`update`的表作为注解SQL                                  |



使用注解并不会额外增加Mycat的执行时间,从解析复杂度以及性能考虑, 注解SQL 应该尽量简单,因为它只是用来做路由的. 

注解可以帮我们解决什么问题呢?



### 6.3 注解使用案例

####  6.3.1 创建表或存储过程

`customer.id=1` 全部路由到146

```mysql
-- 存储过程
/*!mycat: sql=select * from customer where id =1 */ CREATE PROCEDURE test_proc() BEGIN END ; -- 表
/*!mycat: sql=select * from customer where id =1 */ CREATE TABLE test2(id INT);
```



#### 6.3.2 特殊语句自定义分片

Mycat 本身不支持`insert`,`select`, 通过注解支持

```mysql
/*!mycat: sql=select * from customer where id =1 */ INSERT INTO test2(id) SELECT id FROM order_detail;
```



####  6.3.3  多表`shareJoin`

```mysql
/*!mycat:catlet=io.mycat.catlets.ShareJoin */
select a.order_id,b.price from order_info a, order_detail b where a.nums 
```



####  6.3.4 读写分离

读写分离:配置Mycat读写分离后, 默认查询都会从读节点获取数据, 但是有些场景需要获取实时数据, 如果从读节点获取数据可能因延时而无法实现实时, Mycat 支持通过注解`/*balance*/` 来强制从写节点(`write host`) 查询数据.

```mysql
/*balance*/ select a.* from customer a where a.id=6666;
```



读写分离数据库选择(1.6版本之后)

```mysql
/*!mycat: db_type=master */ select * from customer;
/*!mycat: db_type=slave */ select * from customer;
/*#mycat: db_type=master */ select * from customer;
/*#mycat: db_type=slave */ select * from customer;
```

支持直接的`!` 不被mysql 单库兼容

注解支持`#` 不被Mybatis兼容



### 6.4 注解原理

Mycat 在执行SQL 之前会先解析SQL语句, 在获取分片信息后再到对应的物理节点上执行。如果SQL 无法解析,则不能被执行. 如果语句有注解, 则会先会解析注解的内容获取分片信息,再把真正需要执行的SQL语句发送到对应的物理节点上. 

所以我们在使用注解的时候, 应该清楚的知道目标SQL 应该在哪个节点上执行, 注解的SQL 也指向这个分片,这样才能使用. 如果注解没有使用正确的条件, 会导致原始SQL 被发送到所有的节点上执行, 造成数据错误. 



##  7. 分片策略详解

分片的目标是将大量的数据和访问请求均分分布在多个节点上, 通过这种方式提升数据服务的存储和负载能力. 

总体上分为连续分片和离散分片, 还有一种是连续分片和离散分片的结合, 例如先范围后取模. 

比如范围分片(id或者时间)就是典型的连续分片, 单个分区的数量和边界是确定的. 离散分片的分区总数量和边界是确定的, 例如对key 进行哈希运算, 或者再取模。 

关键词: 范围查询、热点数据、扩容. 

连续分片优点: 

- 范围条件查询消耗资源少(不需要汇总数据)
- 扩容无需迁移数据(分片固定)

连续分片缺点:

- 存在热点数据的可能性
- 并发访问能力受限于单一或者少量`DataNode`(访问集中)

离散分片优点:

- 并发访问能力增强(负载到不同的节点)
- 范围条件查询能力提升(并行计算)

离散分片缺点:

- 数据扩容比较困难,设计到数据迁移问题
- 数据库连接消耗比较多



###  7.1 连续分片

#####  范围分片

```xml
<tableRule name="auto-sharding-long">
<rule>
<columns>id</columns>
<algorithm>rang-long</algorithm>
</rule>
</tableRule>
```

```xml
<function name="rang-long" class="io.mycat.route.function.AutoPartitionByLong">
<property name="mapFile">autopartition-long.txt</property>
</function>
```



```properties
# range start-end ,data node index
# K=1000,M=10000. 0-500M=0
500M-1000M=1
1000M-1500M=2
```

特点: 容易出现冷热数据



#####  按自然月分片

建表语句

```mysql
CREATE TABLE `sharding_by_month` (
`create_time` timestamp NULL DEFAULT NULL ON UPDATE 
    `db_nm` varchar(20) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

逻辑表

```xml
<schema name="catmall" checkSQLschema="false" sqlMaxLimit="100">
<table name="sharding_by_month" dataNode="dn1,dn2,dn3" rule="qs-sharding-by-month" />
</schema>
```

分片规则

```xml
<tableRule name="sharding-by-month">
<rule>
<columns>create_time</columns>
<algorithm>qs-partbymonth</algorithm>
</rule>
</tableRule>
```

分片算法

```xml
<function name="qs-partbymonth" class="io.mycat.route.function.PartitionByMonth">
<property name="dateFormat">yyyy-MM-dd</property>
<property name="sBeginDate">2019-10-01</property>
<property name="sEndDate">2019-12-31</property>
</function>
```

- columns 标识将要分片的表字段, 字符串类型, 与`dataformat`格式一致
- `algorithm`:为分片函数, 
- `dataFormat`: 为日期字符串格式
- `sBeginDate`为开始日期
- `sEndDate`:  为结束日期

 注意: 节点个数要大于月份的个数



测试语句

```mysql
INSERT INTO sharding_by_month (create_time,db_nm) VALUES ('2019-10-16', database());
INSERT INTO sharding_by_month (create_time,db_nm) VALUES ('2019-10-27', database());
INSERT INTO sharding_by_month (create_time,db_nm) VALUES ('2019-11-04', database());
INSERT INTO sharding_by_month (create_time,db_nm) VALUES ('2019-11-11', database());
INSERT INTO sharding_by_month (create_time,db_nm) VALUES ('2019-12-25', database());
INSERT INTO sharding_by_month (create_time,db_nm) VALUES ('2019-12-31', database());
```



另外还有按天分片(可以指定多少天一个分片)、按小时分片

###  7.2 离散分片

#####  枚举分片

将所有可能出现的值列举出来,指定分片.例如: 全局34个省, 要将不同的省的数据存放在不同的节点, 可用枚举 的方式

建表语句


```mysql
CREATE TABLE `sharding_by_intfile` (
`age` int(11) NOT NULL,
 `db_nm` varchar(20) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

逻辑表

```xml
<table name="sharding_by_intfile" dataNode="dn$1-3" rule="qs-sharding-by-intfile" />

```

分片规则

```xml
<tableRule name="sharding-by-intfile">
<rule>
<columns>sharding_id</columns>
<algorithm>hash-int</algorithm>
</rule>
</tableRule>
```

分片算法

```xml
<function name="hash-int" class="org.opencloudb.route.function.PartitionByFileMap">
<property name="mapFile">partition-hash-int.txt</property>
<property name="type">0</property>
<property name="defaultNode">0</property>
</function>
```

type: 默认值为0, 0表示`Integer`,非零表示`String`

`PartitionByFileMap.java`  通过map实现

策略文件:  `partition-hash-int.txt`

```text
16=0
17=1
18=2
```

插入数据测试:

```mysql
INSERT INTO `sharding_by_intfile` (age,db_nm) VALUES (16, database());
INSERT INTO `sharding_by_intfile` (age,db_nm) VALUES (17, database());
INSERT INTO `sharding_by_intfile` (age,db_nm) VALUES (18, database())
```

特点: 适用于枚举值固定的场景. 

#####  一致性哈希

一致性hash 有效的解决了分布式数据的扩容问题

建表语句

```mysql
CREATE TABLE `sharding_by_murmur` (
`id` int(10) DEFAULT NULL,
 `db_nm` varchar(20) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

逻辑表

```xml
<schema name="test" checkSQLschema="false" sqlMaxLimit="100">
<table name="sharding_by_murmurhash" primaryKey="id" dataNode="dn$1-3" rule="sharding-by-murmur" />
</schema>
```

分片规则

```xml
<tableRule name="sharding-by-murmur">
<rule>
<columns>id</columns>
<algorithm>qs-murmur</algorithm>
</rule>
</tableRule>
```

分片算法

```xml
<function name="qs-murmur" class="io.mycat.route.function.PartitionByMurmurHash">
<property name="seed">0</property>
<property name="count">3</property>
<property name="virtualBucketTimes">160</property>
</function>
```

测试语句

```mysql
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (1, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (2, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (3, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (4, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (5, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (6, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (7, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (8, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (9, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (10, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (11, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (12, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (13, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (14, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (15, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (16, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (17, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (18, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (19, database());
INSERT INTO `sharding_by_murmur` (id,db_nm) VALUES (20, database());
```

特点: 可以一定程序上 减少数据的迁移. 

#####  十进制取模分片

根据分片键进行十进制求模运算. 

```xml
<tableRule name="mod-long">
<rule>
<columns>sid</columns>
<algorithm>mod-long</algorithm>
</rule>
</tableRule>
```

```xml
<function name="mod-long" class="io.mycat.route.function.PartitionByMod">
<!-- how many data nodes -->
<property name="count">3</property>
</function>
```

特点: 分布均匀, 但是迁移工作量比较大. 



#####  固定分片哈希

这是先求模得到逻辑分片号, 再根据逻辑分片号直接映射到物理分片的一种散列算法

建表语句

```mysql
CREATE TABLE `sharding_by_long` (
`id` int(10) DEFAULT NULL,
    `db_nm` varchar(20) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

逻辑表

```xml
<schema name="test" checkSQLschema="false" sqlMaxLimit="100">
<table name="sharding_by_long" dataNode="dn$1-3" rule="qs-sharding-by-long" />
</schema>
```

分片规则

```xml
<tableRule name="qs-sharding-by-long">
<rule>
<columns>id</columns>
<algorithm>qs-sharding-by-long</algorithm>
</rule>
</tableRule>
```

平均分成8片(%1024的余数, 1024=128*8)

```xml
<function name="qs-sharding-by-long" class="io.mycat.route.function.PartitionByLong">
<property name="partitionCount">8</property>
<property name="partitionLength">128</property>
</function>
```

- partitionCount:  为指定分片个数列表
- partitionLength: 为分片范围列表

![image-20200406153632403](http://files.luyanan.com//img/20200406153633.png)

第二个例子: 

两个数组,分成不均匀的3个节点(%1024的余数,1024=2\*256+1\*512)

```xml
<function name="qs-sharding-by-long" class="io.mycat.route.function.PartitionByLong">
<property name="partitionCount">2,1</property>
<property name="partitionLength">256,512</property>
</function>
```

3个节点, 对1024取模余数的分布

![image-20200406153846049](http://files.luyanan.com//img/20200406153847.png)

测试语句

```mysql
INSERT INTO `sharding_by_long` (id,db_nm) VALUES (222, database());
INSERT INTO `sharding_by_long` (id,db_nm) VALUES (333, database());
INSERT INTO `sharding_by_long` (id,db_nm) VALUES (666, database());
```

特点: 在一定范围内id是连续分布的. 



##### 取模范围分布

逻辑表

```xml
<schema name="test" checkSQLschema="false" sqlMaxLimit="100">
<table name="sharding_by_pattern" primaryKey="id" dataNode="dn$0-10" rule="qs-sharding-by-pattern" />
</schema>
```

建表语句

```mysql
CREATE TABLE `sharding_by_pattern` (
`id` varchar(20) DEFAULT NULL,
 `db_nm` varchar(20) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

分片规则

```xml
<tableRule name="sharding-by-pattern">
<rule>
<columns>user_id</columns>
<algorithm>sharding-by-pattern</algorithm>
</rule>
</tableRule>
```

分片算法

```xml
<function name="sharding-by-pattern" class=" io.mycat.route.function.PartitionByPattern">
<property name="patternValue">100</property>
<property name="defaultNode">0</property>
<property name="mapFile">partition-pattern.txt</property>
</function>
```

`patternValue` 取模基数, 这里设置成100

`partition-pattern.txt` , 一共三个节点

id=19%100=19，在 dn1；
       id=222%100=22，dn2；
       id=371%100=71，dn3

```properties
# id partition range start-end ,data node index
###### first host configuration
1-20=0
21-70=1
71-100=2
0-0=0
```

测试语句: 

```mysql
INSERT INTO `sharding_by_pattern` (id,db_nm) VALUES (19, database());
INSERT INTO `sharding_by_pattern` (id,db_nm) VALUES (222, database());
INSERT INTO `sharding_by_pattern` (id,db_nm) VALUES (371, database());
```

特点: 可以调整节点的数据分布

##### 范围取模分片

建表语句

```mysql
CREATE TABLE `sharding_by_rang_mod` (
`id` bigint(20) DEFAULT NULL,
 `db_nm` varchar(20) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

逻辑表

```xml
<schema name="test" checkSQLschema="false" sqlMaxLimit="100">
<table name="sharding_by_rang_mod" dataNode="dn$1-3" rule="qs-sharding-by-rang-mod" />
</schema>
```

分片规则

```xml
<tableRule name="qs-sharding-by-rang-mod">
<rule>
<columns>id</columns>
<algorithm>qs-rang-mod</algorithm>
</rule>
</tableRule>
```

分片算法

```xml
<function name="qs-rang-mod" class="io.mycat.route.function.PartitionByRangeMod">
<property name="mapFile">partition-range-mod.txt</property>
</function>
```

`partition-range-mod.txt`

```properties
# range start-end ,data node group size
0-20000=1
20001-40000=2
```

解读: 先范围后取模, id在20000以内的, 全部分布到dn1, id在20001-40000的, %2分布到dn2、dn3. 

插入数据

```mysql
INSERT INTO `sharding_by_rang_mod` (id,db_nm) VALUES (666, database());
INSERT INTO `sharding_by_rang_mod` (id,db_nm) VALUES (6667, database());
INSERT INTO `sharding_by_rang_mod` (id,db_nm) VALUES (16666, database());
INSERT INTO `sharding_by_rang_mod` (id,db_nm) VALUES (21111, database());
INSERT INTO `sharding_by_rang_mod` (id,db_nm) VALUES (22222, database());
INSERT INTO `sharding_by_rang_mod` (id,db_nm) VALUES (23333, database());
INSERT INTO `sharding_by_rang_mod` (id,db_nm) VALUES (24444, database());
```

特点:扩容的时候旧数据无需迁移. 

##### 其他分片规则

应用指定分片: `PartitionDirectBySubString`

日期范围哈希: `PartitionByRangeDateHash`

冷热数据分片： `PartitionByHotDate`

也可以自定义分片规则: `extends AbstractPartitionAlgorithm implements RuleAlgorithm。`

###  7.3 切分规则的选择

步骤: 

1. 找到需要切分的大表, 和关联的表
2. 确定分片的字段(尽量使用主键),一般是用最频繁使用的查询条件
3. 考虑单个分片的存储容量和请求,数据增长(业务特性)、扩容和数据迁移问题

例如： 按照什么递增? 序号还是日期?主键是否还有业务意义? 

一般来说,分片数要比当前规划的节点数要大

 总结: 根据业务场景, 合理的选择分片规则





## 8. Mycat 离线扩缩容

当我们规划了数据分片, 而数据已经超过了单个节点的存储上限或者需要下线节点的时候, 就需要对数据进行重新分片. 

###  8.1 Mycat 自带的工具

#### 8.1.1 准备工作

1. mycat 所在环境安装mysql 客户端
2. mycat 的`lib` 目录下添加`mysql`的`jdbc` 驱动
3. 对扩缩容的表所有节点进行备份, 以免迁移失败后的数据恢复

####  8.1.2 步骤

以取模分片`sharding-by-mod` 缩容为例. 

![image-20200406160217558](http://files.luyanan.com//img/20200406160218.png)

1. 复制 `schema.xml`、`rule.xml` 重命名为`newSchema.xml`、`newRule.xml` 放于`conf`目录下
2. 修改`newSchema.xml `和 `newRule.xml` 配置文件为扩缩容后的mycat 配置参数(表的节点数、数据源、路由规则)

注意： 

只有节点变化的表才会进行迁移,仅分片配置变化不会迁移. 

`newSchema.xml`

```xml
<table name="sharding_by_mod" dataNode="dn1,dn2,dn3" rule="qs-sharding-by-mod" />

```

改成(减少了一个节点)

```xml
<table name="sharding_by_mod" dataNode="dn1,dn2" rule="qs-sharding-by-mod" />

```

`newRule.xml` 修改`count` 的个数

```xml
<function name="qs-sharding-by-mod-long" class="io.mycat.route.function.PartitionByMod">
<property name="count">2</property>
</function>
```

3. 修改`conf` 目录下的`migrateTables.properties` 的配置文件,告诉工具那些表需要进行扩容或者缩容, 没有出现在此配置文件的`schema` 表 不会进行数据迁移, 格式:

    注意: 

   1. 不迁移的表, 不要修改dn的个数, 否则会出错
   2. ER表,因为只有主表有分片规则, 字表不会进行迁移

   

```properties
catmall=sharding-by-mod

```

4.  `dataMigrate.sh` 中这个配置必须配置

    通过命令 `"find / -name mysqldump"` 查找`mysqldump` 路径为：`/usr/bin/mysqldump`,指定`#mysql bin` 路径为`/usr/bin`

   ```properties
   #mysql bin 路径
   RUN_CMD="$RUN_CMD -mysqlBin= /usr/bin/
   ```

5. 停止mycat 服务

6. 执行`bin/ dataMigrate.sh` 脚本

   >  必须要配置java 环境变量, 不能用openjdk

7. 脚本执行完成, 如果最后的数据迁移验证通过, 就可以将之前的`newSchema.xml` 和 `newRule.xml` 替换之前的 `schema.xml` 和 `rule.xml `文 件，并重启 mycat 即可。

注意事项： 

1. 保证分片表迁移数据前后路由规则一直(取模-取模)

2. 保证分片表迁移数据前后分片字段一致

3. 全局表将被忽略

4. 不要将非分片表配置到 `migrateTables.properties` 文件中., 

5. 暂时只支持分片表使用Mysql 作为数据源的扩容缩容

   `migrate` 限制比较多, 还可以使用`mysqldump` 方法



### 8.2 `mysqldump` 方式

系统第一次上线, 把单张表迁移到mycat,也可以用`mysqldump`

Mysql 导出

```mysql
mysqldump -uroot -p123456 -h127.0.0.1 -P3306 -c -t --skip-extended-insert gpcat > mysql-1017.sql
```



- -c 代表带列名
- -t 代表只要数据, 不要建表语句
- --skip-extended-insert 代表生成多行 insert（mycat childtable 不支持多行插入 咕泡出品，必属精品 www.gupaoedu.com 28 ChildTable multi insert not provided）



Mycat导入

```mysql
mysql -uroot -p123456 -h127.0.0.1 -P8066 catmall < mysql-1017.sql

```

Mycat导出

```mysql
mysqldump -h192.168.8.151 -uroot -p123456 -P8066 -c -t --skip-extended-insert catmall customer > mycat-cust.sql

```



其他导入方式: 

`load data local infile '/mycat/customer.txt' into table customer;`

`source sql '/mycat/customer.sql';`



## 9. 核心流程总结

官网的架构图

![image-20200406164437442](http://files.luyanan.com//img/20200406164438.png)



#### 9.1  启动

1. mycatServer 启动, 解析配置文件, 包括服务器、分片规则
2. 创建工作线程, 建立前端连接和后端连接



####  9.2 执行SQL

1. 前端连接接受MYSQL 命令

2. 解析MYSQL, mycat 用的是Druid 的 DruidParser

3. 获取路由

4. 改写MYSQL,例如两个条件在两个节点, 则变成两条单独的SQL

    例如： `select * from customer where id in(5000001, 10000001);`

   改写成

   ```mysql
   select * from customer where id = 5000001；（dn2 执行）
   select * from customer where id = 10000001；（dn3 执行）
   ```

5. 与后盾数据库建立连接

6. 发送SQL语句到MYSQL 执行

7. 获取返回结果

8. 处理返回结果, 例如排序、计算等等

9. 返回给客户端





## 10 源码下载和调试环境搭建

### 10.1 下载源代码, 导入工程

```bash
git clone https://github.com/MyCATApache/Mycat-Server
```



### 10.2 配置

`schema.xml`

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
<schema name="TESTDB" checkSQLschema="true" sqlMaxLimit="100">
<table name="travelrecord" dataNode="dn1,dn2,dn3" rule="auto-sharding-long" />
<table name="company" primaryKey="ID" type="global" dataNode="dn1,dn2,dn3" />
<table name="hotnews" primaryKey="ID" autoIncrement="true" dataNode="dn1,dn2,dn3" rule="mod-long" />
</schema>
<dataNode name="dn1" dataHost="localhost1" database="db1" />
<dataNode name="dn2" dataHost="localhost1" database="db2" />
<dataNode name="dn3" dataHost="localhost1" database="db3" />
<dataHost name="localhost1" maxCon="20" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
<heartbeat>select user()</heartbeat>
<writeHost host="hostM1" url="127.0.0.1:3306" user="root" password="123456">
    </writeHost>
</dataHost>
</mycat:schema>
```



### 10.3 表结构

本地数据库创建db1、db2、db3 数据库, 全部执行建表脚本

```mysql
CREATE TABLE `company` (
`id` bigint(20) NOT NULL AUTO_INCREMENT,
 `name` varchar(64) DEFAULT '',
 `market_value` bigint(20) DEFAULT '0', PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4;

CREATE TABLE `hotnews` (
`id` bigint(20) NOT NULL AUTO_INCREMENT,
 `title` varchar(64) DEFAULT '',
 `content` varchar(512) DEFAULT '0',
 `time` varchar(8) DEFAULT '',
 `cat_name` varchar(10) DEFAULT '',
 PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE `travelrecord` (
`id` bigint(20) NOT NULL AUTO_INCREMENT,
 `city` varchar(32) DEFAULT '',
 `time` varchar(8) DEFAULT '', 
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```





### 10.4逻辑表配置

`travelrecord` 表配置

```xml
<table name="travelrecord" dataNode="dn1,dn2,dn3" rule="auto-sharding-long" />
<tableRule name="auto-sharding-long">
<rule>
<columns>id</columns>
<algorithm>rang-long</algorithm>
</rule>
</tableRule>
<function name="rang-long" class="io.mycat.route.function.AutoPartitionByLong">
<property name="mapFile">autopartition-long.txt</property>
</function>
```



`hotnews` 表配置

```xml
<table name="hotnews" primaryKey="ID" autoIncrement="true" dataNode="dn1,dn2,dn3" rule="mod-long" />
<tableRule name="mod-long">
<rule>
<columns>id</columns>
<algorithm>mod-long</algorithm>
</rule>
</tableRule>
<function name="mod-long" class="io.mycat.route.function.PartitionByMod">
<!-- how many data nodes -->
<property name="count">3</property>
</function>
```



`company` 表配置

```xml
<table name="company" primaryKey="ID" type="global" dataNode="dn1,dn2,dn3" />

```



###  10.5 debug 方式启动

debug方式启动`main 方式

`Mycat-Server-1.6.5-RELEASE\src\main\java\io\mycat\MycatStartup.java`



### 10.6 连接本地Mycat 服务

测试语句

```mysql
insert into travelrecord(`id`, `city`, `time`) values(1, '长沙', '20191020');
insert into hotnews(`title`, `content`) values('新闻', 'aaaa');
insert into company(`name`, `market_value`) values('spring', 100);
```



### 10.7 调试入口

连接入口

`io.mycat.net.NIOAcceptor#accept`

SQL入口

`io.mycat.server.ServerQueryHandler#query`