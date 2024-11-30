# UDF 开发

在 Hive 中一共有三种用户自定义函数：

1. **UDF**：一行输入（可以传入多个参数），返回一个（一行一列）结果
2. **UDTF**：表生成函数，一行输入，输出多行结果
3. **UDAF**：聚合函数，多行输入，一行输出

---

## 开发步骤

### 创建项目、导入依赖

```xml title="pom.xml"
<dependency>
    <groupId>org.apache.hive</groupId>
    <artifactId>hive-exec</artifactId>
    <version>2.3.9</version>
</dependency>
<!-- 如果使用 Spark -->
<!-- <dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-common</artifactId>
    <version>3.3.1</version>
</dependency> -->
```

---

### 编写 UDF 逻辑

Hive 提供了两个实现 UDF 的方式：

**第一种：继承 UDF 类**

优点：

- 实现简单
- 支持 Hive 的基本类型、数组和 Map
- 支持函数重载

缺点：

- 逻辑较为简单，只适合用于实现简单的函数

**第二种：继承 GenericUDF 类**

优点：

- 支持任意长度、任意类型的参数
- 可以根据参数个数和类型实现不同的逻辑
- 可以实现初始化和关闭资源的逻辑(initialize、close)

缺点：

- 实现比继承 UDF 要复杂一些

与继承 UDF 相比，GenericUDF 更加灵活，可以实现更为复杂的函数

**关于两者的选择**

如果函数具有以下特点，优先继承 UDF 类：

- 逻辑简单，比如英文转小写函数
- 参数和返回值类型简单，都是 Hive 的基本类型、数组或 Map
- 没有初始化或关闭资源的需求

否则考虑继承 GenericUDF 类

---

### 打包并上传

将代码打包成 jar，并将 jar 上传到 Hive 服务器。

> 打包时，需要把依赖打入 jar 包，但不需要 hive-exec，在 pom.xml 中，将其 scope 设为 provided。因为 Hive 的 lib 目录下已经包括了 hive-exec 的 jar，Hive 会自动将其加载。

---

### 创建函数

在 Hive 中创建函数，指定函数的类名和 jar 包路径。

```sql
add jar <打包好的jar的全路径>;
create [temporary] function <函数名> as 'UDF入口类全路径';
drop [temporary] function [if exists] <函数名>;
```

---

## UDF 开发手册

### 继承 UDF 类

第一种方式的代码实现最为简单，只需新建一个类继承 UDF 类，然后实现 `evaluate()` 的具体逻辑即可。

**支持的参数和返回值类型**：支持 Hive 基本类型、数组和 Map

```java
import org.apache.hadoop.hive.ql.exec.UDF;

public class SimpleUDF extends UDF {
    /**
     * 要求如下：
     * 1. 函数名必须为 evaluate
     * 2. 参数和返回值类型可以为：Java基本类型、Java包装类、org.apache.hadoop.io.Writable等类型、List、Map
     * 3. 函数一定要有返回值，不能为 void
     */
    public int evaluate(int a, int b) {
        return a + b;
    }

    // 支持函数重载
    public Integer evaluate(Integer a, Integer b, Integer c) {
        if (a == null || b == null || c == null)
            return 0;
        return a + b + c;
    }
}
```

!!! note

    对于基本类型，最好不要使用 Java 原始类型，当 null 传给 Java 原始类型参数时，UDF 会报错。Java 包装类还可以用于 null 值判断。

| Hive 类型 | Java 原始类型 | Java 包装类 | hadoop.io.Writable |
| --------- | ------------- | ----------- | ------------------ |
| tinyint   | byte          | Byte        | ByteWritable       |
| smallint  | short         | Short       | ShortWritable      |
| int       | int           | Integer     | IntWritable        |
| bigint    | long          | Long        | LongWritable       |
| string    | String        | -           | Text               |
| boolean   | boolean       | Boolean     | BooleanWritable    |
| float     | float         | Float       | FloatWritable      |
| double    | double        | Double      | DoubleWritable     |

| Hive 类型 | Java 类型 |
| --------- | --------- |
| array     | List      |
| Map<K, V> | Map<K, V> |

---

### 继承 GenericUDF 类

