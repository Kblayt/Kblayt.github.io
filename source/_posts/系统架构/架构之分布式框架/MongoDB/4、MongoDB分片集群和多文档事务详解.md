---
title: 4、MongoDB分片集群和多文档事务详解
date: 2023-06-18 21:34:07
categories:
    - 系统架构
    - 架构之分布式框架
    - MongoDB
tags:
    - MongoDB 
---

# **分片集群架构**

## **分片简介**

分片（shard）是指在将数据进行水平切分之后，将其存储到多个不同的服务器节点上的一种扩展方式。分片在概念上非常类似于应用开发中的“水平分表”。不同的点在于，MongoDB本身就自带了分片管理的能力，对于开发者来说可以做到开箱即用。

### **为什么要使用分片？**

MongoDB复制集实现了数据的多副本复制及高可用，但是一个复制集能承载的容量和负载是有限的。在你遇到下面的场景时，就需要考虑使用分片了：

- 存储容量需求超出单机的磁盘容量。
- 活跃的数据集超出单机内存容量，导致很多请求都要从磁盘读取数据，影响性能。
- 写IOPS超出单个MongoDB节点的写服务能力。

垂直扩容（Scale Up） VS 水平扩容（Scale Out）： 

垂直扩容 ： 用更好的服务器，提高 CPU 处理核数、内存数、带宽等 

水平扩容 ： 将任务分配到多台计算机上

### **MongoDB 分片集群架构**

MongoDB 分片集群（Sharded Cluster）是对数据进行水平扩展的一种方式。MongoDB 使用 分片集群来支持大数据集和高吞吐量的业务场景。在分片模式下，存储不同的切片数据的节点被称为分片节点，一个分片集群内包含了多个分片节点。当然，除了分片节点，集群中还需要一些配置节点、路由节点，以保证分片机制的正常运作。

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/41963)

### **核心概念**

- **数据分片**：分片用于存储真正的数据，并提供最终的数据读写访问。分片仅仅是一个逻辑的概念，它可以是一个单独的mongod实例，也可以是一个复制集。图中的Shard1、Shard2都是一个复制集分片。在生产环境中也一般会使用复制集的方式，这是为了防止数据节点出现单点故障。
- **配置服务器**（Config Server）：配置服务器包含多个节点，并组成一个复制集结构，对应于图中的ConfigReplSet。配置复制集中保存了整个分片集群中的元数据，其中包含各个集合的分片策略，以及分片的路由表等。

- **查询路由**（mongos）：mongos是分片集群的访问入口，其本身并不持久化数据。mongos启动后，会从配置服务器中加载元数据。之后mongos开始提供访问服务，并将用户的请求正确路由到对应的分片。在分片集群中可以部署多个mongos以分担客户端请求的压力。

**环境搭建**

[**分片集群搭建**](http://note.youdao.com/noteshare?id=fa7b18a78fa8a4d1961729b043388b57&sub=B8435BD62E634993AD05F910880F0630)

[**使用mtools搭建分片集群**](http://note.youdao.com/noteshare?id=3c02251c8b4a8bfc98ab392146aa8222&sub=9E0834FE787F413E8EBA774596AB3999)

搭建视频：https://vip.tulingxueyuan.cn/detail/p_622d92aee4b066e9608ee2c9/6

**使用分片集群**

为了使集合支持分片，需要先开启database的分片功能

​                sh.enableSharding("shop")              

执行shardCollection命令，对集合执行分片初始化

​                sh.shardCollection("shop.product",{productId:"hashed"},false,{numInitialChunks:4})              

shop.product集合将productId作为分片键，并采用了哈希分片策略，除此以外，“numInitialChunks：4”表示将初始化4个chunk。 numInitialChunks必须和哈希分片策略配合使用。而且，这个选项只能用于空的集合，如果已经存在数据则会返回错误。

**向分片集合写入数据**

向shop.product集合写入一批数据

​                db=db.getSiblingDB("shop"); var count=0; for(var i=0;i<1000;i++){    var p=[];    for(var j=0;j<100;j++){        p.push({            "productId":"P-"+i+"-"+j,            name:"羊毛衫",            tags:[                {tagKey:"size",tagValue:["L","XL","XXL"]},                {tagKey:"color",tagValue:["蓝色","杏色"]},                {tagKey:"style",tagValue:"韩风"}            ]        });    }    count+=p.length;    db.product.insertMany(p);    print("insert ",count) }                 

**查询数据的分布**

​                db.product.getShardDistribution()              

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/42485)

## **分片策略**

通过分片功能，可以将一个非常大的集合分散存储到不同的分片上，如图：

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/41977)

