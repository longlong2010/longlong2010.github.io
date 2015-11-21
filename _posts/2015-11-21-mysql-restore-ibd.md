---
title: MySQL 使用ibd文件恢复InnoDB表
layout: post
---

>  MySQL的备份一般是使用mysqldump类工具备份文本格式的数据，也可以使用xtrabackup等工具备份原始格式的数据文件，两种方法也各有优缺点。

> 文本格式的数据文件可以用作数据的分析，也比较方便做数据的部分恢复，其实很多时候我们只是需要恢复某一行或几行的数据，但缺点就是数据备份和恢复的过程较慢，不过如果数据在100GB以下使用多线程的mydumper进行备份和恢复速度还是基本可以接受的。

> 二进制格式的备份由于是直接复制文件，所以速度只与数据的大小有关，通常速度很快，并且对InnoDB可以做在线的备份，恢复也比较方便，缺点则是不能够恢复单个表和确定的某些行的数据。

#### 1. 遇到的问题

> 由于数据被错误删除，同时备份又没有使用相应的备份工具备份完整的MySQL数据目录，只是备份了数据库目录里的frm表结构文件和ibd数据文件，在这种情况下尝试进行数据的恢复。

#### 2. 操作方法

> 比较保险的操作方法是新建一个MySQL的数据目录，并启动另外一个MySQL实例，将备份的数据库目录之间复制到MySQL的数据目录中，启动后对待修复的数据表进行操作时出现以下错误
>
```
[Warning] InnoDB: Cannot open table xxxx/yyyy from the internal data dictionary of InnoDB though the .frm file for the table exists. See http://dev.mysql.com/doc/refman/5.6/en/innodb-troubleshooting.html for how you can resolve the problem.
```

> 参考了报错中提示的文档[http://dev.mysql.com/doc/refman/5.6/en/innodb-troubleshooting-datadict.html](http://dev.mysql.com/doc/refman/5.6/en/innodb-troubleshooting-datadict.html)，提供了一种操作的方法，即在另外一个库中新建一个相同的表，然后将其从表空间中移除，然后将待修复表的ibd文件复制到当前的数据库目录中，然后再将此表导入表空间。如文档中给出的例子
>
```mysql
#新建数据库
CREATE DATABASE sakila;
>
#进入新建的数据库
USE sakila;
>
#建与待修复表相同的表
CREATE TABLE actor (
   actor_id SMALLINT UNSIGNED NOT NULL AUTO_INCREMENT,
   first_name VARCHAR(45) NOT NULL,
   last_name VARCHAR(45) NOT NULL,
   last_update TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
   PRIMARY KEY  (actor_id),
   KEY idx_actor_last_name (last_name)
)ENGINE=InnoDB DEFAULT CHARSET=utf8;
>
#将此表移出表空间
ALTER TABLE sakila.actor DISCARD TABLESPACE;
>
```
> 复制数据文件
>
```bash
cp /backup_directory/actor.ibd path/to/mysql-5.6/data/sakila/
```
> 重新将其导入表空间
>
```mysql
ALTER TABLE sakila.actor IMPORT TABLESPACE;
```

> 如果一切正常，应该就可以对修复后的表进行查询操作了，将查询出的数据恢复到损坏的MySQL实例中即可。

