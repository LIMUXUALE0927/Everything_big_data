## 开启 Fetch 抓取（默认已开启）

Fetch 抓取是指，Hive 中对某些情况的查询可以不必使用 MapReduce 计算。 例如：`select * from user_base_info;` 在这种情况下，Hive 可以简单地读取 user_base_info 对应的存储目录下的文件， 然后输出查询结果到控制台。

## 本地模式（默认未开启）

有时 Hive 的输入数据量是非常小的。在这种情况下，为查询触发执行 MR 任务时消耗可能会比实际 job 的执行时间要多的多。对于大多数这种情况，Hive 可以通过**本地模式在单台机器上处理所有的任务**。对于小数据集，执行时间可以明显被缩短。 用户可以通过设置 `hive.exec.mode.local.auto` 的值为 true，来让 Hive 在适当的时候自动启动这个优化。

```sql
--开启本地 mr
set hive.exec.mode.local.auto=true;
--设置 local mr 的最大输入数据量，当输入数据量小于这个值时采用 local mr 的方式， 默认为 134217728，即 128M
set hive.exec.mode.local.auto.inputbytes.max=50000000;
--设置 local mr 的最大输入文件个数，当输入文件个数小于这个值时采用 local mr 的方式， 默认为 4
set hive.exec.mode.local.auto.input.files.max=8;
```

---

## 表的优化

- 空 KEY 过滤

有时 join 超时是因为某些 key 对应的数据太多，而相同 key 对应的数据都会发送到相同的 reducer 上，从而导致内存不够。此时我们应该仔细分析这些异常的 key，很多情况下，这些 key 对应的数据是异常数据，我们需要在 SQL 语句中进行过滤。 此外，在大多数业务中，除了要对 null key 过滤，有时也会对 key 值为 0 的情况 下进行过滤。事实上，就是过滤掉非法的或者异常的、无效的或者无意义的 key 值。

- 空 key 转换

有时虽然某个 key 为空对应的数据很多，但是相应的数据不是异常数据，必须要包含在 join 的结果中，此时我们可以表 a 中 key 为空的字段赋一个随机的值，使得数据随机均匀地分不到不同的 reducer 上。

```sql
--设置 5 个 reduce 个数
set mapreduce.job.reduces = 5;
--JOIN 两张表
select n.* from nullidtable n
left join ori o
on case when n.id is null then concat('hive', rand()) else n.id end = o.id;
```

---

## Join 的优化

Hive 拥有多种 join 算法，包括 Common Join，Map Join，Bucket Map Join，Sort Merge Buckt Map Join 等，下面对每种 join 算法做简要说明：

### Common Join

Common Join 是 Hive 中最稳定的 join 算法，其通过一个 MapReduce Job 完成一个 join 操作。Map 端负责读取 join 操作所需表的数据，并按照关联字段进行分区，通过 Shuffle，将其发送到 Reduce 端，相同 key 的数据在 Reduce 端完成最终的 Join 操作。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310160432024.png)

需要注意的是，sql 语句中的 join 操作和执行计划中的 Common Join 任务并非一对一的关系，一个 sql 语句中的相邻的且关联字段相同的多个 join 操作可以合并为一个 Common Join 任务。

```sql
select
a.val,
b.val,
c.val
from a
join b on (a.key = b.key1)
join c on (c.key = b.key1)
```

上述 sql 语句中两个 join 操作的关联字段均为 b 表的 key1 字段，则该语句中的两个 join 操作可由一个 Common Join 任务实现，也就是可通过一个 Map Reduce 任务实现。

```sql
select
a.val,
b.val,
c.val
from a
join b on (a.key = b.key1)
join c on (c.key = b.key2)
```

上述 sql 语句中的两个 join 操作关联字段各不相同，则该语句的两个 join 操作需要各自通过一个 Common Join 任务实现，也就是通过两个 Map Reduce 任务实现。

