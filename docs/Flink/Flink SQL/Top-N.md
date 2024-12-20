# Top-N 与 窗口 Top-N

## Top-N

Top-N 查询可以根据指定列排序后获得前 N 个最小或最大值。最小值和最大值集都被认为是 Top-N 查询。在需要从批表或流表中仅显示 N 个底部或 N 个顶部记录时，Top-N 查询是非常有用的。并且该结果集还可用于进一步分析。

Flink 使用 OVER 窗口子句和过滤条件的组合来表达一个 Top-N 查询。借助 OVER 窗口的 PARTITION BY 子句能力，Flink 也能支持分组 Top-N。例如：实时显示每个分类下销售额最高的五个产品。对于批处理和流处理模式的 SQL，都支持 Top-N 查询。

下面展示了 Top-N 的语法：

```sql
SELECT [column_list]
FROM (
   SELECT [column_list],
     ROW_NUMBER() OVER ([PARTITION BY col1[, col2...]]
       ORDER BY col1 [asc|desc][, col2 [asc|desc]...]) AS rownum
   FROM table_name)
WHERE rownum <= N [AND conditions]
```

**参数说明：**

- `ROW_NUMBER()`：根据分区数据的排序，为每一行分配一个唯一且连续的序号，从 1 开始。目前，只支持 ROW_NUMBER 作为 OVER 窗口函数。未来会支持 `RANK()` 和 `DENSE_RANK()`。
- `PARTITION BY col1[, col2...]`：指定分区字段。每个分区都会有一个 Top-N 的结果。
- `ORDER BY col1 [asc|desc][, col2 [asc|desc]...]`： 指定排序列。 每个列的排序类型（ASC/DESC）可以不同。
- `WHERE rownum <= N`: Flink 需要 rownum <= N 才能识别此查询是 Top-N 查询。 N 表示将要保留 N 个最大或最小数据。
- `[AND conditions]`: 可以在 WHERE 子句中添加其他条件，但是这些其他条件和 rownum <= N 需要使用 AND 结合。

!!! note

    Top-N 查询是结果更新的。Flink SQL 会根据 ORDER BY 的字段对输入的数据流进行排序，所以如果前 N 条记录发生了变化，那么变化后的记录将作为回撤/更新记录发送到下游。建议使用一个支持更新的存储作为 Top-N 查询的结果表。此外，如果 Top-N 条记录需要存储在外部存储中，结果表应该与 Top-N 查询的唯一键保持一致。

下面的示例展示了在流式表上指定 Top-N SQL 查询 「实时显示每个分类下销售额最高的五个产品」。

```sql
CREATE TABLE ShopSales (
  product_id   STRING,
  category     STRING,
  product_name STRING,
  sales        BIGINT
) WITH (...);

SELECT *
FROM (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS row_num
  FROM ShopSales)
WHERE row_num <= 5
```

### 无排名输出优化

如上所述，rownum 将作为唯一键的一个字段写入到结果表，这可能会导致大量数据写入到结果表。例如，排名第九（比如 product-1001）的记录更新为 1，排名 1 到 9 的所有记录都会作为更新信息逐条写入到结果表。如果结果表收到太多的数据，它将会成为这个 SQL 任务的瓶颈。

**优化的方法是在 Top-N 查询的外层 SELECT 子句中省略 rownum 字段**。因为通常 Top-N 的数据量不大，消费端就可以快速地排序。下面的示例中就没有 rownum 字段，只需要发送变更数据（product-1001）到下游，这样可以减少结果表很多 IO。

下面的示例展示了用这种方法怎样去优化上面的 Top-N：

```sql
-- omit row_num field from the output
SELECT product_id, category, product_name, sales
FROM (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS row_num
  FROM ShopSales)
WHERE row_num <= 5
```

---

## 窗口 Top-N

窗口 Top-N 是特殊的 Top-N，它返回每个分区键的每个窗口的 N 个最小或最大值。

与普通 Top-N 不同，窗口 Top-N 只在窗口最后返回汇总的 Top-N 数据，不会产生中间结果。窗口 Top-N 会在窗口结束后清除不需要的中间状态。

因此，**窗口 Top-N 适用于用户不需要每条数据都更新 Top-N 结果的场景，相对普通 Top-N 来说性能更好**。通常，窗口 Top-N 直接用于 窗口表值函数上。另外，窗口 Top-N 可以用于基于 _窗口表值函数_ 的操作之上，比如窗口聚合，窗口 Top-N 和窗口关联。

窗口 Top-N 的语法和普通的 Top-N 相同，除此之外，窗口 Top-N 需要 PARTITION BY 子句包含**窗口表值函数**或**窗口聚合**产生的 `window_start` 和 `window_end`。否则优化器无法翻译。

下面展示了窗口 Top-N 的语法：

```sql
SELECT [column_list]
FROM (
   SELECT [column_list],
     ROW_NUMBER() OVER (PARTITION BY window_start, window_end [, col_key1...]
       ORDER BY col1 [asc|desc][, col2 [asc|desc]...]) AS rownum
   FROM table_name) -- relation applied windowing TVF
WHERE rownum <= N [AND conditions]
```

