---
title: Zip文件格式解析
layout: post
---
> Zip是一种非常常见的压缩格式，其可以将多个文件打包并压缩为一个zip文件，同时Android所使用的apk包就是一个zip格式的压缩文件。这里我们尝试在不解压缩zip文件的情况下对zip包中的一部分内容进行修改。

#### 1. Zip文件格式
> Zip文件是将多个文件按照顺序进行排列，每个文件包括文件头和文件内容，其结构如下图所示
>
<img alt="" src="//upload.wikimedia.org/wikipedia/commons/thumb/6/63/ZIP-64_Internal_Layout.svg/400px-ZIP-64_Internal_Layout.svg.png" class="thumbimage" srcset="//upload.wikimedia.org/wikipedia/commons/thumb/6/63/ZIP-64_Internal_Layout.svg/600px-ZIP-64_Internal_Layout.svg.png 1.5x, //upload.wikimedia.org/wikipedia/commons/thumb/6/63/ZIP-64_Internal_Layout.svg/800px-ZIP-64_Internal_Layout.svg.png 2x" data-file-width="972" data-file-height="527" width="400" height="217">
>
> 其中每个文件头部分包括签名（固定值0x04034b50），解包需要的Zip版本，通用标志位，压缩方法，文件最后修改时间，文件最后修改日期，CRC-32校验值，压缩后的文件大小，压缩前的文件大小，文件名长度，扩展域长度，文件名，扩展域。其中获取一些文件头部分的代码如下所示
>
```php
<?php
$file = fopen($argv[1], 'r');
>
//signature
fseek($file, 0);
$data = fread($file, 4);
>
//Compressed size
fseek($file, 18);
$data = fread($file, 4);
$size1 = unpack('i', $data);
>
//Uncompressed size
fseek($file, 22);
$data = fread($file, 4);
$size2 = unpack('i', $data);
>
//File last modification time
fseek($file, 10);
$data = fread($file, 2);
>
//File last modification date
fseek($file, 12);
$data = fread($file, 2);
>
//File name length
fseek($file, 26);
$data = fread($file, 2);
$len1 = unpack('s', $data);
>
//File name
fseek($file, 30);
$data = fread($file, $len1[1]);
```

#### 2. MS-DOS时间日期解析
> Zip文件中文件头对应的头部分中的最后修改时间和日期采用了MS-DOS时间日期格式进行保存，其使用了4个字节的各个位段保存了年月日和时分秒这6个数值。
>
> 其中前两个字节保存时间，解析为一个16位整数，其中0～4位为秒数除以2，5～10位为分钟，11～15位为小时
>
> 后两个字节保存日期，解析为一个16位整数，其中0～4位为日，5～8位为月，9～15位年份减去1980
>
> 其解析的代码如下
>
```php
<?php
//...
//File last modification time
fseek($file, 10);
$data = fread($file, 2);
$v = unpack('s', $data);
$h = ($v[1] & (0b11111 << 11)) >> 11;
$i = ($v[1] & (0b111111 << 5)) >> 5;
$s = ($v[1] & (0b11111)) * 2;
>
//File last modification date
fseek($file, 12);
$data = fread($file, 2);
$v = unpack('S', $data);
$y = (($v[1] & (0b1111111) << 9) >> 9) + 1980;
$m = ($v[1] & (0b1111) << 5) >> 5;
$d = ($v[1] & (0b11111));
```
> 其编码的代码如下
>
```php
<?php
$h = date('G');
$i = date('i');
$s = date('s');
$v1 = pack('s', ($h << 11) + ($i << 5) + $s);
>
$y = date('Y') - 1980;
$m = date('n');
$d = date('j');
$v2 = pack('s', ($y << 9) + ($m << 5) + $d);
>
$data = $v1 . $v2;
```

#### 3. 使用实例

