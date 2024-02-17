# Spark 面试题

## Spark 的特点

Spark 是一种基于内存的快速、通用、可扩展的分布式分析计算引擎。

Spark 框架是**基于内存**计算的，它将大量的输入数据和中间数据都缓存到内存中，能够有效地提高交互型 Job 和迭代型 Job 的执行效率。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241933054.png)

Spark 的特点：

- 快速：Spark 基于内存计算，Spark 采用 RDD + DAG 的计算模型，对于复杂的计算，中间结果不需要落盘，只需要在最后一次 reduce 落盘即可。
- 通用：Spark 提供了统一的解决方案，包括批处理、 SQL 查询、流式计算、机器学习和图计算。
- 多语言支持：Spark 提供了 Scala、Java、Python 和 R 语言的 API。
- 兼容性：Spark 可以运行在 Mesos、Kubernetes、Standalone 或云环境中，并且可以访问 HDFS、Alluxio、Cassandra、HBase、Hive 等多种数据源。

**Spark 为什么快？**

1. Spark 在非 Shuffle 的情况，中间结果不需要落盘，能有效减少磁盘 IO。
2. Spark RDD 的分区特性，能够充分利用并行计算的优势。
3. Spark 的 RDD 支持缓存，能够将频繁使用的数据缓存到内存中，不需要重复计算。
4. Spark 的容错机制（lineage + checkpoint），能够将计算失败的节点上的数据重新计算，而不是重头开始计算。
5. Spark 采用多线程模型，而 MR 采用多进程模型，进程的启停需要做很多初始化等工作，效率比较低。

---

## Spark 运行架构

Spark 框架的核心是一个计算引擎，整体来说，它采用了标准 master-slave 的结构。

- Master：管理整个集群的资源，类似于 YARN 的 ResourceManager
- Worker：管理单个服务器的资源，类似于 YARN 的 NodeManager
- Driver：管理单个 Spark 任务在运行时的工作，类似于 YARN 的 ApplicationMaster
- Executor：负责具体的任务执行，类似于 YARN 容器中运行的 Task

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311181708345.png)

由上图可以看出，对于 Spark 框架有两个核心组件：

- Driver
- Executor

Driver：

Spark Driver，指**实际运行在 Spark 应用中 main() 函数的进程**。Driver 独立于 Master 进程，如果是 Yarn 集群，那么 Driver 也可能被调度到 Worker 节点上运行。

Driver 在 Spark 作业执行时主要负责：

- 将用户程序转化为作业 (job)
- 在 Executor 之间调度任务 (task)
- 跟踪 Executor 的执行情况

Executor：

Spark Executor 在物理上是 Worker 节点中的一个 **JVM 进程**，负责在 Spark 作业中运行具体任务 (Task)，任务彼此之间相互独立。Spark 应用启动时，Executor 节点被同时启动，并且始终伴随着整个 Spark 应用的生命周期而存在。如果有 Executor 节点发生了故障或崩溃，Spark 应用也可以继续执行，会将出错节点上的任务调度到其他 Executor 节点上继续运行。

Executor 有两个核心功能:

- 负责运行组成 Spark 应用的任务，并将结果返回给 Driver 进程
- 它们通过自身的块管理器 (Block Manager) 为用户程序中要求缓存的 RDD 提供内存式存储。**RDD 是直接缓存在 Executor 进程内的**，因此任务可以在运行时充分利用缓存数据加速运算。

---

## Spark 部署模式

- 本地模式（单机）：本地模式就是以一个**独立的进程**，通过其内部的**多个线程来模拟**整个 Spark 运行时环境
- Standalone 模式（集群）：Spark 中的各个角色以独立进程的形式存在，并组成 Spark 集群环境
- Spark on YARN 模式（集群）：Spark 中的各个角色运行在 YARN 的容器内部，并组成 Spark 集群环境
- Spark on Kubernetes 模式（集群）：Spark 中的各个角色运行在 Kubernetes 的容器内部，并组成 Spark 集群环境

---

## Spark on Yarn

Yarn 架构：

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311181704165.png)

- **ResourceManager（RM）**：全局的资源管理器，负责管理整个集群的资源
- **NodeManager（NM）**：每个节点上的资源和任务的管理器
- **ApplicationMaster（AM）**：应用程序管理器，负责应用程序的管理工作

---

Spark Driver：

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311181708345.png)

Spark 应用程序由主程序中的 SparkContext（或 SparkSession）对象协调。简单地说，初始化 SparkContext 的代码就是你的 Driver。Driver 进程**管理作业流程并调度任务**，在应用程序运行期间始终可用。

---

Spark on Yarn：

