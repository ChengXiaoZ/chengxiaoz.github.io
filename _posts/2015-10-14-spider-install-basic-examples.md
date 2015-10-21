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

例如，在BACKEND节点的数据库db1有表t，连接用户名root，密码为空

	create table db1.t(a int , b varchar(10));

在PROXY节点，创建spider表

	create table st(a int, b varchar(10))
	ENGINE = SPIDER CONNECTION='wrapper "mysql", host "192.168.226.1", port "3310", database "db1", table "t", user "root"';	
	
在PROXY节点插入的数据，这些数据会被存储到BACKEND节点。
	
	insert into st values(1, 'aa');
	insert into st values(2, 'bb');
	select * from st;
	+------+------+
	| a    | b    |
	+------+------+
	|    1 | aa   |
	|    2 | bb   |
	+------+------+
	
COMMENT子句中的连接参数（用户名、密码和数据库）可以提取到Server中，在CREATE TABLE语句引用。

PROXY节点创建SERVER backend1_db1

	CREATE SERVER backend1_db1 
	FOREIGN DATA WRAPPER mysql	
	OPTIONS(	
	HOST '192.168.226.1', 
	PORT 3310, 
	DATABASE 'db1', 
	USER 'root', 
	PASSWORD '');

在创建spider表时，使用SERVER

	create table st2(a int, b varchar(10))
	ENGINE = SPIDER CONNECTION='wrapper "mysql", table "t", srv "backend1_db1"';

##典型应用案例

下面的案例环境由一个PROXY和2个BACKEND构成，演示了四种基本应用模式。

*  PROXY:IP127.0.0.1 端口3306 root用户密码空

*  BACKEND1:IP192.168.226.1 端口3310 root用户密码空

*  BACKEND2:IP192.168.226.1 端口3320 root用户密码空

定义mysql客户端登录alias

	alias backend1='mysql  --user=root --host=192.168.226.1 --port=3310'
	alias backend2='mysql  --user=root --host=192.168.226.1 --port=3320'
	alias proxy='mysql  --user=root --host=127.0.0.1 --port=3306'

在BACKEND1和BACKEND2，创建数据库db1和db2
	
	backend1 << EOF
	create database db1;
	create database db2;
	EOF
	
	backend2 << EOF
	create database db1;
	create database db2;
	EOF	
	
PROXY节点创建四个server，对应BACKEND1的db1、db2和BACKEND2的db1、db2。

	proxy << EOF
	CREATE SERVER backend1_db1 
	  FOREIGN DATA WRAPPER mysql 
	OPTIONS( 
	  HOST '192.168.226.1',
	  PORT 3310, 
	  DATABASE 'db1',
	  USER 'root',
	  PASSWORD ''
	);
	CREATE SERVER backend1_db2
	  FOREIGN DATA WRAPPER mysql 
	OPTIONS( 
	  HOST '192.168.226.1',
	  PORT 3310, 
	  DATABASE 'db2',
	  USER 'root',
	  PASSWORD ''
	);
	CREATE SERVER backend2_db1 
	  FOREIGN DATA WRAPPER mysql 
	OPTIONS( 
	  HOST '192.168.226.1',
	  PORT 3320, 
	  DATABASE 'db1',
	  USER 'root',
	  PASSWORD ''
	);
	CREATE SERVER backend2_db2
	  FOREIGN DATA WRAPPER mysql 
	OPTIONS( 
	  HOST '192.168.226.1',
	  PORT 3320, 
	  DATABASE 'db2',
	  USER 'root',
	  PASSWORD ''
	);
	EOF


### federation

数据直连，类似Oracle dblink或MariaDB FedX引擎。

对PROXY节点fed表的操作，映射到BACKEND1节点的db1.fed表。

![spider-fed]({{ "/css/pics/spider-deploy-fed.png"}})

BACKEND1节点

	backend1 << EOF
	create table db1.fed(a int , b float, c varchar(10), d char(10), e date);
	EOF

PROXY 节点

	proxy << EOF
	use test;
	create table fed(A int, B float, C varchar(10), D char(10), E date) 
	ENGINE = SPIDER CONNECTION='wrapper "mysql", table "FED" ,srv "backend1_db1"';
	insert into fed values(1, 1.1, 'aaaa1', 'bbbbbb1', null);
	insert into fed values(21, 2, 'aaaa2', 'bbbbbb2', '2011-1-1');
	insert into fed(a) values(300);
	insert into fed values(230, 0.0001, 'aaaa3', 'bbbbbb3', '2012-2-2');
	EOF
	
	proxy -e"select * from test.fed;"
	+------+--------+-------+---------+------------+
	| A    | B      | C     | D       | E          |
	+------+--------+-------+---------+------------+
	|    1 |    1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |      2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+

	backend1 -e"select * from db1.fed;"
	+------+--------+-------+---------+------------+
	| a    | b      | c     | d       | e          |
	+------+--------+-------+---------+------------+
	|    1 |    1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |      2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+	
	
