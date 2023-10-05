## MapReduce 是什么？

MapReduce 是一个**分布式运算程序的编程框架**，MapReduce 核心功能是将用户编写的业务逻辑代码和自带默认组件整合成一个完整的分布式运算程序，并发运行在一个 Hadoop 集群上。

MapReduce 基于分治的思想，将一个大的计算任务拆分成多个子任务交给多个节点并行计算，最后合并结果。

它由两个阶段组成：

- Map 阶段（切分成一个个小的任务）
- Reduce 阶段（汇总小任务的结果）

---

## MapReduce 的优缺点

**MapReduce 的优点：**

- 易于编程：

它简单的实现一些接口，就可以完成一个分布式程序，这个分布式程序可以分布到大量廉价的 PC 机器上运行。也就是说你写一个分布式程序，跟写一个简单的串行程序是一模一样的。

- 高可扩展性：

当你的计算资源不能得到满足的时候，你可以通过简单的增加机器来扩展它的计算能力。

- 高容错性：

MapReduce 设计的初衷就是使程序能够部署在廉价的 PC 机器上，这就要求它具有很高的容错性。比如其中一台机器挂了，它可以把上面的计算任务转移到另外一个节点上运行，不至于这个任务运行失败，而且这个过程不需要人工参与，而完全是由 Hadoop 内部完成的。

**MapReduce 的缺点：**

- 不擅长实时计算

MapReduce 无法像 MySQL 一样，在毫秒或者秒级内返回结果。

- 不擅长流式计算

流式计算的输入数据是动态的，而 MapReduce 的输入数据集是静态的，不能动态变化。这是因为 MapReduce 自身的设计特点决定了数据源必须是静态的。

- 不擅长 DAG 计算

多个应用程序存在依赖关系，后一个应用程序的输入为前一个的输出。在这种情况下，MapReduce 并不是不能做，而是使用后， 每个 MapReduce 作业的输出结果都会写入到磁盘，会造成大量的磁盘 IO，导致性能非常的低下。

---

## MapReduce 进程

在运行一个完整的 MapReduce 程序时，在集群中有 3 类实例进程：

- MRAppMaster：负责整个程序过程调度及状态协调
- MapTask：负责 map 阶段整个数据处理流程
- ReduceTask：负责 reduce 阶段整个数据处理流程

---

## MapReduce 序列化

Mapper 和 Reducer 使用的数据的类型是 Hadoop 自身封装的序列化类型。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202309182319412.png)

!!! question "为什么不用 Java 的序列化？"

Java 的序列化是一个重量级序列化框架（Serializable），一个对象被序列化后，会附带很多额外的信息（各种校验信息，Header，继承体系等），不便于在网络中高效传输。所以，Hadoop 自己开发了一套序列化机制（Writable）。

**Hadoop 序列化特点：**

- 紧凑 ：高效使用存储空间
- 快速：读写数据的额外开销小
- 互操作：支持多语言的交互

如果想要传输 Java Bean，需要实现 Writable 接口，并手动**重写**序列化（`write()`）和反序列化方法（`readFields()`），注意反序列化顺序和序列化的顺序要完全一致。同时，如果需要将自定义的 Bean 放在 Key 中传输，还需要**额外实现 Comparable 接口**，因为 MapReduce 中的 Shuffle 过程要求 Key 必须能排序。

---

## MapReduce 核心思想

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310021748839.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310021836664.png)

### Map 阶段

第一阶段：逻辑切片

- 对待处理目录下所有文件逐个遍历，进行**逻辑切片**形成切片规划。
- Split size = Block size，每一个切片由一个 MapTask 处理。（`FileInputFormat.getSplits()`）

第二阶段：按行读取

- 对切片中的数据**按行读取**，解析返回 `<key,value>` 对。
- key：每一行的起始位置偏移量
- value：本行的文本内容
- `TextInputFormat.LineRecordReader`

第三阶段：Map 方法

- 调用 Mapper 类中的 map() 方法处理数据。
- 每读取解析出来的一个 `<key,value>` ，调用一次 map() 方法处理。（`Mapper.map()`）

第四阶段：分区 partition

- 对 map() 方法计算输出的结果进行**分区 partition**。
- 默认不分区，因为默认只有一个 ReduceTask。
- 分区的数量就是 ReduceTask 运行的数量。（HashPartitioner）

第五阶段：写入内存缓冲区

