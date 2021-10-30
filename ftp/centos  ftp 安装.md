### 1. 安装VSFTPD
> 使用 yum 安装

> yum install -y vsftpd
### 2. 启动 VSFTPD
```
service vsftpd start
 // 查看系统是否监听了21端口
 netstat -nltp | grep 21
```
### 3. 修改配置 `/etc/vsftpd/vsftpd.conf`

```
anonymous_enable=NO
chroot_local_user=YES
chroot_local_user=YES
listen_ipv6=YES
userlist_enable=YES
tcp_wrappers=YES
pasv_min_port=30000
pasv_max_port=30999
allow_writeable_chroot=YES
pam_service_name=vsftpd
```

### 4. 重启
> service vsftpd restart
### 5. 配置登录的用户名和密码
```
adduser -d /home/upload/  -g ftp -s /sbin/nologin beinan
// 配置密码
passwd beinan

```
### 7. 开通端口
> 开通30000-39999 和21的端口号