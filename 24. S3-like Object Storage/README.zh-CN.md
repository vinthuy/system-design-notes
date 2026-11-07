# 第24章：类S3对象存储

## 引言

在本章中，我们将设计一个**对象存储**服务，类似于**Amazon S3**。

存储系统大致分为三大类：
- **块存储**
- **文件存储**
- **对象存储**

**块存储**是20世纪60年代出现的设备。HDD和SSD就是这样的例子。
这些设备通常以物理方式连接到服务器，不过也可以通过高速网络协议实现网络挂载。
服务器可以格式化这些原始块并将它们用作文件系统，也可以将它们的控制权直接移交给服务器。

**文件存储**构建在块存储之上。它提供了更高层次的抽象，使管理文件夹和文件更加容易。

**对象存储**牺牲性能以换取高持久性、海量规模和低成本。
它面向“冷”数据，主要用于归档和备份。
它没有层级目录结构，所有数据都以对象的形式存储在扁平结构中。
与其他存储类型相比，它相对较慢。大多数云厂商都提供对象存储产品——Amazon S3、Google GCS等。

<div style="margin-left:3rem">
    <img src="./images/storage-comparison.png" alt="storage-comparison" width="500" />
</div>

|                 | 块存储                           | 文件存储                                | 对象存储                       |
|-----------------|----------------------------------|-----------------------------------------|--------------------------------|
| 可变内容        | Y                                | Y                                       | N（支持对象版本控制）          |
| 成本            | 高                               | 中到高                                  | 低                             |
| 性能            | 中到高，非常高                   | 中到高                                  | 低到中                         |
| 一致性          | 强一致性                         | 强一致性                                | 强一致性[5]                    |
| 数据访问        | SAS/iSCSI/FC                     | 标准文件访问，CIFS/SMB和NFS             | RESTful API                    |
| 可扩展性        | 中等可扩展性                     | 高可扩展性                              | 海量可扩展性                   |
| 适用于          | 虚拟机（VM）、数据库             | 通用文件系统访问                        | 二进制数据、非结构化数据       |

一些与对象存储相关的术语：
- **桶（Bucket）** - 对象的逻辑容器。名称全局唯一。
- **对象** - 存储在桶中的单个数据片段。包含对象数据和元数据。
- **版本控制** - 在同一桶中保留对象多个变体的功能。
- **统一资源标识符（URI）** - 每个资源都由一个URI唯一标识。
- **服务等级协议（SLA）** - 服务提供方与客户之间的契约。

Amazon S3 Standard-Infrequent Access存储类的SLA：
- 跨多个可用区的持久性为99.999999999%
- 即使整个可用区被摧毁，数据仍具有韧性
- 设计可用性为99.9%

---

## 第1步：理解问题并确定设计范围

- C：应包含哪些功能？
- I：桶创建、对象上传/下载、版本控制、列出桶中的对象
- C：典型的数据大小是多少？
- I：我们需要同时高效地存储超大对象和小对象
- C：我们一年存储多少数据？
- I：100PB
- C：我们可以假设数据持久性为6个九（99.9999%）、服务可用性为4个九（99.99%）吗？
- I：可以，听起来合理

### **非功能性需求**

- **100PB的数据**
- **数据持久性6个九**
- **服务可用性4个九**
- 存储效率。在保持高可靠性和高性能的同时降低存储成本

### **粗略估算**

对象存储的瓶颈很可能在磁盘容量或每秒IO数（IOPS）上。

假设：
- 我们有20%的小对象（小于1mb）、60%的中等对象（1-64mb）和20%的大对象（大于64mb），
- 一块硬盘（SATA，7200rpm）每秒能完成100-150次随机寻道（100-150 IOPS）

基于这些假设，我们可以估算系统能够持久存储的对象总数。
- 为简化计算，我们使用每种对象类型的中位数大小——小对象0.5mb、中等对象32mb、大对象200mb。
- 在100PB存储（10^11 MB）且存储使用率为40%的情况下，可得到6.8亿个对象
- 如果假设元数据为1kb，那么我们需要0.68tb的空间来存储元数据信息

---

## 第2步：提出高层设计并获得认可

在深入设计之前，我们先探讨对象存储的一些有趣特性：
- **对象不可变性** - 对象存储中的对象是不可变的（其他存储系统并非如此）。我们可以删除或替换它们，但不能更新。
- **键值存储** - 对象的URI就是它的键，我们可以通过发起HTTP调用来获取其内容
- **一次写入，多次读取** - 数据访问模式是一次写入、多次读取。根据Linkedin的一些研究，95%的操作是读操作
- 同时支持小对象和大对象

对象存储的设计哲学与UNIX类似——当我们保存文件时，它会在一种称为inode的数据结构中创建文件名，而文件数据则存储在不同的磁盘位置。
inode包含一个文件块指针列表，这些指针指向磁盘上的不同位置。