- map() 输出数据写入**内存缓冲区**。
- 达到 80% 比例溢出到磁盘上。
- 溢写 spill 的时候根据 key 进行排序 sort。
- 默认根据 key 字典序排序。（`WritableComparable.compareTo()`）

第六阶段：溢写文件合并

- 对所有溢写文件进行最终的 merge 合并，成为一个文件。

### Reduce 阶段

第一阶段：复制

- ReduceTask 会主动从 MapTask 复制拉取其输出的键值对。

第二阶段：合并

- 把 copy 来的数据，进行合并 merge，即把分散的数据合并成一个大的数据，再对合并后的数据排序。

第三阶段：分组 + reduce

- 对于合并、排序后的键值对，将 key 相等的键值对组成一组
- 对每一组键值对调用一次 reduce() 方法。

---

## MapReduce 框架原理详解

### InputFormat 数据读入

- 读取数据组件 `InputFormat`（默认 `TextInputFormat`）会通过 `getSplits()` 方法对输入目录中文件进行逻辑规划得到 splits，有多少个 split 就对应启动多少个 MapTask。默认情况下 split 与 block 的是一对一关系。

- 逻辑规划后，由 `RecordReader` 对象（默认 `LineRecordReader`）进行读取，以回车换行作为分隔符，读取一行数据，返回 `<key，value>`。key 表示每行首字符偏移量，value 表示这一行文本内容。

- 对于每一个 `<key,value>`，调用一次 map() 方法处理。

!!! note "block VS split"

- 数据块（Block）：HDFS 存储的数据单位
- 切片（Split）：只是逻辑上对输入进行分片，并不会在磁盘上将其切分存储。**切片是 MapReduce 程序输入数据的单位，一个切片对应启动一个 MapTask。**

- 如果 Block 大小为 128M，切片大小为 100M，那么每个 Block 剩下的 28M，会跨节点进行网络通信传输，并且有额外的 IO 开销，浪费资源。

- **切片时，不考虑数据集整体，而是逐个针对每一个文件单独切片。**

- 每次切片时，都要判断切完剩下的部分是否大于块的 1.1 倍，不大于 1.1 倍就划分为一个切片。

- 切片规划文件会上传至 Yarn，Yarn 上的 MRAppMaster 就可以根据切片规划文件开启对应数量的 MapTask。

**FileInputFormat 实现类**

在运行 MapReduce 程序时，输入的文件格式包括：基于行的日志文件、二进制格式文件、数据库表等。 那么，针对不同的数据类型， MapReduce 是如何读取这些数据的呢？

FileInputFormat 常见的接口实现类包括：TextInputFormat、KeyValueTextInputFormat、NLineInputFormat、CombineTextInputFormat 和自定义 InputFormat 等。

**TextInputFormat**

**TextInputFormat 是默认的 FileInputFormat 实现类**。按行读取每条记录。 键是存储该行在整个文件中的起始字节偏移量， LongWritable 类型。值是这行的内容，不包括任何行终止符（换行符和回车符），Text 类型。

**CombineTextInputFormat**

框架默认的 TextInputFormat 切片机制是对任务按文件规划切片，**不管文件多小，都会是一个单独的切片，都会交给一个 MapTask**，这样如果有大量小文件，就会产生大量的 MapTask，处理效率极其低下。

CombineTextInputFormat 用于**小文件过多的场景**，它可以将多个小文件从逻辑上规划到一个切片中，这样，多个小文件就可以交给一个 MapTask 处理。

虚拟存储切片最大值设置

```java
CombineTextInputFormat.setMaxInputSplitSize(job, 4194304); // 4m
```

注意：虚拟存储切片最大值设置最好根据实际的小文件大小情况来设置具体的值。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202309190126872.png)

---

### Partition 分区

- **ReduceTask 个数的改变导致了数据分区的产生，**而不是数据分区导致了 ReduceTask 个数的改变。
- 如果我们手动设置了 ReduceTask 的个数 > 1，那么会使用相应的 Partitioner。
- 默认 HashPartitioner，按照 key 的 hashCode % numReduceTasks 分区。

```java
public class HashPartitioner<K, V> extends Partitioner<K, V> {
    public int getPartition(K key, V value, int numReduceTasks) {
        return (key.hashCode() & Integer.MAX_VALUE) % numReduceTasks;
    }
}
```

- 具体在 `context.write(key, value)` 里的 `write()` 方法里，有一个 `collector.collect(key, value, partitioner.getPartition())` 方法，里面的 `partitioner` 就是默认的 HashPartitioner。