当在 YARN 上运行 Spark 作业，每个 Spark executor 作为一个 YARN 容器运行。Spark 可以使得多个 Tasks 在同一个容器里面运行。Spark on Yarn 通常有以下两种运行模式：

- Client 模式
- Cluster 模式

---

Client 模式：

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311181711811.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311181713904.png)

1. 在 YARN Client 模式下，spark-submit 提交 Spark Job 之后，就会**在提交的本地机器上启动一个对应的 Driver**
2. Driver 启动后会与 ResourceManager 建立通讯并发起启动 ApplicationMaster 请求
3. ResourceManager 接收到这个 Job 时，会在集群中选一个合适的 NodeManager 并分配一个 Container，及启动 ApplicationMaster（初始化 SparkContext）
4. ApplicationMaster 的功能相当于一个 ExecutorLaucher ，负责向 ResourceManager 申请 Container 资源； ResourceManager 便会与 NodeManager 通信，并启动 Container
5. ApplicationMaster 对指定 NodeManager 分配的 Container 发出启动 Executor 进程请求
6. Executor 进程启动后会向 Driver 反向注册，Executor 全部注册完成后 Driver 开始执行执行 Job 任务
7. Driver 中的 SparkContext 分配 Task 给 Executor 执行，Executor 运行 Task 并向 Driver 汇报运行的状态、进度、以及最终的计算结果；让 Driver 随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务；应用程序运行完成后，ApplicationMaster 向 ResourceManager 申请注销并关闭自己。

---

Cluster 模式：

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311181714154.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311181714211.png)

在 YARN Cluster 模式下，spark-submit 提交 Spark Job 之后，就会在 YARN 集群中启动一个对应的 ApplicationMaster。**此模式中的 Driver 运行在 ApplicationMaster 中**。

---

两种模式的区别：

主要区别在于 Driver 的创建的位置不一样，Client 方式是直接在本地机器上创建一个 Driver 进程，而 Cluster 方式在通过 ResourceManager 在某一个 NodeManager 中创建一个 Driver。

在使用场景当中，Yarn Client 方式一般适用于进行 Job 的调试（Debug），因为 Driver 是在本地可以直接远程断点调试，而且 Driver 会与 Executor 进行大量的通信就会造成占用大量 IO ；Yarn Cluster 方式一般适用于生产环境，因为 Driver 运行在某一个 NodeManager 中就不会出现某一台机器出现网卡激增的情况，缺点就是运行的 Job 日志不能在机器本地实时查看而是需要通过 Job Web 界面查看。

---

## SparkContext

SparkContext 是 Spark 中的主要入口点，它是与 Spark 集群通信的核心对象，可以用来在 Spark 集群中创建 RDD、累加器变量和广播变量等。SparkContext 的核心作用是**初始化 Spark 应用程序运行所需要的核心组件**，包括高层调度器(DAGScheduler)、底层调度器(TaskScheduler)和调度器的通信终端(SchedulerBackend)，同时还会负责 Spark 程序向 Master 注册程序等。

注意，只可以有一个 SparkContext 实例运行在一个 JVM 内存中，所以在创建新的 SparkContext 实例前，必须调用 stop 方法停止当前 JVM 唯一运行的 SparkContext 实例。

---

## Spark Submit

spark-submit 是在 spark 安装目录中 bin 目录下的一个 **shell 脚本文件**，**用于在集群中启动应用程序**（如*.jar、*.py 脚本等）；对于 spark 支持的集群模式，spark-submit 提交应用的时候有统一的接口，不用太多的设置。

常用参数：

```
--master：表示提交任务到哪里执行
    local：提交到本地服务器执行，并分配单个线程
    local[k]：提交到本地服务器执行，并分配 k 个线程
    local[*]：提交到本地服务器执行，并分配本地 core 个数个线程
    spark://HOST:PORT：提交到 standalone 模式部署的 spark 集群中，并指定主节点的 IP 与端口
    yarn：提交到 yarn 模式部署的集群中

--deploy-mode
    spark on yarn 的两种启动方式，client（默认值，应用程序在提交命令所在的节点上运行）或cluster（应用程序在集群中运行）

--class
    应用程序的主类，仅针对 java 或 scala 应用

--name <name>：指定应用程序的名称。
--executor-memory <memory>：指定每个Executor的内存。
--num-executors <num>：指定要分配的Executor的数量。
--driver-memory <memory>：指定Driver程序的内存。
--conf <key=value>：指定其他Spark配置属性。
```

---

## Spark 应用执行流程

