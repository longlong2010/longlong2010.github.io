---
title: 基于稀疏网格的窄带拓扑优化
layout: post
---
> 拓扑优化作为结构设计中优化设计的一种自动化的手段，结合增材制造技术可以实现结构的超轻量化。但由于拓扑优化的过程中需要反复进行有限元的求解，计算量极大较难实现高分辨率模型的优化。
>
> taichi语言的作者胡渊鸣在文章[Narrow-Band Topology Optimization on a Sparsely Populated Grid](https://yuanming.taichi.graphics/publication/2018-narrowband-topopt/)中给出了一种基于稀疏多重网格预条件共轭梯度法的计算方法，其可以大幅节约计算资源实现在单台PC上进行高分辨率的拓扑优化，并给出了程序的源代码，这里主要对该代码的配置和使用做简要介绍。

#### 1. 拓扑优化

> 拓扑优化（topology optimization）是一种根据给定的负载情况、约束条件和性能指标，在给定的区域内对材料分布进行优化的数学方法，是结构优化的一种。其本质上就是求一个材料的分布函数，使得在给定的边界条件下结构达到某种最优的状态，一般情况下会让结构的变形能最小（一定意义上的刚度最大化），优化的过程也是基于梯度下降的方式——沿着梯度方向更新待求函数，同时为了避免出现棋盘效应，还需要对相邻的位置进行平均化。
> 
> 由于优化的目标函数一般都是节点位移的函数，为此每一步优化都需要进行一次完成的有限元求解过程——计算单元刚度矩阵，集成总体刚度矩阵，求解方程组，对于规模较大的问题，进行一次求解就需要花费大量的时间，导致拓扑优化的网格分辨率无法提升到很高，从而无法对结构的微观结构进行进一步优化。
> 
> 随着稀疏存储技术和多重网格预条件方法的引入，使得计算更大规模的拓扑优化问题成为可能，以此优化后的结构在细观上会呈现出一定的仿生特征。

#### 2. 运行环境

> 运行环境的核心是编译遗产(legacy)版本的taichi和代码中附带的一个SPGrid求解器，其用到了Intel编译器和MKL的一些特性，为此还需要安装Intel Parallel Studio Xe。
> 
> 代码说明中中推荐的系统版本为Ubuntu18.04，经测试确实是可以的，而且20.04之后的版本都是不行的，同时其建议的Intel Parallel Studio Xe为2018版，但实际中如果使用2018版会发现其不能识别Ubuntu18.04，编译会因无法识别系统而报错，虽然可以通过一些方法解决，但如果使用2019版本的Intel Parallel Studio Xe则不会存在这个问题。
>
> 目前Intel的官网已经找不到Intel Parallel Studio Xe的下载地址，取而代之的是Intel oneAPI，经测试二者并不能兼容，不过幸好我还是找到了Intel Parallel Studio Xe的下载地址，而且该地址目前依然可用，但其为商业版需要Intel提供的序列号或是证书文件。
> [Intel Parallel Studio Xe 2019](http://registrationcenter-download.intel.com/akdlm/irc_nas/tec/15809/parallel_studio_xe_2019_update5_cluster_edition.tgz)。

#### 3. 安装taichi遗产版

> 可通过提供的Python脚本之间自动化安装，需要提前安装好python3和git，但实际运行中还会缺少很多依赖库，可在报错后安装即可。
```bash
wget https://raw.githubusercontent.com/yuanming-hu/taichi/legacy/install.py
python3 install.py
```

#### 4. 编译SPGrid Solver

> 首先将spgrid\_topo\_opt的代码放置到taichi源代码目录下的projects目录下，配置好Intel编译器的配置后，修改文件SPGrid\_SIMD\_Utilities.h，在其include部分增加
```c
#include <immintrin.h>
```
> 同时还可以去掉Makefile中的-DHASWELL，避免编译中产生大量重复定义的报错，然后就可以开始编译SPGrid Solver，如果CPU支持AVX512（目前能够完整支持的普通桌面CPU应该只有Intel的11代Core）则应该可以使用Makefile中被注释掉的一行进行编译
```bash
source /opt/intel/bin/compilervars.sh -arch intel64
cd solver
make
```

#### 5. 编译其余部分

> 其余部分将作为taichi的一部分，退回到spgrid\_topo\_opt目录下使用ti build指令进行编译，编译前需要定义三个环境变量，分别为MLK路径，GPU对应的CUDA架构版本（没有可以设为0，具体值可以在官网查表），使用double类型（这个必须设置为1否则后面编译会报错），同时如果需要使用对计算结果的可视化功能必须具有CUDA支持才可以
```bash
export TC_MKL_PATH=/opt/intel/compilers_and_libraries_2019/linux/mkl/lib/intel64_lin/
#Pascal架构为6.1，Turing架构为7.5，Ampere架构为8.6
export CUDA_ARCH=75
export TC_USE_DOUBLE=1
```
>
> 然后就可以运行scripts目录中的例子了，需要注意这里的例子都需要较大的内存才能运行，结果文件会被保存到taichi/output目录下
