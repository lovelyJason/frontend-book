# ECMAScript新特性

## ECMAScript 6 新增语法

### let 和 const

- let

  - let 定义变量，**变量不可以重名**，必须先定义再使用

  - **具有块级作用域**

  - 没有变量提升

  - 代码演示

    ```js
    let age = 18;
    {
        // 外部无法score，因为有块级作用域
        let score = 100;
    }
    for (let i = 0; i < 10; i++) {
        // i 只能在此范围内使用，因为有块级作用域
    }
    ```

- const

  - **常量**一旦初始化，不可以重新赋值

  - const 定义常量，常量不可以重名，必须先定义再使用

  - 具有块级作用域

  - 没有变量提升

  - 代码演示

    ```js
    // 常量名一般大写
    const PI = 3.1415926;

    // 报错，常量定义必须进行初始化
    const MAX;
    ```

- 小结

  | 关键字   | 变量提升 | 块级作用域 | 其它           |
  | ----- | ---- | ----- | ------------ |
  | let   | ×    | √     | 先定义再使用，不可以重名 |
  | const | ×    | √     | 先定义再使用，不可以重名 |
  | var   | √    | ×     | 直接使用，可以重名    |

### 解构赋值

1. 数组的解构
2. 对象的解构

ES6 允许按照一定**模式**，从数组和对象中提取值，对变量进行赋值，这被称为解构（Destructuring）。

#### 数组的解构

方便获取数组中的某些项

```js
let arr = [5, 9, 10];
let [a, b, c] = arr;
console.log(a, b, c); // 输出 5 9 10
```

#### 对象的解构

- 方便解析对象中的某些属性的值

```js
// 默认情况获取到的变量需要和对象的属性同名
let obj = {foo: 'aaa', bar: 'bbb'};
let { foo, bar } = obj;

// 更改变量的名称
let obj = {foo: 'aaa', bar: 'bbb'};
let { foo: a, bar: b } = obj;

// 初始值
let obj = {name:'zs',age: 18}
let {name='',age=''} = obj
```

- 解构的过程可以进行模式匹配

```js
let obj = {
    name: 'zs',
    age: 18,
    dog: {
        name: 'BYD',
        age: 1
    }
}
// 通过模式匹配，获取 dog 的 name 和 age
// dog 是匹配源数据中的模式
let { dog: { name, age } } = obj;
```

- 对象解构在实际中的应用

```js
// 假设从服务器上获取的数据如下
let response = {
    data: ['a', 'b', 'c'],
    meta: {
        code: 200,
        msg: '获取数据成功'
    }
}
// 如何获取到 code 和 msg
let { meta: { code, msg } } = response;
```

### 函数

#### 箭头函数

ES6 中允许使用箭头定义函数 (=>  goes to)，目的是**简化函数的定义**并且里面的this也比较特殊。

- 箭头函数的定义

  ```js
  let fn = x => x * 2;
  // 等同于
  let fn = function (x) {
      return x;
  }
  ```

  如果箭头函数的只有一个参数可以省略小括号，否则参数的小括号不可以省略

  ```js
  let fn = (x, y) => x + y;
  // 等同于
  let fn = function (x, y) {
      return x + y;
  };
  ```

  如果箭头函数的代码块中有多余一条语句的话，不能省略大括号，如果需要返回值，必须添加 return

  即便是只有一条语句,但是加了大括号,若需要返回值也需要加return

  ```js
  let fn = (x, y) => {
      console.log(arguments);
      x = 2 * x;
      y = 2 * y;
      return x + y;
  }
  ```

- 箭头函数不可以作为构造函数使用

- 箭头函数内部没有 arguments

- **箭头函数内部的 `this`** 指向外部作用域中的 `this` 

  或者可以认为 箭头函数没有自己的 `this`

#### 参数的默认值

ES6 之前函数不能设置参数的默认值

```js
// ES5 中给参数设置默认值的变通做法
function fn(x, y) {
    y = y || 'world';
    console.log(x, y);
}
// ES6 中给函数设置默认值
function fn(x, y = 'world') {
}
```

