---
layout:     post
title:      javascript快速入门
date:       2019-05-03
author:     haobo
header-img: img/home-bg.jpg
catalog: true
tags:
    - js
    - 前端
---

# 快速入门

### 基本语法
每个语句以;结束。语句块用{...}。

```js
    'Hello, world';
    var x = 1; 
    var y = 2;
```

### 数据类型

#### Number 
js不区分整数和浮点数,统一用Number表示

```js
123; // 整数123
0.456; // 浮点数0.456
1.2345e3; // 科学计数法表示1.2345x1000，等同于1234.5
-99; // 负数
NaN; // NaN表示Not a Number，当无法计算结果时用NaN表示
Infinity; // Infinity表示无限大，当数值超过了JavaScript的Number所能表示的最大值时，就表示为Infinity
```

Number 同样可以做四则运算

```js
1 + 2; // 3
(1 + 2) * 5 / 2; // 7.5
2 / 0; // Infinity
0 / 0; // NaN
10 % 3; // 1
10.5 % 3; // 1.5

```

#### 字符串
字符串用单引号或双引号括起来的任意文本

```js
'abc'
"abc"
```

#### 布尔值
只有true false 两种值

#### 比较运算符
第一种是==比较，它会自动转换数据类型再比较，很多时候，会得到非常诡异的结果；
第二种是===比较，它不会自动转换数据类型，如果数据类型不一致，返回false，如果一致，再比较。
由于JavaScript这个设计缺陷，不要使用==比较，始终坚持使用===比较。

NaN这个特殊的Number与所有其他值都不相等，包括它自己
```js

NaN === NaN // false

```

唯一能判断NaN的方法是通过isNaN()函数

```js

isNaN(NaN) // true

```

#### null和undefined
null表示一个“空”的值，它和0以及空字符串''不同，0是一个数值，''表示长度为0的字符串，而null表示“空”。
JavaScript的设计者希望用null表示一个空的值，而undefined表示值未定义。事实证明，这并没有什么卵用，区分两者的意义不大。大多数情况下，我们都应该用null。undefined仅仅在判断函数参数是否传递的情况下有用。

#### 数组
JavaScript的数组可以包括任意数据类型

```js
[1, 2, 3.14, 'Hello', null, true];

```

另一种创建数组的方法是通过Array()函数实现

```js

new Array(1, 2, 3); // 创建了数组[1, 2, 3]

```

#### 对象
JavaScript的对象是一组由键-值组成的无序集合

```js
var person = {
    name: 'Bob',
    age: 20,
    tags: ['js', 'web', 'mobile'],
    city: 'Beijing',
    hasCar: true,
    zipcode: null
};

```

获取一个对象的属性，我们用对象变量.属性名的方式

```js
person.name; // 'Bob'
person.zipcode; // null

```

#### 变量
变量在JavaScript中就是用一个变量名表示，变量名是大小写英文、数字、$和_的组合，且不能用数字开头。变量名也不能是JavaScript的关键字，如if、while等。申明一个变量用var语句

```js

var a; // 申明了变量a，此时a的值为undefined
var $b = 1; // 申明了变量$b，同时给$b赋值，此时$b的值为1
var s_007 = '007'; // s_007是一个字符串
var Answer = true; // Answer是一个布尔值true
var t = null; // t的值是null
```

在JavaScript中，使用等号=对变量进行赋值。可以把任意数据类型赋值给变量，同一个变量可以反复赋值，而且可以是不同类型的变量，但是要注意只能用var申明一次

```js
var a = 123; // a的值是整数123
a = 'ABC'; // a变为字符串

```

这种变量本身类型不固定的语言称之为动态语言，与之对应的是静态语言。静态语言在定义变量时必须指定变量类型，如果赋值的时候类型不匹配，就会报错。

#### strict 模式

JavaScript在设计之初，为了方便初学者学习，并不强制要求用var申明变量。这个设计错误带来了严重的后果：如果一个变量没有通过var申明就被使用，那么该变量就自动被申明为全局变量

```js

i = 10 // i现在是全局变量
```

在同一个页面的不同的JavaScript文件中，如果都不用var申明，恰好都使用了变量i，将造成变量i互相影响，产生难以调试的错误结果。
使用var申明的变量则不是全局变量，它的范围被限制在该变量被申明的函数体内。
为了修补JavaScript这一严重设计缺陷，ECMA在后续规范中推出了strict模式，在strict模式下运行的JavaScript代码，强制通过var申明变量，未使用var申明变量就使用的，将导致运行错误。
启用strict模式的方法是在JavaScript代码的第一行写上：

