> 白菜Java自习室 涵盖核心知识

> [Java工程师的进阶之路 ElasticSearch篇（一）](https://juejin.im/post/6870798638668087310)<br>
> [Java工程师的进阶之路 ElasticSearch篇（二）](https://juejin.im/post/6871218426266779662)<br>

## 1. ElasticSearch 简介

> Elasticsearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java语言开发的，并作为Apache许可条款下的开放源码发布，是一种流行的企业级搜索引擎。Elasticsearch用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。

**Elasticsearch 是一个分布式、高扩展、高实时的搜索与数据分析引擎**。它能很方便的使大量数据具有搜索、分析和探索的能力。充分利用Elasticsearch的水平伸缩性，能使数据在生产环境变得更有价值。Elasticsearch 的实现原理主要分为以下几个步骤，首先用户将数据提交到Elasticsearch 数据库中，再通过分词控制器去将对应的语句分词，将其权重和分词结果一并存入数据，当用户搜索数据时候，再根据权重将结果排名，打分，再将返回结果呈现给用户。

Elasticsearch是与名为Logstash的数据收集和日志解析引擎以及名为Kibana的分析和可视化平台一起开发的。这三个产品被设计成一个集成解决方案，称为**“Elastic Stack”（以前称为“ELK stack”）**。

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3412d6a981384deaa2e13a4aef2b8b11~tplv-k3u1fbpfcp-zoom-1.image)

Elasticsearch可以用于搜索各种文档。它提供可扩展的搜索，具有接近实时的搜索，并支持多租户。”Elasticsearch是分布式的，这意味着索引可以被分成分片，每个分片可以有0个或多个副本。每个节点托管一个或多个分片，并充当协调器将操作委托给正确的分片。再平衡和路由是自动完成的。“相关数据通常存储在同一个索引中，该索引由一个或多个主分片和零个或多个复制分片组成。一旦创建了索引，就不能更改主分片的数量。

Elasticsearch使用Lucene，并试图通过JSON和Java API提供其所有特性。它支持facetting和percolating，如果新文档与注册查询匹配，这对于通知非常有用。另一个特性称为“网关”，处理索引的长期持久性；例如，在服务器崩溃的情况下，可以从网关恢复索引。Elasticsearch支持实时GET请求，适合作为NoSQL数据存储，但缺少分布式事务。

## 2. ElasticSearch 基本概念

Elasticsearch自顶向下的架构体系：

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72696e3168244e3c91c12806b2964f7c~tplv-k3u1fbpfcp-zoom-1.image)

### 2.1. 集群（Cluster）

> 集群（Cluster）：ES是一个分布式的搜索引擎，一般由多台物理机组成。这些物理机，通过配置一个相同的Cluster Name，互相发现，把自己组织成一个集群。多台ES服务器的结合的统称叫ES集群。

ES集群的健康状态：
* **Green** - 主分片与副本都正常分配
* **Yellow** - 主分片全部正常分配，有副本分片未能正常分配
* **Red** - 有主分片未能分配（例如 当服务器的磁盘容量超过85%时，去创建了一个新的索引）

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/34b170cf1dc24dc790ee387f40f4006d~tplv-k3u1fbpfcp-zoom-1.image)

### 2.2. 节点（Node）

> 节点（Node)：同一个集群中的一个Elasticsearch主机。

1. 节点是一个Elasticsearch的实例，本质上就是一个JAVA进程；
2. 每一个节点都有名字，通过配置文件配置，或者启动时候指定；
3. 每一个节点在启动之后，会分配一个UID，保存在data目录下。

主要Node类型：

* **Data Node** - 存储index数据。Data nodes hold data and perform data related operations such as CRUD, search, and aggregations.
* **Client Node** - 不存储index，处理转发客户端请求到Data Node
* **Master Node** - 不存储index，集群管理，如管理路由信息（routing infomation），判断node是否available，当有node出现或消失时重定位分片（shards），当有node failure时协调恢复。（所有的master node会选举出一个master leader node）

其他Node类型：

