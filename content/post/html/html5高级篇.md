---
title: html基础2
date: 2018-11-12 15:11
categories:
 - 前端
tags:
 - html
---

### 
- 内联块：inline-block
&emsp;&emsp;默认情况下块元素占一行，使用内联属性能够将元素整合到一行，一旦设置这个属性且未设置宽度时，由内容决定撑开宽度。注ie6和ie7不支持。
```
display:inline-block
```
- 浮动：float
&emsp;&emsp;也可以让多个块元素实现在同行展示。使文档流脱离书写顺序，比如float:right 顺序相反
```
float: left|right|none|inherit
```
- clear    
&emsp;&emsp;设置后对应方向不会有浮动元素，为什么要清除浮动，因为浮动元素脱离文档，使得父级元素可能包不住子级元素，导致布局出现问题。
```
方法：
    1. 加高，无法适应子级元素高度动态变化
    2. 父级元素加浮动，需要加的层级太多，不合适，margin自动失效
    3. inline-block，导致左右失效
    4. after伪类清除浮动，设置浮动元素的父级样式，如：
    fatherClass:after {
        content: "";
        display: block;
        clear: both;
    }
```

- 定位：position
```
相对定位
绝对定位：
    使元素完全脱离文档流
    内嵌支持宽高
    有父级则对父级发生偏移，否则对document发生偏移
    相对一般配合绝对进行使用
    
z-index 定位层级
固定定位
    fixed，坚持不动，如右下角的回到顶部小图标按钮
static：默认值
inherit：从父级元素继承，不兼容
```