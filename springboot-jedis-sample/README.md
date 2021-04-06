# SpringBoot集成Jedis框架-实现Redis调用

### Redis原理介绍

##### Redis Epoll原理
Redis是一个单线程却性能非常好的内存数据库，主要用来作为缓存系统。Redis采用网络IO多路复用技术，保证了多个连接的时系统依然有吞吐量表现。

**Redis选择I/O多路复用的原因？**
首先，Redis是跑在单线程中的，所有的操作都是按照顺序线性执行的，但是由于读写操作等待用户输入或者输出都是阻塞的，所以I/O操作往往不能直接返回，这会导致某一个文件的I/O阻塞导致整个进程无法对其它客户端提供服务，而I/O多路复用就是为了解决这个问题。

Redis的IO模型主要基于Epoll实现的，不过它还提供了Select和Kqueue的实现，默认采用Epoll模型
Epoll模型属于诸多IO多路服用模型中的一种，但是相比其他IO多路复用模型技术（Select、Poll等）

**Epoll有诸多优点：**
> - Epoll没有最大并发限制，上线是系统最大文件的数目，具体数目可以 cat /proc/sys/fs/file-max 中查看；
> - 效率提升，Epoll最大的优点就是它只管“活跃”的连接，而跟连接总数无关，因此在实际的网络环境中，Epoll的效率就会远远高于Select和Poll；
> - 内存拷贝，Epoll在这点上使用了“共享内存”，这个内存拷贝也省略了；

**Epoll与Select/Poll的区别：**

> - Select、Poll、Epoll都是IO多路复用机制。I/O多路复用就是一种机制，可以监视多个描述符，一旦某个描述符就绪，能够通知程序进行相应的操作；
> - Select的本质是采用32个整数的32位，即32 x 32=1024来标识，fd的值为1～1024。当fd的值超过1024限制时，就必须修改FD_SETSIZE的大小。这个时候可以标识32*max值范围的fd；
> - poll和select不同，通过一个pollfd数组向内核传递关注的事件，故没有描述符个数的限制，pollfd中的events字段和revents分别用于标示关注的事件和发生的事件，故pollfd数组只需要被初始化一次；
> - Epoll还是Poll的一种优化，返回后不需要对所有的fd进行遍历，在内核中维持了fd的列表。Select和Poll是将内核列表维护在用户态，然后传递到内核态中。与Poll/Select不同，Epoll不再是一个单独的系统调用，而是由epoll_create/epoll_ctl/epoll_wait三个系统调用组成，Epoll在Linux2.6以后内核才支持；

**Select/Poll的几大缺点：**

> - 每次调用select/poll，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大；
> - 同时每次调用select/poll都需要在内核遍历传递进来的所有fd，这个开销在fd很多时也会很大；
> - 针对select支持的文件描述符数量太小时，默认是1024；
> - select返回的是含有整个句柄的数组，应用程序需要遍历整个数组才能发现哪些句柄发生了事件；
> - select的触发方式是水平触发，应用程序如果没有完成对一个已经就绪的文件描述符IO操作，那么之后每次select调用还是会将这些文件描述符通知进程去处理；
> - poll相比select模型，poll使用链表保存描述符，因此没有监视文件数量的限制，pollfd支持复用仅初始化一次，但是依然存在fd拷贝和遍历问题；

**Epoll IO多路复用模型实现机制：**

由于epoll的实现机制与select/poll机制完全不同，上面所说的select的缺点在epoll中不复存在。
epoll没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048，举个例子：在1GB的内存机器上大约是10万左右的最大连接。

Epoll如何实现高并发的？
在select/poll时代，服务器进程每次都需要把连接告诉操作系统（从用户态复制句柄数据到内核态），让操作系统内核去内核查询这些套接字上是否有事件变化，轮训完后再复制到用户态，让服务器应用程序轮询处理已发生的网络事件，这个过程消耗较大，因此select/poll无法处理几千个并发连接。
epoll的设计和实现与select完全不同，epoll通过在linux内核中申请一个简易的文件系统（采用B+Tree结构存储）。把原先的select/poll调用分为3个部分：

> - 调用epoll_create()建立一个epoll对象（在epoll文件系统中这个句柄对象分配资源）；
> - 调用epoll_ctl向epoll对象添加套接字；
> - 调用epoll_wait收集发生事件的连接；
在进程启动时创建一个epoll对象，然后在需要的时候向这个epoll对象中添加或者删除连接。同时，epoll_wait的效率也非常高，因为epoll_wait时，并没有一股脑的想操作系统复制连接句柄数据，内核也不需要去遍历全部连接。

