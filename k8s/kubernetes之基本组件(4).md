# Kunernetes 基本组件

## 1. `Controllers`

> 官网:<https://kubernetes.io/docs/concepts/workloads/controllers/>

### 1.1  ReplicationController(RC)

#### 1.1  介绍

官网: https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/

```text
A ReplicationController ensures that a specified number of pod replicas are running at any one time. In other words, a ReplicationController makes sure that a pod or a homogeneous set of pods is always up and available.
```



`ReplicationController` 定义了一个期望的场景,即声明某种`pod` 的副本数量在任何适合都符合某个预期值,所以`RC` 的定义包含以下几个部分:

- `Pod` 期待的副本数(`relicas`)
- 用于筛选目标`pod` 的`Lable Selector`
- 当`pod` 的副本数量小于预期数量的时候,用于创建新`pod` 的`pod` 模板(`tamplate`)

也就是说,通过`RC` 实现了集群中`Pod` 的高可用,减少了传统IT环境中手工运维的工作. 



#### 1.2  验证

- `kind`:  表示要新建对象的类型
- `spec:selector`: 表示需要管理的`pod`的`lable`, 这里表示包含`app:nginx` 的`label` 的`pod` 都会被改`RC` 管理
- `spec:replicas`: 表示受此`RC` 管理的`pod` 需要运行的副本数
- `spec:template`: 表示用于定义`pod` 的模板, 比如`pod`的名称, 拥有的`Lable`, 以及`pod` 中运行的应用等 ,通过改变`RC` 里`Pod` 模板中的镜像版本,可以实现`Pod` 的升级功能. 
- `kubectl apply -f nginx-pod.yaml` , 此时k8s 会在所有可用的`Node` 上, 创建3个`Pod`, 并且每个`Pod` 都会有一个`app:nginx`的`label`, 同时每个`Pod` 中都运行了一个`nginx` 容器. 

如果某个`pod` 发生问题, `controller manage`  能够即时发现,然后根据`RC` 的定义, 创建一个新的`Pod`

扩缩容: `kubectl scale rc nginx --replicas=5`



1. 创建一个名为`nginx_replication.yaml`的文件

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

2.   根据`nginx_replication.yaml`  创建`pod`

   ```bash
   kubectl apply -f nginx_replication.yaml
   ```

3. 查看`pod`

    ```text
   [root@master-kubeadm-k8s test]# kubectl get pods -o wide
   NAME          READY   STATUS    RESTARTS   AGE   IP               NODE   NOMINATED NODE   READINESS GATES
   nginx-9848t   1/1     Running   0          24s   192.168.80.196   w2     <none>           <none>
   nginx-nfsxf   1/1     Running   0          24s   192.168.80.197   w2     <none>           <none>
   nginx-sndjl   1/1     Running   0          24s   192.168.190.69   w1     <none>           <none>
   
   ```
   
4. 尝试删除一个`Pod`

   ```bash
   kubectl delete pods nginx-9848t
   kubectl get pods
   [root@master-kubeadm-k8s test]# kubectl get pods 
   NAME          READY   STATUS    RESTARTS   AGE
   nginx-4zrp4   1/1     Running   0          25s
   nginx-nfsxf   1/1     Running   0          76s
   nginx-sndjl   1/1     Running   0          76s
   ```

5. 对`pod` 进行扩容

   ```bash
   kubectl scale rc nginx --replicas=5
   [root@master-kubeadm-k8s test]# kubectl get pods 
   NAME          READY   STATUS              RESTARTS   AGE
   nginx-4zrp4   1/1     Running             0          52s
   nginx-72fxq   0/1     ContainerCreating   0          9s
   nginx-b2dhw   0/1     ContainerCreating   0          9s
   nginx-nfsxf   1/1     Running             0          103s
   nginx-sndjl   1/1     Running             0          103s
   ```

6. 删除`pod`

   ```bash
   kubectl delete -f nginx_replication.yaml
   ```



### 1.2 `RelicaSet(RS)`

#### 1.2.1  介绍

> 官网:https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/

```text
A ReplicaSet’s purpose is to maintain a stable set of replica Pods running at any given time. As such, it is often used to guarantee the availability of a specified number of identical Pods
```



在`kubernetes v1.2`的时候, `RC` 就升级成了另外一个概念:`RelicaSet`, 官方解释为"下一代RC" 

