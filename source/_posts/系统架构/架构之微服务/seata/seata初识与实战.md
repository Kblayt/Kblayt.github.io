---
title: seata初识与实战
date: 2024-03-03 13:18:40
categories:
    - 系统架构
    - 架构之微服务组件
    - seata
tags:
    - seata
---

> 参考：https://zhuanlan.zhihu.com/p/250516428

# **1. 分布式事务简介**

分布式事务：https://www.processon.com/view/link/61cd52fb0e3e7441570801ab

## **1.1 本地事务**

​    大多数场景下，我们的应用都只需要操作单一的数据库，这种情况下的事务称之为本地事务(Local Transaction)。本地事务的ACID特性是数据库直接提供支持。本地事务应用架构如下所示：

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54181)

在JDBC编程中，我们通过java.sql.Connection对象来开启、关闭或者提交事务。代码如下所示：

```
Connection conn = ... //获取数据库连接
conn.setAutoCommit(false); //开启事务
try{
   //...执行增删改查sql
   conn.commit(); //提交事务
}catch (Exception e) {
  conn.rollback();//事务回滚
}finally{
   conn.close();//关闭链接
}
```

## **1.2 分布式事务**

在微服务架构中，完成某一个业务功能可能需要横跨多个服务，操作多个数据库。这就涉及到到了分布式事务，需要操作的资源位于多个资源服务器上，而应用需要保证对于多个资源服务器的数据操作，要么全部成功，要么全部失败。本质上来说，分布式事务就是为了保证不同资源服务器的数据一致性。

  

### **典型的分布式事务应用场景**

**1) 跨库事务**

   跨库事务指的是，一个应用某个功能需要操作多个库，不同的库中存储不同的业务数据。下图演示了一个服务同时操作2个库的情况： 

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54186)

**2) 分库分表**

   通常一个库数据量比较大或者预期未来的数据量比较大，都会进行分库分表。如下图，将数据库B拆分成了2个库： 

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54185)

  对于分库分表的情况，一般开发人员都会使用一些数据库中间件来降低sql操作的复杂性。如，对于sql：insert into user(id,name) values (1,"张三"),(2,"李四")。这条sql是操作单库的语法，单库情况下，可以保证事务的一致性。 但是由于现在进行了分库分表，开发人员希望将1号记录插入分库1，2号记录插入分库2。所以数据库中间件要将其改写为2条sql，分别插入两个不同的分库，此时要保证两个库要不都成功，要不都失败，因此基本上所有的数据库中间件都面临着分布式事务的问题。

**3) 微服务架构**

  下图演示了一个3个服务之间彼此调用的微服务架构：

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54184)

  Service A完成某个功能需要直接操作数据库，同时需要调用Service B和Service C，而Service B又同时操作了2个数据库，Service C也操作了一个库。需要保证这些跨服务调用对多个数据库的操作要么都成功，要么都失败，实际上这可能是最典型的分布式事务场景。

   小结：上述讨论的分布式事务场景中，无一例外的都直接或者间接的操作了多个数据库。如何保证事务的ACID特性，对于分布式事务实现方案而言，是非常大的挑战。同时，分布式事务实现方案还必须要考虑性能的问题，如果为了严格保证ACID特性，导致性能严重下降，那么对于一些要求快速响应的业务，是无法接受的。

## **1.3 两阶段提交协议(2PC)**

两阶段提交（Two Phase Commit），就是将提交(commit)过程划分为2个阶段(Phase)：

**阶段1：**

  TM通知各个RM准备提交它们的事务分支。如果RM判断自己进行的工作可以被提交，那就对工作内容进行持久化，再给TM肯定答复；要是发生了其他情况，那给TM的都是否定答复。

  以mysql数据库为例，在第一阶段，事务管理器向所有涉及到的数据库服务器发出prepare"准备提交"请求，数据库收到请求后执行数据修改和日志记录等处理，处理完成后只是把事务的状态改成"可以提交",然后把结果返回给事务管理器。

**阶段2**

  TM根据阶段1各个RM prepare的结果，决定是提交还是回滚事务。如果所有的RM都prepare成功，那么TM通知所有的RM进行提交；如果有RM prepare失败的话，则TM通知所有RM回滚自己的事务分支。

​    以mysql数据库为例，如果第一阶段中所有数据库都prepare成功，那么事务管理器向数据库服务器发出"确认提交"请求，数据库服务器把事务的"可以提交"状态改为"提交完成"状态，然后返回应答。如果在第一阶段内有任何一个数据库的操作发生了错误，或者事务管理器收不到某个数据库的回应，则认为事务失败，回撤所有数据库的事务。数据库服务器收不到第二阶段的确认提交请求，也会把"可以提交"的事务回撤。

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54220)

两阶段提交方案下全局事务的ACID特性，是依赖于RM的。一个全局事务内部包含了多个独立的事务分支，这一组事务分支要么都成功，要么都失败。各个事务分支的ACID特性共同构成了全局事务的ACID特性。也就是将单个事务分支支持的ACID特性提升一个层次到分布式事务的范畴。

