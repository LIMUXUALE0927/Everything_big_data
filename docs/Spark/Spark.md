## Spark vs Hadoop

==Spark 是一种基于内存的快速、通用、可扩展的分布式分析计算引擎。==

Hadoop 属于**一次性数据计算**框架：框架在处理数据的时候，会从存储介质中读取数据，进行逻辑操作，然后将处理的结果重新存储回介质中。通过磁盘 IO 进行作业会消耗大量资源和时间，效率很低。

Spark 提供了更丰富的数据处理模型，而且可以**基于内存**进行数据集的多次迭代，速度更快。

==Spark 和 Hadoop 的根本差异是多个作业之间的数据通信问题== : Spark 多个作业之间数据通信是基于内存，而 Hadoop 是基于磁盘。

在绝大多数的数据计算场景中，Spark 确实会比 MapReduce 更有优势。但是 Spark 是基于内存的，所以在实际的生产环境中，由于**内存的限制**，可能会由于内存资源不够导致 Job 执行失败，此时，MapReduce 其实是一个更好的选择，所以 Spark 并不能完全替代 MR。

## Spark 核心模块

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241933054.png)

## Spark 运行架构

Spark 框架的核心是一个计算引擎，整体来说，它采用了标准 master-slave 的结构。

如下图所示，它展示了一个 Spark 执行时的基本结构。图形中的 Driver 表示 master， 负责管理整个集群中的作业任务调度。图形中的 Executor 则是 slave，负责实际执行任务。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241933776.png)

由上图可以看出，对于 Spark 框架有两个核心组件：

- Driver
- Executor

Driver：

Spark 驱动器节点，用于执行 Spark 任务中的 main 方法，负责实际代码的执行工作。 Driver 在 Spark 作业执行时主要负责：

- 将用户程序转化为作业 (job)
- 在 Executor 之间调度任务 (task)
- 跟踪 Executor 的执行情况
- 通过 UI 展示查询运行情况

Executor：

Spark Executor 是集群中工作节点 (Worker) 中的一个 JVM 进程，负责在 Spark 作业中运行具体任务 (Task)，任务彼此之间相互独立。Spark 应用启动时，Executor 节点被同时启动，并且始终伴随着整个 Spark 应用的生命周期而存在。如果有 Executor 节点发生了故障或崩溃，Spark 应用也可以继续执行，会将出错节点上的任务调度到其他 Executor 节点上继续运行。

Executor 有两个核心功能:

- 负责运行组成 Spark 应用的任务，并将结果返回给驱动器进程
- 它们通过自身的块管理器 (Block Manager) 为用户程序中要求缓存的 RDD 提供内存式存储。RDD 是直接缓存在 Executor 进程内的，因此任务可以在运行时充分利用缓存数据加速运算。

## WordCount

注意使用 Maven 创建项目，add framework support for Scala，然后在 Maven 中添加 Spark 依赖，注意必须使用 JDK1.8。

```
hello world
hello spark
hi java

hello java
hello spark
hi spark
package com.anson

import org.apache.spark.rdd.RDD
import org.apache.spark.{SparkConf, SparkContext}

object Spark01_WordCount {
  def main(args: Array[String]): Unit = {
    // 建立和Spark框架的连接
    val sparkConf = new SparkConf().setMaster("local").setAppName("WordCount")
    val sc = new SparkContext(sparkConf)

    // 执行业务操作
    // 读取文件，获取一行行的数据
    val lines: RDD[String] = sc.textFile("data")

    // 将一行行的数据进行分词，扁平化 hello, world, hello, spark
    val words: RDD[String] = lines.flatMap(_.split(" "))

    // 分组 (hello, hello, hello), (world, world)
    val wordGroup: RDD[(String, Iterable[String])] = words.groupBy(word => word)

    // 对分组后的数据转换
    val wordToCount = wordGroup.map {
      case (word, list) => {
        (word, list.size)
      }
    }

    // 将结果打印到控制台
    val array: Array[(String, Int)] = wordToCount.collect()
    array.foreach(println)

    // 关闭连接
    sc.stop()
  }
}

```

==RDD 的数据只有在调用 `collect()` 方法时，才会真正执行业务逻辑操作，之前的封装全部都是功能的扩展。类似于 Java 中的 IO 流，都用到了装饰者设计模式。== 但是 RDD 是不保存数据的，IO 会保存一部分数据。

