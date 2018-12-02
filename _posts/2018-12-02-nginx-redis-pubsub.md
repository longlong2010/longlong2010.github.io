---
title: 使用Nginx和Redis实现消息推送
layout: post
---

> 实现消息的实时推送，基本的思路就是客户端和服务器之间保持一个长连接，当有消息需要推送给客户端时，服务器发送消息的内容给客户端。而很多人会认为保持太多的长连接会严重影响系统的性能，但如今的网络模型都是基于事件驱动的非阻塞模式，对于总连接数N拥有O(1)的性能，所以并没有必要在这方面有所担心，空闲的连接只是会占用一定数量的内存而已。

#### 1. 消息推送原理

> 客户端和服务端的Nginx保持一个长连接，同时Nginx和后端的Redis通过SUBSCRIBE指令保持一个长连接，CHANNEL的名称为经过加密的字符串，在建立长连接之前通过认证逻辑下发并作为建立长连接的参数（可以为多个），在需要对客户端进行推送时，只需要向对应的CHANNEL中通过PUBLISH指令发送相应的内容即可实现1对1或1对多的内容推送。

#### 2. Nginx和Reids的配置

> 要实现Nginx与Redis的通信，可以使用Nginx的Lua模块来完成，其中需要用到resty-lua作为与Redis进行通信的API。
>
```nginx
location /sub {
    --检测客户端是否断开
    lua_check_client_abort on;
    content_by_lua_block {
        local cjson = require "cjson";
        local redis = require "resty.redis";
        --建立连接
        local r = redis:new();
        r:connect("127.0.0.1", 6379);
        --启动消息订阅
        local res, err = r:subscribe(ngx.unescape_uri(ngx.var.arg_key));
        --如果客户端意外关闭，则断开与Redis的连接并退出运行
        ngx.on_abort(function()
            r:close();
            ngx.exit(499);
        end);
        --循环接收消息
        while not ngx.worker.exiting()
        do
            repeat
                local res, err = r:read_reply();
                if err then
                    break;
                end
                --接收完成后将消息发送给客户端
                local ok, err = ngx.say(cjson.encode(res));
                ngx.flush();
            until true
        end
    }
}
```
> 由于需要处理较高的连接数，所以在Nginx和Redis中需要将支持的最大连接数调到合适的值，由于Nginx对于每一个长连接都需要建立一个SUBSCRIBE的长连接到Redis，因此受到系统端口号的限制，一个IP地址只能支持65535个端口，因此最大连接数设置在60000左右即可。
```bash
#nginx
worker_processes 65535;
#redis
maxclients 60000
```

#### 3. 测试效果

> 与Nginx建立长连接后，向Redis中依次发布两条消息
```
curl http://localhost:8080/sub?key=abc -v
< HTTP/1.1 200 OK
< Server: openresty/1.13.6.2
< Date: Sun, 02 Dec 2018 03:48:39 GMT
< Content-Type: application/octet-stream
< Transfer-Encoding: chunked
< Connection: keep-alive
<
["message","abc","123"]
["message","abc","666"]
```
> 长连接数测试，建立15000个长连接到Nginx
```javascript
const http = require('http');
options = {
	hostname: '127.0.0.1',
	port: 8080,
	path: '/sub?key=abc'
};
list = [];
for (var i = 0; i < 15000; i++) {
	req = http.request(options);
	req.end();
	list.push(req);
}
```
> 15000个长连接建立后，Nginx主进程占用的内存为280M左右
```bash
# Clients
connected_clients:15001
>
  VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
312672 284664    784 S   0.0  28.2   0:59.09 nginx
```

#### 4. 总结

> 使用Nginx和Redis实现了简单的消息推送系统，但存在的缺点是一个IP只能处理60000个连接，不过可以通过在一台服务器上启动多个Nginx实例并绑定不同的IP来充分利用系统的资源。
