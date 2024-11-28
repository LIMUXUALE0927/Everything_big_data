# Hive 性能调优

---

## MapReduce

### MR 流程回顾

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310021836664.png)

---

### MR 参数调优

| 参数名称                                       | 默认值         | 说明                                                                                                                                                                                               |
| ---------------------------------------------- | -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `mapreduce.task.io.sort.mb`                    | 100（单位 MB） | Map 端内存缓冲区大小，建议 512MB+                                                                                                                                                                  |
| `mapreduce.task.io.sort.factor`                | 10             | 排序文件时，一次最多启动合并的文件流的个数。这个属性也在 reduce 中使用。将此值增加到 100 是很常见的。并行合并更多文件可减少合并排序迭代次数并通过消除磁盘 I/O 提高运行时间，但也会使用更多的内存。 |
| `mapreduce.map.output.compress`                | false          | 是否压缩 map 的输出                                                                                                                                                                                |
| `mapreduce.reduce.shuffle.parallelcopies`      | 5              | 用于把 map 输出复制到 reducer 的线程数                                                                                                                                                             |
| `mapreduce.job.reduce.slowstart.completedmaps` | 0.05           | 表示在开始执行 reduce 任务之前，需要完成的 map 任务的比例。默认值为 0.05，即 map 任务完成 5% 时，就可以开始执行 reduce 任务了（开始 copy 数据）。                                                  |

---

### Yarn 回顾

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202402011704309.png)

Yarn 是经典的主从（master/slave）架构。主要由 ResourceManager、NodeManager、ApplicationMaster 和 Container 等组件构成。

YARN 可以看出一个大的云操作系统，它负责为应用程序启动 ApplicationMaster（相当于主线程），然后再由 ApplicationMaster 负责数据切分、任务分配、 启动和监控等工作，而由 ApplicationMaster 启动的各个 Task（相当于子线程）仅负责自己的计算任务。当所有任务计算完成后，ApplicationMaster 认为应用程序运行完成，然后退出。

---

### Yarn 日志

Yarn 提供了两种日志查看方式：

- **ResourceManager Web UI**：可以查看当前正在运行的任务以及部分历史任务
- Job history UI：可以查看一定周期的历史任务（只有 MR 任务，Spark 任务要去 Spark History UI 看）

---

## 参数调优

### Mapper/Reducer 内存与核数

**参数一：设置每个 map 的内存和核数**

- `set mapreduce.map.memory.mb=2048;` 默认为 1024
- `set mapreduce.map.cpu.vcores=1;` 默认为 1

**参数二：设置每个 reduce 的内存大小**

- `set mapreduce.reduce.memory.mb=6144;` 默认为 1024
- `set mapreduce.reduce.cpu.vcores=2;` 默认为 1

有些时候报 OOM、堆内存溢出等错误时，可以适调大这几个参数，通过增加内存 cpu 的方式提高 task 运行速度，但是坏处是可能造成资源的浪费。以上关于 map/reduce 的参数都可以通过客户端直接设置生效，会覆盖集群层面的参数。

常见对应报错：

- 第一种报错："java.lang.OutOfMemoryError: GC overhead limit exceeded"
- 第二种报错："java.lang.OutOfMemoryError: Java heap space"
- 第三种报错："running beyondphysical memory limits. Current usage: 4.3 GB of 4.3 GB physical memory used; 7.4 GB of 13.2 GB virtual memory used. Killing container"

以上常见的三种报错，一般都是由于 map task 或 reduce task 在 container 中运行时处理数据需要的内存资源与申请分配到的资源不匹配造成。这个时候我们可以通过人为增加资源进行尝试。

---

### 如何控制 Map 个数与性能调优

虽然在 Hive 中可以配置 `set mapred.map.tasks=2`，但是这个参数是不生效的，因为 Hive 会根据 `输入数据的大小/split-size` 自动决定 map 个数。精髓是调整 split-size。

```
defaultNum=totalSize/blockSize
expNum=mapred.map.tasks(2)
expMaxNum=max(expNum, defaultNum)//期望大小只有大于默认大小才会生效；
realSplitNum=totalSize/max(mapred.min.split.size, blockSize)
实际的map个数=min(realSplitNum, expMaxNum)
```

**增加切片大小，减少 map 个数。**

案例：一个分区中有 2000 个小文件（1.+MB），原先 79 个 Map，现在 19 个。

```sql
set hive.input.format = org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
set mapred.max.split.size=256000000;
set mapred.min.split.size=128000000;
set mapred.min.split.size.per.node=128000000;
set mapred.min.split.size.per.rack=128000000;
```

一般我们设置 `max.split.size` >= `min.split.size` > `min.size.per.rack` = `min.size.per.node`

---

### 如何控制 Reduce 个数与性能调优

**参数一：直接指定 reduce 个数**

```
set mapred.reduce.tasks/mapreduce.job.reduces=10;
```

**参数二：通过 reduce 处理的数据量，推断 reduce 个数**

```
set hive.exec.reducers.bytes.per.reducer=1000000000;
```

设置每个 Reducer 所能处理的数据量，在 Hive0.14 版本以前默认是 **1GB**，Hive0.14 及之后的版本默认是 **256MB**。

reduce 计算方式：计算 reducer 数的公式很简单：`num=min(hive.exec.reducers.max, (map 输出数据大小)/hive.exec.reducers.bytes.per.reducer)`

**参数三：单个 job 可以使用的最大 reduce 数量**

```
set hive.exec.reducers.max=2999;
```

---

### Hive Task 任务容错机制

**Map/Reduce Task 容错与重试机制：**

如下任务 yarn 界面很常见，比如 reduce 出现了 2 个 failed，17 killed。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202402022153991.png)

Task 指的是具体的 map/reduce task。而实际一个 task 允许尝试多次运行，**每次运行尝试的实例**就被称 Task **Attempt**，也就是 Yarn 任务日志界面 Attempt Type 里统计的数据。

