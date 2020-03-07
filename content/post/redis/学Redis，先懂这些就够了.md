---
title: 学Redis，先懂这些就够了
date: 2020-02-29
categories:
  - Redis
tags:
  - Redis
---
> 前段时间为了学习Redis，专门花了60多大洋买了老钱的《Redis深度历险》，我不是打广告的哈，写得还是不错的，书友们反映都很不错，我不算是一个非常勤奋的人，也没什么天分，知识这东西，不继续学，几个月时间就只剩点印象了，所以就觉得吧，好记性不如烂笔头（当然现在不用笔啦），本文并不打算深入到源码级别，只是对常见的知识点进行梳理，方便以后进一步学习和巩固。加油，go！

目录

- 快速了解Redis
- Redis数据结构
- Redis协议
- 持久化
- 过期策略
- 内存淘汰策略
- 事务
- Redis雪崩、击穿、穿透
- 分布式锁
  - 续期
- 缓存一致性问题
- 集群和高可用

### Redis基础
#### 快速了解Redis
Redis（REmote DIctionary Server，远程字典服务），在面试和项目中也经常和memecahe做对比，但由于其丰富的特性和数据结构，使得它成为现存使用最广泛的基于内存的key-value存储系统。
##### 协议
Redis所使用的协议是RESP（Redis Serialization Protocol）,
##### 快不是没道理
我们都知道Redis宣称的是单线程，but它能有上几万的QPS，大型的互联网应用中能够有效支撑上百万的QPS，    
- 基于内存    
- 采用单线程，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗 CPU    
- IO/多路复用

除了这些我认为，惰性删除和异步持久化也是有效支撑其高效的两个重要特性，后面我会对两者做一个总结。
#### Redis数据结构
Redis除了支持常见的String、List、Set、Sorted Set、Hash等五种数据结构，还支持Bitmap+GEO+HyperLogLog，5.0版本支持Streams，先看一下这几种数据结构的概览：

![数据结构图]()

接下来，了解一下这几种数据结构的实现原理和应用场景
##### 基本数据结构

###### String
String字符串类型根据输入数据有三种形态：
```
127.0.0.1:6379> debug object str
Value at:0x7f49e2e0e140 refcount:1 encoding:int serializedlength:5 lru:5972158 lru_seconds_idle:3
127.0.0.1:6379> set str 'qweqweqweqweqweqweqweqweqweqwe'
OK
127.0.0.1:6379> debug object str
Value at:0x7f49e2e2b380 refcount:1 encoding:embstr serializedlength:14 lru:5972181 lru_seconds_idle:2
127.0.0.1:6379> set str 'qweqweqweqweqweqweqweqweqweqwedsdfffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff'
OK
127.0.0.1:6379> debug object str
Value at:0x7f49e2eb95d0 refcount:1 encoding:raw serializedlength:23 lru:5972196 lru_seconds_idle:3
```
String在Redis中是用一种SDS(Simple Dynamic String)，也是目前我在项目中使用得最多的一种数据结构，就SDS本身来说，可以理解成Java里面的ArrayList动态数组，但更为强悍，能够根据数据量的不同适配不同的编码格式，String类型具体有两种存储方式emb和row，当字符数长度超过44时，则会由emb转换为row方式：

![String数据存储格式](/images/200301_redis_learn_04.png)

