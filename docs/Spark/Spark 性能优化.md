# Spark 性能优化

## Tungsten && AQE

## 配置项优化

在 Spark 分布式计算环境中，计算负载主要由 Executors 承担，Driver 主要负责分布式调度，调优空间有限，因此对 Driver 端的配置项我们不作考虑，**我们要汇总的配置项都围绕 Executors 展开**。

配置项优化主要分为 3 类，分别是**硬件资源类**、**Shuffle 类**和 **Spark SQL 大类**。

### 硬件资源类

首先，我们先来说说与 CPU 有关的配置项，主要包括 `spark.cores.max`、`spark.executor.cores` 和 `spark.task.cpus` 这三个参数。它们分别从集群、Executor 和计算任务这三个不同的粒度，指定了用于计算的 CPU 个数。开发者通过它们就可以明确有多少 CPU 资源被划拨给 Spark 用于分布式计算。

为了充分利用划拨给 Spark 集群的每一颗 CPU，准确地说是每一个 CPU 核（CPU Core），你需要设置与之匹配的并行度，并行度用 `spark.default.parallelism` 和 `spark.sql.shuffle.partitions` 这两个参数设置。对于没有明确分区规则的 RDD 来说，我们用 `spark.default.parallelism` 定义其并行度，`spark.sql.shuffle.partitions` 则用于明确指定数据关联或聚合操作中 Reduce 端的分区数量。

**并行度指的是分布式数据集被划分为多少份**，从而用于分布式计算。换句话说，**并行度的出发点是数据，它明确了数据划分的粒度**。并行度越高，数据的粒度越细，数据分片越多，数据越分散。由此可见，像分区数量、分片数量、Partitions 这些概念都是并行度的同义词。

**并行计算任务则不同，它指的是在任一时刻整个集群能够同时计算的任务数量**。换句话说，**它的出发点是计算任务、是 CPU，由与 CPU 有关的三个参数共同决定**。具体说来，Executor 中并行计算任务数的上限是 spark.executor.cores 与 spark.task.cpus 的商，暂且记为 #Executor-tasks，整个集群的并行计算任务数自然就是 #Executor-tasks 乘以集群内 Executors 的数量，记为 #Executors。因此，最终的数值是：#Executor-tasks \* #Executors。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312151607221.png)

说完 CPU，咱们接着说说与内存管理有关的配置项。我们知道，在管理模式上，Spark 分为堆内内存与堆外内存。

堆外内存又分为两个区域，Execution Memory 和 Storage Memory。要想要启用堆外内存，我们得先把参数 `spark.memory.offHeap.enabled` 置为 true，然后用 `spark.memory.offHeap.size` 指定堆外内存大小。堆内内存也分了四个区域，也就是 Reserved Memory、User Memory、Execution Memory 和 Storage Memory。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312151608197.png)

相比 JVM 堆内内存，off heap 堆外内存有很多优势，如更精确的内存占用统计和不需要垃圾回收机制，以及不需要序列化与反序列化。

对外内存使用紧凑的二进制格式（字节数组），相比 JVM 堆内内存，Spark 通过 Java Unsafe API 在堆外内存中的管理，才会有那么多的优势。

对于需要处理的数据集，如果数据模式比较扁平，而且字段多是定长数据类型，就更多地使用堆外内存。相反地，如果数据模式很复杂，嵌套结构或变长字段很多，就更多采用 JVM 堆内内存会更加稳妥。

`spark.memory.fraction` 参数决定着两者如何瓜分堆内内存，它的系数越大，Spark 可支配的内存越多，User Memory 区域的占比自然越小。

当在 JVM 内平衡 Spark 可用内存和 User Memory 时，你需要考虑你的应用中类似的自定义数据结构多不多、占比大不大？然后再相应地调整两块内存区域的相对占比。如果应用中自定义的数据结构很少，不妨把 `spark.memory.fraction` 配置项调高，让 Spark 可以享用更多的内存空间，用于分布式计算和缓存分布式数据集。