实际生产中，map/reduce task 会因为多方面原因如机器老化，资源不足，进程崩溃，带宽限制等出现部分 map/reduce task 实例失败的情况，这是极其正常且容易发生的事。如果这个时候整个任务就直接报错了，那么代价就太大了。所以 Hadoop 就引入了 task 容错机制（其实也就是 task 重试机制） 。map/reduce 实例失败后，在退出之前向 APPMaster 发送错误报告，错误报告会被记录进用户日志，APPMaster 然后将这个任务实例标为 failed，将其 containner 资源释放给其他任务使用。

通过如下两个参数控制 map/reduce 的 task 一旦失败了 map/reduce 实例可以重试的次数：

```
// 控制 Map Task 失败最大尝试次数，默认值 4
set mapreduce.map.maxattempts=4;
//控制 Reduce Task 失败最大尝试次数，默认值 4
set mapreduce.reduce.maxattempts =4;
```

---

**AppMaster 容错与重试机制：**

AppMaster 也提供了重试机制，YARN 中的应用程序失败之后，最多尝试次数由 `mapred-site.xml` 文件中的 `mapreduce.am.max-attempts` 属性配置：尝试次数默认值为 2，即当 AppMaster 失败 2 次之后，运行的任务将会失败。

**APPMaster 的重试次数还受 YARN 的参数限制**，在 MapReduce 内部，YARN 框架对 AppMaster 的最大尝试次数做了限制。其中，每个在 YARN 中运行的应用程序不能超过这个数量限制，具体限制由 `yarn-site.xml` 文件中的 `yarn.resourcemanager.am.max-attempts` 属性控制（默认为 2）。

---

### Hive 任务实例 failed/killed 的场景

**任务实例出现 failed 的场景：**

任务实例 attempt 长时间没有向 MRAppMaster 报告，后者一直没收到其进度的更新，**一般 attempt 实例与 AppMaster 每 3s 通信一次**，前者像后者报告任务进度和状态；超出阈值（**10 分钟**） ，任务变会被认为“僵死”被标记失败 failed，然后 MRAppMaster 会将其 JVM 杀死，释放资源。然后重新尝试在其他节点启动一个新的任务实例（遵循本地化原则）。

这种情况生产集群还是比较多的，比如 task 运行的 NM 节点异常，主机宕机，网络通信异常等。

---

**任务实例出现 killed 的场景：**

一个任务实例 attempt 被 killed 一般就两种情况：

- 一是客户端主动请求杀死任务，比如客户端通过 `yarn application -kill`，或者关闭执行窗口等
- 二是框架主动杀死任务，即 AppMaster 发出任务 kill 的指令

对于后者，一般是由于作业被杀死或者该任务的备任任务（推测执行）已经执行完成，这个任务不需要继续执行了，所以被 killed。比如 NodeManager 节点故障或者失败任务数过多，比如停止等，这时候上面的所有任务实例都会被标记为 killed。

生产中我们更关注 failed 的 task，这个有时候直接影响我们任务的成功与否。killed 一般不影响任务的失败和成功，但是过多的 killed 我们也需要关注，因为 killed 浪费资源。

---

### Hive 任务推测执行

实际开发中查看 yarn 日志可能会遇到，显示有 257 个 reduce 没跑完，但是下面 attempt 里却有 261 个 reduce 处于 running 状态的情况。

**任务推测执行（speculative execution）：**

MRAppMaster 当监控到一个任务实例的运行速度慢于其他任务实例时，会为该任务启动一个备份任务实例，让这个两个任务同时处理一份数据，如
map/reduce。谁先处理完，成功提交给 MRappMaster 就采用谁的，另一个则会被 killed，这种方式可以有效防止那些长尾任务拖后腿。

任务推测执行的好处就是空间换时间，防止长尾任务拖后腿。**任务推测的坏处就是两个重复任务，浪费资源，降低集群的效率，尤其 reduce 任务的推测执行，拉取 map 数据加大网络 I/O，而且推测任务可能会相互竞争。**

注意：默认集群开启推测执行，可以基于集群计算框架，也可以基于任务类型单独开启，默认 map/reduce 等推测执行都是开启的。map 任务建议开启，reduce 可以结合实际谨慎开启。

推测执行本质是通过提高个别慢 task 的速度进而提高整体的运行效率。但推测执行的使用不好建议是否开启，比如运行的 map/reduce 任务输入数据量很大或者本身执行时间很长的时候，如果你再开启推测执行会造成大量的资源浪费。

---

### 小文件问题

**大量小文件问题的危害：**

- 占用大量 **NameNode 内存**
- 导致 DataNode 的磁盘**寻址效率**低下
- Block 是最小粒度的数据处理单元，大量小文件，大量的块，意味着更多任务调度、任务创建开销，以及更多的任务管理开销

注：虽然可以开启 map 前文件合并，但是这也需要不停地从不同节点建立连接，数据读取，网络传输，然后进行合并，同样会增加消耗资源和增加计算时间，成本也很高。

---

**小文件产生的途径：**

- 流式数据，如 Kafka, SparkStreaming, flink 等，流式增量文件，小窗口文件，如几分钟一次等。
- MapReduce 引擎任务：如果纯 map 任务，大量的 map；如果 mapreduce 任务，大量的 reduce；两者都会造成大量的文件。出现这种情况的原因很多，比如分区表的过度分区，动态分区，输入大量的小文件，参数设置的不合理等，输出没有文件合并等。
- Spark 任务过度并行化，Spark 分区越多，写入的文件就越多。

---

**小文件问题参数优化：**

**参数一：开启 map 前小文件合并，默认开启**

```sql
set hive.input.format = org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
```

**参数二：合并 MR 输出文件**

这个参数表示当一个 MR 作业的平均输出文件大小小于这个数字时，Hive 将启动一个额外的 Map 任务(敲重点，这里有个坑)，将输出文件合并为更大的文件。

