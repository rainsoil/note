#   Service Mesh初入门

## 1.  分布式架构发展史

### 1.1  单机小型机时代

- 1969年,阿帕网诞生,最初是为了军事目的,后来演变成了`Inernet`

- 2000年左右,互联网在中国盛行起来,但是那时候网民数量较少,所以多数企业服务单一,采用集中式部署的方式就能满足需求

  ![image-20200505165154580](http://files.luyanan.com//img/20200505165155.png)

- 一旦小型机或者数据库出现问题,会导致整个系统的故障,而且功能修改发布也不方便,所以不妨把大系统拆分成多个子系统, 比如`用户` 、`商品` 、`论坛`等, 也就是“垂直拆分”, 并且每个子系统都有各自的硬件

  ![image-20200505165357739](http://files.luyanan.com//img/20200505165359.png)



### 1.2 集群化负载均衡时代

- 用户量越来越大,就意味着需要更多的小型机,价格昂贵、操作维护成本高, 不妨把小型机换成普通的PC机,采用多台PC机部署同一个应用的方案,但是此时就需要对这些应用做负载均衡, 因为客户端不知道请求哪一个. 

  > 2013年5月17 日, 阿里集团最后一台IBM 小型机在支付宝下线
  >
  > 这是自2009年"去IOE" 战略以来,“去IOE” 非常重要的一个节点。"去IOE" 指的是摆脱掉IT部署中原有的IBM小型机、`Oracle` 数据库以及`EMC`存储的过度依赖,告别最后一台小型机,意味着整个阿里集团尽管还有一些`Oracle` 数据库和`EMC` 存储,但是`IBM` 小型机已经全部消失了. 

- 负载均衡可以分为硬件和软件,硬件比如`F5`,软件比如`LVS(Linux Virtual Server)` . 负载均衡的思路是对外暴露一个统一的接口,然后根据用户的请求进行对应规则的转发,同时负载均衡还可以做健康检查、限流等

- 有了负载均衡, 后端的应用就可以根据流量的大小进行动态的扩缩容了, 也就是“水平扩展”

![image-20200505171951282](http://files.luyanan.com//img/20200505171952.png)



### 1.3 服务化时代

- 此时发现在用户、订单、商品等有重复的功能,比如登陆、注册、邮件等,一旦项目大了之后, 集群部署多了, 这些重复的功能无疑是浪费资源, 不妨把重复的功能抽离出来, 起个名字叫做"服务Service"

- 其实对于"服务"现在已经比较广泛了, 比如"基础设施即服务`laas`"、"平台即服务`PaaS`" 、"软件即服务`Saas`"等

   ![image-20200505172345052](http://files.luyanan.com//img/20200505172352.png)

- 这时候就急需解决的就是服务之间的调用问题,`RPC(Remote Pricedure Call)`, 同时这种调用关系得维护起来,比如某个服务在哪里? 是不是得知道? 所以不仅仅要解决服务调用的问题,还要解决服务治理的问题,比如像`Dubbo`, 默认采用`Zookeeper` 作为注册中心,`SpringCloud` 使用`Eureka` 作为注册中心

- 当然,要关注的还有很多,比如限流、降级、负载均衡、容错等



### 1.4 分布式微服务时代

- 在分布式架构下,服务可能拆分的没有那么细, 可以进一步的拆分

- `Microservices`

   https://martinfowler.com/articles/microservices.html

- Java 中微服务主流解决方案`SpringCloud`，`SpringCloud` 可以用过引入几个注解,写少量的代码完成微服务架构中的开发,相对`Dubbo`而言简单方便很多, 并且使用的是`Http`协议,所以对多语言也可以很好的支持

  https://spring.io/projects/spring-cloud

  > spring-cloud-bus 
  >
  > spring-cloud-consul 
  >
  > spring-cloud-config 
  >
  > spring-cloud-netflix:eureka、
  >
  > hystrix、
  >
  > feign、
  >
  > zuul、
  >
  > ribbon等 .....

### 1.5 服务网格时代

然后微服务时代有了`SpringCloud` 就完美了吗? 不妨想一想会有哪些问题? 

1. 最初是为了业务而写代码,比如登陆功能,支付功能等,到后面会发现要解决网络通信的问题,虽然有`SpringCloud` 里面的组件帮我们解决了, 但是细想一下他是怎么解决的? 引入`dependenct`,加注解,写配置,最后将这些非业务功能的代码打包一起部署,和业务代码一起,称为"侵入式代码"
2.  微服务中的服务支持不同语言开发,维护不同语言和非业务代码的成本
3. 业务代码开发者应该把更多的精力投入到业务熟悉度上,而不应该是非业务上,`SpringCloud` 虽然能解决微服务领域的很多问题,但是学习成本还是较大的. 
4. 对于`SpringCloud` 而言, 并不是微服务领域的所有问题都有对应的解决方案., 也就是功能受限,比如`Content Based Routing`和`Version Based Routing。`   当然可以将`SpringCloud`和`Service Mesh`结合起来使用,也就是对`SC` 的补充、扩展和加强等. 
5. 互联网公司产品的版本升级是非常频繁的,为了维护各个版本的兼容性、权限、流量等, 因为`SpringCloud` 是"代码侵入式的框架", 这时候版本的升级就注定要让非业务的代码一起,一旦出现问题, 再加上多语言之间的调用,工程师会非常痛苦的. 
6. 其实大家有没有发现,服务拆分的越细,只是感觉上轻量级解耦了, 但是维护成本越高了, 那怎么办呢? 网络的问题当然还是交给网络本身来解决。 



#### 1.5.1 问题解决思路

- 本质上要解决服务之间通信的问题,不应该将非业务的代码融合到业务代码中
- 也就是从客户端发出的请求, 要能够顺利到达对应的服务,这中间的网络通信的过程要和业务代码尽量无关. 

>  通信的问题: 服务发现、负载均衡、版本控制、蓝绿部署

- 在很早之前的单体架构中,其实通信问题也是写在业务代码中的, 那时候怎么解决的呢? 

  > 把网络通信、流量转发等问题放到了计算机网络模型中的TCP/UDP 层,也就是非业务功能代码下沉

- 那就不妨把这些通信的问题都交给计算机网络模型组织去解决咯? 别人肯定不会答应的, 怎么办呢?

  > 既然不答应,那我们就自己想办法去解决通信问题,撸一些产品岂不快哉? 没错,代理嘛.

- 不妨这样在帮每个服务配置一个代理,所有通信的问题都交给这个代理去解决

- 大家肯定接触过, 比如最初`Nginx`、`HaPorxy` 等,其实他们做反向代理把请求转发给其他服务器,也就为`Service Mesh` 的诞生和完成提供了一个解决思路. 

####  1.5.2 一些公司对于代理的探索`Sidecar`

很多公司借鉴了`Proxy` 模式, 推出了`Sidecar`的产品,比如像14年`Netflix`的`Prana`, 15年唯品会的`local proxy`.

其实`Sidecar` 模式和`Proxy` 很类似,但是`Sidecar`功能更全面,也就是`Sidecar` 功能和侵入式框架的功能对应

![image-20200505205031386](http://files.luyanan.com//img/20200505205337.png)

> 问题: 这种`Sidecar`  是为特定的基础设施而设计的,也就是跟公司原有的框架技术绑定在一起,不能成为通用性的产品, 所以还需要继续探索. 

### 1.5.3 `Service Mesh` 之`Linkerd`

> 2016年1月, 离开`Twitter`的基础设施工程师`William Morgan`和`Oliver Gould`, 在`github` 上发布了`linkerd 0.0.7` 版本, 从而第一个`Service Mesh` 项目由此诞生,并且`Linkerd` 是基于`Twitter`的`Finagle` 开源项目, 实现了通用性。 
>
> 后面又出现了`Service Mesh` 的第二个项目`Envoy`, 并且在17年的时候都加入了`CNCF` 项目

![image-20200505210907278](http://files.luyanan.com//img/20200505210908.png)

小结: `linderd` 解决了通用性问题, 在`linkerd` 思想要求所有的流量都走`Sidecar` , 而原来的`Sidecar` 方式可以走直接直连, 这样 一来`Linkerd` 就帮业务人员屏蔽了通信细节,也不需要侵入到业务代码中,让业务开发者更加专注业务本身, 

问题:  但是`Linkerd` 的设计思想在传统运维方式中太难部署和维护了,所以到后来就没有得到广泛的关注, 其实主要的问题是`Linkerd` 只是实现了数据层面的问题,但没有对其进行很好的管理. 

### 1.5.4 `Service Mesh` 之`Istio`

由`Google`、`IBM` 和`Lyft` 共同发起的开源项目,17年5月份的时候发布`0.1 release`  版本,17年10月发布`0.2 release`版本,18年7月份发布`1.0 release` 版本

很明显`Istio` 不仅拥有"数据层面(`Data Plane`)",还拥有"控制平面(`Control Plane`)", 也就是拥有了数据的接管和几种控制能力。 

> 官网: https://istio.io/docs/concepts/what-is-istio/#why-use-istio

> Istio makes it easy to create a network of deployed services with load balancing, service-to-service authentication, monitoring, and more, with [few](https://istio.io/docs/tasks/observability/distributed-tracing/overview/#trace-context-propagation) or no code changes in service code. You add Istio support to services by deploying a special sidecar proxy throughout your environment that intercepts all network communication between microservices, then configure and manage Istio using its control plane functionality, which includes:



## 2. `Service Mesh`

###  2.1 What's a service mesh[William]

> William Morgan :https://buoyant.io/2017/04/25/whats-a-service-mesh-and-why-do-i-need-one

> A service mesh is a dedicated infrastructure layer for making service-to-service communication safe, fast, and reliable. If you’re building a cloud native application, you need a service mesh.

### 2.2 What is a service mesh?[Istio]

> istio官网 ：https://istio.io/docs/concepts/what-is-istio/#what-is-a-service-mesh

> The term service mesh is used to describe the network of microservices that make up such applications and the interactions between them. As a service mesh grows in size and complexity, it can become harder to understand and manage. Its requirements can include discovery, load balancing, failure recovery, metrics, and monitoring. A service mesh also often has more complex operational requirements, like A/B testing, canary rollouts, rate limiting, access control, and end-to-end authentication.

### 2.3 `Linkerd`  和`Istio` 发展历程

- `Mircroservices`

  > Martin Fowler 
  >
  > 14年提出 
  >
  > https://martinfowler.com/articles/microservices.html

- `Linkerd`

  > William Morgan[Buoyant] 
  >
  > Scala语言编写，运行在JVM中，
  >
  > Service Mesh名词的创造者 
  >
  > 16年01月15号，0.0.7发布 
  >
  > 17年01月23号，加入CNCF 
  >
  > 17年04月25号，1.0版本发布

- `Envoy`

  > C++语言编程[Lyft] 
  >
  > 16年09月13号，1.0发布 
  >
  > 17年09月14号，加入CNCF

- `Istio`

  > Google、IBM、Lyft发布0.1版本

### 2.4 国内发展情况

- 蚂蚁金服的`SOFA`

  > 全称Scalable Open Financial Architecture 
  >
  > 前身是SOFA RPC 
  >
  > 18年07月正式开源

- 腾讯的`Tencent Service Mesh`

- 华为的`CSE Mesher`



总结: 基本上都是借鉴了`Sidecar`、`Envoy` 和`Istio`的设计思想

> 最终目录: 
>
> `TCP/IP`协议解决了计算机之间连接的问题, 其实我们很少感知到它的存在
>
> 而目前的服务治理也面临相似的问题, 也就是要让计算机中的服务更好的连接起来,而且要做到业务代码尽可能无感知. 



### 2.5   为何`Service Mesh` 能够迅速走红? 

随着微服务和容器化技术的发展, 使得开发和运维的方式变得非常方便

从而让服务能够方便的迁移到不同的云平台上. 



### 2.6  回顾一下`Service Mesh` 的发展历程

- 借鉴反向代理,对`Proxy` 模式进行探索,让非业务的代码下沉
- 使用`Sidecar` 弥补`Proxy` 模式功能的不足, 解决"侵入式框架" 中非业务代码的问题
- `Linkerd` 解决了传统`Sidecar` 模式中通用性的问题
- `Istio` 增加了控制平面,解决了整个系统中的流量完全控制的问题



## 3. `Istio`

基于`Sidecar` 模式, 数据平面和控制平台,主流`Service Mesh`  解决方案

> 官网: https://istio.io/
>
> `github`: https://github.com/istio



### 3.1  整体感受

数据平面和控制平面

`Sidecar` 的方式解决了数据通信的问题,而在`Istio` 中加入了控制平面,所有的流量都能被有效的控制,也就是通过控制平面可以控制整个系统. 

![image-20200505215921196](http://files.luyanan.com//img/20200505215922.png)



###  3.2  在`Istio` 中到底能解决哪些问题? 

1. 针对`Http`、`gRPC`、`WebSocket` 等协议的自动负载均衡
2. 故障的排查、应用的容错、路由
3. 流量控制、全链路安全访问控制和认证
4. 请求监测、日志分析以及全链路跟踪
5. 应用的升级发布、频率限制和配合等. 

![image-20200505221246910](http://files.luyanan.com//img/20200505221248.png)



### 3.3 `Architecture`

有了感性的认知和功能了解之后,接下来就要考虑落地的问题,也就是`Istio`的架构图该如何设计? 

说白了, 就是根据数据平面和控制平面的理念,该怎么设计架构图呢? 这个官方已经帮我们考虑好了

`Architecutre`: https://istio.io/docs/ops/deployment/architecture/

> An Istio service mesh is logically split into a **data plane** and a **control plane**.
>
> - The **data plane** is composed of a set of intelligent proxies ([Envoy](https://www.envoyproxy.io/)) deployed as sidecars. These proxies mediate and control all network communication between microservices. They also collect and report telemetry on all mesh traffic.
> - The **control plane** manages and configures the proxies to route traffic.

![image-20200505221700348](http://files.luyanan.com//img/20200505221705.png)



## 4.  安装`Istio`

### 4.1 `Getting Started`

> 官网: https://istio.io/docs/setup/getting-started/
>
> - Set up your platform 
> - Download the release 
> - Install Istio

#### 4.1.1 `Set up your platform`

这里使用`kubernetes 1.14 版本`



####  4.1.2 `Download the release`

1. `Go to the Istio release page...`

    放到`master`节点

   > 方式一: 到`github` 上下载

   https://github.com/istio/istio/releases

   >  方式二: macOs or Linux System直接下载，这个地址下载国内下载会很慢，不推荐

   `curl -L https://istio.io/downloadIstio | sh -`

2. 解压 `tar -zxvf istio-1.x.y.tar.gz`, 进入到`Istio` 文件目录下

   > The installation directory contains: 
   >
   > - Installation YAML files for Kubernetes in install/kubernetes 
   > - Sample applications in samples/ 
   > - The istioctl client binary in the bin/ directory. istioctl is used when manually injecting Envoy as a sidecar proxy.

3. 将`itsioctl` 添加到环境变量中

    ```bash
   export PATH=$PWD/bin:$PATH
   
   ```



#### 4.1.3   ` Install Istio`

##### 4.1.3.1   `Kubernetes CRD`

`kubernetes`  平台对于分布式服务部署的很多重要的模块都有系统性的支持,借助如下一些平台资源可以满足大多数分布式系统部署和管理的需求

但是在不同应用业务环境下,对于平台可能有一些特殊的需求,这些需求可以抽象为`kubernetes`的扩展资源,而`kubernetes` 的`CRD(CustomResourceDefinition)` 为这样的需求提供了轻量级的机制,保证新的资源的快速注册和使用. 在更老的版本中, `TPR(ThirdPartyResource)` 是与`CRD` 类似的概念, 但是在1.9 以上的版本中被弃用,而`CRD` 则进入到`beta`的状态. 

> `Istio`就是使用`CRD`在`Kubernetes`上建构出一层`Service Mesh`的实现
>
> `istio-1.0.6/install/kubernetes/helm/istio/templates/crds.yaml`



#####  4.1.3.2  准备镜像

> `istio-1.0.6/install/kubernetes/istio-demo.yaml`

上述`istio-demo.yaml` 文件 中有很多镜像速度较慢,大家记得下载我的,然后tag, 最好保存到自己的镜像仓库中,以便日后使用

1.  从阿里云镜像仓库下载

   ```bash
   docker pull registry.cn-hangzhou.aliyuncs.com/l_third_party/proxy_init:1.0.6
   docker pull registry.cn-hangzhou.aliyuncs.com/l_third_party/hyperkube:v1.7.6_coreos.0
   docker pull registry.cn-hangzhou.aliyuncs.com/l_third_party/galley:1.0.6
   docker pull registry.cn-hangzhou.aliyuncs.com/l_third_party/proxyv2:1.0.6
   docker pull registry.cn-hangzhou.aliyuncs.com/l_third_party/grafana:5.2.3
   docker pull registry.cn-hangzhou.aliyuncs.com/l_third_party/mixer:1.0.6
   docker pull registry.cn-hangzhou.aliyuncs.com/l_third_party/pilot:1.0.6
   docker pull registry.cn-hangzhou.aliyuncs.com/l_third_party/prometheus:v2.3.1
   docker pull registry.cn-hangzhou.aliyuncs.com/l_third_party/citadel:1.0.6
   docker pull registry.cn-hangzhou.aliyuncs.com/l_third_party/servicegraph:1.0.6
   docker pull registry.cn-hangzhou.aliyuncs.com/l_third_party/sidecar_injector:1.0.6
   docker pull registry.cn-hangzhou.aliyuncs.com/l_third_party/all-in-one:1.5
   ```

2. 将阿里云仓库的镜像打成原有的tag

   ```bash
   docker tag registry.cn-hangzhou.aliyuncs.com/l_third_party/proxy_init:1.0.6 docker.io/istio/proxy_init:1.0.6
   docker tag registry.cn-hangzhou.aliyuncs.com/l_third_party/hyperkube:v1.7.6_coreos.0 quay.io/coreos/hyperkube:v1.7.6_coreos.0
   docker tag registry.cn-hangzhou.aliyuncs.com/l_third_party/galley:1.0.6 docker.io/istio/galley:1.0.6
   docker tag registry.cn-hangzhou.aliyuncs.com/l_third_party/proxyv2:1.0.6 docker.io/istio/proxyv2:1.0.6
   docker tag registry.cn-hangzhou.aliyuncs.com/l_third_party/grafana:5.2.3 grafana/grafana:5.2.3
   docker tag registry.cn-hangzhou.aliyuncs.com/l_third_party/mixer:1.0.6 docker.io/istio/mixer:1.0.6
   docker tag registry.cn-hangzhou.aliyuncs.com/l_third_party/pilot:1.0.6 docker.io/istio/pilot:1.0.6
   docker tag registry.cn-hangzhou.aliyuncs.com/l_third_party/prometheus:v2.3.1 docker.io/prom/prometheus:v2.3.1
   docker tag registry.cn-hangzhou.aliyuncs.com/l_third_party/citadel:1.0.6 docker.io/istio/citadel:1.0.6
   docker tag registry.cn-hangzhou.aliyuncs.com/l_third_party/servicegraph:1.0.6 docker.io/istio/servicegraph:1.0.6
   docker tag registry.cn-hangzhou.aliyuncs.com/l_third_party/sidecar_injector:1.0.6 docker.io/istio/sidecar_injector:1.0.6
   docker tag registry.cn-hangzhou.aliyuncs.com/l_third_party/all-in-one:1.5 docker.io/jaegertracing/all-in-one:1.5
   ```

3. 删除阿里云仓库下载的镜像

   ```bash
   docker rmi registry.cn-hangzhou.aliyuncs.com/l_third_party/proxy_init:1.0.6
   docker rmi registry.cn-hangzhou.aliyuncs.com/l_third_party/hyperkube:v1.7.6_coreos.0
   docker rmi registry.cn-hangzhou.aliyuncs.com/l_third_party/galley:1.0.6
   docker rmi registry.cn-hangzhou.aliyuncs.com/l_third_party/proxyv2:1.0.6
   docker rmi registry.cn-hangzhou.aliyuncs.com/l_third_party/grafana:5.2.3
   docker rmi registry.cn-hangzhou.aliyuncs.com/l_third_party/mixer:1.0.6
   docker rmi registry.cn-hangzhou.aliyuncs.com/l_third_party/pilot:1.0.6
   docker rmi registry.cn-hangzhou.aliyuncs.com/l_third_party/prometheus:v2.3.1
   docker rmi registry.cn-hangzhou.aliyuncs.com/l_third_party/citadel:1.0.6
   docker rmi registry.cn-hangzhou.aliyuncs.com/l_third_party/servicegraph:1.0.6
   docker rmi registry.cn-hangzhou.aliyuncs.com/l_third_party/sidecar_injector:1.0.6
   docker rmi registry.cn-hangzhou.aliyuncs.com/l_third_party/all-in-one:1.5
   ```

#####  4.1.3.4  安装`Istio` 核心组件

1. 根据 `istio-1.0.6/install/kubernetes/istio-demo.yaml` 创建资源

   ```bash
   kubectl apply -f istio-demo.yaml
   
   ```

2. 查看核心组件资源

   ```bash
   kubectl get pods -n istio-system
   kubectl get svc -n istio-system # 可以给某个service配置一个ingress规则，访问试试
   
   ```

这样一来,`Istio`的核心组件就安装完成了. 



### 4.2  初步感受`Istio`

#### 4.2.1  手动注入`Sidecar`

大家都知道,k8s 是通过`pod` 来部署业务的,流量的进出都是直接跟`pod` 打交道的

那在`Istio` 中如何体现`Sidecar`的作用呢？ 

`思路:` 在`pod` 中除了业务的`contailer`,再增加一个`contailer`为`Sidecar` 岂不快哉

1. 准备一个资源文件 `first-istio.yaml`

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: first-istio
   spec:
     selector:
       matchLabels:
         app: first-istio
     replicas: 1
     template:
       metadata:
         labels:
           app: first-istio
       spec:
         containers:
         - name: first-istio
           image: registry.cn-hangzhou.aliyuncs.com/luyanan/test-docker-image:v1.0
           ports:
           - containerPort: 8080  
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: first-istio
   spec:
     ports:
     - port: 80
       protocol: TCP
       targetPort: 8080
     selector:
       app: first-istio
     type: ClusterIP
   ```

   ```bash
   kubectl apply -f first-istio.yaml
   kubectl get pods -> # 注意该pod中容器的数量
   kubectl get svc
   curl 192.168.80.227:8080/dockerfile
   curl 10.105.179.205/dockerfile
   ```

2. 删除上面的资源,重新创建, 使用手动注入`Sidecar`的方式

   ```bash
   kubectl delete -f first-istio.yaml
   istioctl kube-inject -f first-istio.yaml | kubectl apply -f -
   ```

3. 查看资源

   ```bash
   kubectl get pods -> # 注意该pod中容器的数量
   kubectl get svc
   kubectl describe pod first-istio-cc5d65fc-rzdxx -> # Containers:first-istio & istio-proxy
   ```

4. 删除资源

   ````bash
   istioctl kube-inject -f first-istio.yaml | kubectl delete -f -
   
   ````

这样一来,就手动给`pod` 注入了`sidecar`的`contailer`, 也就是`envoy sidecar`

实际上,其实这块先是改变了`yaml` 文件的内容,然后再创建	`pod`, `kubectl get pod pod-name -o yaml`



#### 4.2.2   自动注入`Sidecar`

上述每次都进行手动创建肯定是不爽的，那能不能自动注入呢? 

这块需要命名空间的支持,比如有一个命令空间为`istio-demo`,  可以让该命名空间下创建的`pod` 自动注入`sidecar`

1. 创建命名空间

   ```bash
   kubectl create namespace istio-demo
   
   ```

2. 给命名空间添加`label`

   ```bash
   kubectl label namespace istio-demo istio-injection=enabled
   
   ```

3. 创建资源

   ```bash
   kubectl apply -f first-istio.yaml -n istio-demo
   
   ```

4. 查看资源

   ```bash
   kubectl get pods -n istio-demo
   kubectl describe pod pod-name -n istio-demo
   kubectl get svc -n istio-demo
   
   ```

5. 删除资源

   ````bash
   kubectl delete -f first-istio.yaml -n istio-demo
   
   ````



#### 4.2.3  感受`prometheus`和`grafana`

其实`Istio` 已经默认帮我们安装好了`grafana`和`prometheus`,只是对应的`service` 是`ClusterIp`, 我们按照k8s的方式配置一下`ingress`的访问规则,但是要前提会有`Ingress Controller`的支持, 

1. 访问`prometheus`

    搜索`istio-demo.yaml` 文件

   ```yaml
   kind: Service
   name: http-prometheus # 主要是为了定位到prometheus的Service
   ```

2. 配置`prometheus`的`ingrss` 规则

   `prometheus-ingress.yaml`

   ```yaml
   #ingress
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: prometheus-ingress
     namespace: istio-system
   spec:
     rules:
     - host: prometheus.istio.luyanan.com
       http:
         paths:
         - path: /
           backend:
             serviceName: prometheus
             servicePort: 9090
   ```

3. 访问`grafana`

   `istio-demo.yaml` 文件

   ```yaml
   kind: Service
   targetPort: 3000
   ```

4. 配置`grafana` 的`ingress` 规则

   `grafana-ingress.yaml`

   ```yaml
   #ingress
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: grafana-ingress
     namespace: istio-system
   spec:
     rules:
     - host: grafana.istio.luyanan.com
       http:
         paths:
         - path: /
           backend:
             serviceName: grafana
             servicePort: 3000
   ```

5. 根据两个`ingress` 创建资源

   并且访问测试

   ```bash
   istio.luyanan.com/prometheus
   istio.luyanan.com/grafana
   ```

   ```bash
   kubectl apply -f prometheus-ingress.yaml
   kubectl apply -f grafana-ingress.yaml
   kubectl get ingress -n istio-system
   
   ```

   