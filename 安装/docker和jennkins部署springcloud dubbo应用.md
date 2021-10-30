# 基于docker和jenkins实现springboot dubbo的自动化部署

## 1. 容器镜像仓库

我们这里使用的是阿里的容器镜像仓库([https://cr.console.aliyun.com/]:)

### 1.1 创建镜像仓库

![](http://files.luyanan.com//img/20200229163815.png)

我们这里填好仓库名称(只能小写), 然后下一步



![](http://files.luyanan.com//img/20200229163923.png)

点击创建镜像即可. 

###  1.2 获取仓库地址

![](http://files.luyanan.com//img/20200229164023.png)	我们点击这里的管理, ![image-20200229164047657](C:/Users/luyanan/AppData/Roaming/Typora/typora-user-images/image-20200229164047657.png)	复制上面的地址即可. 这就是我们创建的仓库地址

###  1.3  获取访问凭证

我们上传镜像的时候, 需要一个上传的密码(不是阿里云的登陆密码). 我这里为了麻烦获取一个固定的密码

![](http://files.luyanan.com//img/20200229164307.png) 



##  2 .上传jar到docker仓库

### 2.1  编写Dockerfile文件

```text
FROM openjdk:8-jdk-alpine
MAINTAINER luyanan <luyanan0718@163.com>

VOLUME /tmp
COPY target/app.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]

```



### 2.2 配置maven的pom文件

我们这里使用的是`docker-maven-plugin` 插件

```xml

    <properties>
        <!-- 这里配置的是仓库地址-->>
        <docker.repository>registry.cn-hangzhou.aliyuncs.com/luyanan/fieldarmy-web</docker.repository>
        <docker.maven.plugin.version>0.4.3</docker.maven.plugin.version>
    </properties>

<build>
        <finalName>app</finalName>
        <!--<finalName>${project.artifactId}</finalName>-->
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                </configuration>
            </plugin>
        </plugins>
        <pluginManagement>

            <plugins>
                <plugin>
                    <groupId>com.spotify</groupId>
                    <artifactId>docker-maven-plugin</artifactId>
                    <version>${docker.maven.plugin.version}</version>
                    <configuration>
                        <!-- build 时，指定 –no-cache 不使用缓存	-->
                        <noCache>true</noCache>
                        <!-- build 时，指定 –rm=true 即 build 完成后删除中间容器	-->
                        <!--<rm>true</rm>-->
                        <!--build 完成后 push 镜像	 -->
                        <pushImage>true</pushImage>
                        <!--设置镜像的名字为镜像仓库地址-->
                        <imageName>${docker.repository}</imageName>
                        <!--设置镜像的tag为项目的maven构建的jar的名字和版本号-->
                        <imageTags>
                            <!--<imageTag>${project.build.finalName}V${project.version}</imageTag>-->
                            <!--<imageTag>latest</imageTag>-->
                            <imageTag>${project.version}</imageTag>
                        </imageTags>
                        <!--存放Dockerfile的文件夹-->
                        <dockerDirectory>${project.basedir}</dockerDirectory>
                        <!--maven　settings中配置的服务的id-->
                        <serverId>docker-registry</serverId>
                        <!--docker仓库的地址-->
                        <registryUrl>registry.cn-hangzhou.aliyuncs.com</registryUrl>
           
                    </configuration>
                </plugin>

                
            </plugins>

        </pluginManagement>

    </build>

```



### 2.2  配置上传的凭证

需要在maven的`settings.xml`文件中配置

在<servers></servers>中配置

```xml
  <server>
    <id>docker-registry</id>
    <username>阿里云的账号</username>
    <password>上面获取的访问凭证</password>
    <configuration>
      <email>邮箱地址</email>
    </configuration>
  </server>
```



### 2.3 测试打包

在项目目录下执行 `mvn clean package -Dmaven.test.skip=true && mvn docker:build && mvn docker:removeImage` 等执行成功后, 在阿里云的镜像仓库

![](http://files.luyanan.com//img/20200229165317.png)	你就可以看到自己上传的镜像了



## 3. 使用jenkins 部署镜像

这里不讲部署和配置jenkins的过程

需要用到的jenkins插件

- [SSH plugin](http://wiki.jenkins-ci.org/display/JENKINS/SSH+plugin)

###  3.1创建项目

![](http://files.luyanan.com//img/20200229165958.png)

我们创建这么一个项目

![](http://files.luyanan.com//img/20200229170039.png)

勾选这个,设置为多参数

参数为:

- name: 镜像的名称

- command: 启动后执行的命令

- repository: 仓库的地址

- tag: 标签

- port:端口映射

- operate 执行的操作,分为:

  - start: 启动容器
  - stop: 停止容器
  - restart: 重启容器(不会重新拉取镜像)
  - update: 重启拉取镜像,重新启动容器. 

  

  ![](http://files.luyanan.com//img/20200229170326.png)

  ![](http://files.luyanan.com//img/20200229170338.png)

   

![](http://files.luyanan.com//img/20200229170407.png)



### 3.2 执行脚本

我们在**Build** 中选择

![](http://files.luyanan.com//img/20200229170741.png)

![](http://files.luyanan.com//img/20200229170803.png)



**SSH site**需要去

![](http://files.luyanan.com//img/20200229170936.png)中配置

里面执行的脚本

```bash
if [ ! -d "/web/${name}" ]; then
  mkdir -p  /web/${name}
fi
cd /web/${name}
## 是否存在启动脚本
if [ -f "./docker-composeRun.sh" ];then
   echo "file exist"
else
   echo "file not exist"
   wget https://blog-1253651602.cos.ap-beijing.myqcloud.com/software/docker-composeRun.sh
   chmod +x ./docker-composeRun.sh
fi

## 是否存在docker-compose
if [ -f "./docker-compose.yml" ];then
   echo "file exist"
else
   echo "file not exist"
   wget https://blog-1253651602.cos.ap-beijing.myqcloud.com/software/docker-compose.yml
fi

sh  ./docker-composeRun.sh -n ${name} -p ${port} -c ${command} -r ${repository} -t ${tag}  -o "${operate}"
#sh docker-composeRun.sh  -n fieldarmy-blog-single -p 8081:8081 -c '--spring.profiles.active=dev' -t latest -r registry.cn-hangzhou.aliyuncs.com/luyanan/ -o stop

```

脚本的意思是在目标服务器的/web目录下创建 上面参数中执行的 `name的`目录,然后会检测目录下是否存在 `docker-compose.yml`文件和`docker-composeRun.sh`文件,如果不存在则从远程中下载. 然后全部下载成功后, 执行`docker-compose.sh` 脚本

#### docker-compose 

```yml
version: '3'
services:
  base:
    restart: always
    image: ${repository}${name}:${tag}
    container_name: ${name}
    hostname: localhost
    network_mode: host
    ports:
      - ${port}
    env_file:
      - .env
    environment:
      - TZ=Asia/Shanghai
    command: 
      - ${command}
    volumes:
      - ./log:/log
    


```

`docker-compose.yml` 启动的时候, 会默认读取当前目录下的`.env`文件, 所以`docker-composeRun.sh` 脚本的目录就是就是将传入的参数生成`.env`文件. 

```bash
## 输出到.env的内容

#!/bin/sh
#说明
show_usage="args: [-n , -p , -c , -r , -t, -o]\
                                  [--name=, --port=, --command=, --repository=, --tag, --operate]"
#参数
# 容器名称
opt_name=""

# 启动参数
opt_command=""

# 仓库地址
opt_repository=""

## 端口映射
opt_port=''

##  操作
opt_operate=""

GETOPT_ARGS=`getopt -o n:p:c:r:t:o: -al name:,command:,repository:,operate: -- "$@"`
eval set -- "$GETOPT_ARGS"



start(){
   docker-compose up -d
}

stop(){
  docker-compose down
}

restart(){
  docker-compose restart
}

update(){
  stop
  docker-compose pull
  start
}



#获取参数
while [ -n "$1" ]
do
        case "$1" in
                -n|--name) opt_name=$2; shift 2;;
                -c|--command) opt_command=$2; shift 2;;
				-p|--port) opt_port=$2; shift 2;;
				-t|--tag) opt_tag=$2; shift 2;;
                -r|--repository) opt_repository=$2; shift 2;;
                -o|--operate) opt_operate=$2; shift 2;;
                --) break ;;
                *) echo $1,$2,$show_usage; break ;;
        esac
done
if [[ -z $opt_name  || -z $opt_command || -z $opt_repository || -z $opt_operate ]]; then
        echo $opt_operate
        echo $show_usage
        echo "opt_name: $opt_name ,  opt_command: $opt_command , opt_repository: $opt_repository , opt_operate: $opt_operate"
        exit 0
else
    echo "opt_name: $opt_name  , opt_command: $opt_command , opt_repository: $opt_repository , opt_operate: $opt_operate"
	# define web container env
    #repository=registry.cn-hangzhou.aliyuncs.com/luyanan/
    #name=fieldarmy-blog-single
    #tag=latest
    #command=--spring.profiles.active=dev
    #port=8081:8081
	echo repository=$opt_repository > .env
	echo name=$opt_name >> .env
	echo tag=$opt_tag >> .env
	echo command=$opt_command >> .env
	echo port=$opt_port >> .env
	case $opt_operate in 
	   start)
	   echo '启动中'
	   start
	   ;;
	   stop)
	   echo '停止中'
	   stop
	   ;;
	   restart)
	   echo '重启中'
	   restart
	   ;;
	   update)
	   echo '重新拉取镜像'
	   update
	   ;;
	   
	esac

fi


```

当然, 也可以在项目目录下使用`docker-compose` 的命令进行操作. 



## 4. 测试部署

![](http://files.luyanan.com//img/20200229171952.png)

然后,我们就可以进行参数化部署了, 点击`Build` 就可以将生成的镜像部署到远程服务器了. 



## 5. 注意: 

因为解决dubbo项目启动的时候, 注册到注册中心的ip为容器中ip, 访问不到的情况, 所以`docker-compose.yml` 中必须配置

```yml
    hostname: localhost
    network_mode: host
```

