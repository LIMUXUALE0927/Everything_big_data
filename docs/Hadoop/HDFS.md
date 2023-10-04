## 模拟一个分布式文件系统

- 如何解决海量数据存的下的问题？-- 分布式存储
- 如何解决大文件传输效率慢的问题？-- 分块存储
- 如何解决海量数据文件查询的问题？-- 元数据记录
- 如何解决硬件故障数据丢失的问题？-- 副本机制
- 如何解决用户查询视角统一的问题？-- 抽象目录树结构（Namespace）

---

## HDFS 优缺点

HDFS，全称 Hadoop Distributed File System，意为 Hadoop 分布式文件系统。HDFS 是一个**主从架构**，由一个主节点 NameNode 和若干从节点 DataNode 构成。在实际的文件读写过程中，还存在一个客户端，用于在用户与 HDFS 之间进行交互。

HDFS 的使用场景：适合一次写入，多次读出的场景。

**HDFS 的优点：**

- 高容错性：数据自动保存多个副本。某一个副本丢失以后，可以自动恢复。
- 适合处理大数据
- 可构建在廉价机器上，通过多副本机制，提高可靠性。

**HDFS 的缺点：**

- 不适合低延迟数据访问
- 无法高效地对大量小文件进行存储
  - 存储大量小文件的话，它会占用 NameNode 大量的内存来存储文件目录和块信息
  - 小文件的寻址时间会超过读取时间，它违反了 HDFS 的设计目标
- 不支持并发写入、文件随机修改
  - 一个文件只能有一个写，不允许多个线程同时写
  - 仅支持数据 append（追加），不支持文件的随机修改

---

## HDFS 的组成架构

HDFS 是经典的主从架构。

**NameNode：**

- 维护 HDFS 的命名空间（记录了 HDFS 中所有文件和目录的层次结构）
- 存储文件的元数据信息（文件名、目录、位置信息、副本数量等）
- 处理客户端读写请求

**DataNode：**

- 存储实际的数据块（和一些元数据：数据块长度、校验和、时间戳等）
- 执行数据块的读写操作

**SecondaryNameNode：**

- 用于支持 HDFS 的高可用和数据恢复
- 并非 NameNode 的热备，当 NameNode 挂掉的时候，并不能马上替换 NameNode 提供服务
- 辅助 NameNode，分担其工作量，如定期合并 Fsimage 和 EditLog，并推送给 NameNode
- 在紧急情况下，可辅助恢复 NameNode
- 一般生产环境不会使用 2NN，而使用 JournalNode

注：editlog 即 NameNode 增量编辑日志，记录了客户端对 HDFS 的所有写操作，一旦系统出现故障，NameNode 可以从 editlog 中恢复元数据。但是随着时间推移，editlog 内容会越来越多，恢复时间也会越来越慢，因此 Haoop 引入 Secondary NameNode 辅助 NameNode，定期合并 fsimage 和 editlog。

**Client：**

- 与 NameNode 交互，获取文件的位置信息
- 与 DataNode 交互，读写数据
- 通过命令访问 HDFS，进行增删改查

**ZKFC（ZooKeeper Failover Controller）：**

- 监控 NameNode 节点的状态，在节点失效时自动进行故障转移

---

## HDFS 元数据

元数据：描述数据的数据。

HDFS 元数据：

- 命名空间：HDFS 中所有文件和目录的层次结构
- 文件名称、类型、位置、副本信息、权限等
  - 每个文件、目录、Block 大约占用 150 字节
  - 元数据存储在 **NameNode 内存**中，也会通过以 fsimage 和 editlog 持久化的方式保证可靠性。

存在形式：

- editlog：HDFS 的增量编辑日志文件，保存客户端对 HDFS 的所有更改记录，一旦系统出现故障，可以从 editlog 进行恢复。是一个有序的、追加写入的日志文件。
- fsimage：HDFS 元数据镜像文件，其中包含了文件系统的元数据信息，包括文件和目录的层次结构、权限、所有者等。它周期性地将元数据的快照写入本地磁盘。

元数据存放路径：

