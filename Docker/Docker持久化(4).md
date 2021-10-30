# Docker 数据持久化

## 1. `Volume`

1. 创建`mysql` 数据库的`contailer`

   ```bash
   docker run -d --name mysql01 -e MYSQL_ROOT_PASSWORD=jack123 mysql
   
   ```

2. 查看`volume`

   ```bash
   docker volume ls
   ```

3. 具体查看该`volume`

    ```bash
   docker volume inspect
   48507d0e7936f94eb984adf8177ec50fc6a7ecd8745ea0bc165ef485371589e8
   ```

4. 名字不好看,name太长, 修改一下

    `-v mysql01_volume:/var/lib/mysql` 表示给上述的`vloume` 起一个能够识别的名字

   ```bash
   docker run -d --name mysql01 -v mysql01_volume:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=jack123 mysql
   ```

5. 查看`volume` 

   ```bash
   docker volume ls
   docker volume inspect mysql01 vloume
   ```

6. 真的能够持久化保存数据吗? 不妨做个试验

   ```bash
   ## 进入容器
   docker exec -it mysql01 bash
   ## 登录mysql服务
   mysql -uroot -pjack123
   ## 创建测试库
   create database db_test
   ## 退出mysql, 退出`contailer`
   ## 删除mysql 容器
   docker rm -f mysql01
   ## 查看`volume`
   docker volume ls
   ## 发现`volume` 还在
   DRIVER VOLUME NAME
   local mysql01_volume
   # 新建一个mysql container，并且指定使用"mysql01_volume"
   docker run -d --name test-mysql -v mysql01_volume:/var/lib/mysql -e
   MYSQL_ROOT_PASSWORD=jack123 mysql
   # 进入容器,登录mysql 服务,查看数据库
   docker exec -it test-mysql bash
   mysql -uroot -pjack123
   show database;
   # 可以发现db_test仍然在
    可以发现db_test仍然在
   | information_schema |
   | db_test |
   | mysql |
   | performance_schema |
   | sys
   
   ```





##  2. Bing Mounting

1. 创建`tomcat` 容器

   ```bash
   docker run -d --name tomcat01 -p 9090:8080 -v /tmp/test:/usr/local/tomcat/webapps/test tomcat
   ```

2. 查看两个目录

    ```bash
   centos：cd /tmp/test
   tomcat容器：cd /usr/local/tomcat/webapps/test
   ```

3. 在`centos`的`/tmp/test` 中新建一个`1.html`并且也有内容

4. 在`centos7` 上访问该路径,`curl localhost:9090/test/1.html`

5. 在`wim` 浏览器上通过`ip` 访问

   ![image-20200417165051450](http://files.luyanan.com//img/20200417165112.png)



