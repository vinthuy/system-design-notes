# 第 28 章：证券交易所

## 引言
本章我们将设计一个**电子证券交易所**。

它的基本功能是高效地撮合买方和卖方。

主要的证券交易所有**NYSE**、**NASDAQ**等。

<div style="margin-left:3rem">
    <img src="./images/world-stock-exchanges.png" alt="world-stock-exchanges" width="500" />
</div>

---

## 第 1 步：理解问题并确定设计范围
 * C：我们要交易哪些证券？股票、期权还是期货？
 * I：为简单起见，只交易股票
 * C：支持哪些订单类型——下单、撤单、改单？限价单、市价单、条件单呢？
 * I：我们需要支持下单和撤单。订单类型上我们只需要考虑限价单。
 * C：系统需要支持盘后交易吗？
 * I：不需要，只支持正常交易时段
 * C：能描述一下交易所的基本功能吗？
 * I：客户端可以下限价单或撤单，并实时收到撮合结果。他们还应该能实时查看订单簿。
 * C：交易所的规模有多大？
 * I：同时有数万名用户交易，约 100 个标的。每天数十亿笔订单。我们还需要支持用于合规的风控检查。
 * C：什么样的风控检查？
 * I：就做简单的风控检查——例如限制一个用户一天只能交易 100 万股苹果股票
 * C：用户钱包如何参与？
 * I：我们需要确保客户端在下单前有足够资金。用于挂单的资金需要被冻结，直到订单最终完成。

### **非功能性需求**
面试官提到的规模暗示我们要设计一个中小型交易所。
我们还需要保证灵活性，以便将来支持更多标的和用户。

其他非功能性需求：
 * 可用性——至少 99.99%。宕机会损害声誉
 * 容错——需要容错和快速恢复机制，以限制生产事故的影响
 * 延迟——往返延迟应在毫秒级，重点关注第 99 百分位。持续偏高的 P99 延迟会给少数用户带来糟糕的体验。
 * 安全性——我们应该有账户管理系统。为满足法律合规，我们需要支持 KYC 以验证用户身份。我们还应该为公共资源防御 DDoS。

### **粗略估算**
 * 100 个标的，每天 10 亿笔订单
 * 正常交易时段为 09:30 到 16:00（6.5 小时）
 * QPS = 1bil / 6.5 / 3600 = 43000
 * 峰值 QPS = 5*QPS = 215000
 * 市场开盘时的成交量显著更高

---

## 第 2 步：提出高层设计并获得认可

### **业务知识入门**
我们来讨论一些与交易所相关的基本概念。

经纪商在交易所与终端用户之间居中促成交互——例如 Robinhood、Fidelity 等。

机构客户使用专门的交易软件进行大额交易。他们需要专门的处理方式。
例如在大额交易时进行订单拆分，以避免影响市场。

订单类型：
 * 限价单（Limit）——以固定价格买入或卖出。它可能不会立即找到匹配，也可能被部分成交。
 * 市价单（Market）——不指定价格。立即以当前市场价格成交。

价格：
 * 买价（Bid）——买方愿意买入某只股票的最高价格
 * 卖价（Ask）——卖方愿意卖出某只股票的最低价格

美国市场有三档报价——L1、L2、L3。

L1 行情数据包含最优买价/卖价及数量：

<div style="margin-left:3rem">
    <img src="./images/l1-price.png" alt="l1-price" width="500" />
</div>

L2 包含更多价格档位：

<div style="margin-left:3rem">
    <img src="./images/l2-price.png" alt="l2-price" width="500" />
</div>

L3 显示各档位以及每档的排队数量：

<div style="margin-left:3rem">
    <img src="./images/l3-price.png" alt="l3-price" width="500" />
</div>

K 线图（candlestick）显示给定区间的开盘价和收盘价，以及最高价和最低价：

<div style="margin-left:3rem">
    <img src="./images/candlestick.png" alt="candlestick" width="500" />
</div>

