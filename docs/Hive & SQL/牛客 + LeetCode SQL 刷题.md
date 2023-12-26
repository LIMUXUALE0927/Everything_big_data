# SQL 刷题

## 未分类

### [SQL215 查找在职员工自入职以来的薪水涨幅情况](https://www.nowcoder.com/practice/fc7344ece7294b9e98401826b94c6ea5?tpId=82&tqId=29753&rp=1&ru=%2Fexam%2Fcompany&qru=%2Fexam%2Fcompany&sourceUrl=%2Fexam%2Fcompany&difficulty=undefined&judgeStatus=undefined&tags=&title=)

先通过 CTE 过滤出当前还在职的员工信息（`to_date = '9999-01-01'`），再用窗口函数 `first_value()` 找到每个员工入职时的薪水，二者的差就是答案。

```sql
with cte as (select *
             from salaries s1
             where exists (select 1
                           from salaries s2
                           where s2.emp_no = s1.emp_no
                             and s2.to_date = '9999-01-01'))
select emp_no,
       salary - prev_salary as growth
from (select *,
             first_value(salary) over (partition by emp_no order by to_date) as prev_salary
      from cte) t
where to_date = '9999-01-01'
order by growth
```

---

### [SQL254 统计 salary 的累计和 running_total](https://www.nowcoder.com/practice/58824cd644ea47d7b2b670c506a159a6?tpId=82&tags=&title=&difficulty=0&judgeStatus=0&rp=1&sourceUrl=%2Fexam%2Fcompany)

```sql
select emp_no,
       salary,
       sum(salary) over (order by emp_no) as running_total
from salaries
where to_date = '9999-01-01'
```

```sql
select emp_no,
       salary,
       sum(salary) over (order by emp_no rows between unbounded preceding and current row) as running_total
from salaries
where to_date = '9999-01-01'
```

---

### [SQL261 牛客每个人最近的登录日期(二)](https://www.nowcoder.com/practice/7cc3c814329546e89e71bb45c805c9ad?tpId=82&tags=&title=&difficulty=0&judgeStatus=0&rp=1&sourceUrl=%2Fexam%2Fcompany)

```sql
select u.name, c.name, t1.date
from (select *,
             row_number() over (partition by user_id order by date desc) as rn
      from login) t1
inner join user u
on u.id = t1.user_id
inner join client c
on t1.client_id = c.id
where t1.rn = 1
order by u.name
```

---

### :fire: [SQL262 牛客每个人最近的登录日期(三)](https://www.nowcoder.com/practice/16d41af206cd4066a06a3a0aa585ad3d?tpId=82&tags=&title=&difficulty=0&judgeStatus=0&rp=1&sourceUrl=%2Fexam%2Fcompany)

方法一：

```sql
-- 在所有首日登录的用户中，计算第二天登录的用户的比例
select round(sum(date = date_add(t.d, interval 1 day)) / sum(date = t.d), 3)
from (select *,
             min(date) over (partition by user_id) d
      from login) t -- 筛选出每个用户的首日登录日期
```

方法二：

```sql
select round(count(l2.date) / count(*), 3) p
from (select user_id, min(date) first_date
      from login
      group by user_id) l1
left join login l2 on l1.user_id = l2.user_id and l2.date = date_add(l1.first_date, interval 1 day)
```

---

### [SQL263 牛客每个人最近的登录日期(四)](https://www.nowcoder.com/practice/e524dc7450234395aa21c75303a42b0a?tpId=82&tags=&title=&difficulty=0&judgeStatus=0&rp=1&sourceUrl=%2Fexam%2Fcompany)

```sql
select date,
       sum(rk = '1') as new
from (select *,
             row_number() over (partition by user_id order by date) as rk
      from login) t
group by date
order by date
```

---

### :fire: [SQL264 牛客每个人最近的登录日期(五)](https://www.nowcoder.com/practice/ea0c56cd700344b590182aad03cc61b8?tpId=82&tags=&title=&difficulty=0&judgeStatus=0&rp=1&sourceUrl=%2Fexam%2Fcompany)

:star: 解决此类问题的通用方法：

