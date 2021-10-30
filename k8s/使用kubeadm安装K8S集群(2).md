#  使用kubeadm安装K8S集群



## 1. 前提准备

### 1.1 官网介绍

官网:<https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl>

**Github**:https://github.com/kubernetes/kubeadm

**配置要求:**

- One or more machines running one of:
  - Ubuntu 16.04+
  - Debian 9+
  - CentOS 7【课程中使用】
  - Red Hat Enterprise Linux (RHEL) 7
  - Fedora 25+
  - HypriotOS v1.0.1+
  - Container Linux (tested with 1800.6.0)
- 2 GB or more of RAM per machine (any less will leave little room for your apps)
- 2 CPUs or more
- Full network connectivity between all machines in the cluster (public or private network is fine)
- Unique hostname, MAC address, and product_uuid for every node. See here for more details.
- Certain ports are open on your machines. See here for more details.
- Swap disabled. You **MUST** disable swap in order for the kubelet to work properly.



### 1.2. 版本

这里统一版本

```text
Docker       18.09.0
---
kubeadm-1.14.0-0 
kubelet-1.14.0-0 
kubectl-1.14.0-0
---
k8s.gcr.io/kube-apiserver:v1.14.0
k8s.gcr.io/kube-controller-manager:v1.14.0
k8s.gcr.io/kube-scheduler:v1.14.0
k8s.gcr.io/kube-proxy:v1.14.0
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
---
calico:v3.9
```





###  1.3 服务器准备

这里准备两台centos,一个`master`,一个`slave`, 要保证这三台服务器在同一个网络中,机器的配置也要按照上面的要求

#### 1.3.1  虚拟机创建

这是使用`VirtualBox` 创建三台虚拟机,

`Vagrantfile` 文件

```shell
boxes = [
	{
		:name => "master-kubeadm-k8s",
		:mem => "2048",
		:cpu => "2",
		:sshport => 22230
	},
	{
		:name => "worker01-kubeadm-k8s",
		:mem => "2048",
		:cpu => "2",
		:sshport => 22231
	},
	{
		:name => "worker02-kubeadm-k8s",
		:mem => "2048",
		:cpu => "2",
		:sshport => 22232
	}
]
Vagrant.configure(2) do |config|
	config.vm.box = "centos/7"
	boxes.each do |opts|
		config.vm.define opts[:name] do |config|
			config.vm.hostname = opts[:name]
			config.vm.network :public_network, ip: opts[:eth1]
			config.vm.network "forwarded_port", guest: 22, host: 2222, id: "ssh", disabled: "true"
		config.vm.network "forwarded_port", guest: 22, host: opts[:sshport]
			config.vm.provider "vmware_fusion" do |v|
				v.vmx["memsize"] = opts[:mem]
				v.vmx["numvcpus"] = opts[:cpu]
			end
			config.vm.provider "virtualbox" do |v|
				v.customize ["modifyvm", :id, "--memory", opts[:mem]]
			v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
				v.customize ["modifyvm", :id, "--name", opts[:name]]
			end
		end
	end
end
```



然后在当前目录使用`vagrant up` 启动

现在有三台服务区, 

- master-kubeadm-k8s: `192.168.56.10`
- worker01-kubeadm-k8s:`192.168.56.11`
- worker02-kubeadm-k8s:`192.168.56.12`

#### 1.3.2 设置root账号登录

启动之后进入到对应的`centos` 里面, 使得`root` 账户能够登录, 从而使用`xshell` 登录

```bash
	vagrant ssh master-kubeadm-k8s [进入manager-node]
	vagrant ssh worker01-kubeadm-k8s [进入worker01-node]
	vagrant ssh worker02-kubeadm-k8s [进入worker02-node]
```

分别进入三个节点,执行下面的操作,改成可以使用密码登录`root` 账号

```bash
sudo -i [进入root账户]
	vi /etc/ssh/sshd_config [设置root账户可以密码登陆]
		修改PasswordAuthentication yes
	passwd [修改密码]
	systemctl restart sshd [重启sshd服务]
```



### 1.4 更新并安装依赖