```js
'use strict';
```

### 字符串

转义字符\可以转义很多字符，比如\n表示换行，\t表示制表符，字符\本身也要转义，所以\\表示的字符就是\。

ASCII字符可以以\x##形式的十六进制表示

```js
'\x41'; // 完全等同于 'A'
```

还可以用\u####表示一个Unicode字符：

```js
'\u4e2d\u6587'; // 完全等同于 '中文'
```

#### 多行字符串
由于多行字符串用\n写起来比较费事，所以最新的ES6标准新增了一种多行字符串的表示方法，用反引号 * ... * 表示
```js

`这是一个
多行
字符串`;
```

#### 模板字符串

```js

var name = '小明';
var age = 20;
var message = `你好, ${name}, 你今年${age}岁了!`;
alert(message);

```

#### 操作字符串

```js
var s = 'Hello, world!';

s[0]; // 'H'
s[6]; // ' '
s[7]; // 'w'
s[12]; // '!'
s[13]; // undefined 超出范围的索引不会报错，但一律返回undefined

```

需要特别注意的是，字符串是不可变的，如果对字符串的某个索引赋值，不会有任何错误，但是，也没有任何效果：

```js

var s = 'Test';
s[0] = 'X';
alert(s); // s仍然为'Test'

```

JavaScript为字符串提供了一些常用方法，注意，调用这些方法本身不会改变原有字符串的内容，而是返回一个新字符串

##### toUpperCase
```js
var s = 'Hello';
s.toUpperCase(); // 返回'HELLO'
```

##### toLowerCase
```js
var s = 'Hello';
var lower = s.toLowerCase(); // 返回'hello'并赋值给变量lower
lower; // 'hello'
```

##### indexOf
```js

var s = 'hello, world';
s.indexOf('world'); // 返回7
s.indexOf('World'); // 没有找到指定的子串，返回-1
```

##### substring

```js

var s = 'hello, world'
s.substring(0, 5); // 从索引0开始到5（不包括5），返回'hello'
s.substring(7); // 从索引7开始到结束，返回'world'
```

### 数组

JavaScript的Array可以包含任意数据类型，并通过索引来访问每个元素。

```js

var arr = [1, 2, 3.14, 'Hello', null, true];
arr.length; // 6
```

直接给Array的length赋一个新的值会导致Array大小的变化

```js

var arr = [1, 2, 3];
arr.length; // 3
arr.length = 6;
arr; // arr变为[1, 2, 3, undefined, undefined, undefined]
arr.length = 2;
arr; // arr变为[1, 2]
```

Array可以通过索引把对应的元素修改为新的值，因此，对Array的索引进行赋值会直接修改这个Array

```js

var arr = ['A', 'B', 'C'];
arr[1] = 99;
arr; // arr现在变为['A', 99, 'C']
```

如果通过索引赋值时，索引超过了范围，同样会引起Array大小的变化

```js

var arr = [1, 2, 3];
arr[5] = 'x';
arr; // arr变为[1, 2, 3, undefined, undefined, 'x']
```

#### indexOf

与String类似，Array也可以通过indexOf()来搜索一个指定的元素的位置

```js

var arr = [10, 20, '30', 'xyz'];
arr.indexOf(10); // 元素10的索引为0
arr.indexOf(20); // 元素20的索引为1
arr.indexOf(30); // 元素30没有找到，返回-1
arr.indexOf('30'); // 元素'30'的索引为2
```

#### slice

slice()就是对应String的substring()版本，它截取Array的部分元素，然后返回一个新的Array：

```js
var arr = ['A', 'B', 'C', 'D', 'E', 'F', 'G'];
arr.slice(0, 3); // 从索引0开始，到索引3结束，但不包括索引3: ['A', 'B', 'C']
arr.slice(3); // 从索引3开始到结束: ['D', 'E', 'F', 'G']

```

如果不给slice()传递任何参数，它就会从头到尾截取所有元素。利用这一点，我们可以很容易地复制一个Array

#### push和pop

push()向Array的末尾添加若干元素，pop()则把Array的最后一个元素删除掉：

```js
var arr = [1, 2];
arr.push('A', 'B'); // 返回Array新的长度: 4
arr; // [1, 2, 'A', 'B']
arr.pop(); // pop()返回'B'
arr; // [1, 2, 'A']
arr.pop(); arr.pop(); arr.pop(); // 连续pop 3次
arr; // []
arr.pop(); // 空数组继续pop不会报错，而是返回undefined
arr; // []

```