### **2PC存在的问题**

- **同步阻塞问题**

2PC 中的参与者是阻塞的。在第一阶段收到请求后就会预先锁定资源，一直到 commit 后才会释放。

- **单点故障**

由于协调者的重要性，一旦协调者TM发生故障，参与者RM会一直阻塞下去。尤其在第二阶段，协调者发生故障，那么所有的参与者还都处于锁定事务资源的状态中，而无法继续完成事务操作。

- **数据不一致**

若协调者第二阶段发送提交请求时崩溃，可能部分参与者收到commit请求提交了事务，而另一部分参与者未收到commit请求而放弃事务，从而造成数据不一致的问题。

# **2.Seata是什么**

Seata 是一款开源的分布式事务解决方案，致力于提供高性能和简单易用的分布式事务服务。Seata 将为用户提供了 AT、TCC、SAGA 和 XA 事务模式，为用户打造一站式的分布式解决方案。AT模式是阿里首推的模式，阿里云上有商用版本的GTS（Global Transaction Service 全局事务服务）

官网：https://seata.io/zh-cn/index.html

源码: https://github.com/seata/seata

seata版本：v1.5.1

## **2.1 Seata的三大角色**

在 Seata 的架构中，一共有三个角色： 

- **TC (Transaction Coordinator) - 事务协调者**

维护全局和分支事务的状态，驱动全局事务提交或回滚。

- **TM (Transaction Manager) - 事务管理器**

定义全局事务的范围：开始全局事务、提交或回滚全局事务。

- **RM (Resource Manager) - 资源管理器**

管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。

其中，TC 为单独部署的 Server 服务端，TM 和 RM 为嵌入到应用中的 Client 客户端。

在 Seata 中，一个分布式事务的生命周期如下：

1. TM 请求 TC 开启一个全局事务。TC 会生成一个 XID 作为该全局事务的编号。XID会在微服务的调用链路中传播，保证将多个微服务的子事务关联在一起。
2. RM 请求 TC 将本地事务注册为全局事务的分支事务，通过全局事务的 XID 进行关联。
3. TM 请求 TC 告诉 XID 对应的全局事务是进行提交还是回滚。
4. TC 驱动 RM 们将 XID 对应的自己的本地事务进行提交还是回滚。

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54140)

## 2.2 Seata提供的四种不同的分布式事务解决方案

1. XA模式：强一致性分阶段事务模式，牺牲了一定的可用性，无业务侵入![image-20240304125403873](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/image-20240304125403873.png)

2. TCC模式：最终一致的分阶段事务模式，有业务侵入![image-20240304200118799](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/image-20240304200118799.png)![image-20240306121608027](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/image-20240306121608027.png)![image-20240305131716256](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/image-20240305131716256.png)

3. AT模式：最终一致的分阶段事务模式，无业务侵入，也是**Seata的默认模式**![image-20240304132457996](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/image-20240304132457996.png)

4. SAGA模式：长事务模式，有业务侵入

   ![image-20240306122735751](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/image-20240306122735751.png)![image-20240306124118926](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/image-20240306124118926.png)

   总结：![image-20240306123445173](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/image-20240306123445173.png)

## **2.3 Seata AT模式的设计思路**

Seata AT模式的核心是对业务无侵入，是一种改进后的两阶段提交，其设计思路如下:

- 一阶段：业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源。
- 二阶段：

- - 提交异步化，非常快速地完成。
  - 回滚通过一阶段的回滚日志进行反向补偿。

### **一阶段**

业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源。核心在于对业务sql进行解析，转换成undolog，并同时入库，这是怎么做的呢？

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54152)

### **二阶段**

- 分布式事务操作成功，则TC通知RM异步删除undolog

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54150)

- 分布式事务操作失败，TM向TC发送回滚请求，RM 收到协调器TC发来的回滚请求，通过 XID 和 Branch ID 找到相应的回滚日志记录，通过回滚记录生成反向的更新 SQL 并执行，以完成分支的回滚。

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54151)

# **3. Seata快速开始**

Seata分TC、TM和RM三个角色，TC（Server端）为单独服务端部署，TM和RM（Client端）由业务系统集成。

## **3.1 Seata Server（TC）环境搭建**

**Server端存储模式（store.mode）支持三种：**

- file：单机模式，全局事务会话信息内存中读写并持久化本地文件root.data，性能较高
- db：高可用模式，全局事务会话信息通过db共享，相应性能差些
- redis：1.3及以上版本支持,性能较高,存在事务信息丢失风险,请提前配置适合当前场景的redis持久化配置

资源目录：

- https://github.com/seata/seata/tree/v1.5.1/script

- client 

- - 存放client端sql脚本，参数配置

- config-center

- - 各个配置中心参数导入脚本，config.txt(包含server和client)为通用参数文件

