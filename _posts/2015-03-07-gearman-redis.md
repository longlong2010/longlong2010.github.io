---
title: 用Redis做Gearman的持久化队列
layout: post
---

> Gearman很久之前就开始支持持久化队列，但用sqlite进行持久化性能实在是着急，之后支持了MySQL后才感觉靠谱一些，但对于高并发的情况下需要反复的写入和删除数据——消息到达需要写入消息，处理完成后删除消息，对MySQL的性能有着非常高的要求。而在Gearman的源代码中已经提供了将Redis作为持久化队列的代码，但相关的代码有较多的bug，感觉目前只是个隐藏功能，如果不对代码进行处理是没法正常使用的。

#### 1. 修复代码中的bug

> 其中主要修复了默认只能够连接本地Reids、启动时不能够正确获取队列中的全部任务、不能正确解析获取到的任务标识和一些段错误，增加了断线重连的机制。感觉官方根本就没有认真的写这个模块，更没有好好的进行测试。补丁的代码如下，针对最新的1.1.12版本。希望官方之后能修复自己的一大堆bug。 
>
```diff
diff -ruNa gearmand-1.1.12/libgearman-server/plugins/queue/redis/queue.cc gearmand-1.1.12.patch/libgearman-server/plugins/queue/redis/queue.cc
--- gearmand-1.1.12/libgearman-server/plugins/queue/redis/queue.cc      2014-02-12 08:05:28.000000000 +0800
+++ gearmand-1.1.12.patch/libgearman-server/plugins/queue/redis/queue.cc        2014-06-18 10:57:02.147575821 +0800
@@ -85,6 +85,7 @@
   ~Hiredis();
> 
   gearmand_error_t initialize();
+  bool init_redis();
> 
   redisContext* redis()
   {
@@ -113,10 +114,15 @@
 {
 }
> 
+bool Hiredis::init_redis() {
+  int service_port= atoi(service.c_str());
+  _redis = redisConnect(server.c_str(), service_port);
+  return _redis != NULL;
+}
+
 gearmand_error_t Hiredis::initialize()
 {
-  int service_port= atoi(service.c_str());
-  if ((_redis= redisConnect("127.0.0.1", service_port)) == NULL)
+  if (!init_redis())
   {
     return gearmand_gerror("Could not connect to redis server", GEARMAND_QUEUE_ERROR);
   }
@@ -148,7 +154,7 @@
                         const char *function_name,
                         size_t function_name_size)
 {
-  key.resize(function_name_size +unique_size +GEARMAND_QUEUE_GEARMAND_DEFAULT_PREFIX_SIZE +4);
+  key.resize(function_name_size +unique_size +GEARMAND_QUEUE_GEARMAND_DEFAULT_PREFIX_SIZE +2);
   int key_size= snprintf(&key[0], key.size(), GEARMAND_KEY_LITERAL,
                          GEARMAND_QUEUE_GEARMAND_DEFAULT_PREFIX,
                          (int)function_name_size, function_name,
@@ -202,13 +208,19 @@
   build_key(key, unique, unique_size, function_name, function_name_size);
   gearmand_log_debug(GEARMAN_DEFAULT_LOG_PARAM, "hires key: %u", (uint32_t)key.size());
> 
-  redisReply *reply= (redisReply*)redisCommand(queue->redis(), "SET %b %b", &key[0], key.size(), data, data_size);
+  redisReply *reply= (redisReply*)redisCommand(queue->redis(), "SET %s %b", &key[0], data, data_size);
   gearmand_log_debug(GEARMAN_DEFAULT_LOG_PARAM, "got reply");
   if (reply == NULL)
   {
-    return gearmand_log_gerror(GEARMAN_DEFAULT_LOG_PARAM, GEARMAND_QUEUE_ERROR, "failed to insert '%.*s' into redis", key.size(), &key[0]);
+    if (!queue->init_redis())
+    {
+      return gearmand_log_gerror(GEARMAN_DEFAULT_LOG_PARAM, GEARMAND_QUEUE_ERROR, "failed to insert '%.*s' into redis", key.size(), &key[0]);
+    }
+  } 
+  else 
+  { 
+    freeReplyObject(reply);
   }
-  freeReplyObject(reply);
> 
   return GEARMAND_SUCCESS;
 }
@@ -231,12 +243,16 @@
   std::vector<char> key;
   build_key(key, unique, unique_size, function_name, function_name_size);
> 
-  redisReply *reply= (redisReply*)redisCommand(queue->redis(), "DEL %b", &key[0], key.size());
+  redisReply *reply= (redisReply*)redisCommand(queue->redis(), "DEL %s", &key[0]);
   if (reply == NULL)
   {
-    return GEARMAND_QUEUE_ERROR;
+    if (!queue->init_redis()) {
+      return GEARMAND_QUEUE_ERROR;
+    }
+  }
+  else {
+    freeReplyObject(reply);
   }
-  freeReplyObject(reply);
> 
   return GEARMAND_SUCCESS;
 }
@@ -252,7 +268,7 @@
>    
   gearmand_info("hiredis replay start");
> 
-  redisReply *reply= (redisReply*)redisCommand(queue->redis(), "KEYS %s", GEARMAND_QUEUE_GEARMAND_DEFAULT_PREFIX);
+  redisReply *reply= (redisReply*)redisCommand(queue->redis(), "KEYS %s*", GEARMAND_QUEUE_GEARMAND_DEFAULT_PREFIX);
   if (reply == NULL)
   {
     return gearmand_gerror("Failed to call KEYS during QUEUE replay", GEARMAND_QUEUE_ERROR);
@@ -265,9 +281,7 @@
     char unique[GEARMAN_MAX_UNIQUE_SIZE];
> 
     char fmt_str[100] = "";    
-    int fmt_str_length= snprintf(fmt_str, sizeof(fmt_str), "%%%ds-%%%ds-%%%ds",
-                                 int(GEARMAND_QUEUE_GEARMAND_DEFAULT_PREFIX_SIZE),
-                                 int(GEARMAN_FUNCTION_MAX_SIZE),
+    int fmt_str_length= snprintf(fmt_str, sizeof(fmt_str), "%%[^-]-%%[^-]-%%%ds",
                                  int(GEARMAN_MAX_UNIQUE_SIZE));
     if (fmt_str_length <= 0 or size_t(fmt_str_length) >= sizeof(fmt_str))
     {
@@ -293,7 +307,7 @@
     (void)(add_fn)(server, add_context,
                    unique, strlen(unique),
                    function_name, strlen(function_name),
-                   get_reply->str, get_reply->len,
+                   strndup(get_reply->str, get_reply->len), get_reply->len,
                    GEARMAN_JOB_PRIORITY_NORMAL, 0);
     freeReplyObject(get_reply);
   }
```

