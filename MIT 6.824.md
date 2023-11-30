# 课程简介

## 分布式系统的驱动力和挑战

### 采用分布式的原因

- 实现并行任务，利用更多的资源
- 实现容错机制，出错的任务可以在其他机器上完成
- 一些问题在物理上是分开的
- 安全目标，实现隔离

### 分布式系统的难题

- 并发性、时间依赖问题
- 处理局部故障
- 需要精心设计才能达到理想性能

## 基础架构和特点

### 分布式系统的基础架构

存储（storage）、通信（communication）、计算（computation）

### 特点

- **可扩容性**：通过更多的硬件资源实现性能或吞吐量的提升。同时要注意，当单独提升某个部分的硬件时，另一部分可能存在瓶颈（例如web服务器和数据库）

- **可用性**：也就是容错。系统要能在出错时继续运行，向开发人员屏蔽错误。此外，还有可恢复性。当系统出现问题时，会自己停止服务，等待修复。

    实现这个特点，可以用到的两个工具：

    - 非易失存储：存放一些checkpoint或者log
    - 副本：同步是个问题

- **一致性**：可以简单分为强一致和弱一致。强一致保证得到数据是最新的；弱一致中得到的数据可能是以前的。而强一致性代价会很高。

# MapReduce

## map函数和reduce函数

MapRuduce的核心为两个函数：map函数和reduce函数。抽象来讲

```
map(k1,v1) ->list(k2,v2)
reduce(k2,list(v2)) ->list(v2)
```

***map函数*** 以数据块作为输入，输出为<key,value>的list。即`(<k1,v1>, <k2,v2>,...,<kn,vn>)`。

***reduce函数*** 收集map函数生成的<k,v>对，对于每个k，会有一系列的v组成list。对list中的v进行累加（也就是把相同key的所有value累积起来）。

## shuffle

MapReduce中还存在一个操作：shuffle。简单来说，就是保证key相同的<k,v>对会交给相同的reducer处理.

## 各模块作用

主要有两个部分：master和worker。其中worker可以分别执行map和reduce函数，视为mapper和reducer。

### Mapper

1. 向master汇报上线
2. 领取任务，进行map，结果写入到中间数据
3. 中间数据位置发送给master
4. 向master定期发送心跳包

### Reducer

1. 从master获取中间数据位置
2. 领取任务，进行reduce，保存结果
3. 任务结束后报告给master
4. 向master定期发送心跳包

### Master

1. 分割总任务，分配给Mapper
2. 监控mapper和reducer的心跳包
3. 提供RPC服务
4. 获取到mapper的结果后，调用reducer

# GFS（大型分布存储）

## 分布式存储的难点

或者说需要解决的问题：数据分片、容错性、副本同步、一致性的性能问题

## GFS设计目标

大量的快速读写、自动恢复、数据分片，针对大量顺序读写进行优化

此外，GFS只保证弱一致性，以此获得更强的性能

## Master节点

master中主要保存了两个表：

- 文件名 ——> chunk ID/Handle数组`<chunk1,chunk2,...>`。表示对于每个文件，对应了哪些chunk（既分片）

- chunk ID ——> chunk server数组`<server1,server2,...>`。表示对于每个chunk，可以去哪些server上寻找。此外，还有如下信息：

    - chunk的版本号
    - 主chunk所在的server。因为对chunk的写操作要先在主chunk上运行。
    - 主chunk的过期时间

## 读文件

1. client发送文件名和偏移量给master，master根据偏移量计算出chunk ID数组中对应的ID
2. master回复chunk ID，和对应的chunk server数组，并缓存对应关系
3. client（一般来说）选择最近的chunk server，将chunk ID和偏移量发送给server
4. server发送回数据

注：client通过 一个GFS库和master通信，如果偏移量涉及的不止一个chunk，库会拆分读请求、发送给master，并合并返回结果给client

## 写文件（追加）

1. client发送文件名、偏移量给master

2. client计算出chunk ID，根据记录的最新版本号，寻找到所有符合最新版本的chunk server

3. master随机选出其中一个作为primary server，并更新master自己的最新版本号。

    > 如果该chunk存在租期内的primary server，则版本号不会增加

4. master通知primary和secondary server：可以修改对应的chunk、primary和secondary的身份、最新版本号。并告知primary server接下来一段时间它会是primary

5. master将primary server、secondary server数组、最新版本号发送给client

6. client把需要写的数据发送给所有合法server，server会将数据写入缓存/临时位置，并发送回client确认信息

    > 实际中，client把数据只发给最近的server，server再串行发给其他server，充分利用带宽

7. client通知primary server，server计算出追加的offset位置，通知所有secondary所写入的位置

8. secondary如果成功写入到chunk，则通知primary“YES”信息

9. primary收到了所有secondary的YES后，则通知client写入成功；如果至少有一个没回复YES，则通知client写入失败

# VMware FT

> 论文讨论的均为单核处理器下的情况

## 状态转移和复制状态机