```sql
-- 当输出文件小于 32Mb 时我们进行合并。
set hive.merge.smallfiles.avgsize=32000000;
-- 集群层面默认开启 map-only 任务的小文件合并
set hive.merge.mapfiles=true;
-- 注意集群层面默认不开启 map-reduce 任务的小文件合并,如果需要合并需要手动开启；
set hive.merge.mapredfiles=true;
-- 作业结束时合并文件的大小，这个一般最好跟生产集群的 block 大小一致合适
set hive.merge.size.per.task=25600000;
```

案例：一张表某个分区大量小文件，1000 个 kb 级别文件，总存储大小 15.3Mb（单副本）

```sql
set hive.input.format = org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
set hive.merge.mapfiles = true;
set hive.merge.mapredfiles = true;
set hive.merge.smallfiles.avgsize = 32000000;
set hive.merge.size.per.task = 256000000;

create table test as
select
*
from table_name
where dt = '20221030'
```

带坑案例：一张表某个分区大量 Mb 级别小文件，总存储大小 6GB（单副本）

跑完以后合计生成 177 个文件，大小 4.1GB，平均文件大小 4.1GB\*1024/177=23Mb，没有达到的预期平均文件大小 32Mb，为什么？

我们之前说过，map 的个数与切片大小有关：

```sql
set hive.input.format = org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
set mapred.max.split.size=256000000;
set mapred.min.split.size=128000000;
set mapred.min.split.size.per.node=128000000;
set mapred.min.split.size.per.rack=128000000;
```

因此，如果要在 MR 之后另起一个 Map Task 合并输出文件，`mapred.min.split.size.per.rack` 和 `mapred.min.split.size.per.node`，以及 `mapred.min.split.size` 参数的最小值要大于等于 `hive.merge.smallfiles.avgsize` 才会实际生效。

```sql
set hive.input.format = org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
--开启小文件合并，以及小文件大小
set hive.merge.mapfiles = true;
set hive.merge.mapredfiles = true;
set hive.merge.smallfiles.avgsize=32000000;
set hive.merge.size.per.task=25600000;
--控制 map 的输入大小，实现真正的小文件合并
set mapred.max.split.size=256000000;
set mapred.min.split.size=32000000;
set mapred.min.split.size.per.node=32000000;
set mapred.min.split.size.per.rack=32000000;

drop table if exists paas_test.tmp_map_split_files_test_06;
create table ds_test.tmp_map_split_files_test_06
as
select
*
from ds_master.dwd_device_app_runtimes_sec_di
where day='20201211';
-- 数据文件 4.1GB,文件数 65个，平均文件大小 4.1*1024/65=65Mb，满足要求；
```

---

**总结：**

- 合并小文件不仅可以减少对 NameNode 的影响，减少计算资源等，还可以降低表的总存储，尤其列式存储表，合并小文件，变成一个大文件后可以提高文件的压缩率，进而降低存储。
- Map-only 任务，map 数决定了输出文件数。Map-reduce 任务，reduce 个数决定了输出的文件个数。

---

### 并行执行

一段 HQL 会分成多个 job/stage 执行。默认情况下，Hive 一次只会执行一个阶段。不过，某个特定的 job 可能包含众多的阶段，而这些阶段可能并非完全互相依赖的，也就是说有些阶段是可以并行执行的，这样可能使得整个 job 的执行时间缩短。如果有更多的阶段可以并行执行，那么 job 可能就越快完成。

```sql
-- 默认不开启
set hive.exec.parallel=true;
-- how many jobs at most can be executed in parallel
set hive.exec.parallel.thread.number=8;
```

**场景一：union all**

**场景二：multi-insert**

```sql
FROM staged_employees se
INSERT INTO TABLE us_employees
    AS SELECT * WHERE se.cnty = 'US'
INSERT INTO TABLE ca_employees
    AS SELECT * WHERE se.cnty = 'CA'
```

**场景三：不同子查询的 join**

```sql
SELECT t1.column1, t2.column2
FROM
  (SELECT column1, column2 FROM table1) t1
JOIN
  (SELECT column1, column2 FROM table2) t2
ON (t1.column1 = t2.column1);
```

注意：如果 job 因为依赖的问题，只能顺序执行，那么开启这个参数也没用。同时，集群如果繁忙的时候会加剧集群资源争抢，可能是影响别人任务，不过如果集群空闲的话可以增加资源利用率。

---

### JVM 重用

**Hadoop 1.x 版本中的 JVM 重用：**

正常情况下，MapReduce 启动的 JVM 在完成一个 task 之后就退出了，但是如果任务花费时间很短，又要多次启动 JVM 的情况下（比如对很大数据量进行计数操作），JVM 的启动时间就会变成一个比较大的 overhead。在这种情况下，可以使用 jvm 重用的参数：他的作用是让一个 JVM 运行多次任务之后再退出，这样一来也能节约不少 JVM 启动时间。

**Hadoop 2.x 版本中的 JVM 重用：UberTask**

With "Uber" mode enabled, you'll run everything within the container of the AppMaster itself. 这样 AppMaster 不会再为每一个 task 向 RM 申请单独的 container，而是直接在自己的 container 中运行 task，最终达到 JVM 重用的目的。

注意：UberTask 适用于小数据量的任务，大数据量的任务不适合使用 UberTask。

---

### Fetch 抓取

Fetch 抓取是指，Hive 中对某些情况的查询可以不必使用 MapReduce 计算。 例如：`select * from user_base_info;` 在这种情况下，Hive 可以简单地读取 user_base_info 对应的存储目录下的文件， 然后输出查询结果到控制台。Hive 默认启用。

---

### 本地模式

有时 Hive 的输入数据量是非常小的。在这种情况下，为查询触发执行 MR 任务时消耗可能会比实际 job 的执行时间要多的多。对于大多数这种情况，Hive 可以通过**本地模式在单台机器上处理所有的任务**。对于小数据集，执行时间可以明显被缩短。 用户可以通过设置 `hive.exec.mode.local.auto` 的值为 true，来让 Hive 在适当的时候自动启动这个优化。

