# **CentOS7安装Redis单实例**

由于环境差异，安装过程可能遇到各种各样的问题，不要慌，根据错误提示解决即可。

1、下载redis
下载地址在：redis.io
比如把Redis安装到/usr/local/soft/

```
cd /usr/local/soft/
wget http://download.redis.io/releases/redis-5.0.5.tar.gz
```

2、解压压缩包

```
tar -zxvf redis-5.0.5.tar.gz
```

3、安装gcc依赖
Redis是C语言编写的，编译需要

```
yum install gcc
```

4、编译安装

```
cd redis-5.0.5
make MALLOC=libc
```

将/usr/local/soft/redis-5.0.5/src目录下二进制文件安装到/usr/local/bin

```
cd src
make install
```

5、修改配置文件
默认的配置文件是/usr/local/soft/redis-5.0.5/redis.conf
后台启动

```
daemonize no
```

改成

```
daemonize yes
```

下面一行必须改成 bind 0.0.0.0 或注释，否则只能在本机访问

```
bind 127.0.0.1 
```

如果需要密码访问，取消requirepass的注释

```
requirepass yourpassword
```

6、使用指定配置文件启动Redis（这个命令建议配置alias）

```
/usr/local/soft/redis-5.0.5/src/redis-server /usr/local/soft/redis-5.0.5/redis.conf
```

7、进入客户端（这个命令建议配置alias）

```
/usr/local/soft/redis-5.0.5/src/redis-cli
```

8、停止redis（在客户端中）

```
redis> shutdown
```

或

```
ps -aux | grep redis
kill -9 xxxx
```