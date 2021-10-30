#  window10下子系统ubuntu安装Docker

准备工作
开启window子系统
网上有很多教程，这里不做过多的记录，可使用以下教程:Win10安装Ubuntu子系统教程（附安装图形化界面）

![img](https://img-blog.csdnimg.cn/20190704180341303.png)

下载Docker for Window安装包
进入docker官网下载页面

![img](https://img-blog.csdnimg.cn/20190704180156291.png)

有4 5百兆的样子，这里下着就好了，不用等他下完，我们先去做别的，后面再说这个怎么用。

ubuntu下安装Docker
汇总的指令列表

```bash
sudo apt-get remove docker docker-engine docker-ce docker.io
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
      "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) \
      stable"
apt-cache madison docker-ce 
sudo apt-get install docker-ce
sudo service docker start

```



以下是具体的指令说明及执行情况

由于apt官方库里的docker版本可能比较旧，所以先卸载可能存在的旧版本：

```bash
sudo apt-get remove docker docker-engine docker-ce docker.io
```

![img](https://img-blog.csdnimg.cn/20190704182553351.png)

更新apt包索引：

```bash
sudo apt-get update
```



![img](https://img-blog.csdnimg.cn/20190704183307417.png)

安装以下包以使apt可以通过HTTPS使用存储库（repository）：

```
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
```

![img](https://img-blog.csdnimg.cn/20190704183410104.png)

添加Docker官方的GPG密钥：

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

安装stable存储库

```
sudo add-apt-repository \
     "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
     $(lsb_release -cs) \
     stable"
```

查看Docker-ce的版本

```
apt-cache madison docker-ce
```

安装docker-ce

```
sudo apt-get install docker-ce
```

或者指定以上查出来的版本，如

```
sudo apt-get install docker-ce=18.06.3~ce~3-0~ubuntu
```

启动服务

```
sudo service docker start
```

##### 完了？可以用了？然而并没有！！！

 

查看版本及状态
下图看见没"**Docker is not runing**"

![img](https://img-blog.csdnimg.cn/20190704185158566.png)

这里只有client，没有发现server，之前习惯使用的Centos，比较少使用ubuntu，因此一度让我怀疑是我安装的时候出现了问题，因此测试了各种安装方式，均得到了同样的结果；反复找原因之后，这篇文章windows10 linux子系统 ubuntu 18.0运行docker让我明白了，windows10子系统有其特殊性，需要安装docker for windows，用来作为Docker的守护进程，作为Docker的服务端，ubuntu下作为客户端去访问这个守护进程，这也就是文章一开头为什么要下载docker for window的原因


#### 安装 Docker for Windows

 

- 安装
  安装很简单，选择下载好的exe，下一步到结束即可，桌面会出现以下图标即可；
- 问题一,Typer-V的错误
  Typer-V如果没有开启，启动软件的时候会报错，这里开启，重启电脑即可

![img](https://img-blog.csdnimg.cn/2019070419000131.png)

问题二,Cpu虚拟化
在任务管理器中查看

![img](https://img-blog.csdnimg.cn/20190704190157592.png)

开启虚拟化
如果这里是**已禁用**，需要在BOIS开启CPU的虚拟化即可
每个电脑的进入BOIS的方式不太一样，可以百度一下自己的电脑对应型号是如何开启CPU虚拟化的，我的电脑是使用F1进入BOIS

![img](https://img-blog.csdnimg.cn/20190704193816604.png)

![img](https://img-blog.csdnimg.cn/20190704193841116.png)

![img](https://img-blog.csdnimg.cn/20190704194546141.png)

保存重启电脑，在**任务管理器**中确认**虚拟化已启动**即可

 启动成功
成功启动之后，会引导你登录官方Docker的账号(不登录也可以)，同时电脑的右下方会有个小鲸鱼，右键小鲸鱼，可以看到以下效果

![img](https://img-blog.csdnimg.cn/20190704193752784.png)

![img](https://img-blog.csdnimg.cn/20190704191102827.png)

开启2375端口对外提供服务
右键小鲸鱼 > settings > General，勾选上以下选项

![img](https://img-blog.csdnimg.cn/20190704191432100.png)

#### 子进程链接宿主机Docker守护进程

- 配置及刷新环境变量

  ```
  echo "export DOCKER_HOST='tcp://0.0.0.0:2375'" >> ~/.bashrc
  source ~/.bashrc
  ```

- 再次查看版本

![img](https://img-blog.csdnimg.cn/20190704192049463.png)

- 哦豁！！！ 好了。。。。