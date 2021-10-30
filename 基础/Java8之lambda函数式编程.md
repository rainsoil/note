# Java8之lambda函数式编程

## 1. 引言 
java8的最大的特性就是引入Lambda 表达式,即函数式编程,可以将行为进行传递.**总结就是:使用不可变值与函数,函数对不可变值进行处理,映射成另一个值**
## 2. java 重要的函数式接口
### 1. 什么是函数式接口
 函数式接口是只有一个抽象方法的接口,用作Lambda 表达式的类型,使用@FunctionalInterface 注解修饰的类,编译器会检测该类是否只有一个抽象方法或者接口,否则,会报错,可以有多个默认的方法,静态方法.
#### 1.1 java8自带的常用函数式接口
| 函数接口       | 抽象方法       | 功能                   | 参数  |
| -------------- | -------------- | ---------------------- | ----- |
| Predicate      | test(T t)      | 判断真假               | T     |
| Consumer       | accept(T t)    | 消费消息               | T     |
| Function       | R apply(T t)   | 将T 映射为R (转换功能) | T     |
| Supplier       | T get()        | 产生消息               | None  |
| UnaryOperator  | T apply(T t)   | 一元操作               | T     |
| BinaryOperator | apply(T t,U u) | 二元操作               | (T,U) |

```java
package com.notes.utils.lambda;

import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.util.function.*;

/**
 * @author luyanan
 * @since 2019/7/17
 * <p>Lambda测试类</p>
 **/
public class LambdaTest {

    @Test
    public void test() {

        Predicate<Integer> predicate = x -> x > 12;

        // 判断真假
        User user = User.builder().age(15).name("张三").build();
        System.out.println("张三的年龄是否大于12:" + predicate.test(user.getAge()));

        // 消费消息
        Consumer<String> consumer = LambdaTest::say;
        consumer.accept("我相信我就是我");

        // 映射
        Function<User, String> function = User::getName;
        String name = function.apply(user);
        System.out.println(name);
        // 产生消息
        Supplier<Integer> supplier = () -> Integer.valueOf(BigDecimal.TEN.toString());
        System.out.println(supplier.get());


        UnaryOperator<Boolean> unaryOperator = uglily -> !uglily;
        Boolean apply = unaryOperator.apply(true);
        System.out.println(apply);


        BinaryOperator<Integer> binaryOperator = (x, y) -> x * y;
        Integer integer = binaryOperator.apply(2, 4);
        System.out.println(integer);

        test(()->"我是一个工作者");

    }


    //  演示自定义函数式接口编程
    public static void test(Worker worker) {
        String worker1 = worker.worker();
        System.out.println(worker1);
    }

    public interface Worker {
        String worker();
    }

    public static void say(String msg) {
        System.out.println(msg);
    }

}

```
结果
```java
张三的年龄是否大于12:true
我相信我就是我
张三
10
false
8
我是一个工作者
```
以上演示了Lambda接口的使用和自定义一个函数式接口并使用.下面,我们看看java8 将函数式接口封装到流中 如何高效的帮助我们处理集合.

#### 1.2 惰性求值与及早求值
**惰性求值:** 只描述Stream,操作的结果也是Stream,这样的操作成为惰性求值, 惰性求值可以想建造者模式一样链式调用,最后再使用及早求值得到最终结果.

**及早求值:得到的最终的结果而不是Stream,这样的操作称之为及早求值**

### 2 常用的流

#### 2.1 collect(Collectors.toList())
**将流转换为list,还有toSet().toMap()等,及早求值**
```java

    @Test
    public void collectTest() {

        List<User> userList = Stream
                .of(new User("张三", 13), new User("李四", 16), new User("王五", 18))
                .collect(Collectors.toList());
        System.out.println(userList);
    }
```
结果
```java
[User(name=张三, age=13), User(name=李四, age=16), User(name=王五, age=18)]

```

#### 2.2 filter
顾名思义,就是起过滤筛选的作用,内部就是Predicate接口,惰性求值.


- [x]   筛选出年龄大于16的用户

```java
 @Test
    public void filter() {
        List<User> userList = Stream
                .of(new User("张三", 13), new User("李四", 16), new User("王五", 18))
                .collect(Collectors.toList());

        // 筛选出年龄大于15的用户

        List<User> users = userList
                .stream()
                .filter(a -> a.getAge() > 15)
                .collect(Collectors.toList());
        System.out.println(users);
    }
    
```
结果
```java
[User(name=李四, age=16), User(name=王五, age=18)]

```
#### 2.3 map
转换功能,内部就是Function接口.惰性求值.
```java

    @Test
    public void map() {

        List<User> userList = Stream
                .of(new User("张三", 13), new User("李四", 16), new User("王五", 18))
                .collect(Collectors.toList());

        // 将所有的用户姓名 转换成一个集合

        List<String> names  = userList
                .stream()
                .map(User::getName)
                .collect(Collectors.toList());
        System.out.println(names);
    }

```

