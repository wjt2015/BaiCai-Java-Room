> 白菜Java自习室 涵盖核心知识

> [Java工程师的进阶之路 RocketMQ篇（一）](https://juejin.im/post/6864724887996252173)<br>
> [Java工程师的进阶之路 RocketMQ篇（二）](https://juejin.im/post/6864737331484721165)<br>

## 1. RocketMQ 简介

**RocketMQ**前身叫做MetaQ, 在MeataQ发布3.0版本的时候改名为RocketMQ，其本质上的设计思路和Kafka类似，但是和Kafka不同的是其使用Java进行开发，由于在国内的Java受众群体远远多于Scala，所以RocketMQ是很多以Java语言为主的公司的首选。同样的RocketMQ和Kafka都是Apache基金会中的顶级项目，他们社区的活跃度都非常高，项目更新迭代也非常快。

## 2. RocketMQ 架构图

对于RocketMQ的架构图，在大体上来看和Kafka并没有太多的差别，但是在很多细节上是有很多差别的，接下来会一一进行讲述。
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f683b08ce62145f4b45110fd8b316159~tplv-k3u1fbpfcp-zoom-1.image)

## 3. RocketMQ 名词解释

RocketMQ架构图中多个Producer，多个主Broker，多个从Broker，每个Producer可以对应多个Topic，每个Consumer也可以消费多个Topic。

Broker信息会上报至NameServer，Consumer会从NameServer中拉取Broker和Topic的信息。

* **Producer**：消息生产者，向Broker发送消息的客户端

* **Consumer**：消息消费者，从Broker读取消息的客户端

* **Broker**：消息中间的处理节点，这里和kafka不同，kafka的Broker没有主从的概念，都可以写入请求以及备份其他节点数据，RocketMQ只有主Broker节点才能写，一般也通过主节点读，当主节点有故障或者一些其他特殊情况才会使用从节点读，有点类似- 于mysql的主从架构。

* **Topic**：消息主题，一级消息类型，生产者向其发送消息, 消费者读取其消息。

* **Group**：分为ProducerGroup,ConsumerGroup,代表某一类的生产者和消费者，一般来说同一个服务可以作为Group,同一个Group一般来说发送和消费的消息都是一样的。

* **Tag**：Kafka中没有这个概念，Tag是属于二级消息类型，一般来说业务有关联的可以使用同一个Tag,比如订单消息队列，使用Topic_Order,Tag可以分为Tag_食品订单,Tag_服装订单等等。

* **Queue**: 在kafka中叫Partition,每个Queue内部是有序的，在RocketMQ中分为读和写两种队列，一般来说读写队列数量一致，如果不一致就会出现很多问题。

* **NameServer**：Kafka中使用的是ZooKeeper保存Broker的地址信息，以及Broker的Leader的选举，在RocketMQ中并没有采用选举Broker的策略，所以采用了无状态的NameServer来存储，由于NameServer是无状态的，集群节点之间并不会通信，所以上传数据的时候都需要向所有节点进行发送。

很多朋友都在问什么是无状态呢？状态的有无实际上就是数据是否会做存储，有状态的话数据会被持久化，无状态的服务可以理解就是一个内存服务，NameServer本身也是一个内存服务，所有数据都存储在内存中，重启之后都会丢失。

## 4. RocketMQ Topic和Queue

在RocketMQ中的**每一条消息，都有一个Topic，用来区分不同的消息**。一个主题一般会有多个消息的订阅者，当生产者发布消息到某个主题时，订阅了这个主题的消费者都可以接收到生产者写入的新消息。

在Topic中有分为了**多个Queue，这其实是我们发送/读取消息通道的最小单位**，我们发送消息都需要指定某个写入某个Queue，拉取消息的时候也需要指定拉取某个Queue，所以我们的顺序消息可以基于我们的Queue维度保持队列有序，如果想做到全局有序那么需要将Queue大小设置为1，这样所有的数据都会在Queue中有序。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa766a9d5bc9427f863a15872f9b49b0~tplv-k3u1fbpfcp-zoom-1.image)

