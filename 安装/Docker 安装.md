#  Centos 安装

Docker从1.13版本之后采用时间线的方式作为版本号，分为社区版CE和企业版EE。

社区版是免费提供给个人开发者和小型团体使用的，企业版会提供额外的收费服务，比如经过官方测试认证过的基础设施、容器、插件等。

社区版按照stable和edge两种方式发布，每个季度更新stable版本，如17.06，17.09；每个月份更新edge版本，如17.09，17.10。

##  安装docker

### 安装Docker

1、Docker 要求 CentOS 系统的内核版本高于 3.10 ，查看本页面的前提条件来验证你的CentOS 版本是否支持 Docker 。

通过 **uname -r** 命令查看你当前的内核版本

```
 $ uname -r
```

2、使用 `root` 权限登录 Centos。确保 yum 包更新到最

```
$ sudo yum update
```

3、卸载旧版本(如果安装过旧版本的话)

```bash
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

4、安装需要的软件包， yum-util 提供yum-config-manager功能，另外两个是devicemapper驱动依赖的

```bash
$ sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

5、设置yum源

```
$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

 ![](http://files.luyanan.com//img/20190923144207.png)

6、可以查看所有仓库中所有docker版本，并选择特定版本安装

```
$ yum list docker-ce --showduplicates | sort -r
```

![](http://files.luyanan.com//img/20190923144432.png)

7、安装docker

```
$ sudo yum install -y docker-ce docker-ce-cli containerd.io
$ sudo yum install <FQPN>  # 例如：sudo yum install docker-ce-17.12.0.ce
```

 ![](http://files.luyanan.com//img/20190923144319.png)

8、启动并加入开机启动

```
$ sudo systemctl start docker
$ sudo systemctl enable docker
```

9、验证安装是否成功(有client和service两部分表示docker安装启动都成功了)

```
$ docker version
```

![](http://files.luyanan.com//img/20190923151257.png)

 

###  问题

1、因为之前已经安装过旧版本的docker，在安装的时候报错如下：

```
Transaction check error:
  file /usr/bin/docker from install of docker-ce-17.12.0.ce-1.el7.centos.x86_64 conflicts with file from package docker-common-2:1.12.6-68.gitec8512b.el7.centos.x86_64
  file /usr/bin/docker-containerd from install of docker-ce-17.12.0.ce-1.el7.centos.x86_64 conflicts with file from package docker-common-2:1.12.6-68.gitec8512b.el7.centos.x86_64
  file /usr/bin/docker-containerd-shim from install of docker-ce-17.12.0.ce-1.el7.centos.x86_64 conflicts with file from package docker-common-2:1.12.6-68.gitec8512b.el7.centos.x86_64
  file /usr/bin/dockerd from install of docker-ce-17.12.0.ce-1.el7.centos.x86_64 conflicts with file from package docker-common-2:1.12.6-68.gitec8512b.el7.centos.x86_64
```

2、卸载旧版本的包

```
$ sudo yum erase docker-common-2:1.12.6-68.gitec8512b.el7.centos.x86_64
```

![img](https://images2017.cnblogs.com/blog/1107037/201801/1107037-20180128103145287-536100760.png)

3、再次安装docker

```
$ sudo yum install docker-ce
```

## 安装docker-compose

安装docker-compose相对比较简单，可以直接去[https://github.com/docker/com...](https://github.com/docker/compose/releases) 下载然后选择相应的版本，或者直接执行如下命令安装，安装完后docker-compose会被安装到/usr/local/bin目录下

```
 curl -L https://github.com/docker/compose/releases/download/1.24.0-rc1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

### 设置docker-compose可执行

```
sudo chmod +x /usr/local/bin/docker-compose 
```

### 查看docker-compose是否安装成功

```
docker-compose --version 
```