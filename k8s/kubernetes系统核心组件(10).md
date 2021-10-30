#   kubernetes系统核心组件

## 1. `Master`和`Node`

> 官网:https://kubernetes.io/zh/docs/concepts/architecture/master-node-communication/

k8s 集群中的控制节点,负责整个集群的管理和控制,可以做成高可用,防止一台`master` 宕机或者不可用,其中有一些关键的组件, 比如`API Server`、`Controller Manage` 、`Scheduler`等. 

> Node

`Node` 会被`master` 分配一些工作负载,当某个`Node`  不可用的时候, 会将工作负载转移到其他的`Node` 节点上,`Node` 上,`Node` 上有一些关键的进程,如`kubelet`、`kube-proxy`、`docker` 等等. 

> 查看集群中的`Node`

```bash
kubectl get nodes
kubectl describe node node-name
```



## 2. `kubeadm`

### 2.1 `kubeadm init`

![image-20200504170943637](http://files.luyanan.com//img/20200504170958.png)

1.  进行一系列的健康检查[`init` 之前的检查], 以确定这台机器可以部署`kubernetes`

     `kubeadm init pre-flight check:`

   1. `kubeadm` 版本与要安装的`kubernetes`  版本检查
   2. `kuernetes` 安装的系统检查(`contos`版本,`cgroup`、`docker`等)
   3. 用户、主机、端口、`swap`等

2. 生成`kubernetes` 对外提供服务所需要的各种证书对应目录, 也就是生成私钥和数字证书

    `/etc/kubernetes/pki/*`

   1. 自建`ca`,生成`ca.key`和`ca.crt`
   2. `apiserver`的私钥和公钥生成
   3. `apiserver`访问`kubeclt` 使用的客户端私钥和证书
   4. `sa.key` 和`sa.pub`
   5. `etcd` 相关私钥和数字证书

3. 为其他组件生成访问`kube-APIServer` 所需要的配置文件`xxx.conf`

   ```bash
   ls /etc/kubernetes/
   admin.conf controller-manager.conf kubelet.conf scheduler.conf
   ```

   1. 有了`$HOME/.kube/config` 就可以使用`kubectl` 和k8s集群打交道了,这个文件是来自于`admin.config`
   
      ```bash
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
      ```
   
   2. `kubeconfig`  中包含了`cluster`、`user` 和`context` 信息, `kubectl config view`
   
   3. 允许`kubectl ` 快速切换`context`, 管理多集群
   
4.  为`master` 生成`Pod` 配置文件 ,这些组件会被`master` 节点上的`kubectl` 读取到,并且创建对应的资源

    ```bash
    ls /etc/kubernetes/manifests/*.yaml
    kube-apiserver.yaml
    kube-controller-manager.yaml
    kube-scheduler.yaml
    
    ```

    这些`pod` 由`kubectl` 直接管理, 是静态`pod`, 直接使用主机网络

    `kubectl` 读取`manifests` 目录并管理各控制平台组件`pod` 的启动与停止

    要想修改这些`pod`,直接修改`manifests` 下的`yaml` 文件即可.

5.  下载镜像,等待控制平面启动

6.  一旦这些`yaml` 文件出现在被`kubectl` 监视的`/etc/kubernetes/manifests/` 目录下, `kubectl` 就会自动创建这些`yaml` 文件定义的`pod`, 即`master` 组件的容器,`master` 容器启动后,`kubeadm`会通过检查`localhost6443/healthz`  这个`master` 组件的健康状态检查`URL`, 等到`master`组件完全 运行起来. 

    【 `cat kube-apiserver.yaml`里面有健康检查的配置】

7.  为集群生成一个`bootstrap token`, 设置当前`node` 为`master`, `master` 节点将不承担工作负载. 

8.  将`ca.crt` 等`master` 节点的重要信息,通过`COnfigMap` 的方式保存在`etcd` 中, 供后续部署`node` 节点使用

9.  安装默认插件,`kubernetes` 默认`kube-proxy` 和`dns` 两个插件是必须要安装的,`dns` 插件安装了会处于`Pending`状态, 要等网络插件安装完成, 比如`calico`

    ```bash
    kubectl get daemonset -n kube-system
    ```

    可以看到kube-proxy和calico[或者其他网络插件]

###  2.2  `kubeadm join`

```bash
kubeadm join 192.168.0.51:6443 --token yu1ak0.2dcecvmpozsy8loh \ --discovery-token-ca-cert-hash
sha256:5c4a69b3bb05b81b675db5559b0e4d7972f1d0a61195f217161522f464c307b0
```



1. `join` 前检查

2. `discovery-token-ca-cert-hash` 用于验证`master` 的身份

    ```bash
   #可以计算出来, 在w 节点上执行
   openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -pubkey | openssl rsa -pubin -
   outform DER 2>/dev/null | sha256sum | cut -d' ' -f1
   # 最终的hash值
   909adc03d6e4bd676cfc4e04b864530dd19927d8ed9d84e224101ef38ed0bb96
   ```

3. `token` 用于`master` 验证`node`

   ```bash
   # 在master 上节点上,可以查看对应的token
   kubectl get secret -n kube-system | grep bootstrap-token
   # 得到token的值
   kubectl get secret/bootstrap-token-kggzhc -n kube-system -o yaml
   # 对token的值进行解码
   echo NHRzZHp0Y2RidDRmd2U5dw==|base64 -d --->4tsdztcdbt4fwe9w
   # 最终token的值
   kggzhc.4tsdztcdbt4fwe9w
   
   ```



## 3.  核心组件

![image-20200504211211667](http://files.luyanan.com//img/20200504211225.png)



### 3.1 `kubectl`

操作集群的客户端,负责和集群进行打交道

### 3.2 `kube-apiserver`

整个集群的中枢纽带,负责的事情很多 

- `/etc/kubernetes/manifests/kube-apiserver.yaml  # kubectl 管理的静态pod`
- `--insecure-port=0 #默认使用http 非安全协议访问`
- 安全验证的一些文件
- 准入策略的拦截器
-  `--authorization-mode=Node,RBAC`
- `--etcd # 配置apiserver 与etcd 通信`

### 3.3 `kube-scheduler`

单纯的调度`pod`, 按照特定的调度算法和策略，将待调度`pod` 绑定到集群中某个适合的`Node`, 并写入到绑定信息,由对应节点的`kubectl` 服务创建`pod`

1. `/etc/kubernetes/manifests/kube-scheduler.yaml # kubelet管理的静态pod`
2. `--address`表示只在master节点上提供服务，不对外
3. `kubeconfig` 表示



###  3.4 `kube-controller-manage`

负责集群中`Node`、`Pod` 副本, 服务的`endpoint`、命令空间、`Service Account`、资源配置等管理.

会划分成不同类型的`controller`, 每个`controller`都是一个死循环, 在循环中`controller` 通过`apiservice` 监视自己控制资源的状态, 一旦状态发生变化就会努力改变状态,直到变成期望的状态. 

1. `/etc/kubernetes/manifests/kube-controller-manager.yaml # kubelet管理的静态pod`
2. 参数设置`ca-file`
3. 多个`manage`, 是否需要i进行`leader` 选举



### 3.5 `kubectl`

`kubectl` 集群中的所有节点都运行,用于管理`pod` 和`contailer`, 每个`kubectl` 会向`apiserver` 注册本节点的信息,并向`master` 节点上报本节点资源使用的情况

1. `kubectl` 由操作系统`init[systemd]`  进行启动
2. `ls /lib/systemd/system/kubelet.service`
3. `systemctl daemon-reload & systemctl restart kubelet`



### 3.6 `kube-proxy`

集群中的所有节点都运行,像`service` 的操作都是由`kube-proxy` 代理的,对于客户端是透明的. 

1. `kube-proxy` 由`daemonset`  控制器在各个节点上启动唯一实例
2. 配置参数 `：/var/lib/kube-proxy/config.conf(pod内) # 不是静态pod`
3. `kubectl get pods -n kube-system`
4. `kubectl exec kube-proxy-jt9n4 -n kube-system -- cat /var/lib/kube-proxy/config.conf`
5. `mode:"" ---># iptables`



### 3.7 `DNS`

域名解析的问题

### 3.8 ` dashboard`

需要有监控面板能够检测整个集群的状态



###  3.9 `etcd`

整个集群的配置中心,所有集群的状态数据,对象数据都存储在`etcd` 中. 

`kubeadm` 引导启动的k8s 集群, 默认只启动一个`etcd` 节点

1. `/etc/kubernetes/manifests/etcd.yaml # kubelet管理的静态pod`
2. `etcd` 所使用的相关密钥在`/etc/kubernetes/pki/etcd` 里面
3. `etcd` 挂载`master` 节点本地路径 `/var/lib/etcd` 用于运行时数据存储,`tree`



## 4.  `kubernetes` 源码查看方式

> 源码地址: https://github.com/kubernetes/kubernetes
>
> https://github.com/kubernetes/kubernetes/tree/release-1.14



## 5. `kubectl`

> 语法: `kubectl [command] [TYPE] [NAME] [flag]`
>
> 官网: https://kubernetes.io/docs/reference/kubectl/overview/

- `command`:  用于操作k8s 资源对象的命令,比如`apply`、`delete`、`describe`、`get`等
- `TYPE`: 要操作资源对象的类型, 区分大小写,比如`pod[pods/po]`、`deployment`
- `NAME`: 要操作对象的具体名称,若不指定,则返回该资源类型的全部对象(是默认命名空间下的)
- `flags`: 可选

### 5.1 `demo`

- 查看集群的状态

  ```bash
  # 查看集群的信息
  kubectl config view
  # 查看cluster的信息
  kubectl config get-clusters
  ```

- 创建资源

  ```bash
  kubectl apply -f xxx.yaml
  kubectl apply -f <directory>
  ```

- 查看资源对象

  ```bash
  # 查看Pod
  kubectl get pods
  # 查看Service
  kubectl get svc
  ```

- 问题查看调试

  ```bash
  kubectl describe pod pod-name
  kubectl exec -it pod-name
  kubectl logs -f pod-name
  kubectl attach
  
  ```

  



## 6. `API Server`

> 官网: https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/

`API Server` 提供了k8s 各类资源对象的操作,是集群内各个功能模块之间数据交互和通信的中心枢纽,是整个系统的数据总线和数据中心,通常我们通过`kubectl` 和`apiservice` 进行交互. 

`APIService` 通过`kube-apiserver`的进程提供服务,运行在`master` 节点上. 

`kubectl`与`APIServer` 之间是`REST` 调用

```text
The Kubernetes API server validates and configures data for the api objects which
include pods, services, replicationcontrollers, and others. The API Server services
REST operations and provides the frontend to the cluster’s shared state through which
all other components interact.
```

1. 查看`yaml` 文件中的`apiVersion`

   ```bash
   grep -r "apiVersion" .
   
   ```

   ```text
   ./pod_nginx.yaml:apiVersion: apps/v1
   ./my-tomcat.yaml:apiVersion: apps/v1
   ./my-tomcat.yaml:apiVersion: v1
   ./mandatory.yaml:apiVersion: v1
   ./mandatory.yaml:apiVersion: v1
   ./mandatory.yaml:apiVersion: v1
   ./mandatory.yaml:apiVersion: v1
   ./mandatory.yaml:apiVersion: v1
   ./mandatory.yaml:apiVersion: rbac.authorization.k8s.io/v1beta1
   ./mandatory.yaml:apiVersion: rbac.authorization.k8s.io/v1beta1
   ./mandatory.yaml:apiVersion: rbac.authorization.k8s.io/v1beta1
   ...
   
   ```

2. `REST API` 设计

    `API官网`：  https://kubernetes.io/docs/concepts/overview/kubernetes-api/

   `V1.14`: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/

    ![image-20200504215548798](http://files.luyanan.com//img/20200504215549.png)

3. 想要写`pod` 的`yaml` 文件

   ```bash
   https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#pod-v1-core
   ```

4. `kube-apiserver`

    ```bash
   lsof -i tcp:8080
   vi /etc/kubernetes/manifests/kube-apiserver.yaml [kubeadm安装方式]
   # 查询insecure-port，并将修改端口为8080
   insecure-port=8080
   # kubect apply生效，需要等待一会
   kubectl apply -f kube-apiserver.yaml
   ```

5. 查看端口以及访问测试

    可以发现结果和`kubectl`使用一样

   ```bash
   lsof -i tcp:8080
   curl localhost:8080
   curl localhost:8080/api
   curl localhost:8080/api/v1
   curl localhost:8080/api/v1/pods
   curl localhost:8080/api/v1/services
   ```

6. 设计一个`pod`的`url` 请求

   ```bash
   https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#-strong-write-operations-pod-v1-core-strong
   ```

7. 这种操作还是相对比较麻烦的,哪怕使用`kubectl`

    https://github.com/kubernetes-client

   Java ：https://github.com/kubernetes-client/java

   Go ：https://github.com/kubernetes/client-go



## 7.  集群安全机制之`API Server`

>  官网:https://v1-12.docs.kubernetes.io/docs/reference/access-authn-authz/controlling-access/

对于k8s 集群的访问操作, 都是通过`api server` 的`rest api` 来实现的,难道所有的操作都允许吗? 当然是不行的, 这里就涉及到认证、授权和准入等操作. 

![image-20200505083325959](http://files.luyanan.com//img/20200505083355.png)



### 7.1  `API Server认证(Authentication)`

> 官网: https://v1-12.docs.kubernetes.io/docs/reference/access-authn-authz/controlling-access/#authentication

说白了, 就是如何来识别客户端的身份,k8s 集群中提供了3种识别客户端的身份的方式



- `https`  证书认证

   基于`CA` 根证书签名的双向数字证书认证方式

- `HTTP Token` 认证

    通过`Token` 来识别合法用户

- `HTTP Base` 认证

   通过用户名+密码的方式来认证

### 7.2 `API Server`授权(Authorization)

> 官网: https://v1-12.docs.kubernetes.io/docs/reference/access-authn-authz/controlling-access/#authorization

- `ABAC` 授权模式
- `Webhook` 授权模式
- `RBPC` 授权模式



`Role`、`ClusterRole`、`RoleBinding`和`ClusterRoleBinding`

用户可以使用`kubectl` 或者`API` 调用等方式操作这些资源对象

![image-20200505085230364](http://files.luyanan.com//img/20200505085232.png)



### 7.3  `Admission Contro`(准入控制)

> 官网:https://v1-12.docs.kubernetes.io/docs/reference/access-authn-authz/controlling-access/#admission-control



- `Always`

  允许所有请求

- `AlwaysPullImages`

   在启动容器之前总是尝试重新下载镜像

- `AlwaysDeny`

   禁止所有请求



## 8. `Scheduler`

> 官网: https://kubernetes.io/docs/concepts/scheduling/kube-scheduler/

> In Kubernetes, *scheduling* refers to making sure that [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) are matched to [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/) so that [Kubelet](https://kubernetes.io/docs/reference/generated/kubelet) can run them.
>
>  

通过调度算法,为待调度`Pod` 列表的每个`pod`, 从`Node` 列表中选择一个最合适的`Node`

然后,目标节点上的`kubelet` 通过`API Server` 监听到`Kubernetes Schduler` 产生的`Pod`  绑定事件,获取对应的`Pod` 清单,下载`image` 镜像,并启动容器

### 8.1  架构图

 

![image-20200505090156707](http://files.luyanan.com//img/20200505090159.png)



### 8.2  流程描述

> https://kubernetes.io/docs/concepts/scheduling/kube-scheduler/#kube-scheduler-implementation

> 1. `Filtering`
> 2. `Scoring`

1. 预选调度策略：遍历所有目标的`Node`,筛选出符合`Pod` 要求的候选节点
2. 优选调度策略: 在1的基础上, 采用优选策略算法计算出每个候选节点的积分, 积分最高者胜出



### 8.3  预选策略和优选策略

#### 8.3.1 预选策略

> https://kubernetes.io/docs/concepts/scheduling/kube-scheduler/#filtering

- `PodFitsHostPorts`

  > Checks if a Node has free ports (the network protocol kind) for the Pod ports the Pod is requesting.

- `PodFitsHost`

  > Checks if a Pod specifies a specific Node by it hostname.

  `PodFitsResources`

  > Checks if the Node has free resources (eg, CPU and Memory) to meet the requirement of the Pod.



####  8.3.2 优选策略

> https://kubernetes.io/docs/concepts/scheduling/kube-scheduler/#scoring

- `SelectorSpreadPriority`

  > Spreads Pods across hosts, considering Pods that belonging to the same Service, StatefulSet or ReplicaSet

- `InterPodAffinityPriority`

  > Computes a sum by iterating through the elements of weightedPodAffinityTerm and adding “weight” to the sum if the corresponding PodAffinityTerm is satisfied for that node; the node(s) with the highest sum are the most preferred.



### 8.4  实战

####  8.4.1  `Node`

1. 正常创建`Node`, 准备 `scheduler-node-origin.yaml` 资源文件

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: scheduler-node
   spec:
     selector:
       matchLabels:
         app: scheduler-node
     replicas: 1
     template:
       metadata:
         labels:
           app: scheduler-node
       spec:
         containers:
         - name: scheduler-node
           image: registry.cn-hangzhou.aliyuncs.com/luyanan/test-docker-image:v1.0
           ports:
           - containerPort: 8080
   ```

    启动并查看资源

   ```bash
   kubectl apply -f scheduler-node-origin.yaml
   kubectl get pods
   kubectl describe pod pod-name
   
   ```

2. 准备`scheduler-node.yaml` 资源文件

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: scheduler-node
   spec:
     selector:
       matchLabels:
         app: scheduler-node
     replicas: 1
     template:
       metadata:
         labels:
           app: scheduler-node
       spec:
         containers:
         - name: scheduler-node
           image: registry.cn-hangzhou.aliyuncs.com/luyanan/test-docker-image:v1.0
           ports:
           - containerPort: 8080
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
               - matchExpressions:
                 - key: beta.kubernetes.io/arch
                   operator: In
                   values:
                   - amd641
             preferredDuringSchedulingIgnoredDuringExecution:
             - weight: 1
               preference:
                 matchExpressions:
                 - key: disktype
                   operator: NotIn
                   values:
                   - ssd
   
   ```

   主要是体现`node` 的调度

   ```bash
   kubectl get nodes w1 -o yaml
   找到labels
   ```

   ```bash
   [root@master-kubeadm-k8s ~]# kubectl get pods
   NAME READY STATUS RESTARTS AGE
   scheduler-node-84845c99d4-fvw9d 0/1 Pending 0 7s
   ```

   ```bash
   kubectl describe pod scheduler-node-84845c99d4-fvw9d
   Events:
   Type Reason Age From Message
   ---- ------ ---- ---- -------
   Warning FailedScheduling 37s (x2 over 37s) default-scheduler 0/3 nodes are available:
   3 node(s) didn't match node selector.
   ```



#### 8.4.2 `Pod`

```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
      matchExpressions:
      - key: app
        operator: In
        values:
          - k8s
      topologyKey: kubernetes.io/hostname
```





## 9. `kubelet`

> 官网:https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/

在k8s集群中,每个`Node` 上都会启动一个`kubelet` 服务进程,用于处理`master` 节点上发到本节点的任务

管理`pod` 以及`pod` 中的容器,每个`kubelet` 进程会在`API Server` 上注册自身信息,定期向`master` 节点上汇报节点资源使用情况,并通过`cAdvisor` 监控容器和节点资源



## 10 .`kube-proxy`

>  官网:https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/

在k8s 集群中, 每个`Node` 上都会运行一个`kube-proxy` 进行, 它是`service` 的透明代理兼负载均器,核心功能是将某个`Service` 的访问请求转发到后端的多个`Pod` 实例上. 