---

### Sort 排序

排序是 MapReduce 框架中最重要的操作之一。

MapTask 和 ReduceTask 均会对数据按照 key 进行排序。该操作属于 Hadoop 的默认行为。任何应用程序中的数据均会被排序，而不管逻辑上是否需要。

默认排序是按照字典顺序排序，且实现该排序的方法是快速排序。

!!! question "为什么强制要求排序？"

    为了方便分组。如果不排序，到 reduce 阶段时，为了合并计算所有相同的 key 对应的 value，每个 key 都必须 O(n) 地扫描一次文件。如果排序了，每次只需要比较当前 key 是否和上一个相同即可。

对于 MapTask，它会将处理的结果暂时放到环形缓冲区中，**当环形缓冲区使用率达到一定阈值后，再对缓冲区中的数据进行一次快速排序**，并将这些有序数据溢写到磁盘上，而当数据处理完毕后，它会对磁盘上所有文件按照分区进行归并排序。

对于 ReduceTask，它从每个 MapTask 上远程拷贝相应的数据文件，如果文件大小超过一定阈值，则溢写磁盘上，否则存储在内存中。如果磁盘上文件数目达到一定阈值，则进行一次归并排序以生成一个更大文件。当所有数据拷贝完毕后，**ReduceTask 统一对内存和磁盘上的所有数据进行一次归并排序。**

自定义的 bean 对象做为 key 传输，**需要实现 WritableComparable 接口，重写 compareTo 方法**，就可以实现排序。

---

### Combiner 规约

**Combiner in one sentence：对 Map 端的输出先做一次局部合并，以减少在 Map 和 Reduce 节点之间的数据传输量。**

通过使用 Combiner，可以在 Map 任务的本地进行一些预先合并的操作，从而减少输出数据量。这样可以降低数据传输到 Reducer 的成本，并减少网络带宽的使用。特别是在 Map 输出数据具有冗余或重复的情况下，Combiner 可以大大减少数据传输量，提高整体性能。

- Combiner 是 MR 程序中 Mapper 和 Reducer 之外的一种组件。

- Combiner 组件的父类就是 Reducer。

- Combiner 和 Reducer 的区别在于运行的位置：

  - **Combiner 是在每一个 MapTask 所在的节点运行，是局部聚合**
  - **Reducer 是接收全局所有 Mapper 的输出结果，是全局聚合**

- **Combiner 的意义就是对每一个 MapTask 的输出进行局部汇总，以减小网络传输量。**

- **Combiner 能够应用的前提是不能影响最终的业务逻辑**，而且，Combiner 的输出 kv 应该跟 Reducer 的输入 kv 类型要对应起来。

自定义 combiner：

- 自定义一个 Combiner 继承 Reducer，重写 reduce() 方法
- 在 Job 驱动类中设置：

```java
job.setCombinerClass(WordCountCombiner.class);
```

注意：如果 MR 程序没有 Reduce 阶段（reduceTask 个数为 0），那么整个 shuffle 过程也没有了，combiner 也就不会起作用，只会有 map() 方法的输出。

小技巧：直接使用驱动中自己实现的 Reducer 代替 Combiner 即可。

---

### OutputFormat 数据输出

默认实现类为 TextOutputFormat。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202309210145053.png)

---

## :fire: MapTask 工作机制详解

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310030234857.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310030236450.png)

- 内存缓冲区的目的是为了减少磁盘 I/O
- 分区只会在 ReduceTask >= 2 的时候产生。

**详细步骤：**

==**Step 1**==

- 读取数据组件 `InputFormat`（默认 `TextInputFormat`）会通过 `getSplits()` 方法对输入目录中文件进行逻辑规划得到 splits，有多少个 split 就对应启动多少个 MapTask。默认情况下 split 与 block 的是一对一关系。

- 逻辑规划后，由 `RecordReader` 对象（默认 `LineRecordReader`）进行读取，以回车换行作为分隔符，读取一行数据，返回 `<key，value>`。key 表示每行首字符偏移值，value 表示这一行文本内容。

- 读取 split 返回 `<key,value>`，进入用户自己继承的 Mapper 类中，执行用户重写的 map() 方法。RecordReader 读取一行数据，调用一次 map() 方法处理。

==**Step 2**==

- map() 处理完之后，将 map() 的每条结果通过 `context.write()` 由 collector 进行 `collect()` 数据收集。在 `collect()` 中，会先对其进行分区处理，默认使用 HashPartitioner。`collector.collect(key, value, partitioner.getPartition())` 。

