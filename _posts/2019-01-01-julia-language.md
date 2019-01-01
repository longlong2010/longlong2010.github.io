---
title: Julia语言简介
layout: post
---

> 使用Python+numpy做数值计算基本可以达到类似MATLAB的效果，能够快速的进行开发和部署，但由于Python语言本身的限制性能上并不算十分理想。
>
> 目前的动态语言向JIT运行方式转型已经是大趋势，而Python的JIT版本pypy虽然性能大幅提升，但对numpy和scipy的支持还存在很多问题。在寻找了很多解决方案后发现了Julia语言，原生支持多维数组及相关的运算，采用JIT的运行方式，主要面型科学计算领域，官方给出的性能测试结果表明其与LuaJIT的性能相当。

#### 1. 基本语法

> Julia的基本语法和Python非常类似，同时引入了很多数学中习惯使用的表达方式，如范围的判断可以写成1 < x < 10，函数的定义可以写成f(x) = x^2 
>
```julia
#变量赋值
x = 3;
>
#条件
if 1 < x < 10
    @show x
else
    println(1);
end
>
#循环
for i = 1 : 100
    @show i;
end
>
#函数
#标准定义
function add(a::Int, b::Int)
    return a + b;
end
#简化的定义
f(x::Real) = x^2;
```

#### 2. 多维数组

> 在数值计算中最为重要的数据类型，一般常用的是二维数组——矩阵，常用的矩阵运算如下
>
```julia
#生成一个4x4的随机矩阵，4x1的随机向量
M = rand(4, 4);
v = rand(4);
#计算矩阵和向量的乘积
v1 = M * v;
#矩阵的转置和逆
M1 = M';
M2 = M^-1;
```
> 通过JuliaGPU提供的ClArray和CuArray包可以将数组的运算扩展到GPU设备上，而接口保持和一般的Array一致

#### 3. 自定义类型

> Julia中并不支持Python中的面向对象的方式定义类型，而是定义类型和类型的静态方法，并可以重载定义的方法，在定义的类型前增加mutable可使得内部的数据是可修改的
>
```julia
#定义一种抽象类型
abstract type A end;
#定义抽象类型的子类型
mutable struct B <: A
    x::Int64;
    #构造函数
    function B(x::Int64)
        return new(x);
    end
end
#定义方法
function add(v1::A, v2::A)
    return v1.x + v2.x;
end
```

#### 4. 包管理

> Julia和Python的pip类似也有一套自身的包管理系统，在julia控制台中可以进行安装
>
```julia
import Pkg;
#增加和删除包MatrixMarket
Pkg.add("MatrixMarket");
Pkg.rm("MatrixMarket");
```

#### 5. 性能测试

> 使用Julia和Python对比性能对比测试——生成一个随机的10x10的矩阵，并求逆，循环1000000次，代码如下
```julia
for i = 0 : 1000000 - 1
    A = rand(10, 10);
    A^-1;
end
```
```python
import numpy;
for i in range(0, 1000000):
    A = numpy.random.rand(10, 10);
    numpy.linalg.inv(A);
```
> 运行结果如下，循环的次数越多Julia的优势就越明显，而当循环数少时Python反而会快一些，原因应该是循环数少时JIT编译时产生的消耗会变得特别明显
```bash
time python test.py
real    0m15.129s
user    0m14.958s
sys     0m0.027s
>
time julia test.jl
real    0m7.872s
user    0m6.648s
sys     0m1.139s
```