假设这个集合大小是1TB，那么拆分到4个分片上之后，每个分片存储256GB的数据。这个当然是最理想化的场景，实质上很难做到如此绝对的平衡。一个集合在拆分后如何存储、读写，与该集合的分片策略设定是息息相关的。在了解分片策略之前，我们先来介绍一下chunk。

### **什么是chunk**

chunk的意思是数据块，一个chunk代表了集合中的“一段数据”，例如，用户集合（db.users）在切分成多个chunk之后如图所示：

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/41988)

chunk所描述的是范围区间，例如，db.users使用了userId作为分片键，那么chunk就是userId的各个值（或哈希值）的连续区间。集群在操作分片集合时，会根据分片键找到对应的chunk，并向该chunk所在的分片发起操作请求，而chunk的分布在一定程度上会影响数据的读写路径，这由以下两点决定：

- chunk的切分方式，决定如何找到数据所在的chunk
- chunk的分布状态，决定如何找到chunk所在的分片

### **分片算法**

chunk切分是根据分片策略进行实施的，分片策略的内容包括分片键和分片算法。当前，MongoDB支持两种分片算法：

**范围分片（range sharding）**

假设集合根据x字段来分片，x的完整取值范围为[minKey, maxKey]（x为整数，这里的minKey、maxKey为整型的最小值和最大值），其将整个取值范围划分为多个chunk，例如：

- chunk1包含x的取值在[minKey，-75）的所有文档。
- chunk2包含x取值在[-75，25）之间的所有文档，依此类推。

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/42173)

范围分片能很好地满足范围查询的需求，比如想查询x的值在[-30，10]之间的所有文档，这时mongos直接将请求定位到chunk2所在的分片服务器，就能查询出所有符合条件的文档。范围分片的缺点在于，如果Shard Key有明显递增（或者递减）趋势，则新插入的文档会分布到同一个chunk，此时写压力会集中到一个节点，从而导致单点的性能瓶颈。一些常见的导致递增的Key如下：

- 时间值。
- ObjectId，自动生成的_id由时间、计数器组成。
- UUID，包含系统时间、时钟序列。
- 自增整数序列。

**哈希分片（hash sharding）**

哈希分片会先事先根据分片键计算出一个新的哈希值（64位整数），再根据哈希值按照范围分片的策略进行chunk的切分。适用于日志，物联网等高并发场景。

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/42170)

哈希分片与范围分片是互补的，由于哈希算法保证了随机性，所以文档可以更加离散地分布到多个chunk上，这避免了集中写问题。然而，在执行一些范围查询时，哈希分片并不是高效的。因为所有的范围查询都必然导致对所有chunk进行检索，如果集群有10个分片，那么mongos将需要对10个分片分发查询请求。哈希分片与范围分片的另一个区别是，哈希分片只能选择单个字段，而范围分片允许采用组合式的多字段作为分片键。

哈希分片仅支持单个字段的哈希分片：

​                { x : "hashed" }  {x : 1 , y : "hashed"} // 4.4 new              

4.4 以后的版本，可以将单个字段的哈希分片和一个到多个的范围分片键字段来进行组合，比如指定   x:1,y 是哈希的方式。

**分片标签**

MongoDB允许通过为分片添加标签（tag）的方式来控制数据分发。一个标签可以关联到多个分片区间（TagRange）。均衡器会优先考虑chunk是否正处于某个分片区间上（被完全包含），如果是则会将chunk迁移到分片区间所关联的分片，否则按一般情况处理。

分片标签适用于一些特定的场景。例如，集群中可能同时存在OLTP和OLAP处理，一些系统日志的重要性相对较低，而且主要以少量的统计分析为主。为了便于单独扩展，我们可能希望将日志与实时类的业务数据分开，此时就可以使用标签。

为了让分片拥有指定的标签，需执行addShardTag命令

​                sh.addShardTag("shard01","oltp") sh.addShardTag("shard02","oltp") sh.addShardTag("shard03","olap")              

实时计算的集合应该属于oltp标签，声明TagRange

​                sh.addTagRange("main.devices",{shardKey:MinKey},{shardKey:MaxKey},"oltp")              

而离线计算的集合，则属于olap标签

​                sh.addTagRange("other.systemLogs",{shardKey:MinKey},{shardKey:MaxKey},"olap")              

