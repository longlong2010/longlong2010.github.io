---
title: 代码的可移植性
layout: post
---

> 在最开始学习编程的时候，一般的书中都会提到代码的可移植性的问题，其中主要包括硬件平台的可移植性，操作系统的可移植性，运行环境的可移植性等。然而目前很多的代码都是在PC + Windows平台下开发和测试的，开发人员一般也不会考虑太多可移植性的问题。而在很多领域对于平台小型化的要求，不能够使用x86 + Windows的架构，而我们肯定不想为此独立维护两套代码，一般来说充分考虑代码的可移植性就能够解决这一问题。
>
> 以下简单说明一下代码在Windows系统和Linux系统下实现可移植的一些问题。


#### 1. 文件名大小写问题

> 不同的操作系统对于文件名的处理方式不同，而比较大的一个差异在于是否区分文件名的英文大小写，Windows不区分，而Linux和BSD则区分，MacOS需要取决于使用的文件系统。虽然Windows系统下不会区分文件名的大小写，但当文件被复制到其他区分文件名大小写的操作系统时，则会以当前文件命名时的状态被复制。其中对于代码中的影响则主要在include代码文件时，因此在使用include语句时只要保证包含的文件名和文件命名时使用的字母大小写一致就可以做到不同系统的兼容。
>
```c
/*
目录下的文件如下
Test.h  main.c
这里需要和看到的文件名完全一致
*/
#include "Test.h"
>
int main() {
    return 0;
}
```

#### 2. 文件目录结构

> Linux等操作系统使用的是单根节点的文件树，目录之间的分割为/符号，而Windows则使用了磁盘分区的概念，同时目录之间的分割符号为\\符号。一般可以通过使用相对路径来避免处理根的问题，然后通过预编译条件定义使用的目录分割符号。
>
```c
#include <string>
#include <iostream>
#ifdef _WIN32
#define DIRECTORY_SEPARATOR '\\'
#else
#define DIRECTORY_SEPARATOR '/'
#endif
int main() {
    std::string path = "usr";
    path += DIRECTORY_SEPARATOR;
    path += "bin";
    std::cout << path << std::endl;
    return 0;
}
```
> 以上代码在不同的操作系统下会有不同的输出
>
```
Windows下输出为
usr\bin
>
Linux下输出为
usr/bin
```
> 这里也给出了一种判断当前操作系统是否为Windows的方法，即利用预定义变量_WIN32

#### 3. 动态库的导出和导入

> Windows的动态库导出和使用时相对Linux等操作系统要复杂一些，导出动态库时要在函数前声明_declspec(dllexport)，导入声明时需要在函数前声明_declspec(dllimport)。而对于这一点，通常也可以使用预编译语句进行处理——定义两个宏，在Linux系统下其值为空，Windows系统下为对应的两个声明语句，即
>
```c
#ifdef _WIN32
#define DLLExport _declspec(dllexport)
#define DLLImport _declspec(dllimport)
#else
#define DLLExport
#define DLLImport
#endif
```
> 然后在函数声明时，使用定义的两个宏DLLExport和DLLImport，以达到兼容不同系统的作用。

#### 4. 调用Fortran代码

> 在Windows中调用Fortran代码需要在声明时增加__stdcall声明，Linux中则不用，为此可以使用上一节中的方法进行处理，另外Linux下引入Fortran代码时，函数名最后要增加一个下划线，可以先声明带有下划线的函数名，然后在通过定义一个宏来实现和原来一致的函数名
>
> 例如以下的Fortran代码
>
```fortran
real function F(X)
    implicit none
    real :: X
    F = X * X
end function
```
> 导入到C语言中
```c
#include <stdio.h>
#ifdef _WIN32
float __stdcall f(float*);
#else
float f_(float*);
#define f f_
#endif
int main() {
    float x = 12.0;
    float y = f_(&x);
    printf("%f\n", y);
    return 0;
}
```