通过 IO 流来理解 RDD：[Link](https://www.bilibili.com/video/BV11A411L7CK?p=27&vd_source=f3af28d1fd89af1eb80db058885d7130) | [Link](https://www.bilibili.com/video/BV11A411L7CK?p=28&vd_source=f3af28d1fd89af1eb80db058885d7130)

## 分布式计算模拟

基础脚手架：

服务器：`Executor.scala`

```scala
object Executor {
  def main(args: Array[String]): Unit = {
    // 启动服务器，接收数据
    val server = new ServerSocket(9999)
    println("服务器启动，等待接收数据")

    // 等待客户端的连接
    val client: Socket = server.accept()
    val in = client.getInputStream

    val i: Int = in.read()
    println("接收到客户端发送的数据： " + i)

    in.close()
    client.close()
    server.close()
  }
}
```

客户端：`Driver.scala`

```scala
object Driver {
  def main(args: Array[String]): Unit = {
    // 连接服务器
    val client = new Socket("localhost", 9999)

    val out = client.getOutputStream
    out.write(2)
    out.flush()
    out.close()

    client.close()
  }
}
```

---

如果是想要让 Executor 进行计算，那么应该由 Driver 把数据和计算传输到 Executor 中，而不是把计算的逻辑写死在 Executor 当中。

Task 类作为任务：`Task.scala`

```scala
class Task extends Serializable {
  // 数据
  val data = List(1, 2, 3, 4)
  // 计算逻辑
  val logic = (num: Int) => {
    num * 2
  }
  // 计算
  def compute() = {
    data.map(logic)
  }
}
```

服务器：`Executor.scala`

```scala
object Executor {
  def main(args: Array[String]): Unit = {
    // 启动服务器，接收数据
    val server = new ServerSocket(9999)
    println("服务器启动，等待接收数据")

    // 等待客户端的连接
    val client: Socket = server.accept()
    val in = client.getInputStream
    val objIn = new ObjectInputStream(in)
    val task = objIn.readObject().asInstanceOf[Task]
    val ints: List[Int] = task.compute()
    println("计算节点计算的结果为： " + ints)

    objIn.close()
    client.close()
    server.close()
  }
}
```

客户端：`Driver.scala`

```scala
object Driver {
  def main(args: Array[String]): Unit = {
    // 连接服务器
    val client = new Socket("localhost", 9999)

    val out = client.getOutputStream
    val objOut = new ObjectOutputStream(out)
    // 把计算发送到 Executor
    val task = new Task()
    objOut.writeObject(task)
    objOut.flush()
    objOut.close()
    client.close()
    println("客户端数据发送完毕")
  }
}
```

---

多个 Executor：

定义 SubTask 作为真正传输的对象：`SubTask.scala`

```scala
class SubTask extends Serializable {
  var data: List[Int] = _

  var logic: (Int) => Int = _

  def compute() = {
    data.map(logic)
  }
}
```

服务器 1：`Executor.scala`

```scala
object Executor {
  def main(args: Array[String]): Unit = {
    // 启动服务器，接收数据
    val server = new ServerSocket(8888)
    println("服务器8888启动，等待接收数据")

    // 等待客户端的连接
    val client: Socket = server.accept()
    val in = client.getInputStream
    val objIn = new ObjectInputStream(in)
    val task = objIn.readObject().asInstanceOf[SubTask]
    val ints: List[Int] = task.compute()
    println("计算节点8888计算的结果为： " + ints)

    objIn.close()
    client.close()
    server.close()
  }
}
```

服务器 2：`Executor2.scala`

```scala
object Executor2 {
  def main(args: Array[String]): Unit = {
    // 启动服务器，接收数据
    val server = new ServerSocket(9999)
    println("服务器9999启动，等待接收数据")

    // 等待客户端的连接
    val client: Socket = server.accept()
    val in = client.getInputStream
    val objIn = new ObjectInputStream(in)
    val task = objIn.readObject().asInstanceOf[SubTask]
    val ints: List[Int] = task.compute()
    println("计算节点9999计算的结果为： " + ints)

    objIn.close()
    client.close()
    server.close()
  }
}
```

客户端：`Driver.scala`

```scala
object Driver {
  def main(args: Array[String]): Unit = {
    // 连接服务器
    val client1 = new Socket("localhost", 8888)
    val client2 = new Socket("localhost", 9999)
    val task = new Task()
  // -------------------------------------
    val out1 = client1.getOutputStream
    val objOut1 = new ObjectOutputStream(out1)
    val subTask = new SubTask()
    subTask.logic = task.logic
    subTask.data = task.data.take(2)
    objOut1.writeObject(subTask)
    objOut1.flush()
    objOut1.close()
    client1.close()
  // -------------------------------------
    val out2 = client2.getOutputStream
    val objOut2 = new ObjectOutputStream(out2)
    val subTask2 = new SubTask()
    subTask2.logic = task.logic
    subTask2.data = task.data.takeRight(2)
    objOut2.writeObject(subTask2)
    objOut2.flush()
    objOut2.close()
    client2.close()

    println("客户端数据发送完毕")
  }
}
```

## Spark 三大数据结构

- RDD：弹性分布式数据集
- 累加器：分布式共享**只写**变量
- 广播变量：分布式共享**只读**变量

### 累加器

累加器用来把 Executor 端变量信息聚合到 Driver 端。在 Driver 程序中定义的变量，在 Executor 端的每个 Task 都会得到这个变量的一份新的副本，每个 task 更新这些副本的值后， 传回 Driver 端进行 merge。

```scala
val rdd = sc.makeRDD(List(1,2,3,4,5))
// 声明累加器
var sum = sc.longAccumulator("sum");
rdd.foreach(
  num => {
    // 使用累加器
    sum.add(num)
  }
)
// 获取累加器的值
println("sum = " + sum.value)
```

==注意：在转换算子中调用累加器，如果没有行动算子的话，那么相当于没有执行，累加器结果会不正确。因此一般情况下累加器要放在行动算子中使用。==

一些思考：一些简单的统计类累加逻辑可以通过自定义累加器解决，这样可以通过使用全局分布式的累加器避免 shuffle。

### 广播变量

广播变量用来高效分发较大的对象。向所有工作节点发送一个较大的只读值，以供一个或多个 Spark 操作使用。比如，如果你的应用需要向所有节点发送一个较大的只读查询表， 广播变量用起来都很顺手。在多个并行操作中使用同一个变量，但是 Spark 会为每个任务分别发送。

==闭包数据，都是以 Task 为单位发送的，每个 Task 中都包含闭包数据可能会导致一个 Executor 中含有大量重复的数据，占用大量的内存。因此，完全可以将任务中的闭包数据放置在 Executor 的内存中，达到共享的目的。==

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241933066.png)

```scala
// 通过broadcast避免join导致的shuffle
val rdd1 = sc.makeRDD(List(("a", 1), ("b", 2), ("c", 3), ("d", 4)), 4)
val list = List(("a", 4), ("b", 5), ("c", 6), ("d", 7))
// 声明广播变量
val broadcast: Broadcast[List[(String, Int)]] = sc.broadcast(list)
val resultRDD: RDD[(String, (Int, Int))] = rdd1.map {
  case (key, num) => {
    var num2 = 0
    // 使用广播变量
    for ((k, v) <- broadcast.value) {
      if (k == key) {
        num2 = v
      }
    }
    (key, (num, num2))
  }
}
```

---

## RDD 简介

RDD(Resilient Distributed Dataset) 叫做**弹性分布式数据集**，是 Spark 中最基本的数据处理模型。代码中是一个抽象类，它代表一个弹性的、不可变、可分区、里面的元素可并行计算的集合。

弹性

- 存储的弹性：内存与磁盘的自动切换
- 容错的弹性：数据丢失可以自动恢复
- 计算的弹性：计算出错重试机制
- 分片的弹性：可根据需要重新分片

分布式：数据存储在大数据集群不同节点上

数据集：==RDD 封装了计算逻辑，并不保存数据==

数据抽象：RDD 是一个抽象类，需要子类具体实现

不可变：==RDD 封装了计算逻辑，是不可以改变的。==如果想要改变，只能产生新的 RDD，在新的 RDD 里面封装计算逻辑

可分区、并行计算

> ==RDD 的数据只有在调用 `collect()` 方法时，才会真正执行业务逻辑操作，之前的封装全部都是功能的扩展。类似于 Java 中的 IO 流，都用到了装饰者设计模式。== 但是 RDD 是不保存数据的，IO 会保存一部分数据。

通过 IO 流来理解 RDD：[Link](https://www.bilibili.com/video/BV11A411L7CK?p=27&vd_source=f3af28d1fd89af1eb80db058885d7130) | [Link](https://www.bilibili.com/video/BV11A411L7CK?p=28&vd_source=f3af28d1fd89af1eb80db058885d7130)

## RDD 的核心属性

Internally, each RDD is characterized by five main properties:

- A list of partitions
- A function for computing each split
- A list of dependencies on other RDDs
- Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)
- Optionally, a list of preferred locations to compute each split on (e.g. block locations for an HDFS file)

- 分区列表：RDD 数据结构中存在分区列表，用于执行任务时并行计算，是实现分布式计算的重要属性。
- 分区计算函数：Spark 在计算时，是使用分区函数对每一个分区进行计算。
- RDD 之间的依赖关系：RDD 是计算模型的封装，当需求中需要将多个计算模型进行组合时，就需要将多个 RDD 建立依赖关系。
- 分区器 (可选) ：当数据为 KV 类型数据时，可以通过设定分区器自定义数据的分区。
- 首选位置 (可选)：计算数据时，可以根据计算节点的状态选择不同的节点位置进行计算。即判断把计算发送到哪个节点效率最优：移动计算，而不是移动数据。

## RDD 的执行原理

从计算的角度来讲，数据处理过程中需要计算资源 (内存 & CPU) 和计算模型 (逻辑)。执行时，需要将计算资源和计算模型进行协调和整合。