在上图中我们的Producer会通过一些策略进行Queue的选择：

* **非顺序消息**：非顺序消息一般直接采用轮训发送的方式进行发送。

* **顺序消息**：根据某个Key比如我们常见的订单Id,用户Id，进行Hash，将同一类数据放在同一个队列中，保证我们的顺序性。

我们同一组Consumer也会根据一些策略来选Queue，常见的比如平均分配或者一致性Hash分配。
要注意的是当Consumer出现下线或者上线的时候，这里需要做重平衡，也就是Rebalance，**RocketMQ的重平衡机制如下**:

1. 定时拉取broker,topic的最新信息
2. 每隔20s做重平衡
3. 随机选取当前Topic的一个主Broker，这里要注意的是不是每次重平衡所有主Broker都会被选中，因为会存在一个Broker再多个Broker的情况。
4. 获取当前Broker，当前ConsumerGroup的所有机器ID。
5. 然后进行策略分配。

由于重平衡是定时做的，所以这里**有可能会出现某个Queue同时被两个Consumer消费，所以会出现消息重复投递**。

Kafka的重平衡机制和RocketMQ不同，Kafka的重平衡是通过Consumer和Coordinator联系来完成的，当Coordinator感知到消费组的变化，会在心跳过程中发送重平衡的信号，然后由一个ConsumerLeader进行重平衡选择，然后再由Coordinator将结果通知给所有的消费者。

**Queue读写数量不一致怎么办？**

在RocketMQ中Queue被分为读和写两种，在最开始接触RocketMQ的时候一直以为读写队列数量配置不一致不会出现什么问题的，比如当消费者机器很多的时候我们配置很多读的队列，但是实际过程中发现会出现消息无法消费和根本没有消息消费的情况。

1. 当写的队列数量大于读的队列的数量，当大于读队列这部分ID的写队列的数据会无法消费，因为不会将其分配给消费者。
2. 当读的队列数量大于写的队列数量，那么多的队列数量就不会有消息被投递进来。

## 5. RocketMQ 消费模型

一般来说消息队列的消费模型分为两种，**基于推送的消息(push)模型**和**基于拉取(poll)的消息模型**。

基于推送模型的消息系统，由消息代理记录消费状态。消息代理将消息推送到消费者后，标记这条消息为已经被消费，但是这种方式无法很好地保证消费的处理语义。比如当我们把已经把消息发送给消费者之后，由于消费进程挂掉或者由于网络原因没有收到这条消息，如果我们在消费代理将其标记为已消费，这个消息就永久丢失了。如果我们利用生产者收到消息后回复这种方法，消息代理需要记录消费状态，这种不可取。

用过RocketMQ的同学肯定不禁会想到，在RocketMQ中不是提供了两种消费者吗？

1. **MQPullConsumer和MQPushConsumer**

其中MQPushConsumer不就是我们的推模型吗？其实这两种模型都是客户端主动去拉消息，其中的实现区别如下：

* **MQPullConsumer**：每次拉取消息需要传入拉取消息的offset和每次拉取多少消息量，具体拉取哪里的消息，拉取多少是由客户端控制。

* **MQPushConsumer**：同样也是客户端主动拉取消息，但是消息进度是由服务端保存，Consumer会定时上报自己消费到哪里，所以Consumer下次消费的时候是可以找到上次消费的点，一般来说使用PushConsumer我们不需要关心offset和拉取多少数据，直接使用即可。

2. **集群消费和广播消费**

消费模式我们分为两种，集群消费，广播消费:

* **集群消费**: 同一个GroupId都属于一个集群，一般来说一条消息只会被任意一个消费者处理。

* **广播消费**：广播消费的消息会被集群中所有消费者进行消息，但是要注意一下因为广播消费的offset在服务端保存成本太高，所以客户端每一次重启都会从最新消息消费，而不是上次保存的offset。

## 6. RocketMQ 网络模型

在Kafka中使用的原生的socket实现网络通信，而**RocketMQ使用的是Netty网络框架**，现在越来越多的中间件都不会直接选择原生的socket，而是使用的Netty框架，主要得益于下面几个原因:

