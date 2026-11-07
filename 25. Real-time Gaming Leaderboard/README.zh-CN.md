# 第 25 章：实时游戏排行榜

## 引言

我们要为一个在线手游设计一个**排行榜**：

<div style="margin-left:3rem">
    <img src="./images/leaderboard.png" alt="leaderboard" width="500" />
</div>

---

## 第 1 步：理解问题并确定设计范围

- C：排行榜的分数是如何计算的？
- I：用户每赢下一局比赛就能获得一分。
- C：所有玩家都会被纳入排行榜吗？
- I：是
- C：排行榜是否与某个时间段相关联？
- I：每个月都会开启一场新的锦标赛，同时开启一个新的排行榜。
- C：我们能否假设我们只关心前 10 名用户？
- I：我们希望展示前 10 名用户，以及特定用户的排名。如果时间允许，我们可以讨论展示特定用户在排行榜中附近的用户。
- C：一场锦标赛中有多少用户？
- I：500 万 DAU 和 2500 万 MAU
- C：一场锦标赛期间平均进行多少场比赛？
- I：每名玩家平均每天进行 10 场比赛
- C：如果两名玩家得分相同，我们如何确定排名？
- I：在这种情况下他们的排名相同。如果时间允许，我们可以讨论打破平局。
- C：排行榜需要是实时的吗？
- I：是的，我们希望呈现实时结果，或尽可能接近实时。呈现批量结果历史是不可接受的。

### **功能需求**

- 在排行榜上展示前 10 名玩家
- 展示某个用户的具体排名
- 展示给定用户上下各四名的用户（附加项）

### **非功能需求**

- 分数的实时更新
- 分数更新实时反映到排行榜上
- 通用的可扩展性、可用性、可靠性

### **粗略估算**

在 5000 万 DAU 的情况下，如果游戏在 24 小时内玩家分布均匀，我们平均每秒会有 50 名用户。然而，由于分布通常并不均匀，我们可以估算峰值在线用户为每秒 250 名用户。

用户得分的 QPS——假设平均每天进行 10 场比赛，50用户/秒 * 10 = 500QPS。峰值QPS = 2500。

获取前 10 名排行榜的 QPS——假设用户平均每天打开一次，QPS 为 50。

---

## 第 2 步：提出高层设计并获得认可

### **API 设计**

我们需要的第一个 API 是更新用户分数的 API：

```
POST /v1/scores
```

该 API 接收两个参数——`user_id` 和因赢得一局比赛而获得的`points`。

该 API 只应允许游戏服务器访问，而不应对终端客户端开放。

接下来一个是用于获取排行榜前 10 名玩家的 API：

```
GET /v1/scores
```

示例响应：

```
{
  "data": [
    {
      "user_id": "user_id1",
      "user_name": "alice",
      "rank": 1,
      "score": 12543
    },
    {
      "user_id": "user_id2",
      "user_name": "bob",
      "rank": 2,
      "score": 11500
    }
  ],
  ...
  "total": 10
}
```

你也可以获取特定用户的分数：

```
GET /v1/scores/{:user_id}
```

示例响应：

```
{
    "user_info": {
        "user_id": "user5",
        "score": 1000,
        "rank": 6,
    }
}
```

### **高层架构**

<div style="margin-left:3rem">
    <img src="./images/high-level-architecture.png" alt="high-level-architecture" width="500" />
</div>

- 当玩家赢下一局游戏时，客户端向游戏服务发送请求
- 游戏服务校验该胜利是否有效，并调用排行榜服务以更新该玩家的分数
- 排行榜服务在排行榜存储中更新该用户的分数
- 玩家调用排行榜服务以获取排行榜数据，例如前 10 名玩家以及给定玩家的排名

我们考虑过的另一种设计是由客户端直接在排行榜服务内更新自己的分数：

<div style="margin-left:3rem">
    <img src="./images/alternative-design.png" alt="alternative-design" width="500" />
</div>

这种方案并不安全，因为它容易受到中间人攻击。玩家可以架设一个代理，随意修改自己的分数。

另一个额外的注意事项是，对于游戏逻辑由服务器管理的游戏，客户端无需显式调用服务器来记录自己的胜利。服务器会根据游戏逻辑自动为它们完成记录。

另一个额外的考虑是，我们是否应该在游戏服务器和排行榜服务之间放置一个消息队列。如果其他服务对游戏结果感兴趣，这会很有用，但到目前为止面试中并没有明确要求，因此它没有包含在设计中：

