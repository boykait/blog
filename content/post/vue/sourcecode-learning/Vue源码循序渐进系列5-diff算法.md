---
title: Vue源码循序渐进系列5-diff算法
date: 2019-06-04
categories:
  - Vue
  - js
tags:
  - Vue源码循序渐进
---
&emsp;&emsp;知乎上有一篇文章挺有意思，叫做[网上都说操作真实 DOM 慢，但测试结果却比 React 更快，为什么？](https://www.zhihu.com/question/31809713)，旨在抨击虚拟DOM没有那么牛B，打开题主所附的链接进去操作了几把，刨去不同浏览器、硬件等差异条件，发现并没有那么糟糕，总的来说React除了第一次比原生的慢，后面的几乎在渲染效率上较原生Dom、Angular等操作更快，主要原因里面的朋友还是做了讨论：
```javascript
1. React首次需要创建Virtual DOM，这一过程比较耗时；
2. 后期再更新时，React利用diff算法进行差异性更新，而原生DOM操作方式需要先删除之前的DOM节点元素，然后再依次添加节点。
```
&emsp;&emsp;还是比较赞同尤大的说法，React采用Virtual DOM产生的主要是简化普通前端开发人员的工作量，数据驱动让我们能够更加专注于业务逻辑开发，在性能优化方面是相对具有普适性，比较不是所有前端都是能够有完美性能优化的能力，并且花大量时间去做优化工作可能并不会取到很好的正面效果（当然也不是说不去关注前端的性能优化问题）。好吧，言归正传，接下来，针对对Vue数据响应式更新的数据更新过程。
### _update过程
&emsp;&emsp;当Vue组件可观察的数据发生改变时，通过数据的setter中的通知到对应的观察Watcher发生动作，最终我们知道这些改变可能触发组件的Render Watcher进行页面的重新渲染，即执行了mountComponent中的updateComponent组件更新方法:
```javascript
-> core/instance/lifecycle.js

 // 执行更新组件
    updateComponent = () => {
      // vm._render() 由vm.$options.render()生成的vnode节点
      vm._update(vm._render(), hydrating)
    }
```
&emsp;&emsp;_update()接收两个参数，通过_render()返回的一个VNode根节点，以及是否为服务器端渲染标识，在方法内部，核心操作就是调用挂载在原型链上的__patch__方法进行打补丁式的更新渲染操作。
&emsp;&emsp;VNode全部属性约20来个，相比于原生DOM节点操作起来确实轻便不少，看一下关于Virtual DOM的虚拟节点VNode核心属性的相关定义:
```javascript
VNode {
  tag: string | void; // 当前节点的标签名   如： tag: "div"
  data: VNodeData | void; // 当前节点数据（VNodeData类型）如:attrs: {id: "app" staticClass: "aaa bbb ccc" staticStyle: {color: "red"}
  children: ?Array<VNode>; // 虚拟还在节点
  text: string | void; // 文本文字
  elm: Node | void; // 真实DOM节点
  ns: string | void; // 当前节点的名字空间
  context: Component | void; // rendered in this component's scope 编译作用域 Vue {_uid: 0, _isVue: true, $options: {…}, _renderProxy: Proxy, _self: Vue, …}
  key: string | number | void; // 
  parent: VNode | void; // component placeholder node
```

### diff算法
&emsp;&emsp;diff算法是Vue实现补丁式页面渲染的核心，相对于传统的DOM渲染，Vue假设Web UI跨级移动操作少可忽略不计，将diff对比过程定义为同级对比，时间复杂度由O(N^3)降为O(N)，借用一张经典的对比图：

![diff对比图](/images/190607-vue_diff_1.png)

#### diff示例
&emsp;&emsp;给一个diff对比栗子:
```javascript
原孩子VNode列表， oldCh： [a, b, c, d]    
操作后：    
新孩子VNode列表，newwCh: [a, d, e, a]
```
&emsp;&emsp;原oldCh对应的节点在真实DOM树种简单表示为：

![diff对比1](/images/190607-vue_diff_instance_1.png)


&emsp;&emsp;约定oldCh中的起止下标: oldS、oldE，newCh中的起止下标:newS、newE，依照diff原理推演在虚拟本层虚拟孩子节点列表发生变化的过程：
&emsp;&emsp;首先，第1次对比:
```javascript
oldCh： [a, b, c, d]    oldS: a    oldE: d
newCh:  [b, d, e, a]    newS: b    newE: a
```
&emsp;&emsp; oldS和newS对比(a-b)，未能匹配，然后oldS和newE对比(a-a)，对比对比成功，即a在真实DOM树种被移动到最后，触发页面进行渲染：

![diff对比2](/images/190607-vue_diff_instance_2.png)

&emsp;&emsp;第2次对比:
```javascript
oldCh： [b, c, d]    oldS: b    oldE: d
newCh:  [b, d, e]    newS: b    newE: e
```
&emsp;&emsp; oldS和newS对比(b-b)，比对成功，未发生改变，不发生渲染动作:

![diff对比3](/images/190607-vue_diff_instance_3.png)

&emsp;&emsp;第3次对比:
```javascript
oldCh： [c, d]    oldS: c    oldE: d
newCh:  [d, e]    newS: d    newE: e
```
&emsp;&emsp; oldS和newS对比(c-d)，未匹配；然后oldS和newE对比(c-e)，未匹配；再用oldE和newS(d-d)，匹配成功，触发页面进行渲染：

![diff对比4](/images/190607-vue_diff_instance_4.png)

&emsp;&emsp;第4次对比:
```javascript
oldCh： [c]    oldS: c    oldE: c
newCh:  [e]    newS: e    newE: e
```
&emsp;&emsp; oldS和newS对比(c-e)，未匹配；然后oldS和newE对比(c-e)，未匹配；再用oldE和newS(c-e)，匹配未成功；接着再用oldE和newE对比(c-e)，均为成功；最后在oldCh列表中挨着去查找(虽然只有一个节点)，发现确实没找到，那就新创建一个DOM节点，然后插到DOM树上对应位置：

![diff对比5](/images/190607-vue_diff_instance_5.png)

&emsp;&emsp;第5次对比:
```javascript
oldCh： [c]    oldS: c    oldE: c
newCh:  []    newS: null    newE: null
```
&emsp;&emsp; 发现newCh中基本遍历完了，old中还有一个虚拟节点c未处理，标明这个节点是期望应该被移除，那就在DOM树种去掉c对应的真实节点：

![diff对比6](/images/190607-vue_diff_instance_6.png)

### 总结
&emsp;emsp;Vue根据diff思想更新DOM树，以尽量达到最小化回流重绘，不过还是要知道，构造VirtualDOM不是免费的，需要付诸相应的构造时间和内存空间。