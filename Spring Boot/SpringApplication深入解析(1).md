# SpringApplication 深入解析

我们知道,SpringBoot 程序中最重要的类就是`SpringApplication`,`SpringApplication`是SpringBoot驱动Spring应用上下文的引导类. 接下来我们看看`SpringApplication`有什么玄妙之处. 

## 1. 自定义SpringApplication

我们要启动一个SpringBoot程序, 我们需要使用`@SpringBootApplication`注解标注一个启动类, 并且执行`SpringApplication.run()`来启动.

```java
@SpringBootApplication
public class SpringbootDemo1Application {

    public static void main(String[] args) {
        SpringApplication.run(SpringbootDemo1Application.class, args);
    }

}
```

我们先看看`@SpringBootApplication`

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
public @interface SpringBootApplication {
    
    ...   
}
```

我们看到, @SpringBootApplication 注解是一个复合注解, 由三个注解组合而成. 

- @ComponentScan
- @EnableAutoConfiguration
- @SpringBootConfiguration

### @ComponentScan

这是`Spring Framework`3.1中引进来的一个注解, 主要是根据定义的扫描的包路径, 将符合扫描规则的类装配到Spring容器当中. 

```java
@Repeatable(ComponentScans.class)
public @interface ComponentScan {
    @AliasFor("basePackages")
    String[] value() default {};

    @AliasFor("value")
    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};

    Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;

    Class<? extends ScopeMetadataResolver> scopeResolver() default AnnotationScopeMetadataResolver.class;

    ScopedProxyMode scopedProxy() default ScopedProxyMode.DEFAULT;

    String resourcePattern() default "**/*.class";

    boolean useDefaultFilters() default true;

    ComponentScan.Filter[] includeFilters() default {};

    ComponentScan.Filter[] excludeFilters() default {};

    boolean lazyInit() default false;

    @Retention(RetentionPolicy.RUNTIME)
    @Target({})
    public @interface Filter {
        FilterType type() default FilterType.ANNOTATION;

        @AliasFor("classes")
        Class<?>[] value() default {};

        @AliasFor("value")
        Class<?>[] classes() default {};

        String[] pattern() default {};
    }
}
```

- **basePackages**与**value**: 用于给定的包的路径, 进行扫描
- **basePackageClasses**: 用于指定某个类的包的路径进行扫描
- **nameGenerator**: bean的名称的生成器
- **useDefaultFilters**: 是否开启对@Component，@Repository，@Service，@Controller的类进行检测
- **includeFilters**: 包含的过滤条件
  - FilterType.ANNOTATION：按照注解过滤
  - FilterType.ASSIGNABLE_TYPE：按照给定的类型
  - FilterType.ASPECTJ：使用ASPECTJ表达式
  -  FilterType.REGEX：正则
  - FilterType.CUSTOM：自定义规则
- **excludeFilters**: 排除的规则, 用于跟includeFilters 一样. 

那么`@ComponentScan` 注解是在哪里作用的呢? 

我们看`org.springframework.context.annotation.ConfigurationClassParser#doProcessConfigurationClass` 这个方法中

```java
@Nullable
	protected final SourceClass doProcessConfigurationClass(
			ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
			throws IOException {

		if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
			// Recursively process any member (nested) classes first
			processMemberClasses(configClass, sourceClass, filter);
		}

		// Process any @PropertySource annotations
		for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), PropertySources.class,
				org.springframework.context.annotation.PropertySource.class)) {
			if (this.environment instanceof ConfigurableEnvironment) {
				processPropertySource(propertySource);
			}
			else {
				logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
						"]. Reason: Environment must implement ConfigurableEnvironment");
			}
		}

		// Process any @ComponentScan annotations
		Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
		if (!componentScans.isEmpty() &&
				!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
			for (AnnotationAttributes componentScan : componentScans) {
				// The config class is annotated with @ComponentScan -> perform the scan immediately
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
				// Check the set of scanned definitions for any further config classes and parse recursively if needed
				for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
					BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
					if (bdCand == null) {
						bdCand = holder.getBeanDefinition();
					}
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
						parse(bdCand.getBeanClassName(), holder.getBeanName());
					}
				}
			}
		}

        
        ...
    }
```

我们看

```java
Set<BeanDefinitionHolder> scannedBeanDefinitions =
      this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
```

调用了解析器进行解析, 我们继续看 `org.springframework.context.annotation.ComponentScanAnnotationParser#parse`