`RelicaSet` 与`RC` 没有本质上的区别,`kubectl` 中绝大部分作用于`RC`的命令同样适用于`RS`

`RS` 与`RC`的唯一区别是: `RS` 支持基于集合的`Lable Selector`(`Set-based selector`), 而`RC` 只支持基于等式的`Label Selector（equality-based selector）`, 这使得`Relica Set`的功能更强. 

#### 1.2.2 验证一下

```yaml
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  name: frontend
spec:
  matchLabels: 
    tier: frontend
  matchExpressions: 
    - {key:tier,operator: In,values: [frontend]}
  template:
  ...
```

`注意:` 一般情况下,我们很少单独使用`Relica Set`, 他主要是被`Deployment` 这个更高的资源对象所使用, 从而形成一整套`pod` 创建、删除、更新的编排机制. 当我们使用`Deployment` 时, 无须关心他是如何创建和维护`RelicaSet` 的, 这一切都是自动发生的. 同时, 无需担心跟其他机制的不兼容问题(比如`RelicaSet` 不支持`rolling-update`, 但是`Deployment` 支持)

### 1.3  `Deployment`

#### 1.3.1 介绍

> 官网: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

```text
A Deployment provides declarative updates for Pods and ReplicaSets.

You describe a desired state in a Deployment, and the Deployment Controller changes the actual state to the desired state at a controlled rate. You can define Deployments to create new ReplicaSets, or to remove existing Deployments and adopt all their resources with new Deployments.
```

`Deployment` 相对`RC` 最大的一个升级就是我们可以随时知道当前`pod` 的`部署`的进度. 

创建一个`Deployment` 对象来生成对应的`Replica Set` 并完成`pod` 副本的创建过程. 

检查`Deployment` 的状态来看部署的动作是否完成(`Pod` 副本的数量是否达到预期)

####  1.3.2 测试

##### 1.3.2.1 创建`nginx_deployment.yaml`  文件

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```



##### 1.3.2.2 根据`nginx_deployment.yaml` 文件创建`Pod`

```bash
kubectl apply -f nginx_deployment.yaml
```



#####  1.3.2.3 查看`Pod`

**`kubectl get pods -o wide`**

```bash
[root@master-kubeadm-k8s test]# kubectl get pods -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP               NODE   NOMINATED NODE   READINESS GATES
nginx-deployment-6dd86d77d-46qjm   1/1     Running   0          75s   192.168.190.72   w1     <none>           <none>
nginx-deployment-6dd86d77d-7mgsf   1/1     Running   0          75s   192.168.80.199   w2     <none>           <none>
nginx-deployment-6dd86d77d-hzkbh   1/1     Running   0          75s   192.168.80.200   w2     <none>           <none>

```



**`kubectl get deployment`**



```bash
[root@master-kubeadm-k8s test]# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           91s

```



**`kubectl get rs`**

```bash
[root@master-kubeadm-k8s test]# kubectl get rs
NAME                         DESIRED   CURRENT   READY   AGE
nginx-deployment-6dd86d77d   3         3         3       104s

```



**`kubectl get deployment -o wide`**

```bash
[root@master-kubeadm-k8s test]# kubectl get deployment -o wide
NAME               READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES        SELECTOR
nginx-deployment   3/3     3            3           2m4s   nginx        nginx:1.7.9   app=nginx

```



##### 1.3.2.4 查看当前`nginx`的版本

```bash
kubectl get deployment -o wide

```





##### 1.3.2.5  更新`nginx`的`image` 版本

```bash
kubectl set image deployment nginx-deployment nginx=nginx:1.9.1
```

查看结果

```bash
[root@master-kubeadm-k8s test]# kubectl get deployment -o wide
NAME               READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES        SELECTOR
nginx-deployment   3/3     1            3           2m49s   nginx        nginx:1.9.1   app=nginx

```



##  2. `Labels`  和`Selector`

在前面的`yaml`  文件中, 看到很多`label`, 顾名思义,就是给一些资源打上标签的

> 官网:https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/

```text
Labels are key/value pairs that are attached to objects, such as pods. 
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
```

表示名称为`nginx-pod` 的`pod`, 有一个`label` ,`key` 为`app`,`value` 为`nginx`

我们可以将具有同一个`label`的`pod`, 交给`selector`管理. 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:             # 匹配具有同一个label属性的pod标签
    matchLabels:
      app: nginx         
  template:             # 定义pod的模板
    metadata:
      labels:
        app: nginx      # 定义当前pod的label属性，app为key，value为nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```



