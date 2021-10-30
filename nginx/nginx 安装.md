

### 1. 环境安装
#####  1.1    安装nginx需要先将官网下载的源码进行编译，编译依赖gcc环境，如果没有gcc环境，需要安装gcc
> yum install gcc-c++
> ##### 1.2  PCRE(Perl Compatible Regular Expressions)是一个Perl库，包括 perl 兼容的正则表达式库。nginx的http模块使用pcre来解析正则表达式，所以需要在linux上安装pcre库
> yum install -y pcre pcre-devel
> ##### 1.3  zlib库提供了很多种压缩和解压缩的方式，nginx使用zlib对http包的内容进行gzip，所以需要在linux上安装zlib库。
> yum install -y zlib zlib-devel
>
> #####  1.4 OpenSSL 是一个强大的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及SSL协议，并提供丰富的应用程序供测试或其它目的使用。
>
> #####  nginx不仅支持http协议，还支持https（即在ssl协议上传输http），所以需要在linux安装openssl库
> yum install -y openssl openssl-devel
### 2. 编译安装
#### 2.1 将nginx-1.8.0.tar.gz拷贝至linux服务器。
#### 2.2 解压
> tar -zxvf nginx-1.8.0.tar.gz
> cd nginx-1.8.0
#### 2.3 编译
```
 ./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module
make 
make install 
```
### 3. 启动重启
#### 3.1 启动
```
cd /usr/local/nginx/sbin/
./nginx
// 执行./nginx启动nginx，这里可以-c指定加载的nginx配置文件，如下：./nginx -c /usr/local/nginx/conf/nginx.conf.如果不指定-c，nginx在启动时默认加载conf/nginx.conf文件，此文件的地址也可以在编译安装nginx时指定./configure的参数（--conf-path= 指向配置文件（nginx.conf））
```
#### 3.2 停止nginx
##### 3.2.1 快速停止
```
cd /usr/local/nginx/sbin
./nginx -s stop
//此方式相当于先查出nginx进程id再使用kill命令强制杀掉进程。
```
##### 3.2.2  完整停止
```
cd /usr/local/nginx/sbin
./nginx -s quit
 // 此方式停止步骤是待nginx进程处理任务完毕进行停止。
```
### 3.3 重启
##### 3.3.1  先停止再启动
```
// 对nginx进行重启相当于先停止nginx再启动nginx，即先执行停止命令再执行启动命令。
如下：
./nginx -s quit
./nginx
```
##### 3.3.2  重新加载配置文件：
```
// 当nginx的配置文件nginx.conf修改后，要想让配置生效需要重启nginx，使用-s reload不用先停止nginx再启动nginx即可将配置信息在nginx中生效，如下：
./nginx -s reload
```