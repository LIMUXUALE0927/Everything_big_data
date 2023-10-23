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

## NTILE

NTILE(n)，用于将分组数据按照顺序切分成 n 片，返回当前切片值。如果切片不均匀，默认增加第一个切片的分布。

```sql
SELECT *
FROM geeks_demo;
```

| ID  |
| --- |
| 1   |
| 2   |
| 3   |
| 4   |
| 5   |
| 6   |
| 7   |
| 8   |
| 9   |
| 10  |

```sql
SELECT ID,
NTILE (3) OVER (
ORDER BY ID
) Group_number
FROM geeks_demo;
```

| ID  | Group_number |
| --- | ------------ |
| 1   | 1            |
| 2   | 1            |
| 3   | 1            |
| 4   | 1            |
| 5   | 2            |
| 6   | 2            |
| 7   | 2            |
| 8   | 3            |
| 9   | 3            |
| 10  | 3            |

```sql
SELECT
	category_name,
	month,
	FORMAT(net_sales,'C','en-US') net_sales,
	NTILE(4) OVER(
		PARTITION BY category_name
		ORDER BY net_sales DESC
	) net_sales_group
FROM
	sales.vw_netsales_2017;
```

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310231654384.png)

---

## LAG

LAG(col, n, DEFAULT) 用于统计窗口内往上第 n 行值。

第一个参数为列名，第二个参数为往上第 n 行（可选，默认为 1），第三个参数 为默认值（当往上第 n 行为 NULL 时候，取默认值，如不指定，则为 NULL）。
