# 第 27 章：数字钱包

## 引言
**支付平台**通常都有一个**钱包服务**，允许客户在应用中存放资金，之后可以再提取出来。

你也可以用它来支付商品与服务，或者转账给其他使用**数字钱包**服务的用户。相比通过常规支付通道，这种方式可能更快、更便宜。

<div style="margin-left:3rem">
    <img src="./images/digital-wallet.png" alt="digital-wallet" width="500" />
</div>

---

## 第 1 步：理解问题并确定设计范围
 * C：我们是否需要只关注数字钱包之间的转账？还需要支持其他操作吗？
 * I：目前先聚焦于数字钱包之间的转账。
 * C：系统需要支持每秒多少笔交易？
 * I：假设是 100万 TPS
 * C：数字钱包有严格的正确性要求。我们是否可以认为事务性保证就足够了？
 * I：可以
 * C：我们需要证明正确性吗？
 * I：我们可以通过对账来做到这一点，但那只能够发现差异，而无法告诉我们差异的根因。我们希望能从头重放数据以重建历史。
 * C：我们可以假设可用性要求是 99.99% 吗？
 * I：可以
 * C：我们需要考虑外汇兑换吗？
 * I：不需要，这不在范围内

总结一下，我们需要支持的内容：
 * 支持两个账户之间的余额转账
 * 支持 100万 TPS
 * 可靠性为 99.99%
 * 支持事务
 * 支持可重现性

### **粗略估算**
一个部署在云上的传统关系型数据库大约能支持 1000 TPS。

要达到 100万 TPS，我们需要 1000 个数据库节点。但如果每笔转账有两个分支，那么我们实际上需要支持 200万 TPS。

我们的设计目标之一就是提高单个节点能处理的 TPS，从而减少数据库节点的数量。

| 单节点 TPS | 节点数量 |
|--------------|-------------|
| 100          | 20,000      |
| 1,000        | 2,000       |
| 10,000       | 200         |

---

## 第 2 步：提出高层设计并获得认可

### **API 设计**
本次面试中我们只需要支持一个接口：
```
POST /v1/wallet/balance_transfer - transfers balance from one wallet to another
```

请求参数——from_account、to_account、amount（用字符串以避免丢失精度）、currency、transaction_id（幂等键）。

示例响应：
```
{
    "status": "success"
    "transaction_id": "01589980-2664-11ec-9621-0242ac130002"
}
```

### **基于内存的分片方案**
我们的钱包应用为每个用户账户维护账户余额。

表示这种数据的一个好用的数据结构是 `map<user_id, balance>`，它可以用内存中的 Redis 存储来实现。

由于单个 redis 节点无法承受 100万 TPS，我们需要把 redis 集群划分成多个节点。

示例分区算法：
```
String accountID = "A";
Int partitionNumber = 7;
Int myPartition = accountID.hashCode() % partitionNumber;
```

Zookeeper 可以用来存储分区数量以及 redis 节点的地址，因为它是一种高可用的配置存储。

最后，钱包服务是一个无状态服务，负责执行转账操作。它可以轻松地水平扩展：

<div style="margin-left:3rem">
    <img src="./images/wallet-service.png" alt="wallet-service" width="500" />
</div>

尽管这个方案解决了可扩展性问题，但它无法让我们原子性地执行余额转账。

### **分布式事务**
处理事务的一种方法是在标准的、分片的关系型数据库之上使用两阶段提交协议：

<div style="margin-left:3rem">
    <img src="./images/distributed-transactions-relational-dbs.png" alt="distributed-transactions-relational-dbs" width="500" />
</div>

两阶段提交（2PC）协议的工作方式如下：

<div style="margin-left:3rem">
    <img src="./images/2pc-protocol.png" alt="2pc-protocol" width="500" />
</div>

 * 协调者（钱包服务）像平常一样对多个数据库执行读和写操作
 * 当应用准备提交事务时，协调者要求所有数据库准备该事务
 * 如果所有数据库都回复“yes”，那么协调者要求数据库提交事务。
 * 否则，所有数据库都被要求中止该事务

