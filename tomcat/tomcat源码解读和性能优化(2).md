#  tomcat源码解读和性能优化

## 1. 源码解读

### 1.1 `BootStrap`

`BootStrap` 是`tomcat`的入口类.

`org.apache.catalina.startup.Bootstrap#main`

```java
 /**
     * Main method and entry point when starting Tomcat via the provided
     * scripts.
     *
     * @param args Command line arguments to be processed
     */
    public static void main(String args[]) {

        if (daemon == null) {
            // Don't set daemon until init() has completed
            Bootstrap bootstrap = new Bootstrap();
            try {
                bootstrap.init();
            } catch (Throwable t) {
                handleThrowable(t);
                t.printStackTrace();
                return;
            }
            daemon = bootstrap;
        } else {
            // When running as a service the call to stop will be on a new
            // thread so make sure the correct class loader is used to prevent
            // a range of class not found exceptions.
            Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
        }

        try {
            String command = "start";
            if (args.length > 0) {
                command = args[args.length - 1];
            }

            if (command.equals("startd")) {
                args[args.length - 1] = "start";
                daemon.load(args);
                daemon.start();
            } else if (command.equals("stopd")) {
                args[args.length - 1] = "stop";
                daemon.stop();
            } else if (command.equals("start")) {
                daemon.setAwait(true);
                daemon.load(args);
                daemon.start();
            } else if (command.equals("stop")) {
                daemon.stopServer(args);
            } else if (command.equals("configtest")) {
                daemon.load(args);
                if (null==daemon.getServer()) {
                    System.exit(1);
                }
                System.exit(0);
            } else {
                log.warn("Bootstrap: command \"" + command + "\" does not exist.");
            }
        } catch (Throwable t) {
            // Unwrap the Exception for clearer error reporting
            if (t instanceof InvocationTargetException &&
                    t.getCause() != null) {
                t = t.getCause();
            }
            handleThrowable(t);
            t.printStackTrace();
            System.exit(1);
        }

    }

```

`bootstrap.init()` 和 创建`Catalina`



### 1.2 `Catalina`

解析`server.xml` 配置文件

创建`Server`组件,并且调用其`init`和`start`方法

当调用`daemon.load(args);`的时候, 进入到`org.apache.catalina.startup.Bootstrap#load`

```java
 /**
     * Load daemon.
     */
    private void load(String[] arguments)
        throws Exception {

        // Call the load() method
        String methodName = "load";
        Object param[];
        Class<?> paramTypes[];
        if (arguments==null || arguments.length==0) {
            paramTypes = null;
            param = null;
        } else {
            paramTypes = new Class[1];
            paramTypes[0] = arguments.getClass();
            param = new Object[1];
            param[0] = arguments;
        }
        Method method =
            catalinaDaemon.getClass().getMethod(methodName, paramTypes);
        if (log.isDebugEnabled())
            log.debug("Calling startup class " + method);
        method.invoke(catalinaDaemon, param);

    }

```

我们看到用反射调用了`org.apache.catalina.startup.Catalina#load()` 方法

真正的执行步骤