#### rest 参数

rest 参数：剩余参数，以 … 修饰最后一个参数，把多余的参数都放到一个**数组**中。可以替代 arguments 的使用

```js
// 求一组数的最大值
function getMax(...values) {
    let max = values[0];
    for (let i = 0; i < values.length; i++) {
        if (max < values[i]) {
            max = values[i];
        }
    }
    return max;
}
// 调用
console.log(getMax(6, 1, 100, 9, 10));
```

> **注意：rest 参数只能是最后一个参数**

## 内置对象的扩展

1. Array 的扩展
2. String 的扩展
3. Number 的扩展
4. Set
5. Object的扩展

### Array 的扩展

- 扩展运算符

  扩展运算符，可以看成 rest 参数的逆运算，也是 **...** 可以把数组中的每一项展开

  ```js
  // 合并两个数组
  let arr1 = [1, 2];
  let arr2 = [3, 4];
  let arr3 = [...arr1, ...arr2];

  // 把数组展开作为参数，可以替代 apply
  // 求数组的最大值
  let arr = [6, 99, 10, 1];
  let max = Math.max(...arr);
  ```

- Array.from()

  把伪数组转换成数组

  ```js
  let fakeArr = {
    0: 1,
    1: 2,
    2: 3,
    length: 3
  };

  let arr = Array.from(fakeArr);
  console.log(arr);
  ```

- 数组实例的 find() 和 findIndex()

  **find** 找到数组中第一个满足条件的成员并**返回该成员**，如果找不到返回**undefined**。

  **findIndex** 找到数组中第一个满足条件的成员并**返回该成员的索引**，如果找不到返回 **-1**。

  ```js
  // 找到数组中的第一个小于 0 的数字
  let arr = [1, 3, -5, 6, -2];
  let result = arr.find((x) => x < 0);

  // 等同于
  let result = arr.find(function (x) {
      return x < 0;
  });

  // find 回调函数有 3 个参数
  arr.find(function (item, index, ar) {
      // item  当前的值
      // index 当前的值对应的索引
      // ar 原数组
  });
  ```

  > findIndex 的使用和 find 类似

- 数组实例的 includes()

  判断数组是否包含某个值，返回 true / false

### String的扩展

- 模板字符串

  模板字符串解决了字符串拼接不便的问题(内容太长需要换行，拼接多个变量)

  模板字符串使用反引号 **`** 括起来内容

  ```js
  let name = 'zs';
  let age = 18;
  // 拼接多个变量，在模板字符串中使用占位的方式，更易懂
  let str = `我是${name}，今年${age}`;

  // 内容过多可以直接换行
  let html = `
  	<div>
  		<ul>
  			<li>itheima</li>
  		</ul>
  	</div>
  `;
  ```

- includes(), startsWith(), endsWith()

  - `includes()`		返回布尔值，表示是否找到了参数字符串
  - `startsWidth()`         返回布尔值，表示参数字符串是否在原字符串的头部
  - `endsWith()`            返回布尔值，表示参数字符串是否在原字符串的尾部。

- repeat()

  `repeat`方法返回一个新字符串，表示将原字符串重复`n`次。

  ```js
  let html = '<li>itheima</li>';
  html = html.repeat(10);
  ```

### Number的扩展

ES6 将全局方法`parseInt()`和`parseFloat()`，移植到`Number`对象上面，行为完全保持不变。

- Number.parseInt()
- Number.parseFloat()

### Set

Set 是 ES6 中新增的内置对象，类似于数组，但是成员的值都是唯一的，没有重复的值(可以实现数组去重)。

```js
// Set 可以通过一个数组初始化
let set = new Set([1, 2, 1, 5, 1, 6]);
// 数组去重
let arr = [...set]; // 或Arra.from(set)
```

- Set 的成员
  - `size`：属性，获取 `set` 中成员的个数，相当于数组中的 `length`
  - `add(value)`：添加某个值，返回 Set 结构本身,可以链式调用,如果传入了相同的值会被忽略。
  - `delete(value)`：删除某个值，返回一个布尔值，表示删除是否成功。
  - `has(value)`：返回一个布尔值，表示该值是否为`Set`的成员。
  - `clear()`：清除所有成员，没有返回值。

遍历方法: 使用forEach,for of等

### Object

**对象字面量的增强,可接受一个变量作为属性名**

```javascript
var name = 'zs'
const obj = {
  [name]: 18
}
```

**Object.assign**

常用来拷贝对象,Object.assign(源对象, ...obj)

**Object.is**

同值比较,如

```javascript
+0 === -0 // true
NaN == NaN // false
Object.is(+0, -0) // false
Object.is(NaN, NaN) // true
```

### Map

键值对结构,传统对象会将对象的非字符串key转化为字符串

```javascript
const map = new Map()
map.set(key, value)
```

方法: set,get,has,delete,clear
遍历: forEach

### proxy

- Object.defineProperty只能监视属性的读写
- proxy更好的支持数组对象的监视
- proxy以非侵入的方式监视对象的读写

```javascript
const person = {
  name: 'zs',
  age: 18
}