* **Coordination Node** - 负责接收Client的请求，将请求分发到合适的节点，最终把结果汇集到一起
每个节点默认都起到了Coordination Node的职责 
* **Hot&Warm Node** - 不同硬件配置的Data Node，用来实现Hot&Warm架构，降低集群部署的成本
* **Machine Learning Node** - 负责跑机器学习的Job,用来做异常检测
* **Ingest Nodee** - 可以看作是数据前置处理转换的节点，支持 pipeline管道 设置，可以使用 ingest 对数据进行过滤、转换等操作，类似于 logstash 中 filter 的作用。
* **Tribe Nodee** - 5.3开始使用Cross Cluster Search）TribeNode 连接到不同的Elasticsearch集群，并且支持将这些集群当成一个单独的集群处理

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a8f602c1bcaf42399aa21de1efd53e8c~tplv-k3u1fbpfcp-zoom-1.image)

### 2.3. 主分片（Primary shard）

> 主分片（Primary shard）：索引（下文介绍）的一个物理子集。同一个索引在物理上可以切多个分片，分布到不同的节点上。分片的实现是Lucene 中的索引。将数据切分放在每个分片中，分片又被放到集群中的节点上。每个分片都是独立的lucene实例。

注意：**ES中一个索引的分片个数是建立索引时就要指定的，建立后不可再改变**。所以开始建一个索引时，就要预计数据规模，将分片的个数分配在一个合理的范围。

**分片数设置过小**：

1. 导致后续无法增加节点实现水平扩展
2. 单个分片的数据量太大，导致数据重新分配耗时

**分片数设置过大**：

1. 影响搜索结构的相关性打分，影响统计结果的准确性
2. 单个节点上过多的分片，会导致资源浪费，同时也会影响性能

### 2.4. 副本分片（Replica shard）

> 副本分片（Replica shard）：每个主分片可以有一个或者多个副本，个数是用户自己配置的。ES会尽量将同一索引的不同分片分布到不同的节点上，提高容错性。对一个索引，只要不是所有shards所在的机器都挂了，就还能用。

* 副本分片数提高了数据冗余量
* 主分片挂掉以后能够自动由副本分片升为主分片
* 备份分片还能够降低主分片的查询压力(会消耗更多的系统性能)

### 2.5. 索引（Index）

> 索引（Index）：逻辑概念，一个可检索的文档对象的集合。类似与DB中的database概念。同一个集群中可建立多个索引。比如，生产环境常见的一种方法，对每个月产生的数据建索引，以保证单个索引的量级可控。

索引是文档的容器，是一类文档的结合：

* **Index体现了逻辑空间的概念**：每一个索引都有自己的Mapping定义，用于定义包含的文档的字段名和字段类型
* **Shard体现了物理空间的概念**：索引中的数据分散在Shard上

Elasticsearch集群可以包含多个索引（indices），每一个索引可以包含多个类型（types），每一个类型包含多个文档（documents），然后每个文档包含多个字段（Fields），这种面向文档型的储存，也算是NoSQL的一种吧。

ES比传统关系型数据库，对一些概念上的理解：

```
Relational DB -> Databases(数据库) -> Tables(表) -> Rows(行) -> Columns(列)
Elasticsearch -> Indices(索引) -> Types(类型) -> Documents(文档) -> Fields(字段)
```

### 2.6. 类型（Type）

> 类型（Type）：索引的下一级概念，大概相当于数据库中的table。同一个索引里可以包含多个 Type。 

* 在7.0之前，一个index可以设置多个Types
* 6.0开始，Type已经被Deprecated。7.0开始一个索引只能创建一个Type -"_doc"

### 2.7. 文档（Document）

> 文档（Document）：即搜索引擎中的文档概念，也是ES中一个可以被检索的基本单位，相当于数据库中的row，一条记录。

1. Elasticsearch是面向文档的，文档是所有可搜索数据的最小单位
2. 文档会被序列化成JSON格式，保持在Elasticsearch中
3. 每一个文档都有一个UniqueID

文档的元数据（元数据，用于标注稳定的相关信息）：

* **_index** - 文档所属的索引名
* **_type** - 文档所属的类型名                                       
* **_source** - 文档的原始Json数据
* **_version** - 文档的版本信息
* **_score** - 相关性打分

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4703781705e149579e82b391ffd9a1fd~tplv-k3u1fbpfcp-zoom-1.image)

### 2.8. 字段（Field）

> 字段（Field）：相当于数据库中的column。ES中，每个文档，其实是以json形式存储的。而一个文档可以被视为多个字段的集合。比如一篇文章，可能包括了主题、摘要、正文、作者、时间等信息，每个信息都是一个字段，最后被整合成一个json串，落地到磁盘。

### 2.9. 映射（Mapping）