Spark 框架在执行时，先申请资源，然后将应用程序的数据处理逻辑分解成一个一个的计算任务，然后==将任务发到已经分配资源的计算节点上==，按照指定的计算模型进行数据计算，最后得到计算结果。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241936820.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241936829.png)

从以上流程可以看出 RDD 在整个流程中主要用于将逻辑进行封装，并生成 Task 发送给 Executor 节点执行计算。

## RDD 的创建

在 Spark 中创建 RDD 的创建方式可以分为四种：

- 从集合 (内存) 中创建 RDD
- 从外部存储 (文件) 创建 RDD
- 从其他 RDD 创建
- 直接创建 RDD(new)

我们常用前 2 种。

```scala
object RDD_Memory {
  def main(args: Array[String]): Unit = {
    // 准备环境
    // [*]指当前机器可用核心数，如8核，此时Spark会用8个线程模拟
    val sparkConf = new SparkConf().setMaster("local[*]").setAppName("RDD")
    val sc = new SparkContext(sparkConf)

    // 从内存中创建RDD，将内存中集合的数据作为处理的数据源
    val seq = Seq[Int](1, 2, 3, 4)
    // 并行
    //    val rdd = sc.parallelize(seq)
    val rdd = sc.makeRDD(seq) // 底层实现就是调用了RDD对象的parallelize()方法
    rdd.collect().foreach(println)

    // 关闭环境
    sc.stop()
  }
}
object RDD_File {
  def main(args: Array[String]): Unit = {
    // 准备环境
    val sparkConf = new SparkConf().setMaster("local[*]").setAppName("RDD")
    val sc = new SparkContext(sparkConf)

    // 从文件中的数据作为数据源
    // 路径默认以当前环境的根路径为基准，绝对、相对路径都行，也可以是目录名
    val rdd = sc.textFile("data/1.txt")
    rdd.collect().foreach(println)

    // 关闭环境
    sc.stop()
  }
}
```

## RDD 的并行度与分区

默认情况下，Spark 可以将一个作业切分多个任务后，发送给 Executor 节点并行计算，而==能够并行计算的任务数量我们称之为并行度==。这个数量可以在构建 RDD 时指定。记住，这里的==并行执行的任务数量，并不是指的切分任务的数量==，不要混淆了。

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

读取**文件数据**时，数据是按照 Hadoop 文件读取的规则进行切片分区，而切片规则和数据读取的规则有些差异，具体 Spark 核心源码如下：

```java
public InputSplit[] getSplits(JobConf job, int numSplits) throws IOException {
    long totalSize = 0; // compute total size
    for (FileStatus file: files) { // check we have valid files
        if (file.isDirectory()) {
            throw new IOException("Not a file: "+ file.getPath());
        }
        totalSize += file.getLen();
    }
    long goalSize = totalSize / (numSplits == 0 ? 1 : numSplits);
    long minSize = Math.max(job.getLong(org.apache.hadoop.mapreduce.lib.input.
    FileInputFormat.SPLIT_MINSIZE, 1), minSplitSize);
    ...
    for (FileStatus file: files) {
        ...
    }
    if (isSplitable(fs, path)) {
        long blockSize = file.getBlockSize();
        long splitSize = computeSplitSize(goalSize, minSize, blockSize);
    ...
    }
    protected long computeSplitSize(long goalSize, long minSize, long blockSize) {
        return Math.max(minSize, Math.min(goalSize, blockSize));
    }
}
```

假设文件有 7 个字节，`totalSize = 7` ，`goalSize = 7/2 = 3` ，`7/3 = 2 ... 1` ，也就是说 2 个分区够装 6 个字节，还有剩余 1 个字节，但是此时这一个字节 `1/3 = 33.3% > 10%` ，超过了 hadoop 的 1.1 倍原则，会多创建一个分区，因此最终会创建 3 个分区。

Spark 分区数据的分配：

- Spark 读取文件，采取的是 hadoop 的方式读取，即一行一行地读取，和字节数无关。
- 数据读取时以偏移量为单位。

## RDD 算子

RDD 算子（operator）大致可分为 2 类：转换（Transformation）和行动（Action）。

- 转换：功能的补充和封装，将旧的 RDD 包装成新的 RDD。
  - `flatMap` , `map` , `groupBy`
- 行动：触发任务的调度和作业的执行。
  - `collect`

### RDD 转换算子

RDD 根据数据处理方式的不同将算子整体上分为 Value 类型、双 Value 类型和 Key-Value 类型。

#### map

`map` ：将处理的数据逐条进行映射转换，这里的转换可以是类型的转换，也可以是值的转换。

```scala
object RDD_Operator_Transform {
  def main(args: Array[String]): Unit = {
    val sparkConf = new SparkConf().setMaster("local[*]").setAppName("Operator")
    val sc = new SparkContext(sparkConf)

    val rdd = sc.makeRDD(List(1, 2, 3, 4))
    // 转换函数
    def mapFunction(num: Int): Int = {
      num * 2
    }
    //    val mapRDD = rdd.map(mapFunction)
    //    val mapRDD = rdd.map((num: Int) => {num * 2})
    val mapRDD = rdd.map(_ * 2)
    mapRDD.collect().foreach(println)

    sc.stop()
  }
}
```

`map` 并行计算效果：

RDD 的计算：==一个分区内的数据是串行，只有前面一个数据全部的逻辑执行完毕后，才会执行下一个数据。因此，分区内数据的执行是有序的。不同分区是并行执行，因此顺序不定。==

```scala
object RDD_Operator_Map {
  def main(args: Array[String]): Unit = {
    val sparkConf = new SparkConf().setMaster("local[*]").setAppName("Operator")
    val sc = new SparkContext(sparkConf)

    val rdd = sc.makeRDD(List(1, 2, 3, 4), 1)

    val mapRDD1 = rdd.map(num => {
      println(">>>>>>> " + num)
      num
    })

    val mapRDD2 = mapRDD1.map(num => {
      println("####### " + num)
      num
    })

    mapRDD2.collect()

    sc.stop()
  }
}
```

执行结果：

```
>>>>>>> 1
####### 1
>>>>>>> 2
####### 2
>>>>>>> 3
####### 3
>>>>>>> 4
####### 4
```

如果把分区数量设置为 2：

```
>>>>>>> 3
####### 3
>>>>>>> 1
####### 1
>>>>>>> 4
####### 4
>>>>>>> 2
####### 2
```

#### mapPartitions

将待处理的数据以分区为单位发送到计算节点进行处理，这里的处理是指可以进行任意的处理，哪怕是过滤数据。

```scala
object RDD_Operator_Map_Par {
  def main(args: Array[String]): Unit = {
    val sparkConf = new SparkConf().setMaster("local[*]").setAppName("Operator")
    val sc = new SparkContext(sparkConf)

    val rdd = sc.makeRDD(List(1, 2, 3, 4), 2)

    val mpRDD = rdd.mapPartitions(
      iter => {
        println(">>>>>>")
        iter.map(_ * 2)
      }
    )

    mpRDD.collect().foreach(println)

    sc.stop()
  }
}
```

执行结果：

```
>>>>>>
>>>>>>
2
4
6
8
```