第二种方式的代码实现相对复杂一些，需要实现 `initialize()`、`evaluate()`、`close()` 三个方法。

#### initialize()

```java
/**
 * @param arguments 自定义UDF参数的 ObjectInspector 实例
 * @throws UDFArgumentException 参数个数或类型错误时，抛出该异常
 * @return 函数返回值类型
*/
public abstract ObjectInspector initialize(ObjectInspector[] arguments)
    throws UDFArgumentException;
```

`initialize()` 在函数在 GenericUDF 初始化时被调用一次，执行一些初始化操作，包括：

- 判断函数参数个数
- 判断函数参数类型
- 确定函数返回值类型

除此之外，用户在这里还可以做一些自定义的初始化操作，比如初始化 HDFS 客户端等。

**其一：判断函数参数个数**

可通过 arguments 数组的长度来判断函数参数的个数：

```java
// 1. 参数个数检查，只有一个参数
if (arguments.length != 1) // 函数只接受一个参数
    throw new UDFArgumentException("函数需要一个参数"); // 当自定义UDF参数与预期不符时，抛出异常
```

**其二：判断函数参数类型**

ObjectInspector 可用于侦测参数数据类型，其内部有一个枚举类 Category，代表了当前 ObjectInspector 的类型：

```java
public interface ObjectInspector extends Cloneable {
    public static enum Category {
        PRIMITIVE, // Hive 原始类型
        LIST, // Hive 数组
        MAP, // Hive Map
        STRUCT, // 结构体
        UNION // 联合体
    };
}
```

Hive 原始类型又细分了多种子类型，PrimitiveObjectInspector 实现了 ObjectInspector，可以更加具体的表示对应的 Hive 原始类型：

```java
public interface PrimitiveObjectInspector extends ObjectInspector {
  /**
   * The primitive types supported by Hive.
   */
  public static enum PrimitiveCategory {
    VOID, BOOLEAN, BYTE, SHORT, INT, LONG, FLOAT, DOUBLE, STRING,
    DATE, TIMESTAMP, BINARY, DECIMAL, VARCHAR, CHAR, INTERVAL_YEAR_MONTH, INTERVAL_DAY_TIME,
    UNKNOWN
  };
}
```

```java
// 2. 参数类型检查，参数类型为String
if (arguments[0].getCategory() != ObjectInspector.Category.PRIMITIVE // 参数是Hive原始类型
        || !PrimitiveObjectInspector.PrimitiveCategory.STRING.equals(((PrimitiveObjectInspector)arguments[0]).getPrimitiveCategory())) // 参数是Hive的string类型
    throw new UDFArgumentException("函数第一个参数为字符串"); // 当自定义UDF参数与预期不符时，抛出异常
```

**其三：确定函数返回值类型**

initialize() 需要 return 一个 ObjectInspector 实例，用于表示自定义 UDF 返回值类型。**`initialize()` 的返回值决定了 `evaluate()` 的返回值类型。**

ObjectInspector 的源码中，有这么一段注释，其大意是 **ObjectInspector 的实例应该由对应的工厂类获取**，以保证实例的单例等属性。

```java
/**
 * An efficient implementation of ObjectInspector should rely on factory, so
 * that we can make sure the same ObjectInspector only has one instance. That
 * also makes sure hashCode() and equals() methods of java.lang.Object directly
 * works for ObjectInspector as well.
 */
public interface ObjectInspector extends Cloneable { }
```

对于基本类型(byte、short、int、long、float、double、boolean、string)，可以通过 PrimitiveObjectInspectorFactory 的静态字段直接获取：

| Hive 类型 | Writable 类型 | Java 包装类型          |
| --------- | ------------- | ---------------------- |
| tinyint   | writable      | ByteObjectInspector    |
| smallint  | writable      | ShortObjectInspector   |
| int       | writable      | IntObjectInspector     |
| bigint    | writable      | LongObjectInspector    |
| string    | writable      | StringObjectInspector  |
| boolean   | writable      | BooleanObjectInspector |
| float     | writable      | FloatObjectInspector   |
| double    | writable      | DoubleObjectInspector  |

**注意，基本类型返回值有两种：Writable 类型 和 Java 包装类型**：