> 映射（Mapping）：相当于数据库中的schema，用来约束字段的类型，不过 Elasticsearch 的 mapping 可以不显示地指定、自动根据文档数据创建。

索引的Mapping与Settings：

* Mapping定义文档字段的类型
* Setting定义不同的数据分布

### 2.10. REST API 

Elasticsearch提供了一个非常全面和强大的REST API，使用它与集群进行交互：

1. 检查群集，节点和索引运行状况，状态和统计信息
2. 管理您的群集，节点和索引数据和元数据
3. 对索引执行CRUD（创建，读取，更新和删除）和搜索操作
4. 执行高级搜索操作，例如分页，排序，过滤，脚本编写，聚合等

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd38b735937f44ab8411378ab7c371ec~tplv-k3u1fbpfcp-zoom-1.image)

## 3. Elasticsearch 倒排索引

### 3.1. 正向索引

正向索引：对文档中的所有分词进行遍历，获取与 keyword 中分词相同的词，命中一次，这种索引方法需要对每一个文档都进行遍历，性能上不是很好。

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4708b60a40cb4b00ae8b9d762311944d~tplv-k3u1fbpfcp-zoom-1.image)

### 3.2. 倒排索引

倒排索引：对文档的分词结果进行遍历，如果 keyword 命中该分词，那么通过该分词便可以找到所有含有这个分词的文档，性能相对较高。

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e3e2a2492eea4b61994297e167703542~tplv-k3u1fbpfcp-zoom-1.image)

倒排索引可以很大的提高检索的速度，下面举一个例子，来说明一下倒排索引是什么，以及这种方式相比于传统数据库为什么会提高索引的速度。我们来看一个实际的例子，假设有如下的数据：

**文档A**：
```
I love study.
```
**文档B**：
```
And study make me happy.
```
现在输入我们想要检索的关键字：“**study happy**”。

可以想象，在传统数据库中，会用一条类似这样的 sql 语句在数据库表中挨条查找符合要求的记录：
```
SELECT column_name(s)
FROM table_name
WHERE (column_name LIKE 'study' and column_name LIKE 'happy');
```

假如表中存有 1000w 条记录，那么这条 sql 将会依次检索这1000w 条记录，所要消耗的时间随着记录数增多而提高。这种方式在应对海量数据的时候，便有些力不从心。

**那么我们可不可以换一种检索的方式呢？**

既然我们要在 **文档（记录）** 中查找指定的 **单词（关键字）**，最终查找的是 **单词（关键字）** 在某一个文档中存不存在，存在于哪一个文档中。那么我们可以反过来直接根据 **单词（关键字）** 建立一个索引表，通过检索关键字直接找到所有包含关键字的文档，这便是 **倒排索引**。

先拿以上例子说明一下传统数据库存储的方式：
![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/68cf26950e8449d59693c7fadc2675ea~tplv-k3u1fbpfcp-zoom-1.image)

再拿以上例子说明如何建立倒排索引：

文档 A 和 B 中的两个文本，“**I love study.**” 和 “**And study make me happy.**”，可以建立如下索引表：
![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/86ac58e7d5b24314b91e9b7103cd1a3c~tplv-k3u1fbpfcp-zoom-1.image)

当我们检索关键字 “**study happy**” 的时候，可以直接找到 **study** 在 **文档A** 和 **文档B** 中都存在，**happy** 只在 **文档B** 中存在。相比传统数据库的检索方式，这种方式更加直接和高效，特别在数据量极大的情况下。
![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e80b440dbc404b07846f522d050dd4c0~tplv-k3u1fbpfcp-zoom-1.image)

同时我们也可以发现，我们输入的关键字在 文档A 中命中1次，在文档B中命中2次，于是 ElasticSearch 甚至可以得到相关性的结论，**文档B 的相关度比 文档A 高**。

这也是 ElasticSearch 的一个优势所在，它不仅可以快速检索内容，还可以将命中的文档根据相关度进行排序，并显示在搜索结果中。

### 3.3. Lucene 字典树

Lucene 的的倒排索引，增加了最左边的一层**「字典树」term index**，它不存储所有的单词，只存储单词前缀，通过字典树找到单词所在的块，也就是单词的大概位置，再在块里二分查找，找到对应的单词，再找到单词对应的文档列表。

举个例子：

