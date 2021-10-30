# Sentinel的基本应用以及原理分析



##  限流的基本认识

### 场景分析

一个互联网产品, 打算搞一次大促来增加销量以及曝光. 公司的架构师基于往期的流量情况做了一个活动流量的预估. 然后整个公司的各个技术团队开始按照这个目标进行设计和优化, 最终在大家的不懈努力下,达到了链路压测的目标流量峰值. 到了活动开始那天, 大家都盯着监控面板, 看着流量像洪水一样涌进来, 由于前期的宣传工作做得非常好, 使得这个流程远远超过预期的峰值, 后端服务开始不稳定, CPU、内存各种爆表. 部分服务开始出现无响应的情况. 最后, 整个系统开始崩溃, 用户无法正常访问服务, 最后导致公司巨大的损失. 

### 引入限流

在10.1黄金周, 各大旅游景点都是人满为患, 所以有些景点为了避免出现踩踏事故, 会采取限流措施. 

那在架构场景中, 是不是也能这样做呢? 针对这个场景, 能不能设置一个最大的流量限制, 如果超过这个流量, 我们就拒绝提供服务, 从而使得我们的服务不会挂掉. 

当前, 限流虽然能够保证系统不被压垮, 但是对于被限流的用户来说, 就会很不开心. 所以限流其实是一种有损的解决方案, 但是相对于全部不可用, 有损服务是最好的一种解决方法. 

###  限流的作用

除了前面说的限流使用场景之外，限流的设置还能防止恶意请求流量、恶意攻击. 

所以, 限流的基本原理是通过对并发访问/请求进行限速或者一个时间窗口内的请求进行限速来保护系统, 一旦达到限制速率则可以拒绝服务(定向到错误页面或者告知资源没有了)、排队或等待(秒杀、下单)、降级(返回兜底数据或者默认数据,如商品详情库存默认有货). 

一般互联网企业常见的限流有: 限制总并发数(如数据库连接池、线程池)、限制瞬间并发数(nginx的limit_conn模块、用来限制瞬间并发连接数)、限制时间窗口内的平均速率(如Guava 的RateLimiter、nginx的limit_req模块、限制每秒的平均速率);其他的还有限制远程接口调用速率、限制MQ的消费速率、另外还可以根据网络连接数、网络流量、CPU或内存负载等来限流. 

有了限流, 就意味着在处理高并发的时候多了一种保护机制, 不用担心瞬间流量导致系统挂掉或者雪崩, 最后做到有损服务而不是不服务, 但是限流需要做好评估, 不能乱用, 否则一些正常流量出现一些奇怪的问题而导致用户体验很差造成用户损失. 

## 常见的限流算法

###   滑动窗口算法

发送和接收方都会维护一个数据帧的序列, 这个序列被称为窗口。发送方的窗口大小由接收方确定,目的在于控制发送速度, 以免接收方的缓存不够大, 而导致溢出. 同时控制流量也可以避免网络拥塞. 下面图中的4，5，6号数据帧已经被发送出去, 但是未收到关联的ACK， 7，8，9 帧则是等待发送. 可以看出发送端的窗口大小为6, 这时由接收端告知的. 此时如果发送端收到4号ACK， 则窗口的左边缘向右收缩, 窗口的右边缘则向右扩展, 此时窗口就向前滑动了, 则数据帧10 也可以被发送. 