```java
 /**
     * Start a new server instance.
     */
    public void load() {

        long t1 = System.nanoTime();

        initDirs();

        // Before digester - it may be needed
        initNaming();

        // Create and execute our Digester
        Digester digester = createStartDigester();

        InputSource inputSource = null;
        InputStream inputStream = null;
        File file = null;
        try {
            file = configFile();
            inputStream = new FileInputStream(file);
            inputSource = new InputSource(file.toURI().toURL().toString());
        } catch (Exception e) {
            if (log.isDebugEnabled()) {
                log.debug(sm.getString("catalina.configFail", file), e);
            }
        }
        if (inputStream == null) {
            try {
                inputStream = getClass().getClassLoader()
                    .getResourceAsStream(getConfigFile());
                inputSource = new InputSource
                    (getClass().getClassLoader()
                     .getResource(getConfigFile()).toString());
            } catch (Exception e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("catalina.configFail",
                            getConfigFile()), e);
                }
            }
        }

        // This should be included in catalina.jar
        // Alternative: don't bother with xml, just create it manually.
        if (inputStream == null) {
            try {
                inputStream = getClass().getClassLoader()
                        .getResourceAsStream("server-embed.xml");
                inputSource = new InputSource
                (getClass().getClassLoader()
                        .getResource("server-embed.xml").toString());
            } catch (Exception e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("catalina.configFail",
                            "server-embed.xml"), e);
                }
            }
        }


        if (inputStream == null || inputSource == null) {
            if  (file == null) {
                log.warn(sm.getString("catalina.configFail",
                        getConfigFile() + "] or [server-embed.xml]"));
            } else {
                log.warn(sm.getString("catalina.configFail",
                        file.getAbsolutePath()));
                if (file.exists() && !file.canRead()) {
                    log.warn("Permissions incorrect, read permission is not allowed on the file.");
                }
            }
            return;
        }

        try {
            inputSource.setByteStream(inputStream);
            digester.push(this);
            digester.parse(inputSource);
        } catch (SAXParseException spe) {
            log.warn("Catalina.start using " + getConfigFile() + ": " +
                    spe.getMessage());
            return;
        } catch (Exception e) {
            log.warn("Catalina.start using " + getConfigFile() + ": " , e);
            return;
        } finally {
            try {
                inputStream.close();
            } catch (IOException e) {
                // Ignore
            }
        }

        getServer().setCatalina(this);
        getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
        getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());

        // Stream redirection
        initStreams();

        // Start the new server
        try {
            getServer().init();
        } catch (LifecycleException e) {
            if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
                throw new java.lang.Error(e);
            } else {
                log.error("Catalina.start", e);
            }
        }

        long t2 = System.nanoTime();
        if(log.isInfoEnabled()) {
            log.info("Initialization processed in " + ((t2 - t1) / 1000000) + " ms");
        }
    }
```



###  1.3 `Lifecycle`

用来管理各个组件的生命周期

`init`、`start`、`stop`、`destroy`

`LifecycleBase` 实现了`Lifecycle`, 使用的是模板设计模式