- 在 hadoop 根目录下的 `/etc/hadoop/hdfs-site.xml` 中可以设置 `dfs.namenode.name.dir`

元数据的生成条件：

- 时间：默认 1 小时写入一次 fsimage
- 空间：默认 1 百万次事务操作
- 人工触发：进入 safemode 主动触发

在高可用部署模式下，共享存储系统由多个 JournalNode 节点组成。当 Active NameNode 中有事务提交， Active NameNode 会将 editlog 发给 JournalNode 集群。JournalNode 集群通过 Paxos 协议保证数据一致性。Standby NameNode 定期从 JournalNode 读取 editlog，合并到自己的 fsimage 上。

DataNode 汇报元数据信息（Block Report）：

- 全量块汇报（Full Block Report）：将 DN 中存储的 Block 信息都汇报给 NN，默认间隔时间为 6 小时。集群启动时会触发。
- 增量块汇报（Incremental Block Report）：一旦触发增量块汇报，就立即发送。和心跳检测一起汇报。

---

## HDFS 的常用命令

通用格式：

```bash
hdfs dfs -command
hadoop fs -command
```

列出目录中的文件和子目录：

```bash
hdfs dfs -ls <path>
```

创建一个目录：

```bash
hdfs dfs -mkdir <path>
```

复制本地文件到 HDFS：

```bash
hdfs dfs -put <local_path> <hdfs_path>
```

复制 HDFS 文件到本地：

```bash
hdfs dfs -get <hdfs_path> <local_path>
```

从 HDFS 中删除文件或目录：

```bash
hdfs dfs -rm [-r] <path>
```

从 HDFS 中递归地删除目录：

```bash
hdfs dfs -rm -r <path>
```

查看文件的内容：

```bash
hdfs dfs -cat <path>
```

将 HDFS 中的文件合并到一个本地文件中：

```bash
hdfs dfs -getmerge <hdfs_path> <local_file>
```

查看 HDFS 中文件或目录的元数据信息：

```bash
hdfs dfs -stat <path>
```

---

## HDFS 文件块/Block

HDFS 中的文件在物理上是分块存储（Block），块的大小可以通过配置参数 (`dfs.blocksize`) 来规定，默认大小在 Hadoop2.x/3.x 版本中是 128M/256M。

!!! question "为什么要使用块来进行存储？"

- **支持大文件存储：**
  可以将大文件分成若干块，并存储在不同的节点上，因此一个文件的大小不会受到单个节点的存储容量的限制
- **简化系统设计：**
  因为块大小固定，因此能很方便地计算每个节点能够存储多少文件块；其次，方便了元数据管理，元数据不需要和数据块一起存储
- **适合数据备份：**
  每个文件块都可以冗余存储到多个节点上，大大提高了系统的容错性和可用性

!!! question "文件块大小为什么是 128M？增大或减小有什么影响？"

**最终目的是最小化寻址开销。** 磁盘传输速率普遍在 100MB/s 这个级别，通常来说寻址时间占文件传输时间的 1% 是最佳的，因此如果寻址时间为 10ms，那么传输时间应该在 1s 左右，因此 1s \* 100MB/s = 100 MB，取最接近的 2 的次幂 128MB。

如果文件块的大小太小，会寻址时间占比太大，违背了 HDFS 的设计理念。如果文件块的大小太大，会大大增加文件的传输时间。**因此 HDFS 文件块的大小设置主要取决于磁盘的传输速率。**

总结：文件块越大，寻址时间占比就越小，但传输时间越长；文件块越小，寻址时间占比就越大，但传输时间越短。

如果块大小设置过大，块的传输时间就会太长，同时 MapReduce 程序中一个 MapTask 通常只处理一个块，如果块太大也会影响运行速度。如果块大小设置过小，一方面存放大量小文件会占用 NameNode 的大量内存存储元数据，另一方面寻址时间占比太高，违背了 HDFS 的设计原则。

!!! question "1 个 10MB 的文件，实际占用多少空间？"

1 个 10MB 的文件，会占用一整个 Block 块地址，实际磁盘的占用为 10MB \* 3 = 30MB。

---

