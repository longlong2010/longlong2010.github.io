---
title: MySQL-Proxy 实现SQL路由
layout: post
---

> MySQL-Proxy是个进展非常缓慢的开源项目，目前还只有[0.8.5 alpha](http://dev.mysql.com/downloads/mysql-proxy/)版本，不过根据之前的使用经验还算是比较稳定，性能上还算说得过去。个人比较推荐的类似数据库的负载均衡软件最好都是和Web服务装在一起，可以有效的防止流量都通过代理而产生的流量集中和单点问题。不过如果只是简单的做MySQL的自动故障转移和负载均衡则比较推荐用Haproxy，相比MySQL-Proxy更加通用和简单，性能也要更好一些。

#### 1. MySQL-Proxy的安装

> 安装主要依赖libevent、lua 5.1、glib2，注意Lua 5.2是不行的。编译比较简单和其他软件比较基本都比较类似，其配置文件和MySQL一样为ini格式。其中主要的配置就是后端真实MySQL的地址和嵌入的Lua脚本。
>
	[mysql-proxy]
	proxy-backend-addresses=10.0.0.101:3306,10.0.0.102:3306
	proxy-lua-script=conf/proxy.lua
	admin-username=admin
	admin-password=admin
	admin-lua-script=conf/admin.lua
>
> 增加admin插件的支持后可以通过其查看后段真实MySQL的相关状态信息。
>
	 mysql -u admin -padmin  -P4041 -h 127.0.0.1
	 mysql> select * from backends;
	 +-------------+-----------------+---------+------+------------------+
	 | backend_ndx | address         | state   | type |connected_clients |
	 +-------------+-----------------+---------+------+------------------+
	 |           1 | 10.0.0.101:3306 | unknown | rw   |                0 |
	 |           2 | 10.0.0.102:3306 | unknown | rw   |                0 |
	 +-------------+-----------------+---------+------+------------------+

#### 2. MySQL-Proxy用法分析

> 用这个东西肯定不会只用其做MySQL的自动故障转移和负载均衡，它最大的特点就是可以干预MySQL连接，SQL发送，数据接收等阶段的过程，可以在这些过程的某些阶段嵌入一些Lua脚本，和Nginx的嵌入Lua非常类似。
>
> 开始觉得MySQL-Proxy是不能够提供一种根据库名和表名进行自动路由的功能，因为觉得这种透明的代理由于连接的过程需要进行认证，因此在客户端连接到MySQL-Proxy进行认证时，其必须要确定连接那个真实MySQL，不能等到得到具体的SQL以后再决定。但考虑到其能够实现透明的读写分离，必然需要在或得SQL后在对后端的真实MySQL连接进行选择，对其提供的读写分离模块进行了一定的分析，发现实现SQL路由功能是可以实现的。
>
> MySQL-Proxy可以进行控制的点如下
>
	connect_server
	read_handshake
	read_auth
	read_auth_result
	read_query
	read_query_result
>
> 其中connect\_server、read\_query和read\_query\_result用处会比较大一些，分别控制连接过程、读取SQL语句和返回查询结果。

#### 3. SQL路由的实现方法

> #### (1) 连接的建立
>
> 首先需要处理连接的过程，同时连接过程需要使用连接池功能。但这里存在一个缺陷——在connect\_server中只能与后端一个MySQL连接一个连接，在启动开始时不能保证和每个MySQL都至少保持一个连接，而如果根据之后的SQL确定要用到还没有建立过连接的MySQL时就会发生错误（只有在connect\_server阶段才能进行建立连接的动作）。临时的解决办法就是在启动MySQL-Proxy后手动进行一个连接预热的过程，彻底解决则需要改一下MySQL-Proxy的源代码让其能够在初始化时对后端的每个MySQL都建立一个连接。
>
>```lua
local min_idle_connections = 4;
local max_idle_connections = 8;
>
function connect_server()
    local least_idle_conns_ndx = 0;
    local least_idle_conns = 0;
>    
    --寻找一个空闲连接数最小的MySQL进行连接 
    for i = 1, #proxy.global.backends do
        local s = proxy.global.backends[i];
        local pool = s.pool;
        local cur_idle = pool.users[""].cur_idle_connections;
>
        --如果完全没有空闲连接，则直接连接 
        if cur_idle == 0 then
            proxy.connection.backend_ndx = i;
            return
        end
>        
        --否则寻找空闲连接数最小的进行连接 
        if least_idle_conns_ndx == 0 or
           ( cur_idle < min_idle_connections and
             cur_idle < least_idle_conns ) then
            least_idle_conns_ndx = i;
            if s.idling_connections then
                least_idle_conns = s.idling_connections;
            else
                least_idle_conns = 0;
            end
        end
    end
>
    --如果找到可以空闲连接数未满的MySQL则进行连接
    if least_idle_conns_ndx > 0 then
        proxy.connection.backend_ndx = least_idle_conns_ndx;
    end
    if proxy.connection.backend_ndx > 0 then
        local s = proxy.global.backends[proxy.connection.backend_ndx];
        local pool = s.pool;
        local cur_idle = pool.users[""].cur_idle_connections;
        if cur_idle >= min_idle_connections then
            return proxy.PROXY_IGNORE_RESULT;
        end
    end
end
```
>
>
> #### (2) 进行SQL路由
>
> 之后需要做的就是根据SQL语句对后端的MySQL连接进行选择，这里需要注意就是在选择之后可能需要进行数据库的切换操作，即如果操作的数据库和当前不一致需要自动在SQL语句之前增加一个切换数据库的操作，已保证对客户端的透明。
>
>```lua
function read_query(packet)
    local cmd = commands.parse(packet)
        --忽略推出命令
        if packet:byte() == proxy.COM_QUIT then
            proxy.response = {
                type = proxy.MYSQLD_PACKET_ERR,
                errmsg = "ignored the COM_QUIT"
            };
            return proxy.PROXY_SEND_RESULT;
        end
>
        --这里默认选择连接1
        proxy.connection.backend_ndx = 1;
        if packet:byte() == proxy.COM_QUERY then
            --解析表名
            local tokens = tokenizer.tokenize(cmd.query);
            local table = "";
            for i = 1, #tokens do
                if tokens[i].token_name == "TK_LITERAL" then
                    table = tokens[i].text;
                    break;
                end
            end
>
            --如果表名是t2则选择连接2
            if table == "t2" then
                proxy.connection.backend_ndx = 2;
            end
        end
>
    local s = proxy.connection.server;
    local c = proxy.connection.client;
    --如果数据库不一致则自动进行切换
    --这里append方法的第一个参数为query的自定义id
    --在read_query_result方法中会用到
    if cmd.type ~= proxy.COM_INIT_DB and
        c.default_db and c.default_db ~= s.default_db then
        proxy.queries:append(2, string.char(proxy.COM_INIT_DB) .. c.default_db, { resultset_is_needed = true });
    end
    proxy.queries:append(1, packet, { resultset_is_needed = true });
>
    return proxy.PROXY_SEND_QUERY;
end
```
>
> #### (3) 处理返回数据
>
> 这里主要是对之前所说的自动切换数据库的操作的返回结果进行屏蔽，同样是为了保证对客户端的透明。
>
>```lua
function read_query_result(inj)
    --如果是自动切换数据库的返回结果则进行忽略，在read_query中将切换数据库的query的自定义id设置为2
    if inj.id ~= 1 then
       return proxy.PROXY_IGNORE_RESULT;
    end
>
    proxy.connection.backend_ndx = 0;
end
```
>
>
> #### (4) 实际测试
> 实际测试环境使用第一节中表格所示的两个MySQL实例，在10.0.0.101的test库中建表t1，在10.0.0.102的test库建表t2，启动MySQL-Proxy，并进行两次初始连接，查看状态显示如下。
>
	+-------------+-----------------+-------+------+-------------------+
	| backend_ndx | address         | state | type | connected_clients |
	+-------------+-----------------+-------+------+-------------------+
	|           1 | 10.0.0.101:3306 | up    | rw   |                 0 |
	|           2 | 10.0.0.102:3306 | up    | rw   |                 0 |
	+-------------+-----------------+-------+------+-------------------+
>
> 对比第一节的表格，通过MySQL-Proxy两次初始连接，后端MySQL的state都从unknown变为了up，此时MySQL-Proxy与两个后端MySQL都已经有了一个连接。这时连接MySQL-Proxy，进入test库，并查看此时test库中的表，会发现只有一个t1，因为默认会连接10.0.0.101。
>
	mysql -u root -P4040 -h 127.0.0.1
	mysql> use test;
	Reading table information for completion of table and column names
	You can turn off this feature to get a quicker startup with -A
>
	Database changed
	mysql> show tables;
	+----------------+
	| Tables_in_test |
	+----------------+
	| t1             |
	+----------------+
	1 row in set (0.00 sec)
>
> 但在这里查询表t2，可以看到可以正常返回结果，同样直接查询表t1也可以正常返回结果，实现了自动SQL路由的效果。
>
	mysql> show tables;
	+----------------+
	| Tables_in_test |
	+----------------+
	| t1             |
	+----------------+
	1 row in set (0.00 sec)
>
	mysql> SELECT * FROM t2;
	+----+
	| id |
	+----+
	|  1 |
	|  2 |
	|  3 |
	+----+
	3 rows in set (0.00 sec)
>
	mysql> SELECT * FROM t1;
	+----+
	| id |
	+----+
	|  1 |
	+----+
	1 row in set (0.00 sec)
>

#### 4. 总结
> 使用MySQL-Proxy可以基本上实现SQL路由，从而实现对于应用完全透明的分库操作，但如果分库的逻辑比较复杂会导致路由的Lua脚本变得难以维护，同时每新增加一个表都有可能需要修改路由的Lua脚本也会给维护带来较大的难度。并且MySQL-Proxy不会对Lua脚本进行预编译（不知道是出于什么考虑，但好的是修改后也不用对其进行重启），而是接受到每个连接后都会重新编译一次，性能也会是个可能的问题。目前能够实现SQL路由功能的其他软件大多是由一些大公司内部开发的，个别的进行了开源（如360的Atlas和阿里的Cobar），但这些东西主要是适应其自身的业务，通用型会差一些，同时使用量也相对较少，出现问题较难快速解决。
