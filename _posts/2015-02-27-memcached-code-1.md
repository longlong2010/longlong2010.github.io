---
title: Memcached事件模型分析——线程池与连接的建立
layout: post
---

> 之前一直想看一下Memcached的事件模型的实现方法，最近终于有时间来研究一下。看到网上相关的文章也比较多，这里只写一下我自己的理解，如果有错误还请指出。代码为了表达主要的思想都进行了大幅度的简化。 

#### 1. 总体结构

> Memcached总体的事件模型代码在thread.c和memcached.c两个文件中，其中thread.c中主要实现了线程池，memcached.c中主要实现事件响应和连接的处理。事件驱动部分使用了比较常见的libevent。
>
> 主线程主要负责响应外部的连接，而工作线程池则和主线程之间通过管道进行通信——每一个工作线程注册一个对管道的事件响应，主线程在接收到外部连接后，轮询工作线程池找到下一个工作线程，并通过向其对应的管道中写入一个字节的方式触发事件响应并完成处理操作。主线程和工作线程之间通过每个工作线程自身的一个队列进行数据的传递——主线程接收到连接后，将连接相关的信息封装为一个对象推入队列，并通过管道激活对应的工作线程，工作线程从自身的队列中取出相应的数据进行处理。

#### 2. 线程池的实现

> 如第一节中所述，每个工作线程都包含两个管道的描述符，一个事件结构和一个队列。工作线程的结构如下
>
```c++
class conn_queue {
private:
    std::queue<int>* queue;
    pthread_mutex_t lock;
public:
    conn_queue();
    int pop();
    void push(int fd);
    int size();
    ~conn_queue();
};
>
typedef struct {
	pthread_t thread_id;       
	struct event_base *base;   //libevent句柄
	struct event notify_event; //通知事件结构
	int notify_receive_fd;     //触发工作线程的描述符
	int notify_send_fd;     
	conn_queue* queue; 	   	   //连接队列，使用一个封装了自动加锁的队列
} LIBEVENT_THREAD;
```
>
> 每个工作线程的初始化过程如下
>
```c++
//工作线程池指针
static LIBEVENT_THREAD *threads;
>
//初始化线程的事件和连接队列
static void setup_thread(LIBEVENT_THREAD *me) {
	//初始化事件
    me->base = event_init();
>
	//增加对notify_receive_fd描述符的事件响应
    event_set(&me->notify_event, me->notify_receive_fd,
              EV_READ | EV_PERSIST, thread_libevent_process, me);
    event_base_set(me->base, &me->notify_event);
>
    if (event_add(&me->notify_event, 0) == -1) {
        fprintf(stderr, "Can't monitor libevent notify pipe\n");
        exit(1);
    }
>
	//初始化连接队列
	me->queue = new conn_queue();
}
>
void memcached_thread_init(int nthreads, struct event_base *main_base) {
	pthread_mutex_init(&init_lock, NULL);
    pthread_cond_init(&init_cond, NULL);
>	
	threads = calloc(nthreads, sizeof(LIBEVENT_THREAD));
	//初始化每个工作线程
    for (int i = 0; i < nthreads; i++) {
		//开启管道
        int fds[2];
>
        threads[i].notify_receive_fd = fds[0];
        threads[i].notify_send_fd = fds[1];
>
		//初始化工作线程的事件结构和连接队列
        setup_thread(&threads[i]);
    }
>
	//启动线程 事件循环就在线程的回调函数worker_libevent中启动
    for (int i = 0; i < nthreads; i++) {
        create_worker(worker_libevent, &threads[i]);
    }
> 
	//等待全部工作线程启动完毕
	pthread_mutex_lock(&init_lock);
    wait_for_thread_registration(nthreads);
    pthread_mutex_unlock(&init_lock);
}
``` 

#### 3. 外部连接的处理和线程调度

> 对于外部连接的处理相关的逻辑在主线程中，大致和libevent的单线程用法类似——响应监听描述符的事件，这部分的逻辑在memcached的main函数中，原始的代码中对监听描述符和管道描述符的事件响应使用了同一个处理函数event\_handler，而是通过连接的不同状态在drive\_machine中进行区分处理
>
```c++
static struct event_base *main_base;
static int last_thread = -1;
>
void base_event_handler(int sock, short event, void* arg) {
	//接收连接
    struct sockaddr_in cli_addr;
    int newfd;
    socklen_t sin_size;
    sin_size = sizeof(struct sockaddr_in);
    newfd = accept(sock, (struct sockaddr*)&cli_addr, &sin_size);
>
	//选择线程，使用轮询的方式进行选择
    int tid = (last_thread + 1) % THREAD_NUM;
    LIBEVENT_THREAD* thread = threads + tid;
    last_thread = tid;
>
	//将连接描述符推入队列
	thread->queue->push(newfd);
>
	//向管道中写入一个空字符激活触发工作线程的事件响应 
    write(thread->notify_send_fd, " ", 1); 
}
>
int main() {
    main_base = event_init();
	//初始化线程池
    thread_init(THREAD_NUM, main_base);
>
	//开启对地址和端口的监听
    int sfd = server_socket("127.0.0.1", 11212);
	//增加对应监听描述符的事件响应
    struct event listen_ev;
    event_set(&listen_ev, sfd, EV_READ | EV_PERSIST, base_event_handler, NULL);
    event_base_set(main_base, &listen_ev);
    event_add(&listen_ev, NULL);
>
	//启动事件循环    
    event_base_loop(main_base, 0); 
    return 0;
}
```

#### 4. 总结

> Memcached的事件模型是使用libevent实现的多线程TCP类服务器的比较经典的实例，在开发一些轻量级的服务组件时非常具有参考意义，以上只是简单的分析了Memcached的线程池和连接处理的机制，后续的数据的读写和连接的保持和关闭部分还未涉及，之后会对剩下的部分加以分析和研究，最终目的是为了得到一个编写多线程TCP类服务组件的代码框架。 