FIX 是一种用于交换证券交易信息的协议，被大多数厂商使用。示例证券交易：
```
8=FIX.4.2 | 9=176 | 35=8 | 49=PHLX | 56=PERS | 52=20071123-05:30:00.000 | 11=ATOMNOCCC9990900 | 20=3 | 150=E | 39=E | 55=MSFT | 167=CS | 54=1 | 38=15 | 40=2 | 44=15 | 58=PHLX EQUITY TESTING | 59=0 | 47=C | 32=0 | 31=0 | 151=15 | 14=0 | 6=0 | 10=128 |
```

### **高层设计**

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="high-level-design" width="500" />
</div>

交易流程：
 * 客户端通过交易界面下单
 * 经纪商将订单发送到交易所
 * 订单通过客户端网关进入交易所，网关会进行校验、限流、鉴权等。订单随后被转发到订单管理器。
 * 订单管理器根据风控管理器设定的规则执行风控检查
 * 通过风控检查后，订单管理器验证钱包中是否有足够资金来完成该订单
 * 订单被发送到撮合引擎。找到匹配时，撮合引擎为买方和卖方各产生一条成交记录（称为 fill）。两个订单都被赋予序列号，从而保证确定性。
 * 成交记录返回给客户端。

行情数据流（M1-M3）：
 * 撮合引擎生成成交记录流，发送给行情数据发布器
 * 行情数据发布器构建 K 线图，并发送给数据服务
 * 行情数据存储在专用存储中用于实时分析。经纪商连接数据服务以获取及时的行情数据。

报表流（R1-R2）：
 * 报表器从订单和成交记录中收集所有必要的报表字段，并写入数据库
 * 报表字段——client_id、price、quantity、order_type、filled_quantity、remaining_quantity

交易流处于关键路径上，而其余流程不在，因此它们的延迟要求不同。

#### 交易流
交易流处于关键路径上，因此应针对低延迟做高度优化。

其核心是撮合引擎，也称为交叉引擎（cross engine）。主要职责：
 * 维护每个标的的订单簿——即某个标的的买单/卖单列表。
 * 撮合买单和卖单——一次撮合会产生两条成交记录（fill），买方向和卖方向各一条。该功能必须快速且准确
 * 将成交记录流作为行情数据分发
 * 撮合必须以确定性的顺序产生。这是高可用的基础

接下来是定序器（sequencer）——它是让撮合引擎具备确定性的关键组件，通过为每条入站订单和出站 fill 打上序列 ID 来实现。

<div style="margin-left:3rem">
    <img src="./images/sequencer.png" alt="sequencer" width="500" />
</div>

我们为入站订单和出站 fill 打序列号有几个原因：
 * 及时性与公平性
 * 快速恢复/重放
 * 精确一次（exactly-once）保证

从概念上讲，我们可以用 Kafka 作为定序器，因为它本质上就是一个入站和出站消息队列。但为了获得更低的延迟，我们将自己实现它。

订单管理器管理订单状态。它还和撮合引擎交互——发送订单并接收 fill。

订单管理器的职责：
 * 将订单送去风控检查——例如验证用户的交易量小于 100 万
 * 结合用户钱包检查订单，验证有足够资金执行该订单
 * 它将订单发送给定序器，再转发给撮合引擎。为减少带宽，只有必要的订单信息会传给撮合引擎
 * 成交记录（fill）从定序器返回，然后通过客户端网关发送给经纪商

实现订单管理器的主要挑战是状态转换管理。事件溯源是一种可行的方案（将在深入探讨中讨论）。

最后，客户端网关接收用户订单并发送给订单管理器。它的职责：

<div style="margin-left:3rem">
    <img src="./images/client-gateway.png" alt="client-gateway" width="500" />
</div>

由于客户端网关处于关键路径上，它应保持轻量。

可以为不同客户端部署多个客户端网关。例如托管引擎（colo engine）是经纪商在交易所数据中心租用的交易引擎服务器：

<div style="margin-left:3rem">
    <img src="./images/client-gateways.png" alt="client-gateways" width="500" />
</div>

#### 行情数据流
行情数据发布器从撮合引擎接收成交记录，并根据成交记录流构建订单簿/K 线图。

