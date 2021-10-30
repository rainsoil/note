# Dubbo 服务治理以及新功能讲解



##  负载均衡

###  负载均衡的背景

到目前为止,Dubbo 集成zookeeper 解决了服务注册以及服务动态感知的问题, 那么当服务端存在多个节点的集群的时候, zookeeper 上会维护不同集群节点, 对于客户端而言,它需要一种负载均衡来实现目标服务的请求负载. 通过负载均衡, 可以让每个服务器节点获得适合自己处理能力的负载. 

负载均衡可以分为软负载和硬件负载, 在实际开发中, 我们基础软件负载比较多, 比如nginx, 硬件负载现在用的比较少,而且需要有专门的人维护. 

Dubbo 里面默认就集成了负载均衡的算法和实现, 默认提供了4中负载均衡实现. 

### Dubbo 中负载均衡的应用

配置的属性名称: roundrobin/random/leastactive/ consistenthash 

>   <dubbo:service interface="" loadbalance="roundrobin"/>
>
>​     <dubbo:reference interface="" loadbalance="roundrobin"/>

可以在服务端配置, 也可以在客户端配置

如果是基于注解,配置如下: 

```java
@Service(loadbalance = "roundrobin")
public class UserApiImpl implements UserApi {


    @Override
    public String info(String id) {
        System.out.println("info 请求");
        return "张三:" + id;
    }
}
```

或者

```java
    // dubbo 提供了注入的方法
    @Reference(loadbalance = "roundrobin")
    private UserApi userApi;


    @GetMapping("info/{id}")
    public String info(@PathVariable("id") String id) {
        return userApi.info(id);
    }
```

####  演示方式

在  run configurations  中, 配置多个 springboot application , 添加jvm参数是 两个程序启动的端口不一样,然后客户端发起多次调用实现请求的负载均衡. 

>  -Ddubbo.protocol.port=20881 

### Dubbo负载均衡算法

####   RandomLoadBalance 

权重算法,根据权重值进行随机负载. 

它的算法思想很简单, 假设我们由一组服务器 servers = [A,B,C], 他们对应的权重为 weights = [5, 3, 2] ,权重总和为10. 现在把这些权重值平铺在一组坐标值上, [0,5] 区间属于服务器A, [5,8] 区间属于服务器B, [8,10]区间属于服务器C. 接下来通过随机数生成器生成一个范围在[0,10] 之间的随机数, 然后计算这个随机数会落在哪个区间上. 比如数字3 会落在服务器A 对应的区间上, 此时返回服务器A 即可. 权重越大的机器, 在坐标轴上对应的区间范围越大, 因为随机数生成器生成的数字就有更大的概率落在此区间内, 只要随机数生成器产生的随机数分布性很好, 在经过多次选择后, 每个服务器被选中的次数比例接近其权重比例. 

####   LeastActiveLoadBalance 

最少活跃调用数算法, 活跃调用数越小, 表明该服务提供者效率越高,单位时间内处理更多的请求这个是比较科学的负载均衡算法. 

每个服务器提供者对应一个活跃数  active ,初始情况下, 所有服务提供者活跃数均为0, 每收到一个请求, 活跃数+1, 完成请求后则将活跃数减1, 在服务运行一段事件后, 性能好的服务提供者处理请求的速度更快, 因为活跃数下降的也越快, 此时这样的服务提供者能够优先获取到新的服务请求. 

####   ConsistentHashLoadBalance 

hash 一致性算法, 相同参数的请求总是发到同一个提供者. 

当某一台提供者挂时, 原本发往该提供者的请求, 基于虚拟节点,平摊到其他提供者, 不会引起剧烈变动. 

####   RoundRobinLoadBalance

加权轮询算法. 所谓轮询是指将请求轮流分配给每台服务器, 举个例子, 我们有三台服务器:A,B,C. 我们将第一个请求分配给服务器A, 第二个请求分配给服务器B, 第三个请求分配给服务器C, 这个过程就叫轮询. 轮询是一种无状态负载均衡算法, 实现简单, 适用于每台服务器性能相近的场景下, 但现实情况下,我们并不能保证每台服务器性能均相近. 这个时候如果我们需要将等量的请求分配给性能较差的服务器, 这显然是不合理的. 因此, 这个时候我们需要对轮询过程进行加权, 以调控每台服务器的负载. 经过加权后, 每台服务器能够得到的请求数比例, 接近或者等于他们的权重比. 比如服务器A,B,C 的权重比为5:2:1, 那么在8次请求中,服务器A 将收到其中的5次请求， 服务器B 会收到其中的2次请求, 服务器C 则收到其中的1次请求. 

##  集群容错

