---
title: REDIS原理及其相关
date: 2019/03/18
update: 2019/03/20
categories: "框架"
---

#### 前言

最近经常被人问到redis原理相关的问题，本着好记性不如烂笔头的观点，整理如下：

#### REDIS基本类型

其实，REDIS原理是一个很泛泛而谈的话题，我们只能从redis的工作模式，redis的优缺点等方面去分析。

<!-- more -->

首先，REDIS是一个key-value的存储系统，其中，key类型为非二进制安全的字符类型（not binary safe strings），value类型则有多种，如string、list、set、hash、sorted set。

- string类型：string是redis最基本的类型，是二进制安全的，如果只用string，redis可以看作加上持久化功能的memcached；
- list类型：list类型是一个每个元素都是string类型的双向链表，可以通过pop、push操作从链表的头部和尾部添加删除元素，list既可以用作栈，也可以用作队列；
- set类型：set是string类型的无序不重复集合，最多可包含2^32-1个元素，通过hash table实现；
- sorted set类型：string类型的有序不重复集合，与set不同的是每个元素都会关联一个double类型的score，sorted set由skip list和hash table混合实现，当添加一个元素时，该元素到score的映射被添加到hash table中，同时score到元素的映射被添加到skip list之中并按照score排序，实现有序；
- hash类型：是一个string类型的field和value的映射表，可以用于存储对象，占用内存少。

#### REDIS工作方式

redis通过将整个数据库加载到内存中运行，具有极其出色的性能，同时定期通过一定的持久化策略将数据持久化到硬盘中进行保存。redis支持多种数据类型，使其可以实现多种功能，如list可用于FIFO的双向链表，可用于轻量级的高性能消息队列，set可用于tag系统，用户喜好标记等。

redis的主要缺点是因为其运行在内存中，受到物理内存容量大小的限制，不能用于海量数据的场景。





