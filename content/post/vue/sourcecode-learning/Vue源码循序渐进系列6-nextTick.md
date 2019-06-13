---
title: Vue源码循序渐进系列6-nextTick
date: 2019-06-04
categories:
  - Vue
  - js
tags:
  - Vue源码循序渐进
---
&emsp;&emsp;或许像下面的例子，我们得不到想要的结果:
```javascript
<html>
<head>
    <script src="./js/vue.js">
    </script>
</head>
<body>
<div id="app" >	
 <div id="msg">{{message}}</div>
 <a href="#" @click="click">点击</a>
</div>
<script>
    new Vue({
      el: '#app',
	  data(){
		return {
		  message: ''
		};
	  },
	  methods: {
		click() {
		  this.message = 'hello world';
		  // 我就是想在这儿操作DOM
		  console.log(document.getElementById('msg'));
		}
	  }
    });</script>
</html>


// 我们可能想要得到的结果：
<div id="msg">hello world</div>

// but，实际结果：
<div id="msg"></div>
```

&emsp;&emsp;设置了数据，但是在之后操作却得不到设置的结果，vue难道不支持设置完后对原生DOM进行操作，但是我们是有这种需求的哇，比如列表数据轮播，就需要知道所有数据所产生占据的网页高度然后进行轮播计算，如果不支持，那怎么破？显然是开玩笑的，Vue.nextTick/vue.$nextTick可以实现，即：
```javascript
this.$nextTick(() => {
	console.log(document.getElementById('msg'));
  });
```    

&emsp;&emsp;结果同预期相符，怎么放在Vue.nextTick/vue.$nextTick里面就能够保证对更新后的DOM进行操作呢？这也太优秀了。对，它就是能够保证，接下来一探究竟。

### Vue异步渲染
&emsp;&emsp;在Vue方法中可能对多个监听的数据进行操作进行复杂操作比如不停的计算，那么如果没更改一次这样的值就立马渲染相应的数据，那性能消耗无疑非常可观，所以Vue的更新操作是异步的，换句话讲，就是收集了一大波要更新的东西、然后一把梭，这有效的减少了回流重绘操作。
