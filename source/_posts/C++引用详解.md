---
title: C++引用详解
date: 2020-03-8 00:00:00
categories: C/C++
tags:
    - 右值引用
    - 完美转发
toc: true
---

# 引用变量

引用是已定义的变量的别名，它和它引用的变量共用同一块内存空间
其主要用途是用作函数的形参，通过这种用法使用原始数据而不是副本
- 引用在定义时必须初始化
- 一个变量可以有多个引用
- 引用一旦引用一个实体，再不能引用其他实体

## 临时变量

如果引用参数是const，则编译器将在下面两种情况下生成临时变量：

```cpp
int main() {
    const int& a = 1 + 3; // 1. 实参类型正确，但不是左值
    // int& a = 1 + 3; 错误
    const int& b = 3.14;  // 2. 实参类型不正确，但可以转换成正确类型
    // int& b = 3.14; 错误
    return 0;
}
```

这两种情况下编译器都生成一个临时匿名变量，这些临时变量出了作用域后自动消失

**为什么常量引用这种行为可以，其他情况不行？**
从编译器的角度来看，不难发现，如果不加const显然就有可能对变量进行修改，如果此时引用了临时变量，就无法对原来的变量进行修改，所以编译器默认只能用const的引用来指向临时变量


## 引用和指针

语法概念上引用就是一个别名，没有独立空间，和其引用实体共用同一块空间；
底层实现上实际是有空间的，因为引用是按照指针方式来实现的
`int a = 0; int& ra = a; ra = 1;`
`int a = 0; int* pa = &a; *a = 1;`
这两段代码的反汇编是一模一样的

# C++11右值引用

C++11中，右值分为纯右值和将亡值
* 临时变量，常量、一些运算表达式(1+3)等叫纯右值
* 声明周期将要结束的对象叫将亡值

右值引用只能指向右值，使用&&声明，定义时必须初始化
左值引用可以引用左值，也可以引用右值

```cpp
// 纯右值
int&& a = 3 + 1;
int&& b = 3.14 + 3;
cout << a << endl; // 4
cout << b << endl; // 6
```

```cpp
// 演示将亡值
String GetString(char* pStr) {
    String strTemp(pStr);
    return strTemp;
}

void test() {
    String s1(GetString("hehe"));
}
```

编译器用调用GetString，创造一个匿名对象strTemp，返回的临时对像（可能会优化成直接返回）
若String类中未写右值引用版本拷贝构造，则默认使用左值引用版本重新拷贝一份资源给s1，然后释放临时对象。这样就造成不必要的拷贝
在string类中加入以下函数即可优化上述步骤：

```cpp
String(String&& s)
    : _str(s._str)
    , _size(s._size)
    , _capacity(s._capacity) {
    s._str = nullptr;
    s._size = 0;
    s._capacity = 0;
}
```

## 默认移动构造/赋值

编译器隐式生成一个(如果没有用到则不会生成)移动构造函数，如果声明了移动构造或拷贝构造或移动赋值，则编译器不会生成默认版本。编译器生成的默认移动构造函数是浅拷贝

同理，编译器默认生成移动赋值，如果声明了移动赋值或赋值运算符重载或移动构造，则编译器不会生成默认版本

注意：在C++11中，拷贝构造/移动构造/赋值/移动赋值函数必须提供，或者同时不提供，程序才能保证类同时具有拷贝和移动语义

## move()

将一个左值强制转化为右值引用，搭配移动构造使用，可以达到类似对象数据转移的效果

如果将移动构造函数声明为常右值引用或者返回右值的函数声明为常量，都会导致移动语义无法实现 

## 完美转发

函数模板在向其他函数传递自身形参时，如果相应实参是左值，它就应该被转发为左值；如果相应实参是右值，它就应该被转发为右值。
C++11通过forward函数实现：

```cpp
void Fun(int& x) { cout << "lvalue ref" << endl; }
void Fun(int&& x) { cout << "rvalue ref" << endl; }
void Fun(const int& x) { cout << "const lvalue ref" << endl; }
void Fun(const int&& x) { cout << "const rvalue ref" << endl; }

template<typename T>
void PerfectForward(T&& t) { Fun(std::forward<T>(t)); }

int main() {
    PerfectForward(10);           // rvalue ref
    int a;
    PerfectForward(a);            // lvalue ref
    PerfectForward(std::move(a)); // rvalue ref
    const int b = 1;
    PerfectForward(b);            // const lvalue ref
    PerfectForward(std::move(b)); // const rvalue ref
    return 0;
}
```