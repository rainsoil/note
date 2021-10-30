###  linux 环境下搭建vps 服务
> 说明:
>
> > 由于大部分的VPN被封,无意间接触到了VPS（Virtual Private Server 虚拟专用服务器，可用于FQ），所以简单记录下VPS服务搭建流程。

 >> 此教程基于centos7,使用阿里服务器(香港区域,可访问外网)进行搭建

 #### 1. 安装组件
  ##### 1.1 安装python 组件
  ```
  yum install m2crypto python-setuptools easy_install pip
  ```
  ##### 1.2 配置参数
  > 新建并编辑文件:


 ```
 vim /etc/shadowsocks.json
 ```

 > 拷贝如下配置至文件末尾:

````
{
"server":"0.0.0.0",
"server_port":8388,
"local_address":"127.0.0.1",
"local_port":1080,
"password":"password",
"timeout":300,
"method":"aes-256-cfb",
"fast_open":false,
"workers":1
}
````
> 主要参数说明:

>> server_port 表示开放VPS 服务端口,password 表示登录密码,

> 启动服务:
>
> >  启动命令:

>>>  ssserver -c /etc/shadowsocks.json
### 2. 链接VPS
>>>  去 https://github.com/shadowsocks 下载  shadowsocks的window客户端(https://github.com/shadowsocks/shadowsocks-windows/releases),将vps服务器ip,端口号,密码都填写成功就可以链接了

 <font color = red>
 注意: 如果链接不上,查看端口是否开放
 </font>

### 3.后台启动

 >> 由于以上启动方式为直接启动,关闭会话框窗户就会关闭服务,所以使用supervisor 实现后台运行
 >>
 >> > 安装python工具
 ```
 yum install python-setuptools
 ```
 >>>  安装supervisor
```
easy_install supervisor
```
>>> 创建配置文件
```
echo_supervisord_conf >/etc/supervisord.conf
```
>>>  添加任务
```
vi /etc/supervisord.conf
```
>>> 拷贝如下配置至文件末尾:
```
[program:ssserver]command = ssserver -c /etc/shadowsocks.json
autostart=true
autorestart=true
startsecs=3
```
>>> 测试配置是否成功:
>>> >     supervisord -c /etc/supervisord.conf,再使用ps -ef | grep shadowsocks查看进程是否存在，如果进程存在则配置成功。
>>> >     配置开机启动
>>> >     vi /etc/rc.d/rc.local 在末尾行添加supervisord，此外centos7还需要配置文件权限：chmod +x /etc/rc.local，重启服务器即可自动运行。
>>> >     PS