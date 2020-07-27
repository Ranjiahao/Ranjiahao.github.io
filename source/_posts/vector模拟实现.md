---
title: vector模拟实现
date: 2020-03-18 00:00:00
categories: C/C++
tags: stl
---

```cpp
#include <initializer_list>
#include <assert.h>

template <typename T>
class Vector {
public:
    typedef T* iterator;
    typedef const T* const_iterator;

    Vector();
    Vector(size_t n, const T& value = T());
    Vector(const Vector<T>& v);
    Vector(Vector<T>&& v);
    Vector(const std::initializer_list<T>& lst);
    ~Vector();
    Vector<T>& operator=(Vector<T> v);

    iterator begin();
    const_iterator begin() const;
    iterator end();
    const_iterator end() const;

    size_t size() const;
    void resize(size_t n, const T& val = T());
    size_t capacity() const;
    bool empty() const;
    void reserve(size_t n);

    T& operator[](size_t pos);
    const T& operator[](size_t pos) const;

    void push_back(const T& val);
    void pop_back();
    iterator insert(iterator pos, const T& val);
    iterator erase(iterator pos);
    void swap(Vector<T>& v);
    void clear();
private:
    T* _start = nullptr;
    T* _finish = nullptr;
    T* _endOfStorage = nullptr;
};

template <typename T>
Vector<T>::Vector()
    : _start(nullptr)
    , _finish(nullptr)
    , _endOfStorage(nullptr) {}

template <typename T>
Vector<T>::Vector(size_t n, const T& value)
    : Vector() {
    reserve(n);
    while (n--) {
        push_back(value);
    }
}

template <typename T>
Vector<T>::Vector(const Vector<T>& v)
    : _start(new T[v.capacity()])
    , _finish(_start + v.size())
    , _endOfStorage(_start + v.capacity()) {
    for (size_t i = 0; i < v.size(); i++) {
        _start[i] = v[i];
    }
}

template<typename T>
Vector<T>::Vector(Vector<T>&& v)
    : Vector() {
    swap(v);
}

template<typename T>
Vector<T>::Vector(const std::initializer_list<T>& lst)
    : _start(new T[lst.size()])
    , _finish(_start)
    , _endOfStorage(_start + lst.size()) {
    for (const auto& e : lst) {
        *_finish = e;
        ++_finish;
    }
}

template <typename T>
Vector<T>::~Vector() {
    if (_start) {
        delete[] _start;
        _start = _finish = _endOfStorage = nullptr;
    }
}

template <typename T>
Vector<T>& Vector<T>::operator=(Vector<T> v) {
    swap(v);
    return *this;
}

template <typename T>
typename Vector<T>::iterator Vector<T>::begin() {
    return _start;
}

template <typename T>
typename Vector<T>::const_iterator Vector<T>::begin() const {
    return _start;
}

template <typename T>
typename Vector<T>::iterator Vector<T>::end() {
    return _finish;
}

template <typename T>
typename Vector<T>::const_iterator Vector<T>::end() const {
    return _finish;
}

template <typename T>
size_t Vector<T>::size() const {
    return _finish - _start;
}

template <typename T>
void Vector<T>::resize(size_t n, const T& val) {
    if (n <= size()) {
        _finish = _start + n;
    } else {
        if (n > capacity()) {
            reserve(n);
        }
        while (_finish < _start + n) {
            *_finish++ = val;
        }
    }
}

template <typename T>
size_t Vector<T>::capacity() const {
    return _endOfStorage - _start;
}

template<typename T>
bool Vector<T>::empty() const {
    return size() == 0;
}

template <typename T>
void Vector<T>::reserve(size_t n) {
    size_t size = this->size();
    if (n > capacity()) {
        T* tmp = new T[n];
        if (_start) {
            for (size_t i = 0; i < this->size(); i++) {
                tmp[i] = _start[i];
            }
        }
        _start = tmp;
        _finish = _start + size;
        _endOfStorage = _start + n;
    }
}

template <typename T>
T& Vector<T>::operator[](size_t pos) {
    assert(pos < size());
    return _start[pos];
}

template <typename T>
const T& Vector<T>::operator[](size_t pos) const {
    assert(pos < size());
    return _start[pos];
}

template <typename T>
void Vector<T>::push_back(const T& val) {
    insert(end(), val);
}

template <typename T>
void Vector<T>::pop_back() {
    assert(!empty());
    erase(end() - 1);
}

template <typename T>
typename Vector<T>::iterator Vector<T>::insert(iterator pos, const T& val) {
    assert(pos >= begin() && pos <= end());
    size_t len = pos - _start;
    if (_finish == _endOfStorage) {
        // 实际会根据一定算法开辟空间
        size_t newC = capacity() == 0 ? 1 : 2 * capacity();
        reserve(newC);
    }
    // 更新迭代器，增容导致迭代器失效
    pos = _start + len;
    iterator end = _finish;
    while (end > pos) {
        *end = *(end - 1);
        --end;
    }
    *pos = val;
    ++_finish;
    return pos;
}

template <typename T>
typename Vector<T>::iterator Vector<T>::erase(iterator pos) {
    assert(pos < _finish && pos >= _start);
    iterator begin = pos + 1;
    while (begin < _finish) {
        *(begin - 1) = *begin;
        ++begin;
    }
    --_finish;
    // 删除导致迭代器失效，返回下一个元素的位置
    return pos;
}

template <typename T>
void Vector<T>::swap(Vector<T>& v) {
    std::swap(_start, v._start);
    std::swap(_finish, v._finish);
    std::swap(_endOfStorage, v._endOfStorage);
}

template<typename T>
void Vector<T>::clear() {
    _finish = _start;
}
```