- 在 initialize 指定的返回值类型为 Writable 类型 时，在 evaluate() 中 return 的就应该是对应的 Writable 实例
- 在 initialize 指定的返回值类型为 Java 包装类型 时，在 evaluate() 中 return 的就应该是对应的 Java 包装类实例

Array、Map<K, V\> 等复杂类型，则可以通过 ObjectInspectorFactory 的静态方法获取：

| Hive 类型 | ObjectInspectorFactory 的静态方法       | 返回值类型 |
| --------- | --------------------------------------- | ---------- |
| Array     | getStandardListObjectInspector(T t)     | List       |
| Map<K, V> | getStandardMapObjectInspector(K k, V v) | Map<K, V>  |

返回值类型为 Map<String, int\> 的示例：

```java
// 3. 自定义UDF返回值类型为 Map<String, int>
return ObjectInspectorFactory.getStandardMapObjectInspector(
        PrimitiveObjectInspectorFactory.javaStringObjectInspector, // Key 是 String
        PrimitiveObjectInspectorFactory.javaIntObjectInspector // Value 是 int
);
```

完整的 `initialize()` 方法：

```java
@Override
public ObjectInspector initialize(ObjectInspector[] arguments) throws UDFArgumentException {
    // 1. 参数个数检查，只有一个参数
    if (arguments.length != 1) // 函数只接受一个参数
        throw new UDFArgumentException("函数需要一个参数"); // 当自定义UDF参数与预期不符时，抛出异常

    // 2. 参数类型检查，参数类型为String
    if (arguments[0].getCategory() != ObjectInspector.Category.PRIMITIVE // 参数是Hive原始类型
            || !PrimitiveObjectInspector.PrimitiveCategory.STRING.equals(((PrimitiveObjectInspector)arguments[0]).getPrimitiveCategory())) // 参数是Hive的string类型
        throw new UDFArgumentException("函数第一个参数为字符串"); // 当自定义UDF参数与预期不符时，抛出异常

    // 3. 自定义UDF返回值类型为 Map<String, int>
    return ObjectInspectorFactory.getStandardMapObjectInspector(
            PrimitiveObjectInspectorFactory.javaStringObjectInspector, // Key 是 String
            PrimitiveObjectInspectorFactory.javaIntObjectInspector // Value 是 int
    );
}
```

---

#### evaluate()

核心方法，自定义 UDF 的实现逻辑，代码实现步骤可以分为三部分：

1. 参数接收
2. 自定义 UDF 核心逻辑
3. 返回处理结果

**第一步：参数接收**

```java
/**
 * Evaluate the GenericUDF with the arguments.
 *
 * @param arguments
 *          The arguments as DeferedObject, use DeferedObject.get() to get the
 *          actual argument Object. The Objects can be inspected by the
 *          ObjectInspectors passed in the initialize call.
 */
public abstract Object evaluate(DeferredObject[] arguments)
  throws HiveException;
```

通过源码注释可知，`DeferedObject.get()` 可以获取参数的值。再看看 DeferredObject 的源码，`DeferedObject.get()` 返回的是 Object，传入的参数不同，会是不同的 Java 类型，以下是 Hive 常用参数类型对应的 Java 类型：

对于 Hive 基本类型，传入的都是 Writable 类型

| Hive 类型 | Java 类型       |
| --------- | --------------- |
| tinyint   | ByteWritable    |
| smallint  | ShortWritable   |
| int       | IntWritable     |
| bigint    | LongWritable    |
| string    | Text            |
| boolean   | BooleanWritable |
| float     | FloatWritable   |
| double    | DoubleWritable  |
| Array     | ArrayList       |
| Map<K, V> | HashMap<K, V>   |

参数接收示例：

```java
// 只有一个参数：Map<String, int>

// 1. 参数为null时的特殊处理
if (arguments[0] == null)
    return ...

// 2. 接收参数
Map<Text, IntWritable> map = (Map<Text, IntWritable>)arguments[0].get();
```

**第二步：自定义 UDF 核心逻辑**

在 evaluate() 中实现自定义 UDF 的核心逻辑即可，比如统计字符串中每个字符出现的次数：