const personProxy = new Proxy(person, {
  get (target, property) {
    // target为源对象,property为访问proxy对象的属性;返回值会作为访问proxy对象的属性值
  },
  set (target, property, value) {
    // value要写入的属性值
    target[property] = value
    // 可以return表示设置结果
  },
  deleteProperty (targrt, property) {
    delete target[property]
  }
})

// 监视数组
const list = []

const listProxy = new Proxy(list, {
  set (target, property, value) {
    target[property] = value
    return true
  }
})

// 内部会自动推算下标
list.push(10)
```

### Refelect

静态类,统一的对象操作API,统一操作对象的方法,共计13种

Refelct成员方法是Proxy处理对象的默认实现

```javascript
const obj = {}

const proxy = new Proxy(obj, {
  // 默认会定义以下方法
  get (targrt, property) {
    // 自定义逻辑
    return Refelect.get(target, property)
  }
})
```
Refelct静态方法

has: 检测对象是否有某个属性
deleteProperty: 删除对象属性
ownKeys: 获取所有属性名

## Promise 对象

泛指Promise构造函数,生成Promise实例

Promise实例生成以后,用then方法或者then方法第一个参数指定resolve状态下的回调函数,then方法第二个参数指定catch方法或catch方法指定rejected状态的回调函数,另外,catch也可捕获then方法中的异常

- then

then方法返回的是一个新的`Promise实例`,链式调用then后,前面的回调函数完成以后,会将返回结果(return 数据或者return Promise.resolve(),没有return,则为undefined)传入后面的回调函数,如果前一个回调函数返回的是新的`Promise对象`(即有异步操作),后一个回调函数会等待Promise对象的状态发生变化才会被调用

- catch

一般来说，不要在`then`方法里面定义 Reject 状态的回调函数（即`then`的第二个参数），总是使用`catch`方法。

理由是`.then()` `.catch()`的写法可以捕获前面`then`方法执行中的错误，也更接近同步的写法（`try/catch`）.不能直接在catch中return结果,要return Promise.reject()



跟传统的`try/catch`代码块不同的是，如果没有使用`catch`方法指定错误处理的回调函数，Promise 对象抛出的错误不会传递到外层代码，即不会有任何反应。



如果先调用catch方法再调用then方法,则then方法如果报错与前面的catch方法无关.先运行catch方法再运行then方法,如果没有报错,会跳过catch方法



Promise实例对象具有冒泡性质,catch方法中还能再抛出错误

## Symbol

原始数据类型,表示一个独一无二的值,es6以后可以使用symbol作为对象的key

```javascript
const s = Symbol('foo')   // 传入字符串表示描述文本
console.log(typeof s)   // Symbol
```

每次创建的Symbol都是不同的,即使传入相同的描述文本,如果要复用相同的Symbol值,可以

- 使用全局变量缓存下来
- 或者使用Symbol.for('描述文本'),维护了字符串和Symbol的关系

应用:
- 作为对象的key
- 实现对象的私有成员,因为外部无法创建到一个完全一样的Symbol

> tips: 截止到ES2019,一共定义了6种原始类型,加上object,一共7种数据类型,未来还会新增BigInt的数据类型

内置的Symbol常量,可以让自定义对象实现js内置接口

`Symbol.iterator`

```javascript
// 自定义toString标签
const obj = {
  [Symbol.toStringTag]: 'xObject'
}
```

**注意**

- 使用Symbol作为对象的属性名,无法使用for in循环遍历到,Object.keys,JSON.stringify都会使Symbol被忽略,也就说明了Symbol适合作为私有属性
  但是可以通过Object.getOwnPropertySymbols(obj)获取到Symbol类型属性名

## for...of循环

遍历所有数据结构的统一方式

特性

- 可以使用break关键词终止遍历
- 数组,伪数组(如arguments, node节点),map,set都可被遍历,其中遍历map,每次循环会得到键值组成的数组
- 不能遍历普通对象

```javascript
const m = new Map()
m.set('foo', 'bar')
for(const [key, value] of m) {

}
```

## 可迭代接口(接口: 统一的规格标准)

数组,set,map等的原型上都有Symbol.iterator方法,调用该方法返回数组的迭代器对象,迭代器对象上存在着next方法,不断调用next方法就可以实现对数据的遍历

实现原理

```javascript
const obj = {     // 该对象实现了可迭代接口(iteratble)
  [Symbol.iterator]: function () {
    return {      // 实现了迭代器接口(iteraotr)
      next: function () {
        return {      // 实现了迭代结果接口(iterationResult)
          value: '',
          done: true
        }
      }
    }
  }
}
```

### 迭代器模式

```javascript
const todos = {
  life: ['吃饭', '睡觉'],
  learn: ['语文', '数学'],
  // 外部不需要知道内部的细节,对外提供统一的遍历接口
  [Symbol.iterator]: function () {
    const all = [...life, ...learn]
    let index = 0
    return {
      next: function () {
        value: all[index],
        done: index++ >= all.length
      }
    }
  }
}