> 在Nginx向用户发送Zip文件时，使用Lua的body filter修改Zip文件中第一个文件的最后修改时间为当前时间，其中需要使用到Lua的[struct模块](http://www.inf.puc-rio.br/~roberto/struct/struct-0.2.tar.gz)
>
```nginx
    location ~ \.zip$ {
        #记录当前chunked号
        set $c "0";
        header_filter_by_lua_block { 
            ngx.header.content_length = nil;
        }
        body_filter_by_lua_block {
            --替换第一个文件的最后修改时间
            if ngx.var.c == "0" then
                --加载struct模块
                local struct = require("struct");
                --获取当前时间
                local t = os.date("*t");
                --生成MS-DOS格式的时间日期
                local v2 = struct.pack("I2", (t["year"] - 1980) * 2^9 + t["month"] * 2^5 + t["day"]);
                local v1 = struct.pack("I2", t["hour"] * 2^11 + t["min"] * 2^5 + math.floor(t["sec"] / 2));
                ngx.arg[1] = string.sub(ngx.arg[1], 1, 10) .. v1 .. v2 .. string.sub(ngx.arg[1], 15, -1);
            end
            ngx.var.c = ngx.var.c + 1;
        }
    }
```
>
> 如此就能够动态的修改Zip包中的一些内容而不需要每次都重新打包，能够较大程度提高Web服务器的效率。

#### 4. 补充
> 以上的配置中取消了Content-Length头，会导致不能够支持断点续传，为了能够继续支持断点续传需要对请求中的Range头进行简要的分析，并做出对应的处理，同时替换文件尾部的一段随机注释字符串，处理后的配置如下
>
```nginx
#将加载Lua模块放在Nginx初始化阶段
init_by_lua_block {
   struct = require("struct");
}
>
server {
    #....
    location ~ \.zip$ {
        set $c "0";
        body_filter_by_lua_block {
            --替换第一个文件的最后修改时间
            if ngx.var.c == "0" then
                --设置Range的默认起止位置
                local s1 = 0;
                local s2 = ngx.header.content_length;
                if ngx.var.http_range then
                    --分析Range头
                    local m = ngx.re.match(ngx.var.http_range, "bytes=(\\d+)-(\\d+)?");
                    --如果匹配成功则更新起止位置
                    if m then
                        s1 = tonumber(m[1]);
                        if m[2] then
                            s2 = tonumber(m[2]);
                        end
                    end
                end
                --需要更新的时间值在当前的范围内时，则进行修改
                if s1 < 10 and s2 > 15 then
                    local t = os.date("*t");
                    local v2 = struct.pack("I2", (t["year"] - 1980) * 2^9 + t["month"] * 2^5 + t["day"]);
                    local v1 = struct.pack("I2", t["hour"] * 2^11 + t["min"] * 2^5 + math.floor(t["sec"] / 2));
                    --调整修改的位置
                    ngx.arg[1] = string.sub(ngx.arg[1], 1, 10 - s1) .. v1 .. v2 .. string.sub(ngx.arg[1], 15 - s1, -1);
                end
            end
            --替换文件尾部的一段随机注释字符串（需要源文件尾部有一段32字节长度的注释字符）
            if ngx.arg[2] == true then
                if ngx.header.content_range then
                    local m = ngx.re.match(ngx.header.content_range, "bytes (\\d+)-(\\d+)/(\\d+)");
                    if m then
                        local m2 = tonumber(m[2]);
                        local m1 = m2 - string.len(ngx.arg[1]) + 1;
                        local m3 = tonumber(m[3]);
                        if m2 >= m3 - 32 then
                            local s = ngx.md5(math.random());
                            if m1 >= m3 - 32 then
                                ngx.arg[1] = string.sub(s, 1, m2 - m1 + 1);
                            else
                                ngx.arg[1] = string.sub(ngx.arg[1], 1, -1 - (33 + m2 - m3)) .. string.sub(s, 1, m2 - m3);
                            end
                        end
                    end
                else
                    ngx.arg[1] = string.sub(ngx.arg[1], 1, -33) .. ngx.md5(math.random());
                end
            end
            ngx.var.c = ngx.var.c + 1;
        }
    }
}
```