### Map Join

Map Join 算法可以通过两个只有 map 阶段的 Job 完成一个 join 操作。其适用场景为大表 join 小表。若某 join 操作满足要求，则第一个 Job 会读取小表数据，将其制作为 hash table，并上传至 **Hadoop 分布式缓存** (本质上是上传至 HDFS)。第二个 Job 会先从分布式缓存中读取小表数据，并缓存在 MapTask 的内存中，然后扫描大表数据，这样在 map 端即可完成关联操作。

Job1（本地任务，不会提交 Yarn）：读取小表数据、制作哈希表、上传至分布式缓存

Job2：从分布式缓存读取哈希表、加载至内存、完成 join

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310160443267.png)

触发 Map Join：

- 在 SQL 语句中增加 hint 提示（过时，不推荐使用）

- Hive 优化器根据表的数据量大小自动触发

hint 提示：将 ta 作为小表缓存

```sql
select /*+ mapjoin(ta) */
ta.id,
tb.id
from table_a ta
join table_b tb
on ta.id=tb.id;
```

自动触发：

Hive 在编译 SQL 语句阶段，起初所有的 Join 操作均采用 Common Join 算法实现。

之后在物理优化阶段，Hive 会根据每个 Common Join 任务所需表的大小判断该 Common Join 任务是否能够转换为 Map Join 任务，若满足要求，便将 Common Join 任务自动转换为 Map Join 任务。判断标准为小表大小与用户设置的阈值（MapTask 内存）比较。

但**有些 Common Join 任务所需的表大小，在 SQL 的编译阶段是未知的**（例如先进行一个分组聚合，之后再和一张表进行 Join 操作），所以这种 Common Join 任务是否能转换成 Map Join 任务在编译阶是无法确定的。

针对这种情况，Hive 会在编译阶段生成一个**条件任务 (Conditional Task)**，其下会包含一个**计划列表**，计划列表中包含**转换后的 Map Join 任务以及原有的 Common Join 任务**。 最终具体采用哪个计划，是在**运行时决定**的。大致思路如下图所示:

假设我们有 A 和 B 两张表，两张表的大小都未知，那么此时 Map Join 有 2 种情况：

- 缓存 A 表，扫描 B 表

- 缓存 B 表，扫描 A 表

因此会生成多种执行计划。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310160539533.png)

**Hive 如何判断执行计划中的 Common Join Task 能否转换为 Map Join Task？**

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310160542023.png)

注意：

- 对于上面的「寻找大表候选人」这一步：**要考虑 SQL 语句中具体使用的是哪种 Join**。例：A left join B，那么大表必须是 A；如果是 A full outer join B，则根本不能做 Map Join。

- 对于上面的「hive.auto.conver.join.noconditionaltask」的 false 分支，考虑这样一种情况：A inner join B inner join C 关联相同的字段，此时有一个 Map Join 计划尝试将 A 作为大表，B 和 C 作为小表缓存，C 的大小未知，B 的大小已知但太大无法缓存（超过小表阈值），则这种 Map Join 计划一定可以排除。即**排除掉一定不可能成功的 Map Join 计划**。

- 对于上面的「hive.auto.conver.join.noconditionaltask」的 true 分支，此时必须要保证大表外的所有表的大小都已知，并且总和小于小表阈值，才能生成**最优**的 Map Join 计划。

- 对于上面的「生成最优 Map Join 计划」的下一步，考虑这样一种情况：A join B join C on 不同字段，理论上需要 2 个 Common Join，先把 A join B 得到临时表 T，再用 T join C。如果 B 和 C 表的大小都非常小，那么可以把 B 和 C 都缓存到 Mapper 内存中，进行两次 Map Join。

### Bucket Map Join

Bucket Map Join 是**对 Map Join 的改进**，其打破了 Map Join 只适用于大表 join 小表的限制，可用于大表 join 大表的场景。

