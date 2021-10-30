#   Docker Swarm

> `官网`：<https://docs.docker.com/swarm/>

## 1 Install Swarm

### 1.1 环境准备

> (1)根据`Vagrantfile`创建3台`centos`机器
>
> [**大家可以根据自己实际的情况准备3台centos机器，不一定要使用vagrant+virtualbox**]
>
> 新建`swarm-docker-centos7`文件夹，创建`Vagrantfile`

```ruby
boxes = [
    {
        :name => "manager-node",
        :eth1 => "192.168.0.11",
        :mem => "1024",
        :cpu => "1"
    },
    {
        :name => "worker01-node",
        :eth1 => "192.168.0.12",
        :mem => "1024",
        :cpu => "1"
    },
    {
        :name => "worker02-node",
        :eth1 => "192.168.0.13",
        :mem => "1024",
        :cpu => "1"
    }
]

Vagrant.configure(2) do |config|

  config.vm.box = "centos/7"
  
   boxes.each do |opts|
      config.vm.define opts[:name] do |config|
        config.vm.hostname = opts[:name]
        config.vm.provider "vmware_fusion" do |v|
          v.vmx["memsize"] = opts[:mem]
          v.vmx["numvcpus"] = opts[:cpu]
        end

        config.vm.provider "virtualbox" do |v|
          v.customize ["modifyvm", :id, "--memory", opts[:mem]]
		  v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
		  v.customize ["modifyvm", :id, "--name", opts[:name]]
        end

        config.vm.network :public_network, ip: opts[:eth1]
      end
  end

end
```

> (2)进入到对应的centos里面，使得root账户能够登陆，从而使用XShell登陆

```bash
vagrant ssh manager-node/worker01-node/worker02-node
sudo -i
vi /etc/ssh/sshd_config
修改PasswordAuthentication yes
passwd    修改密码
systemctl restart sshd
```

> (3)在win上ping一下各个主机，看是否能ping通

```
ping 192.168.0.11/12/13
```

> (4)在每台机器上安装`docker engine`
>
> `小技巧`：要想让每个shell窗口一起执行同样的命令"查看-->撰写-->撰写窗口-->全部会话"

### 1.2 搭建Swarm集群

> (1)进入`manager`
>
> `提示`：`manager node`也可以作为`worker node`提供服务

```
docker swarm init --advertise-addr=192.168.0.11
```

`注意观察日志，拿到worker node加入manager node的信息`

```
docker swarm join --token SWMTKN-1-0a5ph4nehwdm9wzcmlbj2ckqqso38pkd238rprzwcoawabxtdq-arcpra6yzltedpafk3qyvv0y3 192.168.0.11:2377
```

> (2)进入两个`worker`

```
docker swarm join --token SWMTKN-1-0a5ph4nehwdm9wzcmlbj2ckqqso38pkd238rprzwcoawabxtdq-arcpra6yzltedpafk3qyvv0y3 192.168.0.11:2377
```

`日志打印`

```
This node joined a swarm as a worker.
```

> (3)进入到`manager node`查看集群状态

```bash
docker node ls
```

> (4)`node`类型的转换
>
> 可以将`worker`提升成`manager`，从而保证manager的高可用

```bash
docker node promote worker01-node
docker node promote worker02-node

#降级可以用demote
docker node demote worker01-node
```

### 1.3 在线的

> http://labs.play-with-docker.com

## 2 Swarm基本操作

### 2.1 Service

> (1)创建一个`tomcat`的`service`

```
docker service create --name my-tomcat tomcat
```

> (2)查看当前`swarm`的`service`

```
docker service ls
```

> (3)查看`service`的启动日志

```
docker service logs my-tomcat
```

> (4)查看`service`的详情

```
docker service inspect my-tomcat
```

> (5)查看`my-tomcat`运行在哪个`node`上

```
docker service ps my-tomcat
```

`日志`

```
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
u6o4mz4tj396        my-tomcat.1         tomcat:latest       worker01-node       Running             Running 3 minutes ago  
```

> (6)水平扩展`service`

```
docker service scale my-tomcat=3
docker service ls
docker service ps my-tomcat
```

`日志`：可以发现，其他`node`上都运行了一个`my-tomcat`的`service`

```
[root@manager-node ~]# docker service ps my-tomcat
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
u6o4mz4tj396        my-tomcat.1         tomcat:latest       worker01-node       Running             Running 8 minutes ago                        
v505wdu3fxqo        my-tomcat.2         tomcat:latest       manager-node        Running             Running 46 seconds ago                       
wpbsilp62sc0        my-tomcat.3         tomcat:latest       worker02-node       Running             Running 49 seconds ago  
```

