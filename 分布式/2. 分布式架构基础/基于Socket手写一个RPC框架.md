# 这里基于Socket 手写一个五脏俱全的RPC 框架

##  服务端

我们这里先建两个项目

rpc-server-api : 用于存放需要对外的接口类

rpc-server-prodiver： 存放业务代码的项目

### rpc-server-api

先在项目中加入spring的 pom依赖

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>4.2.8.RELEASE</version>
        </dependency>

    </dependencies>

```



我们这里先建一个注解类

```java
@Component
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface RpcService {

    Class<?> value();

    /**
     * <p>版本号</p>
     *
     * @return {@link String}
     * @author luyanan
     * @since 2019/9/4
     */
    String version() default "";

}

```

再将一个  请求封装类

```java

/**
 * @author luyanan
 * @since 2019/9/3
 * <p>rpc 请求封装</p>
 **/
public class RpcRequest implements Serializable {

    /**
     * <p>类名</p>
     *
     * @author luyanan
     * @since 2019/9/3
     */
    private String className;


    /**
     * <p>方法名</p>
     *
     * @author luyanan
     * @since 2019/9/3
     */
    private String methodName;


    /**
     * <p>参数列表</p>
     *
     * @author luyanan
     * @since 2019/9/3
     */
    private Object[] params;


    public String getClassName() {
        return className;
    }

    public void setClassName(String className) {
        this.className = className;
    }

    public String getMethodName() {
        return methodName;
    }

    public void setMethodName(String methodName) {
        this.methodName = methodName;
    }

    public Object[] getParams() {
        return params;
    }

    public void setParams(Object[] params) {
        this.params = params;
    }
}

```

然后再建 一个 User 对象和IUserService

```java
public class User {

    private String name;

    private Integer age;