![](https://i.imgur.com/HMFqcYY.png)

为什么emb只能存储44个字节数据呢？Redis设计者认为，字符串对象总体大于64个字节就是大字符串，对象头RedisObject的定义：

```
struct RedisObject { 
    int4 type; // 4bits  类型
    int4 encoding; // 4bits 存储格式
    int24 lru; // 24bits 记录LRU信息
    int32 refcount; // 4bytes 
    void *ptr; // 8bytes，64-bit system 
} robj;
```
String类型的过程也是采用了阶段性的扩容机制，内部定义了SDS_MAX_PREALLOC为1MB，当字符串在这个范围内，就会按倍数扩张，即2^n，当超过这个长度，为了避免资源浪费，每次额外扩张1MB的存储空间，String类型的最大长度不超过512MB。    
惰性释放空间

##### List
List有点类似于Java中的LinkedList，但同样，其实现也没有这么简单，是一种叫快速列表的结构(quckList)，当数据量比较少时，会使用连续的存储空间-压缩列表ziplist

##### Hash

##### Set

##### Sorted Set
主要用途是可以实现排行榜，底层原理是使用跳跃表
![](/images/200301_redis_learn_07.png)

跳跃表实现比较简单，判断当前元素是否需要晋升的是使用的类投硬币的随机方式，本身是概率性平衡、而非绝对平衡，我们可以理解最最底层的还是单向链表，但是链表的每个节点的next可以指向多个next后继节点，但同时也有多个前驱节点。[细看好文](https://blog.csdn.net/lz710117239/article/details/78408919)

那么问题来了，为什么用跳表而不用平衡树？
明显嘛，我跳跃表可以查询出比如前10名的排名，单链表遍历Level0层的forword下一节点，反观平衡二叉树就不是很方便了，即使是B+树，存储结构复杂，且查询也比较复杂。
[对比文](https://www.cnblogs.com/stoneFang/p/10714769.html)
这里从内存占用、对范围查找的支持和实现难易程度这三方面总结的原因，我们在前面其实也都涉及到了,
[](https://juejin.im/post/57fa935b0e3dd90057c50fbc#heading-8)

那问题又来了，为什么MySQL的索引结构，采用了B+树，没有使用跳跃表呢？可以抵挡一下，总比问了什么都不懂好。
[](https://segmentfault.com/q/1010000018633282)
[](https://www.cnblogs.com/aspirant/p/11704530.html)



### Redis企业应用

#### 持久化

如何恢复集群
如何将一个集群的数据转移到另外一个集群？

#### 缓存过期处理    
 定期删除+惰性删除
所谓定期删除，指的是redis默认是每隔100ms就随机抽取一些设置了过期时间的key，检查其是否过期，如果过期就删除。redis是每隔100ms随机抽取一些key来检查和删除的。发现过期的key大于1/4，则继续删除，否则结束，定期删除可能会导致很多过期key到了时间并没有被删除掉，所以就得靠惰性删除了。这就是说，在你获取某个key的时候，redis会检查一下 ，这个key如果设置了过期时间那么是否过期了？如果过期了此时就会删除，不会给你返回任何东西。并不是key到时间就被删除掉，而是你查询这个key的时候，redis再懒惰的检查一下，　通过上述两种手段结合起来，保证过期的key一定会被干掉。但是实际上这还是有问题的，如果定期删除漏掉了很多过期key，然后你也没及时去查，也就没走惰性删除，此时会怎么样？如果大量过期key堆积在内存里，导致redis内存块耗尽了，怎么办？
答案是：内存淘汰机制

#### 内存淘汰策略
六种：    
noeviction：当内存不足以容纳新写入数据时，新写入操作会报错，这个一般没人用    
allkeys-lru: 尝试回收最少使用的键（LRU），使得新添加的数据有空间存放。这个是最常用的    
volatile-lru: 尝试回收最少使用的键（LRU），但仅限于在过期集合的键,使得新添加的数据有空间存放。 这个一般没人用吧   
allkeys-random: 回收随机的键使得新添加的数据有空间存放。    
volatile-random: 回收随机的键使得新添加的数据有空间存放，但仅限于在过期集合的键。    
volatile-ttl: 回收在过期集合的键，并且优先回收存活时间（TTL）较短的键,使得新添加的数据有空间存放      
LRU方式的大致思想是选取一批要进行淘汰的数据构成淘汰池，然后按照url淘汰一部分，下一次淘汰时，还是上次淘汰剩余的对象再结合一些新的对象，这样可以使得淘汰率更高，
新版本的LFU淘汰策略了解一下

场景问题：
MySQL里有2000w数据，redis中只存20w的数据，如何保证redis中的数据都是热点数据？


#### Redis雪崩、击穿、穿透
##### 缓存雪崩
缓存雪崩的英文解释就是奔逃的野牛，就是当我们应用系统由于缓存服务器挂掉、或者设置的key大面积在同一时刻失效，此时大量的请求将会直接请求数据库DB服务器或第三方服务，DB服务器表示压力山大啊。
那么有什么解决办法呢？
- 错开key的过期时间，最常用的解决办法就是不同的key设置不同的过期时间；
- 使用锁来解决，单机环境可以使用应用语言比如Java提供的Syncronized或者Lock锁，分布式环境可以使用Redis的setnx命令

##### 缓存击穿
缓存击穿就是热点key失效后，大量针对该key的请求就直接落在了服务器上，
目前使用的项目中遇到过有点类似的场景（当然因为具体的业务限制，也还真没有什么大并发量的请求问题），因为我们项目从前端请求到执行结束有多个服务的串行调用，在测试同学批量删除100条资源的时候，发现当前用户的token失效，失效后就去请求token服务器获取最新token数据，然后因为设计缺陷导致每个请求都去判断缓存中是否存在token，没有就都去请求token服务器，类似：

```
if redisCache.getKey(token) is null:
  //
  lock
     getToken()
     redisCache.setKey(token)
     doSomeThings()
  unlock
else:
  doSomeThings()
```
所以缓存击穿的处理方式就是使用锁，当大量针对同一个key的请求到来是，对第一个拿到lock或分布式锁的请求执行token请求操作，此时其他请求可以自旋等待，当第一个请求得到token并将其设置到缓存中，释放lock锁，然后其余的请求再一个尝试从缓存中获取即可

##### 缓存穿透
缓存穿透就是key对应的数据在数据源并不存在，每次针对此key的请求从缓存获取不到，请求都会到数据源，从而可能压垮数据源。比如用一个不存在的用户id获取用户信息，不论缓存还是数据库都没有，若黑客利用此漏洞进行攻击可能压垮数据库，这也是高可用服务需要考虑到的一个安全性问题。

###### 解决方案
- 缓存空值之所以会发生穿透，就是因为缓存中没有存储这些空数据的key。从而导致每次查询都到数据库去了。那么我们就可以为这些key对应的值设置为null 丢到缓存里面去。后面再出现查询这个key 的请求的时候，直接返回null 。这样，就不用在到数据库中去走一圈了，但是别忘了设置过期时间。但是这种的话可能会导致缓存大量值为null的key；
- BloomFilterBloomFilter 类似于一个hbase set 用来判断某个元素（key）是否存在于某个集合中。这种方式在大数据场景应用比较多，比如 Hbase 中使用它去判断数据是否在磁盘上。还有在爬虫场景判断url 是否已经被爬取过。这种方案可以加在第一种方案中，在缓存之前在加一层 BloomFilter ，在查询的时候先去 BloomFilter 去查询 key 是否存在，如果不存在就直接返回，存在再走查缓存 -> 查 DB。
原理就是一个对一个key进行k个hash算法获取k个值，在比特数组中将这k个值散列后设定为1，然后查的时候如果特定的这几个位置都为1，那么布隆过滤器判断该key存在。
布隆过滤器可能会误判，如果它说不存在那肯定不存在，如果它说存在，那数据有可能实际不存在；
Redis的bitmap只支持2^32大小，对应到内存也就是512MB，误判率万分之一，可以放下2亿左右的数据，性能高，空间占用率及小，省去了大量无效的数据库连接。
BloomFilter还有一个重要的应用场景就是新闻客户端的推荐去重

#### 分布式锁

跟着学习，敲了篇：

```java
    /*************************
     * 测试分布式锁
     * Redis的分布式锁目标是为了让锁住的代码块串行执行
     */

    @Autowired
    private StringRedisTemplate redisTemplate;
    private static final String LOCK_KEY = "lockKey";
    private static final String LOCK_VALUE = "lockValue";
    @Autowired
    private Redisson redisson;

    @RequestMapping("/redis-lock")
    @ResponseBody
    public String redisLock() {
        try {
            // idea 1
            // step1. 分布式锁代替本地lock或syncronized
            //
            Boolean lock = redisTemplate.opsForValue().setIfAbsent(LOCK_KEY, LOCK_VALUE);
            // step2. 为了保证程序宕机时能够整个释放锁，需要设置分布式锁的过期时间
            redisTemplate.expire(LOCK_KEY, 30, TimeUnit.SECONDS);

            // idea 2 代替上面两句，原子执行 避免在设置超时时间之前程序崩溃导致无法释放
            redisTemplate.opsForValue().setIfAbsent(LOCK_KEY, LOCK_VALUE, 30, TimeUnit.SECONDS);
            if (!lock) {
                return "error";
            }
            redisTemplate.opsForValue().set("stock", "value");
            System.out.println("设置成功");
            return "成功";
        } finally {
            redisTemplate.delete(LOCK_KEY);
        }

        // 高并发场景下，请求A超时释放锁，B拿到锁，在此期间A执行结束会finally释放请求B持有的锁
        // 解决方法，设置每个请求lockKey对应的vaule唯一
        String requstLockValue = UUID.randomUUID().toString();
        // 设置redisTemplate.opsForValue().setIfAbsent(LOCK_KEY, requstLockValue , 30, TimeUnit.SECONDS);
        // 释放判断
        if (requstLockValue.equals(redisTemplate.opsForValue().get(LOCK_KEY))) {
            redisTemplate.delete(LOCK_KEY);
        }

        // 上面的设置的固定时间不是很ok，怎么办？续命 1. 能不能设置一个较大值呢？ 2. 可以开启子线程用Timer续命延期，有更高级的方法
        // Redisson
        RLock redisLock = redisson.getLock(LOCK_KEY);
        // 上锁
        redisLock.lock(30, TimeUnit.SECONDS);

        // 解锁
        redisLock.unlock();

        // 主从节点的lock同步问题，即master节点同步数据到对应的slave节点前挂机了，slave升级为master，此时又有可能出现
        // 超卖代码块

        // 高并发分布式锁

    }
```

主从未及时同步问题解决方案：
采用master-slave模式，加锁的时候只对一个节点加锁，即便通过sentinel做了高可用，但是如果master节点故障了，发生主从切换，此时就会有可能出现锁丢失的问题。

目前我们的项目里面没考虑过这个问题，如果需要处理的话，怎么弄呢？
基于以上的考虑，其实redis的作者也考虑到这个问题，他提出了一个RedLock的算法，这个算法的意思大概是这样的：
假设redis的部署模式是redis cluster，总共有5个master节点，通过以下步骤获取一把锁：
获取当前时间戳，单位是毫秒
轮流尝试在每个master节点上创建锁，过期时间设置较短，一般就几十毫秒
尝试在大多数节点上建立一个锁，比如5个节点就要求是3个节点（n / 2 +1）
客户端计算建立好锁的时间，如果建立锁的时间小于超时时间，就算建立成功了
要是锁建立失败了，那么就依次删除这个锁
只要别人建立了一把分布式锁，你就得不断轮询去尝试获取锁
但是这样的这种算法还是颇具争议的，可能还会存在不少的问题，无法保证加锁的过程一定正确

```
RedissonClient redisson = Redisson.create(config);
RLock lock1 = redisson.getFairLock("lock1");
RLock lock2 = redisson.getFairLock("lock2");
RLock lock3 = redisson.getFairLock("lock3");
RedissonRedLock multiLock = new RedissonRedLock(lock1, lock2, lock3);
multiLock.lock();
multiLock.unlock();
```

[分布式锁文章](https://blog.csdn.net/eygle/article/details/96431966)

#### 缓存一致性问题

##### 最终一致性
事务更新
一般在我们的更新业务逻辑需要保证事务性，在Spring或SpringBoot应用系统中使用@Transactional来保证，但该注解只能保证数据库的事务特性，无法保证缓存是否成功，比如写现在数据库成功，但更新缓存失败，此时如何处理呢：

```java
TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
```
带缓存的读操作流程：



带缓存的写流程，一般有如下几种处理手段：
1. 先更新缓存，再更新数据库；
2. 先更新数据库，再更新缓存；
3. 先删除缓存，在更新数据库；
4. 先更新数据库，再删除缓存。

Facebook推出的广受认可的方案是第4中即Cache Aside Pattern，首先说为什么缓存是删除而不是更新呢？
主要是可能产生脏数据，比如现在A、B两个请求更新同一条数据，依次更新数据库，更新成功后准备更新缓存，由于网络延时或其它源于，请求B先于请求A将数据写入到缓存中，如此一来，缓存中最终缓存的是A请求的更新数据，故由此导致了脏数据的产生

##### 强一致性

##### 高并发场景

##### 扩展-秒杀场景中如何处理
超卖问题


#### 事务（最后研究，没太多应用场景）

#### 限流

#### 主从、哨兵、 集群
##### 集群
Redis Cluster是Redis3.0版本后提出的分布式解决方案，主要有如下特点：
- 自动将数据分配，每个master节点上存放一部分数据，
- 内置高可用（切换策略）
基本安装参考[Redis5.0集群版搭建](https://www.jianshu.com/p/5a46fff22baf)搭建了个Redis5.0.5版本集群:

![Redis集群搭建结果](/images/200301_redis_learn_05.png)
通过任意一个节点查看集群状态：
```
[root@localhost redis-cluster]# ./redis-cli -h 127.0.0.1 -p 7001 -c cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:1108
cluster_stats_messages_pong_sent:1163
cluster_stats_messages_sent:2271
cluster_stats_messages_ping_received:1158
cluster_stats_messages_pong_received:1108
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:2271

```
查看集群节点信息：
```
[root@localhost redis-cluster]# ./redis-cli -h 127.0.0.1 -p 7001 -c cluster nodes
4b2d956ad2d12162b3605dfe7fea0d84fcb3f7f2 127.0.0.1:7005@17005 slave 50151ad44620c5b5823340f159353ea84e3c31d7 0 1583070154000 5 connected
0ac26bab5c539d5966f31280784f050cec843413 127.0.0.1:7001@17001 myself,master - 0 1583070155000 1 connected 0-5460
50151ad44620c5b5823340f159353ea84e3c31d7 127.0.0.1:7003@17003 master - 0 1583070156518 3 connected 10923-16383
c8eafaa22a17568f1962060b2398a388b534eb9c 127.0.0.1:7002@17002 master - 0 1583070155000 2 connected 5461-10922
3ce0748fd3b2910870b4aa54df7328f82088b325 127.0.0.1:7004@17004 slave c8eafaa22a17568f1962060b2398a388b534eb9c 0 1583070157529 4 connected
70e3916002a195aad5911c76233b052189748207 127.0.0.1:7006@17006 slave 0ac26bab5c539d5966f31280784f050cec843413 0 1583070155511 6 connected

```

在单机环境上模拟配置的6个节点，在集群中不同的节点访问同一个key，都能够准确定位到key所在的真实节点：
```
[root@localhost redis-cluster]# ./redis-cli -h 127.0.0.1 -p 7001 -c
127.0.0.1:7001> set name boyka
-> Redirected to slot [5798] located at 127.0.0.1:7002
OK
127.0.0.1:7002> get name
"boyka"
127.0.0.1:7002> 
[root@localhost redis-cluster]# ./redis-cli -h 127.0.0.1 -p 7003 -c
127.0.0.1:7003> get name
-> Redirected to slot [5798] located at 127.0.0.1:7002
"boyka"
127.0.0.1:7002> get name 
[root@localhost redis-cluster]# ./redis-cli -h 127.0.0.1 -p 7004 -c
127.0.0.1:7004> get name
-> Redirected to slot [5798] located at 127.0.0.1:7002
"boyka"
127.0.0.1:7002>
```
Redis Cluster采用：


Redis Cluster的每个Master节点上可以挂在多个Slave节点来实现高可用

![](/images/200301_redis_learn_06.png)
Redis Cluster采用虚拟Hash槽分区所有的键根据CRC循环冗余校验算法函数映射到0~16383个槽内，slot=CRC(key)&16383，每一个节点负责维护一部分槽以及槽所映射的键值数据。

CRC是怎么回事？

```
/* --------------cluster.c---------------------------------------------------------------
 * Key space handling
 * -------------------------------------------------------------------------- */

/* We have 16384 hash slots. The hash slot of a given key is obtained
 * as the least significant 14 bits of the crc16 of the key.
 *
 * However if the key contains the {...} pattern, only the part between
 * { and } is hashed. This may be useful in the future to force certain
 * keys to be in the same node (assuming no resharding is in progress). */
 //我们有16384个哈希槽，将会用crc16得到的最小有效14位作为给定key的哈希槽。
 //但是，如果键包含‘{...}’这样的字段，只有在"{“和“}”之间的才会被哈希。这在将来可能有助于将某些键强制放在同一个节点中(假设没有进行重新分片
 
// 计算给定键应该被分配到哪个槽
unsigned int keyHashSlot(char *key, int keylen) {
    int s, e; /* start-end indexes of { and } */

    for (s = 0; s < keylen; s++)
        if (key[s] == '{') break;

    /* No '{' ? Hash the whole key. This is the base case. */
    //0x3FFF转化成十进制就是16383
    if (s == keylen) return crc16(key,keylen) & 0x3FFF;

    /* '{' found? Check if we have the corresponding '}'. */
    for (e = s+1; e < keylen; e++)
        if (key[e] == '}') break;

    /* No '}' or nothing betweeen {} ? Hash the whole key. */
    if (e == keylen || e == s+1) return crc16(key,keylen) & 0x3FFF;

    /* If we are here there is both a { and a } on its right. Hash
     * what is in the middle between { and }. */
    return crc16(key+s+1,e-s-1) & 0x3FFF;
```

```
//crc16tab数组，便于进行“查表法”计算crc结果，提高效率
static const uint16_t crc16tab[256]= {
    0x0000,0x1021,0x2042,0x3063,0x4084,0x50a5,0x60c6,0x70e7,
    0x8108,0x9129,0xa14a,0xb16b,0xc18c,0xd1ad,0xe1ce,0xf1ef,
    0x1231,0x0210,0x3273,0x2252,0x52b5,0x4294,0x72f7,0x62d6,
    0x9339,0x8318,0xb37b,0xa35a,0xd3bd,0xc39c,0xf3ff,0xe3de,
    0x2462,0x3443,0x0420,0x1401,0x64e6,0x74c7,0x44a4,0x5485,
    0xa56a,0xb54b,0x8528,0x9509,0xe5ee,0xf5cf,0xc5ac,0xd58d,
    0x3653,0x2672,0x1611,0x0630,0x76d7,0x66f6,0x5695,0x46b4,
    0xb75b,0xa77a,0x9719,0x8738,0xf7df,0xe7fe,0xd79d,0xc7bc,
    0x48c4,0x58e5,0x6886,0x78a7,0x0840,0x1861,0x2802,0x3823,
    0xc9cc,0xd9ed,0xe98e,0xf9af,0x8948,0x9969,0xa90a,0xb92b,
    0x5af5,0x4ad4,0x7ab7,0x6a96,0x1a71,0x0a50,0x3a33,0x2a12,
    0xdbfd,0xcbdc,0xfbbf,0xeb9e,0x9b79,0x8b58,0xbb3b,0xab1a,
    0x6ca6,0x7c87,0x4ce4,0x5cc5,0x2c22,0x3c03,0x0c60,0x1c41,
    0xedae,0xfd8f,0xcdec,0xddcd,0xad2a,0xbd0b,0x8d68,0x9d49,
    0x7e97,0x6eb6,0x5ed5,0x4ef4,0x3e13,0x2e32,0x1e51,0x0e70,
    0xff9f,0xefbe,0xdfdd,0xcffc,0xbf1b,0xaf3a,0x9f59,0x8f78,
    0x9188,0x81a9,0xb1ca,0xa1eb,0xd10c,0xc12d,0xf14e,0xe16f,
    0x1080,0x00a1,0x30c2,0x20e3,0x5004,0x4025,0x7046,0x6067,
    0x83b9,0x9398,0xa3fb,0xb3da,0xc33d,0xd31c,0xe37f,0xf35e,
    0x02b1,0x1290,0x22f3,0x32d2,0x4235,0x5214,0x6277,0x7256,
    0xb5ea,0xa5cb,0x95a8,0x8589,0xf56e,0xe54f,0xd52c,0xc50d,
    0x34e2,0x24c3,0x14a0,0x0481,0x7466,0x6447,0x5424,0x4405,
    0xa7db,0xb7fa,0x8799,0x97b8,0xe75f,0xf77e,0xc71d,0xd73c,
    0x26d3,0x36f2,0x0691,0x16b0,0x6657,0x7676,0x4615,0x5634,
    0xd94c,0xc96d,0xf90e,0xe92f,0x99c8,0x89e9,0xb98a,0xa9ab,
    0x5844,0x4865,0x7806,0x6827,0x18c0,0x08e1,0x3882,0x28a3,
    0xcb7d,0xdb5c,0xeb3f,0xfb1e,0x8bf9,0x9bd8,0xabbb,0xbb9a,
    0x4a75,0x5a54,0x6a37,0x7a16,0x0af1,0x1ad0,0x2ab3,0x3a92,
    0xfd2e,0xed0f,0xdd6c,0xcd4d,0xbdaa,0xad8b,0x9de8,0x8dc9,
    0x7c26,0x6c07,0x5c64,0x4c45,0x3ca2,0x2c83,0x1ce0,0x0cc1,
    0xef1f,0xff3e,0xcf5d,0xdf7c,0xaf9b,0xbfba,0x8fd9,0x9ff8,
    0x6e17,0x7e36,0x4e55,0x5e74,0x2e93,0x3eb2,0x0ed1,0x1ef0
};

//具体的crc16算法实现
uint16_t crc16(const char *buf, int len) {
    int counter;
    uint16_t crc = 0;
    for (counter = 0; counter < len; counter++)
            crc = (crc<<8) ^ crc16tab[((crc>>8) ^ *buf++)&0x00FF];
    return crc;
}
```
观察上面的crc16算法实现，根据前面 keyHashSlot 函数中的 crc16(key,keylen) & 0x3FFF 可以看到这边会计算‘kenlen’次crc运算，其中每次的运算结果又跟‘key’有关系，因为传入的key值和key的长度不一样，所以基本上可以保证键均匀的分配到各个槽位，各个节点上
[CRC校验算法](https://www.cnblogs.com/esestt/archive/2007/08/09/848856.html)

#### 集群扩容
Redis集群采用节点间通过发送Gossip消息，当一个 Redis 新节点运行并加入现有集群后，我们需要为其迁移槽和数据。首先要为新节点指定槽的迁移计划，确保迁移后每个节点负责相似数量的槽，从而保证这些节点的数据均匀。
1) 首先启动一个 Redis 节点，记为 M4。
2) 使用 cluster meet 命令，让新 Redis 节点加入到集群中。新节点刚开始都是主节点状态，由于没有负责的槽，所以不能接受任何读写操作，后续我们就给他迁移槽和填充数据。
3) 对 M4 节点发送 cluster setslot { slot } importing { sourceNodeId} 命令，让目标节点准备导入槽的数据。
4) 对源节点，也就是 M1，M2，M3 节点发送 cluster setslot { slot } migrating { targetNodeId} 命令，让源节点准备迁出槽的数据。
5) 源节点执行 cluster getkeysinslot { slot } { count } 命令，获取 count 个属于槽 { slot } 的键，然后执行步骤六的操作进行迁移键值数据。
6) 在源节点上执行 migrate { targetNodeIp} " " 0 { timeout } keys { key... } 命令，把获取的键通过 pipeline 机制批量迁移到目标节点，批量迁移版本的 migrate 命令在 Redis 3.0.6 以上版本提供。
7) 重复执行步骤 5 和步骤 6 直到槽下所有的键值数据迁移到目标节点。
8) 向集群内所有主节点发送 cluster setslot { slot } node { targetNodeId } 命令，通知槽分配给目标节点。为了保证槽节点映射变更及时传播，需要遍历发送给所有主节点更新被迁移的槽执行新节点。


#### 集群收缩
收缩节点就是将 Redis 节点下线，整个流程需要如下操作流程。

1) 首先需要确认下线节点是否有负责的槽，如果是，需要把槽迁移到其他节点，保证节点下线后整个集群槽节点映射的完整性。
2) 当下线节点不再负责槽或者本身是从节点时，就可以通知集群内其他节点忘记下线节点，当所有的节点忘记改节点后可以正常关闭。