查看`pod` 的`label` 标签: `kubectl get pods --show-labels`



## 3. `Namespace`

`kubectl get pods` 

```bash
[root@master-kubeadm-k8s test]# kubectl get pods
No resources found.
```

`kubectl get pods -n kube-system`

```bash
[root@master-kubeadm-k8s test]# kubectl get pods -n kube-system
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-99f457df4-z5mx6   1/1     Running   0          3h13m
calico-node-5qrw9                         1/1     Running   0          3h13m
calico-node-hb76r                         1/1     Running   0          3h13m
calico-node-sdp7w                         1/1     Running   0          3h13m
coredns-fb8b8dccf-hlcft                   1/1     Running   0          3h20m
coredns-fb8b8dccf-jqvht                   1/1     Running   0          3h20m
etcd-m                                    1/1     Running   0          3h20m
kube-apiserver-m                          1/1     Running   0          3h19m
kube-controller-manager-m                 1/1     Running   0          3h20m
kube-proxy-hkvhk                          1/1     Running   0          3h19m
kube-proxy-rhwtf                          1/1     Running   0          3h19m
kube-proxy-xmv42                          1/1     Running   0          3h20m
kube-scheduler-m                          1/1     Running   0          3h19m

```

比较一下,上述这两行命令的输入是否一样,发现不一样，是因为`Pod` 属于不同的`namespace`

> 查看当前的命令空间 `kubectl get namespaces/ns`

```bash
[root@master-kubeadm-k8s test]# kubectl get ns
NAME              STATUS   AGE
default           Active   3h21m
kube-node-lease   Active   3h21m
kube-public       Active   3h21m
kube-system       Active   3h21m
```



说白了,命令空间就是为了隔离不同的资源, 比如`Pod`,`Service`、`Deployment`等,可以在输入命令的时候执行命令空间`-n`, 如果不指定,则使用默认的命名空间`default`

### 3.1 创建命令空间

创建`myns-namespace.yaml` 文件

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myns
```

执行文件

```bash
kubectl apply -f myns-namespace.yaml
```

执行`kubectl get namespaces/ns` 查看结果

```bash
[root@master-kubeadm-k8s test]# kubectl get ns
NAME              STATUS   AGE
default           Active   3h22m
kube-node-lease   Active   3h22m
kube-public       Active   3h22m
kube-system       Active   3h22m
myns              Active   20s
```

### 3.2 指定命令空间下的资源

比如创建一个`Pod`, 属于`myns` 命令空间下

`vi nginx-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: myns
spec:
  containers:
  - name: nginx-container
    image: nginx
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f nginx-pod.yaml 

    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-pod
      namespace: myns
    spec:
      containers:
      - name: nginx-container
        image: nginx
        ports:
        - containerPort: 80

```



> 查看`myns` 命名空间下的`Pod` 和资源

```bash
kubectl get pods
[root@master-kubeadm-k8s test]# kubectl get pods
No resources found.
kubectl get pods -n myns
[root@master-kubeadm-k8s test]# kubectl get pods -n myns
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          36s
kubectl get all -n myns
[root@master-kubeadm-k8s test]# kubectl  get all -n myns
NAME            READY   STATUS    RESTARTS   AGE
pod/nginx-pod   1/1     Running   0          61s

kubectl get pods --all-namespaces    #查找所有命名空间下的pod
[root@master-kubeadm-k8s test]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-99f457df4-z5mx6   1/1     Running   0          3h18m
kube-system   calico-node-5qrw9                         1/1     Running   0          3h18m
kube-system   calico-node-hb76r                         1/1     Running   0          3h18m
kube-system   calico-node-sdp7w                         1/1     Running   0          3h18m
kube-system   coredns-fb8b8dccf-hlcft                   1/1     Running   0          3h25m
kube-system   coredns-fb8b8dccf-jqvht                   1/1     Running   0          3h25m
kube-system   etcd-m                                    1/1     Running   0          3h24m
kube-system   kube-apiserver-m                          1/1     Running   0          3h24m
kube-system   kube-controller-manager-m                 1/1     Running   0          3h24m
kube-system   kube-proxy-hkvhk                          1/1     Running   0          3h24m
kube-system   kube-proxy-rhwtf                          1/1     Running   0          3h24m
kube-system   kube-proxy-xmv42                          1/1     Running   0          3h25m
kube-system   kube-scheduler-m                          1/1     Running   0          3h24m
myns          nginx-pod                                 1/1     Running   0          109s

