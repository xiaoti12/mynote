# 类型及其关系

## 数组和切片的关系

数组为值类型，切片为引用类型。切片本质上是一个**结构体**，定义如下

```go
type slices struct{
	array unsafe.Pointer
	len int
	cap int
}
```

也就是说，切片底层会指向一个数组，但切片本身拥有结构体对象的引用值。观察如下代码

```go
a:=[]int{1,2,3}
b:=a
fmt.Println(&a==&b) // false
fmt.Println(&a[0]==&b[0]) //true
b[0]==9
fmt.Println(a[0]) //a[0]==9
```

a和b是不同的slice对象，所以地址不同；但a赋值给b后，由于是引用类型，因此将指向底层数组的指针也复制给了b。

## 基本类型和引用类型

基本类型包括：整数、浮点数、string、bool、[n]T类型

引用类型包括：[]T、map、interface、func、channel

只有引用类型的零值是nil。需要注意的是，error类型的零值也是nil，因为本质是interface

## 切片的索引操作

- 取最后一个元素：`s[len(s)-1]`，需要检验是否为空（长度是否为0）

- 半索引切片：`s[a:]`或`s[:b]`。其中`s[a:]`相当于`s[a:len(s)]`，`s[:b]`相当于`s[:b]`，因此需要满足`a<=len(s)`，取等时为空。

    也就是说，切片`s[a:b]`中**`a`的最小值为0，`b`的最小值为`len(s)`**

## make和new的区别

new(T)会为T类型分配空间并初始化值，返回T类型的指针，即*T

make只能用于slice、map、channel，初始化值并返回该变量的引用，即T

new和make分配内存时尽量分配在栈上，如果发生逃逸则分配到堆上。

```go
var a []int //a==nil
b:=make([]int,0) //b==[]int{} not nil
c:=new([]int) //c is &[]int{}
```

## 内存逃逸

函数中变量的内存一般分配栈内存，随函数返回而回收。当变量内存分配在堆上时，称为发生了内存逃逸。

最容易出现内存逃逸的情况：多级间接赋值，即`Data.Field = Value`，其中Data和Field均为引用类型，即[]T、map、interface、func、channel。

## struct变量比较

go中以下类型的值**不能比较**：slice、map、function

自定义的struct，有以下规则：

- 类型不同不能比较
- 通过强制类型转换后，可能可以比较（只有成员名称且类型均相同，才能强制类型转换）
- 当成员变量含有不可比较值的值时，不能直接比较

## 可寻址和不可寻址

可以直接使用`&`进行取地址的对象是**可寻址**的。

以下这些可以寻址：变量、指针、数组/切片元素的索引、切片

以下这些不可寻址：常量、字符串、函数方法、基本类型的字面量、**map的元素**

由于map元素不可寻址，因此无法通过

> 因为map中key不存在时返回零值，零值为常量；并且map存在k-v迁移的可能，因此不能寻址

```go
m:=map[int][2]int{}
m[1]=[2]int{0,1}
//m[1][1]=123 —> compile error!!
//correct:
m[1]=[2]int{0,123}
```

## interface与nil
`interface{}`包含两个部分：`type`和`data`。例如
```go
a interface{} = 25
```
其中type为int，data为25。因此进行nil判断时，需要type和data均为nil，`interface{}==nil`才成立


# 函数与方法

## 函数传参方式

需要注意的是，Go中函数的传参均为**值传递**。以切片为例子：

```go
a:=[]int{1,2,3}
fmt.Printf("%p\n",&a) // e.g. 0x1000
appendInt(a,4)
fmt.Println(a) // a={1,2,3}
func appendInt(s []int,num int){
    fmt.Printf("%p\n",&s) // e.g. 0x10a0
    s = append(s)
}
```

上述代码中，`a`和`s`两个切片的地址并不相同。如果在函数中对s进行元素修改，a也会一起修改，因为两者在底层指向的数组是相同的；但当调用`s = appends(s)`后，s指向了不同的新数组（cap更大），因此a不会发生变化。综上，当a作为参数传递给函数时，是将a这个**切片结构体的值传递**进去，复制出了新的切片结构体。

## 值方法和指针方法

值方法可以通过值和指针调用；指针方法只能通过指针调用

而如果值是可寻址的，那么编译器会自动取地址，使得值可以调用指针方法