我们发现计算逻辑只被执行了 2 次（分区数量）。==mapPartitions 可以以分区为单位进行数据转换操作，但是会将整个分区的数据加载到内存进行引用，处理完的数据是不会被释放的，存在对象的引用，只有程序结束才会释放。因此在内存较小、数据量较大的场合下，容易导致 OOM。==

⭐ mapPartitions 的小功能：获取每个分区的最大值。

```scala
val mpRDD = rdd.mapPartitions(
  iter => {
    List(iter.max).iterator
  }
)
```

> [!question] map 和 mapPartitions 的区别

数据处理角度：

Map 算子是分区内一个数据一个数据执行，类似于**串行操作**。而 mapPartitions 算子是以分区为单位进行**批处理操作**。

功能的角度：

Map 算子主要目的将数据源中的数据进行转换和改变。但是不会减少或增多数据。MapPartitions 算子需要传递一个迭代器，返回一个迭代器，没有要求的元素的个数保持不变， 所以可以增加或减少数据。

性能的角度：

Map 算子因为类似于串行操作，所以性能比较低，而是 mapPartitions 算子类似于批处
理，所以性能较高。但是 mapPartitions 算子会长时间占用内存，那么这样会导致 OOM 错误。所以在内存有限的情况下，不推荐使用。

---

#### mapPartitionsWithIndex

有时候我们只需要对某一个分区的数据进行处理，那我们如何知道当前处理的是哪个分区呢？通过 `mapPartitionsWithIndex` 即可。

将待处理的数据以分区为单位发送到计算节点进行处理，这里的处理是指可以进行任意的处理，哪怕是过滤数据，在处理时同时可以获取当前分区索引。

```scala
val dataRDD1 = rdd.mapPartitionsWithIndex(
  (index, iter) => {
    if (index == 1) {
      iter
    } else {
      Nil.iterator
    }
  }
)
val dataRDD1 = rdd.mapPartitionsWithIndex(
  (index, iter) => {
    iter.map(
      num => {
        (index, num)
      }
    )
  }
)
```

#### flatMap

将处理的数据进行扁平化后再进行映射处理，所以也被称为扁平映射。

```scala
val rdd: RDD[List[Int]] = sc.makeRDD(List(
  List(1, 2), List(3, 4)
))

val flatRDD: RDD[Int] = rdd.flatMap(
  list => {
  list
  }
)
val rdd: RDD[List[Int]] = sc.makeRDD(List(List(1, 2), 3, List(4, 5)))

val flatRDD: RDD[Int] = rdd.flatMap(
  data => {
    data match {
      case list: List[_] => list
      case num => List(num)
    }
  }
)
```

#### glom

将同一个分区的数据直接转换为相同类型的内存数组进行处理，分区不变。

函数签名：

```scala
def glom(): RDD[Array[T]]
object RDD_Operator_glom {
  def main(args: Array[String]): Unit = {
    val sparkConf = new SparkConf().setMaster("local[*]").setAppName("Operator")
    val sc = new SparkContext(sparkConf)

    val rdd: RDD[Int] = sc.makeRDD(List(1, 2, 3, 4), 2)

    val glomRDD: RDD[Array[Int]] = rdd.glom()

    glomRDD.collect().foreach(data => println(data.mkString(",")))

    sc.stop()
  }
}
```

运行结果：

```
1,2
3,4
```

⭐ 小功能：计算所有分区最大值求和（分区内取最大值，分区间最大值求和）

```scala
object RDD_Operator_glom {
  def main(args: Array[String]): Unit = {
    val sparkConf = new SparkConf().setMaster("local[*]").setAppName("Operator")
    val sc = new SparkContext(sparkConf)

    val rdd: RDD[Int] = sc.makeRDD(List(1, 2, 3, 4), 2)

    val glomRDD: RDD[Array[Int]] = rdd.glom()

    val maxRDD = glomRDD.map(array => {
      array.max
    })

    println(maxRDD.collect().sum)

    sc.stop()
  }
}
```

#### groupBy

==将数据根据指定的规则进行分组，分区数量不会变，但是数据会被打乱，然后重新组合，我们将这样的操作称之为 shuffle。==

极限情况下，数据可能被分在同一个分区中。

一个组的数据在一个分区中，但是并不是说一个分区中只有一个组。多个组是有可能放在同一个分区的。

```scala
object RDD_Operator_groupBy {
  def main(args: Array[String]): Unit = {
    val sparkConf = new SparkConf().setMaster("local[*]").setAppName("Operator")
    val sc = new SparkContext(sparkConf)

    val rdd: RDD[Int] = sc.makeRDD(List(1, 2, 3, 4), 2)

    // 返回的结果会被当做分组的key
    def groupFunction(num: Int) = {
      num % 2
    }

    val groupRDD = rdd.groupBy(groupFunction)

    groupRDD.collect().foreach(println)

    sc.stop()
  }
}
```

运行结果：

```
(0,CompactBuffer(2, 4))
(1,CompactBuffer(1, 3))
```

#### filter

```scala
def filter(f: T => Boolean): RDD[T]
```

将数据根据指定的规则进行筛选过滤，符合规则的数据保留，不符合规则的数据丢弃。

当数据进行筛选过滤后，分区不变，但是分区内的数据可能不均衡，生产环境下，可能会出现**数据倾斜**。

#### sample

函数签名：

```scala
def sample(
withReplacement: Boolean,
fraction: Double,
seed: Long = Utils.random.nextLong): RDD[T]
```

根据指定的规则从数据集中抽取数据，常用于判断**数据倾斜**。

#### distinct

函数签名：

```scala
def distinct()(implicit ord: Ordering[T] = null): RDD[T]
def distinct(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T]
```

底层通过 `reduceByKey` 实现：

```scala
def distinct(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T] = withScope {
  def removeDuplicatesInPartition(partition: Iterator[T]): Iterator[T] = {
    // Create an instance of external append only map which ignores values.
    val map = new ExternalAppendOnlyMap[T, Null, Null](
      createCombiner = _ => null,
      mergeValue = (a, b) => a,
      mergeCombiners = (a, b) => a)
    map.insertAll(partition.map(_ -> null))
    map.iterator.map(_._1)
  }

  partitioner match {
    case Some(_) if numPartitions == partitions.length =>
      mapPartitions(removeDuplicatesInPartition, preservesPartitioning = true)
    case _ => map(x => (x, null)).reduceByKey((x, _) => x, numPartitions).map(_._1)
  }
}
```

#### coalesce

函数签名：

```scala
def coalesce(numPartitions: Int, shuffle: Boolean = false,
partitionCoalescer: Option[PartitionCoalescer] = Option.empty)
(implicit ord: Ordering[T] = null) : RDD[T]
```

根据数据量**缩减分区**，用于大数据集过滤后，提高小数据集的执行效率。
当 spark 程序中存在过多的小任务的时候，可以通过 coalesce 方法，收缩合并分区，减少分区的个数，减小任务调度成本。

```scala
val dataRDD = sparkContext.makeRDD(List(1,2,3,4,5,6),3)
val dataRDD1 = dataRDD.coalesce(2)
```

