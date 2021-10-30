# ES6

`ECMAScript6.0` (以下简称`ES6`,`ECMAScript` 是一种由`Ecma` 国际通过`ECMA-262`标准化的脚本),是`javaScript` 语言的下一代标准,2015年6月份正式发布,从`ES6`  开始的版本号采用年号,如`ES2015`,就是`ES6`. `ES2016` 就是`ES7`

`ECMAScript` 是规范,`JavaScript` 则是具体的实现. 

在`ES6` 中,加入了很多新的特性. 

## 1. 声明变量

在`es6` 中可以使用`let`来声明变量

1. `let` 具有严格的作用域

我们可以用`var` 声明一个变量,但是声明的变量往往会存在越域的问题, 但是用`let` 声明的变量则拥有严格的局部作用域

```javascript
	// var的申明往往会越域
		//let申明的变量具有严格的局部作用域
		{
			var a = 1;
			let b = 2;

		}
		console.log(a);
		console.log(b);

		/**
		 * 		输出结果:
		 *	 let.html:19 Uncaught ReferenceError: b is not defined
		 *	    at let.html:19
		 */
```



2. `var` 可以声明多次,但是`let` 只能声明一次

   ```javascript
   	// var 可以声明多次
   		// 但是`let` 只能声明一次
   		var m = 1;
   		var m = 2;
   		let n = 1;
   		let n = 2;
   		/**
   		 * 输出结果:
   		 * Uncaught SyntaxError: Identifier 'n' has already been declared
   		 */
   
   ```

3. `var`  会存在变量提升,但是`let` 不会存在

   ```javascript
   	// var 会变量提升,let不会存在变量提升
   		console.log(x);
   		var x = 10;
   
   		console.log(y);
   		let y = 10;
   		/**
   		 * 输出结果:
   		 * undefined
             let.html:43 Uncaught ReferenceError: Cannot access 'y' before initialization
              at let.html:43
   		 */
   		
   ```

4. `const` 也可以用来声明变量,区别在于`const` 声明之后不允许改变,而且一旦声明之后必须初始化,否则会报错. 

   ```javascript
   		// const 声明后不允许改变,一旦声明必须初始化,否则会报错
   		const a = 1;
   		a = 3; 
   		/**
   		 * 输出结果:
   		 *  let.html:55 Uncaught TypeError: Assignment to constant variable.
              at let.html:55
   		 */
   ```



全部

```html
<!DOCTYPE html>
<html>
	<head>
		<meta charset="utf-8">
		<title>let</title>
	</head>
	<body>
	</body>

	<script>
		// var的申明往往会越域
		//let申明的变量具有严格的局部作用域
		// {
		// 	var a = 1;
		// 	let b = 2;

		// }
		// console.log(a);
		// console.log(b);

		/**
		 * 		输出结果:
		 *	 let.html:19 Uncaught ReferenceError: b is not defined
		 *	    at let.html:19
		 */
		/**************************************/
		// var 可以声明多次
		// 但是`let` 只能声明一次
		// var m = 1;
		// var m = 2;
		// let n = 1;
		// let n = 2;
		/**
		 * 输出结果:
		 * Uncaught SyntaxError: Identifier 'n' has already been declared
		 */

		/************************************/
		// var 会变量提升,let不会存在变量提升
		console.log(x);
		var x = 10;

		console.log(y);
		let y = 10;
		/**
		 * 输出结果:
		 * undefined
          let.html:43 Uncaught ReferenceError: Cannot access 'y' before initialization
           at let.html:43
		 */
		
	    /***********************************/
		// const 声明后不允许改变,一旦声明必须初始化,否则会报错
		const a = 1;
		a = 3; 
		/**
		 * 输出结果:
		 *  let.html:55 Uncaught TypeError: Assignment to constant variable.
           at let.html:55
		 */
	</script>
</html>

```



##  2. 结构表达式

结构表达式使得我们可以通过特定的语法直接优雅的获取数组或者对象中的一个或者多个的值

- 数组

  ```javascript
  		let arr = [1, 2, 3];
  		let a = arr[0];
  		let b = arr[1];
  		let c = arr[2];
  		console.log(a, b, b);
  		let [d, e, f] = arr;
  		console.log(d, e, f);
  ```

- 对象

  ```java
  // 对象结构
  		const persion = {
  			name: "zhangsan",
  			age: 12,
  			language: ['java', 'js', 'python']
  		}
  
  		// 对象结构
  		const {
  			name: Name,
  			// 当结构的key:value的值不一样的时候,使用冒号区分,当一样的时候, 冒号可以忽略
  			age,
  			language
  		} = persion;
  		console.log(Name, age, language);
  		/**
  		 * 运行结果:
  		 * zhangsan 12 (3) ["java", "js", "python"]
  		 */
  ```

