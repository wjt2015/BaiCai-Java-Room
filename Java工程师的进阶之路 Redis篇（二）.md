> 白菜Java自习室 涵盖核心知识

> [Java工程师的进阶之路 Redis篇（一）](https://juejin.im/post/6850418115840409613)<br>
> [Java工程师的进阶之路 Redis篇（二）](https://juejin.im/post/6863250340738695176)<br>

## 1. Redis 跳跃表

### 1.1. 跳跃表简介

> **跳跃表（skiplist）**是一种随机化的数据结构，由 William Pugh 在论文《Skip lists: a probabilistic alternative to balanced trees》中提出，是一种可以于平衡树媲美的层次化链表结构——查找、删除、添加等操作都可以在对数期望时间下完成。

**Redis** 的五种基本结构中，有一个叫做 **有序列表 zset** 的数据结构，它类似于 Java 中的 **SortedSet** 和 **HashMap** 的结合体，一方面它是一个 set 保证了内部 value 的唯一性，另一方面又可以给每个 value 赋予一个排序的权重值 score，来达到 排序 的目的。它的内部实现就依赖了一种叫做 **「跳跃列表」** 的数据结构。
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c5d1348450f4449dae1a573aba8f2028~tplv-k3u1fbpfcp-zoom-1.image)

**为什么使用跳跃表？**

首先，因为 zset 要支持随机的插入和删除，所以它 不宜使用数组来实现，关于排序问题，我们也很容易就想到 红黑树/ 平衡树 这样的树形结构，为什么 Redis 不使用这样一些结构呢？

1. **性能考虑**： 在高并发的情况下，树形结构需要执行一些类似于 rebalance 这样的可能涉及整棵树的操作，相对来说跳跃表的变化只涉及局部 (下面详细说)；
2. **实现考虑**： 在复杂度与红黑树相同的情况下，跳跃表实现起来更简单，看起来也更加直观；

基于以上的一些考虑，Redis 基于 William Pugh 的论文做出一些改进后采用了 **跳跃表** 这样的结构。

### 1.2. 跳跃表原理

我们先来看一个普通的链表结构：
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d596d306aa8494b90238c83709fc589~tplv-k3u1fbpfcp-zoom-1.image)

我们需要这个链表按照 score 值进行排序，这也就意味着，当我们需要添加新的元素时，我们需要定位到插入点，这样才可以继续保证链表是有序的，通常我们会使用 二分查找法，但二分查找是有序数组的，链表没办法进行位置定位，我们除了遍历整个找到第一个比给定数据大的节点为止 （时间复杂度 O(n)) 似乎没有更好的办法。
但假如我们每相邻两个节点之间就增加一个指针，让指针指向下一个节点，如下图：
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d7282cf31934f7fa1d8aa1eda3b47b6~tplv-k3u1fbpfcp-zoom-1.image)

这样所有新增的指针连成了一个新的链表，但它包含的数据却只有原来的一半 （图中的为 3，11）。
现在假设我们想要查找数据时，可以根据这条新的链表查找，如果碰到比待查找数据大的节点时，再回到原来的链表中进行查找，比如，我们想要查找 7，查找的路径则是沿着下图中标注出的红色指针所指向的方向进行的：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8bcbce35e3ef44e6bff4387d7accecfd~tplv-k3u1fbpfcp-zoom-1.image)

这是一个略微极端的例子，但我们仍然可以看到，通过新增加的指针查找，我们不再需要与链表上的每一个节点逐一进行比较，这样改进之后需要比较的节点数大概只有原来的一半。
利用同样的方式，我们可以在新产生的链表上，继续为每两个相邻的节点增加一个指针，从而产生第三层链表：
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/33330bce644f418495601f2bb8ade204~tplv-k3u1fbpfcp-zoom-1.image)

