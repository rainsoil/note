# Spring Security的工作原理

## 1 结构总览

`Spring Security` 所解决的问题就是`安全访问控制`, 而安全访问控制其实就是对所有进入系统的请求进行拦截,校验每个请求是否能够访问它所期望的资源,我们可以通过`Filter` 或者`AOP` 等技术进行实现,`Spring Security` 对`Web` 资源的保护是通过`Filter` 实现的,所以从这个`Filter` 来入手,逐步深入`Spring Security` 的原理.

当初始化`Spring Security`的时候, 会创建一个名为`org.springframework.security.config.annotation.web.configuration.WebSecurityConfiguration#springSecurityFilterChain`的`Servlet` 过滤器, 类型为`org.springframework.security.web.FilterChainProxy`,它实现了`javax.servlet.Filter`, 因此外部的请求会经过此类,下图是`Spring Security` 的过滤器链结构图

![image-20200724150042320](4.Spring%20Security%E7%9A%84%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.assets/image-20200724150042320.png)



`FilterChainProxy`是一个代理, 真正起到作用的是`FilterChainProxy` 中的`SecurityFilterChain` 中包含的各个`Filter`,  同时这些`Filter` 被`Spring` 管理, 他们是`Spring Security`的核心, 各有各的职责,但是他们并不直接处理用户的认证,也不直接处理用户的授权,而是把他们交给了认证管理器`AuthenticationManager`和决策管理器`AccessDecisionManager` 进行处理, 下图是`FilterChainProxy` 相关类的`UML` 图示

![image-20200724150053640](4.Spring%20Security%E7%9A%84%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.assets/image-20200724150053640.png)

`Spring Security` 功能的实现主要是由一系列的过滤器链相互配合完成. 

![image-20200724150100955](4.Spring%20Security%E7%9A%84%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.assets/image-20200724150100955.png)

下面介绍过滤器链中几个主要的过滤器以及作用

1. `SecurityContextPersistenceFilter ` 这个`Filter` 是整个拦截过程的入口和出口(也就是第一个拦截和最后一个拦截器), 会在请求开始时从配置好的`SecurityContextRepository ` 中获取`SecurityContext`, 然后把它设置给`SecurityContextHolder`. 在请求完成后将`SecurityContextHolder ` 持有的`SecurityContext ` 再保存到配置好的`SecurityContextRepository` ,同时清除 `securityContextHolder ` 所持有的`SecurityContext`. 
2. `UsernamePasswordAuthenticationFilter ` 用于处理来自表单提交的认证, 该表单必须提供对应的用户名和密码,其内部还有登陆成功或者失败后进行处理的`AuthenticationSuccessHandler ` 和 `AuthenticationFailureHandler`,这些都可以根据需求进行做出相应的改变. 
3. `FilterSecurityInterceptor ` 是用于保护`web` 资源的,使用`AccessDecisionManager` 对当前用户进行授权访问,全面已经详细介绍过了. 
4. `ExceptionTranslationFilter ` 能够捕获来自`FilterChain` 的所有的异常,并进行处理. 但是他只会处理两类异常: `AuthenticationException ` 和`AccessDeniedException`, 其他的异常会继续抛出. 



## 2. 认证流程

### 2.1 认证流程

![image-20200724150110004](4.Spring%20Security%E7%9A%84%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.assets/image-20200724150110004.png)

让我们仔细的分析一下认证过程

1. 用户提交用户名、密码被`SecurityFilterChain` 中的`UsernamePasswordAuthenticationFilter ` 过滤器获取到, 封装为请求`Authentication`, 通常情况下是`UsernamePasswordAuthenticationToken` 这个实现类. 
2. 然后过滤器将`Authentication` 提交至认证管理器`AuthenticationManager` 进行认证
3. 认证成功后,`AuthenticationManager` 身份管理器返回一个被填充满了信息的(包括上面提到的权限信息、身份信息、细节信息、但密码通常会被移除)的`Authentication ` 实例
4. `SecurityContextHolder ` 安全上下文容器将第3步填充了信息的`Authentication `  实例,通过`SecurityContextHolder.getContext().setAuthentication(…)` 方法设置到其中. 可以看出`AuthenticationManager` 接口(认证管理器)是认证相关的核心接口,也是发起认证的出发点,他的实现类为`ProviderManager`. 而`Spring Security` 支持多种认证方式,因为`ProviderManager` 维护着一个`List<AuthenticationProvider>` 列表,存放多种认证方式,最终实际的认证工作是由`AuthenticationProvider` 完成的. 咱们知道`web` 表单的对应的`AuthenticationProvider` 实现类为`DaoAuthenticationProvider`,他的内部维护着一个 `UserDetailsService` 负责`UserDetails`的获取, 最终`AuthenticationProvider` 把`UserDetails` 填充至`Authentication`. 