<div style="margin-left:3rem">
    <img src="./images/message-queue-based-comm.png" alt="message-queue-based-comm" width="500" />
</div>

### **数据模型**

我们来讨论存储排行榜数据的可选方案——关系型数据库、Redis、NoSQL。

NoSQL 方案将在深度探讨部分讨论。

#### 关系型数据库方案

如果规模不重要，并且我们没有那么多用户，关系型数据库可以很好地满足我们的需求。

我们可以从一个简单的排行榜表开始，每个月一张表（个人笔记——这并不合理。你只需添加一个`month`列，就能避免每月维护新表带来的麻烦）：

<div style="margin-left:3rem">
    <img src="./images/leaderboard-table.png" alt="leaderboard-table" width="500" />
</div>

表中还有其它需要包含的数据，但它们与我们所要运行的查询无关，因此被省略了。

当用户赢得一分时会发生什么？

<div style="margin-left:3rem">
    <img src="./images/user-wins-point.png" alt="user-wins-point" width="500" />
</div>

如果用户尚不在表中，我们需要先插入该用户：

```
INSERT INTO leaderboard (user_id, score) VALUES ('mary1934', 1);
```

在后续调用中，我们只需更新其分数：

```
UPDATE leaderboard set score=score + 1 where user_id='mary1934';
```

我们如何找到排行榜中的头部玩家？

<div style="margin-left:3rem">
    <img src="./images/find-leaderboard-position.png" alt="find-leaderboard-position" width="500" />
</div>

我们可以运行以下查询：

```
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC;
```

不过这种方式的性能并不好，因为它会进行全表扫描，对数据库表中的所有记录排序。

我们可以通过在`score`上添加索引并使用`LIMIT`操作来避免扫描所有内容，从而对其进行优化：

```
SELECT (@rownum := @rownum + 1) AS rank, user_id, score
FROM leaderboard
ORDER BY score DESC
LIMIT 10;
```

然而，如果用户并不在排行榜的顶部，而你又希望定位其排名，这种方式就无法很好地扩展。

#### Redis 方案

我们希望找到一种即使面对数百万玩家也能良好运行的方案，而无需退回到复杂的数据库查询上。

Redis 是一种内存数据存储，由于它在内存中工作，所以速度很快，并且它拥有适合我们需求的数据结构——有序集合。

有序集合是一种类似于编程语言中集合的数据结构，它允许你让数据结构按给定条件保持有序。在内部，它使用哈希表维护键（user_id）与值（score）之间的映射，并使用跳表按分数顺序将分数映射到用户：

<div style="margin-left:3rem">
    <img src="./images/sorted-set.png" alt="sorted-set" width="500" />
</div>

跳表是如何工作的？
- 它是一种允许快速查找的链表
- 它由有序链表和多级索引组成

<div style="margin-left:3rem">
    <img src="./images/skip-list.png" alt="skip-list" width="500" />
</div>

当数据集足够大时，这种结构使我们能够快速查找特定的值。在下面的示例中（64 个节点），在基础链表中需要遍历 62 个节点才能找到给定值，而在跳表的情况下只需遍历 11 个节点：

<div style="margin-left:3rem">
    <img src="./images/skip-list-performance.png" alt="skip-list-performance" width="500" />
</div>

有序集合比关系型数据库性能更好，因为数据始终保持有序，代价是添加和查找操作为 O(logN)。

相反，以下是我们需要在关系型数据库中运行以查找给定用户排名的嵌套查询示例：

```
SELECT *,(SELECT COUNT(*) FROM leaderboard lb2
WHERE lb2.score >= lb1.score) RANK
FROM leaderboard lb1
WHERE lb1.user_id = {:user_id};
```

我们需要哪些操作来在 Redis 中操作我们的排行榜？
- **ZADD** - 如果用户不存在，则将其插入集合。否则，更新分数。O(logN)时间复杂度。
- **ZINCRBY** - 按给定数量增加用户的分数。如果用户不存在，分数从零开始。O(logN)时间复杂度。
- **ZRANGE/ZREVRANGE** - 获取一组用户，按他们的分数排序。我们可以指定顺序（ASC/DESC）、偏移量和结果大小。O(logN+M)时间复杂度，其中 M 为结果大小。
- **ZRANK/ZREVRANK** - 获取给定用户在 ASC/DESC 顺序中的位置（排名）。O(logN)时间复杂度。