main.devices集合将被均衡地分发到shard01、shard02分片上，而other.systemLogs集合将被单独分发到shard03分片上。

### **分片键（ShardKey）的选择**

在选择分片键时，需要根据业务的需求及范围分片、哈希分片的不同特点进行权衡。一般来说，在设计分片键时需要考虑的因素包括：

- 分片键的基数（cardinality），取值基数越大越有利于扩展。 

- - 以性别作为分片键 ：数据最多被拆分为 2 份 
  - 以月份作为分片键 ：数据最多被拆分为 12 份	

- 分片键的取值分布应该尽可能均匀。

- 业务读写模式，尽可能分散写压力，而读操作尽可能来自一个或少量的分片。

- 分片键应该能适应大部分的业务操作。

### **分片键（ShardKey）的约束** 

ShardKey 必须是一个索引。非空集合须在 ShardCollection 前创建索引；空集合 ShardCollection 自动创建索引 

4.4 版本之前： 

- ShardKey 大小不能超过 512 Bytes； 
- 仅支持单字段的哈希分片键； 
- Document 中必须包含 ShardKey； 
- ShardKey 包含的 Field 不可以修改。 

4.4 版本之后: 

- ShardKey 大小无限制； 
- 支持复合哈希分片键； 
- Document 中可以不包含 ShardKey，插入时被当 做 Null 处理； 
- 为 ShardKey 添加后缀 refineCollectionShardKey 命令，可以修改 ShardKey 包含的 Field； 	

而在 4.2 版本之前，ShardKey 对应的值不可以修改；4.2 版本之后，如果 ShardKey 为非_ID 字段， 那么可以修改 ShardKey 对应的值。

## **数据均衡**

### **均衡的方式**

一种理想的情况是，所有加入的分片都发挥了相当的作用，包括提供更大的存储容量，以及读写访问性能。因此，为了保证分片集群的水平扩展能力，业务数据应当尽可能地保持均匀分布。这里的均匀性包含以下两个方面：

1. 所有的数据应均匀地分布于不同的chunk上。
2. 每个分片上的chunk数量尽可能是相近的。

其中，第1点由业务场景和分片策略来决定，而关于第2点，我们有以下两种选择：

**手动均衡**

一种做法是，可以在初始化集合时预分配一定数量的chunk（仅适用于哈希分片），比如给10个分片分配1000个chunk，那么每个分片拥有100个chunk。另一种做法则是，可以通过splitAt、moveChunk命令进行手动切分、迁移。

**自动均衡**

开启MongoDB集群的自动均衡功能。均衡器会在后台对各分片的chunk进行监控，一旦发现了不均衡状态就会自动进行chunk的搬迁以达到均衡。其中，chunk不均衡通常来自于两方面的因素：

- 一方面，在没有人工干预的情况下，chunk会持续增长并产生分裂（split），而不断分裂的结果就会出现数量上的不均衡；
- 另一方面，在动态增加分片服务器时，也会出现不均衡的情况。自动均衡是开箱即用的，可以极大简化集群的管理工作。

### **chunk分裂**

在默认情况下，一个chunk的大小为64MB，该参数由配置的chunksize参数指定。如果持续地向该chunk写入数据，并导致数据量超过了chunk大小，则MongoDB会自动进行分裂，将该chunk切分为两个相同大小的chunk。

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/42051)

chunk分裂是基于分片键进行的，如果分片键的基数太小，则可能因为无法分裂而会出现jumbo chunk（超大块）的问题。例如，对db.users使用gender（性别）作为分片键，由于同一种性别的用户数可能达到数千万，分裂程序并不知道如何对分片键（gender）的一个单值进行切分，因此最终导致在一个chunk上集中存储了大量的user记录（总大小超过64MB）。

jumbo chunk对水平扩展有负面作用，该情况不利于数据的均衡，业务上应尽可能避免。一些写入压力过大的情况可能会导致chunk多次失败（split），最终当chunk中的文档数大于1.3×avgObjectSize时会导致无法迁移。此外在一些老版本中，如果chunk中的文档数超过250000个，也会导致无法迁移。

### **自动均衡**

MongoDB的数据均衡器运行于Primary Config Server（配置服务器的主节点）上，而该节点也同时会控制chunk数据的搬迁流程。

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/42063)

流程说明：