```java
public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
		ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
				componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);

		Class<? extends BeanNameGenerator> generatorClass = componentScan.getClass("nameGenerator");
		boolean useInheritedGenerator = (BeanNameGenerator.class == generatorClass);
		scanner.setBeanNameGenerator(useInheritedGenerator ? this.beanNameGenerator :
				BeanUtils.instantiateClass(generatorClass));

		ScopedProxyMode scopedProxyMode = componentScan.getEnum("scopedProxy");
		if (scopedProxyMode != ScopedProxyMode.DEFAULT) {
			scanner.setScopedProxyMode(scopedProxyMode);
		}
		else {
			Class<? extends ScopeMetadataResolver> resolverClass = componentScan.getClass("scopeResolver");
			scanner.setScopeMetadataResolver(BeanUtils.instantiateClass(resolverClass));
		}

		scanner.setResourcePattern(componentScan.getString("resourcePattern"));

		for (AnnotationAttributes filter : componentScan.getAnnotationArray("includeFilters")) {
			for (TypeFilter typeFilter : typeFiltersFor(filter)) {
				scanner.addIncludeFilter(typeFilter);
			}
		}
		for (AnnotationAttributes filter : componentScan.getAnnotationArray("excludeFilters")) {
			for (TypeFilter typeFilter : typeFiltersFor(filter)) {
				scanner.addExcludeFilter(typeFilter);
			}
		}

		boolean lazyInit = componentScan.getBoolean("lazyInit");
		if (lazyInit) {
			scanner.getBeanDefinitionDefaults().setLazyInit(true);
		}

		Set<String> basePackages = new LinkedHashSet<>();
		String[] basePackagesArray = componentScan.getStringArray("basePackages");
		for (String pkg : basePackagesArray) {
			String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
					ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
			Collections.addAll(basePackages, tokenized);
		}
		for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
			basePackages.add(ClassUtils.getPackageName(clazz));
		}
		if (basePackages.isEmpty()) {
			basePackages.add(ClassUtils.getPackageName(declaringClass));
		}

		scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
			@Override
			protected boolean matchClassName(String className) {
				return declaringClass.equals(className);
			}
		});
		return scanner.doScan(StringUtils.toStringArray(basePackages));
	}
```

这个方法中, 使用`ClassPathBeanDefinitionScanner`的`doScan` 方法进行解析

我们先看这一段

```java
ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
				componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);
```

```java
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
			Environment environment, @Nullable ResourceLoader resourceLoader) {

		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		this.registry = registry;

		if (useDefaultFilters) {
			registerDefaultFilters();
		}
		setEnvironment(environment);
		setResourceLoader(resourceLoader);
	}
```

我们看这个方法`registerDefaultFilters()`

```java
protected void registerDefaultFilters() {
		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
		ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
		try {
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
			logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
		}
		try {
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
			logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
```

我们看`this.includeFilters.add(new AnnotationTypeFilter(Component.class));` 这段代码, `@ComponentScan` 扫描的时候, 会将包含 `@Component`注解的所有类和"派生类"都扫描.  

### @EnableAutoConfiguration

激活自动装配,也就是启动SpringBoot的自动加载配置, 我们平时会看到很多已`@Enable` 开头的, 比如: 

* `@EnableWebMvc`
* `@EnableTransactionManagement`
* `@EnableAspectJAutoProxy`
* `@EnableAsync`
* ...

### @SpringBootConfiguration

```java
@Configuration
public @interface SpringBootConfiguration {
    @AliasFor(
        annotation = Configuration.class
    )
    boolean proxyBeanMethods() default true;
}
```

我们看到, 这也是一个复合注解,其实其真正意义上还是`@Configuration` 注解,我们再看看`@Configuration`注解,

```java
@Component
public @interface Configuration {
    @AliasFor(
        annotation = Component.class
    )
    String value() default "";

    boolean proxyBeanMethods() default true;
}
```



我们看到, 最后是`@Component`注解. 

