---
title: Vue-Router原理分析-路由模式和install
date: 2019-06-23
categories:
 - Vue-Router
 - Vue
tags:
 - Vue-Router
---

&emsp;&emsp;单页应用(SPA, Single Page Application)的整个Web系统由一个html文件，通过Ajax和后端进行数据交互，通过一些特殊手段去加载渲染页面的不同部分，使得无需刷新整体页面，，就像使用app一样，极大的提升了用户使用体验，在Vue生态中，就是利用Vue的核心插件-Vue-Router来实现Web界面的路由跳转，so，本文就是通过学习Vue-Router，了解Vue-Router完成SPA开发的实现具体原理。 

### Web路由简介   
&emsp;&emsp;先简单说一下路由，早期Web系统路由主要是服务器后端实现，熟悉前后端开发的朋友就知道，在服务器端，通过路由表映射到相应的方法，并执行相应的动作，比如我们想要请求API数据、新页面、文件等：

```javascript

"/" -> goHtml()

"/api/users" -> queryUsers()

```    

&emsp;&emsp;客户端实现路由主要就有两种方式:

```javascript
1. 基于Web Url的hash
2. 基于Hishtory API
```

#### 基于Web Url的hash
&emsp;&emsp; Url中的hash值是“#”号后面的参数值，比如`http://www.example.com/index.html#location1`里面的值localtion1, 主要有如下一些特性(之前看到公司项目的Url中带有“#”，虽然有点莫名，但都没去想想咋来的，汗)：

```javascrpit
1. Url中#后的内容不会发送到服务器；
2. hash值的改变会保存到浏览器的历史浏览记录中，可通过浏览器的回退按钮查看；
3. hash值是可读写的，可通过window.location.hash获取到hash值，利用onhashchange监听hash值的变化(Vue-Router里的hash模式就是在onhashchange中的回调事件中完成路由到界面的更新渲染)。
```
#### 基于Hishtory API
&emsp;&emsp;基于hash模式虽然方便，但带有#号，有同学就会觉这样的Url有点丑啦，为了和同向服务器后端请求的Url风格保持一致，就可以使用Hishtory API模式了。使用基于Hishtory API主要是操作HTML5提供的操作浏览器历史记录API，可以实现地址无刷新更新地址栏，例如：

```javascript
// pushState  state: 用于描述新记录的一些特性。
// 这个参数会被一并添加到历史记录中，以供以后使用。
// 这个参数是开发者根据自己的需要自由给出的。
// pushState作用相当于修改“#”后面的值
window.history.pushState(state, "title", "/userPage");
// 比如浏览器后退
window.addEventListener("popstate", function(e) {
    var state = e.state;
    // do something...
});
// 与pushState相对仅进行Url替换而不产生新的浏览记录的replaceState方法
window.history.replaceState(null, null, "/userPage");

```

#### 两种模式对比

&emsp;&emsp;我们来简单对比一下这两种模式的一些特点：    

|模式 VS | hash | Hishtory API |    
| ------ | ------ | ------ |    
| 美观程度 | 公认略丑，带#号 | 相对美观 |    
| 兼容性 | 好 | 需要浏览器支持HTML5 | 
| 错误URL后端支持 | 否 | 是 |  

&emsp;&emsp;所以两种模式的使用要根据实际项目的需求来定了，接下来是重点啦，Vue-Router内部实现路由就是基于这两种模式！所以提前了解一下前端路由的两种模式算是打个预防针。 

### Vue-Router
&emsp;&emsp; 先回忆一下Vue-Router的使用的四步曲，：

```javascrpit 
import Vue from 'vue';
import VueRouter from 'vue-router'
// 1. 使用Vue-Router插件
Vue.use(VueRouter)
// 2. VueRouter实例化
const router = new VueRouter({
  model: 'history', // 路由模式 hash\history
  routes: [
    {
      name: 'xx',
      path: 'path',
      component: component
    }
  ]
});
// 3. 实例化Vue实例时使用该router路由
new Vue({
  ...
  router,
});

// 4. 通过Vue.$router.push('xx') 或 router-link进行路由跳转
```

