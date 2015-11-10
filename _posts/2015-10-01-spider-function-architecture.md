---
layout: post
title:  "Spider数据引擎实践与优化（三）Spider功能与结构"
date:   2015-10-08
categories: Spider数据引擎实践与优化
---

* content
{:toc}

## 部署结构

Spider部署上由Proxy节点、数据节点和监控节点，三类节点构成：

![deploy]({{ "/css/pics/deploy.png"}})

Backend Node:数据节点，支持MySQL/MariaDB和Oracle。

Proxy Node：完成数据的分布式处理。

Monitoring Node（可选）：对Backend节点的监控，排除故障节点。

## 主要功能

*  支持MySQL/MariaDB客户端接口

*  支持MySQL/MariaDB所有语法

*  支持多种数据节点MySQL、Oracle

*  支持数据分片：Hash、Range

*  支持数据复制：

*  支持跨节点的XA数据访问

*  故障节点（数据复制）自动隔离

*  并行扫描

## 软件结构

下图显示了Spider存储引擎在整个MySQL/MariaDB代码结构中的位置，数据分区功能是由MySQL自带的Partition storage engine完成的。

![spider-architecture1]({{ "/css/pics/spider-architecture1.png"}})

作为数据引擎，Spider可以充分利用MySQL/MariaDB已有的语法解析、优化、执行和分区的功能，从而支持MySQL/MariaDB的所有SQL语句。但同时使得Spider性能存在瓶颈，因为Server层与数据引擎的交互有两个特点：1）一次交互获取表的一条记录，并且不带有该表的查询条件，2）update和delete操作的执行是先读记录后修改记录。这两点使得Server层与数据引擎层有过多的不必要的交互。对于本地数据引擎，由于数据存储在内存中，多次交互带来的负担影响不显著，而Spider引擎通过网络与数据节点交互，这会导致Spider的性能会大幅下降。为了克服此问题，Patched版本对Partition存储引擎、解析模块进行了修改，实现了以下的优化：

*  Direct updating.

*  Direct aggregating.

*  Engine condition pushdown with table partitoning.

*  MRR(include BKA) with table partitioning.


## 常用模式

根据分区和复制的方式，Spider包含[4种使用模式](https://mariadb.com/kb/en/mariadb/spider-storage-engine-core-concepts/)

*  Federation：数据直连，类似Oracle dblink或MariaDB FedX引擎

*  Federation+HA：在Federation的基础上，增加数据复制

*  Sharding:数据分片

*  Sharding+HA：在Sharding的基础上，给每个分片增加数据复制

![spider-application-mode]({{ "/css/pics/spider-application-mode.png"}})

---
本文内容遵从CC版权协议, 可以随意转载, 但必须以超链接形式标明文章原始出处和作者信息及版权声明  