1. Spark 程序提交后，Driver 端创建 SparkContext，SparkContext 向资源管理器（StandAlone/Yarn）注册并申请 Executor 资源。
2. 资源管理器分配 Executor 资源并启动 Executor 进程
3. DAGScheduler 构建 DAG 图，并划分 Stage，将每个 Stage 并行执行的 task 封装为 TaskSet，交给 TaskScheduler。
4. Executor 向 SparkContext 申请 task，TaskScheduler 将 task 和应用程序代码发送给 Executor。
5. Executor 运行 task，运行完释放所有资源。

---

## Spark 调度系统

Spark 的调度系统主要包括三个部分：DAGScheduler, SchedulerBackend 和 TaskScheduler。

任务调度流程：

1. DAGScheduler 根据用户代码构建 DAG，以 Shuffle 为边界划分 Stages
2. DAGScheduler 基于 Stages 创建 TaskSets，并交给 TaskScheduler 请求调度
3. SchedulerBackend 获取集群内可用计算资源（以 WorkOffer 为粒度提供计算资源）
4. TaskScheduler 对于给定的 WorkOffer，结合任务的本地倾向性完成任务调度
5. SchedulerBackend 将任务分发给 Executor 进行执行

---

## Spark 为什么比 MR 快

1. Spark 采用 RDD + DAG 的计算模型，对于复杂的计算，中间结果不需要落盘，只需要在最后一次 reduce 落盘即可。而 MR 每一轮计算只能包含一个 Map 和一个 Reduce，多轮计算的中间结果需要落盘，因此会造成大量的磁盘 IO，降低了计算效率。
2. Spark 对 Shuffle 的优化：MR 的 Shuffle 会强制要求按照 Key 进行排序，而 Spark 则只有部分场景才需要排序（bypass 机制不需要排序）。而排序是非常耗时的操作。
3. Spark 支持将频繁使用的数据进行缓存，从内存中读取，不需要再次计算，而 MR 每次计算都需要从磁盘中读取数据。
4. Spark 采用多线程模型，而 MR 采用多进程模型，进程的启停需要做很多初始化等工作，效率比较低。

---

## SparkSession

SparkSession 是 Spark 2.0 以后 Spark 程序的入口，SparkSession 将 SparkContext、SQLContext、HiveContext 集成到了一起，使得 Spark 程序更加方便。

---

## RDD 是什么

RDD(Resilient Distributed Dataset) 叫做**弹性分布式数据集**，是 **Spark 中最基本的数据处理模型**。代码中是一个抽象类，它代表一个弹性的、不可变、可分区、里面的元素可并行计算的集合。

弹性：

- 存储的弹性：内存与磁盘的自动切换
- 容错的弹性：基于血缘的高效容错
- 计算的弹性：Task 失败后的自动重试
- 分片的弹性：可根据需要重新分片

分布式：数据存储在大数据集群不同节点上

数据集：RDD 封装了计算逻辑，并不保存数据

数据抽象：RDD 是一个抽象类，需要子类具体实现

不可变：RDD 封装了计算逻辑，是不可以改变的。如果想要改变，只能产生新的 RDD，在新的 RDD 里面封装计算逻辑

可分区、并行计算

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311202122901.png)

RDD 是 Spark 对于分布式数据集的抽象，每一个 RDD 都代表着一种**分布式数据形态**。比如 lineRDD，它表示数据在集群中以行（Line）的形式存在；而 wordRDD 则意味着数据的形态是单词，分布在计算集群中。

!!! question "RDD 和普通数据结构/容器的区别？"

    1. RDD只是一个逻辑概念，在内存中并不会真正地为某个 RDD 分配空间（除非需要被缓存）。RDD 中的数据只会在计算中产生，而且在计算完成后消失。
    2. RDD 可以包含多个数据分区，不同分区可以由不同的 task 在不同节点进行处理。

---

## RDD 的五大核心属性

Internally, each RDD is characterized by five main properties:

- A list of partitions
- A function for computing each split
- A list of dependencies on other RDDs
- Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)
- Optionally, a list of preferred locations to compute each split on (e.g. block locations for an HDFS file)

---

- **分区列表**：每个分区为 RDD 的一部分数据，分区的数量决定了 RDD 的并行度。
- **分区计算函数**：Spark 在计算时，是使用分区函数对每一个分区进行计算。
- **RDD 之间的依赖关系**：RDD 是计算模型的封装，当需求中需要将多个计算模型进行组合时，就需要将多个 RDD 建立依赖关系。
- **分区器 (可选)** ：当数据为 KV 类型数据时，可以指定分区器（Hash/Range/自定义）。
- **首选位置 (可选)**：计算数据时，判断把计算发送到哪个节点效率最优：移动计算，而不是移动数据。

---

总结：

