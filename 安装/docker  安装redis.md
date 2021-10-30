#### 1. 拉取容器

> docker pull redis:4.0
#### 2. 查看本地容器
> docker images
##### 3. 启动
```dockerfile
docker run --name redis  -p 6379:6379 -v $PWD/data:/data  -d --restart=always  redis:4.0 redis-server --appendonly yes --requirepass "your passwd"
```