------------

##### Redis指令队列原理
Redis Server就是通过blocking_keys（指令队列）和 ready_keys（响应队列）两个数据结构来实现阻塞操作。但整个阻塞并没有阻塞EventLoop本身，从而实现指令的快速响应。算是一个典型的空间换时间的设计思路。

**指令队列：**
Redis会将每个客户端套接字都关联一个指令队列。客户端的指令通过队列来排队进行顺序处理，先到先服务。

**响应队列：**
Redis同样也会为每个客户端套接字关联一个响应队列。Redis服务器通过响应队列来将指令的返回结果回复给客户端。如果队列为空，那么意味着连接暂时处于空闲状态，不需要去获取写事件。

------------

##### Redis序列化协议
Redis服务器与客户端通过RESP（REdis Serialization Protocol）协议通信。
RESP特点：容器实现、解析快、可读性
RESP底层采用TCP的连接方式，通过TCP进行传输数据，根据解析规则解析相应信息，完成交互。

------------

#### Redis持久化
持久化的作用，当Redis服务宕机或异常崩溃时，可以通过持久化文件进行数据恢复。

#### 1、RDB
把当前数据生成快照，保存到磁盘上，rdb持久化可以手动触发，也可以自动触发

**RDB手动触发命令：**
> - save命令：执行save命令会手动触发RDB持久化，但是save命令会阻塞Redis服务，因为save是使用Redis主进程完成，直到rdb持久化完成，Redis才能继续提供服务。当数据量大时，阻塞时间越长，此时是不能提供服务的，不建议使用；
> - bgsave命令：执行bgsave命令也会触发RDB持久化，和save命令不同的是，采用fork + copy on write的方式，持久化由fork出的子进程完成，Redis主进程只阻塞fork阶段，时间较短；

**RDB自动触发机制：**

Redis自动触发都使用bgsave机制完成
> - 在redis配置文件中设置save相关配置，如save m n，它表示在m秒内被修改n次时，自动触发bgsave操作。

**Redis中的fork()：**

Redis巧妙的运用了fork，当bgsave执行时，Redis主进程会判断当前是否有fork出来的子进程，若有则忽略不执行，若没有则fork出一个子进程来执行rdb文件持久化工作，子进程与主进程共享一份内存空间，由子进程做持久化，主进程又能继续对外的服务，二者互不影响。

**copy on write机制：**

子进程持久话的数据是在fork时的数据，也就是说主进程和子进程都是同一块内存空间，之后主进程又能继续提供服务，当遇到内存数据修改时，需要保证子进程对修改数据不可见的， 这个机制就是由copy on wirte完成的。

原理：主进程fork子进程后，内核把主进程中所有的内存页权限都设置为read-only，然后子进程的地址空间指向主进程。这也就是共享了主进程的内存，当其中主进程写内存时，CPU硬件会检测到内存页权限是read-only的，于是触发内存页中断（page-dault），陷入内核中断例程。中断例程中，内核会把触发的异常页复制一份（仅复制异常页，也就是修改的那个数据页，而不是全部数据页），于是父子进程都各自持有独立的一份数据（主进程是新的，子进程是老的）。

**RDB优点：**

RDB文件是一个紧密的二进制压缩文件，是Redis在某个时间点的全部数据快照。所以使用RDB恢复数据比AOF快，非常适合备份、全量复制、灾难恢复等场景。

**RDB缺点：**

每次进行bgsave操作都需要只想fork操作创建子进程，属于重量级操作，频繁执行成本较高，所以无法做到实时持久化，或者是秒级持久化。

#### 2、AOF

AOF（Append Only File）持久化是把每次写的命令追加写入日志中，当需要数据时重新执行AOF文件中的命令就可以了。AOF解决了数据持久化的实时性。

**AOF持久化流程：**
> - 命令追加（append）：所有写命令都会被追加到AOF缓存区（aof_buf）中；
> - 文件同步（sync）：根据不同策略将AOF缓存区同步到AOF文件中；
> - 文件重写（rewrite）：定期对AOF文件进行重写，以达到压缩的目的；
> - 数据加载（load）：当需要恢复数据时，重新执行AOF文件中的命令；