当用户得一分时会发生什么？

```
ZINCRBY leaderboard_feb_2021 1 'mary1934'
```

每个月都会创建一个新的排行榜，而旧的排行榜会被转移到历史存储中。

当用户获取前 10 名玩家时会发生什么？

```
ZREVRANGE leaderboard_feb_2021 0 9 WITHSCORES
```

示例结果：

```
[(user2,score2),(user1,score1),(user5,score5)...]
```

那么用户获取自己的排行榜位置呢？

<div style="margin-left:3rem">
    <img src="./images/leaderboard-position-of-user.png" alt="leaderboard-position-of-user" width="500" />
</div>

在已知用户排行榜位置的情况下，这可以通过以下查询轻松实现：

```
ZREVRANGE leaderboard_feb_2021 357 365
```

用户的排名可以通过`ZREVRANK <user-id>`获取。

让我们探讨一下我们的存储需求：
- 假设最坏情况，全部 2500 万 MAU 在某个月都参与游戏
- ID 是 24 字符的字符串，分数是 16 位整数，我们需要 26 字节 * 2500万 = 约 650MB 存储
- 即使由于跳表的开销使存储成本翻倍，这也仍然能轻松装入一个现代 redis 集群

另一个需要考虑的非功能需求是支持每秒 2500 次更新。这完全在一个 Redis 服务器的能力范围之内。

其他注意事项：
- 我们可以启动一个 Redis 副本，以避免 redis 服务器崩溃时丢失数据
- 我们仍然可以利用 Redis 持久化，以便在发生崩溃时不丢失数据
- 我们需要 MySQL 中的两张辅助表，用于获取用户详情，例如用户名、显示名称等，以及记录例如用户何时赢得一场比赛
- MySQL 中的第二张表可用于在基础设施发生故障时重建排行榜
- 作为一项小的性能优化，我们可以缓存前 10 名玩家的用户详情，因为它们会被频繁访问

---

## 第 3 步：设计深度探讨

### **是否使用云服务提供商**

我们可以选择自行部署和管理服务，也可以使用云服务提供商为我们管理这些服务。

如果我们选择自行管理服务，我们将使用 redis 存储排行榜数据，使用 mysql 存储用户资料，并且如果希望扩展数据库，可能还会为用户资料添加缓存：

<div style="margin-left:3rem">
    <img src="./images/manage-services-ourselves.png" alt="manage-services-ourselves" width="500" />
</div>

或者，我们可以使用云产品为我们管理大量服务。例如，我们可以使用 AWS API Gateway 将 API 调用路由到 AWS Lambda 函数：

<div style="margin-left:3rem">
    <img src="./images/api-gateway-mapping.png" alt="api-gateway-mapping" width="500" />
</div>

AWS Lambda 使我们无需自行管理或预置服务器即可运行代码。它只在需要时运行，并自动扩展。

用户得分的示例：

<div style="margin-left:3rem">
    <img src="./images/user-scoring-point-lambda.png" alt="user-scoring-point-lambda" width="500" />
</div>

用户获取排行榜的示例：

<div style="margin-left:3rem">
    <img src="./images/user-retrieve-leaderboard.png" alt="user-retrieve-leaderboard" width="500" />
</div>

Lambda 是无服务器架构的一种实现。我们无需管理扩展和环境搭建。

作者建议，如果我们从零开始构建游戏，就采用这种方式。

### **扩展 Redis**

在 500 万 DAU 的情况下，无论从存储还是 QPS 的角度看，单个 Redis 实例都足以应付。

然而，如果我们设想用户基数增长 10 倍达到 5 亿 DAU，那么我们需要 65gb 存储，QPS 会增长到 250k。

这样的规模将需要分片。

实现分片的一种方式是对数据进行范围分区：

<div style="margin-left:3rem">
    <img src="./images/range-partition.png" alt="range-partition" width="500" />
</div>

在这个示例中，我们将基于用户的分数进行分片。我们会在应用代码中维护 user_id 与分片之间的映射。我们可以通过 MySQL 或其他缓存来保存映射本身。

要获取前 10 名玩家，我们会查询分数最高的分片（`[900-1000]`）。

要获取用户的排名，我们需要计算该用户在其所在分片内的排名，并加上其他分片中所有分数更高的用户数。后者是 O(1)操作，因为每个分片的总记录数可以通过 info keyspace命令快速访问。

