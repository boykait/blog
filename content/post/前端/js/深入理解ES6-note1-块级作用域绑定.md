---
title: 深入理解ES6-note1-块级作用域绑定
date: 2019-03-14
categories:
 - 前端
tags:
 - js
 - es6
---

### 块级作用域绑定
创建时间: 2019/03/14
> 目的主要是分清ES6同之前版本的js在定义变量时的一些不同，即let、const针对var在开发中的优势。

#### var定义变量

---
&emsp;&emsp;js中变量作用域可以分为全局作用域和函数作用域，而通过var方式进行什么的变量会存在变量提升的问题，换句话来说，就是所以定的变量会提升到当前作用域的最顶端，如：

```javascript
// 1.
a = 1;
console.log(a)
var a = 0;
// 2.
var a = 0;
a = 1;
console.log(a);
```
&emsp;&emsp;上面两种方式得到的结果是一样的均为1，变量的提升无疑会使得变量的作用域扩大，给初级的前端工程师带来一些疑惑，最经典的问题就是在循环体中创建函数：
```javascript
var funcs = [];
for(var i = 0; i < 10; i++) {
   var func = function() {
       console.log(i);
   }
   funcs.push(func);
}
funcs.forEach(f => f());
```
结果：虽然i定义为在for循环内的局部变量，但结果会循环输出10个10，这是应为变量i被提升后，循环创建的10个方法都保存着对相同变量i的引用，所以在最后打印时全部打印的最后一个引用对应的数字。
可以通过使用立即调用函数表达式IIFE解决这个问题，我的理解就是为变量i通过参数传递方式产生i的副本，而这个副本就是函数的局部变量，是属于当前函数作用域的：
```javascript
var funcs = [];
for(var i = 0; i < 10; i++) {
   funcs.push((function(value) {
      return function() {
           console.log(value);
      }
   })(i));
} 
funcs.forEach(f => f());
```
#### es6块级作用域
&emsp;&emsp;在es6中，有更为简便的方式实现上述功能，那就是使用let：
```javascript
var funcs = [];
for(let i = 0; i < 10; i++) {
   var func = function() {
       console.log(i);
   }
   funcs.push(func);
}
funcs.forEach(f => f());
```
&emsp;&emsp;在es6中，通过let或const申明将变量的作用域控制在最近的{}作用域内，而这两种方式定义的变量不会进行提升，所以如果在以let或const定义的变量之前调用该变量，则会直接报错（Uncaught ReferenceError: xx is not defined），这种就叫做临时死区（temporal dead zone）;由于这两种定义变量的方式能够使变量仅在相应的作用域内有效，所以如果在作用域结束之后再尝试使用该变量，同样会报Uncaught ReferenceError错误。    
注意: **let、const定义的变量在同一作用域内不可重复定义。**

#### 拓展-深入思考？

- [1. js为什么需要进行变量提升](http://boykait.github.io)  
- [2. 使用babel将es6转换成为es5会存在变量提升问题吗？](http://boykait.github.io)
- [3. js变量的初始化过程](http://boykait.github.io)
- [4. js内存模型](http://boykait.github.io)
