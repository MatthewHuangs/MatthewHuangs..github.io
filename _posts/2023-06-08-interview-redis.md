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



### 2、AOF 日志如何实现

### 3、RDB 快照的实现

### 4、为什么会有混合持久化

---

## 四、Redis 集群

### 1、如何实现服务高可用

### 2、集群脑裂导致数据丢失怎么办

---

## 五、Redis 过期删除与内存淘汰

### 1、过期删除的策略

### 2、持久化时/主从时对过期键如何处理

### 3、内存满了会发生什么

### 4、内存淘汰策略

### 5、LRU 算法和 LFU 算法

---

## 六、Redis 缓存机制

### 1、如何避免缓存雪崩、击穿、穿透

### 2、如何设计缓存策略，动态缓存热点数据

### 3、常见的缓存更新策略

### 4、如何保证缓存和数据库数据的一致性

---

## 七、Redis 实战

### 1、如何实现延迟队列

### 2、大 key 如何处理

### 3、管道有什么用

### 4、事务支持回滚吗

### 5、如何实现分布式锁
