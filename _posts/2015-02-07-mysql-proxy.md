---
title: MySQL-Proxy 应用
layout: post
---

> MySQL-Proxy是个进展非常缓慢的开源项目，目前还只有[0.8.5 alpha](http://dev.mysql.com/downloads/mysql-proxy/)版本，不过根据之前的使用经验还算是比较稳定，性能上还算说得过去。个人比较推荐的类似数据库的负载均衡软件最好都是和Web服务装在一起，可以有效的防止流量都通过代理而产生的流量集中和单点问题。不过如果只是简单的做MySQL的自动故障转移和负载均衡则比较推荐用Haproxy，相比MySQL-Proxy更加通用和简单，性能也要更好一些。

#### 1. MySQL-Proxy的安装

> 安装主要依赖libevent、lua 5.1、glib2，注意Lua 5.2是不行的。编译比较简单和其他软件比较基本都比较类似，其配置文件和MySQL一样为ini格式。其中主要的配置就是后端真实MySQL的地址和嵌入的Lua脚本。
>
	[mysql-proxy]
	proxy-backend-addresses=10.0.0.101:3306,10.0.0.102:3306
	proxy-lua-script=conf/proxy.lua

#### 2. MySQL-Proxy用法分析

> 用这个东西肯定不会只用其做MySQL的自动故障转移和负载均衡，它最大的特点就是可以干预MySQL连接，SQL发送，数据接收等阶段的过程，可以在这些过程的某些阶段嵌入一些Lua脚本，和Nginx的嵌入Lua非常类似。
>
> 研究这个东西的最初目的就是看其是否能够提供一种根据库名和表名进行自动路由的功能。但仔细想了一下，对于这种透明的代理由于连接的过程需要进行认证，因此在客户端连接到MySQL-Proxy进行认证时，其必须要确定连接那个真实MySQL，不能等到得到具体的SQL以后再决定。
