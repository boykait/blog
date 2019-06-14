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
          const text = document.getElementById('msg').innerHTML;
	      console.log(text);
		}
	  }
    });</script>
</html>


// 我们可能想要得到的结果：
hello world

// but，实际结果：
""空字符串
```

&emsp;&emsp;设置了数据，但是在之后操作却得不到设置的结果，vue难道不支持设置完后对原生DOM进行操作，但是我们是有这种需求的哇，比如列表数据轮播，就需要知道所有数据所产生占据的网页高度然后进行轮播计算，如果不支持，那怎么破？显然是开玩笑的，Vue.nextTick/vue.$nextTick可以实现，即：
```javascript
this.$nextTick(() => {
	const text = document.getElementById('msg').innerHTML;
	console.log(text);
  });
```    

&emsp;&emsp;结果同预期相符。怎么放在Vue.nextTick/vue.$nextTick里面就能够保证对更新后的DOM进行操作呢？这也太优秀了。对，它就是能够保证，接下来一探究竟。

### Vue异步渲染
&emsp;&emsp;在Vue方法中可能对多个监听的数据进行操作进行复杂操作比如不停的计算，那么如果没更改一次这样的值就立马渲染相应的数据，那性能消耗无疑非常可观，所以Vue的更新操作是异步的，换句话讲，就是收集了一大波要更新的东西、然后一把梭，这有效的减少了回流重绘操作。
&emsp;&emsp;前面关于对[数据响应式原理](./Vue源码循序渐进系列3-数据响应式原理.md)、[Watcher那些事儿](./Vue源码循序渐进系列4-Watcher那些事儿.md)讲了Vue的数据响应式的一些原理，我们知道，Watcher订阅者能够接收数据的更新通知进行update操作，在默认情况下，Watcher的更新并不会马上反应到界面上（这就是上面说的异步更新），而是将当前的Wathcer推入一个队列中-queueWatcher，继续追踪queueWatcher的实现，发现内部最根本的操作就是调用的nextTick去执行watcher对应的回调方法（get数据/渲染页面）：

```javascript
-> core/observer/watcher.js
update () {
    /* istanbul ignore else */
    if (this.lazy) { // 主要是计算属性使用
      this.dirty = true
    } else if (this.sync) { // 是否为同步方式更新
      this.run()
    } else { // 加入到订阅者更新队列(最终也要执行run方法)
      queueWatcher(this)
    }
  }

-> core/observer/scheduler.js
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    // 未执行队列的整体处理工作，直接将当前watcher推入queue
    if (!flushing) {
      queue.push(watcher)
    } else {
      // 放到队列中的对应位置，这个队列是按照watcher.id的下标进行排序的
      // 根据id做一个从小到大的排序，更小的id总是更先更新，因为从id的产生来说，父节点的id总是比子节点的id小
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true
      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      // 核心方法
      nextTick(flushSchedulerQueue)
    }
  }
}

-> core/observer/scheduler.js
function flushSchedulerQueue () {
  flushing = true
  let watcher, id
  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    ...
  }
  ...
}
```
&emsp;&emsp;其中nextTick(flushSchedulerQueue)，将flushSchedulerQueue推入回调数组中等待回调，Wathcer队列queue按watcher.id小而大排序，保证id号小的Wathcer先处理，所有的Watcher处理完毕，即Render Watcher也得以处理。

### nextTick
&emsp;&emsp;当将自定义DOM操作放在nextTick的回调方法中处理时，在nextTick内部会将该回调方法推入callbacks回调数组中（这个回调数组也是存放flushSchedulerQueue的数组），这两者的优先级一样高，就是看谁先被加入callbacks的时机，所以,如果click里面这样写，就不难知道：

```javascript

```javascript
click1() {
  this.message = 'hello world';
  this.$nextTick(() => {
    const text = document.getElementById('msg').innerHTML;
	console.log(text);
  });
  this.message = 'hello world 1';
}

click2() {
  this.$nextTick(() => {
    const text = document.getElementById('msg').innerHTML;
	console.log(text);
  });
  this.message = 'hello world 1';
}
```
```


Vue的下面给出nextTick的实现：

```javascript
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  // 放入回调队列中，等待执行
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {  // 回调数组是否处于等待处理状态
    pending = true
    timerFunc()
  }
}
```
&emsp;&emsp;如上面给出的第一个例子中，在click方法中将`this.message = 'hello world';`，这个操作会触发执行`nextTick(flushSchedulerQueue)`，这个时候，pending初始化时false，pending是标识回调数组是否处于等待处理状态，将pending置为等待处理中，并执行timerFunc方法，timerFunc我们目前可以理解是一个具有延时性的方法，内部有一个延时策略，所以在延时等待期间，可能会不断的执行Vue.nextTick/vue.$nextTick向callbacks继续push回调方法，在延时结束后，会执行我们的callbacks回调方法并清空callbacks数组，备以后使用。
#### 延时策略
&emsp;&emsp;说道延时策略，需要知道事件循环以及宏任务、微任务的概念([js事件循环机制](js事件循环机制.md)),大致是从宏任务队列中选取一个宏任务到主执行栈中执行后，会执行微任务队列中所有的任务，宏任务setTimeout及时设置时间为0最低也会延时4ms，所以微任务的执行可以说相对比较及时，这也是Vue的nextTick会根据浏览器环境优先选择属于微任务的延时方法(Promise
MutationObserver)，如果不支持则采用setImmediate方法，最后，最不济就只有选用setImmediate来执行callbacks回调方法的执行了：
```javascript
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // Use MutationObserver where native Promise is not available,
  // e.g. PhantomJS, iOS7, Android 4.4
  // (#6466 MutationObserver is unreliable in IE11)
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // Fallback to setImmediate.
  // Techinically it leverages the (macro) task queue,
  // but it is still a better choice than setTimeout.
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```

### 总结
通过本次对nextTick的探究，知道了其存在的意义，简结：
1. Vue响应式数据是异步更新的；
2. Vue响应式数据的设置必须在执行Vue.nextTick/vue.$nextTick之前，否则无效，因为nextTick内部的批量回调数组执行时有顺序的；
2. nextTick内部默认使用的延时方案的优先级是:`Promise>MutationObserver>setImmediate>setTimeout`。
