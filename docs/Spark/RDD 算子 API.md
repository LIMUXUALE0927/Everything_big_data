![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311202125210.png)

## RDD 转换算子

RDD 根据数据处理方式的不同将算子整体上分为 Value 类型、双 Value 类型和 Key-Value 类型。

### map

`map` ：将处理的数据逐条进行映射转换，这里的转换可以是类型的转换，也可以是值的转换。

```scala
val rdd = sc.parallelize(List(1, 2, 3, 4, 5)).map(_ * 2)
rdd.foreach(println)
```

`map` 并行计算效果：

RDD 的计算：==一个分区内的数据是串行，只有前面一个数据全部的逻辑执行完毕后，才会执行下一个数据。因此，分区内数据的执行是有序的。不同分区是并行执行，因此顺序不定。==

```scala
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

---

### mapPartitions

map 的输入函数应用于 RDD 中的每个元素，而 mapPartitions 的输入函数应用于每个分区。因此，如果 RDD 中有 N 个元素，有 M 个分区，那么 map 的函数将被调用 N 次，而 mapPartitions 函数将被调用 M 次，每次调用都会处理一个分区的数据。**mapPartitions 把每个分区中的内容作为整体来处理的，因此减少了总的处理数据的次数。**

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312112003249.png)

计算每个分区奇数的和与偶数的和：

```scala
val rdd = sc.parallelize(1 to 9, 3).mapPartitions(iter => {
  var result = List[Int]()
  var odd = 0
  var even = 0
  while (iter.hasNext) {
    val cur = iter.next()
    if (cur % 2 == 0) {
      even += cur
    } else {
      odd += cur
    }
  }
  result = result :+ odd :+ even
  result.iterator
})

rdd.glom().foreach(arr => {
  println(arr.mkString(","))
})
```

执行结果：

```
4,2
5,10
16,8
```

mapPartitions 更像是过程式编程，给定了一组数据后，可以使用数据结构持有中间处理结果，也可以输出任意大小、任意类型的一组数据。mapPartitions 还可以用来实现数据库操作。例如，在 mapPartitions 中先建立数据库连接，然后将每一个新来的数据 `iter.next()` 转化为数据表中的一行，并插入数据库中。然而 map 并不能完成这样的操作，因为 map 是对 RDD 中的每一个元素进行操作，这样会频繁建立数据库连接，效率低下。

mapPartitions 可以以分区为单位进行数据转换操作，但是会将整个分区的数据加载到内存进行引用，处理完的数据是不会被释放的，存在对象的引用，只有程序结束才会释放。因此在内存较小、数据量较大的场合下，容易导致 OOM。

:star: mapPartitions 的小功能：获取每个分区的最大值。

```scala
val mpRDD = rdd.mapPartitions(
  iter => {
    List(iter.max).iterator
  }
)
```

!!! question "map 和 mapPartitions 的区别"

数据处理角度：

Map 算子是分区内一个数据一个数据执行，类似于**串行操作**。而 mapPartitions 算子是以分区为单位进行**批处理操作**。

功能的角度：

Map 算子主要目的将数据源中的数据进行转换和改变。但是不会减少或增多数据。MapPartitions 算子需要传递一个迭代器，返回一个迭代器，没有要求的元素的个数保持不变， 所以可以增加或减少数据。

性能的角度：

Map 算子因为类似于串行操作，所以性能比较低，而是 mapPartitions 算子类似于批处理，所以性能较高。但是 mapPartitions 算子会长时间占用内存，那么这样会导致 OOM 错误。所以在内存有限的情况下，不推荐使用。

---

### mapPartitionsWithIndex

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

---

### flatMap

将处理的数据进行扁平化后再进行映射处理，所以也被称为扁平映射。参数是 map 函数。

```scala
val rdd = sc.parallelize(List(List(1, 2), List(3, 4), List(5, 6))).flatMap(x => x)
rdd.foreach(println)
```

```scala
// 先将 5 转化为 List(5)，然后再进行 flatMap
val rdd = sc.parallelize(List(List(1, 2), List(3, 4), 5)).flatMap(x => {
  x match {
    case list: List[Int] => list
    case num: Int => List(num)
  }
})
rdd.foreach(println)

