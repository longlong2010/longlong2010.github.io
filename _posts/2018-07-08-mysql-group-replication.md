---
title: MySQL Group Replication集群
layout: post
---

> 传统的MySQL的主从复制是主库通过binlog的形式将数据更新发送给从库，其中包含异步的复制，即主库完全不关心从库复制的情况就更新自身的数据，另外半同步复制则是主库需要等待至少一个从库将发送的binlog写入到其relaylog中才会更新自身的数据。
>
> 在此种复制模式下当主库发生故障时，从库的数据可能会存在不一致的问题，此时确定数据完全正确的从库（如果使用异步复制则可能会导致数据直接丢失，而不存在数据正确的从库）再重新调整各个从库之间的复制关系是个相对复杂的过程（尽管有一些工具能够自动处理这一过程）。
>
#### 传统的异步复制 
<img src="https://dev.mysql.com/doc/refman/8.0/en/images/async-replication-diagram.png" width="600"/>
#### 5.5中加入的半同步复制
<img src="https://dev.mysql.com/doc/refman/8.0/en/images/semisync-replication-diagram.png" width="600"/>
>
> 在MySQL 5.7中增加了Group Replication的复制方式，其实现了一种去中心化的复制方式，每个节点都具有相同的数据和地位（在Single-Primary模式下会决议确定一个写入的入口节点），当事务提交时，需要由多数节点决议来确定事务是否进行提交，以保证复制组内数据的一致性。当某个节点发生故障时Group Replication会自动剔除这个节点，如果故障的是Primary节点，则其他节点会决议来确定一个新的Primary节点。
>
#### 5.7中加入的组复制
>
<img src="https://dev.mysql.com/doc/refman/8.0/en/images/gr-replication-diagram.png" width="600"/>

#### 1. Group Replication的配置要求
> 使用Group Replication功能是需要保证满足以下条件
1. MySQL版本需要在5.7.17以上
1. 存储引擎全部使用InnoDB
1. 全部表都必须有主键
1. 网络为IPV4网络
>
> 使用Group Replication功能需要进行以下配置
>
```ini
#全部节点开启binlog
log-bin=binlog
>
#binlog格式必须为row格式
binlog-format=row
>
#开启全局事务标识
gtid-mode=ON
>
#复制信息存入系统表中
master-info-repository=TABLE
relay-log-info-repository=TABLE
>
transaction-write-set-extraction=XXHASH64
>
#开启复制日志
log-slave-updates=ON
>
binlog-checksum=NONE
```

#### 2. Group Replication的配置
> 配置组复制至少配置3个节点，需要进行的配置如下
>
```ini
#复制组的UUID标识
loose-group_replication_group_name="17da1ce4-040d-462e-a444-bfbf55a0f948"
#启动后不自动加入复制组
loose-group_replication_start_on_boot=off
#实例的组复制IP和端口
loose-group_replication_local_address= "127.0.0.1:24901"
#复制组的成员IP和端口，不能使用主机名，用于新的成员加入复制组
loose-group_replication_group_seeds= "127.0.0.1:24901,127.0.0.1:24902,127.0.0.1:24903"
#启动后初始化复制组，仅可在一个节点上设置为on
loose-group_replication_bootstrap_group=off
#单Primary节点模式
group_replication_single_primary_mode=on
```
> 配置组复制使用的帐号，并开启组复制，需要注意每个节点的用户和密码需要相同
>
```sql
SET SQL_LOG_BIN=0;
CREATE USER rpl_user@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO rpl_user@'%';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;
CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='password' FOR CHANNEL 'group_replication_recovery';
```

#### 3. 启动Group Replication
> 连接进入其中一个MySQL实例，执行以下操作开启组复制
>
```sql
INSTALL PLUGIN group_replication SONAME 'group_replication.so';
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;
```
>
> 在另外的实例中执行开启组复制
```sql
START GROUP_REPLICATION;
```
>
> 完成后查看状态可以看到
```sql
SELECT * FROM performance_schema.replication_group_members;
```
>
```
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| group_replication_applier | 01b7de10-81e0-11e8-b5e0-080027f3c535 | archlinux   |       24803 | ONLINE       |
| group_replication_applier | f5e1928b-81df-11e8-bda0-080027f3c535 | archlinux   |       24801 | ONLINE       |
| group_replication_applier | fa8af292-81df-11e8-8566-080027f3c535 | archlinux   |       24802 | ONLINE       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
```

#### 4. 多Primary节点模式（Multi-Primary）
>
> 如果希望能够从任何一个节点都能进行读写操作，则需要将全部节点都增加配置
```ini
group_replication_single_primary_mode=off
```
> 注意全部节点的这个配置必须全部都是相同的，否则配置不一致的节点启动组复制时会发生以下报错
```
Variables such as single_primary_mode or enforce_update_everywhere_checks must have the same value on every server in the group.
```
> 另外在两个不同的节点同时进行锁定（SELECT FOR UPDATE），由于节点之间的锁时不能共享的，会造成两个节点上都能够锁定成功，但在事务提交时，后提交的事务会发生失败。
```
ERROR 1180 (HY000): Got error 149 during COMMIT
```
> 为此还是建议直接使用Single-Primary模式。

#### 5. 总结
>
> Group Replication主要解决了传统主从复制方式在主库故障时可能数据丢失以及从库数据不一致并且切换复杂的问题，简化了MySQL集群的高可用设计，同时使用InnoDB Cluster结合MySQL Router便可以实现对于后端Primary节点和Secondary节点的自动选择，并且通过两个MySQL Router配置Keepalived后，就可以实现对后端MySQL Group Replication集群的透明访问。
