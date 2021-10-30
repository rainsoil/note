#  基于Session进行认证



## 1. 认证流程

用户认证成功后,在服务端生成用户相关的数据, 并保存在`session` 会话中,而发给客户端的`session_id` 保存在浏览器的`cookie`中, 当客户端请求的时候带上`session_id` ,服务端就可以通过验证`session_id`是否存在和`session` 数据来完成用户的合法性校验了. 当用户退出系统或者`session` 销毁的时候, 只需要清空会话的`session` 即可. 这样客户端的`session_id`也就无效了. 

基于`session`的认证机制是由`Servlet` 规范定制的,`servlet`已经实现,用户可以通过`HttpSession` 的操作方法就可以实现,如下是`HttpSession` 相关的操作API

| 方法                                          | 含义                    |
| --------------------------------------------- | ----------------------- |
| `HttpSession getSession(Boolean create)`      | 获取当前的`session`对象 |
| `void setAttribute(String name,Object value)` | 往`session` 中存放数据  |
| `object getAttribute(String name)`            | 从`session`中获取数据   |
| `void removeAttribute(String name);`          | 移除`session`对象       |
| `void invalidate()`                           | 使得`HttpSession`失效   |



## 2. 案例演示

本案例使用`Maven` 和`SpringBoot` 进行实现

**pom.xml依赖**

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

    </dependencies>

```



启动类`SecurityApplication`

```java
/**
 * @author luyanan
 * @since 2020/7/22
 * <p>启动类</p>
 **/
@SpringBootApplication
public class SecurityApplication {
    public static void main(String[] args) {
        SpringApplication.run(SecurityApplication.class, args);
    }

}

```

**控制层`UserController`**,这里使用的是假数据做的登录

```java

/**
 * @author luyanan
 * @since 2020/7/22
 * <p>用户的控制层和</p>
 **/
@Controller
public class UserController {


    static Map<String, String> users = new HashMap<>();

    static {

        users.put("zhangsan", "123456");
        users.put("lisi", "234567");
    }


    @Autowired
    private HttpSession httpSession;


    /**
     * <p>登陆页面</p>
     *
     * @return {@link String}
     * @author luyanan
     * @since 2020/7/22
     */
    @GetMapping("login")
    public String index() {
        return "login";
    }

    /**
     * <p>登陆接口</p>
     *
     * @param userName
     * @param passWord
     * @return {@link String}
     * @author luyanan
     * @since 2020/7/22
     */
    @ResponseBody
    @PostMapping("login")
    public String login(String userName, String passWord) {

        Assert.hasText(userName, "用户名为空");
        Assert.hasText(passWord, "密码为空");
        String s = users.get(userName);
        if (null == s) {
            throw new RuntimeException("用户不存在");
        }
        if (!passWord.equals(s)) {
            throw new RuntimeException("密码不正确");
        }
        httpSession.setAttribute("name", userName);
        return userName + ":登陆成功";
    }


    /**
     * <p>个人详情</p>
     *
     * @return {@link String}
     * @author luyanan
     * @since 2020/7/22
     */
    @ResponseBody
    @GetMapping("info")
    public String info() {
        Object name = httpSession.getAttribute("name");
        return Optional.ofNullable(name).orElse("匿名") + ":访问info";
    }


    /**
     * <p>注销</p>
     *
     * @return {@link String}
     * @author luyanan
     * @since 2020/7/22
     */

    @ResponseBody
    @GetMapping("logout")
    public void logout() {
        httpSession.invalidate();
    }
}

```

里面编写了四个接口,分别为跳转登陆页面、登陆、用户信息(当用户登陆的时候,则会返回用户名称+访问,否则为匿名访问)、注销接口 

**application.yml**

```yaml
spring:
  application:
    name: session
  thymeleaf:
    mode: HTML
```

登陆页面

**login.html**

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
        <input type="text" name="userName" placeholder="用户名">

        <input type="password" name="passWord" placeholder="密码">
        <button type="submit">登陆</button>

    </form>

</div>
</body>
</html>

```



最后的目录结构为 :

![image-20200722204342319](2.%20%E5%9F%BA%E4%BA%8ESession%E8%BF%9B%E8%A1%8C%E8%AE%A4%E8%AF%81.assets/image-20200722204342319.png)



启动项目, 访问路径进行测试

> 不登陆的情况

当我们直接访问`/info`接口的时候

![image-20200722204450244](2.%20%E5%9F%BA%E4%BA%8ESession%E8%BF%9B%E8%A1%8C%E8%AE%A4%E8%AF%81.assets/image-20200722204450244.png)

发现为匿名用户访问,说明用户没有登陆

> 登陆测试

我们访问`/login` 页面

![image-20200722204604859](2.%20%E5%9F%BA%E4%BA%8ESession%E8%BF%9B%E8%A1%8C%E8%AE%A4%E8%AF%81.assets/image-20200722204604859.png)

输入账号密码之后,发现

![image-20200722204620683](2.%20%E5%9F%BA%E4%BA%8ESession%E8%BF%9B%E8%A1%8C%E8%AE%A4%E8%AF%81.assets/image-20200722204620683.png)

显示张三登陆成功,再次访问`/info` 

![image-20200722204647049](2.%20%E5%9F%BA%E4%BA%8ESession%E8%BF%9B%E8%A1%8C%E8%AE%A4%E8%AF%81.assets/image-20200722204647049.png)

发现我们已经获取到登陆用户的名称, 然后调用注销的接口`/logout`

再次访问`/info` 发现又变成了匿名用户登陆了,这就完成了基于`session`的认证