- server

- - server端数据库脚本及各个容器配置

### **db存储模式+Nacos(注册&配置中心)方式部署**

**步骤一：下载安装包**

https://github.com/seata/seata/releases

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/53908)

**步骤二：建表(db模式)**

创建数据库seata，执行sql脚本，https://github.com/seata/seata/tree/v1.5.1/script/server/db

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/53914)

**步骤三：配置Nacos注册中心**

注册中心可以说是微服务架构中的”通讯录“，它记录了服务和服务地址的映射关系。在分布式架构中，服务会注册到注册中心，当服务需要调用其它服务时，就到注册中心找到服务的地址，进行调用。比如Seata Client端(TM,RM)，发现Seata Server(TC)集群的地址,彼此通信。

注意：Seata的注册中心是作用于Seata自身的，和Spring  Cloud的注册中心无关

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54137)

Seata支持哪些注册中心?

1. eureka
2. consul
3. nacos
4. etcd
5. zookeeper
6. sofa
7. redis
8. file (直连)

**配置将Seata Server注册到Nacos，修改conf/application.yml文件**

```
registry:
    # support: nacos, eureka, redis, zk, consul, etcd3, sofa
    type: nacos
    nacos:
      application: seata-server
      server-addr: 127.0.0.1:8848
      group: SEATA_GROUP
      namespace:
      cluster: default
      username:
      password:
```

注意：请确保client与server的注册处于同一个namespace和group，不然会找不到服务。

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/53940)

启动 Seata-Server 后，会发现Server端的服务出现在 Nacos 控制台中的注册中心列表中。

**步骤四：配置Nacos配置中心**

配置中心可以说是一个"大货仓",内部放置着各种配置文件,你可以通过自己所需进行获取配置加载到对应的客户端。比如Seata Client端(TM,RM),Seata Server(TC),会去读取全局事务开关,事务会话存储模式等信息。

注意：Seata的配置中心是作用于Seata自身的，和Spring  Cloud的配置中心无关

Seata支持哪些配置中心?

1. nacos
2. consul
3. apollo
4. etcd
5. zookeeper
6. file (读本地文件, 包含conf、properties、yml配置文件的支持)

**1）配置Nacos配置中心地址，修改conf/application.yml文件**

```
seata:
  config:
    # support: nacos, consul, apollo, zk, etcd3
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      namespace: 7e838c12-8554-4231-82d5-6d93573ddf32
      group: SEATA_GROUP
      data-id: seataServer.properties
     username:
     password:
```

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/53946)

**2）上传配置至Nacos配置中心**

https://github.com/seata/seata/tree/v1.5.1/script/config-center

a) 获取/seata/script/config-center/config.txt，修改为db存储模式，并修改mysql连接配置

```
store.mode=db
store.lock.mode=db
store.session.mode=db
store.db.driverClassName=com.mysql.jdbc.Driver
store.db.url=jdbc:mysql://127.0.0.1:3306/seata?useUnicode=true&rewriteBatchedStatements=true
store.db.user=root
store.db.password=root
```

在store.mode=db，由于seata是通过jdbc的executeBatch来批量插入全局锁的，根据MySQL官网的说明，连接参数中的rewriteBatchedStatements为true时，在执行executeBatch，并且操作类型为insert时，jdbc驱动会把对应的SQL优化成`insert into () values (), ()`的形式来提升批量插入的性能。 根据实际的测试，该参数设置为true后，对应的批量插入性能为原来的10倍多，因此在数据源为MySQL时，建议把该参数设置为true。

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/53887)

b) 配置事务分组， 要与client配置的事务分组一致

- 事务分组：seata的资源逻辑，可以按微服务的需要，在应用程序（客户端）对自行定义事务分组，每组取一个名字。
- 集群：seata-server服务端一个或多个节点组成的集群cluster。 应用程序（客户端）使用时需要指定事务逻辑分组与Seata服务端集群的映射关系。

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/53977)

事务分组如何找到后端Seata集群（TC）？

1. 首先应用程序（客户端）中配置了事务分组（GlobalTransactionScanner 构造方法的txServiceGroup参数）。若应用程序是SpringBoot则通过seata.tx-service-group 配置。
2. 应用程序（客户端）会通过用户配置的配置中心去寻找service.vgroupMapping .[事务分组配置项]，取得配置项的值就是TC集群的名称。若应用程序是SpringBoot则通过seata.service.vgroup-mapping.事务分组名=集群名称 配置
3. 拿到集群名称程序通过一定的前后缀+集群名称去构造服务名，各配置中心的服务名实现不同（前提是Seata-Server已经完成服务注册，且Seata-Server向注册中心报告cluster名与应用程序（客户端）配置的集群名称一致）
4. 拿到服务名去相应的注册中心去拉取相应服务名的服务列表，获得后端真实的TC服务列表（即Seata-Server集群节点列表）

