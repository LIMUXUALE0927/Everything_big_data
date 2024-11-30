# Hive/Spark SQL 相关知识

## Hive SQL 语句的执行原理

### Join 的实现原理

```sql
select u.name, o.orderid
from order o
join user u
on o.uid = u.uid;
```

在 map 的输出 value 中为不同表的数据打上 tag 标记，在 reduce 阶段根据 tag 判断数据来源。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310170306655.png)

### Group By 的实现原理

```sql
select rank, isonline, count(*)
from city
group by rank, isonline;
```

将 Group By 的字段组合为 map 的输出 key 值，利用 MapReduce 的排序，在 reduce 阶段保存 LastKey 区分不同的 key。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310170358198.png)

### Distinct 的实现原理

```sql
select dealid, count(distinct uid) as num
from order
group by dealid;
```

当只有一个 distinct 字段时，如果不考虑 Map 阶段的 Hash GroupBy，只需要将 Group By 字段和 Distinct 字段组合为 map 输出 key，利用 mapreduce 的排序，同时将 Group By 字段作为 reduce 的 key，在 reduce 阶段保存 LastKey 即可完成去重。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310170405313.png)

---

## Spark Distinct 的实现原理

### with one count distinct

[一文搞懂 with one count distinct 执行原理](https://mp.weixin.qq.com/s/Fu5u2M86OyhTR7Zs8Er7AA)

**Aggregate 函数的几种 mode：**

- **Partial**：局部数据的聚合。会根据读入的原始数据更新对应的聚合缓冲区，当处理完所有的输入数据后，返回的是局部聚合的结果
- **PartialMerge**：主要是对 Partial 返回的聚合缓冲区（局部聚合结果）进行合并，但此时仍不是最终结果，还要经过 Final 才是最终结果(count distinct 类型)
- **Final**：起到的作用是将聚合缓冲区的数据进行合并，然后返回最终的结果
- **Complete**：不进行局部聚合计算，应用在不支持 Partial 模式的聚合函数上（比如求百分位 percentile_approx）

非 distinct 类的聚合函数的路线：Partial --> Final

distinct 类的聚合函数的路线：Partial --> PartialMerge --> Partial --> Final

---

### more than one count distinct

[more than one count distinct](https://mp.weixin.qq.com/s/yGzhZ2w0V8KeyV4XQYnTeA)

---

## 递归 SQL

[一篇学会 SQL 中的递归的用法](https://www.51cto.com/article/677598.html)
