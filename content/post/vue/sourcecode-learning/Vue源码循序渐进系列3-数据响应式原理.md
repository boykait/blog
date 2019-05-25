---
title: Vue源码循序渐进系列3-数据响应式原理
date: 2019-05-20
categories:
  - Vue
  - js
tags:
  - Vue源码循序渐进
---

&emsp;&emsp;Vue作为一种MVVM框架，能够实现数据的双向绑定，让Vue技术栈的前端从业者摆脱了繁琐的DOM操作，这完全得益于Vue框架开发者对原生Object对象的深度应用。Vue实现数据响应系统的核心技术就是数据劫持和订阅-发布，基本思想就是通过对数据操作进行截获，在数据进行getter操作的时候更新依赖，即依赖收集过程(更新订阅对象集)；在数据进行setter时通知所有依赖的组件一并进行更新操作(通知订阅对象)。Vue数据响应系统原理示意图如下:    

![Vue数据响应系统](/images/190522-vue_reactive_1.png)

接下来对Vue数据响应系统的这两个核心技术进行探究：

### 1. 数据劫持
&emsp;&emsp;Vue数据劫持主要利用Object.defineProperty来实现对对象属性的拦截操作，拦截工作主要就Object.defineProperty中的getter和setter做文章，看一栗子：    
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
&emsp;&emsp;可以看到，设置了属性的存取器后，在设置和获取person.name的时候，自定义的getter和setter将会被调用，这就给在这两个方法调用时进行其它内部的自定义操作提供了可能。了解了Object.defineProperty的基本使用后，我们开始从Vue源码角度开始逐步了解其具体的应用，在Vue的_init初始化过程中([传送门](./Vue源码循序渐进系列1-源文件追踪和_init方法初探.md))有一个initState操作，该操作主要依次对options中定义的props、methods、data、computed、watch属性进行初始化（主要分析对data的初始化initData）,该过程就主要是实现了将options.data变得可观察:
src/core/instance/state.js
```javascript
function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
   ...
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
     ...
     else if (!isReserved(key)) {
      // 设置data的数据代理
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  // 将data数据属性变为可观察
  observe(data, true /* asRootData */)
}
```
&emsp;&emsp;继续看对observe的定义:
```javascript
export function
observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  // 如果已经处于观察中，则返回观察对象，否则将value变为可观察
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    // 创建观察器
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
src/core/observer/index.js
// Observer 监听器类
class Observer {
  constructor (value: any) {
    this.value = value
    this.dep = new Dep() // 订阅器
    this.vmCount = 0
    def(value, '__ob__', this) // 为value添加__ob__属性
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      // 处理数组，数组中的每个对象都会进行可观察化操作
      this.observeArray(value)
    } else {
      // 处理对象，主要遍历对象中所有属性，并将所有属性变得可观察
      this.walk(value)
    }
  }
}


/**
 * Define a reactive property on an Object.
 */
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  // 定义一个订阅器
  const dep = new Dep()
   ...
  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      // 判断是否有自定义的getter， 没有则直接取值
      const value = getter ? getter.call(obj) : val
      // 判断是通过watcher进行依赖搜集调用还是直接数据调用方式(比如在template目标中直接使用该值，此时
      // Dep.target为undefined，该值为具有全局性，指向当前的执行的watcher)
      if (Dep.target) {
        dep.depend() // 将属性自身放入依赖列表中
        // 如果子属性为对象，则对子属性进行依赖收集
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) { // 处理数组的依赖收集
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      // 判断是否有自定义的getter， 没有则直接设置值，否则通过getter取值
      const value = getter ? getter.call(obj) : val
      ...
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      // 重新计算该对象的数据是否需要进行观察
      childOb = !shallow && observe(newVal)
      // 设置数据后通知所有的watcher订阅者
      dep.notify()
    }
  })
}
```
&emsp;&emsp;通过上面步骤将options中data的的属性变得可观察，defineReactive方法中的闭包方法set比较好理解，就如上面所说，设置新值newVal，并判断该值是否是为非基本数据类型，如若不是，可能就需要从新将newVal变得可观察；然后通知订阅器中所有的订阅者，进行视图更新等操作；get中的Dep.target不太好理解，我也是研究了一两天才明白，这里先不说，反正就是满足Dep.target不为undefined，则进行依赖收集，否则就是普通的数据获取操作，返回数据即可。

### 2. 订阅-发布
&emsp;&emsp;假使大家都晓得订阅-发布是个什么情况(不太清除可自行百度，不在此占据篇幅了)，那么，我们要知道，谁是订阅者，订阅的目标是什么？有了这个疑问，我们接下来看看相关的数据结构定义
	