1. API使用简单，不需要关心过多的网络细节，更专注于中间件逻辑。
2. 性能高。成熟稳定，jdk nio的bug都被修复了。

选择框架是一方面，而想要保证网络通信的高效，网络线程模型也是一方面，我们常见的有1+N(1个Acceptor线程，N个IO线程)，1+N+M(1个acceptor线程，N个IO线程，M个worker线程)等模型，RocketMQ使用的是1+N1+N2+M的模型，如下图所示：
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1d31e9f0e4ea4d31a4ea21a746bfadff~tplv-k3u1fbpfcp-zoom-1.image)

1个acceptor线程，N1个IO线程，N2个线程用来做Shake-hand,SSL验证,编解码;M个线程用来做业务处理。这样的好处将编解码，和SSL验证等一些可能耗时的操作放在了一个单独的线程池，不会占据我们业务线程和IO线程。

## 7. RocketMQ 存储模型

做为一个好的消息系统，高性能的存储，高可用都不可少。
RocketMQ和Kafka的存储核心设计有很大的不同，所以其在写入性能方面也有很大的差别，这是16年阿里中间件团队对RocketMQ和Kafka不同Topic下做的性能测试:
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/448ad469765a45c2b35b535a9db39828~tplv-k3u1fbpfcp-zoom-1.image)

从图上可以看出：

* Kafka在Topic数量由64增长到256时，吞吐量下降了98.37%。
* RocketMQ在Topic数量由64增长到256时，吞吐量只下降了16%。

这是为什么呢？kafka一个topic下面的所有消息都是以partition的方式分布式的存储在多个节点上。同时在kafka的机器上，每个Partition其实都会对应一个日志目录，在目录下面会对应多个日志分段。所以如果Topic很多的时候Kafka虽然写文件是顺序写，但实际上文件过多，会造成磁盘IO竞争非常激烈。

那RocketMQ为什么在多Topic的情况下，依然还能很好的保持较多的吞吐量呢？我们首先来看一下RocketMQ中比较关键的文件:
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b7804f12290847008f8f16698e1d8880~tplv-k3u1fbpfcp-zoom-1.image)

* **commitLog**：消息主体以及元数据的存储主体，存储Producer端写入的消息主体内容,消息内容不是定长的。单个文件大小默认1G ，文件名长度为20位，左边补零，剩余为起始偏移量，比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824；当第一个文件写满了，第二个文件为00000000001073741824，起始偏移量为1073741824，以此类推。消息主要是顺序写入日志文件，当文件满了，写入下一个文件；

* **config**：保存一些配置信息，包括一些Group，Topic以及Consumer消费offset等信息。

* **consumeQueue**：消息消费队列，引入的目的主要是提高消息消费的性能，由于RocketMQ是基于主题topic的订阅模式，消息消费是针对主题进行的，如果要遍历commitlog文件中根据topic检索消息是非常低效的。

**Consumer即可根据ConsumeQueue来查找待消费的消息**。其中，ConsumeQueue（逻辑消费队列）作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值。

consumequeue文件可以看成是基于topic的commitlog索引文件，故consumequeue文件夹的组织方式如下：topic/queue/file三层组织结构，具体存储路径为：HOME \store\index\${fileName}，文件名fileName是以创建时的时间戳命名的，固定的单个IndexFile文件大小约为400M，一个IndexFile可以保存 2000W个索引，IndexFile的底层存储设计为在文件系统中实现HashMap结构，故rocketmq的索引文件其底层实现为hash索引。 

我们发现我们的消息主体数据并没有像Kafka一样写入多个文件，而是写入一个文件,这样我们的写入IO竞争就非常小，可以在很多Topic的时候依然保持很高的吞吐量。有同学说这里的ConsumeQueue写是在不停的写入呢，并且ConsumeQueue是以Queue维度来创建文件，那么文件数量依然很多，在这里ConsumeQueue的写入的数据量很小，每条消息只有20个字节，30W条数据也才6M左右，所以其实对我们的影响相对Kafka的Topic之间影响是要小很多的。我们整个的逻辑可以如下：
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/657006038dfa43249c831117e0d3f663~tplv-k3u1fbpfcp-zoom-1.image)

