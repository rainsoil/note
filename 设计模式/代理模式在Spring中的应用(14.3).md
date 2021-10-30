# 14.3 代理模式在Spring中的应用

先看ProxyFactoryBean类,核心方法为getObject(),我们来看一下源码,
```java

    @Nullable
    public Object getObject() throws BeansException {
        this.initializeAdvisorChain();
        if (this.isSingleton()) {
            return this.getSingletonInstance();
        } else {
            if (this.targetName == null) {
                this.logger.warn("Using non-singleton proxies with singleton targets is often undesirable. Enable prototype proxies by setting the 'targetName' property.");
            }

            return this.newPrototypeInstance();
        }
    }
```
在getObject()方法中,主要调用getSingletonInstance()方法和newPrototypeInstance()方法,在Spring的配置当中,如果不做任何的设置,那么Spring代理生成的的Bean都是单例对象.如果修改scope则每次创建一个新的原型对象.<br>
Spring利用动态代理实现AOP有两个非常重要的类,一个是JdkDynamicAopProxy类和CglibAopProxy类
##### Spring中大代理选择原则:
1. 当Bean有实现接口的时候,Spring就会用JDK的动态代理
2. 当Bean没有实现接口的时候,Spring选择CGlib
3. Spring可以通过配置强制使用CGlib,只需要在Spring的配置文件中添加如下代码:
```
<aop:aspectj-autoproxy proxy-target-class="true"/>
```
参考资料如下: 
https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html