- 分片shard0在持续的业务写入压力下，产生了chunk分裂。
- 分片服务器通知Config Server进行元数据更新。
- Config Server的自动均衡器对chunk分布进行检查，发现shard0和shard1的chunk数差异达到了阈值，向shard0下发moveChunk命令以执行chunk迁移。
- shard0执行指令，将指定数据块复制到shard1。该阶段会完成索引、chunk数据的复制，而且在整个过程中业务侧对数据的操作仍然会指向shard0；所以，在第一轮复制完毕之后，目标shard1会向shard0确认是否还存在增量更新的数据，如果存在则继续复制。
-  shard0完成迁移后发送通知，此时Config Server开始更新元数据库，将chunk的位置更新为目标shard1。在更新完元数据库后并确保没有关联cursor的情况下，shard0会删除被迁移的chunk副本。
- Config Server通知mongos服务器更新路由表。此时，新的业务请求将被路由到shard1。

**迁移阈值**

均衡器对于数据的“不均衡状态”判定是根据两个分片上的chunk个数差异来进行的

| chunk个数 | 迁移阈值 |
| --------- | -------- |
| 少于20    | 2        |
| 20～79    | 4        |
| 80及以上  | 8        |

**迁移速度**

数据均衡的整个过程并不是很快，影响MongoDB均衡速度的几个选项如下：

- _secondaryThrottle：用于调整迁移数据写到目标分片的安全级别。如果没有设定，则会使用w：2选项，即至少一个备节点确认写入迁移数据后才算成功。从MongoDB 3.4版本开始，_secondaryThrottle被默认设定为false, chunk迁移不再等待备节点写入确认。
- _waitForDelete：在chunk迁移完成后，源分片会将不再使用的chunk删除。如果_waitForDelete是true，那么均衡器需要等待chunk同步删除后才进行下一次迁移。该选项默认为false，这意味着对于旧chunk的清理是异步进行的。
- 并行迁移数量：在早期版本的实现中，均衡器在同一时刻只能有一个chunk迁移任务。从MongoDB 3.4版本开始，允许n个分片的集群同时执行n/2个并发任务。

随着版本的迭代，MongoDB迁移的能力也在逐步提升。从MongoDB 4.0版本开始，支持在迁移数据的过程中并发地读取源端和写入目标端，迁移的整体性能提升了约40%。这样也使得新加入的分片能更快地分担集群的访问读写压力。

### **数据均衡带来的问题**

数据均衡会影响性能，在分片间进行数据块的迁移是一个“繁重”的工作，很容易带来磁盘I/O使用率飙升，或业务时延陡增等一些问题。因此，建议尽可能提升磁盘能力，如使用SSD。除此之外，我们还可以将数据均衡的窗口对齐到业务的低峰期以降低影响。

登录mongos，在config数据库上更新配置，代码如下：

​                use config sh.setBalancerState(true) db.settings.update(    {_id:"balancer"},    {$set:{activeWindow:{start:"02:00",stop:"04:00"}}},    {upsert:true} )              

在上述操作中启用了自动均衡器，同时在每天的凌晨2点到4点运行数据均衡操作

对分片集合中执行count命令可能会产生不准确的结果，mongos在处理count命令时会分别向各个分片发送请求，并累加最终的结果。如果分片上正在执行数据迁移，则可能导致重复的计算。替代办法是使用db.collection.countDocuments({})方法，该方法会执行聚合操作进行实时扫描，可以避免元数据读取的问题，但需要更长时间。

在执行数据库备份的期间，不能进行数据均衡操作，否则会产生不一致的备份数据。在备份操作之前，可以通过如下命令确认均衡器的状态:

1. sh.getBalancerState()：查看均衡器是否开启。
2. sh.isBalancerRunning()：查看均衡器是否正在运行。
3. sh.getBalancerWindow()：查看当前均衡的窗口设定。

# **事务简介**

事务（transaction）是传统数据库所具备的一项基本能力，其根本目的是为数据的可靠性与一致性提供保障。而在通常的实现中，事务包含了一个系列的数据库读写操作，这些操作要么全部完成，要么全部撤销。例如，在电子商城场景中，当顾客下单购买某件商品时，除了生成订单，还应该同时扣减商品的库存，这些操作应该被作为一个整体的执行单元进行处理，否则就会产生不一致的情况。

数据库事务需要包含4个基本特性，即常说的ACID，具体如下：

- 原子性（atomicity）：事务作为一个整体被执行，包含在其中的对数据库的操作要么全部被执行，要么都不执行。
-  一致性（consistency）：事务应确保数据库的状态从一个一致状态转变为另一个一致状态。一致状态的含义是数据库中的数据应满足完整性约束。
- 隔离性（isolation）：多个事务并发执行时，一个事务的执行不应影响其他事务的执行。
- 持久性（durability）：已被提交的事务对数据库的修改应该是永久性的。

