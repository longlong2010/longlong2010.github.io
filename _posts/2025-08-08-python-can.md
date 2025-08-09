---
title: 使用Python进行CAN总线通信
layout: post
---

> 控制器局域网总线（CAN，Controller Area Network）是一种用于实时应用的串行通讯协议总线，它可以使用双绞线来传输信号，是世界上应用最广泛的现场总线之一。CAN协议用于汽车中各种不同元件之间的通信，以此取代昂贵而笨重的配电线束。

#### 1. CAN总线的物理结构

> CAN通讯是异步通讯，只有CAN\_H和CAN\_L两条信号线，共同构成一组差分信号线，以差分信号的形式进行通讯。使用闭环总线网络，速度快，距离短，它的总线最大长度为40m，通讯速度最高1Mbps，总线两端各有一个120欧的电阻。
>
> CAN总线上可以挂载多个通讯节点，节点之间的信号进过总线传输，实现节点间通讯。由于 CAN 通讯协议不对节点进行地址编码，而是对数据内容进行编码的，所以网络中的节点个数理论上不受限制,只要总线的负载足够即可，可以通过中继器增强负载。

#### 2. CAN通信协议

> CAN通信协议分为标准格式和扩展格式，在使用层上两者主要的区别在于ID的长度不同，标准帧为11位，扩展帧为29位，传输数据的部分都是0~8个字节的数据段（数据前用4位的标识数据的长度）。

#### 3. CAN接口的配置

> CAN接口在Linux系统中和以太网接口一样作为一个通信接口，可以通过ifconfig和ip命令进行配置和启动。以树莓派中常见的MCP2515为例，除了在内核中配置引导对应的模块外。可通过下列命令对CAN接口进行启用
>
```bash
ip link set can0 up type can bitrate 1000000
ifconfig can0 txqueuelen 65536
ifconfig can0 up
```
> 如果需要在开机时启动CAN接口，可以将相关配置放在rc.local或udev中。
```
KERNEL=="can*"
SUBSYSTEM=="net"
ACTION="add"
PROGRAM="/sbin/ifconfig $name txqueuelen 1024"
RUN+="/usr/bin/ip link set up $name type can bitrate 500000"
```
> 并执行
>
```
udevad control --reload-rules && udevadm trigger
```
> 即可实现CAN接口的自动启动。

#### 4. CAN接口的相关工具

> 可使用命令行工具cansend和candump进行进行CAN通信的快速测试，其格式为
>
```bash
#接收并输出can0接口的数据
candump can0
#接受并输出任何can接口的数据
candump any
#从can0端口发送标识为123，0x11223344的数据
cansend can0 123#11223344
```
> 并且在调试时可以将数据同时发送给自己一份，以便进行调试，可在启动can的命令结尾加上loopback=on

#### 5. 使用Python进行CAN通信

> 在python中使用CAN通信，可安装python的can扩展，可使用pip或系统软件包管理系统进行安装python-can，其使用的方式如所示
>
```python
#使用can0接口，波特率500kbps，loopback模式为关闭
can0 = can.interface.Bus(channel = 'can0', \
    interface='socketcan', bitrate=500000, local_loopback=False);
#创建id为0x00000625的CAN帧，数据为data，格式为标准帧
msg = can.Message(arbitration_id=0x00000625, \
        data=bytearray([0x40, 0x05, 0x60, 0x00, 0x00, 0x00, 0x00, 0x00]), is_extended_id=False);
#发送数据
can0.send(msg);
#接收数据，等待时间1秒
ret = can0.recv(1.0);
```
