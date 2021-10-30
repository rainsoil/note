#  ![image-20200801092800346](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200801092800346.png)![image-20200801092745464](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200801092745464.png)Spring Cloud Security Oauth2.0实现分布式系统认证

## 1. 环境介绍

`Spring-Security-Oauth2.0` 是对`Oauth2.0` 的一种实现,并且跟`Spring Security`相辅相成,与`Spring Cloud` 体系的集成也非常便利,接下来,我们需要对它进行学习,最终使用它来实现我们设计的分布式认证授权解决方案

`Oauth2.0` 的服务提供方涵盖两个服务,即授权服务(`Authorization Server`,也叫认证服务)和资源服务(`resource Server`),使用 `Spring Security Oauth2.0`的时候, 你可以选择把他们放在同一个应用程序中去实现,也可以选择建立使用同一个授权服务的多个资源服务.

**授权服务(`Authorization Server`)** 应包含对接入端以及登录用户的合法性进行验证并颁发`token` 等功能,对令牌的请求端点由`Spring MVC`控制器进行实现,下面是一个配置一个认证服务必须实现的`endpoints`: 

-  `AuthorizationEndpoint`:' 服务于认证请求, 默认`URL`:`/oauth/authorize `

- `TokenEndpoint` 服务于访问令牌的请求,默认`URL`: `/oauth/token`

  资源服务(`Resource Server`),应包含对资源的保护功能,对非法请求进行拦截,对请求`token` 进行解析鉴权等,下面的过滤器用于实现`Oauth2.0` 资源服务

- `OAuth2AuthenticationProcessingFilter`:用来对请求给出的身份令牌解析鉴权.



本教程分别创建`UAA`认证中心和`order` 订单资源服务

![image-20200801092656494](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200801092656494.png)

认证流程如下:

1. 客户端请求`UAA` 授权服务进行认证
2. 认证通过后由`UAA` 颁发令牌
3. 客户端携带令牌`Token`请求资源服务
4. 资源服务校验令牌的合法性,合法则返回资源信息

##  2. 代码演示

### 2.1 创建父工程

创建`maven` 父工程,依赖如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.spring.security</groupId>
    <artifactId>spring-security-oauth2-demo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <packaging>pom</packaging>
    <properties>
        <project.build.sourceEncoding>UTF‐8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF‐8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>

    </properties>


    <dependencyManagement>
        <dependencies>


            <dependency>
                <groupId>org.springframework.security.oauth.boot</groupId>
                <artifactId>spring-security-oauth2-autoconfigure</artifactId>
                <version>2.1.3.RELEASE</version>
            </dependency>
            <dependency>
                <groupId>org.springframework.security</groupId>
                <artifactId>spring-security-jwt</artifactId>
                <version>1.0.10.RELEASE</version>
            </dependency>
            <dependency>
                <groupId>mysql</groupId>
                <artifactId>mysql-connector-java</artifactId>
                <version>5.1.46</version>
            </dependency>
            <dependency>
                <groupId>com.alibaba</groupId>
                <artifactId>fastjson</artifactId>
                <version>1.2.70</version>
            </dependency>
            <dependency>
                <groupId>cn.hutool</groupId>
                <artifactId>hutool-all</artifactId>
                <version>5.3.8</version>
            </dependency>
            <dependency>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>1.18.12</version>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Greenwich.RELEASE</version>
                <scope>import</scope>
                <type>pom</type>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>2.1.3.RELEASE</version>
                <scope>import</scope>
                <type>pom</type>
            </dependency>
        </dependencies>
        
    </dependencyManagement>

    <!--子模块 -->

    <modules>
        <module>uaa</module>
        <module>order</module>

    </modules>


    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>


    <!--aliyun 私服配置 -->
    <repositories>
        <repository>
            <id>public</id>
            <name>aliyun nexus</name>
            <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
            <releases>
            </releases>
        </repository>
    </repositories>

    <pluginRepositories>
        <pluginRepository>
            <id>public</id>
            <name>aliyun nexus</name>
            <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
            <releases>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </pluginRepository>
    </pluginRepositories>
</project>

```

里面主要定义了`Spring Boot`的版本和`Spring Cloud`的版本



### 2.2 创建`UAA` 授权服务工程

1. `pom` 依赖

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>
   
   
       <parent>
           <groupId>com.spring.security</groupId>
           <artifactId>spring-security-oauth2-demo</artifactId>
           <version>1.0-SNAPSHOT</version>
       </parent>
       <groupId>com.spring.security.uaa</groupId>
       <artifactId>uaa</artifactId>
       <version>0.0.1-SNAPSHOT</version>
       <name>uaa</name>
       <description>授权认证服务</description>
   
       <properties>
           <java.version>1.8</java.version>
           <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
           <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
   
       </properties>
   
       <dependencies>
   
           <!--wen组件 -->
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </dependency>
   
           <!--端点监控 -->
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-actuator</artifactId>
           </dependency>
   
           <!--服务调用 -->
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-openfeign</artifactId>
           </dependency>
           <!--负载均衡 -->
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
           </dependency>
   
   
           <!--  熔断-->
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
           </dependency>
   
           <!--注册中心 -->
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-netflix-eureka-client</artifactId>
           </dependency>
   
   
           <!--认证授权相关 -->
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-oauth2</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-security</artifactId>
           </dependency>
   
           <dependency>
               <groupId>org.springframework.security</groupId>
               <artifactId>spring-security-jwt</artifactId>
           </dependency>
   
   
           <!-- 认证授权end-->
   
           <dependency>
               <groupId>mysql</groupId>
               <artifactId>mysql-connector-java</artifactId>
           </dependency>
   
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-jdbc</artifactId>
           </dependency>
   
   
           <dependency>
               <groupId>org.projectlombok</groupId>
               <artifactId>lombok</artifactId>
           </dependency>
   
           <dependency>
               <groupId>com.alibaba</groupId>
               <artifactId>fastjson</artifactId>
           </dependency>
   
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-test</artifactId>
               <scope>test</scope>
               <exclusions>
                   <exclusion>
                       <groupId>org.junit.vintage</groupId>
                       <artifactId>junit-vintage-engine</artifactId>
                   </exclusion>
               </exclusions>
           </dependency>
       </dependencies>
   
   </project>
   
   ```

2. 编写启动类`UaaApplication`

   ```java
   
   @EnableHystrix
   @EnableFeignClients
   @EnableDiscoveryClient
   @SpringBootApplication
   public class UaaApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(UaaApplication.class, args);
       }
   
   }
   
   ```