一个包含 "A", "to", "tea", "ted", "ten", "i", "in", 和 "inn" 的 trie 树。这棵树不会包含所有的term，它包含的是term的一些前缀。通过term index可以快速地定位到term dictionary的某个offset，然后从这个位置再往后顺序查找。再加上一些压缩技术（搜索 Lucene Finite State Transducers） term index 的尺寸可以只有所有term的尺寸的几十分之一，使得用内存缓存整个term index变成可能。

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d558e4a0057e4dae9d16242c3c619b67~tplv-k3u1fbpfcp-zoom-1.image)
![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cdd076e66d8a4c559fd60e9b8aced54e~tplv-k3u1fbpfcp-zoom-1.image)

Lucene 的真正实现会要更加复杂，针对不同的数据结构采用不同的字典索引，使用了**FST模型**、**BKDTree**等结构。真实的倒排记录也并非一个链表，而是采用了**SkipList**、**BitSet**等结构。

## 4. Elasticsearch 分词原理

当一个文档被索引时，每个field都可能会创建一个倒排索引（如果mapping的时候没有设置不索引该field）。

倒排索引的过程就是将文档通过analyzer分成一个一个的term，每一个term都指向包含这个term的文档集合。当查询query时，Elasticsearch会根据搜索类型决定是否对query进行analyze，然后和倒排索引中的term进行相关性查询，匹配相应的文档。

> 例如：
"I am a chinese"可以分为"I"、"am"、"a"、"chinese"这4个单词

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f685aa6f539f4b41aad808e49f5355b2~tplv-k3u1fbpfcp-zoom-1.image)

**分词器（Analyzer）**就是将一段文本按照一定的逻辑，分析成多个词语，同时对这些词语进行常规化(normalization)的一种工具。Elasticsearch的分词器包含CharFilters、Tokenizer和TokenFilters。

```
Analyzer = CharFilters（0个或多个） + Tokenizer(恰好一个) + TokenFilters(0个或多个)
```

* **CharFilters**：使用字符过滤器转变字符
* **Tokenizer**：将文本切分单个或者多个分词
* **TokenFilters**：使用分词过滤器转变每个分词

**ES的分词器执行顺序：CharFilters->Tokenizer->TokenFilters**

如果mapping中只设置了一个analyzer，那么这个analyzer会同时用于索引文档和搜索query。当然索引文档和对query进行analysis也可以使用不同的analyzer。

一个特殊的情况是有的query是需要被analyzed，有的并不需要。例如match query会先用search analyzer进行分析，然后去相应field的倒排索引进行匹配。而term query并不会对query内容进行分析，而是直接和相应field的倒排索引去匹配。

### 4.1. ElasticSearch 内置分词器

* **standard analyzer**：按照非字母和非数字字符进行分隔，单词转为小写
```
测试文本：a*B!c d4e 5f 7-h
分词结果：a、b、c、d4e、5f、7、h
```
* **simple analyzer**：按照非字母字符进行分隔，单词转为小写
```
测试文本：a*B!c d4e 5f 7-h
分词结果：a、b、c、d、e、f、h
```
* **whitespace analyzer**：按照空白字符进行分隔
```
测试文本：a*B!c D d4e 5f 7-h
分词结果：a*B!c、D、d4e、5f、7-h
```
* **stop analyzer**：使用非字母字符进行分隔，单词转换为小写，并去掉停用词(默认为英语的停用词，例如the、a、an、this、of、at等)
```
测试文本：The apple is red
分词结果：apple、red
```
* **language analyzer**：使用指定的语言的语法进行分词，默认为english，没有内置中文分词器

* **pattern analyzer**：使用指定的正则表达式进行分词，默认\\W+，即多个非数字非字母字符

### 4.2. ElasticSearch 中文分词器

* **SmartCN **：一个简单的中⽂或中英⽂混合文本分词器
* **IK分词器**：更智能更友好的中⽂分词器

以上分词器进行一个粗略对比：

|分词器|优势|劣势|
|-- |-- |-- |
|SmartCN|官方插件|中文分词效果惨不忍睹|
|IK分词器|	简单易用，支持自定义词典和远程词典|词库需要自行维护，不支持词性识别|

## 5. Elasticsearch 数据类型

### 5.1. 字符串类型

> string类型：在ElasticSearch 旧版本中使用较多，从ElasticSearch 5.x开始不再支持string，由text和keyword类型替代。

