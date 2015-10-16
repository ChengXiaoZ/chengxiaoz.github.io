---
layout: post
title:  "Spider数据引擎实践与优化（四） 安装与基本案例"
date:   2015-10-14
categories: Spider数据引擎实践与优化
---

* content
{:toc}

## Centos6.4 源码安装

build需要的软件

	yum -y install gcc-c++ ncurses-devel bison libxml2-devel libevent-devel openssl-devel

	yum -y install libssl.so.6

下载PatchedMariaDB版本 [mariadb-10.0.9-spider-3.2-vp-1.1-mroonga-4.0.tgz](http://spiderformysql.com/downloads/spider-3.2/mariadb-10.0.9-spider-3.2-vp-1.1-mroonga-4.0.tgz)

解压文件并建立build目录

	cd /home/test
	
	tar xzvf mariadb-10.0.9-spider-3.2-vp-1.1-mroonga-4.0.tgz

	mv mariadb-10.0.9-spider-3.2-vp-1.1-mroonga-4.0 mariadb-10.0.9-spider-3.2

	mkdir bld

	cd bld

cmake时指定prefix目录，裁剪掉不需要的存储引擎
	
	cmake .. \
	-DCMAKE_INSTALL_PREFIX=/home/test/mariadb-10.0.9-spider-3.2/bld/release \
	-DMYSQL_UNIX_ADDR=/tmp/mariadb1.sock \
	-DMYSQL_TCP_PORT=3306 \
	-DWITHOUT_INNOBASE_STORAGE_ENGINE=1 \
	-DWITHOUT_ARCHIVE_STORAGE_ENGINE=1 \
	-DWITHOUT_BLACKHOLE_STORAGE_ENGINE=1 \
	-DWITHOUT_CASSANDRA_STORAGE_ENGINE=1 \
	-DWITHOUT_CONNECT_STORAGE_ENGINE=1 \
	-DWITHOUT_CSV_STORAGE_ENGINE=1 \
	-DWITHOUT_MROONGA_STORAGE_ENGINE=1 \
	-DWITHOUT_FEDERATED_STORAGE_ENGINE=1 \
	-DWITHOUT_TOKUDB_STORAGE_ENGINE=1 \
	-DWITHOUT_XTRADB_STORAGE_ENGINE=1 \
	-DWITHOUT_FEDERATEDX_STORAGE_ENGINE=1 \
	-DWITHOUT_MYISAMMRG_STORAGE_ENGINE=1 \
	-DWITHOUT_SPINX_STORAGE_ENGINE=1 \
	-DWITHOUT_VP_STORAGE_ENGINE=1 \
	-DWITHOUT_SEQUENCE_STORAGE_ENGINE=1 \
	-DWITHOUT_OQGRAPH_STORAGE_ENGINE=1 \
	-DWITHOUT_TEST_SQL_DISCOVERY_STORAGE_ENGINE=1 \
	-DWITHOUT_EXAMPLE_STORAGE_ENGINE=1

	make -j2

	make install

	cd release

	cp support-files/my-small.cnf my.cnf

编辑my.cnf增加 
	
	basedir=/home/test/mariadb-10.0.9-spider-3.2/bld/release

初始化数据

	scripts/mysql_install_db --user=mariadb --defaults-file=my.cnf 

启动服务

	bin/mysqld --defaults-file=my.cnf --user=mariadb

创建数据字典表

	bin/mysql -uroot
	
	MariaDB [(none)]>source /home/test/mariadb-10.0.9-spider-3.2/bld/release/share/install_spider.sql

	MariaDB [(none)]> SELECT engine, support, transactions, xa FROM information_schema.engines;

	| engine             | support | transactions | xa   |
	|:-------------------|:--------|:-------------|:-----|
	| SPIDER             | YES     | YES          | YES  |
	| PERFORMANCE_SCHEMA | YES     | NO           | NO   |
	| MEMORY             | YES     | NO           | NO   |
	| MRG_MyISAM         | YES     | NO           | NO   |
	| CSV                | YES     | NO           | NO   |
	| Aria               | YES     | NO           | NO   |
	| MyISAM             | DEFAULT | NO           | NO   |

install_spider.sql在mysql库中建立几个字典表

*  spider_link_failed_log

*  spider_link_mon_servers

*  spider_tables

*  spider_xa

*  spider_xa_failed_log

*  spider_xa_member
	
---

## 基本使用

Spider存储引擎中，Spider表与远程表的映射关系、连接信息以及各类参数信息，被保存在CREATE TABLE中的COMMENT子句。

例如，在BACKEND节点的数据库db1有表t，连接用户名stest，密码123456

	create table t(a int , b varchar(10);

在PROXY节点，spider表可以如下创建

	create table st(a int, b varchar(10))
	ENGINE = SPIDER CONNECTION='wrapper "mysql", host "$MySQLServer1", port "$MySQLPort1", database "db1", table "t", user "stest", password "123456"';
	
在PROXY节点插入的数据，这些数据会被存储到BACKEND节点。
	
	insert into st(1, 'aa');	
	select * from st;
	
COMMENT子句中的连接参数和库信息可以提取到Server中，在CREATE TABLE语句引用。

创建SERVER backend1_db1

	CREATE SERVER backend1_db1 
	FOREIGN DATA WRAPPER mysql	
	OPTIONS(	
	HOST '$MySQLServer1', 
	PORT $MySQLPort1, 
	DATABASE 'db1', 
	USER 'stest', 
	PASSWORD '123456');

在创建spider表时，使用SERVER

	create table st(a int, b varchar(10))
	ENGINE = SPIDER CONNECTION='wrapper "mysql", table "t", srv "backend1_db1"';

##典型部署案例

下面的案例环境由一个PROXY节点和2个BACKEND节点构成，演示了四种使用模式。

### 数据环境

### federation

数据直连，类似Oracle dblink或MariaDB FedX引擎

BACKEND1节点

PROXY 节点

![spider-fed]({{ "/css/pics/spider-deploy-fed.png"}})

	create table t01_fed_mysql(A int, B float, C varchar(10), D char(10), E datetime(6))
	ENGINE = SPIDER CONNECTION='wrapper "mysql", host "$MySQLServer1", port "$MySQLPort1", 
	database "db1", table "T01_FED", user "comm", password "123456"';

### Sharding

数据分片，支持range和hash两种算法

range

![spider-deploy-share-range]({{ "/css/pics/spider-deploy-share-range.png"}})

	create table $test_table(A int, B float, C varchar(10), D char(10), E datetime(6))
	ENGINE=SPIDER partition by range(A) 
 (
	partition pt1 values less than (100) 
	comment ' wrapper "mysql", host "$MySQLServer1", port "$MySQLPort1", database "db1", table "T01_FED", user "comm", password "123456"',
	partition pt2 values less than maxvalue 
	comment ' wrapper "mysql", host "$MySQLServer2", port "$MySQLPort2", database "db1", table "T01_FED", user "comm", password "123456"');

share

![spider-deploy-share-key]({{ "/css/pics/spider-deploy-share-key.png"}})

	create table $test_table(A int, B float, C varchar(10), D char(10), E datetime(6))
	ENGINE=SPIDER partition by KEY(A) 
	(
	partition pt1 comment ' wrapper "mysql", host "$MySQLServer1", port "$MySQLPort1", database "db1", table "T01_SHARE_KEY", user "comm", password "123456"',
	partition pt2 comment ' wrapper "mysql", host "$MySQLServer2", port "$MySQLPort2", database "db1", table "T01_SHARE_KEY", user "comm", password "123456"');

### federation & ha

：在Federation的基础上，增加数据复制

![spider-deploy-fed-ha]({{ "/css/pics/spider-deploy-fed-ha.png"}})

### Sharding & ha

：在Sharding的基础上，给每个分片增加数据复制

![spider-deploy-share-range-ha]({{ "/css/pics/spider-deploy-share-range-ha.png"}})

---

本文内容遵从CC版权协议, 可以随意转载, 但必须以超链接形式标明文章原始出处和作者信息及版权声明  