```java
 * <pre>
 *            start()
 *  -----------------------------
 *  |                           |
 *  | init()                    |
 * NEW ->-- INITIALIZING        |
 * | |           |              |     ------------------<-----------------------
 * | |           |auto          |     |                                        |
 * | |          \|/    start() \|/   \|/     auto          auto         stop() |
 * | |      INITIALIZED -->-- STARTING_PREP -->- STARTING -->- STARTED -->---  |
 * | |         |                                                  |         |  |
 * | |         |                                                  |         |  |
 * | |         |                                                  |         |  |
 * | |destroy()|                                                  |         |  |
 * | -->-----<--       auto                    auto               |         |  |
 * |     |       ---------<----- MUST_STOP ---------------------<--         |  |
 * |     |       |                                                          |  |
 * |    \|/      ---------------------------<--------------------------------  ^
 * |     |       |                                                             |
 * |     |      \|/            auto                 auto              start()  |
 * |     |  STOPPING_PREP ------>----- STOPPING ------>----- STOPPED ---->------
 * |     |                                ^                  |  |  ^
 * |     |               stop()           |                  |  |  |
 * |     |       --------------------------                  |  |  |
 * |     |       |                                  auto     |  |  |
 * |     |       |                  MUST_DESTROY------<-------  |  |
 * |     |       |                    |                         |  |
 * |     |       |                    |auto                     |  |
 * |     |       |    destroy()      \|/              destroy() |  |
 * |     |    FAILED ---->------ DESTROYING ---<-----------------  |
 * |     |                        ^     |                          |
 * |     |     destroy()          |     |auto                      |
 * |     -------->-----------------    \|/                         |
 * |                                 DESTROYED                     |
 * |                                                               |
 * |                            stop()                             |
 * --->------------------------------>------------------------------
public interface Lifecycle {
 ...   
    public void init() throws LifecycleException;

    /**
     * Prepare for the beginning of active use of the public methods other than
     * property getters/setters and life cycle methods of this component. This
     * method should be called before any of the public methods other than
     * property getters/setters and life cycle methods of this component are
     * utilized. The following {@link LifecycleEvent}s will be fired in the
     * following order:
     * <ol>
     *   <li>BEFORE_START_EVENT: At the beginning of the method. It is as this
     *                           point the state transitions to
     *                           {@link LifecycleState#STARTING_PREP}.</li>
     *   <li>START_EVENT: During the method once it is safe to call start() for
     *                    any child components. It is at this point that the
     *                    state transitions to {@link LifecycleState#STARTING}
     *                    and that the public methods other than property
     *                    getters/setters and life cycle methods may be
     *                    used.</li>
     *   <li>AFTER_START_EVENT: At the end of the method, immediately before it
     *                          returns. It is at this point that the state
     *                          transitions to {@link LifecycleState#STARTED}.
     *                          </li>
     * </ol>
     *
     * @exception LifecycleException if this component detects a fatal error
     *  that prevents this component from being used
     */
    public void start() throws LifecycleException;


    /**
     * Gracefully terminate the active use of the public methods other than
     * property getters/setters and life cycle methods of this component. Once
     * the STOP_EVENT is fired, the public methods other than property
     * getters/setters and life cycle methods should not be used. The following
     * {@link LifecycleEvent}s will be fired in the following order:
     * <ol>
     *   <li>BEFORE_STOP_EVENT: At the beginning of the method. It is at this
     *                          point that the state transitions to
     *                          {@link LifecycleState#STOPPING_PREP}.</li>
     *   <li>STOP_EVENT: During the method once it is safe to call stop() for
     *                   any child components. It is at this point that the
     *                   state transitions to {@link LifecycleState#STOPPING}
     *                   and that the public methods other than property
     *                   getters/setters and life cycle methods may no longer be
     *                   used.</li>
     *   <li>AFTER_STOP_EVENT: At the end of the method, immediately before it
     *                         returns. It is at this point that the state
     *                         transitions to {@link LifecycleState#STOPPED}.
     *                         </li>
     * </ol>
     *
     * Note that if transitioning from {@link LifecycleState#FAILED} then the
     * three events above will be fired but the component will transition
     * directly from {@link LifecycleState#FAILED} to
     * {@link LifecycleState#STOPPING}, bypassing
     * {@link LifecycleState#STOPPING_PREP}
     *
     * @exception LifecycleException if this component detects a fatal error
     *  that needs to be reported
     */
    public void stop() throws LifecycleException;

    /**
     * Prepare to discard the object. The following {@link LifecycleEvent}s will
     * be fired in the following order:
     * <ol>
     *   <li>DESTROY_EVENT: On the successful completion of component
     *                      destruction.</li>
     * </ol>
     *
     * @exception LifecycleException if this component detects a fatal error
     *  that prevents this component from being used
     */
    public void destroy() throws LifecycleException;
    
    
    ...
    
    
}
```



### 1.4 `Server`

管理`Service`组件, 调用的其`init`和`start` 方法

`org.apache.catalina.Server`是一个接口类, 其唯一的实现类`org.apache.catalina.core.StandardServer`

  ```java
 @Override
    protected void initInternal() throws LifecycleException {

        super.initInternal();

        // Register global String cache
        // Note although the cache is global, if there are multiple Servers
        // present in the JVM (may happen when embedding) then the same cache
        // will be registered under multiple names
        onameStringCache = register(new StringCache(), "type=StringCache");

        // Register the MBeanFactory
        MBeanFactory factory = new MBeanFactory();
        factory.setContainer(this);
        onameMBeanFactory = register(factory, "type=MBeanFactory");

        // Register the naming resources
        globalNamingResources.init();

        // Populate the extension validator with JARs from common and shared
        // class loaders
        if (getCatalina() != null) {
            ClassLoader cl = getCatalina().getParentClassLoader();
            // Walk the class loader hierarchy. Stop at the system class loader.
            // This will add the shared (if present) and common class loaders
            while (cl != null && cl != ClassLoader.getSystemClassLoader()) {
                if (cl instanceof URLClassLoader) {
                    URL[] urls = ((URLClassLoader) cl).getURLs();
                    for (URL url : urls) {
                        if (url.getProtocol().equals("file")) {
                            try {
                                File f = new File (url.toURI());
                                if (f.isFile() &&
                                        f.getName().endsWith(".jar")) {
                                    ExtensionValidator.addSystemResource(f);
                                }
                            } catch (URISyntaxException e) {
                                // Ignore
                            } catch (IOException e) {
                                // Ignore
                            }
                        }
                    }
                }
                cl = cl.getParent();
            }
        }
        // Initialize our defined Services
        for (int i = 0; i < services.length; i++) {
            services[i].init();
        }
    }

  ```



