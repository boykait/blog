---
title: Vue-Router知识点收集
date: 2019-07-06
categories:
 - vue
tags:
 - vue-router
 - vue 
---

Vue-Router，前端路由，核心是改变视图的同时不向服务器端发送请求？ (history API模式是需要发送的哇？)

### 项目演示

介绍一下DEMO实现所用到的技术以及操作流程
可以演示一下从登陆到首页的展示，需要node的支持，这里主要包括介绍路由守卫拦截工作，以及可以展示两种路由模式
演示路由切换时，可以启动nginx配置服务
router还可以结合全选判断，比如在metadata中存入是否需要验证：auth:true,或者keepalive

是否有条件演示Vue-Router的平滑降级呢？下面是<=该系列版本的时候，会导致优雅降级，就是说不支持HTML5的浏览器版本：
IE9
Firefox 3.5
Chrome 3.0
Safari 3.0
Opera 10.5

### 路由守卫
有些需要权限有些不需要怎么实现？
### 路由
介绍路由的由来
#### 后端路由

### 前端路由
实现前端路由的两种模式：hash和history API

hash模式：什么是hash模式？
history API模式， HTML5中新增的API，pushState和replaceState，但需要特定的浏览器版本，换句话说，不支持History API模式，需要进行降级处理，降为hash模式，现在很多用vue-router开发页面的时候，都习惯使用hash路由莫模式，如：https://xxxx/#/index/share?code=dsfsd。这种模式在做pc端开发时候挺好用的。但是在app或者在微信开发的时候，#后面的内容容易被服务忽略，或者在微信授权和自定义分享的时候都会出现不同的状况，更重要的是，hash模式下，域名容易被封掉。

使用场景： hash模式的 ‘#’使得URL看起来不是很美观，如果有这个需求，那么可以配置使用history API模式，
优劣对比：
pushState() 设置的新 URL 可以是与当前 URL 同源的任意 URL；而 hash 只可修改 # 后面的部分，因此只能设置与当前 URL 同文档的 URL；
pushState() 设置的新 URL 可以与当前 URL 一模一样，这样也会把记录添加到栈中；而 hash 设置的新值必须与原来不一样才会触发动作将记录添加到栈中；
pushState() 通过 stateObject 参数可以添加任意类型的数据到记录中；而 hash 只可添加短字符串；
pushState() 可额外设置 title 属性供后续使用。
当然啦，history 也不是样样都好。SPA 虽然在浏览器里游刃有余，但真要通过 URL 向后端发起 HTTP 请求时，两者的差异就来了。尤其在用户手动输入 URL 后回车，或者刷新（重启）浏览器的时候。
hash 模式下，仅 hash 符号之前的内容会被包含在请求中，如 http://www.abc.com，因此对于后端来说，即使没有做到对路由的全覆盖，也不会返回 404 错误。
history 模式下，前端的 URL 必须和实际向后端发起请求的 URL 一致，如 http://www.abc.com/book/id。如果后端缺少对 /book/id 的路由处理，将返回 404 错误。Vue-Router 官网里如此描述：“不过这种模式要玩好，还需要后台配置支持……所以呢，你要在服务端增加一个覆盖所有情况的候选资源：如果 URL 匹配不到任何静态资源，则应该返回同一个 index.html 页面，这个页面就是你 app 依赖的页面。” 刷新界面或者是请求一个不存在的url会报404错误，这就需要服务器端的支持，为什么输入不存在的时候就要请求服务器端呢？正常的情况会请求服务器端吗？ 应该不会呢



父子路由
### 介绍Vue-Router的实现步骤
四个步骤
Vue-Router默认是使用的hash模式

### 造轮子