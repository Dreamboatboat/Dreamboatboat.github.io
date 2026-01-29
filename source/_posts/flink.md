---
title: flink知识点
date: 2026-01-29 19:44:31
tags: bigdata
---

![](flink_architecture.png)
## 开头
客户端（Client）通过 CLI、REST API 或 SQL 提交作业，作业进入 Flink 集群 后由 JobManager 负责作业解析、调度、Checkpoint 和故障恢复；真正的计算由多个 TaskManager 执行，每个 TaskManager 通过 Task Slot 并行运行算子任务。Flink 从 Kafka、数据库、文件系统等 数据源 持续接收流式或批式数据，经过有状态计算后，将结果写入数据库、消息队列或数据湖等 下游系统。整个集群的资源通常由 YARN 或 Kubernetes 统一管理，并配合监控系统进行运行状态与指标采集，体现了 Flink 以流为核心、支持高并发、有状态计算和高可用 的设计特点。

Flink 通过 JobManager 进行统一调度和容错管理，由多个 TaskManager 以 Slot 为资源单位并行执行有状态流计算，依托 YARN 或 Kubernetes 管理资源，从 Kafka 等数据源持续接入数据并实时输出结果，是一个高吞吐、低延迟、支持 Exactly-Once 的流处理系统。

## 1、Flink 的整体架构是怎样的？JobManager 和 TaskManager 各自负责什么？
Flink 采用主从架构，由 JobManager 和多个 TaskManager 组成。JobManager 负责作业接收、执行图生成、任务调度以及 Checkpoint 和故障恢复；TaskManager 负责实际执行算子计算，通过 Task Slot 提供并行能力，并维护本地状态完成数据处理。
## 2、Flink 的 Task Slot 是什么？和线程有什么区别？
Task Slot 是 Flink 中 TaskManager 的资源划分单位，表示可用于执行任务的一份资源配额；它用于控制并行度和资源隔离。线程是操作系统的执行单元，而 Slot 不等同于线程，一个 Slot 内可以运行多个算子链并复用线程，两者关注点分别是资源管理和执行机制。
## 3、Flink 的并行度是如何决定的？可以在哪些层面设置？
Flink 的并行度表示算子同时运行的实例数，最终受 Task Slot 总数 限制。并行度可以在多个层面设置：集群级（默认并行度）、作业级（env.setParallelism）、算子级（operator.setParallelism），优先级从低到高依次覆盖。
## 4、Flink 是如何做到高可用（HA）的？
Flink 的高可用（HA）主要通过 JobManager 的多副本机制 实现：部署多个 JobManager，其中一个为 Active，其他为 Standby，通过 ZooKeeper 或类似协调服务进行主备选举；一旦 Active 挂掉，Standby 接管作业调度，并结合 Checkpoint 恢复任务状态，从而保证作业持续运行不丢数据。
## 5、Flink 中的 Operator Chain 是什么？有什么好处？
Flink 中的 Operator Chain 是将多个连续的算子（如 Map → Filter → FlatMap）在同一个 Task Slot 内串行执行，形成一个链式任务。这样可以减少算子之间的数据序列化和网络传输开销，提高 CPU 利用率和整体吞吐，同时降低任务调度压力，从而优化性能。
## 6、Flink 为什么被称为“流批一体”？底层原理是什么？
Flink 被称为“流批一体”，是因为它将 批处理看作有界流处理，统一了流和批的计算模型：无论是无限数据流还是有界数据集，底层都用 DataStream/DataSet API + 有状态算子 + Operator Chain + Event Time/Watermark 处理，批处理只是特殊情况下的流处理，从而实现同一引擎处理流和批任务。
## 7、Flink 的数据交换（Network Stack）是怎样的？
Flink 的 Network Stack 负责 TaskManager 之间的数据传输，核心是 Result Partition + Input Gates + Buffer Pool。每个算子输出数据写入 Result Partition，数据被切分为固定大小的 Buffer，通过 TCP 或 Netty 在 TaskManager 之间传输；下游算子通过 Input Gate 读取 Buffer，并可实现 流式消费或批式拉取，支持背压（Backpressure）控制流量，保证高吞吐与低延迟。
## 8、Flink 的反压（Backpressure）是如何产生和传播的？
Flink 的 反压（Backpressure） 是当下游算子处理速度跟不上上游发送速度时产生的流控机制。具体过程是：
1、上游算子输出数据写入缓冲区（Buffer Pool）
2、如果缓冲区被填满，无法继续发送数据
3、上游算子暂停或减慢数据生成，形成反压
4、这种压力沿着算子链 向上游传播，直到源算子，保证系统不会因堆积过多数据而 OOM