```



## 4. `Network`

### 4.1  同一个`Pod` 中的容器通信

接下来就要说到跟`Kubernetes` 网络通信相关的内容

我们知道,k8s 最小的操作单位是`Pod`, 先思考以下同一个`Pod` 中多个容器要进行通信. 

由官网的这段话可以看出,同一个`Pod` 中的容器是共享网络`IP`地址和端口号的,通信显然是没问题的. 

```text
Each Pod is assigned a unique IP address. Every container in a Pod shares the network namespace, including the IP address and network ports. 
```

那如果是通过容器的名称进行通信呢? 就需要将所有`Pod`  中的容器加入到同一个容器的网络中,我们把容器称作为`pod` 中的`pause contailer`. 



### 4.2 集群中`Pod` 之间的通信

接下来就聊聊k8s 最小的操作单元,`Pod` 之间的通信. 

我们知道每个`Pod` 之间都会有独立的地址,这个`IP` 地址是被`Pod` 中所有的`contailer` 共享的. 

那多个`Pod` 之间的通信能通过这个`IP`地址吗? 

我认为需要分为两个维度, 一个是集群中同一台集群中的`Pod`, 二是集群中不同机器中的`POd`

准备两个`POd`, 一个`nginx`, 一个是`busybox`

`nginx_pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
name: nginx-pod
labels:
 app: nginx
spec:
containers:
  - name: nginx-container
    image: nginx
    ports:
    - containerPort: 80
```



`busybox_pod.yaml`

> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: busybox
>   labels:
>      app: busybox
> spec:
>   containers:
>     - name: busybox
>       image: busybox
>       command: ['sh', '-c', 'echo The app is running! && sleep 3600']
> ```



将两个`Pod` 运行起来,并且查看运行情况. 

```bash
kubectl apply -f nginx_pod.yaml
```

```bash
kubectl apply -f busybox_pod.yaml
```



```bash
kubectl get pods -o wide
```

```text
[root@master-kubeadm-k8s test]# kubectl get pods -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP               NODE   NOMINATED NODE   READINESS GATES
busybox     1/1     Running   0          35s   192.168.80.202   w2     <none>           <none>
nginx-pod   1/1     Running   0          36m   192.168.190.74   w1     <none>           <none>

```



`发现`：nginx-pod的ip为192.168.190.74     busybox-pod的ip为192.168.80.202



####  4.2.1   同一个集群中同一台机器

1. 来到worker01：`ping 192.168.190.74` 

   ```bash
   
   ```

2. 来到worker01：curl 192.168.190.74

   ```html
   [root@worker01-kubeadm-k8s k8s]# curl 192.168.190.74
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   <style>
       body {
           width: 35em;
           margin: 0 auto;
           font-family: Tahoma, Verdana, Arial, sans-serif;
       }
   </style>
</head>
   <body>
   <h1>Welcome to nginx!</h1>
   <p>If you see this page, the nginx web server is successfully installed and
   working. Further configuration is required.</p>
   
   <p>For online documentation and support please refer to
   <a href="http://nginx.org/">nginx.org</a>.<br/>
   Commercial support is available at
   <a href="http://nginx.com/">nginx.com</a>.</p>
   
   <p><em>Thank you for using nginx.</em></p>
   </body>
   </html>
   
   ```
   
   



#### 4.2.2   同一个集群中不同机器

1. 来到worker02：ping 192.168.190.74

   ```bash
   [root@worker02-kubeadm-k8s k8s]# ping 192.168.190.74
   PING 192.168.190.74 (192.168.190.74) 56(84) bytes of data.
   64 bytes from 192.168.190.74: icmp_seq=1 ttl=63 time=0.805 ms
   64 bytes from 192.168.190.74: icmp_seq=2 ttl=63 time=0.583 ms
   64 bytes from 192.168.190.74: icmp_seq=3 ttl=63 time=0.413 ms
   64 bytes from 192.168.190.74: icmp_seq=4 ttl=63 time=0.567 ms
   64 bytes from 192.168.190.74: icmp_seq=5 ttl=63 time=0.481 ms
   
   ```

