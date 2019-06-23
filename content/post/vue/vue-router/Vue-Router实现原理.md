---
title: Vue-Router原理分析
date: 2019-06-23
categories:
 - Vue-Router
 - Vue
tags:
 - Vue-Router
---

&emsp;&emsp;单页应用(SPA, Single Page Application)整个Web系统由一个html文件，通过Ajax和后端进行数据交互，整个过程无需刷新整体页面，通过一些特殊手段去加载渲染页面的不同部分，就向使用app一样，极大的提升了用户使用体验，本文从Vue的核心插件之一-Vue-Router角度出发，学习Vue-Router完成SPA开发的实现原理。 

### Web路由   
&emsp;&emsp;先简单说一下路由，早期Web系统路由主要是服务器后端实现，熟悉前后端开发的朋友就知道，在服务器端，通过路由表映射相应的方法，执行相应的动作，比如要请求API数据、新页面、文件等：

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
&emsp;&emsp;基于hash模式虽然方便，但带有#号，有同学就会觉这样的Url有点丑啦，
