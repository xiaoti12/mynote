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
2. 重新计算k-v对索引值，迁移到新的哈希表中
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

> 这里的随机化服从一定的统计规律

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

在6.0版本后，会使用多个IO线程来处理网络请求（命令执行仍然是单线程）。

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