#### unshift和shift

如果要往Array的头部添加若干元素，使用unshift()方法，shift()方法则把Array的第一个元素删掉

```js

var arr = [1, 2];
arr.unshift('A', 'B'); // 返回Array新的长度: 4
arr; // ['A', 'B', 1, 2]
arr.shift(); // 'A'
arr; // ['B', 1, 2]
arr.shift(); arr.shift(); arr.shift(); // 连续shift 3次
arr; // []
arr.shift(); // 空数组继续shift不会报错，而是返回undefined
arr; // []
```

#### sort

sort()可以对当前Array进行排序，它会直接修改当前Array的元素位置，直接调用时，按照默认顺序排序：

```js

var arr = ['B', 'C', 'A'];
arr.sort();
arr; // ['A', 'B', 'C']
```

#### reverse

reverse()把整个Array的元素给掉个个，也就是反转：

```js

var arr = ['one', 'two', 'three'];
arr.reverse(); 
arr; // ['three', 'two', 'one']
```

#### splice

splice()方法是修改Array的“万能方法”，它可以从指定的索引开始删除若干元素，然后再从该位置添加若干元素

```js

var arr = ['Microsoft', 'Apple', 'Yahoo', 'AOL', 'Excite', 'Oracle'];
// 从索引2开始删除3个元素,然后再添加两个元素:
arr.splice(2, 3, 'Google', 'Facebook'); // 返回删除的元素 ['Yahoo', 'AOL', 'Excite']
arr; // ['Microsoft', 'Apple', 'Google', 'Facebook', 'Oracle']
// 只删除,不添加:
arr.splice(2, 2); // ['Google', 'Facebook']
arr; // ['Microsoft', 'Apple', 'Oracle']
// 只添加,不删除:
arr.splice(2, 0, 'Google', 'Facebook'); // 返回[],因为没有删除任何元素
arr; // ['Microsoft', 'Apple', 'Google', 'Facebook', 'Oracle']
```

#### concat
concat()方法把当前的Array和另一个Array连接起来，并返回一个新的Array：

```js

var arr = ['A', 'B', 'C'];
var added = arr.concat([1, 2, 3]);
added; // ['A', 'B', 'C', 1, 2, 3]
arr; // ['A', 'B', 'C']
```

请注意，concat()方法并没有修改当前Array，而是返回了一个新的Array。

concat()方法可以接收任意个元素和Array，并且自动把Array拆开，然后全部添加到新的Array里

```js

var arr = ['A', 'B', 'C'];
arr.concat(1, 2, [3, 4]); // ['A', 'B', 'C', 1, 2, 3, 4]
```

#### join

join()方法是一个非常实用的方法，它把当前Array的每个元素都用指定的字符串连接起来，然后返回连接后的字符串：

```js
var arr = ['A', 'B', 'C', 1, 2, 3];
arr.join('-'); // 'A-B-C-1-2-3'
```

如果Array的元素不是字符串，将自动转换为字符串后再连接。

### 对象
JavaScript的对象是动态类型，你可以自由地给一个对象添加或删除属性

```js

var xiaoming = {
    name: '小明'
};
xiaoming.age; // undefined
xiaoming.age = 18; // 新增一个age属性
xiaoming.age; // 18
delete xiaoming.age; // 删除age属性
xiaoming.age; // undefined
delete xiaoming['name']; // 删除name属性
xiaoming.name; // undefined
delete xiaoming.school; // 删除一个不存在的school属性也不会报错
```
如果要检测xiaoming是否拥有某一属性，可以用in操作符:

```js

var xiaoming = {
    name: '小明',
    birth: 1990,
    school: 'No.1 Middle School',
    height: 1.70,
    weight: 65,
    score: null
};
'name' in xiaoming; // true
'grade' in xiaoming; // false
```

判断一个属性是否是xiaoming自身拥有的，而不是继承得到的，可以用hasOwnProperty()方法

```js

var xiaoming = {
    name: '小明'
};
xiaoming.hasOwnProperty('name'); // true
xiaoming.hasOwnProperty('toString'); // false
```

### 条件判断
JavaScript把null、undefined、0、NaN和空字符串''视为false，其他值一概视为true

### 循环

#### for ... in

for循环的一个变体是for ... in循环，它可以把一个对象的所有属性依次循环出来

