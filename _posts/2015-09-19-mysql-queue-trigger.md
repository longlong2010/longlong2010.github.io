---
title: 使用MySQL触发器同步数据
layout: post
---

> 之前尝试做异地间的Memcache数据同步，使用Gearman作为消息队列配合消息处理程序进行数据写入，发现在内网中测试的性能和通过异地之间的专线性能差距非常明显，在异地的情况下即使增加大量的消息处理程序同步的数据流量依然难以增加。由此判断可能是由于异地之间专线的延迟还是要远远高于内网，读写Memcache这种要求低延迟的IO操作会严重受到网络延迟的影响（如果Memcache支持批量写入或许能够缓解这一问题）。因此想到的解决办法就是避免使用高延迟网络做读写Memcache的操作，即将数据先通同步到另一端，然后在进行写入就可以避免网络延迟带来的影响。
>
> 而对于将数据同步到另一端，可以通过一些数据库的复制机制进行，一般会比较容易想到Redis，将数据案时间分List写入，另一端使用脚本循环利用BLPOP方法读取数据进行同步。这也是个很好的办法，难点在于需要自行开发同步数据的程序，程序需要有较好的性能和稳定性，同时还需要便于维护和监控。
>
> 然而说到数据复制机制，我首先想到的就是MySQL，使用Blackhole Engine将数据通过binlog的方式同步到另一端，可靠性要高于Redis——复制中断可恢复，数据也有binlog作为记录也是不错的。对于进行同步的程序有两种办法，一种是通过程序读取复制过来的binlog，这个方法的难点依然在于需要自行开发同步程序，难点与使用Redis进行同步类似，另外一种方法是使用触发器进行同步，每接收一条INSERT语句执行一条对应的Memcache操作即可，这里还需要用到MySQL libmemcached UDF。

#### 1. 安装MySQL libmemcached UDF

