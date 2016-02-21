---
layout:     post
title:      "Mastering Cassandra读书笔记（2）"
subtitle:   "Effective CQL"
date:       2016-01-11 19:35:00
author:     "Pete Kong"
header-img: "img/post-bg-02.jpg"
---

# Cassandra data model

数据在Cassandra里以嵌套的hash map形式存储。准确来说，它是分布式的嵌套的hash map,外层map是有序的分布在各个机器上，内存则存储于单台机器上。

![dataModel]({{ site.baseurl }}/img/maserCass2/dataModel.PNG)

Cassandra的核心的数据结构是column-family(为兼容CQL3，改名为table)和Cell， 这些entity的容器叫做Keyspace。cell是Cassandra data model里的最小的单元。column-family包含着cells。cell本质是一个key-value对，key是cell name, value是cell value。cell对外表现为由cell name/cell value/timestamp组成的三元组，timestamp用作在read repair过程中解决冲突，最新的cell value胜出。所以client和server之间的时间同步十分重要。

### Counter Column

Counter Column是用于计数的特殊的cell,它不能与其他类型的column共同存在于同一个table ,它所在的table的其他column类型必须也是Counter.

Counters需要很严格的一致性，因此对counter执行update操作时，cassadra会先执行一次read，保证所有repliaca一致。因此我们可以利用这个特点对counter做write操作，只需把写一致性的level设置为ONE，因为level越低，读写操作的性能越高。

### column familiy

在CQL3语法里，column familiy和table是交叉使用的。在旧版本的cassadra中，column familiy是table旧叫法，而且它更能提现cassadra的数据结构。column familiy是由row构成，每一个row是一个key-value pair。row的key叫做row key,row 的value是有序的cells集合。在内部，每个column-family存储于单独的文件，column-family之间没有关联关系，因此需要应用程序自己维护他们之间的关系。尽管在CQL3中，它叫做table，但它与关系型数据库的table有很大不同，它可以是个schema-free的map。例如下面的例子，它是一个动态的column familiy, row key是日期，每一个column代表一个城市，value是当天访问该城市的人数。

![city]({{ site.baseurl }}/img/maserCass2/city.PNG)

你可以定义一个与关系型数据库行为类似的column-family,也可以定义一个key-value序列的Key-value pairs，前者称为static或narrow column-family,后者叫做dynamic或wide column-family。dynamic column-family在CQL3里没有过多的提及，但是当你使用compound key创建TBALE时，你就创建了一个dynamic column-family。

下面的例子使用cassandra-cli展示了static和dynamic column-family内在实现的不同。