访问文件时，我们先从inode获取其元数据，然后再获取文件内容。

对象存储的工作原理类似——元数据存储用于保存文件信息，而内容存储在磁盘上：

<div style="margin-left:3rem">
    <img src="./images/object-store-vs-unix.png" alt="object-store-vs-unix" width="500" />
</div>

通过将元数据与文件内容分离，我们可以独立地扩展不同的存储：

<div style="margin-left:3rem">
    <img src="./images/bucket-and-object.png" alt="bucket-and-object" width="500" />
</div>

### **高层设计**

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="high-level-design" width="500" />
</div>

- **负载均衡器** - 将API请求分发到服务副本上
- **API服务** - 无状态服务器，编排对元数据存储、对象存储以及IAM服务的调用。
- **身份与访问管理（IAM）** - 认证、授权、访问控制的中心位置。
- **数据存储** - 存储和检索实际数据。操作基于对象ID（UUID）。
- **元数据存储** - 存储对象元数据

### **上传对象**

<div style="margin-left:3rem">
    <img src="./images/uploading-object.png" alt="uploading-object" width="500" />
</div>

- 通过HTTP PUT请求创建一个名为“bucket-to-share”的桶
- API服务调用IAM以确保用户已获授权并具有写权限
- API服务调用元数据存储以创建桶条目。创建完成后，返回成功响应。
- 桶创建完成后，发送HTTP PUT请求创建一个名为“script.txt”的对象
- API服务验证用户身份并确保用户具有写权限
- 验证通过后，对象负载通过HTTP PUT发送到数据存储。数据存储将其持久化并返回一个UUID。
- API服务调用元数据存储，创建包含object_id、bucket_id和bucket_name以及其他元数据的新条目。

对象上传请求示例：

```
PUT /bucket-to-share/script.txt HTTP/1.1
Host: foo.s3example.org
Date: Sun, 12 Sept 2021 17:51:00 GMT
Authorization: authorization string
Content-Type: text/plain
Content-Length: 4567
x-amz-meta-author: Alex

[4567 bytes of object data]
```

### **下载对象**

桶没有目录层级，但我们可以通过拼接桶名和对象名来创建逻辑层级，以模拟文件夹结构。

获取对象的GET请求示例：

```
GET /bucket-to-share/script.txt HTTP/1.1
Host: foo.s3example.org
Date: Sun, 12 Sept 2021 18:30:01 GMT
Authorization: authorization string
```

<div style="margin-left:3rem">
    <img src="./images/download-object.png" alt="download-object" width="500" />
</div>

- 客户端向负载均衡器发送HTTP GET请求，即`GET /bucket-to-share/script.txt`
- API服务查询IAM以验证用户具有读取该桶的正确权限
- 验证通过后，从元数据存储中获取对象的UUID
- 根据UUID从数据存储中获取对象负载并返回给客户端

---

// sprint 1

## 第3步：设计深入探讨

### **数据存储**

以下是API服务与数据存储的交互方式：

<div style="margin-left:3rem">
    <img src="./images/data-store-interactions.png" alt="data-store-interactions" width="500" />
</div>

数据存储的主要组件：

<div style="margin-left:3rem">
    <img src="./images/data-store-main-components.png" alt="data-store-main-components" width="500" />
</div>

数据路由服务提供RESTful或gRPC API来访问数据节点集群。
它是一个无状态服务，通过增加更多服务器来扩展。

它的主要职责是：
- 查询放置服务，以获得存储数据的最佳数据节点
- 从数据节点读取数据并返回给API服务
- 将数据写入数据节点

放置服务决定哪些数据节点应存储某个对象。
它维护一张虚拟集群映射，用于确定集群的物理拓扑。

<div style="margin-left:3rem">
    <img src="./images/virtual-cluster-map.png" alt="virtual-cluster-map" width="500" />
</div>

该服务还向所有数据节点发送心跳，以判断它们是否应被从虚拟集群中移除。

由于这是一个关键服务，建议维护一个由5或7个副本组成的集群，并通过Paxos或Raft共识算法进行同步。
例如，一个7节点集群可以容忍3个节点故障。

数据节点存储实际的对象数据。
通过将数据复制到多个数据节点来确保可靠性和持久性。

每个数据节点上运行着一个守护进程，它向放置服务发送心跳。

心跳包含：
- 该数据节点管理多少块磁盘驱动器（HDD或SSD）？
- 每块驱动器上存储了多少数据？

#### 数据持久化流程

<div style="margin-left:3rem">
    <img src="./images/data-persistence-flow.png" alt="data-persistence-flow" width="500" />
</div>

- API服务将对象数据转发到数据存储
- 数据路由服务将数据发送到主数据节点
- 主数据节点将数据保存在本地，并复制到两个从数据节点。复制成功后发送响应。
- 对象的UUID返回给API服务。

