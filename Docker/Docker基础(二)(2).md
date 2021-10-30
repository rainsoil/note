#  Docker基础(二)

## 2.1 继续探讨`Image`

说白了.`image` 就是由一层一层的`layer`组成的. 



### 2.1.1 官方`image`

https://github.com/docker-library

**mysql**

https://github.com/docker-library/tomcat/blob/master/8.5/jdk8/openjdk/Dockerfile

### 2.1.2  `Dockerfile`

不妨我们也来制作一个自己的`image`镜像, 顺便来学习一下`Dockerfile` 文件中常用的语法. 

#### 2.1.2.1  `FROM`

指定基础的镜像,比如

```dockerfile
FROM ubuntu:14.04
```

#### 2.1.2.2 `RUN`

在镜像内部执行一些命令,比如安装软件、配置环境等 换行可以使用 ""

```dockerfile
RUN groupadd -r mysql && useradd -r -g mysql mysql
```

#### 2.1.2.3  `ENV`

设置变量的值,`ENV MYSQL_MAJOR 5.7`,通过 `过docker run --e key=value` 修改, 后面可以直接使用`${MYSQL_MAJOR}`

```dockerfile
ENV MYSQL_MAJOR 5.7
```

#### 2.1.2.4  `LABEL`

设置镜像标签

```dockerfile
LABEL email="luyanan0718@163.com"
LABEL name="luyanan"
```

#### 2.1.2.5  `VOLUME`

指定数据的挂载目录

```dockerfile
VOLUME /var/lib/mysql

```

####  2.1.2.6   `COPY`

将主机内的文件复制到镜像内,如果目录不存在,会自动创建所需要的目录, 注意只是复制, 不会提取和解压.

```dockerfile
COPY docker-entrypoint.sh /usr/local/bin/
```

#### 2.1.2.7 `ADD`

将主机的文件复制到镜像中,和`COPY` 类似,只是`ADD` 会对压缩文件提取和解压.

#### 2.1.2.8 `WORKDIR`

指定镜像的工作目录, 之后的命令都是基于此目录工作,如不存在则创建. 

```java
WORKDIR /usr/local
WORKDIR tomcat
RUN touch test.txt
```

> 会在`在/usr/local/tomcat` 下创建`test.txt` 文件

```dockerfile
WORKDIR /root
ADD app.yml test/
```

> 会在`/root/test` 下多出一个`app.yml` 文件

#### 2.1.2.9 `CMD`

容器启动的时候会默认执行的命令, 若有多个`CMD` 命令,则最后一个生效.

```dockerfile
CMD ["mysqld"]
或
CMD mysqld
```

#### 2.1.2.10 `ENTRYPOINT`

跟`CMD` 的使用类似

```dockerfile
ENTRYPOINT ["docker-entrypoint.sh"]
```

跟`CMD`的不同

`docker run` 执行时, 会覆盖`CMD`的命令, 而`ENTRYPOINT` 不会

#### 2.1.2.11 `EXPOSE`

指定镜像要暴漏的端口， 启动镜像时,可以使用`-p`  将该端口映射给宿主机

```dockerfile
EXPOSE 3306
```

### 2.1.3 `Dockerfile` 实战`Spring Boot` 项目

####  2.1.3.1 创建一个`Spring Boot` 项目

#### 1.2.3.2 写一个controller

```java
@RestController
public class DockerController {

    @GetMapping("/dockerfile")

    @ResponseBody
    String dockerfile() {
    return "hello docker" ; 
   }
}
```

#### 2.1.3.3  打包

`mvn clean package` 打成一个`jar` 包,在`target` 下找到`dockerfile-demo-0.0.1-SNAPSHOT.jar`

####  2.1.3.4  新建目录

在`docker` 环境中创建一个`first-dockerfile` 

#### 2.1.3.5  上传

上传 `dockerfile-demo-0.0.1-SNAPSHOT.jar` 到该目录下, 并且在此目录创建`Dockerfile`

#### 2.1.3.6  编写`Dockerfile`

```dockerfile
FROM openjdk:8
MAINTAINER luyanan0718@163.com
LABEL name="dockerfile-demo" version="1.0" author="luyanan"
COPY dockerfile-demo-0.0.1-SNAPSHOT.jar dockerfile-image.jar
CMD ["java","-jar","dockerfile-image.jar"]
```