现在有两个副本primary和backup，需要保持同步。

- 状态转移（State Transfer）：primary发送自己内存、CPU、IO的状态拷贝发送给backup

- 复制状态机（Replicated State Machine）：primary只将外部的不确定事件发送给backup

复制状态机所需要传输的数据往往要远小于状态转移。在没有外部事件或不确定事件时，primary和backup会做相同的事、并得到相同的结果

## 工作原理

### 主要部件

- primary和backup，都运行在VMM上的guest os中，数据存储在disk server中。primary和backup间存在一个log channel

- client，向primary发送请求

- disk server，存储primary和backup的数据 *其实这里的disk server也可以看成收发数据的client*

    这些部件在同一个局域网中

### 工作过程

1. client向primary发送网络数据包请求，产生一个中断
2. primary的VMM将数据同时发送给primary，和backup的VMM
3. primary将应答数据包，通过虚拟NIC发回给client；backup会生成同样的应答包，被VMM丢弃

### primary迁移

在每次定时器中断时，primary都会通过Log channel发送log entry给backup。而如果backup在一段时间内未接收到log entry，则认为primary挂掉，自己接替成为primary，响应client。

### Test-and-Set

当primary和backup无法通信、但正常工作时，backup会尝试接管primary，因此会产生split-brain（脑裂）现象。为了避免此问题，论文提出在disk server中设置一个原子锁，保存某个flag。primary和backup会发送test-and-set请求，拿到正确flag的会上线为primary

## 非确定性事件

非确定性事件可以分为：

- client输入：该输入的到达时间和内容是无法预测的
- 随机指令：例如随机数生成指令、时间戳等等

这些事件通过log channel传送，（Robert教授猜测）log entry包含三个内容：

- 指令序号，即指令自机器启动以来的相对序号
- log类型，可能是网络数据或指令
- 数据，网络数据，或primary上指令执行的结果，backup的VMM则可以伪造指令，得到和primary相同的结果

### log entry buffer

backup和primary会维护一个log buffer，用于接收和发送log。backup当log buffer中不存在指令时，不允许执行指令；当buffer不为空时，backup会知道对应的指令需要，那么自己的指令最多执行到此位置。

通过这种方式，保证backup的指令执行不会快于primary。

## 输出

### output rule

为了解决primary给client响应后、崩溃的情形，论文限定了控制输出。当primary将log entry发送给backup后，接收到backup返回的确认消息时，才会产生输出。

### 重复输出

当primary已经发给client回复，而log entry还在backup的buffer里，primary崩溃了时，backup接管后会发给client一条相同的回复。但由于TCP栈会认为这是重复报文（序列号相同），因此会被舍弃。

# Raft

## 投票选举

### 选举过程

1. 每个节点拥有等待超时时间。在该时间内，若没收到leader的心跳消息，则变为candidate。
2. 成为candidate后，将自己的term增加1，给自己投一票，向其他节点发送请求投票RPC。
3. 其他节点收到请求投票RPC后，如果在该term中未投票，则投给该follower，更新自己的term为消息的term
4. 如果candidate收到的投票数过半，则更新自己为该term内的leader，向其他节点发送appendlog心跳包

### 选举规则

- 如果收到candidate请求投票时，请求的lastlLogIndex和lastLogTerm小于自己，则不投票给该candidate
- 一般来说（论文里），节点的选举等待超时时间为随机150-300ms
- 只要在RPC中发现更新的term，则更新自己的term
- 如果candidate/leader收到term更大/新的请求，则回复自己为follower

## 日志复制

### 发送日志心跳

- 对于RPC参数中的preLogIndex和preLogTerm，通过leader维护的nextIndex[]来构造。对于某个follower，对应的nextIndex的日志前一项，即为preLog。
- 对于RPC中的entries，先采用一次一日志的方法。发送的日志为follower对应的nextIndex的日志

### 接收日志心跳

- 如果RPC参数的term小于自身term，失败
- 如果不存在preLogIndex对应的log，失败（暗示leader要发靠前一些的log）
- preLogIndex对应的log的term和preLogTerm不相符，删除从preLogIndex开始的所有log
- 添加log、同步commitIndex（不能超过自己本地lastLogIndex）、成功

### 接受心跳反馈

leader对每个follower的发送结果：

- 发送失败，则不更新数据，稍后重试；
- 发送成功，而返回失败结果：
    1. term大于自己，切换为follower
    2. 递减nextIndex对应项，下次发送更靠前的log
- 发送成功，返回成功结果，更新follower对应的matchIndex和nextIndex，根据matchIndex更新自己的commitIndex



## 过半投票思想

为了构建自动恢、避免脑裂现象的系统，raft论文提出的一种概念。在系统中，为了完成某种操作，必须有过半的节点（要求节点数量为奇数）批准/完成该操作时，才会进行相应的操作。

对于拥有 **$2f+1$** 个节点的系统，可以容忍 $f$ 个节点的故障。也就是至少要有超过半数的节点正常运行。

