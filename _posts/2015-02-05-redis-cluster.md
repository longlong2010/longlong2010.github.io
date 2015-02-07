---
title: Redis Cluster配置
layout: post
---
> Redis 3.0终于要发布了，目前是RC3并且很可能是最后一个RC版本，2月中就可能会发布正式版。关注这个功能很久了，但一直到离职都没有测试过，之前是用一致性散列算法(ketama)实现了基于客户端的分布式方案，优点是比较简单灵活，配置也比较简单，但缺点就是一些同时操作多个键的命令将无法使用(如mget move等)，同时扩容时数据的迁移也是个麻烦的过程(如果不使用持久化则还好)。
>
> 官方的文档可以参考[这里](http://redis.io/topics/cluster-tutorial)

#### 1. Cluster基本配置
> 基本配置文件如下，其中nodes.6380.conf是Redis自己生成的配置文件，这里只是指定生成文件的路径。
>
>	port 6380
	cluster-enabled yes
	cluster-config-file nodes.6380.conf
	cluster-node-timeout 5000
>
> 其中cluster-node-timeout是一个比较关键的参数，当一个节点在超时时间过后依然无响应，将被判定为失效并由其他的节点提到，如果没有能够替代的节点则写入会失败。

#### 2. Cluster启动
> 每个Redis实例独立进行启动，在启动后其除了监听本来配置的端口外还会监听一个设置的端口+10000的端口作为Cluster的各个节点通信使用。如port设置为6380，则还会开启16380端口。
>
	tcp	0	0 0.0.0.0:6380		0.0.0.0:*	LISTEN
	tcp	0	0 0.0.0.0:16380		0.0.0.0:*	LISTEN
>
> 第一次创建Redis Cluster时需要进行一次初始化操作。初始化的命令需要Ruby支持，并且需要安装Ruby Redis模块，Redis Cluster需要至少三个主节点，所以如果需要启动主从需要注意节点的数量
>
	pacman -S ruby
	gem install redis
	src/redis-trib.rb create 127.0.0.1:6380 127.0.0.1:6381 127.0.0.1:6382 127.0.0.1:6383
>
	#输出如下
	>>> Creating cluster
	Connecting to node 127.0.0.1:6379: OK
	Connecting to node 127.0.0.1:6380: OK
	Connecting to node 127.0.0.1:6381: OK
	Connecting to node 127.0.0.1:6382: OK
	>>> Performing hash slots allocation on 4 nodes...
	Using 4 masters:
	127.0.0.1:6379
	127.0.0.1:6380
	127.0.0.1:6381
	127.0.0.1:6382
	M: db0c19d0d48988ef7971e677a9cb9382c6475f17 127.0.0.1:6379
	slots:0-4095 (4096 slots) master
	M: 1b9bbd93c0c80834be3691b39226ecb186691435 127.0.0.1:6380
	slots:4096-8191 (4096 slots) master
	M: 225e92944e0a8001bcec510b9f00e8e0d75d4fab 127.0.0.1:6381
	slots:8192-12287 (4096 slots) master
	M: 276d31fbce83e0e603419d4abffdcffdce1413d5 127.0.0.1:6382
	slots:12288-16383 (4096 slots) master
>
> 同时用redis-trib.rb工具还可以对Cluster进行增加节点，删除节点，数据转移等操作。

#### 3. 读写Cluster
> 对于Cluster的读写依然可以使用redis-cli，连接Cluster中的任何一个实例都可以，但需要加参数-c使其使用读写Cluster的协议。通过telnet测试发现Redis Cluster本质上也需要客户端进行性支持才能实现，基本上相当于一致性散列算法由Redis完成。
>
> 例如以下的命令会得到反馈，需要连接另一个实例
>
	get abc
	-MOVED 7638 127.0.0.1:6380
	set abc 123
	-MOVED 7638 127.0.0.1:6380
>
> 即客户端需要根据返回的信息判断是成功获得了数据还是需要重新连接。相当于每个Redis实例都是一个代理，但只负责找到地址不负责连接和传递数据。

#### 4. 故障转移及高可用
> Redis Cluster只有在有从节点时才能支持故障转移，如果全部为主节点，当一个节点失效时会返回如下的提示
>
	(error) CLUSTERDOWN The cluster is down. Use CLUSTER INFO for more information

#### 5. 增加节点和数据迁移

> 个人感觉这个是Redis Cluster相比客户端实现分布式最大的一个优势，手动迁移数据的过程是非常麻烦的，同时如果不进行停机还会存在一些冲突的问题。