**AOF同步策略**
> - always：每次写入缓冲区都需要同步到AOF文件中，硬盘的操作比较慢，限制了Redis高并发；
> - no：每次写入缓存区，不进行同步，同步到AOF文件的操作丢给操作系统负责，每次同步AOF文件周期不可控；
> - eversec：每次写入缓存区，由专门的线程每秒钟同步一次，做到了性能与数据安全的兼并；

------------

#### Redis集群
Redis Cluster 是Redis 的分布式解决方案，在3.0版本推出后有效的解决了redis分布式方面的需求，自动将数据分片，每个master上放一部分数据，提供内置的高可用支持，部分master不可以时还是可以继续工作；

**Redis clsuter vs. Master/Slave + Sentinal：**

> - 主从+哨兵模式一个master、多个slave，master负责读写，slave负责复制master数据和提供读服务，配合sentinal集群，可以保证master/slave故障自动切换时高可用；
> - redis集群模式，主要针对海量数据+高并发+高可用场景；

**Redis Cluster的分片算法Hash Slot算法：**

> - redis cluster有固定的16384个hash slot，对每个key计算CRC16值，然后对16384取模，可以获取key对应的hash slot；
> - redis cluster中每个master都会持有部分slot，比如3个master，那么每个master就持有16384/3 ~= 5000多个hash slot；
> - redis cluster中每个节点都会记录哪些槽指派给了自己，哪些槽位分配给了别的节点；
> - 当客户端节点发送key命令时，负责接受的节点需要计算这个key属于那个槽，如果是自己负责的槽位直接执行，如果不是，想客户端返回一个MOVED错误，指引客户端转向正确的节点；
> - 任何一台机器宕机，另外两个节点，不影响的，因为key找的是hash slot找的不是机器；

**Redis节点间的内部通信机制：**

redis cluster节点间采取gossip协议进行通信。
gossip算法如其名，灵感来自办公室的八卦，在有限的时间内所有人都会知道该八卦信息。
市面上集群中的元数据同步分为两种，集中式、最终一致性，gossip追求的是最终一致性：

> - 集中式：如zookeeper、etcd，好处在于元数据的更新和读取，时效性非常好，一旦数据出现变更，立即更新到集中式的存储中，其它节点读取的时候立即就能感知到，坏处是集中式存储的元数据一旦出现故障，会导致短期内不能正常提供服务；
> - gossip：好处在于，元数据的更新比较分散，不是集中在一个地方，元数据更新会陆陆续续，更新到所有节点上，有一定的延迟，但是提升了可用性；

gossip协议包含多种消息，包括：ping、pong、meet、fail等：

> - meet：当某个节点发送meet给新加入的节点，让新节点加入集群中，然后新节点就会开始跟其他节点进行通信；
> - ping：每个节点都会频繁的给其他节点发ping，其中包含自己的状态还有自己维护的redis集群元数据信息，相互通过ping交换元数据；
> - pong：返回ping和meet，包含自己的状态和其他信息，也可以用于信息广播和更新；
> - fail：某个节点判断另一个节点fail之后，就发送fail给其他节点，通知其他节点指定的节点宕机了；

------------

**面向集群的Jedis内部实现原理：**

1、基于Redis客户端，redis-cli -c指令，客户端可能会挑选任意一个redis实例去发送命令，每个redis实例收到命令，都会计算key对应的hash slot，如果在本地就直接执行，负责返回moved给客户端，让客户端进行重定向到hash slot所在的集群；

2、基于JedisCluster的原理，在JedisCluster初始化的时候，就会随机一个node，初始化hashslot -> node 映射表，同时为每个节点创建一个JedisPool连接池。每次基于JedisCluster执行操作，首先JedisCluster都会在本地计算key的hash slot，然后在本地映射表中找到对应的节点。如果redis cluster中的slot产生变化时，JedisCluster执行已经不存在那个node上的key，就会返回moved，那么利用该节点的元数据，更新hash slot -> node 映射表缓存；

------------

#### String字符串类型

字符串是Redis中最常见的数据存储类型，其底层实现就是简单的动态字符串SDS（simple dynamic string），是可以修改的字符串。
它类似Java中ArrayList，它采用预分配冗余空间的方式减少内存的频繁分配。
当字符串长度小于1M时，扩容都是加倍现有的空间，如果超过1M，扩容时一次只会多扩容1M空间（字符串最大长度为512M）

------------

#### Dict字典类型

