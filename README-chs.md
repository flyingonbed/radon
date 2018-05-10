[![Build Status](https://travis-ci.org/radondb/radon.png)](https://travis-ci.org/radondb/radon)
[![Go Report Card](https://goreportcard.com/badge/github.com/radondb/radon)](https://goreportcard.com/report/github.com/radondb/radon)
[![codecov.io](https://codecov.io/gh/radondb/radon/graphs/badge.svg)](https://codecov.io/gh/radondb/radon/branch/master)

# OverView  —— 概述
RadonDB 是一个开源的，基于云的 MySQL  数据库，它具有无限的扩展和高性能。

> RadonDB is an open source, Cloud-native MySQL database for unlimited scalability and performance.

## what is RadonDB? —— 什么是 RadonDB？
RadonDB 是一个基于云的 MySQL 数据库产品，它被设计为完全的分布集群架构，从而可以支持无限的横向、容量和性能扩展。它支持高数据一致性的分布式事务，并把 MySQL 作为数据可靠性保证的存储引擎。RadonDB 兼容 MySQL 协议，并支持自动表切分和自动化功能，以简化维护和操作工作流程。
> RadonDB is a cloud-native database based on MySQL，and architected in fully distributed cluster that enable unlimited scalability (scale-out), capacity and performance. It supported distributed transaction that ensure high data consistency, and leveraged MySQL as storage engine for trusted data reliability. RadonDB is compatible with MySQL protocol, and sup-porting automatic table sharding as well as batch of automation feature for simplifying the maintenance and operation workflow.

## Feature —— 特性
- 自动表切分：跨节点的自动表切分(分区)可以使复杂的维护操作变的极为简单。
> * **Automatic Table Sharding**: automatically sharding (partition) across nodes that mini-mizing the complexity of maintenance and operation.

- 分布式事务：支持分布式事务跨分片(分区)，确保整个事务的原子性、一致性、隔离性和持久性(ACID)。
> * **Distributed Transaction**: supporting distributed transaction across shards (partitions) and securing Atomicity, Consistency, Isolation, Durability (ACID) for whole trans-action process.

- 连接线程池：预先设置一组已连接的线程，可以利用它们来加速SQL集群和存储节点之间的连接效率；并支持自动重新连接和线程重用。
> * **Connection Thread Pool**: presetting a set of connected threads that can be lever-aged to accelerating the efficiency of connection between SQL cluster and storage nodes; supporting automatic reconnection and thread reuse.

- 审计和日志：用户可以选择启用这个功能来审计和记录SQL查询操作的多个维度：查询事件时间、操作语句类型、消耗时间等；该功能有助于确保操作安全和数据一致性；审计日志可以设置为多种模式，以提高灵活性：只读(SQL)，只写，或同时 读/写。
> * **Auditing and Logging**: Users can choose to enable this function for auditing and logging the SQL query operation in multiple dimensions: query event time, opera-tions statement type, consuming time, etc; this function help to secure operation safety and data compliance; the auditing log is able to be set in multiple modes for high flexibility: read (SQL) only, write only, or read/write simultaneously.

## Architecture —— 架构

## Overview —— 概述
RadonDB是基于 MySQL 的新一代分布式关系数据库(MyNewSQL)。它的设计初衷是——创建一个拥有我们的开发人员都想要使用的特性的开源数据库，比如：金融高可用性、大容量数据库、自动水平表切分、可伸缩的和强一致性，本指南详细介绍了 Radon 进程的内部工作原理。
> RadonDB is a new generation of distributed relational database (MyNewSQL) based on MySQL. It was designed to create the open-source database our developers would want to use: one that has features such as financial high availability、large-capacity database、automatic plane split table、 scalable and strong consistency, this guide sets out to detail the inner-workings of the radon process as a means of explanation.

## SQL Layer —— SQL 层

### SQL support —— SQL 支持
在SQL语法级别上，RadonDB 完全兼容 MySQL。您可以在这里查看所有的 SQL 特性[radon_sql_support](docs/radon_sql_support.md)
> On SQL syntax level, RadonDB Fully compatible with MySQL.You can view all of the SQL features RadonDB supports here  [radon_sql_support](docs/radon_sql_support.md)

###  SQL parser, planner, executor —— SQL 解析器，执行计划生成器，执行器

当你的 SQL 节点通过代理从 mysql 客户端接收 SQL 请求后，RadonDB 开始解析该语句，创建一个查询计划，然后执行该计划。
> After your SQL node  receives a SQL request from a mysql client via proxy, RadonDB parses the statement, creates a query plan, and then executes the plan.




                                                                    +----------------+
                                                        x---------->| node1_Executor |
                                +--------------------+  |           +----------------+
                                |      SQL Node      |  |
                                |--------------------|  |
    +-------------+             |     sqlparser      |  |           +----------------+
    |    query    |+----------->|                    |--x---------->| node2_Executor |
    +-------------+             |  Distributed Plan  |  |           +----------------+
                                |                    |  |
                                +--------------------+  |
                                                        |           +----------------+
                                                        x---------->| node3_Executor |
                                                                    +----------------+



``` Parsing —— 解析``` 
接收到的查询被 sqlparser 解析(需要使用 mysql 支持的语法)，并生成了抽象语法树(AST)。
> Received queries are parsed by sqlparser (which describes the supported syntax by mysql) and generated Abstract Syntax Trees (AST).

``` Planning —— 生成执行计划```
通过这个 AST，RadonDB 开始生成一个 planNodes 树来生成该查询的执行计划。

> With the AST, RadonDB begins planning the query's execution by generating a tree of planNodes.

此步骤还包括分析客户机的 SQL 语句与预期的 AST 表达式(包括类型检查)的步骤。
> This step also includes steps analyzing the client's SQL statements against the expected AST expressions, which include things like type checking.

你可以通过使用`EXPLAIN`来查看生成的执行计划(在这个阶段我们只能使用`EXPLAIN`来分析分布式表)
> You can see the a query plan  generates using `EXPLAIN`(At this stage we only use `EXPLAIN` to  analysis  Table distribution).

``` Excuting —— 执行```

通过分布式执行计划并行执行存储层中的执行器。
> Executing an Executor in a storage layer in Parallel with a Distributed Execution Plan.

### SQL with Transaction —— 事务中的 SQL
SQL节点是无状态的，但是为了保证事务的“快照隔离”，它现在处于写多读模式。
> The SQL node is stateless, but in order to guarantee transaction `Snapshot Isolation`, it is currently a write-multiple-read mode.

## Transaction Layer —— 事务层

``` Distributed transaction —— 分布式事务```
RadonDB 提供分布式事务处理能力。如果分布式执行器在不同的存储节点执行，并且其中某个节点执行失败，那么所有的节点都会回滚该事务。这保证了分布式事务的跨节点一致性，并且使数据库处于一致性状态。
> RadonDB provides distributed transaction capabilities. If the distrubuted executor at different storage nodes and one of the nodes failed to execute, then operation of the rest nodes will be rolled back, This guarantees the atomicity of operating across nodes  and makes the database in a consistent state.

```Isolation Levels —— 隔离等级```
RadonDB 实现了一致性级别的 SI (快照隔离，Snapshot Isolation)。只要分布式事务没有提交，或者只有部分分区已经提交，那么其他事务就看不到该操作。
> RadonDB achieves the level of SI (Snapshot Isolation) at the level of consistency. As long as a distributed transaction has not commit, or if some of the partitions have committed, the operation is invisible to other transactions.

``` Transaction with SQL Layer —— SQL 层的事务```
SQL 节点是无状态的，但是为了保证事务“快照隔离”，它现在处于写多读模式。
> The SQL node is stateless, but in order to guarantee transaction `Snapshot Isolation`, it is currently a write-multiple-read mode.

## Issues —— 问题

The [integrated github issue tracker](https://github.com/radondb/radon/issues)
is used for this project.

## License —— 许可证
RadonDB 使用 GPLv3. 许可发行。
RadonDB is released under the GPLv3. See LICENSE