- 分区计算之后，会将数据写入内存，内存中这片区域叫做**环形缓冲区**，缓冲区的作用是批量收集 map 结果，减少磁盘 IO 的影响。`<key, value>` 对以及 partition 的结果都会被写入缓冲区。当然写入之前，key 与 value 值都会被序列化成**字节数组**。

- 环形缓冲区就是内存中的一块区域，**底层是字节数组**。缓冲区是有大小限制，默认是 100MB。

==**Step 3**==

- 当 MapTask 的输出结果很多时，就可能会撑爆内存，所以需要在一定条件下将缓冲区中的数据临时写入磁盘，然后重新利用这块缓冲区。这个从内存往磁盘写数据的过程被称为 Spill，中文可译为**溢写**。

- **溢写的过程是由单独线程来完成，不影响往缓冲区写 map 结果的线程**。溢写线程启动时不应该阻止 map 的结果输出，所以整个缓冲区有个溢写的比例 spill.percent。这个比例默认是 0.8，也就是**当缓冲区的数据已经达到阈值**（buffer size \* spill percent = 100MB \* 0.8 = 80MB），**溢写线程启动，锁定这 80MB 的内存，执行溢写过程**。Map task 的输出结果还可以往剩下的 20MB 内存中写，互不影响。

- 当溢写线程启动后，需要对这 80MB 空间内的 key 做排序 (Sort)。**排序是 MapReduce 模型默认的行为**，这里的排序也是对序列化的字节做的排序。

- 如果 job 设置过 Combiner，**排序之后，就是使用 Combiner 的时候了**。将有相同 key 的键值对的 value 加起来，减少溢写到磁盘的数据量。Combiner 会优化 MapReduce 的中间结果，所以它在整个模型中会多次使用。

溢写的过程总结：

- 分区（如果有）
- 排序
- combiner（如果有）
- 压缩（如果有）

==**Step 4**==

- 每次溢写会在磁盘上生成一个临时文件，如果 map 的输出结果真的很大，有多次这样的溢写发生，磁盘上相应的就会有多个临时文件存在。

- 当整个数据处理结束之后开始对磁盘中的临时文件进行 merge 合并，因为最终的文件只有一个，写入磁盘，并且为这个文件提供了一个索引文件，以记录每个 reduce 对应数据的偏移量。

---

## :fire: ReduceTask 工作机制详解

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310030301547.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310030308950.png)

==**Step 1**==

- **Copy 阶段**，简单地拉取数据。Reduce 进程启动一些数据 copy 线程 (Fetcher)，通过 HTTP 方式请求 MapTask **获取属于自己的数据**。

注：EventFetcher 用于判断哪些 MapTask 已经完成工作，Fetcher 线程用于实际执行拉取数据。

==**Step 2**==

- **Merge & Sort 阶段**。Copy 过来的数据会先放入内存缓冲区中，这里的缓冲区大小要比 Map 端的更为灵活。

- **merge 有三种形式**：内存到内存，内存到磁盘，磁盘到磁盘。**默认情况下第一种形式不启用**。当内存中的数据量到达一定阈值，就启动内存到磁盘的 merge。第二种 merge 方式一直在运行，直到没有 map 端的数据时才结束，然后启动第三种磁盘到磁盘的 merge 方式生成最终的文件。

- 把分散的数据合并成一个大的数据后，还会再对合并后的数据排序。默认 key 的字典序排序。

==**Step 3**==

- **Reduce 阶段**。对排序后的键值对调用 reduce 方法，**key 相等的键值对组成一组，调用一次 reduce() 方法**。所谓分组就是纯粹的前后两个 key 比较，如果相等就继续判断下一个是否和当前相等。（为什么可以这样？因为经过了排序）

- reduce 处理的结果会调用默认输出组件 `TextOutputFormat` 写入到指定的目录文件中。`TextOutputFormat` 默认是一次输出写一行，key 和 value 之间以制表符 `\t` 分割。

---

## Shuffle 工作机制详解

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310030317655.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310030413591.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310030418522.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310030421662.png)

---

## MapReduce 中的排序

在 Map 任务和 Reduce 任务的过程中，一共发生了 3 次排序

- 当 map 函数产生输出时，会首先写入内存的环形缓冲区，当达到设定的阀值，在溢写磁盘之前，后台线程会对缓冲区的数据进行分区，在每个分区中进行内排序。

