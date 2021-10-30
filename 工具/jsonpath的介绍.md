# jsonpath的介绍:

JsonPath是一种简单的方法来提取给定JSON文档的部分内容。 JsonPath有许多编程语言，如Javascript，Python和PHP，Java。

JsonPath提供的json解析非常强大，它提供了类似正则表达式的语法，基本上可以满足所有你想要获得的json内容。

github上有它的应用:https://github.com/json-path/JsonPath 

JsonPath可在Central Maven存储库中找到。 Maven用户将其添加到您的POM:

```xml
<dependency>
    <groupId>com.jayway.jsonpath</groupId>
    <artifactId>json-path</artifactId>
    <version>2.2.0</version>
</dependency>
```

 

JsonPath表达式总是以与XPath表达式结合使用XML文档相同的方式引用JSON结构。

JsonPath中的“根成员对象”始终称为$，无论是对象还是数组。

JsonPath表达式可以使用点表示法

```
$.store.book [0].title
```

或括号表示法

```
$['store']['book'][0]['title']
```

# `jsonpath操作符：`

| 操作                    | 说明                                      |
| ----------------------- | ----------------------------------------- |
| $                       | 查询根元素。这将启动所有路径表达式。      |
| @                       | 当前节点由过滤谓词处理。                  |
| *                       | 通配符，必要时可用任何地方的名称或数字。  |
| ..                      | 深层扫描。 必要时在任何地方可以使用名称。 |
| .<name>                 | 点，表示子节点                            |
| ['<name>' (, '<name>')] | 括号表示子项                              |
| [<number> (, <number>)] | 数组索引或索引                            |
| [start:end]             | 数组切片操作                              |
| [?(<expression>)]       | 过滤表达式。 表达式必须求值为一个布尔值。 |

### 函数

函数可以在路径的尾部调用，函数的输出是路径表达式的输出，该函数的输出是由函数本身所决定的。

| 函数       | 描述                     | 输出     |
| :--------- | :----------------------- | :------- |
| `min()`    | 提供数字数组的最小值     | `Double` |
| `max()`    | 提供数字数组的最大值     | `Double` |
| `avg()`    | 提供数字数组的平均值     | `Double` |
| `stddev()` | 提供数字数组的标准偏差值 | `Double` |
| `length()` | 提供数组的长度           | Integer  |

### 过滤器运算符

过滤器是用于筛选数组的逻辑表达式。一个典型的过滤器将是[?(@.age > 18)]，其中@表示正在处理的当前项目。 可以使用逻辑运算符&&和||创建更复杂的过滤器。 字符串文字必须用单引号或双引号括起来([?(@.color == 'blue')] 或者 [?(@.color == "blue")]).

| 操作符  | 描述                                     |
| :------ | :--------------------------------------- |
| `==`    | left等于right（注意1不等于'1'）          |
| `!=`    | 不等于                                   |
| `<`     | 小于                                     |
| `<=`    | 小于等于                                 |
| `>`     | 大于                                     |
| `>=`    | 大于等于                                 |
| `=~`    | 匹配正则表达式[?(@.name =~ /foo.*?/i)]   |
| `in`    | 左边存在于右边 [?(@.size in ['S', 'M'])] |
| `nin`   | 左边不存在于右边                         |
| `size`  | （数组或字符串）长度                     |
| `empty` | （数组或字符串）为空                     |

接下来我们就用java代码来写一个案例:

### Java操作示例

JSON

```html
{
    "expensive": 10,
    "store": {
        "bicycle": {
            "color": "red",
            "price": 19.95
        },
        "book": [
            {
                "author": "Nigel Rees",
                "category": "reference",
                "price": 8.95,
                "title": "Sayings of the Century"
            },
            {
                "author": "Evelyn Waugh",
                "category": "fiction",
                "price": 12.99,
                "title": "Sword of Honour"
            },
            {
                "author": "Herman Melville",
                "category": "fiction",
                "isbn": "0-553-21311-3",
                "price": 8.99,
                "title": "Moby Dick"
            },
            {
                "author": "J. R. R. Tolkien",
                "category": "fiction",
                "isbn": "0-395-19395-8",
                "price": 22.99,
                "title": "The Lord of the Rings"
            }
        ]
    }
}
```

