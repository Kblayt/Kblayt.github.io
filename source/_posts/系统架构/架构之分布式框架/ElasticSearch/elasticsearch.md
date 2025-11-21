---
title: elasticsearch
date: 2024-03-10 13:11:13
categories:
    - 系统架构
    - 架构之分布式框架
    - elasticsearch
tags:
    - elasticsearch
---

# **全文检索**

**数据分类：**

- 结构化数据： 固定格式，有限长度    比如mysql存的数据
- 非结构化数据：不定长，无固定格式   比如邮件，word文档，日志
- 半结构化数据： 前两者结合     比如xml，html

**搜索分类：**

- 结构化数据搜索：  使用关系型数据库

- 非结构化数据搜索

- - 顺序扫描
  - 全文检索

设想一个关于搜索的场景，假设我们要搜索一首诗句内容中带“前”字的古诗

| name       | content                                                      | author |
| ---------- | ------------------------------------------------------------ | ------ |
| 静夜思     | 床前明月光,疑是地上霜。举头望明月，低头思故乡。              | 李白   |
| 望庐山瀑布 | 日照香炉生紫烟，遥看瀑布挂前川。飞流直下三千尺,疑是银河落九天。 | 李白   |
| ...        | ...                                                          | ...    |

思考：用传统关系型数据库和ES 实现会有什么差别？

如果用像 MySQL 这样的 RDBMS 来存储古诗的话，我们应该会去使用这样的 SQL 去查询

```
select name from poems where content like "%前%"              
```

这种我们称为顺序扫描法，需要遍历所有的记录进行匹配。不但效率低，而且不符合我们搜索时的期望，比如我们在搜索“ABCD"这样的关键词时，通常还希望看到"A","AB","CD",“ABC”的搜索结果。

## **什么是全文检索**

全文检索是指：

- 通过一个程序扫描文本中的每一个单词，针对单词建立索引，并保存该单词在文本中的位置、以及出现的次数
- 用户查询时，通过之前建立好的索引来查询，将索引中单词对应的文本位置、出现的次数返回给用户，因为有了具体文本的位置，所以就可以将具体内容读取出来了

​    ![0](elasticsearch/46866)

搜索原理简单概括的话可以分为这么几步：

- 内容爬取，停顿词过滤比如一些无用的像"的"，“了”之类的语气词/连接词
- 内容分词，提取关键词
- 根据关键词建立倒排索引
- 用户输入关键词进行搜索

## **倒排索引**

索引就类似于目录，平时我们使用的都是索引，都是通过主键定位到某条数据，那么倒排索引呢，刚好相反，数据对应到主键。

​    ![0](elasticsearch/46781)

这里以一个博客文章的内容为例:

**正排索引（正向索引）**

| 文章ID | 文章标题           | 文章内容                                           |
| ------ | ------------------ | -------------------------------------------------- |
| 1      | 浅析JAVA设计模式   | JAVA设计模式是每一个JAVA程序员都应该掌握的进阶知识 |
| 2      | JAVA多线程设计模式 | JAVA多线程与设计模式结合                           |

**倒排索引（反向索引）**

假如，我们有一个站内搜索的功能，通过某个关键词来搜索相关的文章，那么这个关键词可能出现在标题中，也可能出现在文章内容中，那我们将会在创建或修改文章的时候，建立一个关键词与文章的对应关系表，这种，我们可以称之为倒排索引。

like %java设计模式%     java  设计模式

| 关键词   | 文章ID |
| -------- | ------ |
| JAVA     | 1,2    |
| 设计模式 | 1,2    |
| 多线程   | 2      |

简单理解，正向索引是通过key找value，反向索引则是通过value找key。ES底层在检索时底层使用的就是倒排索引。

# **ElasticSearch简介**

## **ElasticSearch是什么**

ElasticSearch（简称ES）是一个分布式、RESTful 风格的搜索和数据分析引擎，是用Java开发并且是当前最流行的开源的企业级搜索引擎，能够达到近实时搜索，稳定，可靠，快速，安装使用方便。

客户端支持Java、.NET（C#）、PHP、Python、Ruby等多种语言。

**官方网站:** https://www.elastic.co/

**下载地址：**https://www.elastic.co/cn/downloads/past-releases#elasticsearch

搜索引擎排名：

​    ![0](elasticsearch/46333)

参考网站：https://db-engines.com/en/ranking/search+engine

