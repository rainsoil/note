#  [Docker 之网络](https://docs.docker.com/network/)篇

## 3.1   计算机网络模型



![10](http://files.luyanan.com//img/20200415113128.png)





![11](http://files.luyanan.com//img/20200415113520.png)



##  3.2 Linux网卡

### 3.2.1  查看网卡(网络接口)

- `ip link show`
- `ls /sys/class/net`
- `ip a`

### 3.2.2  网卡

#### 3.2.2.1 `ip a` 解读

- 状态：`UP`/`DOWN`/`UNKOWN`等 
- `link`/`ether`：`MAC`地址 
- `inet`：绑定的IP地址



#### 3.2.2.2  配置文件

在`linux` 中网卡对应的其实就是文件, 所以找到对应的网卡文件即可. 

比如 `cat /etc/sysconfig/network-scripts/ifcfg-eth0`

#### 3.2.2.3 给网卡添加ip地址

当然, 这块也可以直接修改`ifcfg.*` 文件, 但是我们通过命令添加试试

-  `ip addr add 192.168.0.100/24 dev eth0`
-  删除ip地址  `ip addr delete 192.168.0.100/24 dev eth0`



#### 3.2.2.4  网卡的启动和关闭

重启网卡

`service network restart` / `systemctl restart network`

启动/关闭某个网卡

`ifup/ifdown eth0 or ip link set eth0 up/down`



##  3.3  `Network Namespace`

在linux上,网络的隔离通过`network workspace` 来管理,不同的`Network Namespace`是相互隔离的, 

`ip netns list`: 可以查看机器上的`的network namespace`

```bash
ip netns list #查看
ip netns add ns1 #添加
ip netns delete ns1 #删除

```

### 3.3.1  ` namespace` 实战

1. 创建一个 `network namespace`

   ```bash
   ip netns add ns1
   ```

2. 查看该`namespace` 下网卡的情况

   ```bash
   ip netns exec ns1 ip a
   ```

3. 启动`ns1` 上的`lo` 网卡

   ```bash
   ip netns exec ns1 ifup lo
   or
   ip netns exec ns1 ip link set lo up
   ```

4. 再次查看,可以发现`state` 已经变成了`UNKOWN`

   ```bash
   ip netns exec ns1 ip a
   
   ```

5. 再次创建一个`network namespace`

   ![12](http://files.luyanan.com//img/20200415140658.png)

6. 此时想让两个`namespace` 网络连通起来

   `veth pair`: `Virtual Ethernet Pair`,是一个成对的端口,可以实现上述功能. 

   ![13](http://files.luyanan.com//img/20200415140919.png)

7. 创建一对`link`, 也就是接下来要通过`过veth pair` 连接的`link`

   ```bash
   ip link add veth-ns1 type veth peer name veth-ns2
   
   ```

8. 查看`link` 情况

   ```bash
   ip link
   ```

9. 将`veth-ns1`  加入到`ns1`,将`veth-ns2` 加入到`ns2`

   ```bash
   ip link set veth-ns1 netns ns1
   ip link set veth-ns2 netns ns2
   
   ```

10. 查看宿主机和`ns1`、`ns2`的`link` 情况

    ```bash
    ip link
    ip netns exec ns1 ip link
    ip netns exec ns2 ip link
    ```

11. 此时`veth-ns1` 和`veth-ns2` 还没有`ip`地址,显然通讯还缺少点条件

    ```bash
    ip netns exec ns1 ip addr add 192.168.0.11/24 dev veth-ns1
    ip netns exec ns2 ip addr add 192.168.0.12/24 dev veth-ns2
    
    ```

12. 再次查看,发现`state` 是`DOWN`,并且还是没有`ip`地址

    ```bash
    ip netns exec ns1 ip link
    ip netns exec ns2 ip link
    ```

13. 启动`veth-ns1`和`veth-ns2` 

     ```bash
    ip netns exec ns1 ip link set veth-ns1 up
    ip netns exec ns2 ip link set veth-ns2 up
     ```

14. 再次查看, 发现`state`是`UP`,同时有ip

     ```bash
    ip netns exec ns1 ip a
    ip netns exec ns2 ip a
     ```

15. 此时两次 `network namespace` 互相`ping` 一下,发现是可以`ping`通

    ```bash
    ip netns exec ns1 ping 192.168.0.12
    ip netns exec ns2 ping 192.168.0.11
    ```



### 3.3.2 `Contailer`的`NS`

按照上面的描述,实际上每一个`contailer`,都会有自己的`network  namespace`,并且是独立的,我们可以进入到容器中进行验证

1. 不妨创建两个`contailer` 看看? 

   ```bash
   docker run -d --name tomcat01 -p 8081:8080 tomcat
   docker run -d --name tomcat02 -p 8082:8080 tomcat

   ```
   
2. 进入到两个容器中, 分别查看ip

   ```bash
   docker exec -it tomcat01 ip a
   docker exec -it tomcat02 ip a
   
   ```

3. 互相`ping` 一下是可以`ping`通的

值得我们思考的是,此时`tomcat01`和`tomcat02` 属于两个`network namespace`, 是如何能够`ping`通的, 有些小伙伴可能会想, 不就跟上面的`namespace` 实战是一样的吗? 注意这里并没有`veth-pair` 技术



## 3.4 深入分析`contailer` 网络-`Bridge`

### 3.4.1 `docker0` 默认`bridge`

1. 查看`centos`的网络, `ip a` , 可以发现

   ![image-20200416204611664](http://files.luyanan.com//img/20200416204624.png)

2. 查看`tomcat01`的网络, `docker exec -it tomcat01 ip a`, 可以发现

   ```bash
   [root@bogon ~]# docker exec -it tomcat01 ip a
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
       inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
   7: eth0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
       link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
       inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
         valid_lft forever preferred_lft forever
   ```

3. 在`centos` 中`ping` 一下`tomcat01`的网络, 发现可以`ping` 通

   ```bash
   ping 172.17.0.2
   [root@bogon ~]# ping 172.17.0.2
   PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
   64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.120 ms
   64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.060 ms
   64 bytes from 172.17.0.2: icmp_seq=3 ttl=64 time=0.056 ms
   ```

4. 既然可以`ping` 通, 而且`centos` 和`tomcat01` 又属于不同的`netrwork namespace`,是怎么连接的? 

    很显然,跟之前的实战是一样的, 画个图

   

   ![14](http://files.luyanan.com//img/20200416205041.png)

5. 也就是说,在`tomcat01` 中有一个`eth0`和`centos`的`docker0` 中有一个`veth3`是成对的,类似于之前实战中的`veth-ns1`和`veth-ns2`,不妨再通过一个命令确认一下`brctl`

   ```bash
   安装一下：yum install bridge-utils
   brctl show
   
   ```

6. 那为什么`tomcat01` 和`tomcat02`是可以互通的呢? 不多说, 直接上图

    ![15](http://files.luyanan.com//img/20200416205339.png)

7. 这种网络连接方式我们称之为`bridge`,其实也可以通过命令查看`docker` 中的网络模式,`docker network ls`. `bridge` 也是`docker` 默认的网络模式.

8. 不妨检查一下`bridge`: `docker network inspect bridge`

   ```json
   {
       "Containers": {
           "6ad312b32f62b48935f3c95c58ae061df710bfebbd3d721b467507b9516eeb81": {
               "EndpointID": "aa9c612c79f867e874d0cae1aab45374373b61e9cdbe79925d07ae2e89a1cca0",
               "IPv4Address": "172.17.0.3/16",
               "IPv6Address": "",
               "MacAddress": "02:42:ac:11:00:03",
               "Name": "tomcat02"
           },
           "f49fc396d8e04f2b330163d91bb5d1482715202b4e2fd0c7f42833722787742a": {
               "EndpointID": "c5440b063e8fc0c9c44f3f61bf68f577283417eb23cfa9a361d37973d01a8ba5",
               "IPv4Address": "172.17.0.2/16",
               "IPv6Address": "",
               "MacAddress": "02:42:ac:11:00:02",
               "Name": "tomcat01"
           }
       }
   }
   ```

9. 在`tomcat01` 容器中是可以访问互联网的, 顺便把这张图画一下,`NAT`是通过`iptable` 实现

     

![16](http://files.luyanan.com//img/20200416210124.png)



### 3.4.2  创建自己的`network`

1. 创建自己的`network`, 类型为`bridge`

   ```bash
   docker network create tomcat-net
   or
   docker network create --subnet=172.18.0.0/24 tomcat-net
   
   ```

2. 查看已有的`network`: `docker network ls`

   ![image-20200416210313245](http://files.luyanan.com//img/20200416210316.png)

3. 查看`tomcat-net` 详细信息

   ```bash
   docker network inspect tomcat-net
   ```

4. 创建`tomcat`的容器, 并且指定使用`tomcat-net`

    ```bash
   docker run -d --name custom-net-tomcat --network tomcat-net tomcat
   
   ```

5. 查看`custom-net-tomcat` 的网络信息

    ```bash
   docker exec -it custom-net-tomcat ip a
   
   ```

6. 查看网卡信息

    ```bash
   ip a
   ```

7. 查看网卡接口

   ```bash
   brctl show
   bridge     name     bridge id     STP enabled     interfaces
   br-3012e3afd264     8000.02429780e75d     no     vethf223a4b
   docker0     8000.0242437b1bbd     no     veth3b72761
   veth9d8c470
   ```

8. 此时, 在`custom-net-tomcat` 容器中`ping` 一下`tomcat01`的`ip` 会如何? 发现无法`ping` 通

   ```bash
   docker exec -it custom-net-tomcat ping 172.17.0.2
   PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
   ^C
   --- 172.17.0.2 ping statistics ---
   4 packets transmitted, 0 received, 100% packet loss, time 3000ms
   
   ```

9. 此时如果`tomcat01` 容器能够连接到`tomcat-net` 上应该就可以 

   ```bash
   docker network connect tomcat-net tomcat01
   
   ```

10. 查看`tomcat-net` 网络,可以发现`tomcat01`  这个容器也在其中

11. 此时进入到`tomcat01` 或者`custom-net-tomcat`中, 不仅可以通过`ping` 通, 而且可以通过名字`ping`到, 这时候因为连接到了用户自定义的 `tomcat-net bridge` 上. 

     ```bash
    docker exec -it tomcat01 bash
    
     ```

    ```bash
    root@f49fc396d8e0:/usr/local/tomcat# ping 172.18.0.2
    PING 172.18.0.2 (172.18.0.2) 56(84) bytes of data.
    64 bytes from 172.18.0.2: icmp_seq=1 ttl=64 time=0.048 ms
    64 bytes from 172.18.0.2: icmp_seq=2 ttl=64 time=0.040 ms
    
    ```

    ```bash
    root@f49fc396d8e0:/usr/local/tomcat# ping custom-net-tomcat
    PING custom-net-tomcat (172.18.0.2) 56(84) bytes of data.
    64 bytes from custom-net-tomcat.tomcat-net (172.18.0.2): icmp_seq=1 ttl=64 time=0.030 ms
    64 bytes from custom-net-tomcat.tomcat-net (172.18.0.2): icmp_seq=2 ttl=64 time=0.264 ms
    ```

    但是`ping` `tomcat02` 是不通的

    ```bash
    root@f49fc396d8e0:/usr/local/tomcat# ping 172.17.0.3
    PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
    64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.045 ms
    64 bytes from 172.17.0.3: icmp_seq=2 ttl=64 time=0.066 ms
    ```

    ```bash
    root@f49fc396d8e0:/usr/local/tomcat# ping tomcat02
    PING tomcat02 (220.250.64.26) 56(84) bytes of data.
    
    ```



## 3.5  深入分析`Contailer` 网络-`Host` & `None`

### 3.5.1 `Host`

1.  创建一个`tomcat` 容器,并且指定网络为`host`

   ```bash
   docker run -d --name my-tomcat-host --network host tomcat
   
   ```

2. 查看`ip`地址

   ```bash
   docker exec -it my-tomcat-host ip a
   
   ```

   可以发现和`centos` 是一样的

3. 检查`host` 网络

   ```json
   {
       "Containers": {
           "e1f00d47db344b6688e99c0f5b393e232309fbe1a4d9c3fc3e1ce7c107f3312d": {
               "EndpointID": "f08456d9dca024cf6f911f8d32329ba2587ea89554c96b77c32698ace6998525",
               "IPv4Address": "",
               "IPv6Address": "",
               "MacAddress": "",
               "Name": "my-tomcat-host"
           }
       }
   }
   ```

###  3.5.2 `None`

1. 创建一个`tomcat` 容器, 并且指定网络为`none`

   ```bash
   docker run -d --name my-tomcat-none --network none tomcat
   
   ```

2. 查看`ip` 地址

   ```bash
   docker exec -it my-tomcat-none ip a
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
       inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
   ```

3. 检查`None` 网络

    ```json
   {
       "Containers": {
           "bb3f0db4fa76a25b5377da9c3bbf087ac7ef0de0a3f9c37a4ae959983d33105c": {
               "EndpointID": "26055c08c968f9d6d03d10b3b66dfea004c35f5d2bd4067a2306566973e92f9e",
               "IPv4Address": "",
               "IPv6Address": "",
               "MacAddress": "",
               "Name": "my-tomcat-none"
           }
       }
   }
   ```

   



## 3.6 端口映射以及折腾

### 3.6.1 端口映射

1. 创建一个`tomcat` 容器,名字为`port-tomcat`

   ```bash
   docker run -d --name port-tomcat tomcat
   
   ```

2. 思考一下要访问该`tomcat` 怎么办? 肯定是通过`ip:port`的方式

   ```bash
   docker exec -it port-tomcat bash
   curl localhost:8080
   ```

3. 那如果是在`centos7` 访问呢?

   ```bash
   docker exec -it port-tomcat ip a ---->得到其ip地址，比如172.17.0.4
   curl 172.17.0.4:8080
   ```

**小结:** 之所以能够访问成功,是因为`centos` 上的`docker0` 连接了`port-tomcat`的`network namespace`



4. 那如果还要在`centos7`通过`curl localhost`方式访问呢? 显然要将`port-tomcat` 的8080端口映射到`centos` 上. 

   ```bash
   docker rm -f port-tomcat
   docker run -d --name port-tomcat -p 8090:8080 tomcat
   curl localhost:8090
   ```

### 3.6.2  折腾

1. `centos7` 是运行在`win10` 上的虚拟机上,  如果想要在`win10` 上通过`ip:port` 方式访问呢? 

   ```text
   #此时需要centos和win网络在同一个网段，所以在Vagrantfile文件中
   #这种方式等同于桥接网络。也可以给该网络指定使用物理机哪一块网卡，比如
   #config.vm.network"public_network",:bridge=>'en1: Wi-Fi (AirPort)'
   config.vm.network"public_network"
   centos7: ip a --->192.168.8.118
   win10:浏览器访问 192.168.8.118:9080
   ```

2. 如果也想要把`centos7` 上的8090映射到`win10`的某个端口上呢? 然后浏览器访问`localhost:port`

   ```bash
   #此时需要将centos7上的端口和win10上的端口做映射
   config.vm.network"forwarded_port",guest:8098,host:8090
   #记得vagrant reload生效一下
   win10：浏览器访问 localhost：8098
   ```



### 3.6.3 画个图强化一下

![image-20200416213621301](http://files.luyanan.com//img/20200416213622.png)



## 3.7  多机之间的`contailer` 通信

在同一台`centos7` 机器上, 发现无论怎么折腾, 我们一定有办法让两个`contailer` 通信. 

那如果是在两台`centos7` 机器上的呢? 

![image-20200416213632513](http://files.luyanan.com//img/20200416213634.png)

1. 使得两边的`etho` 能够通信

2. 前提要确保两边的`ip` 地址不一样

3. 将一边的所有信息当成`etho` 传输给另外一端的信息

4. 具体通过`vxlan` 技术实现

    http://www.evoila.de/2015/11/06/what-is-vxlan-and-how-it-works

5. 处在`vxlan`的底层:`underlay`

   处在`xxlan`的上层:`overlay`

![image-20200416213643754](http://files.luyanan.com//img/20200416213645.png)