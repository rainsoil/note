date: 2019年11月8日10:49:12

分类: 

- zookeeper

tag:

-  zookeeper

---
# Zookeeper单机和集群的安装

## 1. 安装 zookeeper

### 1. 下载

通过下面地址可以下载zookeeper

http://apache.fayea.com/zookeeper/

我们先建一个zk 目录, 使用

>  wget http://apache.fayea.com/zookeeper/zookeeper-3.5.5/apache-zookeeper-3.5.5-bin.tar.gz
>
> 下载zookeepe

### 解压

> tar -zxvf apache-zookeeper-3.5.6.tar.gz

解压进目录后就看到

![](http://files.luyanan.com//img/20191106142259.png)

### 3. 常见命令

#### 1. 启动ZK服务

>  bin/zkServer.sh start 

#### 2. 查看ZK 服务状态

>  bin/zkServer.sh status 

#### 3. 停止ZK 服务

>  bin/zkServer.sh stop  

#### 4. 重启ZK 服务

>  bin/zkServer.sh restart 

#### 5. 连接服务器

>  zkCli.sh -timeout 0 -r -server ip:port 



## 单机版安装

一般情况下,在开发测试环境中, 没有那么多资源的情况下, 而且也不需要特别好的稳定性的前提下, 我们可以使用单机部署.

初次使用zookeeper,需要将conf 目录下的 `zoo_sample.cfg` 文件copy 一份重命名为zoo.cfg,修改 datasDir 目录, dataDir 表示日志文件存放的路径

> 进入conf 目录
>
> cp zoo_sample.cfg  zoo.cfg

进入bin 目录下, 执行上面的命令   

> sh zkServer.sh  start   启动

##  集群安装

###  单ip 部署集群

####  修改配置文件, 拷贝多份zookeeper程序, 例如设置三个server, 分别为 apache-zookeeper-3.5.5-bin,apache-zookeeper-3.5.5-bin_2,apache-zookeeper-3.5.5-bin_3   ,每个目录下存放一份zookeeper, 并 修改各自配置文件如下:

apache-zookeeper-3.5.5-bin

```java
 The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/web/zk/apache-zookeeper-3.5.5-bin
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
server.1=192.168.91.128:2888:3888
server.2=192.168.91.128:2889:3889
server.3=192.168.91.128:2890:3890
```



apache-zookeeper-3.5.5-bin_2

```java
 The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/web/zk/apache-zookeeper-3.5.5-bin_2
# the port at which the clients will connect
clientPort=2182
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
server.1=192.168.91.128:2888:3888
server.2=192.168.91.128:2889:3889
server.3=192.168.91.128:2890:3890
```



apache-zookeeper-3.5.5-bin_3

```java
 The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/web/zk/apache-zookeeper-3.5.5-bin_3
# the port at which the clients will connect
clientPort=2183
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
server.1=192.168.91.128:2888:3888
server.2=192.168.91.128:2889:3889
server.3=192.168.91.128:2890:3890
```



注意:

> 同一ip上搭建多个节点的集群的时候, 必须注意端口号的问题，端口必须不一致才行. 
>
> 创建多个节点集群时, 在dataDir 目录下必须创建myid文件, myid 文件用于zookeeper 验证server序列等. myid 只有一行, 并且为当前server 的序号. 例如 server.1 的myid 就是1, server.2 的myid 就是2. 







###  多ip 集群



我们这里准备三台服务器准备集群的安装

> 192.168.91.128
>
> 192.168.91.129
>
> 192.168.91.130

在zookeeper 集群中, 各个节点总共有三个角色,分别是leader，follower,observer.集群模式我们采用模拟3台机器来搭建zookeeper 集群. 分别复制安装包到三台机器上并解压, 同时copy 一份 zoo.cfg.

1. 修改配置文件

     修改端口

   server.1=IP1:2888:3888

   server.2=IP1:2888:3888

   server.3=IP1:2888:3888

   > 2888:访问zookeeper的端口; 3888 : 重新选举leader 的端口
   >
   > server.A= B :C :D 其中:
   >
   > ​          A 是一个数字, 表示这个是第几号服务器
   >
   > ​          B 是这个服务器的ip 地址
   >
   > ​         C 表示是这个服务器与集群中的Leader 服务器交换信息的端口
   >
   > ​           D  表示的是万一集群中的Leader 服务器挂了, 需要一个端口来重新进行选举, 选出一个新的Leader. 
   >
   > ​         而这个端口就是用来执行选举时服务器相互通信的端口, 如果是伪集群的配置方式, 由于B 都是一样的. 
   >
   > 所以不同的Zookeeper 实例通信端口号不能一样, 所以要给他们分配不同的端口号. 
   >
   > 在集群模式下, 集群中每台机器都需要感知到整个集群中是由哪几台机器组成, 在配置文件中, 按照格式server.id = host:port:port, 每一行代表一个机器配置, id: 指的是server ID，用来标识该机器在集群中的机器序号. 
   
2. 新建datadir 目录, 设置 myid

     > 在每台zookeeper 机器上, 我们都需要在数据目录(dataDir)下创建一个myid, 该文件只有一行内容, 对应每台机器的server ID  数字. 比如server.1的myid 文件内容就是1. [必须确保每个服务器的myid 文件中的数字是不同的, 并且和自己所在机器的zoo.cfg 中 server.id 的id 值是一致的, id 的范围是 1到255]

3. 启动zookeeper

启动自个服务器目录下的zookeeper就行了. 



### 连接

可以使用以下命令连接一个zk集群

> ```
> bin/zkCli.sh -server 192.168.229.160:2181,192.168.229.161:2181,192.168.229.162:2181
> ```



![](http://files.luyanan.com//img/20191106151031.png)

如图则显示连接成功

从日志输入上来看, 客户端连接的进程是随机分配的. 






