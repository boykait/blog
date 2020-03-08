---
title: 学MySQL，先懂这些就够了
date: 2020-03-07
categories:
  - MySQL
tags:
  - MySQL
---

> MySQL是当下用得最广泛的一款数据库，活跃的开源社区，易于上手，基本上是现在互联网公司的首选。之前也看了些MySQL相关的知识点，总爱忘，得益于大佬们无私奉献的精彩博文，加以总结形成这篇文章，用到的时候可以当提个醒，后期会再进一步优化提炼。

#### 基础
- 数据库怎么选择字段


- 数据库基本范式

了解数据库的查询流程:    
![数据库查询流程](/images/200301_mysql_learn_03.png)

#### 索引

查看数据的存储目录：
```
show global variables like "%datadir%"
```
看看常用的InnoDB和MyISAM数据库是怎么存储的
![数据库的存储形式](/imagers/200301_mysql_learn_01.png)
其中：
db.opt是存储的创建数据库的字符集：    
```
default-character-set=utf8
default-collation=utf8_general_ci
```  
.frm：描述表结构文件，字段长度
.ibd：InnoDB对应的索引和数据
.MYI：MyISAM存储引擎
.MYD：MyISAM数据文件
疑问：为什么索引和数据，一个分开存，一个合并在一起呢？

索引的基本创建删除操作，如：    
```
// 创建索引
create index index_name on tbl_user(name);     
alter table tbl_user add index `idx_name`(`name`);    
// 创建全文索引
alter table tbl_user add fulltext key index_description (`description`);

// 删除索引
drop index `idx_name` on tbl_user;
// 查询索引
show index from tbl_user;
```



索引类型：
B+树索引、hash索引、全文索引    
那些属性可以做聚簇索引： 
   
- Primary Key；    
- 未指定PK，以属性中第一个非空唯一列为准；    
- 没有指定，也没有符合条件的列属性，那么默认会产生一个隐含字段来建立(长度为6个字节，类型为长整形);    

非聚簇索引
一般情况下需要回表才能查到整条数据，当然如果是覆盖索引就不用回表查询，什么是回表呢？

了解不同存储引擎的索引实现方式对于正确使用和优化索引都非常有帮助，例如知道了InnoDB的索引实现后，就很容易明白为什么不建议使用过长的字段作为主键，因为所有辅助索引都引用主索引，过长的主索引会令辅助索引变得过大。再例如，用非单调的字段作为主键在InnoDB中不是个好主意，因为InnoDB数据文件本身是一颗B+Tree，非单调的主键会造成在插入新记录时数据文件为了维持B+Tree的特性而频繁的分裂调整，十分低效，而使用自增字段作为主键则是一个很好的选择。所以uuid不适合作为数据库表的id，如果是分布式的场景，可以考虑使用Facebook的雪花算法。分库分表如何自增？数据的插入应该是只在主表吗？



###### 复合索引
需要满足左前缀原则，即(A，B，C):
```
- 那么查询的时候，如果查询【A】【A，B】 【A，B，C】，那么可以通过索引查询    
- 如果查询的时候，采用【A，C】，那么C这个虽然是索引，但是由于中间缺失了B，因此C这个索引是用不到的，只能用到A索引    
- 如果查询的时候，采用【B】 【B，C】 【C】，由于没有用到第一列索引，不是最左前缀，那么后面的索引也是用不到了    
- 如果查询的时候，采用范围查询，并且是最左前缀，也就是第一列索引，那么可以用到索引，但是范围后面的列无法用到索引    
```

考考你 比如查询顺序为A、C、B会使用索引吗？    
```
EXPLAIN  SELECT  id, A, B, C  FROM tbl_abc WHERE  A=1 AND C=1 AND B=1; 
```   
结果：    
![调整复合索引的使用顺序](/imagers/200301_mysql_learn_01.png)
聪明的你一定想到了吧，这个是可以的，因为MySQL执行器对查询语句进行分析并优化的结果，后面会详细说。
在使用InnoDB存储引擎时，如果没有特别的需要，请永远使用一个与业务无关的自增(递增)字段作为主键。
那数据在进行删除或者修改的时候，索引的存储页会发生变化吗？

