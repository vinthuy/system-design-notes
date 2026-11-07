# 第 20 章：指标监控与告警系统

## 引言
本章聚焦于设计一个高度可扩展的**指标监控与告警系统**，它对于确保高可用和可靠性至关重要。

---

## 第 1 步：理解问题并确定设计范围
指标监控系统可以指很多不同的东西——例如，当面试官只对基础设施指标感兴趣时，你并不想设计一个日志聚合系统。

让我们先试着理解问题：
 - C: 我们为谁构建这个系统？是为大型科技公司构建内部监控系统，还是构建像DataDog这样的SaaS？
 - I: 我们只为内部使用而构建。
 - C: 我们想收集哪些指标？
 - I: 运维系统指标——CPU负载、内存、数据磁盘空间。但也有每秒请求数这样的高层指标。业务指标不在范围内。
 - C: 我们监控的基础设施规模有多大？
 - I: 1亿日活用户、1000个服务器池、每个池100台机器
 - C: 数据要保留多久？
 - I: 假设保留1年。
 - C: 为了长期存储，我们可以降低指标数据的分辨率吗？
 - I: 新接收的指标保留7天。之后的30天将其汇总为1分钟分辨率。30天之后再进一步汇总为1小时分辨率。
 - C: 支持哪些告警渠道？
 - I: 邮件、电话、PagerDuty或webhook。
 - C: 我们需要收集错误日志或访问日志这类日志吗？
 - I: 不需要
 - C: 我们需要支持分布式系统链路追踪吗？
 - I: 不需要

### **高层需求与假设**
被监控的基础设施规模很大：
 - 1亿DAU
 - 1000个服务器池 * 100台机器 * 每台机器约100个指标 -> 约1000万个指标
 - 1年数据保留期
 - 数据保留策略——原始数据保留7天，1分钟分辨率保留30天，1小时分辨率保留1年

可以监控多种指标：
 - CPU负载
 - 请求数
 - 内存使用量
 - 消息队列中的消息数量

### **非功能性需求**
 - **可扩展性**：系统应可扩展，以容纳更多指标和告警
 - **低延迟**：系统需要为仪表盘和告警提供低查询延迟
 - **可靠性**：系统应高度可靠，以避免漏掉关键告警
 - **灵活性**：系统应能够在未来轻松集成新技术

哪些需求不在范围内？
 - **日志监控**：ELK技术栈在此类场景中非常流行
 - **分布式系统链路追踪**：指收集请求在系统中流经多个服务时的生命周期数据

---

## 第 2 步：提出高层设计并获得认可

### **基础概念**
指标监控与告警系统涉及五个核心组件：

<div style="margin-left:3rem">
    <img src="./images/metrics-monitoring-core-components.png" alt="metrics-monitoring-core-components" width="500" />
</div>

 - **数据收集**：从不同来源收集指标数据
 - **数据传输**：将数据从来源传输到指标监控系统
 - **数据存储**：组织和存储传入的数据
 - **告警**：分析传入的数据，检测异常并生成告警
 - **可视化**：以图表、图形等形式呈现数据

### **数据模型**
指标数据通常被记录为时序，它包含一组带时间戳的值。
该序列可以通过名称和一组可选的标签来标识。

示例 1——生产服务器实例i631在20:00的CPU负载是多少？

<div style="margin-left:3rem">
    <img src="./images/metrics-example-1.png" alt="metrics-example-1" width="500" />
</div>

数据可以用下表来标识：

<div style="margin-left:3rem">
    <img src="./images/metrics-example-1-data.png" alt="metrics-example-1-data" width="500" />
</div>

时序由指标名称、标签以及特定时间的一个数据点来标识。

示例 2——us-west区域所有Web服务器在过去10分钟的平均CPU负载是多少？

```
CPU.load host=webserver01,region=us-west 1613707265 50

CPU.load host=webserver01,region=us-west 1613707265 62

CPU.load host=webserver02,region=us-west 1613707265 43

CPU.load host=webserver02,region=us-west 1613707265 53

...

CPU.load host=webserver01,region=us-west 1613707265 76

CPU.load host=webserver01,region=us-west 1613707265 83
```

这是我们从存储中取出以回答该问题的一个示例数据。
平均CPU负载可以通过对各行最后一列的数值求平均来计算。

上面展示的格式称为行协议（line protocol），市场上许多流行的监控软件都在使用它，例如Prometheus、OpenTSDB。

每个时序由什么构成：

<div style="margin-left:3rem">
    <img src="./images/time-series-data-example.png" alt="time-series-data-example" width="500" />
</div>

一个直观展示数据样貌的好方法：

<div style="margin-left:3rem">
    <img src="./images/time-series-data-viz.png" alt="time-series-data-viz" width="500" />
</div>

 - x轴是时间
 - y轴是你要查询的维度——例如指标名称、标签等。

