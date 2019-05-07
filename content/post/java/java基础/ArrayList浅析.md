---
title: ArrayList浅析
date: 2018-09-14 13:13:47
categories:
 - 后端
tags:
 - Java
---
### 1. 概述
&emsp;&emsp;原以为数组初始化设置了大小，要是数据量超标了，那就没得治了，什么时候有的动态数组这个概念呢？应该是大三暑假一个培训机构的老师帮我们讲的课程里面提及的，说白了就是用C语言的库函数进行数据的拷贝扩容呗，万变不离其宗，java里的ArrayList就相当于是C版的动态数组，本小文主要看看Java中的ArrayList的一些特性。  
&emsp;&emsp;前些年面试总爱被问及ArrayList和LinkedList的区别，不过好像有点现在不入这些大厂的法眼了，哈哈，不过，在实际的开发过程中，ArrayList用得很多，对于几十条或几百条规模的又不确定具体数据量的场景用着屡试不爽，ArrayList本身实现了List、Serializable、RandomAccess、Cloneable等接口，针对ArrayList这个类，本身是比较简单的，我觉得重点可以去学习一下ArrayList的初始化、扩容原理和序列化，其它的有精力可以再探究探究。

### 2. 初始化和扩容
&emsp;&emsp;先看一下ArrayList内部定义的几个变量：

```java
//默认初始化大小
private static final int DEFAULT_CAPACITY = 10;

//长度为0的数组
private static final Object[] EMPTY_ELEMENTDATA = {};

//初始化空数组
private static final Object[]
DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

//存取数据所使用到的数组
transient Object[] elementData;

//数据大小
private int size;
```

对应ArrayList提供了带参和不带参两种构造方法，以前没了解ArrayList的底层时，一直以为在没有传初始化大小时会立即返回一个大小默认为10的数组，但是实际并非如此，看一下不带参的构造方法：

```java
   public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```

&emsp;&emsp;指定了初始化initialCapacity容量大小的构造方法很简单，直接创建一个对应容量大小的数组，那么，调用未设置初始容量的构造方法呢？不带参的ArrayList构造方法所生成的实际上是一个长度为0的空数组，但这个方法上标注了这种方式进行实例化所产生的数组大小为10，那么产生初始化数组是在何时实现的呢？主要是在各个add添加方法和readObject方法时实现的，在add(E e)方法内部，跟踪该方法，发现该方法经过多层调用最后调用了grow(int minCapacity)，这个方法进一步调用了Arrays.copyOf()方法产生一个新数组并实现数据的拷贝。在进行数组扩容时，使用 oldCapacity + (oldCapacity >> 1位移计算让扩充大小增长1.5倍。ArrayList.copyOf()则调用System.arraycopy的native方法进行高效扩容。

```java
private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

&emsp;&emsp;从hugeCapacity方法中可以看出，ArrayList的ArrayList的扩容极限在这个方法中也得以体现-Integer.MAX_VALUE，但ArrayList定义的最大大小为MAX_ARRAY_SIZE=Integer.MAX_VALUE-8，目前解释：①存储Headerwords；②避免一些机器内存溢出，减少出错几率，比如OutOfMemoryError: Java heap space  堆区内存不足（这个可以通过设置JVM参数 -Xmx 来指定）。
OutOfMemoryError: Requested array size exceeds VM limit  超过了JVM虚拟机的最大限制。此外数组作为一个对象，需要一定的内存存储对象头信息，对象头信息最大占用内存不可超过8字节。 ③所以分配最大还是能支持到Integer.MAX_VALUE即最大存储空间范围为整形数据的长度：
```java
private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```
### 序列化
&emsp;&emsp;注意ArrayList中定义的elementData字段前关键字：transient，这个关键字有什么作用呢？， 我们实例化后的ArrayList对象要预留空间，所以这个关键字主要的作用就是为了序列化传输或存储过程中减少存储空间，不让elementData进行序列化，具体的序列化和反序列化操作是调用writeObject和readObject进行实现的，这两个方法就是操作的具体的存储数据的size大小进行处理的，如下：
```java
private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

    /**
     * Reconstitute the <tt>ArrayList</tt> instance from a stream (that is,
     * deserialize it).
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;

        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in capacity
        s.readInt(); // ignored

        if (size > 0) {
            // be like clone(), allocate array based upon size not capacity
            ensureCapacityInternal(size);

            Object[] a = elementData;
            // Read in all elements in the proper order.
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }
```
&emsp;&emsp;既然ArrayList将数据定义为transient，那么其他的集合类是否也是如此定义的呢？简单查看了Java常用的集合类，果然，所有的集合类的数据都不同程度的定义为transient类型，然后实现不同的writeObject和readObject方法。

### 总结
&emsp;&emsp;通过了解JDK源码，让自己有种醍醐灌顶的感觉，喔喔原来这个这个是这么实现的......,感觉还是不错的，再接再厉。