**方法一：约束条件放在 from left join 中**

```sql
select date, ifnull(round(count() / count(), 3), 0)
from （select 去重的日期表）
    left join on （select 新用户的首次登陆日期表 ）
    left join on （select 新用户的次日登录日期表 ）
group by date
order by date
```

**方法二：约束条件直接放在分子分母中**

```sql
select date, ifnull(round(sum(CASE WHEN ...) / sum(CASE WHEN ...), 3), 0)
from login
group by date
order by date
```

方法一：

```sql
select t0.date,
       ifnull(round(count(distinct t2.user_id) / count(t1.user_id), 3), 0) as p
from (select date
      from login
      group by date) t0 -- step1: 日期去重
         left join (select user_id, min(date) as first_login
                    from login
                    group by user_id) t1
                   on t0.date = t1.first_login -- step2: 每个用户的首次登录日期
         left join login as t2
                   on t1.user_id = t2.user_id and
                      t2.date = date_add(t1.first_login, interval 1 day) -- step3: 每个用户的次日登录日期
group by t0.date
order by t0.date;
```

方法二：

```sql
-- 分子：当前日期作为前一天有该用户的登录记录，并且前一天是第一次登录。
-- 分母：当前日期新用户的特征是：当前日期=该用户所有登录日期的最小值
select date,
       ifnull(round(sum(IF((user_id, date) in
                           (select user_id, date_add(date, interval -1 day) from login) and
                           (user_id, date) in (select user_id, min(date) from login group by user_id), 1, 0)) /
                    sum(IF((user_id, date) in (select user_id, min(date) from login group by user_id), 1, 0)), 3),
              0) as p
from login
group by date
order by date;
```

---

### [SQL265 牛客每个人最近的登录日期(六)](https://www.nowcoder.com/practice/572a027e52804c058e1f8b0c5e8a65b4?tpId=82&tags=&title=&difficulty=0&judgeStatus=0&rp=1&sourceUrl=%2Fexam%2Fcompany)

```sql
select name as u_n,
       date,
       sum(number) over (partition by user_id order by date) as ps_num
from (select u.name, l.user_id, l.date, p.number
      from login l
               inner join passing_number p
                          on l.user_id = p.user_id and l.date = p.date
               inner join user u
                          on l.user_id = u.id) t
order by date, name;
```

---

### :fire: [国庆期间每类视频点赞量和转发量](https://www.nowcoder.com/practice/f90ce4ee521f400db741486209914a11)

:star: 滑动窗口

MySQL:

```sql
select *
from (select t1.tag,
             t1.dt,
             sum(t1.if_like_sum) over (partition by t1.tag order by t1.dt rows 6 preceding)    as sum_like_cnt_7d,
             max(t1.if_retweet_sum) over (partition by t1.tag order by t1.dt rows 6 preceding) as max_retweet_cnt_7d
      from (select tag,
                   date(start_time) as dt,
                   sum(if_like)     as if_like_sum,
                   sum(if_retweet)  as if_retweet_sum
            from tb_user_video_log l
                     inner join tb_video_info v on l.video_id = v.video_id
            group by tag, dt) t1) t2
where t2.dt between '2021-10-01' and '2021-10-03'
order by t2.tag desc, t2.dt asc;
```

HQL:

```sql
select *
from (select t1.tag,
             t1.dt,
             sum(t1.if_like_sum) over (partition by t1.tag order by t1.dt rows 6 preceding)    as sum_like_cnt_7d,
             max(t1.if_retweet_sum) over (partition by t1.tag order by t1.dt rows 6 preceding) as max_retweet_cnt_7d
      from (select tag,
                   to_date(start_time) as dt,
                   sum(if_like)        as if_like_sum,
                   sum(if_retweet)     as if_retweet_sum
            from tb_user_video_log l
                     inner join tb_video_info v on l.video_id = v.video_id
            group by tag, to_date(start_time)) t1) t2
where t2.dt between '2021-10-01' and '2021-10-03'
order by t2.tag desc, t2.dt asc;
```

---

### :fire: [近一个月发布的视频中热度最高的 top3 视频](https://www.nowcoder.com/practice/0226c7b2541c41e59c3b8aec588b09ff)

