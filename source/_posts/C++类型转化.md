---
title: C++类型转换
date: 2020-01-16 00:00:00
categories: C/C++
tags: 类型转换
toc: true
---

# static_cast

用于非多态类型的转换（静态转换），编译器隐式执行的任何类型转换都可用static_cast，但它
不能用于两个不相关的类型进行转换

`double d = 3.14;int a = static_cast<int>(d);  `

# reinterpret_cast

为操作数的位模式提供较低层次的重新解释，用于将一种类型转换为另一种不同的类型

```cpp
typedef void (* FUNC)();
void fuc(int i) {
    cout << "void fuc(int i);" << endl;
}
int main() {
    // reinterpret_cast可以让编译器以FUNC的定义方式去看待函数fuc
    FUNC f = reinterpret_cast<FUNC>(fuc);
    f();
    return 0;
}
```

# const_cast

删除变量的const属性 

```cpp
const int a = 1;
int* p = const_cast<int*>(&a);
*p = 3;
cout << a << endl;  // 1
cout << *p << endl; // 3
```

# dynamic_cast

只能用于含有虚函数的类，dynamic_cast会先检查是否能转换成功，能成功则转换，不能则返回0

```cpp
class A {
public:
    virtual void f() {}
};
 
class B: public A {};
 
void fuc(A* pa) {
    // dynamic_cast会先检查是否能转换成功，能成功则转换，不能则返回
    B* pb1 = static_cast<B*>(pa);
    B* pb2 = dynamic_cast<B*>(pa);
    cout << "pb1:" << pb1 << endl;
    cout << "pb2:" << pb2 << endl;
}
 
int main() {
    A a;
    B b;
    fuc(&a); // dynamic_cast 出错返回0
    fuc(&b);
    return 0;
}
```