类似Java中的HashMap结构，但是扩容是有所不同。
渐进式哈希的精髓在于：数据迁移不是一次性完成的，而是通过dictRehash()这个函数分布规划的，并且调用方知道是否需要继续进行渐进式哈希操作。如果dict数据结构中存在海量数据，那么一次性迁移必带来redis性能的下降，redis是单线程内存处理模型的，在实时性要求高的场景下这可能是致命的。而渐进式哈希则将这种代价分摊了，调用方可以在dict做插入、删除、更新的时候执行dictRehash()，最小化迁移代价。

rehash是bucket(桶)为基本单位渐进式数据迁移的，每步完成一个bucket的迁移，直到数据迁移完毕，一个bucket对应哈希表中的一条entry链表。新版本的dictRehash()还加入一个最大访问空桶数(emptry_visits)的限制来进一步减少可能引起的阻塞时间；

------------

#### List列表

Redis列表使用了两种数据结构做为底层实现

> - 压缩列表：ZipList
> - 双向列表：LinkedList

因为双向链表占用内容比较高，所以在创建列表时，优先使用ZipList，当容量达到一定约束时转换为LinkedList

> - 列表中新增一个字符串的长度大于64字节；
> - 列表中节点的长度大于512个；

ZipList是一个特殊的双向链表：

特殊之处在于，没有维护双向指针：prev、next，而是存储上一个entry的长度和当前entry的长度，通过推算长度可以计算出下一个元素的位置；
牺牲了可读性，获取了高效的存储空间，因为简单的字符串情况下，存储元素的指针比存储entry的长度更花费内存，这是典型的时间换空间例子；

ZipList和LinkedList的优缺点：

> - 双向链表LinkedList便于在表的两端进行push和pop操作，在插入节点上复杂度很低，但是它的内存消耗比较大，首先，它在每个节点上除了要保存数据之外，还要额外保存两个指针。其次双向链表的各个单独的内存块，地址不连续，节点多了很容易产生碎片；
> - ZipList存储一段连续的内存上，所以存储效率很高。但是不方便于修改操作，插入和删除操作需要频繁的申请和释放内存。特别是ziplist长度很长的时候，一次realloc可能会导致大批量的数据拷贝；

**Redis3.2后使用QuickList作为List存储结构：**

可以认为QuickList是ziplist和linkedList的结合。quicklist是一个ziplist组成的双向链表，每个节点使用ziplist来保存数据。本质上说，quicklist里保存了一个一个的ziplist；

QuickList是ZipList的一次封装，使用小块的zipList保证内存使用，也保证了性能：

> - quicklist就是一个标准的双向链表的配置，有head和tail；
> - 每个节点就是一个QuickListNode，包含prev和next指针；
> - 每一个QuickListNode包含一个ZipList，zip压缩链表存储健值；

------------

#### Zset有序列表
Zset的两种实现方式，ZipList和SkipList（跳表）

1、ZipList：满足以下两个条件

> - 元素数量小于128个数量时；
> - 每个元素的长度小于64个字节；

2、SkipList：不满足上述两个条件就会使用跳表，具体来说是组合了hashtable和skiplist

> - hashtable用来存储member到score的映射，这样就可以在O(1)时间找到member对应的分数；
> - skiplist按从小到大的顺序存储分数；
> - skiplist每个元素的值都是【score、value】对应；

**SkipList的优势：**

SkipList本质上是并行的有序链表，但它克服了有序链表插入和查找性能不高的问题，它的插入和查询时间复杂度都是O(logN)


#### LRU与LFU淘汰算法
Redis缓存淘汰策略与Redis键的过期删除策略并不完全相同，前者是在Redis内存使用超过一定阈值的时候使用淘汰策略。而后者是通过定时+惰性删除两者结合的方式进行内存淘汰的。
不同触发的条件逻辑：

> - 当某个key被设置了过期时间后，客户端每次对该key的访问（读写）都会去检查一遍key是否过期，如果过期直接删除；
> - 如果过期的key不经常被访问，Redis会定时每秒检测10次，检测被设置有效期的所有key的集合，每次从集合中随机抽取20个key进行删除，如果删除后集合中过期key依然超过25%，那么会继续随机抽取20个key删除；

Redis内存不足时的缓存淘汰策略：

