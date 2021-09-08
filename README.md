# Redis 源码阅读(5.0)

## 数据类型

### 用户数据类型

#### string

#### list

#### hash

#### set

#### zset

#### stream

### 编码数据类型

#### sds

#### dict

#### adlist

adlist 是一个链表，虽然名称是列表，但实际上是链表，数据结构为:

```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;
```

#### ziplist

#### intset

#### quicklist


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