```go
type Foo struct {}
func (f *Foo) pMethod() {}
func (f Foo) vMethod() {}
func main(){
    vf:=Foo{}
    pf:=&Foo{}
    
    vf.vMethod() //✔
    vf.pMethod() //编译器自动变为(&vf).pMethod()
    
    pf.vMethod() //编译器自动变为(*pf).vMethod()
    pf.pMethod() //✔
}

```

总结：遇事不决，定义指针方法

# 内置库

## 类型转换 strconv

- int型和浮点型转换：`int(3.3)`、`float64(6)`
- 字符串类型（通过内置`strconv`包）：
    - 字符串转int型：`strconv.Atoi("123")` （内部为`ParseInt函数`）//第二个值返回error
    - 字符串转浮点型：`strconv.ParseFloat("123.56",64)` //第二个值返回error
    - int型转字符串：`strconv.Itoa(123)` （内部为`FormatInt`函数）
    - 浮点型转字符串：`strconv.FormatFloat(123.56,'f',2,64)` //具体参数含义可看函数文档

## 排序

int切片排序：`sort.Ints(intSlice)`，逆序：`sort.Sort(sort.Reverse(sort.IntSlice(intSlice)))`

浮点数切片排序：`sort.Float64s(floatSlice)`

string切片排序：`sort.Strings(strSlice)`

在1.21后，可以用内置的`slices`包进行排序：

- `slices.Sort(s)`，同时也可以调用`slices.SortFunc(s,fn)`自定义compare函数
- 如果想要逆序排序，可以`slices.Sort(s); slices.Reverse(s)`

## 字符串操作

Go中string为不可更改类型，可以通过索引获取，但不可修改。需要注意的是，string底层是[]byte（即[]uint8）。

- 字符串拆分为切片：strings.Split(s string, seq string)
- 字符串根据空白字符拆分：strings.Fields(s string)（空白字符包括空格、\t、\r等等）
- slice拼接为字符串：strings.Join(s []string, conn string)
- 取子字符串：通过索引 s[a:b]

# channel

## nil与close

- 读写nil的channel会永久阻塞
- 关闭nil的channel会引发panic
- 对于已经closed的channel，写会引发panic；无缓存channel会返回类型的零值，有缓存channel会一直读到空，返回零值

## select与channel

select和channel结合使用（有时会和for{}一起使用）。

case触发的条件：**通道可以接收或者发送**

```go
select{
    //当case中所有通道均可用时，会随机选择一个进行接受
    //如果均不可用且有default分支，则执行default分支
	case <-ch1:
    	//do something
	case x:=<-ch2:
    	//do something with x
    case y,ok:=<-ch3:
        if !ok{
            return 
        }
    	// do something with y
    	// 当ch3已经close后，第一个值返回chan类型的零值，第二个值会返回false
    default:
    //do something
}
```

此外还有两点

```go
var nilChan chan bool //nilChan is nil
stopChan:=make(chan bool)
go func(){
    time.Sleep(1*time.Second)
    close(stopChan)
}()
for {
    select{
        case <-nilChan:
        	//永远不会触发，select自动忽略nil的chan
        case <-stopChan:
        	//stopChan关闭状态会触发此case
        	// do something
    }
}

```

综上，select中使用channel时，有以下这些情况能接收到值：

- 有其他阻塞等待发送给channel的命令
- channel已经关闭

## range channel

对于一个有缓存channel，如果用`for x:=range ch`去接受ch里的值时，会持续接收直到ch关闭。因此，必须要设置关闭ch的语句，否则会发生死锁。在for range之前直接关闭也可以，这样会将ch里的值全部读完、结束循环。

## for range中的坑

1. range的循环变量会被复用，也就是说每次循环中变量在同一个地址，而值不同

   **注意**：该特点在1.22版本中被修改，每次循环变量会使用不同地址

2. range遍历的slice时对其修改，不会影响循环次数（在一开始就确定好）

2. range遍历map时，顺序是无序的，因为会随机选择一个bucket开始

# Goroutine并发

## 防止goroutine泄露

goroutine可能因为如下原因泄露：

- channel阻塞而无法退出
- goroutine进入死循环
- select在所有case上阻塞

排查方法：

- `runtime.NumGoroutine()`获取当前goroutine数量
- pprof确认泄露的地方

解决方案：

- 确保channel正确同步
- 由其他线程通知goroutine正确退出，例如关闭channel

## 并发模型

