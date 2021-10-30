#  docker安装nginx

- 随便启动一个nginx实例，只是为了复制出配置

```shell
docker run -p80:80 --name nginx -d nginx:1.10
```

- 将容器内的配置文件拷贝到/mydata/nginx/conf/ 下

```shell
mkdir -p /mydata/nginx/html
mkdir -p /mydata/nginx/logs
mkdir -p /mydata/nginx/conf
docker container cp nginx:/etc/nginx/*  /mydata/nginx/conf/ 
#由于拷贝完成后会在config中存在一个nginx文件夹，所以需要将它的内容移动到conf中
mv /mydata/nginx/conf/nginx/* /mydata/nginx/conf/
rm -rf /mydata/nginx/conf/nginx
```

- 终止原容器：

```shell
docker stop nginx
```

- 执行命令删除原容器：

```shell
docker rm nginx
```

- 创建新的Nginx，执行以下命令

```shell
docker run -p 80:80 --name nginx \
 -v /mydata/nginx/html:/usr/share/nginx/html \
 -v /mydata/nginx/logs:/var/log/nginx \
 -v /mydata/nginx/conf/:/etc/nginx \
 -d nginx:1.10
```

- 设置开机启动nginx

```
docker update nginx --restart=always
```

- 创建“/mydata/nginx/html/index.html”文件，测试是否能够正常访问

```
echo '<h2>hello nginx!</h2>' >index.html
```

- 访问：http://ngix所在主机的IP:80/index.html