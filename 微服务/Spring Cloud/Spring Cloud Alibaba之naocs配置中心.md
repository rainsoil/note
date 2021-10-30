# Spring Cloud Alibaba之naocs配置中心



## 1.  介绍

> 动态配置服务可以让您以中心化、外部化和动态化的方式管理所有环境的应用配置和服务配置。
>
> 动态配置消除了配置变更时重新部署应用和服务的需要，让配置管理变得更加高效和敏捷。
>
> 配置中心化管理让实现无状态服务变得更简单，让服务按需弹性扩展变得更容易。
>
> Nacos 提供了一个简洁易用的UI ([控制台样例 Demo](http://console.nacos.io/nacos/index.html)) 帮助您管理所有的服务和应用的配置。Nacos 还提供包括配置版本跟踪、金丝雀发布、一键回滚配置以及客户端配置更新状态跟踪在内的一系列开箱即用的配置管理特性，帮助您更安全地在生产环境中管理配置变更和降低配置变更带来的风险。

## 2.  项目实战

### 2.1  nacos  启动

这里就不多说了



### 2.2  依赖添加

```xml
  <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>

        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        </dependency>
```



### 2.3  添加`bootstrap` 配置文件

此文件会优先于`application.yml` 启动

加入配置中心的配置

```yaml
spring:
  cloud:
    nacos:
      config:
        server-addr: localhost:8848
  application:
    name: nacos-config-client

```

`application.yml` 中配置其他的内容

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
server:
  port: 12000

```



###  2.4 添加主启动类

```java
@EnableDiscoveryClient
@SpringBootApplication
public class NacosConfigClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(NacosConfigClientApplication.class, args);
    }

}

```





### 2.5 配置测试

#### 传统方式中,

为了详细说明配置中心的使用,我们先来使用传统的方式

在`application.yml` 配置文件中加入

```yaml
info:
  project:
    version: 1.0
    author: luyanan

```

编写`controller` 查出我们设置的配置

```java
@RestController
public class InfoController {

    @Value("${info.project.version}")
    public String projectVersion;

    @Value("${info.project.author}")
    public String projectAuthor;

    @GetMapping("info")
    public Map info() {

        HashMap info = new HashMap(2);

        info.put("version", projectVersion);
        info.put("author", projectAuthor);
        return info;
    }
}

```



访问接口可以看到我们设置的内容

```json
{
    "version": "1.0",
    "author": "luyanan"
}
```

我们可以通过这种方式来存放我们的配置,但是这样做存在一个问题,如果需要频繁的修改`application.yml` 文件的话,就需要频繁的打包上线,很是不方便. 接下来我们可以用`nacos` 来解决这个问题. 



#### `nacos`作为配置中心

我们查看启动日志

```java
2020-08-11 13:58:25.947  INFO 14876 --- [           main] b.c.PropertySourceBootstrapConfiguration : Located property source: [BootstrapPropertySource {name='bootstrapProperties-nacos-config-client.properties,DEFAULT_GROUP'}, BootstrapPropertySource {name='bootstrapProperties-nacos-config-client,DEFAULT_GROUP'}]

```

我们可以发现,在程序启动的时候,程序会去`nacos` 去查找名称为`nacos-config-client.properties`的配置文件, 所以我们可以来到`nacos`的控制台

1. 选择配置列表,点击最右边的加号按钮可以添加配置

![image-20200811142441839](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83.assets/image-20200811142441839.png)

`Data ID` : `nacos-config-client.properties`

文件的命名规则为：${spring.application.name}-${spring.profiles.active}.${spring.cloud.nacos.config.file-extension}

${spring.application.name}：为微服务名

${spring.profiles.active}：指明是哪种环境下的配置，如dev、test或info

${spring.cloud.nacos.config.file-extension}：配置文件的扩展名，可以为properties、yml等



点击发布. 

2. 查看配置

   

![image-20200811141306361](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83.assets/image-20200811141306361.png)

可以看到我们刚才添加的配置

3. 修改`infoController`,添加`@RefreshScope` 注解

   ```java
   @RefreshScope
   @RestController
   public class InfoController {
   
       @Value("${info.project.version}")
       public String projectVersion;
   
       @Value("${info.project.author}")
       public String projectAuthor;
   
       @GetMapping("info")
       public Map info() {
   
           HashMap info = new HashMap(2);
   
           info.put("version", projectVersion);
           info.put("author", projectAuthor);
           return info;
       }
   }
   ```

4. 重启测试

   重启成功后,再次访问`http://localhost:12000/info` 接口,就可以看到这个时候返回的就不是我们在`application.yml` 配置的内容,而是我们在`nacos` 中配置的内容了. 并且在指明了相同的配置信息的时候,配置中心的配置优先于本地的配置. 

   

   ```java
   {
       "version": "2.0",
       "author": "luyanan"
   }
   ```

   我们再次在配置中心中修改配置 `info.project.author=luyanan` 为`info.project.author=luyanan111111`,这个时候不重启应用,直接刷新接口. 

   ```java
   {
   "version": "2.0",
   "author": "luyanan111111"
   }
   ```

   我们看到,接口返回的配置就是我们在`nacos` 中配置的内容了, 这样就实现了不需要重启就可以动态的修改配置了. 

   



## 3. `Nacos` 支持的三种配置加载方案

`Nacos` 支持`NameSpace + group + dataID`的配置解决方案

详情见: <https://github.com/alibaba/spring-cloud-alibaba/blob/master/spring-cloud-alibaba-docs/src/main/asciidoc-zh/nacos-config.adoc>