- 字符串扩展

  ```javasc
  	// 字符串扩展
  		let str = "hello world"
  		console.log(str.startsWith("hello"));
  		console.log(str.endsWith("world"));
  		console.log(str.includes("e"));
  		console.log(str.includes("hello"));
  
  		/**
  		 * 运行结果:
  		 * true
  		 * true
  		 * true
  		 * true
  		 */
  ```

- 字符串模板

  ```javascript
  //字符串模板
  		let template = ` <div >
  			<span > 我只是一个span < /span> 
  			/div>`
  
  		console.log(template)
  		/**
  		 * 运行结果:
  		 *  <div >
  			<span > 我只是一个span < /span> 
  			/div>
  		 */
  ```

- 字符串插入变量和表达式,变量名写在${}中, ${} 中可以放入`javaScript `表达式

  ```javascript
  //字符串插入变量和表达式,变量名写在${}中, ${} 中可以放入javaScript 表达式
  		function fun() {
  			return "这是一个函数";
  		}
  
  
  		let info = `我是${Name},今年${age}, 我想说:${fun()}`
  		console.log(info);
  		/**
  		 * 运行结果:
  		 * 我是zhangsan,今年12, 我想说:这是一个函数
  		 */
  ```



全部示例

```html
<!DOCTYPE html>
<html>
	<head>
		<meta charset="utf-8">
		<title>结构表达式</title>
	</head>
	<body>
	</body>

	<script>
		//数组结构
		let arr = [1, 2, 3];
		let a = arr[0];
		let b = arr[1];
		let c = arr[2];
		console.log(a, b, b);
		let [d, e, f] = arr;
		console.log(d, e, f);

		/**
		 * 运行结果:
		 * 1 2 2
		 * 1 2 3
		 */
		// 对象结构
		const persion = {
			name: "zhangsan",
			age: 12,
			language: ['java', 'js', 'python']
		}

		// 对象结构
		const {
			name: Name,
			// 当结构的key:value的值不一样的时候,使用冒号区分,当一样的时候, 冒号可以忽略
			age,
			language
		} = persion;
		console.log(Name, age, language);
		/**
		 * 运行结果:
		 * zhangsan 12 (3) ["java", "js", "python"]
		 */

		// 字符串扩展
		let str = "hello world"
		console.log(str.startsWith("hello"));
		console.log(str.endsWith("world"));
		console.log(str.includes("e"));
		console.log(str.includes("hello"));

		/**
		 * 运行结果:
		 * true
		 * true
		 * true
		 * true
		 */


		//字符串模板
		let template = ` <div >
			<span > 我只是一个span < /span> 
			/div>`

		console.log(template)
		/**
		 * 运行结果:
		 *  <div >
			<span > 我只是一个span < /span> 
			/div>
		 */


		//字符串插入变量和表达式,变量名写在${}中, ${} 中可以放入javaScript 表达式
		function fun() {
			return "这是一个函数";
		}


		let info = `我是${Name},今年${age}, 我想说:${fun()}`
		console.log(info);
		/**
		 * 运行结果:
		 * 我是zhangsan,今年12, 我想说:这是一个函数
		 */
	</script>
</html>

```









## 3. 函数优化

#####   参数默认值

```javascript
// 在es6之前, 我们无法给一个函数参数设置默认值,只能采用通用的写法
		function add(a, b) {

			// 判断B 是否为空,为空则给默认值


			b = b || 1;
			return a + b;
		}
		// 传一个参数
		console.log(add(1));

		// 现在可以这么写,直接给参数上加上默认值,没传的话就自动使用默认值

		function add2(a, b = 1) {

			return a + b;
		}
		console.log(add2(1));
```



#####  不定参数

```javascript
	//不定参
		function test3(...values) {

			console.log(values.length);
		}
		test3(1, 2);
		test3(2, 3, 4)
```



##### 箭头函数

```javascript
//箭头函数
		//以前声明一个方法
		var print = function(obj) {
			console.log(obj);;
		}
		//现在
		var print2 = obj => console.log(obj);

		print2("hello");


		var sum = function(a, b) {

			c = a + b;
			return c;
		}

		var sum2 = (a, b) => a + b;
		console.log(sum2(1, 2));

		var sum3 = (a, b) => {
			c = a + b;
			return c;
		}
		console.log(sum3(2, 3));



		const person = {
			name: "张三",
			age: 12,
			language: ['java', 'js', 'css']
		}

		function hello(person) {
			console.log('hello,' + person.name);
		}

		var hello2 = ({
			name
		}) => console.log("hello," + name);
		hello2(person);
```







全部页面

