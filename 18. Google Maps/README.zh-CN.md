# 第 18 章：Google Maps

## 引言

我们将设计一个简化版的 **Google Maps**。

关于 Google Maps 的一些事实：
 * 始于 2005 年
 * 提供多种服务——卫星影像、街道地图、实时路况、路线规划
 * 到 2021 年，日活跃用户达 10 亿，覆盖全球 99%，每天有 2500 万次实时位置信息更新

---

## 第 1 步：理解问题并确立设计范围

候选人与面试官之间的示例问答：
 * C: 我们要应对多少日活跃用户？
 * I: 10 亿 DAU
 * C: 我们应该聚焦哪些功能？
 * I: 位置更新、导航、ETA、地图渲染
 * C: 道路数据有多大？我们能获取到吗？
 * I: 我们从多个来源获取了道路数据，原始数据有 TB 级
 * C: 我们应该考虑路况吗？
 * I: 是的，为了准确的时间估算，我们必须考虑
 * C: 不同的出行方式呢——步行、骑行、驾车？
 * I: 我们应该支持这些
 * C: 多途经点的导航呢？
 * I: 在本次面试范围内我们先不聚焦这个
 * C: 商业地点和照片呢？
 * I: 好问题，但无需考虑这些

我们将聚焦三个关键功能——用户位置更新、包含 ETA 的导航服务、地图渲染。

### **非功能性需求**

- **准确性**：用户不应得到错误的路线指引
- **流畅的导航**：用户应获得流畅的地图渲染体验
- **数据与电量消耗**：客户端应尽可能少用数据和电量。这对移动设备很重要。
- 通用的可用性与可扩展性要求

### **地图基础**

在进入设计之前，有一些与地图相关的概念我们应该理解。

#### 定位系统

地球是一个球体，绕其轴自转。位置由纬度（你向北/向南多远）和经度（你向东/向西多远）来定义：

<div style="margin-left:3rem">
    <img src="./images/partitioning-system.png" alt="partitioning-system" width="500" />
</div>

#### 从 3D 到 2D

将点从 3D 转换到 2D 平面的过程称为“地图投影”。

有不同的实现方式，每种方式都有其优缺点。几乎所有的投影都会扭曲真实的几何形状。

<div style="margin-left:3rem">
    <img src="./images/map-projections.png" alt="map-projections" width="500" />
</div>

Google Maps 选择了一种经过修改的墨卡托投影，称为“Web Mercator”。

#### 地理编码

地理编码是将地址转换为地理坐标的过程。

反向的过程称为“反向地理编码”。

实现这一点的一种方式是使用插值——利用来自不同数据源（例如各种 GIS）的数据，其中街道网络被映射到地理坐标空间。

#### 地理哈希（geohash）

地理哈希（geohash）是一种编码系统，它把一个地理区域编码成一串字母和数字。

它把世界描绘成一个展平的表面，并递归地将其细分为四个象限：

<div style="margin-left:3rem">
    <img src="./images/geohashing.png" alt="geohashing" width="500" />
</div>

#### 地图渲染

地图渲染通过瓦片化实现。世界不会被渲染成一整张巨大的自定义图像，而是被切分成更小的瓦片。

客户端只下载相关的瓦片，并像拼接马赛克一样将它们渲染出来。

不同的缩放级别有不同的瓦片。客户端根据自身的缩放级别选择合适的瓦片。

例如，把整个世界缩到最小只会下载一张 256x256 的瓦片，它代表整个世界。

#### 为导航算法处理道路数据

在大多数路径规划算法中，路口被表示为节点，道路被表示为边：

<div style="margin-left:3rem">
    <img src="./images/road-representation.png" alt="road-representation" width="500" />
</div>

大多数导航算法使用 Dijkstra 或 A* 算法的修改版本。

寻路性能对图的规模很敏感。要想在如此规模下工作，我们无法把整个世界表示成一张图并在其上运行算法。

相反，我们使用一种类似瓦片化的技术——把世界细分成越来越小的图。

路径规划瓦片保存着对相邻瓦片的引用，算法在遍历相互连接的瓦片时可以拼接出一张更大的道路图：

<div style="margin-left:3rem">
    <img src="./images/routing-tiles.png" alt="routing-tiles" width="500" />
