### event_loop 分析 ###

最重要的莫过于 event_dispatcher 对象了。你可以简单地把 event_dispatcher 理解为 poll 或者 epoll，它可以让我们的线程挂起，等待事件的发生。

event_dispatcher_data，它被定义为一个 void * 类型，可以按照我们的需求，任意放置一个我们需要的对象指针。这样，针对不同的实现，例如 poll 或者 epoll，都可以根据需求，放置不同的数据对象

socketPair 是父线程用来通知子线程有新的事件需要处理(也可以通过管道pipe来唤醒sub reactor线程)。pending_head 和 pending_tail 是保留在子线程内的需要处理的新事件。

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

### event_dispacher 分析 ###

为了实现不同的事件分发机制，这里把 poll、epoll 等抽象成了一个 event_dispatcher 结构。event_dispatcher 的具体实现有 poll_dispatcher 和 epoll_dispatcher 两种

```c
/** 抽象的event_dispatcher结构体，对应的实现如select,poll,epoll等I/O复用. */
struct event_dispatcher {
    /**  对应实现 */
    const char *name;

    /**  初始化函数 */
    void *(*init)(struct event_loop * eventLoop);

    /** 通知dispatcher新增一个channel事件*/
    int (*add)(struct event_loop * eventLoop, struct channel * channel);

    /** 通知dispatcher删除一个channel事件*/
    int (*del)(struct event_loop * eventLoop, struct channel * channel);

    /** 通知dispatcher更新channel对应的事件*/
    int (*update)(struct event_loop * eventLoop, struct channel * channel);

    /** 实现事件分发，然后调用event_loop的event_activate方法执行callback*/
    int (*dispatch)(struct event_loop * eventLoop, struct timeval *);

    /** 清除数据 */
    void (*clear)(struct event_loop * eventLoop);
};
```

### channel 对象分析 ###

channel 对象是用来和 event_dispather 进行交互的最主要的结构体，它抽象了事件分发。一个 channel 对应一个描述字，描述字上可以有 READ 可读事件，也可以有 WRITE 可写事件。channel 对象绑定了事件处理函数 event_read_callback 和 event_write_callback。

```c
typedef int (*event_read_callback)(void *data);

typedef int (*event_write_callback)(void *data);

struct channel {
    int fd;
    int events;   //表示event类型

    event_read_callback eventReadCallback;
    event_write_callback eventWriteCallback;
    void *data; //callback data, 可能是event_loop，也可能是tcp_server或者tcp_connection
};
```

### channel_map 对象分析 ###

event_dispatcher 在获得活动事件列表之后，需要通过文件描述字找到对应的 channel，从而回调 channel 上的事件处理函数 event_read_callback 和 event_write_callback，为此，设计了 channel_map 对象。

```c
/**
 * channel映射表, key为对应的socket描述字
 */
struct channel_map {
    void **entries;

    /* The number of entries available in entries */
    int nentries;
};
```

channel_map 对象是一个数组，数组的下标即为描述字，数组的元素为 channel 对象的地址。

这样，**当 event_dispatcher 需要回调 channel 上的读、写函数时，调用 channel_event_activate 就可以**，下面是 channel_event_activate 的实现，在找到了对应的 channel 对象之后，根据事件类型，回调了读函数或者写函数。注意，**这里使用了 EVENT_READ 和 EVENT_WRITE 来抽象了 poll 和 epoll 的所有读写事件类型。**

```c
int channel_event_activate(struct event_loop *eventLoop, int fd, int revents) {
    struct channel_map *map = eventLoop->channelMap;
    yolanda_msgx("activate channel fd == %d, revents=%d, %s", fd, revents, eventLoop->thread_name);

    if (fd < 0)
        return 0;

    if (fd >= map->nentries)return (-1);

    struct channel *channel = map->entries[fd];
    assert(fd == channel->fd);

    if (revents & (EVENT_READ)) {
        if (channel->eventReadCallback) channel->eventReadCallback(channel->data);
    }
    if (revents & (EVENT_WRITE)) {
        if (channel->eventWriteCallback) channel->eventWriteCallback(channel->data);
    }

    return 0;
}
```

