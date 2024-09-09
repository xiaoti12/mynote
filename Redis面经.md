# 线程模型

Redis总体上是**单线程**模型。具体来说，请求的接收、解析、数据读写、返回结果这个过程是由主线程来进行的。

## 单线程原因

由于redis是内存数据库，CPU并不是其性能瓶颈，更多情况是内存大小和网络IO限制。使用多线程后，可能额外引入并发读写、线程切换、死锁等问题

## 多线程模式

### 后台线程

redis目前一共有3个后台线程，用于执行三个任务：

- 关闭文件：从任务队列中关闭文件
- AOF刷盘：AOF配置为everysec时，会定时执行
- Lazy free：释放不用对象

### 网络多线程

在6.0版本后，会使用**多个IO线程**来处理网络请求（命令执行仍然是单线程）。

# 持久化

三种方法：AOF日志、RDB快照、混合持久

## AOF日志

执行写操作命令后，将该命令追加到日志文件中。当redis重启后，重复执行日志命令进行数据恢复。

日志的写入过程：命令写入server的缓冲区—>拷贝到内核缓冲区page cache—>写入硬盘

redis提供了三种从page cache写入硬盘的策略：

- always：每次写操作都会实时写入硬盘，可靠性高、性能开销大
- everysec：每秒将内核缓冲区内容写入硬盘，**默认**机制
- no：由操作系统决定写入实际，性能最好，可靠性最低。

## RDB快照

记录某个时刻内存中的数据，恢复时可以直接读入内存，而不用按命令逐步恢复。

RDB提供了两个命令主动生成rdb快照文件：

- save：在主线程阻塞备份
- bgsave：创建子线程生成rdb文件

此外，还可以配置自动执行bgsave命令：在M秒内对数据至少进行了N次修改后执行命令。

bgsave时使用了**写时复制技术**，保证主线程可以继续修改数据。具体来说，bgsave命令创建的子线程和父进程共享同一片内存（复制父进程页表）。而当父进程修改数据时，会在内存中复制一份并修改页表对应项。

