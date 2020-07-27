---
title: 进程终止与wait参数status
date: 2020-05-02 00:00:00
categories: Linux
tags: 进程
toc: true
---

# 常见进程退出方法

1. exit或\_exit退出
\_exit系统调用函数：`void _exit(int status);`
参数status定义了进程的终止状态，父进程通过wait获取该值。
虽然status是int，但是仅有低8位可以被父进程所用，实际上就是子进程的返回值。
exit库函数：`void exit(int status);`
和\_exit唯一的区别是退出前执行用户通过atexit或on_exit定义的清理函数然后刷新缓存关闭所有打开的流，最后调用\_exit函数退出
2. return退出，和exit一样，exec/exit针对进程间通信，就像call/return针对函数间通信
3. 信号终止

# 常见进程等待方法

1. wait系统调用函数（阻塞）`pid_t wait(int* status);`
status为输出型参数，NULL表示不关心子进程状态
返回值：成功返回被等待进程的pid，失败返回-1
2. waitpid系统调用函数
`pid_ t waitpid(pid_t pid, int* status, int options);`
参数pid若为-1则等待任意一个进程，若>0则等待特定的进程
参数options若为0则阻塞，若为WNOHANG则非阻塞

# wait参数statu

status不仅仅是一个int类型参数，可以看作是一个位图
只研究其低16位：
![img1](img1.png)
我们可以通过status & 0x7f来判断异常信号是否为0；
若为0，则正常退出，然后可以通过(status >> 8) & 0xff来获取子进程返回值。
sys/wait.h中提供了一些宏来简化这些操作：

```c
if (WIFEXITED(status)) {
    // 正常退出：((status) & 0x7f) == 0
    // 打印退出码：(status >> 8) & 0xff
    printf("child return: %d\n", WEXITSTATUS(status));
} else if (WIFSIGNALED(status)) {
    // 异常退出：((signed char) (((status) & 0x7f) + 1) >> 1) > 0
    // 打印异常信号值：(status) & 0x7f
    printf("child signal: %d\n", WTERMSIG(status));
}
```
