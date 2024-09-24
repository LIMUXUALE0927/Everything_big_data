# Flink

## Flink 简介

Apache Flink 是一个分布式流处理引擎，用于对无界和有界数据流进行有状态计算。

**特点：**

- 支持高吞吐、低延迟、高性能的流处理

- 支持带有事件时间的窗口操作

- 支持有状态计算的 Exactly-once 语义

- 支持高度灵活的窗口操作，支持基于 time、count、session，以及 data-driven 的窗口操作

- 支持具有 Backpressure 功能的持续流模型

- 支持基于轻量级分布式快照（Snapshot）实现的容错

- 一个运行时同时支持 Batch on Streaming 处理和 Streaming 处理

- Flink 在 JVM 内部实现了自己的内存管理

- 支持迭代计算

- 支持程序自动优化：避免特定情况下 Shuffle、排序等昂贵操作，中间结果有必要进行缓存

**Flink 的四大基石**

Flink 之所以能这么流行，离不开它最重要的四个基石：**Checkpoint、State、Time、Window**。

- 首先是 Checkpoint 机制，这是 Flink 最重要的一个特性。Flink 基于 Chandy-Lamport 算法实现了一个分布式的一致性的快照，从而提供了一致性的语义。

- 提供了一致性的语义之后，Flink 为了让用户在编程时能够更轻松、更容易地去管理状态，还提供了一套非常简单明了的 State API，包括里面的有 ValueState、ListState、MapState，近期添加了 BroadcastState，使用 State API 能够自动享受到这种一致性的语义。

- 除此之外，Flink 还实现了 Watermark 的机制，能够支持基于事件的时间的处理，或者说基于系统时间的处理，能够容忍数据的延时、容忍数据的迟到、容忍乱序的数据。

- 另外流计算中一般在对流数据进行操作之前都会先进行开窗，即基于一个什么样的窗口上做这个计算。Flink 提供了开箱即用的各种窗口，比如滑动窗口、滚动窗口、会话窗口以及非常灵活的自定义的窗口。

**Flink 的核心组成部分：**

