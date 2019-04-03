---
title: REDIS原理及其相关
date: 2019/03/18
update: 2019/03/20
categories: "框架"
---

>redis的工作原理以及redis各类默认策略

### REDIS基本类型

其实，REDIS原理是一个很泛泛而谈的话题，我们只能从redis的工作模式，redis的优缺点等方面去分析。

<!-- more -->

首先，REDIS是一个key-value的存储系统，其中，key类型为非二进制安全的字符类型（not binary safe strings），value类型则有多种，如string、list、set、hash、sorted set。

- string类型：string是redis最基本的类型，是二进制安全的，如果只用string，redis可以看作加上持久化功能的memcached；
- list类型：list类型是一个每个元素都是string类型的双向链表，可以通过pop、push操作从链表的头部和尾部添加删除元素，list既可以用作栈，也可以用作队列；
- set类型：set是string类型的无序不重复集合，最多可包含2^32-1个元素，通过hash table实现；
- sorted set类型：string类型的有序不重复集合，与set不同的是每个元素都会关联一个double类型的score，sorted set由skip list和hash table混合实现，当添加一个元素时，该元素到score的映射被添加到hash table中，同时score到元素的映射被添加到skip list之中并按照score排序，实现有序；
- hash类型：是一个string类型的field和value的映射表，可以用于存储对象，占用内存少。

### REDIS工作方式

redis通过将整个数据库加载到内存中运行，具有极其出色的性能，同时定期通过一定的持久化策略将数据持久化到硬盘中进行保存。redis支持多种数据类型，使其可以实现多种功能，如list可用于FIFO的双向链表，可用于轻量级的高性能消息队列，set可用于tag系统，用户喜好标记等。

redis的主要缺点是因为其运行在内存中，受到物理内存容量大小的限制，不能用于海量数据的场景。

### REDIS持久化策略

redis的持久化策略主要有两种，RDB模式和AOF模式，两者各有优缺。

- RDB模式是将内存中数据快照备份到.rdb格式的文件中（默认为dump.rdb）,其默认快照保存配置为：
    {% codeblock %}
    save 900 1  #900秒内如果超过1个key被修改，则发起快照保存
    save 300 10 #300秒内容如超过10个key被修改，则发起快照保存
    {% endcodeblock %}
    
- AOF模式是将每个对数据库的写命令按执行顺序添加到appendonly.aof文件中。当redis重启时，通过重新执行文件中保存的写命令来在内存中重建整个数据库的内容。有如下三种方式：
    {% codeblock %}
    appendonly yes                //启用aof持久化方式
    appendfsync always      //每次收到写命令就立即强制写入磁盘，最慢的，但是保证完全的持久化，不推荐使用
    appendfsync everysec      //每秒钟强制写入磁盘一次，在性能和持久化方面做了很好的折中，推荐
    appendfsync no          //完全依赖os，性能最好,持久化没保证
    {% endcodeblock %}



