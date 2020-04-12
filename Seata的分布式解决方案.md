# **基于Seata的分布式事务解决方案**

## 前言

​         上一篇文章已经带大家初步了解了分布式事务的基本解决方案，今天就向大家介绍下分布式事务中间件Seata，以及自己写的一个基于seata的分布式事务的demo

## 正文

### **背景**

​        2019 年 1 月，阿里巴巴中间件团队发起了开源项目 [*Fescar*](https://www.oschina.net/p/fescar)*（Fast & EaSy Commit And Rollback）*，和社区一起共建开源分布式事务解决方案。Fescar 的愿景是让分布式事务的使用像本地事务的使用一样，简单和高效，并逐步解决开发者们遇到的分布式事务方面的所有难题。Fescar 开源后，蚂蚁金服加入 Fescar 社区参与共建，并在 Fescar 0.4.0 版本中贡献了 TCC 模式。为了打造更中立、更开放、生态更加丰富的分布式事务开源社区，经过社区核心成员的投票，大家决定对 Fescar 进行品牌升级，并更名为 Seata，意为：Simple Extensible Autonomous Transaction Architecture，是一套一站式分布式事务解决方案。

### Seata是什么？

Seata是一款开源的分布式解决方案，致力于提供高性能和简单易用的分布式事务服务。它设计的初衷是：

- **对业务无侵入：** 这里的 侵入 是指，因为分布式事务这个技术问题的制约，要求应用在业务层面进行设计和改造。这种设计和改造往往会给应用带来很高的研发和维护成本。把分布式事务问题在中间件这个层次解决掉，不要求应用在业务层面做额外的工作。
- **高性能：** 引入分布式事务的保障，必然会有额外的开销，引起性能的下降。把分布式事务引入的性能损耗降到非常低的水平，让应用不因为分布式事务的引入导致业务的可用性受影响

Seata为用户提供了AT模式、TCC模式、Soga模式以及XA事务模式，为用户打造一站式的分布式事务解决方案。

我们先来了解一下Seata的重要的组件：

- **Transcation Coordinator(TC)**：事务协调器， 维护全局事务的事务状态，负责协调并驱动全局事务的提交或回滚  即Seata-server
- **Transaction Manager(TM)**:  维护全局事务的边界，负责开启一个全局事务，并负责发起全局事务的提交或者回滚的决议   即Seata-Client,事务的发起者，整个微服务调用链的起点
- **Resource Manger(RM)**:  控制分支事务，负责注册分支事务，分支事务状态的汇报，并接受事务协调器的指令，驱动本地事务的提交或者回滚  即Seata-Client, 事务的参与者，负责本地事务的处理以及与Seata-server的交互

一个典型的分布式事务过程：

1. TM 向TC发起创建一个全局事务，TC创建全局事务成功并生成一个全局唯一的XID
2. XID在整个服务调用上下文中传递
3. RM向TC注册分支事务，并入对应XID的全局事务的管辖下
4. TM 向TC 发起对应XID全局事务的提交或回滚决议
5. TC 调度 XID 下管辖的全部分支事务完成提交或回滚请求。

![image-20200412143244576](https://github.com/GBBP1813/notes/blob/master/picture/Seatas%E4%BA%8B%E5%8A%A1%E8%BF%87%E7%A8%8B.jpg)

接下来介绍下Seata各个事务模式

### AT模式

AT模式的使用的提前是：一是基于支持ACID的关系数据库，而是Java应用，通过JDBC访问数据库。AT模式其实是对基于XA解决方案的改进，将资源管理器从数据库层面抽离出来，而已二方包的形式作为中间件层部署在应用的一侧。不依赖与数据库本身对协议的支持，当然也不需要数据库支持 XA 协议。

它的基本步骤：

- **一阶段：**
  1. 解析业务sql
  2. 获取sql执行前的镜像，前镜像
  3. 执行业务sql
  4. 获取sql执行后的镜像，后镜像
  5. 添加undo_log日志，把前后镜像数据和业务sql相关的信息组成回滚日志，添加到undo_log表中
  6. 向TC注册分支事务，并申请全局锁
  7. 事务提交，将业务操作和undo_log一起提交
  8. 释放本地锁与数据库连接资源
- **二阶段-提交：**
  1. 收到TC发出的提交分支事务的请求，将请求放进一个异步任务队列中，马上返回成功给TC
  2. 异步任务阶段的分支提交请求将异步和批量地删除相应 UNDO LOG 记录

- **二阶段-回滚**：
  1. 收到TC发出的回滚请求，开启一个事务
  2. 根据XID和Branch ID 查找对应的UNDO_LOG记录
  3. 数据校验：拿 UNDO LOG 中的后镜与当前数据进行比较，如果有不同，说明数据被当前全局事务之外的动作做了修改。这种情况，需要根据配置策略来做处理
  4. 根据 UNDO LOG 中的前镜像和业务 SQL 的相关信息生成并执行回滚的语句
  5. 提交本地事务

**这时有的同学可能会问为啥它能在第一阶段就能释放锁呢**

Seata的JDBC 数据源代理通过对业务 SQL 的解析，把业务数据在更新前后的数据镜像组织成回滚日志，利用本地事务的 ACID 特性，将业务数据的更新和回滚日志的写入在同一个本地事务中提交。这样就可以保证：任何提交的业务数据的更新一定有相应的回滚日志存在。基于这样的设计，就可以在第一阶段释放本地锁

**那么AT模式如何保证写隔离的呢**

1. 本地事务在commit之前都会去申请全局锁，需要确保拿到全局锁才能commit
2. 拿不到全局锁便不会commit
3. 申请全局锁有限制范围，超出了这个范围，就会回滚本地事务并释放本地锁

**如何保证读隔离**

Seata的AT模式默认的全局隔离级别是读未提交。如果应用在特定场景下，必需要求全局的读已提交 ，目前Seata 的方式是通过 SELECT FOR UPDATE 语句的代理。SELECT FOR UPDATE 语句的执行会申请全局锁，如果 全局锁 被其他事务持有，则释放本地锁（回滚 SELECT FOR UPDATE 语句的本地执行）并重试。这个过程中，查询是被 block 住的，直到全局锁拿到，即读取的相关数据是已提交 的，才返回。出于总体性能上的考虑，Seata目前的方案并没有对所有SELECT语句都进行代理，仅针对 FOR UPDATE 的 SELECT 语句。

**AT模式优缺点**

**优点：**

- 不需要数据库对XA协议的支持
- 并发和吞吐量提升，本地锁在第一阶段就可以释放

**缺点**

- 只能用在支持ACID的关系型数据库
- 事务隔离级别最高支持到读已提交的水平，SQL 的解析还不能涵盖全部的语法等。



### **TCC模式**

TCC模式 可以把自定义的分支事务纳入到全局事务之中，不依赖于底层数据资源对事务的依赖

它的处理过程是：

- 一阶段 prepare 行为：调用自定义的 prepare 逻辑。
- 二阶段 commit 行为：调用自定义的 commit 逻辑。
- 二阶段 rollback 行为：调用自定义的 rollback 逻辑。

![image-20200412160556090](https://github.com/GBBP1813/notes/blob/master/picture/seataTcc.jpg)

```java
@LocalTCC
public interface TccAcountAction {

    @TwoPhaseBusinessAction(name = "DubboTccActionTwo" , commitMethod = "commit", rollbackMethod = "rollback")
    public boolean prepare(BusinessActionContext actionContext,
                           @BusinessActionContextParameter(paramName = "b") String b,
                           @BusinessActionContextParameter(paramName = "c", index = 0) List list);

    public boolean commit(BusinessActionContext actionContext);


    public boolean rollback(BusinessActionContext actionContext);
}
```

整个事务的过程：

- TM 向TC发起创建一个全局事务，TC创建全局事务成功并生成一个全局唯一的XID

- 依次执行分支事务

- - 向TC注册分支事务
  - 执行事务的prepare阶段 即调用prepare方法

- 提交或者回滚全局事务，TC向每个RM发起commit或者rollbac，即调用写好的commit或者rollback方法



**TCC模式的优缺点**

**优点：**

- 可以把众多非事务的资源纳入纳入全局事务的管理，如缓存

**缺点**：

- 要自己实现补偿逻辑



**Soga模式**

Saga模式是SEATA提供的长事务解决方案，在Saga模式中，业务流程中每个参与者都提交本地事务，当出现某一个参与者失败则补偿前面已经成功的参与者，一阶段正向服务和二阶段补偿服务都由业务开发实现。

![image-20200412162608605](/Users/guanyangfna/Library/Application Support/typora-user-images/image-20200412162608605.png)

### Saga的实现：

目前SEATA提供的Saga模式是基于状态机引擎来实现的，机制是：

1. 通过状态图来定义服务调用的流程并生成 json 状态语言定义文件
2. 状态图中一个节点可以是调用一个服务，节点可以配置它的补偿节点
3. 状态图 json 由状态机引擎驱动执行，当出现异常时状态引擎反向执行已成功节点对应的补偿节点将事务回滚

​      注意: 异常发生时是否进行补偿也可由用户自定义决定

4. 可以实现服务编排需求，支持单项选择、并发、子流程、参数转换、参数映射、服务执行状态判断、异常捕获等功能

### 适用场景：

- 业务流程长、业务流程多
- 参与者包含其它公司或遗留系统服务，无法提供 TCC 模式要求的三个接口

### 优势：

- 一阶段提交本地事务，无锁，高性能
- 事件驱动架构，参与者可异步执行，高吞吐
- 补偿服务易于实现

### 缺点：

- 不保证隔离性



小编自己也写了一个基于Seata的AT模式/TCC模式的demo，github地址：https://github.com/GBBP1813/seata-example



**基于分布式事务中间件seata的分布式解决方案**

本Demo工程基于seata/seata-sample实现AT/MT模式

涉及技术栈：

springboot   dubbo zookeeper

**使用本案例demo步骤**

**1.准备项目**

 克隆此项目到本地，导入idea，JDK为1.8  创建业务数据库 sql在项目中 包括undo表

模块：

Seata-order  订单服务模块

Seata-storage 库存服务模块

Seata-account 账户服务模块

Seata-common 公共模块

Seata-transcation-manage TM模块

**2.环境准备**

1）先下载seata-server  地址：https://github.com/seata/seata/releases 下载完后修其 数据库配置

由于demo使用的是db模式存储事务日志，要创建三张表：global_table，branch_table，lock_table，建表sql在上面下载的seata-server的/conf/db_store.sql中； 配置中国事务组也可以自己修改，修改的话项目中也要对应变更

2）启动本地zookeeper

3）启动seata-server    

**3.启动项目**

分别启动项目四个子模块

**4.模拟请求**

AT模式

Commit：

```java
curl -H "Content-Type:application/json" -X POST -d '{"userId":"1","commodityCode":"C201901140001","name":"电脑","count":1,"amount":"10"}' 'localhost:8004/tmManager/buy'
```

Rollback：

```java
curl -H "Content-Type:application/json" -X POST -d '{"userId":"1","commodityCode":"C201901140001","name":"电脑","count":1,"amount":"10"}' 'localhost:8004/tmManager/buy?throwExp=true'
```

TCC模式：

Commit:

```java
curl -H "Content-Type:application/json" -X POST -d ''  'localhost:8004/tcc/bug'
```

Rollback:

```java
curl -H "Content-Type:application/json" -X POST -d ''  'localhost:8104/tcc/bug?throwExp=true'

```