📌 核心点：反压是 自然产生的流控机制，无需人工干预，同时也是监控流式作业性能的重要指标。
## 9、Flink 中的三种时间语义（Processing / Event / Ingestion）有什么区别？
Flink 中有三种时间语义，区别如下：
1、Processing Time（处理时间）
 - 使用 系统当前时间作为时间戳
 - 简单、延迟低，但无法处理乱序或延迟事件
 - 常用于对实时性要求高、容忍轻微不准确的场景

2、Event Time（事件时间）
 - 使用 事件自身的时间戳
 - 需要 Watermark 来处理乱序
 - 可以实现严格的时间窗口计算，保证准确性
 - 流批一体核心语义，生产中最常用

Ingestion Time（摄取时间）
 - 在数据进入 Flink 时 赋予时间戳
 - 介于 Processing 和 Event Time 之间
 - 对乱序处理有限支持，延迟和准确性折中

📌 总结一句话：
Processing Time 快但不准，Event Time 准但复杂，Ingestion Time 折中处理。

## 10、为什么生产中一般使用 Event Time？
生产中一般使用 Event Time，因为它能保证 **结果的时间准确性**：
- 事件的时间戳来源于数据本身，而不是系统处理时间
- 能正确处理 乱序和延迟事件
- 配合 Watermark 可实现精准窗口计算和一致性输出

📌 总结一句话：
Event Time 让计算结果与事件发生时间对齐，是保证业务逻辑正确性的关键。

## 11、Watermark 是什么？如何生成？
Watermark（水印） 是 Flink 用来处理 乱序事件 的关键机制，它表示事件时间的进度，告诉下游算子“我已经看到事件时间 ≤ X 的大部分数据了，可以触发窗口计算”。
生成方式：
1、周期性生成（Periodic Watermark）：源算子定期根据收到的事件时间计算水印，例如 maxEventTime - allowedLateness
2、按事件生成（Punctuated Watermark）：每条事件触发一次水印生成，根据事件内容决定水印值

📌 核心点：
- Watermark 不是每条事件的时间戳，而是流的时间进度标识
- 它允许 延迟和乱序事件 被正确处理，同时触发窗口计算和状态更新

## 12、Watermark 延迟设置不合理会带来什么问题？
Watermark 延迟（allowed lateness 或生成水印时的延迟）设置不合理，会带来两个典型问题：

延迟太小（偏严格）
- 系统过早触发窗口计算
- 后到的乱序事件被丢弃或落入 Side Output
- 结果不准确 → 业务逻辑错

延迟太大（偏宽松）
- 窗口计算被拖延，任务等待时间增加
- 增加状态保存量，可能导致 State 过大 / 内存压力
- 处理延迟变高，吞吐下降

📌 面试总结一句话：
Watermark 延迟需在“准确性”和“延迟/状态开销”之间权衡，过小丢数据，过大延迟高。

## 13、Flink 支持哪些窗口类型？（滚动、滑动、会话）
Flink 支持三类常用窗口类型：

滚动窗口（Tumbling Window）
- 固定长度、不重叠的时间段
- 每条事件只属于一个窗口
- 常用于定期统计

滑动窗口（Sliding Window）
- 固定长度、可重叠的时间段
- 窗口间隔可小于窗口长度
- 每条事件可能属于多个窗口
- 常用于连续统计或滚动指标

会话窗口（Session Window）
- 动态长度，根据 事件间隔 自动划分
- 会话超时后触发窗口计算
- 常用于用户行为分析、事件聚合

📌 面试总结一句话：
滚动窗口固定不重叠，滑动窗口固定可重叠，会话窗口动态按空闲时间分组。
## 14、窗口的 Trigger、Evictor 分别是什么？
在 Flink 窗口机制中：
Trigger（触发器）
- 决定 什么时候计算窗口内容并输出结果
- 可以按 时间（Processing/Event Time）、数据量 或自定义条件触发
- 例如每到达 Watermark 就触发窗口计算

