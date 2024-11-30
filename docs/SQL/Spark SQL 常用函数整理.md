# Spark SQL 常用函数整理

## Date and Timestamp Functions

### 当前时间类

| Function                         | Description                                                                                                                                 |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| current_date()                   | Returns the current date at the start of query evaluation. All calls of current_date within the same query return the same value.           |
| current_timestamp()              | Returns the current timestamp at the start of query evaluation. All calls of current_timestamp within the same query return the same value. |
| unix_timestamp([timeExp[, fmt]]) | Returns the UNIX timestamp of current or specified time.                                                                                    |
| now()                            | Returns the current timestamp at the start of query evaluation.                                                                             |

```sql
SELECT current_date();
+--------------+
|current_date()|
+--------------+
|2024-11-30    |
+--------------+

SELECT current_timestamp();
+--------------------------+
|current_timestamp()       |
+--------------------------+
|2024-11-30 16:50:03.427195|
+--------------------------+

SELECT unix_timestamp();
+--------------------------------------------------------+
|unix_timestamp(current_timestamp(), yyyy-MM-dd HH:mm:ss)|
+--------------------------------------------------------+
|1732959282                                              |
+--------------------------------------------------------+

SELECT now();
+--------------------------+
|now()                     |
+--------------------------+
|2024-11-30 17:35:00.331006|
+--------------------------+
```

### 时间提取类

| Function                | Description                                                                                 |
| ----------------------- | ------------------------------------------------------------------------------------------- |
| year(date)              | Returns the year component of the date/timestamp.                                           |
| quarter(date)           | Returns the quarter of the year for date, in the range 1 to 4.                              |
| month(date)             | Returns the month component of the date/timestamp.                                          |
| day(date)               | Returns the day of month of the date/timestamp.                                             |
| hour(timestamp)         | Returns the hour component of the string/timestamp.                                         |
| minute(timestamp)       | Returns the minute component of the string/timestamp.                                       |
| second(timestamp)       | Returns the second component of the string/timestamp.                                       |
| weekofyear(date)        | Returns the week of the year of the given date.                                             |
| dayofyear(date)         | Returns the day of year of the date/timestamp.                                              |
| dayofmonth(date)        | Returns the day of month of the date/timestamp.                                             |
| dayofweek(date)         | Returns the day of the week for date/timestamp (1 = Sunday, 2 = Monday, ..., 7 = Saturday). |
| ==date_trunc(fmt, ts)== | Returns timestamp `ts` truncated to the unit specified by the format model `fmt`.           |

```sql
select date_trunc('YEAR', '2024-11-30');
+----------------------------+
|date_trunc(YEAR, 2024-11-30)|
+----------------------------+
|2024-01-01 00:00:00         |
+----------------------------+

+-----------------------------+
|date_trunc(MONTH, 2024-11-30)|
+-----------------------------+
|2024-11-01 00:00:00          |
+-----------------------------+

+---------------------------+
|date_trunc(DAY, 2024-11-30)|
+---------------------------+
|2024-11-30 00:00:00        |
+---------------------------+

+--------------------------------------------+
|date_trunc(HOUR, 2024-11-30 17:07:58.078219)|
+--------------------------------------------+
|2024-11-30 17:00:00                         |
+--------------------------------------------+

+----------------------------------------------+
|date_trunc(MINUTE, 2024-11-30 17:07:58.078219)|
+----------------------------------------------+
|2024-11-30 17:07:00                           |
+----------------------------------------------+

+----------------------------------------------+
|date_trunc(SECOND, 2024-11-30 17:07:58.078219)|
+----------------------------------------------+
|2024-11-30 17:07:58                           |
+----------------------------------------------+
```

### 时间计算类

| Function                                                     | Description                                                                                                                                     |
| ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| date_add(start_date, num_days)                               | Returns the date that is `num_days` after `start_date`.                                                                                         |
| date_sub(start_date, num_days)                               | Returns the date that is `num_days` before `start_date`.                                                                                        |
| datediff(endDate, startDate)                                 | Returns the number of days from `startDate` to `endDate`.                                                                                       |
| ==timestampdiff(time_unit, start_timestamp, end_timestamp)== | 返回时间差值 in time_unit. `time_unit`：指定时间单位，可以是以下值之一：`SECOND`, `MINUTE`, `HOUR`, `DAY`, `WEEK`, `MONTH`, `QUARTER`, `YEAR`。 |
| add_months(start_date, num_months)                           | Returns the date that is `num_months` after `start_date`.                                                                                       |
| months_between(timestamp1, timestamp2[, roundOff])           | Returns the number of months between two timestamps.                                                                                            |

