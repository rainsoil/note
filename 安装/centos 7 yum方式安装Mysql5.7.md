# centos 7 yum方式安装Mysql5.7

在CentOS中默认安装有MariaDB（MySQL的一个分支），安装完成之后可以直接覆盖MariaDB。

MySQL版本号：5.7.28

下载yum repository

```bash
wget -i -c http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
```

安装

```bash
yum -y install mysql57-community-release-el7-10.noarch.rpm
```

安装MySQL服务器

```bash
yum -y install mysql-community-server
```

启动MySQL

```bash
systemctl start  mysqld.service
```

查看运行状态

```bash
systemctl status mysqld.service
```

找到MySQL root用户的初始密码：

```bash
grep "password" /var/log/mysqld.log
```

![image-20200403162115604](http://files.luyanan.com//img/20200403162117.png)

使用临时密码连接客户端：

```
mysql -uroot -p:E+,Y_Dp_35j
```

修改密码安全限制，否则不能使用简单密码
临时修改：

```
mysql> set global validate_password_policy=0;
mysql> set global validate_password_length=1;
```

永久修改：
MySQL默认的配置文件：
vim /etc/my.cnf

```
validate_password_policy=0
validate_password_length=1
```

修改后重启MySQL

```
service mysqld restart
```

修改密码：

```
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';
```

授权远程访问：

```
mysql> grant all privileges on *.* to 'root'@'%' identified by '123456'; 
```

如果需要远程连接，注意开放3306端口或者关闭防火墙。

MySQL默认的数据文件目录：

```bash
show variables like ‘datadir’;
/var/lib/mysql/
```



MySQL默认错误日志文件：

```bash
show variables like ‘log_error’;
/var/log/mysqld.log
```

如果忘记了root密码或者用临时密码无法登录：

```
vim /etc/my.cnf
```

在配置文件中加一行skip-grant-tables

```
[mysqld]
skip-grant-tables
```

重启数据库服务

```
service mysqld restart
```

然后使用mysql命令登录，使用以下密码修改密码。

```
mysql> update user set authentication_string=password('123456')  where Host='localhost' and User='root';
```

修改以后，在配置文件中去掉skip-grant-tables，重启数据库服务。
再使用 `mysql -uroot -p123456`登录。修改密码安全限制和授权远程访问依然要做。