⭐ `coalesce` 在默认情况下是不会打乱数据并重新组合的！在上面的例子中，dataRDD 有 3 个分区，每个分区分别是 (1,2), (3, 4), (5, 6)，但是 `coalesce` 之后 dataRDD1 中 2 个分区变成 (1, 2), (3, 4, 5, 6)，也就容易导致**数据倾斜**。

如果想让数据均衡，可以进行 **shuffle** 处理。

> [!question] 如果想要扩大分区该怎么做？
> coalesce 算子可以扩大分区，但是必须进行 shuffle 操作。但是对于这种情况，可以直接使用 repartition 算子。

#### repartition

该操作内部其实执行的是 coalesce 操作，参数 shuffle 的默认值为 true。无论是将分区数多的 RDD 转换为分区数少的 RDD，还是将分区数少的 RDD 转换为分区数多的 RDD，repartition 操作都可以完成，因为无论如何都会经过 shuffle 过程。

#### sortBy

函数签名：

```scala
def sortBy[K](
  f: (T) => K,
  ascending: Boolean = true,
  numPartitions: Int = this.partitions.length)
  (implicit ord: Ordering[K], ctag: ClassTag[K]): RDD[T]
```

该操作用于排序数据。在排序之前，可以将数据通过 f 函数进行处理，之后按照 f 函数处理的结果进行排序，默认为升序排列。排序后新产生的 RDD 的分区数与原 RDD 的分区数一 致。中间存在 **shuffle** 的过程。

```scala
val dataRDD = sparkContext.makeRDD(List(1,2,3,4,1,2),2)
val dataRDD1 = dataRDD.sortBy(num => num, false, 4)
```

#### partitionBy

函数签名：

```scala
def partitionBy(partitioner: Partitioner): RDD[(K, V)]
```

将 (k, v) 类型数据按照指定 Partitioner **重新进行分区**。Spark 默认的分区器是 HashPartitioner。这个方法不是 RDD 类中的方法，但是 RDD 对象可以调用这个方法，底层原理是 **Scala 的隐式转换**（二次编译）。

```scala
val rdd: RDD[(Int, String)] =
   sc.makeRDD(Array((1,"aaa"),(2,"bbb"),(3,"ccc")),3)
import org.apache.spark.HashPartitioner
val rdd2: RDD[(Int, String)] =
   rdd.partitionBy(new HashPartitioner(2))
```

> [!question] paritionBy 和 repartition 的区别？

在 Spark 中，`repartition` 和 `partitionBy` 都是重新分区的算子，其中 `partitionBy` 只能作用于 PairRDD。但是，当作用于 PairRDD 时，`repartition` 和 `partitionBy` 的行为是不同的。`repartition` 是把数据随机打散均匀分布于各个 Partition（直接对数据进行 shuffle）；而 `partitionBy` 则在参数中指定了 Partitioner（默认 HashPartitioner），将每个 (K,V) 对按照 K 根据 Partitioner 计算得到对应的 Partition。==在合适的时候使用 `partitionBy` 可以减少 shuffle 次数，提高效率。==

#### reduceByKey

可以将数据按照相同的 key 对 value 进行聚合。如果 Key 的数据只有一个，是不会参与计算的。

`reduceByKey` 要求分区内（combine）和分区间的聚合规则是相同的。如果分区内和分区间要按照不同规则计算，则要使用 `aggregateByKey` 。（如分区内求最大值，分区间求和）

#### groupByKey

将数据源的数据根据 key 对 value 进行分组。reduceByKey = groupByKey + reduce。

```scala
val rdd = sc.makeRDD(List(("a", 1), ("a", 2), ("a", 3), ("b", 4)))

val groupRDD: RDD[(String, Iterable[Int])] = rdd.groupByKey()
val groupRDD1: RDD[(String, Iterable[(String, Int)])] = rdd.groupBy(_._1)
```

`groupByKey` 和 `groupBy(_._1)` 的区别主要是返回值类型。

`groupByKey` 由于会根据 key 对数据进行分组，因此可能导致 **shuffle**。而在 Spark 中，shuffle 操作必须落盘处理（因此 shuffle 操作效率很低），不能在内存中等待数据，否则容易导致 OOM。

> [!question] groupByKey 和 reduceByKey 的区别？

从功能的角度：`reduceByKey` 其实包含分组和聚合的功能，而 `groupByKey` 只能分组，不能聚合。

从 shuffle 的角度：`reduceByKey` 和 `groupByKey` 由于分组都存在 shuffle 的操作，但是 `reduceByKey` 可以在 shuffle 前对分区内相同 key 的数据进行**预聚合 (combine)** 功能，**这样能够减少落盘的数据量**，而 `groupByKey` 只是进行分组，不存在数据量减少的问题，reduceByKey 性能比较高。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241936831.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241936142.png)

#### aggregateByKey

函数签名：

```scala
def aggregateByKey[U: ClassTag](zeroValue: U)(seqOp: (U, V) => U,
  combOp: (U, U) => U): RDD[(K, U)]
```

将数据根据不同的规则进行分区内计算和分区间计算。

`aggregateByKey` 存在函数柯里化，有 2 个参数列表：

- 第一个参数列表 `zeroValue` 表示初始值，主要用于碰见第一个 key 的时候用于计算
- 第二个参数列表
  - 第一个参数 `seqOp`：分区内计算规则
  - 第二个参数 `combOp`：分区间计算规则

```scala
val rdd = sc.makeRDD(List(
    ("a", 1), ("a", 2), ("c", 3),
    ("b", 4), ("c", 5), ("c", 6)
  ), 2)
// 0:("a",1),("a",2),("c",3) => (a,10)(c,10)
            //   => (a,10)(b,10)(c,20)
// 1:("b",4),("c",5),("c",6) => (b,10)(c,10)
val resultRDD = rdd.aggregateByKey(10)(
  (x, y) => math.max(x, y),
  (x, y) => x + y)
resultRDD.collect().foreach(println)
```

#### combineByKey

函数签名：

```scala
def combineByKey[C](
  createCombiner: V => C, // 将相同key的第一个数据进行结构的转换，实现操作
  mergeValue: (C, V) => C,  // 分区内的计算规则
  mergeCombiners: (C, C) => C): RDD[(K, C)] // 分区间的计算规则
```

最通用的对 key-value 型 rdd 进行聚集操作的聚集函数 (aggregation function)。类似于 aggregate()，combineByKey() 允许用户返回值的类型与输入不一致。

小练习：将数据 List(("a", 88), ("b", 95), ("a", 91), ("b", 93), ("a", 95), ("b", 98)) 求每个 key 的平均值

```scala
val list: List[(String, Int)] = List(("a", 88), ("b", 95), ("a", 91), ("b", 93), ("a", 95), ("b", 98))
val input: RDD[(String, Int)] = sc.makeRDD(list, 2)
val combineRdd: RDD[(String, (Int, Int))] = input.combineByKey(
  (_, 1),
  (acc: (Int, Int), v) => (acc._1 + v, acc._2 + 1),
  (acc1: (Int, Int), acc2: (Int, Int)) => (acc1._1 + acc2._1, acc1._2 + acc2._2))
```

> [!question] reduceByKey、foldByKey、aggregateByKey、combineByKey 的区别?