数据访问模式是写密集且读取呈尖峰状：我们收集大量指标，但它们很少被访问，不过在例如有事故正在发生时会突发访问。

数据存储系统是本设计的核心。
 - 不建议为此问题使用通用数据库，尽管借助专家级调优你也能获得不错的规模表现。
 - 使用NoSQL数据库在理论上可行，但很难设计出一个可扩展的模式来有效存储和查询时序数据。

有许多专门为存储时序数据而定制的数据库。它们中的许多支持自定义查询接口，可以高效地查询时序数据。
 - OpenTSDB是一个分布式时序数据库，但它基于Hadoop和HBase。如果你没有预置那套基础设施，使用这项技术会很困难。
 - Twitter使用MetricsDB，而Amazon提供Timestream。
 - 最流行的两个时序数据库是InfluxDB和Prometheus。
 - 它们被设计用于存储海量时序数据。两者都基于内存缓存 + 磁盘存储。

InfluxDB的规模示例——配置8核和32GB RAM时每秒超过25万次写入：

<div style="margin-left:3rem">
    <img src="./images/influxdb-scale.png" alt="influxdb-scale" width="500" />
</div>

并不期望你理解指标数据库的内部实现，因为这是小众知识。只有你在简历中提到过它，才可能被问到。

就面试而言，理解指标是时序数据，并了解像InfluxDB这样流行的时序数据库，就足够了。

时序数据库的一个很好的特性是能按标签高效地聚合和分析大量时序数据。
例如，InfluxDB会为每个标签建立索引。

然而，关键是要保持标签的基数低，即不要使用太多唯一标签。

### **高层设计**

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="high-level-design" width="500" />
</div>

 - **指标来源**：可以是应用服务器、SQL数据库、消息队列等。
 - **指标收集器**：收集指标数据并写入时序数据库
 - **时序数据库**：将指标存储为时序。提供自定义查询接口，用于分析大量指标。
 - **查询服务**：让从时序数据库中查询和获取数据变得简单。如果数据库接口足够强大，完全可以由它替代。
 - **告警系统**：向各种告警目的地发送告警通知。
 - **可视化系统**：以图形/图表的形式展示指标。

---

## 第 3 步：深入设计
让我们深入探讨系统中几个更有意思的部分。

### **指标收集**
对于指标收集而言，偶尔的数据丢失并不关键。客户端采用发后即忘（fire and forget）的方式是可以接受的。

<div style="margin-left:3rem">
    <img src="./images/metrics-collection.png" alt="metrics-collection" width="500" />
</div>

实现指标收集有两种方式——拉（pull）或推（push）。

拉模型可能长这样：

<div style="margin-left:3rem">
    <img src="./images/pull-model-example.png" alt="pull-model-example" width="500" />
</div>

对于这个方案，指标收集器需要维护一份最新的服务和指标端点列表。
为此我们可以使用Zookeeper或etcd——服务发现。

服务发现包含关于何时、从何处收集指标的配置规则：

<div style="margin-left:3rem">
    <img src="./images/service-discovery-example.png" alt="service-discovery-example" width="500" />
</div>

以下是指标收集流程的详细说明：

<div style="margin-left:3rem">
    <img src="./images/metrics-collection-flow.png" alt="metrics-collection-flow" width="500" />
</div>

 - 指标收集器从服务发现获取配置元数据。这包括拉取间隔、IP地址、超时和重试参数。
 - 指标收集器通过预定义的http端点（例如 `/metrics`）拉取指标数据。这通常由客户端库完成。
 - 或者，指标收集器可以向服务发现注册变更事件通知，以便在服务端点发生变化时收到通知。
 - 另一种选择是指标收集器定期轮询指标端点配置的变化。

在我们的规模下，单个指标收集器是不够的。必须有多个实例。
然而，它们之间也必须存在某种同步机制，以免两个收集器重复收集同一份指标。

一种解决方案是将收集器和服务器放置在一个一致性哈希环上，并只把一组服务器关联到单个收集器：

<div style="margin-left:3rem">
    <img src="./images/consistent-hash-ring.png" alt="consistent-hash-ring" width="500" />
</div>

另一方面，在推模型中，服务主动将指标推送给指标收集器：

<div style="margin-left:3rem">
    <img src="./images/push-model-example.png" alt="push-model-example" width="500" />
</div>

在这种方式中，通常会在服务实例旁安装一个收集代理。
该代理从服务器收集指标并将其推送到指标收集器。

<div style="margin-left:3rem">
    <img src="./images/metrics-collector-agent.png" alt="metrics-collector-agent" width="500" />
</div>

在这种模型下，我们可以在将指标发送给收集器之前先进行聚合，从而减少收集器处理的数据量。