```sql
--开启本地 mr
set hive.exec.mode.local.auto=true;
--设置 local mr 的最大输入数据量，当输入数据量小于这个值时采用 local mr 的方式， 默认为 134217728，即 128M
set hive.exec.mode.local.auto.inputbytes.max=50000000;
--设置 local mr 的最大输入文件个数，当输入文件个数小于这个值时采用 local mr 的方式， 默认为 4
set hive.exec.mode.local.auto.input.files.max=8;
```

---

### 严格模式

默认开启。

- 分区表查询：开启严格模式后，在查询一个分区表时，如果查询条件中没有包含分区字段，Hive 会直接报错。
- order by：开启严格模式后，如果查询中包含 order by 子句，但是没有包含 limit 子句，Hive 会直接报错。
- 笛卡尔积：开启严格模式后，如果查询中包含笛卡尔积，Hive 会直接报错。

---

### 向量化模式

Hive 的矢量化是一次处理一批次行，而不是一次处理一行。与基于行的执行相比，向量化执行避免了大量的虚函数调用，从而提高了指令和数据缓存的命中率它们一起处理一批行，而不是一次处理一行。**与基于行的执行相比，向量化执行避免了大量的虚函数调用，从而提高了指令和数据缓存的命中率。**

但是矢量化的使用是有限制的，不是对所有的语法都是支持，使用不当会适得其反，这也就是为啥矢量化默认是关闭的，需要手动开启。

总结：

- 目前 MapReduce 计算引擎只支持 Map 端的向量化执行模式，Tez 和 Spark 计算引擎可以支持 Map 和 Reduce 端的向量化执行模式。这个参数因为有坑，建议大数据集群层面关闭，大数据开发可以结合实际任务进行开启使用这个功能，提高运行效率，但需注意使用场景；
- 若要使用向量化查询执行，必须将数据存储为 **ORC 格式**，本质就是一次读取一行，变成一次读取 1024 行处理。
- 在 Hive 中提供的向量模式，并不是重写了 Mapper 中的函数，而是通过实现 Inputformat 接口，创建 VertorizedOrcInputFormat 类，来构造一个批量的输入数组
- 向量化查询最好只在简单逻辑时候使用；发现不少向量化查询的坑，包括：group by 不能使用向量化查询，全字段 group by 结果却有重复的（group by, join 等复杂语句不要使用），发现会把空字符串转换为 null。

---

### 动态分区

动态分区是指在插入数据时，不需要指定分区字段的值，而是由 Hive 自动基于查询参数的位置去推断分区的名称，从而建立分区。

```sql
-- 默认 false
set hive.exec.dynamic.partition = true;
-- 默认 strict 模式，表示用户必须指定至少一个静态分区字段，以防用户覆盖所有
-- 分区。Nonstrict 模式表示允许所有分区都可以直接动态分区
set hive.exec.dynamic.partition.mode = nonstrict;
-- 默认 100，一般可以设置大一点，比如 1000，表示每个 maper 或 reducer 可
-- 以允许创建的最大动态分区个数
set hive.exec.max.dynamic.partitions.pernode = 100;
-- 表示一个动态分区语句可以创建的最大动态分区个数
set hive.exec.max.dynamic.partitions = 1000;
-- (默认) 全局可以创建的最大文件个数，超出报错
set hive.exec.max.created.files = 10000;
```

注意：

- 动态分区是基于查询字段的（个数）位置去推断分区的名称，所以动态分区的字段需要放到业务数据的后面。查询的字段个数必须和目标的字段个数相同，不能多，也不能少，否则会报错。但是如果字段的类型不一致的话，则会使用 null 值填充，不会报错。

- 合理的分区方案首先应该是分区数不能过多，产生过多的文件数和目录数，比如每个目录下文件的大小总和最起码要是 block 大小的数倍。其次合理的分区要是可预测的，分区的增长要均匀和稳定的。

- 动态分区弊端可能造成大量文件和目录数，大量小文件，增加元数据压力；其次不利于数据压缩合并，整体会增加数据存储。

---

## Join 的优化

### Hive 原生支持的 Join 类型

Hive 支持的 join 类型有：

- Inner Join
- Left/Right/Full Outer Join
- Left Semi Join
- Cross Join

Hive 拥有多种 join 算法，包括 Common Join，Map Join，Bucket Map Join，Sort Merge Buckt Map Join 等，下面对每种 join 算法做简要说明：

---

### Common Join

Common Join 是 Hive 中最稳定的 join 算法，其通过一个 MapReduce Job 完成一个 join 操作。Map 端负责读取 join 操作所需表的数据，并按照关联字段进行分区，通过 Shuffle，将其发送到 Reduce 端，相同 key 的数据在 Reduce 端完成最终的 Join 操作。

Shuffle 阶段代价非常昂贵，因为它需要排序和合并。减少 Shuffle 和 Reduce 阶段的代价可以提高任务性能。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202402031546970.png)

需要注意的是，sql 语句中的 join 操作和执行计划中的 Common Join 任务并非一对一的关系，一个 sql 语句中的相邻的且关联字段相同的多个 join 操作可以合并为一个 Common Join 任务。

```sql
select a.val,
       b.val,
       c.val
from a
join b on a.key = b.key1
join c on c.key = b.key1
```

上述 sql 语句中两个 join 操作的关联字段均为 b 表的 key1 字段，则该语句中的两个 join 操作可由一个 Common Join 任务实现，也就是可通过一个 Map Reduce 任务实现。

```sql
select a.val,
       b.val,
       c.val
from a
join b on a.key = b.key1
join c on c.key = b.key2
```

上述 sql 语句中的两个 join 操作关联字段各不相同，则该语句的两个 join 操作需要各自通过一个 Common Join 任务实现，也就是通过两个 Map Reduce 任务实现。

---

### Map Join

**Map Join 算法可以通过两个只有 map 阶段的 Job 完成一个 join 操作**。其适用场景为大表 join 小表。若某 join 操作满足要求，则第一个 Job 会读取小表数据，将其制作为**哈希表文件**，并上传至 **Hadoop 分布式缓存** (本质上是上传至 HDFS)。第二个 Job 会先把分布式缓存中的哈希表文件发送给各个 Mapper，各个 Mapper 将此文件加载进内存，然后扫描大表数据，这样在 map 端即可完成 join 操作。此外，如果多个 Mapper 在同一台机器上运行，则分布式缓存只需将哈希表文件的一个副本发送到这台机器上。

