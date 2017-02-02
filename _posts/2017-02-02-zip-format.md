---
title: Zip文件格式解析
layout: post
---
> Zip是一种非常常见的压缩格式，其可以将多个文件打包并压缩为一个zip文件，同时Android所使用的apk包就是一个zip格式的压缩文件。这里我们尝试在不解压缩zip文件的情况下对zip包中的一部分内容进行修改。

#### 1. Zip文件格式
> Zip文件是将多个文件按照顺序进行排列，每个文件包括文件头和文件内容，其结构如下图所示
>
<img alt="" src="//upload.wikimedia.org/wikipedia/commons/thumb/6/63/ZIP-64_Internal_Layout.svg/400px-ZIP-64_Internal_Layout.svg.png" class="thumbimage" srcset="//upload.wikimedia.org/wikipedia/commons/thumb/6/63/ZIP-64_Internal_Layout.svg/600px-ZIP-64_Internal_Layout.svg.png 1.5x, //upload.wikimedia.org/wikipedia/commons/thumb/6/63/ZIP-64_Internal_Layout.svg/800px-ZIP-64_Internal_Layout.svg.png 2x" data-file-width="972" data-file-height="527" width="400" height="217">
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
> 其编码的程序如下
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
```
