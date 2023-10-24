## UDF

一对一的输入输出，只能输入一条记录当中的数据，同时返回一条处理结果。属于最常见的自定义函数。Hive 的 UDF 有两种实现方式或者实现的 API，一种是 UDF 比较简单，一种是 GenericUDF 比较复杂。

如果所操作的数据类型都是基础数据类型，如（Hadoop&Hive 基本 writable 类型，如 Text, IntWritable, LongWriable, DoubleWritable 等）。那么简单的 `org.apache.hadoop.hive.ql.exec.UDF` 就可以做到。 具体地，需要继承 `org.apache.hadoop.hive.ql.UDF`，并实现 evaluate 方法。

如果所操作的数据类型是内嵌数据结构，如 Map，List 和 Set，那么要采用 `org.apache.hadoop.hive.ql.udf.generic.GenericUDF`。 具体地，需要继承 `org.apache.hadoop.hive.ql.udf.generic.GenericUDF`，并实现三个方法：

1. initialize：只调用一次，在任何 evaluate() 调用之前可以接收到一个可以表示函数输入参数类型的 ObjectInspectors 数组。initialize 用来验证该函数是否接收正确的参数类型和参数个数，最后提供最后结果对应的数据类型。
2. evaluate：真正的逻辑，读取输入数据，处理数据，返回结果。
3. getDisplayString：返回描述该方法的字符串，没有太多作用。

---

## UDAF

实现方式：

1. 自定义一个 Java 类，继承 UDAF 类
2. 内部定义一个静态类，实现 UDAFEvaluator 接口
3. 实现 init, iterate, terminatePartial, merge, terminate，共 5 个方法

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310200232537.png)

---

## NTILE

NTILE(n)，用于将分组数据按照顺序切分成 n 片，返回当前切片值。如果切片不均匀，默认增加第一个切片的分布。

```sql
SELECT *
FROM geeks_demo;
```

| ID  |
| --- |
| 1   |
| 2   |
| 3   |
| 4   |
| 5   |
| 6   |
| 7   |
| 8   |
| 9   |
| 10  |

```sql
SELECT ID,
NTILE (3) OVER (
ORDER BY ID
) Group_number
FROM geeks_demo;
```

| ID  | Group_number |
| --- | ------------ |
| 1   | 1            |
| 2   | 1            |
| 3   | 1            |
| 4   | 1            |
| 5   | 2            |
| 6   | 2            |
| 7   | 2            |
| 8   | 3            |
| 9   | 3            |
| 10  | 3            |

```sql
SELECT
	category_name,
	month,
	FORMAT(net_sales,'C','en-US') net_sales,
	NTILE(4) OVER(
		PARTITION BY category_name
		ORDER BY net_sales DESC
	) net_sales_group
FROM
	sales.vw_netsales_2017;
```

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310231654384.png)

---

## LAG

LAG(col, n, DEFAULT) 用于统计窗口内往上第 n 行值。

第一个参数为列名，第二个参数为往上第 n 行（可选，默认为 1），第三个参数 为默认值（当往上第 n 行为 NULL 时候，取默认值，如不指定，则为 NULL）。

---

## 日期相关函数

### DATE_SUB() 和 SUBDATE()

---

## CONCAT_WS()

---

## 多维组合查询

测试数据：

```
2021-03,2021-03-10,userid1
2021-03,2021-03-10,userid5
2021-03,2021-03-12,userid7
2021-04,2021-04-12,userid3
2021-04,2021-04-13,userid2
2021-04,2021-04-13,userid4
2021-04,2021-04-16,userid4
2021-03,2021-03-10,userid2
2021-03,2021-03-10,userid3
2021-04,2021-04-12,userid5
2021-04,2021-04-13,userid6
2021-04,2021-04-15,userid3
2021-04,2021-04-15,userid2
2021-04,2021-04-16,userid1
```

### GROUPING SETS

[Enhanced Aggregation, Cube, Grouping and Rollup - Apache Hive](https://cwiki.apache.org/confluence/display/Hive/Enhanced+Aggregation%2C+Cube%2C+Grouping+and+Rollup)

:star: [Hive SQL - Analytics with GROUP BY and GROUPING SETS, Cubes, Rollups](https://kontext.tech/article/1101/hive-sql-analytics-with-group-by-and-grouping-sets-cubes-rollups)

因为 GROUP BY 的维度是单一的，就是它只能计算某个维度的信息，而不能同时计算多个维度，在一个 GROUPING SETS 查询中，根据不同的单维度组合进行聚合，等价于将不同维度的 GROUP BY 结果进行 UNION ALL 操作。 GROUPING SETS 就是一种将多个 GROUP BY 逻辑 UNION 写在一个 HIVE SQL 语句中的便利写法。GROUPING SETS 会把在单个 GROUP BY 逻辑中没有参与 GROUP BY 的那一列置为 NULL 值，这样聚合出来的结果，未被 GROUP BY 的列将显示为 NULL。

```sql
SELECT month, day, COUNT(DISTINCT userid) AS uv, GROUPING__ID
FROM user_visit
GROUP BY month, day
GROUPING SETS (month, day)
ORDER BY GROUPING__ID;
```

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310241420065.png)

```sql
SELECT month, day, COUNT(DISTINCT userid) AS uv, GROUPING__ID
FROM user_visit
GROUP BY month, day
GROUPING SETS (month, day, (month, day))
ORDER BY GROUPING__ID;
```

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310241422328.png)

