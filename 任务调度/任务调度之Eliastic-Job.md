# 任务调度之Eliastic-Job.md

## 1.1 任务调度高级需求

`Quartz` 的不足:

1. 作业只能通过DB 抢占随机负载, 无法协调. 
2. 作业不能分片, 单个任务数据太多了跑不完,消耗线程,负载不均匀. 
3. 作业日志可视化监控,统计. 



### 1.2 发展历史

`Elastic-Job` 是怎么来的?

在当当的`ddframe` 框架中, 需要一个任务调度系统(作业系统)

![image-20200411134048082](C:/Users/luyanan/AppData/Roaming/Typora/typora-user-images/image-20200411134048082.png)

实现的话有两种思路: 一种是修改开源产品, 一种是基于开源产品搭建(封装), 当当选择了后者,最开始这个调度系统叫做`dd-job`.它是一个无中心化的分布式任务调度框架。 因为数据库缺少分布式协调功能(必须选主), 替换为`Zookeeper`后, 增加了弹性扩容和数据分片的功能. 

`Elastic-Job` 是`ddframe` 中的`dd-job` 作业模块分离出来的作业框架, 基于`Quartz`和`Curator`开发,在2015年开源. 

轻量级,无中心化的解决方案.

为什么哦是无中心化呢? 因为没有统一的调度中心, 集群的每个节点都是对等的, 节点之间通过注册中心进行分布式协调. `Elastic-Job`存在主节点的概念, 但是主节点没有调度的功能,而是用于处理一些集中式的任务, 如分片、清理运行时信息等. 

思考: 如果ZK挂了怎么办？ 

每个任务都有自己独立的线程池. 

从官网开始:

http://elasticjob.io/docs/elastic-job-lite/00-overview/

https://github.com/elasticjob

`Elastic-Job` 最开始只有一个`elastic-job-core` 的项目, 在2.X版本之后分为`Elastic-job-Lite`和`Elastic-job-Cloud` 两个子项目. 其中`Elastic-job-Lite` 定位为轻量级无中心化的解决方案,使用`jar` 包的形式来提供分布式任务的协调服务. 而`Elastic-job-Cloud` 使用`Mesos+Docker` 的解决方案, 额外提供资源管理、应用分发以及进程隔离服务(跟`Lite`的区别只是部署方式不同,他们使用相同的API, 只要开发一次)

### 1.3 功能特性

-  分布式调度协调:使用ZK 实现注册中心
- 错误执行作用重触发(`Misfire`)
- 支持并行调度(任务分片)
- 作业分片一致性,保证同一个分片在分布式环境中仅有一个执行实例
- 弹性扩容缩容: 将任务拆分成N个任务后, 各个服务区分别执行各自分配到的任务项, 一旦有新的服务器加入集群, 或现有服务器下线, `Elastic-job`将在保留本地任务不变的情况下, 下次任务开始前触发任务重分片. 
- 失效转移(`failover`): 弹性扩容缩容在下次作业运行前重新分片, 但本次作用执行的过程中, 下线的服务器所分配的作业将不会重新分配. 失效转移功能可以在本地作用运行中用空闲服务器抓取孤儿作业分片执行. 同样, 失效转移功能也会牺牲部分性能. 
- 支持作业生命周期操作(`listener`)
- 丰富的作业类型(`Simple`、`DataFlow`、`Script`)
- Spring整合以及命名空间提供
- 运维平台



### 1.4 项目架构

应用在各自的节点上执行任务, 通过ZK 注册中心协调,节点注册、节点选举、任务分片、监听都在`Elastic-Job` 的代码中完成. 

