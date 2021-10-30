#  Spring Security 认证授权高级篇

## 1. 自定义认证

`Spring Security` 提供了非常好的认证方法, 比如: 快速上手将用户信息保存到内存中,实际开发中用户信息通常是放到数据库中,`Spring Security`   可以实现从数据库中读取用户信息,`Spring Security` 还支持多种授权方法.

###  1.1  自定义登陆页面

`Spring Security` 会根据启用的功能自动生成一个登录页面URL, 并使用默认`URL` 处理登陆的提交内容,登录i后跳转到默认的`URL`等,尽管自动生成的登陆页面很方便快速启动和运行,但大多数应用程序都希望定义自己的登录页面.

#### 1.1.1 添加`pom` 依赖

```xml
 <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
```



#### 1.2 添加页面, 在项目的`resource/templates`下放入`login.html`页面

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>登陆</title>
</head>
<body>

<div>

    <form action="/login" method="post">
        <input type="text" name="username" placeholder="用户名">

        <input type="password" name="password" placeholder="密码">
        <button type="submit">登陆</button>

    </form>

</div>
</body>
</html>

```

 

目录结构为:

![image-20200725171748284](Untitled.assets/image-20200725171748284.png)



#### 1.1.4 编写跳转逻辑 `LoginController`

```java
@GetMapping("/login-html")
    public String login() {

        return "login";
    }
```





#### 1.1.4  配置安全配置

我们在`WebSecurityConfig` 中修改

```java
    // 配置安全拦截机制
    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http.authorizeRequests()

//                //访问/r/r1资源的 url需要拥有p1权限。
//                .antMatchers("/r/r1").hasAuthority("p1")
//                //访问/r/r2资源的 url需要拥有p2权限。
//                .antMatchers("/r/r2").hasAuthority("p2")
                .antMatchers("/r/**").authenticated()
                .anyRequest().permitAll()
                .and()
                .formLogin()  // (1)
                .loginPage("/")  // (2)
                .loginProcessingUrl("/login-html")  // (3)
                .successForwardUrl("/login‐success") // (4)
                .permitAll(); // (5)
    }
```

步骤介绍

1. 允许表单登陆
2.  指定我们自己的登陆页面, `Spring Security` 以重定向的方式跳转到`/`
3. 指定登陆处理的URL, 也就是用户名、密码表单提交的目的路径
4. 指定登陆成功后的跳转`URL`
5. 我们必须允许所有用户访问我们的登录页面(例如为验证的用户), 这个`formLogin().permitAll() ` 方法允许任务用户访问基于表单登陆的所有`URL`



#### 1.1.5  测试

当用户没有认证的时候,访问系统的资源就会重定向到`/login-html` 页面

![image-20200725174234424](Untitled.assets/image-20200725174234424.png)



当输入账号和密码, 点击登陆的时候，发现报错

![image-20200725174334055](Untitled.assets/image-20200725174334055.png)



**解决:**

`Spring Security` 为防止`CSRF``（Cross-site request forgery跨站请求伪造）` 的发生, 限制了除了`get` 以外的大多数方法. 

**解决方法1:**

屏蔽`CSRF`的控制, 即`Spring Security` 不再限制`CSRF`

配置  `WebSecurityConfig`

```java
  // 配置安全拦截机制
    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http
                .csrf().disable() // //屏蔽CSRF控制，即spring security不再限制CSRF
            
            ....
```



**解决方法2:**

在`login`的页面,添加一个`token`,`Spring Security` 会验证`token`, 如果`token` 通过则可以继续请求. 

`login.html`

```html

    <form action="/login" method="post">
        <input type="hidden"  name="${_csrf.parameterName}"   value="${_csrf.token}"/>
        .....