## **MongoDB多文档事务**

在MongoDB中，对单个文档的操作是原子的。由于可以在单个文档结构中使用内嵌文档和数组来获得数据之间的关系，而不必跨多个文档和集合进行范式化，所以这种单文档原子性避免了许多实际场景中对多文档事务的需求。

对于那些需要对多个文档（在单个或多个集合中）进行原子性读写的场景，MongoDB支持多文档事务。而使用分布式事务，事务可以跨多个操作、集合、数据库、文档和分片使用。

MongoDB 虽然已经在 4.2 开始全面支持了多文档事务，但并不代表大家应该毫无节制地使用它。相反，对事务的使用原则应该是：能不用尽量不用。 通过合理地设计文档模型，可以规避绝大部分使用事务的必要性。

**使用事务的原则：**

- 无论何时，事务的使用总是能避免则避免； 
- 模型设计先于事务，尽可能用模型设计规避事务；
- 不要使用过大的事务（尽量控制在 1000 个文档更新以内）； 
- 当必须使用事务时，尽可能让涉及事务的文档分布在同一个分片上，这将有效地提高效率；

### **MongoDB对事务支持**

| 事务属性           | 支持程度                                                     |
| ------------------ | ------------------------------------------------------------ |
| Atomocity 原子性   | 单表单文档 ： 1.x 就支持复制集多表多行：4.0分片集群多表多行：4.2 |
| Consistency 一致性 | writeConcern, readConcern (3.2)                              |
| Isolation 隔离性   | readConcern (3.2)                                            |
| Durability 持久性  | Journal and Replication                                      |

### **使用方法** 

MongoDB 多文档事务的使用方式与关系数据库非常相似： 

​                try (ClientSession clientSession = client.startSession()) {   clientSession.startTransaction();    collection.insertOne(clientSession, docOne);    collection.insertOne(clientSession, docTwo);    clientSession.commitTransaction();  }              

## **writeConcern**

https://docs.mongodb.com/manual/reference/write-concern/

 writeConcern 决定一个写操作落到多少个节点上才算成功。MongoDB支持客户端灵活配置写入策略（writeConcern），以满足不同场景的需求。

语法格式：

​                { w: <value>, j: <boolean>, wtimeout: <number> }              

1. w: 数据写入到number个节点才向用客户端确认

- - {w: 0} 对客户端的写入不需要发送任何确认，适用于性能要求高，但不关注正确性的场景
  - {w: 1} 默认的writeConcern，数据写入到Primary就向客户端发送确认

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/44022)

- - {w: “majority”} 数据写入到副本集大多数成员后向客户端发送确认，适用于对数据安全性要求比较高的场景，该选项会降低写入性能

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/44020)

1. j: 写入操作的journal持久化后才向客户端确认

- - 默认为{j: false}，如果要求Primary写入持久化了才向客户端确认，则指定该选项为true

1. wtimeout: 写入超时时间，仅w的值大于1时有效。

- - 当指定{w: }时，数据需要成功写入number个节点才算成功，如果写入过程中有节点故障，可能导致这个条件一直不能满足，从而一直不能向客户端发送确认结果，针对这种情况，客户端可设置wtimeout选项来指定超时时间，当写入过程持续超过该时间仍未结束，则认为写入失败。

思考：对于5个节点的复制集来说，写操作落到多少个节点上才算是安全的?

1   2   3   4   5  majority

**测试**

包含延迟节点的3节点pss复制集

​                db.user.insertOne({name:"李四"},{writeConcern:{w:"majority"}}) # 等待延迟节点写入数据后才会响应 db.user.insertOne({name:"王五"},{writeConcern:{w:3}}) # 超时写入失败 db.user.insertOne({name:"小明"},{writeConcern:{w:3,wtimeout:3000}})              

**注意事项** 

- 虽然多于半数的 writeConcern 都是安全的，但通常只会设置 majority，因为这是等待写入延迟时间最短的选择； 
- 不要设置 writeConcern 等于总节点数，因为一旦有一个节点故障，所有写操作都将失败；
-  writeConcern 虽然会增加写操作延迟时间，但并不会显著增加集群压力，因此无论是否等待，写操作最终都会复制到所有节点上。设置 writeConcern 只是让写操作等待复制后再返回而已；
- 应对重要数据应用 {w: “majority”}，普通数据可以应用 {w: 1} 以确保最佳性能。

