---
layout: post
title: "CMU 15-445 Fall 2019 Lecture 02 Advanced SQL"
date: 2022-10-03 08:49:09 +800
categories: cmu15445-fall-2019
tags: cmu15445 database
create-date:  2022-05-03
catalogue: 🌱record/literature 🎍writing/post
---
## Relational Languages
[SQL](https://en.wikipedia.org/wiki/SQL)（**Structured Query Language**）是一门[领域特定语言](https://en.wikipedia.org/wiki/Domain-specific_language)，被设计专门用于管理关系数据库管理系统（[RDBMS](https://en.wikipedia.org/wiki/Relational_database#RDBMS)）中的数据，或者被用于流处理关系数据流管理系统（[RDSMS](https://en.wikipedia.org/wiki/Relational_data_stream_management_system)）中的数据。

从 SQL 标准发展历程来看，从1970年 [Ted Codd](https://en.wikipedia.org/wiki/Edgar_F._Codd) 首次发布到2016年增加了 JSON 和 Polymorphic Table Functions （多态表函数）特性，这个高级语言一直在发展中。当然，实践场景中，经常是大数据库系统设计公司，先支持某一特性，再反推 SQL 标准纳入某一特性。

![SQL History](https://raw.githubusercontent.com/iAInNet/stay-foolish/master/assets/02%20Advanced%20SQL%202022-05-04%2022.15.48.excalidraw.svg)

关系型数据库语言，实际上是一个集合，包含以下操作：
- 数据查询语言（DQL），比如查询数据元组
- 数据操作语言（DML），比如增删改数据元组
- 数据定义语言（DDL），比如创建表格的Create、修改字段的Alter，改变的是表结构
- 数据控制语言（DCL），比如权限控制、授权等等，主要是数据库管理员用于保证安全相关

当然还提供以下特性：
- 视图定义（View definition）
- 完整性约束和引用约束（Integrity Constraints & Referential Constranits）
- 事务（Transactions）

应用层只需要按照 SQL 语法要求编写，接下来只需要提交给数据库系统即可。是数据库系统会负责高效地执行语句，查询优化会重新编排操作，从而生成更加高效的执行计划，执行成功后将结果给应用层。

**需要注意**，SQL 是基于多集运算（[multiset](https://en.wikipedia.org/wiki/Multiset)，允许元素重复）的语言，而不是基于集合运算（[set](https://en.wikipedia.org/wiki/Set_(mathematics))，不允许元素重复）。这就意味着，数据库默认返回的数据是没有顺序的、也有可能出现重复。想要实现顺序和剔除重复，需要 SQL 语句中明确使用 Order By 或者 DISTINCT 等等语法。

这系列课程，主要讲的是数据库系统的设计和实现，所以 SQL 语法的内容，只有这一节。通过介绍了 SQL 所能支持的基础和高级语法，能够在后续学到索引结构、查询优化、故障恢复的时候，明白为了实现这样的语法特性，数据库管理系统要设计多少技术细节。

## Today's Agenda
- Aggregations + Group By
- String / Date / Time Operations
- Output Redirection + Control
- Nested Queries  
- Window Functions
- Common Table Expressions

## Example Database
接下来的 SQL 示例，都是使用三张表：
- 学生表，student(sid, name, login, gpa)
- 成绩表，enrolled(sid, cid, grade)
	- sid -> student.sid
	- cid -> course.cid
- 课程表，course(cid, name)

结构和数据如下所示：

```text

学生表，student

+-------+--------+------------+------+------+
| sid   | name   | login      | age  | gpa  |
+-------+--------+------------+------+------+
| 53655 | Tupac  | shakur@cs  |   26 |  3.5 |
| 53666 | kanye  | kayne@cs   |   39 |  4.0 |
| 53688 | Bieber | jbieber@cs |   22 |  3.9 |
+-------+--------+------------+------+------+

成绩表，enrolled

+-------+------+-------+
| sid   | cid  | grade |
+-------+------+-------+
| 53666 |  445 | C     |
| 53688 |  721 | A     |
| 53688 |  826 | B     |
| 53655 |  445 | B     |
| 53666 |  721 | C     |
+-------+------+-------+

课程表： course

+------+------------------------------+
| cid  | name                         |
+------+------------------------------+
|  445 | Database Systems             |
|  721 | Advanced Database Systems    |
|  826 | Data Mining                  |
|  823 | Advanced Topics in Databases |
+------+------------------------------+
```

⚠️ 本节课堂笔记中的 SQL 语句都是在 MySQL 8 中执行的。

## Aggregations
聚合函数，接收一组元组然后输出一个聚合计算数据。比如 AVG, MIN, MAX, SUM, COUNT 这些方法。
- 只能在 SELECT 语句中使用
- 可以同时计算多个聚合函数
- COUNT，SUM，AVG 支持 DISTINCT，允许聚合结果是剔除重复项之后计算出来的数据
- 在聚合语句中，输出非聚合的原生字段的行为是不确定的，不同数据库系统处理方式不一样。比如：
	- SELECT AVG(s.gpa), e.cid FROM enrolled AS e, student AS s WHERE e.sid = s.sid
	- Postgres 不允许这样的输出
	- MySQL 会输出一个数值，但是无法确认是什么数值
		- MySQL 5.7+ 通过设定 sql_mode = 'ansi'，可以实现和 Postgres 同样性质
	- SQLite 会输出一个数值，无法确认规则，跟 MySQL 的也不一样
	- 总结来说，不要在 SELECT 中混合聚合和非聚合的输出，行为是不可预估的。

## Group by
依据指定属性，将数据元组聚集（映射）到不同子集，然后可以在子集中聚合计算数据。

”不被聚合的数据，必须出现在 GROUP BY 语句中“。其实从子集的概念，就很好理解这句话的含义：不同子集拥有相同列，这些列就是在 GROUP BY 中指定的。而 SELECT 就是在各个子集中，独立执行相同的聚合计算，最后再把结果收集起来，返回给用户。

比如以下这个语句，按照成绩表中的课程ID聚合，然后计算每一个子集的，平均GPA。也就是，计算每一门课的平均GPA。

```sql
SELECT e.cid, AVG(s.gpa) FROM enrolled AS e, student AS s WHERE e.sid = s.sid
GROUP BY e.cid
```

以下语句中的 s.name 不允许出现在 select 语句，因为这个字段并不在 GROUP BY 中，也就不会被映射到各个子集中， SELECT 自热没办法找到在子集中找到对应字段。

```sql
-- 错误 SQL （严格模式）
SELECT AVG(s.gpa), e.cid, s.name
FROM enrolled AS e, student AS s WHERE e.sid = s.sid
GROUP BY e.cid
```

想要在聚合计算结果再过滤操作，这就是 HAVING 操作。这个过滤操作没办法放在 WHERE 语句中，是因为 SELECT 在查看完所有子集的数据之后，才计算出聚合结果。而 WHERE 是在 SELECT 过程，过滤数据，没办法对 SELECT 的结果再做过滤。所以需要引入 HAVING 语法。

```sql
-- 计算每一门课的平均GPA，但是只要 GPA 大于 3.9 的课程
SELECT e.cid, AVG(s.gpa) AS avg_gpa
FROM enrolled AS e, student AS s
WHERE e.sid = s.sid
GROUP BY e.cid
HAVING avg_gpa > 3.9;
```

## String / Date / Time Operations
### String Operations
字符串（String）操作，是数据库系统封装好的，一系列处理字符串类型的方法，可以在 SELECT 和 WHERE 中使用。

#### 字符串大小写问题
MySQL 是大小写不敏感的，也就是说，“xYz” 和 “xyz” 两个取值对于 MySQL 来说是一样的。而SQL-92（标准）、Postgres、SQLite、DB2、Oracle 都是大小写敏感的。

实践上，不要依赖 MySQL 这一特性做判断操作，比如业务上想要不区分大小写的找到相等记录。那么可以将字符串都转化为大写再比较。

如果想要在 MySQL 中严格区分大小写，那就是创建表的时候，为字段添加 Binary 属性。SELECT 的时候，同样使用字段的 Binary 属性进行比较。原理是，对于 CHAR、VARCHAR 和 TEXT 类型，BINARY 属性，可以将字符串转为二进制编码，那么“xYz”和“xyz”的二进制编码自然是不同的了。

#### 单引号还是双引号
MySQL 和 SQLite 是单引号和双引号都支持，而 SQL-92（标准）、Postgres、DB2、Oracle 只支持单引号。

个人建议，实践上，可以都只采用单引号。

#### LIKE 匹配运算
”LIKE“ 被用来做字符串匹配
- `%` 用来匹配任意字符串（包括空格）
- `_` 匹配任意的一个字符

### Date/Time Operations
时间（Date/Time）属性操作，数据库同样封装，一系列操作 Date/Time 属性的方法，可以在 SELECT 和 WHERE 中使用。

不同数据库系统，提供的基础方法，差异很大。所以没有一个标准的写法，根据所用的数据库，需要的时候看官方手册即可。

小常识：全世界最流行的数据库系统是哪一个？ MySQL 天下第（被打断），是的，正是在下： SQLite 。SQLite 因为轻量级的特性，能被集成到移动应用、桌面应用中，提供基础的数据操作服务，最终会被部署在大部分的移动或者桌面设备上。而且，SQLite 源码是 Public Domain ，没有任何知识产权保护，完全免费。

## Output Redirection + Control
### Output Redirection
可以将查询结果写入到其他表中：
- 表格必须**没有**被定义
- 表格将会有和输入的数据，相同的字段名称和类型

```sql
-- SQL-92，使用 INTO 语句
SELECT DISTINCT cid INTO CourseIds FROM enrolled;

-- MySQL，使用 CREATE 语句，将 SELECT 放在括号里
CREATE TABLE CourseIds (SELECT DISTINCT cid FROM enrolled);
```

另外一种，将数据元组插入到其他表中：
- 和上面一个方式不同的是，要插入的那张表一定要存在，并且 SELECT 产生的列要和目标表的列完全匹配。名称、类型、列个数，完全匹配，才能成功。
- 如何处理重复的元组，不同数据库系统有不同的选项和语法，这方面 SQL 标准也没有明确说明。各家做法不一，有的报错回滚，有的跳过重复继续执行。

```sql
-- SQL-92
INSERT INTO CourseIds (SELECT DISTINCT cid FROM enrolled);
```

### Output Control
可以对 SELECT 结果，进行排序：`ORDER BY <column*> [ASC|DESC]` 。

可以对 SELECT 结果，限定获取数量： `LIMIT <count> [OFFSET]`。可以通过 OFFSET 设定返回一个范围：`[offset, offset+limit)`。

## Nested Queries
一个语句嵌套其他语句，这样语句通常很难优化，嵌套的语句可以出现另外一个语句的几乎任何地方。比如：

```sql
-- 查询：参与 15-445 课程的学生的名称
SELECT name FROM student
WHERE sid IN (
	SELECT sid FROM enrolled
	WHERE cid = '15-445'
)
```

- ALL关系，必须和子查询中的所有行匹配
- ANY关系，至少和子查询中的一行匹配
- IN关系，等价于 ANY
- EXISTS关系，用于检查子查询是否至少返回一行数据，不为空是 TRUE，为空就是 FALSE。然后这个 TRUE 和 FALSE 的结果，会用来过滤外查询的数据， TRUE 就留下， FALSE 就剔除。
	- 当然加上 NOT，就跟上面逻辑相反。

```sql
-- ANY关系，只要 sid 和子查询的返回集中任何一条匹配，就符合条件
SELECT * FROM student
WHERE sid = ANY (
	SELECT sid FROM enrolled
	WHERE cid = '15-445'
)
```

```sql
-- ALL关系，比嵌套查询中任何记录都要大的学生ID，那就是查找最大学生ID
SELECT sid, name FROM student 
WHERE sid >= ALL(
	SELECT sid FROM enrolled
)

-- 当然可以通过 MAX 改造嵌套查询，那么可以得到以下语句
SELECT sid, name FROM student 
WHERE sid IN (
	SELECT MAX(sid) FROM enrolled
)
```

```text
输出结果：

+-------+--------+
| sid   | name   |
+-------+--------+
| 53688 | Bieber |
+-------+--------+
```

```sql
-- EXISTS关系，在课程表中找到这么一个课程，这个课程在成绩表中不存在
SELECT * FROM course
WHERE NOT EXISTS (
	SELECT * FROM enrolled
	WHERE course.cid = enrolled.cid
)
```

NOT EXISTS 的实现流程：

course 表中的每一行，都代入到 EXISTS 子查询进行验证，在 enrolled 表存在的该课程（不管成绩表存在几条），当然最终要的是 NOT EXISTS，所以就变成了在 enrolled 表**不存在**该课程。最终结果，就只留下了： 课程 823 ，因为它在成绩表中没有记录。

```text
输出结果：

+------+------------------------------+
| cid  | name                         |
+------+------------------------------+
|  823 | Advanced Topics in Databases |
+------+------------------------------+
```

## Window Functions
在一组相关联的数据元组上，执行滑动计算（窗口函数）运算。就像聚合计算那样，对一组数据进行运算，但是区别是并不会聚合成一个输出值，窗口函数是为每一个数据元组计算出一个新的属性值。格式如下所示：

```sql
SELECT ..., FUNC_NAME() OVER() FROM tablename
```

- OVER 表明了，如何切分数据元组，也可以包含如何排序
	- PARTITION BY  ：按照什么分组
	- ORDER BY  ： 分组内按照什么排序
- FUNC_NAME 是，一般称之为是窗口函数，具体有：
	- ROW_NUMBER() ： 在指定一分组下，当前行的行编号
	- RANK() ：在指定一分组下，当前行的顺序编号
		- 这个要结合 ORDER BY 才有意义，要明确按照什么属性排序，顺序编号才有意义。

```sql
-- 不指定分组方式，计算行号。
SELECT *, ROW_NUMBER() OVER() as row_num FROM enrolled ORDER BY cid
```

 OVER() 没有指定分组方式，那就是在表中所有数据元组上计算行编号（ROW_NUMBER）。
 
```text
输出结果：

+-------+------+-------+---------+
| sid   | cid  | grade | row_num |
+-------+------+-------+---------+
| 53666 |  445 | C     |       1 |
| 53655 |  445 | B     |       2 |
| 53688 |  721 | A     |       3 |
| 53666 |  721 | C     |       4 |
| 53688 |  826 | B     |       5 |
+-------+------+-------+---------+
```

```sql
-- 按照课程ID分组，然后计算行编号
SELECT cid, sid, ROW_NUMBER() OVER (PARTITION BY cid)
FROM enrolled
ORDER BY cid
```

按照 cid 分组就得到了 445、721、826 三个分组，然后执行窗口函数 ROW_NUMBER， 计算行号。要注意，是在一个个分组内计算的，也就得到了以下结果：

```text
输出结果：

+------+-------+--------------------------------------+
| cid  | sid   | ROW_NUMBER() OVER (PARTITION BY cid) |
+------+-------+--------------------------------------+
|  445 | 53666 |                                    1 |
|  445 | 53655 |                                    2 |
|  721 | 53688 |                                    1 |
|  721 | 53666 |                                    2 |
|  826 | 53688 |                                    1 |
+------+-------+--------------------------------------+
```

接下来，来查找成绩表中，各科排名第一的学生信息。拆成两步来展示来实现：
- 先按照课程ID分组，然后按照成绩排名，这样就得到每个课程各自的成绩排名
- 然后将以上语句嵌套在另外一个查询语句中，并且只选取排名第一的记录

```sql
-- 先按照课程ID分组，然后按照成绩排名，这样就得到每个课程各自的成绩排名
SELECT *, RANK() OVER(PARTITION BY cid ORDER by grade DESC) AS rank FROM enrolled
```

```text
输出中间结果：

+-------+------+-------+----+
| sid   | cid  | grade | rk |
+-------+------+-------+----+
| 53666 |  445 | C     |  1 |
| 53655 |  445 | B     |  2 |
| 53666 |  721 | C     |  1 |
| 53688 |  721 | A     |  2 |
| 53688 |  826 | B     |  1 |
+-------+------+-------+----+
```

```sql
-- 然后将以上语句嵌套在另外一个查询语句中，并且只选取排名第一的记录
SELECT ranking.* FROM
(SELECT *, RANK() OVER(PARTITION BY cid ORDER by grade DESC) AS rk FROM enrolled) AS ranking
WHERE ranking.rk = 1
```

```text
最终结果：

+-------+------+-------+----+
| sid   | cid  | grade | rk |
+-------+------+-------+----+
| 53666 |  445 | C     |  1 |
| 53666 |  721 | C     |  1 |
| 53688 |  826 | B     |  1 |
+-------+------+-------+----+
```

## Common Table Expressions
CTE，通用表表达式。封装一段 SQL 语句作为一个方法，这个方法有名称和参数，然后可以直接被更大的语句使用。

从底层数据维度去思考，可以认为，一段 CTE 语句执行后生成了一张【临时表】。这张临时表，有名称，有列名。对应了表达式的方法名称和参数。然后其他语句，可以通过这个名称和列名，直接使用【临时表】的数据。

格式如下所示：

```sql
WITH cteName AS (
	SELECT 1
)
SELECT * FROM cteName
```

绑定列名的话，格式如下：

```sql
WITH cteName (col1, col2) AS (
	SELECT 1, 2
)
SELECT col1 + col2 FROM cteName
```

之前，使用嵌套语句（Nested Query）查询“至少参与一门课的所有学生中，学生ID最大那一条”，现在，使用 CTE 重写。

```sql
-- CTE，查找至少参与一门课程的所有学生中，学生ID最大的那一条
WITH cteSource(maxId) AS (
	SELECT MAX(sid) FROM enrolled
)
SELECT sid, name FROM student, cteSource
WHERE student.sid = cteSource.maxId
```

将原先的嵌套语句抽出来，封装成一个表达式 cteSource，底下的 SELECT 语句可以直接使用这个名称 cteSource 引用到临时表数据，并进行运算。

```text
输出结果：

+-------+--------+
| sid   | name   |
+-------+--------+
| 53688 | Bieber |
+-------+--------+
```

那么，通用表表达式做到哪些子查询做不到的事情吗？

CTE 是一个表达式，所以可以递归调用。而子查询，没办法实现循环的效果，只能显示地手写嵌套层级（或者在程序中拼接构造）。

加上 RECURSIVE （WITH RECURSIVE...AS），可以实现递归调用。基本结构如下：

```sql
-- 递归调用 cteSource，生成 couter 序列，直到大于等于 10 为止
-- UNION ALL 不删除重复项地合并数据结果，当然在这个 case 中，跟 UNION 效果一样，因为数据就没有重复项
-- 要注意设定 counter 的终止条件，< 10，要不然这个语句无法结束
WITH RECURSIVE cteSource (counter) AS (
	(SELECT 1)
	UNION ALL
	(SELECT counter + 1 FROM cteSource WHERE counter < 10)
)
SELECT * FROM cteSource
```

```text
输出结果：

+---------+
| counter |
+---------+
|       1 |
|       2 |
|       3 |
|       4 |
|       5 |
|       6 |
|       7 |
|       8 |
|       9 |
|      10 |
+---------+
```

还是结合应用场景来理解，会更简单一些。比如，一个常见的应用场景是，在一个不限制分类级别的数据库表中，拉取一个大类下所有层级的子类别的话。

表结构和数据如下，删除其他无关层级的业务数据：

```text
+----+--------+
| id | parent |
+----+--------+
|  1 |      0 |
|  2 |      0 |
|  3 |      1 |
|  4 |      1 |
|  5 |      2 |
|  6 |      2 |
|  7 |      3 |
|  8 |      3 |
|  9 |      4 |
+----+--------+
```

![Nested-Level](https://raw.githubusercontent.com/iAInNet/stay-foolish/master/assets/02%20Advanced%20SQL%202022-09-30%2016.09.06.excalidraw.svg)

现在，想要获取【1】这个类别下的，所有层级子类别。但是不清楚有多少层级，这时候就可以利用递归调用，一直递归查找 parent 等于上一层 id 的数据，直到最后一层。

```sql
with recursive cteLevel (id) as (  
    select id from category_level where id = 1  
    union all  
    select c.id from category_level c, cteLevel where c.parent = cteLevel.id  
)
select id from cteLevel;
```

```text
输出结果：

+------+
| id   |
+------+
|    1 |
|    3 |
|    4 |
|    7 |
|    8 |
|    9 |
+------+
```

## Conclusion
SQL 是一个一直发展的语言。如果可以的话，总是使用一个语句解决所有计算。

## References
- [课件，lecture 02 advanced sql](https://15445.courses.cs.cmu.edu/fall2019/slides/02-advancedsql.pdf)
- [CMU 15-445 视频教程：02 - Advanced SQL](https://www.youtube.com/watch?v=6VCHuLqfmV8&list=PLSE8ODhjZXjbohkNBWQs_otTrBTrjyohi&index=2)
- [SQL](https://en.wikipedia.org/wiki/SQL)
- [领域特定语言](https://en.wikipedia.org/wiki/Domain-specific_language)
- [RDBMS](https://en.wikipedia.org/wiki/Relational_database#RDBMS)
- [RDSMS](https://en.wikipedia.org/wiki/Relational_data_stream_management_system)
- [Ted Codd](https://en.wikipedia.org/wiki/Edgar_F._Codd)
- [multiset](https://en.wikipedia.org/wiki/Multiset)
- [set](https://en.wikipedia.org/wiki/Set_(mathematics))
