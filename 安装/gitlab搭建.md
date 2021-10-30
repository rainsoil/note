

#   搭建gitlab

https://about.gitlab.com/install/#centos-7



## 1. 说明

安装`gitlab` 的机器至少需要4G的内存, 因为`gitlab` 比较吃内存. 



## 2. 安装必要的依赖

```bash
sudo yum install -y curl policycoreutils-python openssh-server
sudo systemctl enable sshd
sudo systemctl start sshd
sudo firewall-cmd --permanent --add-service=http
sudo systemctl reload firewalld
```



## 3. 如果想要发送邮件, 就执行

```bash
sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix
```



## 4. 添加`gitlab`的仓库地址

```bash
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.rpm.sh | sudo
bash

```

 这个下载仓库可能速度会很慢,此时可以使用国内的仓库地址

```bash
新建文件 /etc/yum.repos.d/gitlab-ce.repo
内容为
[gitlab-ce]
name=Gitlab CE Repository
baseurl=https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el$releasever/
gpgcheck=0
enabled=1

```



## 5. 设置`gitlab`的域名和安装`gitlab`

```bash
sudo EXTERNAL_URL="https://gitlab.xxx.com" yum install -y gitlab-ee
如果用的是国内仓库地址，则执行以下命令，其实区别就是ee和ce版
sudo EXTERNAL_URL="https://gitlab.xxx.com" yum install -y gitlab-ce

```

此时要么买一个域名，要么在本地的hosts文件中设置一下
安装gitlab服务器的ip地址`https://gitlab.xxx.com`

假如不想设置域名,可以直接安装

```bash
yum install -y gitlab-ee
```

## 6. 重新`configure`

如果没有成功,可以运行 

```bash
gitlab-ctl reconfigure
```



## 7.查看`gitlab`的运行情况

```bash
gitlab-ctl status
```

 可以查看 运行`gitlab` 服务所需要的进程

## 8.访问

浏览器输入 `https://gitlab.xxx.com`  , 此时需要修改`root` 账号的密码

## 9.  配置已经安装好的`gitlab`

 ```bash
vim /etc/gitlab/gitlab.rb
 ```

修改完成之后一定要 `gitlab-ctl reconfigure`