| reduceByKey                                    | CombineByKey                                                                             |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `reduceByKey` 在内部调用 `combineByKey`        | `combineByKey` 是通用 API，由 `reduceByKey` 和 `aggregateByKey` 使用                     |
| `reduceByKey` 的输入类型和 outputType 是相同的 | `combineByKey` 更灵活，因此可以提到所需的 outputType。输出类型不一定需要与输入类型相同。 |

reduceByKey：相同 key 的第一个数据不进行任何计算，分区内和分区间计算规则相同

FoldByKey：相同 key 的第一个数据和初始值进行分区内计算，分区内和分区间计算规则相同

AggregateByKey：相同 key 的第一个数据和初始值进行分区内计算，分区内和分区间计算规则可以不相同

CombineByKey：当计算时，发现数据结构不满足要求时，可以让第一个数据转换结构。分区内和分区间计算规则不相同。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241936834.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241936836.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241936033.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241936040.png)

#### join

在类型为 (K,V) 和 (K,W) 的 RDD 上调用，返回一个相同 key 对应的所有元素连接在一起的 (K,(V,W)) 的 RDD

```scala
val rdd: RDD[(Int, String)] = sc.makeRDD(Array((1, "a"), (2, "b"), (3, "c")))
val rdd1: RDD[(Int, Int)] = sc.makeRDD(Array((1, 4), (2, 5), (3, 6)))
rdd.join(rdd1).collect().foreach(println)
```

两个不同数据源的数据，相同的 key 的 value 会连接在一起，形成元组。如果两个数据源中 key 没有匹配上，那么数据不会出现在结果中；如果两个数据源中的 key 有多个相同的，那么会依次匹配，**可能会出现笛卡尔积**，数据量会几何增长。

运行结果：

```
(1,(a,4))
(2,(b,5))
(3,(c,6))
```

#### cogroup

在类型为 (K,V) 和 (K,W) 的 RDD 上调用，返回一个 `(K,(Iterable<V>,Iterable<W>))` 类型的 RDD

cogroup = connect + group

```scala
val rdd1 = sc.makeRDD(List(("a", 1), ("b", 2)))
val rdd2 = sc.makeRDD(List(("a", 4), ("b", 5), ("c", 6), ("c", 7)))

rdd1.cogroup(rdd2).collect().foreach(println)
```

运行结果：

```
(a,(CompactBuffer(1),CompactBuffer(4)))
(b,(CompactBuffer(2),CompactBuffer(5)))
(c,(CompactBuffer(),CompactBuffer(6, 7)))
```

---

### RDD 行动算子

#### reduce

函数签名：

```scala
def reduce(f: (T, T) => T): T
```

聚集 RDD 中的所有元素，先聚合分区内数据，再聚合分区间数据。

#### collect

在驱动程序中，以数组 Array 的形式返回数据集的所有元素。

#### count

返回 RDD 中元素的个数。

#### first

返回 RDD 中的第一个元素。

#### take

返回一个由 RDD 的前 n 个元素组成的数组。

#### takeOrdered

返回该 RDD 排序后的前 n 个元素组成的数组。

#### aggregate

函数签名：

```scala
def aggregate[U: ClassTag]
  (zeroValue: U)
  (seqOp: (U, T) => U,
  combOp: (U, U) => U): U
```

分区的数据通过初始值和分区内的数据进行聚合，然后再和初始值进行分区间的数据聚合。

aggregateByKey：初始值只会参与分区内的计算
aggregate：初始值会参与分区内的计算，并且参与分区间计算

aggregate 是行动算子，会直接返回计算结果，而不是返回一个 RDD。

```scala
val rdd: RDD[Int] = sc.makeRDD(List(1, 2, 3, 4), 8)

// 将该 RDD 所有元素相加得到结果
//val result: Int = rdd.aggregate(0)(_ + _, _ + _)
val result: Int = rdd.aggregate(10)(_ + _, _ + _)
```

#### fold

折叠操作，aggregate 的简化版操作。当分区内和分区间的计算逻辑相同时，就可以用 fold 代替 aggregate。

#### countByKey

函数签名：

```scala
def countByKey(): Map[K, Long]
```

统计每种 key 的个数。

#### foreach

分布式遍历 RDD 中的每一个元素，调用指定函数。

---

## WordCount 的多种实现方式

```scala
object WordCount {
  def main(args: Array[String]): Unit = {
    val sparkConf = new SparkConf().setMaster("local[*]").setAppName("WordCount")
    val sc = new SparkContext(sparkConf)
    wordCount1(sc)
    sc.stop()
  }

  // group
  def wordCount1(sc: SparkContext): Unit = {
    val rdd = sc.makeRDD(List("Hello Scala", "Hello Spark"))
    val words = rdd.flatMap(_.split(" "))
    val group: RDD[(String, Iterable[String])] = words.groupBy(word => word)
    val wordCount = group.mapValues(iter => iter.size)
  }

  //groupByKey，但性能低
  def wordCount2(sc: SparkContext): Unit = {
    val rdd = sc.makeRDD(List("Hello Scala", "Hello Spark"))
    val words = rdd.flatMap(_.split(" "))
    val wordOne = words.map((_, 1))
    val group: RDD[(String, Iterable[Int])] = wordOne.groupByKey()
    val wordCount = group.mapValues(iter => iter.size)
  }

  //reduceByKey
  def wordCount3(sc: SparkContext): Unit = {
    val rdd = sc.makeRDD(List("Hello Scala", "Hello Spark"))
    val words = rdd.flatMap(_.split(" "))
    val wordOne = words.map((_, 1))
    val wordCount = wordOne.reduceByKey(_ + _)
  }

  //reduceByKey
  def wordCount3(sc: SparkContext): Unit = {
    val rdd = sc.makeRDD(List("Hello Scala", "Hello Spark"))
    val words = rdd.flatMap(_.split(" "))
    val wordOne = words.map((_, 1))
    val wordCount = wordOne.reduceByKey(_ + _)
  }

  //aggregateByKey
  def wordCount4(sc: SparkContext): Unit = {
    val rdd = sc.makeRDD(List("Hello Scala", "Hello Spark"))
    val words = rdd.flatMap(_.split(" "))
    val wordOne = words.map((_, 1))
    val wordCount = wordOne.aggregateByKey(0)(_ + _, _ + _)
  }

  //foldByKey
  def wordCount5(sc: SparkContext): Unit = {
    val rdd = sc.makeRDD(List("Hello Scala", "Hello Spark"))
    val words = rdd.flatMap(_.split(" "))
    val wordOne = words.map((_, 1))
    val wordCount = wordOne.foldByKey(0)(_ + _)
  }

  //combineByKey
  def wordCount6(sc: SparkContext): Unit = {
    val rdd = sc.makeRDD(List("Hello Scala", "Hello Spark"))
    val words = rdd.flatMap(_.split(" "))
    val wordOne = words.map((_, 1))
    val wordCount = wordOne.combineByKey(
      v => v,
      (x: Int, y) => x + y,
      (x: Int, y: Int) => x + y
    )
  }

  //countByKey
  def wordCount7(sc: SparkContext): Unit = {
    val rdd = sc.makeRDD(List("Hello Scala", "Hello Spark"))
    val words = rdd.flatMap(_.split(" "))
    val wordOne = words.map((_, 1))
    val wordCount: collection.Map[String, Long] = wordOne.countByKey()
  }

  //countByValue
  def wordCount8(sc: SparkContext): Unit = {
    val rdd = sc.makeRDD(List("Hello Scala", "Hello Spark"))
    val words = rdd.flatMap(_.split(" "))
    val wordCount: collection.Map[String, Long] = words.countByValue()
  }
}
```