2PC 方案的缺点：
 * 由于锁竞争，性能不佳
 * 协调者是单点故障

### **使用 Try-Confirm/Cancel（TC/C）的分布式事务**
TC/C 是 2PC 协议的一种变体，它借助补偿事务来工作：
 * 协调者要求所有数据库为事务预留资源
 * 协调者收集各数据库的回复——如果为 yes，则要求数据库执行 try-confirm。如果为 no，则要求数据库执行 try-cancel。

TC/C 与 2PC 的一个重要区别是：2PC 执行的是单个事务，而在 TC/C 中有两个独立的事务。

TC/C 按阶段的工作方式如下：

| 阶段 | 操作 | A                   | C                   |
|-------|-----------|---------------------|---------------------|
| 1     | Try       | 余额变化：-$1 | 不做任何事          |
| 2     | Confirm   | 不做任何事          | 余额变化：+$1 |
|       | Cancel    | 余额变化：+$1 | 不做任何事          |

阶段 1——try：

<div style="margin-left:3rem">
    <img src="./images/try-phase.png" alt="try-phase" width="500" />
</div>

 * 协调者在 A 的数据库中开启本地事务，把 A 的余额减少 1$
 * C 的数据库收到一条 NOP 指令，它不做任何事

阶段 2a——confirm：

<div style="margin-left:3rem">
    <img src="./images/confirm-phase.png" alt="confirm-phase" width="500" />
</div>

 * 如果两个数据库都回复“yes”，则开始 confirm 阶段。
 * A 的数据库收到 NOP，而 C 的数据库被指示把 C 的余额增加 1$（本地事务）

阶段 2b——cancel：

<div style="margin-left:3rem">
    <img src="./images/cancel-phase.png" alt="cancel-phase" width="500" />
</div>

 * 如果阶段 1 中任一操作失败，则开始 cancel 阶段。
 * A 的数据库被指示把 A 的余额增加 1$，C 的数据库收到 NOP

下面是 2PC 与 TC/C 的对比：

|      | 第一阶段                                            | 第二阶段：成功              | 第二阶段：失败                        |
|------|--------------------------------------------------------|------------------------------------|-------------------------------------------|
| 2PC  | 事务尚未完成                          | 提交/取消所有事务     | 取消所有事务                   |
| TC/C | 所有事务都已完成——已提交或已取消 | 如有需要则执行新事务 | 反向执行已提交的事务 |

TC/C 也被称为基于补偿的分布式事务。高层操作在业务逻辑中处理。

TC/C 的其他特性：
 * 与数据库无关，只要数据库支持事务即可
 * 分布式事务的细节与复杂性需要在业务逻辑中处理

### **TC/C 的故障模式**
如果协调者在执行过程中挂掉，它需要恢复其中间状态。
这可以通过维护阶段状态表来实现，该表在数据库分片内被原子性地更新：

<div style="margin-left:3rem">
    <img src="./images/phase-status-tables.png" alt="phase-status-tables" width="500" />
</div>

这张表包含什么：
 * 分布式事务的 ID 与内容
 * try 阶段的状态——未发送、已发送、已收到响应
 * 第二阶段的名称——confirm 或 cancel
 * 第二阶段的状态
 * 乱序标记（稍后解释）

使用 TC/C 时的一个注意事项是：当某个分布式事务正在执行中时，会存在一个短暂的时刻，各账户状态彼此不一致：

<div style="margin-left:3rem">
    <img src="./images/unbalanced-state.png" alt="unbalanced-state" width="500" />
</div>

只要我们总能从这个状态中恢复，并且用户无法利用这个中间状态（例如把它花掉），那就没问题。
这一点通过始终先执行扣减、再执行增加来保证。

| try 阶段的选择  | 账户 A | 账户 C |
|--------------------|-----------|-----------|
| 选择 1           | -$1       | NOP       |
| 选择 2（无效） | NOP       | +$1       |
| 选择 3（无效） | -$1       | +$1       |

