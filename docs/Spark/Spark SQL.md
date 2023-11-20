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
