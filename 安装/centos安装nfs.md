#  centos安装**nfs**

##  1. 安装nfs

### 1.1 安装

```bash
yum install nfs-utils rpcbind
```



### 1.2 启动`rpcbind` 服务

```bash
systemctl restart rpcbind.service
```



### 1.3  查看服务状态

```bash
systemctl status rpcbind.service
```



### 1.4  启动NFS服务

```bash
systemctl start nfs.service
```

> 查看状态

```bash
systemctl status nfs.service
```



> 查看RPC注册的端口信息

```bash
rpcinfo -p localhost

```



### 1.5  启动

> 顺序一定是rpcbind->nfs，否则有可能出现错误

```bash
systemctl start rpcbind.service

systemctl enable rpcbind.service

systemctl start nfs.service

systemctl enable nfs.service
```

### 1.6 配置

创建文件存储目录

```bash
mkdir /nfs
```

```bash
vim /etc/exports
```

 在最后面添加一行：

```bash
/nfs *(insecure,rw,sync,no_root_squash,no_subtree_check)
```



​       （/nfs为root目录下新建的一个文件夹，这个文件夹就是nfs服务对外的共享目录，名字可以随便。

，注意如果当前登录用户不是root那么这个目录必须要在root目录下，不可以是当前登录用户的根目录

　　#重载exports配置

```bash
exportfs -r
```



　#查看共享参数

```bash
exportfs -v　
```





此时NFS服务就搭建成功了. 

