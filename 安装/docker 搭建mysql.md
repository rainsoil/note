####  1. 新建 docker 目录

####  2. 创建mysql 目录
#### 3. 拉取容器 
> docker pull mysql:5.7

#### 4.启动docker

```
docker run -p 3306:3306 --name mymysql -v $PWD/conf:/etc/mysql/conf.d -v $PWD/logs:/logs -v $PWD/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7
 //  命令说明
     -p 3306:3306：将容器的 3306 端口映射到主机的 3306 端口
     -v -v $PWD/conf:/etc/mysql/conf.d：将主机当前目录下的 conf/my.cnf 挂载到容器的 /etc/mysql/my.cnf。
     -v $PWD/logs:/logs：将主机当前目录下的 logs 目录挂载到容器的 /logs。
     -v $PWD/data:/var/lib/mysql ：将主机当前目录下的data目录挂载到容器的 /var/lib/mysql 。
     -e MYSQL_ROOT_PASSWORD=123456：初始化 root 用户的密码。
```
#### 5. 查看容器使用情况
> docker ps

#### 6. 修改编码

> 在 config 下创建my.cnf 

>添加:
```
[mysqld]
user=mysql
character-set-server=utf8
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
```
> 重启容器

####  7. 创建自定义用户和数据库

首先进入容器

```bash
docker exec -it mymysql bash
```

登录root用户

```bash
mysql -uroot -p123456;

```



上面的这两个命令就可以通过`alias` 别名简化

```bash
vim ~/.bashrc

```

加入一行

```
alias sql='docker exec -it mymysql mysql -uroot -p123456'
```



编译生效

```bash
source ~/.bashrc

```



进入客户端执行

```bash
mysql> create user 'test'@'%' identified by '123456';
mysql> flush privileges;

mysql> create database db0 DEFAULT CHARSET utf8mb4 COLLATE utf8mb4_general_ci;

mysql> grant all privileges on db0.* to 'test'@'%' identified by '123456';


flush privileges;
```