3. 编写配置文件`application.yml`

   ```yaml
   spring:
     application:
       name: uaa
     main:
       allow-bean-definition-overriding: true
     http:
       encoding:
         charset: UTF-8
         enabled: true
         force: true
     datasource:
       url: jdbc:mysql://localhost:3306/spring-security-oauth2-demo
       username: root
       password: rootroot
       driver-class-name: com.mysql.jdbc.Driver
     mvc:
       throw-exception-if-no-handler-found: true
     resources:
       add-mappings: false
   
   
   server:
     port: 8000
     tomcat:
       remote-ip-header: x‐forwarded‐for
       protocol-header: x‐forwarded‐proto
     use-forward-headers: true
     servlet:
       context-path: /uaa
   
   
   
   
   
   logging:
     level:
       root: info
   
   feign:
     hystrix:
       enabled: true
     compression:
       request:
         enabled: true
         mime-types:
           - text/html
           - appliication/xml
           - application/json
         min-request-size: 2048
       response:
         enabled: true
   
   ```

4. 结构预览

   ![image-20200730134311459](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200730134311459.png)





### 2.3  编写`Order` 资源服务项目

本工程为`Order` 订单服务工程,访问本工程的资源需要认证通过

本工程的目的主要是为了测试认证授权功能,所以不会涉及订单的相关业务逻辑. 

1. `pom.xml`

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>
   
   
       <parent>
           <groupId>com.spring.security</groupId>
           <artifactId>spring-security-oauth2-demo</artifactId>
           <version>1.0-SNAPSHOT</version>
       </parent>
       <groupId>com.spring.security.order</groupId>
       <artifactId>order</artifactId>
       <version>0.0.1-SNAPSHOT</version>
       <name>order</name>
       <description>订单模块</description>
   
   
       <dependencies>
   
           <!-- 认证授权相关-->
   
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-security</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-oauth2</artifactId>
           </dependency>
   
   
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-jdbc</artifactId>
           </dependency>
           <dependency>
               <groupId>mysql</groupId>
               <artifactId>mysql-connector-java</artifactId>
           </dependency>
           <dependency>
               <groupId>com.alibaba</groupId>
               <artifactId>fastjson</artifactId>
           </dependency>
           <dependency>
               <groupId>org.projectlombok</groupId>
               <artifactId>lombok</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-openfeign</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </dependency>
   
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-actuator</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-netflix-eureka-client</artifactId>
           </dependency>
   
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter</artifactId>
           </dependency>
   
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-test</artifactId>
               <scope>test</scope>
               <exclusions>
                   <exclusion>
                       <groupId>org.junit.vintage</groupId>
                       <artifactId>junit-vintage-engine</artifactId>
                   </exclusion>
               </exclusions>
           </dependency>
       </dependencies>
   
   
       <build>
           <plugins>
               <plugin>
                   <groupId>org.apache.maven.plugins</groupId>
                   <artifactId>maven-compiler-plugin</artifactId>
                   <configuration>
                       <source>1.8</source>
                       <target>1.8</target>
                       <encoding>UTF-8</encoding>
                   </configuration>
               </plugin>
               <plugin>
                   <groupId>org.springframework.boot</groupId>
                   <artifactId>spring-boot-maven-plugin</artifactId>
               </plugin>
           </plugins>
       </build>
   
   </project>
   
   ```

2. `Order`的启动类

   ```java
   @EnableHystrix
   @EnableFeignClients
   @EnableDiscoveryClient
   @SpringBootApplication
   public class OrderApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(OrderApplication.class, args);
       }
   
   }
   ```

3. 配置文件`application.yml`

   ```yaml
   server:
     port: 8081
     tomcat:
       remote-ip-header: x‐forwarded‐for
       protocol-header: x‐forwarded‐proto
     use-forward-headers: true
     servlet:
       context-path: /order
   
   
   spring:
     application:
       name: order
     main:
       allow-bean-definition-overriding: true
     http:
       encoding:
         charset: UTF-8
         enabled: true
         force: true
     datasource:
       url: jdbc:mysql://localhost:3306/spring-security-oauth2-demo
       username: root
       password: rootroot
       driver-class-name: com.mysql.jdbc.Driver
     mvc:
       throw-exception-if-no-handler-found: true
     resources:
       add-mappings: false
   
   
   feign:
     hystrix:
       enabled: true
     compression:
       request:
         enabled: true
         mime-types:
           - text/html
           - appliication/xml
           - application/json
         min-request-size: 2048
       response:
         enabled: true
   
   
   ```



项目的基本框架现在已经搭建了

### 2.4 授权服务器设置(`uaa`)

####   2.4.1  ` EnableAuthorizationServer`

可以用` EnableAuthorizationServer`注解并继承`AuthorizationServerConfigurerAdapter` 来配置`Oauth2.0` 授权服务器.

在`config` 包下创建 `AuthorizationServer`

`AuthorizationServerConfigurerAdapter`  要求配置以下几个类,这几个类是由`Spring` 创建的独立的配置对象,他们会被`Spring`传入`AuthorizationServerConfigurerAdapter ` 中进行配置. 

```java
public class AuthorizationServerConfigurerAdapter implements AuthorizationServerConfigurer {
    public AuthorizationServerConfigurerAdapter() {
    }

    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
    }

    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
    }

    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
    }
}

```

- `ClientDetailsServiceConfigurer`:用来配置客户端详情服务(`ClientDetailsService`),客户端详情信息在这里进行初始化,可以通过客户端详细写死在这里或者通过数据库来存储调取详情信息. 
- `AuthorizationServerEndpointsConfigurer`: 用来配置令牌`token` 的访问端点和令牌服务(`token service`)
- `AuthorizationServerSecurityConfigurer`: 用来配置令牌端点的安全约束



#### 2.4.2 配置客户端相信信息

`ClientDetailsServiceConfigurer` 可以使用内存或者JDBC来实现客户端详细信息服务(`ClientDetailsService`)

`ClientDetailsService` 负责查找`ClientDetails`, 而`ClientDetails` 有几个重要的属性如下列表: 

- `clientId`(必须): 用来标识客户的`id`
- `secret`: (需要值得信任的客户端)客户端安全码,如果有的话,
- `scope` :用来限制客户端的访问范围, 如果为空(默认的话), 那么客户端将拥有全部的访问范围. 
- `authorizedGrantTypes`:此客户端可以使用的授权类型,默认为空
- `authorities` 此客户端可以使用的权限(基于`Spring Security  authorities`)

客户端详情(`client details`) 能够在应用程序运行的时候进行更新,可以通过访问底层的存储服务(例如将客户端详情存储在一个关系型数据库的表中,就可以使用`jdbcClientDetailsService`) 或者通过自己实现`ClientRegistrationService` 接口(同时实现`ClientDetailsServcice` 接口) 来进行管理. 

```java
 @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {

         clients.inMemory()// 使用in‐memory存储
                .withClient("c1")// client_id
                .secret(new BCryptPasswordEncoder().encode("secret"))
                .resourceIds("res1")
                .authorizedGrantTypes("authorization_code",
                        "password", "client_credentials", "implicit", "refresh_token")// 该client允许的授权类型 authorization_code,password,refresh_token,implicit,client_credentials
                .scopes("all")// 允许的授权范围
                .autoApprove(false)
//加上验证回调地址
                .redirectUris("http://www.baidu.com");
    }