- 在 Map 任务完成之前，磁盘上存在多个已经分好区，并排好序的溢写文件，这时溢写文件将被合并成一个已分区且已排序的输出文件。由于溢写文件已经经过第一次排序，所有合并文件只需要再做一次归并排序即可使输出文件整体有序。

- 在 reduce 阶段，需要将多个 Map 任务的输出文件 copy 到 ReduceTask 中后合并，由于经过第二次排序，所以合并文件时只需再做一次归并排序即可使输出文件整体有序。

在这 3 次排序中第一次是内存缓冲区溢写时做的内排序，使用的算法是快速排序，第二次排序和第三次排序都是在文件合并阶段发生的，使用的是归并排序。

!!! question "为什么需要排序？"

排序的**主要目的是为了实现数据的分组**，由于在 Reducer 中相同的 key 会被分到同一个分组中，因此对数据进行排序可以保证相同的 key 都在相邻的位置，方便后续的 reduce 过程。

---

## Reduce Side Join

- reduce side join，顾名思义，**在 reduce 阶段执行 join 关联操作**。
- 这也是最容易想到和实现的 join 方式。因为通过 shuffle 过程就可以将相关的数据分到相同的分组中，这将为后面的 join 操作提供了便捷。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310030433450.png)

**步骤：**

- mapper 分别读取不同的数据集；
- mapper 的输出中，通常以 join 的字段作为输出的 key；
- 不同数据集的数据经过 shuffle，key 一样的会被分到同一分组处理；
- 在 reduce 中根据业务需求把数据进行关联整合汇总，最终输出。

**弊端：**

- reduce 端 join 最大的问题是整个 join 的工作是在 reduce 阶段完成的，但是通常情况下 MapReduce 中 reduce 的并行度是极小的（默认是 1 个），这就使得**所有的数据都挤压到 reduce 阶段处理，压力颇大**。虽然可以设置 reduce 的并行度，但是又会导致最终结果被分散到多个不同文件中。
- 并且在数据从 mapper 到 reducer 的过程中，**shuffle 阶段十分繁琐，数据集大时成本极高**。

[12--MapReduce Join 关联操作--reduce side join--Mapper 类代码实现\_哔哩哔哩\_bilibili](https://www.bilibili.com/video/BV11N411d7Zh?p=233&vd_source=f3af28d1fd89af1eb80db058885d7130)

---

## 分布式缓存

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310030504745.png)

**使用方式：**

- Step1：添加缓存文件。可以使用 MapReduce 的 API 添加需要缓存的文件。

```java
//添加归档文件到分布式缓存中
job.addCacheArchive(URI uri);
//添加普通文件到分布式缓存中
job.addCacheFile(URI uri);
```

注意：缓存文件必须放在 HDFS 上（客户端将本地文件上传至 HDFS），即必须在集群模式下才能使用。

- Step2：MapReduce 程序中读取缓存文件。

在 Mapper 类或者 Reducer 类的 `setup()` 方法中，用 `BufferedReader` 获取分布式缓存中的文件内容。

`BufferedReader` 是带缓冲区的字符流，能够减少访问磁盘的次数，提高文件读取性能；并且可以一次性读取一行字符。

```java
protected void setup(Context context) throw IOException,InterruptedException{
   FileReader reader = new FileReader("myfile");
   BufferReader br = new BufferedReader(reader);
......
}
```

---

## Map Side Join

Map Side Join 指的是在 Map 阶段执行 Join 操作，并且**通常程序也没有了 Reduce 阶段**，避免了繁琐的 Shuffle 阶段。

Map Side Join 适合小表 Join 大表的情况。

实现 Map Side Join 的关键是使用 MapReduce 的分布式缓存。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310051522131.png)

**实现思路：**

- 首先分析处理的数据集，使用分布式缓存技术将小的数据集进行分布式缓存。
- MapReduce 框架在执行的时候会自动将缓存的数据分发到各个 MapTask 运行的机器上。
- **在 mapper 初始化的时候从分布式缓存中读取小数据集数据**，然后和自己读取的大数据集进行 join 关联，输出最终的结果。

**优势：**

- 整个 join 的过程没有 shuffle，没有 reduce，减少 shuffle 时候的数据传输成本。
- 并且 mapper 的并行度可以根据输入数据量自动调整（逻辑切片），充分发挥分布式计算的优势。

**缺点：**