![image-20200411142622344](http://files.luyanan.com//img/20200411142623.png)



## 2. Java 开发

### 2.1 `pom` 依赖

```xml
 <dependency>
            <groupId>com.dangdang</groupId>
            <artifactId>elastic-job-lite-core</artifactId>
            <version>2.1.5</version>
        </dependency>
```



### 2.2 任务类型

任务类型有三种:

#### 2.2.1 `SimpleJob`

`SimpleJob`: 简单实现,非经过任何封装的类型,需要实现`SimpleJob`接口

```java
public class MySimpleJob implements SimpleJob {


    @Override
    public void execute(ShardingContext shardingContext) {
        System.out.println(String.format("分片项 ShardingItem: %s | 运行时间: %s | 线程ID: %s | 分片参数: %s ",
                shardingContext.getShardingItem(), new SimpleDateFormat("HH:mm:ss").format(new Date()),
                Thread.currentThread().getId(), shardingContext.getShardingParameter()));
    }
}

```

测试类

```java
/**
 * @author luyanan
 * @since 2020/4/11
 * <p>测试类</p>
 **/
public class SimpleJobTest {

    public static void main(String[] args) {
        // zk注册中心
        ZookeeperRegistryCenter registryCenter = new ZookeeperRegistryCenter(new ZookeeperConfiguration("192.168.31.22:2181", "ejob-standalone"));
        registryCenter.init();


        // 数据源，使用DBCP
/*        BasicDataSource dataSource = new BasicDataSource();
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        dataSource.setUrl("jdbc:mysql://localhost:3306/elastic_job_log");
        dataSource.setUsername("root");
        dataSource.setPassword("123456");
        JobEventConfiguration jobEventConfig = new JobEventRdbConfiguration(dataSource);*/

        // 定义作业核心配置
        JobCoreConfiguration coreConfiguration = JobCoreConfiguration.newBuilder(MySimpleJob.class.getSimpleName(),
                "0/2 * * * * ?", 4).shardingItemParameters("0=RDP, 1=CORE, 2=SIMS, 3=ECIF").build();
        // 定义Simple 类型配置
        SimpleJobConfiguration simpleJobConfiguration = new SimpleJobConfiguration(coreConfiguration, MySimpleJob.class.getCanonicalName());

        // 作业分片策略
        // 基于平均分配算法的分片策略
        String jobShardingStrategyClass = AverageAllocationJobShardingStrategy.class.getCanonicalName();
        // 定义Lite 作业根配置
        // LiteJobConfiguration simpleJobRootConfig = LiteJobConfiguration.newBuilder(simpleJobConfig).jobShardingStrategyClass(jobShardingStrategyClass).build();
        LiteJobConfiguration simpleJobRootConfig = LiteJobConfiguration.newBuilder(simpleJobConfiguration).build();

        // 构建Job
        new JobScheduler(registryCenter, simpleJobRootConfig).init();
        // new JobScheduler(regCenter, simpleJobRootConfig, jobEventConfig).init();
    }

}
```

结果:

```text
分片项 ShardingItem: 2 | 运行时间: 16:00:44 | 线程ID: 27 | 分片参数: SIMS 
分片项 ShardingItem: 3 | 运行时间: 16:00:44 | 线程ID: 28 | 分片参数: ECIF 
分片项 ShardingItem: 0 | 运行时间: 16:00:44 | 线程ID: 25 | 分片参数: RDP 
分片项 ShardingItem: 1 | 运行时间: 16:00:44 | 线程ID: 26 | 分片参数: CORE 
分片项 ShardingItem: 0 | 运行时间: 16:00:46 | 线程ID: 29 | 分片参数: RDP 
分片项 ShardingItem: 1 | 运行时间: 16:00:46 | 线程ID: 30 | 分片参数: CORE 
分片项 ShardingItem: 2 | 运行时间: 16:00:46 | 线程ID: 31 | 分片参数: SIMS 
分片项 ShardingItem: 3 | 运行时间: 16:00:46 | 线程ID: 32 | 分片参数: ECIF 
分片项 ShardingItem: 0 | 运行时间: 16:00:48 | 线程ID: 28 | 分片参数: RDP 
分片项 ShardingItem: 1 | 运行时间: 16:00:48 | 线程ID: 27 | 分片参数: CORE 
分片项 ShardingItem: 2 | 运行时间: 16:00:48 | 线程ID: 28 | 分片参数: SIMS 
分片项 ShardingItem: 3 | 运行时间: 16:00:48 | 线程ID: 25 | 分片参数: ECIF 
分片项 ShardingItem: 0 | 运行时间: 16:00:50 | 线程ID: 29 | 分片参数: RDP 
分片项 ShardingItem: 1 | 运行时间: 16:00:50 | 线程ID: 30 | 分片参数: CORE 
分片项 ShardingItem: 2 | 运行时间: 16:00:50 | 线程ID: 31 | 分片参数: SIMS 
分片项 ShardingItem: 3 | 运行时间: 16:00:50 | 线程ID: 32 | 分片参数: ECIF 
分片项 ShardingItem: 0 | 运行时间: 16:00:52 | 线程ID: 27 | 分片参数: RDP 
分片项 ShardingItem: 1 | 运行时间: 16:00:52 | 线程ID: 26 | 分片参数: CORE 
分片项 ShardingItem: 3 | 运行时间: 16:00:52 | 线程ID: 28 | 分片参数: ECIF 
分片项 ShardingItem: 2 | 运行时间: 16:00:52 | 线程ID: 27 | 分片参数: SIMS 
```



####  2.2.2 `DataFlowJob`

`DataFlowJob`: `DataFlow` 类型用于处理数据流, 必须实现`fetchData()`和 `processData()` 的方法, 一个用来获取数据, 一个用来处理获取到的数据. 

```java
package com.job.elasticjob.dataflow;

import com.dangdang.ddframe.job.api.ShardingContext;
import com.dangdang.ddframe.job.api.dataflow.DataflowJob;

import java.util.Arrays;
import java.util.List;

/**
 * @author luyanan
 * @since 2020/4/11
 * <p>数据流</p>
 **/
public class MyDataFlowJob implements DataflowJob<String> {


    @Override
    public List<String> fetchData(ShardingContext shardingContext) {
        System.out.println("开始获取数据");
        return Arrays.asList("张三", "李四", "王五");
    }

    @Override
    public void processData(ShardingContext shardingContext, List<String> list) {
        // 需要移除数据, 不然会一直循环
        for (String s : list) {
            System.out.println("开始处理任务:" + s);
        }
    }
}

```

测试类

```java
public class DataFlowJobTest {

    public static void main(String[] args) {
        // zk注册中心
        ZookeeperRegistryCenter registryCenter = new ZookeeperRegistryCenter(new ZookeeperConfiguration("192.168.31.22:2181", "ejob-standalone-dataflow"));
        registryCenter.init();

        // 定义作业核心配置
        JobCoreConfiguration coreConfiguration = JobCoreConfiguration.newBuilder(MyDataFlowJob.class.getSimpleName(),
                "0/2 * * * * ?", 4).build();
        // 定义Simple 类型配置
        DataflowJobConfiguration dataflowJobConfiguration = new DataflowJobConfiguration(coreConfiguration,
                MyDataFlowJob.class.getCanonicalName(), true);

        // 作业分片策略
        // 基于平均分配算法的分片策略
        String jobShardingStrategyClass = AverageAllocationJobShardingStrategy.class.getCanonicalName();
        // 定义Lite 作业根配置
        // LiteJobConfiguration simpleJobRootConfig = LiteJobConfiguration.newBuilder(simpleJobConfig).jobShardingStrategyClass(jobShardingStrategyClass).build();
        LiteJobConfiguration simpleJobRootConfig = LiteJobConfiguration.newBuilder(dataflowJobConfiguration).build();

        // 构建Job
        new JobScheduler(registryCenter, simpleJobRootConfig).init();
        // new JobScheduler(regCenter, simpleJobRootConfig, jobEventConfig).init();

    }

}

```

结果:

```tex
开始获取数据
开始处理任务:张三
开始处理任务:李四
开始处理任务:王五
```



#### 2.2.3 `ScriptJob`

`ScriptJob`:`Script` 类型作业为脚本类型作业, 支持`Shell`、`Python`、`Perl` 等所有类型脚本, 只需要指定脚本的内容或者位置. 

```java
    public static void main(String[] args) {
        // zk注册中心
        ZookeeperRegistryCenter registryCenter = new ZookeeperRegistryCenter(
                new ZookeeperConfiguration("192.168.31.22:2181", "ejob-standalone-script"));
        registryCenter.init();
        //定义作业核心配置
        JobCoreConfiguration jobCoreConfiguration = JobCoreConfiguration
                .newBuilder("scriteJob", "0/4 * * * * ?", 2)
                .build();
        //定义script类型配置
        ScriptJobConfiguration scriptJobConfiguration = new ScriptJobConfiguration(jobCoreConfiguration, "D:/1.bat");
        // 作业分片策略
        // 基于平均分配算法的分片策略
        String jobShardingStrategyClass = AverageAllocationJobShardingStrategy.class.getCanonicalName();

        // 定义Lite作业根配置
        // LiteJobConfiguration scriptJobRootConfig = LiteJobConfiguration.newBuilder(scriptJobConfig).jobShardingStrategyClass(jobShardingStrategyClass).build();
        LiteJobConfiguration scriptJobRootConfig = LiteJobConfiguration.newBuilder(scriptJobConfiguration).build();

        // 构建Job
        new JobScheduler(registryCenter, scriptJobRootConfig).init();
        // new JobScheduler(regCenter, scriptJobRootConfig, jobEventConfig).init();

    }
```



### 2.3 `Elastic-Job` 配置

#### 2.3.1 配置步骤

配置手册: http://elasticjob.io/docs/elastic-job-lite/02-guide/config-manual/

1. `ZK` 注册中心配置

2. 作业级别(从底层往下层: `core`-`Type`-`Lite`)

    

   | 配置级别 | 配置类                 | 配置内容                                                     |
   | -------- | ---------------------- | ------------------------------------------------------------ |
   | `core`   | `JobCoreConfiguration` | 用于提供作业核心配置信息，如：作业名称、CRON 表达式、分片总数等。 |
   | `type`   | `JobTypeConfiguration` | 有 3 个子类分别对应 `SIMPLE`, `DATAFLOW` 和 `SCRIPT` 类型作业，提供 3 种作 业需要的不同配置，如：`DATAFLOW` 类型是否流式处理或 `SCRIPT` 类型的命 令行等。`Simple` 和 `DataFlow` 需要指定任务类的路径 |
   | `Root`   | `JobRootConfiguration` | 有 2 个子类分别对应 Lite 和 Cloud 部署类型，提供不同部署类型所需的配 置，如：Lite 类型的是否需要覆盖本地配置或 Cloud 占用 CPU 或 Memory 数量等。<br />可以定义分片策略：http://elasticjob.io/docs/elastic-job-lite/02-guide/job-sharding-strategy/ |

   

```java
 public static void main(String[] args) {
        // zk注册中心
        ZookeeperRegistryCenter registryCenter = new ZookeeperRegistryCenter(new ZookeeperConfiguration("192.168.31.22:2181", "ejob-standalone"));
        registryCenter.init();


        // 数据源，使用DBCP
/*        BasicDataSource dataSource = new BasicDataSource();
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        dataSource.setUrl("jdbc:mysql://localhost:3306/elastic_job_log");
        dataSource.setUsername("root");
        dataSource.setPassword("123456");
        JobEventConfiguration jobEventConfig = new JobEventRdbConfiguration(dataSource);*/

        // 定义作业核心配置
        JobCoreConfiguration coreConfiguration = JobCoreConfiguration.newBuilder(MySimpleJob.class.getSimpleName(),
                "0/2 * * * * ?", 4).shardingItemParameters("0=RDP, 1=CORE, 2=SIMS, 3=ECIF").build();
        // 定义Simple 类型配置
        SimpleJobConfiguration simpleJobConfiguration = new SimpleJobConfiguration(coreConfiguration, MySimpleJob.class.getCanonicalName());

        // 作业分片策略
        // 基于平均分配算法的分片策略
        String jobShardingStrategyClass = AverageAllocationJobShardingStrategy.class.getCanonicalName();
        // 定义Lite 作业根配置
        // LiteJobConfiguration simpleJobRootConfig = LiteJobConfiguration.newBuilder(simpleJobConfig).jobShardingStrategyClass(jobShardingStrategyClass).build();
        LiteJobConfiguration simpleJobRootConfig = LiteJobConfiguration.newBuilder(simpleJobConfiguration).build();

        // 构建Job
        new JobScheduler(registryCenter, simpleJobRootConfig).init();
        // new JobScheduler(regCenter, simpleJobRootConfig, jobEventConfig).init();
    }

```

作业配置分为三级, 分别是`JobCoreConfiguration`、`JobTypeConfiguration`和`LiteJobConfiguration`.`LiteJobConfiguration` 使用`JobTypeConfiguration`,`JobTypeConfiguration` 使用`JobCoreConfiguration`, 层层嵌套

![image-20200411161902396](http://files.luyanan.com//img/20200411161903.png)

`JobTypeConfiguration` 根据不同的实现类型分为`SimpleJobConfiguration` ， `DataflowJobConfiguration` 和 `ScriptJobConfiguration`。

`Elastic-Job` 使用zk 做分布式协调,所有的配置都会写入到ZK中. 

#### 2.3.2 ZK注册中心数据结构

一个任务, 一个二级节点

这里面有些节点是临时节点,只有任务运行的是才能看到. 

> 注意：修改了任务重新运行任务不生效, 是因为zk的信息不会更新,除非把`overwrite`修改成``true`
>

##### config节点

JSON 格式存储

存储任务的配置信息,包括执行类,`cron` 表达式, 分片算法类、分片数量、分片参数等. 



```json
{
    "cron": "0/2 * * * * ?",
    "description": "",
    "disabled": false,
    "failover": false,
    "jobClass": "job.MySimpleJob",
    "jobName": "MySimpleJob",
    "jobParameter": "",
    "jobProperties": {
        "executor_service_handler": "com.dangdang.ddframe.job.executor.handler.impl.DefaultExecutorServiceHandler",
        "job_exception_handler": "com.dangdang.ddframe.job.executor.handler.impl.DefaultJobExceptionHandler"
    },
    "jobShardingStrategyClass": "",
    "jobType": "SIMPLE",
    "maxTimeDiffSeconds": -1,
    "misfire": true,
    "monitorExecution": true,
    "monitorPort": -1,
    "overwrite": false,
    "reconcileIntervalMinutes": 10,
    "shardingItemParameters": "",
    "shardingTotalCount": 1
}
```

`config`节点的数据是通过`ConfigService` 持久化到`zookeeper`中去的, 默认情况下, 如果你修改了`job`的配置, 比如`cron` 表达式、分片数量等是不会更新到`zookeeper`上去的, 除非你在`Lite`级别的配置把参数`overwrite` 修改成`true`

```java
LiteJobConfiguration simpleJobRootConfig = LiteJobConfiguration.newBuilder(simpleJobConfig).overwrite(true).build();
```

##### `instances` 节点

同一个`Job`下的`elasticJob` 的部署实例, 一台机器上可以运行多个`Job`实例, 也就是`Jar`包,`instances` 的命令是`Ip+@-@+PID`. 只有在运行的时候能看到. 

![image-20200411215538575](http://files.luyanan.com//img/20200411215604.png)



#####  `Leader`节点

![image-20200411215703352](http://files.luyanan.com//img/20200411215705.png)

任务实例的主节点信息,通过`zookeeper` 的主节点选举,选出来的主节点信息, 在`elastic -job` 中, 任务的执行可以分布在不同的实例(节点)中, 但是任务分片等核心控制 ,需要由主节点完成, 因为, 任务执行前, 需要选举出主节点. 

下面有三个子节点: 

- `election`: 主节点选举
- `sharding`: 分片
- `failover`:失效转移

`election`:下面的`instance`节点显示了当前主节点的实例ID：`JobInstanceId`

`sharding`:节点下面有一个临时节点, `necessary`,是否需要重新分片的标记. 如果分片数总数变化, 或者任务实例节点上下线或启动/禁用,以及主节点选举, 都会触发设置重分片标记, 主节点会进行分片计算. 

##### `Servers`节点

![image-20200411220850130](http://files.luyanan.com//img/20200411220851.png)

任务实例的信息, 主要是IP地址, 任务实例的IP地址, 跟`instances`不同, 如果多个任务实例在同一台机器上运行则会只出现一个IP子节点, 可在IP地址节点写入`DISABLED` 表示该任务实例禁用. 

######  `sharding` 节点

![image-20200411221249072](http://files.luyanan.com//img/20200411221250.png)

任务的分片信息, 子节点是分片项序号, 从0开始. 分片个数是在任务配置中设置的, 分片项序号的子节点存储详细信息, 每个分片项下的子节点用于控制和记录分片运行状态 , 最主要的子节点就是`instance`.

| 子节点名   | 是否临时节点 | 描述                                                         |
| ---------- | ------------ | ------------------------------------------------------------ |
| `instance` | 否           | 执行该分片项的作业运行实例主键                               |
| `running`  | 是           | 分片项正在运行的状态<br />仅配置`monitorExecution` 时有效    |
| `failover` | 是           | 如果该分片项被失效转移分配给其他作业服务器, 则该节点值记录执行此分派你的作业服务器IP |
| `misfire`  | 否           | 是否开启错误任务重新执行                                     |
| `disabled` | 否           | 是否禁用此分片项                                             |



## 3. 运维平台

### 3.1 下载解压运行

git 下载源码 https://github.com/elasticjob/elastic-job-lite

对`elastic-job-lite-console`打包得到安装包. 

解压缩`elastic-job-lite-console-${version}.tar.gz` 并执行`bin\start.sh（Windows 运行.bat）`. 打开浏览器访问`http://localhost:8899/` 即可访问控制台. 

8899为默认端口号, 可通过启动脚本输入`-p` 自定义端口号.

默认管理员用户名和密码是`root/root`.右上角可以切换语言. 

### 3.2 添加ZK注册中心

第一步,添加注册中心, 输入ZK地址和命名空间, 并连接

运维平台和`elastic-job` 并无关系,是通过读取作业注册中心数据展现作业状态,或更新注册中心数据修改全局配置.

控制台只能控制作业本身是否运行,但不能控制作业进程的启动, 因为控制台和作业本身是完全分离的,控制台并不能控制作业服务器. 

可以对作业机型操作. 

![image-20200412101811263](http://files.luyanan.com//img/20200412101813.png)

修改作业

![image-20200412101833142](http://files.luyanan.com//img/20200412101834.png)

### 3.3 事件追踪

http://elasticjob.io/docs/elastic-job-lite/02-guide/event-trace/

`Elastic-Job` 提供了事件追踪功能, 可以通过事件订阅的方式处理调度过程的重要事件, 用于查询、统计和监控. 

`Elastic-Job-Lite` 在配置中提供了`JobEventConfiguration`, 目前支持数据库方式配置. 

```java
 BasicDataSource dataSource = new BasicDataSource();
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        dataSource.setUrl("jdbc:mysql://localhost:3306/elastic_job_log");
        dataSource.setUsername("root");
        dataSource.setPassword("123456");
        JobEventConfiguration jobEventConfig = new JobEventRdbConfiguration(dataSource);
```

事件追踪的`event_trace_rdb_url` 属性对应库自动创建`JOB_EXECUTION_LOG` 和`JOB_STATUS_TRACE_LOG` 两张表以及若干索引

![image-20200412202323391](http://files.luyanan.com//img/20200412202333.png)

需要在运维平台中添加数据源信息,并且连接:

![image-20200412202504071](http://files.luyanan.com//img/20200412202505.png)

在作业中查询:

![image-20200412202529730](http://files.luyanan.com//img/20200412202530.png)

##  4.Spring 集成与分片详解


### 4.1 pom依赖

```xml
 <elastic-job.version>2.1.5</elastic-job.version>
<dependency>
            <groupId>com.dangdang</groupId>
            <artifactId>elastic-job-lite-core</artifactId>
            <version>${elastic-job.version}</version>
        </dependency>
        <!-- elastic-job-lite-spring -->
        <dependency>
            <groupId>com.dangdang</groupId>
            <artifactId>elastic-job-lite-spring</artifactId>
            <version>${elastic-job.version}</version>
        </dependency>
```



### 4.2 `application.properties` 定义配置类和任务类中要用到的参数

```properties
server.port=${random.int[10000,19999]}
regCenter.serverList = localhost:2181
regCenter.namespace = ejob-springboot

elasticJob.cron = 0/3 * * * * ?
elasticJob.shardingTotalCount = 4
elasticJob.shardingItemParameters = 0=aaa,1=bbb, 2=mic, 3=ccc
elasticJob.jobParameter = 2673
```





### 4.3  创建任务

创建任务类,加上`@Component` 注解

```java
@Component
public class MySimpleJob implements SimpleJob {


    @Override
    public void execute(ShardingContext shardingContext) {
        System.out.println(String.format("------【简单任务】Thread ID: %s, %s,任务总片数: %s, " +
                        "当前分片项: %s.当前参数: %s," +
                        "当前任务名称: %s.当前任务参数 %s",
                Thread.currentThread().getId(),
                new SimpleDateFormat("HH:mm:ss").format(new Date()),
                shardingContext.getShardingTotalCount(),
                shardingContext.getShardingItem(),
                shardingContext.getShardingParameter(),
                shardingContext.getJobName(),
                shardingContext.getJobParameter()

        ));
    }
}
```



### 4.4  注册中心配置

`Bean`的`initMethod`  属性用来指定`Bean` 初始化完成之后要执行的方法, 用来替代继承`InitializingBean`  接口, 以便在容器启动的时候创建注册中心. 

```java
@Configuration
public class ElasticRegistryCenterConfig {

    @Bean(initMethod = "init")
    public ZookeeperRegistryCenter zookeeperRegistryCenter(
            @Value("${regCenter.serverList}") String serverList,
            @Value("${regCenter.namespace}") String namespace) {

        return new ZookeeperRegistryCenter(
                new ZookeeperConfiguration(serverList, namespace));
    }

}
```



### 4.5  作业三级配置

`Core` - `Type` -`Lite`

```java
@Configuration
public class ElasticJobConfig {
    @Autowired
    private ZookeeperRegistryCenter regCenter;

    // 定义调度器
    @Bean(initMethod = "init")
    public JobScheduler simpleJobScheduler(final MySimpleJob simpleJob,
                                           @Value("${elasticJob.cron}") final String cron,
                                           @Value("${elasticJob.shardingTotalCount}") final int shardingTotalCount,
                                           @Value("${elasticJob.shardingItemParameters}") final String shardingItemParameters,
                                           @Value("${elasticJob.jobParameter}") final String jobParameter) {
        return new SpringJobScheduler(simpleJob, regCenter,
                getLiteJobConfiguration(simpleJob.getClass(), cron, shardingTotalCount, shardingItemParameters, jobParameter));
    }

    // 配置任务详细信息
    private LiteJobConfiguration getLiteJobConfiguration(final Class<? extends SimpleJob> jobClass,
                                                         final String cron,
                                                         final int shardingTotalCount,
                                                         final String shardingItemParameters,
                                                         final String jobParameter) {
        return LiteJobConfiguration.newBuilder(new SimpleJobConfiguration(
                JobCoreConfiguration.newBuilder(jobClass.getName(), cron, shardingTotalCount)
                        .shardingItemParameters(shardingItemParameters).jobParameter(jobParameter).build()
                , jobClass.getCanonicalName())
        ).overwrite(true).build();
    }

    @Bean(initMethod = "init")
    public JobScheduler dataFlowJobScheduler(final MyDataFlowJob dataFlowJob,
                                             @Value("${elasticJob.dataflow.cron}") final String cron,
                                             @Value("${elasticJob.dataflow.shardingTotalCount}") final int shardingTotalCount,
                                             @Value("${elasticJob.dataflow.shardingItemParameters}") final String shardingItemParameters) {
        return new SpringJobScheduler(dataFlowJob, regCenter, getDataFlowJobConfiguration(dataFlowJob.getClass(), cron, shardingTotalCount, shardingItemParameters, true));
    }

    private LiteJobConfiguration getDataFlowJobConfiguration(final Class<? extends DataflowJob> jobClass, //任务类
                                                             final String cron,    // 运行周期配置
                                                             final int shardingTotalCount,  //分片个数
                                                             final String shardingItemParameters,
                                                             final Boolean streamingProcess   //是否是流式作业
    ) {  // 分片参数
        return LiteJobConfiguration.newBuilder(new DataflowJobConfiguration(
                JobCoreConfiguration.newBuilder(jobClass.getName(), cron, shardingTotalCount)
                        .shardingItemParameters(shardingItemParameters).build()
                // true为流式作业,除非fetchData返回数据为null或者size为0,否则会一直执行
                // false非流式,只会按配置时间执行一次
                , jobClass.getCanonicalName(), streamingProcess)
        ).overwrite(true).build();
    }

    @Bean(initMethod = "init")
    public JobScheduler scriptJobScheduler(@Value("${elasticJob.script.cron}") final String cron,
                                           @Value("${elasticJob.script.shardingTotalCount}") final int shardingTotalCount,
                                           @Value("${elasticJob.script.shardingItemParameters}") final String shardingItemParameters) {
        return new SpringJobScheduler(null, regCenter,
                getScriptJobConfiguration("script_job", cron, shardingTotalCount, shardingItemParameters,
                        "D:/1.bat"));
    }

    private LiteJobConfiguration getScriptJobConfiguration(final String jobName, //任务名字
                                                           final String cron,    // 运行周期配置
                                                           final int shardingTotalCount,  //分片个数
                                                           final String shardingItemParameters,
                                                           final String scriptCommandLine   //是脚本路径或者命令
    ) {  // 分片参数
        return LiteJobConfiguration.newBuilder(new ScriptJobConfiguration(
                JobCoreConfiguration.newBuilder(jobName, cron, shardingTotalCount)
                        .shardingItemParameters(shardingItemParameters).build()
                // 此处配置文件路径或者执行命令
                , scriptCommandLine)
        ).overwrite(true).build();
    }

    // 动态添加任务
/*    public void addSimpleJobScheduler(final Class<? extends SimpleJob> jobClass,
                                       final String cron,
                                       final int shardingTotalCount,
                                       final String shardingItemParameters){
        JobCoreConfiguration coreConfig = JobCoreConfiguration
                .newBuilder(jobClass.getName(), cron, shardingTotalCount)
                .shardingItemParameters(shardingItemParameters)
                .jobParameter("name")
                .build();
        SimpleJobConfiguration simpleJobConfig = new SimpleJobConfiguration(coreConfig, jobClass.getCanonicalName());
        JobScheduler jobScheduler = new JobScheduler(regCenter, LiteJobConfiguration.newBuilder(simpleJobConfig).build());
        jobScheduler.init();
    }*/

}

```



### 4.6  分片策略

####  4.6.1 分片项和分片参数

任务分片, 是为了实现把一个任务拆分成多个子任务, 在不同的`ejob`实例上运行, 假如说100W 条数据, 在配置文件中指定分为10个子任务(分片项),这10个子任务再按照一定的规则分配到5个实际运行的服务器上执行, 除了直接用分片项`ShardingItem`  获取分片任务之外, 还可以用`item`对应的`parameter` 获取任务. 

```java
JobCoreConfiguration coreConfig = JobCoreConfiguration.newBuilder("MySimpleJob", "0/2 * * * * ?", 4).shardingItemParameters("0=RDP, 1=CORE, 2=SIMS, 3=ECIF").build();
```

`springboot` 工程, 在`application.properties` 中定义. 

定义几个分片项, 一个任务就会有几个线程去运行他. 

注意: 分片个数和分片参数要一一对应,通常把分片项设置的比`Elastic-Job` 服务器个数要大一些, 比如3台服务器,分成9片, 这样如果有服务器宕机,分片还可以相对均匀. 

#### 4.6.2 分片验证

为避免运行的任务太多,看不清楚运行结果, 可以注释在`Elastic-Job` 中注释`DatFlowJob`和`ScriptJob`,`SimpleJob`的分片项改为2. 

然后运行主启动类

多实例运行(单机)

1. 多运行一个点, 任务不会重跑(两个节点各获得一个分片项)
2. 关闭一个节点, 任务不会漏跑



#### 4.7.3 分片策略



http://elasticjob.io/docs/elastic-job-lite/02-guide/job-sharding-strategy/

分片项如何分配到服务器? 这个跟分片策略有关. 

| 策略类                                  | 描述                                                | 具体规则                                                     |
| --------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------ |
| `AverageAllocationJobShardingStrategy`  | 基于平均分配算法的分片策略, 也是默认的分片策略      | 如果分片不能整除, 则不能整除的多余分片将依次追加到序号小的服务器, 如:<br />1. 如果有3台服务器,分成9片, 则每台服务器分到的分片是`1=[0,1,2], 2=[3,4,5], 3=[6,7,8]`<br />2. 如果有三台服务器, 分成8片, 则每台服务器分到的分片是: `1=[0,1,6], 2=[2,3,7], 3=[4,5]`<br />3.如果有三台服务器,分成10片, 则每台服务器分到的分片是: `1=[0,1,2,9], 2=[3,4,5], 3=[6,7,8]` |
| `OdevitySortByNameJobShardingStrategy`  | 根据作业名的哈希值奇偶数决定IP 升将序算法的分片策略 | 1. 作业名是哈希值为奇数则IP升序<br />2. 作业名的哈希值为偶数则Ip降序<br />用于不同的作业平均分配负载至不同的服务器 |
| `RotateServerByNameJobShardingStrategy` | 根据作业名的哈希值对服务器列表进行轮转的分片策略    |                                                              |
| 自定义分片策略                          |                                                     | 实现`JobShardingStrategy` 接口并实现`sharding` 方法, 接口方法参数为作业服务器IP列表和分片策略选项,分片策略选项包括作业名称、分片总数以及分片序列号和个性化参数对照表, 可以根据需求定制化自己的分片策略. |

`AverageAllocationJobShardingStrategy`的缺点是: 一旦分片数小于作业服务器数,作业将永远分配至IP 地址靠前的服务器, 导致IP地址靠后的服务器空闲. 而`OdevitySortByNameJobShardingStrategy`  则可以根据作业名称重新分配服务器负载. 如: 

如果有 3 台服务器，分成 2 片，作业名称的哈希值为奇数，则每台服务器分到的分片是：1=[0], 2=[1], 3=[] 
       如果有 3 台服务器，分成 2 片，作业名称的哈希值为偶数，则每台服务器分到的分片是：3=[0], 2=[1], 1=[]



在`Lite` 配置中指定分片策略:

```java
String jobShardingStrategyClass = AverageAllocationJobShardingStrategy.class.getCanonicalName();
LiteJobConfiguration simpleJobRootConfig = LiteJobConfiguration.newBuilder(simpleJobConfig).jobShardingStrategyClass(jobShardingStrategyClass).build()
```

#### 4.6.4 分片方案

获取到分片项`shardingItem` 之后, 怎么对数据进行分片呢? 

1. 对业务主键进行取模, 获取余数等于分片项的数据

    举例: 获取到的`shardingItem` 是0,1

   在SQL 中加入过滤条件: `where mod(id, 4) in (1, 2)`

   这种方式的缺点: 会导致索引失效,查询数据时会全表扫描 . 

   解决方案: 在查询条件中在增加一个索引条件进行过滤.

2. 在表中增加一个字段, 根据分片数生成一个`mod`的值. 取模的基数要大于机器数,否则在增加机器后, 会导致机器空闲. 假如取模基数是2,而服务器有5台, 那么有三台服务器是永远空闲的.而取模基数是10, 生成10个`shardingItem`，可以分配到5台服务器. 当然,取模基数也可以调整. 

3. 如果从业务层面,可以用`ShardingParamter` 进行分片. 

    假如`0=RDP, 1=CORE, 2=SIMS, 3=ECIF`

   `List = SELECT * FROM user WHERE status = 0 AND SYSTEM_ID = 'RDP' limit 0, 100。`

在SpringBoot中`Elastic-Job` 要配置的东西太多了,有没有更加简单的添加任务的方法呢? 比如在类上添加一个注解, 这个时候我们就要用到`starter`

### 4.7 `Elastic-Job` `starter` 

git上有一个现成的实现

https://github.com/TFdream/elasticjob-spring-boot-starter



##  5. `Elastic-Job`的原理

### 5.1 启动

我们从

```java
 new JobScheduler(registryCenter, simpleJobRootConfig).init();
```

开始分析,首先看`init()` 方法

```java
  /**
     * 初始化作业.
     */
    public void init() {
      
        LiteJobConfiguration liteJobConfigFromRegCenter = schedulerFacade.updateJobConfiguration(liteJobConfig);
      // 设置分片数
      JobRegistry.getInstance().setCurrentShardingTotalCount(liteJobConfigFromRegCenter.getJobName(), liteJobConfigFromRegCenter.getTypeConfig().getCoreConfig().getShardingTotalCount());
      // 构建任务, 创建调度器
        JobScheduleController jobScheduleController = new JobScheduleController(
                createScheduler(), createJobDetail(liteJobConfigFromRegCenter.getTypeConfig().getJobClass()), liteJobConfigFromRegCenter.getJobName());
        // 去ZK上注册任务
        JobRegistry.getInstance().registerJob(liteJobConfigFromRegCenter.getJobName(), jobScheduleController, regCenter);
        // 添加任务信息,并进行节点选举
        schedulerFacade.registerStartUpInfo(!liteJobConfigFromRegCenter.isDisabled());
       // 启动调度器
       jobScheduleController.scheduleJob(liteJobConfigFromRegCenter.getTypeConfig().getCoreConfig().getCron());
    }
```

接下来看`com.dangdang.ddframe.job.lite.internal.schedule.SchedulerFacade#registerStartUpInfo`方法

```java
    /**
     * 注册作业启动信息.
     * 
     * @param enabled 作业是否启用
     */
    public void registerStartUpInfo(final boolean enabled) {
       // 启动所有的监听器
        listenerManager.startAllListeners();
        // 节点选举
        leaderService.electLeader();
        // 服务信息持久化(写入ZK)
        serverService.persistOnline(enabled);
        // 实例信息持久化(写到ZK)
        instanceService.persistOnline();
        // 重新分片
        shardingService.setReshardingFlag();
        // 监控信息监听器
        monitorService.listen();
        // 自诊断恢复, 使本地节点与ZK数据一致
        if (!reconcileService.isRunning()) {
            reconcileService.startAsync();
        }
    }
```

监听器用于监听ZK节点信息变化

启动的时候进行主节点选举

```java
   /**
     * 选举主节点.
     */
    public void electLeader() {
        log.debug("Elect a new leader now.");
        jobNodeStorage.executeInLeader(LeaderNode.LATCH, new LeaderElectionExecutionCallback());
        log.debug("Leader election completed.");
    }
```

`Latch`是 一个分布式锁,选举成功后在`instance` 写入服务器信息

![image-20200412224842087](http://files.luyanan.com//img/20200412224843.png)

服务信息持久化(写到ZK Servers 节点)

`serverService.persistOnline(enabled)`

以下是单机运行多个实例:

![image-20200412224948880](http://files.luyanan.com//img/20200412224950.png)

运行了两个实例

![image-20200412225005046](http://files.luyanan.com//img/20200412225005.png)



### 5.2 任务执行和分片原理

关注两个问题

1. `LiteJob`   是怎么被执行的?
2. 分片项是怎么分配给不同的服务实例的?

在创建`Job`的时候(`createJobDetail`), 创建的是一个实现了`quartz`的`job`接口的`LiteJob`类, `LiteJob`类也实现了`Quartz`的`Job` 接口 . 

在`LiteJOb`的`execute`方法中获取对应类型的执行器, 调用`execute()`

```java
   public static AbstractElasticJobExecutor getJobExecutor(ElasticJob elasticJob, JobFacade jobFacade) {
        if (null == elasticJob) {
            return new ScriptJobExecutor(jobFacade);
        } else if (elasticJob instanceof SimpleJob) {
            return new SimpleJobExecutor((SimpleJob)elasticJob, jobFacade);
        } else if (elasticJob instanceof DataflowJob) {
            return new DataflowJobExecutor((DataflowJob)elasticJob, jobFacade);
        } else {
            throw new JobConfigurationException("Cannot support job type '%s'", new Object[]{elasticJob.getClass().getCanonicalName()});
        }
    }
```

`Elastic-Job` 提供管理任务执行器的抽象类`AbstractElasticJobExecutor`, 核心动作在`execute()` 方法中执行. 

```java
public final void execute() {

```

调用了另一个`execute()` 方法, 122行

```java
 execute(shardingContexts, JobExecutionEvent.ExecutionSource.NORMAL_TRIGGER);
```

```java
    private void execute(final ShardingContexts shardingContexts, final JobExecutionEvent.ExecutionSource executionSource) {
        if (shardingContexts.getShardingItemParameters().isEmpty()) {
            if (shardingContexts.isAllowSendJobEvent()) {
                jobFacade.postJobStatusTraceEvent(shardingContexts.getTaskId(), State.TASK_FINISHED, String.format("Sharding item for job '%s' is empty.", jobName));
            }
            return;
        }
        jobFacade.registerJobBegin(shardingContexts);
        String taskId = shardingContexts.getTaskId();
        if (shardingContexts.isAllowSendJobEvent()) {
            jobFacade.postJobStatusTraceEvent(taskId, State.TASK_RUNNING, "");
        }
        try {
            process(shardingContexts, executionSource);
        } finally {
            // TODO 考虑增加作业失败的状态，并且考虑如何处理作业失败的整体回路
            jobFacade.registerJobCompleted(shardingContexts);
            if (itemErrorMessages.isEmpty()) {
                if (shardingContexts.isAllowSendJobEvent()) {
                    jobFacade.postJobStatusTraceEvent(taskId, State.TASK_FINISHED, "");
                }
            } else {
                if (shardingContexts.isAllowSendJobEvent()) {
                    jobFacade.postJobStatusTraceEvent(taskId, State.TASK_ERROR, itemErrorMessages.toString());
                }
            }
        }
    }
```



在这个`execute`方法中又调用了`com.dangdang.ddframe.job.executor.AbstractElasticJobExecutor#process(com.dangdang.ddframe.job.executor.ShardingContexts, com.dangdang.ddframe.job.event.type.JobExecutionEvent.ExecutionSource)`方法

```java

    private void process(final ShardingContexts shardingContexts, final JobExecutionEvent.ExecutionSource executionSource) {
        Collection<Integer> items = shardingContexts.getShardingItemParameters().keySet();
        // 当只有一个分片的时候, 直接执行
        if (1 == items.size()) {
            int item = shardingContexts.getShardingItemParameters().keySet().iterator().next();
            JobExecutionEvent jobExecutionEvent =  new JobExecutionEvent(shardingContexts.getTaskId(), jobName, executionSource, item);
            process(shardingContexts, item, jobExecutionEvent);
            return;
        }
        final CountDownLatch latch = new CountDownLatch(items.size());
        // 本节点遍历执行相应的分片信息
        for (final int each : items) {
            final JobExecutionEvent jobExecutionEvent = new JobExecutionEvent(shardingContexts.getTaskId(), jobName, executionSource, each);
            if (executorService.isShutdown()) {
                return;
            }
            executorService.submit(new Runnable() {
                
                @Override
                public void run() {
                    try {
                        process(shardingContexts, each, jobExecutionEvent);
                    } finally {
                        latch.countDown();
                    }
                }
            });
        }
        try {
            // 等待所有的分片项任务执行完毕
            latch.await();
        } catch (final InterruptedException ex) {
            Thread.currentThread().interrupt();
        }
    }
    
```



里面又调用了一个`com.dangdang.ddframe.job.executor.AbstractElasticJobExecutor#process(com.dangdang.ddframe.job.executor.ShardingContexts, int, com.dangdang.ddframe.job.event.type.JobExecutionEvent)` 方法

```java
private void process(final ShardingContexts shardingContexts, final int item, final JobExecutionEvent startEvent) {
        if (shardingContexts.isAllowSendJobEvent()) {
            jobFacade.postJobExecutionEvent(startEvent);
        }
        log.trace("Job '{}' executing, item is: '{}'.", jobName, item);
        JobExecutionEvent completeEvent;
        try {
            process(new ShardingContext(shardingContexts, item));
            completeEvent = startEvent.executionSuccess();
            log.trace("Job '{}' executed, item is: '{}'.", jobName, item);
            if (shardingContexts.isAllowSendJobEvent()) {
                jobFacade.postJobExecutionEvent(completeEvent);
            }
            // CHECKSTYLE:OFF
        } catch (final Throwable cause) {
            // CHECKSTYLE:ON
            completeEvent = startEvent.executionFailure(cause);
            jobFacade.postJobExecutionEvent(completeEvent);
            itemErrorMessages.put(item, ExceptionUtil.transform(cause));
            jobExceptionHandler.handleException(jobName, cause);
        }
    }
    
```

然后调用了一个抽象的`com.dangdang.ddframe.job.executor.AbstractElasticJobExecutor#process(com.dangdang.ddframe.job.api.ShardingContext)` 方法

```java
 protected abstract void process(ShardingContext shardingContext);
```

交给具体的实现类（`SimpleJobExecutor`、`DataflowJobExecutor`、 `ScriptJobExecutor`） 去处理



最终调用到任务类

```java
   @Override
    protected void process(final ShardingContext shardingContext) {
        simpleJob.execute(shardingContext);
    }
```

```java
public class MySimpleJob implements SimpleJob {


    @Override
    public void execute(ShardingContext shardingContext) {
        System.out.println(String.format("分片项 ShardingItem: %s | 运行时间: %s | 线程ID: %s | 分片参数: %s ",
                shardingContext.getShardingItem(), new SimpleDateFormat("HH:mm:ss").format(new Date()),
                Thread.currentThread().getId(), shardingContext.getShardingParameter()));
    }
}

```



### 5.3  失效转移

所谓的失效转移,就是在执行任务的过程中发生了异常,这个分片任务可以在其他节点再次执行,

```java
 // 定义作业核心配置
        JobCoreConfiguration coreConfiguration = JobCoreConfiguration.newBuilder(MySimpleJob.class.getSimpleName(),
                "0/2 * * * * ?", 4).shardingItemParameters("0=RDP, 1=CORE, 2=SIMS, 3=ECIF").failover(true).build();
```

`FailoverListenerManager` 监听的是zl的`instance`节点删除事件,如果任务配置了`fairover`等于`true`,其中某个`instance` 与zk 失去联系或者被删除,并且失效的节点又不是本身, 就会触发是失效转移逻辑. 

`Job`的失效转移监听来源于`FailoverListenerManager`的内部类,`JobCrashedJobListener` 的 `dataChanged` 方法

当节点任务失效后会调用`JobCrashedJobListener` 监听器,此监听器会根据实例id 获取所有的分片, 然后调用`FailoverService` 的 `setCrashedFailoverFlag` 方法, 将每个分片id 写到`到/jobName/leader/failover/items`, 假如原来的实例负责1、2分片项, 那`items` 节点就会写入1、2, 代表这两个分片项需要失效转移. 

```java
 @Override
        protected void dataChanged(final String path, final Type eventType, final String data) {
            if (isFailoverEnabled() && Type.NODE_REMOVED == eventType && instanceNode.isInstancePath(path)) {
                String jobInstanceId = path.substring(instanceNode.getInstanceFullPath().length() + 1);
                if (jobInstanceId.equals(JobRegistry.getInstance().getJobInstance(jobName).getJobInstanceId())) {
                    return;
                }
                List<Integer> failoverItems = failoverService.getFailoverItems(jobInstanceId);
                if (!failoverItems.isEmpty()) {
                    for (int each : failoverItems) {
                        // 设置失效的分片项标记
                        failoverService.setCrashedFailoverFlag(each);
                        failoverService.failoverIfNecessary();
                    }
                } else {
                    for (int each : shardingService.getShardingItems(jobInstanceId)) {
                        failoverService.setCrashedFailoverFlag(each);
                        failoverService.failoverIfNecessary();
                    }
                }
            }
        }
```

然后接下来调用`FailoverService` 的 `failoverIfNessary`方法, 首先判断是否需要失效转移,如果可以需要则只需要作业失败转移. 

```java
    /**
     * 如果需要失效转移, 则执行作业失效转移.
     */
    public void failoverIfNecessary() {
        if (needFailover()) {
            jobNodeStorage.executeInLeader(FailoverNode.LATCH, new FailoverLeaderExecutionCallback());
        }
    }
```

条件一： `${JOB_NAME}/leader/failover/items/${ITEM_ID}` 有失效转移的作业分片

条件二: 当前作业不再运行中

```java
 private boolean needFailover() {
        return jobNodeStorage.isJobNodeExisted(FailoverNode.ITEMS_ROOT) && !jobNodeStorage.getJobNodeChildrenKeys(FailoverNode.ITEMS_ROOT).isEmpty()
                && !JobRegistry.getInstance().isJobRunning(jobName);
    }
```

在主节点执行操作

```java
 /**
     * 在主节点执行操作.
     * 
     * @param latchNode 分布式锁使用的作业节点名称
     * @param callback 执行操作的回调
     */
    public void executeInLeader(final String latchNode, final LeaderExecutionCallback callback) {
        try (LeaderLatch latch = new LeaderLatch(getClient(), jobNodePath.getFullPath(latchNode))) {
            latch.start();
            latch.await();
            callback.execute();
        //CHECKSTYLE:OFF
        } catch (final Exception ex) {
        //CHECKSTYLE:ON
            handleException(ex);
        }
    }
```

1. 再次判断是否需要失效转移
2. 从注册中心获取一个`${JOB_NAME}/leader/failover/items/${ITEM_ID}` 作业分片项
3. 在注册中心节点`${JOB_NAME}/sharding/${ITEM_ID}/failover` 注册作业分片项为当前作业节点
4. 然后移除任务转移分片项
5. 最后调用执行, 提交任务

```java
    class FailoverLeaderExecutionCallback implements LeaderExecutionCallback {
        
        @Override
        public void execute() {
            // 判断是否需要失效转悠
            if (JobRegistry.getInstance().isShutdown(jobName) || !needFailover()) {
                return;
            }
            // 从${JOB_NAME}/leader/failover/items/${ITEM_ID}获得一个分片项
            int crashedItem = Integer.parseInt(jobNodeStorage.getJobNodeChildrenKeys(FailoverNode.ITEMS_ROOT).get(0));
            log.debug("Failover job '{}' begin, crashed item '{}'", jobName, crashedItem);
            // 在注册中心节点`${JOB_NAME}/sharding/${ITEM_ID}/failover`注册作业分片项为当前作业节点
            jobNodeStorage.fillEphemeralJobNode(FailoverNode.getExecutionFailoverNode(crashedItem), JobRegistry.getInstance().getJobInstance(jobName).getJobInstanceId());
            // 移除任务转移分片项
            jobNodeStorage.removeJobNodeIfExisted(FailoverNode.getItemsNode(crashedItem));
            // TODO 不应使用triggerJob, 而是使用executor统一调度
            JobScheduleController jobScheduleController = JobRegistry.getInstance().getJobScheduleController(jobName);
            if (null != jobScheduleController) {
                //  提交任务
                jobScheduleController.triggerJob();
            }
        }
    }
```

这里仅仅是触发作业，而不是立即执行. 