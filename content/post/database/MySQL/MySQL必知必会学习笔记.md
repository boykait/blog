---
title: MySQL必知必会学习笔记
date: 2019-09-06
categories:
  - 数据库
  - MySQL
tags:
  - 数据库
  - MySQL  
---

### 第3章 使用MySQL
1. 数据库实例相关操作：
```
mysql> CREATE DATABASE crashcourse; // 创建数据库
mysql> use crashcourse;  // 切换数据库
Database changed
mysql> SHOW DATABASES;   //查看服务器所有数据库实例
mysql> SHOW TABLES;    // 查询当前所使用的数据库实例的所有表
mysql> SHOW COLUMNS FROM users;  // 查询表的数据结构定义 这个结构定义存储在什么地方？ SHOW COLUMNS FROM = DESCRIBE
``` 

### 第4章 检索数据

- SELECT id, name FROM users; 

```
DISTINCT: 可使用排除重复的列数据
LIMIT:  LIMIT start, number 限制查询数据 或者： LIMIT start OFFSET number
使用完全限定的表名的使用：区别查询字段是隶属于哪一张表，避免表关联时候存在同样名称无法分辨。
```

### 第5章 排序检索数据

&emsp;&emsp;使用ORDER BY:
 SELECT id FROM users ORDER BY id, name;

```
先按第一个字段排序，如果第一个字段值相同，则需要更具第二个字段进行排序， 默认是升序排序（ASC）
 可指定排序顺序 ASC 或者 DESC;
```

### 第6章 过滤数据

&emsp;&emsp;使用WHERE过滤：
SELECT id FROM users WHERE name='xiaoming';

```
<> 和 !=
BETWEEN AND 
IS NULL 和 IS NOT NULL
```

### 第7章 数据过滤

&emsp;&emsp;更为强大的过滤方法，利用WHERE和其他过滤进行组合：
SELECT id, name FROM users WHERE 

```
AND: 组合多个过滤条件 id = 1 AND name='xiaoming'
OR：OR优先级低于AND
IN: id IN (1, 2); 当操作的数据量比较大时用IN是否有问题（IN的缺点）？
```

### 第8章 用通配符进行过滤

&emsp;&emsp;LIKE：

```
%：匹配多个，WHERE name LIKE '%xiaomin%';
_：匹配单个，WHERE name LIKE '_xiaoming';
```

注意：使用通配符，在时间开销上可能较大，所以要慎用，如果要使用，尽量将其放到靠后的位置，先匹配精准的条件

### 第9章 用正则表达式进行搜索
&emsp;&emsp;参考正则表达式必知必会

### 第10章 创建计算字段
&emsp;&emsp;GROUP_CONCAT(CONCAT(str1, str2))分组拼接 

### 第11章 使用数据处理函数

### 第12章 汇总数据
&emsp;&emsp;AVG、COUNT、MAX、MIN、SUM

### 第13章 分组数据
&emsp;&emsp; GROUP BY 分组属性 [HAVING 过滤条件]
&emsp;&emsp; 思考：复杂的SQL语句操作的顺序很重要喔！怎么优化？
