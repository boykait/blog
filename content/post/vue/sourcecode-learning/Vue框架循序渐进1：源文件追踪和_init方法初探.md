---
title: Vue框架循序渐进1：源文件追踪和_init方法初探
date: 2019-05-06
categories:
  - vue
tags:
  - vue
  - js
---

> vue无疑是当下最热的三大前端框架之一，也是平时主要使用的项目框架，没怎么经历过刀耕火种的原生页面开发时代，但也基本知道原生方式操作DOM是一件比较繁杂的事情，尤其是复杂的大型项目。vue以数据作为驱动，支持数据双向绑定，尽量避免了对原生DOM的直接操作。虽然写了不少这方面的业务代码，但对整个vue的精髓处于朦胧状态，便萌发了准备写一个vue的系列博客的想法，对它的源码一探究竟。

&emsp;&emsp;基于主流的vue 2.x版本，克隆的版本为2.6.10。克隆好vue源码后，就可以开始vue源码学习之旅了。    
&emsp;&emsp;先写一个vue版的helloworld：
```javascript
<html>
<head>
    <script src="./js/vue.js">
    </script>
</head>
<body>
<div id="app">
    {{message}}
</div>
<script>
    new Vue({
      el: '#app',
      data() {
        return {
          message: 'hello world'
        };
      }
    });
</script>
</body>
</html>

// 在浏览器中
打印message信息: hello world    
```    
&emsp;&emsp;上例中通过实例化Vue对象，并定于message字段，然后在该vue所在的页面作用域中使用该对象，message的具体数据被正确呈现在网页中，我们不禁好奇new Vue()过程中到底做了些工作?

### 1. 源码执行文件追踪     
&emsp;&emsp;vue中内置了很多执行脚本命令，在项目开发过程中，当执行**npm run dev**命令时，实际上会执行：    
```javascript
"dev": "rollup -w -c scripts/config.js --environment TARGET:web-full-dev"
```    
&emsp;&emsp;可以看到这条命令最终执行了scripts下的**config.js**脚本文件，config.js的作用主要就是制定入口文件和目标输出文件：        
```javascript    
// builds中针对不同的TARGET会执行不同的入口文件和产生不同的目标脚本
const builds = { 
...
'web-full-dev': {
    entry: resolve('web/entry-runtime-with-compiler.js'),  // 入口文件，这需要经过alias路径拼接转换找到最终src\platforms\web\entry-runtime-with-compiler.js
    dest: resolve('dist/vue.js'),  // 编译产生的目标文件
    format: 'umd',
    env: 'development',
    alias: { he: './entity-decoder' },
    banner
  },
···
}; 

// 根据不同的TARGET产生对应的配置项
function genConfig(name) {...}

// 判断命令中是否带有TARGET属性，比如"dev": "rollup -w -c scripts/config.js --environment TARGET:web-full-dev"
// 就可以只产生web-full-dev类型的配置
if (process.env.TARGET) {
  module.exports = genConfig(process.env.TARGET)
} else {
  // 否则对所有的builds中所有项产生对应的配置信息
  exports.getBuild = genConfig
  exports.getAllBuilds = () => Object.keys(builds).map(genConfig)
}
```   
&emsp;&emsp;好了，找到了vue执行的**entry-runtime-with-compiler.js**入口脚本文件，文件中引入了Vue的定义，沿着Vue的定义引用链，在**src\core\instance**找到最终关于Vue的原始定义：    
```javascript 
-> import Vue from './runtime/index'
-> import Vue from 'core/index'
-> import Vue from './instance/index'
// 使用函数构造器方式而不使用class便于定义一些混入方法
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  // 执行\src\core\instance\init.js脚本中挂载的_init方法
  this._init(options)
}
```

### 2. Vue的_init初始化
&emsp;&emsp;执行_init方法主要是实现vue的初始化操作，来看一下_init方法的一些定义实现：    
```javascript
Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    ...

    // a flag to avoid this being observed
    vm._isVue = true
    // merge options
    // 合并optons项
    // 如果是子组件
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      // 直接合并
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm) // 代理初始化
    } else {
      vm._renderProxy = vm
    }
    // expose real self
    vm._self = vm
    initLifecycle(vm) // 生命周期初始化
    initEvents(vm)   // 事件初始化
    initRender(vm)   // 渲染初始化
    callHook(vm, 'beforeCreate')  // 回调beforeCreate
    initInjections(vm) // resolve injections before data/props  提供将父组件的provide数据注入到当前组件
    initState(vm)      // 初始化数据state状态
    initProvide(vm) // resolve provide after data/props  将当前组件的provide数据初始化以提供子组件使用
    callHook(vm, 'created')  // 回调created方法
    
    ...

    if (vm.$options.el) {
      vm.$mount(vm.$options.el) // 执行$mount方法挂载
    }
  }
```
        
&emsp;&emsp;生命周期、事件等通过[]方式挂载到了Vue原型上，接下来主要挨着介绍这几个方法的调用。

