---
title: SystemV信号量
date: 2020-05-12 00:00:00
categories: Linux
tags: ipc
toc: true
---

为了防止出现因多个进程同时访问一个共享资源引发的一系列问题，我们需要一种方法，它可以通过生成并使用令牌来授权，在任一时刻只能有一个执行线程访问代码的临界区域。信号量就可以提供这样的一种访问机制，确保多个线程在对共享内存这样的共享资源同时读写时，使之实现同步与互斥，也就是说信号量用来==调协进程对公共资源的访问的==
信号量本质上是一个计数器，也是临界资源，进程对其访问都是原子操作，信号量记录着临界资源的个数
信号量可以使用ipcs -s查看
使用ipcrm -s [semid]删除

# 原理

信号量只能进行等待和发送信号两种操作，即P(sv)和V(sv)
P(sv)：如果sv的值大于零，就给它减1；如果它的值为零，就挂起该进程并将其PCB放入队列中
V(sv)：如果等待队列中有其他进程因等待sv而被挂起，就让一个恢复运行，如果没有进程因等待sv而挂起，就给它加1
所以一个信号量不仅包含临界资源的数目，还必须维护一个队列
实际应用中信号量是不能单独定义的，而是定义一个信号量集，其中包含一组信号量，同一信号集中的信号量使用同一个引用ID，每个信号量集有一个与之对应的结构体，其中记录着各种信息

```c
struct semid_ds {
    struct ipc_perm sem_perm;
    struct sem *sem_base; // 指向sem结构数组，数组中每一个元素对应一个信号量
    unsigned short sem_nsems; // 信号量的个数
    ...
    // 时间相关描述
};
```

其中struct sem才是信号量的描述结构体：

```c
struct sem {
    unsigned short semval; // 信号量的值
    pid_t semid; // 上一次进行操作的进程号
    unsigned short semncnt; // 等待可利用资源出现的进程数
    unsigned short semzcnt; // 等待全部资源可被独占的进程数
}
```
系统中的限制：
SEMVMX最大的信号值
SEMMNI系统允许的最大信号量集个数
SEMMNS系统允许的最大信号量个数
SEMMSL每个信号集中最大的信号量个数

# 创建/打开信号集

>`int semget(key_t key, int nsems, int semflg);`
> 参数key和semflg用法和共享内存相同
> nsems: 信号量集中创建的信号量数量，如果创建信号集，sem_nsems将被设置为nsems；如果打开一个已存在的信号集，此参数就被忽略
> 如果调用成功，则返回信号量集合标识符，否则返回-1，并设置error

# 操作信号量集（原子操作）

>`int semop(int semid, struct sembuf *sops, unsigned nsops);`
> semid: semget返回的信号量集标识符
> sops: 为sembuf结构数组指针，每一个元素表示一个操作，这个函数是一个原子操作，一旦执行就将执行数组中所有的操作
> nsops: 为数组中元素的个数

```c
struct sembuf {
    unsigned short sem_num; // 表示信号量集中某一信号量序号(0~ipc_perm.sem_nsems)
    short sem_op; // 所执行的操作，如-1表示P操作消耗一个临界资源， 1表示V操作释放了一个临界资源
    short sem_flg; // 0表示没资源时阻塞等待，IPC_NOWAIT非阻塞，SEM_UNDO（将已申请的信号量还原为初识状态，即刚开始获得这个信号量时的状态）
}
```

# 控制信号量集

>`int semctl(int semid, int semnum, int cmd, ...);`
> semid: semget返回的信号集标识符
> semnum: 信号集中信号量的序号 
> cmd: 将要采取的动作
> 根据cmd参数具体内容，IPC_RMID删除信号量，可能有第四个参数，是一个联合体，当cmd为GET时返回对应值
> 成功返回0，失败返回-1，设置errno

```c
union semun {
    int val; // 用于SETVAL/GETVAL命令
    struct semid_ds *buf; // 用于IPC_STAT/IPC_SET命令，输入/输出型参数表示存放信号量集数据结构缓冲区
    unsigned short array;
    struct seminfo* _buf;
}
```

# 操作演示

子进程循环输出A和A到显示器上这2个A中间不可被打断
父进程循环输出B和B到显示器上这2个B中间不可被打断

https://github.com/Ranjiahao/Linux/tree/master/ipc/Semaphore
