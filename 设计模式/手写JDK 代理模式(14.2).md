#  14.2 手写JDK代理模式

### 1. 原理

我们直到JDK Proxy采用字节码重组,重新生成新的对象来替代原来的对象来达到动态代理的目的.JDK Proxy生成对象的步骤如下:
1. 拿到被代理对象的引用,并且获取他的所有接口,通过反射获取.
2. JDK Proxy类重新生成一个新的类,同时新的类要实现被代理的类的所有实现的接口.
3. 动态生成新的Java代码,把新加的业务逻辑代码方法由一定的逻辑代码去调用,
4. 编译新生成的java代码 .class
5. 再重新加载到JVM中运行.
6. 以上的过程就叫做字节码重组,JDK中有一个规范,在classpath下只要是$开头的class文件一般都是自动生成的.那么我们有没有办法看到代替后的对象的阵容呢,？做一个这
样测试，我们从内存中的对象字节码通过文件流输出到一个新的 class 文件，然后，利用
反编译工具查看 class 的源代码。来看测试代码:
```java
  System.out.println("------------Cglib代理------------");
        //JDK是采用读取接口的信息
        //CGLib覆盖父类方法
        //目的：都是生成一个新的类，去实现增强代码逻辑的功能

        //JDK Proxy 对于用户而言，必须要有一个接口实现，目标类相对来说复杂
        //CGLib 可以代理任意一个普通的类，没有任何要求

        //CGLib 生成代理逻辑更复杂，效率,调用效率更高，生成一个包含了所有的逻辑的FastClass，不再需要反射调用
        //JDK Proxy生成代理的逻辑简单，执行效率相对要低，每次都要反射动态调用

        //CGLib 有个坑，CGLib不能代理final的方法

        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "D://cglib_proxy_classes");

        RealSubject realSubject = new RealSubject();
        CglibProxyFactory cglibProxyFactory = new CglibProxyFactory(realSubject);
        RealSubject cglinProxy = (RealSubject) cglibProxyFactory.getProxyInstance();
        cglinProxy.save();
        byte[] bytes = ProxyGenerator.generateProxyClass("$Proxy0", new Class[]{RealSubject.class});
        try {
            FileOutputStream fileOutputStream = new FileOutputStream("D://$Proxy0.class");
            fileOutputStream.write(bytes);
            fileOutputStream.close();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }

```
运行之后在D盘可以找到一个$Proxy.class的文件，拖到idea中可以看到如下内容
```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

import com.notes.pattern.proxy.RealSubject;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements RealSubject {
    private static Method m1;
    private static Method m8;
    private static Method m3;
    private static Method m2;
    private static Method m6;
    private static Method m5;
    private static Method m7;
    private static Method m9;
    private static Method m0;
    private static Method m4;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void notify() throws  {
        try {
            super.h.invoke(this, m8, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void save() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void wait(long var1) throws InterruptedException {
        try {
            super.h.invoke(this, m6, new Object[]{var1});
        } catch (RuntimeException | InterruptedException | Error var4) {
            throw var4;
        } catch (Throwable var5) {
            throw new UndeclaredThrowableException(var5);
        }
    }

    public final void wait(long var1, int var3) throws InterruptedException {
        try {
            super.h.invoke(this, m5, new Object[]{var1, var3});
        } catch (RuntimeException | InterruptedException | Error var5) {
            throw var5;
        } catch (Throwable var6) {
            throw new UndeclaredThrowableException(var6);
        }
    }

    public final Class getClass() throws  {
        try {
            return (Class)super.h.invoke(this, m7, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void notifyAll() throws  {
        try {
            super.h.invoke(this, m9, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void wait() throws InterruptedException {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | InterruptedException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m8 = Class.forName("com.notes.pattern.proxy.RealSubject").getMethod("notify");
            m3 = Class.forName("com.notes.pattern.proxy.RealSubject").getMethod("save");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m6 = Class.forName("com.notes.pattern.proxy.RealSubject").getMethod("wait", Long.TYPE);
            m5 = Class.forName("com.notes.pattern.proxy.RealSubject").getMethod("wait", Long.TYPE, Integer.TYPE);
            m7 = Class.forName("com.notes.pattern.proxy.RealSubject").getMethod("getClass");
            m9 = Class.forName("com.notes.pattern.proxy.RealSubject").getMethod("notifyAll");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
            m4 = Class.forName("com.notes.pattern.proxy.RealSubject").getMethod("wait");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

```
我们发现$Proxy0继承了Proxy类,同时还实现了我们的RealSubject接口,而且还重写了save()等方法,而且在静态块中用反射查找了目标对象的所有方法,而且保存了所有方法的引用,在重写的方法用反射调用目标对象的方法.这些代码是由JDK帮我们自动生成的.
接下来我们不依赖JDK动态生成源代码,动态完成编译,然后替代目标对象并执行.
<br>
首先在pom中添加
```java
     <dependency>
             <groupId>com.sun</groupId>
             <artifactId>tools</artifactId>
             <version>1.6</version>
             <scope>system</scope>
             <systemPath>${JAVA_HOME}/lib/tools.jar</systemPath>
        </dependency>
```
创建MyInvocationHandler对象
```java
package com.notes.pattern.proxy.myproxy;

import java.lang.reflect.Method;

/**
 * @author:luyanan
 * @email:luyanan0718@163.com
 * @date 2019/4/10
 * @introduce
 **/
public interface MyInvocationHandler {
    public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable;
}

```
创建MyClassLoader类
```java
package com.notes.pattern.proxy.myproxy;

import java.io.*;

/**
 * @author:luyanan
 * @email:luyanan0718@163.com
 * @date 2019/4/10
 * @introduce 类加载器
 **/
public class MyClassLoader extends ClassLoader {


    private File classFilePath;

    public MyClassLoader() {
        String path = MyClassLoader.class.getResource("").getPath();
        this.classFilePath = new File(path);
    }


    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {

        String className = MyClassLoader.class.getPackage().getName() + "." + name;

        if (null != classFilePath) {

            File classFile = new File(classFilePath, name.replaceAll("\\.", "/")+".class");

            if (classFile.exists()) {

                FileInputStream fis = null;
                ByteArrayOutputStream bos = null;

                try {
                    fis = new FileInputStream(classFile);
                    bos = new ByteArrayOutputStream();

                    byte[] buff = new byte[1024];
                    int len;

                    while ((len = fis.read(buff)) != -1) {
                        bos.write(buff, 0, len);
                    }
                    return defineClass(className, bos.toByteArray(), 0, bos.size());

                } catch (FileNotFoundException e) {
                    e.printStackTrace();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

        }
        return null;
    }
}

```

