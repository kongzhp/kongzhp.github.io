---
layout:     post
title:      "Mastering Cassandra读书笔记（1）"
subtitle:   "Cassandra Architecture"
date:       2015-12-29 19:35:00
author:     "Pete Kong"
header-img: "img/post-bg-02.jpg"
---

# RDBMS问题

传统的关系型数据库在过去40年里取得巨大的成就，应用十分广泛。但是人们慢慢发现它的弱点：复杂的关联查询性能差，垂直扩展成本高，水平扩展困难。另外，如果你需要数据库备份，那么需要一些locking, 这样会损害系统的可用性。后来人们使用caching layer来提高性能，一些ORM已经提供了内置的缓存机制，但是这需要很大的内存消耗。

# 走进NoSQL

NoSQL数据库是解决RDBMS扩展性的问题的数据库。NoSQL提供高扩展性、高可用性，但不保证ACID：原子性、一致性、隔离性和持久性。很多NoSQL的数据库，例如Cassandra提供了另一种ACID，称为BASE： basically available, soft-state,eventual consistency.

## CAP理论

* 一致性

一致性的系统是指同一个请求在同一时间在所有的replicas会得到相同的结果，换句话说，一致性系统就是每个服务器对于每一个请求都是返回同一个结果。

* 可用性

可用性，系统在任何时候都可以工作。

* 分裂容忍（Partioning tolerance）

网络分裂是形容在分布式系统里，两个或多个子系统无法通信。分裂容忍就是一个系统可以在网络分裂的时候仍然可以工作。一个分裂容忍的例子是，一个没有中心master的实时复制系统，例如，在一个系统中，数据是在两个数据中心间拷贝，即使一个数据中心挂了，可用性也没有影响。

下图是基于CAP的各种数据库系统分类：

![CAP]({{ site.baseurl }}/img/maserCass/cap.PNG)

一旦你需要系统扩容，你可能需要水平扩容，这就意味着增加新的机器，一旦系统变成分布式，CAP就派上用场，你只能选择一致性、可用性和分裂容忍之中的两个。

* CA系统：放弃分裂容忍来保证一致性和可用性。在这个系统里，一个事务中的所有相关东西都会放到一台机器或一个系统里。这系统扩容会有严重的问题。

* CP系统：牺牲可用性。假如系统是可用的，那么数据就是一致的，一旦一个节点挂了，那么数据就不可用。一个分shard的系统就是这样的例子。

* AP系统：一个高可用、分裂容忍的系统就是一个always-on的系统，但网络分裂时，会有数据冲突的风险。这个会带来很好的用户体验，因为系统总是可用的，在一些用户场景中，不一致的数据可能很少出现。

* 最终一致系统（BASE）:系统是AP，数据有短暂时间的不一致，但一旦系统检测到不一致，它会自动修复达到最终一致。

![CAP]({{ site.baseurl }}/img/maserCass/eventualConsistent.PNG)

# Cassandra

cassandra是一个分布式，去中心化，错误容忍的，最终一致性的，线性扩容和面向列的数据存储系统。

cassandra数据结构是key-value的数据结构，称为colume-family，可以把它看做一个巨大的excel表，从2.0开始，改名为Table.

cassandra 的cluster 叫做ring,cassandra1.1和之前的版本都需要给每一个节点赋予一个token。通常token是指row key的哈希值。例如，假设现在有一个哈希算法（partitioner）,它会生成从0到127的token值，而且现在由4台物理机器组成cluster.为了负载均衡，每个机器拥有相同数量的token,因此，第一台机器负责1-32，第二个负责33-64，第三个负责65-96，第四个负责97-127和0。此时,cluster就像一个ring一样：

![RING]({{ site.baseurl }}/img/maserCass/ring.PNG)

cassandra1.1及以前的版本中，每添加一个节点就需要给它一个初始的token值，如果移除一个节点就要reset所有token。为解决这些麻烦，从1.2开始引入了virtual node。下图是16个vnodes在4个物理机上的分布：

![vnode]({{ site.baseurl }}/img/maserCass/virtualNode.PNG)

每一个node负责一个持续的token range，物理机有多个node，即负责多个不同的token range，并且由系统自动分配。使用vnode的一个好处是，如果你有一台性能比较强的机器，可以通过配置cassandra.yaml的num_tokens使得该机器承受更多的traffic。另外一个好处是，当需要data repair的时候，使用virtual node技术会更快，因为data repair需要创建Merkle tree，它需要比较所有replica node的数据，如果replica node存储的token范围越小，那么运行的速度就越快。而virtual node就是做到这一点。

## Cassandra如何工作

cassandra的主要组件架构如下图所示：

![component]({{ site.baseurl }}/img/maserCass/component.PNG)

Storage Layer的主类似StorageProxy,它负责处理所有请求。Messaging Layer负责节点内部通信,例如gossip。

你需要知道四种重要的数据容器，MemTable是类似哈希表的数据结构，存在于内存中。它保存实际的cell data。 SStable是MemTable的磁盘版本。当memTable满了，它就需要持久化成SSTable。Commit log是一个append-only的日志，记录发送到cassandra的所有修改操作。

