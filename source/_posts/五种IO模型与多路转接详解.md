---
title: 五种IO模型与及多路复用详解
date: 2020-07-16 00:00:00
categories: Linux
tags:
    - 多路转接
toc: true
---

# 五种IO模型

* 阻塞IO：系统调用会一直等待，直到内核数据准备就绪，然后进行内核和用户空间的数据交换。所有的套接字，默认都是阻塞方式，流程简单，但是任务处理效率低，无法充分利用资源

* 非阻塞IO：若内核还未将数据准备好，系统调用返回EWOULDBLOCK错误码，往往需要循环的方式反复尝试读写文件描述符，对资源的利用充分，但IO不够实时，且对CPU来说是较大的浪费
可以通过`int fcntl(int fd, int cmd, ... /* arg */ );`函数设置`int fl = fcntl(fd, F_GETFL);fcntl(fd, F_SETFL, fl | O_NONBLOCK)`

* 信号驱动IO：建立SIGIO的信号处理程序，内核将数据准备好的时候，使用SIGIO信号通知应用程序进行IO操作，资源利用充分，比非阻塞IO实时

* 多路转接IO：发起多路转接IO，将需要等待的文件描述符添加监控，然后由内核轮询遍历，进程一直等待任意一个文件描述符就绪，然后进行IO操作

* 异步IO：发起异步IO系统调用，直接返回，然后由内核等待，并由内核自动拷贝数据，拷贝完成后通知应用程序，对资源的利用最为充分，且效率最高

* 这五种IO模型处理的效率逐渐增加，对资源(cpu)的利用也更加充分，但是流程也越来越复杂

# 多路转接

## select

系统调用`int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);`
nfds：需要监视的最大的文件描述符值+1
readfds,writefds,exceptfds：分别对应需要检测的可读、可写、异常文件描述符集合
timeout：用来设置select()的等待时间。NULL表示select将一直被阻塞，直到某个文件描述符上发生了事件，0：仅检测描述符集合的状态，然后立即返回，并不等待外部事件的发生，特定的时间值：如果在指定的时间段里没有事件发生，select将超时返回
执行成功则返回文件描述词状态已改变的个数，如果返回0代表在描述词状态改变前已超过timeout时间，没有返回，当有错误发生时则返回-1，错误原因存于errno，此时参数readfds，writefds, exceptfds和timeout的值变成不可预测

**fd_set结构**

```c
#define __FD_SETSIZE        1024
// select.h
typedef long int __fd_mask;
#define __NFDBITS   (8 * (int) sizeof (__fd_mask))
typedef struct
  {
    /* XPG4.2 requires this member name.  Otherwise avoid the name
       from the global namespace.  */
#ifdef __USE_XOPEN
    __fd_mask fds_bits[__FD_SETSIZE / __NFDBITS];
# define __FDS_BITS(set) ((set)->fds_bits)
#else
    __fd_mask __fds_bits[__FD_SETSIZE / __NFDBITS];
# define __FDS_BITS(set) ((set)->__fds_bits)
#endif
  } fd_set;
```

这个结构就是一个整数数组构成的位图，位图中对应的位来表示要监视的文件描述符，提供了一组操作fd_set的接口来方便的操作位图

```c
void FD_CLR(int fd, fd_set *set);  // 用来清除描述词组set中相关fd 的位
int FD_ISSET(int fd, fd_set *set); // 用来测试描述词组set中相关fd 的位是否为真
void FD_SET(int fd, fd_set *set);  // 用来设置描述词组set中相关fd的位
void FD_ZERO(fd_set *set);         // 用来清除描述词组set的全部位
```

**timeval结构**

```c
// time.h
struct timeval    
  {    
    __time_t tv_sec;        /* Seconds.  */    
    __suseconds_t tv_usec;  /* Microseconds.  */                     
  };
```

**原理**

定义指定监控事件的描述符集合（即位图），初始化集合后，将需要监控指定事件的描述符添加到指定事件（可读、可写、异常）的描述符集合中
select将描述符集合拷贝到内核当中，对集合中所有描述符进行轮询判断，当描述符就绪或者等待超时后就调用返回，返回后的集合中只剩下已就绪的描述符（未就绪会在位图中置为0）
用户通过遍历描述符，判断哪些描述符还在集合中，就可以知道哪些描述符已经就绪了，开始处理对应的IO事件

