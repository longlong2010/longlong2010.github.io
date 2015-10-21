---
title: Nginx配置与使用
layout: post
---

> 经过多年的积累和完善，Nginx已经是和Apache一样成为了一种最为常见的Web服务器，其因其使用基于事件的io模型能够承受大量的连接数，同时只占用很小的内存，灵活的配置文件以及丰富的第三方模块得到了越来越多的应用。

> 同时目前Nginx + PHP-FPM已经基本上取代了Apache + Mod-PHP成为了PHP运行环境的主流。但存在一个比较常见的误区，就是Nginx + PHP-FPM的性能要远远高于Apache + Mod-PHP，但事实并不是这样，本质上影响性能的是PHP的执行，而且PHP-FPM的运行模型和Apache基本上是类似的——都是prefork的方式，而这才是性能的瓶颈。个人感觉Nginx + PHP-FPM性能好于Apache主要是由于默认安装的Apache加载了大量无用的模块，同时如果没有做动态静态分离，Nginx在处理静态内容时会有很大的优势。

#### 1. Rewrite配置

> 作为Web服务器，最为常用的配置就是URL的重写，即Rewrite配置，Nginx的Rewrite相对Apache感觉更加简单，同时使用的是PCRE正则书写起来也更加容易一些。这里主要说几个常见的问题
##### 1) MVC入口Rewrite

> 一般使用过Apache Rewrite，都习惯写成
>
```nginx
location / {
	if (!-e $request_filename) {
		rewrite (.*) /index.php?q=$1 last;
	}
}
```

> 而如果使用了新版本的Nginx则建议使用try\_files语法
>
```nginx
location / {
	try_files $uri $uri/ /index.php?q=$uri&$args;
}
```
##### 2) 整个域名跳转

> 经常会遇到将整个域名跳转到另外一个域名并且uri不变的情况，一般较为常见的写法是
>
```nginx
location / {
	rewrite ^/(.*)$ http://xxx.com/$1 permanent;
}
```
>
> 而较好的写法是不使用正则表达式
>
```nginx
location / {
	return 301 $scheme://xxx.com$request_uri;
}
```
##### 3) break和last的区别

> break是当匹配了当前的Rewrite规则后不再执行后面紧跟的Rewrite规则，但会进行执行当前location中的其它配置，而last则是当匹配了当前的Rewrite规则后不再执行后面紧跟的Rewrite规则同时跳出当前的location，根据重写后的uri再进行一次新的location匹配。

> 例如
>
```nginx
location /app/ {
	rewrite ^/app/new/(.*)$ /newapp/$1 break;
	proxy_pass http://backend;
}
```
> 当请求匹配了当前的Rewrite规则后，会继续执行后面的proxy\_pass指令，而代理请求的request\_uri会使用Rewrite以后的值。而如果像如下改了last
>
```nginx
location /app/ {
	rewrite ^/app/new/(.*)$ /newapp/$1 last;
	proxy_pass http://backend;
}
```
>
> 则如果匹配了当前的Rewrite规则后，会跳出当前的location，寻找能匹配/newapp/的location，如果没有则会进入location /，特别注意的是如果rewrite后还到当前的location，则使用last特别容易导致死循环，一定要特别注意，例如以下会造成Rewrite的死循环而导致Nginx出现500错误
>
```nginx
location /app/ {
	rewrite ^/app/(.*)$ /app/new/$1 last;
	proxy_pass http://backend;
}
```
> 会在uri中无限的增加/new/
##### 4) 嵌套if

> Nginx的配置是原生不能支持if的嵌套的，但可以通过变通的方法实现类似的效果——将一个变量进行多次赋值，通过最后的结果再进行判断
>
```nginx
set $_v "";
if ($request_method = "POST") {
	set $_v "1";
}
if ($cookie_L = "") {
	set $_v "2$_v";
}
if ($v = "21") {
	return 403;
}
```

#### 2. 作为后段服务的前端

> Nginx还有一个很大的用处就是将一些后段服务通过相应的模块之间暴露给前段使用，即将一种其它的协议HTTP化，最为常见的是原声的Memcached模块，例如以下的配置可以使得/message/?mkey=xxxx直接读取出对应的内容，如果mkey是服务端传给客户端的一个加密的字符串，则也可以保证数据的安全性
>
```nginx
location /message/ {
	set $memcached_key $arg_mkey;
	memcached_pass memcache;
	error_page 404 = @cache_miss;
}
>
location @cache_miss {
	internal;
	proxy_pass http://backend;
}
>
upstream memcache {
	hash $memcached_key consistent;
	server 127.0.0.1:11211;
	server 127.0.0.1:11212;
	server 127.0.0.1:11213;
}
```
> 类似的模块还有很多如支持MySQL的Drizzle模块，支持MongoDB的Mongo模块，支持Redis的Redis2和HTTP Redis模块，都可以将后段服务HTTP化，从而省去PHP的操作过程从能得到极高的性能。

#### 3. 使用Lua进行扩展

> Nginx的Lua模块是得Nginx能够通过配置就实现各种定制化的功能，例如需要根据多个条件进行判断的Rewrite，对一些请求参数的分析或过滤，其能够通过简单的Lua代码就能够实现一个C模块才能实现的功能，同时也有万全可以接受的性能。这里需要注意的有两点，首先rewrite\_by\_lua的优先级要低于Nginx原生的Rewrite，因此如下的配置不能得到想要的结果
>
```nginx
location / {
    set $a 12; # create and initialize $a
    set $b ''; # create and initialize $b
    rewrite_by_lua 'ngx.var.b = tonumber(ngx.var.a) + 1';
    if ($b = '13') {
       rewrite ^ /bar redirect;
       break;
    }
}
```
> 由于最后的原生Rewrite会优先执行，尽管其写在最后面，因此建议书写配置的使用永远将rewrite\_by\_lua写在最后，避免对配置文件的错误理解，同时如果一定要实现以上的效果，则可以将最后的原生Rewrite改用Lua书写，如下
>
```nginx
location / {
    set $a 12; # create and initialize $a
    set $b ''; # create and initialize $b
    rewrite_by_lua '
		ngx.var.b = tonumber(ngx.var.a) + 1;
		if ngx.var.b == 13 then
			ngx.redirect("/bar");
		end
	';
}
```
> 此外set\_by\_lua的代码中不要运行过于复杂的操作，因为其会阻塞Nginx的事件循环，可能会严重影响Nginx的性能。