下线节点需要将节点自己负责的槽迁移到其他节点，原理与之前节点扩容的迁移槽过程一致。

#### 客户端路由
在集群模式下，Redis 节点接收任何键相关命令时首先计算键对应的槽，在根据槽找出所对应的节点，如果节点是自身，则处理键命令；否则回复 MOVED 重定向错误，通知客户端请求正确的节点。这个过程称为 MOVED 重定向。

需要注意的是 Redis 计算槽时并非只简单的计算键值内容，当键值内容包括大括号时，则只计算括号内的内容。比如说，key 为 user:{10000}:books时，计算哈希值只计算10000。

MOVED 错误示例如下，键 x 所属的哈希槽 3999 ，以及负责处理这个槽的节点的 IP 和端口号 127.0.0.1:6381 。 客户端需要根据这个 IP 和端口号， 向所属的节点重新发送一次 GET 命令请求。

GET x
-MOVED 3999 127.0.0.1:6381
由于请求重定向会增加 IO 开销，这不是 Redis 集群高效的使用方式，而是要使用 Smart 集群客户端。Smart 客户端通过在内部维护 slot 到 Redis节点的映射关系，本地就可以实现键到节点的查找，从而保证 IO 效率的最大化，而 MOVED 重定向负责协助客户端更新映射关系。

