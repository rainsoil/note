# 8. Spring MVC 源码分析

## Spring MVC初体验

----------------


---------------

### 初探Spring MVC请求处理流程
SpringMVC相对于前面的章节算是比较简单的，我们首先引用《SpringinAction》上
的一张图来了解Spring MVC的核心组件和大致处理流程：
![image](http://files.luyanan.com/82daff69-d963-4c04-9997-16f599dcbc68.jpg)

从上图中看到

①、 DispatcherServlet是SpringMVC中的前端控制器(FrontController),
负责接收Request并将Request转发给对应的处理组件。


② 、 HanlerMapping 是 SpringMVC 中 完 成 url 到 Controller 映 射 的 组 件 。
DispatcherServlet 接收 Request,然后从 HandlerMapping 查找处理 Request 的
Controller。


③、Controller 处理 Request,并返回 ModelAndView 对象,Controller 是 SpringMVC
中负责处理 Request 的组件(类似于 Struts2 中的 Action),ModelAndView 是封装结果
视图的组件。

④、⑤、⑥视图解析器解析ModelAndView对象并返回对应的视图给客户端。
在前面的章节中我们已经大致了解到，容器初始化时会建立所有url 和Controller 中的
Method 的对应关系，保存到 HandlerMapping 中，用户请求是根据 Request 请求的
url快速定位到Controller 中的某个方法。在Spring中先将url和Controller的对应关
系,保存到 Map<url,Controller>中。Web 容器启动时会通知 Spring 初始化容器(加载
Bean的定义信息和初始化所有单例Bean),然后SpringMVC会遍历容器中的Bean，获
取每一个Controller中的所有方法访问的url，然后将url和Controller保存到一个Map
中；这样就可以根据 Request 快速定位到 Controller，因为最终处理 Request 的是
Controller 中的方法，Map 中只保留了 url 和 Controller 中的对应关系，所以要根据
Request 的 url 进一步确认 Controller 中的 Method，这一步工作的原理就是拼接
Controller 的 url(Controller 上@RequestMapping 的值)和方法的 url(Method 上
@RequestMapping的值)，与request的url进行匹配，找到匹配的那个方法；确定处
理请求的Method后，接下来的任务就是参数绑定，把Request中参数绑定到方法的形
式参数上，这一步是整个请求处理过程中最复杂的一个步骤。

### Spring MVC九大组件
#### HandlerMappings
HandlerMapping是用来查找Handler的，也就是处理器，具体的表现形式可以是类也
可以是方法。比如，标注了@RequestMapping 的每个 method 都可以看成是一个
Handler，由 Handler 来负责实际的请求处理。 HandlerMapping 在请求到达之后，
它的作用便是找到请求相应的处理器Handler和Interceptors。
##### HandlerAdapters
从名字上看，这是一个适配器。因为SpringMVC中Handler可以是任意形式的，只要
能够处理请求便行, 但是把请求交给 Servlet 的时候，由于 Servlet 的方法结构都是如
doService(HttpServletRequestreq,HttpServletResponseresp) 这样的形式，让固定
的Servlet 处理方法调用 Handler 来进行处理，这一步工作便是HandlerAdapter 要做
的事。
#### HandlerExceptionResolvers 
从这个组件的名字上看，这个就是用来处理Handler过程中产生的异常情况的组件。 具
体来说，此组件的作用是根据异常设置ModelAndView, 之后再交给 render()方法进行
渲 染 ， 而 render() 便 将 ModelAndView 渲 染 成 页 面 。 不 过 有 一 点 ，
HandlerExceptionResolver 只是用于解析对请求做处理阶段产生的异常，而渲染阶段的
异常则不归他管了，这也是Spring MVC 组件设计的一大原则分工明确互不干涉。
#### ViewResolvers 
视图解析器，相信大家对这个应该都很熟悉了。因为通常在SpringMVC的配置文件中，
都会配上一个该接口的实现类来进行视图的解析。 这个组件的主要作用，便是将String

类型的视图名和Locale解析为View类型的视图。这个接口只有一个resolveViewName()
方法。从方法的定义就可以看出，Controller层返回的String类型的视图名viewName，
最终会在这里被解析成为View。View 是用来渲染页面的，也就是说，它会将程序返回
的参数和数据填入模板中，最终生成 html 文件。ViewResolver在这个过程中，主要做
两件大事，即，ViewResolver会找到渲染所用的模板（使用什么模板来渲染？）和所用
的技术（其实也就是视图的类型，如JSP啊还是其他什么Blabla的）填入参数。默认情
况下，SpringMVC会为我们自动配置一个InternalResourceViewResolver，这个是针
对JSP类型视图的。 
##### RequestToViewNameTranslator
这个组件的作用，在于从 Request 中获取 viewName. 因为 ViewResolver 是根据
ViewName 查找 View, 但有的 Handler 处理完成之后，没有设置 View 也没有设置
ViewName， 便要通过这个组件来从Request中查找viewName。 LocaleResolver 在上面我们有看到ViewResolver 的resolveViewName()方法，需要两个参数。那么第
二个参数Locale是从哪来的呢，这就是LocaleResolver 要做的事了。 
#### LocaleResolver
用于从 request 中解析出 Locale, 在中国大陆地区，Locale当然就会是zh-CN之类，
用来表示一个区域。这个类也是i18n的基础。 
#### ThemeResolver
从名字便可看出，这个类是用来解析主题的。主题，就是样式，图片以及它们所形成的
显示效果的集合。Spring MVC中一套主题对应一个properties文件，里面存放着跟当
前主题相关的所有资源，如图片，css样式等。创建主题非常简单，只需准备好资源，然
后新建一个 "主题名.properties" 并将资源设置进去，放在classpath下，便可以在页面

中使用了。 Spring MVC 中跟主题有关的类有 ThemeResolver, ThemeSource 和
Theme。 ThemeResolver 负责从request中解析出主题名， ThemeSource则根据主
题名找到具体的主题， 其抽象也就是 Theme, 通过Theme来获取主题和具体的资源。 
#### MultipartResolver
其实这是一个大家很熟悉的组件，MultipartResolver用于处理上传请求，通过将普通的
Request包装成MultipartHttpServletRequest来实现。MultipartHttpServletRequest
可以通过getFile() 直接获得文件，如果是多个文件上传，还可以通过调用getFileMap
得到Map<FileName,File> 这样的结构。MultipartResolver的作用就是用来封装普通
的request，使其拥有处理文件上传的功能。 

#### FlashMapManager
说到FlashMapManager，就得先提一下FlashMap。
FlashMap用于重定向Redirect时的参数数据传递，比如，在处理用户订单提交时，为
了避免重复提交，可以处理完post请求后redirect到一个get请求，这个get请求可以
用来显示订单详情之类的信息。这样做虽然可以规避用户刷新重新提交表单的问题，但
是在这个页面上要显示订单的信息，那这些数据从哪里去获取呢，因为redirect重定向
是没有传递参数这一功能的，如果不想把参数写进url(其实也不推荐这么做，url有长度
限制不说，把参数都直接暴露，感觉也不安全)， 那么就可以通过flashMap来传递。只
需 要 在 redirect 之 前 ， 将 要 传 递 的 数 据 写 入 request （ 可 以 通 过
ServletRequestAttributes.getRequest() 获 得 ） 的 属 性
OUTPUT_FLASH_MAP_ATTRIBUTE 中，这样在 redirect 之后的 handler 中 Spring 就
会自动将其设置到Model中，在显示订单信息的页面上，就可以直接从Model 中取得
数据了。而FlashMapManager就是用来管理FlashMap的。

### Spring MVC源码分析
根据上面分析的Spring MVC工作机制，从三个部分来分析Spring MVC的源代码。

其一，ApplicationContext初始化时用Map保存所有url和Controller类的对应关系；

其二，根据请求url找到对应的Controller，并从Controller中找到处理请求的方法;

其三，Request参数绑定到方法的形参，执行方法处理请求，并返回结果视图。 
##### 初始化阶段
我们首先找到DispatcherServlet这个类，必然是寻找init()方法。然后，我们发现其init
方法其实在父类HttpServletBean中，其源码如下：
```java
@Override
	public final void init() throws ServletException {
		if (logger.isDebugEnabled()) {
			logger.debug("Initializing servlet '" + getServletName() + "'");
		}

		// Set bean properties from init parameters.
		PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
		if (!pvs.isEmpty()) {
			try {
				//定位资源
				BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
				//加载配置信息
				ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
				bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
				initBeanWrapper(bw);
				bw.setPropertyValues(pvs, true);
			}
			catch (BeansException ex) {
				if (logger.isErrorEnabled()) {
					logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
				}
				throw ex;
			}
		}

		// Let subclasses do whatever initialization they like.
		initServletBean();

		if (logger.isDebugEnabled()) {
			logger.debug("Servlet '" + getServletName() + "' configured successfully");
		}
	}
```
 我们看到在这段代码中，又调用了一个重要的 initServletBean()方法。进入
initServletBean()方法看到以下源码：

```java
@Override
	protected final void initServletBean() throws ServletException {
		getServletContext().log("Initializing Spring FrameworkServlet '" + getServletName() + "'");
		if (this.logger.isInfoEnabled()) {
			this.logger.info("FrameworkServlet '" + getServletName() + "': initialization started");
		}
		long startTime = System.currentTimeMillis();

		try {

			this.webApplicationContext = initWebApplicationContext();
			initFrameworkServlet();
		}
		catch (ServletException ex) {
			this.logger.error("Context initialization failed", ex);
			throw ex;
		}
		catch (RuntimeException ex) {
			this.logger.error("Context initialization failed", ex);
			throw ex;
		}

		if (this.logger.isInfoEnabled()) {
			long elapsedTime = System.currentTimeMillis() - startTime;
			this.logger.info("FrameworkServlet '" + getServletName() + "': initialization completed in " +
					elapsedTime + " ms");
		}
	}
```
 这段代码中最主要的逻辑就是初始化IOC容器，最终会调用refresh()方法，前面的章节
中对IOC容器的初始化细节我们已经详细掌握，在此我们不再赘述。我们看到上面的代

码中，IOC 容器初始化之后，最后有调用了 onRefresh()方法。这个方法最终是在
DisptcherServlet 中实现，来看源码：
```java
	@Override
	protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}

	/**
	 * Initialize the strategy objects that this servlet uses.
	 * <p>May be overridden in subclasses in order to initialize further strategy objects.
	 */
	//初始化策略
	protected void initStrategies(ApplicationContext context) {
		//多文件上传的组件
		initMultipartResolver(context);
		//初始化本地语言环境
		initLocaleResolver(context);
		//初始化模板处理器
		initThemeResolver(context);
		//handlerMapping
		initHandlerMappings(context);
		//初始化参数适配器
		initHandlerAdapters(context);
		//初始化异常拦截器
		initHandlerExceptionResolvers(context);
		//初始化视图预处理器
		initRequestToViewNameTranslator(context);
		//初始化视图转换器
		initViewResolvers(context);
		//
		initFlashMapManager(context);
	}

```
 到这一步就完成了SpringMVC的九大组件的初始化。 接下来， 我们来看url和Controller
的 关 系 是 如 何 建 立 的 呢 ？ HandlerMapping 的 子 类
AbstractDetectingUrlHandlerMapping 实现了 initApplicationContext()方法，所以
我们直接看子类中的初始化容器方法。

```java
@Override
	public void initApplicationContext() throws ApplicationContextException {
		super.initApplicationContext();
		detectHandlers();
	}

	/**
	 * Register all handlers found in the current ApplicationContext.
	 * <p>The actual URL determination for a handler is up to the concrete
	 * {@link #determineUrlsForHandler(String)} implementation. A bean for
	 * which no such URLs could be determined is simply not considered a handler.
	 * @throws org.springframework.beans.BeansException if the handler couldn't be registered
	 * @see #determineUrlsForHandler(String)
	 */
	/**
	 * 建立当前ApplicationContext中的所有controller和url的对应关系
	 */
	protected void detectHandlers() throws BeansException {
		ApplicationContext applicationContext = obtainApplicationContext();
		if (logger.isDebugEnabled()) {
			logger.debug("Looking for URL mappings in application context: " + applicationContext);
		}
		// 获取ApplicationContext容器中所有bean的Name
		String[] beanNames = (this.detectHandlersInAncestorContexts ?
				BeanFactoryUtils.beanNamesForTypeIncludingAncestors(applicationContext, Object.class) :
				applicationContext.getBeanNamesForType(Object.class));

		// Take any bean name that we can determine URLs for.
		// 遍历beanNames,并找到这些bean对应的url
		for (String beanName : beanNames) {
			// 找bean上的所有url(controller上的url+方法上的url),该方法由对应的子类实现
			String[] urls = determineUrlsForHandler(beanName);
			if (!ObjectUtils.isEmpty(urls)) {
				// URL paths found: Let's consider it a handler.
				// 保存urls和beanName的对应关系,put it to Map<urls,beanName>,该方法在父类AbstractUrlHandlerMapping中实现
				registerHandler(urls, beanName);
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Rejected bean name '" + beanName + "': no URL paths identified");
				}
			}
		}
	}
```
 determineUrlsForHandler(String beanName)方法的作用是获取每个Controller 中的
url，不同的子类有不同的实现，这是一个典型的模板设计模式。因为开发中我们用的最
多的就是用注解来配置 Controller 中的 url，BeanNameUrlHandlerMapping 是
AbstractDetectingUrlHandlerMapping的子类,处理注解形式的url映射.所以我们这里

以 BeanNameUrlHandlerMapping 来 进 行 分 析 。 我 们 看
BeanNameUrlHandlerMapping是如何查beanName上所有映射的url。

```java
	/**
	 * Checks name and aliases of the given bean for URLs, starting with "/".
	 */
	@Override
	protected String[] determineUrlsForHandler(String beanName) {
		List<String> urls = new ArrayList<>();
		if (beanName.startsWith("/")) {
			urls.add(beanName);
		}
		String[] aliases = obtainApplicationContext().getAliases(beanName);
		for (String alias : aliases) {
			if (alias.startsWith("/")) {
				urls.add(alias);
			}
		}
		return StringUtils.toStringArray(urls);
	}


```
 到这里HandlerMapping组件就已经建立所有url和Controller的对应关系。 

#### 运行调用阶段

这一步步是由请求触发的，所以入口为DispatcherServlet 的核心方法为doService()，
doService()中的核心逻辑由doDispatch()实现，源代码如下：
```java
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
		if (logger.isDebugEnabled()) {
			String resumed = WebAsyncUtils.getAsyncManager(request).hasConcurrentResult() ? " resumed" : "";
			logger.debug("DispatcherServlet with name '" + getServletName() + "'" + resumed +
					" processing " + request.getMethod() + " request for [" + getRequestUri(request) + "]");
		}

		// Keep a snapshot of the request attributes in case of an include,
		// to be able to restore the original attributes after the include.
		Map<String, Object> attributesSnapshot = null;
		if (WebUtils.isIncludeRequest(request)) {
			attributesSnapshot = new HashMap<>();
			Enumeration<?> attrNames = request.getAttributeNames();
			while (attrNames.hasMoreElements()) {
				String attrName = (String) attrNames.nextElement();
				if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
					attributesSnapshot.put(attrName, request.getAttribute(attrName));
				}
			}
		}

		// Make framework objects available to handlers and view objects.
		request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
		request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
		request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
		request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

		if (this.flashMapManager != null) {
			FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
			if (inputFlashMap != null) {
				request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
			}
			request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
			request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
		}

		try {
			doDispatch(request, response);
		}
		finally {
			if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
				// Restore the original attribute snapshot, in case of an include.
				if (attributesSnapshot != null) {
					restoreAttributesAfterInclude(request, attributesSnapshot);
				}
			}
		}
	}
	
	
	/** 中央控制器,控制请求的转发 **/
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				// 1.检查是否是文件上传的请求
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				// 2.取得处理当前请求的controller,这里也称为hanlder,处理器,
				// 	 第一个步骤的意义就在这里体现了.这里并不是直接返回controller,
				//	 而是返回的HandlerExecutionChain请求处理器链对象,
				//	 该对象封装了handler和interceptors.
				mappedHandler = getHandler(processedRequest);
				// 如果handler为空,则返回404
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
				//3. 获取处理request的处理器适配器handler adapter
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				// 处理 last-modified 请求头
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (logger.isDebugEnabled()) {
						logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
					}
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}

				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
				// 4.实际的处理器处理请求,返回结果视图对象
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				// 结果视图对象的处理
				applyDefaultViewName(processedRequest, mv);
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					// 请求成功响应之后的方法
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}

```
 getHandler(processedRequest)方法实际上就是从 HandlerMapping 中找到 url 和
Controller的对应关系。也就是Map<url,Controller>。我们知道，最终处理Request
的是Controller中的方法，我们现在只是知道了Controller，我们如何确认Controller
中处理Request的方法呢？继续往下看。

从Map<urls,beanName>中取得Controller 后，经过拦截器的预处理方法，再通过反
射获取该方法上的注解和参数，解析方法和参数上的注解，然后反射调用方法获取

ModelAndView 结果视图。最后，调用的就是 RequestMappingHandlerAdapter 的
handle()中的核心逻辑由handleInternal(request, response, handler)实现。

```java
@Override
	protected ModelAndView handleInternal(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ModelAndView mav;
		checkRequest(request);

		// Execute invokeHandlerMethod in synchronized block if required.
		if (this.synchronizeOnSession) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				Object mutex = WebUtils.getSessionMutex(session);
				synchronized (mutex) {
					mav = invokeHandlerMethod(request, response, handlerMethod);
				}
			}
			else {
				// No HttpSession available -> no mutex necessary
				mav = invokeHandlerMethod(request, response, handlerMethod);
			}
		}
		else {
			// No synchronization on session demanded at all...
			mav = invokeHandlerMethod(request, response, handlerMethod);
		}

		if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
			if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
				applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
			}
			else {
				prepareResponse(response);
			}
		}

		return mav;
	}
```
 整个处理过程中最核心的逻辑其实就是拼接Controller的url和方法的url，与Request
的url进行匹配，找到匹配的方法。

```java
	@Override
	protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
		String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
		if (logger.isDebugEnabled()) {
			logger.debug("Looking up handler method for path " + lookupPath);
		}
		this.mappingRegistry.acquireReadLock();
		try {
			HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
			if (logger.isDebugEnabled()) {
				if (handlerMethod != null) {
					logger.debug("Returning handler method [" + handlerMethod + "]");
				}
				else {
					logger.debug("Did not find handler method for [" + lookupPath + "]");
				}
			}
			return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
		}
		finally {
			this.mappingRegistry.releaseReadLock();
		}
	}
```
 通过上面的代码分析，已经可以找到处理Request的Controller 中的方法了，现在看如
何解析该方法上的参数，并反射调用该方法。

```java
/** 获取处理请求的方法,执行并返回结果视图 **/
	@Nullable
	protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		try {
			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
			if (this.argumentResolvers != null) {
				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
			}
			if (this.returnValueHandlers != null) {
				invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
			}
			invocableMethod.setDataBinderFactory(binderFactory);
			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

			ModelAndViewContainer mavContainer = new ModelAndViewContainer();
			mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
			modelFactory.initModel(webRequest, mavContainer, invocableMethod);
			mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

			AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
			asyncWebRequest.setTimeout(this.asyncRequestTimeout);

			WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
			asyncManager.setTaskExecutor(this.taskExecutor);
			asyncManager.setAsyncWebRequest(asyncWebRequest);
			asyncManager.registerCallableInterceptors(this.callableInterceptors);
			asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

			if (asyncManager.hasConcurrentResult()) {
				Object result = asyncManager.getConcurrentResult();
				mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
				asyncManager.clearConcurrentResult();
				if (logger.isDebugEnabled()) {
					logger.debug("Found concurrent result value [" + result + "]");
				}
				invocableMethod = invocableMethod.wrapConcurrentResult(result);
			}

			invocableMethod.invokeAndHandle(webRequest, mavContainer);
			if (asyncManager.isConcurrentHandlingStarted()) {
				return null;
			}

			return getModelAndView(mavContainer, modelFactory, webRequest);
		}
		finally {
			webRequest.requestCompleted();
		}
	}
```

invocableMethod.invokeAndHandle()最终要实现的目的就是：完成 Request 中的参
数和方法参数上数据的绑定。Spring MVC中提供两种Request参数到方法中参数的绑
定方式：

1、通过注解进行绑定，@RequestParam。

2、通过参数名称进行绑定。
使用注解进行绑定，我们只要在方法参数前面声明@RequestParam("name")，就可以
将request中参数name的值绑定到方法的该参数上。使用参数名称进行绑定的前提是
必须要获取方法中参数的名称，Java反射只提供了获取方法的参数的类型，并没有提供
获取参数名称的方法。SpringMVC解决这个问题的方法是用asm框架读取字节码文件，
来获取方法的参数名称。asm框架是一个字节码操作框架，关于asm更多介绍可以参考
其官网。个人建议，使用注解来完成参数绑定，这样就可以省去asm框架的读取字节码
的操作。
```java
@Nullable
	public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
		if (logger.isTraceEnabled()) {
			logger.trace("Invoking '" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
					"' with arguments " + Arrays.toString(args));
		}
		Object returnValue = doInvoke(args);
		if (logger.isTraceEnabled()) {
			logger.trace("Method [" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
					"] returned [" + returnValue + "]");
		}
		return returnValue;
	}

	/**
	 * Get the method argument values for the current request.
	 */
	private Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		MethodParameter[] parameters = getMethodParameters();
		Object[] args = new Object[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
			args[i] = resolveProvidedArgument(parameter, providedArgs);
			if (args[i] != null) {
				continue;
			}
			if (this.argumentResolvers.supportsParameter(parameter)) {
				try {
					args[i] = this.argumentResolvers.resolveArgument(
							parameter, mavContainer, request, this.dataBinderFactory);
					continue;
				}
				catch (Exception ex) {
					if (logger.isDebugEnabled()) {
						logger.debug(getArgumentResolutionErrorMessage("Failed to resolve", i), ex);
					}
					throw ex;
				}
			}
			if (args[i] == null) {
				throw new IllegalStateException("Could not resolve method parameter at index " +
						parameter.getParameterIndex() + " in " + parameter.getExecutable().toGenericString() +
						": " + getArgumentResolutionErrorMessage("No suitable resolver for", i));
			}
		}
		return args;
	}

```
 关于asm框架获取方法参数的部分,这里就不再进行分析了。感兴趣的小伙伴可以继续深
入了解这个处理过程。
到这里,方法的参数值列表也获取到了,就可以直接进行方法的调用了。整个请求过程
中最复杂的一步就是在这里了。到这里整个请求处理过程的关键步骤都已了解。理解了
Spring MVC 中的请求处理流程,整个代码还是比较清晰的。

最后来一张时序图：

![image](http://files.luyanan.com/9732e2ba-ff10-4ae9-a195-21b8eee0d91a.jpg)

#### Spring MVC使用优化建议

上面我们已经对SpringMVC的工作原理和源码进行了分析，在这个过程发现了几个优化
点: 
##### 1、Controller如果能保持单例，尽量使用单例
这样可以减少创建对象和回收对象的开销。也就是说，如果Controller的类变量和实例
变量可以以方法形参声明的尽量以方法的形参声明，不要以类变量和实例变量声明，这
样可以避免线程安全问题。 
##### 2、处理Request的方法中的形参务必加上@RequestParam注解
这样可以避免Spring MVC使用asm框架读取class文件获取方法参数名的过程。即便
Spring MVC对读取出的方法参数名进行了缓存，如果不要读取class文件当然是更好。 
##### 3、缓存URL
阅读源码的过程中，我们发现Spring MVC并没有对处理 url的方法进行缓存，也就是
说每次都要根据请求url去匹配Controller中的方法url，如果把url和Method的关系

缓存起来，会不会带来性能上的提升呢？有点恶心的是，负责解析url和Method对应
关系的 ServletHandlerMethodResolver 是一个 private的内部类，不能直接继承该类
增强代码，必须要该代码后重新编译。当然，如果缓存起来，必须要考虑缓存的线程安
全问题。