```



#### 2.4.3 管理令牌

`AuthorizationServerTokenService`接口定义了一些操作使得你可以对令牌进行一些必要的管理,令牌可以被用来加载身份信息,里面包含了这个令牌的相关权限. 

自己可以创建`AuthorizationServerTokenService` 这个接口的实现,则需要继承`DefaultTokenServices` 这个类, 里面包含了一些有用的实现,可以使用它来修改令牌的格式和令牌的存储. 默认的,当它尝试创建一个令牌的时候,是使用随机值来进行填充的,除了持久化令牌是委托一个`tokenStore` 接口来实现以外,这个类几乎帮你做了所有的事情,并且`TokenStore` 这个接口有一个默认的实现`InMemoryTokenStore`, 基于内存的令牌保存实现. 除了使用这个类之外, 你还可以使用一些其他的预定义的实现,下面有几个版本,他们都实现了`TokenStore` 接口. 

- `InMemoryTokenStore`:这个版本都实现是被默认采用的,它可以完美的工作在单机服务器上(即访问并发量不大的情况下, 并且它在失败的时候不会进行备份), 大多数的项目都可以使用这个版本的实现来进行尝试, 一般可以在开发的时候使用它进行管理, 因为不会被保存在磁盘中, 所以更易于调试. 
- `JdbcTokenStore`: 这是一个基于`JDBC`的实现版本, 令牌会被保存在关系型数据库中,使用这个版本的实现的时候,你可以在不同的服务器之间共享信息,使用这个版本的时候注意把`spring-jdbc`这个依赖加入到项目中. 
- `JwtTokenStore`: 这个版本的全称是`JSON Web Token（JWT）`,它可以把令牌相关的数据进行编码(因此相对于后端服务来说, 它不需要进行存储,这是一个重大的优势),但是它有一个缺点,就是撤销一个已经授权的令牌将会非常困难,所以它通常用来处理一个生命周期很短的令牌以及撤销刷新令牌(`refresh token`), 另一个缺点就是这个令牌会占用的空间会比较大 , 如果你加入了比较多的用户凭证信息,`JwtTokenStore` 不会保存任何数据,它是它在转换令牌以及授权信息等方法跟 `DefaultTokenServices ` 所扮演的角色是一样的. 

1. 定义`TokenConfig`

    在`config`包下定义`TokenConfig`, 我们暂时使用`InMemoryTokenStore`, 生成一个普通的令牌. 

   ```java
   @Configuration
   public class TokenConfig {
   
       @Bean
       public TokenStore tokenStore() {
           // 使用内存存储令牌
           return new InMemoryTokenStore();
       }
   
   }
   ```

2. 定义 `AuthorizationServerTokenServices`

   在`AuthorizationServer` 中定义`AuthorizationServerTokenServices`

   ```java
   
       @Autowired
       private TokenStore tokenStore;
   
       @Autowired
       private ClientDetailsService clientDetailsService;
   
   
       @Bean
       public AuthorizationServerTokenServices authorizationServerTokenServices() {
           DefaultTokenServices services = new DefaultTokenServices();
           services.setClientDetailsService(clientDetailsService);
           services.setSupportRefreshToken(true);
           services.setTokenStore(tokenStore);
           services.setAccessTokenValiditySeconds(7200);//令牌默认的有效时间为2小时
           services.setRefreshTokenValiditySeconds(259200);//刷新令牌默认有效期为3天
   
           return services;
       }
       
   ```

   

#### 2.4.4 令牌访问端点配置

`AuthorizationServerEndpointsConfigurer` 这个对象的实例可以完成令牌服务以及令牌`endpoint` 的配置

##### 配置授权类型(`Grant Types`)

`AuthorizationServerEndpointsConfigurer`  通过设定以下属性来决定支持的授权类型(`Grant Types`)

- `authenticationManager`: 认证管理器 ,当你选择了资源所有者的密码(`password`)授权类型的时候,请设置这个属性注入一个`AuthenticationManager` 对象
- `userDetailsService`:如果你设置了这个属性,那说明你有一个自己定义的`UserDetailsService` 接口的实现,或者你可以把这个东西设置到全局域上面去(例如`GlobalAuthenticationManagerConfigurer` 这个配置对象), 当你设置了这个后, 那么`refresh_token`即刷新令牌授权类型模式的流程中就会包含一个检查,用来确保这个账号是否仍然有效,假如说你禁用了这个账号了的话. 
-  `authorizationCodeServices`: 这个属性是用来设置授权码服务(即` AuthorizationCodeServices` 的实例对象),主要用于`authorization_code` 授权码类型模式. 
- `implicitGrantService`:  这个属性用于设置隐形授权模式,用来管理隐式授权模式的状态. 
- `tokenGranter`: 当你设置了这个东西(即`tokenGranter`接口实现), 那么授权就会交给你完全掌握,并且会忽略掉上面的几个属性, 这个属性一般是用作扩展用途的,即标准的四种授权模式已经满足不了你的需求的时候, 才会用到这个. 





##### 配置授权端点的`URL`(Endpoint URLs)

`AuthorizationServerEndpointsConfigurer`  这个配置对象有一个叫做`pathMapping()` 方法来配置端点`URL`连接, 它有两个参数: 

- 第一个参数: `String` 类型的,这个端点`URL` 的默认链接. 
- 第二个参数: `String` 类型的, 你要进行替代的`URL` 链接. 

以上的参数都将以`/` 字符为开始的字符串,框架默认的`URL`链接如下列表,可以作为这个`pathMapping()` 方法的第一个参数

- `/oauth/authorize`: 授权端点
- `/oauth/token`: 令牌端点
- `/oauth/confirm_access`:用户确认授权提交端点
- `/oauth/error`: 授权服务错误信息端点
- `/oauth/check_token`: 用于资源服务访问的令牌解析端点
- `/oauth/token_key`:提供公有密钥的端点,如果你使用`JWT`令牌的话.



需要注意的是授权端点这个`URL`应该被`Spring Security`保护起来只供授权用户访问. 

在`AuthorizationServer` 中配置令牌访问端点

```java


    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private AuthorizationCodeServices authorizationCodeServices;

    /**
     * <p>令牌访问端点</p>
     *
     * @param endpoints
     * @author luyanan
     * @since 2020/7/30
     */
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.authenticationManager(authenticationManager)
                .authorizationCodeServices(authorizationCodeServices)
                .tokenServices(authorizationServerTokenServices())
                .allowedTokenEndpointRequestMethods(HttpMethod.POST);


    }


    @Bean
    public AuthorizationCodeServices authorizationCodeServices() {

        // 设置授权码模式的授权码如何存储, 暂时采用内存存储的方式
        return new InMemoryAuthorizationCodeServices();
    }

 



