#  spring 中常用的设计模式对比

| 设计模式              | 一句话归纳                    | 举例                                                         |
| --------------------- | ----------------------------- | ------------------------------------------------------------ |
| 工厂模式(Factory)     | 只对结果负责,封装创建过程     | BeanFactory,Calender                                         |
| 单例模式(Singleton)   | 保证独一无二                  | ApplicationContext,Calender                                  |
| 原型模式(Prototype)   | 克隆                          | ArrayList,PrototypeBean                                      |
| 代理模式(Proxy)       | 增强职责                      | ProxyFactoryBean,JDKDynamicAopFactory,CglibAopFactory        |
| 委派模式(Delegate)    | 干活算员工的,功劳算项目经理的 | DispatcherServlet,BeanDefinitionParserDelegate               |
| 策略模式(Strategy)    | 用户选择,结果统一             | InstantiationStrategy                                        |
| 模板模式(Template)    | 流程标准化,自己实现定制       | jdbcTemplate,HttpServlet                                     |
| 适配器模式(Adapter)   | 兼容转换头                    | AdvisorAdapter,HandlerAdapter                                |
| 装饰器模式(Decorator) | 包装,同宗同源                 | BufferedReader,InputStream,OutputStream,HttpHeadResponseDecorator |
| 观察者模式(Observer)  | 任务完成时通知                | ContextLoaderListener                                        |