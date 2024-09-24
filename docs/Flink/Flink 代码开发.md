# Flink 代码开发

## 快速上手

### BatchWordCount

```java
public class BatchWordCount {
    public static void main(String[] args) throws Exception {
        // 1. 创建执行环境
        ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
        // 2. 从文件读取数据 按行读取(存储的元素就是每行的文本)
        DataSource<String> lineDS = env.readTextFile("input/words.txt");
        // 3. 转换数据格式
        FlatMapOperator<String, Tuple2<String, Long>> wordAndOne = lineDS.flatMap((String line, Collector<Tuple2<String, Long>> out) -> {
            String[] words = line.split(" ");
            for (String word : words) {
                out.collect(Tuple2.of(word, 1L));
            }
        }).returns(Types.TUPLE(Types.STRING, Types.LONG));
        // 4. 按照单词分组
        wordAndOne.groupBy(0)
                // 5. 按照单词分组, 求和
                .sum(1)
                // 6. 打印结果
                .print();
    }
}
```

### BoundedStreamWordCount

```java
public class BoundedStreamWordCount {
    public static void main(String[] args) throws Exception {
        // 1. 创建流式执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        // 2. 从文件读取数据
        SingleOutputStreamOperator<Tuple2<String, Long>> DS = env.readTextFile("input/words.txt")
                // 3. 转换数据格式
                .flatMap((String line, Collector<Tuple2<String, Long>> out) -> {
                    String[] words = line.split(" ");
                    for (String word : words) {
                        out.collect(Tuple2.of(word, 1L));
                    }
                }).returns(Types.TUPLE(Types.STRING, Types.LONG));
        // 4. 按照单词分组
        DS.keyBy(t -> t.f0)
                // 5. 按照单词分组, 求和
                .sum(1)
                // 6. 打印结果
                .print();
        // 7. 执行任务
        env.execute();
    }
}
```

### UnboundedStreamWordCount

```java
public class UnboundedStreamWordCount {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.socketTextStream("localhost", 9999)
                .flatMap((String line, Collector<Tuple2<String, Long>> out) -> {
                    String[] words = line.split(" ");
                    for (String word : words) {
                        out.collect(Tuple2.of(word, 1L));
                    }
                }).returns(Types.TUPLE(Types.STRING, Types.LONG))
                .keyBy(t -> t.f0)
                .sum(1)
                .print();
        env.execute();
    }
}
```

---

## DataStream API

### Source

#### 集合数据源

```java
public class SourceTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<Event> stream = env.fromElements(
                new Event("Alice", "www.baidu.com", 123456789L),
                new Event("Bob", "www.google.com", 123456789L),
                new Event("Alice", "www.sina.com", 123456789L)
        );
//        ArrayList<Event> events = new ArrayList<>();
//        events.add(new Event("Alice", "www.baidu.com", 123456789L));
//        events.add(new Event("Bob", "www.google.com", 123456789L));
//        events.add(new Event("Alice", "www.sina.com", 123456789L));
//        DataStreamSource<Event> stream2 = env.fromCollection(events);
        stream.print();
        env.execute();
    }
}
```

#### Socket 数据源

```java
public class SocketTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<String> stream = env.socketTextStream("localhost", 9999);
        stream.print();
        env.execute();
    }
}
```

#### Kafka 数据源

创建 FlinkKafkaConsumer 时需要传入三个参数：

- 第一个参数 topic，定义了从哪些主题中读取数据。可以是一个 topic，也可以是 topic 列表，还可以是匹配所有想要读取的 topic 的正则表达式。当从多个 topic 中读取数据时，Kafka 连接器将会处理所有 topic 的分区，将这些分区的数据放到一条流中去。
- 第二个参数是一个 DeserializationSchema 或者 KeyedDeserializationSchema。Kafka 消息被存储为原始的字节数据，所以需要反序列化成 Java 或者 Scala 对象。下面代码中使用的 SimpleStringSchema，是一个内置的 DeserializationSchema，它只是将字节数组简单地反序列化成字符串。
- 第三个参数是一个 Properties 对象，设置了 Kafka 客户端的一些属性。

```java
public class KafkaTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers", "localhost:9092");
        properties.setProperty("group.id", "consumer-group");
        properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.setProperty("auto.offset.reset", "latest");

        DataStreamSource<String> stream = env.addSource(new FlinkKafkaConsumer<String>("clicks", new SimpleStringSchema(), properties));
        stream.print("Kafka");

        env.execute();
    }
}
```

#### 自定义数据源

创建一个自定义的数据源，需要实现 SourceFunction 接口并且重写两个关键方法：`run()` 和 `cancel()`。

- `run()`方法：使用运行时上下文对象（SourceContext）向下游发送数据
- `cancel()`方法：通过标识位控制退出循环，来达到中断数据源的效果

如果需要自定义并行的数据源（并行度>1），需要实现 ParallelSourceFunction 接口，并重写 `run()` 和 `cancel()`。

```java
public class CustomSource implements SourceFunction<Event> {
    private Boolean running = true;

    @Override
    public void run(SourceContext<Event> sourceContext) throws Exception {
        while (running) {
            sourceContext.collect(new Event("user", "url", System.currentTimeMillis()));
            Thread.sleep(1000);
        }
    }

    @Override
    public void cancel() {
        running = false;
    }
}
```