```java
@Override
public Object evaluate(DeferredObject[] arguments) throws HiveException {
    // 1. 参数接收
    if (arguments[0] == null)
        return new HashMap<>();
    String str = ((Text) arguments[0].get()).toString();

    // 2. 自定义UDF核心逻辑
    // 统计字符串中每个字符的出现次数，并将其记录在Map中
    Map<String, Integer> map = new HashMap<>();
    for (char ch : str.toCharArray()) {
        String key = String.valueOf(ch);
        Integer count = map.getOrDefault(key, 0);
        map.put(key, count + 1);
    }
    // 3. 返回处理结果
    return map;
}
```

**第三步：返回处理结果**

这一步和 `initialize()` 的返回值**一一对应**。

基本类型返回值有两种：**Writable 类型** 和 **Java 包装类型**：

- 在 initialize 指定的返回值类型为 Writable 类型 时，在 evaluate() 中 return 的就应该是对应的 Writable 实例
- 在 initialize 指定的返回值类型为 Java 包装类型 时，在 evaluate() 中 return 的就应该是对应的 Java 包装类实例

Hive 数组和 Map 的返回值类型如下：

| Hive 类型 | Java 类型 |
| --------- | --------- |
| Array<T>  | List<T>   |
| Map<K, V> | Map<K, V> |

---

#### getDisplayString()

`getDisplayString()` 返回的是 explain 时展示的信息。

**注意**：这里不能 return null，否则可能在运行时抛出空指针异常，而且这个出现这个问题还不容易排查。

---

#### close()

资源关闭回调函数，不是抽象方法，可以不实现。

---

#### 完整示例

```java
import org.apache.hadoop.hive.ql.exec.UDFArgumentException;
import org.apache.hadoop.hive.ql.metadata.HiveException;
import org.apache.hadoop.hive.ql.udf.generic.GenericUDF;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspectorFactory;
import org.apache.hadoop.hive.serde2.objectinspector.PrimitiveObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.primitive.PrimitiveObjectInspectorFactory;
import org.apache.hadoop.io.Text;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

public class SimpleGenericUDF extends GenericUDF {
    @Override
    public ObjectInspector initialize(ObjectInspector[] arguments) throws UDFArgumentException {
        // 1. 参数个数检查
        if (arguments.length != 1) // 函数只接受一个参数
            throw new UDFArgumentException("函数需要一个参数"); // 当自定义UDF参数与预期不符时，抛出异常

        // 2. 参数类型检查
        if (arguments[0].getCategory() != ObjectInspector.Category.PRIMITIVE // 参数是Hive原始类型
                || !PrimitiveObjectInspector.PrimitiveCategory.STRING.equals(((PrimitiveObjectInspector)arguments[0]).getPrimitiveCategory())) // 参数是Hive的string类型
            throw new UDFArgumentException("函数第一个参数为字符串"); // 当自定义UDF参数与预期不符时，抛出异常

        // 3. 自定义UDF返回值类型为 Map<String, int>
        return ObjectInspectorFactory.getStandardMapObjectInspector(
                PrimitiveObjectInspectorFactory.javaStringObjectInspector, // Key 是 String
                PrimitiveObjectInspectorFactory.javaIntObjectInspector // Value 是 int
        );
    }

    @Override
    public Object evaluate(DeferredObject[] arguments) throws HiveException {
        // 1. 参数接收
        if (arguments[0] == null)
            return new HashMap<>();
        String str = ((Text) arguments[0].get()).toString();

        // 2. 自定义UDF核心逻辑
        // 统计字符串中每个字符的出现次数，并将其记录在Map中
        Map<String, Integer> map = new HashMap<>();
        for (char ch : str.toCharArray()) {
            String key = String.valueOf(ch);
            Integer count = map.getOrDefault(key, 0);
            map.put(key, count + 1);
        }

        // 3. 返回处理结果
        return map;
    }

    @Override
    public String getDisplayString(String[] children) {
        return "这是一个简单的测试自定义UDF~";
    }
}
```

---

## UDAF 开发手册

[UDF 开发手册 - UDAF](https://juejin.cn/post/6948063953876418590)

## UDTF 开发手册

[UDF 开发手册 - UDTF](https://juejin.cn/post/6929326803748126733)
