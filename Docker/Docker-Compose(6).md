#  Docker之Docker-Compose

> `官网`：<https://docs.docker.com/compose/>

##  1.业务背景

## 2 Docker传统方式实现

### 2.1 写`Python`代码&`build image`

> (1)创建文件夹

```shell
mkdir -p /tmp/composetest
cd /tmp/composetest
```

> (2)创建`app.py`文件，写业务内容

```python
import time

import redis
from flask import Flask

app = Flask(__name__)
cache = redis.Redis(host='redis', port=6379)

def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)

@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello World! I have been seen {} times.\n'.format(count)
```

> (3)新建`requirements.txt`文件

```
flask
redis
```

> (4)编写`Dockerfile`

```dockerfile
FROM python:3.7-alpine
WORKDIR /code
ENV FLASK_APP app.py
ENV FLASK_RUN_HOST 0.0.0.0
RUN apk add --no-cache gcc musl-dev linux-headers
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
COPY . .
CMD ["flask", "run"]
```

> (5)根据`Dockerfile`生成`image`

```
docker build -t python-app-image .
```

> (6)查看`images`：`docker images`

```
python-app-image latest 7e1d81f366b7 3 minutes ago  213MB
```

### 2.2 获取`Redis`的`image`

```
docker pull redis:alpine
```

### 2.3 创建两个`container`

> (1)创建网络

```
docker network ls
docker network create --subnet=172.20.0.0/24 app-net 
```

> (1)创建`python`程序的`container`，并指定网段和端口

```
docker run -d --name web -p 5000:5000 --network app-net python-app-image
```

> (2)创建`redis`的`container`，并指定网段

```
docker run -d --name redis --network app-net redis:alpine
```

### 2.4 访问测试

ip[centos]:5000

##  3 简介和安装

### 3.1 简介

> `官网`：https://docs.docker.com/compose/
>
> ```
> Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration.
> ```

### 3.2 安装

> Linux环境中需要单独安装
>
> `官网`：https://docs.docker.com/compose/install/

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

```
sudo chmod +x /usr/local/bin/docker-compose
```

##  4 `docker compose`实现

> `reference`：<https://docs.docker.com/compose/gettingstarted/>

### 4.1 同样的前期准备

新建目录，比如`composetest`

进入目录，编写`app.py`代码

创建`requirements.txt`文件

编写`Dockerfile`

### 4.2 编写`docker-compose.yaml`文件

默认名称，当然也可以指定，`docker-compose.yaml`

```yaml
version: '3'
services:
  web:
    build: .
    ports:
      - "5000:5000"
    networks:
      - app-net

  redis:
    image: "redis:alpine"
    networks:
      - app-net

networks:
  app-net:
    driver: bridge
```

> (1)通过`docker compose`创建容器

```
docker-compose up -d
```

> (2)访问测试

![image-20200417214115069](http://files.luyanan.com//img/20200417214122.png)

## 5 详解`docker-compose.yml`文件

> (1)`version: '3'`

```
表示docker-compose的版本
```

> (2)`services`

```
一个service表示一个container
```

> (3)`networks`

```
相当于docker network create app-net
```

> (4)`volumes`

```
相当于-v v1:/var/lib/mysql
```

> (5)`image`

```
表示使用哪个镜像，本地build则用build，远端则用image
```

> (6)`ports`

```
相当于-p 8080:8080
```

> (7)`environment`

```
相当于-e 
```

##  6 `docker-compose`常见操作

(1)查看版本

	docker-compose version

(2)根据`yml`创建`service`

	docker-compose up
	
	指定yaml：docker-compose  up -f xxx.yaml
	
	后台运行：docker-compose up

(3)查看启动成功的`service`

	docker-compose ps
	
	也可以使用docker ps

(4)查看`images`

	docker-compose images

(5)停止/启动service

	docker-compose stop/start 

(6)删除`service`[同时会删除掉`network`和`volume`]

	docker-compose down

(7)进入到某个service

	docker-compose exec redis sh

## 7 `scale`扩缩容

> (1)修改`docker-compose.yaml`文件，主要是把`web`的`ports`去掉，不然会报错

```yaml
version: '3'
services:
  web:
    build: .
    networks:
      - app-net

  redis:
    image: "redis:alpine"
    networks:
      - app-net

networks:
  app-net:
    driver: bridge
```

> (2)创建`service`

```
docker-compose up -d
```

> (3)若要对`python`容器进行扩缩容

```
docker-compose up --scale web=5 -d
docker-compose ps
docker-compose logs web
```