- RDD 可以看做是一系列的分区，每个分区就是一个数据集片段
- RDD 之间存在依赖关系
- 算子是作用在分区之上的
- 分区器是作用在 KV 类型的 RDD 上的
- 移动计算，而不是移动数据

---

## Job、Stage、Task

- 一个 Action 算子对应一个 Job
- Stage 按照宽依赖划分
- 一个分区对应一个 Task

理论上：每一个 Stage 下有多少个分区，就有多少个 Task，Task 的数量就是任务的最大并行度。一般情况下一个 Task 使用一个 core。

实际上：最大的并行度，取决于任务运行时使用的 Executor 拥有的最大核数。

> 如果 Task 的数量超过了核数，那么多出来的 Task 就需要等待之前的 Task 执行完毕后才能执行。

---

## 分区与并行度

默认情况下，Spark 可以将一个作业切分多个任务后，发送给 Executor 节点并行计算，而**能够并行计算的任务数量我们称之为并行度**。这个数量可以在构建 RDD 时指定。

分区数：是一个静态的概念，如果是内存数据，那么分区数和设置的并行度一致；如果是 HDFS 文件，那么分区数就是 HDFS 文件 Block 个数。

并行度：是一个动态的概念，取决于当前资源的可用核数。

并行度小于等于分区数。

```scala
object RDD_Par {
  def main(args: Array[String]): Unit = {
    // 准备环境
    val sparkConf = new SparkConf().setMaster("local[*]").setAppName("RDD")
    val sc = new SparkContext(sparkConf)

    // RDD的并行度和分区
    // makeRDD方法的第二个参数可以设置分区数，默认值为defaultParallelism（默认并行度）,
    // 是从上面的sparkConf中获取的，即当前环境的最大可用核数
    val rdd = sc.makeRDD(List(1, 2, 3, 4), 2)
    rdd.saveAsTextFile("output")

    // 关闭环境
    sc.stop()
  }
}
```

读取**内存数据**时，数据可以按照并行度的设定进行数据的分区操作，数据分区规则的 Spark 核心源码如下：

```scala
def positions(length: Long, numSlices: Int): Iterator[(Int, Int)] = {
  (0 until numSlices).iterator.map { i =>
    val start = ((i * length) / numSlices).toInt
    val end = (((i + 1) * length) / numSlices).toInt
    (start, end)
  }
}
object RDD_Par_File {
  def main(args: Array[String]): Unit = {
    // 准备环境
    val sparkConf = new SparkConf().setMaster("local[*]").setAppName("RDD")
    val sc = new SparkContext(sparkConf)

    // textFile可以将文件作为数据源，默认也可以设置分区（第二个参数）
    // minPartitions：最小分区数量
    // math.min(defaultParallelism, 2)
    val rdd = sc.textFile("data/1.txt", 2)
    rdd.saveAsTextFile("output")

    // 关闭环境
    sc.stop()
  }
}
```

读取**文件数据**时，分区个数默认 = HDFS 文件切片的个数。

假设文件有 7 个字节，`totalSize = 7` ，`goalSize = 7/2 = 3` ，`7/3 = 2 ... 1` ，也就是说 2 个分区够装 6 个字节，还有剩余 1 个字节，但是此时这一个字节 `1/3 = 33.3% > 10%` ，超过了 hadoop 的 1.1 倍原则，会多创建一个分区，因此最终会创建 3 个分区。

Spark 分区数据的分配：

- Spark 读取文件，采取的是 hadoop 的方式读取，即一行一行地读取，和字节数无关。
- 数据读取时以偏移量为单位。

---

## RDD 算子

算子：即 RDD 的方法。

- **分布式集合对象上的 API 称之为算子**。
- 本地对象的 API 称之为方法。

RDD 算子分为两大类：Transformation 和 Action。

- Transformation：转换算子，将原有 RDD 转换为新的 RDD，**懒加载，不会触发作业的执行**。
- Action：行动算子，触发作业的执行。

---

## 相似算子比较

## 哪些算子会产生 Shuffle

---

## 宽窄依赖

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311201815658.png)

如果父 RDD 中一个分区的数据只流向子 RDD 中的一个分区，这种依赖被称为**窄依赖（Narrow Dependency）**。

如果父 RDD 中一个分区的数据需要一部分流入子 RDD 的某一个分区，另外一部分流入子 RDD 的另外分区，这种依赖被称为**宽依赖（Shuffle Dependency）**。

!!! summary

    **如果 parent RDD 的一个或多个分区中的数据需要全部流入 child RDD 的某一个或多个分区，则是窄依赖。如果 parent RDD 分区中的数据需要一部分流入 child RDD 的某一个分区，另外一部分流入 child RDD 的另外分区，则是宽依赖。**

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312121446943.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312121446363.png)