## HDFS 心跳机制

- DataNode 启动后向 NameNode 注册，每隔 **3 秒钟**向 NameNode 发送一个心跳 heartbeat
- 心跳返回结果带有 NameNode 给该 DataNode 的命令，如复制块数据到另一 DataNode，或删除某个数据块
- 如果超过 **10 分钟** NameNode 没有收到某个 DataNode 的心跳，则认为该 DataNode 节点不可用
- DataNode 周期性（**6 小时**）的向 NameNode 上报当前 DataNode 上的块状态报告 BlockReport。BlockReport 包含了该 Datanode 上所有的 Block 信息。

心跳的作用：

- 可以判断 DataNode 是否存活
- 通过周期心跳，NameNode 可以向 DataNode 返回指令
- 通过 BlockReport，NameNode 能够知道各 DataNode 的存储情况，如磁盘利用率、块列表
- Hadoop 集群刚开始启动时，99.9%的 block 没有达到最小副本数(`dfs.namenode.replication.min` 默认值为 1)，集群处于安全模式，涉及 BlockReport

---

## HDFS 安全模式

安全模式是 HDFS 所处的一种特殊的只读模式状态，在这种状态下，文件系统只接受读数据请求，而不接受删除、修改等变更请求，是一种保护机制，用于保证文件系统的一致性和完整性。

在 NameNode 主节点启动时，HDFS 首先进入安全模式，集群会开始检查数据块的完整性。DataNode 在启动的时候会向 NameNode 汇报可用的 block 信息，当整个系统达到安全标准时，HDFS 自动离开安全模式。

安全标准指的是 HDFS 中副本个数达到阈值的文件的比例（99.9%）。

---

## :fire: HDFS 写数据流程

核心概念：**Pipeline**

Pipeline 指 HDFS 在写数据过程中采用的一种数据传输方式。客户端将数据块写入第一个 DataNode，第一个 DataNode 保存数据之后再将块复制到第二个 DataNode，后者保存后再复制给第三个 DataNode。

!!! question "为什么采用 Pipeline 传输数据？"

- 最大化网络出口带宽：沿着一个管道传输，客户端不用在多个接受者间分配带宽
- 并行传输：Pipeline 允许多个数据包同时在不同的数据节点上进行传输。多个数据包可以同时在不同的节点上进行传输，大大减少了传输延迟，提高了写入性能。
- 容错机制：Pipeline 提供了容错机制，确保在某个数据节点发生故障时，数据传输可以切换到其他可用的数据节点。这样可以避免因单点故障而导致的写入中断，保证数据的可靠性和写入的连续性。

核心概念：**ACK 应答**

在 HDFS Pipeline 管道传输数据的过程中，传输的反方向会进行 ACK 校验，确保数据传输安全。

!!! note "HDFS 写数据流程"

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202309181615183.png)

- 客户端创建 Distributed FileSystem 模块向 NameNode 请求上传文件
- NameNode 检查：权限、目录是否存在等
- 客户端询问第一个 Block 上传到哪几个 DataNode 服务器上
- NameNode 返回 3 个 DataNode 节点，分别为 dn1、dn2、dn3（机架感知：考虑节点最近和负载均衡）
- 客户端通过 FSDataOutputStream 模块请求 dn1 上传数据，dn1 收到请求会继续调用 dn2，然后 dn2 调用 dn3，建立管道
- dn1、dn2、dn3 逐级应答客户端
- 客户端开始往 dn1 上传第一个 Block（先从磁盘读取数据放到一个本地内存缓存），以 Packet 为单位，dn1 收到一个 Packet 就会传给 dn2，dn2 传给 dn3；dn1 每传一个 packet 会放入一个应答队列等待应答
- 当一个 Block 传输完成之后，客户端再次请求 NameNode 上传第二个 Block 的服务器

**总结：**

- 文件分块 Block（128M）
- 客户端向 NN 发送写请求
- NN 进行检查，记录 Block 信息，返回 3 个 DN
- 建立通信管道
- 客户端向 DN 发送 Block，以流式写入的方式：
  - 将 Block 划分为多个 packet
  - 客户端以 packet 为单位传输数据
  - 发送数据的同时还会将 packet 放入一个应答队列等待应答
