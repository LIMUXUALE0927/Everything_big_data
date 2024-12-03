# Flink CDC

## 介绍

Flink CDC 是一个基于流的数据集成工具，旨在为用户提供一套功能更加全面的编程接口（API）。 该工具使得用户能够以 YAML 配置文件的形式，优雅地定义其 ETL（Extract, Transform, Load）流程，并协助用户自动化生成定制化的 Flink 算子并且提交 Flink 作业。 Flink CDC 在任务提交过程中进行了优化，并且增加了一些高级特性，如表结构变更自动同步（Schema Evolution）、数据转换（Data Transformation）、整库同步（Full Database Synchronization）以及 精确一次（Exactly-once）语义。

Flink CDC 深度集成并由 Apache Flink 驱动，提供以下核心功能：

- 端到端的数据集成框架
- 为数据集成的用户提供了易于构建作业的 API
- 支持在 Source 和 Sink 中处理多个表
- 整库同步
- 具备表结构变更自动同步的能力（Schema Evolution）

https://developer.aliyun.com/article/1399426

Flink 整库同步依赖的 Netflix 的 DBlog 算法，参考：[A Generic Change-Data-Capture Framework](https://netflixtechblog.com/dblog-a-generic-change-data-capture-framework-69351fb9099b)

## 依赖配置

${flink.version} = 1.18.0

```xml title="pom.xml"
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-java</artifactId>
    <version>${flink.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-streaming-java</artifactId>
    <version>${flink.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-clients</artifactId>
    <version>${flink.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-planner_2.12</artifactId>
    <version>${flink.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-runtime</artifactId>
    <version>${flink.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-api-java-bridge</artifactId>
    <version>${flink.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-base</artifactId>
    <version>${flink.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-mysql-cdc</artifactId>
    <version>3.2.0</version>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.33</version>
</dependency>
```

---

## MySQL 库表数据准备

```sql
-- create database
CREATE DATABASE app_db;

USE app_db;

-- create orders table
CREATE TABLE `orders` (
`id` INT NOT NULL,
`price` DECIMAL(10,2) NOT NULL,
PRIMARY KEY (`id`)
);

-- insert records
INSERT INTO `orders` (`id`, `price`) VALUES (1, 4.00);
INSERT INTO `orders` (`id`, `price`) VALUES (2, 100.00);

-- create shipments table
CREATE TABLE `shipments` (
`id` INT NOT NULL,
`city` VARCHAR(255) NOT NULL,
PRIMARY KEY (`id`)
);

-- insert records
INSERT INTO `shipments` (`id`, `city`) VALUES (1, 'beijing');
INSERT INTO `shipments` (`id`, `city`) VALUES (2, 'xian');

-- create products table
CREATE TABLE `products` (
`id` INT NOT NULL,
`product` VARCHAR(255) NOT NULL,
PRIMARY KEY (`id`)
);

-- insert records
INSERT INTO `products` (`id`, `product`) VALUES (1, 'Beer');
INSERT INTO `products` (`id`, `product`) VALUES (2, 'Cap');
INSERT INTO `products` (`id`, `product`) VALUES (3, 'Peanut');
```

---

## DataStream API

```java

public class FlinkCDC_Datastream {
    public static void main(String[] args) throws Exception {
        // 1. 创建流处理环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        // 2. 开启 checkpoint （本地测试可省略）
        env.enableCheckpointing(5000L);
        env.getCheckpointConfig().setCheckpointTimeout(10000L);
        env.getCheckpointConfig().setCheckpointStorage("file:///Users/lihao/Desktop/Flink-CDC/src/main/resources/ck");
        env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
        env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

        // 3. 创建 MySQLSource
        MySqlSource<String> mySqlSource = MySqlSource.<String>builder()
                .hostname("localhost")
                .port(3306)
                .username("root")
                .password("")
                .databaseList("app_db")
                .tableList("app_db.orders")
                // initial() : 初始化读一次全量数据，之后只读增量数据
                // latest() : 从最新的数据开始读
                // earliest() : 从最早的数据开始读
                .startupOptions(StartupOptions.initial())
                .deserializer(new JsonDebeziumDeserializationSchema())
                .build();
        // 4. 读取数据
        DataStreamSource<String> mysqlDS = env.fromSource(mySqlSource, WatermarkStrategy.noWatermarks(), "MySQLSource");
        // 5. 打印输出
        mysqlDS.print();
        // 6. 执行任务
        env.execute();
    }
}
```

```text
{"before":null,"after":{"id":1,"price":"AZA="},"source":{"version":"1.9.8.Final","connector":"mysql","name":"mysql_binlog_source","ts_ms":0,"snapshot":"false","db":"app_db","sequence":null,"table":"orders","server_id":0,"gtid":null,"file":"","pos":0,"row":0,"thread":null,"query":null},"op":"r","ts_ms":1733062090009,"transaction":null}
{"before":null,"after":{"id":2,"price":"JxA="},"source":{"version":"1.9.8.Final","connector":"mysql","name":"mysql_binlog_source","ts_ms":0,"snapshot":"false","db":"app_db","sequence":null,"table":"orders","server_id":0,"gtid":null,"file":"","pos":0,"row":0,"thread":null,"query":null},"op":"r","ts_ms":1733062090010,"transaction":null}
信息: Connected to localhost:3306 at binlog.000002/4045 (sid:6194, cid:51)
```

---

## SQL API

```java
public class FlinkCDC_SQL {
    public static void main(String[] args) {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);
        tableEnv.executeSql("create table orders (\n" +
                "    id int primary key not enforced,\n" +
                "    price double\n" +
                ") with (\n" +
                "    'connector' = 'mysql-cdc',\n" +
                "    'hostname' = 'localhost',\n" +
                "    'port' = '3306',\n" +
                "    'username' = 'root',\n" +
                "    'password' = '',\n" +
                "    'database-name' = 'app_db',\n" +
                "    'table-name' = 'orders'\n" +
                ")");
        tableEnv.executeSql("select * from orders").print();
    }
}
```

```text
+----+-------------+--------------------------------+
| op |          id |                          price |
+----+-------------+--------------------------------+
| +I |           2 |                          100.0 |
| +I |           1 |                            4.0 |
```
