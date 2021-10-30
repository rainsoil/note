#  docker安装可道云私有网盘

## 1. 镜像查找

我们可以在`https://dashboard.daocloud.io/packages/50846266-dc26-473e-9725-4e5e2d8c7534` 中找见 `KodExplorer docker`的`docker` 最新镜像



## 2.   容器启动

```bash
docker  run -d -p 8888:80   daocloud.io/huiqiang/kod:master
或者
docker run -d -p 8888:80  registry.cn-hangzhou.aliyuncs.com/l_third_party/kod:master
```





## 3. 页面访问

浏览器输入:

`localhost:8888`,然后设置管理员账号,就可以了.



## 4. app端使用

我们可以在应用市场搜索 `可道云基础版`,然后输入上面的地址和用户的账号密码即可以登录了. 

这里分享一个基础班, 可以配合我的镜像的版本使用

```text
链接：https://pan.baidu.com/s/1MZKjkN69ERiR6-fBMZ0yfg 
提取码：3nr1
```