MySQL:

```sql
SELECT video_id,
       ROUND((100 * comp_play_rate + 5 * like_cnt + 3 * comment_cnt + 2 * retweet_cnt)
                 / (TIMESTAMPDIFF(DAY, recently_end_date, cur_date) + 1), 0) as hot_index
FROM (SELECT video_id,
             AVG(IF(TIMESTAMPDIFF(SECOND, start_time, end_time)
                        >= duration, 1, 0)) as comp_play_rate,
             SUM(if_like)                   as like_cnt,
             COUNT(comment_id)              as comment_cnt,
             SUM(if_retweet)                as retweet_cnt,
             MAX(DATE(end_time))            as recently_end_date, -- 最近被播放日期
             MAX(DATE(release_time))        as release_date,      -- 发布日期
             MAX(cur_date)                  as cur_date           -- 非分组列，加MAX避免语法错误
      FROM tb_user_video_log
               JOIN tb_video_info USING (video_id)
               LEFT JOIN (SELECT MAX(DATE(end_time)) as cur_date
                          FROM tb_user_video_log) as t_max_date ON 1
      GROUP BY video_id
      HAVING TIMESTAMPDIFF(DAY, release_date, cur_date) < 30) as t_video_info
ORDER BY hot_index DESC
LIMIT 3;
```

HQL:

```sql
with cte as (select *
             from (select l.*,
                          v.duration,
                          v.release_time,
                          tmp.cur
                   from tb_video_info v
                            inner join tb_user_video_log l on v.video_id = l.video_id
                            left join (select max(to_date(l.end_time)) as cur from tb_user_video_log l) tmp on 1 = 1) t
             where to_date(release_time) between date_sub(cur, 29) and cur)
select video_id,
       round((100 * finished_rate + 5 * like_cnt + 3 * comment_cnt + 2 * retweet_cnt)
                 / recent_interval, 0) as hot_index
from (select video_id,
             sum(if(unix_timestamp(end_time) -
                    unix_timestamp(start_time) >= duration, 1, 0)) / count(1) as finished_rate,
             sum(if_like)                                                     as like_cnt,
             count(comment_id)                                                as comment_cnt,
             sum(if_retweet)                                                  as retweet_cnt,
             datediff(max(cur), max(to_date(end_time))) + 1                   as recent_interval
      from cte
      group by video_id) t
order by hot_index desc
limit 3;
```

---

### :fire: [统计活跃间隔对用户分级结果](https://www.nowcoder.com/practice/6765b4a4f260455bae513a60b6eed0af)

MySQL:

```sql
with cte as (select uid,
                    date(in_time)              as in_date,
                    date(out_time)             as out_date,
                    max(date(in_time)) over () as today
             from tb_user_log
             order by uid, in_date, out_date)
select user_grade,
       round(count(1) / max(tmp.total), 2) as ratio
from (select *,
             case
                 when df_latest >= 30 then "流失用户"
                 when df_latest >= 7 then "沉睡用户"
                 when df_earliest < 7 then "新晋用户"
                 else "忠实用户" end as user_grade
      from (select uid,
                   datediff(max(today), min(in_date)) + 1 as df_earliest,
                   datediff(max(today), max(in_date)) + 1 as df_latest
            from cte
            group by uid) t1) t2
         left join (select count(distinct uid) as total from tb_user_log) tmp on 1 = 1
group by user_grade
order by ratio desc, user_grade asc;
```

HQL:

```sql
with cte as (select uid,
                    to_date(in_time)              as in_date,
                    to_date(out_time)             as out_date,
                    max(to_date(in_time)) over () as today
             from tb_user_log
             order by uid, in_date, out_date)
select user_grade,
       round(count(1) / max(tmp.total), 2) as ratio
from (select *,
             case
                 when df_latest >= 30 then "流失用户"
                 when df_latest >= 7 then "沉睡用户"
                 when df_earliest < 7 then "新晋用户"
                 else "忠实用户" end as user_grade
      from (select uid,
                   datediff(max(today), min(in_date)) + 1 as df_earliest,
                   datediff(max(today), max(in_date)) + 1 as df_latest
            from cte
            group by uid) t1) t2
         left join (select count(distinct uid) as total from tb_user_log) tmp on 1 = 1
group by user_grade
order by ratio desc, user_grade asc;
```