在这个新的三层链表结构中，我们试着 **查找 13**，那么沿着最上层链表首先比较的是 11，发现 11 比 13 小，于是我们就知道只需要到 11 后面继续查找，从而**一下子跳过了 11 前面的所有节点**。
可以想象，当链表足够长，这样的多层链表结构可以帮助我们跳过很多下层节点，从而加快查找的效率。

**更进一步的跳跃表**

**跳跃表 skiplist** 就是受到这种多层链表结构的启发而设计出来的。按照上面生成链表的方式，上面每一层链表的节点个数，是下面一层的节点个数的一半，这样查找过程就非常类似于一个二分查找，使得查找的时间复杂度可以降低到 O(logn)。

但是，这种方法在插入数据的时候有很大的问题。新插入一个节点之后，就会打乱上下相邻两层链表上节点个数严格的 2:1 的对应关系。如果要维持这种对应关系，就必须把新插入的节点后面的所有节点 （也包括新插入的节点） 重新进行调整，这会让时间复杂度重新蜕化成 O(n)。删除数据也有同样的问题。

skiplist 为了避免这一问题，它不要求上下相邻两层链表之间的节点个数有严格的对应关系，而是 **为每个节点随机出一个层数(level)**。比如，一个节点随机出的层数是 3，那么就把它链入到第 1 层到第 3 层这三层链表中。为了表达清楚，下图展示了如何通过一步步的插入操作从而形成一个 skiplist 的过程：
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8e573f05279148f293c0a59a0ff8e5fb~tplv-k3u1fbpfcp-zoom-1.image)

从上面的创建和插入的过程中可以看出，每一个节点的层数（level）是随机出来的，而且新插入一个节点并不会影响到其他节点的层数，因此，**插入操作只需要修改节点前后的指针，而不需要对多个节点都进行调整**，这就降低了插入操作的复杂度。

现在我们假设从我们刚才创建的这个结构中查找 23 这个不存在的数，那么查找路径会如下图：
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c15c7895991a49538637ededb29497f1~tplv-k3u1fbpfcp-zoom-1.image)

## 2. Redis 分布式锁

### 2.1. 分布式锁简介

> **锁** 是一种用来解决多个执行线程 访问共享资源 错误或数据不一致问题的工具，本质也就是 **同一时间只允许一个用户操作**。

一般情况下，我们使用分布式锁主要有两个场景：

1. **避免不同节点重复相同的工作**：比如用户执行了某个操作有可能不同节点会发送多封邮件；
2. **避免破坏数据的正确性**：如果两个节点在同一条数据上同时进行操作，可能会造成数据错误或不一致的情况出现；

能够满足这个需求的工具我们都能够使用 (就是其他应用能帮我们加锁的)：

1. **基于 MySQL 中的锁**：MySQL 本身有自带的悲观锁 for update 关键字，也可以自己实现悲观/乐观锁来达到目的；
2. **基于 Zookeeper 有序节点**：Zookeeper 允许临时创建有序的子节点，这样客户端获取节点列表时，就能够当前子节点列表中的序号判断是否能够获得锁；
3. **基于 Redis 的单线程**：由于 Redis 是单线程，所以命令会以串行的方式执行，并且本身提供了像 SETNX(set if not exists) 这样的指令，本身具有互斥性；

### 2.2. Redis 分布式锁

分布式锁类似于 "占坑"，而 Redis 的 `SETNX(SET if Not eXists)` 指令就是这样的一个操作，只允许被一个客户端占有。

**1. 锁超时**

假设现在我们有两台平行的服务 A B，其中 **A 服务在 获取锁之后突然挂了**，那么 B 服务就永远无法获取到锁了：

所以我们需要额外设置一个超时时间，来保证服务的可用性。

但是另一个问题随即而来：**如果在加锁和释放锁之间的逻辑执行得太长，以至于超出了锁的超时限制**，也会出现问题。因为这时候第一个线程持有锁过期了，而临界区的逻辑还没有执行完，与此同时第二个线程就提前拥有了这把锁，导致临界区的代码不能得到严格的串行执行。

