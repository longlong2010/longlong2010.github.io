---
title: 使用Nginx进行页面静态化
layout: post
---
> 对于动态的页面内容，我们可以保存一份静态页面用于在不能提供动态服务时，例如应用服务出现故障或事进行维护等情况时使用保存的静态页面提供一个降级的服务，以降低故障或维护时产生的影响，使网站一只可以处于能够访问的状态。这一过程会涉及两个事情，首先是如果保存一份静态化的网站，另外就是如何利用静态化的网站提供服务以及进行自动的切换。

#### 1. 生成静态化的网站的方法
> 对于生成网站静态化镜像，最为简单的方法就是使用wget进行递归的页面抓取，使用方式大致如下
>
```bash
wget -r http://www.sina.com.cn
```
> wget会分析页面中的地址，并生成一个www.sina.com.cn的目录，里面会按照页面地址的url生成目录结构和对应的文件，理论上只要存在引用关系的页面都可以被收录。操作非常简便，但缺点就是不能够做增量的更新，在网站内容较多时完整生成一次需要较长的时间。

> 另外的方法就是使用CDN保存网站的镜像，总体操作也是比较简单的，但涉及的一些细节处理会稍微麻烦一些，如我们只能够将匿名用户访问的结果作为静态化的结果，在CDN上作这些处理会比较难实施一些。这里尝试使用Nginx的proxy\_cache功能将从后端获得的页面在本地保存一份。

#### 2. 对访问的分流处理
> 由于只能将匿名用户访问的结果作为静态化的结果，则需要对用户身份进行分流，只对匿名用户的请求作静态化存储。一般的网站都会用Cookie来标示用户的身份，将含有合法Cookie的用户认定为某个特定的用户，因此简单的判断就可以是将没有身份认证Cookie的用户认为是匿名用户，这样虽然不能做到准确分流，但总是不会有错误的。对应的Nginx配置如下
>
```nginx
location / {
	if ($cookie_auth != "") {
		rewrite ^ /direct last;
	}
	#缓存文件的配置
}
>
location /direct {
	internal;
	rewrite ^ $request_uri break;
	proxy_pass http://backend;
}
```
>
> 将可能是登录用户的请求重定向到一个直接访问，不进行静态化的location中。

#### 3. 存储静态化页面
> 存储静态化页面主要是用Nginx proxy模块中的proxy\_cache指令，这里主要是需要处理当访问的是一个目录时需要将文件保存成对应的目录下的index.html文件，并且由于有些页面中包含请求参数，不同的请求参数可能会对应两个完全不同的页面，因此需要将请求参数也作为文件名的一部分进行保存。另外由于动态内容没有Last-Modified头，会导致每次都会覆盖之前的文件，如果不需要这样的行为可以进行一下文件存在的判断，如果存在则和登录用户一样处理，其大致的配置如下
>
```nginx
set $store_uri $host$request_uri;
if ($request_uri ~ "/$") {
	set $store_uri $host$request_uri/index.html;
}
if (-f $document_root/$store_uri) {
	rewrite ^ /direct last;
}
proxy_set_header Accept-Encoding ""; 
proxy_pass http://backend;
proxy_store $document_root/$store_uri;
```
>
> 由于同一个server可能会对应多个域名，这里将域名也作为存储目录的一部分进行保存，同时由于存在压缩的问题，这里进行一个proxy的gzip hack，即强制获得一个不压缩的页面，然后再根据用户的请求需求进行压缩处理。

#### 4. 静态化文件的使用和自动切换
> 使用静态化文件时需要注意，如果直接将缓存的目录作为对应的根目录，则当有请求参数时，Nginx会忽略请求参数查找对应的文件，会导致保存的带有参数的文件不起作用，这里需要使用try\_files指令制定查找带有参数的文件，其配置如下
>
```nginx
error_page 500 502 504 = @error;
>
location @error {
	try_files $host$request_uri $host$request_uri/index.html = 404;
} 
```
>
> 同时将错误页面指向这里就可以在出错时自动使用静态化的网站进行临时的服务，待到服务恢复时切换会正常的服务。

#### 5. 最终完整的配置
> 最终完整的配置如下
>
```nginx
server {
    listen      8088;
    server_name  localhost;
>
    charset utf-8;
>
    root data;
>
    location / { 
        if ($cookie_auth != "") {
            rewrite ^ /direct last;
        }   
>
        set $store_uri $host$request_uri;
        if ($request_uri ~ "/$") {
            set $store_uri $host$request_uri/index.html;
        }   
        if (-f $document_root/$store_uri) {
            rewrite ^ /direct last;
        }   
>
        proxy_set_header Accept-Encoding ""; 
        proxy_pass http://backend;
        proxy_store $document_root/$store_uri;
    }   
>
    location /direct {
        rewrite ^ $request_uri break;
        proxy_pass http://backend;
    }   
>
    error_page 500 502 504 = @error;
>
    location @error {
        try_files $host$request_uri $host$request_uri/index.html = 404;
    }   
}
```