```sql
SELECT timestampdiff(SECOND, '2024-03-01 10:00:00', '2024-03-01 10:05:30');
+---------------------------------------------------------------+
|timestampdiff(SECOND, 2024-03-01 10:00:00, 2024-03-01 10:05:30)|
+---------------------------------------------------------------+
|330                                                            |
+---------------------------------------------------------------+

SELECT timestampdiff(MINUTE, '2024-03-01 10:00:00', '2024-03-01 10:05:30');
+---------------------------------------------------------------+
|timestampdiff(MINUTE, 2024-03-01 10:00:00, 2024-03-01 10:05:30)|
+---------------------------------------------------------------+
|5                                                              |
+---------------------------------------------------------------+

SELECT timestampdiff(HOUR, '2024-03-01 10:00:00', '2024-03-01 10:05:30');
+-------------------------------------------------------------+
|timestampdiff(HOUR, 2024-03-01 10:00:00, 2024-03-01 10:05:30)|
+-------------------------------------------------------------+
|0                                                            |
+-------------------------------------------------------------+

SELECT timestampdiff(WEEK, '2024-03-01', '2024-03-11');
+-------------------------------------------+
|timestampdiff(WEEK, 2024-03-01, 2024-03-11)|
+-------------------------------------------+
|1                                          |
+-------------------------------------------+

SELECT timestampdiff(MONTH, '2024-03-01', '2024-04-11');
+--------------------------------------------+
|timestampdiff(MONTH, 2024-03-01, 2024-04-11)|
+--------------------------------------------+
|1                                           |
+--------------------------------------------+

SELECT months_between('1997-02-28 10:30:00', '1996-10-30');
+-----------------------------------------------------+
|months_between(1997-02-28 10:30:00, 1996-10-30, true)|
+-----------------------------------------------------+
|3.94959677                                           |
+-----------------------------------------------------+
```

### 时间格式化类

| Function                           | Description                                                                     |
| ---------------------------------- | ------------------------------------------------------------------------------- |
| date_format(timestamp, fmt)        | Converts `timestamp` to a value of string in `fmt`.                             |
| to_date(date_str[, fmt])           | Parses the `date_str` expression with the `fmt` expression to a date.           |
| from_unixtime(unix_time[, fmt])    | Returns `unix_time` in the specified `fmt`.                                     |
| to_timestamp(timestamp_str[, fmt]) | Parses the `timestamp_str` expression with the `fmt` expression to a timestamp. |
| to_unix_timestamp(timeExp[, fmt])  | Returns the UNIX timestamp of the given time.                                   |
| unix_timestamp([timeExp[, fmt]])   | Returns the UNIX timestamp of current or specified time.                        |

```sql
SELECT from_unixtime(1732960246, 'yyyy-MM-dd HH:mm:ss');
+----------------------------------------------+
|from_unixtime(1732960246, yyyy-MM-dd HH:mm:ss)|
+----------------------------------------------+
|2024-11-30 17:50:46                           |
+----------------------------------------------+

SELECT to_timestamp('2016-12-31 00:12:00');
+---------------------------------+
|to_timestamp(2016-12-31 00:12:00)|
+---------------------------------+
|2016-12-31 00:12:00              |
+---------------------------------+

SELECT to_timestamp('2016-12-31', 'yyyy-MM-dd');
+------------------------------------+
|to_timestamp(2016-12-31, yyyy-MM-dd)|
+------------------------------------+
|2016-12-31 00:00:00                 |
+------------------------------------+

SELECT to_unix_timestamp('2016-04-08', 'yyyy-MM-dd');
+-----------------------------------------+
|to_unix_timestamp(2016-04-08, yyyy-MM-dd)|
+-----------------------------------------+
|1460044800                               |
+-----------------------------------------+

SELECT unix_timestamp();
+--------------------------------------------------------+
|unix_timestamp(current_timestamp(), yyyy-MM-dd HH:mm:ss)|
+--------------------------------------------------------+
|1732960597                                              |
+--------------------------------------------------------+

SELECT unix_timestamp('2016-04-08', 'yyyy-MM-dd');
+--------------------------------------+
|unix_timestamp(2016-04-08, yyyy-MM-dd)|
+--------------------------------------+
|1460044800                            |
+--------------------------------------+
```

**时间转换总结：**