---

## Stage 划分

DAG 记录了 RDD 的转换过程和任务的阶段（stage）。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241936236.png)

用一句话来概括从 DAG 到 Stages 的拆分过程，那就是：**以 Actions 算子为起点，从后向前回溯 DAG，以 Shuffle 操作为边界去划分 Stages**。**每个 Stage 里 Task 的数量由 Stage 中最后一个 RDD 的分区数量决定。**

!!! question "为什么需要划分阶段？"

    1. stage 中生成的 task 不会太大也不会太小，而且是同构的，便于**并行执行**
    2. 可以将多个操作放在一个 task 里处理，使得操作可以进行串行、流水线式的处理，**提高数据处理效率**
    3. **方便容错**，如一个 stage 失效，可以重新运行 stage 而不需要重新运行整个 job

物理计划生成过程：

1. 根据 action 算子将应用划分为多个 job
2. 根据宽依赖将 job 划分为多个 stage
3. 在每个 stage 中，根据最后一个 RDD 的分区个数将 stage 划分为多个 task

---

## Shuffle 机制

Shuffle 在分布式计算场景中，它被引申为**集群范围内跨节点、跨进程的数据分发**。分布式数据集在集群内的分发，会引入大量的**磁盘 I/O** 与**网络 I/O**。

Shuffle 解决的问题是如何将数据重新组织，使其能够在上游和下游 task 之间进行传递和计算。

> Map 阶段和 Reduce 阶段之间如何完成数据交换？
>
> 通过生产与消费 Shuffle 中间文件的方式，来完成集群范围内的数据交换。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311202113518.png)

在 Map 执行阶段，每个 Task 都会**生成包含 data 文件与 index 文件的 Shuffle 中间文件**，如上图所示。也就是说，Shuffle 文件的生成，**是以 Map Task 为粒度的**，Map 阶段有多少个 Map Task，就会生成多少份 Shuffle 中间文件。

Spark Shuffle 分为两种：一种是基于 Hash 的 Shuffle，另一种是基于 Sort 的 Shuffle。但是在 Spark 2.0 版本中，**Hash Shuffle 方式己经不再使用**。Spark 放弃基于 Hash 的 Shuffle 的原因是，**Hash Shuffle 会产生大量的小文件**，当数据量越来越多时，产生的文件量是不可控的，这严重制约了 Spark 的性能及扩展能力。但是基于 Sort 的 Shuffle 会强制要求数据进行排序，这会增加计算时延。

---

### HashShuffle

将每个 task 处理的数据按 key 进行 hash partition，从而将相同 key 都写入同一个磁盘文件中，而每一个磁盘文件都只属于下游 stage 的一个 task。在将数据写入磁盘之前，会先将数据写入内存缓冲中，当内存缓冲填满之后，才会溢写到磁盘文件中去。

未经优化的 Hash Shuffle 会产生 上游 `task 个数 * 下游 task 个数` 的文件数，这会导致大量的小文件和磁盘 IO，严重影响性能。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312121744627.png)

---

### 优化的 HashShuffle

为了优化 HashShuffle 我们可以设置一个参数：`spark.shuffle.consolidateFiles`

相比于未经优化的版本，就是在一个 Executor 中的所有的 task 是可以共用一个 buffer 内存。在 shuffle write 过程中，task 就不是为下游 stage 的每个 task 创建一个磁盘文件了，而是**允许不同的 task 复用同一批磁盘文件**，这样就可以有效将多个 task 的磁盘文件进行一定程度上的合并，从而大幅度减少磁盘文件的数量，进而提升 shuffle write 的性能。此时的文件个数是 `CPU core 的数量 × 下一个 stage 的 task 数量`。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312121749334.png)

---

### HashShuffle 总结

基于 Hash 的 Shuffle 机制的优缺点

优点：

- 可以省略不必要的排序开销。
- 避免了排序所需的内存开销。

缺点：

- 生产的文件过多，会对文件系统造成压力。
- 大量小文件的随机读写带来一定的磁盘开销。
- 数据块写入时所需的缓存空间也会随之增加，对内存造成压力。

---

### SortShuffle

SortShuffleManager 的运行机制主要分成三种：

- 普通运行机制
- bypass 运行机制，当 shuffle read task 的数量小于等于 `spark.shuffle.sort.bypassMergeThreshold` 参数的值时（默认为 200），就会启用 bypass 机制
- Tungsten Sort 运行机制，开启此运行机制需设置配置项 `spark.shuffle.manager=tungsten-sort`。开启此项配置也不能保证就一定采用此运行机制。

**普通运行机制：**