#### 2.1.3.7 构建

基于`Dockerfile` 构建镜像

```bash
docker build -t test-docker-image .
```

#### 2.1.3.8 基于`image` 创建`contailer`

```bash
docker run -d --name user01 -p 6666:8080 test-docker-image
```

#### 2.1.3.9 查看启动日志

```bash
docker logs user01
```

#### 2.1.3.10  访问测试

```bash
宿主机上访问curl localhost:6666/dockerfile
```

返回 `hello docker`



### 2.1.4 镜像仓库

####   2.1.4.1 [docker hub](hub.docker.com)

1. 我们在机器上登陆

   ```bash
   docker login
   
   ```

2. 输入用户名和密码

3. `docker push luyanan0718/test-docker-image`

   注意镜像名称和`docker id` 一致, 不然`push`不成功

4. 给`image` 重命名, 并删除掉原来的

   ```bash
   docker tag test-docker-image luyanan0718/test-docker-image
   docker rmi -f test-docker-image
   
   ```

5. 再次推送,刷新 `hub.docker.com` 后台,发现成功. 

6. 别人下载,并且运行

   ```bash
   docker pull luyanan0718/test-docker-image
   docker run -d --name user01 -p 6661:8080 luyanan0718/test-docker-image
   ```



#### 2.1.4.2  阿里云`docker hub`