// 等价于
val rdd = sc.parallelize(List(List(1, 2), List(3, 4), 5)).flatMap {
  case list: List[Int] => list
  case num: Int => List(num)
}
rdd.foreach(println)
```

---

### glom

将 RDD 中每个分区的数据合并到一个 list 中。

```scala
val rdd: RDD[Int] = sc.makeRDD(List(1, 2, 3, 4), 2)
val glomRDD: RDD[Array[Int]] = rdd.glom()
glomRDD.foreach(data => println(data.mkString(",")))
```

运行结果：

```
1,2
3,4
```

:star: 小功能：计算所有分区最大值求和（分区内取最大值，分区间最大值求和）

```scala
val rdd = sc.makeRDD(List(1, 2, 3, 4), 2)
val glomRDD: RDD[Array[Int]] = rdd.glom()
val maxRDD = glomRDD.map(array => array.max)
println(maxRDD.collect().sum)
```

```scala
val rdd = sc.makeRDD(1 to 10, 2)
val sum = rdd.mapPartitions(iter => {
  List(iter.max).iterator
}).collect().sum
println(sum)
```

---

### groupBy

==将数据根据指定的规则进行分组，分区数量不会变，但是数据会被打乱，然后重新组合，我们将这样的操作称之为 shuffle。==

极限情况下，数据可能被分在同一个分区中。

一个组的数据在一个分区中，但是并不是说一个分区中只有一个组。多个组是有可能放在同一个分区的。

```scala
val rdd = sc.makeRDD(List(1, 2, 3, 4), 2)
val groupRDD = rdd.groupBy(x => x % 2)
groupRDD.foreach(println)
```

运行结果：

```
(0,CompactBuffer(2, 4))
(1,CompactBuffer(1, 3))
```

---

### filter

将数据根据指定的规则进行筛选过滤，符合规则的数据保留，不符合规则的数据丢弃。

当数据进行筛选过滤后，分区不变，但是分区内的数据可能不均衡，生产环境下，可能会出现**数据倾斜**。

---

### sample

函数签名：

```scala
def sample(
  withReplacement: Boolean,
  fraction: Double,
  seed: Long = Utils.random.nextLong): RDD[T]
```

根据指定的规则从数据集中抽取数据，常用于判断**数据倾斜**。

---

### distinct

函数签名：

```scala
def distinct()(implicit ord: Ordering[T] = null): RDD[T]
def distinct(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T]
```

底层通过 `reduceByKey` 实现：

默认情况下是通过 `map(x => (x, null)).reduceByKey((x, _) => x, numPartitions).map(_._1)` 实现的。

即：

```scala
List(1, 2, 1, 2)
((1, null), (2, null), (1, null), (2, null)) => ((1, null), (2, null)) => (1, 2)
```

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

---

### coalesce

函数签名：

```scala
coalesce(numPartitions: Int, shuffle: Boolean = false)(implicit ord: Ordering[T] = null): RDD[T]
```

根据数据量**缩减分区**，用于大数据集过滤后，提高小数据集的执行效率。
当 spark 程序中存在过多的小任务的时候，可以通过 coalesce 方法，收缩合并分区，减少分区的个数，减小任务调度成本。

```scala
val dataRDD = sc.makeRDD(List(1, 2, 3, 4, 5, 6), 3)
val dataRDD1 = dataRDD.coalesce(2)
```

:star: `coalesce` 在默认情况下会将相邻的分区直接合并在一起。在上面的例子中，dataRDD 有 3 个分区，每个分区分别是 (1,2), (3, 4), (5, 6)，但是 `coalesce` 之后 dataRDD1 中 2 个分区变成 (1, 2), (3, 4, 5, 6)，也就容易导致**数据倾斜**。

如果想让数据均衡，可以进行 **shuffle** 处理。

!!! question "如果想要扩大分区该怎么做？"

    coalesce 算子可以扩大分区，但是必须进行 shuffle 操作。但是对于这种情况，可以直接使用 repartition 算子。

---

### repartition

语义与 `coalesce(numPartitions, shuffle = true)` 相同。

该操作内部其实执行的是 coalesce 操作，参数 shuffle 的默认值为 true。无论是将分区数多的 RDD 转换为分区数少的 RDD，还是将分区数少的 RDD 转换为分区数多的 RDD，repartition 操作都可以完成，因为无论如何都会经过 shuffle 过程。

---

### sortBy

函数签名：

```scala
def sortBy[K](
  f: (T) => K,
  ascending: Boolean = true,
  numPartitions: Int = this.partitions.length)
  (implicit ord: Ordering[K], ctag: ClassTag[K]): RDD[T]