## RDD 序列化

闭包检测：

从计算的角度, 算子以外的代码都是在 Driver 端执行, 算子里面的代码都是在 Executor 端执行。那么在 scala 的函数式编程中，就会导致算子内经常会用到算子外的数据，这样就形成了闭包的效果，如果使用的算子外的数据无法序列化，就意味着无法传值给 Executor 端执行，就会发生错误，所以需要在执行任务计算前，检测闭包内的对象是否可以进行序列化，这个操作我们称之为**闭包检测**。

序列化方法和属性：

从计算的角度, 算子以外的代码都是在 Driver 端执行, 算子里面的代码都是在 Executor 端执行。

```scala
object RDD_Serialize {
  def main(args: Array[String]): Unit = {
    val sparkConf = new SparkConf().setMaster("local[*]").setAppName("Serialize")
    val sc = new SparkContext(sparkConf)

    val rdd = sc.makeRDD(List(1, 2, 3, 4))
    val user = new User()

    rdd.foreach(
      num => {
        println("age = " + (user.age + num))
      }
    )

    sc.stop()
  }

  class User {
    var age: Int = 30
  }
}
```

上面的代码会报 Task not serializable 异常，原因是 user 对象没有序列化。解决方法一是在 class User 后面混入 Serializable 特质。另一种方法是用样例类代替 class。样例类会在编译的时候自动完成序列化。

```scala
case class User() {
  var age: Int = 30
}
```

Kryo 序列化框架：

Java 的序列化能够序列化任何的类。但是比较重 (字节多)，序列化后，对象的提交也比较大。Spark 出于性能的考虑，Spark2.0 开始支持另外一种 Kryo 序列化机制。Kryo 速度是 Serializable 的 10 倍。当 RDD 在 Shuffle 数据的时候，简单数据类型、数组和字符串类型已经在 Spark 内部使用 Kryo 来序列化。

注意：即使使用 Kryo 序列化，也要继承 Serializable 接口。

## RDD 依赖关系

相邻的两个 RDD 的关系被称为**依赖关系**。多个连续的 RDD 的依赖关系被称为**血缘关系**。

每个 RDD 都会保存血缘关系。由于 RDD 是不会保存数据的（不落盘，通过内存传递），因此为了提供容错性，需要将 RDD 间的关系保存下来，一旦出现错误，可以根据血缘关系重新读取进行计算。

> RDD 只支持粗粒度转换，即在大量记录上执行的单个操作。将创建 RDD 的一系列 Lineage (血统) 记录下来，以便恢复丢失的分区。RDD 的 Lineage 会记录 RDD 的元数据信息和转换行为，当该 RDD 的部分分区数据丢失时，它可以根据这些信息来重新运算和恢复丢失的数据分区。

通过每次生成 RDD 之后调用 `rdd.toDebugString` 即可查看当前 RDD 的血缘关系。通过调用 `rdd.dependencies` 可以查看当前 RDD 的依赖关系。

如果新的 RDD 的一个分区的数据依赖于旧的 RDD 的一个分区的数据，这种依赖被称为 **OneToOne 依赖**，也叫**窄依赖**。

如果新的 RDD 的一个分区的数据依赖于旧的 RDD 的多个分区的数据，这种依赖被称为 **Shuffle 依赖**，也叫**宽依赖**。

> 窄依赖表示每一个父（上游）RDD 的 Partition 最多被子（下游）RDD 的一个 Partition 使用，窄依赖我们形象的比喻为独生子女。

> 宽依赖表示同一个父（上游）RDD 的 Partition 被多个子（下游）RDD 的 Partition 依赖，会引起 Shuffle，总结：宽依赖我们形象的比喻为多生。

## RDD 阶段划分

DAG(Directed Acyclic Graph) 有向无环图是由点和线组成的拓扑图形，该图形具有方向， 不会闭环。DAG 记录了 RDD 的转换过程和任务的阶段（stage）。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241936236.png)

RDD 的阶段是根据 shuffle 来划分的。阶段的数量 = shuffle 依赖的数量 + 1（ResultStage）。

RDD 任务切分中间分为：Application、Job、Stage 和 Task。

- Application：初始化一个 SparkContext 即生成一个 Application
- Job：一个 Action 算子就会生成一个 Job
- Stage：Stage 等于宽依赖 (ShuffleDependency) 的个数加 1
- Task：一个 Stage 阶段中，最后一个 RDD 的分区个数就是 Task 的个数

Application -> Job -> Stage -> Task 每一层都是 1 对 n 的关系。

## RDD 持久化

### RDD Cache & Persist

RDD 通过 Cache 或者 Persist 方法将前面的计算结果缓存，默认情况下会把数据以缓存在 JVM 的堆内存中。但是并不是这两个方法被调用时立即缓存，而是触发后面的 action 算子时，该 RDD 将会被缓存在计算节点的内存中，并供后面重用。

```scala
// cache 操作会增加血缘关系，不改变原有的血缘关系
println(rdd.toDebugString)
// 数据缓存。
rdd.cache()
// 可以更改存储级别
rdd.persist(StorageLevel.MEMORY_AND_DISK_2)
```

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241936143.png)

缓存有可能丢失，或者存储于内存的数据由于内存不足而被删除，RDD 的缓存容错机制保证了即使缓存丢失也能保证计算的正确执行。通过基于 RDD 的一系列转换，丢失的数据会被重算，由于 RDD 的各个 Partition 是相对独立的，因此只需要计算丢失的部分即可， 并不需要重算全部 Partition。

Spark 会自动对一些 Shuffle 操作的中间数据做持久化操作（比如 reduceByKey）。这样做的目的是为了当一个节点 Shuffle 失败了避免重新计算整个输入。但是，在实际使用的时候，如果想重用数据，仍然建议调用 persist 或 cache。

### RDD Checkpoint

所谓的检查点其实就是通过将 RDD 中间结果写入磁盘由于血缘依赖过长会造成容错成本过高，这样就不如在中间阶段做检查点容错，如果检查点之后有节点出现问题，可以从检查点开始重做血缘，减少了开销。

对 RDD 进行 checkpoint 操作并不会马上被执行，必须执行 Action 操作才能触发。

Checkpoint 需要落盘，因此需要指定检查点保存路径（通常都是 HDFS）。检查点路径保存的数据在作业执行完毕后，不会被删除，但是 cache 会被删除。

### Cache, persist 和 checkpoint 的区别

cache：将数据临时存储在内存中进行数据重用
persist：将数据临时存储在磁盘文件中进行数据重用，涉及到磁盘 IO，性能较低，但更安全

以上 2 种方式在作业执行完毕后，临时保存的数据就会丢失。