此时到`worker01-node`上：`docker ps`，可以发现`container`的`name`和`service`名称不一样，这点要知道

```
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
bc4b9bb097b8        tomcat:latest       "catalina.sh run"   10 minutes ago      Up 10 minutes       8080/tcp            my-tomcat.1.u6o4mz4tj3969a1p3mquagxok
```

> (7)如果某个`node`上的`my-tomcat`挂掉了，这时候会自动扩展

```
[worker01-node]
docker rm -f containerid

[manager-node]
docker service ls
docker service ps my-tomcat
```

> (8)删除`service`

```
docker service rm my-tomcat
```

### 2.2 多机通信`overlay`网络[3.7的延续]

> `业务场景`：workpress+mysql实现个人博客搭建
>
> > <https://hub.docker.com/_/wordpress?tab=description>

#### 2.2.1 传统手动方式实现

#####  2.2.1.1 一台centos上，分别创建容器

```
01-创建mysql容器[创建完成等待一会，注意mysql的版本]
	docker run -d --name mysql -v v1:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=examplepass -e MYSQL_DATABASE=db_wordpress mysql:5.6
	
02-创建wordpress容器[将wordpress的80端口映射到centos的8080端口]
	docker run -d --name wordpress --link mysql -e WORDPRESS_DB_HOST=mysql:3306 -e WORDPRESS_DB_USER=root -e WORDPRESS_DB_PASSWORD=examplepass -e WORDPRESS_DB_NAME=db_wordpress -p 8080:80 wordpress
	
03-查看默认bridge的网络，可以发现两个容器都在其中
	docker network inspect bridge
	
04-访问测试
	win浏览器中输入：ip[centos]:8080，一直下一步
```

##### 2.2.1.2 使用docker compose创建

> `docker-compose`的方式还是在一台机器中，网络这块很清晰

```
01-创建wordpress-mysql文件夹
	mkdir -p /tmp/wordpress-mysql
	cd /tmp/wordpress-mysql
	
02-创建docker-compose.yml文件
```

`文件内容`

```yml
version: '3.1'

services:

  wordpress:
    image: wordpress
    restart: always
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    volumes:
      - wordpress:/var/www/html

  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:
```

```
03-根据docker-compose.yml文件创建service
	docker-compose up -d
	
04-访问测试
	win10浏览器ip[centos]:8080，一直下一步
	
05-值得关注的点是网络
	docker network ls
	docker network inspect wordpress-mysql_default
```

#### 2.2.2 Swarm中实现

> 还是`wordpress+mysql`的案例，在`docker swarm`集群中怎么玩呢？

> (1)创建一个`overlay`网络，用于`docker swarm`中多机通信

```
【manager-node】
docker network create -d overlay my-overlay-net

docker network ls[此时worker node查看不到]
```

> (2)创建`mysql`的`service`

```
【manager-node】
01-创建service
docker service create --name mysql --mount type=volume,source=v1,destination=/var/lib/mysql --env MYSQL_ROOT_PASSWORD=examplepass --env MYSQL_DATABASE=db_wordpress --network my-overlay-net mysql:5.6

02-查看service
	docker service ls
	docker service ps mysql
```

> (3)创建`wordpress`的`service`

```
01-创建service  [注意之所以下面可以通过mysql名字访问，也是因为有DNS解析]
docker service create --name wordpress --env WORDPRESS_DB_USER=root --env WORDPRESS_DB_PASSWORD=examplepass --env WORDPRESS_DB_HOST=mysql:3306 --env WORDPRESS_DB_NAME=db_wordpress -p 8080:80 --network my-overlay-net wordpress

02-查看service
	docker service ls
	docker service ps mysql
	
03-此时mysql和wordpress的service运行在哪个node上，这时候就能看到my-overlay-net的网络
```

> (4)测试

```
win浏览器访问ip[manager/worker01/worker02]:8080都能访问成功
```

> (5)查看`my-overlay-net`

```
docker network inspect my-overlay-net
```

> (6)为什么没有用`etcd`？`docker swarm`中有自己的分布式存储机制

##  3 Routing Mesh

### 3.1 Ingress

> 通过前面的案例我们发现，部署一个`wordpress`的`service`，映射到主机的8080端口，这时候通过`swarm`集群中的任意主机`ip:8080`都能成功访问，这是因为什么？
>
> `把问题简化`：`docker service create --name tomcat  -p 8080:8080 --network my-overlay-net tomcat`