###  1.5 `Service`

管理连接器和`Engine`

`org.apache.catalina.Service` 同样是一个接口类,有唯一的实现`org.apache.catalina.core.StandardService`

![image-20200513101014740](http://files.luyanan.com//img/20200513103841.png)

![image-20200513101027782](http://files.luyanan.com//img/20200513103844.png)

![image-20200513101039045](http://files.luyanan.com//img/20200513103848.png)



`ContainerBase`的类关系图

![image-20200513103825522](http://files.luyanan.com//img/20200513103826.png)



关注到上图图解中的`ContainerBase.startInternal()` 方法

```java
 @Override
    protected synchronized void startInternal() throws LifecycleException {

        // Start our subordinate components, if any
        logger = null;
        getLogger();
        Cluster cluster = getClusterInternal();
        if ((cluster != null) && (cluster instanceof Lifecycle))
            ((Lifecycle) cluster).start();
        Realm realm = getRealmInternal();
        if ((realm != null) && (realm instanceof Lifecycle))
            ((Lifecycle) realm).start();

        // Start our child containers, if any
        Container children[] = findChildren();
        List<Future<Void>> results = new ArrayList<>();
        for (int i = 0; i < children.length; i++) {
            // 这句代码的意思就是调用`contailerBase`下面的一个个子容器的`call`方法
            results.add(startStopExecutor.submit(new StartChild(children[i])));
        }

        boolean fail = false;
        for (Future<Void> result : results) {
            try {
                result.get();
            } catch (Exception e) {
                log.error(sm.getString("containerBase.threadedStartFailed"), e);
                fail = true;
            }

        }
        if (fail) {
            throw new LifecycleException(
                    sm.getString("containerBase.threadedStartFailed"));
        }

        // Start the Valves in our pipeline (including the basic), if any
        if (pipeline instanceof Lifecycle)
            ((Lifecycle) pipeline).start();


        setState(LifecycleState.STARTING);

        // Start our thread
        threadStart();

    }
```

查看`new StartChild` 要执行的`call` 方法

```java
 private static class StartChild implements Callable<Void> {

        private Container child;

        public StartChild(Container child) {
            this.child = child;
        }

        @Override
        public Void call() throws LifecycleException {
            child.start();
            return null;
        }
    }

```

![image-20200513104313879](http://files.luyanan.com//img/20200513104314.png)

`StandardHost` 将一个个`web`项目部署起来

![image-20200513104344727](http://files.luyanan.com//img/20200513104345.png)

`org.apache.catalina.startup.HostConfig#deployApps()`

```java
    protected void deployApps() {

        File appBase = host.getAppBaseFile();
        File configBase = host.getConfigBaseFile();
        String[] filteredAppPaths = filterAppPaths(appBase.list());
        // Deploy XML descriptors from configBase
        deployDescriptors(configBase, configBase.list());
        // Deploy WARs
        deployWARs(appBase, filteredAppPaths);
        // Deploy expanded folders
        deployDirectories(appBase, filteredAppPaths);

    }
```

`StandardContext.startInternal()`解析`web.xml`文件和添加`wrapper`

![image-20200513104729040](http://files.luyanan.com//img/20200513104730.png)

`ContextConfig.webConfig()` 的`step9`解析到`servlets` 包装成`wrapper` 对象

`StandardContext.startInternal()`->最终会调用`if (!loadOnStartup(findChildren()))`

`官网`： https://tomcat.apache.org/tomcat-8.0-doc/architecture/startup/serverStartup.pdf





## 2. 性能优化

`tomcat` 性能指标: 吞吐量、响应时间、错误数、线程池、CPU、内存等. 

使用`jmeter` 进行压测,然后观察相应指标. 

**使用命令查看相关指标**

1.  查看`tomcat`进程`pid`

   ```bash
   ps -ef | grep tomcat
   ```

2. 查看进程的信息

   ```bash
   cat /pro/pid/status
   ```

3. 查看进程的CPU和内存

   ```bash
   top -p pid
   ```

使用工具查看相应的指标

`jconsole、jvisualvm、arthas、psi-probe等`



### 2.1 优化思路

#### 2.1.1 `conf/server.xml` 等核心组件

- `Server`

  官网描述 :[Server interface](https://tomcat.apache.org/tomcat-8.0-doc/api/org/apache/catalina/Server.html) which is rarely customized by users. 【pass】

- `Service`

  官网描述 :The Service element is rarely customized by users. 【pass】

- `Connector`

  官网描述 :Creating a customized connector is a significant effort. 【 need 】

- `Engine`

  官网描述 :The [Engine interface](https://tomcat.apache.org/tomcat-8.0-doc/api/org/apache/catalina/Engine.html) may be implemented to supply custom Engines, though this is uncommon. 【pass】

- `Host`

  官网描述 :Users rarely create custom [Hosts](https://tomcat.apache.org/tomcat-8.0-doc/api/org/apache/catalina/Host.html) because the [StandardHost implementation](https://tomcat.apache.org/tomcat-8.0-doc/api/org/apache/catalina/core/StandardHost.html) provides significant additional functionality. 【pass】

- `Context`

  官网描述 :The [Context interface](https://tomcat.apache.org/tomcat-8.0-doc/api/org/apache/catalina/Context.html) may be implemented to create custom Contexts, but this is rarely the case because the [StandardContext](https://tomcat.apache.org/tomcat-8.0-doc/api/org/apache/catalina/core/StandardContext.html) provides significant additional functionality. 【 maybe 】

`context` 既然代表的是`web`应用,是和我们比较接近的,这块我们考虑对其进行适当的优化. 



#### 2.1.2 `conf/server.xml` 非核心组件

`官网`: https://tomcat.apache.org/tomcat-8.0-doc/config/index.html

- `Listener`

  `Listener`(即监听器) 定义的组件,可以在特定事件发生时执行特定的操作, 被监听的事件通常是`Tomcat`的启动和停止

  ```xml
    <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
    <!-- Prevent memory leaks due to use of particular java/javax APIs-->
    <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
    <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
    <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
  ```

- `Global Resources`

  `GlobalNamingResources` 元素定义了全局资源,通过配置可以看出,该配置是通过读取`$TOMCAT_HOME/ conf/tomcat-users.xml`  实现的. 

  ```xml
    <!-- Global JNDI resources
         Documentation at /docs/jndi-resources-howto.html
    -->
    <GlobalNamingResources>
      <!-- Editable user database that can also be used by
           UserDatabaseRealm to authenticate users
      -->
      <Resource name="UserDatabase" auth="Container"
                type="org.apache.catalina.UserDatabase"
                description="User database that can be updated and saved"
                factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
                pathname="conf/tomcat-users.xml" />
    </GlobalNamingResources>
  ```

- `Valve`

  ```XML
         <!-- Access log processes all example.
               Documentation at: /docs/config/valve.html
               Note: The pattern used is equivalent to using pattern="common" -->
          <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
                 prefix="localhost_access_log" suffix=".txt"
                 pattern="%h %l %u %t &quot;%r&quot; %s %b" />
  ```

- `Realm`

  `Realm` 可以理解为"域", `Realm` 提供了一种用户密码与`web`应用的映射关系,从而达到角色安全管理的作用,在本例中, `Realm` 的配置使用`name`为`UserDatabase`的资源实现,而该资源在`Server`元素中使用`GlobalNamingResources` 配置

  ```xml
        <!-- Use the LockOutRealm to prevent attempts to guess user passwords
             via a brute-force attack -->
        <Realm className="org.apache.catalina.realm.LockOutRealm">
          <!-- This Realm uses the UserDatabase configured in the global JNDI
               resources under the key "UserDatabase".  Any edits
               that are performed against this UserDatabase are immediately
               available for use by the Realm.  -->
          <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
                 resourceName="UserDatabase"/>
        </Realm>
  ```



#### 2.1.3 `conf/web.xml`

全局的`web.xml` 文件有些标签用不到,可以删除掉



####  2.1.4 JVM层面

因为`Tomcat`本身就是一个`java`进程,所以可以使用优化JVM的方式进行优化

## 3. 配置优化

### 3.1 减少`web.xml/server.xml`中标签

最终观察`tomcat` 启动日志(时间/内容)、线程开销、内存大小、GC等. 



- `DefaultServlet`

  官网 :User Guide->Default Servlet

  > The default servlet is the servlet which serves static resources as well as serves the directory listings (if directory listings are enabled).

  ```xml
      <servlet>
          <servlet-name>default</servlet-name>
          <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
          <init-param>
              <param-name>debug</param-name>
              <param-value>0</param-value>
          </init-param>
          <init-param>
              <param-name>listings</param-name>
              <param-value>false</param-value>
          </init-param>
          <load-on-startup>1</load-on-startup>
      </servlet>
  ```

- `JspServlet`

   ```xml
      <servlet>
          <servlet-name>jsp</servlet-name>
          <servlet-class>org.apache.jasper.servlet.JspServlet</servlet-class>
          <init-param>
              <param-name>fork</param-name>
              <param-value>false</param-value>
          </init-param>
          <init-param>
              <param-name>xpoweredBy</param-name>
              <param-value>false</param-value>
          </init-param>
          <load-on-startup>3</load-on-startup>
      </servlet>
    
      <servlet-mapping>
          <servlet-name>default</servlet-name>
          <url-pattern>/</url-pattern>
      </servlet-mapping>
    
      <!-- The mappings for the JSP servlet -->
      <servlet-mapping>
          <servlet-name>jsp</servlet-name>
          <url-pattern>*.jsp</url-pattern>
          <url-pattern>*.jspx</url-pattern>
      </servlet-mapping>
   ```

- `welcome-list-file`

  ```xml
      <welcome-file-list>
          <welcome-file>index.html</welcome-file>
          <welcome-file>index.htm</welcome-file>
          <welcome-file>index.jsp</welcome-file>
      </welcome-file-list>
  
  ```

- `mime-mapping`移除响应的内容

  支持的下载打开类型

  ```xml
  
      <mime-mapping>
          <extension>123</extension>
          <mime-type>application/vnd.lotus-1-2-3</mime-type>
      </mime-mapping>
      <mime-mapping>
          <extension>3dml</extension>
          <mime-type>text/vnd.in3d.3dml</mime-type>
      </mime-mapping>
  ....
  ```

- `session-config` 

  默认的`jsp` 页面有`session`, 就是在于这个配置

  ```xml
      <session-config>
          <session-timeout>30</session-timeout>
      </session-config>
  
  ```



###  3.2 调整优化`server.xml` 中标签

#### 3.2.1 `Connector` 标签

- `protocol` 属性

  ```xml
   <Connector port="8080" protocol="HTTP/1.1"
                 connectionTimeout="20000"
                 redirectPort="8443" />
  ```

  对于`protocol="HTTP/1.1"` ,查看源码

```java
 public Connector(String protocol) {
        setProtocol(protocol);
        // Instantiate protocol handler
        ProtocolHandler p = null;
        try {
            Class<?> clazz = Class.forName(protocolHandlerClassName);
            p = (ProtocolHandler) clazz.newInstance();
        } catch (Exception e) {
            log.error(sm.getString(
                    "coyoteConnector.protocolHandlerInstantiationFailed"), e);
        } finally {
            this.protocolHandler = p;
        }

        if (!Globals.STRICT_SERVLET_COMPLIANCE) {
            URIEncoding = "UTF-8";
            URIEncodingLower = URIEncoding.toLowerCase(Locale.ENGLISH);
        }
    }
```

` setProtocol(protocol);`  因为配置文件中传入的是 `HTTP/1.1` 

```java
 if ("HTTP/1.1".equals(protocol)) {
                setProtocolHandlerClassName
                    ("org.apache.coyote.http11.Http11NioProtocol");
            } else if ("AJP/1.3".equals(protocol)) {
                setProtocolHandlerClassName
                    ("org.apache.coyote.ajp.AjpNioProtocol");
            } else if (protocol != null) {
                setProtocolHandlerClassName(protocol);
            }
```

发现这里调用的是`Http11NioProtocol，`, 说明`tomcat8`中默认的是`NIO`

使用同样的方式看`tomcat7`和`tomcat8.5`, 你会发现`tomcat7`中默认使用的是`BIO`,`tomcat8.5` 中默认使用的是`NIO`

1. `BIO`

   来到tomcat官网`Configuration/HTTP/protocol`

   > org.apache.coyote.http11.Http11Protocol - blocking Java connector org.apache.coyote.http11.Http11NioProtocol - non blocking Java NIO connector org.apache.coyote.http11.Http11Nio2Protocol - non blocking Java NIO2 connector org.apache.coyote.http11.Http11AprProtocol - the APR/native connector.

2. `NIO`

   `tomcat8.0` 中默认使用的就是`NIO`

3. `APR`

   调用本地方法库进行`io`操作



- `executor`属性

  最佳线程公式: （(线程等待时间+线程CPU时间)/线程CPU时间）*CPU数量

  > The Executor represents a thread pool that can be shared between components in Tomcat. Historically there has been a thread pool per connector created but this allows you to share a thread pool, between (primarily) connector but also other components when those get configured to support executors

  默认的可以查看`StandardExecutor` 类

  **设置一些属性**

  `官网`： https://tomcat.apache.org/tomcat-8.0-doc/config/http.html

  1. `acceptCount`

      达到最大连接数之后, 等待队列中还能放多少连接, 超过即拒绝, 配置太多也没有意义. 

     > The maximum queue length for incoming connection requests when all possible request processing threads are in use. Any requests received when the queue is full will be refused. The default value is 100.

  2. `maxConnections`

      超过这个值后,将继续接受连接,但是不处理,能继续接受多少根据`acceptCount`的值. 

     `BIO`:`maxThreads`

     `NIO/NIO2`:10000 ——— `AbstractEndpoint.maxConnections`

     `APR`:8192

     > The maximum number of connections that the server will accept and process at any given time. When this number has been reached, the server will accept, but not process, one further connection. This additional connection be blocked until the number of connections being processed falls below maxConnections at which point the server will start accepting and processing new connections again. Note that once the limit has been reached, the operating system may still accept connections based on the acceptCount setting. The default value varies by connector type. For BIO the default is the value of maxThreads unless an Executor is used in which case the default will be the value of maxThreads from the executor. For NIO and NIO2 the default is 10000. For APR/native, the default is 8192. 
     >
     > Note that for APR/native on Windows, the configured value will be reduced to the highest multiple of 1024 that is less than or equal to maxConnections. This is done for performance reasons. If set to a value of -1, the maxConnections feature is disabled and connections are not counted.

  3. `maxThreads`

     最大工作线程数,也就是用来处理`request`请求的,默认是200, 如果自己配置了`executor`, 并且和`Connector`有关联了,则之前默认的200就会被忽略,取决于CPU的配置. 监控中就可以看到所有的工作线程是什么状态, 通过监控就可以知道开启多少线程合适. 
     
     > The maximum number of request processing threads to be created by this Connector, which therefore determines the maximum number of simultaneous requests that can be handled. If not specified, this attribute is set to 200. If an executor is associated with this connector, this attribute is ignored as the connector will execute tasks using the executor rather than an internal thread pool. Note that if an executor is configured any value set for this attribute will be recorded correctly but it will be reported (e.g. via JMX) as -1 to make clear that it is not used.
     
  4. `minSpareThreads`
  
      最小空闲线程数
  
      > The minimum number of threads always kept running. This includes both active and idle threads. If not specified, the default of 10 is used. If an executor is associated with this connector, this attribute is ignored as the connector will execute tasks using the executor rather than an internal thread pool. Note that if an executor is configured any value set for this attribute will be recorded correctly but it will be reported (e.g. via JMX) as -1 to make clear that it is not used.
  
      可以实践一下,`connector` 配置自定义的线程池
  
      ```xml
         <Connector port="8080" protocol="HTTP/1.1"
                     connectionTimeout="20000"
                     redirectPort="8443" />
      ```
  
      ```xml
         <Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
              maxThreads="150" minSpareThreads="4"/>
      ```
  
      其实这块最好的方式就是结合`BIO`来查看,因为`BIO`是一个`request` 对应一个线程. 
  
      > 值太低, 并发请求多了之后, 多余的则会进入到等待状态. 
      >
      > 值太高,启动`tomcat` 将花费更多的时间
  
  5. `enableLookups`
  
       设置为`fasle`
  
  6. 删掉`AJP`的`Connector`
  
  
  
#### 3.2.2 `Host` 标签

- `autoDeploy`

  `tomcat` 运行时,要用一个线程来进行检查, 生产环境下一定要改成`fasle`

  > This flag value indicates if Tomcat should check periodically for new or updated web applications while Tomcat is running. If true, Tomcat periodically checks the appBase and xmlBase directories and deploys any new web applications or context XML descriptors found. Updated web applications or context XML descriptors will trigger a reload of the web application. The flag's value defaults to true. See Automatic Application Deployment for more information.



#### 3.2.3 `Context` 标签

`reloadable:false`

`reloadable` 如果设置为`true`, `tomcat` 服务器会在运行状态下监视在`WEB-INFO/classes`和`WEB-INFO/lib`  目录下`class`文件的改动,如果检测到有`class` 文件被更新,服务器会自动加载`web`应用. 

在开发阶段, 将`reloadable` 属性设置为`true`, 有助于调试`servlet`和其他的`class`文件,但是这样会加重服务器运行负荷,建议在`web` 应用的发行阶段将`reloadable`设置为`false`

> Set to true if you want Catalina to monitor classes in /WEB-INF/classes/ and /WEB-INF/lib for changes, and automatically reload the web application if a change is detected. This feature is very useful during application development, but it requires significant runtime overhead and is not recommended for use on deployed production applications. That's why the default setting for this attribute is false. You can use the Manager web application, however, to trigger reloads of deployed applications on demand.





## 4. 启动速度慢优化

- 删除没用的`web`应用

  因为`tomcat`启动每次都会部署这些应用

- 关闭`webSocket`

  如果项目中用不到`webSocket`的话，可以删除这两个`websocket-api.jar`和`tomcat-websocket.jar`

- 随机数优化

  设置`JVM`参数

  ```bash
  -Djava.security.egd=file:/dev/./urandom
  ```

- 多个线程启动`web`应用

  ```xml
  <Host startStopThreads="0">
  </Host>
  ```



## 5.  其他方面的优化

- `Connector`

  配置压缩属性`compression="500"`, 文件大于500bytes 才会被压缩

- 数据库优化

  减少对数据库访问等待的时间,可以从数据库的层面进行优化,或者加缓存等等各种方案

- 开启浏览器缓存,`nginx` 静态资源部署





## 6. 常见问题排查

### 6.1 CPU 使用率过高

**可能原因**

GC频繁或者创建了很多的业务线程

**排查**

哪些线程比较消耗CPU 或者多线程上下文频繁切换

**解决思路**

`top -H pid` 查看某个java进程各个线程使用CPU的情况,找到哪个线程占用CPU比较高. 

`jstack pid` 打印出线程信息,定位到上述线程名称



### 6.2 拒绝连接

- `java.net.BindException: Address already in use: JVM_Bind`

   端口被占用,可以使用`netstat -an` 查看端口占用情况,关闭对应的进程或者`tomcat`修改端口

- `java.net.ConnectException: Connection refused: connect`

  `ping`一下服务端的`ip`,可能服务端机器有问题

- `java.net.SocketException: Too many open files`

  可能在高并发的情况下,创建的`socket`过多,文件句柄不够用了,可以关闭无用的句柄,如果都有用,可以增加文件句柄数 `ulimit -n 10000`







 