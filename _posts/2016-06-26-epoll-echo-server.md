---
title: 使用Epoll实现Echo Server
layout: post
---

> 由于一些需求，需要在PC上读取超声波传感器的数据，而超声波传感器使用的是I<SUP>2</SUP>C/TTL接口，虽然可以使用TTL转USB来接入PC，但是TTL接口只能够支持同时连接两个超声波传感器，而这里可能需要使用6个。
>
> 想到的一只解决办法是将超声波传感器连接到Raspberry PI的GPIO的I<SUP>2</SUP>C接口上，然后PC和Raspberry PI使用WIFI进行通信。这里需要在Raspberry PI上启动一个TCP Server，PC通过调用此TCP Server的接口获取超声波传感器的测量数据。虽然也可以让Raspberry PI将超声波传感器的数据发送到PC，但这样就需要让超声波传感器一直处于工作状态，会带来较大的能耗。

#### 1. 实现一个简单的Echo Server

> Echo Server可以作为一个最简单的TCP Server的模版，通过简单的修改就可以实现其他的TCP通信服务。一般的TCP Echo Server实现大致如下，由于Raspberry PI中的主流编程语言是Python，以下代码均使用Python。
>
```python
from SocketServer import BaseRequestHandler, TCPServer;
>
class ServerHandler(BaseRequestHandler):
    def handle(self):
        data = self.request.recv(1024);
        self.request.send(data);
>
server = TCPServer(('', 1987), ServerHandler);
server.serve_forever();
```
> 测试效果如下
>
```
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
Hello
Hello
Connection closed by foreign host.
```
>
> 这里使用了Python中的TCPServer框架，但一般的TCPServer框架所实现的方式基本是类似HTTP Server，即处理完成后就关闭连接，并不能支持连接的服务。而获取超声波传感器的频率可能是比较高的，因此如果每次都重新建立连接会产生较大的开销，同时也增加了测量的延迟。

#### 2. Linux的Epoll系统调用

> Epoll是Linux 2.6中支持的路复用IO接口，是对select/poll的增强版本，显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用效率。目前大多数Linux下的应用程序（Nginx Redis等）都使用了Epoll方式进行网络IO操作。同时Epoll提供了水平触发(LT)和边界触发(ET)两种IO事件。

#### 3. 使用Epoll实现Echo Server

> 这里我们只能直接使用Epoll调用来实现连接复用的Echo Server，其中关键是将数据输出之后重新调整为接收数据的模式，其实现的代码大致如下
>
```python
import socket;
import select;
>
#开启一个Socket
HOST = '';
PORT = 1987
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.bind((HOST, 1987));
sock.listen(1);
>
#初始化Epoll
epoll = select.epoll();
epoll.register(sock.fileno(), select.EPOLLIN);
>
#连接和接受数据
conns = {};
recvs = {};
>
try:
    while True:
        #等待事件发生
        events = epoll.poll(1);
        #事件循环    
        for fd, event in events:
            #如果监听的Socket有时间则接受新连接
            if fd == sock.fileno():
                client, addr = sock.accept();
                client.setblocking(0);
                #注册新连接的输入时间
                epoll.register(client.fileno(), select.EPOLLIN);
                conns[client.fileno()] = client;
                recvs[client.fileno()] = '';
            elif event & select.EPOLLIN:
                #读取数据
                while True:
                    try:
                        buff = conns[fd].recv(1024);
                        if len(buff) == 0:
                            break;
                    except:
                        break;            
                    recvs[fd] += buff;
                #调整输出事件
                if len(buff) != 0:
                    epoll.modify(fd, select.EPOLLOUT);
                else:
                #如果数据为空则连接已断开
                    epoll.modify(fd, select.EPOLLHUP);
            elif event & select.EPOLLOUT:
                #发送数据
                try:
                    n = conns[fd].send(recvs[fd]);
                    recvs[fd] = '';
                    #重新调整为接收数据
                    epoll.modify(fd, select.EPOLLIN);
                except:
                    epoll.modify(fd, select.EPOLLHUP);
            elif event & select.EPOLLHUP:
                #关闭清理连接
                epoll.unregister(fd);
                conns[fd].close();
                del conns[fd];
                del recvs[fd];
finally:
    epoll.unregister(sock.fileno());
    epoll.close();
    sock.close();
```

#### 4. 测试效果

> 使用Telnet连接上Echo Server，先输入Hello，会返回Hello，再输入Hi，则会继续返回Hi 
>
```
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
Hello
Hello
Hi
Hi
```