读就绪：socket内核中接收缓冲区中的字节数大于等于低水位标记SO_RCVLOWAT（默认1字节），此时可以无阻塞的读该文件描述符，并且返回值大于0；socket TCP通信中，对端关闭连接，此时对该socket读，则返回0；监听的socket上有新的连接请求；socket上有未处理的错误
写就绪：socket内核中发送缓冲区中的字节数（发送缓冲区的空闲位置大小）大于等于低水位标记SO_SNDLOWAT，此时可以无阻塞的写，并且返回值大于0；socket的写操作被关闭（close或者shutdown）对一个写操作被关闭的socket进行写操作，会触发SIGPIPE信号；socket使用非阻塞connect连接成功或失败之后；socket上有未读取的错误;
异常就绪：socket上收到带外数据

**缺点**

1. select能监控的最大描述符有限，由宏\_\_FD\_SETSIZE决定，默认是1024个
2. select会将集合拷贝到内核中轮询遍历判断描述符是否就绪，效率会随着描述符的增多而越来越低
3. select监控完毕后返回的集合中只有已就绪的描述符，移除了未就绪的描述符，所以每次监控都必须要重新将描述符加入集合中，重新拷贝到内核
4. select返回的集合是一个位图而不是真正的描述符数组，所以需要用户遍历判断哪个描述符在集合中才能确认其是否就绪

**优点**

1. select遵循posix标准，可以跨平台移植
2. select的超时等待时间较为精确，可以精细到微秒

tcp_select服务器：https://github.com/Ranjiahao/Linux/blob/master/socket/tcp/tcp_select_srv.cc

## poll

系统调用接口`int poll(struct pollfd *fds, nfds_t nfds, int timeout);`
fds：poll函数监听的结构列表，包含了三部分内容，文件描述符、监听的事件集合、返回的事件集合
nfds：表示fds数组的长度
timeout：表示poll函数的超时时间，单位是毫秒(ms)
返回值小于0，表示出错；返回值等于0，表示poll函数等待超时；返回值大于0，表示poll由于监听的文件描述符就绪而返回

**pollfd结构**

```c
// pollfd结构
struct pollfd {
    int fd;        /* file descriptor */
    short events;  /* requested events */
    short revents; /* returned events */
};
```

events和revents取值：POLLIN可读、POLLOUT可写

**原理**

* 定义pollfd结构体数组，将需要监控的描述符以及监控的事件信息添加进去

* 发起监控调用poll，将数组中的数据拷贝到内核当中进行轮询遍历监控，当有描述符就绪或者等待超时后返回，返回时将已就绪的事件添加进pollfd结构体中的revents中（如果没就绪，则为0）

* 监控调用返回后，遍历pollfd数组中的每一个节点的revents，根据对应的就绪时间进行相应操作

**缺点**

1. 每次调用poll都需要把大量的pollfd结构从用户态拷贝到内核中，在内核中轮询判断描述符是否就绪，效率会随着描述符的增加而下降

2. 每次调用返回后需要用户自行判断revents才能知道是哪个描述符就绪了哪个事件
3. 无法跨平台移植
4. 超时等待时间只精确到毫秒

**优点**

1. poll通过描述符事件结构体的方式将select的描述符集合的操作流程合并在一起，简化了操作
2. poll并没有最大数量限制（但是数量过大后性能也是会下降）
3. poll每次监控不需要重新定义事件结构

## epoll

系统调用
`int epoll_create(int size);`创建一个epoll的句柄
自从linux2.6.8之后，size参数是被忽略的，用完之后，必须调用close()关闭

`int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);`注册要监听的事件类型
epfd：epoll_create返回的句柄
op：EPOLL_CTL_ADD，注册新的fd到epfd中；EPOLL_CTL_MOD，修改已经注册的fd的监听事件；EPOLL_CTL_DEL，从epfd中删除一个fd；
fd：需要监听的fd
event：告诉内核需要监听什么事

**epoll_event结构**

```c
typedef union epoll_data
{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;

struct epoll_event
{
  uint32_t events;  /* Epoll events */
  epoll_data_t data;    /* User data variable */
} __EPOLL_PACKED;
```

events取值：EPOLLIN表示对应的文件描述符可以读 (包括对端SOCKET正常关闭)；EPOLLOUT表示对应的文件描述符可以写；EPOLLET将EPOLL设为边缘触发，一般将data.fd设置为需要监听的fd