for (const item of todos) {
  
}
```

## 生成器函数

## ES2016

- 数组实例的includes(),可以查到NaN

- 指数运算法,如`Math.pow(2, 10)`等价于`2 ** 10`

## ES2017

- Object.values
- Object.entries
- Object.getOwnPropertyDescriptors
- String.prototype.padStart, String.prototype.padEnd
- 函数形参尾逗号
- async await

## ECMAScript 6 降级处理

### ES6 的兼容性问题

- ES6 虽好，但是有兼容性问题，IE7-IE11 基本不支持 ES6

  [ES6 兼容性列表](http://kangax.github.io/compat-table/es6/)

- 在最新的现代浏览器、移动端、Node.js 中都支持 ES6

- 后续我们会讲解如何处理 ES6 的兼容性

### ES6 降级处理

因为 ES6 有浏览器兼容性问题，可以使用一些工具进行降级处理，例如：babel

- 降级处理 babel 的使用步骤
  1. 安装Node.js
  2. 命令行中安装 babel
  3. 配置文件 `.babelrc`
  4. 运行


- 安装 Node.js

     [官网](https://nodejs.org/en/)

- 在命令行中，安装 babel

  ```bash
   npm install  @babel/core @babel/cli @babel/preset-env
  ```

  - 项目初始化(项目文件夹不能有中文)

    ```bash
    npm init -y
    ```

  - 配置文件 `.babelrc` (手工创建这个文件)

    babel 的降级处理配置

    ```json
    {
      "presets": ["@babel/preset-env"]
    }
    ```

  - 在命令行中，运行

    ```bash
    # 把转换的结果输出到指定的文件
    npx babel index.js -o test.js
    # 把转换的结果输出到指定的目录
    npx babel src -d lib
    ```

**参考：**[babel官网](https://www.babeljs.cn/)

## 扩展阅读

[ES6 扩展阅读](http://es6.ruanyifeng.com/)