### write in action

发生写操作时，客户端会连接任意一台机器，被连接上的节点叫做coordinator node。当节点收到写请求，它把请求委托给StorageProxy，它的作用是找到所有负责这个写请求的节点，并把请求发送给它们。coordinator不需要等到所有节点返回，它只需根据配置的repliation strategy，有consistency Level的reponse 即可返回写成功。

![write]({{ site.baseurl }}/img/maserCass/writeInAction.PNG)

以下展示了写操作的所有机制：

1. 如果failure Detector检测到没有满足consistency Level数量的live节点，则返回失败。
2. 当有节点超时仍然没有reponse,StorageProxy会把local hint写到本地，当该节点重新活跃后，再replay该请求，这叫做hinted handoff。
3. 如果一个replica节点在其他data center, 它不会把请求发送到那个DC的所有replica node上，它只会发送给那个DC的其中一个节点,带着一个header，指引它把请求发送给该DC的所有其他repliaca node上。
4. 当数据到达某个节点，它会被写到commit log，然后写到memTable.
5. 如果memTable满了，数据会被flush到SSTable，如果达到一点数量的SSTables，compaction进程将会启动，把SSTables merge成一个。

### read in action

与写操作相似，客户端会先找到一个coordinator node,然后它会根据配置的snitch strategy（SimpleSnitch, PropertyFileSnitch, GossipingPropertyFileSnitch...）对存储了该data的节点进行排序，找到最近的Node，发送read请求给它，它返回完整的数据，根据consistency Level, 也把read请求发给满足consistency level数量的节点，但这些节点只需返回数据的摘要。coordinator会比较数据的摘要，假如有冲突，则需要有冲突的节点返回完整的数据，找到时间戳最新的数据，把持有旧数据的节点更新。

当一个节点收到读请求后，它会先从内存里的MemTable找是否存在那个row key，假如没有，就再找磁盘里的SSTables，每个SSTable都有一个在内存里的bloom filter，bloom filter用于检测某个row key是否**可能**存在于对应的SSTable中。除了针对row key的bloom filter, 还有针对每一行的bloom filter,它用于检测该行(row)的某一列（colomn）是否存在于该SSTable中。Cassandra会从younger到older的顺序寻找SSTable，并利用Index file找到每一列的offset和该行关联的bloom filter(检测colomn name)。

![read]({{ site.baseurl }}/img/maserCass/readInAction.PNG)

### commit log/MemTable /SSTable

下图展示了 Commit log/ MemTable/ SSTable之间的关系：

![commit]({{ site.baseurl }}/img/maserCass/commit.PNG)

MemTable是一个colomn familiy在内存的存放方式，可以认为是缓存，它用row key进行排序。不像commit log，它不是append-only,它没有重复的数据。以下是例子：

Write 1: {k1: [{c1, v1}, {c2,v2}, {c3, v3}]}

In CommitLog (new entry, append):

{k1: [{c1, v1},{c2, v2}, {c3,v3}]}
In MemTable (new entry, append):

{k1: [{c1, v1}, {c2, v2}, {c3,v3}]}

Write 2: {k2: [{c4, v4}]}

In CommitLog (new entry, append):

{k1: [{c1, v1}, {c2, v2}, {c3,v3}]}

{k2: [{c4, v4}]}

In MemTable (new entry, append):

{k1: [{c1, v1}, {c2, v2}, {c3,v3}]}

{k2: [{c4, v4}]}

Write 3: {k1: [{c1, v5}, {c6,v6}]}

In CommitLog (old entry, append):

{k1: [{c1, v1}, {c2, v2}, {c3,v3}]}

{k2: [{c4, v4}]}

{k1: [{c1, v5}, {c6, v6}]}

In MemTable (old entry, update):

{k1: [{c1, v5}, {c2, v2}, {c3,v3}, {c6, v6}]}

{k2: [{c4, v4}]}

Bloom filter就像石蕊测试(检测某种化学元素是否存在)，检测某个row是否存在于某个SSTable中。但它是false-positive的，就是说，如果测试结果是true，那么说明可能存在，但是结果是false，那么数据必定不存在于该SSTable。Bloom Filter可以看做是一个长度为L的bit数组，初始是所有元素为0，并且有k个预定义的哈希函数与之关联。

![Bloom]({{ site.baseurl }}/img/maserCass/bloomFilter.PNG)

当向某个SSTable插入数据时，先把row key的值v作为参数传给k个哈希函数，再模除L,得出的值作为bit数组的索引，把该位置的元素设置为1。下面的伪代码展示了这个流程：

//calculate hash, mod it to get location in bit array   

arrayIndex1 = md5(v) % arrayLength

arrayIndex2 = sha1(v) %arrayLength

arrayindex3 = murmur(v) %arrayLength

//set all those indexes to 1

bitArray[arrayIndex1] = 1

bitArray[arrayIndex2] = 1

bitArray[arrayIndex3] = 1

当检测某个key是否存在于SSTable时，通过做同样的操作，找到数组里该位置的元素，看它是否为1，假如为0，那么肯定不存在，假如为1，那么可能存在，因为可能有别的key计算出来的位置是同一个。