```



### 1.2 连接数据库认证

前面的例子中我们将用户信息存储在内存中,但是在实际项目中 用户信息是存储在数据库中,本节实现从数据库中读取用户信息,我们只需要重新定义  `UserDetailService` 即可实现根据用户账号查询数据库

#### 1.2.1 数据库准备

1. 新建数据库

    我们这里新建一个名称为`spring-security-oauth2-demo`的数据库, 

2. 新建`t_user` 表

   ```sql
   CREATE TABLE `t_user` (
    `id` bigint(20) NOT NULL COMMENT '用户id',
    `username` varchar(64) NOT NULL,
    `password` varchar(64) NOT NULL,
    `fullname` varchar(255) NOT NULL COMMENT '用户姓名',
    `mobile` varchar(11) DEFAULT NULL COMMENT '手机号',
    PRIMARY KEY (`id`) USING BTREE
   ) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=DYNAMIC
   ```



#### 1.2.2 代码实现

1. `pom` 依赖添加

   ```xml
    <dependency>
               <groupId>mysql</groupId>
               <artifactId>mysql-connector-java</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-jdbc</artifactId>
           </dependency>
   ```

2. 配置文件 `application.yml` 添加数据库相关配置

   ```yaml
   spring:
     datasource:
       url: jdbc:mysql://localhost:3306/spring-security-oauth2-demo
       username: root
       password: rootroot
   
   
   ```

3. 定义实体类

   ```java
   @Builder
   @AllArgsConstructor
   @NoArgsConstructor
   @Data
   public class UserEntity {
   
       private String id;
       private String username;
       private String password;
       private String fullname;
       private String mobile;
   }
   
   ```

4. 定义`UserDao`

   ```java
   @Repository
   public class UserDao {
   
       @Autowired
       private JdbcTemplate jdbcTemplate;
   
   
       /**
        * <p>根据用户名查询用户信息</p>
        *
        * @param userName 用户名
        * @return {@link UserEntity}
        * @author luyanan
        * @since 2020/7/27
        */
       public UserEntity findByUserName(String userName) {
   
           String sql = "select id,username,password,fullname,mobile from t_user where username = ?";
           List<UserEntity> query = jdbcTemplate.query(sql, new Object[]{userName}, new BeanPropertyRowMapper(UserEntity.class));
           return CollectionUtils.isEmpty(query) ? null : query.get(0);
       }
   
   }
   
   ```

   

5. 定义`UserDetailService`

```java
   /**
    * @author luyanan
    * @since 2020/7/24
    * <p>自定义UserDetailsService</p>
    **/
   @Service
   public class ConsumerUserDetailsService implements UserDetailsService {
   
       @Autowired
       private UserDao userDao;
   
       @Override
       public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
   
           // 这里使用静态数据
   //        return User.withUsername("zhangsan").password("$2a$10$Z0lpyLYr3DYWdpRQPr3i5eVx3Q1LolIz7QUDCcq5aFePUh5IahaEa").authorities("p1").build();
           UserEntity userEntity =
                   userDao.findByUserName(s);
           if (null == userEntity) {
               return null;
           }
   
           UserDetails userDetails =
                   User.withUsername(userEntity.getFullname()).password(userEntity.getPassword()).authorities("p1").build();
           return userDetails;
       }
   
   }
```

6. 测试数据添加

   因为我们使用的是` BCryptPasswordEncoder` 密码编码器,所以我们要生成` BCryptPasswordEncoder` 格式的密码

   ```mysql
   INSERT INTO `spring-security-oauth2-demo`.`t_user`(`id`, `username`, `password`, `fullname`, `mobile`) VALUES (1, 'zhangsan', '$2a$10$Z0lpyLYr3DYWdpRQPr3i5eVx3Q1LolIz7QUDCcq5aFePUh5IahaEa', '张三', '18111111111');
   
   ```

   

7. 结果测试

   当我们访问登录页面,输入账号密码的时候, 发现跳转到了

   ![image-20200727143232200](5.Spring%20Security%20%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83%E9%AB%98%E7%BA%A7%E7%AF%87.assets/image-20200727143232200.png)



## 2. 会话

用户认证通过后,为了避免用户的每次操作都进行认证可以将用户的信息保存在会话中,`Spring Security` 提供会话管理,认证通过后将身份信息放入到`SecurityContextHolder` 上下文中,  `SecurityContextHolder` 与当前线程进行绑定,方便获取用户身份. 

### 2.1 获取用户身份

修改`LoginController`.实现 `r/r1` 和`r/r2` 和`/login-success` 接口 返回用户名信息,`Spring Security` 获取当前登录用户信息的方法为: `SecurityContextHolder.getContext().getAuthentication();`

 ```java