---

**示例一：在窗口聚合后进行窗口 Top-N**

下面的示例展示了在 10 分钟的滚动窗口上计算销售额位列前三的供应商。

```sql
-- tables must have time attribute, e.g. `bidtime` in this table
Flink SQL> desc Bid;
+-------------+------------------------+------+-----+--------+---------------------------------+
|        name |                   type | null | key | extras |                       watermark |
+-------------+------------------------+------+-----+--------+---------------------------------+
|     bidtime | TIMESTAMP(3) *ROWTIME* | true |     |        | `bidtime` - INTERVAL '1' SECOND |
|       price |         DECIMAL(10, 2) | true |     |        |                                 |
|        item |                 STRING | true |     |        |                                 |
| supplier_id |                 STRING | true |     |        |                                 |
+-------------+------------------------+------+-----+--------+---------------------------------+

Flink SQL> SELECT * FROM Bid;
+------------------+-------+------+-------------+
|          bidtime | price | item | supplier_id |
+------------------+-------+------+-------------+
| 2020-04-15 08:05 |  4.00 |    A |   supplier1 |
| 2020-04-15 08:06 |  4.00 |    C |   supplier2 |
| 2020-04-15 08:07 |  2.00 |    G |   supplier1 |
| 2020-04-15 08:08 |  2.00 |    B |   supplier3 |
| 2020-04-15 08:09 |  5.00 |    D |   supplier4 |
| 2020-04-15 08:11 |  2.00 |    B |   supplier3 |
| 2020-04-15 08:13 |  1.00 |    E |   supplier1 |
| 2020-04-15 08:15 |  3.00 |    H |   supplier2 |
| 2020-04-15 08:17 |  6.00 |    F |   supplier5 |
+------------------+-------+------+-------------+

Flink SQL> SELECT *
  FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY window_start, window_end ORDER BY price DESC) as rownum
    FROM (
      SELECT window_start, window_end, supplier_id, SUM(price) as price, COUNT(*) as cnt
      FROM TABLE(
        TUMBLE(TABLE Bid, DESCRIPTOR(bidtime), INTERVAL '10' MINUTES))
      GROUP BY window_start, window_end, supplier_id
    )
  ) WHERE rownum <= 3;
+------------------+------------------+-------------+-------+-----+--------+
|     window_start |       window_end | supplier_id | price | cnt | rownum |
+------------------+------------------+-------------+-------+-----+--------+
| 2020-04-15 08:00 | 2020-04-15 08:10 |   supplier1 |  6.00 |   2 |      1 |
| 2020-04-15 08:00 | 2020-04-15 08:10 |   supplier4 |  5.00 |   1 |      2 |
| 2020-04-15 08:00 | 2020-04-15 08:10 |   supplier2 |  4.00 |   1 |      3 |
| 2020-04-15 08:10 | 2020-04-15 08:20 |   supplier5 |  6.00 |   1 |      1 |
| 2020-04-15 08:10 | 2020-04-15 08:20 |   supplier2 |  3.00 |   1 |      2 |
| 2020-04-15 08:10 | 2020-04-15 08:20 |   supplier3 |  2.00 |   1 |      3 |
+------------------+------------------+-------------+-------+-----+--------+
```

---

**示例二：在窗口表值函数后进行窗口 Top-N**

下面的示例展示了在 10 分钟的滚动窗口上计算价格位列前三的数据。

```sql
Flink SQL> SELECT *
  FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY window_start, window_end ORDER BY price DESC) as rownum
    FROM TABLE(
               TUMBLE(TABLE Bid, DESCRIPTOR(bidtime), INTERVAL '10' MINUTES))
  ) WHERE rownum <= 3;
+------------------+-------+------+-------------+------------------+------------------+--------+
|          bidtime | price | item | supplier_id |     window_start |       window_end | rownum |
+------------------+-------+------+-------------+------------------+------------------+--------+
| 2020-04-15 08:05 |  4.00 |    A |   supplier1 | 2020-04-15 08:00 | 2020-04-15 08:10 |      2 |
| 2020-04-15 08:06 |  4.00 |    C |   supplier2 | 2020-04-15 08:00 | 2020-04-15 08:10 |      3 |
| 2020-04-15 08:09 |  5.00 |    D |   supplier4 | 2020-04-15 08:00 | 2020-04-15 08:10 |      1 |
| 2020-04-15 08:11 |  2.00 |    B |   supplier3 | 2020-04-15 08:10 | 2020-04-15 08:20 |      3 |
| 2020-04-15 08:15 |  3.00 |    H |   supplier2 | 2020-04-15 08:10 | 2020-04-15 08:20 |      2 |
| 2020-04-15 08:17 |  6.00 |    F |   supplier5 | 2020-04-15 08:10 | 2020-04-15 08:20 |      1 |
+------------------+-------+------+-------------+------------------+------------------+--------+
```