### **起源——Lucene**

- 基于Java语言开发的搜索引擎库类
- 创建于1999年，2005年成为Apache 顶级开源项目
- Lucene具有高性能、易扩展的优点
- Lucene的局限性︰

- - 只能基于Java语言开发
  - 类库的接口学习曲线陡峭
  - 原生并不支持水平扩展

### **Elasticsearch的诞生**

Elasticsearch是构建在Apache Lucene之上的开源分布式搜索引擎。

- 2004年 Shay Banon 基于Lucene开发了Compass

- 2010年 Shay Banon重写了Compass，取名Elasticsearch

- - 支持分布式，可水平扩展
  - 降低全文检索的学习曲线，可以被任何编程语言调用

​    ![0](elasticsearch/46329)

Elasticsearch 与 Lucene 核心库竞争的优势在于： 

- 完美封装了 Lucene 核心库，设计了友好的 Restful-API，开发者无需过多关注底层机制，直接开箱即用。
- 分片与副本机制，直接解决了集群下性能与高可用问题。

ES Server进程  3节点   raft  (奇数节点) 

数据分片 -》lucene实例   分片和副本数    1个ES节点可以有多个lucene实例。也可以指定一个索引的多个分片

​    ![0](elasticsearch/50604)

### **ElasticSearch版本特性**

5.x新特性

- Lucene 6.x， 性能提升，默认打分机制从TF-IDF改为BM 25

- 支持Ingest节点/ Painless Scripting / Completion suggested支持/原生的Java REST客户端

- Type标记成deprecated， 支持了Keyword的类型

- 性能优化

- - 内部引擎移除了避免同一文档并发更新的竞争锁，带来15% - 20%的性能提升
  - Instant aggregation,支持分片，上聚合的缓存
  - 新增了Profile API

6.x新特性

- Lucene 7.x

- 新功能

- - 跨集群复制(CCR)
  - 索引生命周期管理
  - SQL的支持

- 更友好的的升级及数据迁移

- - 在主要版本之间的迁移更为简化，体验升级
  - 全新的基于操作的数据复制框架，可加快恢复数据

- 性能优化

- - 有效存储稀疏字段的新方法，降低了存储成本
  - 在索引时进行排序，可加快排序的查询性能

7.x新特性

- Lucene 8.0

- 重大改进-正式废除单个索引下多Type的支持
- 7.1开始，Security 功能免费使用
- ECK - Elasticseach Operator on Kubernetes

- 新功能

- - New Cluster coordination
  - Feature——Complete High Level REST Client
  - Script Score Query

- 性能优化

- - 默认的Primary Shard数从5改为1,避免Over Sharding
  - 性能优化， 更快的Top K

8.x新特性

- Rest API相比较7.x而言做了比较大的改动（比如彻底删除_type）
- 默认开启安全配置
- 存储空间优化：对倒排文件使用新的编码集，对于keyword、match_only_text、text类型字段有效，有3.5%的空间优化提升，对于新建索引和segment自动生效。
- 优化geo_point，geo_shape类型的索引（写入）效率：15%的提升。
- 技术预览版KNN API发布，（K邻近算法），跟推荐系统、自然语言排名相关。
- https://www.elastic.co/guide/en/elastic-stack/current/elasticsearch-breaking-changes.html

### **ElasticSearch vs Solr** 

Solr 是第一个基于 Lucene 核心库功能完备的搜索引擎产品，诞生远早于 Elasticsearch。

当单纯的对已有数据进行搜索时，Solr更快。当实时建立索引时, Solr会产生io阻塞，查询性能较差, Elasticsearch具有明显的优势。

​    ![0](elasticsearch/50800)

​    ![0](elasticsearch/46439)

大型互联网公司，实际生产环境测试，将搜索引擎从Solr转到 Elasticsearch以后的平均查询速度有了50倍的提升。

​    ![0](elasticsearch/46437)

**总结：**

- Solr 利用 Zookeeper 进行分布式管理，而Elasticsearch 自身带有分布式协调管理功能。 
- Solr 支持更多格式的数据，比如JSON、XML、CSV，而 Elasticsearch 仅支持json文件格式。 
- Solr 在传统的搜索应用中表现好于 Elasticsearch，但在处理实时搜索应用时效率明显低于 Elasticsearch。 
- Solr 是传统搜索应用的有力解决方案，但 Elasticsearch更适用于新兴的实时搜索应用。

