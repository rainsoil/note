# **Redis一主二从Sentinel监控配置**

开启哨兵模式，至少需要3个Sentinel实例（奇数个，否则无法选举Leader）。
本例通过3个Sentinel实例监控3个Redis服务（1主2从）。

```bash
IP地址	          节点角色&端口
192.168.8.203	Master：6379 / Sentinel : 26379
192.168.8.204	Slave ：6379 / Sentinel : 26379
192.168.8.205	Slave ：6379 / Sentinel : 26379
```

![image-20200403163130311](http://files.luyanan.com//img/20200403163131.png)

在204和205的redis.conf配置中添加一行

```
slaveof 192.168.8.203 6379
```

在203、204、205创建sentinel配置文件（单例安装后根目录下默认有sentinel.conf，可以先备份默认的配置）

```
cd /usr/local/soft/redis-5.0.5
mkdir logs
mkdir rdbs
mkdir sentinel-tmp
cp sentinel.conf sentinel.conf.bak
>sentinel.conf
vim sentinel.conf
```

sentinel.conf配置文件内容，三台机器相同

```
daemonize yes
port 26379
protected-mode no
dir "/usr/local/soft/redis-5.0.5/sentinel-tmp"
sentinel monitor redis-master 192.168.8.203 6379 2
sentinel down-after-milliseconds redis-master 30000
sentinel failover-timeout redis-master 180000
sentinel parallel-syncs redis-master 1
```

在3台机器上分别启动Redis和Sentinel

```
cd /usr/local/soft/redis-5.0.5/src
./redis-server ../redis.conf
./redis-sentinel ../sentinel.conf
```

哨兵节点的另一种启动方式：

```
./redis-server ../sentinel.conf --sentinel
```

在3台机器上查看集群状态：

```
$ /usr/local/soft/redis-5.0.5/src/redis-cli
redis> info replication
```

![image-20200403163148577](http://files.luyanan.com//img/20200403163151.png)

模拟master宕机，在203执行：

```
redis> shutdown
```

注意看sentinel.conf里面的redis-master被修改了，变成了当前master的IP端口。

```
$ /usr/local/soft/redis-5.0.5/src/redis-cli
redis> info replication
```

这个时候会有一个slave节点被Sentinel设置为master。
再次启动master，它不一定会被选举为master。

slave宕机和恢复测试省略。

注意这里有的同学遇到了坑，
1、slave可以显示master信息，而master没有slave信息。
2、master宕机后slave没有被提升为master。

可能有几个主要原因：
1、master信息配置不正确。
