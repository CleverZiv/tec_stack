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

## 9、查询学习过【张三】老师课程的同学的信息

1. Course 中获得 cname = '张三' 的 cid
2. sc 中获得 cid 为 1 中结果的 sid
3. 根据 2 中的 sid 在 student 中获取同学的信息

```sql
select *
from Student
where SId in (select SId
              from SC
              where CId in (select CId
                            from Course
                            where TId in (select TId
                                          from Teacher
                                          where Tname = '张三')));
```

## 10、查询没有学完所有课程的同学的信息

1. 查询课程总数
2. 查询成绩表中课程数量小于课程总数的 sid
3. 根据 2 中的 sid 在 student 表中查询结果

```sql
select *
from Student
where SId in (select SId
              from SC
              group by SId
              having count(CId) < (select count(1)
                                   from Course));
```

## 11、查询至少有一门课与学号为 01 的同学所学课程相同的同学的信息

1. 查询 01 同学的所有课程
2. 只要上的可能中有 1 中查出的课程，就满足条件
3. 对结果去重

```sql
select distinct b.*
from sc a
         inner join student b on a.SId = b.SId
where a.CId in (select CId from SC where SId = '01')
```

## 12、查询和 01 同学学习的课程完全相同的同学

- 如何保证和 01 同学学习的课程完全相同？
  - 没有学 01 学习的课程内之外的课程
  - 学习的课程数量和 01 的课程数量一致

1. 先计算 01 同学学了哪些课程
2. 再找出哪些同学学的课程里有非 1 中的结果
3. 除下2中的记录后，以 sid 分组，查询课程数量与 01 一致的 sid

```sql
select SId
from sc
where SId not in (select SId
                  from SC
                  where CId not in (select CId from SC where SId = '01'))
group by SId
having count(1) = (select count(1) from SC where SId = '01')
```

## 13、查询没学过“张三”老师讲授的任一门课程的学生姓名

1. 找出学习过张三老师课程的 sid 记录
2. `not in`找出没学过的学生

```go
select d.Sname
from Student d
where d.SId not in (select c.SId
                    from SC c
                    where c.CId in (select b.CId
                                    from Course b
                                    where b.TId = (select a.TId
                                                   from Teacher a
                                                   where a.Tname = '张三')))
```

## 14、查询两门及以上课程不及格的同学的学号、姓名及其平均成绩

1. 筛选出所有低于60分的成绩记录
2. 以 `group by`分组，找出数量大于等于2的同学的 sid
3. 拿着 sid 去获取同学的学号、姓名和平均成绩信息

```go
select a.SId, avg(score) as avg_score
from SC a
         left join Student b on a.SId = b.SId
         inner join (select SId
                     from SC
                     where score < 60
                     group by SId
                     having count(1) > 1) c
                    on a.SId = c.SId
group by a.SId;
```

## 15、检索“01”课程分数小于60，按分数降序排列的学生信息

```go
select b.*, c.score
from Student b
         inner join (select a.SId,a.score
                     from SC a
                     where a.CId = '01'
                       and score < 60
                     order by a.score desc) c
                    on b.SId = c.SId
```

## 16、按平均成绩从高到低显示所有学生的所有课程的成绩以及平均成绩

```go
select a.*, avg_score
from SC a
         left join (select SId, avg(score) as avg_score
                    from SC
                    group by SId) b
                   on a.SId = b.SId
order by avg_score desc
```

### 17、查询各科成绩的最高分、最低分和平均分

要求以如下形式显示：课程id，课程name，最高分，最低分，平均分，及格率，中等率，优良率，优秀率。其中及格>=60，中等70~80，优良80~90，优秀>=90，要求输出课程号和选修人数，查询结果按人数降序排列，若人数相同，按课程号升序排列

分组聚合，考察条件计数

```sql
select CId,
       max(score)                                                             as 最高分,
       min(score)                                                                最低分,
       avg(score)                                                             as 平均分,
       count(1)                                                               as 选修人数,
       sum(case when score >= 60 then 1 else 0 end) / count(1)                as 及格率,
       sum(case when score >= 70 and score < 80 then 1 else 0 end) / count(1) as 中等率,
       sum(case when score >= 80 and score < 90 then 1 else 0 end) / count(1) as 优良率,
       sum(case when score >= 90 then 1 else 0 end) / count(1)                as 优秀率
from SC
group by CId
order by 选修人数 desc, CId asc;
```

注意 `case when` 的用法

### 18、按各科成绩进行排序，并显示排名，score 重复时继续排序

用到 sql 中的变量

```sql
## 变量声明 必须以 @ 开头
select @rank
或
set @rank
## 变量声明且初始化
select @rank:=1
或
set @rank:=1
```

```sql
select
SId,CId,score,@`rank`:=@`rank`+1 as 名次
from sc, (select @rank:=0) as t
order by score desc;
```

#### 18.1 按各科成绩排序，并显示排名，score 重复时合并后排序

```sql
select SId,
       CId,
       score,
       case
           when @score = score then @`rank`
           when @score:=SC.score then @`rank` := @`rank` + 1 end as 名次
from sc,
     (select @rank := 0, @score := 0) as t;
```

注意：第二个 `when`是恒成立的，起到了一个赋值的作用

### 19、查询学生的总成绩，并进行排名，总分相同时保留名次空缺

使用自定义变量

1. 先求学生总成绩
2. 自定义变量记录

```sql
select a.*,
       @rank2:=if(@sco=scos,'',@rank2+1) as 名次,
       @sco:=scos
    from
         (select SId, sum(score) as scos
    from SC
    group by SId
    order by scos desc) a,
    (select @rank2 := 0, @sco := 0) b;
```