#### 2.1 依赖收集(更新订阅者)
```javascript
// src/core/observer/dep.js
export default class Dep {
  static target: ?Watcher;  //target全局
  id: number;
  subs: Array<Watcher>;

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    // 循环对订阅者进行更新操作（调用watcher的update方法）
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

Dep.target = null
const targetStack = []

export function pushTarget (target: ?Watcher) {
  // 将当前的watcher推入堆栈中，关于为什么要推入堆栈，主要是要处理模板或render函数中嵌套了多层组件，需要递归处理
  targetStack.push(target)
  // 设置当前watcher到全局的Dep.target，通过在此处设置，key使得在进行get的时候对当前的订阅者进行依赖收集
  Dep.target = target
}

export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}


// src/core/observer/watcher.js
 addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        // 将订阅的watcher添加到当前的发布者watcher中
        dep.addSub(this)
      }
    }
  }
```

&emsp;&emsp;之前在getter的依赖收集过程中，当Dep.target成立时，会执行`dep.depend()`方法，可以看到，Dep中定义的depend()方法会调用` Dep.target`的`addDep()`方法，这个过程传递的参数就是在defineReactive中定义的dep对象，那么Dep.target到底是什么东西呢，这个可以看Watcher对应get方法中的定义:
```javascript
// Watcher -> get()
 get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      // 调用getter方法
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      // 深层次遍历
      if (this.deep) {
        traverse(value)
      }
      // 弹出
      popTarget()
      // 清除依赖，主要是针对有些发生了改变的依赖进行更新，比如新添加了依赖或者去除了原有依赖
      this.cleanupDeps()
    }
    return value
  }

// Dep-> pushTarget()
export function pushTarget (target: ?Watcher) {
  // 将当前的watcher推入堆栈中，关于为什么要推入堆栈，主要是要处理模板或render函数中嵌套了多层组件，需要递归处理
  targetStack.push(target)
  // 设置当前watcher到全局的Dep.target，通过在此处设置，key使得在进行get的时候对当前的订阅者进行依赖收集
  Dep.target = target
}
```
&emsp;&emsp;因为js是单线程执行，同一时刻只能执行一个Watcher，执行当前Watcher实例时候，Dep.target指向当前Watcher订阅者，当在执行下一个Watcher订阅者的get方法时候，即指向下一个订阅者，即Dep.target永远指向的是当前的Watcher订阅者，然后将当前订阅者添加到dep对应的subs订阅列表中，同时订阅者内部也需要记录有哪些订阅目标(dep)，便于进行依赖收集过程的更新操作。用一张图来表述：

![Dep-Watcher关系图](/images/190523-vue_reactive_1.png)

#### 2.2 更新通知
&emsp;&emsp;在数据劫持的闭包方法setter代码中，通过调用`dep.notify()`进行数据更新通知，这个阶段的主要工作就是将当前订阅目标dep更新消息通知到订阅列表中的订阅者（Watcher），然后订阅者利用注册的回调方法进行视图渲染等操作。
```javascript
// src/core/observer/dep.js
notify () {
	...
	// 循环对订阅者进行更新操作（调用Watcher的update方法）
	for (let i = 0, l = subs.length; i < l; i++) {
	  subs[i].update()
	}
 }

// src/core/observer/watcher.js
update () {
    /* istanbul ignore else */
    if (this.lazy) { // 是否为懒加载(这个何时执行？)
      this.dirty = true
    } else if (this.sync) { // 是否为同步方式更新
      this.run()
    } else { // 加入到订阅者更新队列(最终也要执行run方法)
      queueWatcher(this)
    }
  }
```
&emsp;$emsp;在run()中主要调用`Watcher -> get()`，在这说明一下`get()`中`this.getter.call(vm, vm)`的getter来源主要有两种-函数或表达式，函数比如render function、computed中的getter(), 表达式类似于"person.name"这种类型，经过`parsePath`转换成为getter()方法。[传送门](./Vue源码循序渐进系列4-Watcher那些事.md)中对Watcher的种类进行了总结。

### 总结
&emsp;&emsp;以上就是我对Vue数据响应式系统的一些学习，由于工作原因，前后花了一周左右的时间才写完，初步涉猎，估计还是有不少理解上的不到位，希望有幸能被大家看到并指出，这一过程中参考了不少的前人好文，太多就罗列一二，希望对大家也有帮助：     
   
- [Vue源码解读一：Vue数据响应式原理](https://www.jianshu.com/p/1032ecd62b3a)
- [深度解析 Vue 响应式原理](https://www.cnblogs.com/zhangycun/p/9463814.html)

