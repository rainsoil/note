#  kubernetes组件之Storage



##  1 [`Volume`](https://kubernetes.io/docs/concepts/storage/volumes/)

```text
On-disk files in a Container are ephemeral, which presents some problems for non-trivial applications when running in Containers. First, when a Container crashes, kubelet will restart it, but the files will be lost - the Container starts with a clean state. Second, when running Containers together in a Pod it is often necessary to share files between those Containers. The Kubernetes Volume abstraction solves both of these problems.
```



## 2 `Host` 类型`volume` 实战

> 背景： 定义一个`Pod`, 其中包含两个`contailer`,都使用`pod`的`vloume`

`volume-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: nginx-container
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: volume-pod
      mountPath: /nginx-volume
  - name: busybox-container
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
    volumeMounts:
    - name: volume-pod
      mountPath: /busybox-volume
  volumes:
  - name: volume-pod
    hostPath:
      path: /tmp/volume-pod 

```

1. 创建资源

   ```bash
   kubectl apply -f volume-pod.yaml
   
   ```

2. 查看`pod`的运行情况

   ```bash
   kubectl get pods -o wide
   
   ```

3. 来到运行的`worker` 节点

   ```bash
   docker ps | grep volume
   ls /tmp/volume-pod
docker exec -it containerid sh
   ls /nginx-volume
   ls /busybox-volume
   # 看看是否同步
   ```
   
4. 查看`pod` 中的容器里面的`hosts` 文件,是否一样. 

   > 发现是一样的,并且都是由`pod` 管理的.

   ```bash
   docker exec -it containerid cat /etc/hosts
   ```

5. 所以一般`contailer` 中的存储或者网络中的内容, 不要在`contailer` 层面修改,而是在`pod` 中修改,

    比如下面修改一下网络: 

> ```yaml
> spec:
> hostNetwork: true
> hostPID: true
> hostAliases: 
>   - ip: "192.168.8.61"
>  hostnames: 
>     - "test.jack.com" 
> containers:
>   - name: nginx-container
>  image: nginx
> ```

   

##   3  `PersistentVolume`

>  官网:https://kubernetes.io/docs/concepts/storage/persistent-volumes/

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi    # 存储空间大小
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce     # 只允许一个Pod进行独占式读写操作
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp            # 远端服务器的目录
    server: 172.17.0.2    # 远端的服务器
```



说白了,`PV`是k8s中的资源,`volume`的`plugin` 实现,生命周期独立于`pod`, 封装了底层存储卷实现的字节. 

> 注意: `PV`的维护通常是由运维人员、集群管理员进行维护的. 

##   4 `PersistentVolumeClaim`

> 官网: <https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims>

有了`PV`, 那`Pod` 如何使用呢? 为了方便使用, 我们可以设计出一个`PVC` 来绑定`PV`, 然后把`PVC` 交给`Pod` 来使用即可,

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```



说白了, `PVC` 会匹配满足要求的`PV`[**是根据size和访问模式进行匹配的**],进行一一绑定,然后他们的状态都会变成`Bound`

也就是`PVC` 负责请求`PV` 的大小和访问方式,然后`Pod` 中就可以直接使用`PVC`了. 



## 5 `Pod` 中如何使用`PVC`

> 官网: <https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

##  6 `Pod` 中使用`PVC` 实战

> 背景: 使用`nginx` 持久化存储演示
>
> 1.  共享存储使用`nfs`, 
> 2. 创建`pv` 和`pvc`
> 3. `nginx pod`中使用`pvc`

###  6.1  `master`节点上搭建`nfs`

> `nfs(network file system)`网络文件系统,是`FreeBSD`支持的文件系统中的一种,允许网络中的计算机之间通过`TCP/IP`网络共享资源. 

1. 安装`nfs`

   ```bash
   yum install -y nfs-utils
   ```

2. 创建`nfs` 目录

   ```bash
   mkdir -p /nfs/data/
   ```

3. 授予权限

   ```bash
   chmod -R 777 /nfs/data
   ```

4. 编辑`export` 文件

   ```bash
   vi /etc/exports
   /nfs/data *(rw,no_root_squash,sync)
   ```

5. 使得配置生效

   ```bash
   exportfs -r
   ```

6. 查看生效

   ```bash
   exportfs
   ```

7. 启动`rpcbind`、`nfs` 服务

   ```bash
   systemctl restart rpcbind && systemctl enable rpcbind
   systemctl restart nfs && systemctl enable nfs
   ```

8. 查看`rpc` 注册的情况

   ```bash
   rpcinfo -p localhost
   ```

9. `showmount`测试

   ```bash
   showmount -e master-ip
   ```

10. 所有的节点上都安装客户端

    ```bash
    yum -y install nfs-utils
    systemctl start nfs && systemctl enable nfs
    ```



### 6.2  创建`PV&PVC&Nginx`

1. 在`nfs` 服务器创建所需要的目录

   ```bash
   mkdir -p /nfs/data/nginx
   ```