### Sharding


数据分片，支持range和hash两种算法

**range**

PROXY节点share_range表分为part1和part2两个部分，part1映射到BACKEND1节点的db1.share_range表，part2映射到BACKEND2节点的db1.share_range表。

数据划分的依据是share_range表字段A的取值范围。

![spider-deploy-share-range]({{ "/css/pics/spider-deploy-share-range.png"}})

BACKEND1节点

	backend1 << EOF
	create table db1.share_range(a int , b float, c varchar(10), d char(10), e date);
	EOF
		
BACKEND2节点

	backend2 << EOF
	create table db1.share_range(a int , b float, c varchar(10), d char(10), e date);
	EOF

PROXY 节点

	proxy << EOF
	use test;
	create table share_range(A int, B float, C varchar(10), D char(10), E date)
	ENGINE=SPIDER partition by range(A) 
	(
	partition pt1 values less than (100) comment ' wrapper "mysql", table "share_range", srv "backend1_db1"',
	partition pt2 values less than maxvalue comment ' wrapper "mysql", table "share_range", srv "backend2_db1"');
	
	insert into share_range values(1, 1.1, 'aaaa1', 'bbbbbb1', null);
	insert into share_range values(21, 2, 'aaaa2', 'bbbbbb2', '2011-1-1');
	insert into share_range(a) values(300);
	insert into share_range values(230, 0.0001, 'aaaa3', 'bbbbbb3', '2012-2-2');	
	EOF	

	proxy -e"select * from test.share_range;"
	+------+--------+-------+---------+------------+
	| A    | B      | C     | D       | E          |
	+------+--------+-------+---------+------------+
	|    1 |    1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |      2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+

	backend1 -e"select * from db1.share_range;"
	+------+------+-------+---------+------------+
	| a    | b    | c     | d       | e          |
	+------+------+-------+---------+------------+
	|    1 |  1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |    2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	+------+------+-------+---------+------------+

	backend2 -e"select * from db1.share_range;"
	+------+--------+-------+---------+------------+
	| a    | b      | c     | d       | e          |
	+------+--------+-------+---------+------------+
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+

**hash**

PROXY节点share_key表分为part1和part2两个部分，part1映射到BACKEND1节点的db1.share_key表，part2映射到BACKEND2节点的db1.share_key表。

数据划分的依据是share_key表字段A的hash值。

![spider-deploy-share-key]({{ "/css/pics/spider-deploy-share-key.png"}})


BACKEND1节点

	backend1 << EOF
	create table db1.share_key(a int , b float, c varchar(10), d char(10), e date);
	EOF
		
BACKEND2节点
	
	backend2 << EOF	
	create table db1.share_key(a int , b float, c varchar(10), d char(10), e date);
	EOF
	
PROXY 节点

	proxy << EOF
	use test;
	create table share_key(A int, B float, C varchar(10), D char(10), E date)
	ENGINE=SPIDER partition by KEY(A) 
	(
	partition pt1 comment ' wrapper "mysql", table "SHARE_KEY", srv "backend1_db1"',
	partition pt2 comment ' wrapper "mysql", table "SHARE_KEY", srv "backend2_db1"');
	
	insert into share_key values(1, 1.1, 'aaaa1', 'bbbbbb1', null);
	insert into share_key values(21, 2, 'aaaa2', 'bbbbbb2', '2011-1-1');
	insert into share_key(a) values(300);
	insert into share_key(a) values(6);
	insert into share_key values(230, 0.0001, 'aaaa3', 'bbbbbb3', '2012-2-2');	
	EOF

	proxy -e"select * from test.share_key;"
	+------+--------+-------+---------+------------+
	| A    | B      | C     | D       | E          |
	+------+--------+-------+---------+------------+
	|    1 |    1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |      2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|    6 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+
	
	backend1 -e"select * from db1.share_key;"
	+------+------+-------+---------+------------+
	| a    | b    | c     | d       | e          |
	+------+------+-------+---------+------------+
	|    1 |  1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |    2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	+------+------+-------+---------+------------+

	backend2 -e"select * from db1.share_key;"
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

BACKEND1节点
	
	backend1 << EOF
	create table db1.fed_ha(a int , b float, c varchar(10), d char(10), e date);
	EOF
	
BACKEND2节点

	proxy << EOF
	backend2 << EOF
	create table db2.fed_ha(a int , b float, c varchar(10), d char(10), e date);
	EOF
	
