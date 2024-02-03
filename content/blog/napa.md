+++
title = "Google Napa"
date = "2024-02-03T23:53:45+08:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

# tags = ['Data Warehouse', 'Data Integration']
+++

[Napa](https://research.google/pubs/napa-powering-scalable-data-warehousing-with-robust-query-performance-at-google/) 是 Google 内部的新一代分析型数仓系统，负责将各种 Google 服务持续生成的大量应用数据摄入到数据存储进行持久化，然后对数据维护加工生成 read-optimized 的 table 和 view，供查询分析使用。  
在设计上，Napa 将 data ingestion， data storage， query serving 三个组件完全解耦，这使得 Napa 可以保证查询处理的鲁棒性；同时，这种解耦的设计选择**使得 Napa 能够进行极为精细的资源管理**，让用户能够基于自身需求在 cost、data freshness 以及 query performance 之间做出灵活的权衡。如论文摘要所说：“Robust query processing and flexible configuration of client databases are the hallmark of Napa design”。

## 系统设计：需求与决定
作为一个数据集成分析平台，Napa 的客户最关心的指标通常是「查询性能」和「数据新鲜度」（在数据一致性的前提下），再结合预算考量，Napa 的客户往往需要在「数据新鲜度」、「成本」和「查询性能」之间进行取舍： 

![napa-threeway-tradeoffs](/img/napa/napa-threeway-tradeoffs.png "450px")  

### 面临的挑战
大部分的数据集成平台其实都在某种程度上提供这样的三向权衡，但 Napa 可能在系统设计和工程实现上更优。
Napa 所面临的挑战主要是需求的多样性以及对服务鲁棒性的要求：  

**挑战 1: 需求多样性**
   - 查询响应延迟的要求可能是从毫秒级到分钟级
   - 数据新鲜度要求可能是分钟级到天级别
   - 预算总是有限  
   - 随着业务的变化，需求也会随之发生改变

**挑战 2: 要求查询性能鲁棒**
- 对于交互性的应用来说，查询的鲁棒性对于用户体验至关重要，毕竟没有人想要体验时快时慢的交互体验。因此，确保查询鲁棒性（即查询延迟方差较低）是非常重要的

### 系统设计
Napa 系统在设计上将 Data Ingestion， Storage/Indexing(Data Maintenance) 和 Query Serving 完全解耦:
- 将 Data Ingestion 与 Storage/Indexing 解耦:
    这里的 Data Ingestion 可以理解为更新数据的持久化，Storage/Indexing 可以理解为数据组织优化（提升查询性能）。如果 Data Ingestion 与 Storage/Indexing 耦合，那么 Ingestion 速度会受限于 Storage 工作负载。即实现较好的「查询性能」会牺牲掉「数据新鲜度」
- 将 Storage/Indexing 与 Query Serving 解耦：
    Napa 同时将 Storage/Indexing 与 Query Serving 也做了解耦，以实现数据一致性和更稳定的查询性能。

![system-design-1](/img/napa/napa-system-design-1.png "450px")  
*传统 warehouse 系统的设计*   

---
![system-design-2](/img/napa/napa-system-design-2.png "450px")  
*Napa 系统的设计*

---
在解耦后，Napa 可以用更灵活的工作方式来满足更多样性的需求：
1. Data Ingestion 模块实际上只需要负责将增量数据（delta）持久化，保证数据不丢失即可，Napa 在数据表中使用 LSM 结构组织数据，所以数据更新是 write-optimized 的；
2. Storage 模块负责数据的优化，即 LSM 树的 Compaction，将数据优化到查询性能要求的范围内，同时 Storage 模块也会负责将增量数据更新到物化视图中；
3. 查询延迟取决于 Storage 优化的速度，对于单个数据表来说，LSM 树层级越少，那么查询性能自然更快。Napa 使用 F1 做查询引擎（所以这块其实没什么可讲的，这不属于 Napa 的工作内容）。

## 数据存储与查询优化
如上一节所述，Napa 的主要工作内容在 Data Ingestion 和 Storage/Indexing 上，Napa 数据来源通常是关系型数据库， Ingestion 模块以流的形式连续地将一批一批的更新数据提交到 Napa 的数据表中，然后由 Storage 对数据进行维护。
在这方面 Napa 主要做了三样工作：
1. 基于 LSM 树结构实现数据表和物化视图维护框架；  
2. 引入 Queryable Timestamp(QT) 来保证数据的一致性，以及用来指示 Storage/Indexing 的进度；  
3. 自动生成物化视图来提高查询性能，并且可以通过自动增减物化视图的数量来调节查询性能。

### LSM
Napa 中的数据表和物化视图都会接受大量的更新，因此 Napa 数据存储上使用了 LSM 树结构。LSM 数据结构对写十分友好，并且也利于增量消费。数据表 / 物化视图在存储上被分为若干个单元，每个单元都是一颗 LSM 树。  
如前文所提，Napa 的 Ingestion 和 Storage 是完全解耦的，Ingestion 只负责将更新的数据持久化，持久化的更新数据被称为 deltas，即 LSM 树的 L0。
对于 Query 来说，新写入的 deltas 并不是写入后就立即可查的，因为它们会被归类为 Non-queryable deltas，即不可查询的增量数据。这主要是为了查询性能考虑，LSM 树的查询性能受到 level 的层数的影响，查询时要合并的 delta 越多，查询负载越高。为了满足查询延迟要求，所以新增的 deltas 需要经过 Compaction 后才能真正可查。

新写入的 delta 是不可查的，Napa 用 Queryable Timestamp(QT) 来指示表中可查的数据：
![QT](/img/napa/QT.png "500px")  

### Queryable Timestamp (QT)
Queryable Timestamp 是 Napa 中最有意思的一个概念，表的 QT 是一个时间戳，用于指示可查询（即满足查询性能要求）数据的新鲜度。  一个表的 QT 只有在增量数据被导入且被加工到满足查询要求后才会推进。所以数据表的新鲜度可以表示为 [Now() - QT]。 QT 是控制新鲜度、成本和性能之间的三角关系的关键旋钮。

此外 QT 还被用于维持数据的一致性。  
Napa 中的数据具有强一致性，数据表和视图中的数据具有一致性，同一份数据表在多个数据中心也具有一致性。通常来说我们常见的分布式系统中都使用「事务管理」和「副本同步」来达到这些目的，但这样又会让 Storage/Indexing 中各个表的更新严重耦合。所以 Napa 创新性地用 QT 来指示数据的一致性状态：数据导入和加工操作在每个数据中心 / 数据表 / 视图上异步执行，而元数据操作被定期用于各数据实体之间的状态同步。

每个数据中心的每份数据都有自己的本地 QT，而全局 QT 是根据查询服务的可用性要求从本地 QT 值计算出来的。例如，如果我们有5个Napa副本，本地QT值分别为100、90、83、75、64，并且查询服务需要大多数副本可用，那么所有数据中心的新QT会被设定为83，因为大多数副本至少在83之前是最新的。而一个数据库的 QT 是库中所有表的 QT 的最小值。

奇怪的是，这里没有明确说 QT 是 event time 还是 processing time。我理解只有是 event time 才能有数据一致性的保证，比如数据源可能在一个事务内发生了多个表的更新操作，那么只有用 event time 的 QT 指示一致性才有意义。而如果是 processing time 则似乎没什么意义。
但从文章说明来看这个 QT 似乎好像是个 processing time，不知道是不是我理解有误😅。
> 原文：If QT(table) = X, all data that was ingested into the table before time X can be queried by the client and the data after time X is not part of the query results. In other words, the freshness of a table is [Now() - QT]. QT acts as a barrier such that any data ingested after X is hidden from client queries. The value of QT will advance from X to Y once the data ingested in (Y-X) range has been optimized to meet the query performance requirements.

### 物化视图
Napa 可以自动生成物化视图！  
在查询有聚合函数的情况下，提前聚合的物化视图的数据量会远少于基本数据表，所以添加物化视图可以减少扫描数据量，从而提高查询性能。相比于扫描基本数据表，扫描物化视图还能缓解查询收长尾查询的影响，从而降低查询延迟方差。
物化视图的数量和 delta 的数量都可以被用来调节查询的延迟。
![tune-view-and-delta](/img/napa/tune-view-and-delta.png)

## 松耦合设计带来的灵活性
在 Napa 中，用户对于 query performance， data freshness 和 cost 的需求会被翻译为数据库内部的配置，比如视图数量、数据加工任务的配额、查询时 delta 数量阈值等。得益于系统松耦合的设计，Napa 能够以极大的灵活度来支持不同场景下的数据分析任务：
### Tradeoff freshness
以数据新鲜度为代价，换取更好的查询性能和更低的资源成本
- 维护适量的视图 + 保持较少的 delta 文件 -> 以得到较好的查询性能
- 用便宜的机器来进行数据的加工和维护 -> 保持成本适当

### Tradeoff query performance
以查询性能为代价，换取较好的数据新鲜度和较低的资源成本
- 维护较少的视图 + 允许查询时存在更多的 delta 文件 -> 减少 QT 推进的负载，提高数据新鲜度
- 将相对多的资源分配给 Ingestion 任务，因为 Storage 的工作量比较低 -> 保持成本适当

### Tradeoff costs
以资源为代价，换取数据新鲜度和查询性能
- 充分使用视图来加速查询 + 保持尽可能少的 delta 文件 -> 尽可能降低查询延迟
- 调动充分的资源来保持 QT 的推进 -> 保持数据新鲜度能跟上

## 小结
从系统设计上来看，Napa 与传统的分析性数仓的最大区别是将整个系统拆散成了多个松耦合的“组件”，然后分别提供服务。解耦的设计可以让系统做更精细的资源管理、更易于扩展，同时又能以灵活的方式组合起来满足客户多样化的需求。Napa 可以为客户提供灵活的可配置参数，用于查询性能、新鲜度和成本的权衡。

在具体实现上，和一些依赖列式存储、并行性和压缩来加速扫描的分析系统不同，Napa 更倾向于利用物化视图来保证查询性能；另外 Napa 还巧妙地引入了可查询时间戳（Queryable Timestamp）的概念，QT 可以被用于维护数据一致性，也是控制新鲜度、成本和性能之间的三角关系的关键旋钮。