</div>

这种技术使我们能够显著减少内存带宽，只加载给定起点/终点对所需的瓦片。

然而，对于较长的路线，拼接小而精细的路径规划瓦片仍然会耗费时间和内存。因此，存在不同细节层次的路径规划瓦片，算法会根据我们要前往的目的地使用相应细节层次的瓦片：

<div style="margin-left:3rem">
    <img src="./images/map-routing-hierarchical.png" alt="map-routing-hierarchical" width="500" />
</div>

### **粗略估算**

对于存储，我们需要存：
 * 世界地图——根据我们需要存储的所有瓦片估算约为 70pb，但要考虑到对高度相似的瓦片（例如广袤的沙漠）进行压缩
 * 元数据——体积可忽略不计，因此我们可以在计算中跳过它
 * 道路信息——以路径规划瓦片的形式存储

导航请求的估算 QPS——10 亿 DAU，每周使用 35 分钟 -> 每天 50 亿分钟。
假设 GPS 更新请求是批量发送的，我们得出 200k QPS，以及峰值负载下的 100 万 QPS

---

## 第 2 步：提出高层设计并获得认可

<div style="margin-left:3rem">
    <img src="./images/high-level-design.png" alt="high-level-design" width="500" />
</div>

### **位置服务**

<div style="margin-left:3rem">
    <img src="./images/location-service.png" alt="location-service" width="500" />
</div>

它负责记录用户的位置更新：
 * 位置更新每 `t` 秒发送一次
 * 位置数据流可用于随时间为服务带来改进，例如提供更准确的 ETA、监控路况数据、检测封闭道路、分析用户行为等

与其一直把位置更新发送到服务器，我们可以在客户端侧把更新批量聚合后再发送：

<div style="margin-left:3rem">
    <img src="./images/location-update-batches.png" alt="location-update-batches" width="500" />
</div>

尽管有了这项优化，对于 Google Maps 这样规模的系统，负载仍然很大。因此，我们可以利用一个针对大量写入做过优化的数据库，例如 Cassandra。

我们还可以利用 Kafka 来高效地对位置更新进行流式处理，以供进一步分析。

示例位置更新请求负载：

```
POST /v1/locations
Parameters
  locs: JSON encoded array of (latitude, longitude, timestamp) tuples.
```

### **导航服务**

该组件负责在合理的时间内（有一点延迟是可以的）找到 A 与 B 之间的快速路线。路线不必是最快的，但准确性很重要。

示例请求负载：

```
GET /v1/nav?origin=1355+market+street,SF&destination=Disneyland
```

示例响应：

```json
{
  "distance": {"text":"0.2 mi", "value": 259},
  "duration": {"text": "1 min", "value": 83},
  "end_location": {"lat": 37.4038943, "Ing": -121.9410454},
  "html_instructions": "Head <b>northeast</b> on <b>Brandon St</b> toward <b>Lumin Way</b><div style=\"font-size:0.9em\">Restricted usage road</div>",
  "polyline": {"points": "_fhcFjbhgVuAwDsCal"},
  "start_location": {"lat": 37.4027165, "lng": -121.9435809},
  "geocoded_waypoints": [
    {
       "geocoder_status" : "OK",
       "partial_match" : true,
       "place_id" : "ChIJwZNMti1fawwRO2aVVVX2yKg",
       "types" : [ "locality", "political" ]
    },
    {
       "geocoder_status" : "OK",
       "partial_match" : true,
       "place_id" : "ChIJ3aPgQGtXawwRLYeiBMUi7bM",
       "types" : [ "locality", "political" ]
    }
  ],
  "travel_mode": "DRIVING"
}
```

路况变化和重新规划路线尚未纳入考虑，这些将在深入设计部分处理。

### **地图渲染**

在客户端侧保存整个地图瓦片数据集是不可行的，因为它的体积达到 PB 级。

这些瓦片需要根据客户端的位置和缩放级别按需从服务器获取。

何时应该获取新的瓦片——当用户放大/缩小时，以及导航过程中当他们驶向新的瓦片时。

地图瓦片应如何下发给客户端？
 * 它们可以动态构建，但这会给服务器带来巨大负载，也使缓存变得困难
 * 地图瓦片根据其 geohash 以静态方式下发，客户端可以自行计算 geohash。它们可以静态存储并由 CDN 下发