---

### CUBE

Cube 可以实现 hive 多个任意维度的组合查询，`cube(a,b,c)` 首先会对 `(a,b,c)` 进行 group by，然后依次是 `(a,b),(a,c),(a),(b,c),(b),(c)`，最后在对全表进行 group by，他会统计所选列中值的所有组合的聚合。

Cubes can be used to calculate subtotal of all possible combinations of the set of column in GROUP BY. For instance, `GROUP BY acct, txn_date WITH CUBE` is equivalent to `GROUP BY acct, txn_date GROUPING SETS ( (acct, txn_date), acct, txn_date, ( ))`.

```sql
SELECT month, day, COUNT(DISTINCT userid) AS uv, GROUPING__ID
FROM user_visit
GROUP BY month, day
WITH CUBE
ORDER BY GROUPING__ID;
```

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310241605856.png)

通常在企业中，我们会将组合后的结果进行解析，然后再进行使用。解析脚本如下：

```sql
select case when id & 1 = 1 then a.month else 'all' end                 as month,
       case when id & 2 = 2 then a.day else 'all' end                   as day,
       concat(if(id & 1 = 1, 'month-', ''), if(id & 2 = 2, 'day-', '')) as flag,
       uv
from (select month, day, count(distinct userid) as uv, cast(grouping__id as int) as id
      from user_visit
      group by month, day
      with cube) as a;
```

---

### ROLLUP

ROLLUP clause is used with GROUP BY to compute the aggregations at the hierarchy levels of a dimension. For example, `GROUP BY a, b, c with ROLLUP` assumes that the hierarchy is "a" drilling down to "b" drilling down to "c". `GROUP BY a, b, c, WITH ROLLUP` is equivalent to `GROUP BY a, b, c GROUPING SETS ( (a, b, c), (a, b), (a), ( ))`.

Rollup 可以实现从右到左的递减多级的统计，显示统计某一层次结构的聚合。它是 CUBE 的子集，以最左侧的维度为主，从该维度进行层级聚合。

比如以 month 维度进行层级聚合，可以实现这样的上卷过程：月天的 UV -> 月的 UV -> 总 UV

从细粒度到粗粒度进行统计。

```sql
SELECT month, day, COUNT(DISTINCT userid) AS uv, GROUPING__ID
FROM user_visit
GROUP BY month, day
WITH ROLLUP
ORDER BY GROUPING__ID;
```

等价于

```sql
SELECT month, day, COUNT(DISTINCT userid) AS uv, 3 AS GROUPING__ID
FROM user_visit
GROUP BY month, day
UNION ALL
SELECT month, NULL, COUNT(DISTINCT userid) AS uv, 1 AS GROUPING__ID
FROM user_visit
GROUP BY month
UNION ALL
SELECT NULL, NULL, COUNT(DISTINCT userid) AS uv, 0 AS GROUPING__ID
FROM user_visit
```

---

### `GROUPING__ID`

这个函数返回一个位向量，该位向量对应于每一列是否存在。对于每一列，如果结果集中的某一行已经聚合了该列，则结果集中的某一行的值为“1”，否则该值为“0”。这可以用于在数据中有空值时进行区分。

`grouping__id`是为了区分每条输出结果是属于哪一个 group by 的数据。它是**根据 group by 后面声明的顺序**字段是否存在于当前 group by 中的一个二进制位组合数据。

- `grouping__id` 为 0 的是 group by 中所有列都被选中了，二进制 00，所以标识为 0
- `grouping__id` 为 1 的是 group by 中只有一列被选中了，二进制 01，所以标识为 1
- `grouping__id` 为 3 的是 group by 中没有一列被选中，二进制 11，所以标识为 3

---

### 总结

`GROUPING SETS`、`WITH CUBE`、`WITH ROLLUP` 这三种多维组合方式目的都是为了在不使用 UNION 的情况下实现多种 group by 条件，能够大量降低 SQL 语句的长度。

- `GROUPING SETS`：手动指定想要的 group by 条件组合
- `WITH CUBE`：自动实现所有可能的 group by 条件组合（2^k 个组合）
- `WITH ROLLUP`：实现根据 group by 后面的字段不断删去最右边的字段的 group by 条件组合，从细粒度到粗粒度（k+1 个组合）

---

## 连续登录问题

### 1454. 活跃用户