@Controller
public class LoginController {


    @GetMapping("/login-html")
    public String login() {

        return "login";
    }

    @ResponseBody
    @GetMapping("/r/r1")
    public String r1() {
        return getUserName() + ":访问r1的资源";
    }

    @ResponseBody
    @GetMapping("/r/r2")
    public String r2() {
        return getUserName() + ":访问r2的资源";
    }

    @ResponseBody
    @RequestMapping(value = "login‐success")
    public String loginSuccess() {
        return getUserName() + ":登陆成功";
    }


    /**
     * <p>获取用户名</p>
     *
     * @return {@link String}
     * @author luyanan
     * @since 2020/7/27
     */
    public String getUserName() {

        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (!authentication.isAuthenticated()) {
            return null;
        }
        Object principal = authentication.getPrincipal();
        String userName = null;
        if (principal instanceof UserDetails) {
            userName = ((UserDetails) principal).getUsername();
        } else {
            userName = principal.toString();
        }
        return userName;
    }
}

 ```

结果测试

![image-20200727144849201](5.Spring%20Security%20%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83%E9%AB%98%E7%BA%A7%E7%AF%87.assets/image-20200727144849201.png)

### 2.2 会话控制

我们可以通过以下选项准确控制会话合适创建以及`Spring Security` 如何与之交互. 

| 机制     | 描述                        |
| -------- | --------------------------- |
| `always` | 如果没有`session`就创建一个 |
| `ifRequired` | 如果需要就创建一个`session`(默认)登录时 |
| `never` | `Spring Security` 将不会创建`session`, 但是如果应用中其他地方创建了`session`, 那么`Spring Security` 将会使用它. |
| `stateless` | `Spring Security` 将绝对不会创建`session`, 也不适用`session` |



通过以下配置方式对该选项进行配置

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {

     http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED);
        
    }
```



默认情况下,`Spring Security` 会为每个登录的用户会新建一个`session`, 就是 `ifRequired`

若选用 `never`, 则指示`Spring Security` 对登录成功的用户不创建`session`了, 但若你的应用程序在某地方新建了`session`, 那么`Spring Security` 还是会用它的. 

若使用`stateless`, 则说明`Spring Security` 对登录的用户不会创建`session`, 你的应用程序也不允许创建`session`,并且他会暗示不使用`cookie`, 所以每个请求都需要重新进行身份验证,这种无状态的架构适用于`REST API` 以及无状态认证机制

**会话超时**

可以在`servlet` 容器中设置`session`的超时时间,如下设置`session`的超时时间为 3600s

`spring boot`的配置文件

```yaml
server:
  servlet:
    session:
      timeout: 3600s
```

`session` 超时之后, 可以通过`Spring Security` 设置跳转的路径

```java
http.sessionManagement()
   .expiredUrl("/login‐success?error=EXPIRED_SESSION")
   .invalidSessionUrl("/login‐success?error=INVALID_SESSION");
```

`expiredUrl` 是指`session` 过期, `invalidSessionUrl` 是指传入的`session` 无效. 

**安全会话`cookie`**

我们可以使用`httpOnly` 和`secure` 标签来保护我们的会话`cookie`

- `httpOnly`: 如果为`true`, 那么浏览器脚本将无法访问`cookie`
- `secure`: 如果为`true`, 则`cookie` 仅通过`HTTPS` 来连接发送

`spring boot` 配置文件

```yaml
server:
  servlet:
    session:
      cookie:
        http-only: true
        secure: true
```



## 3.  退出

`Spring Security` 默认实现了`logout` 退出,访问`/logout`, 退出功能`Spring`  也帮我们做好了. 

我们可以自定义退出成功的页面

```java
          .and()
                .logout()
                .logoutSuccessUrl("/login-html?logout")
                .logoutUrl("/logout"); // (5)
```



