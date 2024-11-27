# Spark 性能优化

## 任务优化 SOP

实习期间做了很多数据治理和任务优化的工作，以下是我关于任务优化的思路总结、复盘：

**行动之前**

: **优化前先要思考优化的核心目标是什么？**木桶理论（找短板）

| 问题描述                     | 解决方向                     |
| ---------------------------- | ---------------------------- |
| 单纯某个主题/字段延迟产出    | SLA 治理：排查卡点、模型解耦 |
| 任务运行不稳定，频繁重试     | 排查 OOM、倾斜、资源等问题   |
| 任务运行时间长，寻求优化空间 | 从代码层面和参数层面进行优化 |

**I. 代码层面的优化**

| 优化方向      | 具体操作                                                  |
| ------------- | --------------------------------------------------------- |
| 减少数据量    | 分区裁剪、列裁剪、谓词下推                                |
| 关注 group by | map 端聚合、两阶段聚合                                    |
| 关注 join     | join 表数量过多分拆任务、调整 join 的顺序、使用 broadcast |
| 避免问题操作  | 避免全局排序、笛卡尔积，排查数据倾斜、数据膨胀            |

**II. 参数层面的优化**

| 关注方面        | 具体参数及说明                                                                                          |
| --------------- | ------------------------------------------------------------------------------------------------------- |
| 资源方面        | 内存、CPU 核数、executor 个数、默认并行度、默认 shuffle read 分区个数                                   |
| shuffle 方面    | shuffle read/write 缓冲区大小 ↑（增大），I/O 次数 ↓（减少），shuffle memory fraction↑（增大，默认 0.2） |
| bypassThreshold | 默认 200，非聚合类算子可跳过排序                                                                        |
| 其他参数        | AQE、broadcast 阈值等其他参数                                                                           |

---

## OOM 优化

**Driver 端 OOM：**

1. `collect()`, `take()` 算子拉取过多数据
2. broadcast 广播变量太大（自定义结构/从 executor 拉取）

解决思路：`spark.driver.memory` ↑

**Executor 端 OOM：**

1. user memory：自定义的数据结构
2. `coalesce()` 合并分区
3. 数据倾斜 OOM
4. reduce 端一次拉取过多数据
5. 数据膨胀

解决思路：

1. ↑ executor 内存
2. ↑ 并行度，shuffle read 分区数，让每个 task 处理的数据量 ↓
3. ↑ `spark.shuffle.memoryFraction`

---

## Shuffle 优化

1. 避免 Shuffle：Broadcast 小表；表设计初期就设计好分桶方案
2. Map 端预聚合：减少 Shuffle 的数据量（量大）
3. 解决数据倾斜问题（不均匀）
4. ↑ 并行度和下游 reduce task 的数量，让每一份 Shuffle 的数据量 ↓
5. ↑ shuffle read/write 缓冲区大小，↓ I/O 次数
6. ↑ `spark.shuffle.memoryFraction`，executor 内存中用于 shuffle read 聚合的比例
7. shuffle.manager：默认 sort，调整 bypassThreshold(200)，可以让 groupBy/repartition 等算子不排序
8. RSS

---

## 数据倾斜

**I. Group By 导致的数据倾斜**

1. Map 端聚合：尽量使用 Map 端聚合的算子
2. 两阶段聚合：根据业务调整聚合粒度，`group by (a, b)` → 先 `group by (a, b, c)`
3. 两阶段聚合：先将数据随机打散于多个 reducer，部分聚合；然后按照原先分组字段聚合
4. 对少数倾斜 key 单独处理，再 union

**II. Join 导致的数据倾斜**