Evictor（清除器）
- 决定 窗口中哪些元素在计算前被移除
- 常用于限制窗口大小或去掉过期/无效数据
- 触发计算后可以清空窗口或者保留部分元素

📌 总结一句话：
Trigger 控制 何时计算窗口，Evictor 控制 计算前清理哪些数据，两者配合实现灵活窗口计算。
## 15、Allowed Lateness 和 Side Output 是干什么的？
在 Flink 窗口计算中：
Allowed Lateness（允许延迟）
- 指在窗口关闭后，还允许多长时间接收 迟到事件
- 迟到事件在此时间范围内仍会更新窗口结果
- 超过这个时间，事件会被丢弃或发送到 Side Output

Side Output（侧输出）
- 用于接收 迟到或异常事件，避免丢失数据
- 可以单独做处理或记录日志，保证业务可追溯性

📌 总结一句话：
Allowed Lateness 定义迟到事件的容忍时间，Side Output 用来收集超出时间的事件，实现完整的数据处理与容错。

## 16、Flink 中的 State 有哪几类？（Keyed / Operator State）
Flink 中的 状态（State） 主要分为两类：

Keyed State（键控状态）
- 每个 Key 都有独立状态
- 只能在 KeyedStream 中使用
- 常用类型：ValueState、ListState、MapState、ReducingState、AggregatingState
- 适合按用户、设备等分组维护状态

Operator State（算子状态）
- 状态与算子实例绑定，不依赖 Key
- 每个并行算子实例维护自己的状态
- 常用模式：ListState（用于分布式源的偏移量管理）
- 适合源算子或不需要按 Key 划分的状态

📌 总结一句话：
Keyed State 按 Key 维护，每个 Key 独立；Operator State 按算子实例维护，全局共享或分布式保存。
## 17、Keyed State 和 Operator State 的使用场景有什么区别？
Keyed State 和 Operator State 的使用场景区别主要在于 是否需要按 Key 分组管理状态：

Keyed State（键控状态）
- 适合 按用户、设备、会话等分组维护状态
- 典型场景：统计每个用户的累计行为次数、会话分析、分组聚合
- 优点：状态自动按 Key 隔离，可水平扩展

Operator State（算子状态）
- 适合 算子整体维护状态，不依赖 Key
- 典型场景：Source 偏移量保存（Kafka 消费位点）、算子内部全局计数器
- 优点：方便管理全局状态或分布式源状态，不需要 Key

📌 总结一句话：
Keyed State 用于按 Key 的分组状态，Operator State 用于算子实例的全局或源状态，两者场景互补。
## 18、Flink 中常见的 State 类型有哪些？（Value / List / Map 等）
Flink 中常见的 **状态类型（State）**有几类，按用途和数据结构区分：

1、ValueState
- 保存单个值
- 典型用途：保存每个 Key 的最新状态或计数器

2、ListState
- 保存一个有序列表
- 典型用途：保存事件序列、累积历史数据

3、MapState
- 保存 Key → Value 映射
- 典型用途：复杂聚合或查找，例如用户事件类型统计

4、ReducingState
- 对输入元素按指定函数 增量聚合，只保留一个结果
- 典型用途：求和、求最大值

5、AggregatingState
- 类似 ReducingState，但可以自定义返回类型
- 典型用途：增量计算复杂指标，如平均值

📌 总结一句话：
ValueState 保存单值，ListState 保存列表，MapState 保存映射，Reducing/AggregatingState 用于增量聚合计算。

## 19、State 是存在哪里的？Heap State 和 RocksDB State 有什么区别？
Flink 的 State（状态） 存储位置取决于状态后端（State Backend）：

1、Heap State（默认 Memory / Heap 状态）
- 状态保存在 TaskManager JVM 堆内存
- 优点：访问速度快
- 缺点：状态量大时容易 OOM，不适合海量状态

2、RocksDB State（RocksDB 状态后端）
- 状态保存在 本地 RocksDB（嵌入式 KV 数据库）
- 优点：支持 非常大的状态，超出内存可存磁盘
- 缺点：访问速度比 Heap 慢，但可通过缓存优化

📌 核心区别：
Heap State 内存快但容量有限，RocksDB State 支持大状态并持久化，但访问相对慢，生产中常用于海量 Keyed State。

