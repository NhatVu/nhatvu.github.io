---
layout: post
title:  "90.  PostgreSQL: Partition"
date:   2025-02-19 00:02:00 +0000
category: technical
---
- [1. Overview](#1-overview)
- [2. Declarative Partitioning](#2-declarative-partitioning)
- [3. Limitation](#3-limitation)
- [4. Best practices](#4-best-practices)
- [5. References](#5-references)

SUMMARY: This article discusses table partitions, the benefits of using them to increase performance, and the types of partitions that can be used in PostgreSQL. A lot of content are copied from Postgres documentation, for personal reference only. 

### 1. Overview
Partitioning refers to splitting what is logically one large table into smaller physical pieces. Partitioning can provide several benefits:
- Query performance can be improved dramatically in certain situations, particularly when most of the heavily accessed rows of the table are in a single partition or a small number of partitions
- When queries or updates access a large percentage of a single partition, performance can be improved by using a sequential scan of that partition instead of using an index, which would require random-access reads scattered across the whole table.
- Bulk loads and deletes can be accomplished by adding or removing partitions. These commands also entirely avoid the VACUUM overhead caused by a bulk DELETE.
- Seldom-used data can be migrated to cheaper and slower storage media.

These benefits will normally be worthwhile only when a table would otherwise be very large. The exact point at which a table will benefit from partitioning depends on the application, although a rule of thumb is that the size of the table should exceed the physical memory of the database server.

PostgreSQL offers built-in support for the following forms of partitioning:
- Range Partitioning: Each range's bounds are understood as being inclusive at the lower end and exclusive at the upper end. For example, if one partition's range is from 1 to 10, and the next one's range is from 10 to 20, then value 10 belongs to the second partition not the first.
- List Partitioning: The table is partitioned by explicitly listing which key value(s) appear in each partition.
- Hash Partitioning: The table is partitioned by specifying a modulus and a remainder for each partition

### 2. Declarative Partitioning 
The table that is divided is referred to as a *partitioned table*.

The partitioned table itself is a “virtual” table having no storage of its own. Instead, the storage belongs to partitions and followed the partition bounds. 

All rows inserted into a partitioned table will be routed to the appropriate one of the partitions based on the values of the partition key column(s). 

Updating the partition key of a row will cause it to be moved into a different partition if it no longer satisfies the partition bounds of its original partition.

Inserting data into the parent table that does not map to one of the existing partitions will cause an error; an appropriate partition must be added manually.

It is not possible to turn a regular table into a partitioned table or vice versa. However, it is possible to add an existing regular or partitioned table as a partition of a partitioned table, or remove a partition from a partitioned table turning it into a standalone table; this can simplify and speed up many maintenance processes. See ALTER TABLE to learn more about the ATTACH PARTITION and DETACH PARTITION sub-commands.

If an index is created on parent table, this automatically creates a matching index on each partition, and any partitions you create or attach later will also have such an index.

Make sure Partition pruning is on, by checking `SHOW enable_partition_pruning -- on by default`.

More example, checking link [1] and [2].
### 3. Limitation
- To create a unique or primary key constraint on a partitioned table, the partition keys must not include any expressions or function calls and the constraint's columns must include all of the partition key columns.
- Partitions cannot have columns that are not present in the parent. Tables may be added as a partition with ALTER TABLE ... ATTACH PARTITION only if their columns exactly match the parent.

### 4. Best practices
- One of the most critical design decisions will be the column or columns by which you partition your data. Often the best choice will be to partition by the column or set of columns which most commonly appear in WHERE clauses of queries being executed on the partitioned table. Removal of unwanted data is also a factor to consider when planning your partitioning strategy. An entire partition can be detached fairly quickly, so it may be beneficial to design the partition strategy in such a way that all data to be removed at once is located in a single partition.
- Choosing the target number of partitions that the table should be divided into is also a critical decision to make. Not having enough partitions may mean that indexes remain too large and that data locality remains poor which could result in low cache hit ratios. However, Too many partitions can mean longer query planning times and higher memory consumption during both query planning and execution. When choosing how to partition your table, it's also important to consider what changes may occur in the future
- It is important to consider the overhead of partitioning during query planning and execution. The query planner is generally able to handle partition hierarchies with up to a few thousand partitions fairly well, provided that typical queries allow the query planner to prune all but a small number of partitions. Another reason to be concerned about having a large number of partitions is that the server's memory consumption may grow significantly over time, especially if many sessions touch large numbers of partitions. That's because each partition requires its metadata to be loaded into the local memory of each session that touches it.
  
### 5. References 
1. [5.12. Table Partitioning](https://www.postgresql.org/docs/16/ddl-partitioning.html)
2. [How to use table partitioning to scale PostgreSQL](https://www.enterprisedb.com/postgres-tutorials/how-use-table-partitioning-scale-postgresql)

