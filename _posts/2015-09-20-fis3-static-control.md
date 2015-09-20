---
title: 使用FIS3管理静态文件
layout: post
---

> 在使用了CDN服务后，在更新静态文件后，一般需要通过更改引用处文件的URL的方式来实现文件的更新，如果是手动处理会非常麻烦，容易遗漏同时反复上线代码效率也比较低下。在PHP代码中一般可以通过使用统一的接口引入静态文件，通过数据库管理引用的静态文件的版本，代码上线后修改变化的静态文件的版本即可。但如果引用在静态文件中，主要是css文件中引用图片的情况，这里显然不使用PHP文件中的那种处理方法。一般来看这种情况需要借助一些前端的工具来处理这类问题。
>
> FIS3是由百度开发的一款前端工具，能够实现前端文件管理（加载和依赖处理），优化和部署等操作，主页地址为[http://fis.baidu.com/](http://fis.baidu.com/)。其目前已经成为了一个npm包，在安装了nodejs的情况下直接食用npm安装即可。

#### 1. 安装和原理

> FIS3的安装非常简单，先安装nodejs和npm，然后使用npm安装FIS3
>
```bash
pacman -S nodejs npm
npm install -g fis3
```
> FIS3对于静态文件引用的处理是其会对原始的静态文件进行一个编译的过程，重新再生成一个新的目录作为实际Web服务器输出文件的目录，并且FIS3不会修改静态文件的源代码。

#### 2. 基本使用

> 在项目的目录下增加配置文件fis-conf.js，进行如下配置，将全部的png文件替换为绝对路径，增加/static目录开头，同时增加CDN域名，配置多个会根据散列算法选择一个
>
```js
fis.match('*.png', {
  release: '/static/$0',
  domain: ['http://cdn-domain1', 'http://cdn-domain2']
});
```
>
> 在项目目录下执行以下命令编译生成正式的文件，生成到上一级目录下的dist目录中
>
```bash
fis3 release -d ../dist
```
>
> 编译后文件的变化如下
>
```diff
@@ -16,9 +16,9 @@
 }
>
 li.list-1::before {
-  background-image: url('./img/list-1.png');
+  background-image: url('http://cdn-domain1/static/img/list-1.png');
 }
>
 li.list-2::before {
-  background-image: url('./img/list-2.png');
+  background-image: url('http://cdn-domain1/static/img/list-2.png');
 }
```
>
> 文件会被替换为绝对路径，并且增加了CDN域名。

#### 3. FIS3将文件指纹放在query中

> FIS3默认的文件指纹是将文件的md5校验值的一部分作为文件名的后缀重新生成一份对应的文件，然后引用生成后的文件。但总是把图片文件保存两份，感觉不是一个非常好的处理办法，感觉还是将文件指纹放在query中会比较好，如果要实现将文件指纹放在query中需要简单修改fis3的源代码，需要修改lib/file.js文件，diff如下
>
```diff
@@ -462,7 +462,8 @@
   getUrl: function() {
     var url = this.url;
     if (this.useHash) {
-      url = addHash(url, this);
+      //url = addHash(url, this);
+      this.query = '?' + this.getHash();
     }
     // if (this.useDomain) {
       if (!this.domain) {
```
>
> 在query中增加了文件指纹，增加文件指纹需要在配置中增加useHash选项，同时如果任何图片类型都要做处理则匹配规则使用::image即可
>
```js
fis.match('::image', {
  useHash: true,
  release: '/static/$0',
  domain: ['http://cdn-domain1', 'http://cdn-domain2']
});
```
> 
> 修改后再运行效果如下
>
```diff
@@ -16,9 +16,9 @@
 }
 >
 li.list-1::before {
-  background-image: url('./img/list-1.png');
+  background-image: url('http://cdn-domain1/static/img/list-1.png?69af273');
 }
> 
 li.list-2::before {
-  background-image: url('./img/list-2.png');
+  background-image: url('http://cdn-domain1/static/img/list-2.png?543c384');
 }
```
