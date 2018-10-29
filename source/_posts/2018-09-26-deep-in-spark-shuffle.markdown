---
layout: post
title:  "Inf-- 深入理解 Spark shuffle "
date:   2018-09-26 21:46:30
tags: 
  - Spark
  - Inf
---


## 概述

按以下组织方式行文，
- shuffle 概述。
- spark shuffle 的大致原理。
- spark shuffle 源码解析。
- hadoop 的shuffle 与spark shuffle的异同。
- spark shuffle 调优。


## shuffle 概述 

spark shuffle 是spark job中某些算子触发的操作，更详细点说，当rdd依赖中出现宽依赖的时候，就会触发shuffle 操作，shuffle 操作通常会伴随着不同executor/host之间数据的复制,也正因如此，导致shuffle 的代价高以及对应的复杂性。
举个最简单的例子，spark 中的算子 reduceByKey，该算子会生成一个新的rdd,这个新rdd中会对父rdd中相同key的value 按照指定的函数操作形成一个新的value。复杂的地方在于，相同的key 数据可能存在于父rdd的多个partition中，这就需要我们读取所有partition 中相同key值的数据然后聚合再做计算，这就是一个典型的shuffle操作。
产生shuffle 的算子大致有以下三类：
- reparition 操作：repartition,  coalesce等
- ByKey操作：groupByKey reduceByKey等。
- join操作：cogroup join.