这些数据被发送到数据服务，由其负责向订阅者展示聚合后的数据：

<div style="margin-left:3rem">
    <img src="./images/market-data.png" alt="market-data" width="500" />
</div>

#### 报表流
报表器不在关键路径上，但它仍然是一个重要组件。

<div style="margin-left:3rem">
    <img src="./images/reporting-flow.png" alt="reporting-flow" width="500" />
</div>

它负责交易历史、税务报表、合规报表、结算等。
延迟不是报表流的关键要求。准确性和合规性更为重要。

### **API 设计**
客户端通过经纪商与证券交易所交互，进行下单、查看成交记录、查看行情、下载历史数据用于分析等。

我们在客户端网关和经纪商之间使用 RESTful API 通信。

对于机构客户，使用专有协议以满足其低延迟要求。

创建订单：
```
POST /v1/order
```

参数：
 * symbol——股票代码。String
 * side——买入或卖出。String
 * price——限价单的价格。Long
 * orderType——限价或市价（我们的设计中只支持限价单）。String
 * quantity——订单数量。Long

响应：
 * id——订单 ID。Long
 * creationTime——订单的系统创建时间。Long
 * filledQuantity——已成功成交的数量。Long
 * remainingQuantity——尚待成交的数量。Long
 * status——new/canceled/filled。String
 * 其余属性与输入参数相同

获取成交记录：
```
GET /execution?symbol={:symbol}&orderId={:orderId}&startTime={:startTime}&endTime={:endTime}
```

参数：
 * symbol——股票代码。String
 * orderId——订单 ID。可选。String
 * startTime——查询起始时间的 epoch 值 \[11\]。Long
 * endTime——查询结束时间的 epoch 值。Long

响应：
 * executions——范围内每条成交记录组成的数组（见下方属性）。Array
 * id——成交记录 ID。Long
 * orderId——订单 ID。Long
 * symbol——股票代码。String
 * side——买入或卖出。String
 * price——成交价格。Long
 * orderType——限价或市价。String
 * quantity——已成交数量。Long

获取订单簿：
```
GET /marketdata/orderBook/L2?symbol={:symbol}&depth={:depth}
```

参数：
 * symbol——股票代码。String
 * depth——每侧的订单簿深度。Int

响应：
 * bids——包含价格和数量的数组。Array
 * asks——包含价格和数量的数组。Array

获取 K 线数据：
```
GET /marketdata/candles?symbol={:symbol}&resolution={:resolution}&startTime={:startTime}&endTime={:endTime}
```

参数：
 * symbol——股票代码。String
 * resolution——K 线图窗口长度，单位为秒。Long
 * startTime——窗口起始时间的 epoch 值。Long
 * endTime——窗口结束时间的 epoch 值。Long

响应：
 * candles——每条 K 线数据组成的数组（属性见下方）。Array
 * open——每条 K 线的开盘价。Double
 * close——每条 K 线的收盘价。Double
 * high——每条 K 线的最高价。Double
 * low——每条 K 线的最低价。Double

### **数据模型**
我们的交易所有三种主要数据类型：
 * 产品、订单、成交记录
 * 订单簿
 * K 线图

#### 产品、订单、成交记录
产品描述交易标的的属性——产品类型、交易代码、UI 展示代码等。

这些数据不经常变化，主要用于 UI 渲染。

订单表示一条买入/卖出指令。成交记录是出站的撮合结果。

数据模型如下：

<div style="margin-left:3rem">
    <img src="./images/product-order-execution-data-model.png" alt="product-order-execution-data-model" width="500" />
</div>

在三个流程中都会遇到订单和成交记录：
 * 在关键路径上，它们为高性能而在内存中处理。它们由定序器存储和恢复。
 * 报表器将订单和成交记录写入数据库，用于报表场景
 * 成交记录被转发到行情数据，用于重建订单簿和 K 线图

#### 订单簿
订单簿是某金融工具的买单/卖单列表，按价格档位组织。

针对该模型的高效数据结构需要满足：
 * 常数级查找时间——获取某个价格档位或价格档位区间的挂单量
 * 快速的新增/成交/撤单操作
 * 查询最优买价/卖价
 * 遍历价格档位