注意，上表中的选择 3 是无效的，因为如果不依赖 2PC，我们就无法保证跨不同数据库的事务原子性执行。

需要处理的一个边界情况是乱序执行：

<div style="margin-left:3rem">
    <img src="./images/out-of-order-execution.png" alt="out-of-order-execution" width="500" />
</div>

数据库有可能在收到 try 之前先收到 cancel 操作。这个边界情况可以通过在阶段状态表中添加一个乱序标记来处理。
当我们收到 try 操作时，先检查乱序标记是否已被设置，如果已设置，就返回失败。

### **使用 Saga 的分布式事务**
另一种流行的方法是使用 Saga——一种在微服务架构中实现分布式事务的标准。

它的工作方式如下：
 * 所有操作按顺序排列。所有操作在各自数据库中都是独立的。
 * 操作从头到尾依次执行
 * 当某个操作失败时，整个过程开始通过补偿操作回滚，一直回滚到最开始

<div style="margin-left:3rem">
    <img src="./images/saga.png" alt="saga" width="500" />
</div>

我们如何协调这个工作流？可以采取两种方式：
 * 编排（Choreography）——参与 saga 的所有服务订阅相关事件，并在 saga 中完成自己那部分工作
 * 协调（Orchestration）——由一个协调者指示所有服务按正确顺序执行各自的任务

使用编排方式的挑战在于，业务逻辑被拆分到多个服务中，这些服务之间异步通信。
协调方式能很好地处理复杂性，因此它通常是数字钱包系统中的首选方式。

下面是 TC/C 与 Saga 的对比：

|                                           | TC/C            | Saga                     |
|-------------------------------------------|-----------------|--------------------------|
| 补偿操作                       | 在 Cancel 阶段 | 在回滚阶段        |
| 中心化协调                      | 是             | 是（协调模式） |
| 操作执行顺序                 | 任意             | 线性                   |
| 并行执行的可能性            | 是             | 否（线性执行）    |
| 可能看到部分不一致状态 | 是             | 是                      |
| 应用还是数据库逻辑             | 应用     | 应用              |

主要区别在于 TC/C 可以并行化，因此我们的决策取决于延迟要求——如果需要实现低延迟，我们就应该选择 TC/C 方案。

无论采取哪种方式，我们仍然需要支持审计和重放历史，以便从失败状态中恢复。

### **事件溯源**
在现实中，数字钱包应用可能会被审计，我们必须回答某些问题：
 * 我们是否知道任意时刻的账户余额？
 * 我们如何知道历史余额和当前余额是正确的？
 * 在代码变更之后，我们如何证明系统逻辑是正确的？

事件溯源是一种有助于我们回答这些问题的技术。

它包含四个概念：
 * command（命令）——来自现实世界的意图动作，例如从账户 A 向 B 转账 1$。需要有全局顺序，因此它们被放入 FIFO 队列。
   * 与 event 不同，command 可能失败，并且存在一些随机性，例如 IO 或无效状态。
   * command 可以产生零个或多个 event
   * event 的生成可能包含随机性，例如外部 IO。这一点稍后会再次讨论
 * event（事件）——关于系统中已发生事件的历史事实，例如“从 A 向 B 转账 1$”。
   * 与 command 不同，event 是在我们系统中已经发生的事实
   * 与 command 类似，它们需要有顺序，因此它们被放入 FIFO 队列
 * state（状态）——某个 event 导致的变化结果。例如账户与其余额之间的键值存储。
 * state machine（状态机）——驱动事件溯源过程。它主要负责校验 command，并应用 event 来更新系统状态。
   * 状态机应当是确定性的，因此它不应读取外部 IO 或依赖随机性。

<div style="margin-left:3rem">
    <img src="./images/event-sourcing.png" alt="event-sourcing" width="500" />
</div>

下面是事件溯源的动态视图：

<div style="margin-left:3rem">
    <img src="./images/dynamic-event-sourcing.png" alt="dynamic-event-sourcing" width="500" />
</div>

对钱包服务而言，command 就是余额转账请求。我们可以把它们放入 FIFO 队列，例如 Kafka：

