> 白菜Java自习室 涵盖核心知识

> [Java工程师的进阶之路 Zookeeper篇（一）](https://juejin.im/post/6870052086239887368)<br>
> [Java工程师的进阶之路 Zookeeper篇（二）](https://juejin.im/post/6870067849021358093)<br>

ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务，它包含一个简单的原语集，分布式应用程序可以基于它实现同步服务，配置维护和 命名服务等。Zookeeper是hadoop的一个子项目，其发展历程无需赘述。在分布式应用中，由于工程师不能很好地使用锁机制，以及基于消息的协调 机制不适合在某些应用中使用，因此需要有一种可靠的、可扩展的、分布式的、可配置的协调机制来统一系统的状态。Zookeeper的目的就在于此。

## 1. Zookeeper 系统模型

### 1.1. 成员角色

Zookeeper中的角色主要有以下三类，如下表所示：

* **领导者（Leader）**：领导者负责进行投票的发起和决议，更新系统状态。
* **跟随者（Follower）**：跟随者用于接收客户请求并向客户端返回结果，在选主过程中参与投票。
* **观察者（Observer）**：观察者可以接收客户端连接，将写请求转发给Leader节点。但观察者不参与投票过程，只同步Leader状态。观察者目的是为了扩展系统，提高读取速度。

系统模型如图所示：

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/084c2f73689a4929ad34c91f8adff2bd~tplv-k3u1fbpfcp-zoom-1.image)

### 1.2. 设计目的

1. **最终一致性**：client不论连接到哪个Server，展示给它都是同一个视图，这是Zookeeper最重要的性能。
2. **可靠性**：具有简单、健壮、良好的性能，如果消息m被到一台服务器接受，那么它将被所有的服务器接受。
3. **实时性**：Zookeeper保证客户端将在一个时间间隔范围内获得服务器的更新信息，或者服务器失效的信息。但由于网络延时等原因，Zookeeper不能保证两个客户端能同时得到刚更新的数据，如果需要最新数据，应该在读数据之前调用sync()接口。
4. **等待无关（wait-free）**：慢的或者失效的client不得干预快速的client的请求，使得每个client都能有效的等待。
5. **原子性**：更新只能成功或者失败，没有中间状态。
6. **顺序性**：包括全局有序和偏序两种：全局有序是指如果在一台服务器上消息a在消息b前发布，则在所有Server上消息a都将在消息b前被发布；偏序是指如果一个消息b在消息a后被同一个发送者发布，a必将排在b前面。

## 2. Zookeeper 工作协议

### 2.1. CAP定理

> CAP原则又称CAP定理，指的是在一个分布式系统中，一致性（Consistency）、可用性（Availability）、分区容错性（Partition tolerance）。CAP 原则指的是，这三个要素最多只能同时实现两点，不可能三者兼顾。

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c6353fca4fd54c74b9ae3ef2e018de87~tplv-k3u1fbpfcp-zoom-1.image)

* **一致性（C）**：在分布式系统中的所有数据备份，在同一时刻是否同样的值。（等同于所有节点访问同一份最新的数据副本）
* **可用性（A）**：在集群中一部分节点故障后，集群整体是否还能响应客户端的读写请求。（对数据更新具备高可用性）
* **分区容错性（P）**：以实际效果而言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在C和A之间做出选择。

CAP原则的精髓就是要么AP，要么CP，要么AC，但是不存在CAP。如果在某个分布式系统中数据无副本， 那么系统必然满足强一致性条件， 因为只有独一数据，不会出现数据不一致的情况，此时C和P两要素具备，但是如果系统发生了网络分区状况或者宕机，必然导致某些数据不可以访问，此时可用性条件就不能被满足，即在此情况下获得了CP系统，但是CAP不可同时满足。

### 2.2. 原子广播协议

Zookeeper的核心是原子广播，这个机制保证了各个Server之间的同步。实现这个机制的协议叫做**Zab协议**。**Zab协议有两种模式，它们分 别是恢复模式（选主）和广播模式（同步）**。当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数Server完成了和 leader的状态同步以后，恢复模式就结束了。状态同步保证了leader和Server具有相同的系统状态。

为了保证事务的顺序一致性，zookeeper采用了递增的事务id号（zxid）来标识事务。所有的提议（proposal）都在被提出的时候加上 了zxid。实现中zxid是一个64位的数字，它高32位是epoch用来标识leader关系是否改变，每次一个leader被选出来，它都会有一个 新的epoch，标识当前属于那个leader的统治时期。低32位用于递增计数。

每个Server在工作过程中有三种状态：

* **LOOKING**：当前Server不知道leader是谁，正在搜寻
* **LEADING**：当前Server即为选举出来的leader
* **FOLLOWING**：leader已经选举出来，当前Server与之同步

## 3. Zookeeper 选主流程（恢复模式）

当leader崩溃或者leader失去大多数的follower，这时候zk进入**恢复模式**，恢复模式需要重新选举出一个新的leader，让所有的 Server都恢复到一个正确的状态。Zk的选举算法有两种：一种是基于**basic paxos**实现的，另外一种是基于**fast paxos**算法实现的。系统默认的选举算法为fast paxos。