checkpoint：将数据长久地保存在磁盘文件中进行数据重用，涉及磁盘 IO，性能较低，但更安全。为了保证数据安全，所以一般情况下会独立执行作业。为了能够保证效率，**一般情况下，需要和 cache 联合使用**。

- Cache 缓存只是将数据保存起来，不切断血缘依赖，而在血缘关系中添加新的依赖，一旦出现问题可以从头读取数据。Checkpoint 检查点切断血缘依赖，重新建立新的血缘关系。
- Cache 缓存的数据通常存储在磁盘、内存等地方，可靠性低。Checkpoint 的数据通常存储在 HDFS 等容错、高可用的文件系统，可靠性高。
- 建议对 checkpoint() 的 RDD 使用 Cache 缓存，这样 checkpoint 的 job 只需从 Cache 缓存中读取数据即可，否则需要再从头计算一次 RDD。

## RDD 分区器

Spark 目前支持 Hash 分区、Range 分区和用户自定义分区。Hash 分区为当前的默认分区。分区器直接决定了 RDD 中分区的个数、RDD 中每条数据经过 Shuffle 后进入哪个分区，进而决定了 Reduce 的个数。

- 只有 Key-Value 类型的 RDD 才有分区器，非 Key-Value 类型的 RDD 分区的值是 None
- 每个 RDD 的分区 ID 范围：0 ~ (numPartitions - 1)，决定这个值是属于那个分区的

Hash 分区：对于给定的 key，计算其 hashCode，并除以分区个数取余

Range 分区：将一定范围内的数据映射到一个分区中，尽量保证每个分区数据均匀，而且分区间有序

---

## DataFrame & DataSet

在 Spark 中，**DataFrame 是一种以 RDD 为基础的分布式数据集**，类似于传统数据库中的**二维表格**。DataFrame 与 RDD 的主要区别在于，前者带有 **schema 元信息**，即 DataFrame 所表示的二维表数据集的每一列都带有名称和类型。这使得 Spark SQL 得以洞察更多的结构信息，从而对藏于 DataFrame 背后的数据源以及作用于 DataFrame 之上的变换进行了针对性的优化，最终达到大幅提升运行时效率的目标。反观 RDD，由于无从得知所存数据元素的具体内部结构，Spark Core 只能在 stage 层面进行简单、通用的流水线优化。

DataSet 是分布式数据集合。DataSet 是 Spark 1.6 中添加的一个新抽象，是 DataFrame 的一个扩展。它提供了 RDD 的优势 (强类型，使用强大的 lambda 函数的能力) 以及 Spark SQL 优化执行引擎的优点。DataSet 也可以使用功能性的转换 (操作 map，flatMap，filter 等等)。

- DataSet 是 DataFrameAPI 的一个扩展，是 SparkSQL 最新的数据抽象
- 用户友好的 API 风格，既具有类型安全检查也具有 DataFrame 的查询优化特性
- 用样例类来对 DataSet 中定义数据的结构信息，样例类中每个属性的名称直接映射到 DataSet 中的字段名称
- DataSet 是强类型的。比如可以有 `DataSet[Car]`，`DataSet[Person]`
- DataFrame 是 DataSet 的特列，`DataFrame=DataSet[Row]`，所以可以通过 as 方法将 DataFrame 转换为 DataSet。Row 是一个类型，跟 Car、Person 这些的类型一样，所有的表结构信息都用 Row 来表示。获取数据时需要指定顺序

> [!question] RDD、DataFrame、DataSet 三者的关系

如果同样的数据都给到这三个数据结构，他们分别计算之后，都会给出相同的结果。不同是的他们的执行效率和执行方式。在后期的 Spark 版本中，DataSet 有可能会逐步取代 RDD 和 DataFrame 成为唯一的 API 接口。

共同点：

- RDD、DataFrame、DataSet 全都是 spark 平台下的分布式弹性数据集
- 三者都有惰性机制，在进行创建、转换，如 map 方法时，不会立即执行，只有在遇到 Action 如 foreach 时，三者才会开始遍历运算
- 三者有许多共同的函数，如 filter，排序等;
- 三者都会根据 Spark 的内存情况自动缓存运算，这样即使数据量很大，也不用担心会内存溢出
- 三者都有 partition 的概念
- DataFrame 和 DataSet 均可使用模式匹配获取各个字段的值和类型

不同点：

RDD：

- RDD 一般和 sparkmllib 同时使用
- RDD 不支持 sparksql 操作

DataFrame：

- 与 RDD 和 Dataset 不同，DataFrame 每一行的类型固定为 Row，每一列的值没法直接访问，只有通过解析才能获取各个字段的值
- DataFrame 与 DataSet 一般不与 spark mllib 同时使用
- DataFrame 与 DataSet 均支持 SparkSQL 的操作，比如 select，groupby 之类，还能注册临时表/视窗，进行 sql 语句操作
- DataFrame 与 DataSet 支持一些特别方便的保存方式，比如保存成 csv，可以带上表头，这样每一列的字段名一目了然

DataSet：

- Dataset 和 DataFrame 拥有完全相同的成员函数，区别只是每一行的数据类型不同。DataFrame 其实就是 DataSet 的一个特例 `type DataFrame = Dataset[Row]`
- DataFrame 也可以叫 `Dataset[Row]`，每一行的类型是 Row，不解析，每一行究竟有哪些字段，各个字段又是什么类型都无从得知，只能用上面提到的 getAS 方法或者共性中的第七条提到的模式匹配拿出特定字段。而 Dataset 中，每一行是什么类型是不一定的，在自定义了 case class 之后可以很自由的获得每一行的信息

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241937078.png)

## Spark Session

Spark Core 中，如果想要执行应用程序，需要首先构建上下文环境对象 **SparkContext**， Spark SQL 其实可以理解为对 Spark Core 的一种封装，不仅仅在模型上进行了封装，上下文环境对象也进行了封装。**SparkSession** 是 Spark 最新的 SQL 查询起始点。

```scala
// 创建运行环境
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("SparkSQL")
val spark = SparkSession.builder().config(sparkConf).getOrCreate()
import spark.implicits._
```

## UDF & UDAF

用户可以通过 spark.udf 功能添加自定义函数，实现自定义功能。

```scala
object UDF {
  def main(args: Array[String]): Unit = {
    // 创建运行环境
    val sparkConf = new SparkConf().setMaster("local[*]").setAppName("SparkSQL")
    val spark = SparkSession.builder().config(sparkConf).getOrCreate()
    import spark.implicits._

    val df = spark.read.json("data/user.json")
    df.createOrReplaceTempView("user")

    spark.udf.register("prefixName", (name: String) => {
      "Name: " + name
    })

    spark.sql("select age, prefixName(username) from user").show

    //关闭环境
    spark.close()
  }
}
```

强类型的 Dataset 和弱类型的 DataFrame 都提供了相关的聚合函数， 如 count()， countDistinct()，avg()，max()，min()。除此之外，用户可以设定自己的自定义聚合函数。通过继承 UserDefinedAggregateFunction 来实现用户自定义弱类型聚合函数。从 Spark3.0 版本后，UserDefinedAggregateFunction 已经不推荐使用了。可以统一采用强类型聚合函数 Aggregator。