2. 定义`PV/PVC/Nginx`的`nginx-pv-demo.yaml` 文件

   ```yaml
   # 定义PV
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: nginx-pv
   spec:
     accessModes:
       - ReadWriteMany
     capacity:
       storage: 2Gi    
     nfs:
       path: /nfs/data/nginx     
       server: 192.168.56.10  
       
   ---
   # 定义PVC，用于消费PV
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: nginx-pvc
   spec:
     accessModes:
       - ReadWriteMany
     resources:
       requests:
         storage: 2Gi
     
   ---
   # 定义Pod，指定需要使用的PVC
   apiVersion: apps/v1beta1
   kind: Deployment
   metadata:
     name: nginx
   spec:
     selector:
       matchLabels: 
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
         - image: nginx
           name: mysql
           ports:
           - containerPort: 80
           volumeMounts:
           - name: nginx-persistent-storage
             mountPath: /usr/share/nginx/html
         volumes:
         - name: nginx-persistent-storage
           persistentVolumeClaim:
             claimName: nginx-pvc
   ```

3. 根据`yaml` 创建资源并且查看资源

   ```bash
   kubectl apply -f nginx-pv-demo.yaml
   kubectl get pv,pvc
   kubectl get pods -o wide
   ```

4. 测试持久化存储

   1. 在`/nfs/data/nginx` 上新建文件, 写上内容
   2. `kubectl get pods -o wide` , 获取`nginx` 的ip地址
   3. `curl nginx-pod-ip/1.html`
   4. `kubectl exec -it nginx-pod bash`,进入`/usr/share/nginx/html` 目录查看
   5. `kubectl delete pod nginx-pod`
   6. 查看新的`nginx-pod`的ip 并且访问`nginx-pod-ip/1.html`



## 7  `StorageClass`

上面手动管理`Pv`的方式还是有点`low`, 能不能更加灵活一点点呢? 

`官网`：<https://kubernetes.io/docs/concepts/storage/storage-classes/>

`nfs github`：`github`：<https://github.com/kubernetes-incubator/external-storage/tree/master/nfs>

> A StorageClass provides a way for administrators to describe the “classes” of storage they offer. Different classes might map to quality-of-service levels, or to backup policies, or to arbitrary policies determined by the cluster administrators. Kubernetes itself is unopinionated about what classes represent. This concept is sometimes called “profiles” in other storage systems.
>
> Each StorageClass contains the fields `provisioner`, `parameters`, and `reclaimPolicy`, which are used when a PersistentVolume belonging to the class needs to be dynamically provisioned.
>
> The name of a StorageClass object is significant, and is how users can request a particular class. Administrators set the name and other parameters of a class when first creating StorageClass objects, and the objects cannot be updated once they are created.

`StorageClass`  声明存储的插件,用于自动创建`PV`

说白了, 就是创建`PV`的模板,其中有两个最重要的部分: `PV`属性和创建此`PV` 所需要的插件

可以为`PV` 指定 `storageClassName` 属性, 标识`PV` 归属于哪一个`Class`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

1. 对于`PV` 或者`StorageClass` 只能对应一个后端存储
2. 对于手动的情况,一般我们会创建很多的`PV`, 等有`PVC` 需要使用的事情就可以直接使用了. 
3. 对于自动的情况,那么就由`storageClass` 来自动管理创建
4. 如果`Pod`想要使用共享存储,一般会在创建`PVC`、`PV` 中描述了想要什么类型的后端存储,空间等, k8s 从而会匹配对应的`PV`, 如果没有匹配成功,`Pod`就会处于`Pending` 状态.`Pod` 中使用只需要像使用`volume`  一样,指定名字就可以使用了,
5. 一个`Pod`可以使用多个`PVC`,一个`PVC` 也可以使用多个`Pod` 使用. 
6. 一个`PVC` 可以绑定多个`PV`, 一个`PV` 只能对应一种后端存储. 

