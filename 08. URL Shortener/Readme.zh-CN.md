# 第 8 章：设计一个URL短链接

## 简介
本章讨论类似TinyURL的URL短链接服务的设计。该系统的主要目标包括**URL缩短**、**重定向**以及**高可扩展性**，以应对巨大的流量规模。

### 需求
- 缩短后的URL必须**唯一**且**尽可能短**。
- 每天处理**1亿次URL生成**，并具备10年的支撑能力。
- 以10:1的读写比支持**高效的读操作**。
- 存储3650亿条记录，10年内大约需要**365TB**的存储空间。

---

## 步骤1：高层设计

### API端点
1. **URL缩短：**  
   - 端点：`POST api/v1/data/shorten`  
   - 参数：`{longUrl: longURLString}`  
   - 返回：`shortURL`

2. **URL重定向：**  
   - 端点：`GET api/v1/shortUrl`  
   - 返回`longURL`用于重定向。

    <p align="center">
    <img src="./images/url-redirection.png" alt="URL重定向" width="600">
    </p>

### URL重定向
- **301重定向：**  301重定向表示所请求的URL被“永久”移动到长URL。浏览器会缓存该响应，并且
对同一URL的后续请求将不再发送到URL短链接服务。
- **302重定向：** 临时重定向；适用于跟踪点击等分析场景。

### URL缩短
<p align="center">
    <img src="./images/url-shortening.png" alt="URL缩短" width="400">
</p>

- 使用**哈希函数**生成短URL，将长URL映射为唯一的缩短版本。
- 哈希函数必须满足以下要求：
    - 每个longURL必须被哈希为一个hashValue。
    - 每个hashValue都能映射回对应的longURL。
    

---

## 步骤2：深入设计

### 数据模型
将`<shortURL, longURL>`映射存储在关系型数据库中，以优化内存使用。表结构包括：
- `id`（主键），
- `shortURL`，
- `longURL`。

    <img src="./images/table-schema.png" alt="表结构" width="300">

### 哈希函数
#### 1. Base 62转换：
- 使用字符`[0-9, a-z, A-Z]`对数字进行编码，提供**62种可能的字符**。
- 进制转换是URL短链接常用的另一种方法。
- 可以为短URL分配一个唯一ID，再将ID做Base 62转换得到短URL。
- 7个字符的哈希最多可支持**3.5万亿个唯一URL**，足以覆盖3650亿个URL。

**示例：**  
将ID `2009215674938`转换为Base 62：
- `2009215674938` → `zn9edcu`。

#### 2. 哈希+冲突解决：
- 使用CRC32、MD5或SHA-1等哈希函数。

    <img src="./images/hash-function.png" alt="哈希函数" width="500">

- 一种做法是取哈希值的前7个字符；然而这种方法可能导致哈希冲突。
- 为解决冲突，递归地追加一个新的预定义字符串，直到不再有冲突，但这样做的代价可能很高。
- 使用**布隆过滤器**解决冲突，以实现高效查找。

    <p align="center">
    <img src="./images/url-lookup.png" alt="URL查找" width="500">
    </p>

### 对比

-  **哈希+冲突解决：**
    - 短URL长度固定
    - 不需要唯一ID生成器
    - 可能发生冲突，需要解决
    - 无法找到下一个可用的短URL，因为它不依赖ID

- **Base 62转换**
    - 长度不固定，会随ID增长
    - 需要唯一ID生成器
    - 不会发生冲突
    - 若ID每次递增1，则很容易找到下一个短URL（可能带来安全问题）


---

### URL缩短流程

<p align="center">
    <img src="./images/url-shortening-flow.png" alt="URL缩短" width="500">
</p>

1. 检查`longURL`是否已存在于数据库中。
2. 如果存在，返回已有的`shortURL`。
3. 否则：
   - 使用**分布式ID生成器**生成一个唯一ID。
   - 使用Base 62将ID转换为`shortURL`。
   - 将`<id, shortURL, longURL>`映射存储到数据库中。



---

### URL重定向流程
<p align="center">
    <img src="./images/url-redirecting-flow.png" alt="URL缩短" width="600">
</p>

1. 用户点击一个`shortURL`。
2. 查询`<shortURL, longURL>`映射：
   - 先检查**缓存**以加快访问。
   - 如果缓存中没有，则查询数据库。
3. 将用户重定向到`longURL`。


---

## 其他考虑
### 限流器
- 通过限制每个IP的请求数来防止滥用。

### 可扩展性
1. **Web层：** 无状态，通过增加/移除Web服务器实现扩展。
2. **数据库层：** 使用复制和分片。

### 分析
- 收集点击率、来源和时间戳等数据，用于业务洞察。

### 高可用与可靠性
- 通过数据库复制和容错设计确保服务的一致性与可靠性。