2. 来到`worker02`：`curl 192.168.190.74，同样可以访问`nginx`

3. 来到`master`

   `ping/curl 192.168.190.74          访问的是`worker01`上的`nginx-pod`

   `ping          192.168.80.202`     访问的是`worker02`上的`busybox-pod`
   
4. 来到`worker01`：`ping 192.168.80.202         访问的是`worker02`上的`busybox-pod`

    How to implement the Kubernetes Cluster networking model--Calico

   `官网`：<https://kubernetes.io/docs/concepts/cluster-administration/networking/#the-kubernetes-network-model>

   

    -  pods on a node can communicate with all pods on all nodes without NAT
    -  agents on a node (e.g. system daemons, kubelet) can communicate with all pods on that node
    -  pods in the host network of a node can communicate with all pods on all nodes without NAT



### 4.3  集群内`Service-Cluster IP`

对于上述的`Pod` 虽然实现了集群内部互相通信,但是`Pod` 是不稳定的,比如通过`Deployment`  管理`Pod`, 随时可能对`Pod` 进行扩缩容,这时候`Pod`的`ip` 地址是变化的,能够有一个固定的`IP`, 使得集群内能够访问,也就是之前在架构描述中所提到的, 能够把相同的或者具有关联的`Pod` ,打上`Label`, 组成`Service`, 而`Service` 有固定的`Ip`, 不管`Pod` 怎么创建和销毁,都可以通过`Service`的`IP` 进行访问. 

`Service官网`：<https://kubernetes.io/docs/concepts/services-networking/service/>

```text
An abstract way to expose an application running on a set of Pods as a network service.
With Kubernetes you don’t need to modify your application to use an unfamiliar service discovery mechanism. Kubernetes gives Pods their own IP addresses and a single DNS name for a set of Pods, and can load-balance across them.
```



1.  创建 `whoami-deployment.yaml` 文件, 并且`apply`

    ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: whoami-deployment
     labels:
       app: whoami
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: whoami
     template:
       metadata:
         labels:
           app: whoami
       spec:
         containers:
         - name: whoami
           image: jwilder/whoami
           ports:
           - containerPort: 8000
   ```

2. 查看`Pod` 以及`Service`

   ```bash
   [root@master-kubeadm-k8s test]# kubectl get pods -o wide
   NAME                                 READY   STATUS    RESTARTS   AGE   IP               NODE   NOMINATED NODE   READINESS GATES
   whoami-deployment-678b64444d-bsnt6   1/1     Running   0          52s   192.168.190.75   w1     <none>           <none>
   whoami-deployment-678b64444d-lbzp2   1/1     Running   0          52s   192.168.190.76   w1     <none>           <none>
   whoami-deployment-678b64444d-s2r6b   1/1     Running   0          52s   192.168.80.203   w2     <none>           <none>
   
   ```

   `kubect get svc`:可以发现目前并没有关于`whoami`的`service`

   ```bash
   [root@master-kubeadm-k8s test]# kubectl get svc
   NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
   kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   4h22m
   
   ```

3. 在集群中正常访问

   ```bash
   curl 192.168.190.75:8000/192.168.190.76:8000/192.168.80.203:8000
   ```

4. 创建`whoami`的`service`

   > 注意: 该地址只能在集群内部访问.

   ```bash
   kubectl expose deployment whoami-deployment
   kubectl get svc    
   删除svc   kubectl delete service whoami-deployment
   
   [root@master-kubeadm-k8s test]# kubectl get svc
   NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
   kubernetes          ClusterIP   10.96.0.1        <none>        443/TCP    4h24m
   whoami-deployment   ClusterIP   10.110.144.181   <none>        8000/TCP   8s
   

   ```
   
   

​     可以发现有一个`Cluster IP` 类型的`service`,名称为`whoami-deployment`，IP地址为10.110.144.181

5. 通过`service` 的`cluster ip` 访问. 

   ```bash
   [root@master-kubeadm-k8s test]# curl 10.110.144.181:8000
   I'm whoami-deployment-678b64444d-lbzp2
   [root@master-kubeadm-k8s test]# curl 10.110.144.181:8000
   I'm whoami-deployment-678b64444d-lbzp2
   [root@master-kubeadm-k8s test]# curl 10.110.144.181:8000
   I'm whoami-deployment-678b64444d-s2r6b
   [root@master-kubeadm-k8s test]# curl 10.110.144.181:8000
   I'm whoami-deployment-678b64444d-lbzp2
   ```