* **text 类型**：当一个字段是要被全文搜索的，比如Email内容、产品描述，应该使用text类型。设置text类型以后，字段内容会被分析，在生成倒排索引以前，字符串会被分析器分成一个一个词项。text类型的字段不用于排序，很少用于聚合。
* **keyword 类型**：keyword类型适用于索引结构化的字段，比如email地址、主机名、状态码和标签。如果字段需要进行过滤(比如查找已发布博客中status属性为published的文章)、排序、聚合。keyword类型的字段只能通过精确值搜索到。

### 5.2. 整数类型

|类型|取值范围|
|-- |-- |
|byte	|有符号的8位整数, 范围: [-128 ~ 127]|
|short	|有符号的16位整数, 范围: [-32768 ~ 32767]|
|integer	|有符号的32位整数, 范围: [−2^31 ~ 2^31-1]|
|long	|有符号的64位整数, 范围: [−2^63 ~ 2^63-1]|

在满足需求的情况下，尽可能选择范围小的数据类型。比如，某个字段的取值最大值不会超过100，那么选择byte类型即可。迄今为止吉尼斯记录的人类的年龄的最大值为134岁，对于年龄字段，short足矣。字段的长度越短，索引和搜索的效率越高。

### 5.3. 浮点类型

|类型|取值范围|
|-- |-- |
|float	|32位单精度浮点数|
|double	|64位双精度浮点数|
|half_float	|16位半精度IEEE 754浮点类型|
|scaled_float	|缩放类型的的浮点数, 比如price字段只需精确到分, 57.34缩放因子为100, 存储结果为5734|

对于float、half_float和scaled_float,-0.0和+0.0是不同的值，使用term查询查找-0.0不会匹配+0.0，同样range查询中上边界是-0.0不会匹配+0.0，下边界是+0.0不会匹配-0.0。
其中scaled_float，比如价格只需要精确到分，price为57.34的字段缩放因子为100，存起来就是5734，
优先考虑使用带缩放因子的scaled_float浮点类型。

### 5.4. date类型

日期类型表示格式可以是以下几种：

* **日期格式的字符串**，比如 “2018-01-13” 或 “2018-01-13 12:10:30”
* **long类型的毫秒数**( milliseconds-since-the-epoch，epoch就是指UNIX诞生的UTC时间1970年1月1日0时0分0秒)
* **integer的秒数**(seconds-since-the-epoch)

### 5.5. boolean类型

可以接受表示真、假的字符串或数字:

* **真值**: true, "true", "on", "yes", "1"...
* **假值**: false, "false", "off", "no", "0", ""(空字符串), 0.0, 0

### 5.6. binary类型

进制字段是指用base64来表示索引中存储的二进制数据，可用来存储二进制形式的数据，例如图像。默认情况下，该类型的字段只存储不索引。二进制类型只支持index_name属性。二进制类型是Base64编码字符串的二进制值, 不以默认的方式存储, 且不能被搜索。有两个设置项:

* **doc_values**: 该字段是否需要存储到磁盘上, 方便以后用来排序、聚合或脚本查询。 接受true和false(默认);
* **store**: 该字段的值是否要和_source分开存储、检索, 意思是除了_source中, 是否要单独再存储一份, 接受true或false(默认)。

### 5.7. array类型

ES中没有专门的数组类型, 直接使用[]定义即可;
数组中所有的值必须是同一种数据类型, 不支持混合数据类型的数组:

* **字符串数组**: ["one", "two"];
* **整数数组**: [1, 2];
* **由数组组成的数组**: [1, [2, 3]], 等价于[1, 2, 3];
* **对象数组**: [{"name": "Tom", "age": 20}, {"name": "Jerry", "age": 18}].

### 5.8. object类型

JSON文档是分层的: 文档可以包含内部对象, 内部对象也可以包含内部对象。

### 5.9. ip类型

IP类型的字段用于存储IPv4或IPv6的地址, 本质上是一个长整型字段。

### 5.10. geo类型

地理点类型用于存储地理位置的经纬度对, 可用于:

* 查找一定范围内的地理点;
* 通过地理位置或相对某个中心点的距离聚合文档;
* 将距离整合到文档的相关性评分中;
* 通过距离对文档进行排序。

> 还有一些其他类型使用较少, 如nested，geo_shape，token_count等，这里就不一一介绍了。

> [Java工程师的进阶之路 ElasticSearch篇（一）](https://juejin.im/post/6870798638668087310)<br>
> [Java工程师的进阶之路 ElasticSearch篇（二）](https://juejin.im/post/6871218426266779662)<br>