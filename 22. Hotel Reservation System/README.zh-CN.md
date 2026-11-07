# 第 22 章：酒店预订系统

## 引言
在本章中，我们要设计一个**酒店预订系统**，类似于 Marriott International。

该设计同样适用于其他类型的系统——Airbnb、机票预订、电影票预订。

---

## 第 1 步：理解问题并确定设计范围
在深入设计系统之前，我们应该向面试官提问以明确范围：
 - C: 系统的规模有多大？
 - I: 我们正在为一家拥有 5000 家酒店和 100 万间客房的连锁酒店构建网站
 - C: 客户是在预订时付款，还是在抵达酒店时付款？
 - I: 他们在预订时全额付款。
 - C: 客户只能通过网站预订酒店房间吗？我们是否需要支持其他预订方式，例如电话？
 - I: 他们只能通过网站或 App 进行预订。
 - C: 客户可以取消预订吗？
 - I: 可以
 - C: 还有其他需要考虑的事项吗？
 - I: 有，我们允许 10% 的超售。酒店会卖出比实际拥有更多的房间。酒店这样做是预期客户会取消预订。
 - C: 由于时间不多，我们将重点关注——展示酒店相关页面、酒店房间详情页面、预订房间、管理后台、支持超售。
 - I: 听起来不错。
 - I: 还有一点——酒店价格一直在变化。假设酒店房间的价格每天都会变化。
 - C: 好的。

### **非功能需求**
 - 支持高并发——在旺季可能会有大量客户试图预订同一家酒店。
 - 中等延迟——用户预订时最好能低延迟，但系统花几秒钟处理也是可以接受的。

### **粗略估算**
 - 总共 5000 家酒店和 100 万间客房
 - 假设 70% 的房间被占用，平均入住时长为 3 天
 - 预估每日预订量——100 万 * 0.7 / 3 = 约 240k 次预订/天
 - 每秒预订量——240k / 10^5 秒（一天中的秒数）= 约 3。平均预订 TPS 很低。

让我们估算 QPS。如果我们假设到达预订页面需要三步，且每个页面的转化率为 10%，
那么可以估算出，如果有 3 次预订，就必然有 30 次预订页面浏览和 300 次酒店房间详情页浏览。

<div style="margin-left:3rem">
    <img src="./images/qps-estimation.png" alt="qps-estimation" width="500" />
</div>

---

## 第 2 步：提出高层设计并获得认可
我们将探讨——API 设计、数据模型、高层设计。

### **API 设计**
本节 API 设计聚焦于支撑酒店预订系统所需的核心端点（采用 RESTful 实践）。

一个功能完备的系统需要更丰富的 API，以支持基于大量条件搜索房间，但本节不会涉及这些内容。
原因是它们在技术上并不具有挑战性，因此不在范围内。

**酒店相关 API**
 - `GET /v1/hotels/{id}` - 获取酒店的详细信息
 - `POST /v1/hotels` - 新增一家酒店。仅运维人员可用
 - `PUT /v1/hotels/{id}` - 更新酒店信息。仅运维人员可用
 - `DELETE /v1/hotels/{id}` - 删除一家酒店。该 API 仅运维人员可用

**房间相关 API**
 - `GET /v1/hotels/{id}/rooms/{id}` - 获取房间的详细信息
 - `POST /v1/hotels/{id}/rooms` - 新增一个房间。仅运维人员可用
 - `PUT /v1/hotels/{id}/rooms/{id}` - 更新房间信息。仅运维人员可用
 - `DELETE /v1/hotels/{id}/rooms/{id}` - 删除一个房间。仅运维人员可用

**预订相关 API**
 - `GET /v1/reservations` - 获取当前用户的预订历史
 - `GET /v1/reservations/{id}` - 获取预订的详细信息
 - `POST /v1/reservations` - 创建新预订
 - `DELETE /v1/reservations/{id}` - 取消预订

下面是一个创建预订的示例请求：

```
{
  "startDate":"2021-04-28",
  "endDate":"2021-04-30",
  "hotelID":"245",
  "roomID":"U12354673389",
  "reservationID":"13422445"
}
```

