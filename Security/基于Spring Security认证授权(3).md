# 基于Spring Security 认证授权

## 1. 介绍

`Spring Security` 是一个能够为基于`Spring`的企业应用系统提升声明式的安全访问控制解决方案的安全框架.由于他是`Spring` 生态系统中的一员,因为他伴随着整个`Spring` 生态系统的不断修正、升级,在`Spring Boot` 项目中加入`Spring Security` 更是十分简单,使用`Spring Security` 减少了为企业系统安全控制编写了大量重复代码的工作. 

## 2. 实战

### 2.1 `pom依赖`

```xml
 <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.1.RELEASE</version>
    </parent>

    <dependencies>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>


        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.12</version>
        </dependency>
    </dependencies>
```



### 2.2 安全配置

`Spring Security` 提供了用户名密码登陆、退出、会话管理等认证功能,只需要简单配置就可以使用. 

在`config` 包下定义`WebSecurityConfig`， 安全配置的内容包括: 用户信息、密码编码器、安全拦截机制. 

```java
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    // 配置用户信息
    @Bean
    public UserDetailsService userDetailsService() {
        InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
        manager.createUser(User.withUsername("zhangsan").password("123456").authorities("p1").build());
        manager.createUser(User.withUsername("lisi").password("123456").authorities("p2").build());
        return manager;
    }
    // 密码编码器
    @Bean
    public PasswordEncoder passwordEncoder() {
        return NoOpPasswordEncoder.getInstance();
    }
    // 配置安全拦截机制
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/r/**").authenticated()
                .anyRequest().permitAll()
                .and()
                .formLogin().successForwardUrl("/login-success");
    }
}
```

在`userDetailsService()` 方法中,我们返回了一个`UserDetailsService`  给`Spring` 容器, `Spring Security` 会使得他来获取用户信息,我们暂时使用`InMemoryUserDetailsManager` 实现类,基于内存实现,并在启动创建了`zhangsan`、`lisi` 两个用户, 并设置密码和权限. 

而在`com.spring.security.config.WebSecurityConfig#configure` 中,我们通过`HttpSecurity` 设置了安全拦截规则,其中包含了以下内容：

1. `url` 匹配`/r/**` 的资源,经过认证后才能访问.
2. 其他`url` 完全开放
3. 支持`form` 表单认证, 认证成功后跳转到`/login-success` 页面



### 2.3 编写`LoginController`

```java
@Controller
public class LoginController {
    
    
      @ResponseBody
    @GetMapping("login-success")
    public String loginSuccess() {
        return "登陆成功";
    }
}
```



### 2.4 测试

1.  启动项目, 访问 `http://localhost:8080/login`, 可以看到`Spring Security` 为我们提供的登陆页面

   ![image-20200723215211377](3.%20%E5%9F%BA%E4%BA%8ESpring%20Security%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83.assets/image-20200723215211377.png)

2. 登陆测试

   1. 当我们输入错误的账号密码的时候,

      ![image-20200723215330750](3.%20%E5%9F%BA%E4%BA%8ESpring%20Security%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83.assets/image-20200723215330750.png)

   2. 输入正确的账号密码则会登陆成功
      ![image-20200723215419020](3.%20%E5%9F%BA%E4%BA%8ESpring%20Security%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83.assets/image-20200723215419020.png)

   3. 退出

      请求`/logout` 接口退出

      ![image-20200723215537333](3.%20%E5%9F%BA%E4%BA%8ESpring%20Security%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83.assets/image-20200723215537333.png)

## 3. 授权

实现授权只需要对用户的访问进行拦截校验,校验用户的权限是否可以操作指定的资源,`Spring Security` 默认提供了授权实现方法. 

在`LoginController`中添加`r/r1` 和`r/r2` 接口

```java

    @ResponseBody
    @GetMapping("/r/r1")
    public String r1() {
        return "访问r1的资源";
    }

    @ResponseBody
    @GetMapping("/r/r2")
    public String r2() {
        return "访问r2的资源";
    }
```

在安全配置类`WebSecurityConfig` 中添加授权规则

```java

    // 配置安全拦截机制
    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http.authorizeRequests()

                //访问/r/r1资源的 url需要拥有p1权限。
                .antMatchers("/r/r1").hasAuthority("p1")
                //访问/r/r2资源的 url需要拥有p2权限。
                .antMatchers("/r/r2").hasAuthority("p2")
                .antMatchers("/r/**").authenticated()
                .anyRequest().permitAll()
                .and()
                .formLogin().successForwardUrl("/login‐success");
    }
```



测试:

1. 当账号密码输入成功的时候, 则登陆成功.

2. 访问`r/r1`和`/r/r2`, 有权限的时候则正常访问, 否则则返回403(拒绝访问)

   ![image-20200723220743302](3.%20%E5%9F%BA%E4%BA%8ESpring%20Security%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83.assets/image-20200723220743302.png)