为了避免这个问题，**Redis 分布式锁不要用于较长时间的任务**。如果真的偶尔出现了问题，造成的数据小错乱可能就需要人工的干预。

有一个稍微安全一点的方案是 将锁的 value 值设置为一个随机数，释放锁时先匹配随机数是否一致，然后再删除 key，这是为了 确保当前线程占有的锁不会被其他线程释放，除非这个锁是因为过期了而被服务器自动释放的。

但是匹配 value 和删除 key 在 Redis 中并不是一个原子性的操作，也没有类似保证原子性的指令，所以可能需要使用像 Lua 这样的脚本来处理了，因为 Lua 脚本可以 保证多个指令的原子性执行。

**2. 单点/多点问题**

如果 Redis 采用单机部署模式，那就意味着当 Redis 故障了，就会导致整个服务不可用。

而如果采用主从模式部署，我们想象一个这样的场景：服务 A 申请到一把锁之后，如果作为主机的 Redis 宕机了，那么 服务 B 在申请锁的时候就会从从机那里获取到这把锁，为了解决这个问题，Redis 作者提出了一种 RedLock 红锁 的算法 (Redission 同 Jedis)：
```
// 三个 Redis 集群
RLock lock1 = redissionInstance1.getLock("lock1");
RLock lock2 = redissionInstance2.getLock("lock2");
RLock lock3 = redissionInstance3.getLock("lock3");

RedissionRedLock lock = new RedissionLock(lock1, lock2, lock2);
lock.lock();
// do something....
lock.unlock();
```

### 2.3. Redlock 分布式锁

Redis 官方站这篇文章提出了一种权威的基于 Redis 实现分布式锁的方式名叫 Redlock，此种方式比原先的单节点的方法更安全。它可以保证以下特性：

1. **安全特性**：互斥访问，即永远只有一个 client 能拿到锁
2. **避免死锁**：最终 client 都可能拿到锁，不会出现死锁的情况，即使原本锁住某资源的 client crash 了或者出现了网络分区
3. **容错性**：只要大部分 Redis 节点存活就可以正常提供服务

**单节点实现：**

> SET resource_name my_random_value NX PX 30000

主要依靠上述命令，该命令仅当 Key 不存在时（NX保证）set 值，并且设置过期时间 3000ms （PX保证），值 my_random_value 必须是所有 client 和所有锁请求发生期间唯一的，释放锁的逻辑是：
```
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```
上述实现可以避免释放另一个client创建的锁，如果只有 del 命令的话，那么如果 client1 拿到 lock1 之后因为某些操作阻塞了很长时间，此时 Redis 端 lock1 已经过期了并且已经被重新分配给了 client2，那么 client1 此时再去释放这把锁就会造成 client2 原本获取到的锁被 client1 无故释放了，但现在为每个 client 分配一个 unique 的 string 值可以避免这个问题。至于如何去生成这个 unique string，方法很多随意选择一种就行了。

**Redlock 算法**:

算法很易懂，起 5 个 master 节点，分布在不同的机房尽量保证可用性。为了获得锁，client 会进行如下操作：

1. 得到当前的时间，微秒单位。
2. 尝试顺序地在 5 个实例上申请锁，当然需要使用相同的 key 和 random value，这里一个 client 需要合理设置与 master 节点沟通的 timeout 大小，避免长时间和一个 fail 了的节点浪费时间。
3. 当 client 在大于等于 3 个 master 上成功申请到锁的时候，且它会计算申请锁消耗了多少时间，这部分消耗的时间采用获得锁的当下时间减去第一步获得的时间戳得到，如果锁的持续时长（lock validity time）比流逝的时间多的话，那么锁就真正获取到了。
4. 如果锁申请到了，那么锁真正的 lock validity time 应该是 origin（lock validity time） - 申请锁期间流逝的时间。
5. 如果 client 申请锁失败了，那么它就会在少部分申请成功锁的 master 节点上执行释放锁的操作，重置状态。

