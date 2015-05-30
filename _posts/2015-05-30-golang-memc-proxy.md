---
title: 使用Go语言实现一个简单的Memcache代理服务
layout: post
---

> 之前研究Memcache的源代码主要是想实现一个Memcache的代理服务，实现缓存的异步更新（或过期）操作，但把Memcache的核心代码提取出来还是个不小的工程，其中相当多的代码在做各种错误的处理，估计把其核心代码提取成一个完备的可以使用的项目，估计代码会有几千行之多（这个工作我还会试着继续进行）。这里我们用最近想起的Go语言来实现一个多线程的Memcache代理服务器。

#### 1. Go语言简介

> Go语言是Google在2007年开发的一种编程语言，是一种静态类型语法类似C语言，支持垃圾收集，类型安全，并且支持动态类型容器和内置的标准库（变长数组和哈希表），并且目前已经可以运行在大多数硬件平台和操作系统上。Go语言的官方主页和Google一样在中国大陆是访问不了的，一些相关信息可以参考维基百科[http://en.wikipedia.org/wiki/Go_(programming_language)](http://en.wikipedia.org/wiki/Go_%28programming_language%29)。个人对此语言的最大感受就是其试图让编写并行程序编得更加简单。

#### 2. 协程

> 协程是目前并行编程里一个比较重要的概念，其在使用上比较类似于线程，但其并不一定真正使用了操作系统级别的线程，Lua中的协程就是一种通过栈保存运行状态来模拟的“线程”，因此协程的创建要比创建操作系统级别的线程开销会小很多。其真正的精髓在于协程之间可以通过yeild让出自己的执行，从而降低等待时间，同时相比回调的编程方式要更加直观，更加接近一般的顺序编程。这方面比较好的介绍和文章还比较少，作者对此理解程度也比较有限，前面只是我目前的理解。

> Go语言中的协程使用非常简单，只需要将协程中的代码放到一个函数中，并在执行前增加go关键字即可。例如如下代码
>
```go
package main;
>
import "fmt";
import "time";
>
func main() {
    go func() {
        for i := 1; i < 10; i++ {
            fmt.Println(i);
        }
    } ();
    fmt.Println("Hello World!");
    time.Sleep(3 * time.Second);
}
```
> 执行的结果一般会如下，可见协程还没有执行完，就执行了主函数的后面的部分
>
```
Hello World!
1
2
3
4
5
6
7
8
9
```

#### 3. Go的安装和使用

> 一般的Linux软件包中一般都包含了Go语言的运行环境，在ArchLinux中直接通过pacman即可完成安装，可以看到Go的安装包还是比较大的。
>
```
pacman -S go
>
resolving dependencies...
looking for conflicting packages...
>
Packages (1) go-2:1.4.2-2
>
Total Download Size:    52.56 MiB
Total Installed Size:  343.07 MiB
```
>
> Go语言的Hello World，基本和C语言差不多
>
```go
package main;
>
import "fmt";
>
func main() {
	fmt.Println("Hello World!");
}
```
> 编译和运行，也可以直接运行（其实是编译一个临时文件，然后运行）
>
```
go build hello.go
./hello 
Hello World!
>
go run hello.go
Hello World!
```

#### 4. 使用Go语言实现一个Memcache代理

> 首先使用net包中的Listen创建一个服务地址和端口，然后使用Go语言的Memcache模块和要进行同步的Memcache进行连接，同时将和客户端连接获取数据与向Memcache同步使用两组不同的协程，每和客户端建立一个连接就启动一个协程进行处理，这里解析Memcache协议为了简单使用了正则表达式，如果要提高效率可以模仿Memcache源代码的过程，同时两组协程之间通过keys这个字符串类型的channel进行数据的交互，解析协议完成后，将解析到的key传入channel就会出发同步协程进行工作，比较类似线程中使用消息队列进行数据交互。
>
> 可以看到，使用Go实现的Memcache代理代码只需要不到100行，如果增加了对各种出错的处理应该也不会超过200行，并且由于创建协程的代价较小，不需要像线程一样实现线程池的相关逻辑，也不需要实现通过消息队列进行通信的逻辑，大大简化了并行编程的难度，需要做的只是将可以异步处理的代码，放到协程中执行即可。
>
```go
package main;
>
import "net";
import "ketama";
import "memcache";
import "regexp";
>
func main() {
    l, err := net.Listen("tcp", "127.0.0.1:1987");
    handle_error(err);
	//使用ketama一致性散列算法
    selector, err := ketama.NewFromFile("servers.txt");
    handle_error(err);
	//创建到需要同步的Memcache之间的连接
    mc := memcache.NewFromSelector(selector);
	//创建一个消息队列
    keys := make(chan string);
>
	//开启2个同步协程
    n := 2;
    for i := 0; i < n; i++ {
         go func() {
             for {
                 key := <-keys;
                 mc.Delete(key);
             }
         } ();
    }
>
	//程序结束时关闭地址监听
    defer l.Close();
	//接受客户端的连接
    for {
        conn, err := l.Accept();
        handle_error(err);
>
        go func(conn net.Conn) {
			//结束后关闭和客户端的连接
            defer conn.Close();
            //接收数据，解析命令，并将key放入channel
			for {
                recv := read_network(conn);
                commands := read_commmand(recv);
                if commands != nil {
                    keys <- commands[2];
                    switch commands[1] {
                        case "set":
                            conn.Write([]byte("STORED\r\n"));
                        case "add":
                            conn.Write([]byte("STORED\r\n"));
                        case "replace":
                            conn.Write([]byte("STORED\r\n"));
                        case "delete":
                            conn.Write([]byte("DELETED\r\n"));
                    }
                } else {
                    break;
                }
            }
        } (conn);
    }
}
>
//读取数据
func read_network(conn net.Conn) string {
    buf := make([]byte, 384);
    n, err := conn.Read(buf);
    handle_error(err);
    recv := string(buf[0:n]);
    return recv;
}
>
//解析协议
func read_commmand(in string) []string {
    r := regexp.MustCompile("^(set|delete|add|replace) (\\w+).*\r\n");
    matches := r.FindStringSubmatch(in);
    return matches;
}
>
func handle_error(err error) {
    if err != nil {
        panic(err);
    }
}
```
