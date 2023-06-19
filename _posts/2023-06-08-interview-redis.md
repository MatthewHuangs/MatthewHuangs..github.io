---
layout:       post
title:        "Redis 常见面试题"
author:       "matt"
header-style: text
catalog:      true
# header-img: "img/post-bg-css.jpg"
# header-img-credit: "@WebdesignerDepot"
# header-img-credit-href: "medium.com/@WebdesignerDepot/poll-should-css-become-more-like-a-programming-language-c74eb26a4270"
# header-mask: 0.4
tags:
    - interview
    - redis
---

## 一、Redis 基础知识  

### 1、作用和其他缓存的区别

（1）Redis 的介绍  
Redis（Remote Dictionary Service）是一种基于内存的数据库，对数据的读写操作都是在内存中完成的，读写速度快，常用于缓存、消息队列、分布式锁等场景。

执行命令由单线程控制，且对数据类型的操作都是原子操作，不存在并发竞争的问题。

Redis 还支持事务、持久化、Lua 脚本、多种集群方案（主从复制模式、哨兵模式、切片集群模式）、发布/订阅模式、内存淘汰机制、过期删除机制等。

（2）Redis 和 Memcached 的区别  
- Redis 的数据类型更丰富，Memcached 只支持 key-value 的数据类型；
- Redis 支持数据的持久化，Memcached 数据全部在内存中，容易丢失；
- Redis 原生支持集群模式，Memcached 需要依赖客户端来实现往集群中分片写入数据
- Redis 支持发布/订阅模式、Lua 脚本、事务等功能，Memcached 不支持。

（3）Redis 作为 MySQL 的缓存
- 高性能
第一次访问 MySQL 是从磁盘上读取数据，将数据缓存在 Redis 中，下次访问直接从内存中获取，速度快。需要注意二者双写一致性的问题。
- 高并发  
单台 Redis 的 QPS 是 MySQL 的10倍，可以轻松破10w。

### 2、常用数据类型和场景  

key 是字符串对象，而 value 可以是字符串对象，也可以是集合数据类型的对象。