注意，`reservationID` 是用于避免重复预订的幂等键。详情见[并发问题一节](#concurrency-issues)

### **数据模型**
在选择使用哪种数据库之前，让我们先考虑访问模式。

我们需要支持以下查询：
 - 查看酒店的详细信息
 - 给定日期范围，查找可用的房型
 - 记录一次预订
 - 查询某次预订或历史预订记录

根据我们的估算，系统规模并不大，但我们需要为流量激增做好准备。

基于这些认识，我们选择关系型数据库，原因如下：
 - 关系型数据库适合读多写少的系统。
 - NoSQL 数据库通常针对写入进行优化，但我们知道写入不会很多，因为只有一小部分访问网站的用户会进行预订。
 - 关系型数据库提供 ACID 保证。这对这类系统很重要，因为如果没有它们，我们将无法防止诸如余额为负、重复扣款等问题。
 - 关系型数据库可以轻松地对数据建模，因为其结构非常清晰。

下面是我们的 schema 设计：

<div style="margin-left:3rem">
    <img src="./images/schema-design.png" alt="schema-design" width="500" />
</div>

大多数字段的含义不言自明。唯一值得一提的字段是 `status`，它表示给定房间的状态机：

<div style="margin-left:3rem">
    <img src="./images/status-state-machine.png" alt="status-state-machine" width="500" />
</div>

这种数据模型对于 Airbnb 这类系统很适用，但不适用于酒店——在酒店场景中，用户预订的不是某个特定房间，而是某种房型。
他们预订一种房型，房间号则在预订时确定。

这一不足将在[改进后的数据模型](#improved-data-model)一节中解决。

### **高层设计**
本设计采用了微服务架构。近年来它备受青睐：

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="high-level-design" width="500" />
</div>

 - **用户**：通过手机或电脑预订酒店房间
 - **管理员**：执行退款/取消支付等管理功能
 - **CDN**：缓存 JS 包、图片、视频等静态资源
 - **公共 API 网关**：全托管服务，支持限流、身份认证等。
 - **内部 API**：仅对授权人员可见。通常由 VPN 保护。
 - **酒店服务**：提供酒店和房间的详细信息。酒店和房间数据是静态的，因此可以激进地缓存。
 - **价格服务**：提供未来不同日期的房价。这个领域有一个有趣之处：价格取决于酒店在给定日期的入住率。
 - **预订服务**：接收预订请求并预留酒店房间。同时跟踪随着房间被预订/取消而变化的库存。
 - **支付服务**：处理支付，并在成功后更新预订状态。
 - **酒店管理服务**：仅对授权人员可用。允许执行某些管理功能，用于管理和查看预订、酒店等。

服务间通信可以通过 RPC 框架来承载，例如 gRPC。

---

## 第 3 步：深入设计
让我们进一步深入：
 - 改进后的数据模型
 - 并发问题
 - 可扩展性
 - 解决微服务间的数据不一致

### **改进后的数据模型**
如前一节所述，我们需要修改 API 和 schema，以支持预订房型而非特定房间。

对于预订 API，我们不再预订 `roomID`，而是预订 `roomTypeID`：

```
POST /v1/reservations
{
  "startDate":"2021-04-28",
  "endDate":"2021-04-30",
  "hotelID":"245",
  "roomTypeID":"12354673389",
  "roomCount":"3",
  "reservationID":"13422445"
}
```

下面是更新后的 schema：

<div style="margin-left:3rem">
    <img src="./images/updated-schema.png" alt="updated-schema" width="500" />
</div>

 - **room**：包含房间的信息
 - **room_type_rate**：包含给定房型的价格信息
 - **reservation**：记录客人预订数据
 - **room_type_inventory**：存储酒店房间的库存数据。

让我们看看 `room_type_inventory` 的各个列，因为这张表更有意思：
 - **hotel_id**：酒店的 id
 - **room_type_id**：房型的 id
 - **date**：单个日期
 - **total_inventory**：房间总数减去临时从库存中移除的房间数。
 - **total_reserved**：给定 (hotel_id, room_type_id, date) 已预订的房间总数

设计这张表还有其他方式，但按 (hotel_id, room_type_id, date) 每行一个房间，可以轻松实现
预订管理，并使查询更简单。

表中的行通过每日 CRON 任务预先填充。

示例数据：
| hotel_id | room_type_id | date       | total_inventory | total_reserved |
|----------|--------------|------------|-----------------|----------------|
| 211      | 1001         | 2021-06-01 | 100             | 80             |
| 211      | 1001         | 2021-06-02 | 100             | 82             |
| 211      | 1001         | 2021-06-03 | 100             | 86             |
| 211      | 1001         | ...        | ...             |                |
| 211      | 1001         | 2023-05-31 | 100             | 0              |
| 211      | 1002         | 2021-06-01 | 200             | 16             |
| 2210     | 101          | 2021-06-01 | 30              | 23             |
| 2210     | 101          | 2021-06-02 | 30              | 25             |

用于检查某类房型可用性的示例 SQL 查询：

```
SELECT date, total_inventory, total_reserved
FROM room_type_inventory
WHERE room_type_id = ${roomTypeId} AND hotel_id = ${hotelId}
AND date between ${startDate} and ${endDate}
```

如何使用这些数据检查指定数量房间的可用性（注意我们支持超售）：

```
if (total_reserved + ${numberOfRoomsToReserve}) <= 110% * total_inventory
```

现在我们来估算一下存储量。
 - 我们有 5000 家酒店。
 - 每家酒店有 20 种房型。
 - 5000 * 20 * 2（年）* 365（天）= 7300 万行

7300 万行数据量并不大，单台数据库服务器就能处理。
不过，搭建读复制（可能跨不同可用区）以实现高可用是合理的。

追问——如果预订数据太大，单台数据库放不下，你会怎么做？
 - 只存储当前和未来的预订数据。预订历史可以迁移到冷存储。
 - 数据库分片——我们可以按 `hash(hotel_id) % servers_cnt` 对数据分片，因为我们在查询中总是会带上 `hotel_id`。

### **并发问题**
另一个需要解决的重要问题是重复预订。

有两个问题需要解决：
 - 同一用户点击两次“预订”
 - 多个用户同时尝试预订同一个房间

第一个问题的可视化如下：

<div style="margin-left:3rem">
    <img src="./images/double-booking-single-user.png" alt="double-booking-single-user" width="500" />
</div>

解决这个问题有两种方式：
 - 客户端处理——前端可以在点击一次后禁用预订按钮。但是，如果用户禁用了 javascript，他们就看不到按钮变灰。
 - 幂等 API——在 API 中加入幂等键，使用户能够只执行一次操作，无论该端点被调用多少次：

<div style="margin-left:3rem">
    <img src="./images/idempotency.png" alt="idempotency" width="500" />
</div>

该流程的工作方式如下：
 - 当你在填写信息并进行预订的过程中，系统会生成一个预订订单。该预订订单使用全局唯一标识符生成。
 - 使用上一步生成的 `reservation_id` 提交预订 1。
 - 如果第二次点击“完成预订”，发送的仍是同一个 `reservation_id`，后端会检测到这是一次重复预订。
 - 通过为 `reservation_id` 列添加唯一约束来避免重复，从而防止数据库中存储具有该 id 的多条记录。

<div style="margin-left:3rem">
    <img src="./images/unique-constraint-violation.png" alt="unique-constraint-violation" width="500" />
</div>

如果是多个用户进行同一笔预订呢？

<div style="margin-left:3rem">
    <img src="./images/double-booking-multiple-users.png" alt="double-booking-multiple-users" width="500" />
</div>

 - 假设事务隔离级别不是可串行化
 - 用户 1 和 2 同时尝试预订同一个房间。
 - 事务 1 检查是否有足够的房间——有
 - 事务 2 检查是否有足够的房间——有
 - 事务 2 预订了房间并更新库存
 - 事务 1 也预订了房间，因为它仍然看到 100 间房中有 99 间 `total_reserved` 房间。
 - 两个事务都成功提交了更改

这个问题可以通过某种形式的锁机制来解决：
 - 悲观锁
 - 乐观锁
 - 数据库约束

下面是我们用于预订房间的 SQL：

```sql
# step 1: check room inventory
SELECT date, total_inventory, total_reserved
FROM room_type_inventory
WHERE room_type_id = ${roomTypeId} AND hotel_id = ${hotelId}
AND date between ${startDate} and ${endDate}

# For every entry returned from step 1
if((total_reserved + ${numberOfRoomsToReserve}) > 110% * total_inventory) {
  Rollback
}

# step 2: reserve rooms
UPDATE room_type_inventory
SET total_reserved = total_reserved + ${numberOfRoomsToReserve}
WHERE room_type_id = ${roomTypeId}
AND date between ${startDate} and ${endDate}

Commit
```

#### 方案 1：悲观锁
悲观锁通过在记录被更新期间对其加锁，来防止并发更新。

在 MySQL 中，这可以通过 `SELECT... FOR UPDATE` 查询实现，它会锁定该查询选中的行，直到事务提交。

<div style="margin-left:3rem">
    <img src="./images/pessimistic-locking.png" alt="pessimistic-locking" width="500" />
</div>

优点：
 - 防止应用更新正在被修改的数据
 - 易于实现，并通过串行化更新来避免冲突。在数据争用严重时很有用。

缺点：
 - 当锁定多个资源时，可能发生死锁。
 - 这种方式不可扩展——如果事务被锁定太久，会影响所有试图访问该资源的其他事务。
 - 当查询选中大量资源且事务持续时间很长时，影响会很严重。

由于可扩展性问题，作者不推荐这种方式。

#### 方案 2：乐观锁
乐观锁允许多个用户同时尝试更新一条记录。

有两种常见的实现方式——版本号和时间戳。推荐使用版本号，因为服务器时钟可能不准确。

<div style="margin-left:3rem">
    <img src="./images/optimistic-locking.png" alt="optimistic-locking" width="500" />
</div>

 - 在数据库表中新增一个 `version` 列
 - 在用户修改数据库行之前，先读取版本号
 - 当用户更新该行时，版本号加 1 并写回数据库
 - 如果新的版本号没有超过之前的版本号，数据库校验会阻止该插入

乐观锁通常比悲观锁更快，因为我们没有对数据库加锁。
然而，当并发很高时，其性能往往会下降，因为那会导致大量回滚。

优点：
 - 防止应用编辑过期数据
 - 我们不需要在数据库中获取锁
 - 在数据争用低（即很少出现更新冲突）时是首选方案

缺点：
 - 数据争用高时性能较差

乐观锁对我们的系统来说是个不错的选择，因为预订 QPS 并不是极高。

#### 方案 3：数据库约束
这种方式与乐观锁非常相似，只是防护措施是通过数据库约束来实现的：

```
CONSTRAINT `check_room_count` CHECK((`total_inventory - total_reserved` >= 0))
```

<div style="margin-left:3rem">
    <img src="./images/database-constraint.png" alt="database-constraint" width="500" />
</div>

优点：
 - 易于实现
 - 在数据争用较小时效果很好

缺点：
 - 与乐观锁类似，在数据争用高时性能较差
 - 数据库约束不像应用代码那样容易进行版本控制
 - 并非所有数据库都支持约束

由于实现简单，这也是酒店预订系统的另一个不错选择。

### **可扩展性**
通常，酒店预订系统的负载并不高。

然而，面试官可能会问你，如果系统被一个更大、更热门的旅游网站（例如 booking.com）采用，你会如何处理
在这种情况下，QPS 可能会大 1000 倍。

出现这种情况时，重要的是要弄清楚我们的瓶颈在哪里。所有服务都是无状态的，因此可以通过复制轻松扩展。

然而，数据库是有状态的，它如何扩展并不那么显而易见。

一种扩展方式是实现数据库分片——我们可以将数据拆分到多个数据库，每个数据库只包含一部分数据。

我们可以基于 `hotel_id` 进行分片，因为所有查询都会基于它进行过滤。
假设 QPS 为 30,000，将数据库分成 16 个分片后，每个分片处理 1875 QPS，这在单个 MySQL 集群的负载能力范围之内。

<div style="margin-left:3rem">
    <img src="./images/database-sharding.png" alt="database-sharding" width="500" />
</div>

我们还可以通过 Redis 为房间库存和预订启用缓存。我们可以设置 TTL，让过去日期的旧数据过期。

<div style="margin-left:3rem">
    <img src="./images/inventory-cache.png" alt="inventory-cache" width="500" />
</div>

我们存储库存的方式基于 `hotel_id`、`room_type_id` 和 `date`：

```
key: hotelID_roomTypeID_{date}
value: the number of available rooms for the given hotel ID, room type ID and date.
```

数据一致性是异步发生的，通过 CDC 流式机制来管理——读取数据库变更并将其应用到另一个系统。
Debezium 是将数据库变更与 Redis 同步的常用选择。

使用这种机制，缓存和数据库可能会在一段时间内不一致。
在我们的场景中这没问题，因为数据库会阻止我们做出无效的预订。

这在 UI 上会造成一些问题，因为用户需要刷新页面才能看到“已经没有房间了”，
但无论是否存在这个问题，这种情况都可能发生，例如某人在预订前犹豫了很久。

缓存优点：
 - 减少数据库负载
 - 高性能，因为 Redis 在内存中管理数据

缓存缺点：
 - 维护缓存与 DB 之间的一致性很难。我们需要考虑这种不一致如何影响用户体验。

### **服务间的数据一致性**
单体应用使我们能够使用共享的关系型数据库来保证数据一致性。

在我们的微服务设计中，我们选择了一种混合方式：一些服务是独立的，
但预订和库存 API 由同一个服务处理——该服务同时用于预订和库存 API。

这样做是因为我们想利用关系型数据库的 ACID 保证来确保一致性。

然而，面试官可能会质疑这种方案，因为它不是纯粹的微服务架构——在那种架构中，每个服务都有专用的数据库：

<div style="margin-left:3rem">
    <img src="./images/microservices-vs-monolith.png" alt="microservices-vs-monolith" width="500" />
</div>

这可能导致一致性问题。在单体服务器中，我们可以利用关系型数据库的事务能力来实现原子操作：

<div style="margin-left:3rem">
    <img src="./images/atomicity-monolith.png" alt="atomicity-monolith" width="500" />
</div>

然而，当操作跨越多个服务时，保证这种原子性更具挑战性：

<div style="margin-left:3rem">
    <img src="./images/microservice-non-atomic-operation.png" alt="microservice-non-atomic-operation" width="500" />
</div>

有一些众所周知的技术可以处理这些数据不一致：
 - **两阶段提交**：一种数据库协议，保证跨多个节点的原子事务提交。
   然而它性能不佳，因为单个节点滞后会导致所有节点阻塞该操作。
 - **Saga**：一系列本地事务，如果工作流中的任何步骤失败，就会触发补偿事务。这是一种最终一致的方式。

值得注意的是，解决微服务间的数据不一致是一个具有挑战性的问题，会增加系统复杂度。
考虑到我们更务实的做法——将相互依赖的操作封装在同一个关系型数据库中——值得考虑这样的代价是否值得。

---

## 第 4 步：总结
我们给出了一个酒店预订系统的设计。

我们经历了以下步骤：
 - 收集需求并进行粗略估算，以理解系统规模
 - 在高层设计中，我们给出了 API 设计、数据模型和系统架构
 - 在深入设计中，随着需求变化，我们探讨了替代的数据库 schema 设计
 - 我们讨论了竞态条件并提出了解决方案——悲观锁/乐观锁、数据库约束
 - 通过数据库分片和缓存来扩展系统的方式
 - 最后，我们讨论了如何处理跨多个微服务的数据一致性问题