这里, 我们不得不说说  [Spring 注解编程模型](https://github.com/spring-projects/spring-framework/wiki/Spring-Annotation-Programming-Model)

我们可以看到,

  * `@Service`

    ```java
    @Component
    public @interface Service {
        ...
    }
    ```

  * `@Repository`

    ```java
    @Component
    public @interface Repository {
        ...
    }
    ```

  * `@Controller`

    ```java
    @Component
    public @interface Controller {
        ...
    }
    ```

  * `@Configuration`

    ```java
    @Component
    public @interface Configuration {
    	...
    }
    ```

 在上面的这些我们常用的注解当中, 都"派生了"`@Component`注解. 

## 2. 配置SpringBoot源

`SpringApplication` Spring Boot 应用的引导, 那么我们启动一个Spring上下文, 有那些方式呢? 

### 2.1. 基于 `AnnotationConfigApplicationContext`  注解进行启动上下文

```java
/**
 * @author luyanan
 * @since 2020/3/17
 * <p>注解启动Spring上下文</p>
 **/
@Configuration
public class SpringBootAnnotationDemo {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        // 注册一个Configuration class
        context.register(SpringBootAnnotationDemo.class);
        // 启动上下文
        context.refresh();
        System.out.println(context.getBean(SpringBootAnnotationDemo.class));
    }

}
```

输出结果:

> com.demo.annotation.SpringBootAnnotationDemo$$EnhancerBySpringCGLIB$$8a01a66@527e5409



### 2.2 基于SpringApplicationBuilder

```java
@SpringBootApplication
public class SpringApplicationBuilderDemo {

    public static void main(String[] args) {
        new SpringApplicationBuilder(SpringApplicationBuilderDemo.class)
                .properties("server.port=0")
                .run(args);

    }

}
```

### 2.3 使用`SpringApplication` 启动上下文

```java
@SpringBootApplication
public class SpringbootDemo1Application {

    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(SpringbootDemo1Application.class);
        Map<String, Object> pro = new HashMap<>();
        pro.put("server.port", "0");
        application.setDefaultProperties(pro);
        application.run(args);

    }

}
```

### 2.4 调整为非web程序

```java
// 非web程序
    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(SpringbootDemo1Application.class);
        Map<String, Object> pro = new HashMap<>();
        pro.put("server.port", "0");
        //调整为非web程序
        application.setWebApplicationType(WebApplicationType.NONE);
        application.setDefaultProperties(pro);
        ConfigurableApplicationContext context = application.run(args);
        System.out.println("当前Spring 应用上下文类:" + context.getClass().getName());
    }

```

我们看到, 我们通过`application.setWebApplicationType(WebApplicationType.NONE);` 设置为非web程序，当我们不调整的时候, Spring在启动的时候, 是如何推断出当前为非web程序还是Servlet程序还是webFlux 程序的呢? 

## 3. SpringApplication类型推断

 当我们不设置类型的时候, Spring 是如何给我们推断的呢? 

我们看代码

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```



我们看到`this.webApplicationType`是`WebApplicationType.deduceFromClasspath()` 方法判断出来的, 我们继续看

```java
private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };

	private static final String WEBMVC_INDICATOR_CLASS = "org.springframework.web.servlet.DispatcherServlet";

	private static final String WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler";

	private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";

	private static final String SERVLET_APPLICATION_CONTEXT_CLASS = "org.springframework.web.context.WebApplicationContext";

	private static final String REACTIVE_APPLICATION_CONTEXT_CLASS = "org.springframework.boot.web.reactive.context.ReactiveWebApplicationContext";


static WebApplicationType deduceFromClasspath() {
		if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
				&& !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
			return WebApplicationType.REACTIVE;
		}
		for (String className : SERVLET_INDICATOR_CLASSES) {
			if (!ClassUtils.isPresent(className, null)) {
				return WebApplicationType.NONE;
			}
		}
		return WebApplicationType.SERVLET;
	}
```

* `WebApplicationType.NONE` : 非 Web 类型
  * `Servlet` 不存在
  * Spring Web 应用上下文 `ConfigurableWebApplicationContext`  不存在
    * `spring-boot-starter-web` 不存在
    * `spring-boot-starter-webflux` 不存在

* `WebApplicationType.REACTIVE` : Spring WebFlux
  * `DispatcherHandler`
    * `spring-boot-starter-webflux` 存在
  * `Servlet` 不存在
    * `spring-boot-starter-web` 不存在
* `WebApplicationType.SERVLET` : Spring MVC
  * `spring-boot-starter-web` 存在



当自己设置类型的时候, `org.springframework.boot.SpringApplication#setWebApplicationType`

```java
	public void setWebApplicationType(WebApplicationType webApplicationType) {
		Assert.notNull(webApplicationType, "WebApplicationType must not be null");
		this.webApplicationType = webApplicationType;
	}
```

会将`webApplicationType` 类型设置为自己想要的类型



##  4 SpringBoot事件

### Spring事件

#### Spring内部发送的事件

我们可以通过以下代码来看看Spring内部发送的事件

```java
@Configuration
public class SpringEventDemo {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        // 注册一个Configuration class
        context.register(SpringEventDemo.class);
        context.addApplicationListener(new ApplicationListener<ApplicationEvent>() {
            @Override
            public void onApplicationEvent(ApplicationEvent event) {
                System.err.println(event.getClass().getName());
            }
        });
        // 启动上下文
        context.refresh();
        context.close();


    }

}
```

运行后我们发现 , 发送了两个事件. 

- ContextRefreshedEvent
- ContextClosedEvent



那么这两个事件是在哪里被发送的呢? 

##### ContextRefreshedEvent

我们进去`org.springframework.context.support.AbstractApplicationContext#refresh` 方法

```java
@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
		   ...
	// Last step: publish corresponding event.
				finishRefresh();
            ...

			
	}
```

接下来看`org.springframework.context.support.AbstractApplicationContext#finishRefresh`

```java
protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		clearResourceCaches();

		// Initialize lifecycle processor for this context.
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```

