---
title: REDIS原理及其相关
date: 2019/03/18
update: 2019/03/20
---
&emsp;&emsp;最近经常被人问到redis原理相关的问题，本着好记性不如烂笔头的观点，整理如下：

## REDIS原理

&emsp;&emsp;其实，REDIS原理是一个很泛泛而谈的话题，我们只能从redis的工作模式，redis的优缺点等方面去分析。

&emsp;&emsp;首先，REDIS是一个key-value的存储系统，其中，key类型为非二进制安全的字符类型（not binary safe strings），value类型则有多种，如string、list、set、hash、sorted set。

- string类型：string是redis最基本的类型，是二进制安全的，如果只用string，redis可以看作加上持久化功能的memcached；
- list类型：list类型是一个每个元素都是string类型的双向链表，可以通过pop、push操作从链表的头部和尾部添加删除元素，list既可以用作栈，也可以用作队列；
- set类型：set是string类型的无序不重复集合，最多可包含2^32-1个元素，通过hash table实现；
- sorted set类型：string类型的有序不重复集合，与set不同的是每个元素都会关联一个double类型的score，sorted set由skip list和hash table混合实现，当添加一个元素时，该元素到score的映射被添加到hash table中，同时score到元素的映射被添加到skip list之中并按照score排序，实现有序；
- hash类型：是一个string类型的field和value的映射表，可以用于存储对象，占用内存少。

### REDIS工作方式



