---
title: lambda初识
date: 2018-02-27
categories:
- 后端
tags:
- Java
---
>java繁琐的语法也被摒垢已久，在java8中引入的lambda这一个非常重要的特性。这标志着Java往函数式编程迈进了一小步，java程序猿也可以写出简单优雅的代码了。

### 1. 函数一等公民等级由来
&emsp;&emsp;或许可以说这么说，如果没有将函数进行一等公民，java8中stream、lambda这些新的特性根本无从谈起，函数一等功民化并非由java8独家首创，早在python、javascript、scala等语言中运用得轻车熟路。在传统的一些语言，比如c/c++/c#以及java8之前的语言中，整个编程语言的核心就在于操作值，so，值被当作一等公民，拥有能够进行赋值、传参、返回等操作，与此相对，方法和类等概念就被冠以二等功名的头衔，所谓二等，必然就主起辅助作用。
### 2. 匿名内部类和lambda
&emsp;&emsp;函数作为实参进行传递在c/c++中非常常见(表现为指针形式)，在java中则需要利用匿名内部类，先看一组简单的线程实现代码对比：

```
new Thread(new Runnable() {
	@Override
	public void run() {
		System.out.println("匿名内部类式创建线程!");
	}
}).start();

new Thread(() -> System.out.println("lambda式创建线程 !")).start();
```
输出结果:

```
匿名内部类式创建线程!
lambda式创建线程 !
```
&emsp;&emsp;可以看出，lambda式将操作行为参数化，实现现成的方式较匿名内部类着实简单太多了，而且代码的可读性也好不少。
### 3. lambda和函数式接口
&emsp;&emsp;如上面一个例子中，Runnable作为一个接口，仅包含一个run方法，但必须要写一大堆啰里啰嗦的代码，在JDK中，还有很多比如Callable、ActionListener等，java8中将这样的接口进行简化，即函数式接口，表达形式： ()-> {}。

```
(), 表示参数列表；
{}，lambda主体，即要实现的代码逻辑；
->, 将参数和lambda主体分开。
```
比如：

```
/Callable
//旧api      
Callable<String> c1 = new Callable<String>() {
	@Override
	public String call() throws Exception {
		return "done";
	}
};
//java8
 Callable<String> c2 = () -> "done";

//comparator比较器
//旧api      
Comparator<String> c3 = new Comparator<String>() {
	@Override
	public int compare(String s1, String s2) {
		return s1.compareToIgnoreCase(s2);
	}
};
//java8 
Comparator<String> c2 = (s1, s2) -> s1.compareToIgnoreCase(s2);
```

### 4. lambda在集合中的一些应用
&emsp;&emsp;再看一道例子：

```
// code:
List<Integer> list = Arrays.asList(1, 4, 5, 6, 2, 3);
for(Integer i: list) {
	System.out.print(i + "  ");
}
// result:
1
4
5
6
2
3
```
&emsp;&emsp;上面这道例子使用了外部迭代方法遍历并打印列表原始，外部迭代在可以说是之前对数据列表最常进行的操作之一，但外部迭代有如下不足之处：

 - 只能顺序处理数据；
 - 不能充分利用CPU多核优势；
 - 不利于编译器的优化工作。
 
&emsp;&emsp;在java8中，将外部迭代转换为内部迭代,同样能够输入相同的结果:

```
//可以写成：
list.forEach(i -> System.out::println(i));
//甚至可以写成 
list.forEach(System.out::println);
```
 
&emsp;&emsp;此外，java8中对集合的操作进行了进一步的丰富，在Collection集合类中添加了stream方法，这两个方法会将Collection的实例转换为数据流(关于流将会在后面进一步展开学习)，将lambda和stream结合起来使用，在进行数据的流水线化处理时优势极为明显。
&emsp;&emsp;又来例子：获取列表中前50个大于20的数据的列表： 

```
//旧方法
List<Integer> resList1 = new ArrayList<>();
int num = 0;
for(Integer i: list) {
   if(num > 50) {
	   break;
   } else if(i > 20) {
	   resList1.add(i);
	   num++;
   }
}
//lambda
List<Integer> resList2 = list.stream().filter(i -> i > 20).limit(50).collect(Collectors.toList());
	//旧方法
List<Integer> resList1 = new ArrayList<>();
int num = 0;
for(Integer i: list) {
   if(num > 50) {
	   break;
   } else if(i > 20) {
	   resList1.add(i);
	   num++;
   }
}
//lambda
List<Integer> resList2 = list.stream().filter(i -> i > 20).limit(50).collect(Collectors.toList());
```

&emsp;&emsp;将列表转换为流能够更好地利用多核的优势，直接将stream用parallelStream替代即可。
&emsp;&emsp;stream中的还有很多的方法比如map、reduce、distinct、skep、findAny等很多方法，可以将这些方法进行流水线式拼接组合以满足自身的业务逻辑需求，在这不写太多例子了，自己动手实践吧