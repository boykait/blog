---
title: Vue源码循序渐进2：$mount 
date: 2019-05-12
categories:
  - Vue
tags:
  - Vue源码循序渐进系列
---

&emsp;&emsp;看一下Vue源码中_init()方法最后一步:
```javascript
if (vm.$options.el) {
      vm.$mount(vm.$options.el) // 执行$mount方法挂载
}
```
&emsp;&emsp;这样我们就能够理解了Vue官网中所说的：如果 Vue 实例在实例化时没有收到 el 选项，则它处于“未挂载”状态，没有关联的 DOM 元素。可以使用 vm.$mount() 手动地挂载一个未挂载的实例。如果没有提供 elementOrSelector 参数，模板将被渲染为文档之外的的元素，并且你必须使用原生 DOM API 把它插入文档中。这个方法返回实例自身，因而可以链式调用其它实例方法。
&emsp;&emsp;实例化的Vue挂载到指定的DOM下，Vue的挂载分两种方式，初始化时指定el挂载和手动挂载$mount，后者灵活一些，可以在Vue初始化之后，按需延迟挂载：
```javascript
new Vue({
  el: '#app',
  data: obj
})

new Vue({
  ...
}).$mount('#app')
```
### 1. 了解template和render
&emsp;&emsp;Vue倡导组件化编程，所以我们常常会自定义一些组件满足业务需求，Vue中有两种方式去开发组件：template和render:
```javascript
template模板方式：
new Vue({
      el: '#app',
	  template: `<h4 style="color:red">我是template模板</h4>`,
    });

render函数方式：
new Vue({
	  el: '#app',
	  render: (createElement) => {
		return createElement('div', {//一个包含模板相关属性的数据对象
				'class': {
					foo: true,
					bar: false
				},
				style: {
					color: 'red',
					fontSize: '14px'
				},
				attrs: {
					id: 'foo'
				},
				domProps: {
					innerHTML: '我是render函数'
				}
			});
	  }
	});
```
&emsp;&emsp;实际上二者最终殊途同归，template模板最终需要被转换编译成render函数，所以从这一过程可知，后者的效率会更高，但是开发难度无疑更大。
> 注： 如果一个组件中同时含有template和render，后者的优先级会更高，也就意味着后者会将前者覆盖。
### 2. $mount源码导读
$mount源码：
    
```javascript
/src/platforms/web/entry-runtime-with-compiler.js

// ./entry-runtime.js中定义的$mount方法
const mount = Vue.prototype.$mount
// 定义全局$mount方法
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  // 查找目标容器
  el = el && query(el)

  // 目标容器不能为body和html
  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }

  const options = this.$options
  // resolve template/el and convert to render function
  // 判断是否是通过render方法还是template方式构建DOM， 即判断是否有render方法存在
  if (!options.render) {
    // 模板方式
    let template = options.template
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          // 根据id获取对应模板
          template = idToTemplate(template)
          ...
        }
      } else if (template.nodeType) { // 直接为模板字符串
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
    } else if (el) {
      // 获取el的outerHtml作为模板
      template = getOuterHTML(el)
    }
    if (template) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile')
      }

      // 将template模板编译成render渲染函数
      const { render, staticRenderFns } = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns
      ...
    }
  }
  return mount.call(this, el, hydrating)
}  
```    
&emsp;&emsp;其中compileToFunctions应该说是$mount的核心，该方法就是实现了template模板到render函数的编译过程，该过程主要有三步骤：   
![Template编译生成render函数过程](images/190512-vue_$mount_2.png)     
&emsp;&emsp看一下这个方法的相关定义（后面相关文章再对该过程进行详细展开研究）:
```javascript
->/src/platforms/web/compiler/index.js
 ->/vue/src/compiler/index.js
export const createCompiler = createCompilerCreator(function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  // 将模板转换AST(Abstract Syntax Tree)抽象语法树:
  const ast = parse(template.trim(), options)
  if (options.optimize !== false) {
    // 对AST节点进行静态节点标记
    optimize(ast, options)
  }
  // 主要产生render函数和staticRenderFns函数
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
})
```
&emsp;&emsp;$mount方法最终会调用定义的mountComponent方法进行真正的组件挂载：
```javascript
-> src/latforms/web/runtime/index.js （mountComponent(this, el, hydrating)）
 -> /src/core/instance/lifecycle.js
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  // 判断是是否有render渲染函数
  if (!vm.$options.render) {
    // 创建空虚拟节点VNode
    vm.$options.render = createEmptyVNode
    ···
  }
  // 执行挂载前beforeMount回调操作
  callHook(vm, 'beforeMount')

  let updateComponent
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    // 执行更新组件
    updateComponent = () => {
      ···
      vm._update(vnode, hydrating)
     ···
    }
  } else {
    // 执行更新组件
    updateComponent = () => {
      // vm._render() 由vm.$options.render()生成的vnode节点
      vm._update(vm._render(), hydrating)
    }
  }

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    // 设置 _isMounted标志位，在视图进行更新操作时使用
    vm._isMounted = true
    // 回调mounted钩子
    callHook(vm, 'mounted')
  }
  return vm
}
```
### 3. 总结：$mount生命周期
&emsp;&emsp;先贴一张Vue官网的生命周期图:
![Vue生命周期](/images/190512-vue_$mount_1.png)    
&emsp;&emsp;$mount主要出于图中标记的红框区域，根据流程图逻辑梳理$mount主要的工作：    
```javascript
1. 判断初始化options中是否包含“template”属性，有则先将其编译转化为render函数，否则el的el的outerHtml作为模板然后再编译转换；    
2. 执行mountComponent组件挂载；
```