注意事项：
- 给定一个对象UUID，其复制组通过一致性哈希确定性地选出
- 在步骤4中，主数据节点在返回响应之前先复制对象数据。这是以更高延迟为代价换取强一致性。

<div style="margin-left:3rem">
    <img src="./images/consistency-vs-latency.png" alt="consistency-vs-latency" width="500" />
</div>

#### 数据如何组织

管理数据的一种简单方法是将每个对象存储在一个单独的文件中。

这可行，但在文件系统中存在大量小文件时性能不佳：
- HDD上的数据块被浪费，因为每个文件都占用整个块的大小。典型的块大小为4kb。
- 大量文件意味着大量inode。操作系统处理过多inode的效果不佳，而且inode还有上限。

这些问题可以通过预写日志（WAL）将许多小文件合并成更大的文件来解决。一旦文件达到其容量上限（通常是几GB），就会创建一个新文件：

<div style="margin-left:3rem">
    <img src="./images/wal-optimization.png" alt="wal-optimization" width="500" />
</div>

这种方法的缺点是，对文件的写访问需要串行化。访问同一个文件的多个核心必须相互等待。
为解决这个问题，我们可以将文件限定到特定的核心，以避免锁竞争。

#### 对象查找

为了支持在同一个文件中存储多个对象，我们需要维护一张表，它告诉数据节点：
- `object_id`
- 对象存储所在的`filename`
- 对象起始位置的`file_offset`
- `object_size`

我们可以将这张表部署在像RocksDB这样的基于文件的数据库中，或传统的关系型数据库中。
由于访问模式是低写入+高读取，关系型数据库效果更好。

我们应该如何部署它？
我们可以将数据库部署在一个集群中并单独扩展，由所有数据节点访问。

缺点：
- 我们需要激进地扩展集群以服务所有请求
- 数据节点与数据库集群之间存在额外的网络延迟

另一种方案是利用这样一个事实：数据节点只关心与自身相关的数据，
因此我们可以将关系型数据库部署在数据节点内部。

SQLite是一个不错的选择，因为它是一个轻量级的基于文件的关系型数据库。

#### 更新后的数据持久化流程

<div style="margin-left:3rem">
    <img src="./images/updated-data-persistence-flow.png" alt="updated-data-persistence-flow" width="500" />
</div>

- API服务发送保存新对象的请求
- 数据节点服务将新对象追加到一个名为“/data/c”的文件末尾
- 该对象的新记录被插入到对象映射表中

#### 持久性

数据持久性是我们设计中的一个重要需求。为了实现6个九的持久性，需要仔细检查每一种故障情况。

首先要解决的问题是硬件故障。我们可以通过复制数据节点来最小化故障概率。
但除此之外，我们还应当跨不同故障域进行复制（跨机架、跨数据中心、独立网络等）。
一个关键事件可能导致同一故障域内的多个硬件故障：

<div style="margin-left:3rem">
    <img src="./images/failure-domain-isolation.png" alt="failure-domain-isolation" width="500" />
</div>

假设典型HDD的年故障率为0.81%，制作三份副本就能给我们带来6个九的持久性。

这样复制数据节点能带来我们想要的持久性，但我们也可以利用纠删码来降低存储成本。

纠删码使我们能够使用校验位，从而在发生故障时重建丢失的数据位：

<div style="margin-left:3rem">
    <img src="./images/erasure-coding.png" alt="erasure-coding" width="500" />
</div>

假设这些位就是数据节点。如果其中两个宕机，它们可以用剩下的四个恢复。

存在不同的纠删码方案。在我们的场景中，可以使用8+4纠删码，并将其拆分到不同的故障域中以最大化可靠性：

<div style="margin-left:3rem">
    <img src="./images/erasure-coding-across-failure-domains.png" alt="erasure-coding-across-failure-domains" width="500" />
</div>

纠删码使我们能够实现低得多的存储成本（约降低50%），代价是访问速度，因为数据路由服务必须从多个位置收集数据：

<div style="margin-left:3rem">
    <img src="./images/erasure-coding-vs-replication.png" alt="erasure-coding-vs-replication" width="500" />
</div>