- Job1（本地任务，不会提交 Yarn）：读取小表数据 → 制作哈希表 → 上传至分布式缓存

- Job2：分布式缓存发送哈希表文件给 Mapper → 加载至内存 → 完成 join

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202402031556141.png)

触发 Map Join：

- 在 SQL 语句中增加 hint 提示（已过时）

- Hive 优化器根据表的大小自动推断

```sql
-- 开启 mapjoin 自动推断
set hive.auto.convert.join=true;
-- 设置小表阈值，默认值为 25MB
set hive.mapjoin.smalltable.filesize=25000000;
```

**注意：**

- Map Join 对 full outer join 不生效
- 对于需要保留小表全量的情况不生效，如 `smalltable left join bigtable`，因为如果每个 maptask 都保留一个全量小表，map 端聚合后直接写文件了，没法对每个 map 都全保留的小表进行去重，这样会造成数据膨胀

**自动推断：**

Hive 在编译 SQL 语句阶段，起初所有的 Join 操作均采用 Common Join 算法实现。

之后在物理优化阶段，Hive 会根据每个 Common Join 任务所需表的大小判断该 Common Join 任务是否能够转换为 Map Join 任务，若满足要求，便将 Common Join 任务自动转换为 Map Join 任务。判断标准为小表大小与用户设置的阈值（MapTask 内存）比较。

但**有些 Common Join 任务所需的表大小，在 SQL 的编译阶段是未知的**（比如子查询生成的中间表），所以这种 Common Join 任务是否能转换成 Map Join 任务在编译阶是无法确定的。

针对这种情况，Hive 会在编译阶段生成一个**条件任务 (Conditional Task)**，其下会包含一个**计划列表**，计划列表中包含**转换后的 Map Join 任务以及原有的 Common Join 任务**。 最终具体采用哪个计划，是在**运行时决定**的。大致思路如下图所示:

假设我们有 A 和 B 两张表，两张表的大小都未知，那么此时 Map Join 有 2 种情况：

- 缓存 A 表，扫描 B 表

- 缓存 B 表，扫描 A 表

因此会生成多种执行计划。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310160539533.png)

> **Hive 如何判断执行计划中的 Common Join Task 能否转换为 Map Join Task？**

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310160542023.png)

注意：

- 对于上面的「寻找大表候选人」这一步：**要考虑 SQL 语句中具体使用的是哪种 Join**。例：A left join B，那么大表必须是 A；如果是 A full outer join B，则根本不能做 Map Join。

- 对于上面的「hive.auto.conver.join.noconditionaltask」的 false 分支，考虑这样一种情况：A inner join B inner join C 关联相同的字段，此时有一个 Map Join 计划尝试将 A 作为大表，B 和 C 作为小表缓存，C 的大小未知，B 的大小已知但太大无法缓存（超过小表阈值），则这种 Map Join 计划一定可以排除。即**排除掉一定不可能成功的 Map Join 计划**。

- 对于上面的「hive.auto.conver.join.noconditionaltask」的 true 分支，此时必须要保证大表外的所有表的大小都已知，并且总和小于小表阈值，才能生成**最优**的 Map Join 计划。

- 对于上面的「生成最优 Map Join 计划」的下一步，考虑这样一种情况：A join B join C on 不同字段，理论上需要 2 个 Common Join，先把 A join B 得到临时表 T，再用 T join C。如果 B 和 C 表的大小都非常小，那么可以把 B 和 C 都缓存到 Mapper 内存中，进行两次 Map Join。

---

### Bucket Map Join

Bucket Map Join 是**对 Map Join 的改进**，其打破了 Map Join 只适用于大表 join 小表的限制，可用于大表 join 大表的场景。**但是现在大表 Join 大表的场景在 Hive 中已经被 SMB Map Join 取代。**

**分桶的原理：对分桶字段进行 hash partition，拆分为若干个文件。**

Bucket Map Join 的核心思想是，若能保证：

1. 参与 join 的表均为分桶表
2. 其中一张表的分桶数量是另外一张表分桶数量的整数倍
3. 关联字段为分桶字段，即 join key = cluster by key

就能保证参与 join 的两张表的分桶之间具有明确的关联关系，所以就可以在两表的分桶间进行 Map Join 操作了。这样一来，第二个 Job 的 Map 端就无需再缓存小表的全表数据了，而只需缓存其所需的分桶即可。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202402041546397.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202402041549461.png)

分桶之后就和 Map Join 的操作一致：

- Job1：先由本地 Map 任务将相对小一点的表的每个桶各制作一张哈希表，将所有哈希表上传至分布式缓存

- Job2：按照 BucketInputFormat 读取大表数据，一个桶一个切片，每一个 Mapper 只需要处理大表中的一个桶即可。同时，根据桶之间的对应关系，每个 Mapper 只需要从分布式缓存中读取自己需要的小表的桶的哈希表即可。例如，Mapper1 被分到了 Bucket A-0，因此，Mapper1 只需要去分布式缓存读取 Bucket B-0 即可。

---

### Sort Merge Bucket Map Join

SMB Map Join 是桶表的主要使用方式。

SMB Map Join 要求：

1. 参与 join 的表均为分桶表，且需保证分桶内的数据是有序的
2. 其中一张表的分桶数量是另外一张表分桶数量的整数倍
3. join key = cluster by key = sort by key

SMB Map Join 同 Bucket Join 一样，同样是利用两表各分桶之间的关联关系，在分桶之间进行 join 操作，不同的是分桶之间的 join 操作的实现原理。Bucket Map Join，两个分桶之间的 join 实现原理为 **Hash Join 算法**；而 SMB Map Join，两个分桶之间的 join 实现原理为 **Sort Merge Join 算法**。

