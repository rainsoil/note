# Spring 注解大全

###  核心注解

| 注解        | 介绍                                                         |
| ----------- | ------------------------------------------------------------ |
| @Documented | 将会在被此注解注解的元素的javadoc文档中列出注解，一般都打上这个注解没坏处 |
| @Target     | 注解能被应用的目标元素，比如类、方法、属性、参数等等         |
|             | @Retention                                                   |
| @Inherited  | 如果子类没有定义注解的话，能自动从父类获取定义了继承属性的注解，比如Spring的@Service是没有继承特性的，但是@Transactional是有继承特性的，在OO继承体系中使用Spring注解的时候请特别注意这点，理所当然认为注解是能被子类继承的话可能会引起不必要的Bug，需要仔细斟酌是否开启继承 |
| @Repeatable | Java 8 引入的特性，通过关联注解容器定义可重复注解，小小语法糖提高了代码可读性，对于元素有多个重复注解其实是很常见的事情，比如某方法可以是A角色可以访问也可以是B角色可以访问，某方法需要定时任务执行，要在A条件执行也需要在B条件执行 |
| @Native     | 是否在.h头文件中生成被标记的字段，除非原生程序需要和Java程序交互，否则很少会用到这个元注解 |

### spring 核心注解
| 注解                         | 介绍                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| @Controller                  | 定义表现层组件                                               |
| @Service                     | 定义业务逻辑层组件                                           |
| @Repository                  | 定义数据访问层组件                                           |
| @Component                   | 定义其他组件                                                 |
| @Autowired                   | 自动装配                                                     |
| @Required                    | 用于在setter方法标记属性值需要由Spring 进行装配,目前这个方法已经被spring 废弃,推荐使用构造方法注入 |
| @Qualifier                   | 用于给Bean定义修饰语来注入相应的bean                         |
| @Value                       | 用于注入属性配置或者Spel表达式                               |
| @Lookup                      | 可以实现方法的注入,如果我们的类是单例的,但是又希望Spring注入的依赖的对象是Prototype生命周期（每次new一个出来）的，这个时候可以通过此注解进行方法注入 |
| @EnableTransactionManagement | 用于开启事务管理，使用Spring Boot如果引入Spring Data的话不需要手动开启（不过建议大家在使用事务的时候还是通过日志来验证事务管理是否生效） |
| @Transactional               | 用于开启事务以及设置传播性、隔离性、回滚条件等；             |
| @TransactionalEventListener  | 用于配置事务的回调方法，可以在事务提交前、提交后、完成后以及回滚后几个阶段接受回调事件。 |
| @Order                       | 可以设置Spring管理对象的加载顺序，在之前介绍AOP的文章中我们看到有的时候我们必须通过设置合理的@Order来合理安排切面的切入顺序避免一些问题，还有在一些业务场景中，我们往往会去定义一组类似于Filter的@Component，然后会从容器获得一组Bean，这个时候业务组件的运行顺序往往会比较重要，也可以通过这个方式进行排序 |
| @AliasFor                    | 注解可以设置一组注解属性相互作为别名，对于有歧义的时候会使代码更清晰，此外还有一个用途是创建复合注解，Spring MVC的@GetMapping注解就是基于@RequestMapping这个注解创建的复合注解，我们可以很方便得通过这种方式来实现注解的继承 |


