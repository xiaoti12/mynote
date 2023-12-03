# 基本架构

kafka包含以下基本概念：

- broker：kafka运行的一个节点单位
- topic：消息组织的单位，生产和消费消息都需要指定topic
- partition：一个topic会分为若干个partition（分区）
- producer：生产者，将消息推送到broker
- consumer：消费者，从broker处拉取消息
- consumer group：消费群，每个消费者需要指定group id，共同消费同一topic

<img src="https://www.cloudkarafka.com/img/blog/apache-kafka-partition.png" alt="Part 1: Apache Kafka for beginners - What is Apache Kafka? - CloudKarafka,  Apache Kafka Message streaming as a Service" style="zoom: 33%;" />

# 高性能

## PageCahe

kafka利用了操作系统的pagecache，当有读写操作时，先通过PageCache进行，减少磁盘访问。操作系统会定期异步将pagecache的数据固定到磁盘。

## 顺序写

由于磁盘的特性，顺序读写会比随机读写的速度快。而kafka将数据固定到磁盘后，是追加到末尾的方式，因此速度更快（*注：和数据库B+树不同的地方*）

## 零拷贝

在kafka将数据发送给消费者时，会把消息从disk或者pagecache读到内核区，此时不会复制到用户去，而是直接写入内核socket，避免内核区和用户区的切换

# 生产者运行流程

## 发送过程

1. 将消息封装为ProducerRecord对象
2. 序列化处理（默认或自定义）
3. 根据broker集群的元数据，将消息进行分区（也就是决定会发送到哪个主题的哪个分区）
4. 放入生产者的缓存区，封装成一个batch（默认大小为16KB）
5. sender线程将可以发送的batch发送到broker

## batch控制

batch涉及到的参数：

- batch.size：一个batch的大小，涉及到吞吐量和时延
- linger.ms：batch创建后被发送出去的最晚时间
- max.request.size：每次发送给broker的最大消息大小，需要和batch.size同步调整

# Topic、partition与consumer

## topic与partition

一个topic可以分为多个partition，并且在分布式场景下，同一topic的partition可以分布在不同的broker上。这样可以实现负载均衡，让不同的消费者并发消费，增强可扩展性。

## partition与consumer

一个基本的原则是：一个partition不会被多个consumer消费。因此，partition与（同一group的）consumer是一对一或者多对一的关系。

![image-20231125161731350.png](https://s2.loli.net/2023/12/03/HzlkdW7ObT8Zcir.png)

# 再均衡（rebalance）

## 机制

当有消费者加入或退出时，会触发再均衡，重新分配partition 在此期间，消费者无法读取topic的消息。

消费者会定时向broker发送心跳，如果在broker设定的时间内（通常时前者时间间隔的三倍）没有收到心跳，则认为该消费者已经离开。

# 消息顺序与状态

## 消息的添加

每次添加消息时，kafka会将其放到某个partition的尾部，因此可以保证partition内的消息有序，而不能保证topic内的所有消息有序。（*当 topic 只有一个 partition 时整体有序*）

kafka发送消息时，需要确定topic、data，可以选择partition和key。

- 如果指定了partition，则会发送到对应partition
- 其次如果指定key，则会通过hash函数发送到特定partition，同一key消息会发送到同一partition。
- 如果均为指定，则会在第一次随机生成整数取模作为key，之后进行递增

因此，如果想要保证消息按顺序被消费，可以发送时指定key/partition

![img](https://camo.githubusercontent.com/39a19abce442bb75245e9e5196290bddef5820e725c6f5e118ad9c9165ad207d/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d31312f4b61666b61546f70696350617274696f6e734c61796f75742e706e67)

## 消息的消费

对于每个partition，会使用offset作为唯一表示保证顺序性，表示consumer当前在该partition消费到的位置。当consumer拉取某消息后，默认会自动提交offset（`enable.auto.commit=true`）。

# 主从同步

副本（replica）以分区partition作为基本单位，包含一个leader和follower。

一些基本概念：

- AR：partition的所有副本（Assigned Replica）
- ISR：与leader保持同步的副本（In-Sync Replica）
- OSR：与leader滞后过多的副本（Out-of-Sync Replica）
- LEO：partition所有副本中的最大offset（LogEndOffset），各自维护
- HW：ISR中最小的LEO，消费者最多能消费的位置（HighWatermakr），各自维护

> HW会在leader和follwer的通信过程更新，有点像raft里的LeaderCommit

## 同步复制

1. 生产者从zookeeper获取Leader，向leader发送消息，leader写入本地Log
2. follower向leader拉取消息，写入log，发送ACK
3. leader收到所有follower的ACK后，向生产者发送ack

## 实际复制

当有新消息发到leader，leader会更新自己的LEO。消息发给follower后，根据follower更新HW。

## ACK类型

producer有三种ACK机制，延迟由好到坏，可靠性由坏到好：

- 0：producer不需要leader回复，发送即认为发送成功，发送下一批
- 1：leader收到消息，写入本地Log后，向producer返回确认，默认机制
- -1：leader需要保证ISR的消息均已同步，才向producer返回确认

# 常见参数配置

## Broker端

- auto.create.topics.enable：是否允许自动创建topic（producer和consumer使用不存在topic）
- broker.id：本broker的ID，不设置则生成唯一ID
- logs.dirs：消息存储位置
- log.retention.bytes：设置日志磁盘占用的阈值，超出进行日志段删除，默认为-1，即不删除
- log.retention.hours：日志的最长保存时间
- message.max.bytes：broker可以批量处理消息的最大字节数
- num.partitions：每个topic的默认分区数，默认为1
- batch.size：缓冲区一批数据的最大值

## producer端

- acks：向leader写入数据时的ACK方式
- compression.type：消息的压缩方式，默认不压缩
- batch.size：缓冲区一批数据的最大值
- max.request.size：发往broker单个请求的最大尺寸（包含多个信息）

## consumer端

- group.id：该consumer所属的group
- allow.auto.create.topics：订阅不存在topic时是否自动创建topic（需要broker支持）
- auto.offset.reset：kafka中没找到对应offset时（例如一个新的group），采取的策略
    - earliest：重置到最早可用消息的offset
    - latest：重置到最新消息的offset，默认值
    - none：broker抛出异常
- enable.auto.commit：是否定期提交offset，默认为true
- auto.commit.interval.ms：设置了自动提交offset的时间间隔
- fetch.max.bytes：从broker获取到的消息最大字节，受message.max.bytes影响
- max.poll.records：一次poll能拉取到的消息最大条数