### 增加、删除、修改 channel event ###

```c
int event_loop_add_channel_event(struct event_loop *eventLoop, int fd, struct channel *channel1);

int event_loop_remove_channel_event(struct event_loop *eventLoop, int fd, struct channel *channel1);

int event_loop_update_channel_event(struct event_loop *eventLoop, int fd, struct channel *channel1);
```

前面三个函数提供了入口能力，而真正的实现则落在这三个函数上：

```c
int event_loop_handle_pending_add(struct event_loop *eventLoop, int fd, struct channel *channel);

int event_loop_handle_pending_remove(struct event_loop *eventLoop, int fd, struct channel *channel);

int event_loop_handle_pending_update(struct event_loop *eventLoop, int fd, struct channel *channel);
```

event_loop_handle_pending_add 在当前 event_loop 的 channel_map 里增加一个新的 key-value 对，key 是文件描述字，value 是 channel 对象的地址。之后**调用 event_dispatcher 对象的 add 方法增加 channel event 事件**。**注意这个方法总在当前的 I/O 线程中执行。**



### 多线程设计的几个考虑 ###

main reactor 线程是一个 acceptor 线程，这个线程一旦创建，会以 event_loop 形式阻塞在 event_dispatcher 的 dispatch 方法上，实际上，它在等待监听套接字上的事件发生，也就是已完成的连接，一旦有连接完成，就会创建出连接对象 tcp_connection，以及 channel 对象等。

子线程是一个 event_loop 线程，它阻塞在 dispatch 上，一旦有事件发生，它就会查找 channel_map，找到对应的处理函数并执行它。之后它就会增加、删除或修改 pending 事件，再次进入下一轮的 dispatch。



### 主线程等待多个 sub-reactor 子线程初始化完 ###

主线程需要等待子线程完成初始化，也就是需要获取子线程对应数据的反馈,实际上这是一个多线程的通知问题。使用 mutex 和 condition 两个主要武器。

下面这段代码是主线程发起的子线程创建，调用 event_loop_thread_init 对每个子线程初始化，之后调用 event_loop_thread_start 来启动子线程

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

* 我们再看一下 event_loop_thread_start 这个方法，这个方法一定是主线程运行的。这里我使用了 pthread_create 创建了子线程，子线程一旦创建，立即执行 event_loop_thread_run
* event_loop_thread_run 进行了子线程的初始化工作。这个函数最重要的部分是使用了 **pthread_mutex_lock 和 pthread_mutex_unlock 进行了加锁和解锁**，并使用了 pthread_cond_wait 来守候 eventLoopThread 中的 eventLoop 的变量。

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

子线程执行函数 event_loop_thread_run 一上来也是进行了加锁，之后初始化 event_loop 对象，当初始化完成之后，调用了 pthread_cond_signal 函数来通知此时阻塞在 pthread_cond_wait 上的主线程

这样，主线程就会从 wait 中苏醒，代码得以往下执行。子线程本身也通过调用 event_loop_run 进入了一个无限循环的事件分发执行体中，**等待子线程 reator 上注册过的事件发生**

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

这里主线程和子线程共享的变量正是每个 event_loop_thread 的 eventLoop 对象，这个对象在初始化的时候为 NULL，只有当子线程完成了初始化，才变成一个非 NULL 的值，这个变化是子线程完成初始化的标志

也是信号量守护的变量。通过使用锁和信号量，解决了主线程和子线程同步的问题。当子线程完成初始化之后，主线程才会继续往下执行。

```c
struct event_loop_thread {
    struct event_loop *eventLoop;
    pthread_t thread_tid;        /* thread ID */
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    char * thread_name;
    long thread_count;    /* # connections handled */
};
```

### 增加已连接套接字事件到 sub-reactor 线程中 ###