![Redis 键值对底层数据结构](https://cdn.xiaolincoding.com//mysql/other/3c386666e4e7638a07b230ba14b400fe.png)


（1）String 字符串

- 内部实现  
底层的数据结构是 int 和 SDS（简单动态字符串）。而非 C 字符串的 `char *` 类型。

```c
// sds.h：SDS 数据结构
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[]; /* save data */
};
```

- 优势  
a. SDS 可以保存文本数据和二进制数据，C 语言中 `\0` 表示结束符；  
b. 获取字符串长度的时间复杂度是 O(1)，保存了 `len` 属性；  
c. SDS API 是安全的，拼接字符串会自动扩容，不会造成缓冲区溢出。根据 `alloc-len` 判断，SDS 长度小于 1MB 翻倍扩容，大于时每次增加 1MB；  
d. 节省内存空间，根据不同类型的结构体中 `len` 和 `alloc` 属性，编译过程中按实际占用字节数优化对齐。  


（2）List 列表

- 数据结构  
使用双向链接实现。3.0版本在数据量少的情况下，采用内存紧凑的`压缩列表`来节省内存空间，3.2版本改为 `quicklist`，5.0版本改为 `listpack`。  

- ziplist（3.0之前）  
压缩列表（zl）是由连续内存块组成的顺序型数据结构，查找效率为 O(n)，且会因为增加或修改元素，导致内存重新分配，影响 prevlen，甚至`连锁更新`影响性能。

```c
// ziplist.h
/* Each entry in the ziplist is either a string or an integer. */
typedef struct {
    /* When string is used, it is provided with the length (slen). */
    unsigned char *sval;
    unsigned int slen;
    /* When integer is used, 'sval' is NULL, and lval holds the value. */
    long long lval;
} ziplistEntry;

// ziplist.c
typedef struct zlentry {
    unsigned int prevrawlensize; 
    unsigned int prevrawlen;     
    unsigned int lensize;        
    unsigned int len;            
    unsigned int headersize;     
    unsigned char encoding;      
    unsigned char *p;            
} zlentry;
```
![压缩列表及节点结构](https://cdn.xiaolincoding.com//mysql/other/a3b1f6235cf0587115b21312fe60289c.png)


- quicklist（3.2之后）  
quicklist 是一个`双向链表`，而链表中的每个元素又是一个`压缩列表`。为了解决`连锁更新`的问题。

```c
typedef struct quicklist {
    //quicklist的链表头
    quicklistNode *head;      //quicklist的链表头
    //quicklist的链表尾
    quicklistNode *tail; 
    //所有压缩列表中的总元素个数
    unsigned long count;
    //quicklistNodes的个数
    unsigned long len;       
    ...
} quicklist;

typedef struct quicklistNode {
    //前一个quicklistNode
    struct quicklistNode *prev;     //前一个quicklistNode
    //下一个quicklistNode
    struct quicklistNode *next;     //后一个quicklistNode
    //quicklistNode指向的压缩列表
    unsigned char *zl;              
    //压缩列表的的字节大小
    unsigned int sz;                
    //压缩列表的元素个数
    unsigned int count : 16;        //ziplist中的元素个数 
    ....
} quicklistNode;
```

qicklist 通过 quicklistNode 结构里压缩列表的大小或元素个数，来减少连锁更新带来的性能影响，但并未完全解决。  
![quicklist 的数据结构](https://cdn.xiaolincoding.com//mysql/other/f46cbe347f65ded522f1cc3fd8dba549.png)

- listpack（5.0之后）  
通过不在包含前一个节点的长度，来彻底解决连锁更新的问题。不过还是可以通过 `lpDecodeBacklen` 函数来向左逐个字节解析，获取 prevlen 值。  

![listpack 的数据结构](https://cdn.xiaolincoding.com//mysql/other/c5fb0a602d4caaca37ff0357f05b0abf.png)

（3）Hash 哈希/字典

- 数据结构  
哈希表查询复杂度为 O(1)，Redis 采用了`链式哈希`来解决哈希冲突。
```c
// dict.h
typedef struct dictht {
    //哈希表数组
    dictEntry **table;
    //哈希表大小
    unsigned long size;  
    //哈希表大小掩码，用于计算索引值
    unsigned long sizemask;
    //该哈希表已有的节点数量
    unsigned long used;
} dictht;

typedef struct dictEntry {
    //键值对中的键
    void *key;
  
    //键值对中的值，联合体，部分情况可以节省内存
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    //指向下一个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```

- rehash  
实际使用哈希表时，Redis 会在 dict 结构体中定义两个哈希表，采用`渐进式 rehash` 的方式分次将哈希表1中的数据迁移到哈希表2，并将表2设置为表1，释放表1。  
触发 rehash 的时间跟负载因子的大小（负载因子=哈希表已保存节点数量/哈希表大小）有关。
```c
typedef struct dict {
    …
    //两个Hash表，交替使用，用于rehash操作
    dictht ht[2]; 
    …
} dict;
```

（4）Set 集合  

这里当 Set 对象只包含整数值元素，且元素量不大时，介绍整数集合作为底层实现的情况。  
升级规则：新元素比现有元素类型都要长时，`contents` 数组进行扩容，避免内存浪费，且不支持降级操作。
```c
typedef struct intset {
    //编码方式
    uint32_t encoding;
    //集合包含的元素数量
    uint32_t length;
    //保存元素的数组，类型根据 encoding 的值来改变
    int8_t contents[];
} intset;
```

（5）Zset 有序集合  

zset 结构体中包含`跳表`、`哈希表`两种数据结构，同步更新记录，既能够高效范围查询（zsl 平均 O(logN) 节点查询），也可以高效单点查询（dict O(1)单节点查询）。
```c
typedef struct zset {
    // 用于以常数复杂度获取元素权重，不常提到
    dict *dict;
    zskiplist *zsl;
} zset;
```

- 跳表数据结构  

```c
// server.h
typedef struct zskiplistNode {
    //Zset 对象的元素值
    sds ele;
    //元素权重值，排序用
    double score;
    //后向指针，方便倒叙查询
    struct zskiplistNode *backward;
  
    //节点的level数组，保存每层上的前向指针和跨度，每一个元素代表跳表的一层
    struct zskiplistLevel {
        struct zskiplistNode *forward; 
        unsigned long span; // 跨度用于计算这个节点在 skl 中的排位
    } level[];
} zskiplistNode;
```
![跳表结构图](/img/post/post-230608-skiplist.png)

跳表的相邻两层的节点数量最理想的比例是2：1，查询复杂度可降为 O(logN)。跳表在创建节点的时候，随机生成每个节点的层数（随机数小于0.25时层数+1，直至最大值64/32）。

跳表比平衡树的优势：  
内存占用少；范围查找简单；算法实现难度低，插入/删除操作引发的子树调整少。

（6）后续版本更新

- BitMap（2.2）：二值状态统计的场景；
- HyperLogLog（2.8）：海量数据基数统计的场景；
- GEO（3.2）：存储地理位置信息的场景；
- Stream（5.0）：消息队列，相较于基于 List 类型实现的情况，有自动生成全局唯一的消息 ID 和支持以消费组形式消费数据的特性。  

---

## 二、Redis 线程模型

### 1、关于单线程  

Redis 单线程是指`接收客户端请求->解析请求->进行数据读写等操作->发送数据给客户端`这个过程由一个主线程完成。  

但是在 Redis 程序在启动后，会启动`关闭文件`、`AOF 刷盘`（Append-Only File，持久化内存数据到磁盘）、`释放内存`三个后台线程（BIO）。BIO 相当于一个消费者，轮询任务队列，拿到任务执行相应的耗时方法，防止主线程阻塞。  

- 单线程却仍然快（10w QPS）  
大部分操作在内存中完成，并且采用了高效的数据结构，瓶颈在内存和网络带宽，而不在 CPU（单线程即可的原因）；  
单线程模型避免了多线程之间的竞争，省去了多线程切换的时间和性能上的开销，且不会死锁；  
采用了 I/O 多路复用机制处理大量的客户端 Socket 请求（select/epoll 机制）。

### 2、Redis 6.0

Redis 6.0版本之后，因为随着网络硬件的性能提升，性能瓶颈有时会出现在网络 I/O 的处理上，Redis 开始采用多个 I/O 线程来处理网络请求，但对于命令的执行仍然是单线程。

- Redis 启动后默认会创建6个线程（不包括主线程）：    
**Redis-server：** Redis 的主线程，负责执行命令；
**bio_close_file、bio_aof_fsync、bio_lazy_free：** 后台线程，分别异步处理关闭文件任务、AOF 刷盘、释放内存；  
**io_thd_1、io_thd_2、io_thd_3：** `io-threads` 配置项默认是4，启动3个 I/O 多线程来分担网络 I/O 的压力。

---

## 三、Redis 持久化

### 1、如何实现数据不丢失

Redis 重启后，内存中的数据清空，从磁盘中恢复原有的数据。三种数据持久化的方式：
- AOF 日志：没执行一条写操作命令，就把该命令以追加的方式写入到一个文件中；
- RDB 快照：将某一时刻的内存数据，以二进制的方式写入磁盘；
- 混合持久化方式：Redis 4.0中新增，集成了 AOF 和 RDB 的优势。

### 2、AOF 日志如何实现

- 步骤

> 客户端 -> 发送写命令到 Redis -> 先：执行写命令到内存 -> 后：执行写命令到日志（硬盘）

- AOF 写回策略

Redis 执行完写操作命令后，会将命令追加到 server.aof_buf 缓冲区，通过 write() 系统调用，写入到 AOF文件，此时文件在内核缓冲区中，不在硬盘。
![Redis 写入 AOF 日志的过程](https://cdn.xiaolincoding.com//mysql/other/4eeef4dd1bedd2ffe0b84d4eaa0dbdea-20230309232249413.png)

`Redis.conf`配置文件的`appendfsync`配置项：
Always：每次写操作同步写回；  
Everysec：每秒写回；  
No：由操作系统控制写回。

－ AOF 日志过大的重写机制  
重写时，读取当前数据库中的所有键值对，每一个键值对应用一条命令记录到`新的 AOF 日志文件`中，全部记录完成后替换现有的 AOF 文件。  

重写过程由 `bgrewriteaof 子进程`完成，在父子进程共享内存数据的同时，避免了多线程之间共享内存导致的数据加锁操作。

重写过程中主进程依旧可以正常处理命令，这会涉及到`压缩`（合并操作记录）、`数据不一致`（写操作同时写入 AOF 缓冲区和 AOF 重写缓冲区）等问题。  

> 操作系统相关知识：  
1、信号是进程间通讯的一种方式，且是异步的。  
2、父子进程共享内存数据，子进程只读，父进程修改共享内存后，会发生写时复制，父子进程拥有独立的数据副本，避免加锁操作。

### 3、RDB（Redis Database）快照的实现

RDB 快照记录某一个瞬间的实际内存数据，而 AOF 日志文件记录的时写命令的操作。

生成 RDB 文件的命令有`save`和`bgsave`，前者在主线程中生成，会阻塞主线程，后者不会。

Redis 快照是`全量快照`，记录内存中所有的数据，操作频繁会影响性能。通过 Redis 中的 save 配置选项可以设置。
```bash
# 表示300秒内，对数据库进行了10次的修改，则开始生成 RDB 快照文件
save 300 10
```

在执行 bgsave 时，Redis 可以继续处理操作命令，运用了写时复制技术（Copy-On-Write，COW）。父子进程共享内存数据，写入时会创建一个被修改的数据副本，由 bgsave 子进程读入 RDB 文件。

### 4、为什么会有混合持久化（4.0之后）

- 原因

RDB 数据恢复快，但是快照的频率不好把握；AOF 丢失数据少，恢复速度慢。Redis 4.0后混合使用 AOF 日志和 RDB 快照。

- 用法

在 AOF 日志重写过程，重写子进程会将共享内存中的内存数据以 RDB 的方式写入 AOF 文件，主线程后续操作命令会记录到重写缓冲区，增量命令会以 AOF 文件的方式写入 AOF 文件中。`含有 RDB 和 AOF 格式的 AOF 文件`会替换原有的 AOF 文件。  

- 优劣

前段 RDB 格式内容使得 Redis 可以快速启动，同时结合 AOF 的优点，可以减少数据丢失。  
这种文件格式可读性较差，且不兼容 4.0 版本之前的 Redis。

---

## 四、Redis 集群

### 1、如何实现服务高可用

- 主从复制  

主从服务器读写分离，主服务器进行读写操作，写操作发生后自动将命令同步给从服务器，不等待从服务器执行完毕直接向客户端返回结果。所以，无法保证数据的强一致性（时刻一致）。

![主从复制架构](https://cdn.xiaolincoding.com//mysql/other/2b7231b6aabb9a9a2e2390ab3a280b2d.png)

- 哨兵模式  

哨兵模式（Redis Sentinel）自动监控 Redis 集群的主从节点，一旦发现主节点宕机，自动选择一台从节点切换为主节点，提供主从节点的故障转移功能。

![哨兵模式架构](https://cdn.xiaolincoding.com//mysql/other/26f88373d8454682b9e0c1d4fd1611b4.png)

- 切片集群

Redis 缓存数据量大到一台服务器无法缓存时，采用切片集群（Redis Cluster）方案将数据分布在不同的服务器上，减少系统对单节点的依赖。

Redis Cluster 采用16384个哈希槽（Hash Slot）来处理数据和节点之间的映射关系。根据每个`数据`键值对的 key 哈希取模后对应到一个`哈希槽`中，再将槽通过平均分配或手动分配的方式映射到相应的`Redis 节点`，

![切片集群架构](https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E5%85%AB%E8%82%A1%E6%96%87/redis%E5%88%87%E7%89%87%E9%9B%86%E7%BE%A4%E6%98%A0%E5%B0%84%E5%88%86%E5%B8%83%E5%85%B3%E7%B3%BB.jpg)

### 2、集群脑裂导致数据丢失怎么办

- 集群脑裂

由于网络问题，集群节点之间失联，主从数据不同步；重新选举产生了两个主节点；等网络恢复后，原主节点降为从节点，再与新主节点同步复制的时候会清空自己的缓冲区，导致之前客户端写入数据的丢失。

- 解决方式

结合 Redis 配置文件参数：  
`min-slaves-to-write x`：主节点必须要有 x 个从节点连接，如果小于则禁止主节点写入数据；  
`min-slaves-max-lag x`：主从数据复制和同步的延迟不能超过 x 秒，若果超过则禁止主节点写入数据。

---

## 五、Redis 过期删除与内存淘汰

### 1、过期删除的策略

策略：`惰性删除`+`定期删除`，每个设置了过期时间的 key，Redis 会把该 key 带上过期时间存储到一个过期字典（expires dict）中。  

`惰性删除`：不主动删除过期键，每次从数据库访问 key 时，都检测 key 是否过期，如果过期才删除该 key。  

`定期删除`：每隔一段时间随机从数据库中取出一定数量的 key 进行检查并删除，如果过期数量超过 25% 则再次取出检查（此循环的时间上限为25ms），否则停止等待下一轮检查。  

### 2、持久化时/主从模式对过期键如何处理

- RDB 文件  

**生成阶段：** 对 key 进行过期检查，过期的 key 不会被保存到新的 RDB 文件中；  
**加载阶段：** 若是主服务器，在载入 RDB 文件时进行过期检查；若是从服务器，则不检查直接载入。

- AOF 文件  

**写入阶段：** 如果某过期键没被删除，则 AOF 文件会保留该键，当此键过期后，会向 AOF 文件追加一条 DEL 命令显性删除；  
**重写阶段：** 进行过期检查，已过期的键不会被保存到重写后的 AOF 文件中。  

- 主从模式  

从库不会进行过期扫描，对过期的处理是`被动的`，即使过期了也可的到 key 的值。只有主库 key 到期后，将 AOF 文件添加的 DEL 命令同步到所有的从库，实现删除。

### 3、内存满了会发生什么

Redis 运行内存达到某个阈值（最大运行内存 maxmemory），会触发`内存淘汰机制`。

### 4、内存淘汰策略

Redis 内存淘汰策略共有8种，大致为分进行/不进行数据淘汰2类。

- 不进行数据淘汰的策略  
  - noeviction（3.0之后默认）：不再提供服务，直接返回错误。

- 进行数据淘汰的策略
  - volatile-random：随机淘汰设置了过期时间的任意键值；
  - volatile-ttl：优先淘汰更早过期的键值；
  - volatile-lru：淘汰所有过期时间中最久未使用的键值；
  - volatile-lfu：淘汰所有过期时间中最少使用的键值；
  - allkeys-random：随即淘汰任意键值；
  - allkeys-lru：淘汰整个键值中最久未使用的键值；
  - allkeys-lfu：淘汰整个键值中最少使用的键值。

### 5、LRU 算法和 LFU 算法

```c
typedef struct redisObject {
    ...
      
    // 24 bits，用于记录对象的访问信息
    unsigned lru:24;  
    ...
} robj;
```

- LRU（Least Recently Used）最近最少可用

> 24bits 的 lru 字段用来记录 key 的访问时间戳

传统的 LRU 算法是基于链表结构实现的，链表元素按照操作时间顺序从前往后排列，链表尾部元素代表最久未被使用的元素。

Redis 实现的是在对象结构体中新增一个字段，记录此数据的最后一次访问时间。进行内存淘汰时，用随机采样的方式取 x 个数据，淘汰最久未使用的那个。

- 缓存污染  
一次读取大量数据，这些数据只会被使用一次，但这些数据会留存在 Redis 缓存中很长时间。

- LFU（Least Frequently Used）最近最不常用

> 24bits 的 lru 字段被分为两段存储时间戳和频次

高16bits 存储 ldt（Last Decrement Time）记录 key 的访问时间戳；低8bits 存储 logc（Logistic Counter）记录 key 的访问频次。

---

## 六、Redis 缓存机制

### 1、如何避免缓存雪崩、击穿、穿透

- 缓存雪崩

**定义：** 大量缓存数据在同一时间失效，此时有大量的用户请求访问这些数据，导致数据库压力骤增，严重的会造成数据库宕机，形成一系列连锁反应。

**解决方案：**  
1）随机打散缓存时间，在原有的失效时间基础上增加一个随机值（如1~10分钟）；  
2）设置缓存不过期，通过后台服务来控制缓存数据的生命周期。

- 缓存击穿 

**定义：** 热点数据过期，此时大量的请求访问该热点数据，无法命中缓存开始直接访问数据库。类似缓存雪崩的子集。

**解决方案：**  
1）互斥锁，使用 setNX 设置一个锁定的状态位，保证同一时间只有一个业务线程请求缓存；  
2）不给热点数据设置过期时间。

- 缓存穿透

**定义：** 当用户访问的数据，既不在缓存中，也不在数据库中，没法构建缓存数据。当大量这样的请求到来时，数据库压力骤增，是常见的黑客攻击手段。

**解决方案：**  
1）对非法请求进行限制，直接返回错误；  
2）在缓存中设置空值或默认值；  
3）使用布隆过滤器（Redis 自身支持）快速判断数据是否存在，避免通过查询数据库来判断数据是否存在。

### 2、如何设计缓存策略，动态缓存热点数据

**思路：** 通过数据最新访问时间排名，并过滤掉不常访问的数据，只留下经常访问的数据。

**详情：** 电商场景缓存经常访问的 Top100 商品  
```shell
# 1、定义一个`zset`，变量名为用户 ID，key 对应商品 ID，value 为该商品的访问次数
ZADD "user:1" 0 "product:1" 0 "product:2" ... 0 "product:n"

# 2、当用户1访问了商品1时，增加访问次数
ZINCRBY "user:1" 1 "product:1"

# 3、【如需】联合用户画像相似的3个用户数据 zset
ZUNIONSTORE top100 3 user:1 user:2 user:3 AGGREGATE MAX
ZREVRANGE top100 0 99 WITHSCORES

# 4、读取 Top100 商品1的信息
redis.set(`product:${productID}`, productInfo)

# 5、离线删除数据，防止内存占用
ZREMRANGEBYRANK user:1 0 99
```

### 3、常见的缓存更新策略

- Cache Aside 策略（旁路缓存）

应用程序直接与 DB 和 Redis 交互，负责对缓存的维护。

注意写策略时`不能先删除缓存再更新数据库`，会造成读写并发时数据不一致的问题。反之，因为写入缓存远快于写入数据库，不会有数据不一致的问题。

Cache Asdide 策略适合读多写少的场景，写多场景会频繁清理缓存数据，降低缓存命中率。可以通过添加分布式锁、给更新的缓存设置一个较短的过期时间来解决。

![Cache Aside 策略](https://cdn.xiaolincoding.com//mysql/other/6e3db3ba2f829ddc14237f5c7c00e7ce-20230309232338149.png)

- Read/Write Through 策略（读穿/写穿）

应用程序只与缓存交互，缓存再和数据库交互。
![Read/Write Through 策略](https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E5%85%AB%E8%82%A1%E6%96%87/WriteThrough.jpg)

- Write Back 策略（写回）

更新数据时，只更新缓存，同时将缓存数据设置为脏的，然后立马返回。数据库的更新，会通过批量异步处理更新的方式进行。

Write Back 策略不能应用到 DB 和 Redis 中，因为缺少异步更新的功能，常用于 CPU 、OS 文件系统的缓存。

适合写多的场景，但是数据不是强一致性的，会有数据丢失的风险。

### 4、如何保证缓存和数据库数据的一致性

- 先更新 DB 还是先更新缓？

这两个方案都存在并发问题，当两个请求并发更新同一条数据的时候和，可能会出现缓存和 DB 的数据不一致的问题。

- 先更新 DB 还是先删除缓存？

考虑读写并发的场景，因为缓存的写入速度要远快于数据库的写入，所以`先更新 DB 在删除缓存`可以保证数据的一致性。

这种情况的一致性保证，建立在更新 DB 和删除缓存操作都能执行成功。一是可以给缓存增加过期时间，二是可以采用`更新 DB 再以分布式锁的形式更新缓存，给缓存加上较短的过期时间`的方案。若要保证两个操作都能执行成功，可以添加`重试机制`、`订阅 MySQL binlog，再操作缓存`（阿里巴巴开源的 Cannal 中间件）。

---

## 七、Redis 实战

### 1、如何实现延迟队列

**场景：** 订单超时自动取消。

**解决方案：** 使用 `Zset` 实现，用 Score 属性来存储延迟执行的时间。
![Redis 实现延迟队列](https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E5%85%AB%E8%82%A1%E6%96%87/%E5%BB%B6%E8%BF%9F%E9%98%9F%E5%88%97.png)

### 2、大 key 如何处理

- 定义

key 对应的 value 很大，例如 String 类型的值大于 10KB，Hash、List、Set、ZSet 类型的元素的个数超过5000个。

- 影响 

1、**客户端超时阻塞。** 由于 Redis 执行命令是单线程处理，然后在操作大 key 时会比较耗时，那么就会阻塞 Redis，从客户端这一视角看，就是很久很久都没有响应。

2、**引发网络阻塞。** 每次获取大 key 产生的网络流量较大，如果一个 key 的大小是 1 MB，每秒访问量为 1000，那么每秒会产生 1000MB 的流量，这对于普通千兆网卡的服务器来说是灾难性的。

3、**阻塞工作线程。** 如果使用 del 删除大 key 时，会阻塞工作线程，这样就没办法处理后续的命令。

4、**内存分布不均。** 集群模型在 slot 分片均匀情况下，会出现数据和查询倾斜情况，部分有大 key 的 Redis 节点占用内存多，QPS 也会比较大。

- 找到大 key

1、从节点上执行命令行，会阻塞线程。

`redis-cli -h ${ip} -p ${port} -a ${password} --bigkeys`

2、`SCAN` 命令

3、`RdbTools 第三方开源工具`，解析 RDB 快照文件，将大于 10KB 的 key 输出到表格文件。
`rdb dump.rdb -c memory --bytes 10240 -f redis.csv`

- 删除大 key

删除操作会涉及释放内存、将释放后的内存块插入一个空闲内存块链表，便于管理和再分配。

1、分批次删除

删除大 Hash，使用 `hscan` 命令每次获取100个字段，再用 `hdel` 命令每次删除1个字段；

删除大 List，通过 `ltrim` 命令每次删除少量元素；

删除大 Set，通过 `sscan` 命令每次扫描集合中100个元素，再用 `srem` 命令每次删除一个键；

删除大 ZSet，使用 `zremrangebyrank` 命令，每次删除 top100 个元素。

2、异步删除（4.0之后）

用 `unlink` 命令来代替 del 删除。

还可以通过配置参数（默认关闭，建议开启前三），在达到某些条件后自动进行异步删除：

```shell
lazyfree-lazy-eviction: 表示当 Redis 运行内存超过 maxmeory 时，是否开启 lazy free 机制删除；
lazyfree-lazy-expire: 表示设置的过期时间到期后，是否开启 lazy free 机制删除；
lazyfree-lazy-server-del: 隐式删除已存在的键时，是否开启 lazy free 机制删除；
noslave-lazy-flush: 从节点加载主节点的 RDB 文件前，运行 flushall 清理自身数据时，是否开启 lazy free 机制删除。
```

### 3、管道有什么用

Pipeline 管道技术是`客户端`提供的一种批处理技术，用于一次性处理多个 Redis 命令，减少多个命令执行时的网络等待。注意避免发送的命令过大，或者管道内的数据太多导致网络阻塞。

### 4、事务支持回滚吗

Redis 中`没有提供回滚机制，操作非原子性`，`DISCARD` 命令只能主动放弃事务执行，清空命令队列，不能起到回滚效果。

原因在于 Redis 事务执行通常是编程错误导致，以及违背 Redis 追求简单高效的设置主旨。

### 5、如何实现分布式锁

- 并发问题

读取、保存等操作不是原子操作，所以要用分布式锁来限制程序的并发执行。  
`setnx`（set if not exists）和`expire`是两个指令而非原子指令，所以直接使用无法实现。

- set 扩展参数（2.8之后）

（1）加锁  
Redis 实现分布式锁，针对加锁操作需要满足三个条件： 
a. 加锁包括读取锁变量、检查变量值、设置变量值三个以原子操作方式完成的操作，带上`NX`选项来实现加锁；  
b. 锁变量要设置过期时间，避免锁异常导致无法释放，带上`EX/PX`选项设置过期时间；  
c. 锁变量需要能区分来自不同客户端的加锁操作，避免误释放，将锁变量值设置为客户端唯一的值，用于标记客户端。

```bash
# 以 NX 的方式设置 k-v，只有当 key 不存在时才进行设置操作，并且设置超期时间为10s
SET ${loack:key} ${unique_value} NX EX 10
```

（2）解锁  
解锁操作要保证执行操作的客户端和加锁的客户端是同一个，判断上面锁变量的值`${unique_value}`即可，这里只能使用 Lua 脚本来执行。

```go
// ReleaseLock del 释放分布式锁，除超期情况外，避免其他线程释放锁
// 该方法仍非完全线程安全，1、超时且当前线程未执行完，其他线程仍可以侵入；2、value 的比对最好使用 uuid.New().String()
func ReleaseLock(key string, value interface{}) error {
	luaScript := `
	if redis.call("get",KEYS[1]) == ARGV[1]
	then
		return redis.call("del",KEYS[1])
	else
		return 0
	end
	`
	unlockScript := redis.NewScript(luaScript)

    // func (s *Script) Run(c scripter, keys []string, args ...interface{}) *Cmd
	_, err := unlockScript.Run(redisClient, []string{key}, value).Result()
	return err
}
```

- 优点

性能高效（缓存实现的核心出发点）；  
实现方便，setnx 方法；  
避免单点故障，Redis 跨集群部署。

- 缺点

超时时间不好设置，会收到业务代码执行时间的影响（设置超时时间的同时启动一个守护线程重置超时时间）；  
Redis 主从复制模式中的数据是异步复制的，导致分布式锁不可靠。

- 集群情况下分布式锁的可靠性

基于多个 Redis 节点的分布式锁，即使有节点发生了故障，锁变量仍然是存在的，客户端还是可以完成锁操作。官方推荐`至少部署5个孤立的 Redis 主节点`。

**Redlock（红锁）：** 让客户端和多个独立的 Redis 节点依次请求申请加锁，如果客户端能够和`半数以上`的节点成功地完成加锁操作，那么我们就认为，客户端成功地获得分布式锁，否则加锁失败。