- CSP模型：通过通信来实现共享内存
- 生产者/消费者模型
- 订阅/发布模型
- mutex互斥锁实现内存共享

# map的底层实现

## 底层代码和结构

map的底层结构体是hmap，代码如下：

```go
type hmap struct {
 	count     int // 元素个数，调用 len(map) 时，直接返回此值
	flags     uint8
	B         uint8 // buckets 的对数 log_2
	noverflow uint16
	hash0     uint32
	buckets    unsafe.Pointer // 指向 buckets 数组，大小为 2^B；元素为0则指针为nil
	oldbuckets unsafe.Pointer // 用于扩容 buckets
	nevacuate  uintptr
	extra *mapextra // optional fields
}
```

`buckets`是一个指针，指向的是一个bmap结构体数组，也就是具体的**bucket**。数组的长度即为 $2^B$。

```go
type bmap struct {
	tophash [bucketCnt]uint8
}
```

<img src="https://golang.design/go-questions/map/assets/0.png" alt="hashmap bmap" style="zoom: 33%;" />

在编译时，会创建一个新的bmap结构，用于存储key-value值。

```go
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```

bmap的结构如图，其中前8个字节组成 tophash / topbits 字段，用于进行Hash匹配。key和value各自存放，这种是避免每个key-value对都需要padding的情况。

每个bucket最多存放8个key-value对。当有新的key-value要放入该bucket，需要构建一个新的bucket，通过*overflow指针连接。

<img src="https://golang.design/go-questions/map/assets/2.png" alt="mapacess" style="zoom:33%;" />

## GET / UPDATE 操作

get操作的大致流程：

1. 计算Key的哈希值，64位系统则哈希值为64比特
2. 通过哈希值最后 **B** 位确定在几号bucket。例如B=4，取最后4位0101，则在5号桶，取 [5]bmap
3. 通过哈希值前 **8** 位，与topbits字段进行对比。例如发现和 topbits[3]相同，则偏移量为3
4. 通过上一步找出的偏移量，访问对应的key，和代查找的key进行完整匹配。如果相同，则获取对应value，或者更新值
5. 如果key不同，或是该bucket没有对应topbits，则访问*overflow的下一个桶

## PUT操作

PUT操作定位 bucket 的位置和GET操作相同。下面操作：

1. 遍历当前 bucket 的topbits，找到第一个可以插入的位置存储数据。
2. 如果当前 bucket 满，则创建新的 bucket 存放数据，通过*overflow连接

两个不同key的哈希值后 **B** 位相同，则表示发生了哈希冲突，而 bucket 中8个位置则是解决哈希冲突的方法。

## 溢出 bucket

溢出的bucket存放在 hmap.extra结构中，overflow字段指向bmap切片，所有bmap的bucket均为溢出bucket。当PUT操作产生溢出bucket时，在hmap.extra.overflow中新建一个bucket，并且将已满的bucket最后的*overflow指针指向这个bucket

## 扩容机制

### 扩容触发

有两个触发扩容的条件，任意满足：

1. 装载因子（loadFactor）超过阈值6.5。装载因子 = count / 2^B，也就是每个桶的平均k-v对个数

2. 溢出bucket过多。不能超过`2^min(B,15)`个。就是说，`B<=15`时，溢出bucket不能超过正常bucket；`B>15`时，溢出bucket不能超过`2^15`。

条件2是对条件1的补充。当map总元素少、但bucket数量多时，值存储的很稀疏，查找、插入效率低。

###  扩容大小

- **双倍扩容**：针对装载因子的条件1，新建一个bucket数组，大小翻倍（即B增加1），旧buckets数据重新计算位置，迁移到新的bucket。
- **等量扩容**：针对溢出bucket数量过多的条件2，不扩大容量，而是搬迁、重新排布k-v对，提高bucket利用率。

### 扩容过程

go中map的扩容是**渐进式**的，不会一次性搬迁完毕，每次最多搬迁2个。`hashGrow()`函数会首先创建好新的bucket，并将老的buckets挂到oldbuckets字段上。

实际的搬迁是在`growWork()`函数里，每次修改、插入、删除key时，会尝试搬迁buckets。首先需要先检查oldbuckets是否搬迁完毕（也就是是否为nil）。

对于条件2，由于buckets数量不变，因此对应的buckets需要不变（从溢出bucket挪到正常bucket）

对于条件1，需要重新计算key对应的bucket，进行迁移。

> redis底层的哈希表也是渐进式的