在分布式网络通信中, 容错能力是必须的, 什么叫容错呢? 从字面意思来看, 容: 是容忍, 错: 是错误, 就是容忍错误的意思. 

我们知道网络通信中会有很多不确定的因素, 比如网络延迟、网络中断、服务异常等,会造成当前这次请求出现失败。当服务通信出现这个问题时, 需要采取一定的措施应对, 而dubbo 中提供了容错机制来优雅的处理这种错误. 

在集群调用失败时, dubbo 提供了多种容错方案, 缺省值为 failover  重试. 

> ```
> 
> @Service(loadbalance = "roundrobin",cluster = "failsafe")
> ```

###  Failover Cluster  

失败自动切换, 当出现失败的时候, 重试其他服务器(缺省)

通常用于读操作, 但重试会带来更长的延迟, 可通过` retries="2" `  来设置重试次数(不含第一次)M 

###  Failfast Cluster

快速调用, 只发起一次调用, 失败理解报错

通常用于非幂等性的写操作,比如新增记录等. 

###  Failsafe Cluster 

失败安全, 出现异常时, 直接忽略, 通常用于写入审计日志等操作. 

###   Failback Cluster 

失败自动恢复, 后台记录失败请求，定时重发. 通常用于消息通知操作. 

###   Forking Cluster 

并行调用多个服务器, 只要一个成功即返回。 通常用于实时性要求较高的读操作,但需要浪费更多的服务资源, 可通过 ` forks="2" ` 来设置最大并行数. 

###  Broadcast Cluster  

广播调用所有提供者, 逐个调用, 任意一台报错则报错,(从2.1.0 开始支持), 通常用于通知所有提供者更新资源或日志等本地资源信息. 

在实际应用中,查询语句容错策略建议使用默认 Failover Cluster  , 而增删改建议使用  Failfast Cluster 或者 使用 Failover Cluster（retries=”0”）  策略防止出现 数据重复添加等其他问题, 建议在设计接口的时候把查询接口单独做一个接口提供查询. 



## 服务降级

### 降级的概念

当某个非关键服务出现错误时, 可以通过降级功能来临时屏蔽这个服务. 降级可以有几个层面的分类: 自动降级和人工降级. 按照功能可以分为: 读服务降级和写服务降级. 

1.  对一些非核心服务进行人工降级, 在大促之前通过降级开关关闭哪些推荐内容、评价等对主流程没有影响的功能. 
2. 故障降级, 比如调用的远程服务挂了, 网络故障、或者PRC 服务返回异常, 那么可以直接降级, 降级的方案比如设置默认值、采用兜底数据(系统推荐的行为广告挂了, 可以提前准备静态页面做返回)等等
3. 限流降级, 在秒杀这种流量比较集群并且流量特别大的情况下, 因为突发访问量特别大可能会导致系统支持不了. 这个时候可能采用限流来限制访问量. 当达到阈值时, 后续的请求被降级, 比如进入排队页面, 比如跳转到错误页等. 

那么在Dubbo 中如何实现服务降级呢? Dubbo 中提供了一个mock 的配置, 可以通过mock 来实现当服务提供方出现网络异常或者挂掉以后, 客户端不抛出异常, 而是用过mock 数据返回自定义的数据。 

### Dubbo 实现服务降级

在客户端新建一个mock类, 当出现服务降级的时候, 会被调用. 

```java
package com.dubbo.client;

import com.dubbo.spring.UserApi;

/**
 * @author luyanan
 * @since 2019/11/21
 * <p>降级策略</p>
 **/
public class MockUserApi implements UserApi {


    @Override
    public String info(String id) {
        return "服务出现异常";
    }
}

```



修改客户端的注解, 增加mock配置, 以及修改timeout=1, 表示本次调用的超时时间是1毫秒, 这样可以模拟出失败的场景. 

需要配置 ` cluster=failfast ` ,否则因为默认是 failover  导致客户端会发起3次重试, 等待的时间比较长. 

```java
    // dubbo 提供了注入的方法
    @Reference(loadbalance = "random", mock = "com.dubbo.client.MockUserApi", timeout = 100, cluster = "failfast")
    private UserApi userApi;
```



##  启动时检查

Dubbo 缺省会在启动时检查依赖的服务是否可用, 不可用时会抛出异常, 阻止Spring初始化完成, 以便上线时, 能够及早的发现问题, 默认 ` check="true"`

可以通过 ` check="true"。 ` 关闭检查, 比如, 测试时, 有些服务不关心, 或者出现了循环依赖, 必须有一方先启动, 

>  registry、reference、consumer 都可以配置 check 这个属性 

```java
    @Reference(mock = "com.dubbo.client.HelloApiMock", timeout = 100,cluster = "failfast",check = false)
    private HelloApi helloApi;

```