```



#### 2.4.5  令牌端点的安全约束

`AuthorizationServerSecurityConfigurer` 用来配置令牌端点(`Token Endpoint`) 的安全约束,在`AuthorizationServer` 中配置如下

```java
   //令牌端点安全约束
    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        security.tokenKeyAccess("permitAll()")  //(1)
                .checkTokenAccess("permitAll()") //(2)
                .allowFormAuthenticationForClients();//(3)
    }
```

1. `tokenkey`  这个`endpoint` 当使用`jwtToken` 且使用非对称加密的时候, 资源服务用于获取公钥而开放,这里指这个`endpoint` 完全开放. 
2. `checkToken` 在个`endpoint` 完全公开
3. 允许表单认证



##### 授权服务器总结: 授权服务配置分为三大块,可以关联记忆

既然要完成认证,它首先要知道客户端信息从哪里读取,因此要进行客户端详情配置. 

既然要颁发`token`, 那必须要定义`token` 的相关`endpoint` , 以及`token` 如何存取,以及客户端支持哪些类型的`token`. 

既然暴露了一些`endpoint`, 那对这些`endpoint` 可以定义一些安全上的约束等. 



#### 2.4.5 `web` 安全配置

在`config`包路径下添加`WebSecurityConfig`文件

```java
package com.spring.security.uaa.config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.method.configuration.EnableGlobalMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.provisioning.UserDetailsManager;

import javax.jws.Oneway;

/**
 * @author luyanan
 * @since 2020/7/30
 * <p>web安全配置</p>
 **/
@Configuration
@EnableGlobalMethodSecurity(securedEnabled = true, prePostEnabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {


    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails zhangsan = User.withUsername("zhangsan").password(passwordEncoder().encode("123")).authorities("p1").build();
        UserDetailsManager userDetailsManager = new InMemoryUserDetailsManager();
        userDetailsManager.createUser(zhangsan);
        return userDetailsManager;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }


    // 安全拦截机制
    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http.csrf().disable()
                .authorizeRequests()
                .antMatchers("/r/r1").hasAnyAuthority("p1")
                .antMatchers("/r/r2").hasAnyAuthority("p2")
                .antMatchers("/login").permitAll()
                .anyRequest().authenticated()
                .and().formLogin();
    }
}

```



## 3. 授权模式

### 3.1 授权码模式

#### 3.1.1 授权码模式介绍

下面是授权码模式的交互流程

![image-20200801092730737](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200801092730737.png)



1. 资源拥有者打开客户端,客户端要求资源拥有者给予授权,它将浏览器被重定向到授权服务器,重定向时会附加客户端的身份信息,如:

   ```text
   /uaa/oauth/authorize?client_id=c1&response_type=code&scope=all&redirect_uri=http://www.baidu.com
   
   ```

   参数列表如下: 

   - `client_id`:  客户端准入标识
   - `response_type`：授权码模式固定为`code`
   - `scope`: 客户端权限
   - `redirect_uri`:跳转`url`, 当授权码申请成功后将跳转到此地址,并在后面带上`code`参数(授权码)

2. 浏览器出现向授权服务器授权页面后, 之后将用户同意授权. 

3. 授权服务器将授权码(`AuthorizationCode`) 转经浏览器发送给`client`（通过`redirect_uri`）

4. 客户端拿着授权码向授权服务器索要访问`access_token`, 请求如下:

```text
/uaa/oauth/token?
client_id=c1&client_secret=secret&grant_type=authorization_code&code=5PgfcD&redirect_uri=http://www.baidu.com

```

参数列表如下:

- `cleint_id` 客户端准入标识
- `client_secret`: 客户端密钥
- `grant_type`:授权类型, 添加 `authorization_code`  表示授权码模式
- `code`: 授权码,就是刚刚获取的授权码.注意: 授权码使用一次就无效了, 需要重新申请. 
- `redirect_uri`: 申请授权码时跳转的`url`,一定要和申请授权码时使用的 `redirect_uri`一致

5. 授权服务器返回令牌`access_token`

   这种模式是四种模式中最安全的一种模式,一般用于`client` 是`web`应用服务器端或者第三方的原生`APP` 调用资源服务的时候 , 因为这种模式中`access_token` 不会经过浏览器或者移动端的`APP`, 而是直接从服务端去交换,这样就最大限度的减少了令牌泄漏的危险.

#### 3.1.2 测试

浏览器访问认证页面:

```text
http://localhost:8000/uaa/oauth/authorize?client_id=c1&response_type=code&scope=all&redirect_uri=http://www.baidu.com
```



![image-20200730202057321](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200730202057321.png)

然后输入模拟的账号密码后点击登陆后进入授权页面

![image-20200730202850665](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200730202850665.png)

确认授权后,浏览器会重定向到指定路径(`redirect_uri` 中携带的路径),并附加验证码 `?code=0rlqUV`(每次都不一样), 最后使用该验证码获取`token`.

```text
POST: http://localhost:8000/uaa/oauth/token
```

![image-20200730203558606](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200730203558606.png)



### 3.2 简化模式

#### 3.2.1 简化模式介绍

下图是简化模式交互图

![image-20200801092747749](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200801092747749.png)

1. 资源拥有者打开客户端, 客户端要求资源拥有者给与授权,它将浏览器重定向到授权服务器,重定向时会附加客户端的身份信息,如:

   ```text
   /uaa/oauth/authorize?client_id=c1&response_type=token&scope=all&redirect_uri=http://www.baidu.com
   ```

   参数描述同**授权模式**, 注意`response_type=token`, 说明是简化模式

2. 浏览器出现向授权服务器授权后,之后将用户同意授权

3. 授权服务器将授权码令牌(`access_token`) 以`hash`的形式存放在重定向`url` 的`fargment` 中发送给浏览器

    注: `fargment` 主要用来 标识`URL`所标识资源里的某个资源，在`URL`的末尾通过`#` 作为`fargment`的开头,其中`#`  不属于`fargment`的值, 比如 `https://domain/index#L18` 中`L18` 就是`fargment`  的值,大家只需要通过`js`响应浏览器地址变化栏的方式就能获取到`fargment` 的值

   一般来说, 简化模式用于没有服务端的第三方单页应用,因为没有服务端就无法接受授权码. 



