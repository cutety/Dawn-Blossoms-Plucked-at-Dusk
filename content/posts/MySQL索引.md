---
title: "MySQL索引"
date: 2022-02-28T22:07:15+08:00
draft: false
---

# 建立索引的几大原则
* 最左匹配原则，MySQL会一直匹配直到遇到范围查询(>、<、between、like)就停止匹配。
* `=`和`in`可以乱序，MySQL的查询优化器会优化成索引可以识别的格式。
* 尽量使用区别度高的列作索引，状态、性别这种不适合做索引。区别度的公式为`count(distinct col)/count(*)`，需要join的字段要求0.1以上。
* 索引列尽量不参与计算
* 尽量扩展现有的索引，不要新建索引。

# limit分页查询
分页查询如果需要查询的数据量比较大，那么limit查询的偏移量就可能会很大，造成查询速度显著降低。可以用子查询的方法来解决
```SQL
SELECT * FROM `posts` ORDER BY pid LIMIT 100000, 30
```

```SQL
SELECT * FROM `posts` WHERE PID > (
  SELECT pid FROM `posts` ORDER BY pid LIMIT 100000, 1
) LIMIT 30
```
