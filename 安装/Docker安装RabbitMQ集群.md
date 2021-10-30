# **Docker安装RabbitMQ集群**

环境CentOS 7
机器IP：192.168.8.146
1个磁盘节点+2个内存节点

一、安装docker
这个过程里面可能会遇到很多问题，不要慌，根据错误提示来解决。

参考：https://www.cnblogs.com/lonelyxmas/p/10430207.html
1）更新yum源

```
sudo yum update
```

2）添加仓库

```
sudo yum-config-manager \
    --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

3）查看最新版本
如果之前安装了docker，需要卸载旧版本

```
yum list docker-ce --showduplicates | sort -r
```

4)安装Docker CE版本

```
yum install docker-ce -y
```

二、安装RabbitMQ
1）拉取RabbitMQ镜像（带managment）

```
docker pull rabbitmq:3.7.17-management
```

2）创建docker网络（让容器可以和主机通信）

```
docker network create rabbitmqnet
```

3）创建三个容器，端口分别是 5673 5674 5675

```
docker run -d \
 --name=rabbitmq1 \
 -p 5673:5672 \
 -p 15673:15672 \
 -e RABBITMQ_NODENAME=rabbitmq1 \
 -e RABBITMQ_ERLANG_COOKIE='RABBITMQ_COOKIE' \
 -h rabbitmq1 \
 --net=rabbitmqnet \
 rabbitmq:management
docker run -d \
 --name=rabbitmq2 \
 -p 5674:5672 \
 -p 15674:15672 \
 -e RABBITMQ_NODENAME=rabbitmq1 \
 -e RABBITMQ_ERLANG_COOKIE='RABBITMQ_COOKIE' \
 -h rabbitmq2 \
 --net=rabbitmqnet \
 rabbitmq:management
docker run -d \
 --name=rabbitmq3 \
 -p 5675:5672 \
 -p 15675:15672 \
 -e RABBITMQ_NODENAME=rabbitmq1 \
 -e RABBITMQ_ERLANG_COOKIE='RABBITMQ_COOKIE' \
 -h rabbitmq3 \
 --net=rabbitmqnet \
 rabbitmq:management
```

4）后两个节点作为内存节点加入集群

```
docker exec -it rabbitmq2 /bin/bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster --ram rabbitmq1@rabbitmq1
rabbitmqctl start_app
docker exec -it rabbitmq3 /bin/bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster --ram rabbitmq1@rabbitmq1
rabbitmqctl start_app
```

访问：
http://192.168.8.146:15673/
guest/guest登录

![image-20200403164313354](http://files.luyanan.com//img/20200403164314.png)