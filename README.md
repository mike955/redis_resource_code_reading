# Redis 源码阅读(5.0)

- [Redis 源码阅读(5.0)](#redis-源码阅读50)
  - [数据类型](#数据类型)
    - [数据类型](#数据类型-1)
      - [string](#string)
      - [list](#list)
      - [hash](#hash)
      - [set](#set)
      - [zset](#zset)
      - [stream](#stream)
    - [编码类型](#编码类型)
      - [sds](#sds)
      - [dict](#dict)
      - [adlist](#adlist)
      - [skiplist](#skiplist)
      - [ziplist](#ziplist)
      - [intset](#intset)
  - [持久化](#持久化)
    - [AOF](#aof)
    - [RDB](#rdb)
  - [内存管理](#内存管理)
    - [内存分配](#内存分配)
    - [内存回收](#内存回收)
    - [碎片整理](#碎片整理)
    - [内存淘汰(置换)策略](#内存淘汰置换策略)
  - [高可用](#高可用)
    - [replicate](#replicate)
    - [sentinel](#sentinel)
    - [cluster](#cluster)
  - [事件模型](#事件模型)
    - [事件循环初始化](#事件循环初始化)
    - [添加事件](#添加事件)
    - [执行事件](#执行事件)
  - [慢查询日志](#慢查询日志)
  - [rehash操作](#rehash操作)
## 数据类型

### 数据类型
 
 redis 提供的基本类型有 5 种，分别为：String、List、Hash、Set、Zset。redis 在对这 5 种类型进行操作时，会根据当时的情况选用不同的底层数据结构存储数据。对于 redis 中的每一组键值对数据，redis 会使用 sds 存储 key，使用 redisObject 存储 value；redisObject 数据结构如下:

 ```c
typedef struct redisObject {
    unsigned type:4;        // value 类型，有 5 种，OBJ_STRING、OBJ_LIST、OBJ_SET、OBJ_ZSET、OBJ_HASH
    unsigned encoding:4;    // 编码类型，底层数据类型，以 OBJ_ENCODING_ 开头定义的类型
    unsigned lru:LRU_BITS;  // 缓存淘汰使用，LRU 时为相对于全局 lru 的时间，LFU 时低 8 位为使用频率，高 16 位为访问时间
    int refcount;           // 引用计数
    void *ptr;              // 指针，指向底层数据结构的指针
} robj;
 ```

#### string

#### list

列表，按顺序存储放入的数据，实例上是链表。底层使用链表或压缩列表实现。

#### hash

#### set

#### zset

#### stream

### 编码类型

编码类型是 redis 提供的基本数据类型所使用的底层数据结构。redis 目前总共提供了 ？种编码类型，对数据类型的对应关系为：

- String: sds
- List: adlist、ziplist
- Hash: ziplist、dict
- Sorted Set: ziplist、skipList
- Set: intset、dict

#### sds

sds 是 simple dynamic string 的缩写，意味简单动态字符串。sds 是 redis 定义的一种动态字符串数据结构，redis 中存储的键值对数据的键均使用 sds 存储；sds 的定义在`src/sds.h`文件中，在该文件中，可以发现，redis 定义了 5 种 sds 数据结构。分别为 **sdshdr5、sdshdr8、sdshdr16、sdshdr32、sdshdr64**，使用时根据需要选择不同的数据类型，名称末尾的数字表示字符数组的长度。

以`sdshdr8`为例，数据结构如下:
```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;            // 字节数组长度，字符串长度，不包含空终止符\0
    uint8_t alloc;          // 分配的内存大小，不包头终止符和空终止符\0
    unsigned char flags;    // 字节数组属性，只使用了低 3 位
    char buf[];             // 字节数组，存储字符串数据，空终止符\0 会作为一个单独的字符串存储，这样可以直接重用一部分 c 字符串库函数
};
```
从上面定义可以发现，sds 类型包含四个字段(sdshdr5 已经不在被使用，除外)，其中`len`字段表示字符串长度，sds 存储字符串长度有如下好处：

  - 获取字符串长度时不再需要计算(c 语言获取字符串长度需要遍历字符串，直到遇到 \0 符号，复杂度为 O(n))
  - 进行字符串拷贝时，直接使用 alloc 与 len 的差值，就知道空间是否足够，不够直接扩展，减少溢出概率，加速复制
  - sds 通常会预分配比自身多的内存，使得字符串操作时减少内存分配操作，加快操作速率，带来的影响是提高了内存的使用，是一个空间换时间的操作

由于 sds 存储了字符串长度，是的其可以不用使用 \0 符号作为字符串终止符，让字符串在 redis 中变为了一种安全类型，观察 sds 数据结构可以发现，sds 将数据存储为字符数组，带来的好处是不仅可以存储字符，还可以存储二进制数据，这使得 redis 存储类似于图片这样的二进制数据变得格外容易。

sds 结构体定义时通过添加`__attribute__ ((__packed__))`信息告诉编译器在编译时取消对该结构体的字节对齐，节省了空间，编译器会默认对不足 8 字节的数据分配 8 个字节。


#### dict

字典是 redis 用来数据库底层实现，也是哈希哈希键的底层实现。redis 每一个数据库包含一个字典，redis 中字典数据结果如下：

```c
typedef struct dict {
    dictType *type;           // 一组操作特定 key 的函数
    void *privdata;           // 私有数据，保存传递给 type 特定函数的参数
    dictht ht[2];             // 两哈希表，在进行 rehash 的时候用于拷贝
    long rehashidx;           // 是否在进行 rehash，-1 表示未在进行
    unsigned long iterators;  // 当前运行的迭代器数量
} dict;
```

每一个字典包含两个哈希表 ht[0] 和 ht[1], 正常情况下只会使用到一个哈希表 ht[0]，只有在进行 rehash 操作时才会用到 ht[1]。`rehashidx`字段表示当前 rehash 进度，-1 表示未进行 rehash 操作；dictType 表示一组操作函数，在进行键值操作时使用，每个函数的参数可能不一样，需要函数自己从参数中获取；privdata 表示私有数据，用于传递给 dictType 中函数。

哈希表数据结构如下：

```c
typedef struct dictht {
    dictEntry **table;        // 哈希表数组，数组中的每个元素元素都是一个指向 dictEntry 的指针
    unsigned long size;       // 当前哈希表大小
    unsigned long sizemask;   // 哈希表大小掩码，等于 size-1，计算键索引时用到
    unsigned long used;       // 当前字典中存储的 key 数量
} dictht;
```

每一个哈希表用户存放实际数据，实际数据会存储在 table 数组中，redis 根据哈希函数(hashFunction)计算每一个 key 的哈希值 h，然后将哈希值与 sizemark 字段进行 &  操作，得到 key 在哈希表中的索引，将数据存储到哈希表中的位置。这样对于任何数据的查找，时间复杂度都是 O(1)。

size 字段表示哈希表的大小，used 字段表示当前哈希表已经存储的 key 的数量。used/size 表示当前哈希表的负载，如果负载大于 1，表示需要进行 rehash 操作，扩容哈希表。

对于哈希表中的每个键值对，数据结构为:

```c
typedef struct dictEntry {
    void *key;          // 键
    union {             // 值，可以为 uint64_t、int64_t、 double，或者 val
        void *val;      // 实际上为 redisObject 类型
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;  // 当哈希冲突时使用，指向下一个键值对，可以发现，redis 解决哈希冲突的方法是链式哈希
} dictEntry;
```

redis 使用 dictEntry 表示每个键值对实体，如果值为复杂类型时，会用`src/server.h`文件中的`redisObject`数据结构来表示值。

#### adlist

list 是 redis 实现的链表数据结构，定义在`src/adlist.h`文件中，链表结构如下:

```c
typedef struct list {
    listNode *head;                     // 链表头元素
    listNode *tail;                     // 链表尾元素
    void *(*dup)(void *ptr);            // 链表节点复制函数
    void (*free)(void *ptr);            // 链表节点释放函数
    int (*match)(void *ptr, void *key); // 链表节点值对比函数
    unsigned long len;                  // 链表长度
} list;

typedef struct listNode {       // list 链表中的每个节点
    struct listNode *prev;      // 前一个链表节点
    struct listNode *next;      // 下一个链表节点
    void *value;                // 当前链表值
} listNode;
```

#### skiplist

#### ziplist
压缩列表是列表列表键和哈希键的底层实现。当一个列表项只包含少量数据，并且每一项都是小整数或者短字符串时，会使用压缩列表作为列表底层实现；当哈希表只包含少量键值对，且每个键值对的健和值要么是小整数，要么时段字符串时，使用压缩列表作为哈希表底层实现。

#### intset

整数集合是集合数据的底层实现，在集合只包含整数，且元素不多时，会使用该数据结构作为底层实现。

intset 数据结果如下:
```c
typedef struct intset {
    uint32_t encoding;  // 编码方式，有三种 int16、int32、int64,
    uint32_t length;    // 长度
    int8_t contents[];  // 集合中元素，值按大小排序，不包含重复元素
} intset;
```

intset 包含三个字段，`encoding`字段表示整数集合数据编码方式，`length`表示集合中元素个数；`contents`表示集合中的元素，元素按从小到大排序，不包含重复元素。向集合中添加数据时会检查数据是否存在，同时将数据按从小到大的顺序插入。

虽然`content`的类型为`int8_t`，但是`content`中的类型并不都是`int8_t`，`content`中存储的类型取决于`encoding`字段。这一点可以在向集合中插入数据时进行验证，`intsetAdd`向集合中插入数据时，会获取当前要插入数据的类型 valenc 和集合的`encoding`字段，将两者进行比较，如果 valenc 大于 encoding，会调用`intsetUpgradeAndAdd`对集合进行升级，然后插入数据。获取集合中数据时(`src/intset.c/intsetGet`)，也会通过`encoding`方法来计算数据的位置。

以`sadd`命令为例，看一下 redis 在什么情况下会使用 intset 存储集合数据。对于`sadd`明天，redis 会调用`saddCommand`(src/_t_set.c)方法进行处理，通过`lookupKeyWrite`方法判断 key 是否存在，如果不存在，会调用`setTypeCreate`创建一个集合，`setTypeCreate`会使用`isSdsRepresentableAsLongLong`检查用户传入的值，判断使用 intset 还是哈希表创建集合，分析代码，可以发现，当集合的值全部为整数时会调用`createIntsetObject`创建一个整数集合，否则调用`createSetObject`创建一个哈希表；创建哈希表后，会调用`setTypeAdd`函数向哈希表中添加数据，会通过`intsetLen`函数检查获取集合元素数量，如果当前集合的元素是否超过设置的值(默认为 512)，如果超过，将集合底层数据结构由整数集合转换为字典。

## 持久化

redis 数据持久化分为两个类别，分别为 AOF 和 RDB，AOF 记录的是执行命令，RDB 记录的是数据。

### AOF

与 aof 配置相关的参数有：
    - appendonly: yes/no, 是否开启 aof 持久化, 默认不开启
    - appendfilename: aof 文件名称，默认为 appendonly.aof
    - appendfsync: 将 aof 文件刷到磁盘
      - always: 每次跟新操作均落盘
      - everysec: 每秒钟刷盘一次
      - no: 操作系统决定何时将数据刷到磁盘
    - no-appendfsync-on-rewrite: 重写 aof 文件时，是否执行 aof 数据落盘操作，默认为 no，可能会导致 redis 进程阻塞
    - auto-aof-rewrite-percentage: 当前 aof 文件大小达到上次 aof 文件的百分之多少时，开启 aof 重写，单位为 %
    - auto-aof-rewirte-min-size: 进行 aof 重写时最小文件大小
    - aof-load-truncated: yes/no， 加载 aof 文件时，文件不完整是否退出，默认为 yes
    - aof-use-rdb-preamble: 是否开启混合持久化，默认为 true

redis 初始化时会初始化 aof_buf 变量为空的动态字符串，用户保存 aof 数据。如果用户设置了`appendonly yes`，redis 初始化时会调用`startAppendOnly`方法启动 aof 持久化。初始化 aof 时会判断当前是否在进行 aof 重写，如果在重写等在重写完成。 

### RDB

rdb 持久化与 aof 不同地方在于，aof 记录的是指令，rdb 记录的是数据，aof 根据时间刷盘，dbs 根据操作频率刷盘。

dbs 既可以手动执行也可以根据配置自动执行，手动执行 rdb 持久化的命令为`save`和`bgsave`，rdb 持久化生产的是一个经过压缩的二进制文件，redis 启动时可以根据该文件初始化数据。

redis 将 rdb 持久化参数保存在`saveparams`参数中，`saveparams`包含两个参数，一个记录时间，一个记录修改次数。redis 默认会使用`appendServerSaveParams`添加三个 rdb 参数，分别为 1 小时至少 1 次修改、5 分钟至少 100 次修改、1 分钟至少 1000 次修改。只要这三个参数满足一个，redis 就会生成 rdb 持久化文件。

在 redis 定义任务事件方法中，我们发现了针对 rdb 持久化的相关逻辑。首先通过`saveparamslen`检查 rdb 自动持久化参数是否为空，如果不为空检查当前服务的上次持久化到现在的修改次数`dirty`和时间`unixtime`是否大于设置的 rdb 参数，如果符合条件，调用`rdbSaveBackground`开始 rdb 持久化，自动持化话会在后台进行，不阻塞当前事件循环。

对于手动持计划命令`save`和`bgsave`，通过查看对应的执行函数`saveCommand`和`bgsaveCommand`可以发现，`save`是一个阻塞操作；`bgsave`是一个非阻塞操作。

无论是非阻塞操作还是阻塞操作，最后都是调用`/src/rdb.c`文件中的`rdbSave`方法，该方法会先创建一个文件，然后调用`rdbSaveRio`准备数据，进入写数据，最后重置`dirty`参数为 0.`rdbSaveRio`函数详细描述了写入到 rdb 文件的数据，首先写入 "REDIS" 字符，然后写入当前 rdb 版本号，然后再写入数据，最后写入 crc64 校验和。

自动 rdb 持久化和`bgsave`主动持久化都是一个异步操作，不会阻塞当前事件循环，这两个操作最后都是调用`rdbSaveBackgroud`函数来进行一步持久化。持久化的主要流程为：

    - 调用`fork`函数创建一个子进程
    - 在子进程中调用`rdbSave`来生成 rdb 文件，由于创建的子进程在 fork 时享有和父进程完全相同的内存数据
    - 父进程记录相关数据后返回，不阻塞子进程

由于 fork 操作创建的子进程享有和父进程完全相同的内存数据，因此子进程持久化使用到的数据就是父进程内存中需要持久化的数据，不用重新拷贝一份数据，节省了空间。父子进程的虚拟内存指向的是同一片物理内存，当夫进程内存数据被修改时，操作系统会为修改的页重新拷贝一份供子进程使用，父进程继续使用老的物理页。这就是**copy-on-write**，写时复制。

## 内存管理

redis 是一个内存密集型服务，对内存的需求很对，在某些地方使用了空间换时间的方式(如为字符串分配大于其自身的内存)也加大了其对内存的需求。redis 提供了相关参数来对使用内存进行设置。这里我们主要讨论四个方面：内存分配、内存回收、碎片整理、内存淘汰策略

### 内存分配

redis 没有自己实现内存池，因此选用了第三方库来进行内存分配工作，redis 支持三种 tcmalloc、 jemalloc 、libc 三种内存分配器，具体的内存使用方式根据宏定义来决定。其中 jemalloc 被包含在在源码中(/deps/jemalloc)，libc 是库函数。使用 tcmalloc 需要自行下载安装。

### 内存回收

redis 使用引用计算的方式来进行内存回收，redis 在每个对象结构体中定义了一个字段`refcount`，用来记录该对象是否被使用，当该对象的值为 0 时，表示该对象不会再被使用，内存可以被回收。

对于设置了过期时间的 key，redis 会使用**惰性检查**和**主动检查**两种方式来判断是否过期：
    - 惰性删除：每次获取 key 时，会检查是否已过期
    - 定期删除：在时间任务中执行，默认的策略为缓慢删除(ACTIVE_EXPIRE_CYCLE_SLOW)，该策略会遍历数据库(默认为 16，CRON_DBS_PER_CALL)，遍历设置了过期时间的key，迭代步数为16(iteration & 0xf)，这样可以缩短遍历时间，减少阻塞时间，如果抽样的 key 中有 25% 的key 达到过期时间，会则继续下次抽样，否则结束定时删除过期key，下个周期在进行。

### 碎片整理

redis 提供了碎片整理的相关参数，主要有：
    - activedefrag: 是否开启碎片整理，默认不开启
    - active-defrag-ignore-bytes: 碎片大小超过多少开始整理，默认为 100MB
    - active-defrag-threshold-lower: 碎片率超过多少开始整理，默认 10%
    - active-defrag-threshold-upper: 最大碎片百分比，默认 100%
    - active-defrag-cycle-min: 碎片整理使用最小 cpu 百分比，默认为 5% 
    - active-defrag-cycle-max: 碎片整理使用最大 cpu 百分比，默认为 75%
    - active-defrag-max-scan-fields: 碎片处理最大扫描 key 数量

碎片整理在定时事件中执行，会阻塞事件循环，函数为`activeDefragCycle`，该函数会每次运行 1 秒钟，然后检查是否满足碎片整理条件。

### 内存淘汰(置换)策略

当 reids 使用的内存到达最大设定值后，如果还需要内存来继续存放数据，会根据设定的淘汰策略来删除相关 key，目前有 6 种淘汰策略：
    - volatile-lru: 在设置了过期时间的 key 中使用 lru 算法进行淘汰
    - allkeys-lru: 在所有 key 中使用 lru 算法进行淘汰
    - volatile-lfu: 在设置了过期时间的 key 中使用 lfu 算法进行淘汰
    - allkeys-lfu: 在所有 key 中使用 lfu 算法进行淘汰
    - volatile-random: 从设置了过期时间的 key 中随机选择一个进行淘汰
    - allkeys-random: 从所有的 key 中随机选择一个 key 进行淘汰
    - volatile-ttl: 在设置了过期时间 key 的中选择过期时间最早的进行淘汰
    - noeviction: 不淘汰，对于写请求，直接返回错误

redis 中的 lru (Least Recently Used) 算法不是一种精确的 lru 算法，因为精确的 lru 算法需要使用到更多的内存，lru 淘汰的逻辑为：在集合中选择最近访问次数最少的 key 进行淘汰。

lfu (Least Frequently Used) 是 redis4.0 引入的一种新的内存淘汰策略，这种算法为了应对一种 lru 不太适应的情况：访问频率不高，但最近被访问过。lfu 淘汰的逻辑为：最近最长使用，redis 会维护一个激素及，记录当前 key 访问次数与时间的关系，判断随着之间的推移，key 的访问频率。

redis 在执行命令对应的函数时，会检查内存是否超出，如果超出，会调用`freeMemoryIfNeededAndSafe`方法进行判断，如果满足淘汰策略，会调用相关函数淘汰 key。

## 高可用

redis 提供里三种多机方案，分别为replicate、senetinel、cluster。其中有两种可以作为高可用方案，一种是 Sentinel，一种是 Cluster.

### replicate

复制是 redis 提供的一种基本多机方案，一个 redis 实例(从节点)可以复制另一个 redis 实例(主节点)的数据，作为其从节点，当主节点宕机时，使用从节点提供服务。这样方案需要手动切换。

在从节点上执行 `lavof masterHost masterPort`命令后，从节点将主节点信息存储到相应字段，同时连接到主服务器，在主服务器看来，从服务器实际上是一个客户端，从服务器向主服务器发送指令，接收返回的数据。

集群模式(cluster)不支持复制。

### sentinel

### cluster

## 事件模型

redis 的事件驱动模型是其高性能的关键。redis 会在初始化时创建一个事件循环，事件循环中包含已经被初始化的事件，服务启动后，就是轮训事件循环，直到退出。在`/src/server.c`文件的`main`函数中，redis 进行相关初始化工作后会调用`aeMain`方法来执行事件循环，处理事件任务。

### 事件循环初始化

redis 事件的初始化在`/src/server.c`文件的`initServer`函数中完成，事件初始化代码如下：

```c
server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR); // 设置时间循环大小，默认为 客户端最大连接数+128

...

aeEventLoop *aeCreateEventLoop(int setsize) {
    aeEventLoop *eventLoop;
    int i;

    if ((eventLoop = zmalloc(sizeof(*eventLoop))) == NULL) goto err;
    eventLoop->events = zmalloc(sizeof(aeFileEvent)*setsize);
    eventLoop->fired = zmalloc(sizeof(aeFiredEvent)*setsize);
    if (eventLoop->events == NULL || eventLoop->fired == NULL) goto err;
    eventLoop->setsize = setsize;
    eventLoop->lastTime = time(NULL);
    eventLoop->timeEventHead = NULL;
    eventLoop->timeEventNextId = 0;
    eventLoop->stop = 0;
    eventLoop->maxfd = -1;
    eventLoop->beforesleep = NULL;
    eventLoop->aftersleep = NULL;
    if (aeApiCreate(eventLoop) == -1) goto err;
    /* Events with mask == AE_NONE are not set. So let's initialize the
     * vector with it. */
    for (i = 0; i < setsize; i++)
        eventLoop->events[i].mask = AE_NONE;    // 初始化事件循环为空
    return eventLoop;

err:
    if (eventLoop) {
        zfree(eventLoop->events);
        zfree(eventLoop->fired);
        zfree(eventLoop);
    }
    return NULL;
}
```

根据事件初始化代码发现，redis 在初始化时会先创建事件，数目为 配置的最大客户端连接数+128，之所以是 128 是因为 redis 自身保留使用 32 个事件，另外 96 个事件为 buffer，确保事件数量够使用。事件数量会在服务启动前被确定，服务启动后不会再修改。

redis 调用`aeApiCreate`初始化事件循环，分析代码会发现，`aeApiCreate` 实际上是创建了一个**多路复用**，具体的多路复用根据 redis 编译的环境不同而不同，redis 封装了 evport、epoll、kqueue、select 四种多路复用模型，在`/src/ae.c`文件中会根据宏定义来决定使用哪种多路复用模型。redis 将四种多路复用模型使用相同的函数名进行封装，通过在事件循环结构体的`apidate`中填入不同的参数来供不同的模型使用。可以说封装的非常巧妙。

### 添加事件

redis 初始化事件循环后，事件循环中的所有事件为空，需要添加有效事件。同样是在`/src/server.c`文件的`initServer`函数中，redis 创建相关有效事件,主要代码如下:

```c
// 创建时间事件，用于在后台处理一下相关工作，如客户端操作、过期 key、碎片整理 等工作
if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {            
    serverPanic("Can't create event loop timers.");
    exit(1);
}

...

for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }
if (server.sofd > 0 && aeCreateFileEvent(server.el,server.sofd,AE_READABLE,
    acceptUnixHandler,NULL) == AE_ERR) serverPanic("Unrecoverable error creating server.sofd file event.");

// 创建一个可读的文件事件，在一个客户端阻塞时
if (aeCreateFileEvent(server.el, server.module_blocked_pipe[0], AE_READABLE,
    moduleBlockedClientPipeReadable,NULL) == AE_ERR) {
        serverPanic(
            "Error registering the readable event for the module "
            "blocked clients subsystem.");

aeSetBeforeSleepProc(server.el,beforeSleep);
aeSetAfterSleepProc(server.el,afterSleep);
}
```

通过创建事件的代码可以发现，在 redis 中存在两类事件：**时间事件和文件事件**。

时间事件其实是一个定时任务事件，定时执行相关操作，如果过期 key 检查、碎片整理等任务，具体的定期任务可以通过其注册的时间事件函数`serverCron`来发现。

文件事件更多的像是一个网络 I/O 事件处理器。以 TCP 启动方式为例，针对每一个 tcpfd，redis 会调用`aeCreateFileEvent` 创建一个文件事件，其处理函数为`acceptTcpHandler`, `aeCreateFileEvent`方法会调用`aeApiAddEvent`将事件放入多路复用，当有事件触发时，调用对应的函数进行处理。

`acceptTcpHandler`最后会调用`acceptCommonHandler`对接收到的 tcp 连接进行处理，调用`createClient`创建客户端对象，如果创建成功，会在事件循环中添加一个该客户端的可读性事件`readQueryFromClient`，即当该 tcp 接收到数据时，调用`readQueryFromClient`方法处理接收到的数据。

当需要响应客户端请求时，会向事件循环注册一个写事件，处理函数为`sendReplyToClient`，该函数会将需要返回给客户端的数据返回给客户端。

redis 还在事件循环中注册了两个钩子事件，分别为`beforeSleep`和`afterSleep`，beforeSleep 用于在每次进入事件前调用，afterSleep 用于在多路复用事件执行后调用。

### 执行事件

在初始化事件完成后，redis 会在`main`函数最后调用`aeMain`函数，用于处理事件循环，`aeMain`函数代码如下：

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```

`aeMain`会不断执行事件循环，每次调用时会判断`beforesleep`方法是否为空，如果是会先执行该事件，然后调用`aeProcessEvents`执行事件。

## 慢查询日志

redis 的慢查询日志使用列表(实际上是链表)数据结构存储日志，与慢查询相关的参数有两个：
    - slowlog_log_slower_than: 慢查询日志记录的查询时间阀值，超过该值的查询会记录到慢查询列表，默认值为 10000 微秒
    - slowlog_max_len: 最大慢查询记录数量，默认值为 1000，超过改数量会丢弃最前面的记录

慢查询日志实体数据结构定义在`slowlog.h`文件中，每条慢查询日志记录了如下数据:
    - 查询命令
    - 查询命令参数数量
    - 当前日志 id
    - 查询话费时间，微秒
    - 查询执行的时间点
    - 发送查询的客户端名称
    - 发送查询的客户端网络地址

慢查询日志保存在内存中，重复服务数据会丢失

慢查询日志命令有:
    - `slowlog get [count]`: 查询最近慢查询日志详情
    - `slowlog len`: 当前慢查询日志条数
    - `slow reset`: 清空慢查询日志

## rehash操作

但哈希表中的数过多时，造成哈希冲突的概率会增加，进而对查找性能造成影响。redis 在某些条件下会对哈希表进行 rehash 操作，扩容或缩小哈希表，提高查找效率。

扩容哈希表，在增加 key 时，`_dictKeyIndex` 函数中的`_dictExpandIfNeeded`函数会是否需要进行 rehash 进行判断，进行 rehash 的条件为：

    - 如果当前哈希表大小为 0，扩容哈希表大小为初始大小 4
    - 如果哈希表负载因子大于 1(哈希表的 used 字段大于等于 size 字段)，且无后台 copy-on-write 进程执行(bgsave/bgrewriteaof)
    - 如果哈希表负载因子大于 1(哈希表的 used 字段大于等于 size 字段)，且哈希表的负载因子大于 5

扩容哈希表的函数为`dictExpand`，扩容后的哈希表大小为第一个大于等于 size 的 2 的 N 次方的数字，size 为旧的哈希表中 key 数量(used*2) 的两倍。计算新的哈希表大小后，初始化新的哈希表 ht[1], 并 redis 字典中国中 rehashidx 为 0，表示进行 rehash 中。

rehash 时
    - 添加 key 时，数据插入到  ht[1] 中
    - 查找 key 时，
    - 删除 key 时，

redis 的 rehash 不是一次性的, 而是采用渐进式的方式进行，serverCron(后台每秒钟执行一次) 中会执行 databasesCron 函数，databasesCron 函数会在后台进行过期 key 检查、碎片整理、修改字典大小、reahsh 等操作。

rehash 操作时会对字典中的数据哈希表和过期哈希表分别进行处理，执行 rehash 的函数为`dictRehash`，主要逻辑时从旧的哈希表中取出数据，然后重新计算 key，存储到新的哈希表中，将字典中的 rehashidx 值修改为当前 rehash 进度。

当 rehash 完成后，将 ht[1]的内容复制到ht[0]，清空 ht[1]，使得每次 rehash 时，都是从 ht[0] rehash 数据到 ht[1]。

哈希表缩容，当哈希表的负载因子小于 10% 时，会对哈希表进行缩容，判断条件在`htNeedsResize`函数中，缩容后的哈希表大小为第一个大于等于 used 字段的 2 的 N 次方的数字