创建static column-family:

    # Using cqlsh
    cqlsh:mastering_cassandra>CREATE TABLE demo_static_cf (id int PRIMARY KEY, name varchar,age int);
    cqlsh:mastering_cassandra>INSERT INTO demo_static_cf (id,name, age) VALUES ( 1, 'Jennie',39);
    cqlsh:mastering_cassandra>INSERT INTO demo_static_cf (id,name, age) VALUES ( 2,'Samantha', 23);
    cqlsh:mastering_cassandra>SELECT * FROM demo_static_cf;
    id | age | name
    ----+-----+----------
    1 | 39 | Jennie
    2 | 23 | Samantha
    (2 rows）

    # View using cassandra-cli
    [default@mastering_cassandra] LIST demo_static_cf;
    Using default limit of 100
    Using default cell limit of 100
    -------------------
    RowKey: 1
    => (name=, value=,timestamp=1410338723286000)
    => (name=age, value=00000027,timestamp=1410338723286000)
    => (name=name,value=4a656e6e6965,timestamp=1410338723286000)
    -------------------
    RowKey: 2
    => (name=, value=,timestamp=1410338757669000)
    => (name=age, value=00000017,timestamp=1410338757669000)
    => (name=name,value=53616d616e746861,timestamp=1410338757669000)
    2 Rows Returned.
    Elapsed time: 197 msec(s).

从上面的例子看出，静态的column-family是不同的row key 是分配到不同的行里，跟我们之前的理解是一样。但是如果是使用compound key的table存储方式则不一样：

    # Using cqlsh
    cqlsh:mastering_cassandra>CREATE TABLE demo_wide_row (idtimestamp , city varchar, hitscounter, primary key(id, city))
    cqlsh:mastering_cassandra>UPDATE demo_wide_row SET hits =hits + 1 WHERE id ='2014-09-04+0000' AND city ='NY';
    cqlsh:mastering_cassandra>UPDATE demo_wide_row SET hihits =295/1061hits + 5 WHERE id ='2014-09-04+0000' AND city ='Bethesda';
    cqlsh:mastering_cassandra>UPDATE demo_wide_row SET hihits =hits + 2 WHERE id ='2014-09-04+0000' AND city ='SF';
    cqlsh:mastering_cassandra>UPDATE demo_wide_row SET hits =hits + 3 WHERE id ='2014-09-05+0000' AND city ='NY';
    cqlsh:mastering_cassandra>UPDATE demo_wide_row SET hits =hits + 1 WHERE id ='2014-09-05+0000' AND city ='Baltimore';296/1061
    cqlsh:mastering_cassandra>SELECT * FROM demo_wide_row;
    id                        |city       | hits
    --------------------------+-----------+------
    2014-09-05 05:30:00+0530  |Baltimore  | 1
    2014-09-05 05:30:00+0530  | NY        | 3
    2014-09-04 05:30:00+0530  |Bethesda   | 5
    2014-09-04 05:30:00+0530  | NY        | 1
    2014-09-04 05:30:00+0530  | SF        | 2
    (5 rows)

大家可能期望将会有5行数据存储在数据库里，但实际上只有两行：

    [default@mastering_cassandra]LIST demo_wide_row;
    Using default limit of 100
    Using default cell limit of 100
    -------------------
    RowKey: 2014-09-05 05:30+0530
    => (counter=Baltimore, value=1)
    => (counter=NY, value=3)
    -------------------
    RowKey: 2014-09-04 05:30+0530
    => (counter=Bethesda, value=5)
    => (counter=NY, value=1)
    => (counter=SF, value=2)

    2 Rows Returned.
    Elapsed time: 194 msec(s).

当你使用compound key创建table时，compound key里的第一个key将作为row key, 其他将作为cell name。由于每一行的cell都是以cell name作为排序，因此可以使用以下查询：

    cqlsh:
    mastering_cassandra>SELECT * FROM demo_wide_row WHERE id = '2014-09-04+0000' AND city > 'Baltimore' AND city <'Rockville';
    id                        |     city | hits
    --------------------------+----------+------
    2014-09-04 05:30:00+0530  | Bethesda | 5
    2014-09-04 05:30:00+0530  | NY       | 1
    (2 rows)

当然，创建wide row table不是使用范围查询的唯一方法，也可以使用secondary index。那什么时候使用wide row呢？通常是time-series的数据，例如Twitter feed或者facebook timeline,你可以使用user-id作为row key(compound key的第一个key), timestamp作为cell name(即compound key里的第二个key), cell value则是真正的数据（a tweet or timeline item）。又或者是传感器数据，传感器ID作为row key, timestamp作为cell name, 传感器的值作为cell value。

## secondary index

Cassandra不像关系型数据库一样可以对任何字段进行条件查询，但可以创建secondary index来实现这一点。但创建索引是有代价的： 索引是在额外的column familiy里，每次更新索引列都需要额外的操作维护索引表。Cassandra通过创建一个反向索引存储在一个索引表里。你可以把索引表看做是一个非常大的row，每一个cell都有一个任意的名字。索引表把母表的primary keys的value作为cell name,把母表的索引列的值作为自己的primary key。下面的例子里，userid是主键，city是索引列。

母表：

    user_id    | city
    -----------+---------------------------
             1 | Cape Town
             2 | Kabul
             3 | Mogadishu
             4 | Cape Town
             5 | Baghdad
             6 | Kabul

索引表：

    Baghdad   | 5 |
    –------------------+-----------------------------------------
    Cape Town | 1 |  4 |
    –------------------+-----------------------------------------
    Kabul     | 2 |  6 |
    –------------------+-----------------------------------------
    Mogadishu | 3 |
    –------------------+-----------------------------------------

因此，索引表基本上是一个wide rows。索引表是保存在本地，这意味着在X节点上的索引表只基于存在于X节点的母表的row的数据上创建索引，所以每次对索引列进行条件查询，都会到每个Node上查询索引表。因此，创建索引需要注意一下几点：

* 避免高基数的列（high cardinality）：cardinality是指唯一值的数量。在前面的例子里，我们大概有40,000个城市，如果你有4千万个用户，那大概每个城市有1000个users。如果查询一个城市里的所有user, 假设有25个node, 则需要对25个node进行一次read，读取索引表，然后将有1000次read，从母表读取user。这样看起来比较合理：读取1000行的结果需要1025次read。city对user而言是low cardinality。

但如果是high cardinality的情况，例如使用用户的电话号码作为索引，每个用户都有唯一的电话号码，因此搜索一个电话号码，我们只得到一个user，所以为了查找一条记录，我们需要有26次read。这时候应该创建另一个表，让应用程序维护索引。这样查找一条记录，只需两次read。

* 极端low cardinality的列：例如布尔类型的列，如果对布尔类型的列创建索引，那么每个节点都会有两个很巨大的row，一个row key是true, 一个row key是false。如果true/false是平均分配，对于一个有4千万的表，查询等于true或false的记录，结果将会有2千万个，这样会搞垮服务器

* 频繁update或删除的列。每当删除一个列，cassadra会把它标记一个tombstone,当tombstone数量超过配置的值（默认是100,000cells），cassadra会查询失败。
