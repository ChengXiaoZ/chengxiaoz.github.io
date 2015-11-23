---
layout: post
title:  "Spider数据引擎实践与优化（四）实践案例（UDF）"
date:   2015-11-10
categories: Spider数据引擎实践与优化
---

* content
{:toc}

## SPIDER_DIRECT_SQL

SPIDER_DIRECT_SQL用于在远程服务器执行SQL语句。

语法：SPIDER_DIRECT_SQL('sql', 'tmp_table_list', 'parameters')

*  sql：SQL语句
*  parameters：连接参数
*  tmp_table_list（可选）：临时表。当sql为查询语句，可以建立临时表，存储查询结果。
*  返回值：成功返回1，失败返回0

例子

远程MySQL上的表 

	create table rt( a int);

创建连接用server

	CREATE SERVER node 
	  FOREIGN DATA WRAPPER mysql 
	OPTIONS( 
	  HOST '192.168.226.1',
	  PORT 3310, 
	  DATABASE 'test',
	  USER 'root',
	  PASSWORD ''
	);

非查询语句
	
	SELECT SPIDER_DIRECT_SQL('insert into test.rt values(1)', '', 'srv "node"');
	+----------------------------------------------------------------------+
	| SPIDER_DIRECT_SQL('insert into test.rt values(1)', '', 'srv "node"') |
	+----------------------------------------------------------------------+
	|                                                                    1 |
	+----------------------------------------------------------------------+

查询语句

	create temporary table tempt (a int)engine=myisam;
	SELECT SPIDER_DIRECT_SQL('SELECT * FROM rt', 'tempt', 'srv "node"');
	+--------------------------------------------------------------+
	| SPIDER_DIRECT_SQL('SELECT * FROM rt', 'tempt', 'srv "node"') |
	+--------------------------------------------------------------+
	|                                                            1 |
	+--------------------------------------------------------------+

---

	select * from tempt;
	+------+
	| a    |
	+------+
	|    1 |
	+------+
	
参考

[MariaDB Knowledge Base-SPIDER_DIRECT_SQL](https://mariadb.com/kb/en/mariadb/spider_direct_sql/)
	
---

本文内容遵从CC版权协议, 可以随意转载, 但必须以超链接形式标明文章原始出处和作者信息及版权声明  
