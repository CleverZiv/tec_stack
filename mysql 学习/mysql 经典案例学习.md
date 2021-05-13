## 1、查询 01 课程比 02 课程成绩高的学生信息及课程分数

1. 先根据 SId 连接 Student 和 SC 两张表，并可以创建一张临时表 temp
2. 将 temp 作为 01 课程的数据来源，将 SC 表作为 02 课程的数据源，依据 SId 连接，并且根据 课程名分别做筛选，最后根据分数进行筛选

```sql
select *
from (select a.*, S.CId, S.score
      from Student a
               inner join SC S on a.SId = S.SId) temp
         inner join SC s2 on temp.SId = s2.SId
    and temp.CId = '01' and s2.CId = '02'
where temp.score > s2.score;
```

也可以简化上述 sql，不用显示创建临时表

```sql
select *
from Student a
         inner join SC b on a.SId = b.SId
         inner join SC c on a.SId = c.SId and b.CId = '01' and c.CId = '02'
where b.score > c.score;
```

## 2、查询同时报了 01 课程和 02 课程的学生分数信息

1. SC 表有记录，就表明该学生报了这门课

2. 先筛选出 01 课程的成绩记录，作为临时表 temp

3. 再筛选出 02 课程的成绩记录，并与 01 通过 `inner join`连接，条件是 SId 一致

4. 3 中能匹配上的结果即为同时报了 01 和 02 课程的学生分数记录

```sql
select *
from (
         select *
         from SC
         where CId = '01'
     ) temp1
         inner join
     (select *
      from SC
      where CId = '02'
     ) temp2
     on temp1.SId = temp2.SId;
```

   也可以不用创建这么多临时表，直接使用自关联（自己和自己关联）

```sql
select *
from SC a
         inner join SC b
                    on a.SId = b.SId
where a.CId = '01'
  and b.CId = '02';
```

   

## 3、查询报了 01 课程但可能没报 02 课程的情况（不存在时显示为 null）
其实这题目的题意不是那么好理解，是一个根据答案来描述的题目。它想要达到的效果是，查出所有报了 01 课程的学生，打印出 01 课程信息的同时，如果该学生也报了 02 课程，那么也将 02 课程打印出来，如果该学生没有报 02 课程，那么久置为 null

   1. SC 中查询所有 CId 为 01 的记录
   2. 1 的结果根据 SId 和 CId 为 02 左连接 SC

   ```sql
   select *
   from (
            select *
            from SC
            where SC.CId = '01'
        ) a
            left join SC b
                      on a.SId = b.SId and b.CId = '02';
   ```

   我们同样可以不创建临时表，直接进行自连接

   ```sql
   select *
   from SC a
            left join SC b
                      on a.SId = b.SId and b.CId = '02'
   where a.CId = '01';
   ```

## 4、查询没报 01 课程的同学中报了 02 课程的同学分数情况

首先需要排除所有已经报了 01 课程的同学，然后在剩下的同学里看哪些是报了 02 课程的，把该同学的所有记录输出

   1. 查找报了 01 的同学
   2. 从 SC 中筛选掉 1 中的同学的记录，这作为左表
   3. 右表是 Cid 为 02 的记录
   4. 2和3的结果进行 inner join

   ```sql
   select *
   from (select * from sc where sid not in (select sid from SC where CId = '01')) a
            inner join SC b on a.SId = b.SId
       and b.CId = '02';
   ```

## 5、查询平均成绩超过60分的学生编号、姓名和平均成绩

   1. 首先要去成绩表 SC 中找到平均成绩超过60分的SId
   2. 带着1中的SId去student表中查询学生的信息

   ```sql
   select b.SId, Sname, avg_score
   from (select SId, avg(score) avg_score
         from SC
         group by SId
         having avg(score) > 60) a
            inner join Student b on a.SId = b.SId;
   ```

## 6、查询在 SC 表中存在成绩的学生信息

   `group by`：只能查找出分组字段，其它字段需要以聚合函数（sum、avg）的形式被查询出来

   1. 首先需要找出在 SC 表中存在的所有学生 SId
   2. 然后拿着1中的sid去student表中查找学生信息

   ```sql
   ## 使用 distinct 关键字
   select *
   from (select distinct SId
         from SC) a
            left join Student b on a.SId = b.SId;
   
   ## 使用 group by
   select *
   from (select SId
         from SC
         group by SId) a
            left join Student b on a.SId = b.SId
   ```



## 7、查询所有学生的编号、姓名、课程总数和总分数，没有则为 null

需要根据所有的学生信息左连接成绩信息

1. 先查 SC 表中的信息，并根据 sid 进行分组，该语句作为子查询
2. 用 student 表左连接 1 中创建的临时表，获取到学生编号、姓名相关信息

```sql
select a.SId, a.Sname, b.cons, b.sum_score
from student a
         left join
     (select SId,
             count(CId) as cons,
             sum(score) as sum_score
      from SC
      group by SId) b
     on a.SId = b.SId
```



## 8、查询 【李】姓老师的数量

使用 `like`

```sql
select count(Tname)
from Teacher
where Tname like '李%'
```