当退出操作发生时,将发生

- 使得`Http Session` 无效
- 清除`SecurityContextHolder`
- 跳转到`/login-html?logout` 页面

但是, 类似于配置登陆功能,我们可以进一步自定义退出的功能

```java
       .and()
                .logout()    // (1)
                .logoutSuccessUrl("/login-html?logout") // (2)
                .addLogoutHandler(new LogoutHandler() {// (3)
                    @Override
                    public void logout(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Authentication authentication) {
                        
                    }
                })
                .logoutUrl("/logout")  // (4)
                .addLogoutHandler(new LogoutHandler() { // (5)
                    @Override
                    public void logout(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Authentication authentication) {

                    }
                }).invalidateHttpSession(true); // (6)
```

1. 提供系统退出的支持,使用 `WebSecurityConfigurerAdapter` 会自动被应用
2. 设置触发退出操作的`URL`,默认是`/logout`
3. 退出之后跳转到`URL`,默认是`/login?logout`
4. 定制的`LogoutSuccessHandler` 用于实现用户退出成功的时候的处理,如果指定了这个选项, 那么`logoutSuccessUrl()` 的设置会被忽略.
5. 添加一个`LogoutHandler` , 用于实现用户退出的时候的清理工作,默认 `SecurityContextLogoutHandler` 会被添加为最后一个`LogoutHandler` 
6. 指定是否在退出的时候让`HttpSession` 无效,默认设置为`true`

注意: 如果让`logout` 在`GET` 请求下生效,必须关闭防止`CSRF` 攻击`csrf().disable()`,如果开启了`CSRF`, 必须使用`post` 方法请求`/logout`

`logoutHandler：`

一般来说,`logoutHandler` 的实现类被用来执行必要的清理,因而他们不应该抛出异常 

下面是`Spring Security` 提供的一些实现

-  `PersistentTokenBasedRememberMeServices`: 基于持久化`token`的`RememberMe` 功能的相关清理
- `TokenBasedRememberMeService`: 基于`token` 的`RememberMe`  功能的清理
- `CookieClearingLogoutHandler`: 退出时`Cookie`的相关清理
- `CsrfLogoutHandler`: 负责在退出时移除`csrfToken`
- `SecurityContextLogoutHandler`: 退出时`SecurityContext`的相关清理

链式`API`提供了调用相应的`logoutHandler` 实现的快捷方式, 比如`deleteCookies()`

##  4. 授权

### 4.1 概述

授权的方式包括`web` 授权和方法授权,`web` 授权是通过	`url` 拦截进行授权,方法授权是通过方法拦截进行授权, 他们都会调用`accessDecisionManage` 进行授权决策,若为`web` 授权则拦截器为`FilterSecurityInterceptor`; 若为方法授权则拦截器为`MethodSecurityInterceptor`. 如果同时通过`web` 授权和方法授权则先执行`web` 授权, 在执行方法授权, 最后决策通过, 则允许访问资源, 否则将禁止访问. 

类关系如下：

![image-20200727210236738](5.Spring%20Security%20%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83%E9%AB%98%E7%BA%A7%E7%AF%87.assets/image-20200727210236738.png)

### 4.2 环境准备

#### 4.2.1 数据库环境准备

#### 角色表

```mysql
CREATE TABLE `t_role` (
`id` varchar(32) NOT NULL,
`role_name` varchar(255) DEFAULT NULL,
`description` varchar(255) DEFAULT NULL,
`create_time` datetime DEFAULT NULL,
`update_time` datetime DEFAULT NULL,
`status` char(1) NOT NULL,
PRIMARY KEY (`id`),
UNIQUE KEY `unique_role_name` (`role_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8

