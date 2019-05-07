---
title: vue框架初探
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