**分桶的原理：对分桶字段进行 hash partition，拆分为若干个文件。**

Bucket Map Join 的核心思想是：若能保证参与 join 的表均为分桶表，且关联字段为分桶字段，且其中一张表的分桶数量是另外一张表分桶数量的整数倍，就能保证参与 join 的两张表的分桶之间具有明确的关联关系，所以就可以在两表的分桶间进行 Map Join 操作了。这样一来，第二个 Job 的 Map 端就无需再缓存小表的全表数据了，而只需缓存其所需的分桶即可。

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310160450333.png)

对于上面的图：

对于 Table B 来说，由于有 2 个桶，因此所有 `hashcode % 2 == 0` 的 record 都会进入 Bucket B-0，所有 `hashcode % 2 == 1` 的 record 都会进入 Bucket B-1。

对于 Table A 来说，由于有 4 个桶，因此所有 `hashcode % 4 == 0` 的 record 都会进入 Bucket A-0。所有 `hashcode % 4 == 2` 的 record 都会进入 Bucket A-2。Bucket A-0 和 Bucket A-2 中的这些 record 就对应着 Bucket B-0 中的 record。同理，Bucket A-1 和 Bucket A-3 中的所有 record 就对应着 Bucket B-1 中的 record。

当然，最简单的情况就是两张表的桶数相同。

之后就和 Map Join 的操作一致：

- Job1：先由本地 Map 任务将相对小一点的表的每个桶各制作一张哈希表，将所有哈希表上传至分布式缓存

- Job2：按照 BucketInputFormat 读取大表数据，一个桶一个切片，每一个 Mapper 只需要处理大表中的一个桶即可。同时，根据桶之间的对应关系，每个 Mapper 只需要从分布式缓存中读取自己需要的小表的桶的哈希表即可。例如，Mapper1 被分到了 Bucket A-0，因此，Mapper1 只需要去分布式缓存读取 Bucket B-0 即可。

### Sort Merge Bucket Map Join

Sort Merge Bucket Map Join(简称 SMB Map Join) 基于 Bucket Map Join。SMB Map Join 要求，参与 join 的表均为分桶表，且需保证分桶内的数据是有序的，且分桶字段、 排序字段和关联字段为相同字段，且其中一张表的分桶数量是另外一张表分桶数量的整数倍。

SMB Map Join 同 Bucket Join 一样，同样是利用两表各分桶之间的关联关系，在分桶之间进行 join 操作，不同的是分桶之间的 join 操作的实现原理。Bucket Map Join，两个分桶之间的 join 实现原理为 **Hash Join 算法**；而 SMB Map Join，两个分桶之间的 join 实现原理为 **Sort Merge Join 算法**。

Hash Join 和 Sort Merge Join 均为关系型数据库中常见的 Join 实现算法。Hash Join 的原理相对简单，就是对参与 join 的一张表构建 hash table，然后扫描另外一张表， 然后进行逐行匹配。**Sort Merge Join 需要在两张按照关联字段排好序的表中进行**，其原理如图所示：

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310160518467.png)

**SMB Map Join VS Bucket Map Join**

- 不需要制作哈希表

- 不需要在内存中缓存哈希表（一个桶），因此对桶的大小没有要求

Hive 中的 SMB Map Join 就是对两个分桶的数据按照上述思路进行 Join 操作。可以看出，SMB Map Join 与 Bucket Map Join 相比，在进行 Join 操作时，Map 端是无需对整个 Bucket 构建 hash table，也无需在 Map 端缓存整个 Bucket 数据的，每个 Mapper 只需按顺序逐个 key 读取两个分桶的数据进行 join 即可。

---

## Group By 优化

Hive 中**未经优化**的分组聚合，是通过一个 MapReduce Job 实现的。Map 端负责读取数据，并按照分组字段分区，通过 Shuffle，将数据发往 Reduce 端，各组数据在 Reduce 端完成最终的聚合运算。

