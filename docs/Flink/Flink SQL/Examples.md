# Examples

## 统计每种商品的历史累计销售额

```sql
create table source_table (
    productId string,
    income bigint
) with (
    'connector' = 'datagen',
    'rows-per-second' = '1',
    'fields.income.min' = '1',
    'fields.income.max' = '5'
)

create table sink_table (
    productId string,
    countResult bigint,
    incomeResult bigint,
    avgIncomeResult bigint,
    maxIncomeResult bigint,
    minIncomeResult bigint
) with (
    'connector' = 'print'
)

insert into sink_table
select
    productId,
    count(1) as countResult,
    sum(income) as incomeResult,
    avg(income) as avgIncomeResult,
    max(income) as maxIncomeResult,
    min(income) as minIncomeResult
from source_table
group by productId
```

---

## 统计每种商品每 1min 的累计销售额

```sql
create table source_table (
    pid bigint,
    income bigint,
    time bigint,
    -- 用于定义数据的事件时间戳
    row_timeas to_timestamp_ltz(time, 3),
    -- 用于指定数据的 watermark，最大乱序时间为 5 秒
    watermark for row_time as row_time - interval '5' second
) with (
    ...
);

create table sink_table (
    pid bigint,
    all bigint,
    minutes string
) with (
    ...
);

insert into sink_table
select
    pid,
    sum(income) as all,
    TUMBLE_START(row_time, INTERVAL '1' MINUTE) as minutes
from source_table
group by pid,
    TUMBLE(row_time, INTERVAL '1' MINUTE)
```

---

## 统计每种商品每 1min 的累计销售额（滚动窗口表值函数）

```sql
create table source_table (
    productId string,
    income bigint,
    userId bigint,
    eventTime bigint,
    -- 毫秒级别的 Unix 时间戳
    -- 定义事件时间字段，在本地 IDE 测试时，可以使用下面的语句生成合理的 rowTime
    -- row_time as cast(current_timestamp as timestamp_ltz(3)),
    row_time as to_timestamp_ltz(eventTime, 3),
    -- 用于指定数据的 watermark，最大乱序时间为 5 秒
    watermark for row_time as row_time - interval '5' second,
    -- 定义处理时间字段
    proctime as proctime()
) with ('connector' = 'datagen');

with tmp as (
    -- 第一层按照 userId 取模，实现分桶计算，避免数据倾斜
    select
        productId,
        sum(income) as bucketAll,
        window_time as rtime
    from
        table(
            TUMBLE(
                TABLE source_table,
                DESCRIPTOR(row_time),
                INTERVAL '1' MINUTE
            )
        )
    group by
        window_start,
        window_end,
        window_time,
        productId,
        mod(userId, 100)
)
-- 第二层用于合桶计算
select
    productId,
    sum(bucketAll) as allIncome,
    window_start
from
    table (
        TUMBLE(
            TABLE tmp,
            DESCRIPTOR(rtime),
            INTERVAL '1' MINUTE
        )
    )
group by
    window_start,
    window_end,
    window_time,
    productId
```
