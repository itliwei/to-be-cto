#Redis基础篇

[TOC]

### 一、从常用数据结构说起

上文说到Redis提供了丰富的数据结构，包括STRING（字符串）、LIST（列表）、SET（集合）、HASH（散列）和ZSET（有序集合）基本数据类型。

先来一波操作感受一下：

![](./imgs/20210310215000.jpg)

基本的数据类型的大概使用就到这里，接下来就分析一下它的内部结构是怎么实现的。

### 二、底层实现

Redis是KV类型的数据库，Key-Value我们一般会用什么数据结构存储？

哈希表！没错Redis的最外层确实也是通过hashtable实现的。在Redis里面每个键值对都是一个dictEntry，通过指针指向key的存储结构和value的存储结构，此外还有一个next存储里指向下一个键值对的指针。

```c
typedef struct dictEntry {
    void *key; //key void*表示任意类型指针

    union {                   
       void      *val;//value定义
       uint64_t  u64;
       int64_t   s64;
       double   d;
    } v;
    struct dictEntry *next;   //next指针
} dictEntry;
```

看到这里大家会有疑惑，那过期时间放在哪里？

对，实际上在dicEntry的外面还有一层redisDB。

```c
/* Redis数据库结构体 */
typedef struct redisDb {
    // 数据库键空间，存放着所有的键值对（键为key，值为相应的类型对象）
    dict *dict;                 
    // 键的过期时间
    dict *expires;              
    // 处于阻塞状态的键和相应的client（主要用于List类型的阻塞操作）
    dict *blocking_keys;       
    // 准备好数据可以解除阻塞状态的键和相应的client
    dict *ready_keys;           
    // 被watch命令监控的key和相应client
    dict *watched_keys;         
    // 数据库ID标识
    int id;
    // 数据库内所有键的平均TTL（生存时间）
    long long avg_ttl;         
} redisDb;
```

外层说完了，接下来就以set hello world为例，逐步分析一下。

首先key是字符串，Redis自己实现了一个字符串类型叫SDS，后面我们会细说，所以hello指向一个SDS的存储结构。value是world，也是一个字符串，是不是也用SDS存储呢？

这里就要重点介绍以下了，由前文我们知道Redis的key都是字符串类型，value的类型有多种。那Redis如何是适配这多种类型呢？于是就封装了一层redisObject。而我们所说的Redis数据类型的任何一种，都是通过redisObject存储的。

我们来看一下redisObject的结构：

```c
typedef struct redisObject {
    //对象的数据类型，占4bits，共5种类型
    unsigned type:4;        
    //对象的编码类型，占4bits，共10种类型
    unsigned encoding:4;
    //least recently used
    //实用LRU算法计算相对server.lruclock的LRU时间
    unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */
    //引用计数
    int refcount;
    //指向底层数据实现的指针
    void *ptr;
} robj;
```

到这里就能看出一些奇妙的地方了，type是数据类型，encoding是编码类型，且数量不等。有意思了，那我们接下来看看这个encoding究竟是些什么。

> 127.0.0.1:6379> set number 123
> OK
> 127.0.0.1:6379> object encoding number
> "int"
> 127.0.0.1:6379> set story "long long ago,and long long ago,other long long ago"
> OK
> 127.0.0.1:6379> object encoding story
> "raw"
> 127.0.0.1:6379> set msg "hello world"
> OK
> 127.0.0.1:6379> object encoding msg
> "embstr"

到此我们大概看出来一些端倪，Redis的数据类型背后根据存储的数据不通使用的不同的编码存储。为什么要这样做呢？

节约存储空间！

其实远远不止这些，我们后面会详细说到。接下来就看每种数据结构有哪些编码类型。

### 三、数据结构详解

#### 1、String

上文我们已经分析出了，String底层有三种。

> int：存储8个字节的长整形
>
> embstr：代表embsds格式的SDS，存储小于44字节的字符串
>
> raw：存储大于44字节的SDS

