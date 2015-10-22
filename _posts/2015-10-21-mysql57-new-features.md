---
title: MySQL 5.7新特性
layout: post
---

> MySQL 5.7终于要发布正式版本了，相比MySQL 5.6，MySQL 5.7除了提升性能外增加了非常多的新功能，具体可以参考[官方手册](http://dev.mysql.com/doc/refman/5.7/en/mysql-nutshell.html)，其中主要有在线DDL（相比5.6增加了索引重命名），空间数据类型索引，原生JSON数据类型，多源复制等。

#### 1. 特性变更

> 新版本中对老版本的一些特性进行了调整，有些不再支持，有些则不建议再使用，以下只列出一些比较常见的，具体可以参考手册。
>
##### 1) mysql\_install\_table

> mysql\_install\_table命令不建议在使用，建议改用mysqld --initialize，运行时会有以下提示
>
```
[WARNING] mysql_install_db is deprecated. Please consider switching to mysqld --initialize
```
> 使用方式基本没有变化，其命令格式大致如下
>
```bash
mysqld --initialize --user=mysql \
         --basedir=/opt/mysql/mysql \
	     --datadir=/opt/mysql/mysql/data
```
##### 2) InnoDB不能被禁用

> 由于系统表已经改用InnoDB，因此不能再禁用InnoDB引擎，--skip-innodb，--disable-innodb等配置将被忽略。
>
##### 3) YEAR(2)类型将不再支持

> YEAR(2)类型不再支持，需要升级为YEAR(4)，如果使用会提示以下错误
>
```
ERROR 1818 (HY000): Supports only YEAR or YEAR(4) column.
```
##### 4) 配置参数变化

> 参数storage\_engine变量替换为default\_storage\_engine

> 参数innodb\_use\_sys\_malloc 和innodb\_additional\_mem\_pool\_size不再支持

#### 2. 多源复制

> 多源复制是一个非常好用的特性，可以用在做数据的汇总——将多个MySQL的数据汇总到一个MySQL中，同时也可以利用多源复制提升MySQL复制的性能，在MySQL 5.7之前一只使用Mariadb 10的多源复制特性，其两者在使用和语法上稍有区别，这里只介绍MySQL 5.7中的多源复制语法。具体细节可以参考[官方手册](http://dev.mysql.com/doc/refman/5.7/en/replication-multi-source.html)

> 在使用多源复制特性前需要先修改MySQL存储master-info和relay-info的方式，即从文件存储改为表存储，修改配置文件如下
>
```ini
master_info_repository=TABLE
relay_log_info_repository=TABLE
```
> 同时也可以在线修改
>
```mysql
STOP SLAVE;
SET GLOBAL master_info_repository = 'TABLE';
SET GLOBAL relay_log_info_repository = 'TABLE';
```
> 添加一个新的复制源与直接使用CHANGE MASTER TO命令基本相同，只是需要给当前的MASTER使用FOR CHANNEL语句分配一个CHANNEL名字即可，例如以下的CHANGE MASTER TO语句
>
```mysql
CHANGE MASTER TO MASTER_HOST='master1', MASTER_USER='rpl', MASTER_PORT=3451, MASTER_PASSWORD='' MASTER_LOG_FILE='master1-bin.000006', MASTER_LOG_POS=628 FOR CHANNEL 'master-1';
>
#启动全部的复制
START SLAVE;
#启动某一个复制源的复制
START SLAVE FOR CHANNEL 'master-1';
>
#查看某个复制源的复制状态
SHOW SLAVE STATUS FOR CHANNEL 'master-1';
>
#通过performance_schema的相关表监控复制状态
SELECT * FROM replication_connection_status;
```

#### 3. 原生JSON数据类型

> MySQL 5.7中提供了对JSON数据类型的支持，并且其不仅是能够对JSON数据的格式进行验证，还可以支持一些类似与MongoDB的操作方式，以及对JSON数据进行索引。

> 例如建立一个包含JSON数据类型的表
>
```mysql
CREATE TABLE JsonData(
	id int NOT NULL AUTO_INCREMENT PRIMARY KEY,
	data json NOT NULL
) ENGINE = InnoDB;
```
> 向其中插入几行数据，并进行插叙
>
```mysql
INSERT INTO JsonData (data) VALUES 
('{"id":1, "name":"Fred"}'), ('{"id":2, "name":"Wilma"}'), 
('{"id":"3", "name":"Barney"}'), ('{"id":"4", "name":"Betty"}');
SELECT * FROM JsonData;
```
> 
> 结果如下
>
```
+----+-------------------------------+
| id | data                          |
+----+-------------------------------+
|  1 | {"id": 1, "name": "Fred"}     |
|  2 | {"id": 2, "name": "Wilma"}    |
|  3 | {"id": "3", "name": "Barney"} |
|  4 | {"id": "4", "name": "Betty"}  |
+----+-------------------------------+
4 rows in set (0.00 sec)
```
>
> 同时还可以查询JSON字段中的某个属性，例如以下查询
>
```mysql
SELECT id, JSON_EXTRACT(data, '$.id') FROM JsonData WHERE JSON_EXTRACT(data, '$.name') = 'Fred';
```
>
> 结果如下
>
```
+----+----------------------------+
| id | JSON_EXTRACT(data, '$.id') |
+----+----------------------------+
|  1 | 1                          |
+----+----------------------------+
1 row in set (0.00 sec)
```
>
> 另外可以对JSON数据的进行类似MongoDB中修改操作，如进行以下修改
>
```mysql
UPDATE JsonData SET data =  JSON_SET(data, '$.name', 'longlong') WHERE id = 1;
SELECT * FROM JsonData WHERE id = 1;
```
>
> 结果如下
>
```
+----+-------------------------------+
| id | data                          |
+----+-------------------------------+
|  1 | {"id": 1, "name": "longlong"} |
+----+-------------------------------+
1 row in set (0.00 sec)
```
>
> 此外还有JSON\_INSERT，JSON\_REPLACE，JSON\_APPEND，JSON\_REMOVE等（insert replace set的语意与Memcache的add replace set类似）可以对JSON数据进行修改而不需要从客户端修改后覆盖原来的数据，可以节约很多数据传输，提升操作的速度
>
> 另外还可以对JSON的某个属性建立索引，进行检索，对表进行以下的修改
>
```mysql
ALTER TABLE JsonData ADD name varchar(64) GENERATED ALWAYS AS (JSON_UNQUOTE(JSON_EXTRACT(data, '$.name'))) VIRTUAL;
ALTER TABLE JsonData ADD INDEX(name);
```
> 表结构变为
>
```mysql
CREATE TABLE `JsonData` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `data` json NOT NULL,
  `name` varchar(64) GENERATED ALWAYS AS (JSON_UNQUOTE(JSON_EXTRACT(data, '$.name'))) VIRTUAL,
  PRIMARY KEY (`id`),
  KEY `name` (`name`)
) ENGINE=InnoDB
```
>
> 分析以下语句的执行计划
>
```mysql
EXPLAIN SELECT * FROM JsonData WHERE name = 'longlong';
```
> 结果如下，看到可以使用到索引
>
```
           id: 1
  select_type: SIMPLE
        table: JsonData
   partitions: NULL
         type: ref
possible_keys: name
          key: name
      key_len: 67
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
```
>
#### 未完待续
