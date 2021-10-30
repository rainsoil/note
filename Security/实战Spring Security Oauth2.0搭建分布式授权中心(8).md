

#  实战Spring Security Oauth2.0搭建分布式授权中心

##  1. 需求分析

技术方案如下:

![image-20200801092902342](8.%E5%AE%9E%E6%88%98Spring%20Security%20Oauth2.0%E6%90%AD%E5%BB%BA%E5%88%86%E5%B8%83%E5%BC%8F%E6%8E%88%E6%9D%83%E4%B8%AD%E5%BF%83.assets/image-20200801092902342.png)

1. `UAA` 认证服务负责认证授权
2. 所有的请求经过网关到达微服务
3. 网关将负责鉴权客户端以及请求转发
4. 网关将`token` 解析后传给微服务, 微服务进行授权. 

## 2. 父工程 `spring-security-oauth2-demo`

定义常用的一些jar的版本

`pom.xml`

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
        <module>eureka</module>
        <module>gateway</module>
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



## 3. `eureka` 注册中心

所有的微服务的请求都经过网关,网关从注册中心读取微服务的地址,将请求转发到微服务. 

本节完成注册中心的搭建,注册中心采用`eureka`

创建`maven` 工程

### 3.1 `pom.xml`

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
    <groupId>com.spring.security.eureka</groupId>
    <artifactId>eureka</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>eureka</name>
    <description>注册中心</description>

    <properties>
        <java.version>1.8</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>

    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
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

###  3.2  主启动类 `EurekaApplication`

```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication.class, args);
    }

}

```



### 3.3 `application.yml` 配置文件

```yaml
spring:
  application:
    name: eureka
server:
  port: 8002
eureka:
  server:
    enable-self-preservation: false # 关闭服务器的自我保护,客户端心跳检测15分钟内错误达到80% 服务会保护,导致别人还以为是好的服务
    eviction-interval-timer-in-ms: 10000 #清理间隔（单位毫秒，默认是60*1000）5秒将客户端剔除的服务在服务注册列表中剔除#
    use-read-only-response-cache: true #eureka是CAP理论种基于AP策略，为了保证强一致性关闭此切换CP默认不关闭 false关闭
  client:
    register-with-eureka: false #false:不作为一个客户端注册到注册中心
    fetch-registry: false #为true时，可以启动，但报异常：Cannot execute request on any known server
    instance-info-replication-interval-seconds: 10
    service-url:
      defaultZone: http://localhost:${server.port}/eureka/
  instance:
#    hostname: ${spring.cloud.client.ip‐address}
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${server.port}}

```



### 3.4 测试

启动成功后, 浏览器访问 `http://localhost:8002/`, 出现

![image-20200731211116507](8.%20Spring%20Security%20Oauth2.0%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E6%8E%88%E6%9D%83.assets/image-20200731211116507.png)

即为搭建成功. 



## 4. `UAA` 授权认证中心

它承载了OAuth2.0接入方认证、登入用户的认证、授权以及生成令牌的职责，完成实际的用户认证、授权功能

### 4.1 `pom.xml`

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

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
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



### 4.2  `UaaApplication`  主启动类

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



### 4.3  `application.yml` 配置文件

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



eureka:
  client:
    service-url:
      defaultZone: http://localhost:8002/eureka/
  instance:
    prefer-ip-address: true




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



### 4.4  授权客户端和授权码`sql`导入

