# kubernetes 实战

##  1. 部署 `wordpress+mysql`

### 1.1 创建`命名空间`

```shell
kubectl create namespace wordpress

```



### 1.2  创建`wordpress-db.yaml` 文件

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: mysql-deploy
  namespace: wordpress
  labels:
    app: mysql
spec:
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.6  
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3306
          name: dbport
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: rootPassW0rd
        - name: MYSQL_DATABASE
          value: wordpress
        - name: MYSQL_USER
          value: wordpress
        - name: MYSQL_PASSWORD
          value: wordpress
        volumeMounts:
        - name: db
          mountPath: /var/lib/mysql
      volumes:
      - name: db
        hostPath:
          path: /var/lib/mysql
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: wordpress
spec:
  selector:
    app: mysql
  ports:
  - name: mysqlport
    protocol: TCP
    port: 3306
    targetPort: dbport

```



###  1.3   根据`wordpress-db.yaml` 创建资源(mysql数据库)

```bash
kubectl apply -f wordpress-db.yaml
```



```bash
[root@master-kubeadm-k8s wordpress]# kubectl get pods -n wordpress
NAME                            READY   STATUS    RESTARTS   AGE
mysql-deploy-78cd6964bd-gl8td   1/1     Running   0          22s

```



查看ip

```bash
[root@master-kubeadm-k8s wordpress]# kubectl get svc mysql -n wordpress
NAME    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
mysql   ClusterIP   10.100.181.153   <none>        3306/TCP   93s

```



### 1.4  创建 `wordpress.yaml` 文件

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: wordpress-deploy
  namespace: wordpress
  labels:
    app: wordpress
spec:
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: wdport
        env:
        - name: WORDPRESS_DB_HOST
          value: 10.100.181.153:3306                     
        - name: WORDPRESS_DB_USER
          value: wordpress
        - name: WORDPRESS_DB_PASSWORD
          value: wordpress
---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: wordpress
spec:
  type: NodePort
  selector:
    app: wordpress
  ports:
  - name: wordpressport
    protocol: TCP
    port: 80
    targetPort: wdport
```



### 1.5  根据`wordpress.yaml` 创建资源

```bash
 #修改其中mysql的ip地址,其实也可以使用service的name:mysql
kubectl apply -f wordpress.yaml

[root@master-kubeadm-k8s wordpress]# kubectl get pods -n wordpress 
NAME                                READY   STATUS              RESTARTS   AGE
mysql-deploy-78cd6964bd-gl8td       1/1     Running             0          3m10s
wordpress-deploy-59c77d4968-2rsvh   0/1     ContainerCreating   0          10s


kubectl get svc -n wordpress   # 获取到转发后的端口，如30063

[root@master-kubeadm-k8s wordpress]# kubectl get svc -n wordpress
NAME        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
mysql       ClusterIP   10.100.181.153   <none>        3306/TCP       3m25s
wordpress   NodePort    10.100.36.78     <none>        80:30817/TCP   25s

```



### 1.6  访问测试

`win` 上访问任意宿主机节点的`ip:30817`