## 20、State TTL 是什么？如何避免状态无限增长？
State TTL（Time-To-Live） 是 Flink 用来限制 状态存活时间 的机制。
- 作用：指定状态在多长时间未被访问或更新后自动过期，从而 释放内存/存储
- 避免状态无限增长：通过 TTL 配合 清理策略（On Read / On Write / Periodic）
    - 过期状态会被删除或标记为可回收
    - 配合 RocksDB State 可减轻磁盘压力

📌 总结一句话：
State TTL 控制状态过期时间，防止长期不活跃的数据堆积，保证状态存储可控且系统稳定。

## 21、Flink 的状态是如何做一致性保证的？
Flink 的状态一致性主要通过 Checkpoint + 两阶段提交（Two-Phase Commit）机制 来保证：

1、Checkpoint
- JobManager 定期触发 TaskManager 将 Keyed/Operator State 快照写入持久存储（如 HDFS、S3）
- 一旦作业失败，可以从最新成功的 Checkpoint 恢复状态
- 保证 Exactly-Once 状态一致性
2、Two-Phase Commit（针对外部系统）
- 在写 Sink（如 Kafka、数据库）时，先将结果写入 预提交状态
- Checkpoint 成功后再提交（commit）到外部系统
- 避免重复写入或漏写，保证端到端一致性

📌 总结一句话：
Flink 通过定期 Checkpoint 保存状态快照，并配合 Two-Phase Commit 控制外部系统写入，实现 Exactly-Once 的状态一致性。

## 22、状态和 Checkpoint 之间是什么关系？
在 Flink 中，状态（State）和 Checkpoint 之间的关系可以这样理解：
- 状态是作业运行时保存的变量，包括 Keyed State 和 Operator State，用于算子间或算子内部的增量计算和数据累积。
- Checkpoint 是状态的快照机制，定期将当前状态保存到持久化存储（如 HDFS、S3），用来保证作业失败后可以恢复到最近一致的状态。
- 换句话说：状态是活的，Checkpoint 是状态的持久化副本，两者结合实现 高可用与 Exactly-Once。

📌 面试一句话总结：
状态是作业运行数据，Checkpoint 是状态的持久化快照，用于故障恢复和一致性保证。

## 23、什么是 Checkpoint？Flink 为什么要做 Checkpoint？
Checkpoint 是 Flink 用来 保存作业运行时状态快照 的机制。
- 作用：记录 Keyed State 和 Operator State 的一致性快照，并持久化到可靠存储（如 HDFS、S3）
- 目的：
  - 故障恢复：作业失败后可以从最近的 Checkpoint 恢复，保证不中断
  - Exactly-Once 语义：结合外部系统（如 Kafka）可保证端到端的数据一致性
  - 高可用：与 JobManager HA 配合，防止单点故障导致数据丢失

📌 面试一句话总结：
Checkpoint 是 Flink 的状态快照机制，用于故障恢复、保证状态一致性和 Exactly-Once。

## 24、Checkpoint 的整体流程是怎样的？
Flink Checkpoint 的整体流程可以分为以下几个步骤：

1、触发 Checkpoint
- JobManager 定期触发 Checkpoint（按配置的间隔）
- 通知所有 TaskManager 开始快照

2、快照状态
- 每个 TaskManager 将 Keyed/Operator State 的当前快照写入 本地或远程存储
- 状态数据被切分为多个 subtask snapshot

3、确认完成（Barrier 对齐）
- Flink 使用 Checkpoint Barrier 沿着数据流传播
- 确保快照包含 同一时刻的数据状态，即跨算子一致性

4、JobManager 汇总
- JobManager 收到所有 TaskManager 的确认后，将 Checkpoint 标记为 成功
- 可以用于故障恢复

5、恢复（可选）
- 作业失败时，从最近成功的 Checkpoint 恢复状态和任务位置
- 保证 Exactly-Once 或 At-Least-Once 语义

📌 面试一句话总结：
Checkpoint 通过 Barrier 对齐机制，将作业状态快照持久化到可靠存储，实现故障恢复和状态一致性。

## 25、什么是Checkpoint Barrier机制？
**Checkpoint Barrier（检查点屏障）**是 Flink 实现 状态一致性快照的核心机制。
- 作用：沿着数据流标记“快照边界”，保证所有算子在同一逻辑时间点生成状态快照，从而实现全局一致性。
- 工作流程：
  1.JobManager 触发 Checkpoint 时，发送 Barrier 到源算子
  2.Barrier 随数据流传播，经过每个下游算子
  3.算子在接收到 Barrier 后，将当前状态快照到持久化存储
  4.Barrier 对齐（Alignment）：等待前面未到达的 Barrier，保证同一 Checkpoint 包含所有上游数据