<div style="margin-left:3rem">
    <img src="./images/static-map-tiles.png" alt="static-map-tiles" width="500" />
</div>

CDN 使用户能够从距离用户最近的接入点服务器（POP）获取地图瓦片，从而将延迟降到最低：

<div style="margin-left:3rem">
    <img src="./images/cdn-vs-no-cdn.png" alt="cdn-vs-no-cdn" width="500" />
</div>

在确定地图瓦片时，可以考虑以下选项：
 * 地图瓦片的 geohash 可以在客户端侧计算。如果是这样，我们应谨慎，因为我们要在长期内坚持这种地图瓦片计算方式，而强制客户端升级是很难的
 * 或者，我们可以提供一个简单的 API，代替客户端计算地图瓦片 URL，代价是额外的一次 API 调用

<div style="margin-left:3rem">
    <img src="./images/map-tile-url-calculation.png" alt="map-tile-url-calculation" width="500" />
</div>

---

## 第 3 步：深入设计

### **数据模型**

我们来讨论如何存储我们要处理的不同类型的数据。

#### 路径规划瓦片

初始道路数据集来自不同的数据源。它会基于位置更新数据随时间得到改进。

道路数据是非结构化的。我们有一个周期性的离线处理流水线，它把这份原始数据转换成应用所需的基于图的路径规划瓦片。

我们不会把这些瓦片存到数据库里，因为我们不需要任何数据库特性。我们可以把它们存储在 S3 对象存储中，同时对它们进行激进缓存。

我们还可以利用各种库把邻接表高效地压缩成二进制文件。

#### 用户位置数据

用户位置数据对于更新路况以及进行各种其他分析非常有用。

我们可以使用 Cassandra 存储这类数据，因为它本质上是写密集的。

示例行：

<div style="margin-left:3rem">
    <img src="./images/user-location-data-torw.png" alt="user-location-data-row" width="500" />
</div>

#### 地理编码数据库

该数据库存储经纬度对与地点之间的键值对。

我们可以使用 Redis，因为它读取速度很快，而我们的场景是频繁读、不频繁写。

#### 预计算的世界地图图像

正如我们所讨论的，我们会预先计算地图瓦片图像并把它们存储在 CDN 中。

<div style="margin-left:3rem">
    <img src="./images/precomputed-map-tile-image.png" alt="precomputed-map-tile-image" width="500" />
</div>

### **服务**

#### 位置服务

我们详细看一下该服务的数据库设计以及用户位置是如何存储的。

<div style="margin-left:3rem">
    <img src="./images/location-service-diagram.png" alt="location-service-diagram" width="500" />
</div>

我们可以使用 NoSQL 数据库来承载位置更新带来的大量写入负载。我们优先考虑可用性而非一致性，因为用户位置数据经常变化，并会随着新更新的到来而变得过时。

我们将选择 Cassandra 作为数据库选型，因为它完美契合我们的所有需求。

我们要存储的示例行：

<div style="margin-left:3rem">
    <img src="./images/user-location-row-example.png" alt="user-location-row-example" width="500" />
</div>

 * `user_id` 是分区键，以便快速访问某个用户的所有位置更新
 * `timestamp` 是聚簇键，以便按位置更新被接收的时间排序存储数据

我们还利用 Kafka 把位置更新以流的方式发送给各种其他服务，这些服务出于不同目的需要位置更新：

<div style="margin-left:3rem">
    <img src="./images/location-update-streaming.png" alt="location-update-streaming" width="500" />
</div>

#### 渲染地图

地图瓦片以各种缩放级别存储。在最低缩放级别，整个世界由一张 256x256 的瓦片表示。

随着缩放级别提高，地图瓦片的数量会变为四倍：

<div style="margin-left:3rem">
    <img src="./images/zoom-level-increases.png" alt="zoom-level-increases" width="500" />
</div>

我们可以使用的一项优化是：不在网络上发送完整的图像信息，而是把瓦片表示成矢量（路径和多边形），让客户端动态渲染这些瓦片。

这将带来可观的带宽节省。

#### 导航服务

该服务负责寻找最快的路线：

<div style="margin-left:3rem">
    <img src="./images/navigation-service.png" alt="navigation-service" width="500" />