# 垃圾回收

## 常见方式

- 追踪式：从根对象出发，根据对象的引用关系，一步步扫描，从而确定需要保留的对象和可回收对象。此方式语言：Go、Java等
- 引用计数式：每个对象包含一个被引用计数器，当计数器归零时自动被回收。

> 根对象包括：全局变量、执行栈、寄存器（可能表示指针）

## 触发时机

Go中垃圾回收触发主要有以下场景：

- 分配的堆大小达到阈值
- 距离上次GC超过一定时间，默认2分钟
- 手动调用`runtime.GC`

## 三色标记法

从回收器的角度，对象有三种类型，如果用不同的颜色区分：

- 白色对象：**可能被回收**。未被回收器访问到的对象。开始阶段所有对象均为白色，访问结束时白色对象均不可达。
- 灰色对象：已经被访问到的对象，但回收器还需要对它们的指针进行进一步扫描。
- 黑色对象：**不会被回收**。已被访问到的对象，并且所有指针字段也已经被扫描。因此黑色对象中的任何一个指针不能直接指向白色对象。

灰色可以视为一个“波面”，在访问作为黑色节点和白色节点的缓冲。在GC开始时，所有对象均为白色；扫描到的对象标记为灰色。当对象的所有子节点均完成扫描时后，该对象被标记为黑色。
> 需要灰色中间态是因为：Go的GC是增量式，GC线程和主线程并发进行。当本次GC被暂停后，下次GC时取出灰色状态的对象，继续进行扫描

当整个堆遍历完成后，只剩下黑色对象和不可达的白色对象，回收白色对象。