<div style="margin-left:3rem">
    <img src="./images/command-queue.png" alt="command-queue" width="500" />
</div>

以下是完整图景：

<div style="margin-left:3rem">
    <img src="./images/wallet-service-state-macghine.png" alt="wallet-service-state-machine" width="500" />
</div>

 * 状态机从命令队列中读取 command
 * 从数据库中读取余额状态
 * 校验 command。如果有效，则为每个账户各生成一个 event，共两个
 * 读取下一个 event，并通过更新数据库中的余额（状态）来应用它

使用事件溯源的主要优势是其可重现性。在这个设计中，所有状态更新操作都被保存为所有余额变化的不可变历史。

历史余额始终可以通过从头重放 event 来重建。
由于 event 列表是不可变的，且状态机是确定性的，我们能保证成功重放出任意中间状态。

<div style="margin-left:3rem">
    <img src="./images/historical-states.png" alt="historical-states" width="500" />
</div>

本节开头提出的所有与审计相关的问题都可以依靠事件溯源来解决：
 * 我们是否知道任意时刻的账户余额？——可以从头开始重放 event，直到我们感兴趣的那个时间点
 * 我们如何知道历史余额和当前余额是正确的？——可以通过从头重新计算所有 event 来验证正确性
 * 在代码变更之后，我们如何证明系统逻辑是正确的？——我们可以用不同版本的代码对 event 运行，并验证它们的结果是否一致

回答客户端关于其余额的查询可以借助 CQRS 架构来实现——可以有多个只读状态机，基于不可变的 event 列表负责查询历史状态：

<div style="margin-left:3rem">
    <img src="./images/cqrs-architecture.png" alt="cqrs-architecture" width="500" />
</div>

---

## 第 3 步：深入设计
在本节中，我们将探索一些性能优化，因为我们仍然需要扩展到 100万 TPS。

### **高性能事件溯源**
我们要探索的第一个优化是把 command 和 event 保存到本地磁盘存储，而不是 Kafka 这样的外部存储。

这避免了网络延迟，同时由于我们只做追加写，这种操作对 HDD 来说通常很快。

下一个优化是把最近的 command 和 event 缓存在内存中，以节省从磁盘把它们重新加载回来的时间。

在底层，我们可以通过利用一个名为 mmap 的命令来实现上述优化，它把数据存放在本地磁盘，同时也缓存在内存中：

<div style="margin-left:3rem">
    <img src="./images/mmap-optimization.png" alt="mmap-optimization" width="500" />
</div>

我们还可以做的下一个优化是用 SQLite——一个基于文件的本地关系型数据库——把状态也存放在本地文件系统中。RocksDB 也是一个不错的选择。

对我们的目的而言，我们会选择 RocksDB，因为它使用日志结构合并树（LSM），这种结构针对写操作做了优化。
读性能通过缓存来优化。

<div style="margin-left:3rem">
    <img src="./images/rocks-db-approach.png" alt="rocks-db-approach" width="500" />
</div>

为了优化可重现性，我们可以定期把快照保存到磁盘，这样就不必每次都从头重现某个给定状态。我们可以把快照作为大型二进制文件存放在分布式文件存储中，例如 HDFS：

<div style="margin-left:3rem">
    <img src="./images/snapshot-approach.png" alt="snapshot-approach" width="500" />
</div>

### **可靠的高性能事件溯源**
到目前为止所做的所有优化都很好，但它们让我们的服务变得有状态。出于可靠性考虑，我们需要引入某种形式的复制。

在这么做之前，我们应该分析系统里哪类数据需要高可靠性：
 * state 和 snapshot 总是可以通过从 event 列表重现来重新生成。因此，我们只需要保证 event 列表的可靠性。
 * 有人可能认为我们总可以从 command 列表重新生成 event 列表，但这并不成立，因为 command 是非确定性的。
 * 结论是，我们只需要确保 event 列表的高可靠性