6. 具体查看一下  `whoami-deployment` 的详情信息, 发现有一个`Endpoints`  连接了具体3个`Pod`

   ```bash
   [root@master-kubeadm-k8s test]# kubectl describe svc whoami-deployment
   Name:              whoami-deployment
   Namespace:         default
   Labels:            app=whoami
   Annotations:       <none>
   Selector:          app=whoami
   Type:              ClusterIP
   IP:                10.110.144.181
   Port:              <unset>  8000/TCP
   TargetPort:        8000/TCP
   Endpoints:         192.168.190.75:8000,192.168.190.76:8000,192.168.80.203:8000
   Session Affinity:  None
   Events:            <none>
   
   ```

7. 不妨对`whoami`  扩容成5个

   ```bash
   kubectl scale deployment whoami-deployment --replicas=5
   ```

8. 再次访问 : `curl 10.110.144.181:8000`

9. 再次查看`service`的具体信息: `kubectl describe svc whoami-deployment`

10. 其实对于`sertvice`的创建,不仅仅可以使用`kubectl expose`, 还可以定义一个`yaml` 文件

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-service
    spec:
      selector:
        app: MyApp
      ports:
        - protocol: TCP
          port: 80
          targetPort: 9376
      type: Cluster
    ```

    `conclusion` :  其实 `service` 存在的意思就是为了`Pod` 的不稳定性,而上述探讨的就是关于`Service`的一种类型`Cluster IP`, 只能供访问. 

    > 以`Pod` 为中心,已经讨论了关于集群内的通信方式,接下来就是探讨集群中的`Pod` 访问外部服务,以及外部服务访问集群中的`Pod`

    

### 4.4 `Pod` 访问外部服务

    比较简单,没太多好说的内容, 直接访问即可. 



### 4.5 外部服务访问集群中的`Pod`

####  `Service-NodePort`

也就是`service`的一种类型,可以通过`NodePort` 的方式.

说白了,因为外部能够访问到集群中的物理机器`IP`, 所以就是在集群中每台物理机上暴露一个相同的`IP`,比如说 32008

1.  根据`whoami-deployment.yaml` 创建`Pod`

    ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: whoami-deployment
     labels:
       app: whoami
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: whoami
     template:
       metadata:
         labels:
           app: whoami
       spec:
         containers:
         - name: whoami
           image: jwilder/whoami
           ports:
           - containerPort: 8000
   ```

   

2. 创建`NodePort` 类型的`service`, 名称为 `whoami-deployment`

   ```bash
   kubectl delete svc whoami-deployment
   
   kubectl expose deployment whoami-deployment --type=NodePort
   
   [root@master-kubeadm-k8s test]# kubectl get svc
   NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
   kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP          4h40m
   whoami-deployment   NodePort    10.111.112.95   <none>        8000:30436/TCP   7s
   
   ```

3. 注意上述的端口30436, 实际上就是暴露在集群中物理机器上的端口

   ```bash
   lsof -i tcp:30436
   netstat -ntlp|grep 30436
   ```

4. 浏览器通过物理机器的`ip` 访问

   ```bash
   http://192.168.56.10:32041
   curl 192.168.56.10:30436
   ```

   `conclusion`: `NodePort` 虽然能够实现外部访问`Pod`的需求,但是真的好吗? 其实不好,占用了各个物理主机上的端口. 





#### `Service-LoadBalance`

通常需要第三方云提供商支持, 有约束性. 

####   `Ingress`

`官网`：<https://kubernetes.io/docs/concepts/services-networking/ingress/>

```text
An API object that manages external access to the services in a cluster, typically HTTP.

Ingress can provide load balancing, SSL termination and name-based virtual hosting.
```

