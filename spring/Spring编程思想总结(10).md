# 10 . Spring 编程思想总结

| Spring思想 | 应用场景(特点)                                               | 一句话归纳               |
| ---------- | ------------------------------------------------------------ | ------------------------ |
| OOP        | Object Oriented Programming(面向对象编程)用程序归纳总结生活中的一切事物 | 封装,继承,多态           |
| BOP        | Bean Oriented Programming(面向Bean 编程) 面向Bean(普通的java类)设计程序 | 一切从Bean开始           |
| AOP        | Aspect Origented Programming(面向切面编程) 找出多个类中有一定规律的代码,开发时拆开,运行时再合并,面向切面编程,既面向规则编程 | 解耦,专人做专事          |
| IOC        | Inversion of Control(控制反转) 将new对象的动作交给Spring管理,并由Spring保存已经创建的对象(IOC容器) | 转交控制权(既控制权反转) |
| DL/DI      | Dependency Injection (依赖注入)或者Dependency Lookup(依赖查找) 依赖注入,依赖查找,Spring不仅保存自己创建的对象,而且保存对象和对象之间的关系.注入既赋值,主要三种方式:构造方法,set方法,直接赋值 | 赋值                     |

### AOP在Spring中的应用
SpingAOP是一种编程范式,主要目的是将非功能性需求从功能性需求中分离出来,达到解耦的目的.主要应用场景由:Authentication(权限认证),Auto Caching(自动缓存处理),Error Handling(统一错误处理),Debugging(调式信息输出),Logging(日志记录),transactions(事务处理).现实生活中也常常用AOP思维来解决问题,如飞机组装,汽车组装等.<br>
飞机各部件的零件会交给不同的厂家区生产,最终由组装工厂将各个部件组装成一个整理.将零件的生产交出去的主要目的是解耦,但是解耦之前必须由统一的标准.