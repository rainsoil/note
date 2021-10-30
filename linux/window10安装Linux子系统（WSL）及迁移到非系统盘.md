# window10安装Linux子系统（WSL）及迁移到非系统盘

## 痛点：
在电脑上想要使用linux又想使用windows系统只能安装双系统，因为虚拟机的性能差且使用麻烦，双系统使用起来倒是也还行，ubuntu的桌面系统现在用起来也可以，但是来回切换比较麻烦，故想尝试一下windows自带的这个WSL功能使用起来如何。
记录一下安装过程。

## Linux子系统
WSL（Windows Subsystem for Linux）详细介绍见官网：https://docs.microsoft.com/en-au/windows/wsl/about

安装
安装过程很简单，也不占什么空间：

1.打开控制面板中的程序和功能，打开启用或关闭Windows功能

![img](https://img-blog.csdnimg.cn/20191208124007253.png)

2 向下拉 勾选适用于Linux的Windows子系统，确定

![img](https://img-blog.csdnimg.cn/20191208124124472.png)

3 打开win10带的Microsoft Store 搜索 LINUX

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/2019120812415755.png)

4 选择在Windos上运行Linux,推荐安装ubuntu或debian，其他的都没见过，根据需求自行选择就行

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20191208124303915.png)

5 点击获取安装

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20191208124325318.png)

6 启动即可，初次启动会要求键入用户名和密码，根据需求输入即可，这样就算是安装完成

## 使用
1 右键开始图标打开power shell

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20191208124400308.png)

2 输了bash即可启动, 可以使用 wsl -l查询子系统安装列表

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20191208124452334.png)

3 子系统默认安装在系统系统盘，其他盘在子系统中的mnt文件夹中

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20191208124534741.png)

4 tips:你也可以在指定路径的文件夹中shift+右键打开Linux shell

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20191208124623690.png)

5 字体大小啥的，右键窗口上端进入 默认值进行更改

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20191208124651439.png)

## 迁移到非系统盘
一般大家在安装系统的时候都喜欢把系统盘能的比较小，如果子系统要安装很多东西，比如我要安装docker，会有很多镜像，很占空间在系统盘不是很合适，于是有了迁出系统盘的需求，可以使用LxRunOffline进行迁移

1 下载 LxRunOffline ，https://github.com/DDoSolitary/LxRunOffline/releases，选择最新版下载即可

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20191208124720435.png)

2 解压后，在软件目录打开power shell ，上面交的方法shift + 右键

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20191208124741715.png)

3 使用`LxRunOffline.exe list`命令查看你可以使用子系统名称

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20191208124802141.png)

4 使用 lxrunoffline move 进行迁移 ， -n 指定你要迁移的系统名 ，-d 指定你新系统的迁移路径

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20191208124816278.png)

5 迁移过程会出现WARNING 不用管， 等待一段时间结束就算迁移完成了

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20191208124831623.png)

6 使用LxRunOffline.exe get-dir 查询系统目录，可见已经更改成功

![å¨è¿éæå¥å¾çæè¿°](https://img-blog.csdnimg.cn/20191208124907360.png)

备注：
还有个下载系统包的安装方法，由于下载速度感人没有尝试，方法官网有，链接如下：
https://docs.microsoft.com/en-au/windows/wsl/install-on-server
————————————————
