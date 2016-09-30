---
title: Python中使用同步队列同步测量时间
layout: post
---

> 之前的文章里说过用Python通过Raspberry PI读取超声波传感器的问题，在发送测量指令后需要等待一小段时间来让超声波传感器检测回波，当需要同时得到多个超声波传感器的测量数据时，如果依次进行获取，由于存在等待回波的时间间隔，则不同的超声波传感器的数据会来自与不同时刻，而导致之后的分析结果存在一定的误差，因此需要对测量的时间进行同步。

#### 1. Python的同步队列

> Python语言中提供了队列数据结构Queue，其不但能够实现队列数据结构的功能，还可以为线程间的同步进行支持。其简单的使用方法如下
>
```python
#判断Python版本
if sys.version_info < (3, 0):
    from Queue import Queue;
else:
#Python3 中模块的名字改为了小写
    from queue import Queue;
if __name__ == '__main__':
    q = Queue();
    q.put("Hello World");
    item = q.get();
```
>
> 在线程同步中的使用时不需要考虑进行线程锁定操作，其使用方法如下
>
```python
from threading import Thread;
#工作线程类
class Worker(Thread):
    def __init__(self, q):
        #使用super方法调用父类的方法
        super(Worker, self).__init__();
        self.q = q;
    def run(self):
        while True:
            item = self.q1.get();
            #处理逻辑...
            q1.task_done();
if __name__ == '__main__':
    q = Queue();
    #启动3个线程
    for i in range(3):
        t = Worker(q);
        t.daemon = True;
        t.start();
    #需要处理的数据入队列
    q.put(123);
    #等待队列情况
    q.join();
```

#### 2. 超声波传感器的测量

> 使用I<SUP>2</SUP>C接口的KS103超声波传感器，其测量的代码如下
>
```python
import smbus;
import time;
i2c = smbus.SMBus(1);
#传递I2C接口和传感器地址
def read_data(i2c, addr):
    #发送测量指令
    i2c.write_byte_data(addr, 2, 0xb4);
    #等待响应
    time.sleep(0.1);
    #读取测量数据
    d = i2c.read_byte_data(addr, 2);
    d <<= 8;
    d += i2c.read_byte_data(addr, 3);
    return d;
```

#### 3. 测量过程的时间同步

> 在获取测量数据时，开启大于同时测量个数个线程，然后将测量使用的超声波传感器的地址写入队列，在处理线程中获得到对应的超声波传感器的地址，并调用测量指令获取测量结果，然后将结果在写入一个返回给主线程的同步队列，其主要代码逻辑如下
>
```python
from threading import Thread;
import sys;
if sys.version_info < (3, 0):
    from Queue import Queue;
else:
    from queue import Queue;
from time import sleep;
import time;
import smbus;
>
class Worker(Thread):
    def __init__(self, q1, q2, i2c):
        super(Worker, self).__init__();
        self.q1 = q1;
        self.q2 = q2;
        self.i2c = i2c;
    def read_data(self, addr):
        self.i2c.write_byte_data(addr, 2, 0xb4);
        time.sleep(0.1);
        d = self.i2c.read_byte_data(addr, 2);
        d <<= 8;
        d += self.i2c.read_byte_data(addr, 3);
        return d;
    def run(self):
        while True:
            #获取传感器ID/地址
            addr = self.q1.get();
            #通过同步队列返回结果
            self.q2.put([item, self.read_data(addr), time.time()]);
            q1.task_done();
>
if __name__ == '__main__':
    i2c = smbus.SMBus(1);
    #初始化两个同步队列
    q1 = Queue();
    q2 = Queue();
    #开启三个处理测量的线程
    for i in range(3):
        t = Worker(q1, q2, i2c);
        t.daemon = True;
        t.start();
    while True:
        #将三个传感器的ID/地址写入同步队列
        for addr in [0x68, 0x69, 0x6a]:
            q1.put(addr);
        #等待处理完成
        q1.join();
        #从另一个同步队列中获取测量结果
        items = [];
        for i in range(3):
            items.append(q2.get());
            q2.task_done();
        #按照传感器的ID/地址排序返回数据，并输出
        print(sorted(items, key = lambda x : x[0]));
```
>
> 程序的运行结果如下
>
```
[[0, 0.022960141856268712, 1475155373.765486], [1, 0.9793564760846006, 1475155373.7656171], [2, 0.7148516408250839, 1475155373.7656686]]
[[0, 0.47161535731418647, 1475155373.8682373], [1, 0.5808744331485662, 1475155373.8683677], [2, 0.4473773800210128, 1475155373.8684156]]
[[0, 0.3093647262512067, 1475155373.970781], [1, 0.43642747239785196, 1475155373.9709117], [2, 0.5162445579347241, 1475155373.9709547]]
```
> 可以看出测量时间之差在毫秒以下，能够保证有较好的测量同步性。
