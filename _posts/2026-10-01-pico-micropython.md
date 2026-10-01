---
title: Rapsberry Pico中使用MicroPython
layout: post
---
> Raspberry Pico作为一种超小型化的开发板，相比普通版的树莓开发板，其更像是一个单片机——其没有操作系统，启动后自动加载默认的代码。但带来的好处也是更加简单可控，对于处理简单的硬件控制更加好用，同时体积和功耗也更低，一些精简化的Pico体积仅有邮票大小，非常适合部署在体积极为有限的设备内，并且价格仅有几十元，相比标准的树莓也有巨大的优势。
>
> Pico同样可使用C或Python进行开发，但既然选择使用Pico那肯定用Python会更加简单高效，但其不能使用完整的Python系统，使用的为MicroPython，其实现了Python的基本功能，同时还提供了一些针对Pico的功能，例如UART、I2C端口的通信API等。代码直接可运行，不需要进行编译，调试也非常方便，不过为了保护代码也可以将代码编译成二进制。

#### 1. 安装固件
> 首次使用时需要对Pico进行固件的安装，按住Pico的Boot按键将Pico通过USB连接至PC，此时会识别出一个移动磁盘设备，此时将下载的固件拖入其中即可完成固件的安装，完成后Pico会重启并开始加载固件。
> 
> 开发可以使用Thonny，其为一个针对MicroPython和Pico的IDE，可以直接连接Pico，并上载代码运行和查看运行结果。

#### 2. Pico的代码运行逻辑
> Pico在加载固件后会自动运行main.py，为此可将代码的入口放在这里，通电启动后即开始执行这里的代码，执行结束后退出，一些版本的Pico中也支持通过一个按键进行复位从头再开始执行一次main.py。
> 
> 对于一些需要一直保持运行状态的应用场景，可以将其主循环放在main.py中，其他的代码放在其余文件中，通过import引入进行调用。

#### 3. 代码示例
>
##### 通信接口
> Pico有26个GPIO针脚，其中很多针脚可以实现多种用途，并且都可以进行PWM信号输出，详细可参考官方的文档
> 
```python
#引入Pico中的基础API
from machine import Pin, PWM, UART, I2C;
#定义对应的通信针脚代号
I2C_SCL_PIN = 3;
I2C_SDA_PIN = 2;
UART_TX_PIN = 0;
UART_RX_PIN = 1;
>
#初始化对应通信接口
i2c = I2C(1, scl=Pin(I2C_SCL_PIN), sda=Pin(I2C_SDA_PIN));
uart = UART(0, baudrate=9600, tx=Pin(UART_TX_PIN), rx=Pin(UART_RX_PIN));
```
>
##### 协程实现异步操作
> 由于MicroPython中没有完成的多线程支持，为此只能通过协程实现IO部分的异步处理
>
```python
#接收串口数据
async def uart_rx_task(uart, q):
    #循环接收数据
    while True:
        #如果收到数据则放入队列中
        if uart.any():
            data = uart.read();
            q.append(data);
        else:
            pass;
        #交出执行权给主线程
        await asyncio.sleep(0.1);
#主程序
async def main():
    #启动数据接收协程
    asyncio.create_task(uart_rx_task(uart, q));
    #主循环
    while True:
        #处理队列中的消息
        while len(q) > 0:
            data = q.popleft();
        #处理其他工作...
        #交出执行权给接收协程
        await asyncio.sleep(0.05);
```
> 在main.py中启动主程序的写入如下
````python
if __name__ == '__main__':
    asyncio.run(main());
````

#### 4. 将代码编译成二进制
> 有时我们不想暴露Pico中代码的实现细节，可以使用工具将代码编译成二进制，虽然编译成的二进制也可通过反编译等手段进行代码的还原，但还原后的代码的可读性要差很多，还是可以起到一定的保护作用。一般推荐的工具为mpy-cross，可以将python代码编程成为mpy文件。编译时需要指定CPU架构，Pico对应的CPU架构为armv6m。
```bash
mpy-cross -march=armv6m core.py
```
> 会得到一个core.mpy文件，这里不使用main.py的原因是因为Pico不会自己自动运行main.mpy，所以为了能够自动运行可以将代码封装到core.py中编译为core.mpy，另外写一个main.py调用core.mpy中的运行方法执行代码。
```python
import core.main;
if __name__ == '__main__':
    core.main();
```