在该模式下，数据会先写入一个内存数据结构中，此时根据不同的 shuffle 算子，可能选用**不同的数据结构**。如果是 reduceByKey 这种聚合类的 shuffle 算子，那么会选用 Map 数据结构，一边通过 Map 进行聚合，一边写入内存；如果是 join 这种普通的 shuffle 算子，那么会选用 Array 数据结构，直接写入内存。接着，每写一条数据进入内存数据结构之后，就会判断一下，是否达到了某个临界阈值。如果达到临界阈值的话，那么就会尝试将内存数据结构中的数据溢写到磁盘，然后清空内存数据结构。

在溢写到磁盘文件之前，会先根据 key 对内存数据结构中已有的数据进行排序。排序过后，会分批将数据写入磁盘文件。默认的 batch 数量是 10000 条，也就是说，排序好的数据，会以每批 1 万条数据的形式分批写入磁盘文件。写入磁盘文件是通过 Java 的 BufferedOutputStream 实现的。BufferedOutputStream 是 Java 的缓冲输出流，**首先会将数据缓冲在内存中，当内存缓冲满溢之后再一次写入磁盘文件中，这样可以减少磁盘 IO 次数，提升性能**。

一个 task 将所有数据写入内存数据结构的过程中，会发生多次磁盘溢写操作，也就会产生多个临时文件。**最后会将之前所有的临时磁盘文件都进行合并，这就是 merge 过程**，此时会将之前所有临时磁盘文件中的数据读取出来，然后依次写入最终的磁盘文件之中。此外，由于一个 task 就只对应一个磁盘文件，也就意味着该 task 为下游 stage 的 task 准备的数据都在这一个文件中，因此还会单独写一份**索引文件**，其中标识了下游各个 task 的数据在文件中的 start offset 与 end offset。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312121755437.png)

---

**Bypass 运行机制：**

下游任务数比较少的情况下，基于 Hash Shuffle 实现机制明显比基于 Sort Shuffle 实现机制要快，因此基于 Sort Shuffle 实现机制提供了一个带 Hash 风格的回退方案，就是 bypass 运行机制。对于 Reducer 端分区个数少于配置属性 `spark.shuffle.sort.bypassMergeThreshold` 设置的个数时，使用带 Hash 风格的回退计划。

bypass 运行机制的触发条件如下：

- 分区个数小于 `spark.shuffle.sort.bypassMergeThreshold` 参数的值。
- 不是聚合类的 shuffle 算子。（因为聚合类算子通常需要使用 Map 数据结构分区、聚合和排序）

此时，每个 task 会为每个下游 task 都创建一个临时磁盘文件，并将数据按 key 进行 hash 然后根据 key 的 hash 值，将 key 写入对应的磁盘文件之中。当然，写入磁盘文件时也是先写入内存缓冲，缓冲写满之后再溢写到磁盘文件的。最后，同样会将所有临时磁盘文件都合并成一个磁盘文件，并创建一个单独的索引文件。

该过程的磁盘写机制其实跟未经优化的 HashShuffleManager 是一模一样的，因为都要创建数量惊人的磁盘文件，只是在最后会做一个磁盘文件的合并而已。因此少量的最终磁盘文件，也让该机制相对未经优化的 HashShuffleManager 来说，shuffle read 的性能会更好。

而该机制与普通 SortShuffleManager 运行机制的不同在于：

- 磁盘写机制不同（普通机制是达到阈值就溢写，bypass 机制是 hash partition）
- 不会进行排序。

也就是说，**启用该机制的最大好处在于，shuffle write 过程中，不需要进行数据的排序操作**，也就节省掉了这部分的性能开销。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312121759449.png)

---

**Tungsten Sort 运行机制：**

Tungsten Sort 是对普通 Sort 的一种优化，Tungsten Sort 会进行排序，但排序的不是内容本身，而是内容序列化后字节数组的指针(元数据)，把数据的排序转变为了指针数组的排序，实现了直接对序列化后的二进制数据进行排序。由于直接基于二进制数据进行操作，所以在这里面没有序列化和反序列化的过程。内存的消耗大大降低，相应的，会极大的减少的 GC 的开销。

要实现 Tungsten Sort Shuffle 机制需要满足以下条件：

- Shuffle 依赖中不带聚合操作或没有对输出进行排序的要求。
- Shuffle 的序列化器支持序列化值的重定位（当前仅支持 KryoSerializer Spark SQL 框架自定义的序列化器）。
- Shuffle 过程中的输出分区个数少于 16777216 个。

---

### SortShuffle 总结

基于 Sort 的 Shuffle 机制的优缺点

优点：

- 小文件的数量大量减少，Mapper 端的内存占用变少
- Spark 不仅可以处理小规模的数据，即使处理大规模的数据，也不会很容易达到性能瓶颈

