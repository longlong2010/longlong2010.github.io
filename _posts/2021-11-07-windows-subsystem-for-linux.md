---
title: Windows下的Linux子系统
layout: post
---

> 适用于Linux的Windows子系统(Windows Subsystem for Linux, WSL)可让开发人员按原样运行GNU/Linux环境，包括大多数命令行工具、实用工具和应用程序。相比传统的虚拟机，WSL启动速度要快很多，相比Cygwin之类模拟器，WSL可以运行原生的Linux程序，同时WSL可以直接与Windows系统互通，可以在子系统环境中直接调用Windows程序，非常方便。

#### 1. WSL的安装

> 最新版本的Windows 10已经可以完全支持WSL，微软给出的要求是Windows 10版本2004及更高版本（内部版本 19041 及更高版本）或Windows 11。安装方法为在“设置”中找到“启动或关闭Windows功能”，开启其中“适用于Linux的Windows子系统”和“虚拟机平台”两个功能，重启系统即可。
>
> WSL有WSL1和WSL2之分，两个版本的运行方式有较大的差异：
>
> WSL1更像是Cygwin，其和Windows环境不存在边界，即在Windows环境下可以直接看到子系统下的文件，带来的问题则是IO性能极差（应该是ext4到NFS文件系统的中间层消耗了大量的性能），解压一个40MB的文件居然需要20多分钟，并且CPU满负载运行。
>
> WSL2则更像是虚拟机，其和Windows之间存在可见的边界，具有独立的网络和文件系统，文件系统也是一个虚拟硬盘文件，不过可以通过Windows的网络访问子系统内部的文件，相比使用sftp或是smb也还算是比较方便。今年好像还做到了在子系统内调用Windows下GPU的功能，但需要安装专门的GPU驱动。

#### 2. 安装WSL系统
>
#### 2.1 通过官方源安装
> 可以通过WSL命令行工具或是Microsoft store安装相应的Linux子系统，查询官方支持的Linux环境的命令如下：
```bash
wsl --list --online
以下是可安装的有效分发的列表。
请使用“wsl --install -d <分发>”安装。
>
NAME            FRIENDLY NAME
Ubuntu          Ubuntu
Debian          Debian GNU/Linux
kali-linux      Kali Linux Rolling
openSUSE-42     openSUSE Leap 42
SLES-12         SUSE Linux Enterprise Server v12
Ubuntu-16.04    Ubuntu 16.04 LTS
Ubuntu-18.04    Ubuntu 18.04 LTS
Ubuntu-20.04    Ubuntu 20.04 LTS
```
> 目前官方支持的Linux发行版主要以Ubuntu为主。使用wsl --install -d Ubuntu即可自动完成Linux子系统的安装。
> 
#### 2.2 手动安装
> wsl命令行工具支持通过tar包的形式导入Linux子系统的根文件系统，以Gentoo为例，下载Gentoo的stage3包并导入的命令如下：
```pwh
wget https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/20211101T222649Z/stage3-amd64-systemd-20211101T222649Z.tar.xz
wsl --import Gentoo Z:\WSL\Gentoo stage3-amd64-systemd-20211101T222649Z.tar.xz --version 2
```
> 导入命令的参数分别为导入后子系统的名称，安装路径，镜像文件的路径和使用的WSL版本，这里导入为WSL2。

#### 3. 配置Arch Linux环境

> 在目前各种Linux发行版中定制化程度高，且比较易用的当属Arch Linux，下一版本的SteamOS也将基于Arch，这里介绍手动配置Arch Linux子系统的方法。
>
> 首先现在Arch Linux的最小化文件系统，当前的最新版本为archlinux-bootstrap-2021.11.01-x86\_64.tar.gz，安装方式其实和Gentoo比较类似，但由于镜像文件路径的问题，先要对镜像文件进行简单的处理，由于镜像文件中存在同一个目录中具有文件名只是大小写不同的文件，为此需要在Linux环境进行处理，以root运行以下命令：
```bash
tar -xzpf archlinux-bootstrap-2021.11.01-x86_64.tar.gz
cd  root.x86_64
tar -czpf ../archlinux-bootstrap-2021.11.01-x86_64.tar.gz *
```
> 其实就是去掉镜像中root.x86\_64这层目录。并升级WSL自带的内核，目前的内核是5.10，以管理员身份运行以下命令：
```bash
wsl --update
```
> 然后直接导入处理后镜像文件即可
```
wsl --import Arch Z:\WSL\Arch archlinux-bootstrap-2021.11.01-x86_64.tar.gz --version 2
```
> 完成后使用wsl命令可以进入安装好后的系统，并做基本的配置即可使用
```bash
wsl
#初始化pacman软件源
pacman-key --init
pacman-key --populate
pacman -Sy
```
> 在建立非root用户后，以非root用户登录是可以使用wsl命令行参数实现
```bash
#以非root用户登录，并进入home目录
wsl ~ -u longlong
```

#### 4. 在Windows中访问子系统的文件

> 可以在文件浏览器中输入\\wsl$\\Arch，通过网络登录访问子系统中的文件，或者也可通过映射网络磁盘的方式进行挂载。子系统中则可以直接访问Windows的文件，全部的磁盘驱动器都会被挂在到/mnt下。

#### 5. 在Linux子系统中使用GPU

> 待续