Hash Join 和 Sort Merge Join 均为关系型数据库中常见的 Join 实现算法。Hash Join 的原理相对简单，就是对参与 join 的一张表构建 hash table，然后扫描另外一张表， 然后进行逐行匹配。**Sort Merge Join 需要在两张按照关联字段排好序的表中进行**，其原理如图所示：

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310160518467.png)

**SMB Map Join VS Bucket Map Join**

- 不需要制作哈希表

- 不需要在内存中缓存哈希表（一个桶），因此对桶的大小没有要求

Hive 中的 SMB Map Join 就是对两个分桶的数据按照上述思路进行 Join 操作。可以看出，SMB Map Join 与 Bucket Map Join 相比，在进行 Join 操作时，Map 端是无需对整个 Bucket 构建 hash table，也无需在 Map 端缓存整个 Bucket 数据的，每个 Mapper 只需按顺序逐个 key 读取两个分桶的数据进行 join 即可。

总结：

- SMB Mapjoin 只有 map，没有 reduce，map 个数等于最大分桶数，map 结束直接数写入 hive 表
- SMB Mapjoin 的启动不受 mapjoin 的大小表限制，适合大表 join 大表，前提是需要提前设计好生成好对应的桶排序表
- SMB Mapjoin 弊端也很明显，即不够灵活，比如换个表，换个 join key 则无法生效
- SMB Mapjoin 也有好处：比如对于固定的大表 join 大表一般可以提高 join 效率。同时绝对要比非分桶表具有更快的抽样效率；但是是否比非分桶表一定节省时间，则不一定
- SMB MapJoin 建议使用场景：大表与大表 join 时，如果 key 分布均匀，单纯因为数据量过大，导致任务失败或运行时间过长，可以考虑将大表分桶，来优化任务

---

### Skewed Map Join

整体思路是使用独立的作业和 mapjoin 来处理倾斜的键。所以当 Join 操作中某个表中的一些 Key 数量远远大于其他（包含空值 null 值），则处理该 Key 的 Reduce 将成为任务单瓶颈，这个时候我们可以通过开启 Skewed join 来实现在 map 端对倾斜健进行聚合。

Skew Join 的原理是，为倾斜的大 key 单独启动一个 map join 任务进行计算，其余 key 进行正常的 common join。

注意：

- Skewed Map join 使用优化使用场景很有限，只有 INNER JOIN 的数据倾斜才可以实现优化。对其他 JOIN 如 LEFT/RIGHT JOIN, FULL OUTER JOIN 则无法使用
- Skewed map join 有个巨坑就是这个参数和 parallel 并行同时开启时会出现丢数据的问题 [issue](https://issues.apache.org/jira/browse/HIVE-25269)

---

## Group By 优化

!!! note "针对 group by 的优化需要重点考虑两方面"

    1. **Map 端聚合**
    2. **负载均衡（倾斜优化）**

Hive 中**未经优化**的分组聚合，是通过一个 MapReduce Job 实现的。Map 端负责读取数据，并按照分组字段分区，通过 Shuffle，将数据发往 Reduce 端，各组数据在 Reduce 端完成最终的聚合运算。

Hive 对分组聚合的优化主要围绕着**减少 Shuffle 数据量**进行，具体做法是 **map-side 聚合**。所谓 map-side 聚合，就是在 map 端维护一个（内存中的） **hash table**，利用其完成部分的聚合，然后将部分聚合的结果，按照分组字段分区，发送至 reduce 端，完成最终的聚合。**map-side 聚合能有效减少 shuffle 的数据量，提高分组聚合运算的效率**。

```sql
--是否在 Map 端进行聚合， 默认为 True
set hive.map.aggr = true;
--在 Map 端进行聚合操作的条目数目
set hive.groupby.mapaggr.checkinterval = 100000;
--有数据倾斜的时候进行负载均衡（默认是 false）
set hive.groupby.skewindata = true;
```

上面的 `set hive.groupby.skewindata = true;` 当选项设定为 true，生成的查询计划会有**两个 MR Job**。第一个 MR Job 中，Map 的输出结果会随机分布到 Reduce 中，每个 Reduce 做部分聚合操作，并输出结果，这样处理的结果是相同的 Group By Key 有可能被分发到不同的 Reduce 中，从而达到负载均衡的目的；第二个 MR Job 再根据预处理的数据结果按照 Group By Key 分布到 Reduce 中（这个过程可以保证相同的 Group By Key 被分布到同一个 Reduce 中），最后完成最终的聚合操作。

注意：这个场景对 count(distinct) 的场景不适用，因为 count(distinct) 本身就是一个全局聚合操作，无法在 Map 端完成。

---

### Group By 的四种执行计划模式

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202402031924962.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202402031925774.png)

