---
title: SystemV共享内存
date: 2020-05-07 00:00:00
categories: Linux
tags: ipc
toc: true
---

共享内存就是将同一块物理内存映射到不同进程虚拟内存的共享区域，实现两个进程间通信。一旦这样的内存映射到进程地址空间的共享区，这些进程间数据传递不再涉及到内核，不再通过执行进入内核的系统调用来传递彼此的数据，所以说共享内存是最快的IPC形式
我们可以通过命令查看/删除系统中已有的共享内存
`ipcs -m`/`ipcrm -m [shmid]`
相关操作（系统调用）：

# 创建共享内存

`int shmget(key_t key, size_t size, int shmflg);`
key:进程对共享内存的访问都是间接的，需要提供一个参数key（非0整数，可为ftok() 的返回值，也可以用自己的宏）来告诉操作系统需要哪个共享内存，然后系统返回对应共享内存的标识符，失败返回-1，并设置errno
size:共享内存大小，PAGE_SIZE页面大小(4096)的整数倍，因为这些内存块是以页面为单位进行分配的，实际分配的内存块大小将被扩大到页面大小的整数倍
shmflg:由九个权限标志构成，用法和创建文件时使用的mode模式标志是一样的
IPC_CREAT：如果不存在就创建
IPC_EXCL： 如果已经存在则返回失败
位或权限位：共享内存位或权限位后可以设置共享内存的访问权限，格式和 open() 函数的 mode_t 一样，但可执行权限未使用

**共享内存数据结构**

```c
struct shmid_ds {
    struct ipc_perm shm_perm; /* operation perms */
    int shm_segsz; /* size of segment (bytes) */
    __kernel_time_t shm_atime; /* last attach time */
    __kernel_time_t shm_dtime; /* last detach time */
    __kernel_time_t shm_ctime; /* last change time */
    __kernel_ipc_pid_t shm_cpid; /* pid of creator */
    __kernel_ipc_pid_t shm_lpid; /* pid of last operator */
    unsigned short shm_nattch; /* no. of current attaches */
    unsigned short shm_unused; /* compatibility */
    void *shm_unused2; /* ditto - used by DIPC */
    void *shm_unused3; /* unused */
};
```

# 建立映射关系

`void *shmat(int shmid, const void *shmaddr, int shmflg);`
shmid: 共享内存标识（shmget返回值）
shmaddr: 映射地址，通常为空，让系统来选择共享内存的地址；若不为NULL且shmflg无SHM_RND标记，若设置SHM_RND标记，则连接的地址会自动向下调整为SHMLBA的整数倍
shmflg:
    SHM_RDONLY:只读
    0:共享内存具有可读可写权限
返回值：成功返回一个指针，指向共享内存第一个节；失败返回-1，并设置errno

# 解除映射

`int shmdt(const void *shmaddr);`
shmaddr: 由shmat所返回的指针
成功返回0；失败返回-1，并设置errno

# 控制共享内存

`int shmctl(int shmid, int cmd, struct shmid_ds *buf);`

shmid:由shmget返回的共享内存标识码
cmd:IPC_RMID删除共享内存、IPC_STAT获取共享内存的当前关联值、IPC_SET根据buf设置共享内存关联值。注意：共享内存删除不会直接删除，而是判断映射连接数是否为0，为0直接删除否则拒绝后续其他进程的映射连接，为0时自动删除
buf:保存着共享内存的模式状态和访问权限的数据结构，输入输出型参数
成功返回0；失败返回-1

共享内存并未提供同步机制，一个进程结束对共享内存的写操作之前，并无自动机制可以阻止第二个进程开始对它进行读取。所以我们通常需要用其他的机制来同步对共享内存的访问，比如信号量

# 关于函数ftok

shmget函数中的第一个参数key相当于共享内存的钥匙，可以通过它能访问唯一的内存区域。由于key和内存是一一对应的，所以我们不难想到用这块内存的inod节点来生成密钥，这就有了ftok函数
`系统调用key_t ftok(const char *pathname, int proj_id);`
pathname就时你指定的文件名(该文件必须是存在而且可以访问的)，proj_id是子序号，虽然为int，但是只有8个比特被使用(0-255)
成功返回key值，失败返回-1

生成规则：返回的整数IPC键由proj_id的低序8位，struct stat中st_dev成员的低序8位，st_ino的低序16位组合而成

注意事项：
pathname指定的目录（文件）不能在程序运行期间删除或创建。因为文件每次创建时由系统赋予的索引节点可能不一样。这样一来，通过同一个pathname与proj_id就不能保证生成同一个IPC键

# 代码实现server和client的交互

https://github.com/Ranjiahao/Linux/tree/master/ipc/SharedMemory
