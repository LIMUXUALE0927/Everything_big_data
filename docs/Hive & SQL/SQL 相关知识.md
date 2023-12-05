## MySQL 语句执行顺序

查询语句的执行顺序:

```sql
(7) SELECT
(8) DISTINCT <select_list>
(1) FROM <left_table>
(2) <join_type> JOIN <right_table>
(3) ON <join_condition>
(4) WHERE <where_condition>
(5) GROUP BY <group_by_list>
(6) HAVING <having_condition>
(9) ORDER BY <order_by_condition>
(10) LIMIT <limit_number>
```

```sql
FROM ..., ... -> ON -> (LEFT/RIGHT JOIN) -> WHERE ->
GROUP BY -> HAVING -> SELECT -> DISTINCT ->
ORDER BY -> LIMIT
```

SELECT 是先执行 FROM 这一步的。在这个阶段，如果是多张表联查，还会经历下面的几个步骤:

1.  首先先通过 CROSS JOIN 求笛卡尔积，相当于得到虚拟表 vt(virtual table)1-1
2.  通过 ON 进行筛选，在虚拟表 vt1-1 的基础上进行筛选，得到虚拟表 vt1-2
3.  添加外部行。如果我们使用的是左连接、右链接或者全连接，就会涉及到外部行，也就是在虚拟表 vt1-2 的基础上增加外部行，得到虚拟表 vt1-3。

当然如果我们操作的是两张以上的表，还会重复上面的步骤，直到所有表都被处理完为止。这个过程得到是我们的原始数据。

当我们拿到了查询数据表的原始数据，也就是最终的虚拟表 vt1 ，就可以在此基础上再进行 WHERE 阶段。在这个阶段中，会根据 vt1 表的结果进行筛选过滤，得到虚拟表 vt2 。

然后进入第三步和第四步，也就是 GROUP BY 和 HAVING 阶段。在这个阶段中，实际上是在虚拟表 vt2 的基础上进行分组和分组过滤，得到中间的虚拟表 vt3 和 vt4 。

当我们完成了条件筛选部分之后，就可以筛选表中提取的字段，也就是进入到 SELECT 和 DISTINCT 阶段。

首先在 SELECT 阶段会提取想要的字段，然后在 DISTINCT 阶段过滤掉重复的行，分别得到中间的虚拟表 vt5-1 和 vt5-2。

当我们提取了想要的字段数据之后，就可以按照指定的字段进行排序，也就是 ORDER BY 阶段，得到虚拟表 vt6 。

最后在 vt6 的基础上，取出指定行的记录，也就是 LIMIT 阶段，得到最终的结果，对应的是虚拟表 vt7 。

---

## Hive SQL 语句执行顺序

标准的 Hive SQL 查询语句与 MySQL 的相同，但是由于两者底层的运行原理不一样，查询语句的底层执行顺序略有不同，我们可以拆分成 map 阶段和 reduce 阶段来看。

Map 阶段：

1. 执行 from 加载：进行表的查找与加载。
2. 执行 where 过滤：进行条件过滤与筛选。
3. 执行 select 查询：进行输出项的筛选。
4. 执行 group by 分组：对数据进行分组。
5. map 端文件合并：map 端本地溢出写文件的合并操作，每个 map 最终形成一个临时文件。然后按列映射到对应的 reduce 阶段。

Reduce 阶段：

1. group by：对 map 端发送过来的数据进行分组并进行计算。
2. select：最后过滤列用于输出结果。
3. limit 排序后进行结果输出到 HDFS 文件。

---

## Hive SQL 语句的执行原理

### Join 的实现原理

```sql
select u.name, o.orderid from order o join user u on o.uid = u.uid;
```

在 map 的输出 value 中为不同表的数据打上 tag 标记，在 reduce 阶段根据 tag 判断数据来源。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310170306655.png)

### Group By 的实现原理

```sql
select rank, isonline, count(*) from city group by rank, isonline;
```

将 Group By 的字段组合为 map 的输出 key 值，利用 MapReduce 的排序，在 reduce 阶段保存 LastKey 区分不同的 key。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310170358198.png)

### Distinct 的实现原理

```sql
select dealid, count(distinct uid) num from order group by dealid;
```

当只有一个 distinct 字段时，如果不考虑 Map 阶段的 Hash GroupBy，只需要将 Group By 字段和 Distinct 字段组合为 map 输出 key，利用 mapreduce 的排序，同时将 Group By 字段作为 reduce 的 key，在 reduce 阶段保存 LastKey 即可完成去重。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310170405313.png)