> 3台机器都需要执行的

```bash
yum -y update
yum install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp
```



### 1.5 安装Docker

`3台机器都需要安装Docker,版本为18.09.0`

####  1.5.1  移除旧的依赖

```bash
sudo yum remove docker docker latest docker-latest-logrotate \
    docker-logrotate docker-engine docker-client docker-client-latest docker-common
```



####  1.5.2 安装必要依赖

```bash
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```



#### 1.5.3 添加软件源信息

```bash
sudo yum-config-manager \
    --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

####  1.5.4  配置阿里云镜像加速器(如不需要刻意跳过)

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
"registry-mirrors": ["这边替换成自己的实际地址"]
}
EOF
sudo systemctl daemon-reload
```



####  1.5.5  更新`yum` 缓存

```bash
sudo yum makecache fast
```



####  1.5.6 安装docker

```bash
 ##  [指定安装docker版本]
sudo yum install -y docker-ce-18.09.0 docker-ce-cli-18.09.0 containerd.io 

```



#### 1.5.7  启动并设置开机启动

```bash
sudo systemctl start docker && sudo systemctl enable docker

```



#### 1.5.8 测试docker是否安装成功

```bash
    sudo docker run hello-world
```



### 1.6 修改`hosts` 文件



```bash
01 `master`
# 设置master的hostname，并且修改hosts文件
	sudo hostnamectl set-hostname m
02 `两个worker`
# 设置worker01/02的hostname，并且修改hosts文件
	sudo hostnamectl set-hostname w1
	sudo hostnamectl set-hostname w2
03 `三台机器`
	vi /etc/hosts
# ====================================================================================
192.168.56.10 m
192.168.56.11 w1
192.168.56.12 w2
# ====================================================================================
04 `使用ping测试一下`
	ping m
	ping w1
	ping w2
```



### 1.7  系统基础前提配置

#### 1.7.1 关闭防火墙

```bash
systemctl stop firewalld && systemctl disable firewalld

```



#### 1.7.2 关闭`selinux`

```bash
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```



#### 1.7.3 关闭`swap`

```bash
swapoff -a
sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
```



#### 1.7.4  配置`iptables`的`ACCEPT`规则

```bash
iptables -F && iptables -X && iptables \
    -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT
```



#### 1.7.5 设置系统参数

```bash
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```





##   2. 安装K8S(`kubeadm`, `kubelet` and `kubectl`)

###  2.1 配置`yum` 源

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```



### 2.2 安装`kubeadm`&`kubelet`&`kubectl`

```bash
yum install -y kubeadm-1.14.0-0 kubelet-1.14.0-0 kubectl-1.14.0-0

```



###  2.3 `docker`和`k8s`设置同一个`cgroup`

```bash
# docker
vi /etc/docker/daemon.json
    "exec-opts": ["native.cgroupdriver=systemd"],
    
systemctl restart docker
    
# kubelet，这边如果发现输出directory not exist，也说明是没问题的，大家继续往下进行即可
sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
	
systemctl enable kubelet && systemctl start kubelet
```



###  2.4 `proxy`/`pause`/`scheduler`等国内镜像



#### 2.4.1 查看`kubeadm` 使用的镜像

```bash
kubeadm config images list
```

```text
k8s.gcr.io/kube-apiserver:v1.14.0
k8s.gcr.io/kube-controller-manager:v1.14.0
k8s.gcr.io/kube-scheduler:v1.14.0
k8s.gcr.io/kube-proxy:v1.14.0
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
```

发现这里面的都是国外的一些镜像. 



#### 2.4.2   解决国外镜像不能访问的问题

我们可以使用阿里云的镜像进行下载

1. 创建一个`kubeadm.sh`  脚本, 用于拉取镜像/打`tag` /删除原有的镜像

```bash
#!/bin/bash

set -e

KUBE_VERSION=v1.14.0
KUBE_PAUSE_VERSION=3.1
ETCD_VERSION=3.3.10
CORE_DNS_VERSION=1.3.1

GCR_URL=k8s.gcr.io
ALIYUN_URL=registry.cn-hangzhou.aliyuncs.com/google_containers

