---
layout: post
title:  "Spider数据引擎实践与优化（四）实践案例（oracle）"
date:   2015-11-10
categories: Spider数据引擎实践与优化
---

* content
{:toc}

## Centos6.4 源码安装

安装build需要的软件

	yum -y install gcc-c++ ncurses-devel bison libxml2-devel libevent-devel openssl-devel

	yum -y install libssl.so.6
	
---------	

从[instantclient](http://http://www.oracle.com/technetwork/topics/linuxx86-64soft-092277.html)下载instantclient-basic-linux-11.2.0.4.0.zip、instantclient-sdk-linux-11.2.0.4.0.zip、instantclient-sqlplus-linux-11.2.0.4.0.zip。

解压到/home/test/instantclient_11_2

	unzip instantclient-basic-linux-11.2.0.4.0.zip 
	
	unzip instantclient-sdk-linux-11.2.0.4.0.zip 
	
	unzip instantclient-sqlplus-linux-11.2.0.4.0.zip
	
建立ln
	
	ln libclntsh.so.11.1 libclntsh.so

配置环境变量

/etc/profile中增加

	export PATH=$PATH:/home/soft/instantclient_11_2/

	export LD_LIBRARY_PATH=/home/soft/instantclient_11_2/:$LD_LIBRARY_PATH
	
oracle客户端
	
	sqlplus system/orcl123456@//192.168.226.1/orcl
	
---------	

代码目录为/home/test/MariaDBserver

	mkdir bld-ora

	cd /home/test/MariaDBserver/bld-ora

	cd bld-ora

cmake时指定prefix目录、ORACLE sdk路径，裁剪掉不需要的存储引擎

	cmake .. \
	-DCMAKE_INSTALL_PREFIX=/home/test/MariaDBserver/bld-ora/release \
	-DSPIDER_WITH_ORACLE_OCI=1  \
	-DORACLE_INCLUDE_DIR=/home/test/instantclient_11_2/sdk/include  \
	-DORACLE_OCI_LIBRARY=/home/test/instantclient_11_2/libclntsh.so \
	-DWITHOUT_INNOBASE_STORAGE_ENGINE=1  \
	-DWITHOUT_ARCHIVE_STORAGE_ENGINE=1  \
	-DWITHOUT_BLACKHOLE_STORAGE_ENGINE=1  \
	-DWITHOUT_CASSANDRA_STORAGE_ENGINE=1  \
	-DWITHOUT_CONNECT_STORAGE_ENGINE=1  \
	-DWITHOUT_CSV_STORAGE_ENGINE=1  \
	-DWITHOUT_MROONGA_STORAGE_ENGINE=1  \
	-DWITHOUT_FEDERATED_STORAGE_ENGINE=1  \
	-DWITHOUT_TOKUDB_STORAGE_ENGINE=1  \
	-DWITHOUT_XTRADB_STORAGE_ENGINE=1  \
	-DWITHOUT_FEDERATEDX_STORAGE_ENGINE=1  \
	-DWITHOUT_MYISAMMRG_STORAGE_ENGINE=1  \
	-DWITHOUT_SPINX_STORAGE_ENGINE=1  \
	-DWITHOUT_VP_STORAGE_ENGINE=1  \
	-DWITHOUT_SEQUENCE_STORAGE_ENGINE=1  \
	-DWITHOUT_OQGRAPH_STORAGE_ENGINE=1  \
	-DWITHOUT_TEST_SQL_DISCOVERY_STORAGE_ENGINE=1  \
	-DWITHOUT_EXAMPLE_STORAGE_ENGINE=1
	
	make -j2

	make install

	cd release

	cp support-files/my-large.cnf my.cnf

编辑my.cnf增加 
	
	basedir=/home/test/MariaDBserver/bld-ora/release

初始化数据

	scripts/mysql_install_db --user=mariadb --defaults-file=my.cnf 

启动服务

	bin/mysqld --defaults-file=my.cnf --user=mariadb

加载动态库并创建数据字典表

	bin/mysql -uroot
	
	MariaDB [(none)]>install plugin spider soname 'ha_spider.so';
	
	MariaDB [(none)]>source /home/test/MariaDBserver/bld-ora/release/share/install_spider.sql

	MariaDB [(none)]> SELECT engine, support, transactions, xa FROM information_schema.engines;
	+--------------------+---------+--------------+------+
	| engine             | support | transactions | xa   |
	+--------------------+---------+--------------+------+
	| SPIDER             | YES     | YES          | YES  |
	| PERFORMANCE_SCHEMA | YES     | NO           | NO   |
	| MEMORY             | YES     | NO           | NO   |
	| MRG_MyISAM         | YES     | NO           | NO   |
	| CSV                | YES     | NO           | NO   |
	| Aria               | YES     | NO           | NO   |
	| MyISAM             | DEFAULT | NO           | NO   |
	+--------------------+---------+--------------+------+
	
---

## 基本使用

Spider存储引擎中，Spider表与远程表的映射关系、连接信息以及各类参数信息，被保存在CREATE TABLE中的COMMENT子句。

例如，在Oracle实例的模式db1有表t，用户名db1，密码为abc123

	create table t(a int , b varchar(10));

在PROXY节点，创建spider表，其中wrapper为oracle，表示此连接为Oracle。

	create table ora_st(A int, B varchar(10))
	ENGINE = SPIDER CONNECTION='wrapper "oracle", host "192.168.226.1/orcl", port "1521", database "DB1", table "T", user "db1", password "abc123"';	
	
	
在PROXY节点插入的数据，这些数据会被存储到Oracle BACKEND节点。
	
	insert into ora_st values(1, 'aa');
	insert into ora_st values(2, 'bb');
	select * from ora_st;
	+------+------+
	| a    | b    |
	+------+------+
	|    1 | aa   |
	|    2 | bb   |
	+------+------+
	
COMMENT子句中的连接参数（用户名、密码）可以提取到Server中，在CREATE TABLE语句引用。

PROXY节点创建SERVER ora_backend1

	CREATE SERVER ora_backend1_db1 
	FOREIGN DATA WRAPPER oracle	
	OPTIONS(	
	HOST '192.168.226.1/orcl', 
	PORT 1521, 
	DATABASE 'DB1', 
	USER 'db1', 
	PASSWORD 'abc123');
	
在创建spider表时，使用SERVER

	create table st2(A int, B varchar(10))
	ENGINE = SPIDER CONNECTION='wrapper "oracle", table "T", srv "ora_backend1_db1"';

##典型应用案例

下面的案例环境由一个PROXY和2个oracle BACKEND构成，演示了四种基本应用模式。

*  PROXY:IP127.0.0.1 端口3306 root用户密码空

*  BACKEND1:192.168.226.1/orcl 端口1521 用户system密码orcl123456

*  BACKEND2:192.168.226.1/orcl 端口1521 用户system密码orcl123456

定义mysql客户端登录alias

	alias proxy='mysql  --user=root --host=127.0.0.1 --port=3306'
	
PROXY节点创建6个server。ora_backend1和ora_backend2用于创建BACKEND1和BACKEND2的模式，剩下四个server对应BACKEND1的db1、db2和BACKEND2的db1、db2。

	proxy << EOF
	CREATE SERVER ora_backend1
	  FOREIGN DATA WRAPPER oracle 
	OPTIONS( 
	  HOST '192.168.226.1/orcl',
	  PORT 1521,   
	  DATABASE 'SYSTEM',
	  USER 'system',
	  PASSWORD 'orcl123456'
	);
	
	CREATE SERVER ora_backend2
	  FOREIGN DATA WRAPPER oracle 
	OPTIONS( 
	  HOST '192.168.226.1/orcl2',
	  PORT 1521,   
	  DATABASE 'SYSTEM',
	  USER 'system',
	  PASSWORD 'orcl123456'
	);
	
	CREATE SERVER ora_backend1_db1 
	  FOREIGN DATA WRAPPER oracle 
	OPTIONS( 
	  HOST '192.168.226.1/orcl',
	  PORT 1521,   
	  DATABASE 'DB1',
	  USER 'db1',
	  PASSWORD 'abc123'
	);
	
	CREATE SERVER ora_backend1_db2
	  FOREIGN DATA WRAPPER oracle 
	OPTIONS( 
	  HOST '192.168.226.1/orcl',
	  PORT 1521,   
	  DATABASE 'DB2',
	  USER 'db2',
	  PASSWORD 'abc123'
	);
	
	CREATE SERVER ora_backend2_db1 
	  FOREIGN DATA WRAPPER oracle 
	OPTIONS( 
	  HOST '192.168.226.1/orcl2',
	  PORT 1521,   
	  DATABASE 'DB1',
	  USER 'db1',
	  PASSWORD 'abc123'
	);
	
	CREATE SERVER ora_backend2_db2
	  FOREIGN DATA WRAPPER oracle 
	OPTIONS( 
	  HOST '192.168.226.1/orcl2',
	  PORT 1521,   
	  DATABASE 'DB2',
	  USER 'db2',
	  PASSWORD 'abc123'
	);
	EOF

在BACKEND1和BACKEND2，创建模式db1和db2。

不使用sqlplus登录Oracle，采用Spider引擎提供的udf函数SPIDER_DIRECT_SQL访问远程Oracle。
	
	SELECT SPIDER_DIRECT_SQL('create user db1 identified by abc123', '', 'srv "ora_backend1"');
	SELECT SPIDER_DIRECT_SQL('grant dba to db1', '', 'srv "ora_backend1"');
	SELECT SPIDER_DIRECT_SQL('create user db2 identified by abc123', '', 'srv "ora_backend1"');
	SELECT SPIDER_DIRECT_SQL('grant dba to db2', '', 'srv "ora_backend1"');	
	
	SELECT SPIDER_DIRECT_SQL('create user db1 identified by abc123', '', 'srv "ora_backend2"');
	SELECT SPIDER_DIRECT_SQL('grant dba to db1', '', 'srv "ora_backend2"');
	SELECT SPIDER_DIRECT_SQL('create user db2 identified by abc123', '', 'srv "ora_backend2"');
	SELECT SPIDER_DIRECT_SQL('grant dba to db2', '', 'srv "ora_backend2"');
	

### federation

数据直连，类似Oracle dblink或MariaDB FedX引擎。

对PROXY节点fed表的操作，映射到BACKEND1节点的db1.fed表。

![spider-fed]({{ "/css/pics/spider-deploy-fed.png"}})

BACKEND1节点创建表

	SELECT SPIDER_DIRECT_SQL('create table fed(A int, B float, C varchar(10), D char(10), E date)', '', 'srv "ora_backend1_db1"');

PROXY 节点

	use test;
	create table ora_fed(A int, B float, C varchar(10), D char(10), E date) 
	ENGINE = SPIDER CONNECTION='wrapper "oracle", table "FED" ,srv "ora_backend1_db1"';	
	insert into ora_fed values(1, 1.1, 'aaaa1', 'bbbbbb1', null);
	insert into ora_fed values(21, 2, 'aaaa2', 'bbbbbb2', '2011-1-1');
	insert into ora_fed(a) values(300);
	insert into ora_fed values(230, 0.0001, 'aaaa3', 'bbbbbb3', '2012-2-2');
	
	select * from test.ora_fed;
	+------+--------+-------+---------+------------+
	| A    | B      | C     | D       | E          |
	+------+--------+-------+---------+------------+
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	|    1 |    1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |      2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	+------+--------+-------+---------+------------+	
	
### Sharding


数据分片，支持range和hash两种算法

**range**

PROXY节点share_range表分为part1和part2两个部分，part1映射到BACKEND1节点的db1.share_range表，part2映射到BACKEND2节点的db1.share_range表。

数据划分的依据是share_range表字段A的取值范围。

![spider-deploy-share-range]({{ "/css/pics/spider-deploy-share-range.png"}})

BACKEND1节点创建表

	SELECT SPIDER_DIRECT_SQL('create table share_range(a int , b float, c varchar(10), d char(10), e date)', '', 'srv "ora_backend1_db1"');	
		
BACKEND2节点创建表

	SELECT SPIDER_DIRECT_SQL('create table share_range(a int , b float, c varchar(10), d char(10), e date)', '', 'srv "ora_backend2_db1"');

PROXY 节点

	use test;
	create table ora_share_range(A int, B float, C varchar(10), D char(10), E date)
	ENGINE=SPIDER partition by range(A)
	(
	partition pt1 values less than (100) comment ' wrapper "oracle", table "SHARE_RANGE", srv "ora_backend1_db1"',
	partition pt2 values less than maxvalue comment ' wrapper "oracle", table "SHARE_RANGE", srv "ora_backend2_db1"');	
	
	insert into ora_share_range values(1, 1.1, 'aaaa1', 'bbbbbb1', null);
	insert into ora_share_range values(21, 2, 'aaaa2', 'bbbbbb2', '2011-1-1');
	insert into ora_share_range(a) values(300);
	insert into ora_share_range values(230, 0.0001, 'aaaa3', 'bbbbbb3', '2012-2-2');	

	select * from test.ora_share_range;
	+------+--------+-------+---------+------------+
	| A    | B      | C     | D       | E          |
	+------+--------+-------+---------+------------+
	|    1 |    1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |      2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+

	create temporary table tmp(A int, B float, C varchar(10), D char(10), E date);
	SELECT SPIDER_DIRECT_SQL('select * from db1.share_range', 'tmp', 'srv "ora_backend1_db1"');	
	select * from tmp;
	+------+------+-------+---------+------------+
	| A    | B    | C     | D       | E          |
	+------+------+-------+---------+------------+
	|   21 |    2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	|    1 |  1.1 | aaaa1 | bbbbbb1 | NULL       |
	+------+------+-------+---------+------------+	
	
	delete from tmp;
	SELECT SPIDER_DIRECT_SQL('select * from db1.share_range', 'tmp', 'srv "ora_backend2_db1"');	
	select * from tmp;
	+------+--------+-------+---------+------------+
	| A    | B      | C     | D       | E          |
	+------+--------+-------+---------+------------+
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+

**hash**

PROXY节点share_key表分为part1和part2两个部分，part1映射到BACKEND1节点的db1.share_key表，part2映射到BACKEND2节点的db1.share_key表。

数据划分的依据是share_key表字段A的hash值。

![spider-deploy-share-key]({{ "/css/pics/spider-deploy-share-key.png"}})


BACKEND1节点创建表

	SELECT SPIDER_DIRECT_SQL('create table share_key(a int , b float, c varchar(10), d char(10), e date)', '', 'srv "ora_backend1_db1"');	
		
BACKEND2节点创建表
	
	SELECT SPIDER_DIRECT_SQL('create table share_key(a int , b float, c varchar(10), d char(10), e date)', '', 'srv "ora_backend2_db1"');	
	
PROXY 节点

	use test;
	create table ora_share_key(A int, B float, C varchar(10), D char(10), E date)
	ENGINE=SPIDER partition by KEY(A) 
	(
	partition pt1 comment ' wrapper "oracle", table "SHARE_KEY", srv "ora_backend1_db1"',
	partition pt2 comment ' wrapper "oracle", table "SHARE_KEY", srv "ora_backend2_db1"');	
	
	insert into ora_share_key values(1, 1.1, 'aaaa1', 'bbbbbb1', null);
	insert into ora_share_key values(21, 2, 'aaaa2', 'bbbbbb2', '2011-1-1');
	insert into ora_share_key(a) values(300);
	insert into ora_share_key(a) values(6);
	insert into ora_share_key values(230, 0.0001, 'aaaa3', 'bbbbbb3', '2012-2-2');	

	select * from test.ora_share_key;
	+------+--------+-------+---------+------------+
	| A    | B      | C     | D       | E          |
	+------+--------+-------+---------+------------+
	|    1 |    1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |      2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|    6 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+
	
	delete from tmp;
	SELECT SPIDER_DIRECT_SQL('select * from db1.share_key', 'tmp', 'srv "ora_backend1_db1"');	
	select * from tmp;
	+------+------+-------+---------+------------+
	| a    | b    | c     | d       | e          |
	+------+------+-------+---------+------------+
	|    1 |  1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |    2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	+------+------+-------+---------+------------+

	delete from tmp;
	SELECT SPIDER_DIRECT_SQL('select * from db1.share_key', 'tmp', 'srv "ora_backend2_db1"');	
	select * from tmp;
	+------+--------+-------+---------+------------+
	| a    | b      | c     | d       | e          |
	+------+--------+-------+---------+------------+
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|    6 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+
	

### federation & ha

在Federation的基础上，增加数据复制。

对PROXY节点fed_ha表的操作，同时映射到BACKEND1节点的db1.fed_ha表和BACKEND2节点的db2.fed_ha表。

![spider-deploy-fed-ha]({{ "/css/pics/spider-deploy-fed-ha.png"}})

BACKEND1节点创建表
	
	SELECT SPIDER_DIRECT_SQL('create table fed_ha(a int , b float, c varchar(10), d char(10), e date)', '', 'srv "ora_backend1_db1"');
	
BACKEND2节点创建表

	SELECT SPIDER_DIRECT_SQL('create table fed_ha(a int , b float, c varchar(10), d char(10), e date)', '', 'srv "ora_backend2_db2"');
	
PROXY 节点

	use test;
	create table ora_fed_ha(A int, B float, C varchar(10), D char(10), E date)
	ENGINE = SPIDER CONNECTION='wrapper "oracle", table "FED_HA", srv "ora_backend1_db1 ora_backend2_db2"';
	
	insert into ora_fed_ha values(1, 1.1, 'aaaa1', 'bbbbbb1', null);
	insert into ora_fed_ha values(21, 2, 'aaaa2', 'bbbbbb2', '2011-1-1');
	insert into ora_fed_ha(a) values(300);
	insert into ora_fed_ha values(230, 0.0001, 'aaaa3', 'bbbbbb3', '2012-2-2');	

	select * from test.ora_fed_ha;
	+------+--------+-------+---------+------------+
	| A    | B      | C     | D       | E          |
	+------+--------+-------+---------+------------+
	|    1 |    1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |      2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+
	delete from tmp;
	SELECT SPIDER_DIRECT_SQL('select * from db1.fed_ha', 'tmp', 'srv "ora_backend1_db1"');	
	select * from tmp;
	+------+--------+-------+---------+------------+
	| a    | b      | c     | d       | e          |
	+------+--------+-------+---------+------------+
	|    1 |    1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |      2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+
	delete from tmp;
	SELECT SPIDER_DIRECT_SQL('select * from db2.fed_ha', 'tmp', 'srv "ora_backend2_db2"');	
	select * from tmp;
	+------+--------+-------+---------+------------+
	| a    | b      | c     | d       | e          |
	+------+--------+-------+---------+------------+
	|    1 |    1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |      2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+
	
### Sharding & ha

在Sharding的基础上，给每个分片增加数据复制。

PROXY节点share_range_ha表分为part1和part2两个部分，part1映射到BACKEND1节点的db1.share_range_ha表和BACKEND2节点的db2.share_range_ha表，part2映射到BACKEND2节点的db1.share_range_ha表和BACKEND1节点的db2.share_range_ha表。

数据划分的依据是share_range_ha表字段A的取值范围。


![spider-deploy-share-range-ha]({{ "/css/pics/spider-deploy-share-range-ha.png"}})

BACKEND1节点创建表

	SELECT SPIDER_DIRECT_SQL('create table share_range_ha(a int , b float, c varchar(10), d char(10), e date)', '', 'srv "ora_backend1_db1"');
	SELECT SPIDER_DIRECT_SQL('create table share_range_ha(a int , b float, c varchar(10), d char(10), e date)', '', 'srv "ora_backend1_db2"');
	
BACKEND2节点创建表

	SELECT SPIDER_DIRECT_SQL('create table share_range_ha(a int , b float, c varchar(10), d char(10), e date)', '', 'srv "ora_backend2_db1"');
	SELECT SPIDER_DIRECT_SQL('create table share_range_ha(a int , b float, c varchar(10), d char(10), e date)', '', 'srv "ora_backend2_db2"');
	
PROXY 节点

	use test;
	create table ora_share_range_ha(A int, B float, C varchar(10), D char(10), E date)
	ENGINE=SPIDER partition by range(A) 
 	(
	partition pt1 values less than (100) 
	comment ' wrapper "oracle", table "SHARE_RANGE_HA", srv "ora_backend1_db1 ora_backend2_db2"',
	partition pt2 values less than maxvalue 
	comment ' wrapper "oracle", table "SHARE_RANGE_HA", srv "ora_backend2_db1 ora_backend1_db2"');

	insert into ora_share_range_ha values(1, 1.1, 'aaaa1', 'bbbbbb1', null);
	insert into ora_share_range_ha values(21, 2, 'aaaa2', 'bbbbbb2', '2011-1-1');
	insert into ora_share_range_ha(a) values(300);
	insert into ora_share_range_ha values(230, 0.0001, 'aaaa3', 'bbbbbb3', '2012-2-2');	
	
	select * from test.share_range_ha;
	+------+--------+-------+---------+------------+
	| A    | B      | C     | D       | E          |
	+------+--------+-------+---------+------------+
	|    1 |    1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |      2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+
	
	delete from tmp;
	SELECT SPIDER_DIRECT_SQL('select * from SHARE_RANGE_HA', 'tmp', 'srv "ora_backend1_db1"');	
	select * from tmp;
	+------+------+-------+---------+------------+
	| a    | b    | c     | d       | e          |
	+------+------+-------+---------+------------+
	|    1 |  1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |    2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	+------+------+-------+---------+------------+
	delete from tmp;
	SELECT SPIDER_DIRECT_SQL('select * from SHARE_RANGE_HA', 'tmp', 'srv "ora_backend1_db2"');	
	select * from tmp;
	+------+--------+-------+---------+------------+
	| a    | b      | c     | d       | e          |
	+------+--------+-------+---------+------------+
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+
	delete from tmp;
	SELECT SPIDER_DIRECT_SQL('select * from SHARE_RANGE_HA', 'tmp', 'srv "ora_backend2_db1"');	
	select * from tmp;
	+------+--------+-------+---------+------------+
	| a    | b      | c     | d       | e          |
	+------+--------+-------+---------+------------+
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+
	delete from tmp;
	SELECT SPIDER_DIRECT_SQL('select * from SHARE_RANGE_HA', 'tmp', 'srv "ora_backend2_db2"');	
	select * from tmp;
	+------+------+-------+---------+------------+
	| a    | b    | c     | d       | e          |
	+------+------+-------+---------+------------+
	|    1 |  1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |    2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	+------+------+-------+---------+------------+
		
## mysqltest测试脚本

MySQL提供MySQL Test Framework作为功能测试环境，支持自动回归测试。

上面的测试案例已实现为mysqltest测试脚本。

从github下载脚本，并拷贝到mysql-test目录

	git clone git@github.com:ChengXiaoZ/spiderxtest.git
	
	cp spiderxtest /home/test/MariaDBserver/bld-ora/release/mysql-test/suite/

启动两个Oracle实例，参数见下

设置环境变量

	export PROXY_IP=127.0.0.1
	export PROXY_PORT=3306
	export ORA_BACKEND1_IP=192.168.226.1/orcl
	export ORA_BACKEND2_IP=192.168.226.1/orcl2
	export ORA_BACKEND1_PORT=1521
	export ORA_BACKEND2_PORT=1521
	export ORA_DATABASE=SYSTEM
	export ORA_USER=system
	export ORA_PWD=orcl123456	
	
运行脚本

单表测试案例
	
	./mysql-test-run.pl  spiderxtest.ora_single_table
	
多表测试案例
	
	./mysql-test-run.pl  spiderxtest.ora_multi_table
	
---

本文内容遵从CC版权协议, 可以随意转载, 但必须以超链接形式标明文章原始出处和作者信息及版权声明  