> - noeviction：当内存使用超过配置的时候会返回错误，不会删除任何key；
> - allkeys-lru： 加入key的时候，如果有限，首先通过LRU算法删除最久没有使用的key；
> - volatile-lru：加入key的时候如果有限，首先从设置了过期时间里删除最久没有使用的key；
> - allkeys-random：加入key的时候如果有限，从所有key中随机删除key；
> - volatile-random： 加入key的时候如果有限，从所有设置过期时间里随机删除key；
> - volatile-ttl：从配置了过期时间里，删除快过期的key；
> - volatile-lfu：从所有配置了过期时间里删除使用频率最少的key；
> - allkeys-lfu：从所有key中删除使用频率最少的key；


**Redis中的LRU实现：**

Redis中的LRU和Java中的LRU实现不一样，它并不采用链表的方式存储。
Redis为每一个key维护了一个24位的时钟，同时也维护了一个全局的24位时钟，简单的理解就是当前系统的时间戳。如果要进行LRU，那么首先拿到当前全局的时钟，然后找到内部所有key时钟和全局时钟距离最久的key进行淘汰；

**Redis中的LFU实现 ：**

LRU是Redis4.0后出现的淘汰算法，LRU的最近最少使用实际上是不精确的，因为使用距离时间较最久的key，不代表是使用频率最少的key；

如下图，按照LRU会淘汰A，因为B的访问时间比A更近，但是A的使用频率远超于B，理应淘汰B；
```asp
A~~A~~A~~A~~A~~A~~A~~A~~A~~A~~~|
B~~~~~B~~~~~B~~~~~B~~~~~~~~~~~B |
```

LFU把原先的key对象内部的24位时钟分为了两个部分，前16位还代表时钟，后8位代表一个计数器。
使用LFU淘汰时，会根据计数器中key使用的频率精准的淘汰最少使用频率的key。


------------

### Jedis、Lettuce、Redisson客户端框架介绍以及对比

#### 一、概念
Jedis：是老牌的Redis的Java实现客户端，提供了比较全面的Redis命令的支持；
Redisson：实现了分布式和扩展的Java数据结构；
Lettuce：高级Redis客户端，用于线程安全同步，异步和相应使用，支持集群、Sentinel，管道和编码器

#### 二、优点
Jedis：比较全面的提供了Redis的操作特性；
Redisson：促使使用者对Redis的关注分离，提供很多分布式相关操作服务，例如（分布式锁、分布式集合），可以通过Redis支持延迟队列；
Lettuce：基于Netty框架的事件驱动的通信层，其方法调用是异步的。Lettuce的API是线程安全的，所以可以操作单个Lettuce连接来完成各种操作；

#### 三、可伸缩
Jedis：使用阻塞的I/O，且其方法调用都是同步的，程序流需要等到Sockets处理完I/O才能使用，不支持异步。Jedis客户端实例不是线程安全的，所以要使用连接池配合使用Jedis；
Redisson：基于Netty框架的事件驱动的通信层，其方法调用是异步的。Redisson的API是线程安全的，所以可以操作单个Redisson连接来完成各种操作；
Lettuce：基于Netty框架事件驱动的通信层，其方法调用是异步的。Lettuce的API是现成安全的，所以可以操作单个Lettuce连接来完成各种操作。Lettuce能够支持Redis4，需要Java8以上版本。


### Jedis原理介绍

##### Redis信协议
Jedis Client是Redis官网推荐的一个面向Java的客户端，库文件实现了对Redis各类API进行封装调用。

Redis-Cli与Server端使用一种专门为Redis设计的协议RESP(Redis Serialization Protocol)交互，RESP本身没有指定TCP，但是Redis的上下文只能使用TCP连接。

Jedis通过RESP协议实现了与Redis服务端的通信，Jedis客户端本质上是通过Socket按照RESP协议与Redis通性。

> - 用\r\n做间隔
> - 对于简单的字符串，以+号开头

```asp
set hello world
+OK\r\n
```

> - 对于错误消息，以-开头

```asp
sethx  // 该命令不存在
-ERR unknown command 'sethx'
```

> - 对于整数，以:开头

```asp
dbsize
:100\r\n
```

> - 对于大写字符串，以$开头，接着跟上字符串的长度

```asp
get name
$6\r\nfoobar\r\n      代表一个长6的字符串， foobar 
$0\r\n\r\n            长度为0 的空字符串
$-1\r\n                Null
```

> - 对于数组，以*开头，接上元素的个数

```asp
*0\r\n    一个空的数组
mset name1 foo name2 bar
mget name1 name2
*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n 一个有两个元素的数组   foo  bar
```

