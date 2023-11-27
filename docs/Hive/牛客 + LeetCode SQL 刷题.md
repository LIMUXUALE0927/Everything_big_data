# 牛客 + LeetCode SQL 刷题

## 窗口函数

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

## 其他

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

### [SQL235 构造一个触发器 audit_log](https://www.nowcoder.com/practice/7e920bb2e1e74c4e83750f5c16033e2e?tpId=82&tags=&title=&difficulty=0&judgeStatus=0&rp=1&sourceUrl=%2Fexam%2Fcompany)

```sql
create trigger audit_log
after insert on employees_test
for each row
begin
    insert into audit values(new.id,new.name);
end
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
