#  JVM之运行时数据区

##  1. 结合字节码指令理解java虚拟机栈和栈帧

> 官网: https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6

**栈帧**: 每个栈帧对应一个被调用的方法, 可以理解为一个方法的运行空间. 

每个栈帧包括局部变量表(`Local Variables`)、操作数栈(`Operated Stack`)、指向运行时常量池的引用(`(A reference to the run-time constant pool`)、方法返回地址(`Return Address`) 和附加信息

> 局部变量表: 方法中定义的局部变量以及方法的参数都存放在这张表中
>
> 局部变量表中的变量不可以直接使用,如需要使用的话,必须通过相关指令将其加载至操作数栈中作为操作数使用. 

**操作数栈**: 以压栈和出栈的方式存储操作数

**动态链接** : 每个栈帧都包含一个指向运行时常量池中该栈帧所属的方法的引用,持有这个引用是为了支持方法调用过程中的动态连接(`Dynamic Linking`)

**方法返回地址**: 当一个方法开始执行后,只有两种方式可以退出,一种是遇到方法返回的字节码指令,一种是遇见异常,并且这个异常没有在方法体中得到处理. 

![image-20200507150850005](http://files.luyanan.com//img/20200507150851.png)

准备一个`Person.java` 类

```java
public class Person {
    private String name = "Jack";
    private int age;
    private final double salary = 100;
    private static String address;
    private final static String hobby = "Programming";

    public void say() {
        System.out.println("person say...");
    }

    public static int calc(int op1, int op2) {
        op1 = 3;
        int result = op1 + op2;
        return result;
    }

    public static void order() {
    }

    public static void main(String[] args) {
        calc(1, 2);
        order();
    }

}

```

生成字节码指令, 可以使用`javap`

我们到jdk的bin目录下

```bash
.\javap.exe -c .\Person > D://Person.txt
```



```java
Compiled from "Person.java"
public class com.jvm.Person {
  public com.jvm.Person();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: aload_0
       5: ldc           #2                  // String Jack
       7: putfield      #3                  // Field name:Ljava/lang/String;
      10: aload_0
      11: ldc2_w        #4                  // double 100.0d
      14: putfield      #6                  // Field salary:D
      17: return

  public void say();
    Code:
       0: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #8                  // String person say...
       5: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return

  public static int calc(int, int);
    Code:
       0: iconst_3  // 将int类型常量3 压入[操作数栈]
       1: istore_0 // 将int 类型的值存入[局部变量0]
       2: iload_0  // 从[局部变量0]中装载int 类型值入栈 
       3: iload_1 // 从[局部变量1] 中装载int类型入栈
       4: iadd // 将栈顶元素弹出栈,执行int类型的加法,结果入栈
       5: istore_2  // 将栈顶int 类型值保存到[局部变量2]中
       6: iload_2 // 从[局部变量2] 中装载int类型值入栈
       7: ireturn  // 从方法中返回int类型的数值

  public static void order();
    Code:
       0: return

  public static void main(java.lang.String[]);
    Code:
       0: iconst_1
       1: iconst_2
       2: invokestatic  #10                 // Method calc:(II)I
       5: pop
       6: invokestatic  #11                 // Method order:()V
       9: return
}

```



此时, 你需要有一个能够看懂反编译指令的宝典, 比如官网的： https://docs.oracle.com/javase/specs/jvms/se8/html/index.html. 当然网上也有好多, 大家也可以自行查找. 

![image-20200507205230116](http://files.luyanan.com//img/20200507205317.png)



##  2. 折腾一下

### 2.1 栈指向堆

如果在帧栈中有一个变量, 类型为引用类型,比如`Object obj = new Object();` , 这个时候就是典型的栈中元素指向堆中的对象. 

![image-20200507205517653](http://files.luyanan.com//img/20200507205518.png)



### 2.2 方法区指向堆

方法区中会存放静态变量、常量等数据,如果是下面这种情况,就是典型的方法区中元素指向堆中的对象. 

```java
private static Object obj=new Object();
```

![image-20200507205629165](http://files.luyanan.com//img/20200507205630.png)



### 2.3 堆指向方法区

方法区中会保存类的信息,堆中会有对象,那怎么直到对象是哪个类创建的呢? 

![image-20200507205743181](http://files.luyanan.com//img/20200507205744.png)

思考: 一个对象怎么知道它是由哪个类创建出来的? 怎么记录? 这就需要了解一个java对象的具体信息了. 



###  2.4 Java对象内存布局

一个java对象在内存中包括3部分: 对象头、实例数据和对齐填充

![image-20200507205913356](http://files.luyanan.com//img/20200507205914.png)



## 3.  内存模型

### 3.1 图解

一块是非堆区, 一块是堆区,

堆区分为两大块, 一个是`old` 区, 一个是`Young` 区

`Young`区分为两大块,一个是`Survivor`区(`s0+s1`),一块是`Eden`区. `Eden:s0:s1=8:1:1`

`S0` 和`S1` 一样大, 也可以叫做`From` 和`To` 

![image-20200507210251741](http://files.luyanan.com//img/20200507210252.png) 

根据之前对`Heap` 的介绍可以知道,一般对象和数组的创建会在堆中分配内存空间,关键是堆中有那么多的区域, 那一个对象的创建到底在哪个区域呢? 



### 3.2 对象创建所在区域

一般情况下,新创建的对象都会被分配到`Eden`区, 一般特殊的大的对象会直接分配到`Old` 区

> 比如有对象A.B.C 等创建在`Eden`区,但是`Eden`区的内存空间肯定有限,比如有100M, 假如已经使用了100M 或者达到一个设定的临界值,这时候就需要对`Eden` 内存空间进行清理,即垃圾回收(`Garbage Collect`), 这样的`GC` 我们称之为`Minor GC`,`Minor GC` 指的是`Young`区的`GC`.

> 经过`GC` 之后, 有些对象就会被清理掉,有些对象可能还存活者,对于存活着的对象需要将其复制到`Survivor`区,然后再清理`Eden` 区中的对象. 

### 3.3 `Survivor` 区详解

由图解可以看出,`Survivor ` 可以分为两块, `S0` 和`s1`, 也可以叫做`From` 和`To`

在同一个时间点,`S0`和`s1` 只能有一个区有数据,另外一个是空的, 

> 接着上面的`gc` 来说,比如一开始只有`Eden`区和`From` 中有对象, `To` 中是空的, 
>
> 此时进行一次gc操作,`From` 区中的对象年龄就是`+1`,我们知道`Eden` 区中所有存活的对象会被复制到`To` 区,`From` 区中还能存活的对象会有两个去处. 
>
> 若对象年龄达到之前设置好的年龄阈值,此时对象会被移动到`Old`区, 此时`Eden` 区和`From` 区没有达到阈值的对象会被复制到`To`区,此时`Eden`区和`From` 区已经被清空(被`gc` 的对象肯定没了,没有被`gc`的对象都有了各自的去处). 
>
> 这时候`From` 和`To` 交换角色,之前的`From` 变成了`To`, 之前的`To` 变成了`From`. 
>
> 也就是说, 无论如何都要保证名为`To`的`Survivor` 区域是空的. 
>
> `Minor GC` 会一直重复这样的过程,直到`To`区被填满,然后会将对象复制到老年代. 

### 3.4 `Old` 区详解

从上面的分析可以看出, 一般`Old` 区都是年龄比较大的对象,或者相对超过了某个阈值的对象, 

在`Old` 区也会有`GC`的操作,`Old` 区的`gc` 我们称之为`Major GC`



###  3.5 对象的一辈子理解

我是一个普通的java对象,我出生在`Eden`区,在`Eden` 区我还看到和我长的很像的小兄弟,我们在`Eden` 区中玩了挺长的时间,有一天`Eden` 区中的人实在太多了,我就被迫去了`Survivor`区的`From` 区, 自从去了`Survivor` 区,我就开始漂了,有时候在`Survivor` 的`From`区， 有时候在`Survivor` 的`To`区,居无定所,直到我18岁的时候,爸爸说我成人了,该去社会上去闯闯了, 

于是我就去了老年代那里,老年代里, 人很多,并且年龄都很大,在这里我也认识了很多人,在老年代里， 我生活了20年(每次`GC` 加一岁), 然后被回收.

![image-20200507212700198](http://files.luyanan.com//img/20200507212701.png)



###  3.6 常见问题

- 如何理解`Minor`/`Major`/`Full GC`

  - `Minor GC`: 新生代
  - `Major GC`: 老年代
  - `Full GC`: 新生代和老年代

- 为什么需要`Survivor`区? 只有`Eden`区不行吗? 

   如果没有`Survivor` , `Eden` 区每进行一次`Minor GC`, 并且没有年龄限制的话,存活的对象就会被送到老年代,

  这样一来,老年代很快被填满,触发`Major GC`(因为`Major GC` 一般伴随着`Minor GC`, 也可以看作触发了`Full GC`)

  老年代的内存空间远大于新生代,进行一次`Full GC`  消耗的时间比`Minor GC`长的多. 

  执行时间长有多少坏处呢? 频发的`Full GC`  消耗的时间很长,会影响大型程序的执行和响应速度. 

  可能你会说,那就对老年代的空间进行增加或者较少呢?

  假如增加老年代空间,更多存活对象才能填满老年代，虽然降低`Full GC`频率,但是随着老年代的空间加大,一旦发生`Full GC`,执行所需要的时间更长. 

  假如减少老年代空间,虽然`Full GC`所需时间减少,但是老年代很快就被存活对象填满,`Full GC` 频率增加. 

  所以`Survivor` 的存在意义,就是减少被送到老年代的对象, 进而减少`Full GC` 的发生,`Survivor` 的预筛选保证, 只有经历16次`Minor GC`  还能在新生代中存活的对象,才会被送到老年代. 

- 为什么会需要两个`Survivor`区? 

  最大的好处就是解决了碎片化,也就是说为什么一个`Survivor`区不行? 第一部分中, 我们知道了必须设置`Survivor`区,假设现在只有一个`Survivor`区,我们来模拟一下流程: 

  刚刚创建的对象在`Eden` 中, 一旦`Eden` 满了,触发一次`Minor GC`,`Eden` 中存活的对象就会被移动到`Survivor` 区, 这样继续循环下去, 下一个`Eden` 满了的时候,问题来了,此时进行`Minor GC`,`Eden`和`Survivor` 各有一些存活的对象,如果此时把`Eden` 区的存活对象硬放到`Survivor`区, 很明显这两部分对象所占用的内存是不连续的,也就导致了内存碎片化. 

  永远有一个`Survivor space` 是空的, 另一个是非空的`Survivor space`  无碎片化

- 新生代中`Eden:s0:s1`为什么是`8:1:!`

  新生代中的可用内存: 复制算法用来担保的内存为9:1

  可用内存中`Eden:s1`区为`8:1`

  即新生代中`Eden:s1:s2`=`8:1:1`





## 4.  体验与验证

### 4.1 使用`jvisualvm`查看

`jvisualvm` 插件下载链接:

<https://visualvm.github.io/pluginscenters.html> --> 选择对应版本的链接-> `Tools` --> `Visual GC`

若上述链接找不到合适的,大家也可以自己在网上下载对应的版本。 

![image-20200508114612529](http://files.luyanan.com//img/20200508114621.png)

####  4.2 堆内存溢出

##### 4.2.1 代码示例

```java
@RestController
public class HeapController {

    List<Person> peoples = new ArrayList<>();

    @GetMapping("heap")
    public String heap() throws InterruptedException {

        while (true) {
            peoples.add(new Person());
            Thread.sleep(1);
        }
    }
}
```



#####  4.2.2  访问测试

为了设置参数比如: `-Xmx20M -Xms20M`

访问`heap` 接口发现

```java
Exception in thread "http-nio-8080-exec-2" java.lang.OutOfMemoryError: GC overhead limit
exceeded
```



###  4.3 方法区内存溢出

#####  4.3.1 需要依赖

```xml
<dependency>
            <groupId>asm</groupId>
            <artifactId>asm</artifactId>
            <version>3.3.1</version>
        </dependency>
```

#####  4.3.2 工具类,不断的生成类

```java
public class MetaspaceUtil extends ClassLoader {

    public static List<Class<?>> createClasses() {
        List<Class<?>> classes = new ArrayList<Class<?>>();
        for (int i = 0; i < 10000000; ++i) {
            ClassWriter cw = new ClassWriter(0);
            cw.visit(Opcodes.V1_1, Opcodes.ACC_PUBLIC, "Class" + i, null,
                    "java/lang/Object", null);
            MethodVisitor mw = cw.visitMethod(Opcodes.ACC_PUBLIC, "<init>",
                    "()V", null, null);
            mw.visitVarInsn(Opcodes.ALOAD, 0);
            mw.visitMethodInsn(Opcodes.INVOKESPECIAL, "java/lang/Object",
                    "<init>", "()V");
            mw.visitInsn(Opcodes.RETURN);
            mw.visitMaxs(1, 1);
            mw.visitEnd();
            MetaspaceUtil test = new MetaspaceUtil();
            byte[] code = cw.toByteArray();
            Class<?> exampleClass = test.defineClass("Class" + i, code, 0, code.length);
            classes.add(exampleClass);
        }
        return classes;
    }
}

```



##### 4.2.3  代码

```java
@RestController
public class NonHeapController {

    List<Class<?>> classes = new ArrayList<>();

    @GetMapping("nonheap")
    public String nonHeap() throws InterruptedException {
        while (true) {

            classes.addAll(MetaspaceUtil.createClasses());
            Thread.sleep(1);

        }
    }

}

```

设置`Metaspace`的大小，比如`-XX:MetaspaceSize=50M -XX:MaxMetaspaceSize=50M`

#####  4.2.4  验证

访问`nonheap` 接口发现

```java
java.lang.OutOfMemoryError: Metaspace
    at java.lang.ClassLoader.defineClass1(Native Method) ~[na:1.8.0_191]
    at java.lang.ClassLoader.defineClass(ClassLoader.java:763) ~[na:1.8.0_191]
```



### 4.3  虚拟机栈

#####  4.3.1 代码演示

```java
public class StackOverFlowDemo {

    public static long count = 0;

    public static void method(long count) {
        System.out.println(count++);
        method(count);
    }

    public static void main(String[] args) {
        method(1);
    }

}

```



#####  4.3.2  运行结果

![image-20200508115444751](http://files.luyanan.com//img/20200508115445.png)

####  4.3.3  理解和说明

`Stack Space` 用来做方法的递归调用时压入`Stack Frame(栈帧)`,所以当递归调用太深的时候, 就有可能耗尽`Stack Space`,报出 `StackOverflow`的错误. 

`-Xss128k`: 设置每个线程的堆栈大小,`JDK5` 之后每个线程堆栈大小为1M, 以前每个堆栈大小为256k, 根据应用的线程所需内存大小进行调整,在相同的物理内存下,减少这个值能够生成更多的线程. 但是操作系统对一个进程内的线程数还是有限制的,不能无限生成,经验值为`3000-5000` 左右. 

线程栈的大小是个双刃剑,如果设置过小, 可能会出现栈溢出,特别是该线程内有递归、大的循环时出现溢出的可能性更大, 如果该值设置过大,就有可能影响到创建栈的数量,如果是多线程的应用,就会出现内存溢出的错误. 









