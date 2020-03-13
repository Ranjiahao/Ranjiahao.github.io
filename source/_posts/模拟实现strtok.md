---
title: 模拟实现strtok
date: 2019-06-12 00:00:00
categories: C/C++
tags: strtok
---
> **strtok函数原型：**
> #include <string.h>
> char\* strtok(char\* str, const char\* delimiters);

# 模拟实现

```c
char* Strtok(char* str, const char* delim) {
    static char* p = NULL;
    if (str) {
        p = str;
    } else if (!p) {
        return NULL;
    }
    str = p + strspn(p, delim);
    p = str + strcspn(str, delim);
    if (p == str) {
        p = NULL;
        return NULL;
    }
    if (*p) {
        *p = '\0';
        ++p;
    } else {
        p = NULL;
    }
    return str;
}

int main() {
    char str[] = "My name is---ran jia hao---";
    char* pch;
    pch = Strtok(str, " -");
    while (pch) {
        printf("%s\n", pch);
        pch = Strtok(NULL, " -");
    }
    return 0;
}
```

# 源码

```C
char* strtok_r(char* string_org, const char* demial) {
    static unsigned char* last; // 保存分隔后剩余的部分
    unsigned char* str;         // 返回的字符串
    const unsigned char* ctrl = (const unsigned char*)demial; // 分隔字符

    // 把分隔字符放到一个索引表中，定义32是因为ASCII字符表最多是0~255个，也是说用最大的255右移3位，也就是除以8一定会是32中的一个数
    unsigned char map[32];
    for (int count = 0; count < 32; count++) {
        map[count] = 0;
    }

    // 把匹配字符放入表中
    do {
        map[*ctrl >> 3] |= (1 << (*ctrl & 7));
    } while (*ctrl++);

    // 原始字符串是否为空，如果为空表示第二次获取剩余字符的分隔部分
    if (string_org) {
        str = (unsigned char*)string_org;
    } else {
        str = last;
    }

    // 在表中查找是否有匹配的字符，如果有略过
    while ((map[*str >> 3] & (1 << (*str & 7))) && *str) {
        str++;
    }

    // 重置需要扫描的字符串
    string_org = (char*)str;

    //开始扫描
    for (; *str; str++) {
        if (map[*str >> 3] & (1 << (*str & 7))) {
            *str++ = '\0';//当找到时，把匹配字符填为0，并且把str指向下一位。
            break; //退出循环
        }
    }
    last = str; // 把剩余字符串的指针保存到静态变量last中。
    if (string_org == (char*)str) {
        return NULL; //没有找到，也就是没有移动指针的位置，返回NULL
    } else {
        return string_org; //找到了，返回之前字符串的头指针
    }
}
```