在读取数据的过程中我们需要关注以下两个问题： 

- 从哪里读？ 
- 什么样的数据可以读？ 

第一个问题是是由 readPreference 来解决，第二个问题则是由 readConcern 来解决

## **readPreference**

readPreference决定使用哪一个节点来满足正在发起的读请求。可选值包括：

- primary: 只选择主节点，默认模式； 
- primaryPreferred：优先选择主节点，如果主节点不可用则选择从节点； 
- secondary：只选择从节点； 
- secondaryPreferred：优先选择从节点， 如果从节点不可用则选择主节点； 
- nearest：根据客户端对节点的 Ping 值判断节点的远近，选择从最近的节点读取。

合理的 ReadPreference 可以极大地扩展复制集的读性能，降低访问延迟。

**readPreference 场景举例**

- 用户下订单后马上将用户转到订单详情页——primary/primaryPreferred。因为此时从节点可能还没复制到新订单；
- 用户查询自己下过的订单——secondary/secondaryPreferred。查询历史订单对 时效性通常没有太高要求； 
- 生成报表——secondary。报表对时效性要求不高，但资源需求大，可以在从节点单独处理，避免对线上用户造成影响； 
- 将用户上传的图片分发到全世界，让各地用户能够就近读取——nearest。每个地区的应用选择最近的节点读取数据。

**readPreference 配置**

通过 MongoDB 的连接串参数：

​                mongodb://host1:27107,host2:27107,host3:27017/?replicaSet=rs0&readPre ference=secondary              

通过 MongoDB 驱动程序 API：

​                MongoCollection.withReadPreference(ReadPreference readPref)              

Mongo Shell：

​                db.collection.find().readPref( "secondary" )              

**从节点读测试**

1. 主节点写入{count:1} , 观察该条数据在各个节点均可见 

​                \# mongo --host rs0/localhost:28017 rs0:PRIMARY> db.user.insert({count:1})              

在primary节点中调用readPref("secondary")查询从节点用直连方式（mongo localhost:28017）会查到数据，需要通过mongo --host rs0/localhost:28017方式连接复制集，参考： https://jira.mongodb.org/browse/SERVER-22289

1. 在两个从节点分别执行 db.fsyncLock() 来锁定写入（同步）

​                \# mongo localhost:28018 rs0:SECONDARY> rs.secondaryOk() rs0:SECONDARY> db.fsyncLock()              

1. 主节点写入 {count:2} 

​                rs0:PRIMARY> db.user.insert({count:2}) rs0:PRIMARY> db.user.find() rs0:PRIMARY> db.user.find().readPref("secondary") rs0:SECONDARY> db.user.find()              

1. 解除从节点锁定 db.fsyncUnlock() 

​                rs0:SECONDARY> db.fsyncUnlock()               

1. 主节点中查从节点数据

​                rs0:PRIMARY> db.user.find().readPref("secondary")              

   

**扩展：Tag**

readPreference 只能控制使用一类节点。Tag 则可以将节点选择控制到一个或几个节点。考虑以下场景：

- 一个 5 个节点的复制集；
- 3 个节点硬件较好，专用于服务线上客户；
- 2 个节点硬件较差，专用于生成报表；

可以使用 Tag 来达到这样的控制目的：

- 为 3 个较好的节点打上 {purpose: "online"}；
- 为 2 个较差的节点打上 {purpose: "analyse"}；
- 在线应用读取时指定 online，报表读取时指定 analyse。

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/44016)

​                \# 为复制集节点添加标签 conf = rs.conf() conf.members[1].tags = { purpose: "online"} conf.members[4].tags = { purpose: "analyse"} rs.reconfig(conf) #查询 db.collection.find({}).readPref( "secondary", [ {purpose: "analyse"} ] )              

**注意事项**

- 指定 readPreference 时也应注意高可用问题。例如将 readPreference 指定 primary，则发生故障转移不存在 primary 期间将没有节点可读。如果业务允许，则应选择 primaryPreferred；
- 使用 Tag 时也会遇到同样的问题，如果只有一个节点拥有一个特定 Tag，则在这个节点失效时将无节点可读。这在有时候是期望的结果，有时候不是。例如：

- - - 如果报表使用的节点失效，即使不生成报表，通常也不希望将报表负载转移到其他节点上，此时只有一个节点有报表 Tag 是合理的选择；
    - 如果线上节点失效，通常希望有替代节点，所以应该保持多个节点有同样的 Tag；