images=(kube-proxy:${KUBE_VERSION}
kube-scheduler:${KUBE_VERSION}
kube-controller-manager:${KUBE_VERSION}
kube-apiserver:${KUBE_VERSION}
pause:${KUBE_PAUSE_VERSION}
etcd:${ETCD_VERSION}
coredns:${CORE_DNS_VERSION})

for imageName in ${images[@]} ; do
  docker pull $ALIYUN_URL/$imageName
  docker tag  $ALIYUN_URL/$imageName $GCR_URL/$imageName
  docker rmi $ALIYUN_URL/$imageName
done
```

2. 运行脚本和查看镜像

    ```bash
   # 运行脚本
   sh ./kubeadm.sh
   
   # 查看镜像
   docker images
   ```

3. 将这些镜像推送到自己的阿里云仓库(可选)

    登陆阿里云仓库

   ```bash
   # 登录自己的阿里云仓库
   docker login --username=xxx registry.cn-hangzhou.aliyuncs.com
   ```

   编写`kubeadm-push-aliyun.sh` 用于推送镜像

   ```bash
   #!/bin/bash
   
   set -e
   
   KUBE_VERSION=v1.14.0
   KUBE_PAUSE_VERSION=3.1
   ETCD_VERSION=3.3.10
   CORE_DNS_VERSION=1.3.1
   
   GCR_URL=k8s.gcr.io
   ALIYUN_URL=xxx
   
   images=(kube-proxy:${KUBE_VERSION}
   kube-scheduler:${KUBE_VERSION}
   kube-controller-manager:${KUBE_VERSION}
   kube-apiserver:${KUBE_VERSION}
   pause:${KUBE_PAUSE_VERSION}
   etcd:${ETCD_VERSION}
   coredns:${CORE_DNS_VERSION})
   
   for imageName in ${images[@]} ; do
     docker tag $GCR_URL/$imageName $ALIYUN_URL/$imageName
     docker push $ALIYUN_URL/$imageName
     docker rmi $ALIYUN_URL/$imageName
   done
   ```

    运行脚本

   ```bash
    sh ./kubeadm-push-aliyun.sh
   ```



参照上面的步骤,能凑成一个`k8s-environment-install.sh`  环境安装的脚本

```bash
##  kudeadm 安装master节点
echo "更新并安装依赖"
yum -y update
yum install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp

echo "卸载docker"
sudo yum remove docker docker latest docker-latest-logrotate \
    docker-logrotate docker-engine docker-client docker-client-latest docker-common

echo "安装必要的依赖"
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

echo "添加软件源信息"

sudo yum-config-manager \
    --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

echo "配置阿里云镜像加速器,并设置cgroup"
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://oqnpta2w.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload

echo "更新yum缓存"

sudo yum makecache fast

echo "安装docker"

sudo yum install -y docker-ce-18.09.0 docker-ce-cli-18.09.0 containerd.io 

echo "docker 启动并设置开机自启"

sudo systemctl start docker && sudo systemctl enable docker

echo "验证Docker"

docker version

echo "关闭防火墙"
systemctl stop firewalld && systemctl disable firewalld

echo "关闭selinux"
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

echo "关闭swap"
swapoff -a
sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab

echo " 配置iptables的ACCEPT规则"
iptables -F && iptables -X && iptables  \
    -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT

echo "设置系统参数"
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system


echo "安装K8S(kubeadm, kubelet and kubectl)"
echo "配置yum 源"

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

echo "安装kubeadm&kubelet&kubectl"
yum install -y kubeadm-1.14.0-0 kubelet-1.14.0-0 kubectl-1.14.0-0

echo "docker和k8s设置同一个cgroup"
# kubelet，这边如果发现输出directory not exist，也说明是没问题的，大家继续往下进行即可
sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

systemctl enable kubelet && systemctl start kubelet

echo "查看kubeadm 使用的镜像"
kubeadm config images list
echo "使用阿里云镜像进行下载"
set -e

KUBE_VERSION=v1.14.0
KUBE_PAUSE_VERSION=3.1
ETCD_VERSION=3.3.10
CORE_DNS_VERSION=1.3.1