## **Elastic Stack介绍**

   在Elastic Stack之前我们听说过ELK，ELK分别是Elasticsearch，Logstash，Kibana这三款软件在一起的简称，在发展的过程中又有新的成员Beats的加入，就形成了Elastic Stack。

​    ![0](elasticsearch/46325)

 　　　　　　　              Elastic Stack生态圈

在Elastic Stack生态圈中Elasticsearch作为数据存储和搜索，是生态圈的基石，Kibana在上层提供用户一个可视化及操作的界面，Logstash和Beat可以对数据进行收集。在上图的右侧X-Pack部分则是Elastic公司提供的商业项目。

指标分析/日志分析：

​    ![0](elasticsearch/46442)

## **ElasticSearch应用场景**

- 站内搜索
- 日志管理与分析
- 大数据分析
- 应用性能监控
- 机器学习

国内现在有大量的公司都在使用 Elasticsearch，包括携程、滴滴、今日头条、饿了么、360安全、小米、vivo等诸多知名公司。除了搜索之外，结合Kibana、Logstash、Beats，Elastic Stack还被广泛运用在大数据近实时分析领域，包括日志分析、指标监控、信息安全等多个领域。它可以帮助你探索海量结构化、非结构化数据，按需创建可视化报表，对监控数据设置报警阈值，甚至通过使用机器学习技术，自动识别异常状况。

**通用数据处理流程：**

​    ![0](elasticsearch/46527)

# **ElasticSearch快速开始**

## **ElasticSearch安装运行**

### **环境准备**

- 运行Elasticsearch，需安装并配置JDK

- - 设置$JAVA_HOME

- 各个版本对Java的依赖 https://www.elastic.co/support/matrix#matrix_jvm

- - Elasticsearch 5需要Java 8以上的版本	
  - Elasticsearch 从6.5开始支持Java 11
  - 7.0开始，内置了Java环境

- ES比较耗内存，建议虚拟机4G或以上内存，jvm1g以上的内存分配

可以参考es的环境文件elasticsearch-env.bat

​    ![0](elasticsearch/46255)

ES的jdk环境生效的优先级配置ES_JAVA_HOME>JAVA_HOME>ES_HOME

### **下载并解压ElasticSearch**

下载地址： https://www.elastic.co/cn/downloads/past-releases#elasticsearch

选择版本：7.17.3

​    ![0](elasticsearch/46614)

**ElasticSearch文件目录结构**

| 目录    | 描述                                                         |
| ------- | ------------------------------------------------------------ |
| bin     | 脚本文件，包括启动elasticsearch，安装插件，运行统计数据等    |
| config  | 配置文件目录，如elasticsearch配置、角色配置、jvm配置等。     |
| jdk     | java运行环境                                                 |
| data    | 默认的数据存放目录，包含节点、分片、索引、文档的所有数据，生产环境需要修改。 |
| lib     | elasticsearch依赖的Java类库                                  |
| logs    | 默认的日志文件存储路径，生产环境需要修改。                   |
| modules | 包含所有的Elasticsearch模块，如Cluster、Discovery、Indices等。 |
| plugins | 已安装插件目录                                               |

**主配置文件elasticsearch.yml**

- cluster.name

当前节点所属集群名称，多个节点如果要组成同一个集群，那么集群名称一定要配置成相同。默认值elasticsearch，生产环境建议根据ES集群的使用目的修改成合适的名字。

- node.name

当前节点名称，默认值当前节点部署所在机器的主机名，所以如果一台机器上要起多个ES节点的话，需要通过配置该属性明确指定不同的节点名称。

- path.data

配置数据存储目录，比如索引数据等，默认值 $ES_HOME/data，生产环境下强烈建议部署到另外的安全目录，防止ES升级导致数据被误删除。

- path.logs

配置日志存储目录，比如运行日志和集群健康信息等，默认值 $ES_HOME/logs，生产环境下强烈建议部署到另外的安全目录，防止ES升级导致数据被误删除。

- bootstrap.memory_lock

配置ES启动时是否进行内存锁定检查，默认值true。

ES对于内存的需求比较大，一般生产环境建议配置大内存，如果内存不足，容易导致内存交换到磁盘，严重影响ES的性能。所以默认启动时进行相应大小内存的锁定，如果无法锁定则会启动失败。