![](https://raw.githubusercontent.com/LIMUXUALE0927/image/main/img/202411301816857.png)

```sql
SELECT to_timestamp('2024-11-30');
+------------------------+
|to_timestamp(2024-11-30)|
+------------------------+
|2024-11-30 00:00:00     |
+------------------------+

SELECT to_date('2024-11-30 18:03:43.358491');
+-----------------------------------+
|to_date(2024-11-30 18:03:43.358491)|
+-----------------------------------+
|2024-11-30                         |
+-----------------------------------+

SELECT unix_timestamp('2024-11-30 18:03:43.358491', 'yyyy-MM-dd HH:mm:ss.SSSSSS');
+----------------------------------------------------------------------+
|unix_timestamp(2024-11-30 18:03:43.358491, yyyy-MM-dd HH:mm:ss.SSSSSS)|
+----------------------------------------------------------------------+
|1732961023                                                            |
+----------------------------------------------------------------------+

SELECT from_unixtime(1732961023);
+----------------------------------------------+
|from_unixtime(1732961023, yyyy-MM-dd HH:mm:ss)|
+----------------------------------------------+
|2024-11-30 18:03:43                           |
+----------------------------------------------+
```

### 日期维表常用

| Function                                                     | Description                                                                    |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------ |
| make_date(year, month, day)                                  | Creates a date from the given year, month, and day.                            |
| make_timestamp(year, month, day, hour, min, sec[, timezone]) | Creates a timestamp from the given year, month, day, hour, minute, and second. |
| next_day(start_date, day_of_week)                            | 返回 `start_date` 之后的第一个 `day_of_week`.                                  |
| last_day(date)                                               | Returns the last day of the month which the date belongs to.                   |
| sequence(start, end[, step])                                 | Generates a sequence of integers from `start` to `end` with a step of `step`.  |

```sql
SELECT make_date(2013, 7, 15);
+----------------------+
|make_date(2013, 7, 15)|
+----------------------+
|2013-07-15            |
+----------------------+

SELECT make_timestamp(2014, 12, 28, 6, 30, 45.887);
+-------------------------------------------+
|make_timestamp(2014, 12, 28, 6, 30, 45.887)|
+-------------------------------------------+
|2014-12-28 06:30:45.887                    |
+-------------------------------------------+

SELECT next_day('2024-11-30', 'Monday');
+----------------------------+
|next_day(2024-11-30, Monday)|
+----------------------------+
|2024-12-02                  |
+----------------------------+

SELECT last_day('2024-11-01');
+--------------------+
|last_day(2024-11-01)|
+--------------------+
|2024-11-30          |
+--------------------+

SELECT sequence(to_date('2024-11-01'), to_date('2024-11-05'), interval 1 day);
+--------------------------------------------------------------------+
|sequence(to_date(2024-11-01), to_date(2024-11-05), INTERVAL '1' DAY)|
+--------------------------------------------------------------------+
|[2024-11-01, 2024-11-02, 2024-11-03, 2024-11-04, 2024-11-05]        |
+--------------------------------------------------------------------+

SELECT sequence(to_timestamp('2024-11-01 00:00:00'), to_timestamp('2024-11-01 03:00:00'), interval 1 hour);
+--------------------------------------------------------------------------------------------------+
|sequence(to_timestamp(2024-11-01 00:00:00), to_timestamp(2024-11-01 03:00:00), INTERVAL '01' HOUR)|
+--------------------------------------------------------------------------------------------------+
|[2024-11-01 00:00:00, 2024-11-01 01:00:00, 2024-11-01 02:00:00, 2024-11-01 03:00:00]              |
+--------------------------------------------------------------------------------------------------+
```

```sql
-- 本年第一天
SELECT trunc(current_date(), 'YEAR');
-- 本年最后一天
SELECT last_day(trunc(current_date(), 'YEAR'));

-- 本季度第一天
SELECT trunc(current_date(), 'QUARTER');
-- 本季度最后一天
SELECT last_day(trunc(current_date(), 'QUARTER'));

-- 本月第一天
SELECT trunc(current_date(), 'MONTH');
-- 本月最后一天
SELECT last_day(trunc(current_date(), 'MONTH'));

-- 本周一
SELECT date_sub(next_day(current_date(), 'Monday'), 7);
-- 本周日
SELECT date_sub(next_day(current_date(), 'Monday'), 1);

-- 上周一
SELECT date_sub(next_day(current_date(), 'Monday'), 14);
-- 上周日
SELECT date_sub(next_day(current_date(), 'Monday'), 8);
```
