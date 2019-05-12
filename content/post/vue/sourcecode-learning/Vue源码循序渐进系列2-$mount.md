---
title: Vue源码循序渐进2：$mount 
date: 2019-05-12
categories:
  - Vue
tags:
  - Vue源码循序渐进系列

&emsp;&emsp;看一下Vue源码中_init()方法最后一步:
```javascript

```

在Vue官网中有说明如果 Vue 实例在实例化时没有收到 el 选项，则它处于“未挂载”状态，没有关联的 DOM 元素。可以使用 vm.$mount() 手动地挂载一个未挂载的实例。如果没有提供 elementOrSelector 参数，模板将被渲染为文档之外的的元素，并且你必须使用原生 DOM API 把它插入文档中。这个方法返回实例自身，因而可以链式调用其它实例方法。