```html
<!DOCTYPE html>
<html>
	<head>
		<meta charset="utf-8">
		<title></title>
	</head>
	<body>
	</body>

	<script>
		// 在es6之前, 我们无法给一个函数参数设置默认值,只能采用通用的写法
		function add(a, b) {

			// 判断B 是否为空,为空则给默认值


			b = b || 1;
			return a + b;
		}
		// 传一个参数
		console.log(add(1));

		// 现在可以这么写,直接给参数上加上默认值,没传的话就自动使用默认值

		function add2(a, b = 1) {

			return a + b;
		}
		console.log(add2(1));



		//不定参
		function test3(...values) {

			console.log(values.length);
		}
		test3(1, 2);
		test3(2, 3, 4)

		//箭头函数
		//以前声明一个方法
		var print = function(obj) {
			console.log(obj);;
		}
		//现在
		var print2 = obj => console.log(obj);

		print2("hello");


		var sum = function(a, b) {

			c = a + b;
			return c;
		}

		var sum2 = (a, b) => a + b;
		console.log(sum2(1, 2));

		var sum3 = (a, b) => {
			c = a + b;
			return c;
		}
		console.log(sum3(2, 3));



		const person = {
			name: "张三",
			age: 12,
			language: ['java', 'js', 'css']
		}

		function hello(person) {
			console.log('hello,' + person.name);
		}

		var hello2 = ({
			name
		}) => console.log("hello," + name);
		hello2(person);
	</script>
</html>

```



## 4. 对象优化



##### 方法优化,可以获取map的键值对等`keys`,`values`,`entries`

```javascript
const person = {
			name: "张三",
			age: 15,
			language: ["java", "python", "css"]
		}


		console.log(Object.keys(person)); //["name", "age", "language"]
		console.log(Object.values(person)); //["jack", 21, Array(3)]
		console.log(Object.entries(person)); //[Array(2), Array(2), Array(2)]
```



##### 对象合并

```javascript
const target = {
			a: 1
		}
		const source1 = {
			b: 2
		}
		const source2 = {
			c: 3
		}
		Object.assign(target, source1, source2);
		console.log(target); //{a: 1, b: 2, c: 3}
```



#####  声明对象简写

```javasc
	// 2. 声明对象简写
		const age = 23;
		const name = "张三"
		const person1 = {
			age,
			name
		}; //声明对象简写
		console.log(person1);
```



##### 对象的属性简写

```javascript

		// 对象的函数属性简写
		let person3 = {
			name: "张三",
			age: 25,
			// 以前
			eat: function(food) {

				console.log(this.name + "再吃" + food);
			},

			//现在使用箭头函数, 箭头函数中 this不能使用, 对象.属性

			eat2: food => console.log(person3.name + "在吃" + food),
			eat3(food) {
				console.log(person3.name + "再吃" + food);
			}
		}
		person3.eat("苹果");
		person3.eat2("香蕉");
		person3.eat3("橘子");
```



##### 对象扩展运算符

- 对象拷贝(深拷贝)

```javascript
	//(1) 拷贝对象(深拷贝)
		let p1 = {
			name: "张三",
			age: 13
		}
		let someone = { ...p1
		};
		console.log(someone); // {name: "张三", age: 13}
```

- 合并对象

  ```javascript
  //(2)合并对象
  		let age1 = {
  			age: 16
  		};
  		let name1 = {
  			name: "zhangsan"
  		}
  		let p2 = {
  			name: "张三"
  		}
          // ...代表取出该对象的所有属性拷贝到当前对象中
  		p2 = { ...age1,
  			...name1
  		};
  		console.log(p2); // {age: 16, name: "zhangsan"}
  ```





## 5. `map`和`reduce`

##### `map` *map()：接收一个函数，将原数组中的所有元素用这个函数处理后放入新数组返回。*

```javascript
// 数组中新增了map和reduce方法
		// map(): 接受一个函数,将原数组中的所有元素用这个函数处理后放入新数组返回. 
		let arr = ["1", "2", "3"];
		let arr1 = arr.map((item) => {
			return item + 2;
		});
		let arr2 = arr.map(item => item + 2);
		console.log(arr1);
		console.log(arr2);
```



##### `reduce`

> 为数组中的每一个元素依次执行回调函数,不包含数组中被删除的元素或者从未被赋值的函数,  arr.reduce(callback,[initialValue]);

		 * 1. previousValue 上一次调用回调返回的值, 或者是提供的初始的值(initialValue)
		 * 2. currentValue  数组中当前被处理的元素
		 * 3. idnex 当前元素在数组中的索引
		 * 4. array 调用 reduce的数组
