---
title: 使用Nginx动态缩放图片
layout: post
---
> 在使用图片时，为了加快加载速度一般会根据页面中实际使用的尺寸使用原图的缩略图，一般情况各种规格的缩略图会在上传之后进行生成。而实际产品设计中可能会用到没有生成过的规格的缩略图，而增加一种规格的缩略图会是一个很大的维护操作，比较容易想到的方法是在实际使用时动态生成缩略图。

#### 1. Nginx的image filter模块
> Nginx原生的image filter模块使用libgd对图片进行操作，支持缩放，旋转，裁剪三种操作。例如以下配置可将图片缩放为150X100，并旋转90度
```nginx
location /img/ {
    proxy_pass   http://backend;
    image_filter resize 150 100;
    image_filter rotate 90;
}
```
>
> 配合Nginx的Lua脚本可以实现对图片的动态缩放要求，例如以下配置可提取URL中的缩放后图片宽高，进行缩放
>
```nginx
location /resize/ {
    set $w 0;
    set $h 0;
    image_filter resize $w $h;
    rewrite_by_lua_block {
        --通过正则表达式提取图片缩放后的宽和高
        local m = ngx.re.match(ngx.var.uri, '^/resize(.*[^/]+)_(\\d+)x(\\d+)\\.jpg$');
        --匹配失败则返回404
        if not m then
            ngx.exit(ngx.HTTP_NOT_FOUND);
        end
        --设置缩放后的宽和高
        ngx.var.w = m[2];
        ngx.var.h = m[3];
        --执行Rewrite
        ngx.req.set_uri(m[1] .. '.jpg', false);
    }
}
```
>
> 注意image filter模块进行的缩放操作是保持宽高比的。另外由于使用的是libgd缩放的质量和性能都比较一般，同时如果需要根据图片的原始信息做更加复杂的处理，则image filter模块会较为难以实现。

#### 2. Nginx配合OpenCV Lua模块
> OpenCV作为一个比较强大的计算机视觉处理库，实现图片的缩放自然很简单，同时相比安装Lua的OpenCV模块，自己写一个wrapper反而来得更加容易一些
```c++
#include <opencv2/opencv.hpp>
#include <vector>
#include <cstring>
#include <iostream>
//这里必须声明为extern "C"，否则在调用时会提示找不到方法
extern "C" {
    uchar* im_resize(const char* buf, int len, int h, int w, const char* ext, int& ret_len);
}
uchar* im_resize(const char* buf, int len, int h, int w, const char* ext, int& ret_len) {
    //破获抛出的异常，便于在Lua中进行调试
    try {
        //通过图片二进制数据流获得图片矩阵
        cv::Mat im = cv::imdecode(std::vector<char>(buf, buf + len), CV_LOAD_IMAGE_COLOR);
        //这里声明为UMat是便于在支持OpenCL的环境中进行加速
        cv::UMat src;
        cv::UMat dst;
        src = im.getUMat(cv::ACCESS_RW);
        //调用OpenCV的缩放方法
        cv::resize(src, dst, cv::Size(h, w));
        //重新生成压缩格式
        std::vector<uchar> ret;
        cv::imencode(ext, dst, ret);
        //返回结果
        ret_len = ret.size();
        return ret.data();
    } catch (cv::Exception& e) {
        std::cout << e.what() << std::endl;
        return NULL;
    }
}
```
> 
> 编译的命令为
```bash
g++ -O2 -shared -fPIC `pkg-config opencv` resize.cc -o libresize.so
```
>
> 在nginx中使用Lua脚本进行调用，在Luajit中可以使用ffi模块直接调用动态库中的函数，需要使用ffi.cdef对函数的格式进行声明（类似include头文件）
```nginx
location /resize/ {
    content_by_lua_block {
        --使用ffi直接调用动态库中的函数
        local ffi = require('ffi');
        --声明函数的格式，类似include头文件
        ffi.cdef[[
        char* im_resize(const char* buf, int len, int h, int w, const char* ext, int& ret_len);
        ]]
        ---加载动态库
        local resize = ffi.load('./libresize.so');
        --通过正则表达式提取图片缩放后的宽和高
        local m = ngx.re.match(ngx.var.uri, '^/resize(.*[^/]+)_(\\d+)x(\\d+).jpg$');
        --匹配失败则返回404
        if not m then
            ngx.exit(ngx.HTTP_NOT_FOUND);
        end
        --获取图像内容
        local res = ngx.location.capture(m[1] .. '.jpg');
        --如果没有返回200则直接返回对应的状态
        if res.status ~= ngx.HTTP_OK then
            ngx.exit(res.status);
        end
        --获得图像的内容和长度
        local body = res.body;
        local n = string.len(body);
        --初始化结果长度指针
        local ret_len = ffi.new('int[1]');
        --调用缩放函数
        local ret = resize.im_resize(body, n, tonumber(m[2]), tonumber(m[3]), '.jpg', ret_len);
        --返回结果
        ngx.header['Last-Modified'] = res.header['Last-Modified'];
        ngx.print(ffi.string(ret, ret_len[0]));
    }
}
```
> 如此实现的图片缩放功能经测试，处理能力在300~400rps（CPU为AMD Ryzen 2200G启用OpenCL加速），基本上还可以接受。