![image-20200429091417323](http://files.luyanan.com//img/20200429091441.png)

## 2. 部署`SpringBoot 项目`

### 2.1 流程

确定服务-->编写Dockerfile制作镜像-->上传镜像到仓库-->编写K8S文件-->创建



### 2.2  准备项目

这里准备一个项目`springboot-demo`,提供一个接口

```java
@RestController
public class K8SController {
    @RequestMapping("/k8s")
    public String k8s(){
        return "hello K8s!";
    }
}
```



### 2.3 项目打包

创建目录

```bash
mkdir springboot-demo
```



执行`mvn clean package` ,  生成`jar` 文件, 并上传到`springboot-demo` 目录



### 2.4 编写`Dockerfile` 文件

`vi Dockerfile`

```dockerfile
FROM openjdk:8-jre-alpine
COPY springboot-demo-0.0.1-SNAPSHOT.jar /springboot-demo.jar
ENTRYPOINT ["java","-jar","/springboot-demo.jar"]
```



### 2.5  根据`Dockerfile` 创建`image`

```bash
docker build -t springboot-demo-image .
```

### 2.6  使用`docker run` 创建`container`

```bash
docker run -d --name s1 springboot-demo-image
```

### 2.7  访问测试

```bash
docker inspect s1
curl ip:8080/k8s
```



### 2.8  将镜像推送到镜像仓库

登陆阿里云仓库

```bash
docker login --username=lu914596513 registry.cn-hangzhou.aliyuncs.com
```



打`tag`

```bash
docker tag springboot-demo-image registry.cn-hangzhou.aliyuncs.com/luyanan/springboot-demo-image:v1.0
```

推送

```bash
docker push registry.cn-hangzhou.aliyuncs.com/luyanan/springboot-demo-image:v1.0
```



### 2.9 编写k8s 配置文件

`vi springboot-demo.yaml`

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
        image: registry.cn-hangzhou.aliyuncs.com/itcrazy2016/springboot-demo-image:v1.0
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
# 创建Ingress，定义访问规则，一定要记得提前创建好nginx ingress controller
apiVersion: extensions/v1beta1
kind: Ingress
metadata: 
  name: springboot-demo
spec: 
  rules: 
  - host: k8s.demo.gper.club
    http: 
      paths: 
      - path: /
        backend: 
          serviceName: springboot-demo
          servicePort: 80
```



```bash
 kubectl apply -f springboot-demo.yaml
```



### 2.10  查看资源

```bash
kubectl get pods
```



```bash
kubectl get pods -o wide
```

```bash
curl pod_id:8080/k8s
```

```bash
kubectl get svc
```

```bash
kubectl scale deploy springboot-demo --replicas=5
```



### 2.11  `win` 配置`hosts` 文件(一定要提前创建好`nginx ingress controller`)

```
192.168.0.61 springboot.luyanan.com
```

### 2.12 浏览器访问

浏览器访问 `http://springboot.luyanan.com/k8s`



## 3. 部署分布式Nacos项目

### 3.1  传统方式

#### 3.1.1  项目准备

准备两个`SpringBoot` 项目,名称为`order`和`user`, 表示两个服务



#### 3.1.2 Nacos  下载部署`Nacos server1.0.0`

`github`：<https://github.com/alibaba/nacos/releases>

1. `tar zxvf ` 解压
2. 进入到`bin`  目录, `sh startup.sh -m standalone `  [需要有java 环境的支持]
3. 浏览器访问 `ip:8848/nacos`
4. 用户名: `nacos/nacos`

#### 3.1.3  将应用注册到nacos

将应用注册到`nacos`, 记得修改`SpringBoot` 项目中的`application.yml` 文件

将`user/order` 服务注册到`nacos`, 

使得`user` 服务能够找到`order`服务



#### 3.1.4   启动两个项目,然后查看`nacos` 的服务列表

#### 3.1.5  验证

为了验证`user` 能够发现`order` 的地址,访问`localhost:8080/user/test`, 查看日志输出,从而测试是否可以`ping` 通`order`的地址



### 3.2 k8s 方式

 `user` 和`order` 是k8s的`pod`

如果将`user`和`order` 都迁移到k8s中, 那服务注册和发现会有什么问题吗? 

#### 3.2.1  打包上传

生成`jar` 文件, 并上传到`master`节点的`user`和`order` 目录下



#### 3.2.2  编写`Dockerfile` 文件

在对应的目录, 编写`Dockerfile` 文件

```dockerfile
FROM openjdk:8-jre-alpine
COPY user-0.0.1-SNAPSHOT.jar /user.jar
ENTRYPOINT ["java","-jar","/user.jar"]
```

```dockerfile
FROM openjdk:8-jre-alpine
COPY order-0.0.1-SNAPSHOT.jar /order.jar
ENTRYPOINT ["java","-jar","/order.jar"]
```

#### 3.2.3  `image` 创建

分别在对应的目录下, 根据`Dockerfile` 创建`image`


```xshell
docker build -t user-image:v1.0 .
docker build -t order-image:v1.0 .
```



#### 3.2.4  将镜像推送至阿里云仓库

登陆阿里云仓库

```bash
docker login --username=lu914596513 registry.cn-hangzhou.aliyuncs.com
```



打`tag`

```bash
docker tag springboot-demo-image registry.cn-hangzhou.aliyuncs.com/luyanan/user-image:v1.0
```

推送

```bash
docker push registry.cn-hangzhou.aliyuncs.com/luyanan/user-image:v1.0
```

#### 3.2.5  编写kubernetes  配置文件

`vi user.yaml/order.yaml`

```yaml
# 以Deployment部署Pod
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: user
spec: 
  selector: 
    matchLabels: 
      app: user
  replicas: 1
  template: 
    metadata:
      labels: 
        app: user
    spec: 
      containers: 
      - name: user
        image: registry.cn-hangzhou.aliyuncs.com/itcrazy2016/user-image:v1.0
        ports: 
        - containerPort: 8080
---
# 创建Pod的Service
apiVersion: v1
kind: Service
metadata: 
  name: user
spec: 
  ports: 
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector: 
    app: user
---
# 创建Ingress，定义访问规则，一定要记得提前创建好nginx ingress controller
apiVersion: extensions/v1beta1
kind: Ingress
metadata: 
  name: user
spec: 
  rules: 
  - host: k8s.demo.gper.club
    http: 
      paths: 
      - path: /
        backend: 
          serviceName: user
          servicePort: 80
```

```yaml
# 以Deployment部署Pod
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: order
spec: 
  selector: 
    matchLabels: 
      app: order
  replicas: 1
  template: 
    metadata:
      labels: 
        app: order
    spec: 
      containers: 
      - name: order
        image: registry.cn-hangzhou.aliyuncs.com/itcrazy2016/order-image:v1.0
        ports: 
        - containerPort: 9090
---
# 创建Pod的Service
apiVersion: v1
kind: Service
metadata: 
  name: order
spec: 
  ports: 
  - port: 80
    protocol: TCP
    targetPort: 9090
  selector: 
    app: order
```

```bash
kubectl apply -f user.yaml/order.yaml
```

 启动 



#### 3.2.6  查看资源

```shell
kubectl get pods
kubectl get pods -o wide
kubectl get svc
kubectl get ingress

```



#### 3.2.7 查看nacos 上的服务信息

可以发现,注册到nacos 上的服务ip地址为`pod`的ip, 比如192.168.80.206/192.168.190.82



#### 3.2.8 访问测试

```shell
# 01 集群内
curl user-pod-ip:8080/user/test
kubectl logs -f <pod-name> -c <container-name>   [主要是为了看日志输出，证明user能否访问order]

```

**结论:** 如果服务都是在k8s集群中,最终将`pod ip`注册到了`nacos` ,那么最终服务同通过`pod ip`发现. 



####  3.2.9 user传统和order迁移K8s

假如user现在不在K8s集群中，order在K8s集群中

比如user使用本地idea中的，order使用上面K8s中的

1. 启动本地idea中的user服务
2. 查看nacos server中的user服务列表
3. 访问本地的localhost:8080/user/test，并且观察idea中的日志打印，发现访问的是order的pod id，此时肯定是不能进行服务调用的，怎么解决呢？
4. 解决思路

> ```text
> 之所以访问不了，是因为order的pod ip在外界访问不了，怎么解决呢？
> 01 可以将pod启动时所在的宿主机的ip写到容器中，也就是pod id和宿主机ip有一个对应关系
> 02 pod和宿主机使用host网络模式，也就是pod直接用宿主机的ip，但是如果服务高可用会有端口冲突问题[可以使用pod的调度策略，尽可能在高可用的情况下，不会将pod调度在同一个worker中]我们来演示一个host网络模式的方式，修改order.yaml文件
> ```

5. 修改之后apply之前可以看一下各个节点的9090端口是否被占用

` lsof -i tcp:9090`

```yaml
 ...
 metadata:
      labels: 
        app: order
    spec: 
    # 主要是加上这句话，注意在order.yaml的位置
      hostNetwork: true
      containers: 
      - name: order
        image: registry.cn-hangzhou
...
```

6. kubectl apply -f order.yaml 

- kubectl get pods -o wide   --->找到pod运行在哪个机器上，比如w2

 - 查看w2上的9090端口是否启动

7. 查看nacos server上order服务

 可以发现此时用的是w2宿主机的9090端口

8. 本地idea访问测试

>  localhost:8080/user/test