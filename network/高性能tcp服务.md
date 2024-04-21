### 主要设计思路 ###

* event_loop 它就是一个无限循环着的事件分发器，一旦有事件发生，它就会回调预先定义好的回调函数，完成事件的处理。具体来说，event_loop 使用 poll 或者 epoll 方法将一个线程阻塞，等待各种 I/O 事件的发生。
* 对各种注册到 event_loop 上的对象，我们抽象成 channel 来表示，例如注册到 event_loop 上的监听事件，注册到 event_loop 上的套接字读写事件等
* event_dispatcher 是对事件分发机制的一种抽象，也就是说，可以实现一个基于 poll 的 poll_dispatcher，也可以实现一个基于 epoll 的 epoll_dispatcher。在这里，我们统一设计一个 event_dispatcher 结构体
* channel_map 保存了描述字到 channel 的映射，这样就可以在事件发生时，根据事件类型对应的套接字快速找到 channel 对象里的事件处理函数。

#### I/O 模型和多线程模型设计 ####

thread_pool 维护了一个 sub-reactor 的线程列表，它可以提供给主 reactor 线程使用，每次当有新的连接建立时，可以从 thread_pool 里获取一个线程，以便用它来完成对新连接套接字的 read/write 事件注册，将 I/O 线程和主 reactor 线程分离。

#### Buffer 和数据读写 ####

* buffer 对象屏蔽了对套接字进行的写和读的操作，如果没有 buffer 对象，连接套接字的 read/write 事件都需要和字节流直接打交道，这显然是不友好的

* tcp_connection 这个对象描述的是已建立的 TCP 连接。它的属性包括接收缓冲区、发送缓冲区、channel 对象等。这些都是一个 TCP 连接的天然属性。tcp_connection 是大部分应用程序和我们的高性能框架直接打交道的数据结构。我们不想把最下层的 channel 对象暴露给应用程序



#### 反应堆模式设计 ####