    public User(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

IUserService

```java

public interface IUserService {


    String saveUser(User user);


    String info(String id);

}

```

### rpc-server-prodiver

实现IUserService的业务

UserServiceImpl

```java
@RpcService(IUserService.class)
public class UserServiceImpl implements IUserService {


    @Override
    public String saveUser(User user) {
        System.out.println("保存的用户为:" + user);
        return "success";
    }

    @Override
    public String info(String id) {
        return "id:Tom";
    }
}

```

接下来是有关socket  通信的代码

RpcService

```java
@Component
public class RpcService implements InitializingBean, ApplicationContextAware {

    private int port;

    public RpcService() {
    }

    static ExecutorService executorService = Executors.newCachedThreadPool();
    static Map<String, Object> handlerMap = new HashMap<>();

    public RpcService(int port) {
        this.port = port;
    }

    @Override
    public void afterPropertiesSet() throws Exception {

        ServerSocket serverSocket = null;
        try {
            serverSocket = new ServerSocket(port);
            while (true) {
                Socket socket = serverSocket.accept();
                executorService.submit(new SocketHandler(handlerMap, socket));
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (null != serverSocket) {
                try {
                    serverSocket.close();
                } catch (IOException e) {

                }
            }
        }


    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {

        Map<String, Object> objectMap = applicationContext.getBeansWithAnnotation(com.formula.rpc.api.annotaion.RpcService.class);
        for (Object value : objectMap.values()) {
            //  获取注解
            com.formula.rpc.api.annotaion.RpcService rpcService = value.getClass().getAnnotation(com.formula.rpc.api.annotaion.RpcService.class);

            String name = rpcService.value().getName();
            String version = rpcService.version();
            if (!StringUtils.isEmpty(version)) {
                name += "-" + version;
            }
            handlerMap.put(name, value);
        }
    }



}

```

SocketHandler

```java
package com.formula.rpc.provider;

import com.formula.rpc.api.RpcRequest;

import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.net.Socket;
import java.util.HashMap;
import java.util.Map;

/**
 * @author luyanan
 * @since 2019/9/3
 * <p>处理</p>
 **/
public class SocketHandler implements Runnable {

    Map<String, Object> handlerMap = null;

    private Socket socket;

    public SocketHandler(Map<String, Object> handlerMap, Socket socket) {
        this.handlerMap = handlerMap;
        this.socket = socket;
    }

    @Override
    public void run() {


        ObjectInputStream objectInputStream = null;
        ObjectOutputStream objectOutputStream = null;
        try {
            objectInputStream = new ObjectInputStream(socket.getInputStream());
            RpcRequest request = (RpcRequest) objectInputStream.readObject();
            //  利用反射调用
            Class clazz = Class.forName(request.getClassName());
            Class<?>[] types = new Class[request.getParams().length];
            for (int i = 0; i < request.getParams().length; i++) {
                types[i] = request.getParams()[i].getClass();
            }

            Method method = clazz.getMethod(request.getMethodName(), types);
            Object result = method.invoke(handlerMap.get(request.getClassName()), request.getParams());
            objectOutputStream = new ObjectOutputStream(socket.getOutputStream());
            objectOutputStream.writeObject(result);
            objectOutputStream.flush();

        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } finally {
            if (objectInputStream != null) {
                try {
                    objectInputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (objectOutputStream != null) {
                try {
                    objectOutputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }


    }
}

```

配置类 RpcConfig

```
package com.formula.rpc.provider;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

/**
 * @author luyanan
 * @since 2019/9/4
 * <p></p>
 **/
@ComponentScan(basePackages = "com.formula.rpc")
@Configuration
public class RpcConfig {

    @Bean
    public RpcService rpcService() {
        return new RpcService(8080);
    }

}

```



启动类  ServerAPP

```
package com.formula.rpc.provider;

import com.formula.rpc.api.IUserService;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

/**
 * @author luyanan
 * @since 2019/9/4
 * <p></p>
 **/
public class ServerAPP {

    public static void main(String[] args) {

        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(RpcConfig.class);
        applicationContext.start();
    }

}

```



##  客户端

这里使用一个动态代理来实现远程调用

首先要在客户端的poml 里面加入 api的依赖

```xml
     <dependency>
            <groupId>com.formula</groupId>
            <artifactId>rpc-server-api</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
```



RpcProxyClient

```java
package com.formula.rpc.client;

import java.lang.reflect.Proxy;

/**
 * @author luyanan
 * @since 2019/9/4
 * <p></p>
 **/
public class RpcProxyClient {


    public <T> T getService(Class<?> interfaceClss, String host, int port) {
        return (T) Proxy.newProxyInstance(interfaceClss.getClassLoader(),
                new Class[]{interfaceClss}, new RemoteInvocationHandler(host, port));
    }

}

```

RemoteInvocationHandler

```java
package com.formula.rpc.client;

import com.formula.rpc.api.RpcRequest;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

/**
 * @author luyanan
 * @since 2019/9/4
 * <p></p>
 **/
public class RemoteInvocationHandler implements InvocationHandler {


    private String host;

    private int port;

    public RemoteInvocationHandler(String host, int port) {
        this.host = host;
        this.port = port;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //  这是使用socket发送请求

        if (method.getName().equals("toString")) {
            return method.invoke(method.getDeclaringClass(), args);
        }
        RpcRequest request = new RpcRequest();
        request.setClassName(method.getDeclaringClass().getName());
        request.setMethodName(method.getName());
        request.setParams(args);
        RpcNetTransport netTransport = new RpcNetTransport(host, port);
        return netTransport.send(request);
    }
}

```

 主启动类

````java
package com.formula.rpc.client;

import com.formula.rpc.api.IUserService;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

/**
 * @author luyanan
 * @since 2019/9/4
 * <p></p>
 **/
public class ClientAPP {


    public static void main(String[] args) {
        RpcProxyClient proxyClient = new RpcProxyClient();
        IUserService userService = proxyClient.getService(IUserService.class, "localhost", 8080);
        System.out.println(userService.info("1"));

    }

}

````

