---
title: string模拟实现
date: 2020-03-16 00:00:00
categories: C/C++
tags: stl
---

```cpp
// string.h
#include <iostream>
#include <assert.h>
 
class String {
public:
    friend std::istream& operator>>(std::istream& _cin, String& s);
    friend std::ostream& operator<<(std::ostream& _cout, const String& s);
 
    typedef char* Iterator;
    typedef const char* const_Iterator;
 
    String(const String& s);
    String(const char* str = "");
    String(String&& s);
    ~String();
    String& operator=(String s);
 
    Iterator begin();
    const_Iterator begin() const;
    Iterator end();
    const_Iterator end() const;
 
    size_t size() const;
    void resize(size_t n, char c = '\0');
    size_t capacity() const;
    void reserve(size_t n);
    void clear();
    bool empty() const;
 
    char& operator[](size_t pos);
    const char& operator[](size_t pos) const;
 
    String& operator+=(const String& s);
    String& operator+=(const char* str);
    String& operator+=(char c);
    String& insert(size_t pos, const char* str);
    String& insert(size_t pos, size_t n, const char c);
    String& erase(size_t pos, size_t n);
    void swap(String& s);
 
    char* c_str() const;
    size_t find(const char* str, size_t pos = 0) const;
    size_t find_first_of(char c, size_t pos = 0) const;
 
    static size_t npos;
private:
    size_t _size;
    size_t _capacity;
    char* _str;
};



// string.cc
#include "string.h"

std::istream& operator>>(std::istream& _cin, String& s) {
    for (char c = _cin.get(); !isspace(c); _cin.get(c)) {
        s += c;
    }
    return _cin;
}

std::ostream& operator<<(std::ostream& _cout, const String& s) {
    _cout << s._str;
    return _cout;
}

String::String(const String& s)
    : _size(0)
    , _capacity(0)
    , _str(nullptr) {
    String tmp(s._str);
    swap(tmp);
}

String::String(const char* str)
    : _size(strlen(str))
    , _capacity(_size)
    , _str(new char[_capacity + 1]) {
    strcpy(_str, str);
}

String::String(String&& s)
    : String() {
    swap(s);
}

String::~String() {
    if (_str) {
        delete[] _str;
        _str = nullptr;
        _size = 0;
        _capacity = 0;
    }
}

String& String::operator=(String s) {
    swap(s);
    return *this;
}

String::Iterator String::begin() {
    return _str;
}

String::const_Iterator String::begin() const {
    return _str;
}

String::Iterator String::end() {
    return _str + _size;
}

String::const_Iterator String::end() const {
    return _str + _size;
}

size_t String::size() const {
    return _size;
}

void String::resize(size_t n, char c) {
    if (n < _size) {
        _size = n;
        _str[_size] = '\0';
    } else {
        if (n > _capacity) {
            reserve(n);
        }
        for (size_t i = _size; i < n; i++) {
            _str[i] = c;
        }
        _size = n;
        _str[_size] = '\0';
    }
}

size_t String::capacity() const {
    return _capacity;
}

void String::reserve(size_t n) {
    if (n > _capacity) {
        char* tmp = new char[n + 1]; // 实际会根据一定算法开辟空间
        if (_str) {
            strcpy(tmp, _str);
            delete[] _str;
        }
        _str = tmp;
        _capacity = n;
    }
}

void String::clear() {
    _size = 0;
    _str[_size] = '\0';
}

bool String::empty() const {
    return _size == 0;
}

char& String::operator[](size_t pos) {
    assert(pos <= _size);
    return _str[pos];
}

const char& String::operator[](size_t pos) const {
    assert(pos <= _size);
    return _str[pos];
}

String& String::operator+=(const String& s) {
    insert(_size, s._str);
    return *this;
}

String& String::operator+=(const char* str) {
    insert(_size, str);
    return *this;
}

String& String::operator+=(char c) {
    insert(_size, 1, c);
    return *this;
}

String& String::insert(size_t pos, const char* str) {
    assert(pos <= _size);
    int len = strlen(str);
    if (_size + len > _capacity) {
        reserve(_size + len);
    }
    size_t end = _size + len;
    while (end > pos + len - 1) {
        _str[end] = _str[end - len];
        --end;
    }
    while (*str) {
        _str[pos++] = *str++;
    }
    _size += len;
    return *this;
}

String& String::insert(size_t pos, size_t n, const char c) {
    assert(pos <= _size);
    if (_size + n > _capacity) {
        reserve(_size + n);
    }
    size_t end = _size + n;
    while (end > pos + n - 1) {
        _str[end] = _str[end - n];
        --end;
    }
    for (size_t i = 0; i < n; ++i) {
        _str[pos++] = c;
    }
    _size += n;
    return *this;
}

String& String::erase(size_t pos, size_t n) {
    assert(pos < _size);
    if (n >= _size - pos) {
        _size = pos;
        _str[_size] = '\0';
    } else {
        size_t begin = pos + n;
        while (begin <= _size) {
            _str[pos++] = _str[begin];
            ++begin;
        }
        _size -= n;
    }
    return *this;
}

void String::swap(String& s) {
    std::swap(_str, s._str);
    std::swap(_size, s._size);
    std::swap(_capacity, s._capacity);
}

char* String::c_str() const {
    return _str;
}

size_t String::find(const char* str, size_t pos) const {
    char* ret = strstr(_str + pos, str);
    if (ret != NULL) {
        return ret - _str;
    } else {
        return npos;
    }
}

size_t String::find_first_of(char c, size_t pos) const {
    for (size_t i = pos; i < _size; i++) {
        if (_str[i] == c) {
            return i;
        }
    }
    return npos;
}

size_t String::npos = -1;
```

