---
title: JVM内存模型
date: 2019-09-20
categories:
 - JVM
 - java
tags:
 - JVM
 - java
---

> 分析对比Java7和Java8的内存模型差异，由此开启学习JVM的路程，

### 1. Java7-JVM内存模型
&emsp;&emsp;基本概念也不再copy了，直接上自己画的内存模型图：

![Java7-JVM内存模型图](/images/190920-jvm_memory_model_1.png)

&emsp;&emsp;JVM内存模型的运行时数据区域包含的5个主要的部分想必不陌生，线程共享部分：方法区、堆，线程私有部分：栈、本地方法栈、程序计数器，不过，光看这个也没用啊，还是想想java程序运行过程中到底经历了什么吧？
&emsp;&emsp;整一个简单的栗子：
```java
package com.boykait.memorymodule;

public class MemoryModuleMain {
    private int sum() {
        int a = 1;
        int b = 2;
        int c = (a + b) * 10;
        return c;
    }

    public static void main(String[] args) {
        MemoryModuleMain mm = new MemoryModuleMain();
        System.out.println( mm.sum());
    }
}

结果： 30

```
&emsp;&emsp;我们的代码在经过编译后生成对应的二进制文件`MemoryModuleMain.class`，执行时首先加载该文件（具体类加载过程以后再进一步深入学习），加载到JVM的方法区成为类对象，用javap看看这个文件反编译对应指的令集:`javap -c -l MemoryModuleMain.class > MemoryModuleMain.text`：

```java
Compiled from "MemoryModuleMain.java"
public class com.boykait.memorymodule.MemoryModuleMain {
  public com.boykait.memorymodule.MemoryModuleMain();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return
    LineNumberTable:
      line 3: 0
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
          0       5     0  this   Lcom/boykait/memorymodule/MemoryModuleMain;

  public int sum();
    Code:
       0: iconst_1
       1: istore_1
       2: iconst_2
       3: istore_2
       4: iload_1
       5: iload_2
       6: iadd
       7: bipush        10
       9: imul
      10: istore_3
      11: iload_3
      12: ireturn
    LineNumberTable:
      line 5: 0
      line 6: 2
      line 7: 4
      line 8: 11
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
          0      13     0  this   Lcom/boykait/memorymodule/MemoryModuleMain;
          2      11     1     a   I
          4       9     2     b   I
         11       2     3     c   I

  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class com/boykait/memorymodule/MemoryModuleMain
       3: dup
       4: invokespecial #3                  // Method "<init>":()V
       7: astore_1
       8: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
      11: aload_1
      12: invokevirtual #5                  // Method sum:()I
      15: invokevirtual #6                  // Method java/io/PrintStream.println:(I)V
      18: return
    LineNumberTable:
      line 12: 0
      line 13: 8
      line 14: 18
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
          0      19     0  args   [Ljava/lang/String;
          8      11     1    mm   Lcom/boykait/memorymodule/MemoryModuleMain;
}

```


1. 内存模型
2. 每个线程的栈帧有多少层（每个方法压入栈叫栈帧）能够分配多少个线程
栈帧的组成
javap
程序计数器里面存放的什么
JVM指令手册
Java7和Java8的内存模型的区别（Java8中的方法区？ 元空间）
栈堆关系  方法区 堆关系

对象组成 对象头， 类元指针 调用对应方法