![](https://static001.geekbang.org/resource/image/7a/61/7ab9f89544aba2021a9d2ceb94ad9661.jpg?wh=3499*1264)

当 event_loop_run 完成之后，线程进入循环，首先执行 dispatch 事件分发，一旦有事件发生，就会调用 channel_event_activate 函数，在这个函数中完成事件回调函数 eventReadcallback 和 eventWritecallback 的调用，最后再进行 event_loop_handle_pending_channel，用来修改当前监听的事件列表，完成这个部分之后，又进入了事件分发循环。

##### event_loop 分析 #####

最重要的莫过于 event_dispatcher 对象了。你可以简单地把 event_dispatcher 理解为 poll 或者 epoll，它可以让我们的线程挂起，等待事件的发生

就是 event_dispatcher_data，它被定义为一个 void * 类型，可以按照我们的需求，任意放置一个我们需要的对象指针。这样，针对不同的实现，例如 poll 或者 epoll，都可以根据需求，放置不同的数据对象。



socketPair 是父线程用来通知子线程有新的事件需要处理。pending_head 和 pending_tail 是保留在子线程内的需要处理的新事件。

```c
struct event_loop {
    int quit;
    const struct event_dispatcher *eventDispatcher;

    /** 对应的event_dispatcher的数据. */
    void *event_dispatcher_data;
    struct channel_map *channelMap;

    int is_handle_pending;
    struct channel_element *pending_head;
    struct channel_element *pending_tail;

    pthread_t owner_thread_id;
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    int socketPair[2];
    char *thread_name;
};
```

看一下 event_loop 最主要的方法 event_loop_run 方法，前面提到过，event_loop 就是一个无限 while 循环，不断地在分发事件。

```c
/**
 *
 * 1.参数验证
 * 2.调用dispatcher来进行事件分发,分发完回调事件处理函数
 */
int event_loop_run(struct event_loop *eventLoop) {
    assert(eventLoop != NULL);

    struct event_dispatcher *dispatcher = eventLoop->eventDispatcher;

    if (eventLoop->owner_thread_id != pthread_self()) {
        exit(1);
    }

    yolanda_msgx("event loop run, %s", eventLoop->thread_name);
    struct timeval timeval;
    timeval.tv_sec = 1;

    while (!eventLoop->quit) {
        //block here to wait I/O event, and get active channels
        dispatcher->dispatch(eventLoop, &timeval);

        //handle the pending channel
        event_loop_handle_pending_channel(eventLoop);
    }

    yolanda_msgx("event loop end, %s", eventLoop->thread_name);
    return 0;
}
```

##### 多线程设计的几个考虑 #####

当用户期望使用多个 sub-reactor 子线程时，主线程会创建多个子线程，每个子线程在创建之后，按照主线程指定的启动函数立即运行，并进行初始化。随之而来的问题是，主线程如何判断子线程已经完成初始化并启动，继续执行下去呢？

从 **thread_pool 中取出一个线程，通知这个线程有新的事件加入。而这个线程很可能是处于事件分发的阻塞调用之中，如何协调主线程数据写入给子线程**

子线程是一个 event_loop 线程，它阻塞在 dispatch 上，一旦有事件发生，它就会查找 channel_map，找到对应的处理函数并执行它。之后它就会增加、删除或修改 pending 事件，再次进入下一轮的 dispatch。



##### 主线程等待多个 sub-reactor 子线程初始化完 #####

调用 event_loop_thread_init 对每个子线程初始化，之后调用 event_loop_thread_start 来启动子线程。

```c
//一定是main thread发起
void thread_pool_start(struct thread_pool *threadPool) {
    assert(!threadPool->started);
    assertInSameThread(threadPool->mainLoop);

    threadPool->started = 1;
    void *tmp;

    if (threadPool->thread_number <= 0) {
        return;
    }

    threadPool->eventLoopThreads = malloc(threadPool->thread_number * sizeof(struct event_loop_thread));
    for (int i = 0; i < threadPool->thread_number; ++i) {
        event_loop_thread_init(&threadPool->eventLoopThreads[i], i);
        event_loop_thread_start(&threadPool->eventLoopThreads[i]);
    }
}
```

使用了 pthread_create 创建了子线程，子线程一旦创建，立即执行 event_loop_thread_run,pthread_mutex_lock 和 pthread_mutex_unlock 进行了加锁和解锁，并使用了 **pthread_cond_wait 来守候 eventLoopThread 中的 eventLoop 的变量。**



```c
//由主线程调用，初始化一个子线程，并且让子线程开始运行event_loop
struct event_loop *event_loop_thread_start(struct event_loop_thread *eventLoopThread) {
    pthread_create(&eventLoopThread->thread_tid, NULL, &event_loop_thread_run, eventLoopThread);

    assert(pthread_mutex_lock(&eventLoopThread->mutex) == 0);

    while (eventLoopThread->eventLoop == NULL) {
        assert(pthread_cond_wait(&eventLoopThread->cond, &eventLoopThread->mutex) == 0);
    }
    assert(pthread_mutex_unlock(&eventLoopThread->mutex) == 0);

    yolanda_msgx("event loop thread started, %s", eventLoopThread->thread_name);
    return eventLoopThread->eventLoop;
}
```

子线程执行函数 event_loop_thread_run 一上来也是进行了加锁，之后初始化 event_loop 对象，**当初始化完成之后，调用了 pthread_cond_signal 函数来通知此时阻塞在 pthread_cond_wait 上的主线程。**这样，主线程就会从 wait 中苏醒，代码得以往下执行。

```c
void *event_loop_thread_run(void *arg) {
    struct event_loop_thread *eventLoopThread = (struct event_loop_thread *) arg;

    pthread_mutex_lock(&eventLoopThread->mutex);

    // 初始化化event loop，之后通知主线程
    eventLoopThread->eventLoop = event_loop_init();
    yolanda_msgx("event loop thread init and signal, %s", eventLoopThread->thread_name);
    pthread_cond_signal(&eventLoopThread->cond);

    pthread_mutex_unlock(&eventLoopThread->mutex);

    //子线程event loop run
    eventLoopThread->eventLoop->thread_name = eventLoopThread->thread_name;
    event_loop_run(eventLoopThread->eventLoop);
}
```

只有当子线程完成了初始化，才变成一个非 NULL 的值，这个变化是子线程完成初始化的标志，也是信号量守护的变量。通过使用锁和信号量，解决了主线程和子线程同步的问题。

##### 增加已连接套接字事件到 sub-reactor 线程中 #####

把这个已连接套接字相关的 I/O 事件交给 sub-reactor 子线程负责检测。这样的好处是，main reactor 只负责连接套接字的建立，可以一直维持在一个非常高的处理效率，在多核的情况下，多个 sub-reactor 可以很好地利用上多核处理的优势。



如何让 sub-reactor 线程从 event_dispatcher 的 dispatch 上返回呢？答案是构建一个类似管道一样的描述字，让 event_dispatcher 注册该管道描述字，当我们想让 sub-reactor 线程苏醒时，往管道上发送一个字符就可以了。



在 event_loop_init 函数里，调用了 socketpair 函数创建了套接字对，这个套接字对的作用就是往这个套接字的一端写时，另外一端就可以感知到读的事件。其实，这里也可以直接使用 UNIX 上的 pipe 管道，作用是一样的。

```c
struct event_loop *event_loop_init() {
    ...
    //add the socketfd to event 这里创建的是套接字对，目的是为了唤醒子线程
    eventLoop->owner_thread_id = pthread_self();
    if (socketpair(AF_UNIX, SOCK_STREAM, 0, eventLoop->socketPair) < 0) {
        LOG_ERR("socketpair set fialed");
    }
    eventLoop->is_handle_pending = 0;
    eventLoop->pending_head = NULL;
    eventLoop->pending_tail = NULL;
    eventLoop->thread_name = "main thread";

    struct channel *channel = channel_new(eventLoop->socketPair[1], EVENT_READ, handleWakeup, NULL, eventLoop);
    event_loop_add_channel_event(eventLoop, eventLoop->socketPair[1], channel);

    return eventLoop;
}
```

看看这个 handleWakeup 函数：

```c
int handleWakeup(void * data) {
    struct event_loop *eventLoop = (struct event_loop *) data;
    char one;
    ssize_t n = read(eventLoop->socketPair[1], &one, sizeof one);
    if (n != sizeof one) {
        LOG_ERR("handleWakeup  failed");
    }
    yolanda_msgx("wakeup, %s", eventLoop->thread_name);
}
```

这个函数就是简单的从 socketPair[1]描述字上读取了一个字符而已，除此之外，它什么也没干。它的主要作用就是让子线程从 dispatch 的阻塞中苏醒。



如果有新的连接产生，主线程是怎么操作的？在 handle_connection_established 中，通过 accept 调用获取了已连接套接字, 接下来调用 thread_pool_get_loop 获取一个 event_loop。**thread_pool_get_loop 的逻辑非常简单，从 thread_pool 线程池中按照顺序挑选出一个线程来服务。接下来是创建了 tcp_connection 对象。**

```c
//处理连接已建立的回调函数
int handle_connection_established(void *data) {
    struct TCPserver *tcpServer = (struct TCPserver *) data;
    struct acceptor *acceptor = tcpServer->acceptor;
    int listenfd = acceptor->listen_fd;

    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);
    //获取这个已建立的套集字，设置为非阻塞套集字
    int connected_fd = accept(listenfd, (struct sockaddr *) &client_addr, &client_len);
    make_nonblocking(connected_fd);

    yolanda_msgx("new connection established, socket == %d", connected_fd);

    //从线程池里选择一个eventloop来服务这个新的连接套接字
    struct event_loop *eventLoop = thread_pool_get_loop(tcpServer->threadPool);

    // 为这个新建立套接字创建一个tcp_connection对象，并把应用程序的callback函数设置给这个tcp_connection对象
    struct tcp_connection *tcpConnection = tcp_connection_new(connected_fd, eventLoop,tcpServer->connectionCompletedCallBack,tcpServer->connectionClosedCallBack,tcpServer->messageCallBack,tcpServer->writeCompletedCallBack);
    //callback内部使用
    if (tcpServer->data != NULL) {
        tcpConnection->data = tcpServer->data;
    }
    return 0;
}
```

在调用 tcp_connection_new 创建 tcp_connection 对象的代码里，可以看到先是创建了一个 channel 对象，并注册了 READ 事件，之后调用 event_loop_add_channel_event 方法往子线程中增加 channel 对象。

```c
tcp_connection_new(int connected_fd, struct event_loop *eventLoop,
                   connection_completed_call_back connectionCompletedCallBack,
                   connection_closed_call_back connectionClosedCallBack,
                   message_call_back messageCallBack, write_completed_call_back writeCompletedCallBack) {
    ...
    //为新的连接对象创建可读事件
    struct channel *channel1 = channel_new(connected_fd, EVENT_READ, handle_read, handle_write, tcpConnection);
    tcpConnection->channel = channel1;

    //完成对connectionCompleted的函数回调
    if (tcpConnection->connectionCompletedCallBack != NULL) {
        tcpConnection->connectionCompletedCallBack(tcpConnection);
    }
  
    //把该套集字对应的channel对象注册到event_loop事件分发器上
    event_loop_add_channel_event(tcpConnection->eventLoop, connected_fd, tcpConnection->channel);
    return tcpConnection;
}
```



接下来的部分是重点，如果当前增加 **channel event 的不是当前 event loop 线程自己，就会调用 event_loop_wakeup 函数把 event_loop 子线程唤醒**。唤醒的方法很简单，就是往刚刚的 socketPair[0]上写一个字节，别忘了，event_loop 已经注册了 socketPair[1]的可读事件。**如果当前增加 channel event 的是当前 event loop 线程自己，则直接调用 event_loop_handle_pending_channel 处理新增加的 channel event 事件列表。**

```c
int event_loop_do_channel_event(struct event_loop *eventLoop, int fd, struct channel *channel1, int type) {
    //get the lock
    pthread_mutex_lock(&eventLoop->mutex);
    assert(eventLoop->is_handle_pending == 0);
    //往该线程的channel列表里增加新的channel
    event_loop_channel_buffer_nolock(eventLoop, fd, channel1, type);
    //release the lock
    pthread_mutex_unlock(&eventLoop->mutex);
    //如果是主线程发起操作，则调用event_loop_wakeup唤醒子线程
    if (!isInSameThread(eventLoop)) {
        event_loop_wakeup(eventLoop);
    } else {
        //如果是子线程自己，则直接可以操作
        event_loop_handle_pending_channel(eventLoop);
    }

    return 0;
}
```



































