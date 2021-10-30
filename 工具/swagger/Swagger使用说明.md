# Swagger的基本使用

## Swagger 注解说明

springfox 把对接口的描述都通过注解的方式来完成, 主要包括以下几种注解: 
1. 数据模型注解 
   - @ApiModel: 实体描述
   - @ApiModelProperty: 实体属性描述
2. 接口注解
   - @ApiOperation:接口描述
   - @ApiIgnore 忽略此接口
3. 控制器注解
   
   - @Api: 控制器描述
4. 输入参数注解
   - @ApiImplicitParams: 接口多个参数
   - @ApiImplicitParam: 接口单个参数
   - @ApiParam 单个参数描述
5. 响应数据注解
   - @ResponseHeader: 响应值header
   - @ApiResponses: 响应值集
   - @ApiResponse : 单个响应值
   
   
###  1.数据模型注解说明

开发过程中,对于MVC模式的接口,使用M(Model)进行数据通信,因此, 需要对此数据模型进行有效的描述。 对应的注解有: 

- @ApiModel: 实体描述
- @ApiModelProperty: 实体字段描述



####  1. @ApiModel 注解说明

| 属性名称       | 数据类型        | 默认值     | 描述                                                         |
| -------------- | --------------- | ---------- | ------------------------------------------------------------ |
| value          | String          | 类名       | 为模型提供备用名称                                           |
| description    | String          | ""         | 提供详细的类描述                                             |
| parent         | Class<?> parent | Void.class | 为模型提供父类以允许描述继承关系                             |
| discriminatory | String          | “”         | 支持模型继承和多态，使用鉴别器的字段的名称，可以断言需要使用哪个子类型 |
| subTypes       | Class<?>[]      | {}         | 从此模型继承的子类型数组                                     |
| reference      | String          | “”         | 指定对应类型定义的引用，覆盖指定的任何其他元数据             |



下面为 @ApiModelProperty 的使用说明

| 注解属性        | 类型    | 默认值 | 描述                   |
| --------------- | ------- | ------ | ---------------------- |
| value           | String  | ""     | 字段说明               |
| name            | String  | ""     | 字段说明               |
| dataType        | String  | ""     | 字段类型               |
| required        | boolean | false  | 是否为必填             |
| example         | String  | ""     | 举例说明               |
| hidden          | boolean | false  | 是否在文档中隐藏此字段 |
| allowEmptyValue | boolean | false  | 是否允许为空           |
| allowableValues | String  | ""     | 该字段允许的值         |



本实例中, 用`User` 类使用注解进行描述: 

```java
@ApiModel( description = "用户信息")
@Data
public class User {


    @ApiModelProperty(value = "id", required = true, example = "1")
    private Long id;


    @ApiModelProperty(value = "姓名", required = true, example = "张三")
    private String name;

    @ApiModelProperty(value = "年龄", example = "10")
    private Integer age;

    @ApiModelProperty(value = "密码", hidden = true, required = true)
    private String passWord;
}
```

使用这些注解后, 显示的文档如下: 

