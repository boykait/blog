---
title: canvas 航线、彗星扫尾轨迹效果
date: 2019-01-21 21:54
categories:
 - 前端
tags:
 - canvas
---

#### 背景
- 最近在研究canvas相关的项目，有个需求就是在绘制的贝塞尔曲线上有一个彗星尾巴的效果，网上找了一些案例，如：
- [markline.js——轻量级canvas绘制标记线的库](https://www.jqhtml.com/10559.html)
- [canvas抛物线运动轨迹](https://www.cnblogs.com/anxiaoyu/p/6676957.html)
- [动画—圆形扩散、运动轨迹](https://segmentfault.com/a/1190000008560571canvas)
- 但都没能满足要求，不过还是感谢这些博文提供的一些思路，思前想后，终于实现相关的demo：

#### 知识点
- canvas基本知识

#### 实现思路
- 思路其实不难，主要有如下步骤
      - 1 绘制贝塞尔曲线，起始点、控制点、终点自定；
      - 2 定义小球数组，该小球数组记录贝塞尔曲线上每一份对应的x, y坐标以及小球坐标点的透明度、半径（小球的透明度由0->1降低，半径逐渐增大），当小球数组长度超过指定数组长度，清除最先进入的一个小球对应的坐标；
      - 根据贝塞尔曲线轨迹绘制当前的小球数组。

- ok，看一下效果图吧：

![](https://img2018.cnblogs.com/blog/1297782/201901/1297782-20190121014903966-848092301.gif)

- 顺附demo源码：

```
    <!DOCTYPE HTML>
<html lang="en">

	<head>
		<meta charset="UTF-8">
		<title></title>
		<style type="text/css">
			html,
			body {
				padding: 0;
				margin: 0;
				
			}
			canvas {
				border: 1px solid red;
			}
		</style>
	</head>

	<body>
		<canvas id="canvas" width="500" height="500"></canvas>

		<script type="text/javascript">
			var canvas = document.getElementById('canvas');
			var ctx= canvas.getContext('2d');
			var width = canvas.width;
			var height = canvas.height;
		
			//画贝塞尔曲线参数 起始 两个控制点，终点以及当前和总的比例数
			let sx = 10,
			sy = 10,
			cx1 = 50,
			cy1 = 10,
			cx2 = 150,
			cy2 = 500,
			ex = 500,
			ey = 500,
			t = 0,
			n = 200;
			//小球运动轨迹信息数组
			let bubbles = [{a:1, x:1, y: 1, r:4, s: 1}];

			let bubbleNum = 0;
			function test() {
				t++;
				bubbleNum++;
				ctx.clearRect(0, 0, 5000, 5000);
				ctx.lineWidth = 3;
				ctx.strokeStyle = 'green'
				ctx.beginPath();
				ctx.moveTo(sx, sy);
				ctx.bezierCurveTo(cx1, cy1, cx2, cy2, ex, ey);
				ctx.stroke();
				//计算当前的贝塞尔曲线坐标
				let x = sx * Math.pow((1 - t / n), 3) + 3 * cx1 * (t / n) * Math.pow((1 - t / n), 2) + 3 * cx2 * Math.pow((t / n), 2) * (1 - t / n) + ex * Math.pow((t / n), 3);
    			let y = sy * Math.pow((1 - t / n), 3) + 3 * cy1 * (t / n) * Math.pow((1 - t / n), 2) + 3 * cy2 * Math.pow((t / n), 2) * (1 - t / n) + ey * Math.pow((t / n), 3);

				//移动操作
				if (bubbleNum > 50) {
					bubbles.shift()
				}

				for(var i=0;i<bubbles.length;i++){
					bubbles[i].a = (i+1)*0.02;
					bubbles[i].s = (i+1)*0.02;					
				}
				let b = {a: 1, s: 1, x: x, y: y};
				bubbles.push(b);
				//渲染bubble数组
				for (let j = 0; j < bubbles.length; j++) {
					let b = bubbles[j];
					ctx.save();
					ctx.beginPath();
					ctx.globalAlpha = b.a; // 值是0-1,0代表完全透明，1代表完全不透明
	    			ctx.fillStyle = 'greenyellow';
	   				ctx.arc(b.x, b.y, b.s * 4, 0, 2 * Math.PI, false);
	    			ctx.fill();
	    			ctx.restore();
				}
				
    			if (x == 500) {
					t = 0;
				}
				console.log(t);
				requestAnimationFrame(test);
			}
			
			test();
		</script>
	</body>

</html>
```

- 效果是实现了，但是每次需要清除整个canvas画布，如果需要重绘的内容较多，性能上估计需要进一步优化了。