```mysql

CREATE TABLE `oauth_client_details` (
                                        `client_id` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '客户端标\r\n识',
                                        `resource_ids` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL COMMENT '接入资源列表',
                                        `client_secret` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL COMMENT '客户端秘钥',
                                        `scope` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
                                        `authorized_grant_types` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
                                        `web_server_redirect_uri` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
                                        `authorities` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
                                        `access_token_validity` int(11) DEFAULT NULL,
                                        `refresh_token_validity` int(11) DEFAULT NULL,
                                        `additional_information` longtext CHARACTER SET utf8 COLLATE utf8_general_ci,
                                        `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                                        `archived` tinyint(4) DEFAULT NULL,
                                        `trusted` tinyint(4) DEFAULT NULL,
                                        `autoapprove` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
                                        PRIMARY KEY (`client_id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC COMMENT='接入客户端信息';


INSERT INTO `oauth_client_details`(`client_id`, `resource_ids`, `client_secret`, `scope`, `authorized_grant_types`, `web_server_redirect_uri`, `authorities`, `access_token_validity`, `refresh_token_validity`, `additional_information`, `create_time`, `archived`, `trusted`, `autoapprove`) VALUES ('c1', 'res1', '$2a$10$H1wKlULSV5J9XsA9AACC9OsNtIYOaFEAH3gi5nYsIkWGkqiYLCwli', 'ROLE_ADMIN,ROLE_USER,ROLE_API', 'client_credentials,password,authorization_code,implicit,refresh_token', 'http://www.baidu.com', NULL, 7200, 259200, NULL, '2020-08-01 14:13:18', 0, 0, 'false');


CREATE TABLE `oauth_code` (
                              `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
                              `code` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL,
                              `authentication` blob,
                              KEY `code_index` (`code`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=COMPACT;

```



### 4.5 配置`jwt`的`token` 令牌方案

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



### 4.6 授权服务器配置

```java
@Configuration
@EnableAuthorizationServer
public class AuthorizationServer extends AuthorizationServerConfigurerAdapter {


    @Autowired
    private TokenStore tokenStore;

    @Autowired
    private ClientDetailsService clientDetailsService;


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
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public ClientDetailsService clientDetailsService(DataSource dataSource) {
        ClientDetailsService clientDetailsService = new JdbcClientDetailsService(dataSource);
        ((JdbcClientDetailsService) clientDetailsService).setPasswordEncoder(passwordEncoder());
        return clientDetailsService;
    }

    @Bean
    public AuthorizationCodeServices authorizationCodeServices(DataSource dataSource) {

        // 设置授权码模式的授权码如何存储, 暂时采用内存存储的方式
//        return new InMemoryAuthorizationCodeServices();
        return new JdbcAuthorizationCodeServices(dataSource);

    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {

        clients.withClientDetails(clientDetailsService);
//        clients.inMemory()// 使用in‐memory存储
//                .withClient("c1")// client_id
//                .secret(new BCryptPasswordEncoder().encode("secret"))
//                .resourceIds("res1")
//                .authorizedGrantTypes("authorization_code",
//                        "password", "client_credentials", "implicit", "refresh_token")// 该client允许的授权类型 authorization_code,password,refresh_token,implicit,client_credentials
//                .scopes("all")// 允许的授权范围
//                .autoApprove(false)
////加上验证回调地址
//                .redirectUris("http://www.baidu.com");

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



### 4.7  安全配置

```java
@Configuration
@EnableGlobalMethodSecurity(securedEnabled = true, prePostEnabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {


    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails zhangsan = User.withUsername("zhangsan").password(passwordEncoder().encode("123")).authorities("p1").build();
        UserDetails lisi = User.withUsername("lisi").password(passwordEncoder().encode("123")).authorities("p2").build();
        UserDetailsManager userDetailsManager = new InMemoryUserDetailsManager();
        userDetailsManager.createUser(zhangsan);
        userDetailsManager.createUser(lisi);
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

                .antMatchers("/login").permitAll()
                .anyRequest().authenticated()
                .and().formLogin();
    }


}
```







## 5 . 网关

网关整个`Oauth2.0` 有两种思路, 一种是认证服务器生成`jwt` 令牌, 所有请求统一在网络层验证,判断权限等操作,另一种是由各资源服务处理,网关只做请求转发. 

我们选用第一种,我们把`API` 网关作为`Oauth2.0`的资源服务器角色,实现接入客户端权限拦截,令牌解析并转发当前登陆用户信息(`jsonToken` ) 给微服务,这样下游微服务就不需要关心令牌格式解析以及`Oauth2.0` 的相关机制. 

`API`网关在认证授权体系里面主要负责两件事情:

1. 作为`Oauth2.0`的资源服务角色,实现接入方权限拦截
2. 令牌解析并转发当前登陆用户信息(明文`token`) 给微服务

微服务拿到明文`token`(明文`token`中包含登陆用户的身份和权限信息) 后也需要做两件事情:

1. 用户授权拦截(看当前用户是否有权限访问该资源)
2. 将用户信息存储进当前线程上下文(有利于后续业务逻辑随时获取当前用户信息)



### 5.1  `pom.xml` 依赖

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
    <groupId>com.spring.security.gateway</groupId>
    <artifactId>gateway</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>gateway</name>
    <description>服务网关</description>

    <properties>
        <java.version>1.8</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>

    </properties>

    <dependencies>

        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-security</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-oauth2</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-jwt</artifactId>
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
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
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



### 5.2  主启动类

```java
@EnableZuulProxy
@EnableDiscoveryClient
@SpringBootApplication
public class GatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }

}

```



### 5.3 `application.yml` 配置文件

```yaml
spring:
  application:
    name: gateway
  main:
    allow-bean-definition-overriding: true
server:
  port: 8004

zuul:
  retryable: true
  ignored-services: "*"
  add-host-header: true
  sensitive-headers: "*"
  routes:
    uaa:
      stripPrefix: false
      path: /uaa/**
    order:
      stripPrefix: false
      path: /order/**
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8002/eureka/
  instance:
    prefer-ip-address: true

management:
  endpoints:
    web:
      exposure:
        include: refresh,health,info,env

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



### 5.4 资源服务配置

```java
@Configuration
public class ResourceServerConfig {


    public static final String RESOURCE_ID = "res1";


    /**
     * uaa的资源拦截
     */
    @Configuration
    @EnableResourceServer
    public class UaaServerConfig extends ResourceServerConfigurerAdapter {

        @Autowired
        private TokenStore tokenStore;


        @Override
        public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
            resources.tokenStore(tokenStore)
                    .resourceId(RESOURCE_ID).stateless(true);
        }

        @Override
        public void configure(HttpSecurity http) throws Exception {
            http.authorizeRequests().antMatchers("/uaa/**").permitAll();
        }
    }


    /**
     * 订单服务资源拦截
     */
    @Configuration
    @EnableResourceServer
    public class OrderServerConfig extends ResourceServerConfigurerAdapter {

        @Autowired
        private TokenStore tokenStore;

        @Override
        public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
            resources.resourceId(RESOURCE_ID).tokenStore(tokenStore).stateless(true);
        }

        @Override
        public void configure(HttpSecurity http) throws Exception {
            http.authorizeRequests().antMatchers("/order/**").access("#oauth2.hasScope('ROLE_API')");
        }
    }
}

```



### 5.5   令牌配置

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



### 5.6  安全配置

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {


    //安全拦截机制
    @Override
    protected void configure(HttpSecurity http) throws Exception {


        http
                .authorizeRequests()
                .antMatchers("/**")
                .permitAll()
                .and()
                .csrf()
                .disable();
    }
}

```



### 5.7   过滤器配置

编写过滤器, 将解析后的明文`token` 放入`header`  中转发至内部服务

```java
public class AuthFilter extends ZuulFilter {


    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() throws ZuulException {

        //1. 获取令牌内容
        RequestContext ctx = RequestContext.getCurrentContext();

        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (!(authentication instanceof OAuth2Authentication)) {

            // 无token 访问资源的情况,目前只有`uaa` 服务直接暴露
            return null;
        }

        OAuth2Authentication oAuth2Authentication = (OAuth2Authentication) authentication;

        Authentication userAuthentication = oAuth2Authentication.getUserAuthentication();
        Object principal = userAuthentication.getPrincipal();
        // 2. 组装明文token,转发给内部服务,放入header中,名称为 json-token
        List<String> authorities = new ArrayList<>();
        userAuthentication.getAuthorities().stream().forEach(a -> {
            authorities.add(((GrantedAuthority) a).getAuthority());
        });

        OAuth2Request oAuth2Request = oAuth2Authentication.getOAuth2Request();
        Map<String, String> requestParameters =
                oAuth2Request.getRequestParameters();

        Map<String, Object> jsonToken = new HashMap<>(requestParameters);
        if (null != userAuthentication) {
            jsonToken.put("principal", userAuthentication.getName());
            jsonToken.put("authorities", authorities);
        }

        ctx.addZuulRequestHeader("json-token", URLUtil.encode(JSON.toJSONString(jsonToken), "utf-8"));
        return null;
    }


}

```



### 5.8  配置类

将编写的过滤器注册成`Bean`和配置`CORS` 

```java
@Configuration
public class ZuulConfig {

    @Bean
    public AuthFilter authFilter() {
        return new AuthFilter();
    }

    @Bean
    public FilterRegistrationBean corsFilter() {
        final UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        final CorsConfiguration config = new CorsConfiguration();
        config.setAllowCredentials(true);
        config.addAllowedOrigin("*");
        config.addAllowedHeader("*");
        config.addAllowedMethod("*");
        config.setMaxAge(18000L);
        source.registerCorsConfiguration("/**", config);
        CorsFilter corsFilter = new CorsFilter(source);
        FilterRegistrationBean bean = new FilterRegistrationBean(corsFilter);
        bean.setOrder(Ordered.HIGHEST_PRECEDENCE);
        return bean;
    }


}

```





##  6. `Order` 资源服务

本工程为Order订单服务工程，访问本工程的资源需要认证通过。 本工程的目的主要是测试认证授权的功能，所以不涉及订单管理相关业务

### 6.1 `pom.xml` 依赖

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

        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
        </dependency>
        <!-- 认证授权相关-->

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-oauth2</artifactId>
        </dependency>


<!--        <dependency>-->
<!--            <groupId>org.springframework.boot</groupId>-->
<!--            <artifactId>spring-boot-starter-jdbc</artifactId>-->
<!--        </dependency>-->
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
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
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



### 6.2  主启动类

```java
@EnableHystrix
@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
public class OrderApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext applicationContext = SpringApplication.run(OrderApplication.class, args);

    }

}

```



###  6.3 `application.yml` 配置文件

```yaml
server:
  port: 8001
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
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8002/eureka/
  instance:
    prefer-ip-address: true

management:
  endpoints:
    web:
      exposure:
        include: refresh,health,info,env


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



### 6.4  令牌配置

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



### 6.5  资源服务配置

```java
@Configuration
@EnableResourceServer
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {

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


    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/**")
                .access("#oauth2.hasScope('ROLE_ADMIN')")
                .and()
                .csrf()
                .disable()
                .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS);
    }


}
```





### 6.6 `web`安全配置

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {


    //安全拦截机制
    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http.csrf().disable()
                .authorizeRequests()
                .antMatchers("/**").authenticated()// 所有r/**的请求都必须通过认证
                .anyRequest().permitAll(); // 除了r/r** 之外的请求都放行
    }
}

```



### 6.7  token拦截器

定义filter拦截token，并形成Spring Security的Authentication对象

```java
@Component
public class TokenAuthenticationFilter extends OncePerRequestFilter {


    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        String token = request.getHeader("json-token");

        if (null != token) {
            // 1. 解析token
            String decode = URLUtil.decode(token, "utf-8");
            JSONObject userJson = JSON.parseObject(decode);
            UserEntity userEntity = new UserEntity();
            userEntity.setUsername(userJson.getString("principal"));

            JSONArray authoritiesArray = userJson.getJSONArray("authorities");

            String[] authorities = authoritiesArray.toArray(new
                    String[authoritiesArray.size()]);

            // 2. 新建并填充 authentication
            UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(userEntity, null, AuthorityUtils.createAuthorityList(authorities));
            authenticationToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            // 3.将authentication 保存到上下文中
            SecurityContextHolder.getContext().setAuthentication(authenticationToken);

        }
        filterChain.doFilter(request, response);
    }
}
```



### 6.8 编写资源

`UserEntity`

````java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class UserEntity {

    private String username;

    private String password;

}

````

`OrderController`

```java
@RestController
public class OrderController {


    @GetMapping("r1")
    @PreAuthorize("hasAnyAuthority('p1')")
    public String r1() {
        UserEntity userEntity = (UserEntity) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        return userEntity.getUsername() + ":访问r1资源";
    }

    @GetMapping("r2")
    @PreAuthorize("hasAnyAuthority('p2')")
    public String r2() {
        UserEntity userEntity = (UserEntity) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        return userEntity.getUsername() + ":访问r2资源";
    }
}

```



## 7 集成测试

本案例测试过程的描述

1. 采用`Oauth2.0` 的密码模式从`UAA` 获取到`token`
2. 使用该`token` 通过网关访问订单服务中的测试资源

### 7.1 token 获取

访问`http://localhost:8004/uaa/oauth/token`, 注意:此处的端口为网关的端口

<img src="8.%E5%AE%9E%E6%88%98Spring%20Security%20Oauth2.0%E6%90%AD%E5%BB%BA%E5%88%86%E5%B8%83%E5%BC%8F%E6%8E%88%E6%9D%83%E4%B8%AD%E5%BF%83.assets/image-20200801152554057.png" alt="image-20200801152554057" style="zoom:200%;" />

返回内容:

```json
{
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsicmVzMSJdLCJ1c2VyX25hbWUiOiJ6aGFuZ3NhbiIsInNjb3BlIjpbIlJPTEVfQURNSU4iLCJST0xFX1VTRVIiLCJST0xFX0FQSSJdLCJleHAiOjE1OTYyNzA1NTAsImF1dGhvcml0aWVzIjpbInAxIl0sImp0aSI6ImE1ODFkN2JiLWQ3ZmQtNDc3Yi1hNWM0LTgxN2UwOGQ4N2QzYiIsImNsaWVudF9pZCI6ImMxIn0.AkD2rd8AnAst2gv0KdwkNta6d0Hy1tz-qNO-euMmHkw",
    "token_type": "bearer",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsicmVzMSJdLCJ1c2VyX25hbWUiOiJ6aGFuZ3NhbiIsInNjb3BlIjpbIlJPTEVfQURNSU4iLCJST0xFX1VTRVIiLCJST0xFX0FQSSJdLCJhdGkiOiJhNTgxZDdiYi1kN2ZkLTQ3N2ItYTVjNC04MTdlMDhkODdkM2IiLCJleHAiOjE1OTY1MjI1NTAsImF1dGhvcml0aWVzIjpbInAxIl0sImp0aSI6Ijg5NTIyMjA0LWUwYTgtNDNiNy1hYWM4LTUzMWUyNWEyZTYzMSIsImNsaWVudF9pZCI6ImMxIn0.tbtuWBgYZfBiAZLdwZq27CqFwW5PyytEcbQfSpjxHXo",
    "expires_in": 7199,
    "scope": "ROLE_ADMIN ROLE_USER ROLE_API",
    "jti": "a581d7bb-d7fd-477b-a5c4-817e08d87d3b"
}
```



### 携带`token` 访问`Order` 资源

访问路径为:`http://localhost:8004/order/r1`

![image-20200801152849236](8.%E5%AE%9E%E6%88%98Spring%20Security%20Oauth2.0%E6%90%AD%E5%BB%BA%E5%88%86%E5%B8%83%E5%BC%8F%E6%8E%88%E6%9D%83%E4%B8%AD%E5%BF%83.assets/image-20200801152849236.png)

发现可以访问成功. 



使用`token` 访问`r2` 资源, 由于`r2` 资源需要`p2`的权限

所以:

![image-20200801153020890](8.%E5%AE%9E%E6%88%98Spring%20Security%20Oauth2.0%E6%90%AD%E5%BB%BA%E5%88%86%E5%B8%83%E5%BC%8F%E6%8E%88%E6%9D%83%E4%B8%AD%E5%BF%83.assets/image-20200801153020890.png)



### 破坏测试

无`token` 访问测试返回内容

```json
{
    "error": "unauthorized",
    "error_description": "Full authentication is required to access this resource"
}
```





破坏`token`访问测试返回内容

```json
{
    "error": "invalid_token",
    "error_description": "Cannot convert access token to JSON"
}
```





## 8.  扩展用户信息

### 8.1 需求分析

目前`jwt` 用户中存储了用户的身份信息,权限信息,网关将`token` 明文转发给微服务使用, 目前用户身份信息仅包含了用户的账号,微服务还需要用户的`id`, 手机号等其他的信息

所以,本案例将提供扩展用户的思路和方法,满足为服务使用用户信息的需求. 

下面分析`JWT` 令牌中扩展用户信息的方案: 

在认证阶段`DaoAuthenticationProvider` 会调用`UserDetailService` 查询用户信息,这里是可以获取到齐全的用户信息的, 由于`JWT` 令牌中的用户身份信息来源子`UserDetails`,`UserDetails` 中仅仅定义了`username` 为用户的身份信息,这里有两个思路: 

- 第一是可以扩展`UserDetails`,使之可以包含更多的自定义属性
- 第二种是扩展`username`的内容, 比如存入`json` 作为`username` 的内容

相比较而言,第二种方案比较简单还不用破坏`UserDetails` 的结构,我们采用方案二. 

### 8.2 修改  `UserDetailService`

从数据库查询到`user`, 将整体`user`存入到`UserDetails`对象中., 

// 这里还是模拟使用虚拟用户, 不从数据库中查询

修改用户的实体, 增加

```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class UserEntity {

    private String id;

    private String username;

    private String password;

    private String mobile;
}

```



增加自定义的`ConsumerUserDetailsService`


````java
@Service
public class ConsumerUserDetailsService implements UserDetailsService {


    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        UserEntity zhangsan = UserEntity.builder().id("1").username("zhangsan").password(passwordEncoder.encode("123")).mobile("18111111111").build();

        return User.withUsername(JSON.toJSONString(zhangsan)).password(zhangsan.getPassword()).authorities("p1").build();

    }
}
````

修改 `WebSecurityConfig`

```java
@Configuration
@EnableGlobalMethodSecurity(securedEnabled = true, prePostEnabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {



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

                .antMatchers("/login").permitAll()
                .anyRequest().authenticated()
                .and().formLogin();
    }


}

```



###  8.3 修改资源服务过滤器

`TokenAuthenticationFilter`

```java
@Component
public class TokenAuthenticationFilter extends OncePerRequestFilter {


    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        String token = request.getHeader("json-token");

        if (null != token) {
            // 1. 解析token
            String decode = URLUtil.decode(token, "utf-8");
            JSONObject userJson = JSON.parseObject(decode);
            String principal = userJson.getString("principal");

            UserEntity userEntity = JSON.parseObject(principal, UserEntity.class);

            JSONArray authoritiesArray = userJson.getJSONArray("authorities");

            String[] authorities = authoritiesArray.toArray(new
                    String[authoritiesArray.size()]);

            // 2. 新建并填充 authentication
            UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(userEntity, null, AuthorityUtils.createAuthorityList(authorities));
            authenticationToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            // 3.将authentication 保存到上下文中
            SecurityContextHolder.getContext().setAuthentication(authenticationToken);

        }
        filterChain.doFilter(request, response);
    }
}

```

以上过程就完成自定义用户信息的方案