shuffle 操作通常会伴随着磁盘io,数据的序列化/反序列化,网络io，这些操作相对比较耗时间，往往会成为一个分布式计算任务的瓶颈，spark 也为此花了大力气进行spark shuffle的优化。从最早的hash based shuffle 到 consolidateFiles 优化，再到1.2的默认sort based shuffle，以及最近的Tungsten-sort Based Shuffle，spark shuffle一直在不断演进。关于这部分演进内容可以参考[Spark Shuffle的技术演进](https://www.jianshu.com/p/4c5c2e535da5)
整体上来说，可以分为hash based shuffle和 sort based shuffle，不过随着spark shuffle的演进，两者的界限越来越越模糊，虽然2.0版本中hash based shuffle 退出历史舞台，其实只是作为Sort Based Shuffle 的一种case出现。

## spark shuffle 原理

因为hash based shuffle 已经退出历史舞台，所以这里直接以spark 2.3 的sort based shuffle 为例，看下spark shuffle的原理。
shuffle 的整个生命周期由shuffleManager 来管理，spark 2.3中，唯一的支持方式为SortShuffleManager，SortShuffleManager 中定义了writer 和reader 对应shuffle的map 和reduce阶段。writer 有三种运行模式：
- BypassMergeSortShuffleWriter
- SortShuffleWriter
- UnsafeShuffleWriter
![shuffle](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/shuffle-2018-09-25.jpeg)

下面分别介绍一下这三种writer 的原理：

### BypassMergeSortShuffleWriter

首先，BypassMergeSortShuffleWriter的运行机制的触发条件如下：
- shuffle reduce task(即partition)数量小于spark.shuffle.sort.bypassMergeThreshold参数的值。
- 没有map side aggregations。
note: map side aggregations是指在map端的聚合操作，通常来说一些聚合类的算子都会都map端的aggregation。不过对于groupByKey 和combineByKey， 如果设定mapSideCombine为false，就不会有 map side aggregations。

触发之后，整体的处理流程大致如下：

![bypass-sort-shuffle](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/sort-shuffle-bypass.png)

每个task 会为每一个下游的reduce task 创建一个临时文件(图中下游reduce task应该有三个，这张图直接引用的美团技术博客的，不改了)，将key按照hash 存入对应临时文件中，因为写入磁盘文件是通过Java的BufferedOutputStream实现的，BufferedOutputStream是Java的缓冲输出流，首先会将数据缓冲在内存中，当内存缓冲满溢之后再一次写入磁盘文件中，这样可以减少磁盘IO次数，提升性能。所以图中会有内存缓冲的概念。最后，会将所有临时文件合并成一个磁盘文件，并创建一个索引文件标识下游各个reduce task的数据在文件中的start offset与end offset。
该过程的磁盘写机制其实跟未经优化的HashShuffleManager是一样的，也会创建很多的临时文件（所以触发条件中会有reduce task 数量限制），只是在最后会做一个磁盘文件的合并，对于shuffle  reader 会更友好一些。


### SortShuffleWriter

这种writer 思想上照抄了mapreduce 的shuffle。

![sort-shuffle](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/sort-shuffle-common.png)

该模式下，数据首先写入一个内存数据结构中，此时根据不同的shuffle算子，可能选用不同的数据结构。如果是reduceByKey这种聚合类的shuffle算子，那么会选用Map数据结构，一边通过Map进行聚合，一边写入内存；如果是join这种普通的shuffle算子，那么会选用Array数据结构，直接写入内存。接着，每写一条数据进入内存数据结构之后，就会判断一下，是否达到了某个临界阈值。如果达到临界阈值的话，那么就会尝试将内存数据结构中的数据溢写到磁盘，然后清空内存数据结构。
在溢写到磁盘文件之前，会先根据key对内存数据结构中已有的数据进行排序。排序过后，会分批将数据写入磁盘文件。默认的batch数量是10000条，也就是说，排序好的数据，会以每批1万条数据的形式分批写入磁盘文件。写入磁盘文件也是通过Java的BufferedOutputStream实现的。
一个task将所有数据写入内存数据结构的过程中，会发生多次磁盘溢写操作，也就会产生多个临时文件。最后会将之前所有的临时磁盘文件都进行合并，这就是merge过程，此时会将之前所有临时磁盘文件中的数据读取出来，然后依次写入最终的磁盘文件之中。此外，由于一个task就只对应一个磁盘文件，也就意味着该task为下游stage的task准备的数据都在这一个文件中，因此还会单独写一份索引文件，其中标识了下游各个task的数据在文件中的start offset与end offset。

hadoop shuffle map 阶段会不断地以健值对的形式把数据输出到在内存中构造的一个环形数据结构（更有效地使用内存空间）中，这里是存在map/array中。

BypassMergeSortShuffleWriter 与该机制相比：第一，磁盘写机制不同；第二，不会进行排序。也就是说，启用BypassMerge机制的最大好处在于，shuffle write过程中，不需要进行数据的排序操作，也就节省掉了这部分的性能开销，当然需要满足那两个触发条件。

### UnsafeShuffleWriter

触发条件有三个：
- Serializer支持relocation。这是指Serializer可以对已经序列化的对象进行排序，这种排序起到的效果和先对数据排序再序列化一致。（目前只能使用kryoSerializer），
- 没有map side aggregations
- shuffle reduce task(即partition)数量不能大于支持的上限(2^24)

UnsafeShuffleWriter 将record序列化后插入sorter，然后对已经序列化的record进行排序，并在排序完成后写入磁盘文件作为spill file，再将多个spill file合并成一个输出文件。在合并时会基于spill file的数量和IO compression codec选择最合适的合并策略。


## spark shuffle 源码解析




## spark shuffle 调优


## 参考文章

[SOS - Optimizing Shuffle](https://www.youtube.com/watch?v=fm3Hgxuz2TM)

['map-side' aggregation in Spark](https://stackoverflow.com/questions/31283932/map-side-aggregation-in-spark)

[spark的shuffle和Hadoop的shuffle（mapreduce)的区别和关系是什么？](https://www.zhihu.com/question/27643595/answer/293665719)

[Spark 2.1.0 - Shuffle逻辑分析](https://www.jianshu.com/p/500e8976642f)

[hadoop的mapReduce和Spark的shuffle过程的详解与对比及优化](https://blog.csdn.net/u010697988/article/details/70173104)

[Spark性能优化指南——高级篇](https://tech.meituan.com/spark_tuning_pro.html)

[Spark Shuffle原理及相关调优](http://sharkdtu.com/posts/spark-shuffle.html)

[彻底搞懂 Spark 的 shuffle 过程（shuffle write）](https://toutiao.io/posts/eicdjo/preview)

[Shuffle operations](https://spark.apache.org/docs/2.1.1/programming-guide.html#shuffle-operations)

[Apache Spark Shuffles Explained In Depth](http://hydronitrogen.com/apache-spark-shuffles-explained-in-depth.html)

[Spark Shuffle的技术演进](https://www.jianshu.com/p/4c5c2e535da5)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***