#### 2.1 initLifecycle
&emsp;&emsp; 初始化生命周期除了主要初始化一些自身的属性，最主要是将自身作为孩子节点挂载到非抽象上级父组件下，以便于最终能够形成树形结构。
```javascript
initLifecycle (vm: Component) {
  const options = vm.$options

  // locate first non-abstract parent
  let parent = options.parent
  if (parent && !options.abstract) {
    // 抽象组件自身不会渲染一个 DOM 元素，也不会出现在父组件链中，比如<keep-alive> <transition>。
    // 父组件为抽象类型，则跳过，直接向着parent链向上级寻找，直到找到非抽象类型的父组件
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    // 将该组件作为最近非抽象父组件的子组件
    parent.$children.push(vm)
  }

  vm.$parent = parent  // 设置当前组件的父组件
  vm.$root = parent ? parent.$root : vm // 设置 $root

  vm.$children = []
  vm.$refs = {}

  vm._watcher = null
  vm._inactive = null
  vm._directInactive = false
  vm._isMounted = false
  vm._isDestroyed = false
  vm._isBeingDestroyed = false
}
```
#### 2.2 initEvents

```javascript
initEvents (vm: Component) {
  // vm._events表示的是父组件绑定在当前组件上的事件
  vm._events = Object.create(null)
  vm._hasHookEvent = false
  // init parent attached events
  // 父组件所绑定到当前组件上的方法，
  const listeners = vm.$options._parentListeners
  if (listeners) {
    // 将父组件模板中注册的事件放到当前组件实例的listeners里面
    updateComponentListeners(vm, listeners)
  }
}
```
#### 2.3 initRender
```javascript
initRender (vm: Component) {
  vm._vnode = null // the root of the child tree
  vm._staticTrees = null // v-once cached trees
  const options = vm.$options
  const parentVnode = vm.$vnode = options._parentVnode // the placeholder node in parent tree
  // 渲染的内容
  const renderContext = parentVnode && parentVnode.context
  // 处理插槽
  vm.$slots = resolveSlots(options._renderChildren, renderContext)
  vm.$scopedSlots = emptyObject
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  // 使用tempate模板方式定义的组件
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  // normalization is always applied for the public version, used in
  // user-written render functions.
  // 使用render function定义的渲染方式
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

  // $attrs & $listeners are exposed for easier HOC creation.
  // they need to be reactive so that HOCs using them are always updated
  const parentData = parentVnode && parentVnode.data

  /* istanbul ignore else */
  if (process.env.NODE_ENV !== 'production') {
    // 非生产环境，定义数据响应式渲染更新，
    // 内部采用了订阅-发布模式对数据进行监听(后面会进一步研究)
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
    }, true)
    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$listeners is readonly.`, vm)
    }, true)
  } else {
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true)
    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, null, true)
  }
}
```
#### 2.4 callHook
&emsp;&emsp;callHook方法，作用主要是对Vue定义的钩子方法进行回调，Vue内部定义了一些回调的方法，这些方法在Vue实例的不同阶段发挥作用，init阶段主要使用Vue的beforeCreate和created：   
```javascript
  // src/core/config.js下：
  // _lifecycleHooks: [
  //   'beforeCreate',
  //   'created',
  //   'beforeMount',
  //   'mounted',
  //   'beforeUpdate',
  //   'updated',
  //   'beforeDestroy',
  //   'destroyed',
  //   'activated',
  //   'deactivated'
  // ],
  pushTarget() // ？这个作用待考究
  const handlers = vm.$options[hook] //获取回调方法的引用
  const info = `${hook} hook`
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      invokeWithErrorHandling(handlers[i], vm, null, vm, info)
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook)
  }
  popTarget()
```    
#### 2.5 initInjections和initProvide
&emsp;&emsp;这两者的作用是配对的，提供Vue组件的依赖注入功能，和后端第三方注入方式不太一样，父组件提供数据，子组件根据父组件提供的进行数据注入，initInjections方法提供将父组件的provide数据注入到当前组件，initProvide方法将当前组件的provide数据初始化以提供子组件使用。(Vue通过订阅发布模式实现数据的变更侦听，后面会专门探究这方面的实现原理)
```javascript
\src\core\instance\inject.js
export function initInjections (vm: Component) {
  const result = resolveInject(vm.$options.inject, vm)
  if (result) {
    toggleObserving(false)
    // 循环遍历所有的注入数据进行响应式监听
    Object.keys(result).forEach(key => {
      /* istanbul ignore else */
      if (process.env.NODE_ENV !== 'production') {
        defineReactive(vm, key, result[key], () => {
          warn(
            `Avoid mutating an injected value directly since the changes will be ` +
            `overwritten whenever the provided component re-renders. ` +
            `injection being mutated: "${key}"`,
            vm
          )
        })
      } else {
        // 数据响应定义
        defineReactive(vm, key, result[key])
      }
    })
    toggleObserving(true)
  }
}

export function initProvide (vm: Component) {
  // 将provide数据绑定到_provide, 如果是方法，将方法返回结果绑定到provide上
  const provide = vm.$options.provide
  if (provide) {
    vm._provided = typeof provide === 'function'
      ? provide.call(vm)
      : provide
  }
}
```

#### 2.6 initState
&emsp;&emsp;initState数据状态初始化，主要对Vue组件中的props、methods、data、computed、watch等和数据绑定相关的定义进行初始化操作：
```javascript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)  // 初始化props
  if (opts.methods) initMethods(vm, opts.methods) // 初始化methods
  if (opts.data) {  //初始化data
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)   // 初始化计算属性
  if (opts.watch && opts.watch !== nativeWatch) {  // 初始化watch属性
    initWatch(vm, opts.watch)
  }
}
```