```

该操作用于排序数据。在排序之前，可以将数据通过 f 函数进行处理，之后按照 f 函数处理的结果进行排序，默认为升序排列。排序后新产生的 RDD 的分区数与原 RDD 的分区数一致。中间存在 **shuffle** 的过程。

```scala
val rdd = sc.makeRDD(List(("e", 5), ("e", 4), ("e", 1), ("a", 1), ("b", 2), ("c", -3), ("d", 4)))
rdd.sortBy(tuple => (tuple._1, -tuple._2)).collect().foreach(println)
```

```
(a,1)
(b,2)
(c,-3)
(d,4)
(e,5)
(e,4)
(e,1)
```

---

### partitionBy

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

!!! question "paritionBy 和 repartition 的区别？"

在 Spark 中，`repartition` 和 `partitionBy` 都是重新分区的算子，其中 `partitionBy` 只能作用于 PairRDD (K, V)。但是，当作用于 PairRDD 时，`repartition` 和 `partitionBy` 的行为是不同的。`repartition` 是把数据随机打散均匀分布于各个 Partition（直接对数据进行 shuffle）；而 `partitionBy` 则在参数中指定了 Partitioner（默认 HashPartitioner），将每个 (K,V) 对按照 K 根据 Partitioner 计算得到对应的 Partition。**在合适的时候使用 `partitionBy` 可以减少 shuffle 次数，提高效率。**

---

### groupByKey

将数据源的数据根据 key 对 value 进行分组。reduceByKey = groupByKey + reduce。

`groupByKey` 和 `groupBy(_._1)` 的区别主要是返回值类型。

```scala
val rdd = sc.makeRDD(List(("a", 1), ("a", 2), ("a", 3), ("b", 4)))

val groupRDD: RDD[(String, Iterable[Int])] = rdd.groupByKey()
val groupRDD1: RDD[(String, Iterable[(String, Int)])] = rdd.groupBy(_._1)
```

**groupByKey 在不同情况下会生成不同类型的 RDD 和数据依赖关系**。如果 RDD1 调用 groupByKey 生成 RDD2，那么 RDD2 的类型和数据依赖关系如下：

- 如果 RDD1 和 RDD2 的分区器不同，会产生 Shuffle Dependency
- 如果 RDD1 和 RDD2 的分区器相同，并且分区数相同，那么就不需要 Shuffle Dependency

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312112008266.png)

`groupByKey` 由于会根据 key 对数据进行分组，因此可能导致 **shuffle** 过程中产生大量中间数据、占用内存大，容易导致 OOM。

!!! question "groupByKey 和 reduceByKey 的区别？"

从功能的角度：`reduceByKey` 其实包含分组和聚合的功能，而 `groupByKey` 只能分组，不能聚合。

从 shuffle 的角度：`reduceByKey` 和 `groupByKey` 由于分组都存在 shuffle 的操作，但是 `reduceByKey` 可以在 shuffle 前对分区内相同 key 的数据进行**预聚合 (combine)** 功能，**这样能够减少落盘的数据量**，而 `groupByKey` 只是进行分组，不存在数据量减少的问题，reduceByKey 性能比较高。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241936831.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202307241936142.png)

---

### reduceByKey

```scala
def reduceByKey(func: (V, V) => V): RDD[(K, V)]
def reduceByKey(func: (V, V) => V, numPartitions: Int): RDD[(K, V)]
```

可以将数据按照相同的 key 对 value 进行聚合。如果 Key 的数据只有一个，是不会参与计算的。

`reduceByKey` 要求分区内（combine）和分区间的聚合规则是相同的。如果分区内和分区间要按照不同规则计算，则要使用 `aggregateByKey` 。（如分区内求最大值，分区间求和）

---

### aggregateByKey

函数签名：

```scala
def aggregateByKey[U: ClassTag](zeroValue: U)(seqOp: (U, V) => U,
  combOp: (U, U) => U): RDD[(K, U)]
```

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312112009299.png)

通用版的 `reduceByKey`，**将数据根据不同的规则进行分区内计算和分区间计算。**

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

---

### combineByKey

函数签名：

```scala
def combineByKey[C](
  createCombiner: V => C, // 将相同key的第一个数据进行结构的转换，实现操作
  mergeValue: (C, V) => C,  // 分区内的计算规则
  mergeCombiners: (C, C) => C): RDD[(K, C)] // 分区间的计算规则
