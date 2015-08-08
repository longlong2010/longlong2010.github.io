---
title: Git pre-receive钩子检测语法错误
layout: post
---
> 开发中经常会因为一些低级失误提交了一些带有语法错误的代码，然后这些错误是很容易被检测出来的，例如使用php -l就可以分析代码中的语法错误，然后如果要求开发者每次都手动执行语法检测命令，会非常麻烦同时也容易遗漏，我们可以使用Git提供的钩子脚本在代码提交时进行语法错误的检测。

#### 1. pre-commit钩子
> 最为简单常用的方法是使用Git的pre-commit钩子，在开发的代码库目录的.git/hooks目录下放置一个文件名为pre-commit的可执行脚本，只要使用了可执行的标记#!以及可执行权限，使用任何语言都可以。例如使用perl实现的脚步如下。
>
```perl
#!/usr/bin/perl
>
use strict;
>
my $commit_id = `git rev-parse --verify HEAD`;
my $against = "4b825dc642cb6eb9a060e54bf8d69288fbee4904";
>
if ($commit_id) {
    $against="HEAD";
}
>
my @list = `git diff-index --cached --name-only $against --`;
my $ret = 0;
>
for my $x (@list) {
    chomp $x;
    if ($x =~ /\.php$/) {
        if (-f "$x") {
            `php -l $x`;
            if ($? != 0) {
                $ret = 1;
                goto end;
            }
        }
    }
}
>
end:
exit($ret);
```
>
> 然而使用pre-commit脚本存在一个很大的缺陷，钩子脚本需要放置在每个开发者的每个项目中，很容易遗漏，同时代码库重新clone后也需要记得重新放置钩子脚本，没有办法从根源上禁止有语法错误的代码的提交。

#### 2. pre-receive钩子
> 这时很容易想到的方法就是在中心代码库收到push时进行语法的检测，如果存在语法错误则拒绝当前的push，如此之需要维护好中心库就可以避免存在语法错误的代码被提交。Git在接受到push时会调用pre-receive钩子，如果pre-receive返回非0的返回值，则当前的push会被拒绝。在调用pre-receive脚本时，会从标准输入传递三个参数，分别为之前的版本，push的版本和push的分支。使用perl实现的脚本如下。
>
```perl
#!/usr/bin/perl
use File::Temp;
use strict;
>
#从标准输入中读取参数信息
my $line = <>;
#处理后得到旧版本，但前版本和分支
my @argv = split(/\s/, $line);
my $old = $argv[0];
my $new = $argv[1];
my $branch = $argv[2];
#比较两个版本的差异
my @out = `git diff --name-status $old $new`;
>
for my $x (@out) {
	my @param = split(/\s/, $x);
	#如果文件不是被删除且是php文件，则进行语法检测
	if ($param[0] ne 'D' and $param[1] =~ /\.php$/) {
		#取出文件的内容，将其写入到一个临时文件中
		my $content = `git show $new:$param[1]`;
		my $fh = File::Temp->new(SUFFIX => '.php', UNLINK => 1);
		print $fh $content;
		#进行语法检测
		my $result = `php -l $fh 2>&1`;
		#如果出错则输出错误，输出时需要将临时文件名替换称为原来的文件名，并拒绝push
		if ($? != 0) {
			if ($result =~ s#$fh#$param[1]#g) {
				print $result;
			}
			exit(1);
		}
	}
}
exit(0);
```
>
> 最后我们来测试一下pre-receive钩子的效果，我们故意制造一个语法错误
>
```php
<?php
>
echo1 "Hello\n";
```
> 将其提交并push
>
```bash
git add hello.php
git commit -m "make error"
git push
```
> Git会返回如下不错，并拒绝当前的push
>
```
Counting objects: 3, done.
Writing objects: 100% (3/3), 265 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
remote: PHP Parse error:  syntax error, unexpected '"Hello\n"' (T_CONSTANT_ENCAPSED_STRING) in hello.php on line 3
remote: Errors parsing hello.php
To file:///home/longlong/my-test.git/
 ! [remote rejected] master -> master (pre-receive hook declined)
error: failed to push some refs to 'file:///home/longlong/my-test.git/'
```
>
> 如此就从根本上保证了存在语法错误的代码不会被提交到中心库，可以避免存在语法错误的代码不小心被推到线上环境。不过由于进行了语法检测push操作要比之前明显慢一些。