非生产环境可能机器内存本身就很小，能够供给ES使用的就更小，如果该参数配置为true的话很可能导致无法锁定内存以致ES无法成功启动，此时可以修改为false。

- network.host

配置能够访问当前节点的主机，默认值为当前节点所在机器的本机回环地址127.0.0.1 和[::1]，这就导致默认情况下只能通过当前节点所在主机访问当前节点。可以配置为 0.0.0.0 ，表示所有主机均可访问。

- http.port

配置当前ES节点对外提供服务的http端口，默认值 9200

- discovery.seed_hosts

配置参与集群节点发现过程的主机列表，说白一点就是集群中所有节点所在的主机列表，可以是具体的IP地址，也可以是可解析的域名。

- cluster.initial_master_nodes

配置ES集群初始化时参与master选举的节点名称列表，必须与node.name配置的一致。ES集群首次构建完成后，应该将集群中所有节点的配置文件中的cluster.initial_master_nodes配置项移除，重启集群或者将新节点加入某个已存在的集群时切记不要设置该配置项。

```
#ES开启远程访问  
network.host: 0.0.0.0
```

### **修改JVM配置**

修改config/jvm.options配置文件，调整jvm堆内存大小

```
vim jvm.options
-Xms4g
-Xmx4g
```

配置的建议

- Xms和Xms设置成—样
- Xmx不要超过机器内存的50%
- 不要超过30GB - https://www.elastic.co/cn/blog/a-heap-of-trouble

### **启动ElasticSearch服务**

**Windows**

**直接运行elasticsearch.bat**

**Linux（centos7）**

ES不允许使用root账号启动服务，如果你当前账号是root，则需要创建一个专有账户

```
#非root用户
bin/elasticsearch 

# -d 后台启动
bin/elasticsearch -d
```

​    ![0](elasticsearch/49275)

注意：es默认不能用root用户启动，生产环境建议为elasticsearch创建用户。

```
#为elaticsearch创建用户并赋予相应权限
adduser es
passwd es
chown -R es:es elasticsearch-17.3
```

​      

运行http://localhost:9200/

​    ![0](elasticsearch/46552)

如果ES服务启动异常，会有提示：

​    ![0](elasticsearch/48965)

**启动ES服务常见错误解决方案**

[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]

ES因为需要大量的创建索引文件，需要大量的打开系统的文件，所以我们需要解除linux系统当中打开文件最大数目的限制，不然ES启动就会抛错

```
#切换到root用户
vim /etc/security/limits.conf

末尾添加如下配置：
  *	    soft 	nofile 	65536
  *     hard 	nofile 	65536
  *     soft 	nproc 	4096
  *	    hard 	nproc 	4096
```

​    

[2]: max number of threads [1024] for user [es] is too low, increase to at least [4096]

无法创建本地线程问题,用户最大可创建线程数太小

```
vim /etc/security/limits.d/20-nproc.conf

改为如下配置：
* soft nproc 4096
```

[3]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

最大虚拟内存太小,调大系统的虚拟内存

```
vim /etc/sysctl.conf
追加以下内容：
vm.max_map_count=262144
保存退出之后执行如下命令：
sysctl -p
```

​      

[4]: the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured

缺少默认配置，至少需要配置discovery.seed_hosts/discovery.seed_providers/cluster.initial_master_nodes中的一个参数.

- discovery.seed_hosts:  集群主机列表
- discovery.seed_providers: 基于配置文件配置集群主机列表
- cluster.initial_master_nodes: 启动时初始化的参与选主的node，生产环境必填

```
vim config/elasticsearch.yml
#添加配置
discovery.seed_hosts: ["127.0.0.1"]
cluster.initial_master_nodes: ["node-1"]

#或者  单节点（集群单节点）
discovery.type: single-node
```

 

### **客户端Kibana安装**

Kibana是一个开源分析和可视化平台，旨在与Elasticsearch协同工作。

**1）下载并解压缩Kibana**

下载地址：https://www.elastic.co/cn/downloads/past-releases#kibana

选择版本：7.17.3

​    ![0](elasticsearch/46611)

**2）修改Kibana.yml**

```
vim config/kibana.yml

server.port: 5601
server.host: "localhost"  #服务器ip
elasticsearch.hosts: ["http://localhost:9200"]  #elasticsearch的访问地址
i18n.locale: "zh-CN"   #Kibana汉化
```

  

