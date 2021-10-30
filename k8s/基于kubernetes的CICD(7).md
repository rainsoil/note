# 基于kubernetes的CICD.md



> 思考: 当我们需要部署一个项目到k8s中的时候,需要编写`Dockerfile`-> 打包-> `push` 镜像-> 编写k8s 配置文件-> 在进行启动等等。 
>
> 当我们需要对这个项目的某些代码进行修改的时候,还要重新打包->push镜像->k8s启动等等. 

![image-20200430100403943](http://files.luyanan.com//img/20200430100422.png)

> 思考: 如果能够按照上述图解一样,在本地进行开发,然后`git push` 到`guthub`, 就能访问最终的应用多好. 

## 1. 环境准备

### 1.1 基础环境准备(在安装`jenkins`的机器上)

####  1.1.1   安装java 

1. 下载jdk 资源并上传到指定的机器

    ```bash
   resources/cicd/jdk-8u181-linux-x64.tar.gz
   
    ```

2. 配置环境变量

   ```bash
   vim /etc/profile
   export JAVA_HOME=/usr/local/java/jdk1.8.0_181
   export
   CLASSPATH=.:${JAVA_HOME}/jre/lib/rt.jar:${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools
   .jar
   export PATH=$PATH:${JAVA_HOME}/bin
   
   
   ```

3. 重启环境变量

   ```bash
   source /etc/profile
   java -version
   ```



#### 1.1.2  maven 安装

1. 上传maven 到指定的机器

   ```bash
   resources/cicd/apache-maven-3.6.2-bin.tar.gz
   
   ```

2. 配置环境变量

   ```bash
   vim /etc/profile
   export MAVEN_HOME=/usr/local/maven/apache-maven-3.6.2
   export PATH=$PATH:$JAVA_HOME/bin:$MAVEN_HOME/bin
   ```

3. 刷新并验证

   ```bash
   source /etc/profile
   mvn -version
   ```

4. 在`setting.xml` 文件中配置阿里云镜像

   ```xml
     <mirror>
         <id>alimaven</id>
         <name>aliyun maven</name>
         <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
         <mirrorOf>central</mirrorOf>        
     </mirror>
   ```



#### 1.1.3  git安装

1. 下载安装

   ```bash
   yum install git -y
   ```

2. 配置git

   ```bash
   git config --global user.name "luyanan0718"
   git config --global user.email "luyanan0718@163.com"
   ssh-keygen -t rsa -C "luyanan0718@163.com" --->将公钥上传到github:/root/.ssh/id_rsa.pub
   ```





###  1.2 项目准备

这里准备一个 `springboot-demo` 项目,git地址为: `https://github.com/luyanan0718/springboot-demo.git`

### 1.3 jenkens

> 必须在k8s集群中, 因为后面需要在jenkins的目录下创建文件执行,比如这里选用的是w2节点

1.  操作前须知

`jenkins官网`:https://jenkins.io/

`入门指南`:<https://jenkins.io/zh/doc/pipeline/tour/getting-started/

2. 找到对应的资源

   ```bash
   resources/cicd/jenkins.war
   ```

   ```
   wget http://mirrors.jenkins.io/war-stable/latest/jenkins.war
   ```

3.  启动

   ```bash
   nohup java -jar jenkins.war --httpPort=8080 & 
   tail -f nohup.out 
   ```

4.   浏览器访问`ip:port` ,输入密码

    密码:  `cat /root/.jenkins/secrets/initialAdminPassword`

5. 安装推荐的插件

6. 创建用户或者直接使用root用户

7. [系统管理]->[全局工具配置] 中配置java、maven、git等

### 1.4 Docker Hub

可以自己搭建`Docker`仓库, 也可以使用阿里云的



### 1.5 k8s 集群



## 2. 必要的测试

### 1.2.1 `pipeline` 任务

1. 创建`jenkins`的`task`

2. 拉取github代码, 在最下面编写`pipeline`,然后保存和立即构建,同时查看`Console output`

    ```bash
   node {
      def mvnHome
      stage('Preparation') { // for display purposes
   
         git 'https://github.com/luyanan0718/springboot-demo.git'
   
      }
   }
    ```

3. 来到 `jenkins` 所在的服务器,

   ```bash
   ls /root/.jenkins/workspace/springboot-demo
   ```

4. 配置`springboot-demo` 的`task`,  修改`pipeline`的内容, 增加`maven`构建,然后"保存和立即构建", 同时可以查看`Console Output`

   ```bash
   node {
      def mvnHome
      stage('Preparation') {
         git 'https://github.com/itcrazy2016/springboot-demo.git'
      }
      
      stage('Maven Build') { 
         sh "mvn clean package"
      }
   }
   ```

   

5.  来到`jenkins` 所在的服务器

   ```bash
   ls /root/.jenkins/workspace/springboot-demo
   ```

至此,我们已经可以通过在`jenkins`上手动构建的方式,拿到`github` 上的代码,并且使用`maven` 进行构建了. 



### 1.2.2  `git push`触发`jenkins` 自动构建

> 最好的话,当用户 进行`git  commit/push` 提交代码到`github`的时候,能够通知`jenkins` 自动构建. 



1.  在`github` 上 配置`jenkins`的`webhook` 地址

   ```bash
   http://ip:8080/springboot-demo
   ```

    这个地址一定是要`github` 能够访问到的地址

2. 生成 `Personal access tokens`

   `jenkins`  访问`github` 需要授权,所以在`github` 上生成`token`  交给`jenkins` 使用 ，即`Personal access tokens`

   `github`的`Settings`[个人信息右上角]-->`Developer settings`-->`Personal access tokens`-->`Generate new token`

   最后保存好该`token`，比如:**72f048b514e95d6fe36f86d84374f2dcce402b43

3. `jenkins`  安装插件

   - 安装github plugin插件:[系统管理]->[插件管理]->[可选插件]
   - 安装gitlab插件和gitlab hook插件:[系统管理]->[插件管理]->[可选插件]

4. 配置`Github Server`

    [系统管理]->[系统配置]->[找到github服务器]->[添加github服务器]



## 3.  核心实战走起



### 3.1 `build&push` 镜像

来到`jenkins` 的`workspace`目录，  

```bash
cd /root/.jenkins/workspace
```

1.  准备一个文件,名称为 `springboot-demo-build-image.sh`

   ```bash
   mkdir /root/.jenkins/workspace/scripts/
   
   vi /root/.jenkins/workspace/scripts/springboot-demo-build-image.sh
   ```

2. 编写 `springboot-demo-build-image.sh` 文件

   ```bash
   # 进入到springboot-demo目录
   cd ../springboot-demo
   
   # 编写Dockerfile文件
   
   cat <<EOF > Dockerfile
   FROM openjdk:8-jre-alpine
   COPY target/springboot-demo-0.0.1-SNAPSHOT.jar /springboot-demo.jar
   ENTRYPOINT ["java","-jar","/springboot-demo.jar"]
   EOF
   
   echo "Dockerfile created successfully!"
   
   # 基于指定目录下的Dockerfile构建镜像
   docker build -t registry.cn-hangzhou.aliyuncs.com/luyanan/springboot-demo:v1.0 .
   
   # push镜像，这边需要阿里云镜像仓库登录，在w2上登录
   docker push registry.cn-hangzhou.aliyuncs.com/luyanan/springboot-demo:v1.0
   ```

3. 增加`pipeline`

   ```bash
   node {
      def mvnHome
      stage('Preparation') {
         git 'https://github.com/itcrazy2016/springboot-demo.git'
      }
      
      stage('Maven Build') { 
         sh "mvn clean package"
      }
      
      stage('Build Image') { 
         sh "/root/.jenkins/workspace/scripts/springboot-demo-build-image.sh"
      }
   }
   ```

4. 采坑

   ```bash
   # 01 文件权限
   /root/.jenkins/workspace/springboot-demo@tmp/durable-7dbf7e73/script.sh: line 1: /root/.jenkins/workspace/scripts/springboot-demo-build-image.sh: Permission denied
   # 解决
   chmod +x /root/.jenkins/workspace/scripts/springboot-demo-build-image.sh
   
   
   # 02 docker没有运行
   Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
   # 解决
   systemctl start docker
   systemctl enable docker
   
   
   # 03 push权限
   docker login --username=luyanan0718@163.com registry.cn-hangzhou.aliyuncs.com
   ```

   

### 3.2  kubernetes 拉取镜像并运行

1. 编写`springboot-demo.yaml` 文件

   ```yaml
   # 以Deployment部署Pod
   apiVersion: apps/v1
   kind: Deployment
   metadata: 
     name: springboot-demo
   spec: 
     selector: 
       matchLabels: 
         app: springboot-demo
     replicas: 1
     template: 
       metadata:
         labels: 
           app: springboot-demo
       spec: 
         containers: 
         - name: springboot-demo
           image: registry.cn-hangzhou.aliyuncs.com/luyanan/springboot-demo:v1.0
           ports: 
           - containerPort: 8080
   ---
   # 创建Pod的Service
   apiVersion: v1
   kind: Service
   metadata: 
     name: springboot-demo
   spec: 
     ports: 
     - port: 80
       protocol: TCP
       targetPort: 8080
     selector: 
       app: springboot-demo
   ---
   # 创建Ingress，定义访问规则
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata: 
     name: springboot-demo
   spec: 
     rules: 
     - host: springboot.jack.com
       http: 
         paths: 
         - path: /
           backend: 
             serviceName: springboot-demo
             servicePort: 80
   ```

2. 编写`k8s-deploy-springboot-demo.sh` 文件

   `vi /root/.jenkins/workspace/scripts/k8s-deploy-springboot-demo.sh`

   ```bash
   kubectl delete -f springboot-demo.yaml
   
   kubectl apply -f /root/.jenkins/workspace/scripts/springboot-demo.yaml
   
   echo "k8s deploy success!"
   ```

3. 编写`pipeline`

   ```bash
   node {
      def mvnHome
      stage('Preparation') {
         git 'https://github.com/itcrazy2016/springboot-demo.git'
      }
      
      stage('Maven Build') { 
         sh "mvn clean package"
      }
      
      stage('Build Image') { 
         sh "/root/.jenkins/workspace/scripts/springboot-demo-build-image.sh"
      }
      
      stage('K8S Deploy') { 
         sh "/root/.jenkins/workspace/scripts/k8s-deploy-springboot-demo.sh"
      }
   }
   ```

4. 采坑

   ```shell
   # 01 权限
   /root/.jenkins/workspace/springboot-demo@tmp/durable-8404142a/script.sh: line 1: /root/.jenkins/workspace/scripts/k8s-deploy-springboot-demo.sh: Permission denied
   # 解决
   chmod +x /root/.jenkins/workspace/scripts/k8s-deploy-springboot-demo.sh
   
   
   # 02 worker02执行不了kubectl
   切换到master上，cd ~  --->  cat .kube/config  --->复制内容
   切换到worker02上   cd ~  --->  vi .kube/config   --->粘贴内容
   ```