创建MyProxy类
```java
package com.notes.pattern.proxy.myproxy;

import com.notes.pattern.proxy.RealSubject;

import javax.tools.JavaCompiler;
import javax.tools.JavaFileObject;
import javax.tools.StandardJavaFileManager;
import javax.tools.ToolProvider;
import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;

/**
 * @author:luyanan
 * @email:luyanan0718@163.com
 * @date 2019/4/11
 * @introduce 用来生成源码的工具类
 **/
public class MyProxy {

    public static final String ln = "\r\n";
    public static final String pkg = "package ";

    public static Object newProxyInstance(MyClassLoader classLoader,
                                          Class<?>[] interfaces, MyInvocationHandler h) {

        // 1. 动态的生成源代码.java文件
        String src = generateSrc(interfaces);
        System.out.println(src);
        // 2.  java文件输出到磁盘
        try {
            String filePath = MyProxy.class.getResource("").getPath();

            File f = new File(filePath + "$Proxy0.java");
            FileWriter fileWriter = new FileWriter(f);
            fileWriter.write(src);
            fileWriter.flush();
            fileWriter.close();
            // 3. 把生成的.java文件编译成 .class文件


            JavaCompiler javaCompiler = ToolProvider.getSystemJavaCompiler();
            StandardJavaFileManager manage = javaCompiler.getStandardFileManager(null, null, null);


            Iterable<? extends JavaFileObject> iterable = manage.getJavaFileObjects(f);


            JavaCompiler.CompilationTask task = javaCompiler.getTask(null, manage, null, null, null, iterable);

            task.call();
            manage.close();
            // 4. 将编译好的.class文件加载到JVM 内存中

            Class<?> proxyClass = classLoader.findClass("$Proxy0");

            Constructor<?> constructor = proxyClass.getConstructor(MyInvocationHandler.class);
            f.delete();

            // 5. 返回字节码重组后的新的代码对象
            return constructor.newInstance(h);
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }


        return null;
    }

    private static String generateSrc(Class<?>[] interfaces) {
        StringBuffer sb = new StringBuffer();

        sb.append(pkg).append(MyProxy.class.getPackage().getName()).append(";" + ln);
        sb.append("import ").append(interfaces[0].getPackage().getName() +"."+ interfaces[0].getSimpleName()).append(";" + ln);
        sb.append("import java.lang.reflect.*;" + ln);
        sb.append("import com.notes.pattern.proxy.myproxy.MyInvocationHandler;"+ln);
        sb.append("public class $Proxy0 implements " + interfaces[0].getName() + "{" + ln);
        sb.append(" MyInvocationHandler h ;" + ln);
        sb.append("public $Proxy0(MyInvocationHandler h){");
        sb.append(" this.h = h;");
        sb.append("}" + ln);


        for (Method method : interfaces[0].getMethods()) {

            Class<?>[] params = method.getParameterTypes();

            StringBuffer paramNames = new StringBuffer();
            StringBuffer paramsValues = new StringBuffer();
            StringBuffer paramsClasses = new StringBuffer();


            for (int i = 0; i < params.length; i++) {


                Class<?> param = params[i];
                String type = param.getName();
                String paramName = toLowerFirstCase(param.getSimpleName());

                paramNames.append(type + " " + paramName);
                paramsValues.append(paramName);
                paramsClasses.append(param.getName() + ".class");


                if (i > 0 && i < params.length - 1) {

                    paramNames.append(",");
                    paramsClasses.append(",");
                    paramsValues.append(",");

                }
            }


            sb.append("public " + method.getReturnType().getName() + " " + method.getName() + "(" + paramNames.toString() + ") {" + ln);
            sb.append(" try{" + ln);
            sb.append(" Method m = " + interfaces[0].getName() + ".class.getMethod(\"" +
                    method.getName() + "\",new Class[]{" + paramsClasses.toString() + "});" + ln);
            sb.append((hasReturnValue(method.getReturnType()) ? "return " : "") +
                    getCaseCode("this.h.invoke(this,m,new Object[]{" + paramsValues + "})", method.getReturnType()) + ";" + ln);

//            sb.append(")catch(Error _ex){ }");
            sb.append("}catch(Throwable e){" + ln);
            sb.append("throw new UndeclaredThrowableException(e);" + ln);
            sb.append("}");
            sb.append(getReturnEmptyCode(method.getReturnType()));

            sb.append("}");
        }

        sb.append("}" + ln);
        return sb.toString();
    }

    private static Map<Class, Class> mappings = new HashMap<Class, Class>();

    static {
        mappings.put(int.class, Integer.class);
    }

    private static String getReturnEmptyCode(Class<?> returnClass) {
        if (mappings.containsKey(returnClass)) {

            return "return 0;";
        } else if (returnClass == void.class) {

            return "";
        } else {
            return "return null";
        }
    }


    private static String getCaseCode(String code, Class<?> returnClass) {


        if (mappings.containsKey(returnClass)) {
            return "((" + mappings.get(returnClass).getName() + ")" + code + ")." + returnClass.getSimpleName() + "Value()";
        }
        return code;
    }

    private static boolean hasReturnValue(Class<?> clazz) {
        return clazz != void.class;
    }

    private static String toLowerFirstCase(String src) {
        char[] chars = src.toCharArray();
        chars[0] += 32;
        return String.valueOf(chars);
    }

    public static void main(String[] args) {
        System.out.println(generateSrc(new Class[]{RealSubject.class}));
    }

}

```