---

### Shuffle 类

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312151613377.png)

首先，在 Map 阶段，计算结果会以中间文件的形式被写入到磁盘文件系统。同时，为了避免频繁的 I/O 操作，Spark 会把中间文件存储到写缓冲区（Write Buffer）。这个时候，我们可以通过设置 `spark.shuffle.file.buffer` 来扩大写缓冲区的大小，**缓冲区越大，能够缓存的落盘数据越多，Spark 需要刷盘的次数就越少，I/O 效率也就能得到整体的提升**。

其次，在 Reduce 阶段，因为 Spark 会通过网络从不同节点的磁盘中拉取中间文件，它们又会以数据块的形式暂存到计算节点的读缓冲区（Read Buffer）。缓冲区越大，可以暂存的数据块越多，在数据总量不变的情况下，拉取数据所需的网络请求次数越少，单次请求的网络吞吐越高，网络 I/O 的效率也就越高。这个时候，我们就可以通过 `spark.reducer.maxSizeInFlight` 配置项控制 Reduce 端缓冲区大小，**来调节 Shuffle 过程中的网络负载**。

除此之外，Spark 还提供了一个叫做 `spark.shuffle.sort.bypassMergeThreshold` 的配置项，去处理一种特殊的 Shuffle 场景。

自 1.6 版本之后，Spark 统一采用 Sort shuffle manager 来管理 Shuffle 操作，在 Sort shuffle manager 的管理机制下，无论计算结果本身是否需要排序，Shuffle 计算过程在 Map 阶段和 Reduce 阶段都会引入排序操作。

因此，**在不需要聚合，也不需要排序的计算场景中，我们就可以通过设置 `spark.shuffle.sort.bypassMergeThreshold` 的参数，来改变 Reduce 端的并行度（默认值是 200）**。当 Reduce 端的分区数小于这个设置值的时候，我们就能避免 Shuffle 在计算过程引入排序。

---

## OOM 问题

### Driver OOM

Driver 的主要职责是任务调度，同时参与非常少量的任务计算，因此 Driver 的内存配置一般都偏低，也没有更加细分的内存区域。所以 Driver 端的 OOM 问题自然不是调度系统的毛病，只可能来自它涉及的计算任务，主要有两类：

- 创建小规模的分布式数据集：使用 parallelize、createDataFrame 等 API 创建数据集
- 收集计算结果：通过 take、show、collect 等算子把结果收集到 Driver 端

因此 Driver 端的 OOM 逃不出 2 类病灶：

- 创建的数据集超过内存上限
- 收集的结果集超过内存上限

调节 Driver 端侧内存大小我们要用到 `spark.driver.memory` 配置项，预估数据集尺寸可以用“先 Cache，再查看执行计划”的方式，示例代码如下。

```scala
val df: DataFrame = _
df.cache.count
val plan = df.queryExecution.logical
val estimated: BigInt = spark
.sessionState
.executePlan(plan)
.optimizedPlan
.stats
.sizeInBytes
```

---

### Executor OOM

我们知道，执行内存分为 4 个区域：Reserved Memory、User Memory、Storage Memory 和 Execution Memory。

**在 Executors 中，与任务执行有关的内存区域才存在 OOM 的隐患**。其中，Reserved Memory 大小固定为 300MB，因为它是硬编码到源码中的，所以不受用户控制。而对于 Storage Memory 来说，即便数据集不能完全缓存到 MemoryStore，Spark 也不会抛 OOM 异常，额外的数据要么落盘（MEMORY_AND_DISK）、要么直接放弃（MEMORY_ONLY）。

因此，当 Executors 出现 OOM 的问题，我们可以先把 Reserved Memory 和 Storage Memory 排除，然后锁定 Execution Memory 和 User Memory 去找毛病。

User Memory 用于存储用户自定义的数据结构，如数组、列表、字典等。因此，如果这些数据结构的总大小超出了 User Memory 内存区域的上限，就可能产生 OOM。

