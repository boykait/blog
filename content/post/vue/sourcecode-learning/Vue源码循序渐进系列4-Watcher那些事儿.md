---
title: Vue源码循序渐进系列4-Watcher那些事儿
date: 2019-05-25
categories:
  - Vue
  - js
tags:
  - Vue源码循序渐进
---

&emsp;&emsp;上一篇[数据响应式原理](./Vue源码循序渐进系列3-数据响应式原理.md)对Vue的实现MVVM的核心思想进行了学习，里面提到订阅-发布模式的订阅者主要用于响应数据发射变化的更新通知，当然，我们可以这么认为，Vue中的发布者其实也有可能是订阅者，可以订阅来自其其它组件的更新通知。本文主要对Vue中有哪些Watcher、在什么时候这些Wathcer会被触发，以及从源码角度尝试总结。

&emsp;&emsp;想一下，我们需要数据响应的场景? 比如一个购物车功能，看某宝的购物车界面(自动忽略购买内容 ^^)：

![购物车图片](/images/190525-vue_watcher_1.png)

&emsp;&emsp;在购物车方式下单前，我们需要考虑： 需要选择哪些来购买，选择的商品可能买多件，选择好要购买的商品的时候，我们要对要花费的RMB进行实时计算，也就是点击页面的复选框和修改数量的按钮都会影响核算的总消费，这个就可以利用到Vue里面的计算属性了：
```javascript
new Vue({
  name: 'cart',
  data () {
    return {
      selectedCarts: []
    }
  },
  watch: {
    /**
     * 监视selectedCarts变化
     * */
    selectedCarts: {
      handler: function (oldVal, newVal) {
        // do something
      },
      deep: true
    }
  },
  computed: {
    /**
     * 计算总价格
     * @returns {number}
     */
    totalPrice () {
      let totalPrice = 0.0
      this.selectedCarts.forEach((cart) => {
        totalPrice += cart.num * cart.price;
      })
      return totalPrice;
    }
  }
});
```

&emsp;&emsp;上面示例，就可以`computed`里的总价格`totalPrice`就可以根据选中的购物车条目`selectedCarts`计算得出，在计算出总价格后，会在页面呈现出计算的结果。此外，我们可以通过Vue的watch属性观察`selectedCarts`的变化，根据新旧值比较，可以下发更新购物车记录操作（数量）。我们来看一下这个例子中需要Vue数据做出响应的几个地方：
```javascript
1. 通过computed属性计算选中的购物车条目的总价格；
2. 通过监视选中的条目下发更新功能；
3. 总价格发生变动时，页面要及时呈现。
```
&emsp;&emsp;1、2、3点基本就蕴含Vue中的几种Watcher： 1.自定义Watcher； 2. Computed属性（实际上也是Watcher）； 3.渲染Watcher（Render Watcher），接下来对这几种Watcher细细评味。

### 1. 自定义Watcher
&emsp;&emsp;自定义Watcher可以监视的对象包括基本属性、对象、数组(后两种都需要指定deep深层次监听属性)，具体使用可以看Vue官网[watch](https://cn.vuejs.org/v2/api/#watch)，好了，知道自定义Wathcer怎么使用，接下来就看一看Vue内部是怎么使用的：
```javascript
-> vue/src/core/instance/state.js
function initWatch (vm: Component, watch: Object) {
  for (const key in watch) {
    const handler = watch[key]
    // 对数组中的每一个元素进行监视
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}

```




&emsp;&emsp;再了解一下Watcher主要有哪些类型：
- 组件实例化时会产生一个Watcher，在组件$mount的时候，在`mountComponent()`中会实例化一个Watcher，并挂载到vm的_watchers上，这个watcher最终会回调Vue的渲染函数。
```javascript
 new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
 updateComponent = () => {
      // vm._render() 由vm.$options.render()生成的vnode节点
      vm._update(vm._render(), hydrating)
    } 
```
- 用户定义的computed属性，Vue会在initState阶段为每一个computed属性生成一个Wathcer实例，但是我们还是需要知道，computed生成的Wathcer实例会存到_computedWatchers中，是基于缓存的；
```javascript
function initComputed (vm: Component, computed: Object) {
  // $flow-disable-line
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  for (const key in computed) {
    const userDef = computed[key]
    ...
    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }
    ...
    }
  }
}
```