![](https://hcavh6tzye.feishu.cn/space/api/box/stream/download/asynccode/?code=MDc2NjU2ZWJiNGJhYjIxMmY2ZTdkYmVmN2UwNTFhY2RfNGhrbFJNOFU3VENlaVg1bXY2TnlQbUM1azl0b2dqTXVfVG9rZW46S0hFd2JkdTBCb29RTkx4dngzcmN6dUg3bjllXzE3MjcxNTQxNzE6MTcyNzE1Nzc3MV9WNA)

Flink 分别提供了面向流式处理的接口（DataStream API）和面向批处理的接口（DataSet API）。因此，Flink 既可以完成流处理，也可以完成批处理。Flink 支持的拓展库涉及机器学习（FlinkML）、复杂事件处理（CEP）、以及图计算（Gelly），还有分别针对流处理和批处理的 Table API。

## 状态化流处理

几乎所有数据都是以连续事件流的形式产生，任何一个处理事件流的应用，如果要支持多条记录的转换操作，都必须是有状态的，即能够存储和访问中间结果。

Apache Flink 会将应用状态存储在本地内存或嵌入式数据库中。由于采用的是分布式架构，Flink 需要对本地状态予以保护，以避免因应用或机器故障导致数据丢失。为了实现该特性，Flink 会定期将应用状态的一致性检查点（checkpoint）写入远程持久化存储。

有状态的流处理应用通常会从事件日志中读取事件记录。事件日志负责存储事件流并将其分布式化。由于事件只能以追加的形式写入持久化日志中，所以其顺序无法在后期改变。写入事件日志的数据流可以被相同或不同的消费者重复读取。得益于日志的追加特性，无论向消费者发布几次，事件的顺序都能保持一致。事件日志系统可以持久化输入事件并以确定的顺序将其重放。一旦出现故障，Flink 会利用之前的检查点恢复状态并重置事件日志的读取位置，以此来使有状态的流处理应用恢复正常。

**Lambda 架构**

Lambda 架构在传统周期性批处理架构的基础上添加了一个由低延迟流处理引擎所驱动的「提速层」（speed layer）。在该架构中，**到来的数据会同时发往流处理引擎和写入批量存储**。流处理引擎会近乎实时地计算出近似结果，并将其写入「提速表」中。批处理引擎周期性地处理批量存储的数据，将精确结果写入批处理表，随后将「提速表」中对应的非精确结果删除。为了获取最终结果，应用需要将「提速表」中的近似结果和批处理表中的精确结果合并。

**缺点：**

1. 该架构需要在拥有不同 API 的两套独立处理系统之上实现两套语义相同的应用逻辑

2. 流处理引擎计算的结果只是近似的

3. Lambda 架构较难配置和维护

暂时无法在飞书文档外展示此内容

## Ch2：流处理基础

**状态**

在抽象层次上，我们可以将状态视为 Flink 中算子的记忆，它记住有关过去输入的信息，并可用于影响未来输入的处理。状态是计算过程中生成的数据信息，在 Apache Flink 的容错、故障恢复和检查点中起着非常重要的作用。

**算子**

算子是数据流程序的基本功能单元，他们从输入获取数据，对其进行计算，然后产生数据并发往输出以供后续处理。**没有输入端的算子称为数据源，没有输出端的算子称为数据汇。**

![](https://hcavh6tzye.feishu.cn/space/api/box/stream/download/asynccode/?code=YTdjNTMzZWFhMmZkZmI5NjhjMjI4MWJhZmE2OTk2ZDRfZHY3b0d0RVRzSGo3aHJDbndrS0ZGTjFOT3RMR1BOUzNfVG9rZW46S2ZGb2JnWFNSb1AyTFZ4S3BGM2NOeE5KbmFmXzE3MjcxNTQxNzE6MTcyNzE1Nzc3MV9WNA)

![](https://hcavh6tzye.feishu.cn/space/api/box/stream/download/asynccode/?code=ZmM0MmE1MGE5ODdmOTE0NDAzZThhMjZiNWRhMjcyNTZfY2FJTnh3ZGUzOGxQdUZzUGR5WU5NdU1tUWMzVGxFZDdfVG9rZW46S3g0TGJLQmRjb0FacUp4cW5VeWM0UVZnbm1iXzE3MjcxNTQxNzE6MTcyNzE1Nzc3MV9WNA)

**数据交换策略**

数据交换策略定义了如何将数据分配给物理 Dataflow 图中的不同任务。这些策略可以由执行引擎根据算子的语义自动选择，也可以人为显式指定。

- **转发策略（Forward Strategy）：**在发送端任务和接收端任务之间一对一地进行数据传输。如果两个任务位于同一物理机上（通常由任务调度器决定），则此交换策略可以避免网络通信。

- **广播策略（Broadcast Strategy）：**会把每个数据发往下游算子的全部并行任务。该策略会把数据复制多份且涉及到网络通信，因此代价十分昂贵。

- **基于键值的策略（Key-based Strategy）：**根据某一键值属性对数据分区，并保证键值相同的数据项会交由同一任务处理。

- **随机策略（Random Strategy）：**会将数据均匀分配至算子的所有任务，以实现计算任务的负载均衡。

![](https://hcavh6tzye.feishu.cn/space/api/box/stream/download/asynccode/?code=YmEyMzlkMDhlZDQ1MWFiOWJhOTJlNTNjNzk1NmI5YWNfSDRDUU92T0xGcVFZWW9XN2RkdGdUQW56bjdhMGxIWUVfVG9rZW46UVBFNWJpV2l5bzJRaHd4RzNGUmNpekxFbmtjXzE3MjcxNTQxNzE6MTcyNzE1Nzc3MV9WNA)

**延迟和吞吐**

由于流式应用会持续执行且输入可能是无限的，因此流式应用需要尽可能快地计算结果，同时还要应对很高的事件接入速率。**我们用延迟和吞吐表示这两方面的性能需求。**

- 延迟表示处理一个事件所需要的时间

- 吞吐表示系统每单位时间可以处理多少事件

- 一旦事件到达速率过高，系统就会被迫开始缓冲事件。如果系统持续以力不能及的高速率接收数据，那么缓冲区可能会用尽，继而可能导致数据丢失。**这种情形通常被称为「背压」（backpressure）。**

通过并行处理多条数据流，可以在处理更多事件的同时降低延迟。