结果
> [张三, 李四, 王五]
#### 2.4 flatNap
将多个Stream 合并为一个Stream.惰性求值.

```java

    @Test
    public void flatMap() {

        // 合并Stream
        List<User> users1 = new ArrayList<>();
        users1.add(new User("张三", 11));
        users1.add(new User("李四", 12));
        List<User> users2 = new ArrayList<>();
        users2.add(new User("王五", 13));
        users2.add(new User("赵柳", 14));
        //  合并
        List<User> userList = users1
                .stream()
                .flatMap(user -> users2.stream())
                .collect(Collectors.toList());
        System.out.println(userList);
    }

```
结果
```java
[User(name=王五, age=13), User(name=赵柳, age=14), User(name=王五, age=13), User(name=赵柳, age=14)]

```

#### 2.5 max和min
集合中求最大值和最小值
```java

   @Test
    public void maxOrMin() {

        //  求最大值/最小值
        List<User> userList = Stream
                .of(new User("张三", 13), new User("李四", 16), new User("王五", 18))
                .collect(Collectors.toList());
        //  求年龄的最小的用户/年龄最大的用户
        User max = userList.stream().max(Comparator.comparing(User::getAge)).get();
        System.out.println("年龄最大的用户:"+max);
        User min = userList.stream().min(Comparator.comparing(User::getAge)).get();
        System.out.println("年龄最小的用户:"+min);
    }

```
结果
```java
年龄最大的用户:User(name=王五, age=18)
年龄最小的用户:User(name=张三, age=13)
```
#### 2.6 count
统计功能,一般都是结合filter 使用,因为先筛选出我们需要的再统计即可.及早求值.
```java
 @Test
    public void count() {

        List<User> userList = Stream
                .of(new User("张三", 13), new User("李四", 16), new User("王五", 18))
                .collect(Collectors.toList());
        // 统计
        long count = userList.stream().filter(a -> a.getAge() > 10).count();
        System.out.println(count);
    }
```
#### 2.7 reduce
reduce 操作可以实现从一组值中生成一个值,再上述例子中用到的count,min,max,因为常用而被纳入标准库.事实上,这些方法都是reduce操作.

```java
    @Test
    public void reduce() {
        Integer reduce = Stream
                .of(1, 2, 3, 4)
                .reduce(0, (acc, x) -> acc + x);
        System.out.println(reduce);

    }

```
结果
> 10
>
> ### 3  高级集合类及收集器
#### 3.1 数据分块
常用的流操作 是将其分解成两个集合,Collectors.partitioningBy 帮我们实现了,接受一个Predicate 函数式接口
```java
    @Test
    public void partitioningBy() {

        List<User> userList = Stream
                .of(new User("张三", 13), new User("李四", 16), new User("王五", 18))
                .collect(Collectors.toList());


        Map<Boolean, List<User>> listMap = userList.stream().collect(Collectors.partitioningBy(a -> a.getName().contains("张")));
        System.out.println(listMap);

    }

```
结果
> {false=[User(name=李四, age=16), User(name=王五, age=18)], true=[User(name=张三, age=13)]}
> ####  3.2 数据分组
> 数据分组 是一种更自然的分割数据操作，可以使
> 用任意值对数据分组。Collectors.groupingBy接收一个Function做转换。
```java

    @Test
    public void groupingBy() {
        List<User> userList = Stream
                .of(new User("张三", 13), new User("李四", 16), new User("王五", 18))
                .collect(Collectors.toList());
        Map<String, List<User>> listMap = userList
                .stream()
                .collect(Collectors.groupingBy(User::getName));

        System.out.println(listMap);
    }

```
结果
```
{李四=[User(name=李四, age=16)], 张三=[User(name=张三, age=13)], 王五=[User(name=王五, age=18)]}

```
####  3.3  字符串拼接
```java
    @Test
    public void join() {
        // 字符串拼接
        List<User> userList = Stream
                .of(new User("张三", 13), new User("李四", 16), new User("王五", 18))
                .collect(Collectors.toList());

        String collect = userList
                .stream()
                .map(User::getName)
                .collect(Collectors.joining(","));
        System.out.println(collect);
    }
```
结果
```
张三,李四,王五

```