当有事件发生时，也就是一个连接已完成建立，如果我们有多个 sub-reactor 子线程，我们期望的结果是，把这个已连接套接字相关的 I/O 事件交给 sub-reactor 子线程负责检测

sub-reactor 线程是一个无限循环的 event loop 执行体，在没有已注册事件发生的情况下，这个线程阻塞在 event_dispatcher 的 dispatch 上。你可以简单地认为阻塞在 poll 调用或者 epoll_wait 上，这种情况下，主线程如何能把已连接套接字交给 sub-reactor 子线程呢？

如果我们能**让 sub-reactor 线程从 event_dispatcher 的 dispatch 上返回，再让 sub-reactor 线程返回之后能够把新的已连接套接字事件注册上，这件事情就算完成了**

答案是构建一个类似管道一样的描述字，让 event_dispatcher 注册该管道描述字，当我们想让 sub-reactor 线程苏醒时，往管道上发送一个字符就可以了。

在 event_loop_init 函数里，**调用了 socketpair 函数创建了套接字对，往这个套接字的一端写时，另外一端就可以感知到读的事件**。其实，这里也可以直接使用 UNIX 上的 pipe 管道，作用是一样的

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

要特别注意的是这句代码，这告诉 event_loop 的，是注册了 socketPair[1]描述字上的 READ 事件，如果有 READ 事件发生，就调用 handleWakeup 函数来完成事件处理。

这个函数就是简单的从 socketPair[1]描述字上读取了一个字符而已，除此之外，它什么也没干。它的主要作用就是让子线程从 dispatch 的阻塞中苏醒。

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

如果有新的连接产生，主线程是怎么操作的?

1. 在 handle_connection_established 中，通过 accept 调用获取了已连接套接字，将其设置为非阻塞套接字（切记）
2. 接下来调用 thread_pool_get_loop 获取一个 event_loop。thread_pool_get_loop 的逻辑非常简单，从 thread_pool 线程池中按照顺序挑选出一个线程来服务
3. 创建了 tcp_connection 对象。

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

请注意，到现在为止的操作都是在主线程里执行的。下面的 event_loop_do_channel_event 也不例外

* 如果能够获取锁，主线程就会调用 event_loop_channel_buffer_nolock 往子线程的数据中增加需要处理的 channel event 对象。所有增加的 channel 对象以列表的形式维护在子线程的数据结构中。
* **如果当前增加 channel event 的不是当前 event loop 线程自己，就会调用 event_loop_wakeup 函数把 event_loop 子线程唤醒**
* 唤醒的方法很简单，就是往刚刚的 socketPair[0]上写一个字节，别忘了，event_loop 已经注册了 socketPair[1]的可读事件
* 如果当前增加 channel event 的是当前 event loop 线程自己，则直接调用 event_loop_handle_pending_channel 处理新增加的 channel event 事件列表。

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

如果是 event_loop 被唤醒之后，接下来也会执行 event_loop_handle_pending_channel 函数。你可以看到在循环体内从 dispatch 退出之后，也调用了 event_loop_handle_pending_channel 函数。

```c
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

        //这里处理pending channel，如果是子线程被唤醒，这个部分也会立即执行到
        event_loop_handle_pending_channel(eventLoop);
    }

    yolanda_msgx("event loop end, %s", eventLoop->thread_name);
    return 0;
}
```

event_loop_handle_pending_channel 函数的作用是遍历当前 event loop 里 pending 的 channel event 列表，将它们和 event_dispatcher 关联起来（调用event_dispatcher add方法），从而修改感兴趣的事件集合。

这里有一个点值得注意，因为 event loop 线程得到活动事件之后，会回调事件处理函数，这样像 onMessage 等应用程序代码也会在 event loop 线程执行，如果这里的业务逻辑过于复杂，就会导致 event_loop_handle_pending_channel 执行的时间偏后，从而影响 I/O 的检测。所以，**将 I/O 线程和业务逻辑线程隔离，让 I/O 线程只负责处理 I/O 交互，让业务线程处理业务**，是一个比较常见的做法。







