#### 3.2.2 代码测试

浏览器访问认证页面

```text
http://localhost:8000/uaa/oauth/authorize?client_id=c1&response_type=token&scope=all&redirect_uri=http://www.baidu.com
```

![image-20200730205559045](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200730205559045.png)

然后输入账号密码点击登陆后进入授权页面

![image-20200730205633069](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200730205633069.png)

确认授权后,浏览器会重定向到指定路径(`redirect_uri` 中携带的路径),并以`Hash`的形式存放在重定向`uri`的`fargment` 中. 

如:

```text
https://www.baidu.com/#access_token=c68eeda7-abbc-4e38-90b1-93c64f61e8b9&token_type=bearer&expires_in=5777
```



###  3.3  密码模式

#### 3.3.1 授权码模式介绍

下图是密码模式交互图

![image-20200801092803981](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200801092803981.png)

1. 资源拥有者将用户名、密码发送给客户端

2. 客户端拿着资源拥有者的用户名、密码向授权服务器请求令牌(`access_token`), 请求如下:

   ```text
   /uaa/oauth/token?
   client_id=c1&client_secret=secret&grant_type=password&username=shangsan&password=123
   ```

   参数列表如下：

   - client_id： 客户端准入标识
   - client_secret: 客户端密钥
   - grant_type: 授权类型,填写`password`  表示密码模式
   - username: 资源拥有者的用户名
   - password: 资源拥有者的密码

3. 授权服务器将令牌`access_token` 发送给`client`

    这种模式十分简单,但是却意味着直接将用户敏感信息泄露给了`client`, 因此这就说明了这种模式只能用于`client` 是我们呢自己开发的模式下,因此密码模式一般用于我们自己开发的, 第一方原生APP 或者第一方单页面应用



#### 3.3.2 代码测试

```text
POST http://localhost:8000/uaa/oauth/token
```

请求参数:

![image-20200730211928433](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200730211928433.png)



### 3.4 客户端模式

#### 3.4.1 客户端模式介绍

![image-20200801092815292](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200801092815292.png)

1. 客户端向授权服务器发送自己的身份信息,并请求令牌(`access_token`)

2. 确认客户端身份无误后, 将令牌(`access_token`) 发送给`client`, 请求如下:

   ```text
   /uaa/oauth/token?client_id=c1&client_secret=secret&grant_type=client_credentials
   
   ```

   参数列表如下:

   - `client_id`: 客户端准入标识
   - `client_secret`: 客户端密钥
   - `grant_type`: 授权类型,添加`client_credentials` 表示客户端模式

   这种模式是最方便但是最不安全的模式,因此这就要求我们对`client` 完全信任,而`client`本身也是安全的. 因此这种模式一般用来提供给我们完全信任的服务器端服务, 比如:合作方系统对接,拉取一组用户信息.

#### 3.4.2 客户端模式代码演示

```text
POST http://localhost:53020/uaa/oauth/token
```



![image-20200730213104989](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200730213104989.png)



## 4. 资源服务测试

### 4.1 资源服务器配置

`@EnableResourceServer` 注解到一个`@Configuration` 配置类上,并且必须使用`ResourceServerConfigurer` 这个配置对象来进行配置(可以选择继承`ResourceServerConfigurerAdapter` 然后覆盖里面的方法, 参数就是这个对象的实例), 下面是一些可以配置的属性:

 `ResourceServerSecurityConfigure` 中主要包括

- `tokenServices`: `ResourceServerTokenServices` 类的实例,用来实现令牌服务
- `tokenStore`:`TokenStore`类的实例,指定令牌如何访问, 与`tokenServices` 配置可选
-  `resourceId`: 这个资源服务的`ID`, 这个属性是可选的,但是推荐设置并在授权服务中进行验证.
- 其他的扩展属性例如`tokenExtractor` 令牌提取器用来提取请求中的令牌.

`HttpSecurity`  配置这个与`Spring Security` 的配置类似

- 请求匹配器, 用来设置需要进行保护的资源,默认的情况下是保护资源服务的全部路径
- 通过`http.authorizeRequests()` 来设置受保护资源的访问规则
- 其他的自定义权限保护规则通过`HttpSecurity`    来进行配置.

`@EnableResourceServer` 注解自动增加了一个类型为`OAuth2AuthenticationProcessingFilter` 的过滤器链,

编写 `ResouceServerConfig`:

```java
@Configuration
@EnableResourceServer
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {


    public static final String RESOURCE_ID = "res1";


    @Override
    public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
        resources.resourceId(RESOURCE_ID)
                .tokenServices(tokenServices())
                .stateless(true);
    }


    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/**")
                .access("#oauth2.hasScope('all')")
                .and()
                .csrf()
                .disable()
                .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS);
    }



}

```



### 4.2  验证`token`

`ResourceServerTokenServices` 是组成授权服务的另一半,如果你的授权服务和资源服务在同一个应用程序上的话, 你可以使用`DefaultTokenServices`, 这样的话, 就不需要考虑关于实现所有必要接口的一致性问题,如果你的资源服务是分开开的,那么你就必须要确保能够有匹配服务提供的` ResourceServerTokenServices`, 它知道如何对令牌进行解码. 

令牌解析方法: 使用`DefaultTokenServices` 在资源服务器本地配置令牌存储、解码、解析方式,使用`RemoteTokenServices` 资源服务器通过`HTTP` 请求来解码令牌,每次都请求授权服务器端点`/oauth/check_token`

使用授权服务的`/oauth/check_token` 端点你需要在授权服务将这个端点暴露出去,以便资源服务可以进行访问,这在咱们的授权服务i的配置中已经提到了,下面是一个例子,在这个例子中, 我们在授权服务中配置了`/oauth/check_token` 和 `/oauth/token_key` 这两个端点. 