反过来，指标收集器可能会拒绝推送请求，因为它无法处理负载。
因此，将收集器加入负载均衡器后面的自动扩缩组很重要。

那么哪种更好？两种方式各有取舍，不同系统采用不同方式：
 - Prometheus采用拉架构
 - Amazon Cloud Watch和Graphite采用推架构

以下是推和拉之间的一些主要差异：
|                                        | 拉                                                                                                                                                                                                    | 推                                                                                                                                                                                                                                    |
|----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 易于调试                         | 应用服务器上用于拉取指标的/metrics端点可以随时用来查看指标。你甚至可以在自己的笔记本电脑上这样做。拉胜出。                                          | 如果指标收集器没有收到指标，问题可能由网络故障引起。                                                                                                                                        |
| 健康检查                           | 如果某台应用服务器不响应拉取，你可以快速判断出该应用服务器是否宕机。拉胜出。                                                                           | 如果指标收集器没有收到指标，问题可能由网络故障引起。                                                                                                                                        |
| 短生命周期任务                       |                                                                                                                                                                                                         | 有些批处理任务可能生命周期很短，存活时间不足以被拉取。推胜出。可以通过为拉模型引入推送网关（push gateway）来解决 [22]。                                                                 |
| 防火墙或复杂网络环境 | 让服务器拉取指标要求所有指标端点都可达。在多数据中心场景下这可能会成为问题。它可能需要更精细的网络基础设施。 | 如果指标收集器配置了负载均衡器和自动扩缩组，就可以从任何地方接收数据。推胜出。                                                                                             |
| 性能                            | 拉方式通常使用TCP。                                                                                                                                                                         | 推方式通常使用UDP。这意味着推方式提供更低延迟的指标传输。反对意见是：与发送指标负载相比，建立TCP连接的开销很小。 |
| 数据真实性                      | 要收集指标的应用服务器事先在配置文件中定义好。从这些服务器收集的指标可以保证是真实的。                                                 | 任何类型的客户端都可以向指标收集器推送指标。可以通过白名单限制接受哪些服务器的指标，或要求身份认证来解决。                                                                   |

没有明显的赢家。大型组织可能需要同时支持两者。有些情况下可能根本无法安装推送代理。

### **扩展指标传输管道**

<div style="margin-left:3rem">
    <img src="./images/metrics-transmission-pipeline.png" alt="metrics-transmission-pipeline" width="500" />
</div>

无论我们使用推模型还是拉模型，指标收集器都部署在自动扩缩组中。

然而，如果时序数据库宕机，就有可能丢失数据。为缓解这一点，我们将部署一套排队机制：

<div style="margin-left:3rem">
    <img src="./images/queuing-mechanism.png" alt="queuing-mechanism" width="500" />
</div>

 - 指标收集器将指标数据推送到kafka
 - 消费者或流处理服务（如Apache Storm、Flink或Spark）处理数据并将其推送到时序数据库

这种方式有几个优点：
 - Kafka被用作高度可靠且可扩展的分布式消息平台
 - 它将数据收集与数据处理解耦
 - 它可以通过在Kafka中保留数据来防止数据丢失

可以将Kafka配置为每个指标名称一个分区，这样消费者就可以按指标名称聚合数据。
为了扩展，我们可以进一步按标签分区，并对指标进行分类/优先级排序，让某些指标优先被收集。

<div style="margin-left:3rem">
    <img src="./images/metrics-collection-kafka.png" alt="metrics-collection-kafka" width="500" />
</div>

