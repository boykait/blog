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
&emsp;&emsp;_update()接收两个参数，通过_render()返回的一个VNode根节点，以及是否为服务器端渲染标识，在方法内部，