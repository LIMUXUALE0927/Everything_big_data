## UDF

一对一的输入输出，只能输入一条记录当中的数据，同时返回一条处理结果。属于最常见的自定义函数。Hive 的 UDF 有两种实现方式或者实现的 API，一种是 UDF 比较简单，一种是 GenericUDF 比较复杂。

如果所操作的数据类型都是基础数据类型，如（Hadoop&Hive 基本 writable 类型，如 Text, IntWritable, LongWriable, DoubleWritable 等）。那么简单的 `org.apache.hadoop.hive.ql.exec.UDF` 就可以做到。 具体地，需要继承 `org.apache.hadoop.hive.ql.UDF`，并实现 evaluate 方法。

如果所操作的数据类型是内嵌数据结构，如 Map，List 和 Set，那么要采用 `org.apache.hadoop.hive.ql.udf.generic.GenericUDF`。 具体地，需要继承 `org.apache.hadoop.hive.ql.udf.generic.GenericUDF`，并实现三个方法：

1. initialize：只调用一次，在任何 evaluate() 调用之前可以接收到一个可以表示函数输入参数类型的 ObjectInspectors 数组。initialize 用来验证该函数是否接收正确的参数类型和参数个数，最后提供最后结果对应的数据类型。
2. evaluate：真正的逻辑，读取输入数据，处理数据，返回结果。
3. getDisplayString：返回描述该方法的字符串，没有太多作用。

---

## UDAF

实现方式：

1. 自定义一个 Java 类，继承 UDAF 类
2. 内部定义一个静态类，实现 UDAFEvaluator 接口
3. 实现 init, iterate, terminatePartial, merge, terminate，共 5 个方法

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310200232537.png)

---
