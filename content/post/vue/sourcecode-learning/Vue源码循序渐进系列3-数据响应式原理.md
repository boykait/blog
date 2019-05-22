---
title: Vue源码循序渐进系列3-数据响应式原理
date: 2019-05-20
categories:
  - Vue
  - js
tags:
  - Vue源码循序渐进
---

&emsp;&emsp;Vue作为一种MVVM框架，能够实现数据的双向绑定，让Vue技术栈的前端从业者摆脱了繁琐的DOM操作，这完全得益于Vue框架开发者对原生Object对象的深度应用。这里就主要分析实现Vue数据的响应式设置和监听的原理：数据劫持和订阅-发布。       

&emsp;&emsp;Vue实现数据响应系统的核心技术就是数据劫持和订阅-发布，基本思想就是通过对数据操作进行截获，在数据进行getter操作的时候更新依赖，即依赖收集过程(更新订阅对象集)；在数据进行setter时通知所有依赖的组件一并进行更新操作(通知订阅对象)。Vue数据响应系统原理示意图如下:

![Vue数据响应系统](/images/190522-vue_reactive_1.png)

### 数据劫持
&emsp;&emsp;Vue数据劫持主要利用Object.defineProperty来实现对对象属性的拦截操作，与其说拦截，不如说是为属性添加getter和setter方法，然后对getter和setter方法进行数据的监听和派发工作，Object.defineProperty示例:    
```javascript
var person = {
    name: '小明',
}
var name = person.name;
Object.defineProperty(person, 'name', {
    get: function() {
	   console.log('name getter方法被调用');
       return name; 
    },
    set: function(val) {
      console.log('name setter方法被调用');
      name = '李' + val;
    }
})

console.log(person.name);
person.name = '小亮';
console.log(person.name);

// 结果
name getter方法被调用    
小明    
name setter方法被调用    
name getter方法被调用    
```
&emsp;&emsp;可以看到，设置了属性的存取器后，在设置和获取person.name的时候，自定义的getter和setter将会被调用，这就给在这两个方法调用时进行其它内部的自定义操作提供了可能。Vue开发者正是利用Object.defineProperty的这个属性，实现了数据的响应式变化操作。Vue源码的数据劫持操作如下:
```javascript
src/core/observer/index.js
```

