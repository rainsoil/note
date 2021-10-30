# Kubernetes之入门篇

## 1. K8S核心组件和架构图

[K8S Docs Concepts](https://kubernetes.io/docs/concepts/)

1.  先从`contailer` 开始, K8S 既然是容器编排工具, 那么一定会有`contailer`

    ![image-20200420103421591](http://files.luyanan.com//img/20200420103423.png)

2. 那k8s 如何操作这些`contailer`呢?  k8s 不想直接操作这些`contailer`, 因为操作`contailer` 是`docker` 做的事情,k8s 中有自己的操作单位,称之为`Prod`

    说白了, `Prod` 就是一个或者多个`contailer`的集合. 

   ![image-20200420103705134](http://files.luyanan.com//img/20200420103706.png)

 看看官网是怎么描述的? https://kubernetes.io/docs/concepts/workloads/pods/pod/

```text
A Pod (as in a pod of whales or pea pod) is a group of one or more containers (such as Docker containers), with shared storage/network, and a specification for how to run the containers. A Pod’s contents are always co-located and co-scheduled, and run in a shared context. A Pod models an application-specific “logical host” - it contains one or more application containers which are relatively tightly coupled — in a pre-container world, being executed on the same physical or virtual machine would mean being executed on the same logical host
```



3. 那`Prod` 的维护谁来做呢? 那就是`ReplicaSet`, 通过`selector` 来进行管理

    ![image-20200420104257993](http://files.luyanan.com//img/20200420104304.png)

​    看官网是怎么描述的. https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/

```text
A ReplicaSet is defined with fields, including a selector that specifies how to identify Pods it can acquire, a number of replicas indicating how many Pods it should be maintaining, and a pod template specifying the data of new Pods it should create to meet the number of replicas criteria. A ReplicaSet then fulfills its purpose by creating and deleting Pods as needed to reach the desired number. When a ReplicaSet needs to create new Pods, it uses its Pod template.
```

4. `Prod` 和`ReplicaSet` 的状态如何维护和检测呢? `Deployment`

    ![image-20200420112110918](http://files.luyanan.com//img/20200420112116.png)

 看看官网是如何描述的, https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

```text
A Deployment provides declarative updates for Pods and ReplicaSets.

You describe a desired state in a Deployment, and the Deployment Controller changes the actual state to the desired state at a controlled rate. You can define Deployments to create new ReplicaSets, or to remove existing Deployments and adopt all their resources with new Deployments.
```

5. 不妨把相同的或者有关联的`Prod` 分门别类一下, 那怎么分类呢? `Lable`

    ![image-20200420112403867](http://files.luyanan.com//img/20200420112411.png)

官网是如何描述的： https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/

```text
Labels are key/value pairs that are attached to objects, such as pods
```

6. 使得相同的`lable` 的`service` 要是能够有一个名称就好了,`service`

   ![image-20200420112901781](http://files.luyanan.com//img/20200420112903.png)

看官网上怎么说:https://kubernetes.io/docs/concepts/services-networking/service/

```text
An abstract way to expose an application running on a set of Pods as a network service.

With Kubernetes you don’t need to modify your application to use an unfamiliar service discovery mechanism. Kubernetes gives Pods their own IP addresses and a single DNS name for a set of Pods, and can load-balance across them.
```



7. 上述说了那么多, `Prod` 运行在哪里呢? 当然是机器了,比如一台`centos`机器,我们把这个机器称之为`Node`

    ![image-20200420113324004](http://files.luyanan.com//img/20200420113325.png)

看看官网怎么说: https://kubernetes.io/docs/concepts/architecture/nodes/

```text
A node is a worker machine in Kubernetes, previously known as a minion. A node may be a VM or physical machine, depending on the cluster. Each node contains the services necessary to run pods and is managed by the master components. The services on a node include the container runtime, kubelet and kube-proxy. See The Kubernetes Node section in the architecture design doc for more details.
```

8. 难道只有一个`Node`, 显然不太合适, 多台`Node`共同组成集群. 

   ![image-20200420200134882](http://files.luyanan.com//img/20200420200149.png)

9. 此时我们把目光转移到由3个`Node` 节点组成的`Matser-Node` 集群. 

    ![image-20200420200300876](http://files.luyanan.com//img/20200420200302.png)

10. 这个集群要配合完成一些工作,总要有一些组件的支持把? 接下来我们想想有哪些组件, 然后画一个相对完整的架构图. 

    1. 总要有一个操作集群的客户端, 也就是跟集群打交道

        **kubectl**

    2. 请求肯定是到达`Master Node`, 然后再分配给`Worker Node`, 创建`Prod` 之类的, 关键是命令通过`kubectl` 过来之后, 是不是要认证授权一下? 

    3. 请求过来之后,`Master Node` 中谁来接收? 

        **`APIServer`**

    4. `API` 收到请求后,接下来是调用哪个`Worker Node` 创建`Prod`、`Contailer` 之类的，的要有调度策略

        [**`Scheduler`**][https://kubernetes.io/docs/concepts/scheduling/kube-scheduler/]

    5. `Scheduler` 通过不同的策略,真正要分发请求到不同的`worker Node` 上创建内容,具体谁负责?

        **`Controlelr Manage`**

    6. `workder Node` 接收到创建请求后,具体谁来负责`kebectl` 服务,最终会调用`kebectl` 会调用`Docker Engine`,创建对应容器,这边是不是也反应出来一点, 在`Node` 上需要有`Docker Engine`,不然怎么创建维护容器呢? 

    7. 会不会涉及到域名解析的问题?

        **`DNS`**

    8. 是否需要有监控面板能够检测到整个集群的状态?

        **`Dashboard`**

    9. 集群中这些数据如何保存? 分布式存储

        **`ETCD`**

    10. 至于像容器的持久化存储,网络等

        ![image-20200420202817169](http://files.luyanan.com//img/20200420202818.png)

    

    11. 不妨把这个图翻转一下方便查看

        ![image-20200420202858629](http://files.luyanan.com//img/20200420202900.png)

12. k8s官网架构图

    https://kubernetes.io/docs/concepts/architecture/cloud-controller/

    ![image-20200420202933162](http://files.luyanan.com//img/20200420202940.png)



## 2. k8s的安装

### 2.1 最艰难的方式

[Kelsey Hightower](https://github.com/kelseyhightower)

### 2.2 在线play-with-k8s(只能用4个小时)

网址: https://labs.play-with-k8s.com/

```text
[node1 ~]$
                          WARNING!!!!

 This is a sandbox environment. Using personal credentials
 is HIGHLY! discouraged. Any consequences of doing so, are
 completely the user's responsibilites.

 You can bootstrap a cluster as follows:

 1. Initializes cluster master node:

 kubeadm init --apiserver-advertise-address $(hostname -i)


 2. Initialize cluster networking:

 kubectl apply -n kube-system -f \
    "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"


 3. (Optional) Create an nginx deployment:

 kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/application/nginx-app.yaml
```



### 2.3 `Cloud` 上搭建

> github:https://github.com/kubernetes/kops



### 2.4 企业级解决方法`CoreOS`

> coreos:https://coreos.com/tectonic/



### 2.5 `Minikube`

> k8s 单节点, 适合本地学习使用

官网: https://kubernetes.io/docs/setup/learning-environment/minikube/

`Github:` https://github.com/kubernetes/minikube



### 2.6 `Kubeadm`

本地多节点

`Github:` https://github.com/kubernetes/kubeadm





## 3. 使用`Minikube` 搭建单节点k8s

### 3.1 window

[kebectl官网](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-windows)

[`minikebe`官网](https://kubernetes.io/docs/tasks/tools/install-minikube/)

1. 选择任意一种虚拟化方式

   - `Hyper-V`
   - `VirtualBox`

2. 安装`kebectl`

   - 根据官网步骤或直接下载

      https://storage.googleapis.com/kubernetesrelease/release/v1.16.2/bin/windows/amd64/kubectl.exe

   - 配置`kubectl.exe`  所在路径的的环境变量,使得cmd 窗口可以直接使用`kebectl` 命令

   - `kebectl version` 检查是否配置成功

3. 安装`minikebe`

   1. 根据官网步骤或者直接下载

       https://github.com/kubernetes/minikube/releases/download/v1.5.2/minikubewindows-amd64.exe

   2. 修改`minikube-windows-amd64.exe` 为 `minikube.exe`

   3. 配置`minikube` 所在路径的环境变量,使得`cmd` 创建可以直接使用`minikube` 命令

   4. `minikube version` 检查是否配置

4. 使用`minikube` 创建单节点的k8s

   ```bash
   minikube start --vm-driver=virtualbox --image-repository=gcr.azk8s.cn/googlecontainers
   ```

5. 小结

   > 其实就是通过`minikube` 创建一个虚拟机,这个虚拟机中安装好了单节点的k8s 环境然后通过`kubectl` 进行交互

   ```bash
   ##  创建k8s
   minikube start
   ## 删除k8s
   minikube delete
   ## 进入到k8s的容器中
   minikube ssh
   ## 查看状态
   minikube status
   ## 进入到dashboard
   minikube dashboard
   ```



###  3.2 centos

[`kebectl官网`](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-linux)

[`minikube`](https://kubernetes.io/docs/tasks/tools/install-minikube/)

1. 安装docker

2. 安装`kubectl`

   - 下载

     ```bash
     curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
     
     ```

     

   - 授权

     ```bash
     chmod +x ./kubectl
     ```

   - 添加到环境变量

     ```bash
     sudo mv ./kubectl /usr/local/bin/kubectl
     
     ```

   - 检查

     ```bash
     kubectl version
     
     ```

3. 安装`minikube`

   1. 下载

      ```bash
      wget https://github.com/kubernetes/minikube/releases/download/v1.5.2/minikubelinux-amd64
      ```

   2. 配置环境变量

      ```bash
      sudo mv minikube-linux-amd64 minikube && chmod +x minikube && mv minikube
      /usr/local/bin/
      ```

   3. 检查

      ```bash
      minikube version
      ```

   使用`minikebe`  创建单节点的k8s

   ```bash
   minikube start --vm-driver=none --image-repository=gcr.azk8s.cn/googlecontainers
   ```



### 3.3 Mac OS

也是下载安装`kebectl` 和`minikebe`,选择`virtualbox`,然后`minikebe start`, 就可以通过`kebectl` 操作. 



## 4.  先感受一下`K8s`

既然已经通过`minikebe` 搭建了单节点的k8s,不妨先来感受一下组件的存在和操作 



### 4.1 查看连接信息

```bash
kubectl config view
kubectl config get-contexts
kubectl cluster-info
```



### 4.2  体验`Prod`

1. 创建`prod_nginx.yaml`

   ```yaml
   apiVersion: v1
   kind: Pod
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

2. 根据`prod_nginx.yaml` 文件创建`prod`

   ```bash
   kubectl apply -f pod_nginx.yaml
   ```

3. 查看`prod`

   ```bash
   kubectl get pods
   kubectl get pods -o wide
   kubectl describe pod nginx
   ```

4. 进入到`nginx`容器中

   ```bash
   # kubectl进入
   kubectl exec -it nginx bash
   # 通过docker进入
   minikube ssh
   docker ps
   docker exec -it containerid bash
   ```

5. 访问nginx, 端口转发

   ```bash
   # 若在minikube中，直接访问
   # 若在物理主机上，要做端口转发
   kubectl port-forward nginx 8080:80
   
   ```

6. 删除`prod`

   ```bash
   kubectl delete -f pod_nginx.yaml
   
   ```

   