为了实现 event 的高可靠性，我们需要在多个节点之间复制这个列表。我们需要保证：
 * 没有数据丢失
 * 日志文件内数据的相对顺序在各副本之间保持一致

为此，我们可以采用某种共识算法，例如 Raft。

在 Raft 中，有一个处于活跃状态的 leader，还有一些处于被动状态的 follower。如果 leader 挂掉，某个 follower 会接替它。
只要超过半数的节点存活，系统就能继续运行。

<div style="margin-left:3rem">
    <img src="./images/raft-replication.png" alt="raft-replication" width="500" />
</div>

采用这种方式，所有节点都基于 event 列表更新状态。Raft 保证 leader 和 follower 拥有相同的 event 列表。

### **分布式事件溯源**
到目前为止，我们设法设计出了一个单节点性能高且可靠的系统。

我们还需要应对一些局限：
 * 单个 raft 组的容量是有限的。到了某个阶段，我们需要对数据分片并实现分布式事务
 * 在 CQRS 架构中，请求/响应流程较慢。客户端需要定期轮询系统才能得知其钱包已被更新

轮询不是实时的，因此用户可能需要等一段时间才能得知其余额发生了更新。此外，如果轮询频率过高，还可能让查询服务过载：

<div style="margin-left:3rem">
    <img src="./images/polling-approach.png" alt="polling-approach" width="500" />
</div>

为了减轻系统负载，我们可以引入反向代理，由它代表用户发送 command，并代表用户轮询响应：

<div style="margin-left:3rem">
    <img src="./images/reverse-proxy.png" alt="reverse-proxy" width="500" />
</div>

这减轻了系统负载，因为我们可以用单个请求为多个用户获取数据，但它仍然没有解决实时回执的要求。

我们还能做的最后一个改动，是让只读状态机在响应可用时主动把响应推回给反向代理。这能给用户带来更新实时发生的感觉：

<div style="margin-left:3rem">
    <img src="./images/push-state-machines.png" alt="push-state-machines" width="500" />
</div>

最后，为了进一步扩展系统，我们可以把系统分片成多个 raft 组，并在它们之上使用协调器通过 TC/C 或 Saga 来实现分布式事务：

<div style="margin-left:3rem">
    <img src="./images/sharded-raft-groups.png" alt="sharded-raft-groups" width="500" />
</div>

下面是我们最终系统中一笔余额转账请求的示例生命周期：
 * 用户 A 向 Saga 协调者发送一个包含两个操作的分布式事务——`A-1` 和 `C+1`。
 * Saga 协调者在阶段状态表中创建一条记录，以追踪该事务的状态
 * 协调者确定需要把 command 发送到哪些分区。
 * 分区 1 的 raft leader 收到 `A-1` command，对其进行校验，将其转换为 event，并在 raft 组内的其他节点上复制该 event
 * 事件结果被同步到读状态机，由它把响应推送回协调者
 * 协调者创建一条记录，表明该操作已成功，并继续执行下一个操作——`C+1`
 * 下一个操作的执行方式与第一个类似——确定分区、发送 command、执行、读状态机推送响应
 * 协调者创建一条记录，表明操作 2 也成功了，最后把结果告知客户端

---

## 第 4 步：总结
以下是我们设计方案的演进过程：
 * 我们从使用内存 Redis 的方案开始。这种方案的问题在于它不是持久化存储。
 * 我们转向使用关系型数据库，并在其上使用 2PC、TC/C 或分布式 saga 来执行分布式事务。
 * 接着，我们引入了事件溯源，以便让所有操作都可审计
 * 我们一开始使用外部数据库和队列把数据存入外部存储，但那样性能不佳
 * 我们进而把数据存入本地文件存储，利用追加写操作的性能优势。我们还使用缓存来优化读路径
 * 之前的方案虽然性能好，但并不持久。因此我们引入了带复制的 Raft 共识，以避免单点故障
 * 我们还采用了 CQRS 加上反向代理，代表用户管理事务的生命周期
 * 最后，我们把数据分区到多个 raft 组，并使用分布式事务机制——TC/C 或分布式 saga——对它们进行协调
