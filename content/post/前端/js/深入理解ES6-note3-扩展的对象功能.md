---
title: 深入理解ES6-note3-扩展的对象功能
date: 2019-03-23
categories:
 - 前端
tags:
 - js
 - es6
---

> js对象是编程操作的中心，ES6中为对象简化了一些使用表达方式，也增加了很多的新的操作。

- 学习目标
```
1. 字面量

2. 新增方法

3. 原型使用升级
```

#### 字面量

&emsp;&emsp;ES6的字面量的初始化的简化操作确实是深受程序员的喜爱（至少我感觉还不错），它提供的主要功能是能够从作用域链中查找相同名称属性并进行赋值，减少了同样的属性需要写两次的麻烦还容易出错：
```javascript
// ES5
function createPerson(name, age) {
    return {
        name: name,
        age: age
    }
}
createPerson('小明', 23);

// ES6 
function createPerson(name, age) {
    return {
        name,
        age
    }
}
createPerson('小明', 23);
```
&emsp;&emsp;ES6中对对象方法也做了简化：
```javascript
var person = {
    name: '小明',
    age: 23,
    getName() {
        return this.name;
    }
    //  getName: function() {
    //    return this.name;
    //  }
}
```
&emsp;&emsp;除了写起来简单，关键是还能够在简化方法里使用super（非简化方法不能使用super）  
&emsp;&emsp;字面量其他特性还有比如去除了重复属性的校验，还有对象属性的枚举顺序等。


#### 新增方法
&emsp;&emsp; 为Object新增的两个方法比较常用，is()和assgin()；
is()方法主要是用来判断两个对象是否相等，当然是严格等"==="，不过也有些不同，它认为 -0不等于+0， NaN等于NaN：
```javascript
console.log(1 === 1);
console.log(Object.is(1, 1));

console.log(+0 === -0);
console.log(Object.is(+0, -0));

console.log(NaN === NaN);
console.log(Object.is(NaN, NaN));

// 结果：true true true false false true
```
&emsp;&emsp;不过具体用普通等、严格等或是 Object.is() 方法还是得看实际的应用场景。    
&emsp;&emsp;记得之前在一个React项目中开发使用了Object.assgin()方法，因为不能在原state状态上更改reduce结果，就用这个方法来从新拷贝数据并进行return了，当时不是很懂该方法的具体含义，只是跟着用着，现在晓得这种方式叫做混入（Mixin），主要用于组合对象，能够将多个供应者的数值或方法传给接收者对象，这个时候，多个供应者是有序的，且如果存在相同属性，后面的供应者能够覆盖前面的供应者数据：
```javascript
Object.assign({}, {name: '小明', age: 123}, {name: '小强', age: 22, sex: '男'})

// 混入结果： {name: "小强", age: 22, sex: "男"}
```
&emsp;&emsp;Object.assgin()方法对对象做的是浅拷贝，但也不完全是浅拷贝，第一层数据属性拷贝前后是各自独立的，第二层就只是引用拷贝了，测试如下：
```javascript
let a = {name: '小明', age: 123, scores: [{chinese: 89, math: 95, english: 88}]};

let b = Object.assign({}, a);

a.name = '小强';
a.scores[0].chinese = 90;
console.log(a);
console.log(b);

// 结果：
{name: "小强", age: 123, scores: Array(1)}
age: 123
name: "小强"
scores: Array(1)
0: {chinese: 90, math: 95, english: 88}

{name: "小明", age: 123, scores: Array(1)}
age: 123
name: "小明"
scores: Array(1)
0: {chinese: 90, math: 95, english: 88}
```
&emsp;&emsp;改变了a中的scores里面的数据，拷贝前后的对象的该数据项都发生了改变，这种就是对象属性的浅复制。

