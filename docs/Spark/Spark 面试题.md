## Spark on Yarn

Yarn 架构：

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311181704165.png)

- **ResourceManager（RM）**：全局的资源管理器，负责管理整个集群的资源
- **NodeManager（NM）**：每个节点上的资源和任务的管理器
- **ApplicationMaster（AM）**：应用程序管理器，负责应用程序的管理工作

---

Spark Driver：

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311181708345.png)

Spark 应用程序由主程序中的 SparkContext（或 SparkSession）对象协调。简单地说，初始化 SparkContext 的代码就是你的 Driver。Driver 进程**管理作业流程并调度任务**，在应用程序运行期间始终可用。

---

Spark on Yarn：

当在 YARN 上运行 Spark 作业，每个 Spark executor 作为一个 YARN 容器运行。Spark 可以使得多个 Tasks 在同一个容器里面运行。Spark on Yarn 通常有以下两种运行模式：

- Client 模式
- Cluster 模式

---

Client 模式：

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311181711811.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311181713904.png)

1. 在 YARN Client 模式下，spark-submit 提交 Spark Job 之后，就会**在提交的本地机器上启动一个对应的 Driver**
2. Driver 启动后会与 ResourceManager 建立通讯并发起启动 ApplicationMaster 请求
3. ResourceManage 接收到这个 Job 时，会在集群中选一个合适的 NodeManager 并分配一个 Container，及启动 ApplicationMaster（初始化 SparkContext）
4. ApplicationMaster 的功能相当于一个 ExecutorLaucher ，负责向 ResourceManager 申请 Container 资源； ResourceManage 便会与 NodeManager 通信，并启动 Container
5. ApplicationMaster 对指定 NodeManager 分配的 Container 发出启动 Executor 进程请求
6. Executor 进程启动后会向 Driver 反向注册，Executor 全部注册完成后 Driver 开始执行执行 Job 任务
7. Driver 中的 SparkContext 分配 Task 给 Executor 执行，Executor 运行 Task 并向 Driver 汇报运行的状态、进度、以及最终的计算结果；让 Driver 随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务；应用程序运行完成后，ApplicationMaster 向 ResourceManager 申请注销并关闭自己。

---

Cluster 模式：

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311181714154.png)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202311181714211.png)

在 YARN Cluster 模式下，spark-submit 提交 Spark Job 之后，就会在 YARN 集群中启动一个对应的 ApplicationMaster。**此模式中的 Driver 运行在 ApplicationMaster 中**。

---

两种模式的区别：

主要区别在于 Driver 的创建的位置不一样，Client 方式是直接在本地机器上创建一个 Driver 进程，而 Cluster 方式在通过 ResourceManager 在某一个 NodeManager 中创建一个 Driver。

在使用场景当中，Yarn Client 方式一般适用于进行 Job 的调试（Debug），因为 Driver 是在本地可以直接远程断点调试，而且 Driver 会与 Executor 进行大量的通信就会造成占用大量 IO ；Yarn Cluster 方式一般适用于生产环境，因为 Driver 运行在某一个 NodeManager 中就不会出现某一台机器出现网卡激增的情况，缺点就是运行的 Job 日志不能在机器本地实时查看而是需要通过 Job Web 界面查看。

---

## Spark 的三个开发入门类

- SparkConf

SparkConf 是 Spark 的配置类，作用是将默认配置文件中的 kv 对加载到内存之中。另外我们看到该类中有一个核心的数据结构：ConcurrentHashMap，因此是一个线程线程安全的 kv 结构。该类中有大量的 set 和 get 方法。

注意：一旦 SparkConf 对象被传递给 Spark，它就会被克隆，并且用户不能再修改它。Spark 不支持在运行时修改配置。

- SparkContext

SparkContext 是 Spark 中的主要入口点，它是与 Spark 集群通信的核心对象。SparkContext 的核心作用是初始化 Spark 应用程序运行所需要的核心组件，包括高层调度器(DAGScheduler)、底层调度器(TaskScheduler)和调度器的通信终端(SchedulerBackend)，同时还会负责 Spark 程序向 Master 注册程序等。

- SparkSession

SparkSession 也是 Spark 中的主要入口点。SparkSession 主要用在 SparkSQL 中，当然也可以用在其他场合，他可以代替 SparkContext。SparkSession 实际上封装了 SparkContext。

---

## RDD

---

## Spark OOM

- **情况一：在 Driver 端出现 OOM**

一般来说 Driver 的内存大小不用设置，但是当出现使用 collect() 等 Action 算子时，Driver 会将所有的数据都拉取到 Driver 端，如果数据量过大，就会出现 OOM 的情况。

- **情况二：mapPartitions OOM**

mapPartitions 可以以分区为单位进行数据转换操作，但是会将整个分区的数据加载到内存进行引用，处理完的数据是不会被释放的，存在对象的引用，只有程序结束才会释放。因此在内存较小、数据量较大的场合下，容易导致 OOM。