1. 使用 Map Join：小表的情况
2. 大表，长尾现象由空值导致的话：将空值处理为随机数
3. 大表 Join 大表，长尾现象由热点值导致：拆分热点 key 与非热点 key，再合并
4. 大表 Join 大表：一张表 1-N 加盐，一张表膨胀扩容 N 倍，参考[Spark 性能优化指南——高级篇](https://tech.meituan.com/2016/05/12/spark-tuning-pro.html)

---

## 小文件治理

### 背景

- **消息提示与刷数关联**：收到数据平台消息，告知我负责的某张表小文件过多，且前一天刚做过动态分区刷数。
- **下游任务异常**：同一张表，当天下游任务运行比预期慢很多，查看 Spark 的 syslog 日志，发现大量“Reading ORC rows”信息，由此确认是上游表小文件过多（多达 2.6 万个小文件）的问题。另外，也可通过 Spark WebUI 查看任务的 DAG 中的统计信息，了解 Map 读文件个数以及 Reduce 写文件个数来判断。

### 大量小文件问题的危害

- **占用 NameNode 内存**：会占用大量 NameNode 内存资源。
- **影响磁盘寻址与读文件效率**：导致 DataNode 的磁盘寻址效率低下，进而使得 Map Task 读文件效率下降。
- **增加任务开销**：大量的 Map 任务，每个 Map 开启一个 JVM 执行，带来大量任务初始化、调度方面的开销。
- **文件合并的资源消耗**：虽可开启 Map 前文件合并，但这需要不断从不同节点建立连接、进行数据读取、网络传输后再合并，同样会增加资源消耗、延长计算时间，成本较高。

### 小文件产生的途径

- **流式数据**：如来自 Kafka、Flink 等的流式增量文件，以及小窗口文件（如几分钟产生一次等）。
- **动态分区刷数**：一个 Reduce 会写多个分区，从而产生多个文件。
- **多个 union all**：小文件个数等于多个 select 的 partition 数之和。
- **Spark 任务过度并行化**：Spark 分区越多，写入的文件就越多。
- **广播导致无 Shuffle**：大表 broadcast join 小表时，没有 Shuffle，小文件数等于 map task 的个数（读大表情况较多）。
- **MapReduce 引擎任务**：
  : 若为纯 map 任务，大量的 map 会造成大量文件；若为 mapreduce 任务，大量的 reduce 也会造成大量文件。
  : 出现这种情况原因多样，比如分区表的过度分区、动态分区、输入大量小文件、参数设置不合理以及输出没有文件合并等。

### 滚动治理和存量治理

**滚动治理**：需从集群层面配置一些参数。

**方案一：使用 AQE 的自动分区合并功能**

**原理**：开启之后 AQE 将合并 Reduce Task 中过小的数据分区，默认每个分区合并后约 64MB。

**参数设置**：

```bash
set spark.sql.adaptive.enabled=true
set spark.sql.adaptive.coalescePartitions.enabled=true
```

![](https://raw.githubusercontent.com/LIMUXUALE0927/image/main/img/202411271556903.png)

**方案二：Distribute By Rand**

**原理**：“Distribute by”用来控制 Map 端输出结果的分发。动态分区刷数时，一个 Reduce 可能会写多个分区，易产生大量小文件，极端情况下可能产生“Reduce Task \* 分区个数”个小文件。可通过按分区键进行 Shuffle 的方式，让单个分区的数据全部进入同 1 个或 n 个 Reduce，使该分区只产生 1 个或 n 个文件。

**示例代码**：

```sql
insert overwrite table xxx partition (dt)
select
    user_id,
    user_name,
    user_age,
    dt
from
    xxx
distribute by dt -- 每个分区1个文件
distribute by dt, ceil(rand() * 5) -- 每个分区5个文件
distribute by dt,
            ceil(rand()*if(data_amount/10000000>1000,1000,data_amount/10000000)) -- 一个partition处理1000w条数据,最多一个分区划分1000个partition
```

**方案三：Spark Repartition Hint**
**Coalesce 和 Repartition 的区别**：

- **Coalesce**：只能减少分区数，仅合并分区，最大程度减少了数据移动。
- **Repartition**：提示可以增加或减少分区数量，会对数据进行 RoundRobin 的 Shuffle，确保数据平均分配；增加了一个新 Stage，不会影响现有阶段的并行性，而 Coalesce 会影响现有阶段的并行性，因其不会添加新阶段。

**相关参数**：

**参数一：开启 map 前小文件合并（默认开启）**

```sql
set hive.input.format = org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
```

**参数二：合并 Map 任务和 MR 任务输出文件**

- 设置`smallfiles.avgsize`。
- 集群层面默认开启 map-only 任务的小文件合并。
- 开启 map-reduce 任务的小文件合并。
- 结束时合并文件的大小，一般最好跟生产集群的 block 大小一致合适。该参数表示当一个 MR 作业的平均输出文件大小小于此数字时，Hive 将启动一个额外的 Map 任务（需注意此处有个“坑”），将输出文件合并为更大的文件。

```sql
-- 当输出文件小于32Mb时我们进行合并。
set hive.merge.smallfiles.avgsize=32000000;
-- map-only任务的小文件合并，默认值为true
set hive.merge.mapfiles=true;
-- map-reduce任务的小文件合并，默认值为false
set hive.merge.mapredfiles=true;
-- 作业结束时合并文件的大小，这个一般最好跟生产集群的block大小一致合适
set hive.merge.size.per.task=25600000;
```

**存量治理**：可以写一个专门用于小文件合并的任务，然后通过元数据数仓获取部门所有成员负责的表的文件数，按从高到低的顺序依次治理。实际运行合并小文件任务时，注意先在测试环境运行，然后验数，之后再切换线上环境的表即可。