GCR_URL=k8s.gcr.io
ALIYUN_URL=registry.cn-hangzhou.aliyuncs.com/google_containers

images=(kube-proxy:${KUBE_VERSION}
kube-scheduler:${KUBE_VERSION}
kube-controller-manager:${KUBE_VERSION}
kube-apiserver:${KUBE_VERSION}
pause:${KUBE_PAUSE_VERSION}
etcd:${ETCD_VERSION}
coredns:${CORE_DNS_VERSION})

for imageName in ${images[@]} ; do
  docker pull $ALIYUN_URL/$imageName
  docker tag  $ALIYUN_URL/$imageName $GCR_URL/$imageName
  docker rmi $ALIYUN_URL/$imageName
done

echo "镜像查看"
docker images


```

上述脚本执行完了之后, 要在各个服务器添加`hostname`,修改hosts文件,

###  2.5  `kube` `init`  初始化`master

#### 2.5.1 `kube `    `init  流程

1. 进行一系列的检查,以确定这台机器可以部署`kebernetes`

2.  生成`kubernetes`  对外提供服务所需要的各种证书和对应目录

3. 为其他组件生成访问`kube-ApiServer` 所需的配置文件

   ```bash
       ls /etc/kubernetes/
       admin.conf  controller-manager.conf  kubelet.conf  scheduler.conf
   ```

4. 为`Master` 组件生成`prod` 配置文件

   ```bash
       ls /etc/kubernetes/manifests/*.yaml
       kube-apiserver.yaml 
       kube-controller-manager.yaml
       kube-scheduler.yaml
   ```

5. 生成`etcd` 的`Prod YAML` 文件

   ```bash
     ls /etc/kubernetes/manifests/*.yaml
       kube-apiserver.yaml 
       kube-controller-manager.yaml
       kube-scheduler.yaml
   	etcd.yaml
   ```

6. 一旦这些`YAML` 文件出现在被`kubernetes` 监视的`/etc/kubernetes/manifests/`  目录下, `kubelet` 就会自动创建这些`yaml` 文件定义的`prod`,即`master` 组件的容器。 `master` 容器启动后,`kubeam` 就会通过检查`localhost：6443/healthz`  这个`master` 组件的健康状态检查`URL`,等待`master` 组件完全运行起来

7. 为集群生成一个`bootstrap token`

8. 将`ca.crt` 等`Master` 节点的重要信息,通过`configMap` 的方式保存在`edcd` 中,以供后续`node`节点使用

9. 最后一步是安装默认插件,`kubernetes`默认`kebe-proxy` 和`dns` 插件是必须安装的. 





#### 2.5.2  初始化`master`节点

官网: <https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/>

> 注意: 此操作是在主节点上执行

```bash
# 本地有镜像
kubeadm init --kubernetes-version=1.14.0 --apiserver-advertise-address=192.168.56.10 --pod-network-cidr=10.244.0.0/16
【若要重新初始化集群状态：kubeadm reset，然后再进行上述操作】
```

**切记,一定要保存要`kubeadm join`的信息**,这是我自己的

```bash
kubeadm join 192.168.56.10:6443 --token m3hnyu.rxs97uf6fq7mknk5 \
    --discovery-token-ca-cert-hash sha256:10a5b6b85f42b2fb6391d0e8fbb3c3dc35ea391376a963378670c93518600490
```



#### 2.5.3 根据日志提示

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

此时 `bubectl cluster -info` 查看一下是否成功. 



####  2.5.4  查看`prod`验证一下

等待一会儿, 可以发现像 `etc`,`controller`，`scheduler` 等组件都以`prod`的方式安装成功了. 

```bash
[root@master-kubeadm-k8s k8s]# kubectl get pods -n kube-system
NAME                                         READY   STATUS    RESTARTS   AGE
coredns-fb8b8dccf-lnhf8                      0/1     Pending   0          2m31s
coredns-fb8b8dccf-xzrvk                      0/1     Pending   0          2m31s
etcd-master-kubeadm-k8s                      1/1     Running   0          111s
kube-apiserver-master-kubeadm-k8s            1/1     Running   0          103s
kube-controller-manager-master-kubeadm-k8s   1/1     Running   0          115s
kube-proxy-g6lts                             1/1     Running   0          2m30s
kube-scheduler-master-kubeadm-k8s            1/1     Running   0          105s
```