缺点：

- 如果 Mapper 中 Task 的数量过大，依旧会产生很多小文件，此时在 Shuffle 传数据的过程中到 Reducer 端， Reducer 会需要同时大量地记录进行反序列化，导致大量内存消耗和 GC 负担巨大，造成系统缓慢，甚至崩溃
- 强制了在 Mapper 端必须要排序，即使数据本身并不需要排序
- 它要基于记录本身进行排序，这就是 Sort-Based Shuffle 最致命的性能消耗。

---

## Shuffle 为什么需要排序以及为什么要合并溢写文件

为什么需要排序：

1. 方便 map 端和 reduce 端聚合，相同 key 的数据会被放到一起，避免每次都要全量扫描，减少磁盘 IO
2. 溢写前的排序可以让每个溢写文件局部有序，方便最终的合并（merge），减少磁盘 IO

为什么要合并溢写文件：

- 避免产生大量的小文件，减少集群的管理资源开销
- 减少 reduce 端读取的文件数量，减少内存占用和磁盘 IO

---

## Spark 为什么让 Task 以线程的方式运行而不是进程

在 Hadoop MapReduce 中，每个 map/reduce task 以一个 Java 进程的方式运行。这样的好处是 task 之间相互独立，每个 task 独享进程资源，不会相互干扰，而且监控管理也比较方便。但坏处是 task 之间**不方便共享数据**（e.g.多个 map task 需要读取同一份字典，需要将字典加载到每个 map task 进程中，造成重复加载、资源浪费）。另外，在应用执行过程中，需要不断启停新旧 task，**进程的启停需要做很多初始化等工作**，效率比较低。

为了**数据共享**和**提高执行效率**，Spark 采用以线程为最小的执行单位，但缺点是线程间会有资源竞争。

---

## cache 和 persist

缓存机制的主要目的是加速计算。

- `rdd.cache()`：将 RDD 持久化到内存中
- `rdd.persist()`：将 RDD 持久化到内存或磁盘中
- `rdd.unpersist()`：将 RDD 从缓存中移除

Spark 的缓存具有容错机制，如果一个缓存的 RDD 的某个分区丢失了，Spark 将按照原来的计算过程，自动重新计算并进行缓存。

cache() 是 lazy 执行的，只有遇到 Action 算子时才会执行。

`cache() = persist(StorageLevel.MEMORY_ONLY)`

RDD 的 cache 默认方式采用 MEMORY_ONLY，DataFrame 的 cache 默认采用 MEMORY_AND_DISK（使用未序列化的 Java 对象格式，优先尝试将数据保存在内存中。如果内存不够存放所有的数据，会将数据写入磁盘文件中）。

缓存由 Executor 的 blockManager 管理，一旦 Driver 执行结束，blockManager 也会被释放，因此缓存的数据也会丢失。缓存 RDD 的分区存储在 blockManager 的 memoryStore 中，memoryStore 包含了一个 LinkedHashMap，用来存放 RDD 的分区。同时 Spark 也直接利用了 LinkedHashMap 的 LRU 功能实现了缓存替换。

---

## checkpoint

**Spark 的错误容忍机制的核心方法**：

1. 重新执行计算任务（根据 lineage 数据依赖关系，在错误发生时回溯需要重新计算的部分）
2. checkpoint 机制

所谓的检查点其实就是通过将 RDD 中间结果写入磁盘由于血缘依赖过长会造成**容错成本过高**，这样就不如在中间阶段做检查点容错，如果检查点之后有节点出现问题，可以从检查点开始重做血缘，减少了开销。

RDD Checkpoint 也是将 RDD 进行持久化，但是**只支持硬盘存储**。和缓存不同，Checkpoint 被认为是安全的，**会切断 RDD 的血缘关系**。

我们知道缓存是分散存储在各个 Executor 上的，Checkpoint 是**收集各个分区的数据，集中存储在一个地方**，这个地方可以是 HDFS、本地文件系统等。

checkpoint() 是等到 job 计算结束之后再重新启动该 job 计算一遍，对其中需要 checkpoint 的 RDD 进行持久化。也就是说，需要 checkpoint 的 RDD 会被计算两次。

在实际使用中，Checkpoint 一般和 Cache 配合使用，**先将数据 Cache 到内存中，然后再将数据 Checkpoint 到磁盘中**。这样执行 checkpoint 的 Job 只需要从内存中读取数据，而不需要重新计算，从而提高了效率。