#### 2.模块的安装

> 如果安装了hiredis库，则在安装Gearman时就会自动加载Redis持久化队列模块，hireids的代码在Redis代码的deps/hiredis目录下，直接编译安装即可。正确的安装了Redis持久化队列模块后，运行gearmand有以下的提示，则证明模块已经正常被安装和加载。
>
	redis:
	--redis-server arg    Redis server
	--redis-port arg      Redis server port/service

#### 3.运行的效果

> 启动一个gearmand并连接相应的Redis实例，如下启动一个监听4730端口的gearmand并连接14730端口的Redis，这里端口的设计方法是方便启动多个gearmand时批量进行管理，默认对应Redis的端口是gearmand端口加10000。gearmand在启动时就会和Redis进行连接，如果连接失败则启动会失败。
>
	/usr/local/sbin/gearmand -d -p 4730 -u nobody -P /var/run/gearmand/gearmand.4730.pid -t 4 -j 3 -l /var/log/gearmand/gearmand.4730 -q redis --redis-server 127.0.0.1 --redis-port 14730
>
> 通过gearman命令行工具向test队列写入一个异步消息，消息的内容为123，然后连接Redis进行查看，最后再使用gearadmin查看队列的状况，重启gearmand再查看队列的状况，最后启动一个Worker获取消息
>
```bash
#写入消息
echo "123" | gearman -f test -b
#连接Redis
/opt/redis/bin/redis-cli  -p 14730
#查看Redis中的内容，发现有一条记录
127.0.0.1:14730> keys *
1) "_gear_-test-205d3b6e-c479-11e4-9ad2-00237d29f08a"
#查看数据，发现是发送过来的123
127.0.0.1:14730> get _gear_-test-205d3b6e-c479-11e4-9ad2-00237d29f08a
"123\n"
127.0.0.1:14730>
#查看队列的状况，发现有一个消息
gearadmin --status
test	1	0	0
.
#重启gearmand
/etc/init.d/gearmand.4730 restart
#再查看队列的状况，发现消息仍然存在
gearadmin --status
test	1	0	0
.
#启动一个worker，发现正常的获得到了发送的消息
gearman -w -f test
123
```
> 如此就完成了对持久化正确性的验证，由于Redis是内存的数据库因此相比MySQL性能会有一定的保证，之前测试的结果是相比不开启持久化队列性能大概要下降一半左右。不过为了保证可靠性必然要付出一定的性能代价。