##  多版本支持

当一个接口实现,出现不兼容升级, 可以用版本号过渡, 版本号不同的服务相互间不引用. 

可以按照以下步骤进行版本迁移: 

1. 在低压力时间段, 先升级一半提供者为新版本. 
2. 再将所有消费者升级为新版本
3. 然后将剩下的一半提供者升级为新版本. 

##  主机绑定

###  默认的主机绑定方式

1. 通过 LocalHost.getLocalHost()  获取本机地址
2. 如果是127.* 等 loopback (环路地址)地址, 则扫描各网卡, 获取网卡ip
   1. 如果是springboot, 修改配置 ` dubbo.protocol.host=”” `
   2. 如果注册地址获取不正确, 可以通过在 dubbo.xml 中加入主机地址的配置`  <dubbo:protocol host="205.182.231.231">`





##  Duboo 新的功能

### 动态配置规划

动态配置是Dubbo 2.7版本中引入的一个新的功能, 简单来说, 就是把dubbo.properties 中的属性进行集中式存储, 存储在其他的服务器上. 

那么如果需要用到集中式存储, 还需要一些配置中心组件来支撑. 

目前Dubbo 能支持的配置中心有:  apollo、nacos、zookeeper

其次, 从另外一个角度来看, 我们之前用zookeeper 实现服务注册和发现, 本质上就是使用zookeeper 实现了配置中心, 这个配置中心只是维护了服务注册和服务感知的功能. 在2.7 的版本中, Dubbo 对配置中心做了延展, 除了服务注册以外, 还可以把其他的数据存储在zookeeper上, 从而更好的进行维护. 

####  在dubbo-admin 添加配置

应用名称可以是 global ,或者对应当前服务的应用名, 如果是 global  表示全局配置, 针对所有应用可见. 

配置的内容, 实际上就是 dubbo.properties 中配置的信息, 只是统一存储在了zookeeper 中而已. 



####  本地配置文件中添加配置中心

在 application.properties 中添加配置中心的配置项, app-name 对应的是上一步创建的项目名, 

```java
dubbo.config-center.address=zookeeper://192.168.86.128:2181
dubbo.config-center.app-name=dubbo-spring-server
## 存于配置中心的配置项, 本地仍然需要配置一份, 这样目的是为了保证可靠性. 
```

#### 配置的优先级

引入配置中心后, 配置的优先级就需要关注了, 默认情况下, 外部配置的优先级最高, 也就意味着配置中心上的配置会覆盖本地的配置, 当然我们也可以调整优先级. 

>  dubbo.config-center.highest-priority=false  

####  配置中心的原理

默认所有的配置都存储在 /dubbo/config   节点, 具体节点结构图如下: 

- namespace:用于不同配置的环境隔离. 
- config: Dubbo 约定的固定节点, 不可更改, 所有配置和服务治理规则都存储在此节点下. 
-  dubbo/application: 分别用来隔离全局配置, 应用级别配置, dubbo 是默认group值, application 对应应用名. 
-  dubbo.properties : 此节点的node value 存储具体配置内容. 

![](http://files.luyanan.com//img/20191121152037.png)



### 元数据中心

Duboo 2.7 的另外一个功能, 就是增加了元数据的配置. 

在Dubbo 2.7 之前, 所有的配置信息, 比如服务接口名称、重试次数、版本号、负载策略、容错策略等, 所有的参数都是基于url 形式配置在zookeeper 上的, 这种方式会造成一些问题. 

1. url 内容过多,导致数据存储空间增大. 
2. url 需要涉及到网络传输, 数据量多大会造成网络传输过慢. 
3. 网络传输慢, 会造成服务地址感知的延迟变大, 影响服务的正常响应. 

服务提供者这边的配置参数有30多个, 有一半是不需要作为注册中心进行存储的, 而消费者这边可配置的参数有25个以上, 只有个别是需要传递到注册中心的, 所以, 在Dubbo2.7 中对元数据进行了改造, 简单来说, 就是把属于服务治理的数据发布到注册中心, 其他的配置数据统一发布到元数据中心, 这样一来大大降低了注册中心的负载. 

####  元数据配置. 

元数据中心目前支持 redis 和zookeeper .官方推荐是采用redis, 毕竟redis本身对于非结构化存储的数据读写性能比较高, 当然, 也可以使用zookeeper 来实现. 

在配置中心中添加元数据中心的地址 

```properties
## 元数据配置
dubbo.metadata-report.address=zookeeper://192.168.86.128:2181
## 注册到注册中心的url 是否采用精简模式(与低版本兼容)
dubbo.registry.simplified=true

```





