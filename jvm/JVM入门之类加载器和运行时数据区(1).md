#  JVM入门之类加载器和运行时数据区

##  1.  官网

### 1.1 `JDK8`

> `官网`: https://docs.oracle.com/javase/8/



### 1.2 The relation of JDK/JRE/JVM

`Reference` -> `Developer Guides` -> 定位到:`https://docs.oracle.com/javase/8/docs/index.html`

> Oracle has two products that implement Java Platform Standard Edition (Java SE) 8: Java SE Development Kit (JDK) 8 and Java SE Runtime Environment (JRE) 8.
>
> JDK 8 is a superset of JRE 8, and contains everything that is in JRE 8, plus tools such as the compilers and debuggers necessary for developing applets and applications. JRE 8 provides the libraries, the Java Virtual Machine (JVM), and other components to run applets and applications written in the Java programming language. Note that the JRE includes components not required by the Java SE specification, including both standard and non-standard Java components.
>
> The following conceptual diagram illustrates the components of Oracle's Java SE products:

![image-20200507090556779](http://files.luyanan.com//img/20200507090612.png)



##  2.源码到类文件

### 2.1 源码

准备一个`Person` 的类

```java
public class Person {
    private String name;
    private int age;
    private static String address;
    private final static String hobby = "Programming";

    public void say() {
        System.out.println("person say...");
    }

    public int calc(int op1, int op2) {
        return op1 + op2;
    }

}
```

> 编译:  `javac Person.java ->Person.class`



### 2.2 编译过程

`Person.java` -> `语法分析器` -> `tokens流` -> `语法分析器` ->  `语法树/抽象语法树` -> `语法分析器` -> `注解抽象语法树` ->`字节码生成器` -> `Person.class 文件`



### 2.3   类文件(`class` 文件)

> 官网`The class File Format `:  https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html

```text
cafe babe 0000 0033 002c 0a00 0600 1d09
001e 001f 0800 200a 0021 0022 0700 2307
0024 0100 046e 616d 6501 0012 4c6a 6176
612f 6c61 6e67 2f53 7472 696e 673b 0100
0361 6765 0100 0149 0100 0761 6464 7265
7373 0100 0568 6f62 6279 0100 0d43 6f6e
7374 616e 7456 616c 7565 0800 2501 0006
3c69 6e69 743e 0100 0328 2956 0100 0443
6f64 6501 000f 4c69 6e65 4e75 6d62 6572
5461 626c 6501 0012 4c6f 6361 6c56 6172
6961 626c 6554 6162 6c65 0100 0474 6869
7301 0010 4c63 6f6d 2f6a 766d 2f50 6572
736f 6e3b 0100 0373 6179 0100 0463 616c
6301 0005 2849 4929 4901 0003 6f70 3101
0003 6f70 3201 000a 536f 7572 6365 4669
6c65 0100 0b50 6572 736f 6e2e 6a61 7661
0c00 0f00 1007 0026 0c00 2700 2801 000d
......
```

**`magic`(魔数):**

> The `magic` item supplies the magic number identifying the `class` file format; it has the value `0xCAFEBABE`.

```text
cafe babe
```

**`minor_version`,` major_version`**

```text
0000 0034 对应10进制的52，代表JDK 8中的一个版本

```

 **`constant_pool_count`**

```text
0027 对应十进制27，代表常量池中27个常量

```

```text
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

> .class 字节码文件
>
> - 魔数与`class` 文件版本
> - 常量池
> - 访问标识
> - 类索引、父类索引、接口索引
> - 字段表集合
> - 方法表集合
> - 属性表集合





## 3. 类文件到虚拟机(类加载过程)

### 3.1 装载(`Load`)

> 查找和导入`class`  文件

1. 通过一个类的全限定名获取定义此类的二进制字节流
2. 将这个字节流所代表的静态存储结构转化为方法去的运行时数据结构
3. 在`java` 堆中生成一个代表这个类的`java.lang.Class` 对象, 作为对方法区中这些数据的访问入口

### 3.2 链接(`Link`)

####  3.2.1  验证(`Verify`)

> 保证被加载类的正确性

- 文件格式验证
- 元数据验证
- 字节码验证
- 符号引用验证



#### 3.2.2 准备(`Prepare`)

> 为类的静态变量分配内存,并将其初始化为默认值



#### 3.2.3 初始化(`Initialize`)

> 对类的静态变量、静态代码块执行初始化操作



### 3.4 类加载机制图解

> 使用和卸载并不算是类加载过程,只是画完整了一下

![image-20200507110040791](http://files.luyanan.com//img/20200507110042.png)



## 4. 类装载器`ClassLoader`

在装载(`Load`) 阶段,其中第(1) 步同故宫类的全限定名获取其定义的二进制字节流,需要借助类装载器完成, 顾名思义,就是用来装载`Class`  文件的. 

1. 通过一个类的全限定名获取定义此类的二进制字节流



###  4.1 分类

1. `Bootstrap ClassLoader` 负责加载`$JAVA_HOME` 中`jre/lib/rt.jar` 里面所有的`class` 或`Xbootclassoath` 选项指定的`jar`包,由`C++` 实现,不是`ClassLoader` 子类
2. `Extension ClassLoader` 负责加载`java`  平台中扩展功能的一些`jar`包. 包括`$JAVA_HOME` 中`jre/lib/*.jar` 或`-Djava.ext.dirs` 指定目录下的`jar`包. 
3. `App ClassLoader` 负责加载`classpath` 中指定的`jar` 包以及`-Djava.class.path`  所指定目录下的类和`jar` 包
4. `Custom ClassLoader` 通过`java.lang.ClassLoader` 的子类自定义加载`class`,属于应用策划稿内需根据自身需要自定义的`classLoader`,如`tomcat`、`jboss` 都会根据`j2ee` 规范自动实现`classLoader`



### 4.2  图解

![image-20200507113213585](http://files.luyanan.com//img/20200507113214.png)

### 4.3  加载原理

检查某个类是否已经加载,顺序是自底向上的,从`Costom ClassLoader`到`Bootstrap ClassLoader`逐层检查,只要某个`ClassLoader` 已经加载,就视为已经加载此类,保证此类所以的`ClassLoader` 只加载一次. 

加载的顺序是自顶向下的,也就是由上层来逐层的尝试加载此类. 

**双亲委派模式**

**定义**: 如果一个类加载器在接收到加载类的请求后, 它首先不会自己去尝试去加载这个类,而是把这个请求任务委托给父类加载器去完成,依次递归, 如果父类加载器可以完成类加载任务,就成功返回,只有父类加载器无法完成此加载任务的时候,才自己去加载. 

**优势**:java类随着加载它的类加载器一起具备了一种带有优先级的层次关系,比如,`Java` 中的`Object`类, 它存放在`rt.jar` 之中, 无论哪一个类加载器要加载此类, 最终都是委派给处于模型最顶端的启动类加载器进行加载,因此`Object` 类在各种类加载环境中都是同一个类. 如果不采用双亲委派模型,那么由各个类加载器自己加载的话,那么系统中就会存在多种不同的`Object` 类. 

**破坏:** 可以继承`ClassLoader` 类, 然后重写 其中的`loadClass` 方法,其他方式大家可以自己了解扩展一下. 





## 5 运行时数据区(`Run-Time Data Areas`)

在装载阶段的第2、3步, 可以发现有运行时数据、堆、方法区等名词

2. 将这个字节流所代表的静态存储结构转化位方法区的运行时数据结构
3. 在java堆中生成一个代表这个类的`java.lang.Class` 对象,作为对方法区中这些数据的访问入口

说白了,就是类文件被类装载器装载进来之后,类中的内容(比如常量、变量、方法、对象等这些数据总要有个去处,也就是要存储起来, 存储的位置肯定在JVM中有对应的空间)



### 5.1 官网概况

https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5

> The Java Virtual Machine defines various run-time data areas that are used during execution of a program. Some of these data areas are created on Java Virtual Machine start-up and are destroyed only when the Java Virtual Machine exits. Other data areas are per thread. Per-thread data areas are created when a thread is created and destroyed when the thread exits.

### 5.2  图解

![image-20200507134911637](http://files.luyanan.com//img/20200507134912.png)



### 5.3 常规理解

####  5.3.1 `Method Area`(方法区)

方法区是各个线程共享的内存区域,在虚拟机启动时候创建

用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据

虽然java虚拟机规范把方法区描述为堆的一个逻辑部分, 但是它却又一个别名叫做`Non-Heap`(非堆), 目的是与Java 堆区分开来. 

当方法区无法满足内存分配需求的时候,将抛出`OutOfMemoryError` 异常

> The Java Virtual Machine has a *method area* that is shared among all Java Virtual Machine threads. The method area is analogous to the storage area for compiled code of a conventional language or analogous to the "text" segment in an operating system process. It stores per-class structures such as the run-time constant pool, field and method data, and the code for methods and constructors, including the special methods ([§2.9](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.9)) used in class and instance initialization and interface initialization.

> **If memory in the method area cannot be made available to satisfy an allocation request, the Java Virtual Machine throws an** `OutOfMemoryError`**.**



此时回看装载阶段的第二步: 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构

> 如果这个时候把从`Class` 文件到装载的第1、2步合并起来理解的话, 可以画个图

![image-20200507135435696](http://files.luyanan.com//img/20200507135436.png)



> 值得说明的是

1. 方法区在`JDK1.8`z  中就是`Metaspace`, `jdk1.6或7` 中就是`Perm Space`

2. `Run-Time Constant Pool`

    `Class` 文件中除了有类的版本、字段、方法、接口描述信息外，还有一项信息就是常量池,用于存放编译时期生成的各种字面量和符号引用,这部分内容将在类加载后进入方法区的运行时常量池中存放. 

   > Each run-time constant pool is allocated from the Java Virtual Machine's method area (§2.5.4).s



####  5.3.2 `Heap`(堆)

java堆是java虚拟机所管理的内存中最大的一块,在虚拟机启动的时候创建, 被所有线程共享

java对象实例以及数组都在堆上进行分配. 

> The Java Virtual Machine has a *heap* that is shared among all Java Virtual Machine threads. The heap is the run-time data area from which memory for all class instances and arrays is allocated.
>
> The heap is created on virtual machine start-up.

此时回看装载阶段的第三步: 在java 堆中生成一个代表这个类的`java.lang.Class` 对象,作为方法区中这些数据的入口

![image-20200507140154830](http://files.luyanan.com//img/20200507140155.png)





####  5.3.3 ` Java Virtual Machine Stacks`(java虚拟机栈)

经过上面的分析,类加载机制的装载过程已经完成,后续的链接、初始化也会相应的生效. 

假如目前的阶段是初始化完成了,后续做啥呢? 肯定是`User`使用了, 不用的话这样折腾来折腾去有什么意义呢? 那怎样才能被使用到呢? 换句话说里面的内容怎么样才能被执行呢? 比如通过主函数`main`调用其他的方法,这种方法实际上是`main`线程执行之后调用的方法,即要想使用里面的内容,得要以线程为单位,执行相应的方法才行. 

那一个线程执行的状态是如何维护的呢? 一个线程可以执行多少方法? 这样的关系是如何维护的呢? 

虚拟机栈是一个线程执行的区域,保存着一个线程中方法的调用状态, 换句话说,一个java线程的运行状态,由一个虚拟机栈来保存,所以虚拟机栈肯定是线程私有的,独有的,随着线程的创建而创建. 

每一个被线程执行的方法,为该栈的栈帧,即每一个方法对应一个栈帧. 

调用一个方法, 就会向栈中压入一个栈帧,一个方法调用完成后,就会把该栈帧从栈中弹出. 

> Each Java Virtual Machine thread has a private *Java Virtual Machine stack*, created at the same time as the thread. A Java Virtual Machine stack stores frames ([§2.6](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6)). 

**画图理解栈和栈帧**

![image-20200507141043336](http://files.luyanan.com//img/20200507141044.png)



####  5.3.4 ` The pc Register`(程序计数器)

我们都知道一个JVM 进程中有多个线程在执行,而线程中的内容是否能够拥有执行权, 是根据CPU调度来的. 

假如线程A正在执行到某个地方,突然失去了CPU的执行权,切换到线程B了,然后当线程A 重新获取到CPU执行权的时候,怎么才能继续执行呢? 这就是需要在线程中维护一个变量,记录线程执行到的位置. 

程序计数器占用的内存很小,由于java虚拟机的多线程是通过线程轮流切换, 并分配处理器执行时间的方法来实现的,在任意时刻,一个处理器只会执行一个线程中的指令. 因此,为了线程切换后能够恢复到正确的执行位置,每条线程需要有一个独立的程序计数器(线程私有). 

如果线程正在执行java方法,则计数器记录的是正在执行的虚拟机字节码指令的地址, 

如果执行的是`Native` 方法, 则这个计数器为空. 

> The Java Virtual Machine can support many threads of execution at once (JLS §17). Each Java Virtual Machine thread has its own pc (program counter) register. At any point, each Java Virtual Machine thread is executing the code of a single method, namely the current method (§2.6) for that thread. If that method is not native, the pc register contains the address of the Java Virtual Machine instruction currently being executed. If the method currently being executed by the thread is native, the value of the Java Virtual Machine's pc register is undefined. The Java Virtual Machine's pc register is wide enough to hold a returnAddress or a native pointer on the specific platform.



####  5.3.5 ` Native Method Stacks`(本地方法栈)

如果当前线程执行的方法是`Native`类型的,这些方法就会在本地方法栈中执行. 