认证核心组件的大体关系如下: 

![image-20200724154239540](4.Spring%20Security%E7%9A%84%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.assets/image-20200724154239540.png)



### 2.2  `AuthenticationProvider`

通过前面的`Spring Security` 认证流程我们知道,认证管理器`AuthenticationManager` 委托`AuthenticationProvider` 完成认证工作. 

```java
public interface AuthenticationProvider {
    Authentication authenticate(Authentication var1) throws AuthenticationException;

    boolean supports(Class<?> var1);
}

```

`authenticate` 方法定义了认证的实现过程,他的参数是一个`Authentication`, 里面包含了登陆用户所提交的用户、密码等. 而返回值也是一个`Authentication`, 这个`Authentication`是在认证成功后,将用户的权限以及其他信息重新组装后生成的. 

` Spring Security` 中维护着一个`List<AuthenticationProvider>` 列表, 存放多种认证方式, 不同的认证方式使用不同的`AuthenticationProvider`. 如使用用户名密码的登陆的时候,会使用`AuthenticationProvider1`, 使用短信登陆的时候使用`AuthenticationProvider2` 等等这样的例子. 

每个`AuthenticationProvider` 需要实现`supports` 方法来表明自己支持的认证方式,如我们使用表单方式认证,在提交请求的时候,`Spring Security` 会生成`UsernamePasswordAuthenticationToken`, 它是一个`Authentication`,它里面封装着用户提交的用户名、密码信息, 而对应的哪个`AuthenticationProvider` 来处理它? 

我们在`DaoAuthenticationProvider` 的基类 `AbstractUserDetailsAuthenticationProvider` 发现以下代码,

```java
 public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }
```

也就是说, 当`web` 表单提交用户名、密码的时候,`Spring Security` 由`DaoAuthenticationProvider` 处理. 

最后我们来看一下`Authentication`(认证信息)的结构, 它是一个接口,我们之前提到的`UsernamePasswordAuthenticationToken` 就是它的实现之一. 

```java
public interface Authentication extends Principal, Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();

    Object getCredentials();

    Object getDetails();

    Object getPrincipal();

    boolean isAuthenticated();

    void setAuthenticated(boolean var1) throws IllegalArgumentException;
}

```

1. `Authentication` 是`Spring Security` 包中的接口, 直接继承自`Principal` 类,而`Principal` 是`java.security.Principal` 包中的,它是表示一个抽象的主体身份,任何主体都有一个名称,所以包含一个`getName` 方法. 
2.   `getAuthorities` 权限信息列表,默认是`GrantedAuthority` 接口的一些实现类,通常是代表权限信息的一系列字符串. 
3. `getCredentials` 凭证信息,用户输入的密码字符串,在认证过后通常会被移除,用户保证安全. 
4. `getDetails()`,  细节信息,`web` 应用中的实现接口通常为 `WebAuthenticationDetails`,它记录了访问者的ip地址和`sessionId` 信息. 
5. `getPrincipal()` 身份信息,大部分情况下返回的是`UserDetails` 接口的实现类,`UserDetail` 代表用户的详细信息,拿从`Authentication` 中取出来的`UserDetails` 就是当前登陆的用户信息,它也是框架中常用的接口之一. 



### 2.3 `UserDetailsService`

#### 2.3.1 认识`UserDetailsService`

我们知道`DaoAuthenticationProvider` 处理了`web` 表单的认证逻辑,认证成功后得到一个 `Authentication(UsernamePasswordAuthenticationToken实现)`,里面包含了身份信息`(Principal)`.这个身份信息是一个`Object`,  大多数情况下它可以被强转为`UserDetails` 对象. 