---

### [SQL218 获取所有非 manager 员工当前的薪水情况](https://www.nowcoder.com/practice/8fe212a6c71b42de9c15c56ce354bebe?tpId=82&tqId=29753&rp=1&ru=%2Fexam%2Fcompany&qru=%2Fexam%2Fcompany&sourceUrl=%2Fexam%2Fcompany&difficulty=undefined&judgeStatus=undefined&tags=&title=)

```sql
select de.dept_no, e.emp_no, s.salary
from employees e
inner join dept_emp de
using (emp_no)
inner join salaries s
using (emp_no)
where not exists (
    select 1
    from dept_manager dm
    where e.emp_no = dm.emp_no
)
```

---

### [SQL219 获取员工其当前的薪水比其 manager 当前薪水还高的相关信息](https://www.nowcoder.com/practice/f858d74a030e48da8e0f69e21be63bef?tpId=82&tqId=29753&rp=1&ru=%2Fexam%2Fcompany&qru=%2Fexam%2Fcompany&sourceUrl=%2Fexam%2Fcompany&difficulty=undefined&judgeStatus=undefined&tags=&title=)

```sql
select emp_no, manager_no, emp_salary, manager_salary
from (
    select s.emp_no, dept_no, salary as emp_salary
    from dept_emp de
    inner join salaries s
    using (emp_no)
) t1

inner join (
    select dm.emp_no as manager_no, dept_no, salary as manager_salary
    from dept_manager dm
    inner join salaries
    using (emp_no)
) t2

on t1.dept_no = t2.dept_no and
t1.emp_salary > t2.manager_salary
```

---

### [SQL247 按照 dept_no 进行汇总](https://www.nowcoder.com/practice/6e86365af15e49d8abe2c3d4b5126e87?tpId=82&tags=&title=&difficulty=0&judgeStatus=0&rp=1&sourceUrl=%2Fexam%2Fcompany)

```sql
select dept_no,
       group_concat(emp_no)
from dept_emp
group by dept_no
```

---

### [SQL253 获取有奖金的员工相关信息](https://www.nowcoder.com/practice/5cdbf1dcbe8d4c689020b6b2743820bf?tpId=82&tags=&title=&difficulty=0&judgeStatus=0&rp=1&sourceUrl=%2Fexam%2Fcompany)

```sql
select e.emp_no,
       e.first_name,
       e.last_name,
       b.btype,
       s.salary,
       case
           when b.btype = 1 then s.salary * 0.1
           when b.btype = 2 then s.salary * 0.2
           else s.salary * 0.3
           end as bonus
from employees e
         join salaries s on e.emp_no = s.emp_no
         join emp_bonus b on e.emp_no = b.emp_no
where s.to_date = '9999-01-01';
```

---

### [SQL259 异常的邮件概率](https://www.nowcoder.com/practice/d6dd656483b545159d3aa89b4c26004e?tpId=82&tags=&title=&difficulty=0&judgeStatus=0&rp=1&sourceUrl=%2Fexam%2Fcompany)

```sql
select date,
       round(sum(type = 'no_completed') / count(*), 3) as p
from email e
         inner join user u1 on e.send_id = u1.id and u1.is_blacklist = 0
         inner join user u2 on e.receive_id = u2.id and u2.is_blacklist = 0
group by date
order by date;
```

---

### :fire: [验证刷题效果，输出题目真实通过率](https://www.nowcoder.com/practice/c4fd4b545a704877b510f18503ad523f)