Hive 对分组聚合的优化主要围绕着**减少 Shuffle 数据量**进行，具体做法是 **map-side 聚合**。所谓 map-side 聚合，就是在 map 端维护一个（内存中的） **hash table**，利用其完成部分的聚合，然后将部分聚合的结果，按照分组字段分区，发送至 reduce 端，完成最终的聚合。**map-side 聚合能有效减少 shuffle 的数据量，提高分组聚合运算的效率**。

```sql
--是否在 Map 端进行聚合， 默认为 True
set hive.map.aggr = true;
--在 Map 端进行聚合操作的条目数目
set hive.groupby.mapaggr.checkinterval = 100000;
--有数据倾斜的时候进行负载均衡（默认是 false）
set hive.groupby.skewindata = true;
```

上面的 `set hive.groupby.skewindata = true;` 当选项设定为 true，生成的查询计划会有**两个 MR Job**。第一个 MR Job 中，Map 的输出结果会随机分布到 Reduce 中，每个 Reduce 做部分聚合操作，并输出结果，这样处理的结果是相同的 Group By Key 有可能被分发到不同的 Reduce 中，从而达到负载均衡的目的；第二个 MR Job 再根据预处理的数据结果按照 Group By Key 分布到 Reduce 中（这个过程可以保证相同的 Group By Key 被分布到同一个 Reduce 中），最后完成最终的聚合操作。

---

## count(distinct) 优化

数据量大的情况下，由于 COUNT DISTINCT 操作需要用一个 Reduce Task 来完成，这一个 Reduce 需要处理的数据量太大，就会导致整个 Job 很难完成，一般 COUNT DISTINCT 使用先 GROUP BY 再 COUNT 的方式替换。

```sql
--直接去重
select count(distinct id) from bigtable;
--改写后去重
select count(id) from (select id from bigtable group by id) a;
```

虽然会多用一个 Job 来完成，但在数据量大的情况下，group by 依旧是去重的一个优化手段。如果说需要统计的字段有 Null 值，最后只需要 null 值单独处理后 union 即可。

---

## 数据倾斜问题

数据倾斜问题，通常是指参与计算的数据分布不均，即某个 key 或者某些 key 的数据量远超其他 key，导致在 shuffle 阶段，大量相同 key 的数据被发往同一个 Reducer，进而导致该 Reducer 所需的时间远超其他 Reducer，成为整个任务的瓶颈。

Hive 中的数据倾斜常出现在**分组聚合**和 **Join** 操作的场景中。

### Group By 产生数据倾斜

使用 Hive 对数据做分组聚合的时候某种类型的数据量特别多，而其他类型数据的数据量特别少。

Hive 中未经优化的分组聚合，是通过一个 MapReduce Job 实现的。Map 端负责读取数据，并按照分组字段分区，通过 Shuffle，将数据发往 Reduce 端，各组数据在 Reduce 端完成最终的聚合运算。

如果 group by 分组字段的值分布不均，就可能导致大量相同的 key 进入同一 Reducer， 从而导致数据倾斜问题。

由分组聚合导致的数据倾斜问题，有以下两种解决思路:

- **Map-Side 聚合**

开启 Map-Side 聚合后，数据会现在 Map 端完成部分聚合工作。这样经过 Map 端的初步聚合后，可以缓解发往 Reducer 的数据的倾斜程度。最佳状态下，Map 端聚合能完全屏蔽数据倾斜问题。

- **Skew-GroupBy 优化**

Skew-GroupBy 的原理是启动两个 MR 任务，第一个 MR 按照随机数分区，将数据分散发送到 Reducer，完成部分聚合，第二个 MR 按照分组字段分区，完成最终聚合。

```sql
--启用分组聚合数据倾斜优化
set hive.groupby.skewindata=true;
```

- 根据业务，合理调整分组维度

可以单独将倾斜程度大的字段单拎出来计算，再 union 结果。

