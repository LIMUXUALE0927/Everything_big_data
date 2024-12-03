# Flink SQL

Apache Flink 有两种关系型 API 来做流批统一处理：Table API 和 SQL。Table API 是用于 Scala 和 Java 语言的查询 API，它可以用一种非常直观的方式来组合使用选取、过滤、join 等关系型算子。Flink SQL 是基于 Apache Calcite 来实现的标准 SQL。无论输入是连续的（流式）还是有界的（批处理），在两个接口中指定的查询都具有相同的语义，并指定相同的结果。

## TableEnvironment

TableEnvironment 是 Table API 和 SQL 的核心概念。它承担着以下职责：

- 在内部的 catalog 中注册 Table
- 注册外部的 catalog
- 加载可插拔模块
- 执行 SQL 查询
- 注册自定义函数（scalar、table 或 aggregation）
- 负责 DataStream 和 Table 之间的转换（面向 StreamTableEnvironment）

Table 总是与特定的 TableEnvironment 绑定。不能在同一条查询中使用不同 TableEnvironment 中的表，例如，对它们进行 join 或 union 操作。

TableEnvironment 可以通过静态方法 `TableEnvironment.create()` 创建：

```java
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.TableEnvironment;

EnvironmentSettings settings = EnvironmentSettings
    .newInstance()
    .inStreamingMode()
    //.inBatchMode()
    .build();

TableEnvironment tEnv = TableEnvironment.create(settings);
```

或者，用户可以从现有的 StreamExecutionEnvironment 创建一个 StreamTableEnvironment 与 DataStream API 互操作：

```java
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;

StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tEnv = StreamTableEnvironment.create(env);
```

---

## 翻译 SQL 与查询优化

不论输入数据源是流式的还是批式的，Table API 和 SQL 查询都会被转换成 DataStream 程序。查询在内部表示为逻辑查询计划，并被翻译成两个阶段：

- 优化逻辑执行计划
- 翻译成 DataStream 程序

Apache Flink 使用并扩展了 Apache Calcite 来执行复杂的查询优化，涵盖了一系列基于规则和成本的优化，具体如下：

- **基于 Apache Calcite 的子查询解相关**：用于特定的子查询处理优化。
- **投影剪裁**：对投影相关操作进行优化剪裁。
- **分区剪裁**：针对分区相关内容实施优化处理。
- **过滤器下推**：将过滤器进行合理下推以优化整体流程。
- **子计划消除重复数据以避免重复计算**：通过消除子计划中的重复数据，减少不必要的重复运算。

- **特殊子查询重写**：包含两部分内容，

  : - **将 IN 和 EXISTS 转换为 left semi-joins**：实现特定逻辑下的转换优化。
  : - **将 NOT IN 和 NOT EXISTS 转换为 left anti-join**：完成对应逻辑的转换，提升查询效率。

- **可选 join 重新排序**：通过 `table.optimizer.join-reorder-enabled` 启用，可对 join 操作进行重新排序优化。

**注意**：当前仅在子查询重写的结合条件下支持 IN / EXISTS / NOT IN / NOT EXISTS。

优化器不仅基于计划，而且还基于可从数据源获得的丰富统计信息以及每个算子（例如 io，cpu，网络和内存）的细粒度成本来做出明智的决策。

高级用户可以通过 CalciteConfig 对象提供自定义优化，通过调用 TableEnvironment＃getConfig＃setPlannerConfig 将其提供给 TableEnvironment。