![三色标记法全貌](https://golang.design/go-questions/memgc/assets/gc-blueprint.png)

## 强弱不变性

当垃圾回收和用户代码并发运行时，可能破坏垃圾回收的正确性。三色标记算法中，当以下两个条件同时满足时会破坏垃圾回收的正确性，如图：

- 条件1：赋值器让黑色对象引用白色对象（令C指向B）
- 条件2：灰色对象到达白色对象的未访问路径被破坏（删除A指向B的路径）

> 对于条件1，如果让白色对象被黑色对象而不是灰色对象引用，则不会被扫描，因为回收器认为黑色对象的子节点已经全扫描完成

同时满足后，本来应该访问到的白色对象会被回收

![image-20231129192413179.png](https://s2.loli.net/2023/12/03/UjkGCvIm6p2writ.png)

因此有以下两个性质：

- 强三色不变性：满足原有的三色不变性，即条件1、2均不满足
- 弱三色不变性：最多只让条件1生效，也就是必须保证存在能到达可达白色对象的路径。

因此，还需要引入黑色赋值器和灰色赋值器。

## 写屏障

Go采用了下述两种写屏障的混合。基本思想是：对正在被覆盖的对象进行着色，且如果当前栈未扫描完成，则同样对指针进行着色。

### Dijkstra 插入屏障

该屏障的基本思想是避免条件1，也就是避免让黑色对象引用白色对象。

大致来说，如果某对象被黑色对象引用，则将其标记为灰色。

 ### Yuasa 删除屏障

该屏障的基本思想是避免条件2，也就是防止丢失灰色对象到白色对象的路径。

简单来说，在删除灰色对象对子对象的引用之前，将子对象标记为灰色

# GMP模型

GMP模型是Go语言调度goroutine的机制。

## Goroutine特性

- 启动空间小，在栈上只要2-4KB的内存，并且可以动态增大
- 在用户态工作，切换代价小
- 与内核线程的关系是M:N，即M个内核线程可以调度N个goroutine

> 用户线程和内核线程一对一、一对多的关系

## GM模型

go在早期使用的调度模型，性能较差。线程M想要执行协程G，需要访问带锁全局队列，会造成激烈的锁竞争。此外，G创建的G'会交给其他M'执行，但G和G'相关性很高，因此造成的局部性。

![14-old调度器.png](https://cdn.learnku.com/uploads/images/202003/11/58489/uWk9pzdREk.png!large)

## GMP模型框架

- G：goroutine协程
- M：machine，对应内核线程
- P：processer处理器，包含运行goroutine的资源

![16-GMP-调度.png](https://cdn.learnku.com/uploads/images/202003/11/58489/Ugu3C2WSpM.jpeg!large)

如图可见G与P是多对一的关系，P和运行的M是一对一的关系 还包括如下结构：

- 全局队列：存放等待运行的G，带锁
- P本地队列：存放等待运行的G，并且只由该P来运行，可以减少锁的竞争。最多256个
- P列表：在程序启动时创建，最多有`GOMAXPROCS`个
- M列表：不固定，调度器会唤醒或者创建新的M

## 为什么要有P

- 没有P，就退化成了GM模型，导致锁竞争和线程调度复杂性；P的引入可以实现G的调度本地化
- P可以管理栈的分配和回收、内存分配，调度G运行；M负责真正执行G
- P的数量可以用于控制并发度，P的数量限制了多少个M同时执行G

## P与M的关系

### 数量

- P由`$GOMAXPROCS`或者设定的`runtime.GOMAXPROCS()`决定。在任何时刻都有`GOMAXPROCS`个goroutine在同时运行。
- Go默认最大支持10000个M运行，或者由`runtime.debug.SetMaxThreads`函数决定。当当前M阻塞时，P会创建、切换、唤醒M'。

### 创建时机

- runtime确定P的数量后，会创建相应数量的P
- 当P需要新的M执行G、并且没有空闲的M时，会创建新的M

## 调度基本策略

- **线程复用**
    - working stealing：当前M无可用的G时，会从其他M对应的P偷取G（当全局G队列为空时），而不是销毁线程
    - hand off：当前M因为G的系统调用被阻塞时，释放当前P、转移给其他空闲M

- **并行运行**：`GOMAXPROCS`限制了P的数量，即并发的程度
- **抢占**：一个G最多占用CPU 10ms，防止其他G被饿死（从1.14版本引入抢占式调度）
- **全局G队列**：GMP模型中仍然保持了全局G队列。当P执行的G创建的G'放不进P队列中，会放入全局G队列；M对应P的队列为空时，会从全局G队列获取

## G的流动过程

1. 通过`go func()`创建一个goroutine
2. 新的G优先保存在当前P的本地队列，如果队列已满，则从队列取一半+新的G，放到全局G队列
3. M从对应P队列中弹出一个可执行G，如果为空则从全局G队列获取，也为空则从其他P本地队列偷取，如果两个为空则由G0进行自旋（最多`GOMAXPROCS`个M/P在自旋）
4. G将相关运行参数给M
5. 当G进行系统调用导致对应M阻塞时，从休眠队列唤醒一个M'，或者创建新的M'，和当前P绑定
6. 当前G执行完后，将结果返回并被销毁。M会从P本地队列获取新的G，返回第3步

## G0(G-zero)和M0

M0是启动程序后的主线程，负责初始化操作、启动第一个M。之后和其他M一起执行程序。

G0是每次启动M时第一个创建的goroutine，仅用于调度。每个M都会有自己的G0，全局G0是M0的G0

# Channel的底层实现

## 数据结构

channel的底层为hchan结构体，包含如下重要字段：

- qcount：缓存的元素数量
- buf：指向底层循环数组，针对有缓存类型
- closed：是否被关闭
- sendx：已发送元素在循环数组里的索引，针对有缓存类型
- recvx：已接收元素在循环数组里的索引，针对有缓存类型
- sendq：等待发送的goroutine队列
- recvq：等待接收的goroutine队列
- lock：互斥锁，保证线程安全

sendq和recvq为waitq队列，waitq是一个sudog链表结构，sudog封装了goroutine

channel的数据存放在堆上，可以保证在不同的函数内channel访问同一块内存

![chan data structure](https://golang.design/go-questions/channel/assets/0.png)

## 发送操作

1. 如果channel为nil，当前goroutine会被调度器挂起；如果channel已经closed，触发panic
2. 如果channel为有缓存且有空位，则数据会被保存到buf循环数组中
3. 如果channel为无缓存或无空位，则goroutine（G1）被接入到sendq队列中，然后被调度器挂起，释放绑定M1
4. G2从channel接收（有空位）后，从sendq队列弹出一个sudog（即封装的G1），填充循环数组，将G1放入G2对应P2的本地队列中

> 挂起goroutine用的是`gopark`函数

## 接收操作

和发送操作基本一致。

1. 如果channel为nil，goroutine被挂起；如果channel已经closed，返回零值和false
2. 如果是有缓存，直接从buf循环数组读取
3. 如果是无缓存或无数据，则阻塞，被接入recvq队列
4. 下一个发送goroutine将其唤醒