### spring 上下文注解
| 注解                    | 解释                                                         |
| ----------------------- | ------------------------------------------------------------ |
| @Configuration          | 用于标注配置类，启用Java配置方式的Bean配置                   |
| @Bean                   | 用于配置一个Bean                                             |
| @ComponentScan          | （@ComponentScans用于配置一组@ComponentScan，Java 8可以直接使用重复注解特性配置多个@ComponentScan）用于扫描包方式配置Bean； |
| @PropertySource         | 以及 @PropertySources用于导入配置文件；                      |
| @Conditional            | 用于设置关联的条件类，在合适的时候启用Bean的配置（Spring Boot自动配置根基） |
| @Import                 | 用于导入其他配置类                                           |
| @ImportResource         | 用于导入非java配置方式的XML配置                              |
| @Profile                | 用于指定在核实的Profile下启动配置                            |
| @Lazy                   | 用于告知容器延迟到使用的时候实例化Bean                       |
| @Description            | 用于给Bean设置描述                                           |
| @Scope                  | 用于设置Bean的生命周期                                       |
| @Primary                | 用于在定义多个Bean的时候指定首选的Bean                       |
| @EventListener          | 用于设置回调方法监听Spring指定的以及自定义的各种事件         |
| @EnableAspectJAutoProxy | 用于开启支持Aspectj的@Aspect的切面配置支持,使用Spring Boot引入了AOP启动器的话不需要显式开启 |
### Spring Web 注解
| 注解              | 解释                                                         |
| ----------------- | ------------------------------------------------------------ |
| @RequestScope     | bean 跟随请求的生命周期创建                                  |
| @SessionScope     | Bean 跟随会话的生命周期创建                                  |
| @ApplicationScope | Bean 跟谁应用程序的生命周期创建                              |
| @ResponseStatus   | 可以用到方法上,也可以用到异常上,前者会直接使请求得到指定的响应代码或者原因(可以配合@ExpectionHandler使用),后者可以实现遇到指定的异常的时候给出指定的相应代码或原因 |
| @ResponseBody     | 将返回的内容序列化之后输出到请求体                           |
| @RequestBody      | 接受请求数据                                                 |
| @RequestHeader    | 从header中获取                                               |
| @CookieValue      | 从cookie 获取                                                |
| @SessionAttribute | 从会话中                                                     |
| @RequestAttribute | 从请求的Attribute中（比如过滤器和拦截器手动设置的一些临时数据） |
| @RequestParam     | 从请求参数（处理简单数据，键值对）                           |
| @PathVariable     | 从路径片段                                                   |
| @MatrixAttribute  | 矩阵变量允许我们采用特殊的规则在URL路径后加参数（分号区分不同参数，逗号为参数增加多个值） |
| @ControllerAdvice | 允许我们在集中的地方配置控制器（有@RequestMapping的方法）相关的增强（@RestControllerAdvice也是差不多的，只是相当于为@ExceptionHandler加上了@ResponseBody） |
| @ExceptionHandler | 进行统一的全局异常处理；                                     |
| @InitBinder       |                                                              |
用来设置WebDataBinder，WebDataBinder用来自动绑定前台请求参数到Model中；第三是 @ModelAttribute让全局的@RequestMapping都能获得在此处设置的键值


### SpringBoot 注解
| 注解                                                   | 解释                                                         |
| ------------------------------------------------------ | ------------------------------------------------------------ |
| @ConfigurationProperties                               | 很常用（配合 @EnableConfigurationProperties注解来设置需要启用的配置类），用来自定义配置类和配置文件进行关联 |
| @DeprecatedConfigurationProperty                       | 用于标记废弃的配置以及设置替代配置和告知废弃原因；           |
| @ConfigurationPropertiesBinding                        | 用于指定自定义的转换器用于配置解析的时的类型转换；           |
| @NestedConfigurationProperty                           | 用于关联外部的类型作为嵌套配置类                             |
| @EnableAutoConfiguration                               | 可以启用自动配置                                             |
| @ConditionalOnBean                                     |                                                              |
| 用于仅当容器中已经包含指定的Bean类型或名称时才匹配条件 |                                                              |
| @ConditionalOnClass                                    | 仅当classpath上存在指定类时条件匹配；                        |
| @ConditionalOnCloudPlatform                            | 仅当指定的云平台处于活动状态时条件匹配；                     |
| @ConditionalOnExpression                               | 依赖于SpEL表达式的值的条件元素的配置注解；                   |
| @ConditionalOnJava                                     | 基于应用运行的JVM版本的条件匹配；                            |
| @ConditionalOnJndi                                     | 基于JNDI可用和可以查找指定位置的条件匹配；                   |
| @ConditionalOnMissingBean                              | 仅当容器中不包含指定的Bean类型或名称时条件匹配；             |
| @ConditionalOnMissingClass                             | 仅当classpath上不存在指定类时条件匹配；                      |
| @ConditionalOnNotWebApplication                        | 仅当不是WebApplicationContext（非Web项目）时条件匹配，对应 @ConditionalOnWebApplication；@ConditionalOnProperty是检查指定的属性是否具有指定的值； |
| @ConditionalOnResource                                 | 表示仅当 classpath 上存在指定资源时条件匹配；                |
| @ConditionalOnSingleCandidate                          | 仅当容器中包含指定的Bean类并且可以判断只有单个候选者时条件匹配。其实所有这些实现原理都是扩展SpringBootCondition抽象类（实现之前提到的Condition接口），我们完全可以实现自己的条件注解（配合 @Conditional注解关联到自己实现的SpringBootCondition） |