接下来问题来了，什么是SDS呢？SDS全称是Simple Dynamic String，即简单动态字符串。

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* 长度 */
    uint8_t alloc; /* 分配的内存大小 */
    unsigned char flags; /* 属性，标志不同种类的sds */
    char buf[];/* 内容 */
};
```

本质上就是一个char数组！

那么既然是一个char数组，为什么要通过SDS实现呢？

#####SDS

很简单，因为C语言中没有字符串类型，只有char数组。但是使用char数组会有一些问题，什么问题呢，大家可以想一下。

> 1、必须分配足够的空间，否则可能会溢出
>
> 2、如果要获取长度，需要遍历数组，时间复杂度是O(n)
>
> 3、如果长度变更，需要重新内存分配
>
> 4、字符串规则是遇到第一个'/0'即为结束，因此不能存放二进制的内容

好了，既然有了这些问题，SDS是如何解决的呢？

> 1、SDS实现了动态扩容，无需担心溢出的问题
>
> 2、定义了len属性，获取长度时间复杂度是O(1)
>
> 3、通过空间预分配和惰性空间释放，放置了重新分配内存
>
> 4、判断结束标志示len属性，避免了二进制不安全

哇，到这里是不是感觉到Redis的编码格式设计的很巧妙。别着急，慢慢来，还有。

embstr和raw，为什么要设计两个编码格式呢？就是为了长度不同吗？SDS也已经满足了呀？

实际原因是embstr的使用，只分配了一次内存，redisObject和SDS是一起分配的，二raw是分配了两次内存

![image-20210310231752229](/Users/vince/Library/Application Support/typora-user-images/image-20210310231752229.png)

那这三种类型之间是怎么转换的呢？

> 1、int 不在时整形，转成raw
>
> 2、int大小超过long的范围，转成embstr
>
> 3、embstr超过44字节，转成raw

注意：不可回转！！！

到此，String的数据结构就介绍完毕了。最后大家都思考一下String的使用场景有哪些？

1、缓存：热点数据，提升检索效率；

2、数据共享：session共享多个服务器共用；

3、分布式锁：setNx方法，判断是否添加成功；

4、全局唯一ID：INT类型的incrby；

5、计数器：INT类型

6、限流：INT类型

哈哈，是不是感觉String好强大，不需要其他数据类型了？那么问题来了，如果我要存一个对象怎么办？举个例子，存一个学生信息，包括姓名、年龄、学号等信息。大家可能会说，String存储一个json就行了，没错可以实现。但是我如果只想获取学生的年龄呢？

接下来介绍的Hash类型，就是解决这个问题的。

#### 2、Hash

Hash类型是指Redis键值对中的值本身又是一个键值对结构，形如`value=[{field1，value1}，...{fieldN，valueN}]`，hash的value只能是字符串，不能嵌套其他类型。同样是存储字符串，Hash和String有什么区别呢？如下图所示:

![image-20210310235701634](/Users/vince/Library/Application Support/typora-user-images/image-20210310235701634.png)

从上图可以明显看出：

> 1、把所有相关的值聚集到一个key中，节省内存空间
>
> 2、只使用一个key，可以有效减少key冲突
>
> 3、当需要批量获取值的时候，只需要使用一个命令，减少内存/IO/CPU

那么它的底层编码是如何实现的呢？是不是使用dicEntry实现的呢？我们先来操作一波：

> 127.0.0.1:6379> hset user1 name aaaaaaaaaaaaaaaaaaa
> (integer) 1
> 127.0.0.1:6379> hset user2 name aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
> (integer) 1
> 127.0.0.1:6379> object encoding user1
> "ziplist"
> 127.0.0.1:6379> object encoding user2
> "hashtable"

可以看到，哈希类型的内部编码有两种：ziplist(压缩列表),hashtable(哈希表)。又有两个陌生的结构，不要慌张，我们下文会逐个分析。

#####ziplist

ziplist是一个经过特殊编码的，由连续的内存块组成的双向链表。它不存储指向上一个节点和下一个节点的指针，而是存储上一个节点的长度和当前节点的长度。这让数据在内存中更为紧凑，同时可以轻易的得到前驱后驱数据项的位置。

```
<zlbytes><zltail><zllen><entry>...<entry><zlend>
```

![image-20210311001744632](/Users/vince/Library/Application Support/typora-user-images/image-20210311001744632.png)

------------图片

接下来我们具体看下实际元素里，究竟是怎么存储的。

```c
typedef struct zlentry {
    //prevrawlen 前驱节点的长度
    //prevrawlensize 编码前驱节点的长度prevrawlen所需要的字节大小
    unsigned int prevrawlensize, prevrawlen;
    //len 当前节点值长度
    //lensize 编码当前节点长度len所需的字节数
    unsigned int lensize, len;
    //当前节点header的大小 = lensize + prevrawlensize
    unsigned int headersize;
    //当前节点的编码格式
    unsigned char encoding;
    //指向当前节点的指针，以char *类型保存
    unsigned char *p;
} zlentry;                 
```

------------图片

那么什么时候使用ziplist呢？

> 1. hash对象保存的键值对数量<512
> 2. 所有键值对字符串长度小于54字节

如果任何一个条件不满足，存储结构就会转成hashtable。接下来介绍hashtable

##### hashtable

前面我们知道了，Redis的KV结构是通过dictEntry来实现的。在hashtable中，又对dictEntry进行了多层封装。

```c
typedef struct dictht {
    // 两个哈希表
    dictEntry **table;
    // 哈希表的大小
    unsigned long size;
    // 哈希表大小掩码
    unsigned long sizemask;
    // 哈希表中数据项数量
    unsigned long used;
} dictht;
```

dictEntry放在了dictht(hashtable)里面了

```c
typedef struct dict {
    // 哈希表的类型，包括哈希函数，比较函数，键值的内存释放函数
    dictType *type;
    // 存储一些额外的数据
    void *privdata;
    // 两个哈希表
    dictht ht[2];
    // 哈希表重置下标，指定的是哈希数组的数组下标
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */
    // 绑定到哈希表的迭代器个数
    int iterators; /* number of iterators currently running */
} dict;
```

dictht放在了dict里面了。

从源码可以看出，它是一个数组+链表的结构。如图：

![image-20210311003113646](/Users/vince/Library/Application Support/typora-user-images/image-20210311003113646.png)

我们发现dictht后面是个null，说明第二个hashtable没有数据。那么为什么要定义两个hashtable，其中一个不用呢？

答案是为了扩展哈希表！redis 为每个数据集配备两个哈希表，能在不中断服务的情况下扩展哈希表。平时哈希表扩展的做法是，为新的哈希表另外开辟一个空间，将原哈希表的数据重新计算哈希值，以移动到新哈希表。如果原哈希表数据过多，中间大量的计算过程较好费大量时间，这段时间 redis 将不能提供服务。



使用场景：和String一样