`DaoAuthenticationProvider`  中包含了一个`UserDetailsService`实例, 它负责根据用户名提出用户信息`UserDetail`(包含密码), 而后`DaoAuthenticationProvide` 会去对比`UserDetailsService` 提取的用户密码与用户提交的密码是否匹配作为认证成功的依据,因此可以通过将自定义的`UserDetailsService` 作为`Spring Bean` 来自定义身份验证. 

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String var1) throws UsernameNotFoundException;
}

```



很多人把 `把DaoAuthenticationProvider` 和`UserDetailsService` 的职责搞混淆, 其实 `UserDetailsService` 只负责从特定的地方(通常是数据库)加载用户信息,仅此而已. 而 `DaoAuthenticationProvider` 的职责更大,它完成完整的认证流程, 并且会把`UserDetails` 填充到`Authentication`. 

上面一直提到的`UserDetails` 是用户信息

```java
public interface UserDetails extends Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();

    String getPassword();

    String getUsername();

    boolean isAccountNonExpired();

    boolean isAccountNonLocked();

    boolean isCredentialsNonExpired();

    boolean isEnabled();
}

```







它和`Authentication` 接口很类似,比如他们都拥有`username`、`authorities`。 `Authentication` 的`getCredentials()` 与`UserDetails` 中的`getPassword()` 需要被区别对待,前者是用户提交的密码凭证,后者是用户实际存储的密码,认证其实就是这两者的对比. `Authentication` 中的`getAuthorties()` 实际上的由`UserDetails` 的`getAuthories()` 传递而形成的. 还记得`Authentication`  接口中的`getDetails()` 方法吗?  其中的`UserDetails` 用户详细信息便是经过了`AuthenticationProvider` 认证之后被填充的. 

通过实现`UserDetailsService` 和`UserDetails`,我们可以完成对用户信息以及用户信息字段的扩展. 

`Spring Security`  提供的 `InMemoryUserDetailsManager`(内存认证), `JdbcUserDetailsManager(jdbc认证)` 就是`UserDetailsService` 的实现类,主要区别无非是从内存还是从数据库加载用户. 

###  2.4 测试

**自定义`UserDetailsService`**

```java
@Service
public class ConsumerUserDetailsService implements UserDetailsService {


    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {

        // 这里使用静态数据
        return User.withUsername("zhangsan").password("123456").authorities("p1").build();
    }
}

```



###  2.5 `PasswordEncoder`

#### 2.5.1  认识`PasswordEncoder`

`DaoAuthenticationProvider` 认证处理器 通过`UserDetailsService` 获取到`UserDetails` 后, 它是如何与请求`Authentication` 中的密码做对比呢? 

`Spring Security` 为了适应多种多样的加密类型, 做了抽象,`DaoAuthenticationProvider` 通过`PasswordEncoder` 接口的`matchers` 方法进行密码的对比,而具体的密码对比细节取决于实现. 

```java
public interface PasswordEncoder {
    String encode(CharSequence var1);

    boolean matches(CharSequence var1, String var2);

    default boolean upgradeEncoding(String encodedPassword) {
        return false;
    }
}

```

而`Spring Security` 内置了很多的`PasswordEncoder`,能够开箱即用,使用某种`PasswordEncoder` 只需要进行如下配置即可

```java
    // 密码编码器
    @Bean
    public PasswordEncoder passwordEncoder() {
        return NoOpPasswordEncoder.getInstance();
    }