> (1)记得使用一个自定义的`overlay`类型的网络

```
--network my-overlay-net
```

> (2)查看`service`情况

```
docker service ls
docker service ps tomcat
```

> (3)访问3台机器的`ip:8080`测试

```
发现都能够访问到tomcat的欢迎页
```

### 4.2 Internal

> 之前在实战`wordpress+mysql`的时候，发现`wordpress`中可以直接通过`mysql`名称访问
>
> 这样可以说明两点，第一是其中一定有`dns`解析，第二是两个`service`的`ip`是能够`ping`通的
>
> `思考`：不妨再创建一个service，也同样使用上述`tomcat`的`overlay`网络，然后来实验
>
> `docker service create --name whoami -p 8000:8000 --network my-overlay-net -d  jwilder/whoami`

> (1)查看`whoami`的情况

```
docker service ps whoami
```

> (2)在各自容器中互相`ping`一下彼此，也就是容器间的通信

```
#tomcat容器中ping whoami
docker exec -it 9d7d4c2b1b80 ping whoami
64 bytes from bogon (10.0.0.8): icmp_seq=1 ttl=64 time=0.050 ms
64 bytes from bogon (10.0.0.8): icmp_seq=2 ttl=64 time=0.080 ms


#whoami容器中ping tomcat
docker exec -it 5c4fe39e7f60 ping tomcat
64 bytes from bogon (10.0.0.18): icmp_seq=1 ttl=64 time=0.050 ms
64 bytes from bogon (10.0.0.18): icmp_seq=2 ttl=64 time=0.080 ms
```

> (3)将`whoami`进行扩容

```
docker service scale whoami=3
docker service ps whoami     #manager,worker01,worker02
```

> (4)此时再``ping whoami service`，并且访问`whoam`i服务

```
#ping
docker exec -it 9d7d4c2b1b80 ping whoami
64 bytes from bogon (10.0.0.8): icmp_seq=1 ttl=64 time=0.055 ms
64 bytes from bogon (10.0.0.8): icmp_seq=2 ttl=64 time=0.084 ms

#访问
docker exec -it 9d7d4c2b1b80 curl whoami:8000  [多访问几次]
I'm 09f4158c81ae
I'm aebc574dc990
I'm 7755bc7da921
```

`小结`：通过上述的实验可以发现什么？`whoami`服务对其他服务暴露的ip是不变的，但是通过`whoami`名称访问8000端口，确实访问到的是不同的`service`，就说明访问其实是像下面这张图。

也就是说`whoami service`对其他服务提供了一个统一的VIP入口，别的服务访问时会做负载均衡。

## 5 Stack

> docker stack deploy：https://docs.docker.com/engine/reference/commandline/stack_deploy/
>
> compose-file：https://docs.docker.com/compose/compose-file/
>
> 有没有发现上述部署`service`很麻烦？要是能够类似于`docker-compose.yml`文件那种方式一起管理该多少？这就要涉及到`docker swarm`中的`Stack`，我们直接通过前面的`wordpress+mysql`案例看看怎么使用咯。

> (1)新建service.yml文件

```yml
version: '3'

services:

  wordpress:
    image: wordpress
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    networks:
      - ol-net
    volumes:
      - wordpress:/var/www/html
    deploy:
      mode: replicated
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      update_config:
        parallelism: 1
        delay: 10s

  db:
    image: mysql:5.7
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql
    networks:
      - ol-net
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager

volumes:
  wordpress:
  db:

networks:
  ol-net:
    driver: overlay
```

> (2)根据`service.yml`创建`service`

```
docker statck deploy -c service.yml my-service
```

> (3)常见操作

```
01-查看stack具体信息
	docker stack ls
	NAME                SERVICES            ORCHESTRATOR
	my-service          2                   Swarm
	
02-查看具体的service
	docker stack services my-service
	
ID                  NAME                   MODE                REPLICAS            IMAGE               PORTS
icraimlesu61        my-service_db          global              1/1                 mysql:5.7           
iud2g140za5c        my-service_wordpress   replicated          3/3                 wordpress:latest    *:8080->80/tcp

03-查看某个service
	docker service inspect my-service-db
	
"Endpoint": {
            "Spec": {
                "Mode": "vip"
            },
            "VirtualIPs": [
                {
                    "NetworkID": "kz1reu3yxxpwp1lvnrraw0uq6",
                    "Addr": "10.0.1.5/24"
                }
            ]
        }

```

> (4)访问测试

win浏览器ip[manager,worker01,worker02]:8080