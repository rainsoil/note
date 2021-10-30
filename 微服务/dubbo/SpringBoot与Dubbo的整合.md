# SpringBoot 与Dubbo 的整合



##  新建项目

我们先新建一个基于Springboot 的项目

##  jar包

添加dubbo 和zookeeper 的jar

```xml

        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>4.0.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>4.0.0</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.apache.dubbo/dubbo-spring-boot-starter -->
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>2.7.1</version>
        </dependency>

    

        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.7.2</version>
        </dependency>
```



## 编写接口和实现类

```java
package com.dubbo.spring;

/**
 * @author luyanan
 * @since 2019/11/20
 * <p>对外暴漏一个接口</p>
 **/

public interface UserApi {

    String info(String id);

}

```

```java
import com.dubbo.spring.UserApi;
import org.apache.dubbo.config.annotation.Reference;
import org.apache.dubbo.config.annotation.Service;

/**
 * @author luyanan
 * @since 2019/11/20
 * <p></p>
 **/

//  @Service  Dubbo 的注解
@Service
public class UserApiImpl implements UserApi {


    @Override
    public String info(String id) {
        System.out.println("info 请求");
        return "张三:" + id;
    }
}


```



##  编写controller

```java
/**
 * @author luyanan
 * @since 2019/11/20
 * <p></p>
 **/
@RestController
@RequestMapping("user")
public class UserController {

    // dubbo 提供了注入的方法
    @Reference
    private UserApi userApi;


    @GetMapping("info/{id}")
    public String info(@PathVariable("id") String id) {
        return userApi.info(id);
    }
}

```



## 添加dubbo 的配置文件

```properties
##  dubbo 项目名称
dubbo.application.name=dubbo-spring-server
## dubbo 扫描路径
dubbo.scan.base-packages=com.dubbo.spring
## 注册中心地址
dubbo.registry.address=zookeeper://192.168.86.128:2181

```



运行后, 我们访问  http://localhost:8080/user/info/100000 

>  张三:100000 

