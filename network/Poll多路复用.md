select 方法是多个 UNIX 平台支持的非常常见的 I/O 多路复用技术，它通过描述符集合来表示检测的 I/O 对象，通过三个不同的描述符集合来描述 I/O 事件 ：可读、可写和异常。但是 select 有一个缺点，那就是所支持的文件描述符的个数是有限的。在 Linux 系统中，select 的默认最大值为 1024。



### poll 函数介绍

```c
int poll(struct pollfd *fds, unsigned long nfds, int timeout); 
　　　
返回值：若有就绪描述符则为其数目，若超时则为0，若出错则为-1
```

```c
struct pollfd {
    int    fd;       /* file descriptor */
    short  events;   /* events to look for */
    short  revents;  /* events returned */
 };
```

首先是描述符 fd，然后是描述符上待检测的事件类型 events，注意这里的 events 可以表示多个不同的事件，具体的实现可以通过使用二进制掩码位操作来完成，例如，POLLIN 和 POLLOUT 可以表示读和写事件。



和 select 非常不同的地方在于，poll 每次检测之后的结果不会修改原来的传入值，而是将结果保留在 revents 字段中，这样就不需要每次检测完都得重置待检测的描述字和感兴趣的事件。我们可以把 revents 理解成“returned events”。



poll 函数的原型。**参数 nfds** 描述的是数组 fds 的大小，简单说，就是**向 poll 申请的事件检测的个数。**



最后一个参数 timeout，描述了 poll 的行为。如果是一个 <0 的数，表示在有事件发生之前永远等待；如果是 0，表示不阻塞进程，立即返回；如果是一个 >0 的数，表示 poll 调用方等待指定的毫秒数后返回。



#### 基于 poll 的服务器程序 ####

这个程序可以同时处理多个客户端连接，并且一旦有客户端数据接收后，同步地回显回去

```c
#define INIT_SIZE 128

int main(int argc, char **argv) {
    int listen_fd, connected_fd;
    int ready_number;
    ssize_t n;
    char buf[MAXLINE];
    struct sockaddr_in client_addr;

    listen_fd = tcp_server_listen(SERV_PORT);

    //初始化pollfd数组，这个数组的第一个元素是listen_fd，其余的用来记录将要连接的connect_fd
    struct pollfd event_set[INIT_SIZE];
    event_set[0].fd = listen_fd;
    event_set[0].events = POLLRDNORM;

    // 用-1表示这个数组位置还没有被占用
    int i;
    for (i = 1; i < INIT_SIZE; i++) {
        event_set[i].fd = -1;
    }

    for (;;) {
        if ((ready_number = poll(event_set, INIT_SIZE, -1)) < 0) {
            error(1, errno, "poll failed ");
        }

        if (event_set[0].revents & POLLRDNORM) {
            socklen_t client_len = sizeof(client_addr);
            connected_fd = accept(listen_fd, (struct sockaddr *) &client_addr, &client_len);

            //找到一个可以记录该连接套接字的位置
            for (i = 1; i < INIT_SIZE; i++) {
                if (event_set[i].fd < 0) {
                    event_set[i].fd = connected_fd;
                    event_set[i].events = POLLRDNORM;
                    break;
                }
            }

            if (i == INIT_SIZE) {
                error(1, errno, "can not hold so many clients");
            }

            if (--ready_number <= 0)
                continue;
        }

        for (i = 1; i < INIT_SIZE; i++) {
            int socket_fd;
            if ((socket_fd = event_set[i].fd) < 0)
                continue;
            if (event_set[i].revents & (POLLRDNORM | POLLERR)) {
                if ((n = read(socket_fd, buf, MAXLINE)) > 0) {
                    if (write(socket_fd, buf, n) < 0) {
                        error(1, errno, "write error");
                    }
                } else if (n == 0 || errno == ECONNRESET) {
                    close(socket_fd);
                    event_set[i].fd = -1;
                } else {
                    error(1, errno, "read error");
                }

                if (--ready_number <= 0)
                    break;
            }
        }
    }
}
```



如果对应 pollfd 里的文件描述字 fd 为负数，poll 函数将会忽略这个 pollfd，所以将 event_set 数组里其他没有用到的 fd 统统设置为 -1。这里 -1 也表示了当前 pollfd 没有被使用的意思。

如果系统内核检测到监听套接字上的连接建立事件，使用了如 event_set[0].revent 来和对应的事件类型进行位与操作

调用 accept 函数获取了连接描述字。接下来，就是把连接描述字 connect_fd 也加入到 event_set 里，而且说明了我们感兴趣的事件类型为 POLLRDNORM，也就是套接字上有数据可以读



启动这个服务器程序，然后通过 telnet 连接到这个服务器程序。为了检验这个服务器程序的 I/O 复用能力，我们可以多开几个 telnet 客户端，并且在屏幕上输入各种字符串。

```shell
#1
$telnet 127.0.0.1 43211
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
a
a
aaaaaaaaaaa
aaaaaaaaaaa
afafasfa
afafasfa
fbaa
fbaa
^]


telnet> quit
Connection closed.


#2
telnet 127.0.0.1 43211
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
b
b
bbbbbbb
bbbbbbb
bbbbbbb
bbbbbbb
^]


telnet> quit
Connection closed.
```













