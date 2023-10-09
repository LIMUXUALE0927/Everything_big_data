## Yarn 概述

**Yarn 是一个通用的资源管理和调度系统，负责为运算程序提供服务器运算资源**，相当于一个分布式的操作系统平台，而 MapReduce 等运算程序则相当于运行于操作系统之上的应用程序。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310032318894.png)

Yarn 的特点：

- 分布式计算：Yarn 允许用户在集群上部署分布式应用
- 集群资源管理：Yarn 提供了对集群资源的高效管理，包括 CPU、内存和网络带宽等
- 多租户：Yarn 支持多个应用程序运行在同一集群上，而不会互相干扰
- 灵活性：Yarn 支持多种类型的应用程序，包括 MapReduce、Spark、Storm 等

---

## Yarn 基础架构

Yarn 是经典的主从（master/slave）架构。主要由 ResourceManager、NodeManager、ApplicationMaster 和 Container 等组件构成。

**ResourceManager（RM）**：全局的资源管理器，负责管理整个集群的资源

主要作用如下：

- 处理客户端请求
- 监控 NodeManager
- 启动或监控 ApplicationMaster
- 资源的分配与调度

> ResouceManager = 调度器（Scheduler） + 应用程序管理器（Application Manager）

其中， 调度器可以根据容量、队列等限制条件（如每个队列分配一定的资源，最多执行一定数量的作业等），将系统中的资源分配给各个正在运行的应用程序。

应用程序管理器主要负责管理整个系统中所有应用程序，可以接收 job 的提交请求，为应用分配第一个 Container 来运行 ApplicationMaster，包括应用程序提交、与调度器协商资源以启动 ApplicationMaster、监控 ApplicationMaster 运行状态并在失败时重新启动它等

**NodeManager（NM）**：每个节点上的资源和任务的管理器

主要作用如下：

- 管理单个节点上的资源和运行任务
- 处理来自 ResourceManager 的命令
- 处理来自 ApplicationMaster 的命令
- 定期汇报本节点的资源使用情况和各个 Container 的运行状态

**ApplicationMaster（AM）**：应用程序管理器，负责应用程序的管理工作

主要作用如下：

- 为应用程序申请资源并分配给内部的任务
- 任务的监控与容错

**Container**：YARN 中的资源抽象，它封装了某个节点上的多维度资源，如内存、CPU、磁盘、 网络等

---

## :fire: Yarn 作业提交流程/工作机制

[127 尚硅谷\_Hadoop_Yarn 工作机制\_哔哩哔哩\_bilibili](https://www.bilibili.com/video/BV1Qp4y1n7EN?p=127&vd_source=f3af28d1fd89af1eb80db058885d7130)

简单版：

- 用户将应用程序提交到 ResourceManager 上；

- ResourceManager 为应用程序 ApplicationMaster 申请资源，并与某个 NodeManager 通信启动第一个 Container，以启动 ApplicationMaster；

- ApplicationMaster 与 ResourceManager 注册进行通信，为内部要执行的任务申请资源，一旦得到资源后，将于 NodeManager 通信，以启动对应的 Task；

- 所有任务运行完成后，ApplicationMaster 向 ResourceManager 注销，整个应用程序运行结束。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202309291842974.png)

详细版 V1：

- 用户通过 Yarn 客户端向 RM 申请提交应用程序
- RM 为应用程序的 ApplicationMaster 分配一个 Container，然后与对应的 NM 通信，要求在上面启动 AM
- AM 启动后向 RM 注册，与 RM 保持周期性心跳，通过 RPC 通信提交自己的资源需求
- AM 通过心跳领取到资源，与对应的 NM 通信，启动对应的 Container
- NM 会为即将启动的任务设置好环境，下载好需要的资源（jar 包、配置文件等），通过任务启动脚本启动任务
- 拉起后的任务，通过心跳与 AM 汇报自己的运行情况
- 任务执行完后，AM 通知 RM 清理 Container，注销自己

详细版 V2：