**失败重试**:

如果一个 client 申请锁失败了，那么它需要稍等一会在重试避免多个 client 同时申请锁的情况，最好的情况是一个 client 需要几乎同时向 5 个 master 发起锁申请。另外就是如果 client 申请锁失败了它需要尽快在它曾经申请到锁的 master 上执行 unlock 操作，便于其他 client 获得这把锁，避免这些锁过期造成的时间浪费，当然如果这时候网络分区使得 client 无法联系上这些 master，那么这种浪费就是不得不付出的代价了。

**释放锁**:

释放锁操作很简单，就是依次释放所有节点上的锁就行了。

**性能、崩溃恢复和 fsync**:

如果我们的节点没有持久化机制，client 从 5 个 master 中的 3 个处获得了锁，然后其中一个重启了，这是注意 整个环境中又出现了 3 个 master 可供另一个 client 申请同一把锁！ 违反了互斥性。如果我们开启了 AOF 持久化那么情况会稍微好转一些，因为 Redis 的过期机制是语义层面实现的，所以在 server 挂了的时候时间依旧在流逝，重启之后锁状态不会受到污染。但是考虑断电之后呢，AOF部分命令没来得及刷回磁盘直接丢失了，除非我们配置刷回策略为 fsnyc = always，但这会损伤性能。解决这个问题的方法是，当一个节点重启之后，我们规定在 max TTL 期间它是不可用的，这样它就不会干扰原本已经申请到的锁，等到它 crash 前的那部分锁都过期了，环境不存在历史锁了，那么再把这个节点加进来正常工作。

## 3. Redis 布隆过滤器

### 3.1. 布隆过滤器简介

> **布隆过滤器（Bloom Filter）** 是 1970 年由布隆提出的。它 实际上 是一个很长的二进制向量和一系列随机映射函数 (下面详细说)，实际上你也可以把它 简单理解 为一个不怎么精确的 set 结构，当你使用它的 contains 方法判断某个对象是否存在时，它可能会误判。但是布隆过滤器也不是特别不精确，只要参数设置的合理，它的精确度可以控制的相对足够精确，只会有小小的误判概率。

当布隆过滤器说某个值存在时，这个值 可能不存在；当它说不存在时，那么 一定不存在。打个比方，当它说不认识你时，那就是真的不认识，但是当它说认识你的时候，可能是因为你长得像它认识的另外一个朋友 (脸长得有些相似)，所以误判认识你。

基于上述的功能，我们大致可以把布隆过滤器用于以下的场景之中：

