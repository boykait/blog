---
title: Vue源码循序渐进系列4-Watcher那些事
date: 2019-05-25
categories:
  - Vue
  - js
tags:
  - Vue源码循序渐进
---
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
- 用户自定义Watcher