`int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);`收集在epoll监控的事件中已经发送的事件（异步阻塞）
events：分配好的epoll_event结构体数组，epoll将会把发生的事件赋值到events数组中（events不可以是空指针，内核只负责把数据复制到这个events数组中，不会去帮助我们在用户态中分配内存）
maxevents：告诉内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size参数
timeout：超时时间（毫秒，0会立即返回，-1是永久阻塞)
如果函数调用成功，返回对应I/O上已准备好的文件描述符数目，如返回0表示已超时，返回小于0表示函数失败

**原理**

```c
struct eventpoll {
    ....
    /*红黑树的根节点，这颗树中存储着所有添加到epoll中的需要监控的事件*/
    struct rb_root rbr;
    /*双链表中则存放着将要通过epoll_wait返回给用户的满足条件的事件*/
    struct list_head rdlist;
    ....
};

struct epitem {
    struct rb_node rbn;//红黑树节点
    struct list_head rdllink;//双向链表节点
    struct epoll_filefd ffd; //事件句柄信息
    struct eventpoll *ep; //指向其所属的eventpoll对象
    struct epoll_event event; //期待发生的事件类型
}
```

每一个epoll对象都有一个独立的eventpoll结构体，用于存放通过epoll_ctl方法向epoll对象中添加进来的事件，这些事件都会挂载在红黑树中，如此，重复添加的事件就可以通过红黑树而高效的识别出来(红黑树的插入时间效率是log(n)，其中n为树的高度)而所有添加到epoll中的事件都会与设备(网卡)驱动程序建立回调关系，也就是说，当响应的事件发生时会调用这个回调方法，这个回调方法在内核中叫ep_poll_callback，它会将发生的事件添加到rdlist双链表中，在epoll中，对于每一个事件，都会建立一个epitem结构体，当调用epoll_wait检查是否有事件发生时，只需要检查eventpoll对象中的rdlist双链表中是否有epitem元素即可，如果rdlist不为空，则把发生的事件复制到用户态，同时将事件数量返回给用户，这个操作的时间复杂度是O(1)

**优点**

1. 文件描述符数目无上限  
2. 数据拷贝轻量，只在调用EPOLL_CTL_ADD将文件描述符结构拷贝到内核中
3. 事件回调机制，避免使用遍历，而是使用回调函数的方式，将就绪的文件描述符结构加入到就绪队列中，epoll_wait返回直接访问就绪队列就知道哪些文件描述符就绪，这个操作时间复杂度O(1)即使文件描述
   符数目很多, 效率也不会受到影响

**缺点**

1. 无法跨平台移植
2. 超时等待时间只精确到毫秒
3. 在活动连接较多的时候，由于会大量触发回调函数，所以此时epoll的效率未必会select和poll高，所以epoll适用于连接数量多，但是活动连接少的情况

应用场景：对于多连接，且多连接中只有一部分连接比较活跃时，比较适合使用epoll  

### LT模式与ET模式

epoll默认状态下就是LT工作模式，只要接收缓冲区中数据大于低水位标记（可读），或者发送缓冲区中剩余空间大于低水位标记（可写）就会一直触发事件，支持阻塞读写和非阻塞读写，LT模式简单稳定
tcp_epoll_lt服务器：https://github.com/Ranjiahao/Linux/blob/master/socket/tcp/tcp_epoll_lt_srv.cc

ET模式，只有新数据到来是触发可读时间，或者剩余空间大小从无到有的时候才会触发事件，只支持非阻塞的读写
tcp_epoll_et服务器：https://github.com/Ranjiahao/Linux/blob/master/socket/tcp/tcp_epoll_et_srv.cc
使用ET能够减少epoll触发的次数，但是代价就是必须一次把所有的数据都处理完，代码复杂，还有一种场景适合ET模式使用，如果我们需要接受一条数据，但是这条数据因为某种问题导致其发送不完整，需要分批发送。所以此时的缓冲区中数据只有部分，如果此时将其取出，则会增加维护数据的开销，正确的做法应该是等待后续数据到达后将其补全，再一次性取出。但是如果此时使用的是LT模式，就会因为缓冲区不为空而一直触发事件，所以这种情况下使用ET会比较好