解决 User Memory 端 OOM 的思路和 Driver 端的并无二致，也是先对数据结构的消耗进行预估，然后相应地扩大 User Memory 的内存配置。不过，相比 Driver，User Memory 内存上限的影响因素更多，总大小由 spark.executor.memory \* （ 1 - `spark.memory.fraction`）计算得到。

要说 OOM 的高发区，非 Execution Memory 莫属。

实际上，**数据量并不是决定 OOM 与否的关键因素，数据分布与 Execution Memory 的运行时规划是否匹配才是**。一旦分布式任务的内存请求超出 1/N 这个上限，Execution Memory 就会出现 OOM 问题。

假设 Execution Memory 和 Storage Memory 内存空间都是 180MB。

#### 数据倾斜

我们先来看第一个数据倾斜的例子。节点在 Reduce 阶段拉取数据分片，3 个 Reduce Task 对应的数据分片大小分别是 100MB 和 300MB。显然，第三个数据分片存在轻微的数据倾斜。由于 Executor 线程池大小为 3，因此每个 Reduce Task 最多可获得 360MB \* 1 / 3 = 120MB 的内存空间。Task1、Task2 获取到的内存空间足以容纳分片 1、分片 2，因此可以顺利完成任务。

Task3 的数据分片大小远超内存上限，即便 Spark 在 Reduce 阶段支持 Spill 和外排，120MB 的内存空间也无法满足 300MB 数据最基本的计算需要，如 PairBuffer 和 AppendOnlyMap 等数据结构的内存消耗，以及数据排序的临时内存消耗等等。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312151635411.png)

因此，针对以这个案例为代表的数据倾斜问题，我们至少有 2 种调优思路：

- 消除数据倾斜，让所有的数据分片尺寸都不大于 100MB
- 调整 Executor 线程池、内存、并行度等相关配置，提高 1/N 上限到 300MB

在充分利用 CPU 的前提下解决 OOM 的问题：

- 维持并发度、并行度不变，增大执行内存设置，提高 1/N 上限到 300MB
- 维持并发度、执行内存不变，使用相关配置项来提升并行度将数据打散，让所有的数据分片尺寸都缩小到 100MB 以内

---

#### 数据膨胀

我们再来看第二个数据膨胀的例子。节点在 Map 阶段拉取 HDFS 数据分片，3 个 Map Task 对应的数据分片大小都是 100MB。按照之前的计算，每个 Map Task 最多可获得 120MB 的执行内存，不应该出现 OOM 问题才对。

尴尬的地方在于，磁盘中的数据进了 JVM 之后会膨胀。在我们的例子中，数据分片加载到 JVM Heap 之后翻了 3 倍，原本 100MB 的数据变成了 300MB，因此，OOM 就成了一件必然会发生的事情。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312151637591.png)

因此，针对以这个案例为代表的数据膨胀问题，我们还是有至少 2 种调优思路：

- 把数据打散，提高数据分片数量、降低数据粒度，让膨胀之后的数据量降到 100MB 左右
- 加大内存配置，结合 Executor 线程池调整，提高 1/N 上限到 300MB

---

#### 总结

发生在 Driver 端的 OOM 可以归结为两类：

- 创建的数据集超过内存上限
- 收集的结果集超过内存上限

应对 Driver 端 OOM 的常规方法，是先适当预估结果集尺寸，然后再相应增加 Driver 侧的内存配置。

Execution Memory 区域 OOM 的产生的原因是数据分布与 Execution Memory 的运行时规划不匹配，也就是分布式任务的内存请求超出了 1/N 上限。解决 Execution Memory 区域 OOM 问题的思路总的来说可以分为 3 类：

- 消除数据倾斜，让所有的数据分片尺寸都小于 1/N 上限
- 把数据打散，提高数据分片数量、降低数据粒度，让膨胀之后的数据量降到 1/N 以下
- 加大内存配置，结合 Executor 线程池调整，提高 1/N 上限

---
