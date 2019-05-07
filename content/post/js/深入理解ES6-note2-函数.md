---
title: 深入理解ES6-note2-函数
date: 2019-03-18
categories:
 - 前端
tags:
 - js
 - es6
---

> 掌握ES6中函数的使用，且与ES5使用的不同

#### 知识点
```
1. 函数参数传递

2. 函数双重用途

3. 箭头函数

4. 尾调优化
```

#### 1. 函数参数传递
&emsp;&emsp;在ES5中，我们可能需要对一些参数数值进行校验，以保证方法的健壮性，虽然可以通过try-catch的方式来进行异常捕获，但我们还是希望如果在某些数值异常的情况下，程序仍能够正常运行。所以：
```javascript
function sum(a, b) {
    a = (a !== undefined ) ? a : 1;
    b = (b !== undefined ) ? b : 1;
    console.log(parseInt(a + b));
}
sum();

// 输出结果：2
```
&emsp;&emsp;显然，这种方式看起来似乎不太美观，判断a、b是否为undefined可以理解是相同的判断模式，既然如此，能否将这个不太美观的写法交由js解释器处理呢，在ES6中，答案是可以的，下面的写法，执行后仍然可以得到相同的结果：
```javascript
function sum(a = 1, b = 1) {
    console.log(parseInt(a + b));
}
sum();

// 输出结果：2
```
&emsp;&emsp;在调用时候，a、b不传值的一个会被附上默认值，相较第一种而言，确实显得精简不少，这就是函数传参设置默认参数的方法，当然，我们可以通过如下方式为参数b赋值：

```javascript
function sum(a = 1, b = a) {
    console.log(parseInt(a + b));
}
sum(2);

// 输出结果：4
```
&emsp;&emsp;上面这个示例也是极好理解的，相当于：
```javascript
var a = 2;
if (a === undefined) {
    a = 1;
}
var b = a;
console.log(parseInt(a + b));
```
&emsp;&emsp; **注**：原书中相似例子“参数默认的临时性死区”中，将这种转述为 "let first = 1......"，个人认为是不正确的，应为在方法形参那我们虽然定义了变量a，但方法体中，我们仍然可以定义“var a = 2;”类似的写法，这和var类型的定义的变量的使用规则是相符合的，此外，还需要我们注意的是，函数的参数传递过程存在临时死区TDZ，假如：
```javascript
function sum(a = b, b = 1) {
    console.log(parseInt(a + b));
}
sum(undefined, 2);

// 输出结果：Uncaught ReferenceError: b is not defined
```
### 2. 函数双重用途
&emsp;&emsp;在ES5中，如果想要自定义一个类，并且要为该类设计一些行为，我们可以首先定义一个函数作为类，再为该函数的原型prototype属性添加行为方法，如下面操作：
```javascript
function Person(name, age) {
    this.name = name;
    this.age = age;
}
Person.prototype.printInfo = function() {
    console.log('name: ' + this.name + '; age: ' + this.age);
}

function test() {
    var p = new Person('小明', 20);
    p.printInfo();
}
test();
// 结果：name: 小明; age: 20
```
&emsp;&emsp;ok，很好，上面的例子能够正确执行，因为我们一切操作都符合函数操作的规范，但是，如果我们将test方法中定义变量p时的new关键字去掉呢，结果会是什么呢？显然是不正确的，会报出：“Uncaught TypeError: Cannot read property 'printInfo' of undefined”错误，所以函数的两种用途：
```
1. 当做真正执行操作的方法去执行；

2. 当做抽象出来的类并能够进行实例化操作。
```
&emsp;&emsp;函数的双重作用的最关键点就是使用了“new”关键字，当使用“new”时，Person函数内部的construct方法会被执行，此时，这样的函数叫做构造器，关键点来了，如何判断Person是作为那种进行调用的并进行正确的操作呢，假如我们想Person能够作为构造器使用，在ES5中，可以进行如下判断：
```javascript
function Person(name, age) {
    if (this instanceof Person) {
        this.name = name;
        this.age = age;
    } else {
        throw new Error('error');
    }
}
```
&emsp;&emsp;但是这里面的缺点就是使用调用call方法可破:
```javascript
var p = Person('小明', 20);
var p1 = Person.call(p, '小明1', 21);
```
&emsp;&emsp;ES6中通过new.target元属性解决了这个问题，当construct方法被调用时，new.target会被填入new运算符的作用目标（一般指构造器），否则该该元属性为undefined。

#### 箭头函数
&emsp;&emsp;箭头函数无疑是ES6函数一个比较重大的改变，引入箭头函数最主要解决的问题就是解决this的问题，在上面的例子中，我们将printInfo方法稍微进行改造，延迟200ms打印
```javascript
Person.prototype.printInfo = function() {
   setTimeout(function() {
        console.log('name: ' + this.name + '; age: ' + this.age);
   }, 200);
}
// 结果 name: ; age: undefined
```
&emsp;&emsp;这和我们想象中的结果是不一样的，因为在setTimeout内部的this所指向的调用对象是window非当前的p对象，window无name和age两个属性，故打印出期望之外的结果，先看一下this的一些定义：
```
1. this总是代表它的直接调用者

2.在默认情况(非严格模式下,未使用 'use strict'),没找到直接调用者,则this指的是 window

3.在严格模式下,没有直接调用者的函数中的this是 undefined

4.使用call,apply,bind(ES5新增)绑定的,this指的是 绑定的对象
```
&emsp;&emsp;或许可以这么做，将当前的this进行缓存，用另外一个变量_self进行指向，在setTimeout中使用_self：
```javascript
Person.prototype.printInfo = function() {
   let _self = this;
   setTimeout(function() {
        console.log('name: ' + _self.name + '; age: ' + _self.age);
   }, 200);
}
// 结果 name: 小明; age: 20
```
&emsp;&emsp;ES6中定义的箭头函数就是主要解决5中this的问题，上面的示例又可以进行如下表示，代码显得更加简洁：
```javascript
Person.prototype.printInfo = function() {
   setTimeout(() => console.log('name: ' + this.name + '; age: ' + this.age), 200);
}
// 结果 name: 小明; age: 20
```
&emsp;&emsp;箭头函数的特点：

```
1. 无：this、super、arguments、new.target；
2. 不能调用new；
3. 没有prototype属性
```
&emsp;&emsp;当然，箭头函数还有很多的一些用处，主要就是因为写着让代码看着更简洁，更爽嘛，比如sort、map、reduce等等方法中。

#### 尾调用优化
&emsp;&emsp;对内存模型有一些了解的朋友可能知道(js/java有些类似)，函数的调用是存放在线程所在的栈帧(stack frame)中，这也即是说，函数存在最大的递归调用次数（具体多少次，看硬件设施以及设置的相关参数等，不过最大的递归调用次数是存在的），超过这个阈值，就会报栈溢出问题，尾调用优化就主要尝试来解决这个问题，给出一个经典的斐波那契数列求和：1、1、2、3、5、8、13、21、34、……
```javascript
// 普通递归操作

function func(n) {
    if (n === 0 || n === 1) {
        return 1;
    } else {
        return func(n - 1) + func(n - 2);
    }  
}
var res = func(20);
console.log(res);
```