insert into `t_role`(`id`,`role_name`,`description`,`create_time`,`update_time`,`status`) values
('1','管理员',NULL,NULL,NULL,'');
```

##### 角色用户关系表

```mysql
CREATE TABLE `t_user_role` (
`user_id` varchar(32) NOT NULL,
`role_id` varchar(32) NOT NULL,
`create_time` datetime DEFAULT NULL,
`creator` varchar(255) DEFAULT NULL,
PRIMARY KEY (`user_id`,`role_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8

insert into `t_user_role`(`user_id`,`role_id`,`create_time`,`creator`) values
('1','1',NULL,NULL);

```



##### 权限表

```mysql
CREATE TABLE `t_permission` (
`id` varchar(32) NOT NULL,
`code` varchar(32) NOT NULL COMMENT '权限标识符',
`description` varchar(64) DEFAULT NULL COMMENT '描述',
`url` varchar(128) DEFAULT NULL COMMENT '请求地址',
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8

insert into `t_permission`(`id`,`code`,`description`,`url`) values ('1','p1','测试资源
1','/r/r1'),('2','p3','测试资源2','/r/r2');
```



##### 角色权限表

```mysql
CREATE TABLE `t_role_permission` (
`role_id` varchar(32) NOT NULL,
`permission_id` varchar(32) NOT NULL,
PRIMARY KEY (`role_id`,`permission_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8

insert into `t_role_permission`(`role_id`,`permission_id`) values ('1','1'),('1','2');
```

####  4.2.2  修改`UserDetailsService`

##### 修改`Dao`  接口

1. 修改`dao` 接口

   在`UserDao` 中添加:

   ```java
   
       /**
        * <p>根据用户id查询权限列表</p>
        *
        * @param userId
        * @return {@link List< String>}
        * @author luyanan
        * @since 2020/7/27
        */
       public List<String> findPermissionByUserId(String userId) {
   
           String sql = "SELECT * FROM t_permission WHERE id IN(\n" +
                   "SELECT permission_id FROM t_role_permission WHERE role_id IN(\n" +
                   "\tSELECT role_id FROM t_user_role WHERE user_id = ? \n" +
                   ")\n" +
                   ")";
           List<PermissionEntity> query = jdbcTemplate.query(sql, new Object[]{userId}, new BeanPropertyRowMapper<>(PermissionEntity.class));
           return query.stream().map(PermissionEntity::getCode).collect(Collectors.toList());
   
       }
   ```

2. 修改`UserDetailsService`

    实现从数据库中读取权限

   ```java
   @Service
   public class ConsumerUserDetailsService implements UserDetailsService {
   
       @Autowired
       private UserDao userDao;
   
       @Override
       public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
   
           // 这里使用静态数据
   //        return User.withUsername("zhangsan").password("$2a$10$Z0lpyLYr3DYWdpRQPr3i5eVx3Q1LolIz7QUDCcq5aFePUh5IahaEa").authorities("p1").build();
           UserEntity userEntity =
                   userDao.findByUserName(s);
           if (null == userEntity) {
               return null;
           }
           List<String> permission = userDao.findPermissionByUserId(userEntity.getId());
   
           UserDetails userDetails =
                   User
                           .withUsername(userEntity.getFullname())
                           .password(userEntity.getPassword())
                           .authorities(permission.toArray(new String[permission.size()]))
                           .build();
           return userDetails;
       }
   
   }
   
   ```

   

### 4.3 `web` 授权

在上面的例子中, 我们完成了认证拦截,并对`r/**` 下的资源进行了简单的授权保护,但是我们想要灵活的授权控制该怎么做呢? 通过给`http.authorizeRequests()` 添加多个子节点来定制需求到我们的`URL`, 如下所示： 

```java
 protected void configure(HttpSecurity http) throws Exception {


        http
                .csrf().disable() // //屏蔽CSRF控制，即spring security不再限制CSRF


                .authorizeRequests()   //（1）

                //访问/r/r1资源的 url需要拥有p1权限。
                .antMatchers("/r/r1").hasAuthority("p1")//（2）
                //访问/r/r2资源的 url需要拥有p2权限。
                .antMatchers("/r/r2").hasAuthority("p2")//（3）
                .antMatchers("/r/r3").access("hasAuthority('p1') and hasAuthority('p2')")//（4）
                .antMatchers("/r/**").authenticated()//（5）
                .anyRequest().permitAll()//（6）
                .and()
```

1. `http.authorizeRequests()` 方法有多个子节点, 每个`mathcer` 按照他们声明的顺序进行
2. 指定`r/r1` 拥有`p1`的权限才可以访问
3. 指定`r/r2` 拥有`p2` 的权限才可以访问
4. 指定`r/r3` 同时拥有`p1`和`p2`的权限才可以访问
5. 指定了处`r/r1`、`r/r2`、`r/r3` 之外的`r/r**` 资源, 同时通过身份认证就能过够访问
6. 剩余的尚不匹配的资源,不做保护。

**注意:**

规则的顺序是最重要的,更具体的规则应该先写,现在以`/admin` 开始是所有内容都需要具体也有`ADMIN`角色的身份验证用户,即使是`admin/login`  路径,(因为`/admin/login` 已经被`/admin/**` 规则匹配, 所以第二个规则被忽略)

```java
.antMatchers("/admin/**").hasRole("ADMIN")
.antMatchers("/admin/login").permitAll()

```



因此登陆页面的规则应该是在`/admin/**`  规则之前,例如:

```java
.antMatchers("/admin/login").permitAll()
.antMatchers("/admin/**").hasRole("ADMIN")
```

保护`URL` 常用的方法有:

-  `authenticated()`: 保护`URL`, 需要用户登陆
- `permitAll()`: 指定`URL` 无需保护,一般应用与静态资源文件
- `hasRole(String role)` 限制单个角色访问,角色将会被增加`ROLE`, 所以`ADMIN` 将和`ROLE_ADMIN` 进行比较.
- `hasAuthority(String authority)` 限制单个权限访问
- `hasAnyRole(String… roles)`: 允许多个角色访问
- `hasAnyAuthority(String… authorities)` 允许多个权限访问
- `access(String attribute)`: 该方法使用`SPEL`表达式 ,所以可以创建复杂的限制
- `hasIpAddress(String ipaddressExpression)`: 限制地址或者子网

### 4.3  方法授权

现在我们已经掌握了如何使用` http.authorizeRequests()`对`web` 资源进行授权保护,从`Spring Security2.0` 版本开始,它支持服务层方法的安全性支持,本节学习`@PreAuthorize`,`@PostAuthorize`, `@Secured`  三类注解

我们可以在任何`@Configuration`实例上使用`@EnableGlobalMethodSecurity` 注释来启动基于注册的安全性. 

以下内容将启动`Spring Security` 的`@Secured` 注释

```java
@EnableGlobalMethodSecurity(securedEnabled = true)
```

然后向方法(类或者接口) 上添加注解就hi限制对方法的访问,`Spring Security` 的原生注释支持为该方法定义了一组属性,这些将被传递给`AccessDecisionManage` 以供它做出实际的决定.

```java
public interface IUserService {


    @Secured("IS_AUTHENTICATED_ANONYMOUSLY")
    UserEntity findById(String id);

    @Secured("IS_AUTHENTICATED_ANONYMOUSLY")
    List<UserEntity> findByPage(int page);

    @Secured("ROLE_TELLER")
    void save(UserEntity userEntity);
}

```

以上配置表明`findById`、`findByPage` 方法可匿名访问,底层使用`WebExpressionVote` 投票器,可从`AffirmativeBased` 第23 行代码进行跟踪

`save` 方法需要`TELLER` 角色才访问,底层使用`RoleVoter` 投票器

使用如下代码可启用`prePost` 注解的支持

```java
@EnableGlobalMethodSecurity(prePostEnabled = true)
```

相应的代码如下:

```java
   @PreAuthorize("isAnonymous()")
    UserEntity findById(String id);

    @PreAuthorize("isAnonymous()")
    List<UserEntity> findByPage(int page);

    @PreAuthorize("hasAuthority('findById') and hasAuthority('findByPage')")
    void save(UserEntity userEntity);
```

以上配置表明 `findById` 和`findByPage` 方法可以匿名访问,`save` 方法需要同时拥有`findById` 和`findByPage`的权限才可以访问,底层使用`WebExpressionVote` 投票器,可从**`AffirmativeBased`** 第23行代码