```javascript

		//reduce()  为数组中的每一个元素依次执行回调函数,不包含数组中被删除的元素或者从未被赋值的函数
		// [2,40,3,4]
		// arr.reduce(callback,[initialValue]);

		/**
		 * 1. previousValue 上一次调用回调返回的值, 或者是提供的初始的值(initialValue)
		 * 2. currentValue  数组中当前被处理的元素
		 * 3. idnex 当前元素在数组中的索引
		 * 4. array 调用 reduce的数组
		 */


		let result = arr.reduce((a,b) => {
			console.log("上一次处理后:" + a);
			console.log("当前正在处理:" + b);
			return a + b;
		}, 100);
		console.log(result);
```





## 6. `promise`

所谓`Primise` 就是一个对象,用来传递异步操作的消息,它代表了某个未来才会知道结果的事件(通常是一个异步操作), 并且这个事件提供了统一的`API`, 可供进一步处理. 

`Promise` 构造函数接受一个函数作为参数,该函数的两个参数分别为`resolve` 方法和`reject` 方法. 

如果异步操作成功, 则用`resolve` 方法将`Promise`对象的状态, 从`未完成` 变成`成功`, 即从`pending` 变为`resolved`

如果异步操作失败,则用`reject` 方法将`Promise` 对象的状态从`未完成` 变成`失败`,即从`pending` 变为`rejected`

这里演示一个使用`ajax` 的复杂查询

先准备数据

`user.json` 用户 

`user_corse_1.json` 课程

`corse_score_10.json`得分



以前的操作

```javascript
	// 1.查出当前的用户信息
		// 2. 根据当前用户的id查询它的课程
		// 3. 根据当前课程id查询分数

		$.ajax({
			url: "mock/user.json",
			success(data) {
				console.log("查询用户", data);


				$.ajax({
					url: `mock/user_corse_${data.id}.json`,
					success(data) {
						console.log("查询到课程", data);
						$.ajax({
							url: `mock/corse_score_${data.id}.json`,
							success(data) {
								console.log("查询到分数", data);
							},
							error(error) {

								console.log("查询分数出现异常", error);
							}
						});
					},
					error(error) {

						console.log("查询课程出现异常", error);
					}
				});



			},
			error(error) {
				console.log("出现异常了", error);
			}
		});

```

嵌套比较复杂

使用`Promise`

```javascript
//Promise可以封装异步操作
		let p = new Promise((resolve, reject) => { //传入成功解析,失败则拒绝

			// 1.异步操作

			$.ajax({

				url: "mock/user.json",
				success(data) {
					console.log("查询到用户:", data);
					resolve(data);
				},
				error(error) {
					reject(error);
				}
			});
		});

		//成功之后做什么
		p.then((obj) => {
			return new Promise((resolve, reject) => {

				$.ajax({

					url: `mock/user_corse_${obj.id}.json`,
					success(data) {
						console.log("查询到用户课程:", data);
						resolve(data);
					},
					error(error) {
						reject(error);
					}
				});
			});

		}).then((obj) => {
			return new Promise((resolve, reject) => {

				$.ajax({

					url: `mock/corse_score_${obj.id}.json`,
					success(data) {
						console.log("查询到用户分数:", data);
						resolve(data);
					},
					error(error) {
						reject(error);
					}
				});
			});
		})
```



还是有点复杂, 抽象一下方法

```javascript
// 自己定义一个方法整合一下
		function get(url, data) {
			return new Promise((resolve, reject) => {

				$.ajax({
					url: url,
					data: data,
					success(data) {

						resolve(data);
					},
					error(error) {
						reject(error);
					}
				});
			});
		}



		get("mock/user.json")
			.then((data) => {

				console.log("查询到用户", data);
				return get(`mock/user_corse_${data.id}.json`);
			}).then((data) => {
				console.log("查询到课程",
					data);
				return get(`mock/corse_score_${data.id}.json`);
			}).then((data) => {

				console.log("查询到分数", data);
			}).catch((error) => {

				console.log("出现异常", error);;
			});
```



## 7. 模块化

模块化就是把代码进行拆分,方便重复使用，类似于`java` 中的导包,而`js` 只是换了个概念,是导模块

模块功能主要有两个命令组成, `export` 和`import`

- `export`:用于规定模块的对外接口
- `import`:用于导入其他模块提供的功能

`user.js`

```javascript
var name = "张三"
var age = 12

function add(a, b) {

	return a + b;
}


export {
	name,
	age,
	add
}

```

`hello.js`

```javascript
// export const util = {
// 	sum(a, b) {
// 		return a + b;
// 	}   
// }

export default {
	sum(a, b) {
		return a + b;
	}
}

// export {util}
// export 不仅可以到处对象,一切js变量都可以导出, 比如基本类型变量, 函数、数组、对象

```

`main.js`

```javascript
import hello from "./hello.js"
import {
	name,
	add
} from "./user.js"

hello.sum(1, 2);
console.log(name);
add(2, 3)

```





本笔记所演示代码: https://github.com/lyn-workspace/webNotes.git