---

### count(distinct) 产生数据倾斜

如果数据量非常大，执行如 `select a, count(distinct b) from t group by a;` 类型的 SQL 时，会出现数据倾斜的问题。

解决方法：

- 使用 sum ... group by 代替。

```sql
select a, sum(1) from (select a,b from t group by a,b) group by a;
```

详见上文 count(distinct) 优化

- 在业务逻辑优化效果的不大情况下，有些时候是可以将倾斜的数据单独拿出来处理，最后 union 回去。

---

### Join 产生数据倾斜

未经优化的 Join 操作，默认是使用 common join 算法，也就是通过一个 MapReduce Job 完成计算。Map 端负责读取 join 操作所需表的数据，并按照关联字段进行分区，通过 Shuffle，将其发送到 Reduce 端，相同 key 的数据在 Reduce 端完成最终的 Join 操作。

如果关联字段的值分布不均，就可能导致大量相同的 key 进入同一 Reduce，从而导致数据倾斜问题。

由 Join 导致的数据倾斜问题，有如下三种解决方案：

- **Map Join**

使用 map join 算法，join 操作仅在 map 端就能完成，没有 shuffle 操作，没有 reduce 阶段，自然不会产生 reduce 端的数据倾斜。该方案适用于**大表 join 小表时发生数据倾斜的场景**。

- **Skew Join**

Skew Join 的原理是，为倾斜的大 key 单独启动一个 map join 任务进行计算，其余 key 进行正常的 common join。原理图如下：

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310161600880.png)

注意：在数据倾斜时，如果是**大表 Join 大表**，那么也无法使用 Bucket Map Join，因为分桶（过程为 MR）也是倾斜的。此时需要使用 Skew Join。

```sql
--启用 skew join 优化
set hive.optimize.skewjoin=true;
--触发 skew join 的阈值，若某个 key 的行数超过该参数值，则触发
set hive.skewjoin.key=100000;
```

这种方案对参与 join 的源表大小没有要求，但是对两表中倾斜的 key 的数据量有要求， 要求一张表中的倾斜 key 的数据量比较小 (方便走 map join)。

- **调整 SQL 语句**

若参与 join 的两表均为大表，其中一张表的数据是倾斜的，此时也可通过以下方式对 SQL 语句进行相应的调整。

假设原始 SQL 语句如下：A，B 两表均为大表，且其中一张表的数据是倾斜的。

```sql
select *
from A join B
on A.id=B.id;
```

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310161615260.png)

图中 1001 为倾斜的大 key，可以看到，其被发往了同一个 Reduce 进行处理。

调整 SQL 语句如下：

```sql
select
	*
from(
    select --打散操作
    	concat(id,'_',cast(rand()*2 as int)) id, value
	from A
)ta join(
	select --扩容操作
        concat(id,'_',0) id, value
    from B
    union all
    select
        concat(id,'_',1) id, value
    from B
)tb
on ta.id=tb.id;
```

调整之后的 SQL 语句执行计划如下图所示：

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202310161619269.png)

---

### 空值产生数据倾斜

遇到需要进行 join 的但是关联字段有数据为空。如日志中，常会有信息丢失的问题，比如日志中的 user_id，如果取其中的 user_id 和用户表中 的 user_id 关联，会碰到数据倾斜的问题。数据量大时也会产生数据倾斜，如表一的 id 需要和表二的 id 进行关联。

解决方法：

- 单独把空值拎出来再 union

```sql
select a.* from log a join users b
on a.user_id is not null and a.user_id = b.user_id
union all
select a.* from log a where a.user_id is null;
```

- 给空值分配随机的 key 值

```sql
select * from log a left outer join users b
on case when a.user_id is null then
concat('hive', rand()) else a.user_id end = b.user_id;
```

一般分配随机 key 值得方法更好一些。

---

### 合理设置 Mapper 和 Reducer 的个数