其他注意事项：
- 复制需要200%的存储开销（在3副本情况下），而纠删码为50%
- 纠删码[能带来11个九的持久性](https://github.com/Backblaze/erasure-coding-durability)，而复制只有6个九
- 纠删码需要更多计算来生成和存储校验块

总之，复制对延迟敏感的应用更有用，而纠删码在存储成本效率和持久性方面更有吸引力。
纠删码的实现难度也大得多。

#### 正确性校验

如果磁盘完全故障，那么故障很容易检测。但如果磁盘内存的一部分被损坏，情况就不那么直接了。

为了检测这种情况，我们可以使用校验和——即文件内容的哈希，可用于验证文件的完整性。

在我们的场景中，我们会为每个文件和每个对象存储校验和：

<div style="margin-left:3rem">
    <img src="./images/checksums-for-correctness.png" alt="checksums-for-correctness" width="500" />
</div>

在纠删码（8+4）的情况下，我们需要分别获取8份数据中的每一份，并验证它们各自的校验和。

// sprint 2

### **元数据数据模型**

表结构：

<div style="margin-left:3rem">
    <img src="./images/metadata-data-model.png" alt="metadata-data-model" width="500" />
</div>

我们需要支持的查询：
- 按名称查找对象ID
- 基于名称插入/删除对象
- 列出桶中共享同一前缀的对象

用户可创建的桶数量通常有限制，因此桶表的规模很小，可以放进单个数据库服务器。
但我们仍需要为该服务器的读吞吐量进行扩展。

不过，对象表可能无法放进单个数据库服务器。因此，我们可以通过分片来扩展该表：
- 按bucket_id分片会导致热点问题，因为一个桶可能有数十亿个对象
- 按bucket_id分片会使负载分布更均匀，但我们的查询会很慢
- 我们选择按`hash(bucket_name, object_name)`分片，因为大多数查询都基于对象/桶名称。

不过，即使采用这种分片方案，列出桶中的对象也会很慢。

### **列出桶中的对象**

在单个数据库中，基于前缀（看起来像目录）列出对象是这样工作的：

```
SELECT * FROM object WHERE bucket_id = "123" AND object_name LIKE `abc/%`
```

当数据库被分片后，这很难实现。为此，我们可以在每个分片上运行该查询，并在内存中聚合结果。
不过，这使得分页变得困难，因为不同分片包含的结果数量不同，我们需要为每个分片单独维护limit/offset。

我们可以利用这样一个事实：对象存储通常并不针对列出对象进行优化，因此我们可以牺牲列出的性能。
我们也可以创建一个反规范化的表用于列出对象，按桶ID分片。
由于该查询被隔离在单个数据库实例中，这将使我们的列出查询足够快。

### **对象版本控制**

版本控制的工作方式是增加另一个`object_version`列，其类型为TIMEUUID，使我们能够基于它排序记录。

每个新版本都会产生一个新的`object_id`：

<div style="margin-left:3rem">
    <img src="./images/object-versioning.png" alt="object-versioning" width="500" />
</div>

删除对象会创建一个带有特殊`object_id`的新版本，表示该对象已被删除。对它的查询会返回404：

<div style="margin-left:3rem">
    <img src="./images/deleting-versioned-object.png" alt="deleting-versioned-object" width="500" />
</div>

### **优化大文件的上传**

大文件上传可以通过分片上传（multipart upload）来优化——将大文件拆分为若干块，分别独立上传：

<div style="margin-left:3rem">
    <img src="./images/multipart-upload.png" alt="multipart-upload" width="500" />
</div>

- 客户端调用服务以发起分片上传
- 数据存储返回一个唯一标识该次上传的upload ID
- 客户端将大文件拆分为若干块，使用upload id分别独立上传
- 当某个块上传完成后，数据存储返回一个etag，它是一个md5校验和，用于标识该上传块
- 所有分片上传完成后，客户端发送完成分片上传的请求，其中包含upload_id、分片编号和所有etag
- 数据存储从各分片重新组装出该对象。这个过程可能需要几分钟。之后，向客户端返回成功响应。

此时可以移除不再有用的旧分片。我们可以引入垃圾回收器来处理这件事。

### **垃圾回收**

垃圾回收是回收不再使用的存储空间的过程。数据变成垃圾有几种方式：
- **惰性对象删除** - 对象被标记为已删除，但实际上并未被删除
- **孤儿数据** - 例如上传中途失败，旧分片需要被删除
- **损坏的数据** - 未通过校验和验证的数据

垃圾回收器还负责回收副本中的未使用空间。
在复制模式下，数据会同时从主节点和副本中删除。在纠删码（8+4）模式下，数据会从全部12个节点上删除。

为便于删除，我们将使用一种称为压缩（compaction）的过程：
- 垃圾回收器将未被删除的对象从“data/b”复制到“data/d”
- 复制完成后，使用数据库事务更新`object_mapping`表
- 为避免产生过多小文件，压缩只对超过一定阈值的文件执行

<div style="margin-left:3rem">
    <img src="./images/compaction.png" alt="compaction" width="500" />
</div>

---

## 第4步：总结

我们涵盖的内容：
- 设计一个类S3的对象存储
- 比较对象存储、块存储和文件存储之间的差异
- 涵盖了桶中对象的上传、下载、列出和版本控制
- 深入探讨了设计——数据存储和元数据存储、复制和纠删码、分片上传（multipart upload）、分片（sharding）
