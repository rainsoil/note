# **CentOS7安装Zookeeper**

创建目录

```
mkdir -p /usr/local/soft/zookeeper
cd /usr/local/soft/zookeeper
```

下载解压

```
wget https://archive.apache.org/dist/zookeeper/zookeeper-3.4.9/zookeeper-3.4.9.tar.gz
tar -zxvf zookeeper-3.4.9.tar.gz
cd zookeeper-3.4.9
mkdir data
mkdir logs
```

修改配置文件

```
cd conf
cp zoo_sample.cfg zoo.cfg
```

修改zoo.cfg

```
# 数据文件夹
dataDir=/usr/local/services/zookeeper/zookeeper-3.4.9/data

# 日志文件夹
dataLogDir=/usr/local/services/zookeeper/zookeeper-3.4.9/logs
```

配置环境变量

```
vim /etc/profile
```

在尾部追加

```
# zk env
export ZOOKEEPER_HOME=/usr/local/soft/zookeeper/zookeeper-3.4.9/
export PATH=$ZOOKEEPER_HOME/bin:$PATH
export PATH
```

编译生效

```
source /etc/profile
```

启动ZK

```
cd ../bin
zkServer.sh start
```

查看状态

```bash
zkServer.sh status
```