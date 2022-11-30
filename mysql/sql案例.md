# 创建表

```sql
CREATE TABLE student (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `name` varchar(100) NOT NULL,
    `score` int(11) NOT NULL,
    `year` int(11) NOT NULL,
    PRIMARY KEY (`id`)
) ENGINE = InnoDB DEFAULT CHARSET=utf8mb4;
```
# 插入测试数据
```sql
insert student(id, name, score, year) values
    (null, 'ZhangSan', 88, 2018),
    (null, 'ZhangSan', 80, 2019),
    (null, 'ZhangSan', 90, 2020),
    (null, 'LiSi', 98, 2018),
    (null, 'LiSi', 77, 2019),
    (null, 'LiSi', 88, 2020),
    (null, 'WangWu', 88, 2018),
    (null, 'WangWu', 60, 2019),
    (null, 'WangWu', 99, 2020);
```

```sql
select * from student as s1
left join (select max(s2.score) as max_score,s2.year from student as s2 group by year) as s3
on s1.year=s3.year and s1.score = s3.max_score
where s3.year is not null 
;
```