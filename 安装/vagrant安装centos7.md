#  基于vagrant 安装centos7 环境

## 1. 安装`Vagrant`

1. 访问官网

https://www.vagrantup.com/

2. 点击Download
   Windows，MacOS，Linux等

3. 选择对应的版本

4. 傻瓜式的安装

   1. 在命令行输入`vagrant` ,出现

      ![image-20200414114016716](http://files.luyanan.com//img/20200414121803.png)

就表明安装成功了.



## 2. 安装virtual box

1. 访问官网`https://www.virtualbox.org/`
2. 选择左侧的 “Downloads”
3. 选择对应的操作系统版本
4. 傻瓜式安装
5. [win10中若出现]安装virtualbox快完成时立即回滚，并提示安装出现严重错误
       (1)打开服务
       (2)找到Device Install Service和Device Setup Manager，然后启动
       (3)再次尝试安装

## 3. 安装centos

1. 创建`centos7` 目录, 并进去启动(目录全路径不要有中文空格)

2. 在此目录下打开`cmd` ,运行 `vagrant init centos/7`, (`centos7`是镜像,镜像可以去镜像仓库`https://app.vagrantup.com/boxes/search` 查找) 

3. 或者你也可以下载 `virtualbox.box` 文件到本地, 然后再将`virtualbox.box` 文件添加到`vagrant`  管理的镜像中. 

    添加镜像并起名叫centos/7：`vagrant box add centos/7 D:\virtualbox.box`

   使用`vagrant bos list` 就可以查看本地的镜像了

4. 镜像有了,接下来根据Vagrantfile文件启动创建虚拟机

    来到`centos` 文件夹,在此目录下打开`cmd` 窗口,执行`vagrant up`, 然后打开`virtual box`,可以看到`centos7` 创建成功. 

5. `vagrant` 常用命令

    - `vagrant ssh    `

       进入刚才创建的centos7中

    - `vagrant status`

       查看centos7的状态

    - `vagrant halt`

       停止/关闭centos7

    - `vagrant destroy`

       删除centos7

    - Vagrantfile中也可以写脚本命令，使得centos7更加丰富

       但是要注意，修改了Vagrantfile，要想使正常运行的centos7生效，必须使用vagrant reload



至此，使用vagrant+virtualbox搭建centos7完成，后面可以修改Vagrantfile对虚拟机进行相应配置

## 4. 使用xshell 连接虚拟机

1. 使用虚拟机的默认账号连接 

    在`centos` 目录下执行`vagrant ssh-config`

   关注:Hostname  Port  IdentityFile
   	IP:127.0.0.1
   	port:2222
   	用户名:vagrant
   	密码:vagrant
   	文件:Identityfile指向的文件private-key

2. 使用root账号登陆

   1. vagrant ssh   进入到虚拟机中
   2. 

    ```bash
   sudo -i
    ```

   3. 修改配置

       ````bash
      vi /etc/ssh/sshd_config
      ````

      修改 `PasswordAuthentication yes`

   4. 修改密码
   
        使用`passwd` 修改密码
   
   5. 重启
   
        `systemctl restart sshd` 重启
   
   

## 5. `Vagrantfile`通用写法

```ruby
ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  config.vm.box = "centos/7"

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # NOTE: This will enable public access to the opened port
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine and only allow access
  # via 127.0.0.1 to disable public access
  # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  # config.vm.network "private_network", ip: "192.168.33.10"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
    config.vm.provider "virtualbox" do |vb|
        vb.memory = "4000"
        vb.name= "centos7-01"
        vb.cpus= 2
    end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  # config.vm.provision "shell", inline: <<-SHELL
  #   apt-get update
  #   apt-get install -y apache2
  # SHELL
end
```



      ## 6. `box` 的打包分发

1.  退出虚拟机
      	`vagrant halt`

2. 打包
   	`vagrant package --output first-docker-centos7.box`

3. ​	得到`first-docker-centos7.box`

4.  将`first-docker-centos7.box`添加到其他的vagrant环境中

    `vagrant box add first-docker-centos7 first-docker-centos7.box`

5. 得到`Vagrantfile`

    `vagrant init first-docker-centos7`

6. 根据`Vagrantfile`启动虚拟机

    `vagrant up` [此时可以得到和之前一模一样的环境，但是网络要重新配置]

   

​    

​      

