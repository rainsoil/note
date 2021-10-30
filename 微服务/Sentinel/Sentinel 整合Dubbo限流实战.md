#  Sentinel 整合Dubbo限流实战



##  创建生产者项目

这里需要提供一个对外的api项目和一个Dubbo服务

![](http://files.luyanan.com//img/20191216215503.png)



###  添加依赖

```xml
  <dependency>
            <groupId>com.sentinel</groupId>
            <artifactId>sentinel-dubbo-api</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>

        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-client</artifactId>
            <version>4.0.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>4.0.1</version>
        </dependency>

        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.7.2</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
       
```



###   api服务

在api项目中创建接口类

```java
package com.sentinel;

/**
 * @author luyanan
 * @since 2019/12/16
 * <p></p>
 **/
public interface SentinelService {

    String sayello(String name);

}

```



###  生产者服务

接口实现类

```java
package com.sentinel;

import org.apache.dubbo.config.annotation.Service;

/**
 * @author luyanan
 * @since 2019/12/16
 * <p></p>
 **/
@Service
public class SentinelServiceImpl  implements  SentinelService {


    @Override
    public String sayello(String name) {
        System.out.println("sayHello :" + name);
        return "hello " + name;
    }
}

```



dubbo 配置类

```java
package com.sentinel;

import org.apache.dubbo.config.ApplicationConfig;
import org.apache.dubbo.config.ProtocolConfig;
import org.apache.dubbo.config.RegistryConfig;
import org.apache.dubbo.config.spring.context.annotation.DubboComponentScan;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author luyanan
 * @since 2019/12/16
 * <p></p>
 **/
@Configuration
@DubboComponentScan("com.sentinel")
public class ProviderConfig {


    @Bean
    public ApplicationConfig applicationConfig() {

        ApplicationConfig config = new ApplicationConfig();
        config.setName("sentinel-provider");
        config.setOwner("luyanan");
        return config;
    }


    @Bean
    public RegistryConfig registryConfig() {
        RegistryConfig config = new RegistryConfig();
        config.setAddress("zookeeper:192.168.9.106:2181");
        config.setCheck(false);
        return config;
    }

    @Bean
    public ProtocolConfig protocolConfig() {
        ProtocolConfig config = new ProtocolConfig();
        config.setPort(20880);
        config.setName("dubbo");
        return config;
    }
}

```



###  启动类

```java
package com.sentinel;


import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

import java.io.IOException;

public class SentinelDubboProviderApplication {

    public static void main(String[] args) throws IOException {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(ProviderConfig.class);
        ((AnnotationConfigApplicationContext) applicationContext).start();
        System.in.read();
    }

}

```



##  消费者项目

###  添加依赖

```xml
 <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>4.0.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-client</artifactId>
            <version>4.0.1</version>
        </dependency>
        <dependency>
            <groupId>com.sentinel</groupId>
            <artifactId>sentinel-dubbo-api</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.7.2</version>
        </dependency>
```

###  测试类

编写测试类

```java
package com.sentinel;

import org.apache.dubbo.config.annotation.Reference;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author luyanan
 * @since 2019/12/16
 * <p></p>
 **/
@RestController
public class SyaHelloController {

    @Reference
    private SentinelService sentinelService;

    @GetMapping("say")
    public String sayHello() {

        String result = sentinelService.sayello("Tom");
        System.out.println(result);
        return result;
    }

}

```



###  添加配置信息

```java
dubbo.scan.base-packages=com.sentinel
dubbo.application.name=sentinel-constumer
dubbo.registry.address=zookeeper://192.168.9.106:2181

```



###  设置限流的基准

`sentinel-provider`  用于向外界提供服务, 处理各个消费者的调用请求, 为了保护 `provider` 不被激增的流量拖垮影响稳定性, 可以向 `provider` 配置 QPS 模式的限流. 这样当每秒的请求量超过设置的阈值时会超过自动拒绝多的请求. 限流粒度可以是服务接口和服务方法两个粒度 . 若希望整个服务接口的QPS 不超过一定数值, 则可以为对应服务接口资源(resourceName 为接口全限定名) 配置QPS 阈值, 若希望服务的某个方法的QPS 不超过一定数值, 则可以为对应服务方法资源(resourceName 为接口全限定名:方法签名) 配置QPS 阈值. 

```java
package com.sentinel;


import com.alibaba.csp.sentinel.slots.block.RuleConstant;
import com.alibaba.csp.sentinel.slots.block.flow.FlowRule;
import com.alibaba.csp.sentinel.slots.block.flow.FlowRuleManager;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

public class SentinelDubboProviderApplication {

    public static void main(String[] args) throws IOException {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(ProviderConfig.class);
        ((AnnotationConfigApplicationContext) applicationContext).start();
        initFlowerRule();
        System.in.read();
    }

    private static void initFlowerRule() {
        List<FlowRule> ruleList = new ArrayList<>();

        FlowRule flowRule = new FlowRule();

        flowRule.setResource("com.sentinel.SentinelService:sayello(java.lang.String)");
        flowRule.setCount(10); // 限定阈值
        flowRule.setGrade(RuleConstant.FLOW_GRADE_QPS);//  限定阈值类型(QPS/并发线程数)
        flowRule.setLimitApp("default");// 流针对的调用来源, 若为default则不区分来源
        //流量控制手段（直接拒绝、Warm Up、匀速排队）
        flowRule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_DEFAULT);

        ruleList.add(flowRule);
        FlowRuleManager.loadRules(ruleList);

    }


}

```

启动时加入 JVM 参数 -Dcsp.sentinel.dashboard.server=localhost:8080 指定控制台地址和端口



## 参数解释

###  LimitApp

很多场景下, 根据调用方来限流也是非常重要的,比如有两个服务A 和B   都向 `service provider` 发起调用请求, 我们希望只对于来着服务 B 的请求进行限流, 则可以设置限流规则的 `limitpp` 为服务B 的名称. `Sentinel Dubbo Adapter` 会自动解析 Dubbo消费者(调用方) 的 `application name` 作为调用方名称(origin), 在进行资源保护的时候都会带上调用方的名称. 若限流规则未配置调用方(default),  则该限流规则对所有调用方生效。 若限流规则配置了调用方,则限流规则仅对指定调用方生效. 

> 注：Dubbo 默认通信不携带对端`application name` 信息, 因此需要开发者在调用端手动的将 `application anme` 置入 `attachement` 中, provider 再进行响应的解析. `sentinel Dubbo Adapter` 实现了一个Filter Adapter,  如果根据调用端限流, 可以在调用端手动将 `application name` 置入到 `attachement` 中, key为 dubboApplication`. 
>



​            



####  演示流程

1. 修改 `provider` 中限流规则:  `flowRule.setLimitApp("springboot-study");`

2. 在 consumer 工程中, 做如下处理, 其中一个通过 `attachment` 传递了一个消费者的`application name`,另外一个没有传, 通过 jemeter 工具进行测试. 

   ```java
   @GetMapping("/say")
   public String sayHello(){
   RpcContext.getContext().setAttachment("dubboApplication","springbootstudy");
   String result=sentinelService.sayHello("Mic");
   return result;
   }
   @GetMapping("/say2")
   public String say2Hello(){
   String result=sentinelService.sayHello("Mic");
   return result;
   }
   ```

   

### ControlBehavior

当QPS 超过某个阈值后, 则采取措施进行流量控制. 流量控制的手段包括以下几种: 直接拒绝、Warm up、匀速排队. 对应FlowRule 中的`controlBehavior` 字段

####  直接拒绝（RuleConstant.CONTROL_BEHAVIOR_DEFAULT）

直接拒绝方式是默认的流量控制方式, 当QPS 超过任意规则的阈值后, 新的请求会被立即拒绝. 拒绝方式为抛出`FlowException`. 这种方式适用于对系统处理能力确切已知的情况下, 比如通过压测确定了系统的准确水位时. 

####  Warm Up（RuleConstant.CONTROL_BEHAVIOR_WARM_UP）

Warm Up方式,即预热/冷启动方式, 当系统长期处于低并发的情况下, 流量突然增加到QPS的最高峰值, 可能会造成系统的瞬间流量过大把系统压垮. 所以`Warm Up` ,相当于处理请求的数量是缓慢增加的, 经过一段时间后,到达系统处理请求个数的最大值. 

####  匀速排队（RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER）

匀速排队方式会严格控制请求通过的间隔时间, 也即是让请求以均匀的速度通过, 对应的是漏桶算法. 

它的原理是: 以固定的时间间隔让请求通过, 当请求过来的时候, 如果当前请求距离上个通过的请求通过的时间间隔不小于预设值, 则让当前请求通过. 否则, 计算当前请求的预期通过时间, 如果该i请求的预期通过时间小于规则预设的 `timeout` 时间 ， 则该请求会等待直到预设时间来通过. 反之, 则马上抛出阻塞异常. 

可以设置一个最长排队等待时间:  flowRule.setMaxQueueingTimeMs(5 * 1000); // 最长排队等待时 间：5s

这种方式主要用于处理间隔性突发的流量, 假如消息队列, 想象一下这样的场景, 在某一秒有大量的请求到来, 而接下来的几秒则处于空闲状态, 我们希望系统能够在接下来的空闲期间逐渐处理这些请求，而不是在第一秒就直接拒绝多余的请求. 

## 如何实现分布式限流

在前面的案例中,我们只是基于`sentinel`的基本使用和单机限流的使用, 假设有这样一个场景, 我们现在把`provider` 部署了10个集群, 希望调用这个服务器的api 的总的QPS 是100, 意味着每一台集群的QPS 是10, 理想情况下总的QPS 就是100, 但是实际上由于负载均衡的流量分发并不是非常均匀, 就会导致总的QPS 不足100时就会被限了. 在这个场景中, 仅仅依靠单机来实现总的流量的控制是有问题的, 所以最好的能实现集群限流. 

###  架构图

要想使用集群流控功能, 我们需要在应用端配置动态规则源, 并通过 `sentinel`控制台进行推送, 如下图所示:

![](http://files.luyanan.com//img/20191220135530.png)

###  搭建token-server

![](http://files.luyanan.com//img/20191220151145.png)



#### 添加jar依赖

```xml

        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-cluster-server-default</artifactId>
            <version>1.6.3</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-transport-simple-http</artifactId>
            <version>1.6.3</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
            <version>1.6.3</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
            <version>1.7.25</version>
        </dependency>

```



####  编写启动类TokenClusterServer

```java
public class TokenClusterServer {


    public static void main(String[] args) {
        ClusterTokenServer tokenServer = new SentinelDefaultTokenServer();
        ClusterServerConfigManager.loadGlobalTransportConfig(new ServerTransportConfig()
                .setIdleSeconds(600)
                .setPort(9999));

        ClusterServerConfigManager.loadServerNamespaceSet(Collections.singleton("App"));

        try {
            tokenServer.start();
        } catch (Exception e) {
            e.printStackTrace();
        }

    }


}

```



####  DataSourceInitFunc

```java
public class DataSourceInitFunc implements InitFunc {


    /**
     * <p>nacos 的远程服务host</p>
     *
     * @author luyanan
     * @since 2019/12/20
     */
    private final String remoteAddr = "192.168.86.128";

    /**
     * <p>nacos groupId</p>
     *
     * @author luyanan
     * @since 2019/12/20
     */
    private final String groupId = "SENTINEL_GROUP";


    /**
     * <p>namespace 不同, 限流规则不同</p>
     *
     * @author luyanan
     * @since 2019/12/20
     */
    private static final String FLOW_POSTFIX = "-flow-rules";

    @Override
    public void init() throws Exception {
        ClusterFlowRuleManager.setPropertySupplier(namespace -> {
            ReadableDataSource<String, List<FlowRule>> dataSource =
                    new NacosDataSource<List<FlowRule>>(remoteAddr, groupId, namespace + FLOW_POSTFIX,
                            s -> JSON.parseObject(s, new TypeReference<List<FlowRule>>() {
                            }));
            return dataSource.getProperty();
        });

    }
}
```



####  resources 目录添加扩展点

/META-INF/services/com.alibaba.csp.sentinel.init.InitFunc = 自定义扩展点

>  com.sentinel.token.DataSourceInitFunc

####  启动Sentinel dashboard

> java -Dserver.port=8081 -Dcsp.sentinel.dashboard.server=localhost:8080 - Dproject.name=sentinel-dashboard -jar sentinel-dashboard-1.6.3.jar



####   启动nacos 并增加配置

1. 启动nacos 服务 

   > nohup sh startup.sh -m standalone &

2. 增加限流配置

    ![](http://files.luyanan.com//img/20191220152626.png)

   



####  配置JVM参数

配置如下JVM参数, 连接到 `sentinel dashboard`, 

```bash
-Dproject.name=App -Dcsp.sentinel.dashboard.server=192.168.86.128:8081 -
Dcsp.sentinel.log.use.pid=true
```



> 服务启动后, 在 `$user.home$/logs/csp/`  可以找到`sentinel-record.log.pid*.date` 文件, 如果看到日志文件中获取了远程服务的信息, 说明 `token-server` 启动成功了. 

### Dubbo 接入分布式限流

#### jar依赖

```xml
<dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-dubbo-adapter</artifactId>
            <version>1.6.3</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-cluster-client-default</artifactId>
            <version>1.6.3</version>
        </dependency>

        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
            <version>1.6.3</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-annotation-aspectj</artifactId>
            <version>1.6.3</version>
        </dependency>

        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-transport-simple-http</artifactId>
            <version>1.6.3</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-core</artifactId>
            <version>1.6.3</version>
        </dependency>

        <dependency>
            <groupId>com.sentinel</groupId>
            <artifactId>sentinel-dubbo-api</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>

        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-client</artifactId>
            <version>4.0.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>4.0.1</version>
        </dependency>

        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.7.2</version>
        </dependency>
```



#### 增加扩展点

扩展点需要在resources/META-INF/services/增加扩展的配置

> com.alibaba.csp.sentinel.init.InitFunc = 自定义扩展点

```java
package com.sentinel2;

import com.alibaba.csp.sentinel.cluster.client.ClientConstants;
import com.alibaba.csp.sentinel.cluster.client.config.ClusterClientAssignConfig;
import com.alibaba.csp.sentinel.cluster.client.config.ClusterClientConfig;
import com.alibaba.csp.sentinel.cluster.client.config.ClusterClientConfigManager;
import com.alibaba.csp.sentinel.datasource.ReadableDataSource;
import com.alibaba.csp.sentinel.datasource.nacos.NacosDataSource;
import com.alibaba.csp.sentinel.init.InitFunc;
import com.alibaba.csp.sentinel.slots.block.flow.FlowRule;
import com.alibaba.csp.sentinel.slots.block.flow.FlowRuleManager;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.TypeReference;
import org.aspectj.weaver.ast.Or;

import javax.xml.crypto.Data;
import java.util.List;

/**
 * @author luyanan
 * @since 2019/12/20
 * <p></p>
 **/
public class DataSourceInitFunc implements InitFunc {


    /**
     * <p>token server 的ip</p>
     *
     * @author luyanan
     * @since 2019/12/20
     */
    private final String CLUSTER_SERVER_HOST = "192.168.86.128";


    /**
     * <p>token-server 的端口</p>
     *
     * @author luyanan
     * @since 2019/12/20
     */
    private static final int CLUSTER_SERVER_PORT = 9999;


    /**
     * <p>请求超时时间</p>
     *
     * @author luyanan
     * @since 2019/12/20
     */
    private static final int REQUEST_TIME_OUT = 20000;


    private final static String APP_NAME = "APP";

    /**
     * <p>nacos 服务的ip</p>
     *
     * @author luyanan
     * @since 2019/12/20
     */
    private final static String REOMOTE_ADDR = "192.168.86.128";

    /**
     * <p>group id</p>
     *
     * @author luyanan
     * @since 2019/12/20
     */
    private final static String GROUP_ID = "SENTINEL_GROUP";

    /**
     * <p>限流规则后缀</p>
     *
     * @author luyanan
     * @since 2019/12/20
     */
    private static final String FLOW_POSTFIX = "-flow-rules";


    @Override
    public void init() throws Exception {
        loadClusterClientConfig();
        registerClusterFlowRuleProperty();
    }

    /**
     * 注册动态规则 Property,
     * 当Client与Server连接中断, 退化为本地限流时需要用到的规则
     * 该配置为必选项,客户端会从nacos 上加载限流规则, 请求tokenserver 的时候, 会带上要check 的规则id
     */
    private void registerClusterFlowRuleProperty() {
        // 使用Nacos 数据源作为配置中心, 需要在REMOTE_ADDRSS 上启动一个Nacos 的服务
        ReadableDataSource<String, List<FlowRule>> ds = new
                NacosDataSource<List<FlowRule>>(REOMOTE_ADDR, GROUP_ID, APP_NAME + FLOW_POSTFIX,
                source -> JSON.parseObject(source, new
                        TypeReference<List<FlowRule>>() {
                        }));

        //  为集群客户端注册动态规则源
        FlowRuleManager.register2Property(ds.getProperty());

    }

    /**
     * 通过硬编码的方式,配置连接到token-server 服务的地址，(这种在实际使用过程中不建议使用, 后续可以基于动态配置源改造)
     */
    private void loadClusterClientConfig() {
        ClusterClientAssignConfig assignConfig = new ClusterClientAssignConfig();
        assignConfig.setServerHost(CLUSTER_SERVER_HOST);
        assignConfig.setServerPort(CLUSTER_SERVER_PORT);

        ClusterClientConfigManager.applyNewAssignConfig(assignConfig);
        ClusterClientConfig clientConfig = new ClusterClientConfig();
        clientConfig.setRequestTimeout(REQUEST_TIME_OUT); //  token-client  请求token-server 获取令牌的时间
        ClusterClientConfigManager.applyNewConfig(clientConfig);

    }
}

```



####  配置JVM参数

这里的project_name  要包含在 token-server 中配置的 namesapce

token-server 会根据客户端对应的namespace(默认为 project.name定义的应用名)下的连接数来计算总的阈值

> -Dproject.name=App -Dcsp.sentinel.dashboard.server=192.168.86.128:8081 - Dcsp.sentinel.log.use.pid=true



> 服务启动后, 在 `$user.home$/logs/csp/` 可以找到`sentinel-record.log.pid*.date` 文件, 如果看到日志文件中获取到了 token-server 的信息, 说明连接成功了 

####  演示集群限流

所谓集群限流, 就是多个服务节点使用同一个限流规则, 从而对多个节点的总流量进行限制, 添加一个 sentinel-server , 同时运行两个程序. 



