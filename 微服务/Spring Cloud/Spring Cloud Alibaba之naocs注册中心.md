# Spring Cloud Alibaba之naocs注册中心

## 1. 确定版本

我们先要确定下载的`nacos`的版本号, 

`spring cloud alibaba`的官网表明了各个组件的版本依赖关系和与`Spring cloud`和`Spring Boot`的依赖关系

[https://github.com/alibaba/spring-cloud-alibaba/wiki/%E7%89%88%E6%9C%AC%E8%AF%B4%E6%98%8E#%E7%BB%84%E4%BB%B6%E7%89%88%E6%9C%AC%E5%85%B3%E7%B3%BB](https://github.com/alibaba/spring-cloud-alibaba/wiki/版本说明#组件版本关系)



## 组件版本关系

| Spring Cloud Alibaba Version                    | Sentinel Version | Nacos Version | RocketMQ Version | Dubbo Version | Seata Version |
| ----------------------------------------------- | ---------------- | ------------- | ---------------- | ------------- | ------------- |
| 2.2.1.RELEASE or 2.1.2.RELEASE or 2.0.2.RELEASE | 1.7.1            | 1.2.1         | 4.4.0            | 2.7.6         | 1.2.0         |
| 2.2.0.RELEASE                                   | 1.7.1            | 1.1.4         | 4.4.0            | 2.7.4.1       | 1.0.0         |
| 2.1.1.RELEASE or 2.0.1.RELEASE or 1.5.1.RELEASE | 1.7.0            | 1.1.4         | 4.4.0            | 2.7.3         | 0.9.0         |
| 2.1.0.RELEASE or 2.0.0.RELEASE or 1.5.0.RELEASE | 1.6.3            | 1.1.1         | 4.4.0            | 2.7.3         | 0.7.1         |

## 毕业版本依赖关系(推荐使用)

| Spring Cloud Version        | Spring Cloud Alibaba Version      | Spring Boot Version |
| --------------------------- | --------------------------------- | ------------------- |
| Spring Cloud Hoxton.SR3     | 2.2.1.RELEASE                     | 2.2.5.RELEASE       |
| Spring Cloud Hoxton.RELEASE | 2.2.0.RELEASE                     | 2.2.X.RELEASE       |
| Spring Cloud Greenwich      | 2.1.2.RELEASE                     | 2.1.X.RELEASE       |
| Spring Cloud Finchley       | 2.0.2.RELEASE                     | 2.0.X.RELEASE       |
| Spring Cloud Edgware        | 1.5.1.RELEASE(停止维护，建议升级) | 1.5.X.RELEASE       |



## 2. 下载

`https://github.com/alibaba/nacos/releases` 去挑选合适的版本下载,下载. 

如果只是为了使用的话, 我们可以下载编译好的版本

![image-20200811112833571](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83.assets/image-20200811112833571.png)



## 3. 编写客户端

### 3.1 `pom.xml` 编写

```xml

    <properties>
        <java.version>1.8</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <spring-boot.version>2.2.5.RELEASE</spring-boot.version>
        <spring-cloud-alibaba.version>2.2.1.RELEASE</spring-cloud-alibaba.version>
        <spring-cloud.version>Hoxton.SR3</spring-cloud.version>

        <!--        <spring-boot.version>2.2.1.RELEASE</spring-boot.version>-->
        <!--        <spring-cloud-alibaba.version>2.2.0.RELEASE</spring-cloud-alibaba.version>-->

    </properties>
   <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>


        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>

    </dependencyManagement>
```



###  3.2 编写启动类

```java
@EnableDiscoveryClient
@SpringBootApplication
public class SpringCloudNacosDiscoveryClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudNacosDiscoveryClientApplication.class, args);
    }

}

```



### 3.3 编写配置文件 `application.yml`

```yaml
spring:
  application:
    name: spring-cloud-nacos-discovery-client



  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
server:
  port: 12000

```



### 3.4  启动项目

启动项目之前,确保之前安装的`nacos` 已经启动

等项目启动成功后,我们访问`http://localhost:8848/nacos` nacos 的后台, 可以看到

![image-20200811114603305](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83.assets/image-20200811114603305.png)

我们可以看到, `spring-cloud-nacos-discovery-client`  这个项目已经注册到了`nacos` 上面了. 



`nacos` 中可以通过命名空间来区分不同的环境

![image-20200811121858155](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83.assets/image-20200811121858155.png)

新建一个`dev`的命令空间,点击确定之后,我们会发现列表会多出来一条, 还有一个命令空间ID,

![image-20200811121942740](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83.assets/image-20200811121942740.png)

这个ID就是我们所需要的, 我们复制这个id, 到项目的`application.yml`配置文件,修改配置文件为

```yaml

  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        ## 指定命令空间
        namespace: d033967d-013d-4f01-9d23-8326105615ac
```



然后重启项目, 再次查看`nacos` 的控制台

![image-20200811122215601](Spring%20Cloud%20Alibaba%E4%B9%8Bnaocs%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83.assets/image-20200811122215601.png)

我们发现在原本的`public` 旁边多了一个`dev`的`tab`,点击`dev` 我们可以发现我们注册的服务. 