![image-20200503125900857](http://files.luyanan.com//img/20200503125912.png)****

有了`StorageClass` 之后的`PVC` 可以变成这样. 

> ```yaml
> kind: PersistentVolumeClaim
> apiVersion: v1
> metadata:
> name: test-claim1
> spec:
> accessModes:
>     - ReadWriteMany
> resources:
>  requests:
>     storage: 1Mi
>   storageClassName: nfs
> ```

`StorageClass` 之所以能够动态供给`PV`, 是因为`Provisioner`，也就是`Dynamic Provisioning`

但是`NFS` 这种类型, k8s 中是默认没有`Provisioner` 插件的, 需要自己创建. 



##  8 `StorageClass` 实战

`github`：<https://github.com/kubernetes-incubator/external-storage/tree/master/nfs>

1. 准备一个`NFS` 服务器.并且确保可以正常工作,创建持久化需要的目录

   `path: /nfs/data/jack`

   `server: 121.41.10.13  `

   `比如mkdir -p /nfs/data/jack`

2. 创建资源

   `rbac.yaml`

   ```yaml
   kind: ClusterRole
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: nfs-provisioner-runner
   rules:
     - apiGroups: [""]
       resources: ["persistentvolumes"]
       verbs: ["get", "list", "watch", "create", "delete"]
     - apiGroups: [""]
       resources: ["persistentvolumeclaims"]
       verbs: ["get", "list", "watch", "update"]
     - apiGroups: ["storage.k8s.io"]
       resources: ["storageclasses"]
       verbs: ["get", "list", "watch"]
     - apiGroups: [""]
       resources: ["events"]
       verbs: ["create", "update", "patch"]
     - apiGroups: [""]
       resources: ["services", "endpoints"]
       verbs: ["get"]
     - apiGroups: ["extensions"]
       resources: ["podsecuritypolicies"]
       resourceNames: ["nfs-provisioner"]
       verbs: ["use"]
   ---
   kind: ClusterRoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: run-nfs-provisioner
   subjects:
     - kind: ServiceAccount
       name: nfs-provisioner
        # replace with namespace where provisioner is deployed
       namespace: default
   roleRef:
     kind: ClusterRole
     name: nfs-provisioner-runner
     apiGroup: rbac.authorization.k8s.io
   ---
   kind: Role
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: leader-locking-nfs-provisioner
   rules:
     - apiGroups: [""]
       resources: ["endpoints"]
       verbs: ["get", "list", "watch", "create", "update", "patch"]
   ---
   kind: RoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: leader-locking-nfs-provisioner
   subjects:
     - kind: ServiceAccount
       name: nfs-provisioner
       # replace with namespace where provisioner is deployed
       namespace: default
   roleRef:
     kind: Role
     name: leader-locking-nfs-provisioner
     apiGroup: rbac.authorization.k8s.io
   ```

   然后启动 `kubectl apply -f rbac.yaml`

3. `deployment.yaml`

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: nfs-provisioner
   ---
   kind: Deployment
   apiVersion: extensions/v1beta1
   metadata:
     name: nfs-provisioner
   spec:
     replicas: 1
     strategy:
       type: Recreate
     template:
       metadata:
         labels:
           app: nfs-provisioner
       spec:
         serviceAccount: nfs-provisioner
         containers:
           - name: nfs-provisioner
             image: registry.cn-hangzhou.aliyuncs.com/open-ali/nfs-client-provisioner
             volumeMounts:
               - name: nfs-client-root
                 mountPath: /persistentvolumes
             env:
               - name: PROVISIONER_NAME
                 value: example.com/nfs
               - name: NFS_SERVER
                 value: 192.168.56.10
               - name: NFS_PATH
                 value: /nfs/data/jack
         volumes:
           - name: nfs-client-root
             nfs:
               server: 192.168.56.10
               path: /nfs/data/jack
   ```

   启动`kubectl apply -f deployment.yaml`

4. `class.yaml`

   ```yaml
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     name: example-nfs
   provisioner: example.com/nfs
   ```

   启动  `kubectl apply -f class.yaml`

5. 创建`pvc` , `my-pvc.yaml`

   ```yaml
   kind: PersistentVolumeClaim
   apiVersion: v1
   metadata:
     name: my-pvc
   spec:
     accessModes:
       - ReadWriteMany
     resources:
       requests:
         storage: 1Mi
     # 这个名字要和上面创建的storageclass名称一致
     storageClassName: example-nfs
   ```

6. 创建`nginx` 资源 , `nginx-pod.yaml`

   ```yaml
   kind: Pod
   apiVersion: v1
   metadata:
     name: nginx
   spec:
     containers:
     - name: nginx
       image: nginx
       volumeMounts:
         - name: my-pvc
           mountPath: "/usr/jack"
     restartPolicy: "Never"
     volumes:
       - name: my-pvc
         persistentVolumeClaim:
           claimName: my-pvc
   ```

   ```bash
   kubectl apply -f nginx-pod.yaml
   kubectl exec -it nginx bash
   cd /usr/jack
   # 进行同步数据测试
   ```



## 9  `PV`的状态和回收策略

###  9.1 `PV`的状态

- `Available`: 表示当前的`PV` 没有被绑定
- `Bound`:  `PVC` 没有在使用`PV`,需要管理员手动释放`PV`
- `Released`:  资源回收失败



### 9.2 `PV` 回收策略

- `Retain`: 表示删除`PVC`的时候,`PV` 不会一起删除,而是变成`Released` 状态等待管理员手动清理
-  `Recycle`: 在`Kubernetes` 新版本就不用了,采用动态`PV` 的方式来替代
- `Delete`: 表示删除`PVC`的时候,`PV` 也会一起删除,同时也删除`PV` 所指向的实际存储地址. 

> 注意: 目前只有NFS和HostPath支持Recycle策略。AWS EBS、GCE PD、Azure Disk和Cinder支持Delete策略