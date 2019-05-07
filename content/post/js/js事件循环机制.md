---
title: js事件处理机制之事件循环
date: 2019-04-20
categories:
  - js
tags:
  - js事件处理
---
> 查询数据记录、添加修改、下载文件、看视频、听音乐等都是通过js事件同服务器产生的交互行为，js做为一种单线程的脚本语言，如何做到多个请求异步执行、互不影响呢，js中主要通过事件循环模拟解决了这方面的问题，    
&emsp;&emsp;首先，js方法的执行是按照栈的方式执行的，试看下面的例子：

```javascript
function f1() {
   console.log('call f1'); 
}

function f2() {
    f1();
    console.log('call f2');
}
f2();

// 结果
call f1
call f2
```
&emsp;&emsp;js引擎在调用f2()方法时，先为f2方法构建一个执行环境-栈帧，然后将栈帧推入栈中，当发现f2中调用了f1方法时候，继续为f1生成栈帧结构，同理推入栈中，执行f1，打印"call f1"后，将f1从栈中弹出，再继续执行打印"call f2", f2执行完毕，将f2弹出。简单来说，上面这个示例是同步执行的，在一个方法中逐条代码执行，遇到函数调用，就需要将函数入栈，执行完毕再顺序出栈。        
&emsp;&emsp;为上面示例的f1方法调用添加一个延时：
```javascript
function f1() {
    console.log('call f1'); 
}

function f2() {
    setTimeout(() => {
      f1();
    }, 100)
	console.log('call f2');
}
f2();

// 结果
call f2
call f1
```
&emsp;&emsp;输出上面的结果，相比大家也并不觉得奇怪，因为setTimeout方法可能在实际项目开发过程中用得不少，使用setTimeout及时将延迟时间置为0s，上例输出的结果仍然不变，所以这种带异步延时的操作的实现机制就需要我们好好探究探究。

&emsp;&emsp;js利用时间循环处理异步问题，事件循环机制主要将非同步操作的操作注册到事件注册表中，待到合适的时机，再将该注册事件取出执行，注册事件一般不止一个，所以我们得考虑执行这类型事件的一个顺序机制，是的，就像我们银行排队办理业务类似，js引擎采用事件队列FIFO方式处理多个需要执行的回调事件，在此，针对不同的事件有不同的事件来源，这类事件源产生的事件任务分为
宏任务（macroTask）和微任务（microTask）,这两种不同的任务源会存入不同的任务队列，常用的macroTask类型有（需补充为什么要划分成这两种任务以及任务执行顺序）：
```javascript
setTimeout
setInterval
setImmediate
I/O
UI rendering
```
&emsp;&emsp;microTask类型有：
```javascript
process.nextTick
promises
Object.observe
MutationObserver
```
&emsp;&emsp;我们要知道的是，无论是宏任务还是微任务，在事件注册表中注册的事件是这两类任务对应事件源的回调方法，给出一个例子:
```javascript
console.log('begin');
setTimeout(() => {
  console.log('setTimeout');
}, 100);

new Promise((resolve, reject) => {
    console.log('promise');
    resolve();
}).then(() => {
    console.log('promise then');
});   
console.log('end');

// 结果
begin
promise
end
promise then
setTimeout
```
&emsp;&emsp;js事件执行有一定的顺序，先执行一次宏任务（第一次调用执行本段代码的操作就是执行了一次宏任务），再执行微任务，执行微任务时会将所有的微任务队列全都依次执行，然后再从宏队列中摘取任务至调用栈中继续执行。下面以顺序图的方式来梳理该过程的执行过程：
![cc](/images/1904026-js_event_loop_1.gif)