需要注意的是：`hive.map.aggr` 和 `hive.groupby.skewindata` 两个参数同时开启的时候，可能会产生 bug，计算出错误的结果。详见：[issue](https://issues.apache.org/jira/browse/HIVE-18980) 和 [blog](https://blog.csdn.net/Dreamy_zsy/article/details/112679598)

总结：

- `count(distinct)` 操作比较特殊，无法进行中间的聚合操作，因此该 `hive.map.aggr=true` 参数对有 `count(distinct)` 操作的 sql 不适用。
- 对于 `hive.map.aggr=true` 参数建议集群层面保持官方默认值 true 即可。但是对于 `hive.groupby.skewindata` 建议慎用，可能会有意想不到的结果（建议通过改造 SQL 代码的方式手动实现 `hive.groupby.skewindata` 的功能）。
- Hive 的 Map 端聚合类似 Combiner 功能，但不同，本质是使用**哈希表**来缓存数据，聚合数据。**使用 Map 聚合往往是为了减少 Map 任务的输出， 减少传输到下游任务的 Shuffle 数据量**，但**如果数据经过聚合后不能明显减少，那无疑就是浪费机器的 I/O 资源，比如对于数据重复率很低的 map 端聚合其实是无效的**。

手动加盐代码示例：

```sql
select date,
       app_id,
       count(uid) as pv
from source_tb
group by date,
         app_id;
```

某个 app 流量远超其他 app 就可能倾斜，因此可以改写：

```sql
with t1 as (select date,
                   -- 加随机前缀，用 “_” 连接
                   concat(cast(cast(RAND() * 100 as int) as string), "_", app_id) as new_app_id,
                   uid
            from source_tb),
     t2 as (select date,
                   new_app_id,
                   count(uid) as pv
            from t1
            group by date,
                     new_app_id)
select date,
       -- 用 “_” 拆分为两个部分，第二个部分为原始的 app_id
       split(new_app_id, '_')[1] as app_id,
       sum(pv)                   as pv
from t2
group by date,
         split(new_app_id, '_')[1]
```

---

## count(distinct) 优化

数据量大的情况下，由于 COUNT DISTINCT 操作需要用一个 Reduce Task 来完成，这一个 Reduce 需要处理的数据量太大，就会导致整个 Job 很难完成，一般 COUNT DISTINCT 使用先 GROUP BY 再 COUNT 的方式替换。

```sql
--直接去重
select count(distinct id) from bigtable;
--改写后去重
select count(id) from (select id from bigtable group by id) a;
```

虽然会多用一个 Job 来完成，但在数据量大的情况下，group by 依旧是去重的一个优化手段。如果说需要统计的字段有 Null 值，最后只需要 null 值单独处理后 union 即可。

---

## Hive SQL 优化

### Hive SQL 执行顺序

Map 端：

1. 首先执行 FROM，进行表的 scan 查找与加载;
2. 然后执行 WHERE 操作，Hive 对语句进行了优化，如果符合谓词下推，则进行谓词下推。这也是指定分区后不会全表扫描的原因。
3. 然后执行 JOIN 语句，按照 on 里面的 join-key 进行 shuffle 数据，注意只有使用到列才会读取 shuffle 出去，shuffle 的 key 是 join-key(支持组合键)。（注意这里先不考虑 map-join，groupby 的 map 聚合等优化操作）

Reduce 端：

1. 如果 on 里有非等值的过滤表达式如 `t1.db != 'abc'`，注意不是非等值链接。在实际 join 关联前先过滤 on 里面的谓词；
2. 其他然后才是 group by, having, order by, limit 等。注意实际执行计划需要看 explain，会因为不同版本和执行优化细节上有差异。

!!! question "过滤条件放 on 和 where 的区别"

      - 过滤条件放 where 中是表连接之后再过滤，而放在 on 中是在表连接之前过滤。
      - 对于 inner join 来说，二者完全相同。
      - 对于 left join 来说，on 条件是在生成临时表时使用的条件，它不管 on 中的条件是否为真，都会返回左边表中的记录。而 where 条件是在临时表生成好后，再对临时表进行过滤的条件。这时已经没有 left join 的含义（必须返回左边表的记录）了，条件不为真的就全部过滤掉。

总结：

Hive SQL 中 LEFT JOIN 单独针对左表的过滤条件必须放在 WHERE 上，放在 ON 上的效果是不可预期的（放在 on 里的时候在 reduce 阶段执行），单独针对右表的查询条件放在 ON 上是先过滤右表，再和左表联表，放在 WHERE 条件上则是先联表再过滤，语义上存在差别。

---

### Hive SQL 优化思路

Hive SQL 本质上可以被分成 3 种模式， 即过滤模式、 聚合模式和连接模式。

1. 过滤模式：从过滤的粒度来看，主要分为：行过滤、 数据列过滤、文件过滤和目录过滤 4 种方式。具体 SQL 语法上体现在 where，having，distinct 语句上；
2. 聚合模式：distinct 聚合，count 计数聚合，数值相关的聚合模式，行转列聚合模式。
3. Join 模式：有 shuffle 的 join 和无 shuffle 的 join。

---

### 分区裁剪

分区裁剪本质是为了减少数据量，避免全表扫描。虽然分区过滤条件我们写在 where 子句里，但是分区过滤是发生在 map 之前，这就是分区表目录结构的优势，相当于直接指定路径读取文件了。

分区裁剪有时候会有失效的情况，比如使用 udf 函数对其进行处理时，可能存在分区裁剪失效的情况，具体我们可以通过 explain 命令查看。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202402040039873.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202402040043227.png)

总结：

- 如果是 left join 分区裁剪条件一般放到 where 里主表生效，放到 on 里从表生效。如果想同时上线列裁剪，那么就把主表的分区裁剪放到 where 子句里，把从表的分区裁剪放到 on 里。同理 right join 反过来即可。

---

### 谓词下推

Predicate（谓词）即条件表达式，SQL 中的谓词主要有 LIKE、BETWEEN、IS NULL、IS NOT NULL、IN、EXISTS 其结果为布尔值，即 true 或 false。