MyProxyFactory对象
```java
package com.notes.pattern.proxy.myproxy;

import java.lang.reflect.Method;

/**
 * @author:luyanan
 * @email:luyanan0718@163.com
 * @date 2019/4/11
 * @introduce
 **/
public class MyProxyFactory implements MyInvocationHandler {
    private Object target;


    public Object getInstance(Object target) {

        this.target = target;
        Class<?> targetClass = target.getClass();
        return MyProxy.newProxyInstance(new MyClassLoader(), targetClass.getInterfaces(), this);
    }


    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        before();
        Object invoke = method.invoke(this.target, args);
        after();
        return invoke;
    }


    public void before() {
        System.out.println("方法开启之前");
    }

    public void after() {
        System.out.println("方法执行之后");
    }
}


```

测试类
```java

        System.out.println("------------------ 自定义代理----------------------");
        Subject instance = (Subject) new MyProxyFactory().getInstance(new RealSubject());


        instance.save();
```

结果
```java
------------------ 自定义代理----------------------
package com.notes.pattern.proxy.myproxy;
import com.notes.pattern.proxy.Subject;
import java.lang.reflect.*;
import com.notes.pattern.proxy.myproxy.MyInvocationHandler;
public class $Proxy0 implements com.notes.pattern.proxy.Subject{
 MyInvocationHandler h ;
public $Proxy0(MyInvocationHandler h){ this.h = h;}
public void save() {
 try{
 Method m = com.notes.pattern.proxy.Subject.class.getMethod("save",new Class[]{});
this.h.invoke(this,m,new Object[]{});
}catch(Throwable e){
throw new UndeclaredThrowableException(e);
}}}

方法开启之前
保存。。。。。。。
方法执行之后
```