> MySQL的UDF安装非常简单，首先下载[libmemcached UDF源代码](https://launchpad.net/memcached-udfs|https://launchpad.net/memcached-udfs)，其依赖libmemcached，并且不能使用太新的版本，Centos6中使用的0.31是可以的。安装过程如下
>
```bash
./configure
make
make install
cp /usr/local/lib/libmemcached_functions_mysql.so /usr/lib64/mysql/plugin/
mysql -u root < sql/install_functions.sql
```

#### 2. 启动多个MySQL实例

> 为了便于测试和最后的部署可能都需要用到多MySQL实例，这里记录一下操作的方法，首先初始化两个MySQL数据目录
>
```bash
mysql_install_db --user=mysql --datadir=/var/lib/mysql/3306/
mysql_install_db --user=mysql --datadir=/var/lib/mysql/3307/
```
>
> 然后使用类似以下的配置文件，由于只需要使用Blackhole Engine，大多参数也无太大意义，设置小一些反而节约一些资源
>
```ini
>
[mysqld_multi]
mysqld  = /usr/bin/mysqld_safe
mysqladmin = /usr/bin/mysqladmin
[mysqld3306]
port		= 3306
socket		= /var/lib/mysql/3306/mysqld.sock
datadir		= /var/lib/mysql/3306
key_buffer_size = 16M
max_allowed_packet = 1M
table_open_cache = 64
sort_buffer_size = 512K
net_buffer_length = 8K
read_buffer_size = 256K
read_rnd_buffer_size = 512K
myisam_sort_buffer_size = 8M
log-bin=mysql-bin
binlog_format=mixed
server-id	= 3306
innodb_data_file_path = ibdata1:10M:autoextend
innodb_buffer_pool_size = 16M
innodb_additional_mem_pool_size = 2M
innodb_log_file_size = 5M
innodb_log_buffer_size = 8M
innodb_flush_log_at_trx_commit = 2
innodb_lock_wait_timeout = 50
[mysqld3307]
port		= 3307
socket		= /var/lib/mysql/3307/mysqld.sock
datadir		= /var/lib/mysql/3307
key_buffer_size = 16M
max_allowed_packet = 1M
table_open_cache = 64
sort_buffer_size = 512K
net_buffer_length = 8K
read_buffer_size = 256K
read_rnd_buffer_size = 512K
myisam_sort_buffer_size = 8M
log-bin=mysql-bin
binlog_format=mixed
server-id	= 3307
innodb_data_file_path = ibdata1:10M:autoextend
innodb_buffer_pool_size = 16M
innodb_additional_mem_pool_size = 2M
innodb_log_file_size = 5M
innodb_log_buffer_size = 8M
innodb_flush_log_at_trx_commit = 2
innodb_lock_wait_timeout = 50
```
>
> 启动多实例的方法如下，启动两个实例
>
```bash
mysqld_multi start 3306
mysqld_multi start 3307
```
>

#### 3. 初始化libmemcached UDF

> 在MySQL启动后需要手动初始化libmemcached UDF，设置对应的Memcache服务器，同时设置分布式的散列方式，初始化脚本如下
>
```sql
SELECT memc_servers_set('127.0.0.1:11211,127.0.0.1:11212');
SELECT memc_servers_behavior_set('MEMCACHED_BEHAVIOR_DISTRIBUTION', 'MEMCACHED_DISTRIBUTION_CONSISTENT_KETAMA');
SELECT memc_servers_behavior_set('MEMCACHED_BEHAVIOR_KETAMA_WEIGHTED', 1);
```
>
> 注意，每次MySQL启动时都需要进行以上的初始化，如下测试一下是否初始化成功，返回1则说明写入成功
>
```sql
SELECT memc_set('abc', '123');
+------------------------+
| memc_set('abc', '123') |
+------------------------+
|                      1 |
+------------------------+
>
SELECT memc_get('abc');
+-----------------+
| memc_get('abc') |
+-----------------+
| 123             |
+-----------------+
```

#### 4. libmemcached UDF修改支持自定义flags
>
> libmemcached UDF有一个很大的缺点，就是Memcache的flags参数写死为0，而我们在同步的过程中需要将原来的flags值也进行同步，因此需要对libmemcached UDF的代码进行一点的调整，这里只改了set方法，其他的方法修改方式都是类似的。
>
```diff
diff -Nur memcached_functions_mysql-1.1/src/args.c memcached_functions_mysql-1.1.patched/src/args.c
--- memcached_functions_mysql-1.1/src/args.c    2009-11-11 10:56:52.000000000 +0800
+++ memcached_functions_mysql-1.1.patched/src/args.c    2015-09-19 10:11:59.062512226 +0800
@@ -120,6 +120,30 @@
        case MEMC_ADD_BY_KEY:
        case MEMC_REPLACE:
        case MEMC_REPLACE_BY_KEY:
+    {
+       
+         if (args->arg_count >= max_args - 1)
+         {
+                 if (args->arg_type[max_args - 2] == STRING_RESULT)
+                 {
+                         container->flags= (uint32_t)atoi(args->args[max_args - 2]);
+                 }
+                 else if (args->arg_type[max_args - 2] == INT_RESULT)
+                 {
+                         container->flags= *((uint32_t*)args->args[max_args -2]);
+                 }
+                 else
+                 {
+                         container->flags= (uint32_t)0;
+                 }
+
+         }
+         else
+         {
+                 container->flags= (uint32_t) 0;
+         }
+         fprintf(stderr, "flags %d\n", (int) container->expiration);
+       }
        case MEMC_APPEND:
        case MEMC_APPEND_BY_KEY:
        case MEMC_PREPEND:
diff -Nur memcached_functions_mysql-1.1/src/common.h memcached_functions_mysql-1.1.patched/src/common.h
--- memcached_functions_mysql-1.1/src/common.h  2009-11-15 06:53:23.000000000 +0800
+++ memcached_functions_mysql-1.1.patched/src/common.h  2015-09-19 09:50:30.480512194 +0800
@@ -11,6 +11,7 @@
 struct memc_function_st {
   unsigned int offset;
   time_t expiration;
+  uint32_t flags;
   memcached_st memc;
   memcached_result_st results;
   memcached_string_st *stats_string;
diff -Nur memcached_functions_mysql-1.1/src/set.c memcached_functions_mysql-1.1.patched/src/set.c
--- memcached_functions_mysql-1.1/src/set.c     2009-11-11 10:56:52.000000000 +0800
+++ memcached_functions_mysql-1.1.patched/src/set.c     2015-09-19 10:01:36.611512903 +0800
@@ -30,7 +30,7 @@
   unsigned int count;
   memc_function_st *container;
> 
-  container= prepare_args(args, message, MEMC_SET, 2, 3);
+  container= prepare_args(args, message, MEMC_SET, 2, 4);
   if (container == NULL)
     return 1;
> 
@@ -67,7 +67,7 @@
   rc= memcached_set(&container->memc,
                     args->args[0], (size_t)args->lengths[0],
                     args->args[1], (size_t)args->lengths[1],
-                    container->expiration, (uint16_t)0);
+                    container->expiration, container->flags);
> 
   return (rc != MEMCACHED_SUCCESS) ? (long long) 0 : (long long) 1;
 }
```
>
> 如此调整后memc\_set方法的第三个参数是flags，第四个参数是过期时间，不传都默认为0。

#### 5. PHP Memcached模块中增加获取写入原始数据的方法

> 由于PHP Memcached模块中会对保存的数据进行如序列化、压缩等处理，而在同步时UDF很难做到一样的处理方法，因此需要将PHP Memcached模块处理后的数据获取出来，我们在PHP Memcached模块中增加一个getPayload的方法来实现这一功能，其实现如下
>
```c++
static zend_function_entry memcached_class_methods[] = {
	//...
	MEMC_ME(getPayload,         arginfo_getPayload)
	{ NULL, NULL, NULL }
};
>
ZEND_BEGIN_ARG_INFO_EX(arginfo_getPayload, 0, 0, 1)
    ZEND_ARG_INFO(0, value)
    ZEND_ARG_INFO(1, flags)
ZEND_END_ARG_INFO()
>
static PHP_METHOD(Memcached, getPayload)
{
    zval* zflags = NULL;
    uint32_t flags;
    zval* value;
    char  *payload;
    size_t payload_len;
>
    MEMC_METHOD_INIT_VARS;
>
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z|z", &value, &zflags) == FAILURE) {
        return;
    }
>
    MEMC_METHOD_FETCH_OBJECT;
>
    if (i_obj->compression) {
        flags |= MEMC_VAL_COMPRESSED;
    }
>
    if (i_obj->serializer == SERIALIZER_IGBINARY) {
        flags |= MEMC_VAL_IGBINARY;
    }
>
    payload = php_memc_zval_to_payload(value, &payload_len, &flags TSRMLS_CC);
>
    if (zflags != NULL) {
        convert_to_long(zflags);
        ZVAL_LONG(zflags, flags);
    }
    RETURN_STRINGL(payload, payload_len, 0);
}
```
>
> 其使用效果如下
>
```php
<?php
$mem = new Memcached();
var_dump($mem->getPayload(array(), $flags), $flags);
```
>
> 输入如下
>
```
string(6) "a:0:{}"
int(1)
```

#### 5. 建立同步的数据表盒触发器
>
> 建立用于同步的表，其表结构如下
>
```sql
CREATE TABLE `MemcQueue` (
  `k` varchar(250) NOT NULL,
  `expiration` int(11) NOT NULL,
  `flags` int(10) unsigned NOT NULL,
  `v` mediumblob NOT NULL,
  PRIMARY KEY (`k`)
) ENGINE=BLACKHOLE DEFAULT CHARSET=latin1
```
>
> 在从库中建立触发器
>
```sql
DELIMITER |
CREATE TRIGGER queue_trigger 
AFTER INSERT ON MemcQueue
FOR EACH ROW BEGIN
	SET @mm = memc_set(new.k, new.v, new.flags, new.expiration);		
END |
```

#### 6. 测试效果
>
> 测试使用脚本如下
>
```php
<?php
$key = 'test';
$expire = 600;
>
$pdo = new PDO('mysql:host=127.0.0.1;dbname=test', 'root', '');
$stat = $pdo->prepare("INSERT INTO MemcQueue VALUES (?, ?, ?, ?)");
>
$mem = new Memcached();
$mem->setOption(Memcached::DISTRIBUTION_CONSISTENT, 1);
$mem->setOption(Memcached::OPT_LIBKETAMA_COMPATIBLE, 1);
$mem->addServers(array(array('127.0.0.1', 11211), array('127.0.0.1', 11212)));
$v = array();
for ($i = 0; $i < 5; $i++) {
	$v[] = $i;
}
$payload = $mem->getPayload($v, $flags);
for ($i = 0; $i < 10000; $i++) {
	$stat->execute(array($key, $expire, $flags, $payload));
}
var_dump($mem->get($key));
?>
```
>
> 执行效果如下
>
```
time php mem.php 
array(5) {
  [0]=>
  int(0)
  [1]=>
  int(1)
  [2]=>
  int(2)
  [3]=>
  int(3)
  [4]=>
  int(4)
}
>
real	0m2.141s
user	0m0.105s
sys		0m0.131s
```