#### 原型使用升级
&emsp;&emsp;ES6中原型存在更大的使用空间，在早前ES5版本，对象原型一旦初始化后就保持不变，最多可以通过Object.getPrototypeOf()方法获取对象原型，ES6中可以通过Object.setPrototypeOf(targetObj, sourceObj)设置对象的原型，这个操作最主要的操作就是通过去set对象的内部属性[[Prototype]]上的值来完成的：
```javascript
let person = {
    sayHello() {
        console.log('hello, person');
    }
}
let dog = {
    sayHello() {
        console.log('hello, dog')
    }
}
let xiaoming = Object.create(person);
xiaoming.sayHello();
Object.setPrototypeOf(xiaoming, dog);
xiaoming.sayHello();

// 结果：
hello, person
hello, dog
```
&emsp;&emsp;ES6中还提供了super来获取继承父级对象中的方法:
```javascript
let person = {
    sayHello() {
        return 'hello';
    }
}

let man = {
    sayHello() {
        console.log(super.sayHello() + ', i am a man！')
    }
}
Object.setPrototypeOf(man, person);
man.sayHello();

// 结果：
hello, i am a man！
```
&emsp;&emsp;super用得更多的其实还是在class部分类的继承部分（后面再继续学习相关使用）
#### 拓展-深入思考？
##### js深拷贝
 &emsp;&emsp;Object.assgin()方法无法实现深拷贝，那么具体主要应用在哪些场景中，如何实现深拷贝呢？  
  &emsp;&emsp;Object.assgin()方法好处主要是简化了对象的混入操作。    
 &emsp;&emsp;场景一：在实际项目中，前后端主要通过JSON方式传输数据，如果数据是JSON格式，则可以通过JSON提供的两个方法stringify()和parse()，即先将数据转换为JSON字符串，然后再讲该字符串进行解析：
```javascript
let person = {name: '小明', age: 23, scores: {chinese: 89, math: 95, english: 88}};
let personStr = JSON.stringify(person);
let newPerson = JSON.parse(personStr);
//测试是否是深拷贝
newPerson.scores.chinese = 90;
console.log(person);
console.log(newPerson);

// 结果，newPerson的语文成绩由89变成90，而person对象中的不变。
 {name: "小明", age: 23, scores: {…}}age: 23name: "小明"scores: {chinese: 89, math: 95, english: 88}
{name: "小明", age: 23, scores: {…}}age: 23name: "小明"scores: {chinese: 90, math: 95, english: 88}
```
&emsp;&emsp;但如果对象中存在方法属性，那么JSON方式不能成功了，因为它不能拷贝方法，还有个就是不能拷贝正则表达式，其实最主要还是JSON.stringify()的时候忽略了对方法的拷贝，所以对有这些个方面的使用时就需用其他方式了，比如递归进行深拷贝操作：
```javascript
function deepClone(obj){
    let objClone = Array.isArray(obj)?[]:{};
    if(obj && typeof obj==="object"){
        for(key in obj){
            if(obj.hasOwnProperty(key)){
                //判断ojb子元素是否为对象，如果是，递归复制
                if(obj[key]&&typeof obj[key] ==="object"){
                    objClone[key] = deepClone(obj[key]);
                }else{
                    //如果不是，简单复制
                    objClone[key] = obj[key];
                }
            }
        }
    }
    return objClone;
}

let person = {name: '小明', age: 23, scores: {chinese: 89, math: 95, english: 88}, getName() {return this.name}};
let newPerson = deepClone(person);
console.log(person);
console.log(newPerson);

// 结果：
VM1085:21 
{name: "小明", age: 23, scores: {…}, getName: ƒ}
age: 23
getName: ƒ getName()
name: "小明"
scores: {chinese: 89, math: 95, english: 88}
__proto__: Object
VM1085:22 
{name: "小明", age: 23, scores: {…}, getName: ƒ}
age: 23
getName: ƒ getName()
name: "小明"
scores: {chinese: 89, math: 95, english: 88}
__proto__: Object
```
2. 修改原型的应用场景？

