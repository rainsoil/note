# 5.  Mybatis 设计模式总结

| 设计模式   | 类                                                           |
| ---------- | ------------------------------------------------------------ |
| 工厂       | SqlSessionFactory,ObjectFactory,MapperProxyFactory           |
| 建造者     | XMLConfigBuilder,XMLMapperBuilder,XMLStatementBuilder        |
| 单例模式   | SqlSessionFactory,Configuration                              |
| 代理模式   | 绑定:MappeProxy<br> 延迟加载:ProxyFactory(CGLIB,javassit)<br> 插件:Plugin<br> Spring继承Mybatis:SqlSessionTemplate 的内部类SqlSessionIntercepor<br> Mybatis 自带连接池:PoolDataSource管理的 PooledConnection<br> 日志打印:  ConnectionLogger,StatementLogger |
| 适配器模式 | logger 模块,对于Log4j,JDK logger,这些没有直接实现 slg4j接口的日志组件,需要适配器 |
| 模板方法   | BaseExecutor与子类SimpleExecutor,BatchExecutor,ReuseExecutor |
| 装饰器模式 | LoggerCache,LruCache等对PrepetualCache的装饰,CachingExecutor对其他Executor的装饰 |
| 责任链模式 | InterceptorChain                                             |