</div>

我们逐一来看该子系统中的每个组件。

首先，我们有地理编码服务，它把一个地址解析为一个经纬度对位置。

示例请求：

```
https://maps.googleapis.com/maps/api/geocode/json?address=1600+Amphitheatre+Parkway,+Mountain+View,+CA
```

示例响应：

```json
{
   "results" : [
      {
         "formatted_address" : "1600 Amphitheatre Parkway, Mountain View, CA 94043, USA",
         "geometry" : {
            "location" : {
               "lat" : 37.4224764,
               "lng" : -122.0842499
            },
            "location_type" : "ROOFTOP",
            "viewport" : {
               "northeast" : {
                  "lat" : 37.4238253802915,
                  "lng" : -122.0829009197085
               },
               "southwest" : {
                  "lat" : 37.4211274197085,
                  "lng" : -122.0855988802915
               }
            }
         },
         "place_id" : "ChIJ2eUgeAK6j4ARbn5u_wAGqWA",
         "plus_code": {
            "compound_code": "CWC8+W5 Mountain View, California, United States",
            "global_code": "849VCWC8+W5"
         },
         "types" : [ "street_address" ]
      }
   ],
   "status" : "OK"
}
```

路线规划服务计算出一条建议路线，并根据当前路况针对行驶时间进行优化。

最短路径服务针对对象存储中的路径规划瓦片运行 A* 算法的一个变体，以计算出最优路径：
 * 它接收起点/终点对，把它们转换成经纬度对，再根据这些对推导出 geohash，从而得到路径规划瓦片
 * 算法从起始的路径规划瓦片开始，开始遍历它，直到找到一条通往目的瓦片的足够好的路径

<div style="margin-left:3rem">
    <img src="./images/shortest-path-service.png" alt="shortest-path-service" width="500" />
</div>

ETA 服务由路线规划服务调用，基于机器学习算法获取时间估算，根据路况数据预测 ETA。

排序服务负责根据用户传入的筛选条件对不同可能路径进行排序，即避开收费道路或高速公路的标志。

更新服务异步更新一些重要的数据库，使它们保持最新。

#### 改进——自适应 ETA 与重新规划路线

我们可以做的一项改进是，根据新获得的路况数据自适应地更新进行中的路线。

实现这一点的一种方式是把当前正在沿某条路线导航的用户存储到数据库中，存储他们应该经过的所有瓦片。

数据可能看起来像这样：

```
user_1: r_1, r_2, r_3, …, r_k
user_2: r_4, r_6, r_9, …, r_n
user_3: r_2, r_8, r_9, …, r_m
...
user_n: r_2, r_10, r21, ..., r_l
```

如果某个瓦片上发生了交通事故，我们可以找出路径经过该瓦片的所有用户，并为他们重新规划路线。

为了减少数据库中存储的瓦片数量，我们可以改为存储起始路径规划瓦片以及不同分辨率层级上的若干路径规划瓦片，直到目的瓦片也被包含在内：

```
user_1, r_1, super(r_1), super(super(r_1)), ...
```

<div style="margin-left:3rem">
    <img src="./images/adaptive-eta-data-storage.png" alt="adaptive-eta-data-storage" width="500" />
</div>

利用这一点，我们只需检查某个用户的最终瓦片是否包含事故瓦片，就能判断该用户是否受到影响。

我们还可以跟踪正在导航的用户的所有可能路线，并在存在更快的替代路线时通知他们。

#### 下发协议

我们有几个选项，可以让我们从服务器主动向客户端推送数据：
 * 移动端推送通知不可行，因为负载有限，而且 Web 应用无法使用
 * WebSocket 通常比长轮询更好的选择，因为它对服务器造成的计算开销更小
 * 我们也可以使用服务器发送事件（SSE），但更倾向于 WebSocket，因为它支持双向通信，这在例如最后一公里配送功能中会很有用

---

## 第 4 步：总结

这是我们的最终设计：

<div style="margin-left:3rem">
    <img src="./images/final-design.png" alt="final-design" width="500" />
</div>

我们还可以提供一项额外功能：多途经点导航，它可以卖给 Uber 或 Lyft 这样的企业客户，用于确定访问一组地点的最优路径。