| JsonPath (点击链接测试)                                      | 结果                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`$.store.book[*\].author`](http://jsonpath.herokuapp.com/?path=$.store.book[*].author) | 获取json中store下book下的所有author值                        |
| [`$..author`](http://jsonpath.herokuapp.com/?path=$..author) | 获取所有json中所有author的值                                 |
| [`$.store.*`](http://jsonpath.herokuapp.com/?path=$.store.*) | 所有的东西，书籍和自行车                                     |
| [`$.store..price`](http://jsonpath.herokuapp.com/?path=$.store..price) | 获取json中store下所有price的值                               |
| [`$..book[2\]`](http://jsonpath.herokuapp.com/?path=$..book[2]) | 获取json中book数组的第3个值                                  |
| [`$..book[-2\]`](http://jsonpath.herokuapp.com/?path=$..book[2]) | 倒数的第二本书                                               |
| [`$..book[0,1\]`](http://jsonpath.herokuapp.com/?path=$..book[0,1]) | 前两本书                                                     |
| [`$..book[:2\]`](http://jsonpath.herokuapp.com/?path=$..book[:2]) | 从索引0（包括）到索引2（排除）的所有图书                     |
| [`$..book[1:2\]`](http://jsonpath.herokuapp.com/?path=$..book[1:2]) | 从索引1（包括）到索引2（排除）的所有图书                     |
| [`$..book[-2:\]`](http://jsonpath.herokuapp.com/?path=$..book[-2:]) | 获取json中book数组的最后两个值                               |
| [`$..book[2:\]`](http://jsonpath.herokuapp.com/?path=$..book[2:]) | 获取json中book数组的第3个到最后一个的区间值                  |
| [`$..book[?(@.isbn)\]`](http://jsonpath.herokuapp.com/?path=$..book[?(@.isbn)]) | 获取json中book数组中包含isbn的所有值                         |
| [`$.store.book[?(@.price < 10)\]`](http://jsonpath.herokuapp.com/?path=$.store.book[?(@.price < 10)]) | 获取json中book数组中price<10的所有值                         |
| [`$..book[?(@.price <= $['expensive'\])]`](http://jsonpath.herokuapp.com/?path=$..book[?(@.price <= $['expensive'])]) | 获取json中book数组中price<=expensive的所有值                 |
| [`$..book[?(@.author =~ /.*REES/i)\]`](http://jsonpath.herokuapp.com/?path=$..book[?(@.author =~ /.*REES/i)]) | 获取json中book数组中的作者以REES结尾的所有值（REES不区分大小写） |
| [`$..*`](http://jsonpath.herokuapp.com/?path=$..*)           | 逐层列出json中的所有值，层级由外到内                         |
| [`$..book.length()`](http://jsonpath.herokuapp.com/?path=$..book.length()) | 获取json中book数组的长度                                     |

 

上面的json字符串的读取案例:

(1)

```
String json = "...";

List<String> authors = JsonPath.read(json, "$.store.book[*].author");
```

(2)

如果你只想读取一次，那么上面的代码就可以了

如果你还想读取其他路径，现在上面不是很好的方法，因为他每次获取都需要再解析整个文档。所以，我们可以先解析整个文档，再选择调用路径。

```java
String json = "...";
Object document = Configuration.defaultConfiguration().jsonProvider().parse(json);

String author0 = JsonPath.read(document, "$.store.book[0].author");
String author1 = JsonPath.read(document, "$.store.book[1].author");
```



(3)

当在java中使用JsonPath时，重要的是要知道你在结果中期望什么类型。 JsonPath将自动尝试将结果转换为调用者预期的类型。

```java
// 抛出 java.lang.ClassCastException 异常
List<String> list = JsonPath.parse(json).read("$.store.book[0].author")

// 正常
String author = JsonPath.parse(json).read("$.store.book[0].author")
```



(4)

默认情况下，MappingProvider SPI提供了一个简单的对象映射器。 这允许您指定所需的返回类型，MappingProvider将尝试执行映射。 在下面的示例中，演示了Long和Date之间的映射。

```java
String json = "{\"date_as_long\" : 1411455611975}";
Date date = JsonPath.parse(json).read("$['date_as_long']", Date.class);
```



(5)

如果您将JsonPath配置为使用JacksonMappingProvider或GsonMappingProvider，您甚至可以将JsonPath输出直接映射到POJO中。

```java
Book book = JsonPath.parse(json).read("$.store.book[0]", Book.class); 
```

另外一个案例:

```
{ "store": {
    "book": [ 
      { "category": "reference",
        "author": "Nigel Rees",
        "title": "Sayings of the Century",
        "price": 8.95
      },
      { "category": "fiction",
        "author": "Evelyn Waugh",
        "title": "Sword of Honour",
        "price": 12.99,
        "isbn": "0-553-21311-3"
      }
    ],
    "bicycle": {
      "color": "red",
      "price": 19.95
    }
  }
}
private` `static` `void` `jsonPathTest() {
  ``JSONObject json = jsonTest();``//调用自定义的jsonTest()方法获得json对象，生成上面的json
  ``//输出book[0]的author值
  ``String author = JsonPath.read(json, ``"$.store.book[0].author"``);  
  ``//输出全部author的值，使用Iterator迭代
  ``List<String> authors = JsonPath.read(json, ``"$.store.book[*].author"``); 
  ``//输出book[*]中category == 'reference'的book
  ``List<Object> books = JsonPath.read(json, ``"$.store.book[?(@.category == 'reference')]"``);       
  ``//输出book[*]中price>10的book
  ``List<Object> books = JsonPath.read(json, ``"$.store.book[?(@.price>10)]"``);
  ``//输出book[*]中含有isbn元素的book
  ``List<Object> books = JsonPath.read(json, ``"$.store.book[?(@.isbn)]"``);
  ``//输出该json中所有price的值
  ``List<Double> prices = JsonPath.read(json, ``"$..price"``);
  ``//可以提前编辑一个路径，并多次使用它
  ``JsonPath path = JsonPath.compile(``"$.store.book[*]"``);
  ``List<Object> books = path.read(json);
}
```

 