我们看`publishEvent(new ContextRefreshedEvent(this));` 这里发送了一个`ContextRefreshedEvent` 事件

##### ContextClosedEvent

我们再看看这个事件在哪里发送的发送的呢? 

我们进入到`org.springframework.context.support.AbstractApplicationContext#close`

```java
@Override
	public void close() {
		synchronized (this.startupShutdownMonitor) {
			doClose();
			// If we registered a JVM shutdown hook, we don't need it anymore now:
			// We've already explicitly closed the context.
			if (this.shutdownHook != null) {
				try {
					Runtime.getRuntime().removeShutdownHook(this.shutdownHook);
				}
				catch (IllegalStateException ex) {
					// ignore - VM is already shutting down
				}
			}
		}
	}

```

看`org.springframework.context.support.AbstractApplicationContext#doClose` 方法

```java
protected void doClose() {
		// Check whether an actual close attempt is necessary...
		if (this.active.get() && this.closed.compareAndSet(false, true)) {
			if (logger.isDebugEnabled()) {
				logger.debug("Closing " + this);
			}

			LiveBeansView.unregisterApplicationContext(this);

			try {
				// Publish shutdown event.
				publishEvent(new ContextClosedEvent(this));
			}
            
            ...
        }
```

我们看`publishEvent(new ContextClosedEvent(this));` 这里发送了`ContextClosedEvent` 事件



###  自定义事件

当我们自定义一个事件的时候, 那么这个事件是在哪里发送的呢? 

```java

//自定义事件
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        // 注册一个Configuration class
        context.register(SpringEventDemo.class);
        context.addApplicationListener(new ApplicationListener<ApplicationEvent>() {
            @Override
            public void onApplicationEvent(ApplicationEvent event) {
                System.err.println(event.getClass().getName());
            }
        });
        // 启动上下文
        context.refresh();
        context.publishEvent(new MyApplicationEvent("自定义事件"));

    }

    public static class MyApplicationEvent extends ApplicationEvent {
        /**
         * Create a new {@code ApplicationEvent}.
         *
         * @param source the object on which the event initially occurred or with
         *               which the event is associated (never {@code null})
         */
        public MyApplicationEvent(Object source) {
            super(source);
        }
    }
```

输出结果

```java

org.springframework.context.event.ContextRefreshedEvent
com.demo.event.SpringEventDemo$MyApplicationEvent
```

 那么这个事件是在哪里被发送的呢? 

我们从`context.publishEvent(new MyApplicationEvent("自定义事件"))` 这个方法进去

```java
	@Override
	public void publishEvent(ApplicationEvent event) {
		publishEvent(event, null);
	}
```

继续看`org.springframework.context.support.AbstractApplicationContext#publishEvent(java.lang.Object, org.springframework.core.ResolvableType)`

```java
protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
		Assert.notNull(event, "Event must not be null");

		// Decorate event as an ApplicationEvent if necessary
		ApplicationEvent applicationEvent;
		if (event instanceof ApplicationEvent) {
			applicationEvent = (ApplicationEvent) event;
		}
		else {
			applicationEvent = new PayloadApplicationEvent<>(this, event);
			if (eventType == null) {
				eventType = ((PayloadApplicationEvent<?>) applicationEvent).getResolvableType();
			}
		}

		// Multicast right now if possible - or lazily once the multicaster is initialized
		if (this.earlyApplicationEvents != null) {
			this.earlyApplicationEvents.add(applicationEvent);
		}
		else {
			getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
		}

		// Publish event via parent context as well...
		if (this.parent != null) {
			if (this.parent instanceof AbstractApplicationContext) {
				((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
			}
			else {
				this.parent.publishEvent(event);
			}
		}
	}
```

我们看到,  事件是通过`getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);` 发送的. 





### SpringBoot 的事件

接下来我们看看SpringBoot的事件有那些呢?

```java
@SpringBootApplication
public class SpringBootEvent {

    public static void main(String[] args) {
        new SpringApplicationBuilder(SpringBootEvent.class)
                .listeners(event -> System.err.println(event.getClass().getName()))
                .run(args).close();
    }

}

```

我们看输出结果得知, 事件有: 

- `org.springframework.boot.context.event.ApplicationStartingEvent`
- `org.springframework.boot.context.event.ApplicationEnvironmentPreparedEvent`
- `org.springframework.boot.context.event.ApplicationContextInitializedEvent`
- `org.springframework.boot.context.event.ApplicationPreparedEvent`
- `org.springframework.context.event.ContextRefreshedEvent`
- `org.springframework.boot.web.servlet.context.ServletWebServerInitializedEvent`
- `org.springframework.boot.context.event.ApplicationReadyEvent`
- `org.springframework.context.event.ContextClosedEvent`
- `org.springframework.boot.context.event.ApplicationFailedEvent`（容器启动失败的时候才会发送到事件）