Redis 集群支持在线迁移槽( slot ) 和数据来完成水平伸缩，当 slot 对应的数据从源节点到目标节点迁移过程中，客户端需要做到智能迁移，保证键命令可正常执行。例如当 slot 数据从源节点迁移到目标节点时，期间可能出现一部分数据在源节点，而另一部分在目标节点。
一致性hash算法

##### 故障转移
当 Redis 集群内少量节点出现故障时通过自动故障转移保证集群可以正常对外提供服务。

当某一个 Redis 节点客观下线时，Redis 集群会从其从节点中通过选主选出一个替代它，从而保证集群的高可用性。这块内容并不是本文的核心内容，感兴趣的同学可以自己学习。

但是，有一点要注意。默认情况下，当集群 16384 个槽任何一个没有指派到节点时整个集群不可用。执行任何键命令返回 CLUSTERDOWN Hash slot not served 命令。当持有槽的主节点下线时，从故障发现到自动完成转移期间整个集群是不可用状态，对于大多数业务无法忍受这情况，因此建议将参数 cluster-require-full-coverage 配置为 no ，当主节点故障时只影响它负责槽的相关命令执行，不会影响其他主节点的可用性。

集群中备份



其它拓展问题

Redis有多少个数据库，以及如何设置数据库，数据库之间数据是否隔离？ 
Redis由db0~db15共计16个数据库，可以通过select进行数据库的切换，不同db之间的key数据是相互隔离的，集群模式时不支持select操作，使用默认的db0：
```
127.0.0.1:6379> config get databases
1) "databases"
2) "16"
127.0.0.1:6379> set name xiaoming    # 默认使用db0数据库
OK
127.0.0.1:6379> get name
"xiaoming"
127.0.0.1:6379> select 1            # 切换到db1
OK
127.0.0.1:6379[1]> set name xiaobai # 验证两个数据库的key设置是否影响
OK
127.0.0.1:6379[1]> get name
"xiaobai"
127.0.0.1:6379[1]> select 0
OK
127.0.0.1:6379> get name
"xiaoming"
```