或者，我们可以通过 Redis Cluster 使用哈希分区。它是一个代理，基于类似一致性哈希（但不完全相同）的分区方式将数据分布到各个 redis 节点上：

<div style="margin-left:3rem">
    <img src="./images/hash-partition.png" alt="hash-partition" width="500" />
</div>

在这种设置下，计算前 10 名玩家很有挑战性。我们需要获取每个分片的前 10 名玩家，并在应用中合并结果：

<div style="margin-left:3rem">
    <img src="./images/top-10-players-calculation.png" alt="top-10-players-calculation" width="500" />
</div>

哈希分区存在一些限制：
- 如果我们需要获取前 K 名用户，而 K 很大，延迟可能会增加，因为我们需要从所有分片获取大量数据
- 延迟会随着分区数量的增长而增加
- 没有直接的方法来确定用户的排名

由于以上这些原因，作者倾向于针对该问题使用固定分区。

其他注意事项：
- 一项最佳实践是为写密集的 redis 节点分配所需内存的两倍，以便在需要时容纳快照
- 我们可以使用一个名为 Redis-benchmark 的工具来跟踪 redis 设置的性能，并做出数据驱动的决策

### **替代方案：NoSQL**

另一个值得考虑的替代方案是使用合适的 NoSQL 数据库，并针对以下方面进行优化：
- 大量写入
- 在同一分区内按分数高效地对条目排序

DynamoDB、Cassandra 或 MongoDB 都很合适。

在本章中，作者决定使用 DynamoDB。它是一个全托管的 NoSQL 数据库，提供可靠的性能和出色的可扩展性。当我们需要查询不属于主键的字段时，它还支持使用全局二级索引。

<div style="margin-left:3rem">
    <img src="./images/dynamo-db.png" alt="dynamo-db" width="500" />
</div>

让我们从一个用于存储国际象棋游戏排行榜的表开始：

<div style="margin-left:3rem">
    <img src="./images/chess-game-leaderboard-table-1.png" alt="chess-game-leaderboard-table-1" width="500" />
</div>

这样运行良好，但如果我们需要按分数查询任何内容，它就无法很好地扩展。因此，我们可以将分数作为排序键：

<div style="margin-left:3rem">
    <img src="./images/chess-game-leaderboard-table-2.png" alt="chess-game-leaderboard-table-2" width="500" />
</div>

这个设计的另一个问题是我们按月份分区。这会导致热点分区，因为最新月份的访问会相比其他月份不均衡。

我们可以使用一种称为写入分片（write sharding）的技术，即为每个键附加一个分区编号，该编号通过`user_id % num_partitions`计算得出：

<div style="margin-left:3rem">
    <img src="./images/chess-game-leaderboard-table-3.png" alt="chess-game-leaderboard-table-3" width="500" />
</div>

一个重要的权衡是我们应该使用多少个分区：
- 分区越多，写入可扩展性越高
- 然而，读取可扩展性会变差，因为我们需要查询更多分区来收集聚合结果

使用这种方法要求我们使用前面见过的“分散-聚集”（scatter-gather）技术，其时间复杂度会随着我们添加更多分区而增长：

<div style="margin-left:3rem">
    <img src="./images/scatter-gather-2.png" alt="scatter-gather-2" width="500" />
</div>

为了对分区数量做出良好的评估，我们需要进行一些基准测试。

这种 NoSQL 方案仍有一个主要缺点——很难计算某个用户的具体排名。

如果我们有足够的规模需要分片，那么我们或许可以告诉用户他们的分数处于什么“百分位”。

可以定期运行一个 cron 作业来分析分数分布，据此确定用户的百分位，例如：

```
10th percentile = score < 100
20th percentile = score < 500
...
90th percentile = score < 6500
```

---

## 第 4 步：总结

如果时间允许，还可以讨论的其他事项：
- **更快的获取** - 我们可以通过 Redis 哈希缓存用户对象，映射为`user_id -> user object`。相比查询数据库，这可以实现更快的获取。
- **打破平局** - 当两名玩家分数相同时，我们可以根据最后一场比赛的时间对他们排序，从而打破平局。
- **系统故障恢复** - 在大规模 Redis 故障的情况下，我们可以通过遍历 MySQL WAL 条目并使用一个临时脚本重建排行榜