![](https://raw.githubusercontent.com/MXJULY/image/main/img/202312050537078.png)

MySQL:

```sql
select user_id,
       count(distinct if(result_info = 1, question_id, null)) / count(distinct question_id) as question_pass_rate,
       sum(result_info) / count(result_info) as pass_rate,
       count(result_info) / count(distinct question_id) as question_per_cnt
from done_questions_record
group by user_id
having question_pass_rate > 0.6
order by user_id;
```

HQL:

```sql
select user_id,
       count(distinct if(result_info = 1, question_id, null)) / count(distinct question_id) as question_pass_rate,
       sum(result_info) / count(result_info) as pass_rate,
       count(result_info) / count(distinct question_id) as question_per_cnt
from done_questions_record
group by user_id
having count(distinct if(result_info = 1, question_id, null)) / count(distinct question_id) > 0.6
order by user_id
```

---

### :fire: [未完成试卷数大于 1 的有效用户](https://www.nowcoder.com/practice/46cb7a33f7204f3ba7f6536d2fc04286)

HQL:

```sql
select uid,
       sum(if(submit_time is null, 1, 0))                            as incomplete_cnt,
       sum(if(submit_time is not null, 1, 0))                        as complete_cnt,
       collect_set(concat_ws(':', string(to_date(start_time)), tag)) as detail
from exam_record er
         inner join examination_info ei
                    on er.exam_id = ei.exam_id
where year(start_time) = 2021
group by uid
having complete_cnt > 1
   and incomplete_cnt > 1
   and incomplete_cnt < 5
order by incomplete_cnt desc;
```

---

### [连续两次作答试卷的最大时间窗](https://www.nowcoder.com/practice/9dcc0eebb8394e79ada1d4d4e979d73c)

MySQL:

```sql
with t1 as (select *,
                   datediff(date(start_time), lag(start_time) over (partition by uid order by start_time)) +
                   1 as diff
            from exam_record
            where year(start_time) = '2021')
select uid,
       days_window,
       round(avg_cnt * days_window, 2) as avg_exam_cnt
from (select uid,
             max(diff)                                                            as days_window,
             count(start_time) / (datediff(max(start_time), min(start_time)) + 1) as avg_cnt
      from t1
      group by uid
      having count(distinct date(start_time)) >= 2) t
order by days_window desc, avg_exam_cnt desc
```

HQL:

Hive 不支持 `having count(distinct date(start_time)) >= 2`

```sql
with cte as (select *,
                    datediff(to_date(start_time), lag(start_time) over (partition by uid order by start_time)) +
                    1 as diff
             from exam_record
             where year(start_time) = '2021')
select uid,
       days_window,
       round(avg_cnt * days_window, 2) as avg_exam_cnt
from (select *
      from (select uid,
                   max(diff)                                                            as days_window,
                   count(start_time) / (datediff(max(start_time), min(start_time)) + 1) as avg_cnt,
                   count(distinct to_date(start_time))                                  as cnt
            from cte
            group by uid) t1) t2
where cnt >= 2
order by days_window desc, avg_exam_cnt desc
```

---

### :fire: [未完成率较高的 50%用户近三个月答卷情况](https://www.nowcoder.com/practice/3e598a2dcd854db8b1a3c48e5904fe1c)

MySQL:

```mysql
# 统计SQL试卷上用户未完成率及百分比排位:
with t1 as (select *,
                   percent_rank() over (order by incomplete_rate desc) as prk
            from (select uid,
                         sum(if(submit_time is null, 1, 0)) / count(start_time) as incomplete_rate
                  from exam_record er
                           inner join examination_info ei on er.exam_id = ei.exam_id
                  where tag = 'SQL'
                  group by uid) tmp),
# 有试卷作答记录的近三个月
     t2 as (select er.uid,
                   date_format(start_time, '%Y%m')                                   as start_month,
                   dense_rank() over (order by date_format(start_time, '%Y%m') desc) as drk,
                   submit_time
            from exam_record er)
select uid,
       start_month,
       count(start_month) as total_cnt,
       count(submit_time) as complete_cnt
from t2
where uid in (select uid from t1 where prk <= 0.5)
  and uid in (select uid from user_info where level = 6 or level = 7)
  and drk <= 3
group by uid, start_month
order by uid, start_month;
```

Spark SQL:

```sql
with t1 as (select *,
                   percent_rank() over (order by incomplete_rate desc) as prk
            from (select uid,
                         sum(if(submit_time is null, 1, 0)) / count(start_time) as incomplete_rate
                  from exam_record er
                           inner join examination_info ei on er.exam_id = ei.exam_id
                  where tag = 'SQL'
                  group by uid) tmp),
     t2 as (select er.uid,
                   date_format(start_time, 'yyyy-MM')                                   as start_month,
                   dense_rank() over (order by date_format(start_time, 'yyyy-MM') desc) as drk,
                   submit_time
            from exam_record er)
select uid,
       start_month,
       count(start_month) as total_cnt,
       count(submit_time) as complete_cnt
from t2
where uid in (select uid from t1 where prk <= 0.5)
  and uid in (select uid from user_info where level = 6 or level = 7)
  and drk <= 3
group by uid, start_month
order by uid, start_month;
```

---

### [每月及截止当月的答题情况](https://www.nowcoder.com/practice/1ce93d5cec5c4243930fc5e8efaaca1e)

MySQL:

```sql
with cte as (select uid,
                    start_month,
                    sum(if(rn = 1, 1, 0)) as is_first
             from (select uid,
                          date_format(start_time, '%Y%m')                          as start_month,
                          row_number() over (partition by uid order by start_time) as rn
                   from exam_record) t
             group by start_month, uid)
select *,
       max(month_add_uv) over (order by start_month) as max_month_add_uv,
       sum(month_add_uv) over (order by start_month) as sum_month_add_uv
from (select start_month,
             count(distinct uid) as mau,
             sum(is_first)       as month_add_uv
      from cte
      group by start_month) t1
order by start_month;
```

HQL:

```sql
with cte as (select uid,
                    start_month,
                    sum(if(rn = 1, 1, 0)) as is_first
             from (select uid,
                          date_format(start_time, 'yyyyMM')                        as start_month,
                          row_number() over (partition by uid order by start_time) as rn
                   from exam_record) t
             group by start_month, uid)
select *,
       max(month_add_uv) over (order by start_month) as max_month_add_uv,
       sum(month_add_uv) over (order by start_month) as sum_month_add_uv
from (select start_month,
             count(distinct uid) as mau,
             sum(is_first)       as month_add_uv
      from cte
      group by start_month) t1
order by start_month;
```

---

### :fire: [根据指定记录是否存在输出不同情况](https://www.nowcoder.com/practice/f72d3fc27dc14f3aae76ee9823ccca6b)

使用 EXISTS 和 NOT EXISTS 来实现互斥条件。

MySQL:

```sql
with cte as (select ui.uid,
                    level,
                    count(start_time)                                                                 as total_cnt,
                    count(start_time) - count(submit_time)                                            as incomplete_cnt,
                    round(ifnull((count(start_time) - count(submit_time)) / count(start_time), 0), 3) as incomplete_rate
             from user_info ui
                      left join exam_record er on er.uid = ui.uid
             group by ui.uid)
select uid,
       incomplete_cnt,
       incomplete_rate
from cte
where exists(select 1
             from cte
             where level = 0
               and incomplete_cnt > 2)
  and level = 0
union all
select uid,
       incomplete_cnt,
       incomplete_rate
from cte
where not exists(select 1
                 from cte
                 where level = 0
                   and incomplete_cnt > 2)
  and total_cnt > 0
order by incomplete_rate;
```

HQL:

注意：Hive 中的使用 EXISTS 必须添加相关性条件，否则会报错。

```sql
with t1 as (select ui.uid,
                   max(level)                                                                     as level,
                   count(start_time)                                                              as total_cnt,
                   count(start_time) - count(submit_time)                                         as incomplete_cnt,
                   round(nvl((count(start_time) - count(submit_time)) / count(start_time), 0), 3) as incomplete_rate
            from user_info ui
                     left join exam_record er on er.uid = ui.uid
            group by ui.uid),
     t2 as (select *
            from t1
            where level = 0 and incomplete_cnt > 2)
select uid,
       incomplete_cnt,
       incomplete_rate
from t1
where exists (select 1
              from t2
              where t2.uid = t1.uid) -- 添加相关性条件
  and level = 0
union all
select uid,
       incomplete_cnt,
       incomplete_rate
from t1
where not exists (select 1
                  from t2
                  where t2.uid = t1.uid) -- 添加相关性条件
  and total_cnt > 0
order by incomplete_rate;
```

---

### [10 月的新户客单价和获客成本](https://www.nowcoder.com/practice/d15ee0798e884f829ae8bd27e10f0d64)

=== "SQL"

      ```mysql
      with t1 as (select order_id,
                        uid,
                        date(event_time)                                         as order_date,
                        row_number() over (partition by uid order by event_time) as rn,
                        total_amount
                  from tb_order_overall),
      t2 as (select *
                  from t1
                  where rn = 1
                  and date_format(order_date, '%Y-%m') = '2021-10')
      select round(avg(amount), 1) as avg_amount,
            round(avg(cost), 1)   as avg_cost
      from (select avg(total_amount)                                                    as amount,
                  (sum(price * cnt) - max(total_amount)) / count(distinct t2.order_id) as cost
            from t2
                  inner join tb_order_detail tod on t2.order_id = tod.order_id
            group by t2.order_id) t;
      ```

=== "HQL"

      ```sql
      with t1 as (select order_id,
                        uid,
                        to_date(event_time)                                      as order_date,
                        row_number() over (partition by uid order by event_time) as rn,
                        total_amount
                  from tb_order_overall),
      t2 as (select *
                  from t1
                  where rn = 1
                  and date_format(order_date, 'yyyy-MM') = '2021-10')
      select round(avg(amount), 1) as avg_amount,
            round(avg(cost), 1)   as avg_cost
      from (select avg(total_amount)                                                    as amount,
                  (sum(price * cnt) - max(total_amount)) / count(distinct t2.order_id) as cost
            from t2
                  inner join tb_order_detail tod on t2.order_id = tod.order_id
            group by t2.order_id) t;
      ```

---

### :fire: :fire: [工作日各时段叫车量、等待接单时间和调度时间](https://www.nowcoder.com/practice/34f88f6d6dc549f6bc732eb2128aa338)

=== "SQL"

      ```mysql
      with t1 as (select tcr.uid,
                        tcr.event_time, -- 下单时间
                        tcr.order_id,
                        tco.order_time, -- 接单时间
                        tco.start_time, -- 开始时间
                        tco.finish_time,
                        case
                        when hour(tcr.event_time) in (7, 8) then '早高峰'
                        when hour(tcr.event_time) between 9 and 16 then '工作时间'
                        when hour(tcr.event_time) in (17, 18, 19) then '晚高峰'
                        else '休息时间' end as period
                  from tb_get_car_record tcr
                        inner join tb_get_car_order tco on tcr.order_id = tco.order_id
                  where dayofweek(tcr.event_time) between 2 and 6)
      select period,
            count(1)                                                                      as get_car_num,
            round(avg((unix_timestamp(order_time) - unix_timestamp(event_time)) / 60), 1) as avg_wait_time,
            round(avg((unix_timestamp(start_time) - unix_timestamp(order_time)) / 60), 1) as avg_dispatch_time
      from t1
      group by period
      order by get_car_num;
      ```

=== "HQL"

      ```sql
      with t1 as (select tcr.uid,
                        tcr.event_time, -- 下单时间
                        tcr.order_id,
                        tco.order_time, -- 接单时间
                        tco.start_time, -- 开始时间
                        tco.finish_time,
                        case
                        when hour(tcr.event_time) in (7, 8) then '早高峰'
                        when hour(tcr.event_time) between 9 and 16 then '工作时间'
                        when hour(tcr.event_time) in (17, 18, 19) then '晚高峰'
                        else '休息时间' end as period
                  from tb_get_car_record tcr
                        inner join tb_get_car_order tco on tcr.order_id = tco.order_id
                  where dayofweek(tcr.event_time) between 2 and 6)
      select period,
            count(1)                                                          as get_car_num,
            round(avg((unix_timestamp(order_time) - unix_timestamp(event_time)) / 60), 1) as avg_wait_time,
            round(avg((unix_timestamp(start_time) - unix_timestamp(order_time)) / 60), 1) as avg_dispatch_time
      from t1
      group by period
      order by get_car_num;
      ```

---