> 注意： `coredns`  没有安装启动,需要安装网络插件. 



#### 2.5.4  健康检查

```bash
curl -k https://localhost:6443/healthz
```



### 2.6  部署`calico`网络插件

选择网络插件: ：<https://kubernetes.io/docs/concepts/cluster-administration/addons/>

calico网络插件：<https://docs.projectcalico.org/v3.9/getting-started/kubernetes/>

> `calico` 同样在`master` 节点上执行

安装

```bash
kubectl apply -f https://docs.projectcalico.org/v3.9/manifests/calico.yaml

```

验证

```bash

# 确认一下calico是否安装成功
kubectl get pods --all-namespaces -w
```



###  2.7 `kebu joib`

> 记得保存初始化`master`节点的最后打印信息,

```bash
kubeadm join 192.168.56.10:6443 --token m3hnyu.rxs97uf6fq7mknk5 \
    --discovery-token-ca-cert-hash sha256:10a5b6b85f42b2fb6391d0e8fbb3c3dc35ea391376a963378670c93518600490
```

#### 2.7.1  在`worker1` 和`worker2`上执行命令

```bash
kubeadm join 192.168.56.10:6443 --token m3hnyu.rxs97uf6fq7mknk5 \
    --discovery-token-ca-cert-hash sha256:10a5b6b85f42b2fb6391d0e8fbb3c3dc35ea391376a963378670c93518600490
```



####  2.7.2  在`master`节点上检查集群信息

```bash
[root@master-kubeadm-k8s k8s]# kubectl get nodes
NAME                   STATUS     ROLES    AGE     VERSION
master-kubeadm-k8s     Ready      master   8m20s   v1.14.0
worker01-kubeadm-k8s   NotReady   <none>   20s     v1.14.0
worker02-kubeadm-k8s   NotReady   <none>   28s     v1.14.0

```



###  2.8 体验`Prod`

#### 2.8.1  定义`prod` 文件, 比如`prod_nginx_rs.yaml`

```yaml
cat > pod_nginx_rs.yaml <<EOF
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      name: nginx
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```



####  2.8.2 根据`prod_nginx_rs.yaml` 文件创建`prod`

```bash
kubectl apply -f pod_nginx_rs.yaml
```

####  2.8.3  查看`prod`

```bash
kubectl get pods
kubectl get pods -o wide
kubectl describe pod nginx
```



####  2.8.4  感受通过`rs` 将 `prod` 进行扩容

```bash
kubectl scale rs nginx --replicas=5
kubectl get pods -o wide
```




#### 2.8.5  删除`prod`

```bash
kubectl delete -f pod_nginx_rs.yaml
```



## 3. 常见问题

### 3.1 alico网络插件安装小技巧

文档中我们是通过kubectl apply -f https://docs.projectcalico.org/v3.9/manifests/calico.yaml
进行安装的，其实说白了也就是根据一个calico.yaml文件创建K8s中对应的资源，大家可以把这个文件下载下来看一下，发现里面是有用到几个image的，如果直接apply， 需要有pull image的过程，可能比较慢，或者拉取镜像失败。大家可以这样，根据查看到的image的版本，把里面所需要的image先通过docker pull拉取下来，然后再apply。比如:
docker pull calico/pod2daemon-flexvol:vxxx
docker pull calico/cni:vxxx
…

### 3.2  kubeadm init生成的join信息忘记了或者过期了怎么办？

有些小伙伴可能没有及时保存最后的join信息，或者24小时之后过期了，这时候可以重新生成
（1）重新生成token

```
kubeadm token create
```

（2）获取ca证书sha256编码hash值

```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

（3）重新生成join信息

```
kubeadm join 主节点ip地址:6443 --token token填这里  --discovery-token-ca-cert-hash sha256:哈希值填这里
```