**3）运行Kibana**

注意：kibana也需要非root用户启动

```
bin/kibana
#后台启动
nohup  bin/kibana &

#查询kibana进程
netstat -tunlp | grep 5601
```

​       

**访问Kibana:** [**http://localhost:5601/**](http://localhost:5601/)

​    ![0](elasticsearch/46817)

**cat API**

```
/_cat/allocation         #查看单节点的shard分配整体情况
/_cat/shards          #查看各shard的详细情况
/_cat/shards/{index}     #查看指定分片的详细情况
/_cat/master          #查看master节点信息
/_cat/nodes           #查看所有节点信息
/_cat/indices         #查看集群中所有index的详细信息
/_cat/indices/{index}      #查看集群中指定index的详细信息
/_cat/segments        #查看各index的segment详细信息,包括segment名, 所属shard, 内存(磁盘)占用大小, 是否刷盘
/_cat/segments/{index}#查看指定index的segment详细信息
/_cat/count           #查看当前集群的doc数量
/_cat/count/{index}   #查看指定索引的doc数量
/_cat/recovery        #查看集群内每个shard的recovery过程.调整replica。
/_cat/recovery/{index}#查看指定索引shard的recovery过程
/_cat/health          #查看集群当前状态：红、黄、绿
/_cat/pending_tasks   #查看当前集群的pending task
/_cat/aliases         #查看集群中所有alias信息,路由配置等
/_cat/aliases/{alias} #查看指定索引的alias信息
/_cat/thread_pool     #查看集群各节点内部不同类型的threadpool的统计信息,
/_cat/plugins         #查看集群各个节点上的plugin信息
/_cat/fielddata       #查看当前集群各个节点的fielddata内存使用情况
/_cat/fielddata/{fields}     #查看指定field的内存使用情况,里面传field属性对应的值
/_cat/nodeattrs              #查看单节点的自定义属性
/_cat/repositories           #输出集群中注册快照存储库
/_cat/templates              #输出当前正在存在的模板信息
```

   

### **Elasticsearch安装分词插件**

Elasticsearch提供插件机制对系统进行扩展

以安装analysis-icu这个分词插件为例

**在线安装**

```
#查看已安装插件
bin/elasticsearch-plugin list
#安装插件
bin/elasticsearch-plugin install analysis-icu
#删除插件
bin/elasticsearch-plugin remove analysis-icu
```

   

注意：安装和删除完插件后，需要重启ES服务才能生效。

测试分词效果

```
POST _analyze
{
    "analyzer":"icu_analyzer",
    "text":"中华人民共和国"
}
```

​      

​    ![0](elasticsearch/46920)

**离线安装**

本地下载相应的插件，解压，然后手动上传到elasticsearch的plugins目录，然后重启ES实例就可以了。

比如ik中文分词插件：https://github.com/medcl/elasticsearch-analysis-ik

测试分词效果

```
#ES的默认分词设置是standard，会单字拆分
POST _analyze
{
    "analyzer":"standard",
    "text":"中华人民共和国"
}

#ik_smart:会做最粗粒度的拆
POST _analyze
{
    "analyzer": "ik_smart",
    "text": "中华人民共和国"
 }

#ik_max_word:会将文本做最细粒度的拆分
POST _analyze
{
    "analyzer":"ik_max_word",
    "text":"中华人民共和国"
}

```

​    

创建索引时可以指定IK分词器作为默认分词器

```
PUT /es_db
{
    "settings" : {
        "index" : {
            "analysis.analyzer.default.type": "ik_max_word"
        }
    }
}
```

​     

​    ![0](elasticsearch/49281)

## **ElasticSearch基本概念**

### **关系型数据库 VS ElasticSearch**

- 在7.0之前，一个 Index可以设置多个Types

- 目前Type已经被Deprecated，7.0开始，一个索引只能创建一个Type - “_doc”

- 传统关系型数据库和Elasticsearch的区别:

- - Elasticsearch- Schemaless /相关性/高性能全文检索
  - RDMS —事务性/ Join	

​    ![0](elasticsearch/46708)

### **索引（Index）**

一个索引就是一个拥有几分相似特征的文档的集合。比如说，可以有一个客户数据的索引，另一个产品目录的索引，还有一个订单数据的索引。

一个索引由一个名字来标识（必须全部是小写字母的），并且当我们要对对应于这个索引中的文档进行索引、搜索、更新和删除的时候，都要使用到这个名字。

​    ![0](elasticsearch/46764)

### **文档（Document）**

- Elasticsearch是面向文档的，文档是所有可搜索数据的最小单位。

- - 日志文件中的日志项
  - 一本电影的具体信息/一张唱片的详细信息
  - MP3播放器里的一首歌/一篇PDF文档中的具体内容

- 文档会被序列化成JSON格式，保存在Elasticsearch中

- - JSON对象由字段组成
  - 每个字段都有对应的字段类型(字符串/数值/布尔/日期/二进制/范围类型)

- 每个文档都有一个Unique ID

- - 可以自己指定ID或者通过Elasticsearch自动生成

- 一篇文档包含了一系列字段，类似数据库表中的一条记录

- JSON文档，格式灵活，不需要预先定义格式

- - 字段的类型可以指定或者通过Elasticsearch自动推算
  - 支持数组/支持嵌套

**文档元数据**

​    ![0](elasticsearch/46726)

元数据，用于标注文档的相关信息：

- _index：文档所属的索引名
-  _type：文档所属的类型名
- _id：文档唯—ld
- _source: 文档的原始Json数据
- _version:  文档的版本号，修改删除操作_version都会自增1
- _seq_no:  和_version一样，一旦数据发生更改，数据也一直是累计的。Shard级别严格递增，保证后写入的Doc的_seq_no大于先写入的Doc的_seq_no。
- _primary_term: _primary_term主要是用来恢复数据时处理当多个文档的_seq_no一样时的冲突，避免Primary Shard上的写入被覆盖。每当Primary Shard发生重新分配时，比如重启，Primary选举等，_primary_term会递增1。

## **ElasticSearch索引操作**

https://www.elastic.co/guide/en/elasticsearch/reference/7.17/index.html

### **创建索引**

索引命名必须小写，不能以下划线开头

格式: PUT /索引名称

```
#创建索引
PUT /es_db
{
	"mappings": {
		"properties": {
			"字段名":{
				"type": "text", //字段类型
				"analyzer": "ik_smart" //分词器选择
			},
			"字段名2":{
				"type": "keyword",
				"index": "false"
			}
			"字段名3":{
				"properties": {
					"子字段": {
						"type": "keyword"
					}
				}
			}
		}
	}
}

字段类型：text，keyword，日期类型，布尔类型，数值类型，数组类型，对象类型，范围类型

#创建索引时可以设置分片数和副本数
PUT /es_db
{
    "settings" : {
        "number_of_shards" : 3,
        "number_of_replicas" : 2
    }
}

#修改索引配置
PUT /es_db/_settings
{
    "index" : {
        "number_of_replicas" : 1
    }
}
```

   

​    ![0](elasticsearch/46891)

### **查询索引**

格式: GET /索引名称

```
#查询索引
GET /es_db

#es_db是否存在
HEAD /es_db
```

​     

​    ![0](elasticsearch/46896)

​		

### **删除索引**

格式: DELETE /索引名称

```
DELETE /es_db
```

### 修改索引

索引库和mapping一旦创建无法修改，但是可以添加新的字段

```
PUT /索引库名/_mapping
{
	"properties":{
		"新字段名":{
			"type":"integer"
		}
	}
}
```



## **ElasticSearch文档操作**

示例数据

```
PUT /es_db
{
    "settings" : {
        "index" : {
            "analysis.analyzer.default.type": "ik_max_word"
        }
    }
}

PUT /es_db/_doc/1
{
"name": "张三",
"sex": 1,
"age": 25,
"address": "广州天河公园",
"remark": "java developer"
}
PUT /es_db/_doc/2
{
"name": "李四",
"sex": 1,
"age": 28,
"address": "广州荔湾大厦",
"remark": "java assistant"
}

PUT /es_db/_doc/3
{
"name": "王五",
"sex": 0,
"age": 26,
"address": "广州白云山公园",
"remark": "php developer"
}

PUT /es_db/_doc/4
{
"name": "赵六",
"sex": 0,
"age": 22,
"address": "长沙橘子洲",
"remark": "python assistant"
}

PUT /es_db/_doc/5
{
"name": "张龙",
"sex": 0,
"age": 19,
"address": "长沙麓谷企业广场",
"remark": "java architect assistant"
}	
	
PUT /es_db/_doc/6
{
"name": "赵虎",
"sex": 1,
"age": 32,
"address": "长沙麓谷兴工国际产业园",
"remark": "java architect"
}	
```

​           

### **添加（索引）文档** 

- 格式: [PUT | POST] /索引名称/[_doc  | _create ]/id

```
# 创建文档,指定id
# 如果id不存在，创建新的文档，否则先删除现有文档，再创建新的文档，版本会增加
PUT /es_db/_doc/1
{
"name": "张三",
"sex": 1,
"age": 25,
"address": "广州天河公园",
"remark": "java developer"
}	

#创建文档，ES生成id
POST /es_db/_doc
{
"name": "张三",
"sex": 1,
"age": 25,
"address": "广州天河公园",
"remark": "java developer"
}
```

​        

​    ![0](elasticsearch/46918)

注意:POST和PUT都能起到创建/更新的作用，PUT需要对一个具体的资源进行操作也就是要确定id才能进行更新/创建，而POST是可以针对整个资源集合进行操作的，如果不写id就由ES生成一个唯一id进行创建新文档，如果填了id那就针对这个id的文档进行创建/更新

​    ![0](elasticsearch/47001)

Create -如果ID已经存在，会失败

​    ![0](elasticsearch/46977)

### **修改文档** 

- 全量更新，整个json都会替换，格式: [PUT | POST] /索引名称/_doc/id

如果文档存在，现有文档会被删除，新的文档会被索引

```
# 全量更新，替换整个json
PUT /es_db/_doc/1
{
"name": "张三",
"sex": 1,
"age": 25
}

#查询文档
GET /es_db/_doc/1
```

​        

​    ![0](elasticsearch/46942)

- 使用_update部分更新，格式: POST /索引名称/_update/id

update不会删除原来的文档，而是实现真正的数据更新

```
# 部分更新：在原有文档上更新
# Update -文档必须已经存在，更新只会对相应字段做增量修改
POST /es_db/_update/1
{
  "doc": {
    "age": 28
  }
}

#查询文档
GET /es_db/_doc/1
```

​     

​    ![0](elasticsearch/46988)

- 使用 _update_by_query 更新文档

```
POST /es_db/_update_by_query
{
  "query": { 
    "match": {
      "_id": 1
    }
  },
  "script": {
    "source": "ctx._source.age = 30"
  }
}
```

​    ![0](elasticsearch/47023)

### **并发场景下修改文档**

_seq_no和_primary_term是对_version的优化，7.X版本的ES默认使用这种方式控制版本，所以当在高并发环境下使用乐观锁机制修改文档时，要带上当前文档的_seq_no和_primary_term进行更新：

```
POST /es_db/_doc/2?if_seq_no=21&if_primary_term=6
{
  "name": "李四xxx"
}
```

如果版本号不对，会抛出版本冲突异常，如下图：

​    ![0](elasticsearch/50609)

### **查询文档**

- 根据id查询文档，格式: GET /索引名称/_doc/id

```
GET /es_db/_doc/1
```

​        

- 条件查询 _search，格式： /索引名称/_doc/_search

```
# 查询前10条文档
GET /es_db/_doc/_search
```

ES Search API提供了两种条件查询搜索方式：

- REST风格的请求URI，直接将参数带过去
- 封装到request body中，这种方式可以定义更加易读的JSON格式

```
#通过URI搜索，使用“q”指定查询字符串，“query string syntax” KV键值对

#条件查询, 如要查询age等于28岁的 _search?q=*:***
GET /es_db/_doc/_search?q=age:28

#范围查询, 如要查询age在25至26岁之间的 _search?q=***[** TO **]  注意: TO 必须为大写
GET /es_db/_doc/_search?q=age[25 TO 26]

#查询年龄小于等于28岁的 :<=
GET /es_db/_doc/_search?q=age:<=28
#查询年龄大于28前的 :>
GET /es_db/_doc/_search?q=age:>28

#分页查询 from=*&size=*
GET /es_db/_doc/_search?q=age[25 TO 26]&from=0&size=1

#对查询结果只输出某些字段 _source=字段,字段
GET /es_db/_doc/_search?_source=name,age

#对查询结果排序 sort=字段:desc/asc
GET /es_db/_doc/_search?sort=age:desc
```

​        

通过请求体的搜索方式会在第二节课中详细讲解（DSL）

```
GET /es_db/_search
{
  "query": {
    "match": {
      "address": "广州白云"
    }
  }
}
```

### **删除文档**

格式: DELETE /索引名称/_doc/id

```
DELETE /es_db/_doc/1
```

## **ElasticSearch文档批量操作**

批量操作可以减少网络连接所产生的开销，提升性能

- 支持在一次API调用中，对不同的索引进行操作
- 可以在URI中指定Index，也可以在请求的Payload中进行
- 操作中单条操作失败，并不会影响其他操作
- 返回结果包括了每一条操作执行的结果

### **批量写入**

批量对文档进行写操作是通过_bulk的API来实现的

- 请求方式：POST
- 请求地址：_bulk
- 请求参数：通过_bulk操作文档，一般至少有两行参数(或偶数行参数)

- - 第一行参数为指定操作的类型及操作的对象(index,type和id)
  - 第二行参数才是操作的数据

参数类似于：

```
{"actionName":{"_index":"indexName", "_type":"typeName","_id":"id"}}
{"field1":"value1", "field2":"value2"}
```

 

- actionName：表示操作类型，主要有create,index,delete和update

**批量创建文档create**

```
POST _bulk
{"create":{"_index":"article", "_type":"_doc", "_id":3}}
{"id":3,"title":"fox老师","content":"fox老师666","tags":["java", "面向对象"],"create_time":1554015482530}
{"create":{"_index":"article", "_type":"_doc", "_id":4}}
{"id":4,"title":"mark老师","content":"mark老师NB","tags":["java", "面向对象"],"create_time":1554015482530}
```

​         

**普通创建或全量替换index**

```
POST _bulk
{"index":{"_index":"article", "_type":"_doc", "_id":3}}
{"id":3,"title":"图灵徐庶老师","content":"图灵学院徐庶老师666","tags":["java", "面向对象"],"create_time":1554015482530}
{"index":{"_index":"article", "_type":"_doc", "_id":4}}
{"id":4,"title":"图灵诸葛老师","content":"图灵学院诸葛老师NB","tags":["java", "面向对象"],"create_time":1554015482530}
```

​          

- 如果原文档不存在，则是创建
- 如果原文档存在，则是替换(全量修改原文档)

**批量删除delete**

```
POST _bulk
{"delete":{"_index":"article", "_type":"_doc", "_id":3}}
{"delete":{"_index":"article", "_type":"_doc", "_id":4}}
```

**批量修改update**

```
POST _bulk
{"update":{"_index":"article", "_type":"_doc", "_id":3}}
{"doc":{"title":"ES大法必修内功"}}
{"update":{"_index":"article", "_type":"_doc", "_id":4}}
{"doc":{"create_time":1554018421008}}
```

  

**组合应用**

```
POST _bulk
{"create":{"_index":"article", "_type":"_doc", "_id":3}}
{"id":3,"title":"fox老师","content":"fox老师666","tags":["java", "面向对象"],"create_time":1554015482530}
{"delete":{"_index":"article", "_type":"_doc", "_id":3}}
{"update":{"_index":"article", "_type":"_doc", "_id":4}}
{"doc":{"create_time":1554018421008}}
```

### **批量读取**

es的批量查询可以使用mget和msearch两种。其中mget是需要我们知道它的id，可以指定不同的index，也可以指定返回值source。msearch可以通过字段查询来进行一个批量的查找。

**_mget**

```
#可以通过ID批量获取不同index和type的数据
GET _mget
{
"docs": [
{
"_index": "es_db",
"_id": 1
},
{
"_index": "article",
"_id": 4
}
]
}

#可以通过ID批量获取es_db的数据
GET /es_db/_mget
{
"docs": [
{
"_id": 1
},
{
"_id": 4
}
]
}
#简化后
GET /es_db/_mget 
{
 "ids":["1","2"]  
 }
```

​      

​    ![0](elasticsearch/47101)

**_msearch**

在_msearch中，请求格式和bulk类似。查询一条数据需要两个对象，第一个设置index和type，第二个设置查询语句。查询语句和search相同。如果只是查询一个index，我们可以在url中带上index，这样，如果查该index可以直接用空对象表示。

```
GET /es_db/_msearch
{}
{"query" : {"match_all" : {}}, "from" : 0, "size" : 2}
{"index" : "article"}
{"query" : {"match_all" : {}}}
```

​    ![0](elasticsearch/47219)