### 3.1 `NameSpace`  方案

通过命令空间来实现环境的区分

下面是配置实例: 

1. 创建`命名空间`

   `命名空间->创建命名空间`

   

![image-20200811144644870](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83.assets/image-20200811144644870.png)

创建两个命名空间,`dev` 和`test`

2.  回到配置列表,可以看到我们创建的命名空间

   ![image-20200811144757793](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83.assets/image-20200811144757793.png)

3.  下面我们在`dev`的命令空间下创建`nacos-config-client.properties  `的配置文件

   ![image-20200811150101616](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83.assets/image-20200811150101616.png)

   这个时候,直接访问`http://localhost:12000/info` 的时候会发现配置并没有改变,原因是因为`nacos` 会默认使用`public` 命令空间下配置的规则, 

4. 程序指定命令空间

   具体要想使用我们自定义的哪个命名空间,就需要在`bootstrap.yml` 配置文件中指定哪个配置文件的`ID`,

   这个命名空间的ID来源于我们第一步创建的命名空间

   ![image-20200811150340763](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83.assets/image-20200811150340763.png)

   修改`bootstrap`的配置, 加入

   ```yml
   spring:
     cloud:
       nacos:
         config:
           ## 指定配置使用的命名空间ID
           namespace: d033967d-013d-4f01-9d23-8326105615ac
   ```

   然后重启,再次访问接口

   ```json
   {
       "version": "2.0",
       "author": "luyanan-dev"
   }
   ```

   就可以发现已经变成了我们设置的`dev`命名空间的配置文件, 

   但是这种命名空间的粒度还是不够细化,对此我们可以为每个项目创建一个命名空间

5. 为每个项目创建命名空间

   ![image-20200811150753752](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83.assets/image-20200811150753752.png)

6. 回到配置列表下,克隆`public` 的配置规则到`nacos-config-client` 命名空间下. 

   ![image-20200811150948445](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83.assets/image-20200811150948445.png)

![image-20200811151206393](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83.assets/image-20200811151206393.png)



7. 修改`bootstrap.yml`的命名空间配置为`nacos-config-client`的

   然后重启重新访问接口,发现这时候读取的就是`nacos-config-client` 命名空间下的配置文件了. 

### 3.2 `Data Id` 方案

通过指定`spring.profile.active`和配置文件的`DataId` , 来使得不同环境下读取不同的配置,读取配置的时候,默认使用的命名空间是`public`,默认分组是`default_group`下的`DataID`



### 3.3 `DataID` 方案

通过`Group` 实现环境区分

实例: 通过使用不同的组来读取不同的配置,还是以上面的`nacos-config-client` 为例

我们还是使用`nacos-config-client` 命名空间,先清空该命名空间下的所有配置

然后新建`Group` 为`tmp`的配置文件

![image-20200811153350391](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83.assets/image-20200811153350391.png)

然后修改项目的分组为`tmp`

```yaml
spring:
  cloud:
    nacos:
      config:
        ##  指定分组
        group: tmp
```

接下来重启项目,再次访问路径就可以看到`tmp` 分组下的配置了. 



### 3.4 同时加载多个配置集

当微服务的数量很庞大的时候,将所有的配置文件都写入到一个配置文件中,显然是不太合适的,为此我们可以将配置文件按照功能的不同拆分成不同的配置文件. 

如下面的配置文件:

```yaml
spring:        
  datasource:
    url: jdbc:mysql://localhost:3306/test
    username: root
    password: rootroot

mybatis-plus:
  global-config:
    db-config:
      id-type: auto
  mapper-locations: classpath:/mapper/**/*.xml

```

我们就可以将关于数据源的配置放入到写入到一个配置文件中, 

将框架相关的配置写入到另外一个配置文件中

将上述的配置配置交给`nacos` 来管理,

1. 创建`datasource.yml`  配置文件存储数据库相关的配置

   ```yaml
   spring:        
     datasource:
       url: jdbc:mysql://localhost:3306/test
       username: root
       password: rootroot
   
   ```

   在`nacos-config-client`  的命名空间下创建`datasource.yml` 配置文件

   ![image-20200811155054242](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83.assets/image-20200811155054242.png)

2. 将跟`mybatis` 相关的配置放入`mybatis.yml` 配置文件中

   同样,新建`mybatis.yml` 配置文件

   ```yaml
   mybatis-plus:
     global-config:
       db-config:
         id-type: auto
     mapper-locations: classpath:/mapper/**/*.xml
   ```

   ![image-20200811155234548](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83.assets/image-20200811155234548.png)

3. 修改项目的`bootstrap.yml` 配置文件,加载`datasource.yml` 和`mybatis.yml`  配置文件

   ```yaml
   spring:
     cloud:
       nacos:
         config:
           ## 指定配置使用的命名空间ID
           namespace: db8304ef-bf5d-4259-b5d8-c2d532d45e6b
           ##  指定分组
           group: tmp
           extension-configs:
             - data-id: datasource.yml
               group: dev
               refresh: true
             - data-id: mybatis.yml
               group: dev
               refresh: true
   ```





## 4. 小结

1. 微服务任何配置信息,任何配置文件都可以放在配置中心
2. 只需要在`bootstrap` 配置文件中说明加载配置中心的那些配置文件即可
3. `@Value` 和`ConfigurationProperties` 都可以用来获取配置中心的信息
4. 配置中心有的优先使用配置中心的,没有的则使用本地配置的. 