### 3.1. Basic Paxos流程

1. 选举线程由当前Server发起选举的线程担任，其主要功能是对投票结果进行统计，并选出推荐的Server；
2. 选举线程首先向所有Server发起一次询问(包括自己)；
3. 选举线程收到回复后，验证是否是自己发起的询问(验证zxid是否一致)，然后获取对方的id(myid)，并存储到当前询问对象列表中，最后获取对方提议的leader相关信息(id,zxid)，并将这些信息存储到当次选举的投票记录表中；
4. 收到所有Server回复以后，就计算出zxid最大的那个Server，并将这个Server相关信息设置成下一次要投票的Server；
5. 线程将当前zxid最大的Server设置为当前Server要推荐的Leader，如果此时获胜的Server获得n/2 + 1的Server票数， 设置当前推荐的leader为获胜的Server，将根据获胜的Server相关信息设置自己的状态，否则，继续这个过程，直到leader被选举出来。

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef7cfc826b964fcf8ebf06b41fecc770~tplv-k3u1fbpfcp-zoom-1.image)

通过流程分析我们可以得出：**要使Leader获得多数Server的支持，则Server总数必须是奇数2n+1，且存活的Server的数目不得少于n+1**。

每个Server启动后都会重复以上流程。在恢复模式下，如果是刚从崩溃状态恢复的或者刚启动的server还会从磁盘快照中恢复数据和会话信息，zk会记录事务日志并定期进行快照，方便在恢复时进行状态恢复。
    
### 3.2. Fast Paxos流程

Fast Paxos流程是在选举过程中，某Server首先向所有Server提议自己要成为leader，当其它Server收到提议以后，解决epoch和 zxid的冲突，并接受对方的提议，然后向对方发送接受提议完成的消息，重复这个流程，最后一定能选举出Leader。

流程图如下所示：

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aba26174b4b5415c827edc7998e782eb~tplv-k3u1fbpfcp-zoom-1.image)

## 4. Zookeeper 同步流程（广播模式）

选完leader以后，zk就进入状态**广播模式**。

1. leader等待server连接；
2. Follower连接leader，将最大的zxid发送给leader；
3. Leader根据follower的zxid确定同步点；
4. 完成同步后通知follower 已经成为uptodate状态；
5. Follower收到uptodate消息后，又可以重新接受client的请求进行服务了。

流程图如下所示：

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/52bb4416df2e417cbe33346e069829b2~tplv-k3u1fbpfcp-zoom-1.image)

## 5. Zookeeper 工作流程（Leader和Follower）

### 5.1. Leader工作流程

Leader主要有三个功能：

1. 恢复数据；
2. 维持与Follower(含Observer)的心跳，接收Follower请求并判断Follower的请求消息类型；
3. Follower的消息类型主要有**PING消息、REQUEST消息、ACK消息、REVALIDATE消息**，根据不同的消息类型，进行不同的处理。

PING消息是指Follower(含Observer)的心跳信息；REQUEST消息是Follower发送的提议信息，包括写请求及同步请求；ACK消息是 Follower的对提议的回复，超过半数的Follower通过，则commit该提议；REVALIDATE消息是用来延长SESSION有效时间。

Leader的工作流程简图如下所示，在实际实现中，流程要比下图复杂得多，启动了三个线程来实现功能。

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b57af747c489448a871ce2a798dcac2c~tplv-k3u1fbpfcp-zoom-1.image)

### 5.2. Follower工作流程

Follower主要有四个功能：

1. 向Leader发送请求（PING消息、REQUEST消息、ACK消息、REVALIDATE消息）；
2. 接收Leader消息并进行处理；
3. 接收Client的请求，如果为写请求，发送给Leader进行投票；
4. 返回Client结果。

Follower的消息循环处理如下几种来自Leader的消息：

1. **PING消息**： 心跳消息；
2. **PROPOSAL消息**：Leader发起的提案，要求Follower投票；
3. **COMMIT消息**：服务器端最新一次提案的信息；
4. **UPTODATE消息**：表明同步完成；
5. **REVALIDATE消息**：根据Leader的REVALIDATE结果，关闭待revalidate的session还是允许其接受消息；
6. **SYNC消息**：返回SYNC结果到客户端，这个消息最初由客户端发起，用来强制得到最新的更新。

Follower的工作流程简图如下所示，在实际实现中，Follower是通过5个线程来实现功能的。

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cbb71fd541f04557838415d9f9359850~tplv-k3u1fbpfcp-zoom-1.image)

对于Observer的流程不再叙述，Observer流程和Follower的唯一不同的地方就是Observer不会参加Leader发起的投票。

> [Java工程师的进阶之路 Zookeeper篇（一）](https://juejin.im/post/6870052086239887368)<br>
> [Java工程师的进阶之路 Zookeeper篇（二）](https://juejin.im/post/6870067849021358093)<br>