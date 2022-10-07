---
layout: post
title: "CMU 15-445 Fall 2019 Lecture 01 Course Introduction and the Relational Model"
date: 2022-10-01 08:35:19 +800
categories: cmu15445-fall-2019
tags: cmu15445 database
create-date: 2022-04-19
catalogue: 🌱record/literature 🎍writing/post
---
## Course Overview
CMU 15-445，主要介绍了，如何设计和实现面向磁盘的数据库管理系统。包含以下课程内容：

1. 关系型数据库
2. 存储
3. 执行
4. 并发控制
5. 故障恢复
6. 分布式数据库
7. 嘉宾演讲

## Database
数据库，是有内部联系的数据集合，这些数据通常抽象了真实世界的某个切面。是现代绝大多数计算机软件的核心组成部分。

当创建一个数据库，里面存储的数据是有内部关系的，而且这些有关系的数据反映的是现实世界中的某个场景。比如：
- 创建账户表、金额表，记录不同账户余额变动情况。
- 创建专辑表、作曲家表，体现不同作曲家和专辑表的关系。

那么，我们如何存储这些数据呢？

一个简单做法，使用 CSV 格式明文写在两个文件，一个文件一个实体。以专辑和作曲家为例子，可以得到，以下两个文件内容：

```text
Artists(name, year, country)
----------------------------
Wu Tang clan, 1992, USA
Notorious BIG, 1992, USA
Ice Cube, 1989, USA
```

```text
Alnum(name, artist, year)
-------------------------
"Enter the Wu Tang","Wu Tang Clan",1993
"St.Ides Mix Tape","Wu Tang Clan",1994
"AmeriKKKa's Most Wanted","Ice Cube",1990
```

现在，应用层要查找，“作曲家 Ice Cube 的出生年份”。那么需要实现以下代码：

```python
for line in file:
	record = parse(line)
	if "Ice Cube" == record[0]:
		print int(record[1])
```

这个实现成本是很高的，几乎不可复用的。要再找不同作曲家出生年份就得重写一段代码，甚至于，要找同一个作曲家的出生国家，都得改造上面的代码。所以，底层数据存储模型如果设计的过于简单，那么应用层就需要花费大量精力实现数据访问的细节。

这就是数据库设计提出的初衷，封装数据访问相关实现，从而降低应用层的维护、迁移、开发等等成本：
- 使用简单的数据结构存储对象
- 提供高级语言访问数据
- 物理存储细节交给数据库系统实现

## Database Management System
数据库管理系统是什么？粗略地说，就是定义、创建、更新以及管理上述数据库的软件系统。

常见的数据库管理系统有：MySQL、Oracle、PostgreSQL。

不过，在日常交流中，通常我们说起“数据库”的时候，实际上包含了：“数据库”（存储部分）以及“数据库管理系统”（软件系统）。

## Data Models
数据模型，是在数据库中描述数据的概念集合。

概念集合，这个词有点抽象。可以换个更通俗的描述，“方法论”。所以，上面那句话翻译翻译就是：

数据模型，是数据库系统如何存储数据的方法论。不同领域，会采用不同的格式去抽象真实世界的数据。比如以下数据模型分类：

![Data Model](https://raw.githubusercontent.com/iAInNet/stay-foolish/master/assets/01%20Course%20Introduction%20and%20the%20Relational%20Model%202022-05-03%2022.32.15.excalidraw.svg)

Relational、No SQL、Machine Learning、Obsolete/Rare 这些词汇只是数据模型的名称。在这个名称之下，是一系列概念，利用对应的概念集合描述数据，就可以归类到特定模型之下。

本课程的主要内容，是依据“Relational”这一系列概念构建的数据库，我们叫它“关系型数据库”，核心模式（Schema）是关系（Relation）。

还是上面作曲家和专辑的，转化为用关系模型进行描述，得到下图结构：

![Relational Model Example](https://raw.githubusercontent.com/iAInNet/stay-foolish/master/assets/01%20Course%20Introduction%20and%20the%20Relational%20Model%202022-05-03%2022.47.28.excalidraw.svg)

- 关系（relation）：表示实体【属性关系】的无序集合，一般也会叫做“表”
	- 比如，作曲家有三个属性（名称、出生年、出生地），聚合成一个关系（表）
- 元组（tuple）：属性关系的一种取值组合
	- 比如，作曲家（Artist）三个属性的一个取值组合（Ice Cube，1989，USA）构成了作曲家（Artist）这个关系下一条数据元组，抽象了真实世界的作曲家：“Ice Cube”。
	- NULL 可以是任何属性的取值
- 关系主键（primary key）：唯一标识一个元组
	- 自动生成的唯一正整数
		- SEQUENCE (SQL2003 标准)
		- AUTO_INCREMENT(MySQL)
- 关系外键（foreign key）：一个关系中属性指向另外一个关系的元组
	- 比如，ArtistAlbum 中的 artist_id 属性，它的取值，指向 Artist 关系的数据元组。

[关系模型](https://en.wikipedia.org/wiki/Relational_model)是 Ted Codd 于1970年提出。核心思想是将，现实中的实体转换为关系，并将数据的各种操作抽象为关系代数运算。每一个操作都会使用一个或者多个关系作为输入，并输出一个新的关系。这些操作还可以链接起来，从而组合出来更加复杂的运算。

再将这些关系代数运算，使用一组更高级的语言封装。可以让不那么懂数学的人，快速写出同等含义的数学运算式子，也就演化出了一门高级查询语言： SQL 。

## Conclusion
- 数据库是普遍存在的，可以认为是当代信息化社会的基石之一也不为过。
- 关系代数定义了如何在关系型数据库中操作数据的原语

## References

1. [课件，lecture 01 introduction](https://15445.courses.cs.cmu.edu/fall2019/slides/01-introduction.pdf)
2. [CMU 15-445 视频教程： 01 - Course Introduction & Relational Model](https://www.youtube.com/watch?v=oeYBdghaIjc&list=PLSE8ODhjZXjbohkNBWQs_otTrBTrjyohi&index=1&t=5s)
3. [Relational model](https://en.wikipedia.org/wiki/Relational_model)