```java
//令牌端点安全约束
    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        security.tokenKeyAccess("permitAll()")  //(1)
                .checkTokenAccess("permitAll()") //(2)
                .allowFormAuthenticationForClients();//(3)
    }
```

在资源服务配置`RemoteTokenServices`, 在`ResourceServerConfig` 中配置:

```java

    public static final String RESOURCE_ID = "res1";
    /**
     * 资源服务令牌解析服务
     *
     * @return
     */
    @Bean
    public ResourceServerTokenServices tokenServices() {

        //使用远程服务请求授权服务器校验token , 必须指定校验token 的url,client_id,client_secret
        RemoteTokenServices services = new RemoteTokenServices();
        services.setCheckTokenEndpointUrl("http://localhost:8000/uaa/oauth/check_token");

        services.setClientId("c1");
        services.setClientSecret("secret");
        return services;
    }

    @Override
    public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
        resources.resourceId(RESOURCE_ID)
                .tokenServices(tokenServices())
                .stateless(true);
    }
```



### 4.3 编写资源

在`controller` 包下编写`OrderController`, 此`controller` 表示订单资源的访问类

```java
@RestController
@RequestMapping("r")
public class OrderController {


    @GetMapping("r1")
    @PreAuthorize("hasAnyAuthority('p1')")
    public String r1() {
        return "访问r1资源";
    }

}
```



### 4.4 添加安全访问控制

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {


    //安全拦截机制
    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http.csrf().disable()
                .authorizeRequests()
                .antMatchers("/r/**").authenticated()// 所有r/**的请求都必须通过认证
                .anyRequest().permitAll(); // 除了r/r** 之外的请求都放行
    }
}

```



### 4.5 代码测试

##### 4.5.1 申请令牌

我们这里使用的是密码模式

![image-20200731100041979](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200731100041979.png)

返回结果:

```json
{
    "access_token": "915fe34e-72e6-4b1f-b1ee-74c4b91d483a",
    "token_type": "bearer",
    "refresh_token": "08190be7-0b32-4576-a1e7-170211aefe5e",
    "expires_in": 7199,
    "scope": "all"
}
```



##### 4.5.2 请求资源

按照`oauth2.0`的协议要求,请求资源需要携带`token`,如下：

`token`的参数名为:`Authorization`, 值为: `Bearer token值`

![image-20200731100457034](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200731100457034.png)

当我们输入一个错误的`token`的话可以看到

![image-20200731100519668](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200731100519668.png)

提示`token` 无效

> 具体代码见  `https://github.com/lyn-workspace/spring-security-oauth2-demo`的 `spring-security-oauth-memory` 分支



## 5. `JWT` 令牌

### 5.1 `JWT` 介绍

通过上面的测试我们发现,当资源服务和授权服务不在一起的时候, 资源服务使用`RemoteTokenServices` 远程请求授权服务验证`token`, 如果访问量大的话会影响系统的性能. 

解决上面的问题:

令牌采用`JWT` 格式即可以解决上面的问题,用户认证通过会得到一个`JWT`令牌,`JWT` 令牌中已经包含了用户相关的信息,客户端只需要携带`JWT` 访问资源服务, 资源服务根据事先约定好的算法自行完成令牌校验,无需每次都请求认证服务完成授权. 

#### 5.1.1 什么是`JWT`

`JSON Web Token（JWT）` 是一个开放的行业标准(`RFC 7519`), 它定义了一种简洁、自包含的协议格式,用于通信双方传递`json` 数据,传递的信息经过数据签名可以被验证和信任,`JWT` 可以使用`HMAC` 算法或者使用`RSA` 的公钥/私钥 来对加密签名,防止防止被篡改. 

官网: https://jwt.io/

标准: https://tools.ietf.org/html/rfc7519

`JWT` 令牌的优点：

1. `jwt` 基于`json`,非常方便解析
2. 可以在令牌中自定义丰富的内容,易扩展. 
3. 通过非对称加密算法以及数字签名,`JWT`防止被篡改,安全性高. 
4. 资源服务使用`JWT`可以不依赖认证服务就可以完成授权. 

缺点: 

1. `JWT` 令牌较长,占存储空间较大。 



#### 5.1.2 `JWT`令牌结构

通过学习`JWT` 令牌结构为自定义`jwt` 令牌打好基础. 

`jwt` 令牌由三部分组成, 每个部分中间使用`(.)` ,比如`xxxx.yyyy.zzzzz`

-  `Header`

   头部包含令牌的类型(即`jwt`) 以及使用的哈希算法(如`HMAC SHA256`或`RSA`)

  一个例子如下： 

  下面是`Header` 部分的内容: 

  ```json
  {
   "alg": "HS256",
   "typ": "JWT"
  }
  ```

  将上面的内容使用`Base64Url` 编码, 得到一个字符串就是`jwt` 令牌的一部分. 

- `Payload`

   第二部分是负载, 内容也是一个`json` 对象,内容也是一个`json`对象,她是存放有效信息的地方,它可以存放`jwt` 现成的字段,比如`iss`(签发者),`exp`(过期时间戳),`sub`(面向的用户)等, 也可以自定义字段. 

  最后将第二部分负载使用`Base64Url` 编码,得到一个字符串就是`JWT` 令牌的第二部分

  一个例子:

  ```json
  {
   "sub": "1234567890",
   "name": "456",
   "admin": true
  }
  
  ```

- `Signature`

   第三部分是签名,此部分用于防止`jwt` 内容被篡改的, 

  这个部分使用`Base64Url` 将前面两部分进行编码,编码后使用点`(.)` 连接组成字符串,最后使用`header` 中声明签名算法进行签名

  一个例子:

  ```java
  HMACSHA256(
   base64UrlEncode(header) + "." +
   base64UrlEncode(payload),
   secret)
  
  ```

  `base64UrlEncode(header)`: `jwt` 令牌的第一部分

  `base64UrlEncode(payload)`: `jwt` 令牌的第二部分

  `secret`: 签名所使用的密钥

### 5.2 配置`jwt` 令牌服务

在`uaa` 中配置`JWT` 令牌服务,即可简单实现生成`jwt` 格式的令牌 

1. `TokenConfig`

   ```java
   @Configuration
   public class TokenConfig {
       private String SIGNING_KEY = "uaa123";
   
       @Bean
       public TokenStore tokenStore() {
           // 使用内存存储令牌
           return new JwtTokenStore(accessTokenConverter());
       }
   
   
       @Bean
       public JwtAccessTokenConverter accessTokenConverter() {
           JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
           //对称密钥,资源服务器使用改密钥进行验证
           converter.setSigningKey(SIGNING_KEY);
           return converter;
       }
   
   }
   
   ```