📌 面试一句话总结：
Barrier 是沿数据流传播的检查点标记，用于对齐算子状态，保证全局一致性。
## 26、两阶段提交（Two-Phase Commit）在 Flink 中是怎么用的？
在 Flink 中，**Two-Phase Commit（两阶段提交）**主要用于 保证向外部系统写入的 Exactly-Once 语义，典型场景是 Kafka、数据库、消息队列等 Sink。
工作流程：
1、预提交（Pre-Commit / Phase 1）
- Task 将结果写入 临时/缓冲状态（本地或外部事务）
- 不立即对外可见
2、Checkpoint 成功（Commit / Phase 2）
- JobManager 确认 Checkpoint 成功后
- Task 将临时结果 正式提交（commit）到外部系统
3、失败处理
- 如果作业失败，未提交的临时数据会被 回滚（abort）
- 避免重复写入或数据丢失

📌 面试一句话总结：
Two-Phase Commit 在 Flink 中用于将计算结果先写入缓冲/事务，Checkpoint 成功后再正式提交，实现外部系统的 Exactly-Once。

## 27、Checkpoint 和 Savepoint 有什么区别？
Checkpoint 和 Savepoint 都是 Flink 的状态快照机制，但用途和管理方式不同：
1、Checkpoint（检查点）
- 自动触发，由 Flink 定期创建
- 用于容错恢复，失败作业自动从最近成功的 Checkpoint 恢复
- 生命周期短，通常由系统自动管理和清理
- 优先关注 Exactly-Once 一致性

2、Savepoint（保存点）
- 手动触发，用于用户主动保存作业状态
- 用于作业升级、重启或迁移
- 生命周期长，不会自动删除
- 可用于从旧版本作业恢复到新版本，便于运维

📌 面试一句话总结：
Checkpoint 用于系统自动容错，Savepoint 用于手动备份、作业升级或迁移。

## 28、Flink 和 Spark Streaming / Structured Streaming 的区别？
Flink 和 Spark Streaming / Structured Streaming 的区别可以从 处理模型、延迟、状态管理和容错机制 来总结：

1、处理模型
- Flink：真正的 流处理引擎，以事件为核心，支持无限流和有界流
- Spark Streaming（旧版）：微批处理，将流切分成小批次
- Structured Streaming（Spark 新版）：微批或 Continuous 模式，但底层仍偏批

2、延迟
- Flink：低延迟，毫秒级处理
- Spark Streaming：批级延迟，通常秒级
- Structured Streaming：延迟介于两者，微批模式受批次大小影响

3、状态管理
- Flink：内建 Keyed State / Operator State + Checkpoint，支持大状态、Exactly-Once
- Spark：状态管理依赖 Checkpoint +外部存储，状态量大时压力大

- 4、容错与一致性
- Flink：通过 Checkpoint + Barrier + Two-Phase Commit 实现端到端 Exactly-Once
- Spark：通过 微批+写幂等 Sink 或 Checkpoint 实现至少一次（At-Least-Once）

📌 面试一句话总结：
Flink 是原生流处理、低延迟、状态丰富、Exactly-Once；Spark Streaming 是微批处理，延迟高，Structured Streaming 改进微批但仍偏批，两者在流批模型和状态管理上差异明显。

## 29、什么场景适合用 Flink，而不是 Spark？
Flink 更适合以下场景，而 Spark Streaming / Structured Streaming 不够优：

1、低延迟实时计算
- 毫秒级处理要求，如交易风控、实时推荐
2、复杂有状态流处理
- 大规模 Keyed State、会话分析、动态窗口
3、严格的 Exactly-Once 语义
- 端到端精确一次写入 Kafka、数据库
4、乱序事件处理
- 事件时间 + Watermark 支持乱序数据聚合
5、流批一体
- 同一作业既能处理有界批数据，也能处理无限流

📌 面试一句话总结：
当业务要求低延迟、复杂状态管理、乱序事件处理或严格一致性时，用 Flink；Spark 更适合延迟容忍、批式流处理场景。