![img](https://cdn.xiaolincoding.com//mysql/other/ebd620db8a1af66fbeb8f4d4ef6adc68-20230309232308604.png)

## 混合持久化

RDB恢复速度快，但不好确定快照的保存频率；AOF丢失数据少，但数据恢复慢。

Redis 4.0后引入混合持久化机制。过程如下：

- 运行子线程，将内存数据以RDB方式写入AOF文件
- 开始写入RDB的过程后，将主线程运行命令放入缓冲区
- RDB写入完成后，将增量命令以AOF方式写入AOF文件

也就是说，此时AOF文件包含两部分：RDB内存数据 + AOF命令数据

这种方法一定程度上解决了RDB保存期间空窗时间导致的数据丢失问题。

# 缓存雪崩/击穿/穿透

![图片](https://cdn.xiaolincoding.com//mysql/other/061e2c04e0ebca3425dd75dd035b6b7b.png)

## 雪崩

大量缓存集体过期/Redis崩溃，请求全部转向数据库

### 解决

缓存过期：

1. 过期时间加上随机数
2. 互斥锁，保证同时只有一个进程更新缓存
3. 不设置过期，后台进程更新缓存，保证缓存永久有效

Redis故障：

1. 熔断机制/请求限流
2. 构建Redis集群

## 击穿

频繁被访问的热点数据过期，高并发请求被转向数据库。*可以认为是雪崩的子集*

### 解决

1. 互斥锁
2. 不设置过期，后台进程更新缓存

## 穿透

数据既不在缓存中，也不在数据库中。此时无法通过恢复缓存来缓解数据库压力

### 原因

1. 业务误操作
2. 恶意DDOS攻击

### 解决

1. 在controller/service层判断请求参数合法性、字段是否存在
2. 针对查询数据，在缓存中设置空值/默认值
3. 使用**布隆过滤器**判断数据是否存在

> 布隆过滤器使用若干个哈希函数、对应位置1，可能存在误判（不存在的查询成了存在），但不会存在漏判（查询到不存在的，一定不存在）

# 分布式锁
Redis可以设置键来实现分布式锁。实现方式：使用`SETNX`命令（SET if Not Exists）尝试设置一个键，只有在键不存在时设置成功，表示获得锁。同时，通过`EXPIRE`命令设置键的过期时间，防止死锁。需要注意的是，要保证`SETNX`和 `EXPIRE`命令是原子性的。

此外，分布式锁还有如下实现方式：
- 在数据库中通过表记录锁，锁抢占；但性能较低
- 在zookeeper中创建瞬时节点，创建成功则获得锁

# 主从复制

主从复制主要是保证数据的高可用性，尽量不丢失数据。其他作用：

1. 故障恢复：当主节点宕机，其他节点依然可以提供服务；
2. 负载均衡：主节点提供写服务，从节点提供读服务，分担压力；
3. 高可用基石：是哨兵和 cluster 实施的基础

主次复制中一致性的保证：**读写分离**，即写数据全部由主服务器进行，之后同步给从服务器；读数据可以分布到从服务器

## 同步过程

分为三个步骤：建立链接、初始同步、追加同步

### 建立链接

在从服务器上执行`replicateof`命令，确定主服务器的IP。之后从服务器发送`psync`命令，表示进行数据同步

### 初始同步

主服务器执行`bgsave`生成RDB文件，发送给从服务器。生成RDB文件的同时（包括从服务器去确认前），将接受到的写操作命令写入`replication buffer`。这种方式为**全量同步**

### 追加同步

从服务器接受到RDB文件、载入内存后，会回复ACK，此时主服务器发送`replication buffer`，即追加的命令。

之后，主从服务器之间维护TCP长连接，主服务器发送写命令给从服务器。

## 增量同步

当从服务器断连、再次连接上主服务器时，会向主服务器中传输自己的**offset**，主服务器根据双方**offset**的差异，进行下一步操作。这里的offset，指的是写命令在环形缓冲区`backlog_buffer`中的偏移。

- 差异的数据在缓冲区中，就进行增量同步，即传输这部分命令
- 差异的数据已经不存在（被覆盖），则使用全量同步

![图片](https://cdn.xiaolincoding.com//mysql/other/2db4831516b9a8b79f833cf0593c1f12.png)

# 内存管理
## 内存淘汰
### 淘汰策略
3.0版本之后默认的淘汰策略：noeviction，当运行内存超过最大设置内存后，不淘汰数据；如果有新的写入报错。其他策略：

- random：随机淘汰有过期时间的kv值
- ttl：优先淘汰更早过期的kv值
- lru：优先淘汰最久未使用的kv值
- lfu：优先淘汰最少使用的键值

### LRU算法
LRU算法一般需要使用链表，但Redis没有使用，因为会引入额外空间开销和大量的链表移动

Redis的实现方式：近似LRU算法，在对象结构体中添加**最后访问时间**字段。当需要淘汰时，随机取N（默认5）个值，淘汰其中最久没使用的值

## 过期删除
### 数据结构
Redis会维护过期字典，存放key和过期时间
```c
typedef struct redisDb {
    dict *dict;    /* 数据库键空间，存放着所有的键值对 */
    dict *expires; /* 键的过期时间 */
    ....
} redisDb;
```

### 过期策略
常见的三种过期删除策略：
- 定时删除：设置过期时间时创建定时时间，到期自动删除，内存友好、CPU不友好
- 惰性删除：访问key时，检查到key已过期则删除，CPU友好、内存不友好
- 定期删除策略：每隔一段时间随机从过期字典取一些key检查，删除其中过期key

Redis采用的策略：惰性删除+定期删除。惰性删除时可以配置同步或异步删除；定期删除每秒10次、每次随机抽取20个key（写死）

# 数据一致性

## 先数据库后缓存（x）

对于一个写/更新数据的请求，当先更新数据库、再更新缓存时，可能出现如下情况：

1. 写入数据库失败，缓存过期后数据丢失
2. 请求并发时，请求B1/2插在请求A1和请求A2之间，导致数据不一致

![image.png](https://s2.loli.net/2024/03/21/jlzxZsObr7U3ugH.png)

## 先缓存后数据库（x）

与上同理，当请求B1/2插在请求A1和请求A2之间，导致数据不一致

![image.png](https://s2.loli.net/2024/03/21/2q7ZDnzoyjGPlLR.png)

## 旁路缓存

策略：写/更新数据时，不写入/删除缓存数据，直接保存到数据库。等缓存未命中时，更新数据到缓存

![image.png](https://s2.loli.net/2024/03/22/GladoTvmp1ySBwL.png)

注意写策略中的顺序：先操作数据库，再删除缓存

> 也可以先删除缓存，更更新数据库。为了防止删除缓存后，其他请求未命中、读取更新旧数据到缓存，采取【**延迟双删**】的策略，也就是更新完数据后少许延迟、再次删除缓存

### 不一致情况

假如某次缓存未命中，①读取DB旧值、②回写缓存 两步中，更新数据库、删除缓存，会导致数据不一致、缓存中为旧值。但实际很难发生，因为②缓存写入很快，一般会比删除缓存更快完成

### 缓存删除失败

当写策略中，缓存删除失败时，会导致缓存过期前，数据库和缓存数据不一致。

解决方法（基于异步）：

1. 基于消息队列的重试：将要删除的数据放入消息队列，每次取出操作；失败则重新放入
2. 订阅数据库binlog：获取MySQL变更日志（binlog），提供给下游删除

# 底层数据类型

数据类型和数据结构之间的关系

![image.png](https://s2.loli.net/2023/12/03/mywqNkhajFoOtvE.png)

## Ziplist（压缩列表）

在3.2以前，是列表和Hashmap的底层实现之一。在3.2以后，是listpack的组成部分。

### 结构字段

![img](https://pdai.tech/images/db/redis/db-redis-ds-x-6.png)

- zlbytes：4字节，整个ziplist所占内存字节数
- zltail：4字节，ziplist最后一个entry的偏移量
- zllen：2字节，整个ziplist中entry的数量。如果超过2字节范围（65535），则需要实际遍历得到长度
- zlend：终止字符，值为`0xFF`。ziplist保证entry的首字节不会是`0xFF`

###  entry结构

一般结构：`prevlen`+`encoding`+`entry-data`。当存储int型时，不使用`entry-data`

- prevlen：前一个entry的长度，向前遍历时使用
    - 长度<254时，prevlen为1字节
    - 长度>=254时，prevlen为5个字节：`0xFE`+实际长度
- encoding：根据data类型为string还是int，以及数据长度表示
    - 前两位为11：表示存储int，后跟的字节表示int的值
    - 前两位为00/01/10：表示存储string，后跟的字节/比特表示string长度

## Quicklist

ziplist需要申请连续内存，会加速内存碎片化。为了解决这一问题，3.2版本后引入了quicklist，是一种以ziplist为节点的双向链表结构。

![img](https://pdai.tech/images/db/redis/db-redis-ds-x-4.png)

### 结构字段

对于quicklist整体结构，包含以下重要字段：

```c
typedef struct quicklist {
    quicklistNode *head; // 指向ziplist链表的头节点
    quicklistNode *tail; // 指向ziplist链表的尾节点
    unsigned long count;        //所有listNode（即ziplist）中entry数量总和
    unsigned long len;          //listNode的数量总和
    //...etc...
} quicklist;
```

对于每个quicklistNode，包含以下重要字段：

```c
typedef struct quicklistNode { 
    struct quicklistNode *prev; 
    struct quicklistNode *next;
    unsigned char *zl; // 指向所保存数据的结构
    unsigned int sz;             // ziplist（被压缩前）的字节数 
    unsigned int count : 16;     // ziplist中数据项个数 
    unsigned int encoding : 2;   // ziplist是否被压缩 
    // ...etc... 
} quicklistNode;
```

## 哈希表

采用链地址法。

![img](https://pdai.tech/images/db/redis/db-redis-ds-x-13.png)

### 结构字段

- table：指向哈希表数组
- size：数组大小
- sizemask：大小掩码，值为size-1
- used：已经存入的节点数量

### 哈希表结构

每个表项为`*dictEntry`。包括：key、value、*next。

哈希表为*dictEntry数组。对于每个key，会计算其hash，并取其哈希表的索引（`hashval & sizesmask`）。如果存在哈希冲突，则将最后一个dictEntry项的next指针指向新的entry。

### 扩容和收缩

当哈希表保存的键值对太多或太少时，会进行rehash操作。下面是扩容的步骤：

1. 创建一个新的哈希表，空间大小为原哈希表已使用空间（used字段）的一倍
2. 使用**渐进式**扩容。当对key进行查询、删除等操作时，再重新计算k-v对索引值，迁移到新的哈希表中
3. 全部迁移完毕后，释放原哈希表内存

触发扩容条件（任意满足）：

- 正在执行`BGSAVE`命令或者`BGREWRITEAOF`命令，负载因子>=5
- 没有执行`BGSAVE`命令或者`BGREWRITEAOF`命令，负载因子>=1

其中负载因子 = 已保存节点数量 / 哈希表大小（即 used / size）

## 跳表

跳表是有序集合的底层实现之一（另一个是ziplist），本质可以理解为多层索引的链表。相比于一般的链表，查找复杂度从`O(N)`降低到了`O(logN)`，是一种空间换时间的结构。

![Redis 数据结构skiplist - Redis 源码日志- UDN开源文档](https://doc.yonyoucloud.com/doc/wiki/project/redis/images/redis11.png)

### 结构字段

```c
typedef struct zskiplist {
    struct zskiplistNode *header, *tail; //header指向最高层索引的header
    unsigned long length; //节点总数
    int level; //总层数
} zskiplist
```

### 节点字段

```c
typedef struct zskiplistNode {
    robj *obj; // 内容
    double score; // 设置的有序值
    struct zskiplistNode *backward; //指向前一个节点
    struct zskiplistLevel {
        struct zskiplistNode *forward;//指向比自己score高的某个节点
        unsigned int span;//节点在第i层到其下一个节点需跳过的节点数，相邻时为1
    } level[]; //最多32个该结构
} zskiplistNode;
```

![img](https://pica.zhimg.com/70/v2-2cb8c8662d5610fa956211ff13889137_1440w.avis?source=172ae18b&biz_tag=Post)

> span字段可以计算查找节点时的跳跃程度，在`rank`等命令时可以使用

### 查找

每次查找会从最高级索引层出发。在当前索引层遍历时，如果下一个节点大于要查找的值，则下降一层索引层（`zskiplistLevel`数组里的下一个），重复该步骤；否则移动到下一个节点。

### 插入

跳表不要求上下两层索引节点的个数有严格对应关系，而是为每个节点随机化一个层数（即随机化`zskiplistLevel`数组长度）。在查找到节点对应的位置时，修改每一层中节点前后的指针。

> 这里的随机化服从幂次定律：越大的层数出现概率越小

### 与树形结构区别

跳表和平衡树（AVL、红黑树等）的查找复杂度均为`O(logN)`。在redis中，使用跳表有以下好处：

- **实现方式更简单**，平衡树需要实现左右旋
- 可以按区间遍历元素，链表的遍历比树更简单
- 在平衡树上插入和删除操作逻辑复杂

跳表的内存占用会多，但不明显（仍然为`O(N)`级）。

相比与MySQL中的B+树，两者有以下不同：

- B+树可以很矮，读取时磁盘IO次数少；而跳表的索引层可能很高（不过redis是内存数据库）
- B+数插入节点时可能会引发数据页的分裂、树结构调整

在读多写少的情况下，B+树的性能会优秀很多

