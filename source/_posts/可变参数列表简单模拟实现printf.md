---
title: 可变参数列表简单模拟实现printf
date: 2019-06-03 00:00:00
categories: C/C++
tags: 可变参数列表
toc: true
---
# 声明格式

> type VarArgFunc(type FixedArg1, type FixedArg2, …);
参数分为固定参数和可选参数两部分。固定参数至少一个，可选参数数目不定。

# stdarg宏

可变参数列表是通过宏来实现的，定义于stdarg.h头文件中，是标准库的一部分。这个头文件声明了一个类型va_list和三个宏va_start、va_arg和va_end，我们可以定义一个类型为va_list的变量，与这几个宏配合使用来访问参数的值
```c
typedef char * va_list;
#define _INTSIZEOF(n)       ( (sizeof(n)+sizeof(int)-1) & ~(sizeof(int)-1) )
#define va_start(ap,v)      ( ap = (va_list)&v + _INTSIZEOF(v) )
#define va_arg(ap, type)    ( *(type *)((ap += _INTSIZEOF(type)) - _INTSIZEOF(type)) )
#define va_end(ap)          ( ap = (va_list)0 )
```
* \_INTSIZEOF宏考虑到某些系统需要内存地址对齐，由于参数列表中的可变参数部分没有原型，所以可变参数传递给函数的值都将执行缺省参数类型提升。从宏名看应按照sizeof(int)即堆栈粒度对齐，即参数在内存中的地址均为sizeof(int)=4的倍数。例如，若在1≤sizeof(n)≤4，则\_INTSIZEOF(n)＝4；若5≤sizeof(n)≤8，则\_INTSIZEOF(n)=8
* va_start宏根据(va_list)&v得到第一个可变参数前的一个固定参数在堆栈中的内存地址，加上\_INTSIZEOF(v)即v所占内存大小后，使ap指向固定参数后下个参数
==固定参数的地址用于va_start宏，因此不能声明为寄存器变量（地址无效）或作为数组类型（长度难定）==
* va_arg宏取得type类型的可变参数值。首先ap+=\_INTSIZEOF(type)即ap跳过当前可变参数而指向下个变参的地址；然后ap-\_INTSIZEOF(type)得到当前变参的内存地址，类型转换后返回当前变参值
* va_end宏使ap不再指向有效的内存地址。该宏的某些实现定义为((void\*)0)编译时不会为其产生代码，调用与否并无区别。但某些实现中va_end宏用于函数返回前完成一些必要的清理工作：如va_start宏可能以某种方式修改堆栈，导致返回操作无法完成，va_end宏可将有关修改复原；又如va_start宏可能对参数列表动态分配内存以便于遍历va_list，va_end宏可释放此前动态分配的内存。因此，从使用va_start宏的函数中退出之前，必须调用一次va_end宏

函数内可多次遍历可变参数，但每次必须以va_start宏开始，因为遍历后ap指针不再指向首个变参

# 实现简易的myPrintf

```c
// 该函数无返回值，即不记录输出的字符数目；接受"%c"按字符输出、"%d"按整数输出、"%s"按字符串输出
void printch(char c) {
    putchar(c);
}

void printdec(int i) {
    if (i) {
        printdec(i / 10);
        putchar(i % 10 + '0');
    }
}

void printstr(char* str) {
    while (*str) {
        putchar(*str);
        ++str;
    }
}

void myPrintf(char* str, ...) {
    va_list arg;
    va_start(arg, str);
    while (*str) {
        if (*str == '%') {
            ++str;
            switch (*str) {
                case 'c': {
                              char c = va_arg(arg, char);
                              printch(c);
                              break;
                          }
                case 'd': {
                              int i = va_arg(arg, int);
                              printdec(i);
                              break;
                          }
                case 's': {
                              char* s = va_arg(arg, char*);
                              printstr(s);
                              break;
                          }
                default:
                          break;
            }
            ++str;
        } else {
            putchar(*str);
            ++str;
        }
    }
    va_end(arg);
}
```