1. MR 程序提交到客户端所在的节点。
2. YarnRunner 向 ResourceManager 申请一个 Application。
3. RM 将该应用程序的资源路径返回给 YarnRunner。
4. 该程序将运行所需资源提交到 HDFS 上。
5. 程序资源提交完毕后，申请运行 MRAppMaster。
6. RM 将用户的请求初始化成一个 Task。
7. 其中一个 NodeManager 领取到 Task 任务。
8. 该 NodeManager 创建容器 Container，并产生 MRAppmaster。
9. Container 从 HDFS 上拷贝资源到本地。
10. MRAppMaster 向 RM 申请运行 MapTask 资源。
11. RM 将运行 MapTask 任务分配给另外两个 NodeManager，另两个 NodeManager 分别领取任务并创建容器。
12. MR 向两个接收到任务的 NodeManager 发送程序启动脚本，这两个 NodeManager 分别启动 MapTask，MapTask 对数据分区排序。
13. MRAppMaster 等待所有 MapTask 运行完毕后，向 RM 申请容器，运行 ReduceTask。
14. ReduceTask 向 MapTask 获取相应分区的数据。
15. 程序运行完毕后，MR 会向 RM 申请注销自己。

---

## Yarn 资源调度器

目前，Hadoop 作业调度器主要有 3 种：FIFO Scheduler、容量调度器（Capacity Scheduler）和公平调度器（Fair Scheduler）。Apache Hadoop 3.1.3 默认的资源调度器是 Capacity Scheduler。

具体设置见：`yarn-default.xml`

**FIFO Scheduler**:

单队列，根据提交作业的先后顺序，先来先服务。

优点：简单易懂

缺点：不支持多队列，大任务可能会占用集群所有资源，阻塞其他任务

---

**Capacity Scheduler**:

Capacity Scheduler 是 Yahoo 开发的**多用户**调度器。基于容量的调度算法，对集群资源进行划分，将资源划分为多个资源池，并按照资源池的比例分配资源，以实现资源的共享和隔离。该调度器适合于多用户、多队列的共享资源场景。

容量调度器特点：

- **多队列**：每个队列可配置一定的资源量，每个队列采用 FIFO 调度策略。
- **容量保证**：管理员可为每个队列设置资源最低保证和资源使用上限。
- **灵活性**：如果一个队列中的资源有剩余，可以暂时共享给那些需要资源的队列，而一旦该队列有新的应用程序提交，则其他队列借调的资源会归还给该队列。
- **多租户**: 支持多用户共享集群和多应用程序同时运行。 为了防止同一个用户的作业独占队列中的资源，该调度器会对同一用户提交的作业所占资源量进行限定。

---

**Fair Scheduler**:

Fair Scheduler 是 Facebook 开发的**多用户**调度器。基于公平调度算法，通过维护各个任务的资源需求和使用情况，动态地分配资源，以保证各个任务获得公平的资源分配。适用于多用户、多队列的共享资源场景。

与容量调度器的相同点 ：

- 多队列：支持多队列多作业
- 容量保证：管理员可为每个队列设置资源最低保证和资源使用上限
- 灵活性：如果一个队列中的资源有剩余，可以暂时共享给那些需要资源的队列，而一旦该队列有新的应用程序提交，则其他队列借调的资源会归还给该队列。
- 多租户：支持多用户共享集群和多应用程序同时运行；为了防止同一个用户的作业独占队列中的资源，该调度器会对同一用户提交的作业所占资源量进行限定。

与容量调度器不同点：

- 核心调度策略不同

  - 容量调度器：优先选择**资源利用率低**的队列
  - 公平调度器：优先选择对资源的**缺额**比例大的

- 每个队列可以单独设置资源分配方式

  - 容量调度器：FIFO、 DRF（内存 + CPU）
  - 公平调度器：FIFO、FAIR、DRF

---

总结：

- FIFO Scheduler：单队列，按照任务提交顺序依次分配资源，不考虑任务的优先级和资源需求。适用于小型简单应用场景。
- Capacity Scheduler：基于容量的调度算法，对集群资源进行划分，将资源划分为多个资源池，并按照资源池的比例分配资源，以实现资源的共享和隔离。该调度器适合于多用户、多队列的共享资源场景。
- Fair Scheduler：基于公平调度算法，通过维护各个任务的资源需求和使用情况，动态地分配资源，以保证各个任务获得公平的资源分配。适用于多用户、多队列的共享资源场景。
