# Docker实战

## 1. Mysql 高可用集群



###  1. 拉取`pxc`镜像

```bash
docker pull percona/percona-xtradb-cluster:5.7.21
```
### 2. 复制`pxc` 镜像(实则重命名)

```bash
docker tag percona/percona-xtradb-cluster:5.7.21 pxc

```

### 3. 删除`pxc` 原来的镜像

```bash
docker rmi percona/percona-xtradb-cluster:5.7.21

```

### 4. 创建一个单独的网段,给`mysql` 数据库集群使用

```bash
(1)docker network create --subnet=172.18.0.0/24 pxc-net
(2)docket network inspect pxc-net [查看详情]
(3)docker network rm pxc-net [删除]

```



### 5.  创建和删除`volume`

```bash
创建：docker volume create --name v1
删除：docker volume rm v1
查看详情：docker volume inspect v1
```



### 6.  创建单个`pxc` 容器demo

```bash
[CLUSTER_NAME PXC集群名字]
[XTRABACKUP_PASSWORD数据库同步需要用到的密码]

docker run -d -p 3301:3306
-v v1:/var/lib/mysql
-e MYSQL_ROOT_PASSWORD=jack123
-e CLUSTER_NAME=PXC
-e XTRABACKUP_PASSWORD=jack123
--privileged --name=node1 --net=pxc-net --ip 172.18.0.2
pxc
```



### 7. 搭建`pxc` 集群

#### 7.1 准备三个数据卷

```bash
docker volume create --name v1
docker volume create --name v2
docker volume create --name v3

```



#### 7.2 运行三个`pxc` 容器

```bash
docker run -d -p 3301:3306 -v v1:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=jack123 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=jack123 --privileged --name=node1 --net=pxc-net --ip 172.18.0.2 pxc

[CLUSTER_JOIN将该数据库加入到某个节点上组成集群]
docker run -d -p 3302:3306 -v v2:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=jack123 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=jack123 -e CLUSTER_JOIN=node1 --privileged --name=node2 --net=pxc-net --ip 172.18.0.3 pxc

docker run -d -p 3303:3306 -v v3:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=jack123 -e CLUSTER_NAME=PXC -e XTRABACKUP_PASSWORD=jack123 -e CLUSTER_JOIN=node1 --privileged --name=node3 --net=pxc-net --ip 172.18.0.4 pxc

```

#### 7.3  MYSQL工具连接测试一下



### 8. 数据库的负载均衡

1.  拉取`haproxy`镜像

   ```bash
   docker pull haproxy
   
   ```

2. 创建`harproxy` 配置文件,这里使用 `bind mounting` 的方式

   ```bash
   touch /tmp/haproxy/haproxy.cfg
   
   ```

   `haproxy.cfg`

   ```bash
   global
   	#工作目录，这边要和创建容器指定的目录对应
   	chroot /usr/local/etc/haproxy
   	#日志文件
   	log 127.0.0.1 local5 info
   	#守护进程运行
   	daemon
   defaults
   	log global
   	mode http
   	#日志格式
   	option httplog
   	#日志中不记录负载均衡的心跳检测记录
   	option dontlognull
   	#连接超时（毫秒）
   	timeout connect 5000
   	#客户端超时（毫秒）
   	timeout client 50000
   	#服务器超时（毫秒）
   	timeout server 50000
   	#监控界面
   	listen admin_stats
   	#监控界面的访问的IP和端口
   	bind 0.0.0.0:8888
   	#访问协议
   	mode http
   	#URI相对地址
   	stats uri /dbs_monitor
   	#统计报告格式
   	stats realm Global\ statistics
   	#登陆帐户信息
   	stats auth admin:admin
   	#数据库负载均衡
   	listen proxy-mysql
   	#访问的IP和端口，haproxy开发的端口为3306
   	#假如有人访问haproxy的3306端口，则将请求转发给下面的数据库实例
   	bind 0.0.0.0:3306
   	#网络协议
   	mode tcp
   	#负载均衡算法（轮询算法）
   	#轮询算法：roundrobin
   	#权重算法：static-rr
   	#最少连接算法：leastconn
   	#请求源IP算法：source
   	balance roundrobin
   	#日志格式
   	option tcplog
   	#在MySQL中创建一个没有权限的haproxy用户，密码为空
   	#Haproxy使用这个账户对MySQL数据库心跳检测
   	option mysql-check user haproxy
   	server MySQL_1 172.18.0.2:3306 check weight 1 maxconn 2000
   	server MySQL_2 172.18.0.3:3306 check weight 1 maxconn 2000
   	server MySQL_3 172.18.0.4:3306 check weight 1 maxconn 2000
   	#使用keepalive检测死链
   	option tcpka
   
   ```

3. 创建`haproxy`容器

   ```bash
   docker run -it -d -p 8888:8888 -p 3306:3306 -v /tmp/haproxy:/usr/local/etc/haproxy --name haproxy01 --privileged --net=pxc-net haproxy
   ```

4. 根据`haproxy.cfg` 文件启动`haproxy`

   ```bash
   docker exec -it haproxy01 bash
   haproxy -f /usr/local/etc/haproxy/haproxy.cfg
   ```