2. 定义`Jwt` 令牌服务

    修改`AuthorizationServer`

   ```java
   
       @Autowired
       private JwtAccessTokenConverter accessTokenConverter;
   
       public AuthorizationServerTokenServices authorizationServerTokenServices() {
           DefaultTokenServices tokenServices = new DefaultTokenServices();
           tokenServices.setClientDetailsService(clientDetailsService);
           tokenServices.setSupportRefreshToken(true);
           tokenServices.setTokenStore(tokenStore);
   
           TokenEnhancerChain tokenEnhancerChain = new TokenEnhancerChain();
           tokenEnhancerChain.setTokenEnhancers(Arrays.asList(accessTokenConverter));
           tokenServices.setTokenEnhancer(tokenEnhancerChain);
           tokenServices.setAccessTokenValiditySeconds(7200);//令牌的默认有效时间 2小时
           tokenServices.setRefreshTokenValiditySeconds(259200); //刷新令牌的默认有效时间3天
           return tokenServices;
       }
   
   ```





### 5.3 生成`jwt` 令牌

![image-20200731163312613](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200731163312613.png)

生成的结果:

```json
{
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsicmVzMSJdLCJ1c2VyX25hbWUiOiJ6aGFuZ3NhbiIsInNjb3BlIjpbImFsbCJdLCJleHAiOjE1OTYxOTE1ODIsImF1dGhvcml0aWVzIjpbInAxIl0sImp0aSI6Ijc5ODYwMTI2LWFmNDItNGY4NC1hNzI5LTRjYzU0MDQ4NDVlNCIsImNsaWVudF9pZCI6ImMxIn0.K9dyhBgEfQvH4hcixhU4r0eo8iV9WVZhL8q_WbpCRWE",
    "token_type": "bearer",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsicmVzMSJdLCJ1c2VyX25hbWUiOiJ6aGFuZ3NhbiIsInNjb3BlIjpbImFsbCJdLCJhdGkiOiI3OTg2MDEyNi1hZjQyLTRmODQtYTcyOS00Y2M1NDA0ODQ1ZTQiLCJleHAiOjE1OTY0NDM1ODIsImF1dGhvcml0aWVzIjpbInAxIl0sImp0aSI6IjI0N2ZiNWM4LTBhYzUtNDY4OS1iYTUwLTA2MDMyNThlYjk4ZSIsImNsaWVudF9pZCI6ImMxIn0.1qm0fluVqht7VpvaLYohL0zs6bUuQKqXJfhg9tlFeXo",
    "expires_in": 7198,
    "scope": "all",
    "jti": "79860126-af42-4f84-a729-4cc5404845e4"
}
```





### 5.4  校验`jwt`  令牌

资源服务需要和授权服务拥有一样的签名,令牌服务

1.  将授权服务中的`TokenConfig` 复制到资源服务中

2. 屏蔽资源服务原来的令牌服务类

   ```java
   
       @Autowired
       private TokenStore tokenStore;
   
       public static final String RESOURCE_ID = "res1";
   
       /**
        * 资源服务令牌解析服务
        *
        * @return
        */
   //    @Bean
   //    public ResourceServerTokenServices tokenServices() {
   //
   //        //使用远程服务请求授权服务器校验token , 必须指定校验token 的url,client_id,client_secret
   //        RemoteTokenServices services = new RemoteTokenServices();
   //        services.setCheckTokenEndpointUrl("http://localhost:8000/uaa/oauth/check_token");
   //
   //        services.setClientId("c1");
   //        services.setClientSecret("secret");
   //        return services;
   //    }
       @Override
       public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
           resources.resourceId(RESOURCE_ID)
                   .tokenStore(tokenStore)
                   .stateless(true);
       }
   ```

3. 测试

   1. 申请`jwt` 令牌

   2. 使用 令牌请求资源

      ![image-20200731163857149](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200731163857149.png)

   3. 令牌申请成功后可以使用 `/uaa/oauth/check_token` 查询令牌的有效期,并查询令牌的内容, 例如如下： 

      ![image-20200731164512645](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200731164512645.png)



##  6.  完善环境配置

截至目前客户端信息和授权码信息都是存储到内存中的,生成环境中通常都会存储在数据库中,下面完善环境的配置

### 6.1 创建表

`oauth_client_details` 表用来存储客户端信息

```mysql
DROP TABLE IF EXISTS `oauth_client_details`;


CREATE TABLE `oauth_client_details`  (
 `client_id` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '客户端标
识',
 `resource_ids` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL
COMMENT '接入资源列表',
 `client_secret` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL
COMMENT '客户端秘钥',
 `scope` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
 `authorized_grant_types` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT
NULL,
 `web_server_redirect_uri` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT
NULL,
 `authorities` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
 `access_token_validity` int(11) NULL DEFAULT NULL,
 `refresh_token_validity` int(11) NULL DEFAULT NULL,
 `additional_information` longtext CHARACTER SET utf8 COLLATE utf8_general_ci NULL,
 `create_time` timestamp(0) NOT NULL DEFAULT CURRENT_TIMESTAMP(0) ON UPDATE
CURRENT_TIMESTAMP(0),
 `archived` tinyint(4) NULL DEFAULT NULL,
 `trusted` tinyint(4) NULL DEFAULT NULL,
 `autoapprove` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
PRIMARY KEY (`client_id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci COMMENT = '接入客户端信息'
ROW_FORMAT = Dynamic;


INSERT INTO `oauth_client_details` VALUES ('c1', 'res1',
'$2a$10$NlBC84MVb7F95EXYTXwLneXgCca6/GipyWR5NHm8K0203bSQMLpvm', 'ROLE_ADMIN,ROLE_USER,ROLE_API',
'client_credentials,password,authorization_code,implicit,refresh_token', 'http://www.baidu.com',
NULL, 7200, 259200, NULL, '2019‐09‐09 16:04:28', 0, 0, 'false');
INSERT INTO `oauth_client_details` VALUES ('c2', 'res2',
'$2a$10$NlBC84MVb7F95EXYTXwLneXgCca6/GipyWR5NHm8K0203bSQMLpvm', 'ROLE_API',
'client_credentials,password,authorization_code,implicit,refresh_token', 'http://www.baidu.com',
NULL, 31536000, 2592000, NULL, '2019‐09‐09 21:48:51', 0, 0, 'false');

