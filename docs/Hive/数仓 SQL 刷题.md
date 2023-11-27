# 数仓 SQL 刷题

## 经典例题

### TOP K 问题

!!! question

    查询 2019 年 10 月的数据，统计每个服务器，登录次数 top10 的玩家 id

表：`ods_game_dev.ods_user_login`

```mysql
with cte as (select server_id,
                    user_id,
                    count(user_id) as cnt
             from ods_user_login
             where part_date >= '2019-10-01'
               and part_date < '2019-11-01'
               and op_type = '1'
             group by server_id,
                      user_id)
select *
from (select *,
             row_number() over (partition by server_id order by cnt desc) as rn
      from cte) t
where t.rn <= 10;
```

---

### 连续登录问题

!!! question

    求连续登录三天的用户列表

表：`ods_game_dev.ods_user_login`

```mysql
select    t2.user_id,
          date_sub(t2.login_date, t2.rn) as date_diff,
          count(1)                       as cnt
from      (
          select    t1.user_id,
                    t1.login_date,
                    row_number() over (partition by t1.user_id order by t1.login_date) as rn
          from      (
                    select    user_id,
                              min(part_date) login_date
                    from      ods_user_login
                    where     op_type = '1'
                    group by  user_id
                    ) t1
          ) t2
group by  t2.user_id,
          date_sub(t2.login_date, t2.rn)
having    count(1) = 3;
```

---

:fire: 如果我们允许间隔一天还算连续登录，该如何实现？