5. 在MYSQL数据库上创建用户,用于心跳检测

   ```bash
   CREATE USER 'haproxy'@'%' IDENTIFIED BY '';
   [小技巧[如果创建失败，可以先输入一下命令]:
   	drop user 'haproxy'@'%';
   	flush privileges;
   	CREATE USER 'haproxy'@'%' IDENTIFIED BY '';
   ]
   
   ```

6. `win` 浏览器访问

   ```bash
   http://centos_ip:8888/dbs_monitor
   用户名密码都是:admin
   ```

7. 在`win` 上的mysql 客户端连接`haproxy1`

   ```bash
   ip:centos_ip
   port:3306
   user:root
   password:jack123
   ```

8. 在`haproxy`  连接上进行数据操作,然后查看数据库集群各个节点



## 2. `Nginx` 和`Spring Boot` 项目+`MYSQL`

### 2.1   网络

####  2.1.1 网络

创建一个单独的网段

```bash
docker network create --subnet=172.18.0.0/24 pro-net

```

#### 2.1.2  网络划分

`mysql` -> `172.18.0.6`

`springboot`->`172.18.0.11/12/13`

`nginx` -> `172.18.0.10`

![image-20200417200012981](http://files.luyanan.com//img/20200417200014.png)



### 2.2. MYSQL

####  2.2.1  创建`volume`

```bash
docker volume create v1

```

#### 2.2.2  创建mysql 容器

```bash
docker run -d --name my-mysql -v v1:/var/lib/mysql -p 3301:3306 -e MYSQL_ROOT_PASSWORD=jack123 --net=pro-net --ip 172.18.0.6 mysql
```



#### 2.2.3  客户端连接, 执行mysql文件

```text
name:my-mysql
ip:centos-ip
端口:3301
user:root
password:jack123
```

```mysql
create schema db_test_springboot collate utf8mb4_0900_ai_ci;
use db_test_springboot;
create table t_user
(
    id int not null
    primary key,
    username varchar(50) not null,
    password varchar(50) not null,
    number varchar(100) not null
);    
```



###  2.3 `SpringBoot` 项目

>  `SpringBoot` + `Mybatis` 实现`CRUD` 操作,名称为`springboot-mybatis`



1. 在本地测试该项目的功能

主要是修改`application.yml` 文件中数据库的相关配置

2. 在项目根目录下执行`mvn clean package` 打成一个`jar` 包

记得修改一下`application.yml` 文件数据库的配置

`mvn clean  package -Dmaven.test.skip=true`

在`target` 目录下找到`springboot-mybatis-0.0.1-SNAPSHOT.jar`

3 在`docker`  环境中新建一个目录`springboot-mybatis`

4 上传`springboot-mybatis-0.0.1-SNAPSHOT.jar` 到该目录下,并且在此目录下创建`Dockerfile`

```dockerfile
FROM openjdk:8-jre-alpine
MAINTAINER luyanan0718
LABEL name="springboot-mybatis" version="1.0" author="luyanan0718"
COPY springboot-mybatis-0.0.1-SNAPSHOT.jar springboot-mybatis.jar
CMD ["java","-jar","springboot-mybatis.jar"]

```



5 基于`Dockerfile` 构建镜像

```bash
docker build -t sbm-image .

```



6  基于`image` 创建`contailer`

```bash
docker run -d --name sb01 -p 8081:8080 --net=pro-net --ip 172.18.0.11 sbm-image

```

7 查看日志

```bash
docker logs sb01
```

8  验证

在浏览器中访问`http://192.168.8.118:8081/user/listall`

####  2.3.1 网络问题

因为`sb01` 和`my-mysql` 在同一个`bridge` 的网段上, 所以是可以互相`ping` 通的,比如:

```bash
docker exec -it sb01 ping 172.18.0.6
or
docker exec -it sb01 ping my-mysql

```

so? `application.yml` 文件不妨这样修改一下, 也就是把ip地址直接换成容器的名字

```text
url: jdbc:mysql://my-mysql/db_test_springboot?
```



#### 2.3.2 创建多个项目容器

```bash
docker run -d --name sb01 -p 8081:8080 --net=pro-net --ip 172.18.0.11 sbm-image
docker run -d --name sb02 -p 8082:8080 --net=pro-net --ip 172.18.0.12 sbm-image
docker run -d --name sb03 -p 8083:8080 --net=pro-net --ip 172.18.0.13 sbm-image
```



### 2.4 Nginx

1. 在`centos`  的`/tmp/nginx` 下新建`nginx.conf` 文件,并进行相应的配置

    ```properties
   user nginx;
   worker_processes 1;
   events {
       worker_connections 1024;
   }
   http {
       include /etc/nginx/mime.types;
   	default_type application/octet-stream;
   	sendfile on;
   	keepalive_timeout 65;
   server {
   		listen 80;
   		location / {
   		proxy_pass http://balance;
   	}
   }
   	upstream balance{
   		server 172.18.0.11:8080;
   		server 172.18.0.12:8080;
   		server 172.18.0.13:8080;
   	}
   	include /etc/nginx/conf.d/*.conf;
   }
   
    ```

   

2. 创建`nginx` 容器

    注意: 先在centos7上创建/tmp/nginx目录，并且创建nginx.conf文件，写上内容

   ```bash
   docker run -d --name my-nginx -p 80:80 -v /tmp/nginx/nginx.conf:/etc/nginx/nginx.conf --network=pro-net --ip 172.18.0.10 nginx
   ```

3. win 浏览器访问: `ip/user/install`

   