**Producer不断的再往CommitLog添加新的消息，有一个定时任务ReputService会不断的扫描新添加进来的CommitLog，然后不断的去构建ConsumerQueue和Index**。

注意：这里指的都是普通的硬盘，在SSD上面多个文件并发写入和单个文件写入影响不大。

Kafka中每个Partition都会是一个单独的文件，所以当消费某个消息的时候，会很好的出现顺序读，我们知道OS从物理磁盘上访问读取文件的同时，会顺序对其他相邻块的数据文件进行预读取，将数据放入PageCache，所以Kafka的读取消息性能比较好。

**RocketMQ读取流程如下**：

1. 先读取ConsumerQueue中的offset对应CommitLog物理的offset
2. 根据offset读取CommitLog

ConsumerQueue也是每个Queue一个单独的文件，并且其文件体积小，所以很容易利用PageCache提高性能。而CommitLog，由于同一个Queue的连续消息在CommitLog其实是不连续的，所以会造成随机读，RocketMQ对此做了几个优化：

* Mmap映射读取，Mmap的方式减少了传统IO将磁盘文件数据在操作系统内核地址空间的缓冲区和用户应用程序地址空间的缓冲区之间来回进行拷贝的性能开销
* 使用DeadLine调度算法+SSD存储盘

由于Mmap映射受到内存限制，当不在Mmmap映射这部分数据的时候(也就是消息堆积过多)，默认是内存的40%，会将请求发送到SLAVE,减缓Master的压力。

## 8. RocketMQ 高可用性

**集群模式**

我们首先需要选择一种集群模式，来适应我们可忍耐的可用程度，一般来说分为三种：

* **单Master**: 这种模式，可用性最低，但是成本也是最低，一旦宕机，所有都不可用。这种一般只适用于本地测试。
* **单Master多SLAVE**: 这种模式，可用性一般，如果主宕机，那么所有写入都不可用，读取依然可用，如果master磁盘损坏，可以依赖slave的数据。
* **多Master**: 这种模式，可用性一般，如果出现部分master宕机，那么这部分master上的消息都不可消费，也不可写数据，如果一个Topic的队列在多个Master上都有，那么可以保证没有宕机的那部分可以正常消费，写入。如果master的磁盘损坏会导致消息丢失。
* **多Master多Slave**：这种模式，可用性最高，但是维护成本也最高，当master宕机了之后，只会出现在这部分master上的队列不可写入，但是读取依然是可以的，并且如果master磁盘损坏，可以依赖slave的数据。

一般来说投入生产环境的话都会选择第四种，来保证最高的可用性。

**消息的可用性**

当我们选择好了集群模式之后，那么我们需要关心的就是怎么去存储和复制这个数据，rocketMQ对消息的刷盘提供了同步和异步的策略来满足我们的，当我们选择同步刷盘之后，如果刷盘超时会给返回FLUSH_DISK_TIMEOUT，如果是异步刷盘不会返回刷盘相关信息，选择同步刷盘可以尽最大程度满足我们的消息不会丢失。

除了存储有选择之后，我们的主从同步提供了同步和异步两种模式来进行复制，当然选择同步可以提升可用性，但是消息的发送RT时间会下降10%左右。

**Dleger**

我们上面对于master-slave部署模式已经做了很多分析，我们发现，当master出现问题的时候，我们的写入怎么都会不可用，除非恢复master，或者手动将我们的slave切换成master，导致了我们的Slave在多数情况下只有读取的作用。RocketMQ在最近的几个版本中推出了Dleger-RocketMQ，使用Raft协议复制CommitLog，并且自动进行选主，这样master宕机的时候，写入依然保持可用。

## 9. RocketMQ 定时/延时消息

定时消息和延时消息在实际业务场景中使用的比较多，比如下面的一些场景：

