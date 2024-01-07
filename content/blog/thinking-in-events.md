+++
title = "Thinking in Events: From Databases to Distributed Collaboration Software"
date = "2024-01-07T22:56:10+08:00"
+++

Thinking in Events[1] 是一篇关于「基于事件的分布式系统」的系统概述文章。作者基于其在流处理和实时协作软件两个领域的研究经验，提出了一种事件系统分类法，系统性地总结与归类了不同主题的 event-based systems ，以揭示它们之间的共性和差异，以及其实现背后的一些关键权衡。

## 按照使用方式分类

文章从使用者的角度出发对事件进行分类，一个事件可以被认为是一个通知（notification）或是持久记录（persistent record）：
- a notification: 关于某件事情发生了的通知
- a persistent record：某件事情发生的持久化记录  

当然也可能两者都是。

以事件的使用方式为区分，事件系统可以被分为三类：
- **User interface events, reactive/functional reactive programming**  
    事件只被用作通知，不会被持久化保存，系统则是可能会对事件做出反应。比如大部分的 UI 框架都可以被归于这一类，用户的输入事件会被调度到回调函数或事件处理程序中进行处理，比如说浏览器中处理「点击」或是「键入」事件的 JavaScript 函数。

- **Time-series database, fact table in star schema**  
    事件不一定有通知的元素，系统只需要将接收到的事件记录下来以用作后续的查询、分析使用。比如记录随时间推移发生的事件的时序数据库，或是数据仓库中雪花模型 / 星型模型中用于记录事件的事实表。从用户的角度来看，事件的主要目的是允许对事件历史进行追溯式查询和分析，比如检查趋势并生成显示某些指标随时间变化的报告。

- **Stream processing**  
    如果一个事件既有通知的元素，也会被持久化，那么处理这类事件的系统可以被归类为「Stream processing」，这个分类的名字其实也是十分的模糊。（如作者所说，I'm just going to give that a label of stream processing, for lack of a better term.）这类系统又可以被细分为很多类。

![persistent-or-notification](/img/thinking-in-events/persistent-or-notification.png)

### Stream processing 系统的细分

对于 Stream processing 的分类，文章以是否在流处理中使用窗口作为最重要的区分标准：
- **Stream analytics, CEP**  
    某些流处理系统主要设计用于那些「需要在时间上较为接近的事件进行组合」的应用，比如欺诈检测系统可能需要检测客户信用卡上最近活动的异常模式；
    还有一些流处理系统需要组合「可能在时间上任意远的事件」，最典型的例子就是像 twitter 时间线上推文的回溯处理。
    这两类系统的共同点是需要在某个时间窗口上（有限长/无限长）将一系列的事件聚合然后执行处理并输出。

- **Database replication, materialized views**  
    对于那些既不需要使用 window，也不需要 join，但是会将 event 视为 notification 且将其持久化的流处理系统，文章将其归类到数据库的复制、物化视图更新这一类。
    从事件驱动的角度来说，我们可以将每个「修改数据库的事务」视为「更新事件」，将更新的执行（即在数据库状态中反映更新）视为对该事件的处理，将数据从一台机器复制到另一台机器视为该事件在分布式系统中的传播。

![stream-processing-windowed](/img/thinking-in-events/stream-processing-windowed.png)

### Event log 是否完全有序

Database replication 系统又可以根据 event log 是否完全有序进一步细分：

**Event log 全局有序**  

一般来说主从模型的数据库产生的日志都是完全有序的，因此从节点基于完全有序的日志总是可以得到确定性的结果。对于这类模型，文章根据系统数据主模型的不同进行分类，这也是作者在之前的 Turning the database inside-out[2] 中持有的观点。

  - **Traditional database replication (WAL)**  
  传统数据库将 database state 视为主要模型，其接受并处理命令式数据变更查询，然后出于故障恢复或是数据复制的目的，产生一份数据变更的 data change events，在这样的模型中 event 是系统的副作用。

  - **Event sourcing**  
  反过来也可以将 data change events 作为主要数据模型，这样 database state 就变成了处理这些事件的副作用。这样做的好处是所有系统可以捕捉并记录 data change 的意图和含义，并且可以基于重放 event 来重建、修改状态，而不是仅仅记录状态的变化。
  这种模型的缺点也很明显，那就是 event log 与 database state 之间的转换会比较复杂，event 越灵活，将其转换为结构性的 database state 的复杂度就越大。

**Event log 局部有序**  

但不是所有系统都可以保持日志完全有序，当实现 CAP 中的 Consistent 代价过大时，系统会选择只产生局部有序的日志，这样最多可以做到 casual order。  
日志部分有序的复制形式也被称为 **optimistic replication**。因为不能保证复制节点以相同的顺序处理事件，所以即使用确定性的逻辑对事件进行处理也无法确保产生相同的结果。当然那些状态更新不依赖于事件顺序的应用不受此影响，比如累计求和。  
同样，根据数据主模型的不同，文章将 Event log 局部有序的系统也分为两类：

- **Time warp**  
  通过使用逻辑时间戳，也可以将部分有序的日志进行确定性的全局排序。只要确保每个复制节点最终都可以接收到完整的日志，就可以确保最终一致性，为了达到这一点，在接收到没有遵守顺序的事件时，系统必须能够将其状态回滚到与插入位置对应的时间点，应用新事件，然后重放那些时间戳大于新事件的事件。这种方法被称为 **Time warp**。
![logical-timestamp](/img/thinking-in-events/logical-timestamp.png)

- **Conflict-free replicated data types (CRDTs)**  
如果将 partial order data change events 作为主数据模型，那么这类系统可以被归类为 Conflict-free replicated data types (CRDTs)。通过依赖于可交换性，CRDT 可以确保复制节点收敛到一致的状态，而不要求所有复制节点以相同的顺序处理事件。
CRDTs 的状态变化模型非常适用于用户能够直接操纵状态的应用程序：例如，在线文档编辑时，用户可以在文档的任何位置插入或删除文本。因为不需要进行回滚，CRDTs 通常可以比 Time warp 提供更好的性能。  
CRDTs 的一个缺点是它们仅支持数据类型接口提供的预定义操作，比如 CRDT 允许向 list 中插入或删除元素，但大多数并不支持对 list 元素进行重新排序。相比之下，Time warp 模型更加灵活，其允许使用任何确定性的纯函数来处理事件。
![database-replication](/img/thinking-in-events/database-replication.png)

## 分类全图一览
![a-taxonomy-of-event-based-systems](/img/thinking-in-events/a-taxonomy-of-event-based-systems.png)


## 参考文献

[1] Thinking in Events: From Databases to Distributed Collaboration Software: https://martin.kleppmann.com/2021/07/02/debs-keynote-thinking-in-events.html  
[2] Turning the database inside-out: https://www.confluent.io/blog/turning-the-database-inside-out-with-apache-samza/