[SQL 求连续登陆天数(允许间隔)\_hive 连续登录天数](https://blog.csdn.net/weixin_59612053/article/details/121679716)

---

### 行转列

!!! question

    查询所有玩家中，每个玩家战斗力 top3 的角色名和对应的角色创建时间，若不满三个角色放空，输出 `user_id, role_id1, create_time1, role_id2, create_time2, role_id3, create_time3`

表：`ods_game_dev.ods_role_create`, `ods_game_dev.ods_fight_power`

参考：[Hive 函数相关](./Hive 函数相关.md) 中的行转列 Case 2

```mysql
with      fight_power_table as (
          select    *
          from      (
                    select    user_id,
                              role_id,
                              role_name,
                              row_number() over (partition by user_id order by combat_power desc) as rn
                    from      ods_game_dev.ods_fight_power
                    ) t1
          where     t1.rn <= 3
          ),
          role_table as (
          select    role_id,
                    from_unixtime(event_time, 'yyyy-MM-dd') as create_time
          from      ods_game_dev.ods_role_create
          )
select    t3.user_id,
          max(t3.role_id1)     as role_id1,
          max(t3.create_time1) as create_time1,
          max(t3.role_id2)     as role_id2,
          max(t3.create_time2) as create_time2,
          max(t3.role_id3)     as role_id3,
          max(t3.create_time3) as create_time3
from      (
          select    t1.user_id,
                    if(t1.rn = 1, t1.role_id, null)     as role_id1,
                    if(t1.rn = 1, t2.create_time, null) as create_time1,
                    if(t1.rn = 2, t1.role_id, null)     as role_id2,
                    if(t1.rn = 2, t2.create_time, null) as create_time2,
                    if(t1.rn = 3, t1.role_id, null)     as role_id3,
                    if(t1.rn = 3, t2.create_time, null) as create_time3
          from      fight_power_table t1
          left join role_table t2 on t1.role_id = t2.role_id
          ) t3
group by  t3.user_id;
```

---

## 练习阶段 1：基础入门

```mysql
create table
  deliver_record_detail (
    user_id int,
    job_id int,
    job_city string,
    device string,
    min_salary int,
    max_salary int,
    deliver_date date
  );
```

### 1

!!! question

    查询投递简历中要求最高薪资前三的用户

```mysql
select user_id, max_salary
from deliver_record_detail
order by max_salary desc
limit 3
```

| user_id | max_salary |
| ------- | ---------- |
| 105     | 60         |
| 102     | 30         |
| 101     | 20         |

---

### 2

!!! question

    查询在 pc 上投递的所有投递记录

```mysql
select *
from deliver_record_detail
where device = 'pc'
```

| user_id | job_id | job_city | device | min_salary | max_salary | deliver_date |
| ------- | ------ | -------- | ------ | ---------- | ---------- | ------------ |
| 102     | 14550  | 北京     | pc     | 10         | 15         | 2021-07-07   |
| 103     | 23654  | 北京     | pc     | 12         | NULL       | 2021-04-09   |
| 105     | 75432  | 厦门     | pc     | 30         | 60         | 2016-08-15   |
| 204     | 49642  | 北京市   | pc     | NULL       | NULL       | 2019-05-15   |

---

### 3

!!! question

    查询投递简历中最低薪资（min_salary）与最高薪资（max_salary）差别大于 2 的职位的投递用户 user_id

```mysql
select user_id
from deliver_record_detail
where max_salary - min_salary > 2
```

| user_id |
| ------- |
| 102     |
| 102     |
| 105     |

---

```mysql
create table
  questions_pass_record_detail (
    user_id int,
    question_type string,
    device string,
    pass_count int,
    date1 date
  );
```

### 4

!!! question

    统计每天刷题数超过 5 的 user_id 以及刷题数，返回结果字段为顺序以及名称为 `date_time|user_id|total_pass_count`

```mysql
select date1 as date_time, user_id, sum(pass_count) as total_pass_count
from questions_pass_record_detail
group by date1, user_id
having sum(pass_count) > 5
```

| date_time  | user_id | total_pass_count |
| ---------- | ------- | ---------------- |
| 2018-08-15 | 103     | 60               |
| 2019-05-15 | 104     | 20               |
| 2020-05-16 | 105     | 299              |
| 2021-04-09 | 102     | 9                |
| 2021-04-15 | 106     | 37               |
| 2021-07-07 | 102     | 15               |
| 2022-07-25 | 105     | 550              |

---

### 5

!!! question

    统计不同类型题目的刷题总数 `passCnt`，并按刷题总数进行升序排列，查询返回结果名称和顺序如下 `question_type|passCnt`

```mysql
select question_type, sum(pass_count) as passCnt
from questions_pass_record_detail
group by question_type
order by passCnt
```

| question_type | passcnt |
| ------------- | ------- |
| python        | 12      |
| java          | 39      |
| sql           | 944     |

---

### 6

!!! question

    找出 sql 类题目的单次最大刷题数，将单次最大刷题数命名为`maxCnt`，最小刷题数命名为`minCnt`，平均刷题数命名为`avgCnt`

```mysql
select max(pass_count) as maxCnt, min(pass_count) as minCnt, avg(pass_count) avgCnt
from questions_pass_record_detail
where question_type = 'sql'
```

| maxcnt | mincnt | avgcnt |
| ------ | ------ | ------ |
| 550    | 15     | 188.8  |

---

### 7

同第 5 题

---

### 8

!!! question

    查询每天刷题通过数最多的前二名用户 id 和刷题数，输出按照日期升序排序，查询返回结果名称和顺序为 `date_time|user_id|pass_count`

```mysql
select t2.date_time, t2.user_id, t2.pass_count
from (select *,
             row_number() over (partition by date_time order by pass_count desc) as rn
      from (select date1 as date_time, user_id, sum(pass_count) as pass_count
            from questions_pass_record_detail
            group by date1, user_id) t1) t2
where t2.rn <= 2
order by date_time
```

| date_time  | user_id | pass_count |
| ---------- | ------- | ---------- |
| 2018-08-15 | 103     | 60         |
| 2019-05-15 | 104     | 20         |
| 2020-03-01 | 101     | 2          |
| 2020-05-16 | 105     | 299        |
| 2021-04-09 | 102     | 9          |
| 2021-04-15 | 106     | 37         |
| 2021-07-07 | 102     | 15         |
| 2022-03-17 | 102     | 3          |
| 2022-07-25 | 105     | 550        |

---

### 9

!!! question

    查询用户 user_id，刷题日期 date （每组按照 date 降序排列）和该用户的下一次刷题日期 nextdate（若是没有则为 None），组之间按照 user_id 升序排序，每组内按照 date 升序排列，查询返回结果名称和顺序为 `user_id|date_time|nextdate`

```mysql
with      cte as (
          select    user_id,
                    date1             as date_time,
                    row_number() over (partition by user_id order by  date1) as rn
          from      questions_pass_record_detail
          )
select    t1.user_id,
          t1.date_time,
          t2.date_time
from      cte t1
left join cte t2 on t1.rn + 1 = t2.rn
```

| t1.user_id | t1.date_time | t2.date_time |
| ---------- | ------------ | ------------ |
| 101        | 2020-03-01   | 2021-07-07   |
| 101        | 2020-03-01   | 2022-07-25   |
| 101        | 2020-03-01   | 2021-04-15   |
| 102        | 2021-04-09   | 2021-07-07   |
| 102        | 2021-04-09   | 2022-07-25   |
| 102        | 2021-04-09   | 2021-04-15   |
| 102        | 2021-07-07   | 2022-03-17   |
| 102        | 2022-03-17   | NULL         |
| 103        | 2018-08-15   | 2021-07-07   |
| 103        | 2018-08-15   | 2022-07-25   |
| 103        | 2018-08-15   | 2021-04-15   |
| 104        | 2019-05-15   | 2021-07-07   |
| 104        | 2019-05-15   | 2022-07-25   |
| 104        | 2019-05-15   | 2021-04-15   |
| 105        | 2020-05-16   | 2021-07-07   |
| 105        | 2020-05-16   | 2022-07-25   |
| 105        | 2020-05-16   | 2021-04-15   |
| 105        | 2022-07-25   | 2022-03-17   |
| 106        | 2021-04-15   | 2021-07-07   |
| 106        | 2021-04-15   | 2022-07-25   |
| 106        | 2021-04-15   | 2021-04-15   |
| 106        | 2021-04-15   | 2022-03-17   |

---

```mysql
CREATE TABLE user_info
(
    user_id         INT,
    graduation_year INT,
    register_time   TIMESTAMP,
    gender STRING,
    age             INT,
    university STRING
);
```

### 10

!!! question

    对于2021年以来有刷题的用户，请查询他们的 user_id 和毕业院校

```mysql
select user_id, university
from user_info
where year(register_time) > 2021;
```

---

```mysql
CREATE TABLE deliver_record
(
    user_id           INT,
    job_id            INT,
    platform          STRING,
    resume_id         INT,
    resume_if_checked INT,
    deliver_date      DATE
);

CREATE TABLE job_info
(
    job_id     INT,
    boss_id    INT,
    company_id INT,
    post_time  TIMESTAMP,
    salary     INT,
    job_city   STRING
);
```

### 11

!!! question

    查询每个公司 `company_id` 查看过的投递用户数 `cnt`，`resume_if_checked` 简历是否被查看 1 被查看 0 未被查看，查询返回结果名称和顺序 `company_id|cnt`

```mysql
select    j.company_id,
          sum(distinct resume_if_checked) as cnt
from      deliver_record d
inner     join job_info j on d.job_id = j.job_id
group by  j.company_id
```

| company_id | cnt |
| ---------- | --- |
| 2          | 1   |
| 3          | 1   |
| 6          | 0   |
| 9          | 0   |
| 16         | 1   |

---

### 12

!!! question

    查询职位城市在北京的 job_id 和 company_id，与职位工资高于 100000 的 job_id 和 company_id，二者合并输出不去重，查询返回结果名称和顺序为 `job_id|company_id`

```mysql
select job_id, company_id
from job_info
where job_city = '北京'
union all
select job_id, company_id
from job_info
where salary > 100000
```

| \_u1.job_id | \_u1.company_id |
| ----------- | --------------- |
| 18903       | 3               |
| 18903       | 3               |
| 34992       | 9               |
| 34992       | 9               |
| 22889       | 16              |

---

```mysql
CREATE TABLE customers_info
(
    customer_id             INT,
    gender                  STRING,
    city                    STRING,
    country                 STRING,
    age                     INT,
    latest_place_order_date DATE
);
```

### 13

!!! question

    按城市对客户进行排序,如果城市为空，则按国家排序，返回全部字段

```mysql
select    *
from      customers_info
order by  city, country
```

---

### 14

!!! question

    按年龄给客户分群（age_group 分为 '20 以下' ，'20-50' ， '50 以上'，'未填写' 四个群体）, 并计算各群体人数并命名为 user_count

```mysql
SELECT category, count(*) as user_count
from (SELECT customer_id,
             age,
             (case
                  when age < 20 then "20以下"
                  when age >= 20 and age <= 50 then "20-50"
                  when age > 50 then "50以上"
                  else "未填写" end) as category
      from customers_info) t
group by category
```

| category | user_count |
| -------- | ---------- |
| 20-50    | 3          |
| 20 以下  | 1          |
| 50 以上  | 1          |
| 未填写   | 1          |

---

### 15

```mysql
CREATE TABLE done_questions_record
(
    user_id       INT,
    question_id   INT,
    question_type STRING,
    done_time     TIMESTAMP,
    result_info   INT
)
```

!!! question

    输出提交次数大于 2 次的用户 ID 且倒序排列

```mysql
select user_id, count(question_id) as cnt
from done_questions_record
group by user_id
having cnt > 2
order by user_id desc
```

| user_id | cnt |
| ------- | --- |
| 105     | 6   |
| 104     | 3   |
| 103     | 5   |
| 102     | 3   |
| 101     | 4   |

---

### 16

!!! question

    `question_pass_rate` 表示每个用户不同题目的通过率（同一用户同一题重复提交通过仅计算一次）；`pass_rate` 表示每个用户的提交正确率（只要有提交一次即计算一次）；`question_per_cnt` 表示平均每道不同的题目被提交的次数（只要有一次提交即计算一次），请你输出题目通过率 `question_pass_rate` > 60% 的用户的提交正确率 `pass_rate` 与每题目平均提交次数 `question_per_cnt`。按照用户名升序排序。`result_info` '是否通过，1：通过； 0：不通过'，查询返回结果名称和顺序为 `user_id|question_pass_rate|pass_rate|question_per_cnt`

```mysql
select *
from (select user_id,
             sum(case when max_result_info = 1 then 1 else 0 end) / count(distinct question_id) as question_pass_rate,
             sum(case when result_info = 1 then 1 else 0 end) / count(distinct question_id)     as user_pass_rate,
             count(*)                                                                           as question_per_cnt
      from (select user_id,
                   question_id,
                   max(result_info) as max_result_info,
                   min(result_info) as result_info
            from done_questions_record
            group by user_id, question_id) t1
      group by user_id) t2
where question_pass_rate > 0.6
order by user_id
```

| t2.user_id | t2.question_pass_rate | t2.user_pass_rate  | t2.question_per_cnt |
| ---------- | --------------------- | ------------------ | ------------------- |
| 101        | 1                     | 0.6666666666666666 | 3                   |
| 102        | 1                     | 1                  | 3                   |
| 104        | 0.6666666666666666    | 0.6666666666666666 | 3                   |
| 105        | 0.6666666666666666    | 0.3333333333333333 | 3                   |
| 106        | 1                     | 0                  | 1                   |

---

### 17

!!! question "Q1"

    将附件中 `ip_china.csv.zip` 文件加载为 Hive 内部表，保持格式与 csv header 一致，表需要开启压缩

```bash
tar -zxvf ip_china.csv.zip
```

```mysql
create table if not exists ip_china
(
    ip_start      string,
    ip_end        string,
    long_ip_start string,
    long_ip_end   string,
    country       string,
    province      string
)
row format delimited fields terminated by ','
stored as TEXTFILE
location '/user/hive/warehouse/ip_china'
tblproperties ('compression.codec' = 'gzip');
```

```bash
hdfs dfs -put ip_china.csv /user/hive/warehouse/ip_china
```

---

!!! question "Q2"

    将附件中 login_data.csv.zip 文件加载为 Hive 外部表，保持格式与 csv header 一致，表需要开启压缩，需要按日分区

```bash
tar -zxvf login_data.csv.zip
```

```mysql
create external table if not exists filter_data
(
    logtime    string,
    account_id bigint,
    ip         string
)
    row format delimited fields terminated by ',';

load data local inpath '/opt/module/datas/login_data.csv' into table filter_data;
```

```mysql
create external table if not exists login_data
(
    logtime    string,
    account_id bigint,
    ip         string
)
    partitioned by (dt string)
    row format delimited fields terminated by ','
    stored as TEXTFILE
    location '/user/hive/warehouse/login_data'
    tblproperties ('compression.codec' = 'gzip');

set hive.exec.dynamic.partition.mode=nonstrict;

insert overwrite table login_data partition (dt)
select logtime, account_id, ip, substr(logtime, 1, 10) as dt
from filter_data
where logtime != 'logtime'; -- 跳过标题行
```

---

!!! question "Q3"

    通过 Q1，Q2 加载的数据，将用户登陆表中的 ip 转化为对应的国家地区并分区表，增加字段，将登陆时间转化成日期存入

---

!!! question "Q4"

    请输出每个分区下，每个 province 的去重登陆人数。输出结构为 pt，province，cnt_login

```mysql

```

---

!!! question "Q5"

    请输出总量数据下，存在登陆数据的各个 province 中，登陆时间最早的前 3 人及对应的登陆时间，若不满 3 人，需要留空。输出结构为`province，account_id_1, login_time_1, account_id_2, login_time_2, account_id_3, login_time_3`

```mysql

```
