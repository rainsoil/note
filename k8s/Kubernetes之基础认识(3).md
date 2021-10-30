# Kubernets 基础认识

##   1. `yaml` 文件

### 1.1  简介

YAML（IPA: /ˈjæməl/）是一个可读性高的语言，参考了XML、C、Python等。

理解：Yet Another Markup Language

后缀：可以是.yml或者是.yaml，更加推荐.yaml，其实用任意后缀都可以，只是阅读性不强



### 1.2 基础

- 区分大小写
- 缩进表示层级关系,相同层级的元素左对齐
- 缩进只能使用空格, 不能使用`TAB`
- `#` 表示当前行的注释
- 是`JSON` 文件的超集, 两个可以相互转换
- `--` 表示分隔符, 可以在一个文件中定义多个结构
- 使用`key:value` , 其中`:` 和`value` 之间要有一个英文空格。 

### 1.3 `Maps`

#### 1.3.1  简单

```yaml
apiVersion: v1
kind: Pod
---
```

- `---` 表示分隔符,可选,要定义多个结构一定要分割

- `apiVersion` 表示`key`, `v1` 表示`value`, 英文`:`后面一定要有个空格. 

- `kind` 表示`key`,  `Pod` 表示`value`

   也可以这样写, `apiVersion: "v1"`



转换为`JOSN`

```json
{
"apiVersion": "v1",
"kind": "Pod"
}
```



####  1.3.2 复杂的

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
```



- metadata表示key，下面的内容表示value，该value中包含两个直接的key：name和labels

- name表示key，nginx-deployment表示value

- name表示key，nginx-deployment表示value

- labels表示key，下面的表示value，这个值又是一个map

- app表示key，nginx表示value

- 相同层级的记得使用空间缩进，左对齐

  

  `转换为JSON格式`

```json
{
"apiVersion": "apps/v1",
"kind": "Deployment",
"metadata": {
         "name": "nginx-deployment",
         "labels": {
                    "app": "nginx"
                   }
        }
}
```



### 1.4 `Lists`

```bash
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container01
    image: busybox:1.28
  - name: myapp-container02
    image: busybox:1.28
```



- containers表示key，下面的表示value，其中value是一个数组

- 数组中有两个元素，每个元素里面包含name和image

- image表示key，myapp-container表示value

`转换成JSON格式`

```json
{
"apiVersion": "v1",
"kind": "Pod",
"metadata": {
           "name": "myapp",
           "labels": {
                       "app": "myapp"
                     }
         },
"spec": {
 "containers": [{
                 "name": "myapp-container01",
                 "image": "busybox:1.28",
                }, 
                {
                 "name": "myapp-container02",
                 "image": "busybox:1.28",
                }]
      }
}
```



### 1.5  找个`k8s`的`yaml` 文件

> 官网: <https://kubernetes.io/docs/reference/>

```yaml
# yaml格式对于Pod的定义：
apiVersion: v1          #必写，版本号, 比如 v1
kind: Pod               #必写，类型, 比如pod
metadata:               #必写，元数据
  name: nginx           #必写，pod 名称
  namespace: default    #表示pod 所属的命名空间
  labels:
    app: nginx                  # 自定义标签
spec:                           #必写，pod中容器的详细定义
  containers:                   #必写，pod中容器列表
  - name: nginx                 #必写，容器名称
    image: nginx                #必写，容器的镜像名称
    ports:
    - containerPort: 80         #表示容器的端口
```



## 2. `contailer`

> 官网: <https://kubernetes.io/docs/concepts/containers/>

### 2.1 `Docker` 世界中

可以通过`docker run`  运行一个容器

或者定义一个`yml` 文件, 本机使用`docker-compose.yml`, 多机使用`docker swarm` 创建. 

###  2.2 k8s 世界中

同样以一个`yaml` 文件, `contailer` 运行在`pod` 中 

## 3. `Pod`

`官网`：<https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/>

### 3.1  what is pod

```text
  A Pod is the basic execution unit of a Kubernetes application–the smallest and simplest unit in the Kubernetes object model that you create or deploy. A Pod represents processes running on your cluster.
  A Pod encapsulates an application’s container (or, in some cases, multiple containers), storage resources, a unique network identity (IP address), as well as options that govern how the container(s) should run. A Pod represents a unit of deployment: a single instance of an application in Kubernetes, which might consist of either a single container or a small number of containers that are tightly coupled and that share resources.
```



### 3.2 Pod 初体验

#### 3.2.1  创建一个`pod`的`yaml` 文件,名称为`nginx_pod.yaml`

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



####  3.2.2 根据该`nginx_pod.yaml` 文件创建`pod`

```bash
kubectl apply -f nginx_pod.yaml
```



#### 3.2.3  查看`pod`

#####  3.2.3.1  `kubectl get pods`

```text
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          29s
```



#####  3.2.3.2  `kubectl get pods -o wide`

```text
[root@master-kubeadm-k8s test]# kubectl get pods -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP               NODE   NOMINATED NODE   READINESS GATES
nginx-pod   1/1     Running   0          26s   192.168.190.68   w1     <none>           <none>

```



#####  3.2.3.3 `kubectl describe pod nginx-pod`

```text
[root@master-kubeadm-k8s test]# kubectl describe pod nginx-pod
Name:               nginx-pod
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               w1/192.168.56.11
Start Time:         Tue, 28 Apr 2020 03:29:54 +0000
Labels:             app=nginx
Annotations:        cni.projectcalico.org/podIP: 192.168.190.68/32
                    kubectl.kubernetes.io/last-applied-configuration:
                      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"labels":{"app":"nginx"},"name":"nginx-pod","namespace":"default"},"spec":{"c...
Status:             Running
IP:                 192.168.190.68
Containers:
  nginx-container:
    Container ID:   docker://ce377530ef64beaa3dea9d4f7b0f0f2ac2db133874431d1af03ca77fc0a151b6
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:86ae264c3f4acb99b2dee4d0098c40cb8c46dcf9e1148f05d3a51c4df6758c12
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Tue, 28 Apr 2020 03:30:06 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-mq5wb (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-mq5wb:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-mq5wb
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  55s   default-scheduler  Successfully assigned default/nginx-pod to w1
  Normal  Pulling    55s   kubelet, w1        Pulling image "nginx"
  Normal  Pulled     44s   kubelet, w1        Successfully pulled image "nginx"
  Normal  Created    43s   kubelet, w1        Created container nginx-container
  Normal  Started    43s   kubelet, w1        Started container nginx-container
```



#### 3.2.4 可以发现这个`pod` 运行在`worker01` 节点上

于是来到了`worker02` 节点上,`docker ps` 一下

 ```text
ce377530ef64        nginx                  "nginx -g 'daemon of…"   2 minutes ago       Up 2 minutes                            k8s_nginx-container_nginx-pod_default_86b00ae5-8900-11ea-94ae-5254008afee6_0

 ```

不妨进入到该容器中试试(发现只有`worker01` 上有该容器,因为pod 运行在`worker01 上`)： 

```bash
docker exec -it ce377530ef64 bash
```

####  3.2.5  访问`nginx` 容器

```bash
curl 192.168.190.68
```

发现任何一个集群的`Node` 上都能访问成功. 

#### 3.2.6  删除`Pod`

```bash
kubectl delete -f nginx_pod.yaml
```