```

`NoOpPasswordEncoder` 采用字符串匹配法, 不对密码做加密比较处理. 密码比较流程如下： 

1. 用户输入密码(明文)
2. `DaoAuthencationProvider` 获取`UserDetails`(其中存储了用户的正确密码)
3. `DaoAuthencationProvider`  使用`passwordEncoder` 对输入的密码和正确的密码进行校验,密码一致则通过, 否则则校验失败. 

` NoOpPasswordEncoder` 的校验规则就是那输入的用户和`UserDetails` 中的正确的密码进行字符串对比,字符串内容一致则校验通过, 否则校验失败. 

实际项目中, 推荐使用 `BCryptPasswordEncoder`, `Pbkdf2PasswordEncoder`, `SCryptPasswordEncoder` 等. 

####  2.5.2 使用 `BCryptPasswordEncoder`

1. 配置  `BCryptPasswordEncoder`

   ```java
    @Bean
       public PasswordEncoder passwordEncoder() {
           return new BCryptPasswordEncoder();
       }
   ```

2. 生成 `BCrypt` 格式的密码

    添加依赖
    
    ```xml
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-test</artifactId>
            </dependency>
    ```
    
    测试代码
    
    ```java
       @Test
        public void test() {
            String hashpw = BCrypt.hashpw("123", BCrypt.gensalt());
            System.out.println(hashpw);
            //校验原始密码和BCrypt 是否一致
            System.out.println(BCrypt.checkpw("123", hashpw));
        }
    ```
    
    结果
    
    ```text
    $2a$10$Z0lpyLYr3DYWdpRQPr3i5eVx3Q1LolIz7QUDCcq5aFePUh5IahaEa
    true
    ```
    
    
    
3. 修改安全配置类

     将`UserDetaiuls` 中的原始密码修改为`BCrypt` 格式的

    ```java
        // 配置用户信息
        @Bean
        public UserDetailsService userDetailsService() {
            InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
            manager.createUser(User.withUsername("zhangsan").password("$2a$10$Z0lpyLYr3DYWdpRQPr3i5eVx3Q1LolIz7QUDCcq5aFePUh5IahaEa").authorities("p1").build());
            manager.createUser(User.withUsername("lisi").password("$2a$10$Z0lpyLYr3DYWdpRQPr3i5eVx3Q1LolIz7QUDCcq5aFePUh5IahaEa").authorities("p2").build());
            return manager;
        }
    ```

在我们的实际项目中, 数据库中存储的密码并不是原始密码,都是经过加密处理的. 



## 3. 授权流程

### 3.1 授权流程

我们知道,`Spring Security` 可以通过`http.authorizeRequests()` 对`web` 请求进行授权保护,`Spring Security` 使用标准的`Filter` 建立了对`web` 请求的拦截,最终实现对资源的授权访问. 

`Spring Security` 的授权流程如下: 

![image-20200725153931710](4.Spring%20Security%E7%9A%84%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.assets/image-20200725153931710.png)

   分析授权流程:

1. **拦截请求:** 已认证用户访问受保护的`web` 资源将被`SecurityFilterChain` 中的`FilterSecurityInterceptor`子类拦截, 

2. **获取资源访问策略**:  `FilterSecurityInterceptor` 会从 `SecurityMetadataSource `的子类`DefaultFilterInvocationSecurityMetadataSource` 获取要访问当前资源所需要的权限`Collection<ConfigAttribute>`.

   `SecurityMetadataSource`  其实就是读取访问策略的抽象,而读取的内容,其实就是我们配置的访问规则,读取访问策略如:

   ```java
    http.authorizeRequests()
   
                   //访问/r/r1资源的 url需要拥有p1权限。
                   .antMatchers("/r/r1").hasAuthority("p1")
                   //访问/r/r2资源的 url需要拥有p2权限。
                   .antMatchers("/r/r2").hasAuthority("p2")
                   .antMatchers("/r/**").authenticated()
        ......
   ```

3. 最后, `FilterSecurityInterceptor` 会调用`AccessDecisionManager` 进行授权决策,若决策通过,则允许访问资源,否则不允许. 

   `AccessDecisionManager`(访问决策管理器) 的核心接口如下: 

   ```java
   public interface AccessDecisionManager {
       void decide(Authentication var1, Object var2, Collection<ConfigAttribute> var3) throws AccessDeniedException, InsufficientAuthenticationException;
   
       boolean supports(ConfigAttribute var1);
   
       boolean supports(Class<?> var1);
   }
   
   ```

    这里着重说一下`decide`的参数

   - `authentication`: 要访问资源的访问者的身份
   - `object`:  要访问的受保护资源,`web` 请求对应的`FilterInvocation`
   - `configAttributes`: 是受保护资源的访问策略,通过`SercurotyMetadataSource` 获取

   `decide` 接口就是用来鉴定当前用户是否有访问对应受保护资源的权限的. 



### 3.2  授权决策

`AccessDecisionManager` 采用投票的方式来确定是否能够访问受保护资源. 

![image-20200725155742080](4.Spring%20Security%E7%9A%84%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.assets/image-20200725155742080.png)

通过上图可以看出,`AccessDecisionManager` 包含了一系列`AccessDecisionVoter` 将会被用来对`Authencation` 是否有权访问受保护对象进行投票,`AccessDecisionManager` 根据投票的结果,做出最终的决策. 

 `AccessDecisionVoter`  是一个接口,其中定义了三个方法,具体结构如下所示: 

```java
public interface AccessDecisionVoter<S> {
    int ACCESS_GRANTED = 1;
    int ACCESS_ABSTAIN = 0;
    int ACCESS_DENIED = -1;

