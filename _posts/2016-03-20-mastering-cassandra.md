---
layout:     post
title:      "Mastering Cassandra读书笔记（3）"
subtitle:   "Performance Tuning"
date:       2016-03-20 19:35:00
author:     "Pete Kong"
header-img: "img/post-bg-02.jpg"
---

# Write performance

Cassandra写是很简单的操作：只需要往commit log记录操作日志，然后把记录写在内存中。这些已经没什么可以调优了，但是如果把memtable flush到磁盘，compactions，和写commit log这几件事同时发生，有可能会影响写性能，所以，如果更快的磁盘，commit logs和data files(SSTable)是放置到不同的磁盘中，这些都会提高写性能。

# Read performance

Cassandra的读操作相对复杂： 它需要从内存或者磁盘里读； 它可能需要从不同的SSTtable中收集数据片段，才能返回完整的一行数据；它可能需要从不同的node中收集数据，处理tombstones，验证数据摘要，然后返回。然而，提高读性能的普适方法是缓存：把最常访问的数据放到内存，减少磁盘访问，保持最短磁盘访问路径。另外，更快的网络，和更少的网络通信，或低级的一致性level也是可以帮助提高读性能

## 选择合适的compaction strategy