[Redis Cluster深入与实践](https://juejin.im/entry/59ce61c75188257e876a3a1a)

大量的数据插入操作？
使用pipe管道模式

修改配置不重启Redis会实时生效吗？
可以的，使用CONFIG SET 即可，可以通过CONFIG GET */具体配置项查看

[Redis 高级用法](https://juejin.im/entry/5720237071cfe400573e8919)



[有点意思：Redis在京东到家的订单中的使用](https://juejin.im/entry/598c0700f265da3e2c70fa83)

[Redis 布隆过滤器实战「缓存击穿、雪崩效应」](https://juejin.im/post/5c9442ae5188252d77392241)

[同步秒杀实现：Redis在秒杀功能的实践](https://juejin.im/post/5aa241c4518825558453983d#heading-5)

老钱：    
[求不更学不动之Redis5.0新特性Stream尝鲜](https://juejin.im/post/5b10ad586fb9a01e4072a520)    
[见缝插针 —— 深入 Redis HyperLogLog 内部数据结构分析](https://juejin.im/post/5b88d338e51d45387e51df63)
[HyperLogLog 算法的原理讲解以及 Redis 是如何应用它的](https://juejin.im/post/5c7900bf518825407c7eafd0)

Redession的特点，主要用在分布式环境下，尤其是对分布式锁有很好的封装实现，比Jedis功能简单，RadLock是实现分布式锁的核心，可以使用看门狗完成锁续命，其实核心思想是在请求获取到分布式锁之后，会启用一个子线程进行续命操作每隔10秒检测一下，就是延长锁的持有时间默认30秒(需要继续掌握Redession的原理和用法)    
[理解续期精品：【肥朝】面试官问我，Redis分布式锁如何续期？懵了](https://juejin.im/post/5d122f516fb9a07ed911d08c#heading-3)    
[Redis RedLock 完美的分布式锁么？](https://juejin.im/post/59f592c65188255f5c5142d2)

[限流策略](https://juejin.im/entry/5a15179b518825619a02548c)
[Redis应用-限流](https://juejin.im/post/5d1ec228f265da1bb56517cc)
[高并发下漏洞桶限流设计方案 - Redis](https://juejin.im/post/5d11cef9e51d4550a629b2a6)

面试官问：为什么选择Redis不选Memcache？
[缓存Memcached 与 Redis 相同点差异点分析](https://juejin.im/entry/59a3d988518825242670c5b2)

[如何处理redis集群中hot key和big key](https://juejin.im/post/5c19be51f265da615c593351)

[高并发和海量数据下的 9 个 Redis 经典案例剖析！](https://juejin.im/post/5d2304e6f265da1b7f29a26b)
[深入理解Redis的scan命令](https://juejin.im/post/5bbcc8325188255c74553ae3)
[［Redis源码阅读］当你输入get/set命令的时候，Redis做了什么](https://juejin.im/post/5b1dcc6b5188257d9b79d8a1)
[缓存数据库Redis结合Lua脚本解析](https://juejin.im/entry/5aa1f7b151882555666f4239)

[你所不知道的Redis热点问题以及如何发现热点](https://juejin.im/post/5cee39a26fb9a07ed36e8c4c)
[玩转Redis集群之Cluster](https://juejin.im/post/5c1bb40a6fb9a049f36211b0)