![](http://files.luyanan.com//img/20191130113054.png)



### 2. 接口输入参数注解说明

在介绍Swagger 的接口输入参数说明之前, 有必要对SpringMvc的接口参数有一个清晰的了解. 我们知道SpringMvc 是通过@RequestMapping 把方法与请求的URL 对应起来, 而调用接口方法时需要将参数传给控制器. 一般来说, SpringMVC 的参数传递包括以下几种:

- 无注解参数
- @RequestParam 注解参数
- 数组参数
- JSON 对象参数
- REST风格参数

### 3. Swagger 输入参数注解说明

对于上面提供的SpringMvc参数,springfox 已经做了自动处理, 在 `swagger-ui` 文档中显示的时候, 也会根据参数类型进行显示. 在  swagger 中, 参数类型分为: 

- path: 以地址的形式提交数据, 根据id 查询用户的接口就是这种形式传参
- query：Query String的方式传参, 对应无注解和 `@RequestParam` 注解参数
- header: 请求头(header) 中的参数
- form: 以Form 表单的形式提交, 比如上传文件, 属于此类. 
- body: 以JSON 方式传参. 

在描述输入参数的时候, swagger 有以下注解: 

- `@ApiImplicitParams` 描述接口的参数集
- `@ApiImplicitParam` 描述接口单个参数, 与 `@ApiImplicitParams` 组合使用
- `@ApiParam` 描述单个参数, 可选

其中, `@ApiParam` 与单个参数一起用, `@ApiImplicitParam` 使用在接口描述中,两者的属性基本一致, `@ApiImplicitParam`的属性如下所示：



| 注解属性      | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| paramType     | 查询参数类型, 表示参数在哪里, 可取值:path、query、header、form、body |
| dataType      | 参数的数据类型, 只作为标志说明, 并没有实际验证. 可按照实际类型填写, 比如String、User |
| name          | 参数的名字                                                   |
| value         | 参数意义的描述                                               |
| required      | 是否必填                                                     |
| allowMultiple | 是否有多个, 在使用JSON 数组对象时使用.                       |
| example       | 数据示例                                                     |

示例如下: 

```java
   @ApiOperation(value = "添加多个用户", notes = "使用JSON数组的方式添加")
    @ApiImplicitParams(@ApiImplicitParam(name = "user", value = "用户数组", required = true, dataType = "User", allowMultiple = true))
    @PostMapping("/users")
    public void addUsers(@RequestBody List<User> users) {

    }
```

![](http://files.luyanan.com//img/20191130142314.png)



###  4.接口响应数据注解说明

对于使用了`@RestController `注解的控制器, SpringMVC 会自动将返回的数据转换为JSON 数据.

对于响应消息, swagger 提供了默认的401、403、404的响应消息, 也可以自己针对接口进行指定, 有以下注解可以使用. 

- @ApiResponses 响应消息集
- @ApiResponses 响应消息, 描述一个错误的响应信息, 与 @ApiResponses 一起使用
- @ResponseHeader 响应头设置



####  @ApiResponses 属性描述

| 属性     | 描述         |
| -------- | ------------ |
| code     | 错误码       |
| message  | 错误信息     |
| response | 抛出异常的类 |

```java
 @ApiResponses(@ApiResponse(code = 401, message = "错误信息", response = NullPointerException.class))
    @PostMapping("/users")
    public void addUsers(@RequestBody List<User> users) {

    }

```





### 5. 接口描述注解说明



#### @API 接口类说明注解

  用在类上, 说明该类的作用, 可以标记一个Controller 类作为swagger 文档资源, 使用方式: 

#####  使用

```java
@Api(value = "/user", description = "用户", tags = "用户")
@RestController
@RequestMapping("user")
public class UserController {
```

##### 属性说明

| 属性             | 描述                                               |
| ---------------- | -------------------------------------------------- |
| vaue             | url的路径值                                        |
| tags             | 标签, 如果设置这个值, value 的值就会被覆盖         |
| *description*    | 对API 资源的描述                                   |
| *basePath*       | 基本路径, 不需要配置                               |
| *position*       | 如果配置多个API， 想改变显示的顺序位置             |
| *produces*       | *For example, "application/json, application/xml"* |
| *consumes*       | *For example, "application/json, application/xml"* |
| *protocols*      | *Possible values: http, https, ws, wss.*           |
| *authorizations* | *高级特性认证时配置*                               |
| *hidden*         | *配置为true 将在文档中隐藏*                        |



####  @ApiOperation  接口描述注解

 配置在方法上, 说明方法的作用, 每一个url 资源的定义. 

#####  使用示例

```java
  @ApiOperation( notes = "根据id查询用户信息", response = User.class)
    public User user() {
        return new User();
    }
```

#####  属性介绍

| 属性                | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| *value*             | url的路径值                                                  |
| *tags*              | 如果设置这个值, value 的值就会被隐藏                         |
| *description*       | 对api资源的介绍                                              |
| *basePath*          | 基本路径, 可以不设置                                         |
| *position*          | 如果设置多个API 想改变显示的顺序位置                         |
| *produces*          | *For example, "application/json, application/xml"*           |
| consumes            | For example, "application/json, application/xml"             |
| *protocols*         | *Possible values: http, https, ws, wss.*                     |
| *authorizations*    | *高级特性认证时配置*                                         |
| *hidden*            | *配置为true 将在文档中隐藏*                                  |
| *response*          | *返回的对象*                                                 |
| *responseContainer* | *这些对象是有效的 "List", "Set" or "Map".，其他无效*         |
| *httpMethod*        | *"GET", "HEAD", "POST", "PUT", "DELETE", "OPTIONS" and "PATCH"* |
| *code*              | http状态码, 默认为200                                        |
| *extensions*        | 扩展属性                                                     |



####  @ApiIgnore

添加此注解的接口, swagger 在扫描的时候会忽略

###  6. 响应头注解

####  @ResponseHeader

 响应由设置

######  示例

```java
    @ResponseHeader(name = "token",description = "token")
    public User user() {
        return new User();
    }
```



#####  属性描述

| 属性                | 描述                     |
| ------------------- | ------------------------ |
| name                | 响应头名称               |
| *description*       | 头介绍                   |
| *response*          | 默认响应类Void           |
| *responseContainer* | *参考ApiOperation中配置* |


##  接口认证

对于前后端分离的项目, 现在很多采用jwt的认证(属于Bearer认证),需要先获取token, 然后在调用token 的时候, 添加类似 `Authorization: Bearer <token>` 的请求头来对接口进行认证, 针对这种方式, 在swgger 中有以下三种方式来实现. 

### 1. 添加接口认证参数

针对需要认证的接口, 直接使用 `@ApiImplicitParam`, 其中参数 `ParamType=header` , 如下: 

```java
@ApiImplicitParam(name = "Authorization", value = "token，格式: Bearer &lttoken&gt", required = false, dataType = "String",paramType = "header")
```



这种方式的缺点是, 针对每一个接口,都需要添加这个参数描述, 而描述都是一样的, 重复工作. 



### 2. 添加全局接口参数



swagger  中, 方法 `globalOperationParameters` 可以设置全局的参数

```java
//全局header参数
ParameterBuilder tokenPar = new ParameterBuilder();
List<Parameter> pars = new ArrayList<Parameter>();
tokenPar.name("Authorization").description("token令牌")
    .modelRef(new ModelRef("string"))
    .parameterType("header")
    .required(true).build();
pars.add(tokenPar.build());
docket.globalOperationParameters(pars);
```

这种方式缺点也明显, 由于设置了全局参数, 则所有接口都需要此参数, 若某些接口不需要, 则需要进行特殊处理. 

### 3. 添加安全认证上下文

设置认证模式, 并使用正则对需要认证的接口进行筛选, 这样swagger 界面提供统一的认证界面, 如下: 

```java
docket.securitySchemes(securitySchemes())
        .securityContexts(securityContexts());
...//省略
private List<ApiKey> securitySchemes() {
        return Lists.newArrayList(
                new ApiKey("Authorization", "Authorization", "header"));
    }
private List<SecurityContext> securityContexts() {
    return Lists.newArrayList(
        SecurityContext.builder()
        .securityReferences(defaultAuth())
        //正则式过滤,此处是所有非login开头的接口都需要认证
        .forPaths(PathSelectors.regex("^(?!login).*$"))
        .build()
    );
}
List<SecurityReference> defaultAuth() {
    AuthorizationScope authorizationScope = new AuthorizationScope("global", "认证权限");
    return Lists.newArrayList(
        new SecurityReference("Authorization", new AuthorizationScope[]{authorizationScope}));
}
```