订单簿执行示例：

<div style="margin-left:3rem">
    <img src="./images/order-book-execution.png" alt="order-book-execution" width="500" />
</div>

完成这笔大额订单后，随着买卖价差扩大，价格上升。

订单簿实现的伪代码示例：
```
class PriceLevel{
    private Price limitPrice;
    private long totalVolume;
    private List<Order> orders;
}

class Book<Side> {
    private Side side;
    private Map<Price, PriceLevel> limitMap;
}

class OrderBook {
    private Book<Buy> buyBook;
    private Book<Sell> sellBook;
    private PriceLevel bestBid;
    private PriceLevel bestOffer;
    private Map<OrderID, Order> orderMap;
}
```

为了更高效的实现，我们可以用双向链表代替标准列表：
 * 下新订单是 O(1)，因为我们是把订单加到链表尾部。
 * 撮合一笔订单是 O(1)，因为我们从链表头部删除订单
 * 撤单意味着从订单簿中删除订单。我们利用 `orderMap` 实现 O(1) 查找和 O(1) 删除（因为 `Order` 持有链表中前一个元素的引用）。

<div style="margin-left:3rem">
    <img src="./images/order-book-impl.png" alt="order-book-impl" width="500" />
</div>

该数据结构也被行情数据服务用于重建订单簿。

#### K 线图
K 线数据在行情数据服务中根据某一时间区间内的订单处理计算得出：
```
class Candlestick {
    private long openPrice;
    private long closePrice;
    private long highPrice;
    private long lowPrice;
    private long volume;
    private long timestamp;
    private int interval;
}

class CandlestickChart {
    private LinkedList<Candlestick> sticks;
}
```

为避免占用过多内存的一些优化：
 * 使用预分配的环形缓冲区存放 K 线，以减少分配次数
 * 限制内存中的 K 线数量，其余的持久化到磁盘

我们将使用内存列式数据库（例如 KDB）做实时分析。收盘后，数据持久化到历史数据库。

---

## 第 3 步：设计深入探讨
关于现代交易所，有一点很有意思：与大多数其他软件不同，它们通常把所有东西都跑在一台巨大的服务器上。

我们来深入看看细节。

### **性能**
对交易所而言，在所有百分位上都拥有良好的整体延迟非常重要。

我们如何降低延迟？
 * 减少关键路径上的任务数量
 * 通过减少网络/磁盘使用和/或缩短任务执行时间，来缩短每个任务的耗时

为实现第一个目标，我们剥离了关键路径上所有无关的职责，甚至移除了日志以达成最优延迟。

如果按最初的设计，会有几个瓶颈——服务之间的网络延迟以及定序器的磁盘使用。

这样的设计可以实现几十毫秒的端到端延迟。而我们要做到几十微秒。

因此，我们将所有东西放在一台服务器上，各进程通过 mmap 作为事件存储进行通信：

<div style="margin-left:3rem">
    <img src="./images/mmap-bus.png" alt="mmap-bus" width="500" />
</div>

另一个优化是使用应用循环（执行关键任务的 while 循环），并将其绑定到同一个 CPU 上以避免上下文切换：

<div style="margin-left:3rem">
    <img src="./images/application-loop.png" alt="application-loop" width="500" />
</div>

使用应用循环的另一个副作用是不存在锁竞争——即多个线程争夺同一资源。

现在我们来看看 mmap 如何工作——它是一个 UNIX 系统调用，把磁盘上的文件映射到应用的内存。

我们可以用一个小技巧：把文件创建在 `/dev/shm` 下，它代表“共享内存”。这样我们完全没有磁盘访问。

### **事件溯源**
事件溯源在[数字钱包章节](../chapter28)中有深入讨论。所有细节可参考那一章。

简而言之，我们不存储当前状态，而是存储不可变的状态转换：

<div style="margin-left:3rem">
    <img src="./images/event-sourcing.png" alt="event-sourcing" width="500" />
</div>

 * 左侧——传统模式
 * 右侧——事件溯源模式