![image-20200424221234922](http://files.luyanan.com//img/20200424221259.png)

可以发现, `ingress` 就是帮助我们访问集群中的服务的,不过在看`Ingress` 之前,我们还是先出一个案例出来. 

跟简单, 在k8s集群中部署`tomcat`

浏览器想要访问这个`tomcat` , 也就是外部要访问`tomcat`, 用之前的`Service-NodePort` 的方式是可以的,比如暴露一个32008端口,只需要访问`192.168.56.10:32008`即可。

vi my-tomcat.yaml

kubectl apply -f my-tomcat.yaml

kubectl get pods

kubectl get deployment

kubectl get svc

```text
[root@master-kubeadm-k8s test]# kubectl get svc
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP          4h43m
tomcat-service      NodePort    10.96.199.157   <none>        80:31719/TCP     11s

```



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  labels:
    app: tomcat
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      containers:
      - name: tomcat
        image: tomcat
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
spec:
  ports:
  - port: 80   
    protocol: TCP
    targetPort: 8080
  selector:
    app: tomcat
  type: NodePort  
```

显然, `Service-NodePort` 的方式生产环境不推荐使用,那接下来就基于上述需求,使用`ingress` 实现访问`tomcat`的需求. 

`官网Ingress`:<https://kubernetes.io/docs/concepts/services-networking/ingress/>

`GitHub Ingress Nginx`:<https://github.com/kubernetes/ingress-nginx>

`Nginx Ingress Controller`:<https://kubernetes.github.io/ingress-nginx/

1. 以`Deployment` 方式创建`Pod`, 该`Pod` 为`Ingress Nginx Controller`, 要想要外部访问,可以通过`Service` 的`NodePort` 或者`HostPort` 方式,这里选择`HostPort`, 比如指定`worker01` 运行. 

   ```yaml
   # 确保nginx-controller运行到w1节点上
   kubectl label node w1 name=ingress   
   
   # 使用HostPort方式运行，需要增加配置
   hostNetwork: true
   
   # 搜索nodeSelector，并且要确保w1节点上的80和443端口没有被占用，镜像拉取需要较长的时间，这块注意一下哦
   kubectl apply -f mandatory.yaml  
   
   kubectl get all -n ingress-nginx
   ```

   `mandatory.yaml  `

   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: ingress-nginx
     labels:
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
   
   ---
   
   kind: ConfigMap
   apiVersion: v1
   metadata:
     name: nginx-configuration
     namespace: ingress-nginx
     labels:
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
   
   ---
   kind: ConfigMap
   apiVersion: v1
   metadata:
     name: tcp-services
     namespace: ingress-nginx
     labels:
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
   
   ---
   kind: ConfigMap
   apiVersion: v1
   metadata:
     name: udp-services
     namespace: ingress-nginx
     labels:
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
   
   ---
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: nginx-ingress-serviceaccount
     namespace: ingress-nginx
     labels:
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
   
   ---
   apiVersion: rbac.authorization.k8s.io/v1beta1
   kind: ClusterRole
   metadata:
     name: nginx-ingress-clusterrole
     labels:
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
   rules:
     - apiGroups:
         - ""
       resources:
         - configmaps
         - endpoints
         - nodes
         - pods
         - secrets
       verbs:
         - list
         - watch
     - apiGroups:
         - ""
       resources:
         - nodes
       verbs:
         - get
     - apiGroups:
         - ""
       resources:
         - services
       verbs:
         - get
         - list
         - watch
     - apiGroups:
         - ""
       resources:
         - events
       verbs:
         - create
         - patch
     - apiGroups:
         - "extensions"
         - "networking.k8s.io"
       resources:
         - ingresses
       verbs:
         - get
         - list
         - watch
     - apiGroups:
         - "extensions"
         - "networking.k8s.io"
       resources:
         - ingresses/status
       verbs:
         - update
   
   ---
   apiVersion: rbac.authorization.k8s.io/v1beta1
   kind: Role
   metadata:
     name: nginx-ingress-role
     namespace: ingress-nginx
     labels:
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
   rules:
     - apiGroups:
         - ""
       resources:
         - configmaps
         - pods
         - secrets
         - namespaces
       verbs:
         - get
     - apiGroups:
         - ""
       resources:
         - configmaps
       resourceNames:
         # Defaults to "<election-id>-<ingress-class>"
         # Here: "<ingress-controller-leader>-<nginx>"
         # This has to be adapted if you change either parameter
         # when launching the nginx-ingress-controller.
         - "ingress-controller-leader-nginx"
       verbs:
         - get
         - update
     - apiGroups:
         - ""
       resources:
         - configmaps
       verbs:
         - create
     - apiGroups:
         - ""
       resources:
         - endpoints
       verbs:
         - get
   
   ---
   apiVersion: rbac.authorization.k8s.io/v1beta1
   kind: RoleBinding
   metadata:
     name: nginx-ingress-role-nisa-binding
     namespace: ingress-nginx
     labels:
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: Role
     name: nginx-ingress-role
   subjects:
     - kind: ServiceAccount
       name: nginx-ingress-serviceaccount
       namespace: ingress-nginx
   
   ---
   apiVersion: rbac.authorization.k8s.io/v1beta1
   kind: ClusterRoleBinding
   metadata:
     name: nginx-ingress-clusterrole-nisa-binding
     labels:
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: nginx-ingress-clusterrole
   subjects:
     - kind: ServiceAccount
       name: nginx-ingress-serviceaccount
       namespace: ingress-nginx
   
   ---
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-ingress-controller
     namespace: ingress-nginx
     labels:
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx
   spec:
     replicas: 1
     selector:
       matchLabels:
         app.kubernetes.io/name: ingress-nginx
         app.kubernetes.io/part-of: ingress-nginx
     template:
       metadata:
         labels:
           app.kubernetes.io/name: ingress-nginx
           app.kubernetes.io/part-of: ingress-nginx
         annotations:
           prometheus.io/port: "10254"
           prometheus.io/scrape: "true"
       spec:
         # wait up to five minutes for the drain of connections
         terminationGracePeriodSeconds: 300
         serviceAccountName: nginx-ingress-serviceaccount
         hostNetwork: true
         nodeSelector:
           name: ingress
           kubernetes.io/os: linux
         containers:
           - name: nginx-ingress-controller
             image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1
             args:
               - /nginx-ingress-controller
               - --configmap=$(POD_NAMESPACE)/nginx-configuration
               - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
               - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
               - --publish-service=$(POD_NAMESPACE)/ingress-nginx
               - --annotations-prefix=nginx.ingress.kubernetes.io
             securityContext:
               allowPrivilegeEscalation: true
               capabilities:
                 drop:
                   - ALL
                 add:
                   - NET_BIND_SERVICE
               # www-data -> 33
               runAsUser: 33
             env:
               - name: POD_NAME
                 valueFrom:
                   fieldRef:
                     fieldPath: metadata.name
               - name: POD_NAMESPACE
                 valueFrom:
                   fieldRef:
                     fieldPath: metadata.namespace
             ports:
               - name: http
                 containerPort: 80
               - name: https
                 containerPort: 443
             livenessProbe:
               failureThreshold: 3
               httpGet:
                 path: /healthz
                 port: 10254
                 scheme: HTTP
               initialDelaySeconds: 10
               periodSeconds: 10
               successThreshold: 1
               timeoutSeconds: 10
             readinessProbe:
               failureThreshold: 3
               httpGet:
                 path: /healthz
                 port: 10254
                 scheme: HTTP
               periodSeconds: 10
               successThreshold: 1
               timeoutSeconds: 10
             lifecycle:
               preStop:
                 exec:
                   command:
                     - /wait-shutdown
   
   ---
   
   ```

   

1. 查看`w1` 的80 和443 端口

   ```bash
   lsof -i tcp:80
   lsof -i tcp:443
   ```

2. 创建`tomcat` 的`Pod` 和`service`

   记得将之前的`tomcat` 删除.  `kubectl delete -f my-tomcat.yaml`

   ```bash
   vi tomcat.yaml
   
   kubectl apply -f tomcat.yaml
   
   kubectl get svc 
   
   kubectl get pods
   ```

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: tomcat-deployment
     labels:
       app: tomcat
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: tomcat
     template:
       metadata:
         labels:
           app: tomcat
       spec:
         containers:
         - name: tomcat
           image: tomcat
           ports:
           - containerPort: 8080
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: tomcat-service
   spec:
     ports:
     - port: 80   
       protocol: TCP
       targetPort: 8080
     selector:
       app: tomcat
   ```

3. 创建`ingress` 以及定义转发规则

   ```bash
   kubectl apply -f nginx-ingress.yaml
   
   kubectl get ingress
   
   kubectl describe ingress nginx-ingress
   ```

   ```yaml
   #ingress
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: nginx-ingress
   spec:
     rules:
     - host: tomcat.luyanan.com
       http:
         paths:
         - path: /
           backend:
             serviceName: tomcat-service
             servicePort: 80
   ```

4. 修改`win` 的`hosts` 文件, 添加`dns`解析

   ```bash
   192.168.56.10 tomcat.luyanan.com
   ```

5. 打开浏览器,访问  `tomcat.luyanan.com`

总结：如果以后想要使用Ingress网络，其实只要定义ingress，service和pod即可，前提是要保证nginx ingress controller已经配置好了。