```js
var o = {
    name: 'Jack',
    age: 20,
    city: 'Beijing'
};
for (var key in o) {
    console.log(key); // 'name', 'age', 'city'
}

```

过滤掉对象继承的属性，用hasOwnProperty()来实现

```js

var o = {
    name: 'Jack',
    age: 20,
    city: 'Beijing'
};
for (var key in o) {
    if (o.hasOwnProperty(key)) {
        console.log(key); // 'name', 'age', 'city'
    }
}
```

Array也是对象，而它的每个元素的索引被视为对象的属性，因此，for ... in循环可以直接循环出Array的索引

```js

var a = ['A', 'B', 'C'];
for (var i in a) {
    console.log(i); // '0', '1', '2'
    console.log(a[i]); // 'A', 'B', 'C'
}
```

### Map和Set

#### map 

```js

var m = new Map([['Michael', 95], ['Bob', 75], ['Tracy', 85]]);
m.get('Michael'); // 95
```

初始化Map需要一个二维数组，或者直接初始化一个空Map

```js

var m = new Map(); // 空Map
m.set('Adam', 67); // 添加新的key-value
m.set('Bob', 59);
m.has('Adam'); // 是否存在key 'Adam': true
m.get('Adam'); // 67
m.delete('Adam'); // 删除key 'Adam'
m.get('Adam'); // undefined
```

#### set 

Set和Map类似，也是一组key的集合，但不存储value。由于key不能重复，所以，在Set中，没有重复的key。

```js

var s = new Set([1, 2, 3, 3, '3']);
s; // Set {1, 2, 3, "3"}
```

add(key)方法可以添加元素到Set中，可以重复添加，但不会有效果

```js
s.add(4);
s; // Set {1, 2, 3, 4}
s.add(4);
s; // 仍然是 Set {1, 2, 3, 4}

```

delete(key)方法可以删除元素

```js
var s = new Set([1, 2, 3]);
s; // Set {1, 2, 3}
s.delete(3);
s; // Set {1, 2}
```

小结: Map和Set是ES6标准新增的数据类型，请根据浏览器的支持情况决定是否要使用。

### iterable

具有iterable类型的集合可以通过新的for ... of循环来遍历。

用for ... of循环遍历集合，用法如下：

```js

var a = ['A', 'B', 'C'];
var s = new Set(['A', 'B', 'C']);
var m = new Map([[1, 'x'], [2, 'y'], [3, 'z']]);
for (var x of a) { // 遍历Array
    console.log(x);
}
for (var x of s) { // 遍历Set
    console.log(x);
}
for (var x of m) { // 遍历Map
    console.log(x[0] + '=' + x[1]);
}
```

for ... in循环由于历史遗留问题，它遍历的实际上是对象的属性名称。一个Array数组实际上也是一个对象，它的每个元素的索引被视为一个属性。

当我们手动给Array对象添加了额外的属性后，for ... in循环将带来意想不到的意外效果

```js

var a = ['A', 'B', 'C'];
a.name = 'Hello';
for (var x in a) {
    console.log(x); // '0', '1', '2', 'name'
}
```

for ... in循环将把name包括在内，但Array的length属性却不包括在内。
for ... of循环则完全修复了这些问题，它只循环集合本身的元素：
```js

var a = ['A', 'B', 'C'];
a.name = 'Hello';
for (var x of a) {
    console.log(x); // 'A', 'B', 'C'
}
```

更好的方式是使用iterable内置的forEach方法，它接收一个函数，每次迭代就自动回调该函数。以Array为例

```js
'use strict';
var a = ['A', 'B', 'C'];
a.forEach(function (element, index, array) {
    // element: 指向当前元素的值
    // index: 指向当前索引
    // array: 指向Array对象本身
    console.log(element + ', index = ' + index);
});

```

set 和 array 类似。但Set没有索引，因此回调函数的前两个参数都是元素本身

```js

var s = new Set(['A', 'B', 'C']);
s.forEach(function (element, sameElement, set) {
    console.log(element);
});
```

map的回调函数参数为

```js

var m = new Map([[1, 'x'], [2, 'y'], [3, 'z']]);
m.forEach(function (value, key, map) {
    console.log(value);
});
```

如果对某些参数不感兴趣，由于JavaScript的函数调用不要求参数必须一致，因此可以忽略它们。

```js

var a = ['A', 'B', 'C'];
a.forEach(function (element) {
    console.log(element);
});
```