谓词下推对 SQL 代码编写的参考依据：[wiki](https://cwiki.apache.org/confluence/display/Hive/OuterJoinBehavior#OuterJoinBehavior-PredicatePushdownRules)

1. 对于 Inner Join、Full Join，条件写在 on 后面，还是 where 后面，性能上面没有区别；
2. 对于 Left Join ，右侧的表写在 on 后面、左侧的表写在 where 后面，性能上有提高；
3. 对于 Right Join，左侧的表写在 on 后面、右侧的表写在 where 后面，性能上有提高；

---

### Order By 优化

Order by 由于全局排序，数据全部发往一个 Reducer，通常会导致性能问题。因此，我们可以通过 distribute by + sort by 的方式来优化。

```sql
SELECT *
FROM (
  SELECT *
  FROM your_table
  DISTRIBUTE BY column_name
  SORT BY column_name ASC
) subquery
ORDER BY column_name ASC
LIMIT 100;
```

---

---

## 数据倾斜问题

数据倾斜问题，通常是指参与计算的数据分布不均，即某个 key 或者某些 key 的数据量远超其他 key，导致在 shuffle 阶段，大量相同 key 的数据被发往同一个 Reducer，进而导致该 Reducer 所需的时间远超其他 Reducer，成为整个任务的瓶颈。

Hive 中的数据倾斜常出现在**分组聚合**和 **Join** 操作的场景中。

### Group By 产生数据倾斜

使用 Hive 对数据做分组聚合的时候某种类型的数据量特别多，而其他类型数据的数据量特别少。

Hive 中未经优化的分组聚合，是通过一个 MapReduce Job 实现的。Map 端负责读取数据，并按照分组字段分区，通过 Shuffle，将数据发往 Reduce 端，各组数据在 Reduce 端完成最终的聚合运算。

如果 group by 分组字段的值分布不均，就可能导致大量相同的 key 进入同一 Reducer， 从而导致数据倾斜问题。

由分组聚合导致的数据倾斜问题，有以下两种解决思路:

- **Map-Side 聚合**

开启 Map-Side 聚合后，数据会现在 Map 端完成部分聚合工作。这样经过 Map 端的初步聚合后，可以缓解发往 Reducer 的数据的倾斜程度。最佳状态下，Map 端聚合能完全屏蔽数据倾斜问题。

- **Skew-GroupBy 优化**

Skew-GroupBy 的原理是启动两个 MR 任务，第一个 MR 按照随机数分区，将数据分散发送到 Reducer，完成部分聚合，第二个 MR 按照分组字段分区，完成最终聚合。

```sql
--启用分组聚合数据倾斜优化
set hive.groupby.skewindata=true;
```

- 根据业务，合理调整分组维度

可以单独将倾斜程度大的字段单拎出来计算，再 union 结果。

---

### count(distinct) 产生数据倾斜

如果数据量非常大，执行如 `select a, count(distinct b) from t group by a;` 类型的 SQL 时，会出现数据倾斜的问题。

解决方法：

- 使用 group by 代替。

```sql
select a, count(1)
from (select a, b from t group by a, b)
group by a;
```

详见上文 count(distinct) 优化

- 在业务逻辑优化效果的不大情况下，有些时候是可以将倾斜的数据单独拿出来处理，最后 union 回去。

---

### Join 产生数据倾斜

未经优化的 Join 操作，默认是使用 common join 算法，也就是通过一个 MapReduce Job 完成计算。Map 端负责读取 join 操作所需表的数据，并按照关联字段进行分区，通过 Shuffle，将其发送到 Reduce 端，相同 key 的数据在 Reduce 端完成最终的 Join 操作。

如果关联字段的值分布不均，就可能导致大量相同的 key 进入同一 Reduce，从而导致数据倾斜问题。

由 Join 导致的数据倾斜问题，有如下三种解决方案：

- **Map Join**

使用 map join 算法，join 操作仅在 map 端就能完成，没有 shuffle 操作，没有 reduce 阶段，自然不会产生 reduce 端的数据倾斜。该方案适用于**大表 join 小表时发生数据倾斜的场景**。

- **Skew Join**

Skew Join 的原理是，为倾斜的大 key 单独启动一个 map join 任务进行计算，其余 key 进行正常的 common join。原理图如下：

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310161600880.png)

注意：在数据倾斜时，如果是**大表 Join 大表**，那么也无法使用 Bucket Map Join，因为分桶（过程为 MR）也是倾斜的。此时需要使用 Skew Join。

```sql
--启用 skew join 优化
set hive.optimize.skewjoin=true;
--触发 skew join 的阈值，若某个 key 的行数超过该参数值，则触发
set hive.skewjoin.key=100000;
```

这种方案对参与 join 的源表大小没有要求，但是对两表中倾斜的 key 的数据量有要求， 要求一张表中的倾斜 key 的数据量比较小 (方便走 map join)。

- **调整 SQL 语句**

若参与 join 的两表均为大表，其中一张表的数据是倾斜的，此时也可通过以下方式对 SQL 语句进行相应的调整。

假设原始 SQL 语句如下：A，B 两表均为大表，且其中一张表的数据是倾斜的。

```sql
select *
from A join B
on A.id=B.id;
```

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310161615260.png)

图中 1001 为倾斜的大 key，可以看到，其被发往了同一个 Reduce 进行处理。

调整 SQL 语句如下：

```sql
select
	*
from(
    select --打散操作
    	concat(id,'_',cast(rand()*2 as int)) id, value
	from A
)ta join(
	select --扩容操作
        concat(id,'_',0) id, value
    from B
    union all
    select
        concat(id,'_',1) id, value
    from B
)tb
on ta.id=tb.id;
```

调整之后的 SQL 语句执行计划如下图所示：

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310161619269.png)

---

### 空值产生数据倾斜

如果表中有大量的 null 值，所有的 null 值都会被分配到同一个 reducer 上，这样就会导致数据倾斜。需要注意的是，数据被分发到同一个 reducer 的原因并不是能不能 join 上，而是 Shuffle 阶段的 hash 操作，只要 hash 结果相同，就会被分发到同一个 reducer 上。

解决方法：

- 单独把空值拎出来再 union

```sql
select a.*
from log a
join users b
on a.user_id is not null and a.user_id = b.user_id
union all
select a.*
from log a
where a.user_id is null;
```

- 给空值分配随机的 key 值

```sql
select *
from log a
left join users b
on coalesce(a.user_id, concat('hive', rand())) = b.user_id;
```

一般分配随机 key 值得方法更好一些。

---

## Hive SQL 优化思路总结

**核心思想：减少数据量、减少 job 数、避免数据倾斜。**

1. 分区裁剪：尽量使用分区字段进行过滤，避免全表扫描
2. 列裁剪：不会减少读取的数据量，但是能减少后续处理的数据量
3. 谓词下推，提前过滤
4. Join 的表不宜过多，可以建立中间表
5. Join 时将相同连接条件的放在一起，减少 job 数
6. 避免数据倾斜：大 join key 打散、添加随机数、使用 skew join 等
7. sort by + distribute by 代替 order by
8. group by 代替 distinct
9. 避免使用笛卡尔积
10. 使用 CTE 将一些公共子查询提取出来，避免重复计算
