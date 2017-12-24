---
layout: post
title: mongodump 在数据量大的时候非常缓慢
date: 2017.11.28 21:10 +08:00
tags: 
 - mongo 
 - mongodump
---

这边的解决方案是增加 `--forceTableScan`

> --forceTableScan
>
> Forces mongodump to scan the data store directly: typically, mongodump saves entries as they appear in the index of the _id field. Use --forceTableScan to skip the index and scan the data directly. Typically there are two cases where this behavior is preferable to the default:
>
> If you have key sizes over 800 bytes that would not be present in the _id index.
>
> Your database uses a custom _id field.
>
> When you run with --forceTableScan, mongodump does not use $snapshot. As a result, the dump produced by mongodump can reflect the state of the database at many different points in time.

我们知道，mongodb会把document存在dbname.1,dbname.2...的文件里边，当文档增多时，会新增新的dbname.n文件。mongodump默认会通过_id字段去读取文档。默认的objectId其实就是时间戳，使用objectid作为id的，mongodump时相当于按时间顺序去读取文档，会以此遍历dbname.n文件。

而我使用的是自己设的_id。mongodump遍历\_id字段的顺序与时序关系不大，会随机访问dbname.n文件，产生大量的随机读。使用这个参数，可以不通过\_id去读取。

mongodump默认使用snapshot，会通过扫描_id 索引，然后去读取实际的文档。TableScan会按照mongodb的**物理存储顺序**进行扫描，没有读取index的过程。但是tablescan潜在的问题是，如果一个文档在dump的过程中移动了（物理上），有可能会最终输出两次。