    boolean supports(ConfigAttribute var1);

    boolean supports(Class<?> var1);

    int vote(Authentication var1, S var2, Collection<ConfigAttribute> var3);
}
```

` vote()` 方法的返回结果会是`AccessDecisionVoter`   定义好的三个常量之一,`ACCESS_GRANTED` 表示同意,`ACCESS_DENIED` 表示拒绝, `ACCESS_ABSTAIN` 表示弃权. 如果一个 `AccessDecisionVoter`   不能判断当前`Authentication` 是否拥有了访问受保护对象的权限,则其`vote()` 方法的返回值应当为 弃权. 

`Spring Security` 内置了三个基于投票的  `AccessDecisionManage` 实现类如下,他们分别是 `AffirmativeBased`、`ConsensusBased`和`UnanimousBased`.

**`AffirmativeBased`**的逻辑是:

1. 只要有`AccessDecisionVoter`的投票结果为 `ACCESS_GRANTED`,则同意用户进行访问
2. 如果全部弃权也表示通过
3. 如果没有一个人投赞成票,但是有人投反对票, 则将抛出 `AccessDeniedException`. 

`Spring Security` 默认使用的是 **`AffirmativeBased`**

**ConsensusBased**的逻辑是:

1. 如果赞成票多于反对票则表示通过. 
2. 反过来, 如果反对票多于赞同票则将抛出 `AccessDeniedException`
3. 如果赞同票和反对票相同且不等于0, 并且属性  `allowIfEqualGrantedDeniedDecisions`的值为`true`. 则表示通过, 否则将抛出`AccessDeniedException`, 参数 `allowIfEqualGrantedDeniedDecisions`的默认值为`true`. 
4. 如果所有的`AccessDecisionVoter` 都弃权了, 则将视参数 `allowIfAllAbstainDecisions`的值而定, 如果该值为`true` 则表示通过,否则将抛出异常`AccessDeniedExeception`. 参数 `allowIfAllAbstainDecisions` 的值默认为`false`. 

` UnanimousBased`的逻辑与另外两种实现有种不太一样,另外两种 会一次性把受保护的对象的配置属性全部传递给`AccessDecisionVoter`  进行投票, 而` UnanimousBased` 一次只会传递一个 `ConfigAttribute` 给`AccessDecisionVoter`   进行投票 , 也就是意味着 我们的`AccessDecisionVoter`  `的逻辑是只要传递进来的 ``ConfigAttribute` 中有一个能够匹配则投赞成票,但是放到 ` `UnanimousBased`  中其投票结果就不一定是赞成了,

` UnanimousBased`的逻辑具体来说是这样的: 

1. 如果受保护的对象配置的某一个 `ConfigAttribute`  被任意的 `AccessDecisionVoter`   反对了,则 抛出`AccessDeniedException`.
2. 如果没有反对票,但是有赞成票,则通过. 
3. 如果全部弃权了, 则将视参数  `allowIfAllAbstainDecisions`的值而定,如果为`true` 则表示通过,`false` 则抛出`AccessDeniedException`.

`Spring Security`也内置一些投票者实现类如`RoleVoter`、`AuthenticatedVoter`和`WebExpressionVoter`等，可以 自行查阅资料进行学习