- 只有在执行 Map 端连接操作的其中一个表足够小以适应内存时，Map 端连接才是适当的。因此，对于两个表都包含大量数据的情况，执行 Map 端连接是不适合的。

[17--MapReduce Join 关联操作--map side join--案例需求与实现思路\_哔哩哔哩\_bilibili](https://www.bilibili.com/video/BV11N411d7Zh?p=238&vd_source=f3af28d1fd89af1eb80db058885d7130)

---

## :fire: MapReduce 优化

MapReduce 的优化核心在于**修改数据文件类型、合并小文件、使用压缩**等方式，通过降低 IO 开销来提升 MapReduce 过程中 Task 的执行效率。

此外，也可以通过调节一些**参数**来从整体上提升 MapReduce 的性能。

从整个 MapReduce 流程上来看：

- 数据输入阶段：合并小文件、采用 CombineTextInputFormat 解决大量小文件的问题
- Map 阶段：减少溢写次数（增加缓冲区大小、提高阈值）、减少合并次数（增加 merge 的文件数量阈值）、使用 Combiner 进行局部聚合、使用压缩
- Reduce 阶段：设置合理的 ReduceTask 个数
- 考虑数据倾斜问题

### 文件存储格式优化

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310030544471.png)

**行式存储适合数据的写入，列式存储适合数据的查询。**

**Sequence File：**

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310021715272.png)

优点：

- 二进制格式存储，比文本文件更紧凑
- 支持不同级别的压缩（Record 级别和 Block 级别）
- 文件支持拆分和并行处理，适合 MapReduce 程序

缺点：

- 二进制格式不方便查看
- 特定于 Hadoop，只有 Java API 可以与之交互

SequenceFile 文件，主要由一条条 record 记录组成；每个 record 是键值对形式的

- SequenceFile 文件可以作为小文件的存储容器；

  - 每条 record 保存一个小文件的内容
  - 小文件名作为当前 record 的键；
  - 小文件的内容作为当前 record 的值；
  - 如 10000 个 100KB 的小文件，可以编写程序将这些文件放到一个 SequenceFile 文件。

- 一个 SequenceFile 是**可分割**的，所以 MapReduce 可将文件切分成块，每一块独立操作。

我们直接将数据写入一个 SequenceFile 文件，省去小文件作为中间媒介。

---

**ORC File：**

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310030556568.png)

- **ORC 文件空间更小、列处理更快、可以构建索引、支持压缩**
- 列式存储适合数据查询，也适合数据压缩、编码优化。
- 一个 Stripe = 一个 horizontal partition（行组）。
- ORC 文件是以**二进制方式存储**的，所以是不可以直接读取。

---

### 数据压缩优化

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310032141227.png)

MR 过程中两个输出阶段可以进行压缩：

- Map 的输出阶段，减少 Mapper 和 Reducer 间网络传输的开销，加快文件传输效率
- Reduce 的输出，减少磁盘存储的空间

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310032146978.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310032156462.png)

---

### 小文件优化

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310032300695.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310032301276.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310032301459.png)

---

### Shuffle 优化

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310032308831.png)

**减少溢写次数：**

- 调整缓冲区大小
- 调整溢写触发比例的阈值

**减少 merge 次数：**

- 默认每次生成 10 个临时文件进行合并，增加文件个数，可以减少 merge 的次数

---

### 参数优化

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310032157027.png)

!!! question "在调整参数属性之后，如何评估 MR 的性能？"

    使用基准测试。
    ```java
    yarn jar /hadoop-3.1.4/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-3.1.4-tests.jar
    ```

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310032251737.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310032254302.png)

---

## MapReduce 数据倾斜

数据倾斜是指在数据处理过程中，某些特定的键（Key）或分区（Partition）上的数据量远远超过其他键或分区，导致部分计算节点的负载不均衡，从而降低整个作业的性能。

常见的数据倾斜通常有两种：

- 数据频率倾斜：某些 key 对应的键值对数量远远大于其他 key 对应的键值对数量
- 数据大小倾斜：部分记录的大小远远大于平均值

在 MapReduce 中，数据倾斜可能发生在 Map 端和 Reduce 端。Reduce 端的数据倾斜常常来源于 MapReduce 的默认分区器。

**解决方案：**

- 自定义分区：基于业务背景进行自定义分区，或者对倾斜严重的 key 再做一层 hash，将数据打散
- 使用 Combiner 进行局部聚合
- 动态调整 Reducer 的个数，使得数据能够更均匀地分布到各个 Reducer 上