* 订单超时未支付自动关闭，因为在很多场景中下单之后库存就被锁定了，这里需要将其进行超时关闭。
* 需要一些延时的操作，比如一些兜底的逻辑，当做完某个逻辑之后，可以发送延时消息比如延时半个小时，进行兜底检查补偿。
* 在某个时间给用户发送消息，同样也可以使用延时消息。

在开源版本的RocketMQ中延时消息并不支持任意时间的延时，需要设置几个固定的延时等级，目前默认设置为：1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h，从1s到2h分别对应着等级1到18，而阿里云中的版本(要付钱)是可以支持40天内的任何时刻（毫秒级别）。我们先看下在RocketMQ中定时任务原理图：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e9effe0a753540a88de84b99122f5fb1~tplv-k3u1fbpfcp-zoom-1.image)

**Step1**：Producer在自己发送的消息上设置好需要延时的级别。

**Step2**: Broker发现此消息是延时消息，将Topic进行替换成延时Topic，每个延时级别都会作为一个单独的queue，将自己的Topic作为额外信息存储。

**Step3**: 构建ConsumerQueue

**Step4**: 定时任务定时扫描每个延时级别的ConsumerQueue。

**Step5**: 拿到ConsumerQueue中的CommitLog的Offset，获取消息，判断是否已经达到执行时间

**Step6**: 如果达到，那么将消息的Topic恢复，进行重新投递。如果没有达到则延迟没有达到的这段时间执行任务。

可以看见延时消息是利用新建单独的Topic和Queue来实现的，如果我们要实现40天之内的任意时间度，基于这种方案，那么需要402460601000个queue，这样的成本是非常之高的，那阿里云上面的支持任意时间是怎么实现的呢？这里猜测是持久化二级TimeWheel时间轮，二级时间轮用于替代我们的ConsumeQueue，保存Commitlog-Offset，然后通过时间轮不断的取出当前已经到了的时间，然后再次投递消息。

## 10. RocketMQ 事务消息

事务消息同样的也是RocketMQ中的一大特色，其可以帮助我们完成分布式事务的最终一致性:
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f025aa734833439b87454fd11b75fcdc~tplv-k3u1fbpfcp-zoom-1.image)

具体使用事务消息步骤如下：

**Step1**：调用sendMessageInTransaction发送事务消息

**Step2**: 如果发送成功，则执行本地事务。

**Step3**: 如果执行本地事务成功则发送commit，如果失败则发送rollback。

**Step4**: 如果其中某个阶段比如commit发送失败，rocketMQ会进行定时从Broker回查，本地事务的状态。

事务消息的使用整个流程相对之前几种消息使用比较复杂，下面是事务消息实现的原理图：
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c11d6e75fa1d4647bcedb007e174331f~tplv-k3u1fbpfcp-zoom-1.image)

**Step1**: 发送事务消息,这里也叫做halfMessage，会将Topic替换为HalfMessage的Topic。

**Step2**: 发送commit或者rollback，如果是commit这里会查询出之前的消息，然后将消息复原成原Topic，并且发送一个OpMessage用于记录当前消息可以删除。如果是rollback这里会直接发送一个OpMessage删除。

**Step3**: 在Broker有个处理事务消息的定时任务，定时对比halfMessage和OpMessage,如果有OpMessage且状态为删除，那么该条消息必定commit或者rollback，所以就可以删除这条消息。

**Step4**: 如果事务超时（默认是6s），还没有opMessage，那么很有可能commit信息丢了，这里会去反查我们的Producer本地事务状态。

**Step5**: 根据查询出来的信息做Step2。

我们发现RocketMQ实现事务消息也是通过修改原Topic信息，和延迟消息一样，然后模拟成消费者进行消费，做一些特殊的业务逻辑。当然我们还可以利用这种方式去做RocketMQ更多的扩展。

> [Java工程师的进阶之路 RocketMQ篇（一）](https://juejin.im/post/6864724887996252173)<br>
> [Java工程师的进阶之路 RocketMQ篇（二）](https://juejin.im/post/6864737331484721165)<br>