[1454. 活跃用户](https://leetcode.cn/problems/active-users/)

[题解](https://leetcode.cn/problems/active-users/solutions/1311505/4chong-fang-fa-xiang-jie-huo-yue-yong-hu-s8js/)

```sql
select distinct id, name
from (select id,
             subdate(login_date, rn) first_login_date
      from (select distinct id,
                            login_date,
                            dense_rank() over (partition by id order by login_date) as rn
            from Logins) t1
      group by id, first_login_date
      having count(*) >= 5) t2
left join accounts using (id);
```

执行时间：834 ms

```sql
select distinct ac.id, ac.name
from (select a.id, a.login_date
      from Logins a
      join Logins b
      on a.id = b.id
      and b.login_date between a.login_date and date_add(a.login_date, interval 4 day)
      group by a.id, a.login_date
      having count(distinct b.login_date) >= 5) t
inner join accounts ac on t.id = ac.id
order by ac.id;
```

执行时间：994 ms

---

### 统计最长连续活跃天数

数据和上题相同，统计每个用户连续登录的最长天数。

思路：先借助窗口函数生成一列连续递增的数字；将日期与数字列相减，如果日期是连续的，那差值就是一样的；统计差值列相同取值的最大个数就是最长连续交易天数。

```sql
select id, max(times) as max_times
from (select t2.id,
             t2.date_diff,
             count(1) as times
      from (select id,
                   t1.login_date,
                   subdate(login_date, rn) date_diff
            from (select id,
                         login_date,
                         dense_rank() over (partition by id order by login_date) as rn
                  from Logins) t1) t2
      group by t2.id, t2.date_diff) t3
group by id;
```

---

## 行转列（聚合数据）

数据：

| user_id | order_id |
| ------- | -------- |
| 104399  | 1715131  |
| 104399  | 2105395  |
| 104399  | 1758844  |
| 104399  | 981085   |
| 104399  | 2444143  |
| 104399  | 1458638  |
| 104399  | 968412   |
| 104400  | 1609001  |
| 104400  | 2986088  |
| 104400  | 1795054  |

查看每个用户的下单列表。

```sql
select user_id,
       concat_ws(',', collect_list(order_id))
from user_order
group by user_id;
```

`collect_list()` 和 `collect_set()` 用于分组时将一列合并为一个字符串数组。

`concat_ws(seperator, list)` 用于将 list 用 seperator 间隔，合并为一个字符串。

结果：

| user_id | orders                                                |
| ------- | ----------------------------------------------------- |
| 104399  | 1715131,2105395,1758844,981085,2444143,1458638,968412 |
| 104400  | 1609001,2986088,1795054                               |

---

## 列转行（lateral view）

数据：

| user_id | orders                                                |
| ------- | ----------------------------------------------------- |
| 104399  | 1715131,2105395,1758844,981085,2444143,1458638,968412 |
| 104400  | 1609001,2986088,1795054                               |

要查看用户与订单的映射关系。

```sql
select user_id, order_id
from data_source
lateral view explode(split(orders, ',')) t as order_id;
```

结果：

| user_id | order_id |
| ------- | -------- |
| 104399  | 1715131  |
| 104399  | 2105395  |
| 104399  | 1758844  |
| 104399  | 981085   |
| 104399  | 2444143  |
| 104399  | 1458638  |
| 104399  | 968412   |
| 104400  | 1609001  |
| 104400  | 2986088  |
| 104400  | 1795054  |

---

## TOPK 问题

[184. 部门工资最高的员工](https://leetcode.cn/problems/department-highest-salary/)

```sql
select d.name as Department, t2.name as Employee, t2.salary as Salary
from (
    select *
    from (
        select *,
        dense_rank() over(partition by departmentId order by salary desc) rk
        from employee e
    ) t1
    where t1.rk = 1
) t2
inner join department d
on t2.departmentId = d.id
```

---

请统计每个店铺访问次数 top3 的访客信息.输出店铺名称、访客 id、访问次数。

Visit 表：

| user_id | shop   | visit_date |
| ------- | ------ | ---------- |
| 1       | Shop A | 2023-10-01 |
| 2       | Shop B | 2023-10-01 |
| 3       | Shop A | 2023-10-02 |
| 1       | Shop C | 2023-10-02 |
| 2       | Shop A | 2023-10-03 |
| 3       | Shop B | 2023-10-03 |
| 1       | Shop B | 2023-10-03 |
| 2       | Shop C | 2023-10-04 |
| 3       | Shop C | 2023-10-04 |
| 1       | Shop A | 2023-10-04 |
| 2       | Shop B | 2023-10-05 |
| 3       | Shop A | 2023-10-05 |
| 1       | Shop C | 2023-10-05 |

```sql
select shop, user_id, cnt
from (select *,
             dense_rank() over (partition by shop order by cnt desc) rk
      from (select user_id,
                   shop,
                   count(visit_date) as cnt
            from Visit
            group by shop, user_id) t1) t2
where rk <= 3;
```