到目前为止我们的设计是这样的：

<div style="margin-left:3rem">
    <img src="./images/design-so-far.png" alt="design-so-far" width="500" />
</div>

 * 外部域使用 FIX 协议与我们的客户端网关交互
 * 订单管理器接收新订单事件，校验后加入其内部状态。订单随后被发送到撮合核心
 * 如果订单被撮合，则生成 `OrderFilledEvent` 并通过 mmap 发送
 * 其他组件订阅事件存储并完成各自那部分处理

还有一个额外的优化——所有组件都持有一份订单管理器的副本，它被打包成库，以避免为管理订单而产生额外调用

在这个设计中，定序器发生了变化：它不再是事件存储，而是一个单一写入者，在把事件转发到事件存储之前先对其定序：

<div style="margin-left:3rem">
    <img src="./images/sequencer-deep-dive.png" alt="sequencer-deep-dive" width="500" />
</div>

### **高可用**
我们的目标是 99.99% 的可用性——每天只有 8.64 秒的宕机时间。

为此，我们必须找出交易所架构中的单点故障：
 * 为核心服务（例如撮合引擎）设置备份实例，处于待命状态
 * 积极地自动化故障检测和向备份实例的故障转移

无状态服务（如客户端网关）可以通过增加服务器轻松地水平扩展。

对于有状态组件，如果不是主节点，我们可以处理入站事件，但不发布出站事件：

<div style="margin-left:3rem">
    <img src="./images/leader-election.png" alt="leader-election" width="500" />
</div>

为检测主副本是否宕机，我们可以发送心跳来检测其是否已失效。

该机制仅在单台服务器的边界内有效。
如果我们想扩展它，可以把整台服务器设置为热/温副本，并在故障时进行故障转移。

为了在各副本之间复制事件存储，我们可以使用可靠的 UDP 以获得更快的通信。

### **容错**
如果连温实例也宕机了怎么办？这是低概率事件，但我们应该为此做好准备。

大型科技公司通过将核心数据复制到多个城市的数据中心来应对这个问题，以缓解例如自然灾害的影响。

需要考虑的问题：
 * 如果主实例宕机，我们如何、何时故障转移到备份实例？
 * 我们如何在备份实例中选择主节点？
 * 需要多长的恢复时间（RTO——恢复时间目标）？
 * 哪些功能需要恢复？我们的系统能否在降级状态下运行？

如何应对这些问题：
 * 系统可能因缺陷（同时影响主节点和副本）而宕机，我们可以用混沌工程来暴露这类边界情况和灾难性后果
 * 不过初期我们可以手动执行故障转移，直到对系统的故障模式积累足够认识
 * 可以用主节点选举（例如 Raft）来确定主节点宕机时哪个副本成为主节点

跨不同服务器进行复制的示例：

<div style="margin-left:3rem">
    <img src="./images/replication-across-servers.png" alt="replication-across-servers" width="500" />
</div>

主节点选举任期示例：

<div style="margin-left:3rem">
    <img src="./images/leader-election-terms.png" alt="leader-election-terms" width="500" />
</div>

