# Window10 利用WSL2 安装Linux子系统以及docker

# 前言

windows10目前推出了WSL2，相对于WSL采用API转换的方式， WSL2 则完全不同，win10 开始内置了一个轻量级虚拟机，经过不断的优化，这个虚拟机实现了与 windows 的高度集成，实现了虚拟机的高性能运行，WSL2 便是运行在虚拟机上的一个完整的 linux 内核。因此WSL2给了在windows更接近原生linux的体验，同时wsl2 的开启速度有了非常明显的提升，几乎不需要再等待。本文探讨在win10专业版上利用WSL2安装docker的2种方式。

# 操作实践

## 1.开启安装windows10的WSL2功能

- 更新windows10系统

要升级 windows 系统到 win10 v2004 的**内部版本 19041** 或更高版本

升级 Windows 可以使用官方的更新助手，非常方便，地址：https://www.microsoft.com/zh-cn/software-download/windows10，在更新过程中，系统可能或多次重启。

![image-20200614181600789](https://360linux.oss-cn-hangzhou.aliyuncs.com/img/image-20200614181600789.png)

- 打开系统虚拟机平台

系统更新并重启后，我们就可以开始 wsl 的升级了

首先，需要打开“系统虚拟机平台”功能，在“控制面板\所有控制面板项\程序和功能”中选择“启用或者关闭Windows功能”，勾选对应选项即可：

![img](https://360linux.oss-cn-hangzhou.aliyuncs.com/img/43ec810de4bd7636e04b415a81c8aea9.png)

也可以通过在管理员权限下的 cmd 或 PowerShell 中执行：

```
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

- 下载 wsl2 需要使用的 linux 内核

在 https://docs.microsoft.com/zh-cn/windows/wsl/wsl2-kernel 页面点击下载 linux 内核更新包，下载完点击安装

- 启用"适用于 Linux 的 Windows 子系统"这个功能

启用"适用于 Linux 的 Windows 子系统"这个功能，然后才能在 Windows 上安装 Linux 发行版，如果之前使用过旧的wsl，此功能应该开启过。以管理员身份打开 PowerShell 运行如下所示的命令：

```
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

也可以在“控制面板\所有控制面板项\程序和功能”中选择“启用或者关闭Windows功能”，勾选对应选项即可。

- **重启系统**并设置WSL 2 设置为默认版本

```
# wsl命令可以设置单独某个具体wsl的linux版本为1版本但是2版本，wsl2速度较于旧版wsl快了很多，有了高铁还蹬啥自行车。
wsl --set-default-version 2
```

查看是不是WSL2，

```
wsl -l -v
```

## 2.安装配置 Linux 发行版

选择实用比较多的ubuntu版本，其他版本未测试能否安装成功docker。

- 打开 Microsoft Store，搜索 Terminal，安装 Windows Terminal，用于后面和 WSL 子系统交互。

![img](https://360linux.oss-cn-hangzhou.aliyuncs.com/img/641.png)

- 搜索 Ubuntu，选择安装。

![image-20200614184849150](https://360linux.oss-cn-hangzhou.aliyuncs.com/img/image-20200614184849150.png)

安装完成后，第一次打开 Ubuntu 的时候，将打开一个控制台窗口，会等待几分钟来进行配置，启动完成后为 Ubuntu 创建一个用户和密码（***如果第一次启动ubuntu失败，可以重启windows10系统再次试下***）。

使用 Windows Terminal 来操作 Ubuntu 系统了，在 Windows Terminal 中选择 Ubuntu 发行版就可以跳转到 Ubuntu 终端中，使用上面我们配置的用户名和密码登录即可：

![image-20200614185133006](https://360linux.oss-cn-hangzhou.aliyuncs.com/img/image-20200614185133006.png)

由于默认情况下我们不知道 root 用户的密码，所以如果我们想要使用 root 用户的话可以使用 passwd 命令为 root 用户设置一个新的密码，同时为了避免sudo切换root是需要输入密码，把自己配置的用户名加到sudo免密中，命令如下：

```
 # 替换leap为自己单独配置的用户名
 sudo echo "leap ALL=(ALL:ALL) NOPASSWD: ALL" >>/etc/sudoers 
```

- 更换ubuntu的apt安装源

默认的安装源相对国内很慢，我们更换源到阿里云，登录到ubuntu到操作如下：

```
cp /etc/apt/sources.list /etc/apt/sources.list.bak

echo "deb http://mirrors.aliyun.com/ubuntu/ focal main restricted
deb http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted
deb http://mirrors.aliyun.com/ubuntu/ focal universe
deb http://mirrors.aliyun.com/ubuntu/ focal-updates universe
deb http://mirrors.aliyun.com/ubuntu/ focal multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-updates multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-security main restricted
deb http://mirrors.aliyun.com/ubuntu/ focal-security universe
deb http://mirrors.aliyun.com/ubuntu/ focal-security multiverse">/etc/apt/sources.list
```

执行更新：

```
apt update && apt upgrade -y
```

## 3.安装docker

### 设置仓库

更新 apt 包索引。

```
$ sudo apt-get update
```

安装 apt 依赖包，用于通过HTTPS来获取仓库:

```bash
$ sudo apt-get install \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg-agent \
  software-properties-common
```



安装最新版本的 Docker Engine-Community 和 containerd ，或者转到下一步安装特定版本：

```
$ sudo apt-get install docker-ce docker-ce-cli containerd.io
```

要安装特定版本的 Docker Engine-Community，请在仓库中列出可用版本，然后选择一种安装。列出您的仓库中可用的版本：

```bash
$ apt-cache madison docker-ce

 docker-ce **|** 5:18.09.1~3-0~ubuntu-xenial **|** https:**//**mirrors.ustc.edu.cn**/**docker-ce**/**linux**/**ubuntu  xenial**/**stable amd64 Packages
 docker-ce **|** 5:18.09.0~3-0~ubuntu-xenial **|** https:**//**mirrors.ustc.edu.cn**/**docker-ce**/**linux**/**ubuntu  xenial**/**stable amd64 Packages
 docker-ce **|** 18.06.1~ce~3-0~ubuntu    **|** https:**//**mirrors.ustc.edu.cn**/**docker-ce**/**linux**/**ubuntu  xenial**/**stable amd64 Packages
 docker-ce **|** 18.06.0~ce~3-0~ubuntu    **|** https:**//**mirrors.ustc.edu.cn**/**docker-ce**/**linux**/**ubuntu  xenial**/**stable amd64 Packages
 ...
```

启动

```bash
service docker start 
```

测试 Docker 是否安装成功，输入以下指令，打印出以下信息则安装成功:

## 手动安装

### 卸载旧版本

Docker 的旧版本被称为 docker，docker.io 或 docker-engine 。如果已安装，请卸载它们：

```
$ sudo apt-get remove docker docker-engine docker.io containerd runc
```

当前称为 Docker Engine-Community 软件包 docker-ce 。

安装 Docker Engine-Community，以下介绍两种方式。

### 使用 Docker 仓库进行安装

在新主机上首次安装 Docker Engine-Community 之前，需要设置 Docker 仓库。之后，您可以从仓库安装和更新 Docker 。

### 设置仓库

更新 apt 包索引。

```
$ sudo apt-get update
```

安装 apt 依赖包，用于通过HTTPS来获取仓库:

```bash
$ sudo apt-get install \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg-agent \
  software-properties-common
```



添加 Docker 的官方 GPG 密钥：

```
$ curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
```

9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88 通过搜索指纹的后8个字符，验证您现在是否拥有带有指纹的密钥。

```bash
$ sudo apt-key fingerprint 0EBFCD88
  
pub  rsa4096 2017-02-22 **[**SCEA**]**
   9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid      **[** unknown**]** Docker Release **(**CE deb**)** **<**docker**@**docker.com**>**
sub  rsa4096 2017-02-22 **[**S**]**
```



使用以下指令设置稳定版仓库

```bash
$ sudo add-apt-repository \
  "deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/ \
 $(lsb_release -cs) \
 stable"
```



### 安装 Docker Engine-Community

更新 apt 包索引。

```
$ sudo apt-get update
```

安装最新版本的 Docker Engine-Community 和 containerd ，或者转到下一步安装特定版本：

```
$ sudo apt-get install docker-ce docker-ce-cli containerd.io
```

要安装特定版本的 Docker Engine-Community，请在仓库中列出可用版本，然后选择一种安装。列出您的仓库中可用的版本：

```bash
$ apt-cache madison docker-ce

  docker-ce | 5:18.09.1~3-0~ubuntu-xenial | https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu  xenial/stable amd64 Packages
  docker-ce | 5:18.09.0~3-0~ubuntu-xenial | https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu  xenial/stable amd64 Packages
  docker-ce | 18.06.1~ce~3-0~ubuntu       | https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu  xenial/stable amd64 Packages
  docker-ce | 18.06.0~ce~3-0~ubuntu       | https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu  xenial/stable amd64 Packages
  ...
```



使用第二列中的版本字符串安装特定版本，例如 5:18.09.1~3-0~ubuntu-xenial。

```
$ sudo apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io
```

测试 Docker 是否安装成功，输入以下指令，打印出以下信息则安装成功:

```bash
$ sudo docker run hello-world

Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
1b930d010525: Pull complete                                                                                                                                  Digest: sha256:c3b4ada4687bbaa170745b3e4dd8ac3f194ca95b2d0518b417fb47e5879d9b5f
Status: Downloaded newer image for hello-world:latest


Hello from Docker!
This message shows that your installation appears to be working correctly.


To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.


To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash


Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/


For more examples and ideas, visit:
 https://docs.docker.com/get-started/
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