![](http://files.luyanan.com//img/20191212160950.png)

[滑动窗口演示地址]:https://media.pearsoncmg.com/aw/ecs_kurose_compnetwork_7/cw/content/interactiveanimations/selective-repeat-protocol/index.html

###  漏桶(控制传输速率Leaky bucket)

漏桶算法思路是: 不断的往桶里面注水, 无论注水的速度是大还是小, 水都是按照固定的速率往外漏水, 如果桶满了, 水会溢出. 

桶本身具有一个恒定的速率往下漏水, 而上方时快时慢的会由水进入桶中,当桶还未满时, 上方的水可以加入. 一旦水满, 上方的水就无法加入. 桶满正是算法中的一个关键的触发条件(即流量异常判断成立的条件).而此条件下如何处理上方流下的水, 有两种方式: 

在桶满水之后, 常见的两种处理方式:

1. 暂时拦截住上方水的向下流动, 等待桶中的一部分水漏走后, 再放行上方水. 
2. 溢出的上方水直接抛弃.

特点： 



1. 漏水的速率是固定的. 
2. 即使存在突然注水量变大的情况, 漏水的速率也是固定的. 

![](http://files.luyanan.com//img/20191212161823.png)



###   令牌桶(能够解决突发流量)

令牌桶算法是网络流量整形(Traffic Shaping) 和速率限制(Rate Limiting) 中最常用的一种算法. 典型情况下, 令牌桶算法用来控制发送到网络上的数据数目, 并允许突发数据的发送. 

令牌桶是一个存放固定容量令牌(token)的桶, 按照固定速率往桶里添加令牌, 令牌桶算法实际上由三部分组成, 两个流和一个桶, 分别是令牌桶、数据流和令牌桶. 

####  令牌流和令牌桶

系统会以一定的速度生成令牌, 并将其防止到令牌桶中, 可以将令牌桶想象成一个缓存区(可以用队列这种数据结构来实现), 当缓冲区填满的时候, 新生成的令牌会被扔掉, 这里有两个变量很重要. 

第一个是生成令牌的速度, 一般成为 `rate`,  比如我们设定rate=2, 即每秒生成2个令牌，也就是每 1/2 秒生成一个令牌. 

第二个是令牌桶的大小, 一般称为burst, 比如我们设定burst = 10, 即令牌桶最大只能容纳10个令牌. 

![](http://files.luyanan.com//img/20191212163617.png)

有三种情况可能发生: 

1. *数据流的速率等于令牌流的速率*, 这种情况下每个到来的数据包或者请求都能对应一个令牌, 然后无延迟的通过队列. 
2. **数据流的速率小于令牌桶的速率**, 通过队列的数据包或者请求只消耗了一部分令牌, 剩下的令牌会在令牌桶中积累下来, 直到桶被装满, 剩下的令牌可以在突发请求的时候消耗掉. 
3. **数据流的速率大于令牌流的速率** , 这意味着桶里的令牌很快就被耗尽, 导致服务中断一段时间, 如果数据包或者请求持久到来, 将发生丢包或者拒绝响应. 



## 使用Sentinel 实现限流

### 介绍

sentinel 是Alibaba 开源的一个面向分布式服务架构的轻量级流量控制组件,主要以流量为切入点, 从限流、流量整型、熔断降级、系统负载保存等多个维度来保证微服务的稳定性. 



###  简单实现

我们先再pom.xml 中添加sentinel 的依赖

```xml
   <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-core</artifactId>
            <version>1.6.3</version>
        </dependency>
```

代码示例

```java
package com.sentinel;

import com.alibaba.csp.sentinel.Entry;
import com.alibaba.csp.sentinel.SphU;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import com.alibaba.csp.sentinel.slots.block.RuleConstant;
import com.alibaba.csp.sentinel.slots.block.flow.FlowRule;
import com.alibaba.csp.sentinel.slots.block.flow.FlowRuleManager;

import java.util.ArrayList;
import java.util.List;

/**
 * @author luyanan
 * @since 2019/12/13
 * <p></p>
 **/
public class SentinelDemo {

    private final static String resource = "hello";

    /**
     * <p>初始化限流规则</p>
     *
     * @return {@link }
     * @author luyanan
     * @since 2019/12/13
     */
    private static void initFlowRules() {


        List<FlowRule> rules = new ArrayList<>();
        FlowRule flowRule = new FlowRule();
        // 资源(方法名称/接口)
        flowRule.setResource(resource);
        // 限流的阈值类型  qps和线程数
        flowRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        // 设置数量
        flowRule.setCount(1000);
        rules.add(flowRule);
        FlowRuleManager.loadRules(rules);
    }

    public static void main(String[] args) {
        // 初始化规则
        initFlowRules();
        while (true) {
            Entry entry = null;
            try {
                entry = SphU.entry(resource);
                System.out.println("hello world");
            } catch (BlockException e) {
                // 如果被限流, 则会抛出异常
                e.printStackTrace();
            } finally {
                if (entry != null) {
                    // 释放
                    entry.exit();
                }
            }
        }
    }

}

```

通过上面的代码, 我们就基本完成了一个单机版的限流

###  接入控制台

####  获取sentinel 控制台

从release页面获取最新版本的控制台jar包

####  启动

使用如下命令启动控制台

```bash
java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard.jar

```

其中 `-Dserver.port=8080` 用于指定 Sentinel 控制台端口为 `8080`。

从 Sentinel 1.6.0 起，Sentinel 控制台引入基本的**登录**功能，默认用户名和密码都是 `sentinel`

####  客户端接入控制台

控制台启动后, 客户端需要按照以下步骤接入到控制台

####  1. 添加依赖

```xml
 <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-transport-simple-http</artifactId>
            <version>1.6.3</version>
        </dependency>
```

####  2. 配置启动参数

启动时加入 JVM 参数 `-Dcsp.sentinel.dashboard.server=consoleIp:port` 指定控制台地址和端口。若启动多个应用，则需要通过 `-Dcsp.sentinel.api.port=xxxx` 指定客户端监控 API 的端口（默认是 8719）

####  3. 然后就可以访问 `localhost:8080` 查看访问控制台了





###  使用注解实现限流

Sentinel 支持通过 `@SentinelResource` 注解定义资源并配置 `blockHandler` 和 `fallback` 函数来进行限流之后的处理。示例：

####  添加依赖

```xml

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-annotation-aspectj</artifactId>
            <version>1.6.3</version>
        </dependency>
```



#### 编写配置类和加载规则

```java
package com.sentinel;

import com.alibaba.csp.sentinel.annotation.aspectj.SentinelResourceAspect;
import com.alibaba.csp.sentinel.slots.block.RuleConstant;
import com.alibaba.csp.sentinel.slots.block.flow.FlowRule;
import com.alibaba.csp.sentinel.slots.block.flow.FlowRuleManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.ArrayList;
import java.util.List;

/**
 * @author luyanan
 * @since 2019/12/16
 * <p></p>
 **/
@Configuration
public class SentinelConfiguration {


    @Bean
    public SentinelResourceAspect sentinelResourceAspect() {
        initFlowRules();
        return new SentinelResourceAspect();
    }

    /**
     * <p>初始化规则</p>
     *
     * @author luyanan
     * @since 2019/12/16
     */
    private void initFlowRules() {

        List<FlowRule> ruleList = new ArrayList<>();

        FlowRule flowRule = new FlowRule();
        // 资源的地址
        flowRule.setResource("hello");
        // 限流的类型
        flowRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        flowRule.setCount(10);

        ruleList.add(flowRule);
        FlowRuleManager.loadRules(ruleList);

    }


}

```



####  编写测试的方法

```java
/**
 * @author luyanan
 * @since 2019/12/16
 * <p></p>
 **/
@RestController
@RequestMapping("user")
public class SentinelController {


    @SentinelResource(value = "hello")
    @GetMapping("hello")
    public String hello() {
        System.out.println("hello world");
        return "hello";
    }
}

```









## 规则的种类

Sentinel 的所有规则都可以在内存态中动态地查询及修改，修改之后立即生效。同时 Sentinel 也提供相关 API，供您来定制自己的规则策略。

Sentinel 支持以下几种规则：**流量控制规则**、**熔断降级规则**、**系统保护规则**、**来源访问控制规则** 和 **热点参数规则**。

### 流量控制规则 (FlowRule)

#### 流量规则的定义

重要属性：

| Field           | 说明                                                         | 默认值                        |
| --------------- | ------------------------------------------------------------ | ----------------------------- |
| resource        | 资源名，资源名是限流规则的作用对象                           |                               |
| count           | 限流阈值                                                     |                               |
| grade           | 限流阈值类型，QPS 模式（1）或并发线程数模式（0）             | QPS 模式                      |
| limitApp        | 流控针对的调用来源                                           | `default`，代表不区分调用来源 |
| strategy        | 调用关系限流策略：直接、链路、关联                           | 根据资源本身（直接）          |
| controlBehavior | 流控效果（直接拒绝 / 排队等待 / 慢启动模式），不支持按调用关系限流 | 直接拒绝                      |

同一个资源可以同时有多个限流规则，检查规则时会依次检查。

#### 通过代码定义流量控制规则

理解上面规则的定义之后，我们可以通过调用 `FlowRuleManager.loadRules()` 方法来用硬编码的方式定义流量控制规则，比如：

```java
private void initFlowQpsRule() {
    List<FlowRule> rules = new ArrayList<>();
    FlowRule rule = new FlowRule(resourceName);
    // set limit qps to 20
    rule.setCount(20);
    rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    rule.setLimitApp("default");
    rules.add(rule);
    FlowRuleManager.loadRules(rules);
}
```



##   原理分析

###  入口

```java
 entry= SphU.entry(resource); 			
```
进入到

```java
    public static Entry entry(String name) throws BlockException {
        return Env.sph.entry(name, EntryType.OUT, 1, OBJECTS0);
    }
```



###  Env.sph

```java
public class Env {

    public static final Sph sph = new CtSph();

    static {
        // If init fails, the process will exit.
        InitExecutor.doInit();
    }

}

```

我们发现有一个静态块

###  InitExecutor.doInit();

```java
 public static void doInit() {
        if (!initialized.compareAndSet(false, true)) {
            return;
        }
        try {
            ServiceLoader<InitFunc> loader = ServiceLoader.load(InitFunc.class);
            List<OrderWrapper> initList = new ArrayList<OrderWrapper>();
            for (InitFunc initFunc : loader) {
                RecordLog.info("[InitExecutor] Found init func: " + initFunc.getClass().getCanonicalName());
                insertSorted(initList, initFunc);
            }
            for (OrderWrapper w : initList) {
                w.func.init();
                RecordLog.info(String.format("[InitExecutor] Executing %s with order %d",
                    w.func.getClass().getCanonicalName(), w.order));
            }
        } catch (Exception ex) {
            RecordLog.warn("[InitExecutor] WARN: Initialization failed", ex);
            ex.printStackTrace();
        } catch (Error error) {
            RecordLog.warn("[InitExecutor] ERROR: Initialization failed with fatal error", error);
            error.printStackTrace();
        }
    }
```

我们发现, 这里通过SPI 机制加载`InitFunc` 的实现类, 然后分别调用他们的 `init` 方法. 

###  Env.sph.entry

```java
  @Override
    public Entry entry(String name, EntryType type, int count, Object... args) throws BlockException {
        StringResourceWrapper resource = new StringResourceWrapper(name, type);
        return entry(resource, count, args);
    }
```

这里通过name 和type  包装一个`StringResourceWrapper`, 即抽象的资源. 

###  CtSph.entry

```java
   public Entry entry(ResourceWrapper resourceWrapper, int count, Object... args) throws BlockException {
        return entryWithPriority(resourceWrapper, count, false, args);
    }
```

###   entryWithPriority

```java
private Entry entryWithPriority(ResourceWrapper resourceWrapper, int count, boolean prioritized, Object... args)
        throws BlockException {
        Context context = ContextUtil.getContext();
        if (context instanceof NullContext) {
            // The {@link NullContext} indicates that the amount of context has exceeded the threshold,
            // so here init the entry only. No rule checking will be done.
            return new CtEntry(resourceWrapper, null, context);
        }

        if (context == null) {
            // Using default context.
            context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
        }

        // Global switch is close, no rule checking will do.
        if (!Constants.ON) {
            return new CtEntry(resourceWrapper, null, context);
        }

        ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper);

        /*
         * Means amount of resources (slot chain) exceeds {@link Constants.MAX_SLOT_CHAIN_SIZE},
         * so no rule checking will be done.
         */
        if (chain == null) {
            return new CtEntry(resourceWrapper, null, context);
        }

        Entry e = new CtEntry(resourceWrapper, chain, context);
        try {
            chain.entry(context, resourceWrapper, null, count, prioritized, args);
        } catch (BlockException e1) {
            e.exit(count, args);
            throw e1;
        } catch (Throwable e1) {
            // This should not happen, unless there are errors existing in Sentinel internal.
            RecordLog.info("Sentinel unexpected exception", e1);
        }
        return e;
    }
```



###  InternalContextUtil.internalEnter

```java
   private final static class InternalContextUtil extends ContextUtil {
        static Context internalEnter(String name) {
            return trueEnter(name, "");
        }

        static Context internalEnter(String name, String origin) {
            return trueEnter(name, origin);
        }
    }
```

###  trueEnter

```java
protected static Context trueEnter(String name, String origin) {
        Context context = contextHolder.get();
        if (context == null) {
            Map<String, DefaultNode> localCacheNameMap = contextNameNodeMap;
            DefaultNode node = localCacheNameMap.get(name);
            if (node == null) {
                if (localCacheNameMap.size() > Constants.MAX_CONTEXT_NAME_SIZE) {
                    setNullContext();
                    return NULL_CONTEXT;
                } else {
                    try {
                        LOCK.lock();
                        node = contextNameNodeMap.get(name);
                        if (node == null) {
                            if (contextNameNodeMap.size() > Constants.MAX_CONTEXT_NAME_SIZE) {
                                setNullContext();
                                return NULL_CONTEXT;
                            } else {
                                node = new EntranceNode(new StringResourceWrapper(name, EntryType.IN), null);
                                // Add entrance node.
                                Constants.ROOT.addChild(node);

                                Map<String, DefaultNode> newMap = new HashMap<>(contextNameNodeMap.size() + 1);
                                newMap.putAll(contextNameNodeMap);
                                newMap.put(name, node);
                                contextNameNodeMap = newMap;
                            }
                        }
                    } finally {
                        LOCK.unlock();
                    }
                }
            }
            context = new Context(node, name);
            context.setOrigin(origin);
            contextHolder.set(context);
        }

        return context;
    }
```



###  lookProcessChain

```java
ProcessorSlot<Object> lookProcessChain(ResourceWrapper resourceWrapper) {
        ProcessorSlotChain chain = chainMap.get(resourceWrapper);
        if (chain == null) {
            synchronized (LOCK) {
                chain = chainMap.get(resourceWrapper);
                if (chain == null) {
                    // Entry size limit.
                    if (chainMap.size() >= Constants.MAX_SLOT_CHAIN_SIZE) {
                        return null;
                    }

                    chain = SlotChainProvider.newSlotChain();
                    Map<ResourceWrapper, ProcessorSlotChain> newMap = new HashMap<ResourceWrapper, ProcessorSlotChain>(
                        chainMap.size() + 1);
                    newMap.putAll(chainMap);
                    newMap.put(resourceWrapper, chain);
                    chainMap = newMap;
                }
            }
        }
        return chain;
    }

```



###  chain = SlotChainProvider.newSlotChain(); 构建一个Chanin

```java
public static ProcessorSlotChain newSlotChain() {
        if (builder != null) {
            return builder.build();
        }

        resolveSlotChainBuilder();

        if (builder == null) {
            RecordLog.warn("[SlotChainProvider] Wrong state when resolving slot chain builder, using default");
            builder = new DefaultSlotChainBuilder();
        }
        return builder.build();
    }
```



### new DefaultSlotChainBuilder()

```java
public class DefaultSlotChainBuilder implements SlotChainBuilder {
    public DefaultSlotChainBuilder() {
    }

    public ProcessorSlotChain build() {
        ProcessorSlotChain chain = new DefaultProcessorSlotChain();
        chain.addLast(new NodeSelectorSlot());
        chain.addLast(new ClusterBuilderSlot());
        chain.addLast(new LogSlot());
        chain.addLast(new StatisticSlot());
        chain.addLast(new SystemSlot());
        chain.addLast(new AuthoritySlot());
        chain.addLast(new FlowSlot());
        chain.addLast(new DegradeSlot());
        return chain;
    }
}
```

即最后的调用顺序如下： NodeSelectorSlot => ClusterBuilderSlot => LogSlot => StatisticSlot => AuthoritySlot => SystemSlot => FlowSlot => DegradeSlot
 如果想改变他们的调用顺序，可通过SPI机制实现



### NodeSelectorSlot

构造调用链，具体参考 [NodeSelectorSlot](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fall4you%2Fsentinel-tutorial%2Fblob%2Fmaster%2Fsentinel-principle%2Fsentinel-slot-chain%2Fsentinel-slot-chain.md)



```dart
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, Object obj, int count, boolean prioritized, Object... args)
    throws Throwable {
    // 每个资源对应一个 ProcessorSlotChain
    // 一个资源可以对应多个Context
    // 一个ContextName 对应一个 DefaultNode , 即 一个资源可能对应多个 DefaultNode, 但 一个资源只有一个 ClusterNode
    // 针对同一段代码，不同线程对应的Context实例是不一样的，但是对应的Context Name是一样的，所以这时认为是同一个Context，Context我们用Name区分
    DefaultNode node = map.get(context.getName());
    if (node == null) {
        synchronized (this) {
            node = map.get(context.getName());
            if (node == null) {
                node = new DefaultNode(resourceWrapper, null);
                // key 为 ontextName , vaue 为 DefaultNode
                HashMap<String, DefaultNode> cacheMap = new HashMap<String, DefaultNode>(map.size());
                cacheMap.putAll(map);
                cacheMap.put(context.getName(), node);
                map = cacheMap;
                // Build invocation tree
                ((DefaultNode) context.getLastNode()).addChild(node);
            }

        }
    }
    context.setCurNode(node);
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```

### ClusterBuilderSlot

具体参考 [ClusterBuilderSlot](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fall4you%2Fsentinel-tutorial%2Fblob%2Fmaster%2Fsentinel-principle%2Fsentinel-slot-chain%2Fsentinel-slot-chain.md)

每个资源对应一个ClusterNode，并且DefaultNode引用了ClusterNode

### LogSlot

记录日志用的，先执行下面的 Solt， 如果报错了或者被Block了，记录到日志中



```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode obj, int count, boolean prioritized, Object... args)
    throws Throwable {
    try {
        fireEntry(context, resourceWrapper, obj, count, prioritized, args);
    } catch (BlockException e) {
        EagleEyeLogUtil.log(resourceWrapper.getName(), e.getClass().getSimpleName(), e.getRuleLimitApp(),
            context.getOrigin(), count);
        throw e;
    } catch (Throwable e) {
        RecordLog.warn("Unexpected entry exception", e);
    }
}
```

### StatisticSlot

核心实现，各种计数的实现逻辑，基于时间窗口实现。 基于触发请求通过 和 请求Block 的回调逻辑，回调逻辑在 MetricCallbackInit 中初始化了， 最终还是靠 StatisticSlotCallbackRegistry



```java
// 省略了一些代码
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count, boolean prioritized, Object... args) throws Throwable {
    try {
        // 执行下来的Solt ，判断是否通过
        fireEntry(context, resourceWrapper, node, count, prioritized, args);

        // Request passed, add thread count and pass count.
        node.increaseThreadNum();
        node.addPassRequest(count);
    } catch (BlockException e) {
        // Blocked, set block exception to current entry.
        context.getCurEntry().setError(e);

        // Add block count.
        node.increaseBlockQps(count);
        if (context.getCurEntry().getOriginNode() != null) {
            context.getCurEntry().getOriginNode().increaseBlockQps(count);
        }
    }
}
```

DefaultNode 继承自 StatisticNode , 在 StatisticNode 中有两个属性



```java
// 第一个参数表示 窗口的个数；第二个参数表示 窗口对多长时间进行统计  比如 QPS xx/秒   那就是 1000 毫秒， 所以窗口的长度为  1000/个数
private transient volatile Metric rollingCounterInSecond = new ArrayMetric(SampleCountProperty.SAMPLE_COUNT, IntervalProperty.INTERVAL);

// 窗口长度为1000 60个 刚好一分钟
private transient Metric rollingCounterInMinute = new ArrayMetric(60, 60 * 1000, false);
```

ArrayMetric 持有 LeapArray ， LeapArray 主要有两个实现类 OccupiableBucketLeapArray 、 BucketLeapArray ， 但根据当前时间获取窗口的核心实现在 LeapArray 抽象类中

滑动窗口简单理解就是： 根据任何时间，都可以获取一个对应的窗口，在该窗口内，保存着在窗口长度时间内通过的请求数、被block的请求数、异常数、RT。基于这些数据，我们就可以得到对应的资源的QPS、RT等指标信息。

核心方法在 LeapArray#currentWindow ， 整体思路如下

1. 根据当前时间获取时间窗口的下标  (time/windowLength) % array.length()
2. 计算当前时间对应时间窗口的开始时间  time - time % windowLength
3. 根据下标获取时间窗口，这里分三种情况：
    (1) 根据下标没有获取到窗口，此时创建一个窗口。此时代表窗口没有创建 或者 窗口还没有开始滑动， 所以对应的下标位置为null
    (2) 根据下标获取到窗口，并且该窗口的开始时间和上面计算的开始时间一样，此时直接返回该窗口
    (3) 根据下标获取到窗口，但是该窗口的开始时间大于上面计算的开始时间，这时需要用计算的开始时间重置该窗口的开始时间，这就类似于窗口在滑动



```csharp
public WindowWrap<T> currentWindow(long timeMillis) {
    if (timeMillis < 0) {
        return null;
    }

    // 计算窗口数组下标
    int idx = calculateTimeIdx(timeMillis);
    // 计算开始时间
    long windowStart = calculateWindowStart(timeMillis);
    while (true) {
        WindowWrap<T> old = array.get(idx);
        if (old == null) {
            WindowWrap<T> window = new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
            if (array.compareAndSet(idx, null, window)) {
                // Successfully updated, return the created bucket.
                return window;
            } else {
                //  循环重试 Contention failed, the thread will yield its time slice to wait for bucket available.
                Thread.yield();
            }
        } else if (windowStart == old.windowStart()) {
            return old;
        } else if (windowStart > old.windowStart()) {
            if (updateLock.tryLock()) {
                try {
                    // Successfully get the update lock, now we reset the bucket.
                    return resetWindowTo(old, windowStart);
                } finally {
                    updateLock.unlock();
                }
            } else {
                // Contention failed, the thread will yield its time slice to wait for bucket available.
                Thread.yield();
            }
        } else if (windowStart < old.windowStart()) {
            // Should not go through here, as the provided time is already behind.
            return new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
        }
    }
}
```

todo OccupiableBucketLeapArray 还不太理解

### AuthoritySlot

黑白名单规则校验，非常简单



```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count, boolean prioritized, Object... args) throws Throwable {
    checkBlackWhiteAuthority(resourceWrapper, context);
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```

1. 加载所有的黑白名单规则
2. 遍历所有黑白名单规则，调用 AuthorityRuleChecker#passCheck 方法，如果不通过则抛出 AuthorityException
3. 校验逻辑：从 Context 中拿到 originName, 然后判断 originName 是否在 规则的 limitApp 中, 然后判断是 黑名单 还是白名单，然后校验返回结果

### SystemSlot

仅对入口流量有效，校验顺序 QPS -> 线程数 -> RT -> BBR -> CPU



```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count, boolean prioritized, Object... args) throws Throwable {
    SystemRuleManager.checkSystem(resourceWrapper);
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```

### FlowSlot

限流处理

三种拒绝策略：直接拒绝、WarnUP、匀速排队

三种限流模式：直接、关联、链路



```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                    boolean prioritized, Object... args) throws Throwable {
    checkFlow(resourceWrapper, context, node, count, prioritized);

    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```

FlowRuleChecker#checkFlow

1. 获取所有限流规则
2. 遍历规则，执行 FlowRuleChecker#canPassCheck => FlowRuleChecker#passLocalCheck => rule.getRater().canPass(selectedNode, acquireCount, prioritized)
3. rule.getRater() 返回一个 TrafficShapingController 对象， 它有3种实现(代码中有4中，但官方文档只介绍了3种)，即对应上面的三种流控模式，每个规则对用的 TrafficShapingController 是在加载规则的时候就确定了

// FlowRuleUtil#generateRater



```cpp
private static TrafficShapingController generateRater(/*@Valid*/ FlowRule rule) {
    if (rule.getGrade() == RuleConstant.FLOW_GRADE_QPS) {
        switch (rule.getControlBehavior()) {
            case RuleConstant.CONTROL_BEHAVIOR_WARM_UP:
                return new WarmUpController(rule.getCount(), rule.getWarmUpPeriodSec(),
                    ColdFactorProperty.coldFactor);
            case RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER:
                return new RateLimiterController(rule.getMaxQueueingTimeMs(), rule.getCount());
            case RuleConstant.CONTROL_BEHAVIOR_WARM_UP_RATE_LIMITER:
                return new WarmUpRateLimiterController(rule.getCount(), rule.getWarmUpPeriodSec(),
                    rule.getMaxQueueingTimeMs(), ColdFactorProperty.coldFactor);
            case RuleConstant.CONTROL_BEHAVIOR_DEFAULT:
            default:
                // Default mode or unknown mode: default traffic shaping controller (fast-reject).
        }
    }
    return new DefaultController(rule.getCount(), rule.getGrade());
}
```

##### DefaultController

比较简单，判断逻辑 (当前的Count + 本次调用) 是否大于 规则中设置的 阈值

##### WarmUpController

让QPS在指定的时间内增加到 阈值， 目前每太看懂

##### RateLimiterController

也比较简单，先按规则中配置的QPS计算每个请求的平均响应时间，然后判断当前请求是否能够等那么久(规则中的时间窗口)

##### 三种限流模式在哪里体现？

其实这个主要就是判断 你的指标数据应该要从哪个 Node 中获取，这部分逻辑在 FlowRuleChecker#selectNodeByRequesterAndStrategy 方法中

1. 直接： 根据你的 originName 和 limitApp 来判断是取 ClusterNode 还是 OriginNode
2. 关联： 根据关联的资源名取对应的 ClusterNode
3. 链路： 判断关联的资源 和 当前的 contextName 是否一致，是则返回 当前的 DefaultNode

### DegradeSlot

降级处理

目前有三种降级模式：基于RT、基于异常比例、基于一分钟异常数



```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count, boolean prioritized, Object... args)
    throws Throwable {
    DegradeRuleManager.checkDegrade(resourceWrapper, context, node, count);
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```

##### 基于RT

从时间窗口获取RT和规则中配置的阈值进行比较， 通过则 重置计数，然后直接返回； 不通过则 计数加1，如果 计数 >= 5，则进行降级处理

##### 基于异常比例

前提条件 QPS > =5 , 然后用 1s异常数/1s总请求数 , 和规则中配置的阈值进行比较

##### 基于1分钟异常数

直接用1分钟内的异常数和规则的阈值做比较

##### 如何按时间窗口降级

定时任务 + flag
 如果降级了， 设置 flag = true , 在 时间窗口秒后， 重置 flag = false ，然后再 passCheck 方法的入口处， 如果 flag = true 就直接降级