!!! note "Cache VS Checkpoint"

      - **目的不同**：cache 的目的是加速计算，checkpoint 的目的更多是为了容错，在 job 运行失败后能够快速恢复
      - **存储性质和位置不同**：cache 是为了读写速度快，因此主要使用内存；而 checkpoint 是为了能够可靠读写，因此主要使用分布式文件系统
      - **写入速度和规则不同**：数据缓存速度较快，对 job 的执行时间影响较小；而 checkpoint 速度较慢，会额外启动专门的 job 进行持久化
      - **对 lineage 的影响不同**：cache 不会切断 RDD 的血缘关系，checkpoint 会切断 RDD 的血缘关系
      - **应用场景不同**：cache 适合会被多次读取、占用空间不是很大的 RDD；checkpoint 适用于数据依赖关系复杂、重新计算代价较高的 RDD

      - cache 是分散存储，各个 Executor 并行执行，效率高
      - checkpoint 是集中存储，需要将数据从各个 Executor 收集到 Driver
      - cache 的数据在 Driver 执行结束后会被删除，而 checkpoint 的数据持久化到了 HDFS 或本地磁盘，不会被删除，也能被其他 Driver program 使用

---

## 广播变量

广播变量：分布式共享**只读**变量

广播变量（Broadcast Variables）是一种用于高效分发较大数据集给所有工作节点的机制。它们可以用于**在每个节点上缓存一个只读的大型变量，以便在并行操作中共享**。

由于 Executor 是进程，进程内各线程资源共享，因此，可以将数据放置在 Executor 的内存中，达到共享的目的。

相比于闭包数据都是以 Task 为单位发送的，每个 Task 中都包含闭包数据可能会导致一个 Executor 中含有大量重复的数据，占用大量的内存。因此，完全可以将任务中的闭包数据放置在 Executor 的内存中，达到共享的目的。

当多个 Executor 上需要同一份数据时，可以考虑使用广播变量形式发送数据。尤其当该份数据在多个 Stage 中使用时，通过广播变量一次性发送的方案，执行性能将明显优于当数据需要时再发送的多次发送方案。

总结：

- **减少数据传输**：通过将只读变量广播到集群中，避免了在网络上传输大量数据的开销，减少了网络通信的负担。
- **提高性能**：广播变量在每个节点上只有一份副本，减少了内存使用和垃圾回收的压力，提高了任务的执行效率。
- **共享数据**：广播变量可以在集群中的所有任务中共享，使得每个任务都可以访问相同的数据，方便数据共享和操作。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311210341962.png)

---

## 累加器

累加器：分布式共享**只写**变量

累加器用来把 Executor 端变量信息聚合到 Driver 端。在 Driver 程序中定义的变量，在 Executor 端的每个 Task 都会得到这个变量的一份新的副本，每个 task 更新这些副本的值后，传回 Driver 端进行 merge。

累加器区别于广播变量的特性是：累加器取值 accumulator.value 在 Executor 端无法被读取，只能在 Driver 端被读取，而 Executor 端只能执行累加操作。

注意：在转换算子中调用累加器，如果没有行动算子的话，那么相当于没有执行，累加器结果会不正确。因此一般情况下累加器要放在行动算子中使用。同时注意，如果使用累加器的 RDD 被多次行动算子使用，那么累加器会被多次调用，这样可能会导致产生非预期的结果（建议在行动算子前 `cache()`）。

一些思考：一些简单的统计类累加逻辑可以通过自定义累加器解决，这样可以通过使用全局分布式的累加器避免 shuffle。

> 累加器解决了什么问题？
>
> 在分布式代码执行中，进行全局累加或统计信息。

使用方法：先定义累加器变量，然后在 RDD 算子中调用 add 函数，从而更新累加器状态，最后通过调用 value 函数来获取累加器的最终结果。

总结：

- **分布式累加**：累加器变量可以在并行任务中进行累加操作，无论任务在集群中的哪个节点执行，都可以正确地对变量进行更新，确保了数据的一致性。
- **统计信息收集**：累加器变量可以用于收集任务执行期间的统计信息，例如计数、求和、最大值、最小值等。这对于分布式计算中的调试和监控非常有用。

---

## Spark OOM

- **情况一：在 Driver 端出现 OOM**

一般来说 Driver 的内存大小不用设置，但是当出现使用 collect() 等 Action 算子时，Driver 会将所有的数据都拉取到 Driver 端，如果数据量过大，就会出现 OOM 的情况。

- **情况二：mapPartitions OOM**

mapPartitions 可以以分区为单位进行数据转换操作，但是会将整个分区的数据加载到内存进行引用，处理完的数据是不会被释放的，存在对象的引用，只有程序结束才会释放。因此在内存较小、数据量较大的场合下，容易导致 OOM。

- **情况三：Hash Join OOM**