[阿里云仓库](https://cr.console.aliyun.com/cn-hangzhou/instances/repositories)

[参考手册](https://cr.console.aliyun.com/repository/cn-hangzhou/dreamit/image-repo/details)

1.  登陆到阿里云仓库

    ```bash
    sudo docker login --username=luyanan0718@163.com registry.cnhangzhou.aliyuncs.com
    ```

    

2. 输入密码

3. 创建命名空间

4. 给`image`打`tag`

   ```bash
   sudo docker tag [ImageId] registry.cn-hangzhou.aliyuncs.com/luyanan0718/testdocker-image:v1.0
   ```

5. 推送镜像到`docker` 阿里云仓库

   ```bash
   sudo docker push registry.cn-hangzhou.aliyuncs.com/luyanan0718/test-dockerimage:v1.0
   
   ```

6. 别人下载, 并且运行

   ```bash
   docker pull registry.cn-hangzhou.aliyuncs.com/luyanan0718/test-dockerimage:v1.0
   docker run -d --name user01 -p 6661:8080 registry.cnhangzhou.aliyuncs.com/luyanan0718/test-docker-image:v1.0
   ```

   

#### 2.1.4.3  搭建自己的Docker Harbor

1. 访问github上的`harbor` 项目

   https://github.com/goharbor/harbor

2. 下班版本,比如1.7.1 

   https://github.com/goharbor/harbor/releases

3. 找一台安装了 `docker-compose`的服务器, 上传并解压

   ```bash
   tar -zxvf xxx.tar.gz
   ```

4. 进入到 `harbor` 目录

   需要`harbor.cfg` 文件,主要是ip地址修改为当前机器的ip地址

   同时也可以看到`Harbor`的密码,默认是 `Harbor12345`

5. 安装`Harbor`,需要一些时间

   ```bash
   sh install.sh
   ```

6. 浏览器访问,输入ip地址, 输入用户名和密码即可. 



### 2.1.5 image常见操作

1. 查看本地的image 列表

   ```bash
   docker image
   docker image ls
   ```

2. 获取远端镜像

   ```baash
   dokcer pull
   ```

3. 删除镜像[注意此镜像如果正在使用,或者有关联的镜像,则需要先处理完]

   ```bash
   docker image rm imageid
   docker rmi -f imageid
   docker rmi -f $(docker image ls) 删除所有镜像
   ```

4. 运行镜像

    ```bash
   docker run image
   ```

5. 发布镜像

    ```bash
   docker push
   ```



## 2.2 深入探讨`Contailer`

> 既然`contailer` 是由`image` 运行起来的,那么是否可以理解为`contailer` 与`image`有某种关系呢?



![image-20200414220020523](http://files.luyanan.com//img/20200414220047.png)

理解: 其实可以理解为`contailer` 只是基于`image` 之后的`layer`  而已, 也就是可以通过`docker run image` 创建出一个`contailer`

### 2.2.1 `contailer` 到`image`

>  既然`contailer`是基于`image` 之上的,想想是否能够基于一个`contailer` 反推出一个`image` 呢？ 
>
> 肯定是可以的,比如通过`docker run`运行一个`contailer`出来,这时候对`contailer`  对一些修改,然后再生成一个`image`, 这时候`image`的由来就不仅仅只能通过`Dockerfile`了. 

**实验:**

1. 拉取一个`centos image`

   ```bash
   docker pull centos
   ```

2. 根据`centos`的镜像创建出一个`contailer`

   ```bash
   docker run -d -it --name my-centos centos
   ```

3. 进入`my-centos`容器中

    ```bash
   docker exec -it my-centos bash
   ```

4. 输入`vim` 命令 

    ```bash
   bash: vim: command not found
   ```
   
5. 我们要做的是, 对该`centos`  进行修改,也就是安装一下`vim` 命令,然后将其生成一个新的`centos`

6. 在`centos` 的`contailer` 中安装`vim`

     ```bash
    yum install -y vim
    ```

7. 退出容器,将其生成一个新的`centos`, 名称为`vim-centos-image`

     ```bash
    docker commit my-centos vim-centos-image
    ```

8. 查看镜像列表,并且基于`vim-centos-image` 创建新的容器

    ```bash
    docker run -d -it --name my-vim-centos vim-centos-image
    ```

9. 进入到`my-vim-centos` 容器中, 检查`vim`  命令是否存在. 

     ```bash
    docker exec -it my-vim-centos bash
    vim
    ```

    

结论: 可以通过`docker commit` 命令基于一个`contailer` 重新生成一个`iimage`, 但是一般得到`image` 方式不建议这样做,不然`image` 怎么来的就全然不知了. 



### 2.2.2 `contailer` 资源限制

如果不对`container` 的资源做限制,他就会无限制的使用物理机的资源, 这样显然是不合适的,

查看资源情况: `docker status`

#### 2.2.2.1  内存限制

```bash
--memory Memory limit
如果不设置 --memory-swap，其大小和memory一样
docker run -d --memory 100M --name tomcat1 tomcat

```

#### 2.2.2.2 CPU限制

```bash
--cpu-shares 权重
docker run -d --cpu-shares 10 --name tomcat2 tomcat

```

#### 2.2.2.3  图形化资源监控

https://github.com/weaveworks/scope

```bash
sudo curl -L git.io/scope -o /usr/local/bin/scope
sudo chmod a+x /usr/local/bin/scope
scope launch 39.100.39.63

```

```bash
# 停止scope
scope stop
# 同时监控两台机器，在两台机器中分别执行如下命令
scope launch ip1 ip2
```



### 2.2.3  `contailer` 常见操作

1.  根据镜像创建容器

    ```bash
   docker run -d --name -p 9090:8080 my-tomcat tomcat
   
   ```

2. 查看运行中的`contailer`

   ```bash
   docker ps
   ```

3. 查看所有的`contailer`(包含退出的)

   ```bash
   docker ps -a
   ```

4. 删除`contailer`

    ```bash
   docker rm containerid
   docker rm -f $(docker ps -a) 删除所有container
   ```

    

5. 根据`contailer` 生成`image`

    `docker  commit`

6. 进入到一个`contailer` 中

    ```bash
   docker exec -it contailer bash
   ```

7. 查看某个`contailer`的日志

    ```bash
   docker logs contailer
   ```

8. 查看容器资源使用情况

   ```bash
   docker status
   ```

9. 查看容器详情信息

    ```bash
   docker inspect container
   
   ```

10. 停止/启动容器

    ```bash
    docker stop/start contailer
    ```



##  2.3  底层技术支持

`contailer` 是一种轻量级的虚拟化技术, 不用模拟硬件创建虚拟机

`Docker` 是基于`Linux Kernel`的`Namespace`、`CGroups`、`UnionFileSystem`等技术封装成的一种自 定义容器格式，从而提供一套虚拟运行环境。

`Namespace`：用来做隔离的，比如`pid`[进程]、`net`[网络]、`mnt`[挂载点]等

`CGroups`: `Controller Groups`用来做资源限制，比如内存和`CPU`等

`Union file systems`：用来做`image`和`container`分层