c) 在nacos配置中心中新建配置，dataId为seataServer.properties，配置内容为上面修改后的config.txt中的配置信息

从v1.4.2版本开始，seata已支持从一个Nacos dataId中获取所有配置信息,你只需要额外添加一个dataId配置项。

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54003)

添加后查看：

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54005)

**步骤五：启动Seata Server**

启动命令:

​                bin/seata-server.sh              

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54044)

启动成功，查看控制台，账号密码都是seata。http://localhost:7091/#/login

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54055)

在Nacos注册中心中可以查看到seata-server注册成功

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54053)

支持的启动参数

| 参数 | 全写         | 作用                       | 备注                                                         |
| ---- | ------------ | -------------------------- | ------------------------------------------------------------ |
| -h   | --host       | 指定在注册中心注册的 IP    | 不指定时获取当前的 IP，外部访问部署在云环境和容器中的 server 建议指定 |
| -p   | --port       | 指定 server 启动的端口     | 默认为 8091                                                  |
| -m   | --storeMode  | 事务日志存储方式           | 支持file,db,redis，默认为 file 注:redis需seata-server 1.3版本及以上 |
| -n   | --serverNode | 用于指定seata-server节点ID | 如 1,2,3..., 默认为 1                                        |
| -e   | --seataEnv   | 指定 seata-server 运行环境 | 如 dev, test 等, 服务启动时会使用 registry-dev.conf 这样的配置 |

比如：

```
bin/seata-server.sh -p 8091 -h 127.0.0.1 -m db        
```

## **3.2 Seata Client快速开始**

### **Spring Cloud Alibaba整合Seata AT模式实战**

**业务场景**

用户下单，整个业务逻辑由三个微服务构成：

- 库存服务：对给定的商品扣除库存数量。
- 订单服务：根据采购需求创建订单。
- 帐户服务：从用户帐户中扣除余额。

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54076)

**1) 环境准备**

- 父pom指定微服务版本

| Spring Cloud Alibaba Version | Spring Cloud Version     | Spring Boot Version | Seata Version |
| ---------------------------- | ------------------------ | ------------------- | ------------- |
| 2.2.8.RELEASE                | Spring Cloud Hoxton.SR12 | 2.3.12.RELEASE      | 1.5.1         |

- 启动Seata Server(TC)端，Seata Server使用nacos作为配置中心和注册中心
- 启动nacos服务

**2)  微服务导入seata依赖**

