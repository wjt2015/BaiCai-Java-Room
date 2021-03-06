> 白菜Java自习室 涵盖核心知识

> [Java工程师的进阶之路 Redis篇（一）](https://juejin.im/post/6850418115840409613)<br>
> [Java工程师的进阶之路 Redis篇（二）](https://juejin.im/post/6863250340738695176)<br>

## 1. Redis 简介

Redis 是一个开源，高级的键值存储和一个适用的解决方案，用于构建高性能，可扩展的 Web 应用程序。Redis 也被作者戏称为 数据结构服务器 ，这意味着使用者可以通过一些命令，基于带有 TCP 套接字的简单 服务器-客户端 协议来访问一组 可变数据结构 。(在 Redis 中都采用键值对的方式，只不过对应的数据结构不一样罢了)

**Redis 的优点**:

1. **异常快** - Redis 非常快，每秒可执行大约 110000 次的设置(SET)操作，每秒大约可执行 81000 次的读取/获取(GET)操作。
2. **支持丰富的数据类型** - Redis 支持开发人员常用的大多数数据类型，例如列表，集合，排序集和散列等等。这使得 Redis 很容易被用来解决各种问题，因为我们知道哪些问题可以更好使用地哪些数据类型来处理解决。
3. **操作具有原子性** - 所有 Redis 操作都是原子操作，这确保如果两个客户端并发访问，Redis 服务器能接收更新的值。
4. **多实用工具** - Redis 是一个多实用工具，可用于多种用例，如：缓存，消息队列(Redis 本地支持发布/订阅)，应用程序中的任何短期数据，例如，web应用程序中的会话，网页命中计数等。

## 2. Redis 基本数据结构

Redis 有 5 种基础数据结构，它们分别是：string(字符串)、list(列表)、hash(字典)、set(集合) 和 zset(有序集合)。

### 2.1. 字符串 string

Redis 中的字符串是一种 **动态字符串**，这意味着使用者可以修改，它的底层实现有点类似于 Java 中的 ArrayList，有一个字符数组，从源码的 sds.h/sdshdr 文件 中可以看到 Redis 底层对于字符串的定义 SDS，即 Simple Dynamic String 结构。

> 常用命令: set,get,decr,incr,mget 等。

String 数据结构是简单的 key-value 类型，value 其实不仅可以是 String，也可以是数字。 常规 key-value 缓存应用； 常规计数：微博数，粉丝数等。

### 2.2. 列表 list

Redis 的列表相当于 Java 语言中的 **LinkedList**，注意它是链表而不是数组。这意味着 list 的插入和删除操作非常快，时间复杂度为 O(1)，但是索引定位很慢，时间复杂度为 O(n)。

> 常用命令: lpush,rpush,lpop,rpop,lrange 等。

list 就是链表，Redis list 的应用场景非常多，也是 Redis 最重要的数据结构之一，比如微博的关注列表，粉丝列表，消息列表等功能都可以用 Redis 的 list 结构来实现。

Redis list 的实现为一个双向链表，即可以支持反向查找和遍历，更方便操作，不过带来了部分额外的内存开销。

另外可以通过 lrange 命令，就是从某个元素开始读取多少个元素，可以基于 list 实现分页查询，这个很棒的一个功能，基于 Redis 实现简单的高性能分页，可以做类似微博那种下拉不断分页的东西（一页一页的往下走），性能高。

### 2.3. 字典 hash

Redis 中的字典相当于 Java 中的 **HashMap**，内部实现也差不多类似，都是通过 "数组 + 链表" 的链地址法来解决部分 哈希冲突，同时这样的结构也吸收了两种不同数据结构的优点。

> 常用命令： hget,hset,hgetall 等。

hash 是一个 string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象，后续操作的时候，你可以直接仅仅修改这个对象中的某个字段的值。 比如我们可以 hash 数据结构来存储用户信息，商品信息等等。

### 2.4. 集合 set

Redis 的集合相当于 Java 语言中的 HashSet，它内部的键值对是无序、唯一的。它的内部实现相当于一个特殊的字典，字典中所有的 value 都是一个值 NULL。

> 常用命令： sadd,spop,smembers,sunion 等。

Redis 中的 set 类型是一种无序集合，集合中的元素没有先后顺序。

当你需要存储一个列表数据，又不希望出现重复数据时，set 是一个很好的选择，并且 set 提供了判断某个成员是否在一个 set 集合内的重要接口，这个也是 list 所不能提供的。可以基于 set 轻易实现交集、并集、差集的操作。

比如：在微博应用中，可以将一个用户所有的关注人存在一个集合中，将其所有粉丝存在一个集合。Redis 可以非常方便的实现如共同关注、共同粉丝、共同喜好等功能。

### 2.5. 有序列表 zset

这可能使 Redis 最具特色的一个数据结构了，它类似于 Java 中 SortedSet 和 HashMap 的结合体，一方面它是一个 set，保证了内部 value 的唯一性，另一方面它可以为每个 value 赋予一个 score 值，用来代表排序的权重，它的内部实现用的是一种叫做 **「跳跃表」** 的数据结构。

> 常用命令： zadd,zrange,zrem,zcard 等。

和 set 相比，sorted set 增加了一个权重参数 score，使得集合中的元素能够按 score 进行有序排列。

比如： 在直播系统中，实时排行信息包含直播间在线用户列表，各种礼物排行榜，弹幕消息（可以理解为按消息维度的消息排行榜）等信息，适合使用 Redis 中的 Sorted Set 结构进行存储。

## 3. Redis 的线程模型

Redis 内部使用文件事件处理器 `file event handler`，这个文件事件处理器是单线程的，所以 Redis 才叫做单线程的模型。它采用 IO 多路复用机制同时监听多个 socket，根据 socket 上的事件来选择对应的事件处理器进行处理。

文件事件处理器的结构包含 4 个部分：

* **多个 socket**
* **IO 多路复用程序**
* **文件事件分派器**
* **事件处理器**（连接应答处理器、命令请求处理器、命令回复处理器）

多个 socket 可能会并发产生不同的操作，每个操作对应不同的文件事件，但是 IO 多路复用程序会监听多个 socket，会将 socket 产生的事件放入队列中排队，事件分派器每次从队列中取出一个事件，把该事件交给对应的事件处理器进行处理。

## 4. Redis 设置过期时间

Redis 中有个设置时间过期的功能，即对存储在 Redis 数据库中的值可以设置一个过期时间。作为一个缓存数据库，这是非常实用的。如我们一般项目中的 token 或者一些登录信息，尤其是短信验证码都是有时间限制的，按照传统的数据库处理方式，一般都是自己判断过期，这样无疑会严重影响项目性能。

我们 set key 的时候，都可以给一个 expire time，就是过期时间，通过过期时间我们可以指定这个 key 可以存活的时间。

如果假设你设置了一批 key 只能存活 1 个小时，那么接下来 1 小时后，Redis 是怎么对这批 key 进行删除的？

### 4.1. 定期删除

Redis 默认是每隔 100ms 就随机抽取一些设置了过期时间的 key，检查其是否过期，如果过期就删除。注意这里是随机抽取的。为什么要随机呢？你想一想假如 Redis 存了几十万个 key ，每隔 100ms 就遍历所有的设置过期时间的 key 的话，就会给 CPU 带来很大的负载！

### 4.2. 惰性删除

定期删除可能会导致很多过期 key 到了时间并没有被删除掉。所以就有了惰性删除。假如你的过期 key，靠定期删除没有被删除掉，还停留在内存里，除非你的系统去查一下那个 key，才会被 Redis 给删除掉。这就是所谓的惰性删除，也是够懒的哈！

## 5. Redis 内存淘汰机制

Redis 提供 6 种数据淘汰策略：

1. **volatile-lru**：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
2. **volatile-ttl**：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
3. **volatile-random**：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
4. **allkeys-lru**：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 key（这个是最常用的）
5. **allkeys-random**：从数据集（server.db[i].dict）中任意选择数据淘汰
6. **no-eviction**：禁止驱逐数据，也就是说当内存不足以容纳新写入数据时，新写入操作会报错。

4.0 版本后增加以下两种：

1. **volatile-lfu**：从已设置过期时间的数据集(server.db[i].expires)中挑选最不经常使用的数据淘汰
2. **allkeys-lfu**：当内存不足以容纳新写入数据时，在键空间中，移除最不经常使用的 key

## 6. Redis 持久化机制

Redis 支持持久化，而且支持两种不同的持久化操作。Redis 的一种持久化方式叫快照（snapshotting，RDB），另一种方式是只追加文件（append-only file,AOF）。这两种方法各有千秋，下面我会详细这两种持久化方法是什么，怎么用，如何选择适合自己的持久化方法。

### 6.1. RDB（snapshotting）持久化

Redis 可以通过创建快照来获得存储在内存里面的数据在某个时间点上的副本。Redis 创建快照之后，可以对快照进行备份，可以将快照复制到其他服务器从而创建具有相同数据的服务器副本（Redis 主从结构，主要用来提高 Redis 性能），还可以将快照留在原地以便重启服务器的时候使用。

快照持久化是 Redis 默认采用的持久化方式，在 Redis.conf 配置文件中默认有此下配置：

```
save 900 1           #在900秒(15分钟)之后，如果至少有1个key发生变化，Redis就会自动触发BGSAVE命令创建快照。

save 300 10          #在300秒(5分钟)之后，如果至少有10个key发生变化，Redis就会自动触发BGSAVE命令创建快照。

save 60 10000        #在60秒(1分钟)之后，如果至少有10000个key发生变化，Redis就会自动触发BGSAVE命令创建快照。
```

### 6.2. AOF（append-only file）持久化

与快照持久化相比，AOF 持久化 的实时性更好，因此已成为主流的持久化方案。默认情况下 Redis 没有开启 AOF（append only file）方式的持久化，可以通过 appendonly 参数开启：

```
appendonly yes
```

开启 AOF 持久化后每执行一条会更改 Redis 中的数据的命令，Redis 就会将该命令写入硬盘中的 AOF 文件。AOF 文件的保存位置和 RDB 文件的位置相同，都是通过 dir 参数设置的，默认的文件名是 appendonly.aof。

在 Redis 的配置文件中存在三种不同的 AOF 持久化方式，它们分别是：

```
appendfsync always    #每次有数据修改发生时都会写入AOF文件,这样会严重降低Redis的速度
appendfsync everysec  #每秒钟同步一次，显示地将多个写命令同步到硬盘
appendfsync no        #让操作系统决定何时进行同步
```

为了兼顾数据和写入性能，用户可以考虑 appendfsync everysec 选项 ，让 Redis 每秒同步一次 AOF 文件，Redis 性能几乎没受到任何影响。而且这样即使出现系统崩溃，用户最多只会丢失一秒之内产生的数据。当硬盘忙于执行写入操作的时候，Redis 还会优雅的放慢自己的速度以便适应硬盘的最大写入速度。

### 6.3. Redis 4.0 对于持久化机制的优化

Redis 4.0 开始支持 RDB 和 AOF 的混合持久化（默认关闭，可以通过配置项 `aof-use-rdb-preamble` 开启）。

如果把混合持久化打开，AOF 重写的时候就直接把 RDB 的内容写到 AOF 文件开头。这样做的好处是可以结合 RDB 和 AOF 的优点, 快速加载同时避免丢失过多的数据。当然缺点也是有的， AOF 里面的 RDB 部分是压缩格式不再是 AOF 格式，可读性较差。

## 7. Redis 事务机制

Redis 通过 MULTI、EXEC、WATCH 等命令来实现事务(transaction)功能。事务提供了一种将多个命令请求打包，然后一次性、按顺序地执行多个命令的机制，并且在事务执行期间，服务器不会中断事务而改去执行其他客户端的命令请求，它会将事务中的所有命令都执行完毕，然后才去处理其他客户端的命令请求。

在传统的关系式数据库中，常常用 ACID 性质来检验事务功能的可靠性和安全性。在 Redis 中，事务总是具有原子性（Atomicity）、一致性（Consistency）和隔离性（Isolation），并且当 Redis 运行在某种特定的持久化模式下时，事务也具有持久性（Durability）。

**补充注意**：

> Redis 同一个事务中如果有一条命令执行失败，其后的命令仍然会被执行，没有回滚。

## 8. Redis 发布/订阅功能

### 8.1. PubSub 简介

基于 list 结构的消息队列，是一种 Publisher 与 Consumer 点对点的强关联关系，Redis 为了消除这样的强关联，引入了另一种概念：频道 (channel)

![](https://user-gold-cdn.xitu.io/2020/7/15/1734edf8716cf207?w=1240&h=443&f=png&s=41686)

当 Publisher 往 channel 中发布消息时，关注了指定 channel 的 Consumer 就能够同时受到消息。但这里的 问题 是，消费者订阅一个频道是必须 明确指定频道名称 的，这意味着，如果我们想要 订阅多个 频道，那么就必须 显式地关注多个 名称。

为了简化订阅的繁琐操作，Redis 提供了 模式订阅 的功能 Pattern Subscribe，这样就可以 一次性关注多个频道 了，即使生产者新增了同模式的频道，消费者也可以立即受到消息：

![](https://user-gold-cdn.xitu.io/2020/7/15/1734edfadf5b9b5f?w=1240&h=443&f=png&s=78916)

例如上图中，所有 位于图片下方的 Consumer 都能够受到消息。

Publisher 往 wmyskxz.chat 这个 channel 中发送了一条消息，不仅仅关注了这个频道的 Consumer 1 和 Consumer 2 能够受到消息，图片中的两个 channel 都和模式 wmyskxz.* 匹配，所以 Redis 此时会同样发送消息给订阅了 wmyskxz.* 这个模式的 Consumer 3 和关注了在这个模式下的另一个频道 wmyskxz.log 下的 Consumer 4 和 Consumer 5。

另一方面，如果接收消息的频道是 wmyskxz.chat，那么 Consumer 3 也会受到消息。

**PubSub 的缺点**

尽管 Redis 实现了 PubSub 模式来达到了 多播消息队列 的目的，但在实际的消息队列的领域，几乎 找不到特别合适的场景，因为它的缺点十分明显：

* **没有 Ack 机制，也不保证数据的连续**： PubSub 的生产者传递过来一个消息，Redis 会直接找到相应的消费者传递过去。如果没有一个消费者，那么消息会被直接丢弃。如果开始有三个消费者，其中一个突然挂掉了，过了一会儿等它再重连时，那么重连期间的消息对于这个消费者来说就彻底丢失了。
* **不持久化消息**： 如果 Redis 停机重启，PubSub 的消息是不会持久化的，毕竟 Redis 宕机就相当于一个消费者都没有，所有的消息都会被直接丢弃。

基于上述缺点，Redis 的作者甚至单独开启了一个 Disque 的项目来专门用来做多播消息队列，不过该项目目前好像都没有成熟。不过后来在 2018 年 6 月，Redis 5.0 新增了 Stream 数据结构，这个功能给 Redis 带来了 持久化消息队列，从此 PubSub 作为消息队列的功能可以说是就消失了。

### 8.2. 强大的 Stream | 持久化的发布/订阅系统

Redis Stream 从概念上来说，就像是一个 仅追加内容 的 消息链表，把所有加入的消息都一个一个串起来，每个消息都有一个唯一的 ID 和内容，这很简单，让它复杂的是从 Kafka 借鉴的另一种概念：消费者组(Consumer Group) (思路一致，实现不同)：

![](https://user-gold-cdn.xitu.io/2020/7/15/1734edffa2920934?w=1240&h=577&f=png&s=82438)

上图就展示了一个典型的 Stream 结构。每个 Stream 都有唯一的名称，它就是 Redis 的 key，在我们首次使用 xadd 指令追加消息时自动创建。我们对图中的一些概念做一下解释：

* **Consumer Group**：消费者组，可以简单看成记录流状态的一种数据结构。消费者既可以选择使用 XREAD 命令进行 独立消费，也可以多个消费者同时加入一个消费者组进行 组内消费。同一个消费者组内的消费者共享所有的 Stream 信息，同一条消息只会有一个消费者消费到，这样就可以应用在分布式的应用场景中来保证消息的唯一性。
* **last_delivered_id**：用来表示消费者组消费在 Stream 上 消费位置 的游标信息。每个消费者组都有一个 Stream 内 唯一的名称，消费者组不会自动创建，需要使用 XGROUP CREATE 指令来显式创建，并且需要指定从哪一个消息 ID 开始消费，用来初始化 last_delivered_id 这个变量。
* **pending_ids**：每个消费者内部都有的一个状态变量，用来表示 已经 被客户端 获取，但是 还没有 ack 的消息。记录的目的是为了 保证客户端至少消费了消息一次，而不会在网络传输的中途丢失而没有对消息进行处理。如果客户端没有 ack，那么这个变量里面的消息 ID 就会越来越多，一旦某个消息被 ack，它就会对应开始减少。这个变量也被 Redis 官方称为 PEL (Pending Entries List)。

**消息 ID**:

消息 ID 如果是由 XADD 命令返回自动创建的话，那么它的格式会像这样：timestampInMillis-sequence (毫秒时间戳-序列号)，例如 1527846880585-5，它表示当前的消息是在毫秒时间戳 1527846880585 时产生的，并且是该毫秒内产生的第 5 条消息。

这些 ID 的格式看起来有一些奇怪，为什么要使用时间来当做 ID 的一部分呢？ 一方面，我们要 满足 ID 自增 的属性，另一方面，也是为了 支持范围查找 的功能。由于 ID 和生成消息的时间有关，这样就使得在根据时间范围内查找时基本上是没有额外损耗的。
当然消息 ID 也可以由客户端自定义，但是形式必须是 "整数-整数"，而且后面加入的消息的 ID 必须要大于前面的消息 ID。

**消息内容**:

消息内容就是普通的键值对，形如 hash 结构的键值对。

## 9. Redis 集群模式

### 9.1. 主从复制

主从复制，是指将一台 Redis 服务器的数据，复制到其他的 Redis 服务器。前者称为 主节点(master)，后者称为 从节点(slave)。且数据的复制是 单向 的，只能由主节点到从节点。Redis 主从复制支持 主从同步 和 从从同步 两种，后者是 Redis 后续版本新增的功能，以减轻主节点的同步负担。

![](https://user-gold-cdn.xitu.io/2020/7/15/1734ef0a2c737aaa?w=1240&h=390&f=png&s=44524)

**主从复制主要的作用**

* **数据冗余**： 主从复制实现了数据的热备份，是持久化之外的一种数据冗余方式。
* **故障恢复**： 当主节点出现问题时，可以由从节点提供服务，实现快速的故障恢复 (实际上是一种服务的冗余)。
* **负载均衡**： 在主从复制的基础上，配合读写分离，可以由主节点提供写服务，由从节点提供读服务 （即写 Redis 数据时应用连接主节点，读 Redis 数据时应用连接从节点），分担服务器负载。尤其是在写少读多的场景下，通过多个从节点分担读负载，可以大大提高 Redis 服务器的并发量。
* **高可用基石**： 除了上述作用以外，主从复制还是哨兵和集群能够实施的 基础，因此说主从复制是 Redis 高可用的基础。

**主从复制实现原理简析**

![](https://user-gold-cdn.xitu.io/2020/7/15/1734ef0e1bf5fe8d?w=1240&h=620&f=png&s=161169)

**身份验证 | 主从复制安全问题**

通过在 主节点 配置 requirepass 来设置密码，这样就必须在 从节点 中对应配置好 masterauth 参数 (与主节点 requirepass 保持一致) 才能够进行正常复制了。

**SYNC 命令是一个非常耗费资源的操作**

每次执行 SYNC 命令，主从服务器需要执行如下动作：

1. 主服务器 需要执行 BGSAVE 命令来生成 RDB 文件，这个生成操作会 消耗 主服务器大量的 CPU、内存和磁盘 I/O 的资源；
2. 主服务器 需要将自己生成的 RDB 文件 发送给从服务器，这个发送操作会 消耗 主服务器 大量的网络资源 (带宽和流量)，并对主服务器响应命令请求的时间产生影响；
3. 接收到 RDB 文件的 从服务器 需要载入主服务器发来的 RBD 文件，并且在载入期间，从服务器 会因为阻塞而没办法处理命令请求；

特别是当出现 断线重复制 的情况是时，为了让从服务器补足断线时确实的那一小部分数据，却要执行一次如此耗资源的 SYNC 命令，显然是不合理的。

**PSYNC 命令的引入**

所以在 Redis 2.8 中引入了 PSYNC 命令来代替 SYNC，它具有两种模式：

1. 全量复制： 用于初次复制或其他无法进行部分复制的情况，将主节点中的所有数据都发送给从节点，是一个非常重型的操作；
2. 部分复制： 用于网络中断等情况后的复制，只将 中断期间主节点执行的写命令 发送给从节点，与全量复制相比更加高效。需要注意 的是，如果网络中断时间过长，导致主节点没有能够完整地保存中断期间执行的写命令，则无法进行部分复制，仍使用全量复制；

部分复制的原理主要是靠主从节点分别维护一个 复制偏移量，有了这个偏移量之后断线重连之后一比较，之后就可以仅仅把从服务器断线之后确实的这部分数据给补回来了。

### 9.2. Redis Sentinel 哨兵

![](https://user-gold-cdn.xitu.io/2020/7/15/1734ef139e3a4072?w=1240&h=531&f=png&s=60653)

上图 展示了一个典型的哨兵架构图，它由两部分组成，哨兵节点和数据节点：

* **哨兵节点**： 哨兵系统由一个或多个哨兵节点组成，哨兵节点是特殊的 Redis 节点，不存储数据；
* **数据节点**： 主节点和从节点都是数据节点；

在复制的基础上，哨兵实现了 自动化的故障恢复 功能，下方是官方对于哨兵功能的描述：

* **监控（Monitoring）**： 哨兵会不断地检查主节点和从节点是否运作正常。
* **自动故障转移（Automatic failover）**： 当 主节点 不能正常工作时，哨兵会开始 自动故障转移操作，它会将失效主节点的其中一个 从节点升级为新的主节点，并让其他从节点改为复制新的主节点。
* **配置提供者（Configuration provider）**： 客户端在初始化时，通过连接哨兵来获得当前 Redis 服务的主节点地址。
* **通知（Notification）**： 哨兵可以将故障转移的结果发送给客户端。

其中，监控和自动故障转移功能，使得哨兵可以及时发现主节点故障并完成转移。而配置提供者和通知功能，则需要在与客户端的交互中才能体现。

**客户端原理**

Jedis 客户端对哨兵提供了很好的支持。如上述代码所示，我们只需要向 Jedis 提供哨兵节点集合和 masterName ，构造 JedisSentinelPool 对象，然后便可以像使用普通 Redis 连接池一样来使用了：通过 pool.getResource() 获取连接，执行具体的命令。

在整个过程中，我们的代码不需要显式的指定主节点的地址，就可以连接到主节点；代码中对故障转移没有任何体现，就可以在哨兵完成故障转移后自动的切换主节点。之所以可以做到这一点，是因为在 JedisSentinelPool 的构造器中，进行了相关的工作；主要包括以下两点：

1. 遍历哨兵节点，获取主节点信息： 遍历哨兵节点，通过其中一个哨兵节点 + masterName 获得主节点的信息；该功能是通过调用哨兵节点的 sentinel get-master-addr-by-name 命令实现；
2. 增加对哨兵的监听： 这样当发生故障转移时，客户端便可以收到哨兵的通知，从而完成主节点的切换。具体做法是：利用 Redis 提供的 发布订阅 功能，为每一个哨兵节点开启一个单独的线程，订阅哨兵节点的 + switch-master 频道，当收到消息时，重新初始化连接池。

**如何挑选新的主服务器**

Sentinel 使用以下规则来选择新的主服务器：

1. 在失效主服务器属下的从服务器当中， 那些被标记为主观下线、已断线、或者最后一次回复 PING 命令的时间大于五秒钟的从服务器都会被 淘汰。
2. 在失效主服务器属下的从服务器当中， 那些与失效主服务器连接断开的时长超过 down-after 选项指定的时长十倍的从服务器都会被 淘汰。
3. 在 经历了以上两轮淘汰之后 剩下来的从服务器中， 我们选出 复制偏移量（replication offset）最大 的那个 从服务器 作为新的主服务器；如果复制偏移量不可用，或者从服务器的复制偏移量相同，那么 带有最小运行 ID 的那个从服务器成为新的主服务器。

### 9.3. Redis 集群架构

![](https://user-gold-cdn.xitu.io/2020/7/15/1734ef18d4922949?w=1240&h=531&f=png&s=75657)

上图 展示了 Redis Cluster 典型的架构图，集群中的每一个 Redis 节点都 互相两两相连，客户端任意 直连 到集群中的 任意一台，就可以对其他 Redis 节点进行 读写 的操作。

**集群基本原理**

![](https://user-gold-cdn.xitu.io/2020/7/15/1734ef1a74e55a81?w=1240&h=531&f=png&s=40299)

Redis 集群中内置了 16384 个哈希槽。当客户端连接到 Redis 集群之后，会同时得到一份关于这个 集群的配置信息，当客户端具体对某一个 key 值进行操作时，会计算出它的一个 Hash 值，然后**把结果对 16384 求余数**，这样每个 key 都会对应一个编号在 0-16383 之间的哈希槽，Redis 会根据节点数量 大致均等 的将哈希槽映射到不同的节点。

再结合集群的配置信息就能够知道这个 key 值应该存储在哪一个具体的 Redis 节点中，如果不属于自己管，那么就会使用一个特殊的 MOVED 命令来进行一个跳转，告诉客户端去连接这个节点以获取数据：
```
GET x
-MOVED 3999 127.0.0.1:6381
```
MOVED 指令第一个参数 3999 是 key 对应的槽位编号，后面是目标节点地址，MOVED 命令前面有一个减号，表示这是一个错误的消息。客户端在收到 MOVED 指令后，就立即纠正本地的 槽位映射表，那么下一次再访问 key 时就能够到正确的地方去获取了。

**集群的主要作用**

1. **数据分区**： 数据分区 (或称数据分片) 是集群最核心的功能。集群将数据分散到多个节点，一方面 突破了 Redis 单机内存大小的限制，存储容量大大增加；另一方面 每个主节点都可以对外提供读服务和写服务，大大提高了集群的响应能力。Redis 单机内存大小受限问题，在介绍持久化和主从复制时都有提及，例如，如果单机内存太大，bgsave 和 bgrewriteaof 的 fork 操作可能导致主进程阻塞，主从环境下主机切换时可能导致从节点长时间无法提供服务，全量复制阶段主节点的复制缓冲区可能溢出。
2. **高可用**： 集群支持主从复制和主节点的 自动故障转移 （与哨兵类似），当任一节点发生故障时，集群仍然可以对外提供服务。

### 9.4. 数据分区方案简析

**方案一：哈希值 % 节点数**

哈希取余分区思路非常简单：计算 key 的 hash 值，然后对节点数量进行取余，从而决定数据映射到哪个节点上。

不过该方案最大的问题是，当新增或删减节点时，节点数量发生变化，系统中所有的数据都需要 重新计算映射关系，引发大规模数据迁移。

**方案二：一致性哈希分区**

一致性哈希算法将 整个哈希值空间 组织成一个虚拟的圆环，范围是 [0 , 232-1]，对于每一个数据，根据 key 计算 hash 值，确数据在环上的位置，然后从此位置沿顺时针行走，找到的第一台服务器就是其应该映射到的服务器：

![](https://user-gold-cdn.xitu.io/2020/7/15/1734efa4d2068fba?w=1240&h=655&f=png&s=104288)

与哈希取余分区相比，一致性哈希分区将 增减节点的影响限制在相邻节点。以上图为例，如果在 node1 和 node2 之间增加 node5，则只有 node2 中的一部分数据会迁移到 node5；如果去掉 node2，则原 node2 中的数据只会迁移到 node4 中，只有 node4 会受影响。

一致性哈希分区的主要问题在于，当 节点数量较少 时，增加或删减节点，对单个节点的影响可能很大，造成数据的严重不平衡。还是以上图为例，如果去掉 node2，node4 中的数据由总数据的 1/4 左右变为 1/2 左右，与其他节点相比负载过高。

**方案三：带有虚拟节点的一致性哈希分区**

该方案在 一致性哈希分区的基础上，引入了 虚拟节点 的概念。Redis 集群使用的便是该方案，其中的虚拟节点称为 槽（slot）。槽是介于数据和实际节点之间的虚拟概念，每个实际节点包含一定数量的槽，每个槽包含哈希值在一定范围内的数据。

在使用了槽的一致性哈希分区中，槽是数据管理和迁移的基本单位。槽 解耦 了 数据和实际节点 之间的关系，增加或删除节点对系统的影响很小。仍以上图为例，系统中有 4 个实际节点，假设为其分配 16 个槽(0-15)；

> 槽 0-3 位于 node1；4-7 位于 node2；以此类推....

如果此时删除 node2，只需要将槽 4-7 重新分配即可，例如槽 4-5 分配给 node1，槽 6 分配给 node3，槽 7 分配给 node4；可以看出删除 node2 后，数据在其他节点的分布仍然较为均衡。

### 9.5. 节点通信机制简析

集群的建立离不开节点之间的通信，例如我们上访在 快速体验 中刚启动六个集群节点之后通过 redis-cli 命令帮助我们搭建起来了集群，实际上背后每个集群之间的两两连接是通过了 CLUSTER MEET <ip> <port> 命令发送 MEET 消息完成的，下面我们展开详细说说。

**两个端口**

在 哨兵系统 中，节点分为 数据节点 和 哨兵节点：前者存储数据，后者实现额外的控制功能。在 集群 中，没有数据节点与非数据节点之分：所有的节点都存储数据，也都参与集群状态的维护。为此，集群中的每个节点，都提供了两个 TCP 端口：

* **普通端口**： 即我们在前面指定的端口 (7000等)。普通端口主要用于为客户端提供服务 （与单机节点类似）；但在节点间数据迁移时也会使用。
* **集群端口**： 端口号是普通端口 + 10000 （10000是固定值，无法改变），如 7000 节点的集群端口为 17000。集群端口只用于节点之间的通信，如搭建集群、增减节点、故障转移等操作时节点间的通信；不要使用客户端连接集群接口。为了保证集群可以正常工作，在配置防火墙时，要同时开启普通端口和集群端口。

**Gossip 协议**

节点间通信，按照通信协议可以分为几种类型：单对单、广播、Gossip 协议等。重点是广播和 Gossip 的对比。

* 广播是指向集群内所有节点发送消息。优点 是集群的收敛速度快(集群收敛是指集群内所有节点获得的集群信息是一致的)，缺点 是每条消息都要发送给所有节点，CPU、带宽等消耗较大。
* Gossip 协议的特点是：在节点数量有限的网络中，**每个节点都 “随机” 的与部分节点通信** （并不是真正的随机，而是根据特定的规则选择通信的节点），经过一番杂乱无章的通信，每个节点的状态很快会达到一致。Gossip 协议的 优点 有负载 (比广播) 低、去中心化、容错性高 (因为通信有冗余) 等；缺点 主要是集群的收敛速度慢。

**消息类型**

集群中的节点采用 **固定频率（每秒10次） 的 定时任务** 进行通信相关的工作：判断是否需要发送消息及消息类型、确定接收节点、发送消息等。如果集群状态发生了变化，如增减节点、槽状态变更，通过节点间的通信，所有节点会很快得知整个集群的状态，使集群收敛。

节点间发送的消息主要分为 5 种：meet 消息、ping 消息、pong 消息、fail 消息、publish 消息。不同的消息类型，通信协议、发送的频率和时机、接收节点的选择等是不同的：

* **MEET 消息**： 在节点握手阶段，当节点收到客户端的 CLUSTER MEET 命令时，会向新加入的节点发送 MEET 消息，请求新节点加入到当前集群；新节点收到 MEET 消息后会回复一个 PONG 消息。
* **PING 消息**： 集群里每个节点每秒钟会选择部分节点发送 PING 消息，接收者收到消息后会回复一个 PONG 消息。PING 消息的内容是自身节点和部分其他节点的状态信息，作用是彼此交换信息，以及检测节点是否在线。PING 消息使用 Gossip 协议发送，接收节点的选择兼顾了收敛速度和带宽成本，具体规则如下：(1)随机找 5 个节点，在其中选择最久没有通信的 1 个节点；(2)扫描节点列表，选择最近一次收到 PONG 消息时间大于 cluster_node_timeout / 2 的所有节点，防止这些节点长时间未更新。
* **PONG消息**： PONG 消息封装了自身状态数据。可以分为两种：第一种 是在接到 MEET/PING 消息后回复的 PONG 消息；第二种 是指节点向集群广播 PONG 消息，这样其他节点可以获知该节点的最新信息，例如故障恢复后新的主节点会广播 PONG 消息。
* **FAIL 消息**： 当一个主节点判断另一个主节点进入 FAIL 状态时，会向集群广播这一 FAIL 消息；接收节点会将这一 FAIL 消息保存起来，便于后续的判断。
* **PUBLISH 消息**： 节点收到 PUBLISH 命令后，会先执行该命令，然后向集群广播这一消息，接收节点也会执行该 PUBLISH 命令。

### 9.6. 数据结构简析

节点为了存储集群状态而提供的数据结构中，最关键的是 clusterNode 和 clusterState 结构：前者记录了一个节点的状态，后者记录了集群作为一个整体的状态。

**clusterNode 结构**

clusterNode 结构保存了 一个节点的当前状态，包括创建时间、节点 id、ip 和端口号等。每个节点都会用一个 clusterNode 结构记录自己的状态，并为集群内所有其他节点都创建一个 clusterNode 结构来记录节点状态。

```
typedef struct clusterNode {
    //节点创建时间
    mstime_t ctime;
    //节点id
    char name[REDIS_CLUSTER_NAMELEN];
    //节点的ip和端口号
    char ip[REDIS_IP_STR_LEN];
    int port;
    //节点标识：整型，每个bit都代表了不同状态，如节点的主从状态、是否在线、是否在握手等
    int flags;
    //配置纪元：故障转移时起作用，类似于哨兵的配置纪元
    uint64_t configEpoch;
    //槽在该节点中的分布：占用16384/8个字节，16384个比特；每个比特对应一个槽：比特值为1，则该比特对应的槽在节点中；比特值为0，则该比特对应的槽不在节点中
    unsigned char slots[16384/8];
    //节点中槽的数量
    int numslots;
    …………
} clusterNode;
```

**clusterState 结构**

clusterState 结构保存了在当前节点视角下，集群所处的状态。

```
typedef struct clusterState {
    //自身节点
    clusterNode *myself;
    //配置纪元
    uint64_t currentEpoch;
    //集群状态：在线还是下线
    int state;
    //集群中至少包含一个槽的节点数量
    int size;
    //哈希表，节点名称->clusterNode节点指针
    dict *nodes;
    //槽分布信息：数组的每个元素都是一个指向clusterNode结构的指针；如果槽还没有分配给任何节点，则为NULL
    clusterNode *slots[16384];
    …………
} clusterState;
```

> [Java工程师的进阶之路 Redis篇（一）](https://juejin.im/post/6850418115840409613)<br>
> [Java工程师的进阶之路 Redis篇（二）](https://juejin.im/post/6863250340738695176)<br>