```

`oauth_code` 表 用来存储授权码:

```mysql
DROP TABLE IF EXISTS `oauth_code`;
CREATE TABLE `oauth_code`  (
`create_time` timestamp(0) NOT NULL DEFAULT CURRENT_TIMESTAMP,
`code` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
`authentication` blob NULL,
INDEX `code_index`(`code`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Compact;
```

### 6.2 配置授权服务

####  6.2.1 修改`修改AuthorizationServer`

`ClientDetailsService`和`AuthorizationCodeServices` 从数据库中读取

```java
@Configuration
@EnableAuthorizationServer
public class AuthorizationServer extends AuthorizationServerConfigurerAdapter {


    @Autowired
    private TokenStore tokenStore;

    @Autowired
    private ClientDetailsService clientDetailsService;


    @Bean
    public AuthorizationServerTokenServices authorizationServerTokenServices() {
        DefaultTokenServices services = new DefaultTokenServices();
        services.setClientDetailsService(clientDetailsService);
        services.setSupportRefreshToken(true);
        services.setTokenStore(tokenStore);
        services.setAccessTokenValiditySeconds(7200);//令牌默认的有效时间为2小时
        services.setRefreshTokenValiditySeconds(259200);//刷新令牌默认有效期为3天

        return services;
    }


    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private AuthorizationCodeServices authorizationCodeServices;

    /**
     * <p>令牌访问端点</p>
     *
     * @param endpoints
     * @author luyanan
     * @since 2020/7/30
     */
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.authenticationManager(authenticationManager)
                .authorizationCodeServices(authorizationCodeServices)
                .tokenServices(authorizationServerTokenServices())
                .allowedTokenEndpointRequestMethods(HttpMethod.POST);


    }


    @Bean
    public AuthorizationCodeServices authorizationCodeServices() {

        // 设置授权码模式的授权码如何存储, 暂时采用内存存储的方式
        return new InMemoryAuthorizationCodeServices();
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {

// clients.withClientDetails(clientDetailsService);
        clients.inMemory()// 使用in‐memory存储
                .withClient("c1")// client_id
                .secret(new BCryptPasswordEncoder().encode("secret"))
                .resourceIds("res1")
                .authorizedGrantTypes("authorization_code",
                        "password", "client_credentials", "implicit", "refresh_token")// 该client允许的授权类型 authorization_code,password,refresh_token,implicit,client_credentials
                .scopes("all")// 允许的授权范围
                .autoApprove(false)
//加上验证回调地址
                .redirectUris("http://www.baidu.com");

    }


    //令牌端点安全约束
    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        security.tokenKeyAccess("permitAll()")  //(1)
                .checkTokenAccess("permitAll()") //(2)
                .allowFormAuthenticationForClients();//(3)
    }
}

```

### 6.3 测试

#### 6.3.1 测试申请令牌

使用密码模式申请令牌,客户端信息需要和数据库中的信息一致

![image-20200801091256042](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200801091256042.png)

#### 6.3.2 测试授权码模式

生成的授权码存在数据库中

访问`http://localhost:8000/uaa/oauth/authorize?client_id=c1&response_type=code&scope=all&redirect_uri=http://www.baidu.com`,输入账号密码以及授权之后, 可以看到重定向后的`url` 为`https://www.baidu.com/?code=iTtRVv`,授权码为`iTtRVv`

这个时候我们看数据库`oauth_code` 这个表, 同样插入了一条数据,

![image-20200801091511894](7.Spring%20Cloud%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%AE%A4%E8%AF%81.assets/image-20200801091511894.png)

这样就实现了授权码保存在了数据库中. 而不是像我们之前的保存在内存中. 



## 7. 附录

### `HttpSecurity`

`HttpSecurity`的配置列表

| 方法                  | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| `openidLogin()`       | 用于基于`Openid`的验证                                       |
| `headers()`           | 将安全头添加到响应中                                         |
| `cors()`              | 配置跨域资源共享(`cors()`)                                   |
| `sessionManagement()` | 允许配置会话管理                                             |
| `portMapper()`        | 允许配置一个`PortMapper(HttpSecurity#(getSharedObject(class)))`，其他提供`SecurityConfigurer`的对象使用 `PortMapper` 从 `HTTP` 重定 向到 `HTTPS` 或者从 `HTTPS` 重定向到 `HTTP`。默认情况下，`Spring Security`使用一个`PortMapperImpl`映射 `HTTP` 端口8080到 `HTTPS` 端口 8443，`HTTP` 端口80到 `HTTPS` 端口443 |
| `jee()`               | 配置基于容器的预认证,在这种情况下,认证由`Servlet` 容器管理   |
| `x509()`              | 配置基于`x509`                                               |
| `rememberMe`          | 允许配置"记住我"的验证                                       |
| `authorizeRequests()` | 允许基于使用`HttpServletRequests` 限制访问                   |
| `requestCache()`      | 允许配置请求缓存                                             |
| `exceptionHandling()` | 允许配置错误处理                                             |
| `securityContext()`   | 在`HttpServletRequest`之间的`SecurotyContextHolder`上设置`SecurotyContext`的管理,当使用`WebSecurityConfigurerAdapte`时, 这才自动应用 |
| `servletApi()`        | 将`HttpServletRequest`方法与在其上找到的值集成到`SecurityContext中。 当使用` `WebSecurityConfigurerAdapter`时，这将自动应用 |
| `csrf()`              | 添加`csrf` 的支持, 当使用`WebSecurityConfigurerAdapter`的时候默认启用 |
| `logout()`            | 添加退出登录支持,当使用`WebSecurityConfigurerAdapter`的时候, 自动应用,默认情况是,访问`URL` `/logout`, 使`HTTP Session` 无效来清除用户,请求已配置的任何`#rememberMe()` 身份验证,清除`SecurityContextHolder`, 然后重定向到`/login?success` |
| `anonymous()`         | 允许配置匿名用户的标识方法,当与`WebSecurityConfigurerAdapter` 结合使用的时候,将自动应用,默认情况下,匿名用户使用`org.springframework.security.authentication.AnonymousAuthenticationToken`,并包含角色 `ROLE_ANONYMOUS` |
| `formLogin()`         | 支持基于表单的身份验证, 如果为指定 `定FormLoginConfigurer#loginPage(String)`,则将生成默认登录页面 |
| `oauth2Login()`       | 根据外部`Oauth2.0` 或者`OpenId Connect 1.0` 提供程序配置身份验证 |
| `requiresChannel()`   | 配置通道安全,为了使该配置有效,必须提供至少一个所需信道的映射 |
| `httpBasic()`         | 配置`http Basic`的验证                                       |
| `addFilterAt()`       | 在指定的`Filter`类的位置添加过滤器                           |