```
<!-- seata-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

**3)微服务对应数据库中添加undo_log表(仅AT模式)**

https://github.com/seata/seata/blob/v1.5.1/script/client/at/db/mysql.sql

```
-- for AT mode you must to init this sql for you business database. the seata server not need it.
CREATE TABLE IF NOT EXISTS `undo_log`
(
    `branch_id`     BIGINT       NOT NULL COMMENT 'branch transaction id',
    `xid`           VARCHAR(128) NOT NULL COMMENT 'global transaction id',
    `context`       VARCHAR(128) NOT NULL COMMENT 'undo_log context,such as serialization',
    `rollback_info` LONGBLOB     NOT NULL COMMENT 'rollback info',
    `log_status`    INT(11)      NOT NULL COMMENT '0:normal status,1:defense status',
    `log_created`   DATETIME(6)  NOT NULL COMMENT 'create datetime',
    `log_modified`  DATETIME(6)  NOT NULL COMMENT 'modify datetime',
    UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8mb4 COMMENT ='AT transaction mode undo table';
```

​         

**4) 微服务application.yml中添加seata配置**

```
seata:
  application-id: ${spring.application.name}
  # seata 服务分组，要与服务端配置service.vgroup_mapping的后缀对应
  tx-service-group: default_tx_group
  registry:
    # 指定nacos作为注册中心
    type: nacos
    nacos:
      application: seata-server
      server-addr: 127.0.0.1:8848
      namespace:
      group: SEATA_GROUP

  config:
    # 指定nacos作为配置中心
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      namespace: 7e838c12-8554-4231-82d5-6d93573ddf32
      group: SEATA_GROUP
      data-id: seataServer.properties
```

[]()           

注意：请确保client与server的注册中心和配置中心namespace和group一致

**5） 在全局事务发起者中添加@GlobalTransactional注解**

核心代码

```
@Override
@GlobalTransactional(name="createOrder",rollbackFor=Exception.class)
public Order saveOrder(OrderVo orderVo){
    log.info("=============用户下单=================");
    log.info("当前 XID: {}", RootContext.getXID());
    
    // 保存订单
    Order order = new Order();
    order.setUserId(orderVo.getUserId());
    order.setCommodityCode(orderVo.getCommodityCode());
    order.setCount(orderVo.getCount());
    order.setMoney(orderVo.getMoney());
    order.setStatus(OrderStatus.INIT.getValue());

    Integer saveOrderRecord = orderMapper.insert(order);
    log.info("保存订单{}", saveOrderRecord > 0 ? "成功" : "失败");
    
    //扣减库存
    storageFeignService.deduct(orderVo.getCommodityCode(),orderVo.getCount());
    
    //扣减余额
    accountFeignService.debit(orderVo.getUserId(),orderVo.getMoney());

    //更新订单
    Integer updateOrderRecord = orderMapper.updateOrderStatus(order.getId(),OrderStatus.SUCCESS.getValue());
    log.info("更新订单id:{} {}", order.getId(), updateOrderRecord > 0 ? "成功" : "失败");
    
    return order;
    
}
```

**6）测试分布式事务是否生效**

- 分布式事务成功，模拟正常下单、扣库存，扣余额
- 分布式事务失败，模拟下单扣库存成功、扣余额失败，事务是否回滚

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54068)

# **4. Seata XA模式**

## **4.1 整体机制**

在 Seata 定义的分布式事务框架内，利用事务资源（数据库、消息服务等）对 XA 协议的支持，以 XA 协议的机制来管理分支事务的一种 事务模式。

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/55865)

**AT和XA模式数据源代理机制对比**

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/55847)

**XA 模式的使用**

从编程模型上，XA 模式与 AT 模式保持完全一致。只需要修改数据源代理，即可实现 XA 模式与 AT 模式之间的切换。

```
@Bean("dataSource")
public DataSource dataSource(DruidDataSource druidDataSource) {
    // DataSourceProxy for AT mode
    // return new DataSourceProxy(druidDataSource);

    // DataSourceProxyXA for XA mode
    return new DataSourceProxyXA(druidDataSource);
}
```

## **4.2 Spring Cloud Alibaba整合Seata XA实战**

对比Seata AT模式配置，只需修改两个地方：

- 微服务数据库不需要undo_log表，undo_log表仅用于AT模式
- 修改数据源代码模式为XA模式

```
seata:
  # 数据源代理模式 默认AT
  data-source-proxy-mode: XA
```

# **5. 什么是TCC**

TCC 基于分布式事务中的二阶段提交协议实现，它的全称为 Try-Confirm-Cancel，即资源预留（Try）、确认操作（Confirm）、取消操作（Cancel），他们的具体含义如下：

1. Try：对业务资源的检查并预留；
2. Confirm：对业务处理进行提交，即 commit 操作，只要 Try 成功，那么该步骤一定成功；
3. Cancel：对业务处理进行取消，即回滚操作，该步骤回对 Try 预留的资源进行释放。

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/54287)

- **XA是资源层面的分布式事务，强一致性，在两阶段提交的整个过程中，一直会持有资源的锁。**
- **TCC是业务层面的分布式事务，最终一致性，不会一直持有资源的锁。**

TCC 是一种侵入式的分布式事务解决方案，以上三个操作都需要业务系统自行实现，对业务系统有着非常大的入侵性，设计相对复杂，但优点是 TCC 完全不依赖数据库，能够实现跨数据库、跨应用资源管理，对这些不同数据访问通过侵入式的编码方式实现一个原子操作，更好地解决了在各种复杂业务场景下的分布式事务问题。

**以用户下单为例**

**try-commit**

try 阶段首先进行预留资源，然后在 commit 阶段扣除资源。如下图：

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/55787)

**try-cancel**

try 阶段首先进行预留资源，预留资源时扣减库存失败导致全局事务回滚，在 cancel 阶段释放资源。如下图：

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/55786)

## **5.1 Seata TCC 模式**

一个分布式的全局事务，整体是 两阶段提交 的模型。全局事务是由若干分支事务组成的，分支事务要满足 两阶段提交 的模型要求，即需要每个分支事务都具备自己的：

- 一阶段 prepare 行为
- 二阶段 commit 或 rollback 行为

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/55681)

在Seata中，AT模式与TCC模式事实上都是两阶段提交的具体实现，他们的区别在于：

AT 模式基于 支持本地 ACID 事务的关系型数据库：

- 一阶段 prepare 行为：在本地事务中，一并提交业务数据更新和相应回滚日志记录。
- 二阶段 commit 行为：马上成功结束，自动异步批量清理回滚日志。
- 二阶段 rollback 行为：通过回滚日志，自动生成补偿操作，完成数据回滚。

相应的，TCC 模式不依赖于底层数据资源的事务支持：

- 一阶段 prepare 行为：调用自定义的 prepare 逻辑。
- 二阶段 commit 行为：调用自定义的 commit 逻辑。
- 二阶段 rollback 行为：调用自定义的 rollback 逻辑。

简单点概括，SEATA的TCC模式就是手工的AT模式，它允许你自定义两阶段的处理逻辑而不依赖AT模式的undo_log。

## **5.2 Seata TCC模式接口如何改造**

假设现有一个业务需要同时使用服务 A 和服务 B 完成一个事务操作，我们在服务 A 定义该服务的一个 TCC 接口：

```
public interface TccActionOne {
    @TwoPhaseBusinessAction(name = "prepare", commitMethod = "commit", rollbackMethod = "rollback")
    public boolean prepare(BusinessActionContext actionContext, @BusinessActionContextParameter(paramName = "a") String a);

    public boolean commit(BusinessActionContext actionContext);

    public boolean rollback(BusinessActionContext actionContext);
}
```

同样，在服务 B 定义该服务的一个 TCC 接口：

```
public interface TccActionTwo {
    @TwoPhaseBusinessAction(name = "prepare", commitMethod = "commit", rollbackMethod = "rollback")
    public void prepare(BusinessActionContext actionContext, @BusinessActionContextParameter(paramName = "b") String b);

    public void commit(BusinessActionContext actionContext);

    public void rollback(BusinessActionContext actionContext);
}
```

在业务所在系统中开启全局事务并执行服务 A 和服务 B 的 TCC 预留资源方法：

```
@GlobalTransactional
public String doTransactionCommit(){
    //服务A事务参与者
    tccActionOne.prepare(null,"one");
    //服务B事务参与者
    tccActionTwo.prepare(null,"two");
}
```

以上就是使用 Seata TCC 模式实现一个全局事务的例子，TCC 模式同样使用 @GlobalTransactional 注解开启全局事务，而服务 A 和服务 B 的 TCC 接口为事务参与者，Seata 会把一个 TCC 接口当成一个 Resource，也叫 TCC Resource。

## **5.3 TCC如何控制异常**

在 TCC 模型执行的过程中，还可能会出现各种异常，其中最为常见的有空回滚、幂等、悬挂等。TCC 模式是分布式事务中非常重要的事务模式，但是幂等、悬挂和空回滚一直是 TCC 模式需要考虑的问题，Seata 框架在 1.5.1 版本完美解决了这些问题。

### **如何处理空回滚**

空回滚指的是在一个分布式事务中，在没有调用参与方的 Try 方法的情况下，TM 驱动二阶段回滚调用了参与方的 Cancel 方法。

**那么空回滚是如何产生的呢？**

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/55759)

如上图所示，全局事务开启后，参与者 A 分支注册完成之后会执行参与者一阶段 RPC 方法，如果此时参与者 A 所在的机器发生宕机，网络异常，都会造成 RPC 调用失败，即参与者 A 一阶段方法未成功执行，但是此时全局事务已经开启，Seata 必须要推进到终态，在全局事务回滚时会调用参与者 A 的 Cancel 方法，从而造成空回滚。

**要想防止空回滚，那么必须在 Cancel 方法中识别这是一个空回滚，Seata 是如何做的呢？**

Seata 的做法是新增一个 TCC 事务控制表，包含事务的 XID 和 BranchID 信息，在 Try 方法执行时插入一条记录，表示一阶段执行了，执行 Cancel 方法时读取这条记录，如果记录不存在，说明 Try 方法没有执行。

### **如何处理幂等**

幂等问题指的是 TC 重复进行二阶段提交，因此 Confirm/Cancel 接口需要支持幂等处理，即不会产生资源重复提交或者重复释放。

**那么幂等问题是如何产生的呢？**

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/55758)

如上图所示，参与者 A 执行完二阶段之后，由于网络抖动或者宕机问题，会造成 TC 收不到参与者 A 执行二阶段的返回结果，TC 会重复发起调用，直到二阶段执行结果成功。

**Seata 是如何处理幂等问题的呢？**

同样的也是在 TCC 事务控制表中增加一个记录状态的字段 status，该字段有 3 个值，分别为：

1. tried：1
2. committed：2
3. rollbacked：3

二阶段 Confirm/Cancel 方法执行后，将状态改为 committed 或 rollbacked 状态。当重复调用二阶段 Confirm/Cancel 方法时，判断事务状态即可解决幂等问题。

### **如何处理悬挂**

悬挂指的是二阶段 Cancel 方法比 一阶段 Try 方法优先执行，由于允许空回滚的原因，在执行完二阶段 Cancel 方法之后直接空回滚返回成功，此时全局事务已结束，但是由于 Try 方法随后执行，这就会造成一阶段 Try 方法预留的资源永远无法提交和释放了。

**那么悬挂是如何产生的呢？**

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/55757)

如上图所示，在执行参与者 A 的一阶段 Try 方法时，出现网路拥堵，由于 Seata 全局事务有超时限制，执行 Try 方法超时后，TM 决议全局回滚，回滚完成后如果此时 RPC 请求才到达参与者 A，执行 Try 方法进行资源预留，从而造成悬挂。

**Seata 是怎么处理悬挂的呢？**

在 TCC 事务控制表记录状态的字段 status 中增加一个状态：

- suspended：4

当执行二阶段 Cancel 方法时，如果发现 TCC 事务控制表有相关记录，说明二阶段 Cancel 方法优先一阶段 Try 方法执行，因此插入一条 status=4 状态的记录，当一阶段 Try 方法后面执行时，判断 status=4 ，则说明有二阶段 Cancel 已执行，并返回 false 以阻止一阶段 Try 方法执行成功。

## **5.4 Spring Cloud Alibaba整合Seata TCC实战**

**业务场景**

用户下单，整个业务逻辑由三个微服务构成：

- 库存服务：对给定的商品扣除库存数量。
- 订单服务：根据采购需求创建订单。
- 帐户服务：从用户帐户中扣除余额。

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/55687)

**1) 环境准备**

- 父pom指定微服务版本

| Spring Cloud Alibaba Version | Spring Cloud Version     | Spring Boot Version | Seata Version |
| ---------------------------- | ------------------------ | ------------------- | ------------- |
| 2.2.8.RELEASE                | Spring Cloud Hoxton.SR12 | 2.3.12.RELEASE      | 1.5.1         |

- 启动Seata Server(TC)端，Seata Server使用nacos作为配置中心和注册中心
- 启动nacos服务

**2)  微服务导入seata依赖**

spring-cloud-starter-alibaba-seata内部集成了seata，并实现了xid传递

```
<!-- seata-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

 **3）微服务application.yml中添加seata配置**

```
seata:
  application-id: ${spring.application.name}
  # seata 服务分组，要与服务端配置service.vgroup_mapping的后缀对应
  tx-service-group: default_tx_group
  registry:
    # 指定nacos作为注册中心
    type: nacos
    nacos:
      application: seata-server
      server-addr: 127.0.0.1:8848
      namespace:
      group: SEATA_GROUP

  config:
    # 指定nacos作为配置中心
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      namespace: 7e838c12-8554-4231-82d5-6d93573ddf32
      group: SEATA_GROUP
      data-id: seataServer.properties
```

注意：请确保client与server的注册中心和配置中心namespace和group一致

**4）定义TCC接口**

TCC相关注解如下：

- @LocalTCC 适用于SpringCloud+Feign模式下的TCC，@LocalTCC一定需要注解在接口上，此接口可以是寻常的业务接口，只要实现了TCC的两阶段提交对应方法便可
- @TwoPhaseBusinessAction 注解try方法，其中name为当前tcc方法的bean名称，写方法名便可（全局唯一），commitMethod指向提交方法，rollbackMethod指向事务回滚方法。指定好三个方法之后，seata会根据全局事务的成功或失败，去帮我们自动调用提交方法或者回滚方法。
- @BusinessActionContextParameter 注解可以将参数传递到二阶段（commitMethod/rollbackMethod）的方法。
- BusinessActionContext 便是指TCC事务上下文

```
/**
 * 通过 @LocalTCC 这个注解，RM 初始化的时候会向 TC 注册一个分支事务。
 */
@LocalTCC
public interface OrderService {

    /**
     * TCC的try方法：保存订单信息，状态为支付中
     *
     * 定义两阶段提交，在try阶段通过@TwoPhaseBusinessAction注解定义了分支事务的 resourceId，commit和 cancel 方法
     *  name = 该tcc的bean名称,全局唯一
     *  commitMethod = commit 为二阶段确认方法
     *  rollbackMethod = rollback 为二阶段取消方法
     *  BusinessActionContextParameter注解 传递参数到二阶段中
     *  useTCCFence seata1.5.1的新特性，用于解决TCC幂等，悬挂，空回滚问题，需增加日志表tcc_fence_log
     */
    @TwoPhaseBusinessAction(name = "prepareSaveOrder", commitMethod = "commit", rollbackMethod = "rollback", useTCCFence = true)
    Order prepareSaveOrder(OrderVo orderVo, @BusinessActionContextParameter(paramName = "orderId") Long orderId);

    /**
     *
     * TCC的confirm方法：订单状态改为支付成功
     *
     * 二阶段确认方法可以另命名，但要保证与commitMethod一致
     * context可以传递try方法的参数
     *
     * @param actionContext
     * @return
     */
    boolean commit(BusinessActionContext actionContext);

    /**
     * TCC的cancel方法：订单状态改为支付失败
     * 二阶段取消方法可以另命名，但要保证与rollbackMethod一致
     *
     * @param actionContext
     * @return
     */
    boolean rollback(BusinessActionContext actionContext);
}

/**
 * 通过 @LocalTCC 这个注解，RM 初始化的时候会向 TC 注册一个分支事务。
 */
@LocalTCC
public interface StorageService {

    /**
     * Try: 库存-扣减数量，冻结库存+扣减数量
     *
     * 定义两阶段提交，在try阶段通过@TwoPhaseBusinessAction注解定义了分支事务的 resourceId，commit和 cancel 方法
     *  name = 该tcc的bean名称,全局唯一
     *  commitMethod = commit 为二阶段确认方法
     *  rollbackMethod = rollback 为二阶段取消方法
     *  BusinessActionContextParameter注解 传递参数到二阶段中
     *
     * @param commodityCode 商品编号
     * @param count 扣减数量
     * @return
     */
    @TwoPhaseBusinessAction(name = "deduct", commitMethod = "commit", rollbackMethod = "rollback", useTCCFence = true)
    boolean deduct(@BusinessActionContextParameter(paramName = "commodityCode") String commodityCode,
                   @BusinessActionContextParameter(paramName = "count") int count);

    /**
     *
     * Confirm: 冻结库存-扣减数量
     * 二阶段确认方法可以另命名，但要保证与commitMethod一致
     * context可以传递try方法的参数
     *
     * @param actionContext
     * @return
     */
    boolean commit(BusinessActionContext actionContext);

    /**
     * Cancel: 库存+扣减数量，冻结库存-扣减数量
     * 二阶段取消方法可以另命名，但要保证与rollbackMethod一致
     *
     * @param actionContext
     * @return
     */
    boolean rollback(BusinessActionContext actionContext);
}

/**
 * 通过 @LocalTCC 这个注解，RM 初始化的时候会向 TC 注册一个分支事务。
 */
@LocalTCC
public interface AccountService {

    /**
     * 用户账户扣款
     *
     * 定义两阶段提交，在try阶段通过@TwoPhaseBusinessAction注解定义了分支事务的 resourceId，commit和 cancel 方法
     *  name = 该tcc的bean名称,全局唯一
     *  commitMethod = commit 为二阶段确认方法
     *  rollbackMethod = rollback 为二阶段取消方法
     *
     * @param userId
     * @param money 从用户账户中扣除的金额
     * @return
     */
    @TwoPhaseBusinessAction(name = "debit", commitMethod = "commit", rollbackMethod = "rollback", useTCCFence = true)
    boolean debit(@BusinessActionContextParameter(paramName = "userId") String userId,
                  @BusinessActionContextParameter(paramName = "money") int money);

    /**
     * 提交事务，二阶段确认方法可以另命名，但要保证与commitMethod一致
     * context可以传递try方法的参数
     *
     * @param actionContext
     * @return
     */
    boolean commit(BusinessActionContext actionContext);

    /**
     * 回滚事务，二阶段取消方法可以另命名，但要保证与rollbackMethod一致
     *
     * @param actionContext
     * @return
     */
    boolean rollback(BusinessActionContext actionContext);
}
```

​         

TCC 幂等、悬挂和空回滚问题如何解决？

TCC 模式中存在的三大问题是幂等、悬挂和空回滚。在 Seata1.5.1 版本中，增加了一张事务控制表，表名是 tcc_fence_log 来解决这个问题。而在@TwoPhaseBusinessAction 注解中提到的属性 useTCCFence 就是来指定是否开启这个机制，这个属性值默认是 false。

**5）微服务增加tcc_fence_log日志表**

```
# tcc_fence_log 建表语句如下（MySQL 语法）
CREATE TABLE IF NOT EXISTS `tcc_fence_log`
(
    `xid`           VARCHAR(128)  NOT NULL COMMENT 'global id',
    `branch_id`     BIGINT        NOT NULL COMMENT 'branch id',
    `action_name`   VARCHAR(64)   NOT NULL COMMENT 'action name',
    `status`        TINYINT       NOT NULL COMMENT 'status(tried:1;committed:2;rollbacked:3;suspended:4)',
    `gmt_create`    DATETIME(3)   NOT NULL COMMENT 'create time',
    `gmt_modified`  DATETIME(3)   NOT NULL COMMENT 'update time',
    PRIMARY KEY (`xid`, `branch_id`),
    KEY `idx_gmt_modified` (`gmt_modified`),
    KEY `idx_status` (`status`)
) ENGINE = InnoDB
DEFAULT CHARSET = utf8mb4;
```

**6）TCC接口的业务实现**

参考课堂代码

**7） 在全局事务发起者中添加@GlobalTransactional注解**

核心代码

```
@GlobalTransactional(name="createOrder",rollbackFor=Exception.class)
public Order saveOrder(OrderVo orderVo) {
    log.info("=============用户下单=================");
    log.info("当前 XID: {}", RootContext.getXID());

    //获取全局唯一订单号  测试使用
    Long orderId = UUIDGenerator.generateUUID();

    //阶段一： 创建订单
    Order order = orderService.prepareSaveOrder(orderVo,orderId);

    //扣减库存
    storageFeignService.deduct(orderVo.getCommodityCode(), orderVo.getCount());
    //扣减余额
    accountFeignService.debit(orderVo.getUserId(), orderVo.getMoney());

    return order;
}
```

**8）测试分布式事务是否生效**

- 分布式事务成功，模拟正常下单、扣库存，扣余额
- 分布式事务失败，模拟下单扣库存成功、扣余额失败，事务是否回滚

​    ![0](seata%E5%88%9D%E8%AF%86%E4%B8%8E%E5%AE%9E%E6%88%98/55750)
