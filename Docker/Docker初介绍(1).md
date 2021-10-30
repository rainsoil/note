# Docker初介绍

##  1. What is Docker

### 1.1 [官网首页](https://www.docker.com/)

> Modernize your applications, accelerate innovation Securely build, share and run modern applications anywhere

### 1.2 [Docs](https://docs.docker.com/get-started/)

> Docker is a platform for developers and sysadmins to develop, deploy, and run applications with containers. The use of Linux containers to deploy applications is called containerization. Containers are not new, but their use for easily deploying applications is.

### 1.3 由来(演进过程)

刚开始我们部署一个`jar`的时候

![image-20200413114147162](http://files.luyanan.com//img/20200413114211.png)

问题: 成本高, 部署满,浪费资源,硬件限制,不利于迁移扩展. 

接下来进入虚拟化时代



![01](http://files.luyanan.com//img/20200413122313.png)

优点: 相对利用资源, 相对容易扩展. 

缺点: 容器太重了, 一上来占用较多的物理资源,移植性差,资源利用率低. 



**容器时代**

![02](http://files.luyanan.com//img/20200413122439.png)



### 1.4 再次理解Docker

> Docker is a platform for developers and sysadmins to develop, deploy, and run applications with containers. The use of Linux containers to deploy applications is called containerization. Containers are not new, but their use for easily deploying applications is.

发现还是比较容易理解的, 但是这一句`Containers are not new` ,也就是容器化技术在很早之前就出现了,比较常见的容器化技术有`OpenVZ`、`LXC`、`RKT` 等. 

###  1.5 Docker 的优势和应用场景

> http://www.docker.com/  ---> `Solutions`

1. 有助于`Microservices`的落地和部署
2. 充分利用物理机资源, 同时能够整合服务器资源
3. 提高开发效率,测试效率,部署效率,有利于`DevOps`的落地
4. 云原生落地, 应用能够更好的迁移. 





## 2. 容器(`Container`)和镜像(`image`)

### 2.1 `image`

> A Docker container image is a lightweight, standalone, executable package of software that includes everything needed to run an application: code, runtime, system tools, system libraries and settings.

### 2.2 `container`

Why is docker?->[What is a container](https://www.docker.com/resources/what-container)

> A container is a standard unit of software that packages up code and all its dependencies so the application runs quickly and reliably from one computing environment to another.

###  2.3 ` Relation between image and container`

> Container images become containers at runtime and in the case of Docker containers- images become containers when they run on Docker Engine



### 2.4 ` View from Docs`

从帮助文档的角度看: 

[docker官网](http://www.docker.com/)->Resources->Docs->Get started->Get started with Docker->Orientation->Images and containers

> A container is launched by running an image. An image is an executable package that includes everything needed to run an application--the code, a runtime, libraries, environment variables, and configuration files. A container is a runtime instance of an image--what the image becomes in memory when executed (that is, an image with state, or a user process). You can see a list of your running containers with the command, docker ps, just as you would in Linux.



## 3. Containers and virtual machines

[docker官网](www.docker.com)->Resources->Docs->Get started->Get started with Docker- >Orientation->Containers and virtual machines

> A container runs natively on Linux and shares the kernel of the host machine with other containers. It runs a discrete process, taking no more memory than any other executable, making it lightweight.
>
>  By contrast, a virtual machine (VM) runs a full-blown “guest” operating system with virtual access to host resources through a hypervisor. In general, VMs provide an environment with more resources than most applications need.



![image-20200413145116672](http://files.luyanan.com//img/20200413145117.png)



##  4. `Docker Engine and Architecture`

https://docs.docker.com/engine/docker-overview/

###  4.1 Docker Engine

Docker Engine is a client-server application with these major components:

- A server which is a type of long-running program called a daemon process (the dockerd command). 
- A REST API which specifies interfaces that programs can use to talk to the daemon and instruct it what to do.
-  A command line interface (CLI)client (the docker command).


![04](http://files.luyanan.com//img/20200413145258.png) 

### 4.2 Docker Architecture

> Docker uses a client-server architecture. The Docker client talks to the Docker daemon, which does the heavy lifting of building, running, and distributing your Docker containers. The Docker client and daemon can run on the same system, or you can connect a Docker client to a remote Docker daemon. The Docker client and daemon communicate using a REST API, over UNIX sockets or a network interface.



![05](http://files.luyanan.com//img/20200413145357.png)