通过RESP执行命令完整步骤如下：

> 1.输入命令
>
> 2.将命令编码成字节流
>
> 3.通过TCP发送客户端
>
> 4.服务端解析字节流
>
> 5.服务端执行命令
>
> 6.将结果编码成字节流
>
> 7.通过TCP链接发送给客户端
>
> 8.解析字节流
>
> 9.得到执行结果


##### Jedis连接池
虽然基于内存的Redis数据库有着超高的性能，但是底层的网络通信却占用了一次数据请求的大量时间，因为每次数据交互需要先建立连接。

在Jedis中每次new Jedis()都会创建新的TCP连接，使用后再断开连接，对于频繁访问Redis的场景显然不是高效的方式。

Jedis连接池则可以实现在客户端建立多个连接并不释放，当需要使用连接的时候通过一定的算法获取已建立的连接，使用完了以后则还给连接池，这样可以减少Redis每次操作时创建新连接带来的IO消耗。

客户端连接Redis使用的是TCP连接，直连的方式每次都要建立新的TCP连接，而连接池的方式是可以预先初始化好jedis连接，每次只需要从jedis连接池借用即可，借用和归还操作是在本地进行的，只有少量的并发同步开销，远远小于新建TCP连接的开销。

此外，连接池的方式可以有效保护和控制资源的使用，而直连的方式无法限制jedis对象的个数，并且可能存在连接泄漏的情况。

##### SpringBoot集成Jedis-实现Redis调用实战
1、引入Jedis依赖
```java
<dependency>
	<groupId>redis.clients</groupId>
	<artifactId>jedis</artifactId>
	<version>2.6.2</version>
</dependency>

	<dependency>
	<groupId>org.apache.commons</groupId>
	<artifactId>commons-pool2</artifactId>
	<version>2.6.2</version>
</dependency>
```

2、添加Jedis配置（application.properties）
```java
#Redis默认库
spring.redis.database=0
#Redis地址
spring.redis.host=10.211.55.6
#Redis端口
spring.redis.port=6379
#Redis密码
spring.redis.password=
#控制一个pool可分配多少个jedis实例
spring.redis.jedis.pool.max-active=100
#表示borrow一个jedis实例是，最大等待的时间，如果超时，直接抛出jedisConnectionException
spring.redis.jedis.pool.max-wait=1000
#控制一个poll最多有多少个状态为idle的jedis实例，超出这个阈值会clonse掉超出的连接
spring.redis.jedis.pool.max-idle=100
spring.redis.jedis.pool.min-idle=0
#Jedis连接超时时间
spring.redis.timeout=10000
```

3、配置JedisPool的Bean
```java
@Configuration
public class RedisConfig {

    @Value("${spring.redis.host}")
    private String host;

    @Value("${spring.redis.port}")
    private int port;

    @Value("${spring.redis.timeout}")
    private int timeout;

    @Bean
    @ConfigurationProperties("redis")
    public JedisPoolConfig jedisPoolConfig() {
        return new JedisPoolConfig();
    }

    @Bean(destroyMethod = "close")
    public JedisPool jedisPool() {
        JedisPool jedisPool = new JedisPool(jedisPoolConfig(), host, port, timeout * 1000);
        return jedisPool;
    }
}
```

4、封装JedisPool的JedisUtil工具类
```java
@Component
@Slf4j
public class JedisUtil {

    @Autowired
    private JedisPool jedisPool;

    /**
     * @param key
     * @param indexdb 选择redis库 0-15
     * @return 成功返回value 失败返回null
     */
    public String get(String key, int indexdb) {
        Jedis jedis = null;
        String value = null;
        try {
            jedis = jedisPool.getResource();
            jedis.select(indexdb);
            value = jedis.get(key);
            log.info(value);
        } catch (Exception e) {

            log.error(e.getMessage());
        } finally {
            returnResource(jedisPool, jedis);
        }
        return value;
    }
	
	.....省略其余CMD操作
		
}		
```

5、编写JedisUtil的测试类，进测试
```java
@SpringBootTest
class JedisTest {

    @Autowired
    private JedisUtil jedisUtil;

    @Test
    public void testCmd(){
        jedisUtil.set("name", "ipman", 0);
        jedisUtil.expire("name", 86400, 0);
        System.out.println(jedisUtil.get("name", 0));
        System.out.println(jedisUtil.ttl("name", 0));
    }
}
```