---
title: html基础1
date: 2018-11-10 11:54
categories:
 - 前端
tags:
 - html
---

### 第一章
- 样式：style
```
1. 行间样式：单标签使用样式
直接写在html标签中。
<div style="width=10px;heigth=20px;">
</div>

2. 内联样式：单文件多标签共用
    <style>
      .xx {
        width:10px;
        height:20px;
      }
    </style>
    
3. 外联样式：多文件共用
    <link rel="stylesheet" href="demo.css"/>
    rel="stylesheet"告诉浏览器这是link过来的是一个样式文件
```

- 边框：border
```
包括三部分：粗细度、样式（实线、虚线）、 颜色
    如：border:1px solid red 
    
可以设置border各个方向的样式
    border-top, -right, -bottom, -left
```

- 背景：background  
&emsp;&emsp;不会占用容器宽高度
```
background-color: #0cc
background-image: url(imag/1.jpg)
background-repeat:
    no-repeat repeat-x repeat-y repeat(默认)
background-position:
    10px 10px
    10% 10%
    left|center|right top|center|bottom
background-attachment: 是否滚动（默认滚动）
    fixed, scoll
```

- 文字设置：font
```
font-weight: bold #文字加粗bold normal
font-style: italic #斜体
font-size: 10px  #大小
font-family: "楷体" #字体
```
- 文本设置：text
```
color
text-align: #对齐方式 left|center|right(默认left)
text-indent: 2em; #首行缩进(em缩进单位)
text-decoration: underline #下划线 underline none
letter-spacing: 10px; #每个字符的间距
word-spacing: 10px; #以空格为单位进行解析
```

- 内边距：padding
```
设置后会撑大容器的大小：
padding-top
-left
-right
-bottom

还可以分组设置左右和上下的大小：
padding: 100px 50px; #设置上下100px 左右50px
padding: 100px 50px 20px; #上100px 左右50px 下20px
padding: 100px 50px 20px 10px; #上100px 左50px 下20px 右10px
```

- 外边距：margin
```
设置后会撑大容器的大小：
margin-top
-left
-right
-bottom

问题1： margin-top传递，子div能传递给父div
    解决：设置子div的该属性会导致父div的top属性，设置border数据能解决这个问题。
问题2：margin上下叠压
    解决：使用marginkey将某一个元素的margin方向设置成预想的值，margin叠压会取最大的margin。
```

-盒模型
&emsp;&emsp;s设置标签的大小和边距、边框。
```
盒子大小
盒子高度
盒子宽度
```

### 第二章
- a标签 超链接
```
伪类：给元素添加一些默认的效果，比如：
    <style>
			a:hover { #鼠标悬停的颜色
				color: red
			}
			a:link {  # 未访问初始颜色
				color: black;
			}
			a:visited { #访问后的颜色
				color: plum;
			}
		</style>
a标签还有很多其他的属性设置
```
- 语义标签
```
页眉：header 
导航：nav 链接列表
页脚：footer
板块：section #划分页面不同的区域， 先用section 再用div，区域级别的，比如项目详情里的详情页面的基本信息和日志就是两个不同的区域，标准的就该用section进行区分。
article： 结构完整的独立内容    
侧边栏：aside 和主版块无关的，比如页面引用、广告位等
```

- 其它主要常用使用的标签
```
h1~h6 #标题 一个页面最好只有一个：h1-logo
p #段落
strong #强调 加粗
em #强调 斜体
span #区分样式
ul #组合标签 无序列表<ul><li></li></ul>
ol #有序标签 有编号
mark #标记
```

- 标签样式初始化：reset   
&emsp;&emsp; 浏览器的默认样式最好都不要使用，不同浏览器的样式可能不一致，所以需要自定义初始化样式
```
需要重置项：
    盒模型相关
    标签特有样式
    如：
    h1,h2,h3,p,dd,dl {
        margin: 0
    }
    ul,ol {
        margin: 0;
        padding: 0;
        list-style: none
    }
```

- 选择器    
&emsp;&emsp;id选择器、类选择器、标签选择器、群组选择器、包含选择器以及通配符
```
<style>
    #idxx {  #id选择器，唯一
        
    }
    
    .classxx { #样式选择器
        
    }
    
    div { #标签选择器
         
    }
    
    div, a, p { #群组选择器
        
    }
    
    .classxx div { #包含选择器，空格以表示下级
        
    }
    
    [域] * {
        
    }
</style>
```

- 选择器优先级    
&emsp;&emsp;style(行间样式)> id选择器 > class选择器 > 内联选择器 > 外联选择器


#### 第三章
- 块元素和内联元素
```
块元素：非常霸气的撑满一行的元素或能够撑满父级元素，支持所有的css： div p
内联元素：不支持宽高，内容撑开宽度，代码换行被解析，不支持margin
```

