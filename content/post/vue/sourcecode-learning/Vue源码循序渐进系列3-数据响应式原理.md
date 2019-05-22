---
title: Vue源码循序渐进系列3-数据响应式原理
date: 2019-05-20
categories:
  - Vue
  - js
tags:
  - Vue源码循序渐进
---

&emsp;&emsp;Vue作为一种MVVM框架，能够实现数据的双向绑定，让Vue技术栈的前端从业者摆脱了繁琐的DOM操作，这完全得益于Vue框架开发者对原生Object对象的深度应用。这里就主要分析实现Vue数据的响应式设置和监听的原理：数据劫持和订阅-发布。       

&emsp;&emsp;Vue实现数据响应系统的核心技术就是数据劫持和订阅-发布，基本思想就是通过对数据操作进行截获，在数据进行getter操作的时候更新依赖，即依赖收集过程(更新订阅对象集)；在数据进行setter时通知所有依赖的组件一并进行更新操作(通知订阅对象)。Vue数据响应系统原理示意图如下:

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
      // 处理数组
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
&emsp;&emsp;订阅-发布
#### 2.1 订阅器Dep