- Tag 有时需要与优先级、选举权综合考虑。例如做报表的节点通常不会希望它成为主节点，则优先级应为 0。

## **readConcern**

在 readPreference 选择了指定的节点后，readConcern 决定这个节点上的数据哪些是可读的，类似于关系数据库的隔离级别。可选值包括：

- available：读取所有可用的数据;
- local：读取所有可用且属于当前分片的数据;
- majority：读取在大多数节点上提交完成的数据;
- linearizable：可线性化读取文档，仅支持从主节点读;
- snapshot：读取最近快照中的数据，仅可用于多文档事务;

**readConcern: local 和 available**

在复制集中 local 和 available 是没有区别的，两者的区别主要体现在分片集上。

考虑以下场景：

- 一个 chunk x 正在从 shard1 向 shard2 迁移；
- 整个迁移过程中 chunk x 中的部分数据会在 shard1 和 shard2 中同时存在，但源分片 shard1仍然是chunk x 的负责方：

- - - 所有对 chunk x 的读写操作仍然进入 shard1；
    - config 中记录的信息 chunk x 仍然属于 shard1；

- 此时如果读 shard2，则会体现出 local 和 available 的区别：

- - - local：只取应该由 shard2 负责的数据（不包括 x）；
    - available：shard2 上有什么就读什么（包括 x）；

**注意事项：**

- 虽然看上去总是应该选择 local，但毕竟对结果集进行过滤会造成额外消耗。在一些无关紧要的场景（例如统计）下，也可以考虑 available；
- MongoDB <=3.6 不支持对从节点使用 {readConcern: "local"}；
- 从主节点读取数据时默认 readConcern 是 local，从从节点读取数据时默认readConcern 是 available（向前兼容原因）。

**readConcern: majority**

只读取大多数据节点上都提交了的数据。考虑如下场景：

- 集合中原有文档 {x: 0}；
- 将x值更新为 1；

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/44021)

如果在各节点上应用 {readConcern: "majority"} 来读取数据：

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/44017)

**考虑 t3 时刻的 Secondary1，此时：**

- 对于要求 majority 的读操作，它将返回 x=0；
- 对于不要求 majority 的读操作，它将返回 x=1；

如何实现？

节点上维护多个 x 版本（MVCC 机制），MongoDB 通过维护多个快照来链接不同的版本：

- 每个被大多数节点确认过的版本都将是一个快照；
- 快照持续到没有人使用为止才被删除；

**测试readConcern: majority vs local**

1. 安装 3 节点复制集。 
2. 注意配置文件内 server 参数 enableMajorityReadConcern

​                replication:  replSetName: rs0  enableMajorityReadConcern: true              

1. 将复制集中的两个从节点使用 db.fsyncLock() 锁住写入（模拟同步延迟）
2. 测试

​                rs0:PRIMARY> db.user.insert({count:10}) rs0:PRIMARY> db.user.find().readConcern("local") rs0:PRIMARY> db.user.find().readConcern("majority")               

主节点测试结果：

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/44023)

在某一个从节点上执行 db.fsyncUnlock()，从节点测试结果：

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/44015)

结论：

- 使用 local 参数，则可以直接查询到写入数据
- 使用 majority，只能查询到已经被多数节点确认过的数据
- update 与 remove 与上同理。

**readConcern: majority 与脏读**

MongoDB 中的回滚：

- 写操作到达大多数节点之前都是不安全的，一旦主节点崩溃，而从节点还没复制到该次操作，刚才的写操作就丢失了；
- 把一次写操作视为一个事务，从事务的角度，可以认为事务被回滚了。

所以从分布式系统的角度来看，事务的提交被提升到了分布式集群的多个节点级别的“提交”，而不再是单个节点上的“提交”。

在可能发生回滚的前提下考虑脏读问题：

- 如果在一次写操作到达大多数节点前读取了这个写操作，然后因为系统故障该操作

回滚了，则发生了脏读问题；

使用 {readConcern: "majority"} 可以有效避免脏读

**如何安全的读写分离**

考虑如下场景: 

1. 向主节点写入一条数据;
2. 立即从从节点读取这条数据。

思考： 如何保证自己能够读到刚刚写入的数据?

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/44018)

下述方式有可能读不到刚写入的订单

​                db.orders.insert({oid:101,sku:"kite",q:1}) db.orders.find({oid:101}).readPref("secondary")              

使用writeConcern+readConcern majority来解决

​                db.orders.insert({oid:101,sku:"kite",q:1},{writeConcern:{w:"majority"}}) db.orders.find({oid:101}).readPref("secondary").readConcern("majority")              