- 一个 Block 传输完毕之后，继续下一个 Block

!!! question "传输数据的单位是什么？"

以 packet（64K）为单位，一个 packet 内包含多个 chunk(512byte) + checksum(4byte)。

客户端在创建输出流的时候，会先创建一个**缓冲队列**，缓冲队列中存储的是 chunk + checksum = 516 bytes。攒够到 64K 时，形成一个 packet，放入 ack 应答队列，再发送。

客户端维护一个 ack 应答队列，每次 packet 发送成功，客户端都会收到一个 ack 应答消息，删除 ack 队列中对应的 packet。对于没收到 ack 应答消息的 packet，ack 队列会把这个 packet 再加入发送缓冲队列，再次尝试发送。

!!! question "写数据时 NameNode 是如何选择 DataNode 节点的？"

基于**副本放置策略/机架感知机制**。第一个副本在客户端所在节点，如果客户端在集群外，随机选择一个。第二个副本在另一个机架的随机一个节点。第三个副本在第二个副本所在机架的随机节点。

这种策略减少了机架间的数据传输，提高了写操作的效率。同时，由于机架错误的概率远比节点错误的概率低，这种策略也能保证数据的可靠性。

!!! question "写数据时 DataNode 突然宕机怎么办？"

客户端写数据时与 DN 建立 pipeline 管道，管道正向是客户端向 DN 传输数据，反向是接收 DN 的 ack 确认。

- 如果 DN 宕机，客户端接收不到 ack 确认，为了确保数据不会丢失，ack 队列中的所有 packet 会被重新添加到 data queue（缓冲队列） 末尾
- 客户端会通知 NameNode 为当前传输的 Block 生成新的版本，在正常的 DataNode 节点上已经保存好的 block 的 ID 版本会升级，这样发生故障的 DataNode 节点上的 Block 在节点恢复后会被删除
- 失效节点会从 pipeline 中被删除，剩下的数据写入到 pipeline 中的其他 2 个节点中
- NameNode 分配一个新的 DataNode
- 把更新后的 Block 复制一份到新的 DataNode 中

---

## :fire: HDFS 读数据流程

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202309181726379.png)

- 客户端通过 DistributedFileSystem 向 NameNode 请求读取文件
- NameNode 通过查询元数据，找到文件块所在的 DataNode 地址
- 挑选一台 DataNode（先考虑就近原则，然后随机）服务器，请求读取数据
- DataNode 开始传输数据给客户端（从磁盘里面读取数据输入流，以 Packet 为单位来做校验）
- 客户端以 Packet 为单位接收，先在本地缓存，然后写入目标文件

## HDFS 读写流程中的容错机制

Case 1：写数据时 DataNode 挂了

- 如果 DN 宕机，客户端接收不到 ack 确认，为了确保数据不会丢失，ack 队列中的所有 packet 会被重新添加到 data queue（缓冲队列） 末尾
- 客户端会通知 NameNode 为当前传输的 Block 生成新的版本，在正常的 DataNode 节点上已经保存好的 block 的 ID 版本会升级，这样发生故障的 DataNode 节点上的 Block 在节点恢复后会被删除
- 失效节点会从 pipeline 中被删除，剩下的数据写入到 pipeline 中的其他 2 个节点中
- NameNode 分配一个新的 DataNode
- 把更新后的 Block 复制一份到新的 DataNode 中

Case 2：读数据时 DataNode 挂了

- Client 会从 NameNode 给的 Block 地址中选择下一个 DataNode 读取数据，并且记录此有问题的 DataNode，不会再从它上面读取数据

Case 3：读数据时发现 Block 数据出错了

- client 读取 block 数据时，同时会读取到 block 的校验和，若 client 针对读取过来的 block 数据，计算检验和，其值与读取过来的校验和不一样，说明 block 数据损坏。

- 然后 client 从存储此 block 副本的其它 DataNode 上读取 block 数据（也会计算校验和）。同时，client 会告知 namenode 此情况