建立索引应该注意到：
- 尽量采用自增（单机）/递增（分布式）来生成主键，避免使用uuid；
- 使用短索引；
- 尽量选择区分度较好的字段来建立索引，比如身份证号、姓名，类似性别、班级就不太合适；
- 能用覆盖索引吗？考虑场景

###### 避免索引失效
- 不要模糊匹配以%开头；
- 复合索引药满足最左匹配；
- 尽量避免使用 != 或 not in或 <> 等否定操作符；
- 不要在列上进行运算；
 - 如： 假如birthday是索引字段：select * from user where YEAR(birthday)<1990会导致索引失效。
- 范围查询对多列查询的影响；
 - 如果查询中的某个列有范围查询，则其右边所有列都无法使用索引优化查找。举个例子，你有个查询。
- 索引不会包含有NULL值的列。
 - 只要列中包含有NULL值都将不会被包含在索引中，复合索引中只要有一列含有NULL值，那么这一列对于此复合索引就是无效的。所以我们在数据库设计时不要让字段的默认值为NULL。貌似是因为索引中不会存储null，所以无法使用索引，至于加函数和运算是因为优化器没办法很好的处理，所以这部分也是不能使用索引的。反驳声音：[MySQL中IS NULL、IS NOT NULL、!=不能用索引？胡扯！](https://juejin.im/post/5d5defc2518825591523a1db)
- 可以适当考虑多列索引
	- 一个常见的错误。为多个列创建独立的索引，大部分并不能提高MySQL的查询性能。因为执行查询时，MySQL只能使用一个索引，会从多个索引中选择一个限制最为严格的索引。
- 尽量避免使用 or 来连接条件
 - 应该尽量避免在 where 子句中使用 or 来连接条件，因为这会导致索引失效而进行全表扫描。

###### 慢查询调优
- explain看查询计划：了解各个字段含有及取值
```
id : 表示SQL执行的顺序的标识,SQL从大到小的执行
select_type：表示查询中每个select子句的类型
table：显示这一行的数据是关于哪张表的，有时不是真实的表名字
type：表示MySQL在表中找到所需行的方式，又称“访问类型”。常用的类型有： ALL, index,  range, ref, eq_ref, const, system, NULL（从左到右，性能从差到好）
possible_keys：指出MySQL能使用哪个索引在表中找到记录，查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询使用
Key：key列显示MySQL实际决定使用的键（索引），如果没有选择索引，键是NULL。
key_len：表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度（key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的）
ref：表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值
rows： 表示MySQL根据表统计信息及索引选用情况，估算的找到所需的记录所需要读取的行数，理论上行数越少，查询性能越好
Extra：该列包含MySQL解决查询的详细信息
```
- 注意使用order by limit的场景优化
- 可以结合类似MyBatis框架做关联延时加载

注意：索引并不是适用于任何情况。对于中型、大型表适用。对于小型表全表扫描更高效。而对于特大型表，考虑”分区”技术。

###### 强制索引
有时，因为使用 MySQL 的优化器机制，原本应该使用索引的优化器，反而选择执行全表扫描或者执行的不是预期的索引。此时，可以通过强制索引的方式引导优化器采取正确的执行计划。
使用强制索引，SQL 语句只使用建立在 index_col_name 上的索引，而不使用其它的索引。
select * from tbl_name force index (index_col_name)，不要滥用强制索引，因为 MySQL 的优化器会同时评估 I/O 和 CPU 的成本，一般情况下，可以自动分析选择最合适的索引。
如果优化器成本评估错误，因而没有选择最佳方案，最好的方法应该是将合适的索引修改得更好。
如果某个 SQL 语句使用强制索引，需要在系统迭代开发过程中时时维护强制索引，一方面，需要保证使用的强制索引最优，另外一面，需要保证所使用的强制索引不能被误删，不然将导致 SQL 报错。
因此，如果某个 SQL 语句必须要使用强制索引，建议在团队内部开展严格地评审后才可以使用。


小结问答：    

- MySQL为什么使用B+而不是二叉、平衡、B、跳跃表等其它数据结构（经典问题）？    
 - [MySQL索引为什么要用B+树实现](https://juejin.im/entry/5bc1ea0a5188255c2f424209)    
- 聚簇和非聚簇分别存储的什么数据？    
InnoDB的数据文件本身就是索引文件，B+Tree的叶子节点上的data就是数据本身，key为主键，非叶子节点存放<key,address>，address就是下一层的地址
非聚簇索引，叶子节点上的data是主键(即聚簇索引的主键，所以聚簇索引的key，不能过长)。为什么存放的主键，而不是记录所在地址呢，理由相当简单，因为记录所在地址并不能保证一定不会变，但主键可以保证.
- B+树高多少？一般可以存储多少数据？
- 大数据量的limit分页如何优化？
- 索引的溢出数据是如何处理的？

##### 索引执行计划
[理解索引：MySQL执行计划详细介绍](https://juejin.im/post/5b1243eff265da6e0b6ff277)

MyISAM是使用的非聚簇索引

#### 存储引擎

#### 锁和事务

事务四大特性： 原子性、一致性、隔离性、持久性，ACID

四种隔离级别：
假如A、B两个事务
读未提交（read uncommitted）：A可能读到B还没有提交的修改结果，然后后面B又修改了，md，感觉问题大哟，读取了脏数据；  
读已提交（read committed）：A读取了B已经提交的数据，但是存在的问题是A对同一条数据的读就可能存在不同的结果了，所以该模式的问题是不可重复读；    
可重复读（repeatable read）：MySQL默认的隔离级别，但在定义上可重复读级别是会存在幻读问题，MySQL通过next-keys解决该问题。
序列化（serializable）：没啥问题，就是慢，如果事务B一旦开始了，没有提交或者回滚之前，事务A是不能写入数据的，即使B只是做查询操作。

如果想看操作例子，[去这](https://www.jianshu.com/p/4e3edbedb9a8)，记录得不错，我这就不重复工作了。

接下来要好好理理这个隔离级别中使用的技术，
###### MVCC
- 版本链：[详细](https://juejin.im/post/5c9b1b7df265da60e21c0b57#heading-6)
对于使用InnoDB存储引擎的表来说，它的聚簇索引记录中都包含两个必要的隐藏列（row_id并不是必要的，我们创建的表中有主键或者非NULL唯一键时都不会包含row_id列）：    
  - trx_id： 每次对某条聚簇索引记录进行改动时，都会把对应的事务id赋值给trx_id隐藏列。     
  - roll_pointer：每次对某条聚簇索引记录进行改动时，都会把旧的版本写入到undo日志中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息。疑问，会存多少个版本呢？    
- ReadView：[详细](https://juejin.im/post/5c9b1b7df265da60e21c0b57#heading-7)

###### RR模式之next-keys

#### 锁
搞清楚这几种锁吧：[对mysql乐观锁、悲观锁、共享锁、排它锁、行锁、表锁概念的理解](https://juejin.im/entry/5b5abebb6fb9a04f8b786b39)
[惊！史上最全的select加锁分析](https://juejin.im/post/5d5671a2e51d45620821cea7)

一致性非锁定读和一致性锁定读
一致性的非锁定读是指InnoDB存储引擎通过行MVCC的方式来读取当前执行时间数据库中行的数据。 如果读取的行的时候有正在执行的 Delete 或者 Update 操作，这时读取操作不会等待行上锁的释放，而是InnoDB引擎会去读取行的一个快照数据。    
**可重复读级别**：是InnoDB默认的事务隔离级别，对于快照数据，非一致性读总是读取事务开始时的行数据版本。    
**读已提交级别**： 事务隔离级别下，对于快照数据，非一致性读总是读取被锁定行的最新一份快照数据。
默认情况下，InnoDB是一致性非锁定读，如果有些业务场景需要显式的对数据库读取操作进行加锁以保证数据逻辑的一致性。这就需要进行加锁了，加锁方式上面描述共享锁和排他锁的时候已经提到过，这里不再重复。
```
select ... for update 和 select ... lock in share mode
```
加锁的前提是必须在一个事务中，所以开始一个事务，然后进行加共享锁，如果未进行commit， SessionB执行update操作则会等待，等待的时候默认是50s，可以查看相关mysql配置，如果再超时之前，SessionA执行了commit操作，则SessionB会马上执行成功。

###### 锁算法
前面事务部分讲到，可重复读级别是通过next-keys来解决的，它是包括行记录锁（RecordLock）和间隙锁（Gap Lock）结合而成的，其中：    
RecordLock: 表示单个行记录上的锁，会去锁定索引记录，如果InnoDB存储引擎表在建立的时候没有设置任何一个索引，则InnoDB会去使用隐式的主键来锁定。    
Gap Lock: 间隙锁，锁定一个范围，但不包含记录本身。
[锁算法](https://juejin.im/post/5c8bd3ebf265da2db2797d92#heading-8)

###### 数据库死锁
[死锁](https://juejin.im/post/5c8bd3ebf265da2db2797d92#heading-14)
[定位死锁问题](https://juejin.im/entry/57e7685abf22ec00586ed574)

#### 优化


#### 集群

#### 分库分表

#### 其它
MySQL查询执行优化器

MySQL的连接和关闭：mysql -u -p -h -P
```
-u：指定用户名-p：指定密码-h：主机-P：端口
```

innodb引擎的特性
```
插入缓冲（insert buffer)
二次写(double write)
自适应哈希索引(ahi)
预读(read ahead)
```

通过对MySQL前面基础知识的学习，再结合大佬们总结的面试题，MySQL这关可以过了        
[互联网公司面试必问的mysql题目(上)](https://juejin.im/post/5baafdccf265da0af93b05e4)       
[互联网公司面试必问的mysql题目(下）](https://juejin.im/post/5ba1f32ee51d450e805b43f2)    
[企业面试题｜最常问的MySQL面试题集合（一）](https://juejin.im/entry/5b57ec015188251aa8292a69)    
[企业面试题｜最常问的MySQL面试题集合（二）](https://juejin.im/entry/5b57ebdcf265da0f61320e6f)    
[企业面试题｜最常问的MySQL面试题集合（三）](https://juejin.im/entry/5b59cdf8f265da0f7e6295ba)    
[[灵魂拷问]MySQL面试高频一百问(工程师方向)](https://juejin.im/post/5d351303f265da1bd30596f9)    
[史上最详细的一线大厂Mysql面试题详解](https://juejin.im/post/5cb6c4ef51882532b70e6ff0)    

更多扩展    
[Mysql加锁与实践](https://juejin.im/post/5bc8460ce51d450e4714675d)
[[译]MySQL InnoDB 锁——官方文档](https://juejin.im/entry/5abdb0da6fb9a028c06aeb3c)    
[MySQL 在高并发下的 订单撮合 系统使用 共享锁 与 排他锁 保证数据一致性](https://juejin.im/post/5bf7d142518825369c5640de)    
[MySQL太慢？试试这些诊断思路和工具](https://juejin.im/entry/5bac3addf265da0af93b08ae)    
[记住，永远不要在MySQL中使用“utf8”](https://juejin.im/entry/5b3055046fb9a00e315c2849)    
[万字总结：学习MySQL优化原理](https://juejin.im/entry/59ccaabf6fb9a00a5143aff6)    
[『MySQL』搞懂 InnoDB 锁机制 以及 高并发下如何解决超卖问题](https://juejin.im/post/5c8bd3ebf265da2db2797d92#heading-14)    