&emsp;&emsp;接下来，就逐次窥探上面四步曲中Vue-Router具体的一些实现细节。

#### 使用Vue-Router插件    
&emsp;&emsp;Vue-Router遵循Vue插件的开发规范，通过调用Vue内部方法Vue.use()对VueRouter进行install(实际上是回调VueRouter中所定义的install方法)，这一过程完成了VueRouter的挂载工作。
```javascript
-> vue\src\core\global-api\use.js
Vue.use = function (plugin: Function | Object) {
  ... 
  plugin.install.apply(plugin, args);
  ...
}

```

&emsp;&emsp;回到VueRouter源码中，分析一下install过程的执行流程：

```javascript
import View from './components/view'
import Link from './components/link'

export let _Vue

// install实现
export function install (Vue) {

  // 如果已注册实例，则返回
  if (install.installed && _Vue === Vue) return
  install.installed = true

  _Vue = Vue 

  const isDef = v => v !== undefined

  const registerInstance = (vm, callVal) => {
    // 父虚拟节点
    let i = vm.$options._parentVnode
    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
      i(vm, callVal)
    }
  }

  Vue.mixin({
    // 路由钩子函数，在vue执行beforeCreate钩子函数回调时会一起调用，因为beforeCreate是一个数组
    beforeCreate () {
      Vue.$options = options
      if (isDef(this.$options.router)) {
        this._routerRoot = this
        this._router = this.$options.router
        // 调用init
        this._router.init(this)
        // 调用Vue内部util方法使得当前的url实现响应式化
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      // 注册实例
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })
  // 将$router和$route挂载到Vue的原型链上
  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this._routerRoot._router }
  })

  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })

  // 注册路由视图和路由链接两个组件
  Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)

  // 执行Vue的选项合并策略
  const strats = Vue.config.optionMergeStrategies
  // use the same hook merging strategy for route hooks
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}

```    
&emsp;&emsp;可以看出，在install内部主要做了三个事：    
- 1. 就是利用Vue的mixin方法混入了全局beforeCreate和destoryed，在首次加载执行beforeCreate时，指定_routerRoot为当前的Vue实例，并且执行Vue-Router的init初始化工作（install最核心还是init初始化，这个具体细节稍后进行介绍），在beforeCreate方法内，主要是对父组件（也就是router所挂载的节点）进行初始化操作，并设置当前的_routerRoot为该组件对应的实例，在该逐渐中去初始化init路由的一些配置（比如设置路由模式等），其他组件比如（router-link, router-view会将_routerRoot设置为在options参数中包含了router选项的父(祖先)组件），并执行registerInstance方法，该方法中的registerRouteInstance目前其实是router-view组件中所定义的一个方法，主要作用是执行当前组件的渲染工作，如下:

```javascript
const registerInstance = (vm, callVal) => {
    // 父虚拟节点
    let i = vm.$options._parentVnode
    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
      i(vm, callVal)
    }
  }
````
&emsp;&emsp;这个混入相当于是定义，具体执行时等各组件在进行实例化时按照生命周期回调beforeCreate和destoryed这两个钩子函数，destroyed作用是在进行比如路由切换后、实际上作用就是清除上一个展示在router-view中的组件所渲染的内容。
- 2.  将$router和$route挂载到Vue的原型链上，可以通过this.$router和this.$route进行使用；

### 总结
&emsp;&emsp;由于希望篇幅不要太大，不然看起来比较吃力，所以本篇文章就写到这里，关于实例化和具体使用所涉及到的原理工作后面文章再讨论。在这对本篇文章做一个小结： 
   
- 简要介绍了前端路由，以及实现前端路由的两种模式 URL hash和history API并对这两者做了对比；
- 介绍了Vue是如何use Vue-Router组件，实际上内部就是回调Vue-Router内部定义的install方法；
- 介绍了install方法的一些主要工作职能：
  - 混入beforeCreate和destoryed方法；
  - 全局挂载$router和$route；
  - 注册router-link和router-view两个组件。