关于 Raft 的工作原理细节，[看这里](https://thesecretlivesofdata.com/raft/)

最后，我们还需要考虑数据丢失容忍度——在情况变得严重之前我们可以丢失多少数据？
这将决定我们备份数据的频率。

对于证券交易所，数据丢失是不可接受的，所以我们必须频繁备份数据，并依靠 Raft 的复制来降低数据丢失的概率。

### **撮合算法**
稍微绕一下，看看撮合如何通过伪代码工作：
```
Context handleOrder(OrderBook orderBook, OrderEvent orderEvent) {
    if (orderEvent.getSequenceId() != nextSequence) {
        return Error(OUT_OF_ORDER, nextSequence);
    }

    if (!validateOrder(symbol, price, quantity)) {
        return ERROR(INVALID_ORDER, orderEvent);
    }

    Order order = createOrderFromEvent(orderEvent);
    switch (msgType):
        case NEW:
            return handleNew(orderBook, order);
        case CANCEL:
            return handleCancel(orderBook, order);
        default:
            return ERROR(INVALID_MSG_TYPE, msgType);

}

Context handleNew(OrderBook orderBook, Order order) {
    if (BUY.equals(order.side)) {
        return match(orderBook.sellBook, order);
    } else {
        return match(orderBook.buyBook, order);
    }
}

Context handleCancel(OrderBook orderBook, Order order) {
    if (!orderBook.orderMap.contains(order.orderId)) {
        return ERROR(CANNOT_CANCEL_ALREADY_MATCHED, order);
    }

    removeOrder(order);
    setOrderStatus(order, CANCELED);
    return SUCCESS(CANCEL_SUCCESS, order);
}

Context match(OrderBook book, Order order) {
    Quantity leavesQuantity = order.quantity - order.matchedQuantity;
    Iterator<Order> limitIter = book.limitMap.get(order.price).orders;
    while (limitIter.hasNext() && leavesQuantity > 0) {
        Quantity matched = min(limitIter.next.quantity, order.quantity);
        order.matchedQuantity += matched;
        leavesQuantity = order.quantity - order.matchedQuantity;
        remove(limitIter.next);
        generateMatchedFill();
    }
    return SUCCESS(MATCH_SUCCESS, order);
}
```

该撮合算法使用 FIFO 算法来决定某个价格档位上哪些订单参与撮合。

### **确定性**
功能确定性通过我们使用的定序器技术得到保证。

事件发生的实际时间并不重要：

<div style="margin-left:3rem">
    <img src="./images/determinism.png" alt="determinism" width="500" />
</div>

延迟确定性是我们必须跟踪的。我们可以基于对 99 或 99.99 百分位延迟的监控来计算它。

会导致延迟尖峰的因素例如 Java 中的垃圾回收事件。

### **行情数据发布器优化**
行情数据发布器从撮合引擎接收撮合结果，并据此重建订单簿和 K 线图。

由于内存不是无限的，我们只保留部分 K 线。客户端可以选择想要多细粒度的信息。更细粒度的信息可能需要更高的价格：

<div style="margin-left:3rem">
    <img src="./images/market-data-publisher.png" alt="market-data-publisher" width="500" />
</div>

环形缓冲区（又称循环缓冲区）是一个固定大小的队列，头部与尾部相连。空间是预分配的，以避免分配操作。该数据结构也是无锁的。

另一个优化环形缓冲区的技术是填充（padding），它确保序列号永远不会与其他任何东西处于同一缓存行中。

### **行情数据的分发公平性与多播**
我们需要确保订阅者同时收到数据，因为如果某人比别人先收到数据，就会获得关键的市场洞察，从而可能用来操纵市场。

为实现这一点，我们在向订阅者发布数据时可以使用基于可靠 UDP 的多播。

数据可以通过互联网以三种方式传输：
 * 单播——一个源，一个目的地
 * 广播——一个源到整个子网
 * 多播——一个源到不同子网上的一组主机

理论上，通过使用多播，所有订阅者应同时收到数据。

然而 UDP 是不可靠的，数据可能无法到达所有人。不过可以通过重传来增强它。

### **托管**
交易所为经纪商提供将其服务器托管在与交易所同一数据中心的能力。

这大幅降低延迟，可以视为一种 VIP 服务。

### **网络安全**
由于存在一些面向互联网的服务，DDoS 对交易所是一个挑战。我们的选项如下：
 * 将公共服务和数据与私有服务隔离，使 DDoS 攻击不会影响最重要的客户
 * 使用缓存层存储不经常更新的数据
 * 加固 URL 以抵御 DDoS，例如优先使用 `https://my.website.com/data/recent` 而非 `https://my.website.com/data?from=123&to=456`，因为前者更易于缓存
 * 需要有效的允许名单/阻止名单机制。
 * 可以使用限流来缓解 DDoS

---

## 第 4 步：总结
其他有趣的补充：
 * 并非所有交易所都依赖把所有东西放在一台大服务器上，但仍有一些这样做
 * 现代交易所更多地依赖云基础设施，也依赖自动做市商（AMM）来避免维护订单簿
