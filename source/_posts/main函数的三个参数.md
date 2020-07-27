---
title: main函数的三个参数
date: 2020-04-17 00:00:00
categories: Linux
tags: 环境变量
toc: true
---

C程序总是从main函数开始执行的，main函数的原型是：
`int main(int argc, char* argv[], char* env[])`

# argc与argv

argc表示传入main函数的参数个数
argv[]表示传入main函数的参数序列或指针，并且第一个参数argv[0]一定是程序的名称，并且包含了程序所在的完整路径，所以确切的说需要我们输入的main函数的参数个数应该是argc-1个

```c
// main.c
int main(int argc,char *argv[]) {
    for (int i=0; i < argc; i++) {
        printf("argv[%d]=[%s]\n",i,argv[i]);
    }
}
```

编译这段程序`gcc main.c -o main`生成可执行程序main
`./main`运行：输出结果为argv[0]=[./main]
`./main -a abc`运行：输出结果为
argv[0]=[./main]
argv[1]=[-a]
argv[2]=[abc]

可以总结出以下几点：
argv[0] 指向程序运行的全路径名 　　
argv[1] 指向在DOS命令行中执行程序名后的第一个字符串 　　
argv[2] 指向执行程序名后的第二个字符串
...
argv[argc]为NULL

# env

每个进程都会收到一张环境表，环境表是一个字符指针数组，每个指针指向一个以NULL结尾的环境字符串，env[]的每一个元素都包含ENVVAR=value形式的字符串，其中ENVVAR为环境变量，value 为ENVVAR的对应值
环境变量通常具有全局特性，系统设置更加方便(切换用户会还原)

## 常见的环境变量

PATH: 命令的搜索路径
HOME: 用户家目录
SHELL: 当前Shell通常是/bin/bash

## 环境变量相关命令

`echo $(NAME)`可以查看指定环境变量
`export (val)`声明一个新的环境变量
`env`显示所有环境变量
`unset (NAME)`清除环境变量
`set`显示本地定义的shell变量和环境变量

## 通过代码查看环境变量

由于环境变量具有继承特性，我们可以创建进程来继承shell的环境变量并且打印出来
* 通过第三个参数获得

```c
int main(int argc, char *argv[], char *env[]) {
    for(int i = 0; env[i] != NULL; ++i) {
        printf("%s\n", env[i]);
    }
    return 0;
}
```
* 通过libc中定义的全局变量environ获取
```c
int main(int argc, char *argv[]) {
    extern char **environ;
    for(int i = 0; environ[i]; ++i) {
        printf("%s\n", environ[i]);
    }
    return 0;
}
```
* 通过系统调用接口获取或设置环境变量

getenv 和putenv 也是定义在stdlib.h 中，函数原型如下：

`char *getenv(const char *name);`
`int putenv(char *str);`

## ./main执行和main执行

有些指令可以直接执行，不需要带路径，而我们写的可执行程序必须要加上路径，这是由于默认的搜索路径为PATH，我们只需将PATH修改即可：
`PATH=$PATH:./`
但是需要注意的是不要覆盖PATH防止类似ls的命令用不了

## 是否应该使用第三个参数env

ISO C/ISO C++ ,POSIX 标准都不支持main三个参数的定义形式，VC和GNU编译器都扩展了main函数的定义，所以目前可以这样使用。如果要编写更加可移植的程序，应该使用全局环境变量environ来代替env的作用，如果要访问特定的环境变量，应该使用getenv和putenv函数