PROXY 节点

	proxy << EOF
	use test;
	create table fed_ha(A int, B float, C varchar(10), D char(10), E date)
	ENGINE = SPIDER CONNECTION='wrapper "mysql", table "FED_HA", srv "backend1_db1 backend2_db2"';
	
	insert into fed_ha values(1, 1.1, 'aaaa1', 'bbbbbb1', null);
	insert into fed_ha values(21, 2, 'aaaa2', 'bbbbbb2', '2011-1-1');
	insert into fed_ha(a) values(300);
	insert into fed_ha values(230, 0.0001, 'aaaa3', 'bbbbbb3', '2012-2-2');	
	EOF

	proxy -e"select * from test.fed_ha;"
	+------+--------+-------+---------+------------+
	| A    | B      | C     | D       | E          |
	+------+--------+-------+---------+------------+
	|    1 |    1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |      2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+
	backend1 -e"select * from db1.fed_ha;"
	+------+--------+-------+---------+------------+
	| a    | b      | c     | d       | e          |
	+------+--------+-------+---------+------------+
	|    1 |    1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |      2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+
	backend2 -e"select * from db2.fed_ha;"
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

BACKEND1节点

	backend1 << EOF
	create table db1.share_range_ha(a int , b float, c varchar(10), d char(10), e date);
	create table db2.share_range_ha(a int , b float, c varchar(10), d char(10), e date);
	EOF
	
BACKEND2节点

	proxy << EOF
	create table db1.share_range_ha(a int , b float, c varchar(10), d char(10), e date);
	create table db2.share_range_ha(a int , b float, c varchar(10), d char(10), e date);
	EOF
	
PROXY 节点

	proxy << EOF
	use test;
	create table share_range_ha(A int, B float, C varchar(10), D char(10), E date)
	ENGINE=SPIDER partition by range(A) 
	(
	partition pt1 values less than (100) 
	comment ' wrapper "mysql", table "SHARE_RANGE_HA", srv "backend1_db1 backend2_db2"',
	partition pt2 values less than maxvalue 
	comment ' wrapper "mysql", table "SHARE_RANGE_HA", srv "backend2_db1 backend1_db2"');
	insert into share_range_ha values(1, 1.1, 'aaaa1', 'bbbbbb1', null);
	insert into share_range_ha values(21, 2, 'aaaa2', 'bbbbbb2', '2011-1-1');
	insert into share_range_ha(a) values(300);
	insert into share_range_ha values(230, 0.0001, 'aaaa3', 'bbbbbb3', '2012-2-2');	
	EOF
	
	proxy -e"select * from test.share_range_ha;"
	+------+--------+-------+---------+------------+
	| A    | B      | C     | D       | E          |
	+------+--------+-------+---------+------------+
	|    1 |    1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |      2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+
	backend1 -e"select * from db1.share_range_ha;"
	+------+------+-------+---------+------------+
	| a    | b    | c     | d       | e          |
	+------+------+-------+---------+------------+
	|    1 |  1.1 | aaaa1 | bbbbbb1 | NULL       |
	|   21 |    2 | aaaa2 | bbbbbb2 | 2011-01-01 |
	+------+------+-------+---------+------------+
	backend1 -e"select * from db2.share_range_ha;"
	+------+--------+-------+---------+------------+
	| a    | b      | c     | d       | e          |
	+------+--------+-------+---------+------------+
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+
	backend2 -e"select * from db1.share_range_ha;"
	+------+--------+-------+---------+------------+
	| a    | b      | c     | d       | e          |
	+------+--------+-------+---------+------------+
	|  300 |   NULL | NULL  | NULL    | NULL       |
	|  230 | 0.0001 | aaaa3 | bbbbbb3 | 2012-02-02 |
	+------+--------+-------+---------+------------+
	backend2 -e"select * from db2.share_range_ha;"
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
	
	mv spiderxtest /home/test/mariadb-10.0.9-spider-3.2/bld/release/mysql-test/suite/

启动BACKEND1和BACKEND2，root用户口令为空



设置环境变量

	export BACKEND1_IP=192.168.226.1
	export BACKEND1_PORT=3310
	export BACKEND2_IP=192.168.226.1
	export BACKEND2_PORT=3320
	export PROXY_IP=127.0.0.1
	export PROXY_PORT=3306
	
运行脚本

单表测试案例
	
	./mysql-test-run.pl  spiderxtest.single_table
	
多表测试案例
	
	./mysql-test-run.pl  spiderxtest.multi_table
	
---

本文内容遵从CC版权协议, 可以随意转载, 但必须以超链接形式标明文章原始出处和作者信息及版权声明  