```

combineByKey()是一个通用的基础聚合操作。常用的聚合操作，如 aggregateByKey()、reduceByKey()都是利用 combineByKey()实现的。

**aggregateByKey() 和 combineByKey() 唯一的区别：combineByKey() 的 createCombiner 是一个初始化函数，而 aggregateByKey() 的 zeroValue 是一个初始值**。比如：combineByKey() 可以根据每个 record 的 Value 值为每个 record 定制初始值。

小练习：将数据 List(("a", 88), ("b", 95), ("a", 91), ("b", 93), ("a", 95), ("b", 98)) 求每个 key 的平均值

```scala
val list: List[(String, Int)] = List(("a", 88), ("b", 95), ("a", 91), ("b", 93), ("a", 95), ("b", 98))
val input: RDD[(String, Int)] = sc.makeRDD(list, 2)
val combineRdd: RDD[(String, (Int, Int))] = input.combineByKey(
  (_, 1),
  (acc: (Int, Int), v) => (acc._1 + v, acc._2 + 1),
  (acc1: (Int, Int), acc2: (Int, Int)) => (acc1._1 + acc2._1, acc1._2 + acc2._2))
```

---

### foldByKey

```scala
def foldByKey(zeroValue: V, numPartitions: Int)(func: (V, V) => V): RDD[(K, V)]
```

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312112015625.png)

foldByKey 是 aggregateByKey 的简化操作，将 aggregateByKey 中的分区内和分区间计算规则相同的情况进行简化，同时多了初始值 zeroValue。

#### 总结

!!! question "reduceByKey、foldByKey、aggregateByKey、combineByKey 的区别?"

| reduceByKey                                    | combineByKey                                                                             |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `reduceByKey` 在内部调用 `combineByKey`        | `combineByKey` 是通用 API，由 `reduceByKey` 和 `aggregateByKey` 使用                     |
| `reduceByKey` 的输入类型和 outputType 是相同的 | `combineByKey` 更灵活，因此可以提到所需的 outputType。输出类型不一定需要与输入类型相同。 |

- reduceByKey：没有初始值，分区内和分区间计算规则相同

- foldByKey：有初始值的 reduceByKey，分区内和分区间计算规则相同

- aggregateByKey：有初始值，分区内和分区间计算规则可以不同

- combineByKey：最通用的聚合操作。有初始化函数，发现数据结构不满足要求时，可以让第一个数据转换结构。分区内和分区间计算规则可以不同。

---

### join

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

---

### cogroup

在类型为 (K,V) 和 (K,W) 的 RDD 上调用，返回一个 `(K,(Iterable<V>,Iterable<W>))` 类型的 RDD

cogroup = connect + group

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312112029894.png)

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

## RDD 行动算子

### reduce

函数签名：

```scala
def reduce(f: (T, T) => T): T
```

聚集 RDD 中的所有元素，先聚合分区内数据，再聚合分区间数据。

### collect

在驱动程序中，以数组 Array 的形式返回数据集的所有元素。

### count

返回 RDD 中元素的个数。

### first

返回 RDD 中的第一个元素。

### take

返回一个由 RDD 的前 n 个元素组成的数组。

### takeOrdered

返回该 RDD 排序后的前 n 个元素组成的数组。

### aggregate

函数签名：

```scala
def aggregate[U: ClassTag]
  (zeroValue: U)
  (seqOp: (U, T) => U,
  combOp: (U, U) => U): U
```

分区的数据通过初始值和分区内的数据进行聚合，然后再和初始值进行分区间的数据聚合。

- aggregateByKey：初始值只会参与分区内的计算

- aggregate：初始值会参与分区内的计算，并且参与分区间计算

aggregate 是行动算子，会直接返回计算结果，而不是返回一个 RDD。

```scala
val rdd: RDD[Int] = sc.makeRDD(List(1, 2, 3, 4), 8)

// 将该 RDD 所有元素相加得到结果
//val result: Int = rdd.aggregate(0)(_ + _, _ + _)
val result: Int = rdd.aggregate(10)(_ + _, _ + _)
```

### fold

折叠操作，aggregate 的简化版操作。当分区内和分区间的计算逻辑相同时，就可以用 fold 代替 aggregate。

### countByKey

函数签名：

```scala
def countByKey(): Map[K, Long]
```

统计每种 key 的个数。

### foreach

分布式遍历 RDD 中的每一个元素，调用指定函数。

### save 相关

- saveAsTextFile
- saveAsSequenceFile
- saveAsObjectFile

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