**readConcern: linearizable** 

只读取大多数节点确认过的数据。和 majority 最大差别是保证绝对的操作线性顺序 

- 在写操作自然时间后面的发生的读，一定可以读到之前的写 
- 只对读取单个文档时有效； 
- 可能导致非常慢的读，因此总是建议配合使用 maxTimeMS；

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/44019)

**readConcern: snapshot**

 {readConcern: “snapshot”} 只在多文档事务中生效。将一个事务的 readConcern 设置为 snapshot，将保证在事务中的读： 

- 不出现脏读；
- 不出现不可重复读；
- 不出现幻读。

 因为所有的读都将使用同一个快照，直到事务提交为止该快照才被释放。

**小结** 

- available：读取所有可用的数据 
- local：读取所有可用且属于当前分片的数据，默认设置 
- majority：数据读一致性的充分保证，可能你最需要关注的 
- linearizable：增强处理 majority 情况下主节点失联时候的例外情况 
- snapshot：最高隔离级别，接近于关系型数据库的Serializable

**事务隔离级别**

- 事务完成前，事务外的操作对该事务所做的修改不可访问

​                db.tx.insertMany([{ x: 1 }, { x: 2 }]) var session = db.getMongo().startSession() # 开启事务 session.startTransaction() var coll = session.getDatabase("test").getCollection("tx") #事务内修改 {x:1, y:1} coll.updateOne({x: 1}, {$set: {y: 1}}) #事务内查询 {x:1} coll.findOne({x: 1})  //{x:1, y:1} #事务外查询 {x:1} db.tx.findOne({x: 1})  //{x:1} #提交事务 session.commitTransaction() # 或者回滚事务 session.abortTransaction()              

- 如果事务内使用 {readConcern: “snapshot”}，则可以达到可重复读 Repeatable Read

​                var session = db.getMongo().startSession() session.startTransaction({ readConcern: {level: "snapshot"}, writeConcern: {w: "majority"}}) var coll = session.getDatabase('test').getCollection("tx") coll.findOne({x: 1})  db.tx.updateOne({x: 1}, {$set: {y: 1}}) db.tx.findOne({x: 1})  coll.findOne({x: 1})  session.abortTransaction()              

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/44008)

### **事务超时**

在执行事务的过程中，如果操作太多，或者存在一些长时间的等待，则可能会产生如下异常：

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/44011)

原因在于，默认情况下MongoDB会为每个事务设置1分钟的超时时间，如果在该时间内没有提交，就会强制将其终止。该超时时间可以通过transactionLifetimeLimitSecond变量设定。

### **事务写机制**

 MongoDB 的事务错误处理机制不同于关系数据库： 

- 当一个事务开始后，如果事务要修改的文档在事务外部被修改过，则事务修改这个 文档时会触发 Abort 错误，因为此时的修改冲突了。 这种情况下，只需要简单地重做事务就可以了； 
-  如果一个事务已经开始修改一个文档，在事务以外尝试修改同一个文档，则事务以外的修改会等待事务完成才能继续进行。

**写冲突测试**

开3个 mongo shell 均执行下述语句

​                var session = db.getMongo().startSession() session.startTransaction() var coll = session.getDatabase('test').getCollection("tx")              

窗口1： 正常结束

​                coll.updateOne({x: 1}, {$set: {y: 1}})                 

窗口2：  异常 – 解决方案：重启事务

​                coll.updateOne({x: 1}, {$set: {y: 2}})              

​    ![0](4%E3%80%81MongoDB%E5%88%86%E7%89%87%E9%9B%86%E7%BE%A4%E5%92%8C%E5%A4%9A%E6%96%87%E6%A1%A3%E4%BA%8B%E5%8A%A1%E8%AF%A6%E8%A7%A3/44010)

窗口3：事务外更新，需等待 

​                db.tx.updateOne({x: 1}, {$set: {y: 3}})              

### **注意事项** 

- 可以实现和关系型数据库类似的事务场景 
- 必须使用与 MongoDB 4.2 兼容的驱动； 
- 事务默认必须在 60 秒（可调）内完成，否则将被取消； 
- 涉及事务的分片不能使用仲裁节点； 
- 事务会影响 chunk 迁移效率。正在迁移的 chunk 也可能造成事务提交失败（重试 即可）；
- 多文档事务中的读操作必须使用主节点读； 
- readConcern 只应该在事务级别设置，不能设置在每次读写操作上。