为此问题使用Kafka的主要缺点是维护/运维开销。
另一种选择是使用像 [Gorilla](https://www.vldb.org/pvldb/vol8/p1816-teller.pdf) 这样的大规模摄取系统。
可以说，使用它在可扩展性上不亚于用Kafka做排队。

### **聚合可以发生的位置**
指标可以在多个位置进行聚合。不同选择之间存在取舍：
 - **收集代理**：客户端收集代理只支持简单的聚合逻辑。例如收集1分钟的计数器并发送给指标收集器。
 - **摄取管道**：要在写入数据库之前聚合数据，我们需要像Flink这样的流处理引擎。这减少了写入量，但由于我们不存储原始数据，会损失数据精度。
 - **查询侧**：我们可以在通过可视化系统运行查询时聚合数据。这样没有数据丢失，但由于要处理大量数据，查询可能很慢。

### **查询服务**
将查询服务与时序数据库分离，可以把可视化和告警系统与数据库解耦，从而使我们能够将数据库与客户端解耦并随意替换它。

我们可以在这里增加一个缓存层，以减轻时序数据库的负载：

<div style="margin-left:3rem">
    <img src="./images/cache-layer-query-service.png" alt="cache-layer-query-service" width="500" />
</div>

我们也可以完全不添加查询服务，因为大多数可视化和告警系统都有强大的插件来集成大多数时序数据库。
如果选择了一款合适的时序数据库，我们可能也不需要引入自己的缓存层。

大多数时序数据库不支持SQL，仅仅是因为SQL在查询时序数据时效率不高。下面是一个用于计算指数移动平均的SQL查询示例：

```
select id,
       temp,
       avg(temp) over (partition by group_nr order by time_read) as rolling_avg
from (
  select id,
         temp,
         time_read,
         interval_group,
         id - row_number() over (partition by interval_group order by time_read) as group_nr
  from (
    select id,
    time_read,
    "epoch"::timestamp + "900 seconds"::interval * (extract(epoch from time_read)::int4 / 900) as interval_group,
    temp
    from readings
  ) t1
) t2
order by time_read;
```

下面是同样的查询用Flux编写的版本——Flux是InfluxDB使用的查询语言：

```
from(db:"telegraf")
  |> range(start:-1h)
  |> filter(fn: (r) => r._measurement == "foo")
  |> exponentialMovingAverage(size:-10s)
```

### **存储层**
谨慎选择时序数据库很重要。

根据Facebook发表的研究，对运营存储的查询中约85%是针对过去26小时的数据。

如果我们选择一款能利用这一特性的数据库，可能会对系统性能产生显著影响。InfluxDB就是这样一个选项。

无论我们选择哪个数据库，都可以采用一些优化手段。

数据编码和压缩可以显著减小数据体积。这些特性通常内建于优秀的时序数据库中。

<div style="margin-left:3rem">
    <img src="./images/double-delta-encoding.png" alt="double-delta-encoding" width="500" />
</div>

在上面的示例中，我们可以存储时间戳增量，而不是存储完整的时间戳。

另一种可以采用的技术是降采样——将高分辨率数据转换为低分辨率数据，以减少磁盘占用。

我们可以对旧数据使用这一技术，并让数据科学家可以配置规则，例如：
 - 7天——不降采样
 - 30天——降采样到1分钟
 - 1年——降采样到1小时

例如，下面是一个10秒分辨率的指标表：
| 指标 | 时间戳            | 主机名 | 指标值 |
|--------|----------------------|----------|--------------|
| cpu    | 2021-10-24T19:00:00Z | host-a   | 10           |
| cpu    | 2021-10-24T19:00:10Z | host-a   | 16           |
| cpu    | 2021-10-24T19:00:20Z | host-a   | 20           |
| cpu    | 2021-10-24T19:00:30Z | host-a   | 30           |
| cpu    | 2021-10-24T19:00:40Z | host-a   | 20           |
| cpu    | 2021-10-24T19:00:50Z | host-a   | 30           |

降采样到30秒分辨率：
| 指标 | 时间戳            | 主机名 | 指标值（平均值） |
|--------|----------------------|----------|--------------------|
| cpu    | 2021-10-24T19:00:00Z | host-a   | 19                 |
| cpu    | 2021-10-24T19:00:30Z | host-a   | 25                 |

最后，我们还可以使用冷存储来存放不再使用的旧数据。冷存储的财务成本要低得多。

### **告警系统**

<div style="margin-left:3rem">
    <img src="./images/alerting-system.png" alt="alerting-system" width="500" />
</div>

配置被加载到缓存服务器。规则通常以YAML格式定义。下面是一个示例：

```
- name: instance_down
  rules:

  # Alert for any instance that is unreachable for >5 minutes.
  - alert: instance_down
    expr: up == 0
    for: 5m
    labels:
      severity: page
```

告警管理器从缓存中获取告警配置。根据配置规则，它还会按预定义的间隔调用查询服务。
如果某条规则被满足，就会创建一个告警事件。

告警管理器的其他职责包括：
 - 过滤、合并和去重告警。例如，如果单个实例的告警被触发多次，只会生成一个告警事件。
 - 访问控制——重要的是只允许特定人员执行告警管理操作
 - 重试——管理器确保告警至少被传播一次。

告警存储是一个键值数据库，如Cassandra，它保存所有告警的状态。它确保通知至少被发送一次。
一旦告警被触发，它就被发布到Kafka。

最后，告警消费者从Kafka拉取告警数据，并将通知发送到不同渠道——邮件、短信、PagerDuty、webhook。

在现实中，告警系统有许多现成的解决方案。很难有理由自研一套系统。

### **可视化系统**
可视化系统展示一段时间内的指标和告警。下面是一个用Grafana构建的仪表盘：

<div style="margin-left:3rem">
    <img src="./images/grafana-dashboard.png" alt="grafana-dashboard" width="500" />
</div>

构建高质量的可视化系统非常困难。很难有理由不使用像Grafana这样的现成解决方案。

---

## 第 4 步：总结
这是我们的最终设计：

<div style="margin-left:3rem">
    <img src="./images/final-design.png" alt="final-design" width="500" />
</div>