1. **大数据判断是否存在**：这就可以实现出上述的去重功能，如果你的服务器内存足够大的话，那么使用 HashMap 可能是一个不错的解决方案，理论上时间复杂度可以达到 O(1 的级别，但是当数据量起来之后，还是只能考虑布隆过滤器。
2. **解决缓存穿透**：我们经常会把一些热点数据放在 Redis 中当作缓存，例如产品详情。 通常一个请求过来之后我们会先查询缓存，而不用直接读取数据库，这是提升性能最简单也是最普遍的做法，但是 如果一直请求一个不存在的缓存，那么此时一定不存在缓存，那就会有 大量请求直接打到数据库 上，造成 缓存穿透，布隆过滤器也可以用来解决此类问题。
3. **爬虫/ 邮箱等系统的过滤**：平时不知道你有没有注意到有一些正常的邮件也会被放进垃圾邮件目录中，这就是使用布隆过滤器 误判 导致的。

### 3.2. 布隆过滤器原理

> 布隆过滤器 **本质上 是由长度为 m 的位向量或位列表（仅包含 0 或 1 位值的列表）组成，最初所有的值均设置为 0**。

所以我们先来创建一个稍微长一些的位向量用作展示：

当我们向布隆过滤器中添加数据时，**会使用 多个 hash 函数对 key 进行运算，算得一个证书索引值，然后对位数组长度进行取模运算得到一个位置，每个 hash 函数都会算得一个不同的位置**。再把位数组的这几个位置都置为 1 就完成了 add 操作，例如，我们添加一个 wmyskxz：

向布隆过滤器查查询 key 是否存在时，跟 add 操作一样，**会把这个 key 通过相同的多个 hash 函数进行运算，查看 对应的位置 是否 都 为 1，只要有一个位为 0，那么说明布隆过滤器中这个 key 不存在**。如果这几个位置都是 1，并不能说明这个 key 一定存在，只能说极有可能存在，因为这些位置的 1 可能是因为其他的 key 存在导致的。

就比如我们在 add 了一定的数据之后，查询一个 不存在 的 key：

很明显，1/3/5 这几个位置的 1 是因为上面第一次添加的 wmyskxz 而导致的，所以这里就存在 误判。幸运的是，布隆过滤器有一个可以预判误判率的公式，比较复杂，感兴趣的朋友可以自行去阅读，比较烧脑... 只需要记住以下几点就好了：

1. 使用时 **不要让实际元素数量远大于初始化数量**；
2. 当实际元素数量超过初始化数量时，应该对布隆过滤器进行 重建，重新分配一个 size 更大的过滤器，再将所有的历史元素批量 add 进行；

### 3.3. 布隆过滤器使用

Redis 官方 提供的布隆过滤器到了 **Redis 4.0 提供了插件功能**之后才正式登场。布隆过滤器作为一个插件加载到 Redis Server 中，给 Redis 提供了强大的布隆去重功能。

布隆过滤器有两个基本指令，`bf.add` 添加元素，`bf.exists` 查询元素是否存在，它的用法和 set 集合的 `sadd` 和 `sismember` 差不多。注意 `bf.add` 只能一次添加一个元素，如果想要一次添加多个，就需要用到 `bf.madd` 指令。同样如果需要一次查询多个元素是否存在，就需要用到 `bf.mexists` 指令。
```
127.0.0.1:6379> bf.add codehole user1
(integer) 1
127.0.0.1:6379> bf.add codehole user2
(integer) 1
127.0.0.1:6379> bf.add codehole user3
(integer) 1
127.0.0.1:6379> bf.exists codehole user1
(integer) 1
127.0.0.1:6379> bf.exists codehole user2
(integer) 1
127.0.0.1:6379> bf.exists codehole user3
(integer) 1
127.0.0.1:6379> bf.exists codehole user4
(integer) 0
127.0.0.1:6379> bf.madd codehole user4 user5 user6
1) (integer) 1
2) (integer) 1
3) (integer) 1
127.0.0.1:6379> bf.mexists codehole user4 user5 user6 user7
1) (integer) 1
2) (integer) 1
3) (integer) 1
4) (integer) 0
```

## 4. Redis HyperLogLog

### 4.1. HyperLogLog 简介

> HyperLogLog 是最早由 Flajolet 及其同事在 2007 年提出的一种 估算基数的近似最优算法。但跟原版论文不同的是，好像很多书包括 Redis 作者都把它称为一种 新的数据结构(new datastruct) (算法实现确实需要一种特定的数据结构来实现)。

**1. 关于基数统计:**

**基数统计(Cardinality Counting)** 通常是用来统计一个集合中不重复的元素个数。

思考这样的一个场景： 如果你负责开发维护一个大型的网站，有一天老板找产品经理要网站上每个网页的 **UV(独立访客，每个用户每天只记录一次)**，然后让你来开发这个统计模块，你会如何实现？

如果统计 **PV(浏览量，用户没点一次记录一次)**，那非常好办，给每个页面配置一个独立的 Redis 计数器就可以了，把这个计数器的 key 后缀加上当天的日期。这样每来一个请求，就执行 INCRBY 指令一次，最终就可以统计出所有的 PV 数据了。

但是 UV 不同，它要去重，同一个用户一天之内的多次访问请求只能计数一次。这就要求了每一个网页请求都需要带上用户的 ID，无论是登录用户还是未登录的用户，都需要一个唯一 ID 来标识。

你也许马上就想到了一个 简单的解决方案：那就是 **为每一个页面设置一个独立的 set 集合** 来存储所有当天访问过此页面的用户 ID。但这样的 问题 就是：

1. **存储空间巨大**： 如果网站访问量一大，你需要用来存储的 set 集合就会非常大，如果页面再一多.. 为了一个去重功能耗费的资源就可以直接让你 老板打死你；
2. **统计比较复杂**： 这么多 set 集合如果要聚合统计一下，又是一个复杂的事情；

**基数统计的常用方法：**

对于上述这样需要 基数统计 的事情，通常来说有两种比 set 集合更好的解决方案：

**第一种：B 树**

B 树最大的优势就是插入和查找效率很高，如果用 B 树存储要统计的数据，可以快速判断新来的数据是否存在，并快速将元素插入 B 树。要计算基础值，只需要计算 B 树的节点个数就行了。

不过将 B 树结构维护到内存中，能够解决统计和计算的问题，但是 并没有节省内存。

**第二种：bitmap**

bitmap 可以理解为通过一个 bit 数组来存储特定数据的一种数据结构，每一个 bit 位都能独立包含信息，bit 是数据的最小存储单位，因此能大量节省空间，也可以将整个 bit 数据一次性 load 到内存计算。如果定义一个很大的 bit 数组，基础统计中 每一个元素对应到 bit 数组中的一位，例如：

bitmap 还有一个明显的优势是 可以轻松合并多个统计结果，只需要对多个结果求异或就可以了，也可以大大减少存储内存。可以简单做一个计算，如果要统计 1 亿 个数据的基数值，大约需要的内存：`100_000_000/ 8/ 1024/ 1024 ≈ 12 M`，如果用 32 bit 的 int 代表 每一个 统计的数据，大约需要内存：`32 * 100_000_000/ 8/ 1024/ 1024 ≈ 381 M`

可以看到 bitmap 对于内存的节省显而易见，但仍然不够。统计一个对象的基数值就需要 12 M，如果统计 1 万个对象，就需要接近 120 G，对于大数据的场景仍然不适用。

**2. 概率算法：**

实际上目前还没有发现更好的在 大数据场景 中 准确计算 基数的高效算法，因此在不追求绝对精确的情况下，使用概率算法算是一个不错的解决方案。

概率算法 不直接存储 数据集合本身，通过一定的 概率统计方法预估基数值，这种方法可以大大节省内存，同时保证误差控制在一定范围内。目前用于基数计数的概率算法包括:

1. **Linear Counting(LC)**：早期的基数估计算法，LC 在空间复杂度方面并不算优秀，实际上 LC 的空间复杂度与上文中简单 bitmap 方法是一样的（但是有个常数项级别的降低），都是 O(Nmax)
2. **LogLog Counting(LLC)**：LogLog Counting 相比于 LC 更加节省内存，空间复杂度只有 O(log2(log2(Nmax)))
3. **HyperLogLog Counting(HLL)**：HyperLogLog Counting 是基于 LLC 的优化和改进，在同样空间复杂度情况下，能够比 LLC 的基数估计误差更小
其中，HyperLogLog 的表现是惊人的，上面我们简单计算过用 bitmap 存储 1 个亿 统计数据大概需要 12 M 内存，而在 HyperLoglog 中，只需要不到 1 K 内存就能够做到！在 Redis 中实现的 HyperLoglog 也只需要 12 K 内存，在 标准误差 0.81% 的前提下，能够统计 264 个数据！

### 4.2. HyperLogLog 原理

们来思考一个抛硬币的游戏：你连续掷 n 次硬币，然后说出其中连续掷为正面的最大次数，我来猜你一共抛了多少次。

我们给定一系列的随机整数，记录下低位连续零位的最大长度 K，即为图中的 `maxbit`，通过这个 K 值我们就可以估算出随机数的数量 N。会发现 K 和 N 的对数之间存在显著的线性相关性：**N 约等于 2k**。

**分桶平均**

**如果 N 介于 2k 和 2k+1 之间，用这种方式估计的值都等于 2k，这明显是不合理的**，所以我们可以使用多个 BitKeeper 进行加权估计，就可以得到一个比较准确的值了。

这个过程有点 类似于选秀节目里面的打分，一堆专业评委打分，但是有一些评委因为自己特别喜欢所以给高了，一些评委又打低了，所以一般都要 **屏蔽最高分和最低分**，然后 **再计算平均值**，这样的出来的分数就差不多是公平公正的了。
上述代码就有 1024 个 "评委"，并且在计算平均值的时候，采用了 **调和平均数，也就是倒数的平均值**，它能有效地平滑离群值的影响。

**真实的 HyperLogLog**

有一个神奇的网站，可以动态地让你观察到 HyperLogLog 的算法到底是怎么执行的：http://content.research.neustar.biz/blog/hll.html

其中的一些概念这里稍微解释一下，您就可以自行去点击 step 来观察了：

* m 表示分桶个数： 从图中可以看到，这里分成了 64 个桶；
* 蓝色的 bit 表示在桶中的位置： 例如图中的 101110 实则表示二进制的 46，所以该元素被统计在中间大表格 Register Values 中标红的第 46 个桶之中；
* 绿色的 bit 表示第一个 1 出现的位置： 从图中可以看到标绿的 bit 中，从右往左数，第一位就是 1，所以在 Register Values 第 46 个桶中写入 1；
* 红色 bit 表示绿色 bit 的值的累加： 下一个出现在第 46 个桶的元素值会被累加；

### 4.3. Redis 中的 HyperLogLog

在 Redis 的 HyperLogLog 实现中，用的是 16384 个桶，即：214，也就是说，就像上面网站中间那个 `Register Values` 大表格有 16384 格。

而Redis 最大能够统计的数据量是 264，即每个桶的 `maxbit` 需要 6 个 bit 来存储，最大可以表示 `maxbit = 63`，于是总共占用内存就是：`(214) x 6 / 8` (每个桶 6 bit，而这么多桶本身要占用 16384 bit，再除以 8 转换成 KB),算出来的结果就是 `12 KB`。

Redis 对于内存的优化非常变态，当 计数比较小 的时候，大多数桶的计数值都是 零，这个时候 Redis 就会适当节约空间，转换成另外一种 稀疏存储方式，与之相对的，正常的存储模式叫做 密集存储，这种方式会恒定地占用 12 KB。

HyperLogLog 提供了两个指令 `PFADD` 和 `PFCOUNT`，字面意思就是一个是增加，另一个是获取计数。`PFADD` 和 set 集合的 `SADD` 的用法是一样的，来一个用户 ID，就将用户 ID 塞进去就是，`PFCOUNT` 和 `SCARD` 的用法是一致的，直接获取计数值：
```
> PFADD codehole user1
(interger) 1
> PFCOUNT codehole
(integer) 1
> PFADD codehole user2
(integer) 1
> PFCOUNT codehole
(integer) 2
> PFADD codehole user3
(integer) 1
> PFCOUNT codehole
(integer) 3
> PFADD codehole user4 user 5
(integer) 1
> PFCOUNT codehole
(integer) 5
```

> [Java工程师的进阶之路 Redis篇（一）](https://juejin.im/post/6850418115840409613)<br>
> [Java工程师的进阶之路 Redis篇（二）](https://juejin.im/post/6863250340738695176)<br>