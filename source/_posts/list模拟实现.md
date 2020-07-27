---
title: list模拟实现
date: 2020-03-30 00:00:00
categories: C/C++
tags: stl
---

```cpp
#include <assert.h>
#include <initializer_list>

template <typename T>
struct ListNode {
    ListNode(const T& val = T())
        : _prev(nullptr)
        , _next(nullptr)
        , _data(val) {}

    ListNode<T>* _prev;
    ListNode<T>* _next;
    T _data;
};

// 三个模板参数可以兼容普通迭代器和const迭代器
template <typename T, typename Ptr, typename Ref>
struct _ListIterator {
    typedef ListNode<T> Node;
    typedef _ListIterator<T, Ptr, Ref> Self;

    _ListIterator(Node* node = nullptr)
        : _node(node) {}

    Ref operator*() {
        return _node->_data;
    }

    Ptr operator->() {
        return &(_node->_data);
    }

    // ++iterator
    Self& operator++() {
        _node = _node->_next;
        return *this;
    }

    // iterator++
    Self operator++(int) {
        Self tmp(*this);
        _node = _node->_next;
        return tmp;
    }

    // --iterator
    Self& operator--() {
        _node = _node->_prev;
        return *this;
    }

    // iterator--
    Self operator--(int) {
        Self tmp(*this);
        _node = _node->_prev;
        return tmp;
    }

    bool operator!=(const Self& it) {
        return _node != it._node;
    }

    bool operator==(const Self& it) {
        return _node == it._node;
    }

    Node* _node;
};

template <typename T>
class List {
public:
    typedef ListNode<T> Node;
    typedef _ListIterator<T, T*, T&> iterator;
    typedef _ListIterator<T, const T*, const T&> const_iterator;

    List()
        : _head(new Node) {
        _head->_next = _head;
        _head->_prev = _head;
    }

    List(size_t n, const T& val = T())
        : List() {
        while (n--) {
            push_back(val);
        }
    }

    List(const List<T>& lst)
        : List() {
        for (const auto& e : lst) {
            push_back(e);
        }
    }

    List(List<T>&& lst)
        : List() {
        std::swap(_head, lst._head);
    }

    List(const std::initializer_list<T>& lst)
        : List() {
        for (const auto& e : lst) {
            push_back(e);
        }
    }

    List<T>& operator=(List<T> lst) {
        std::swap(_head, lst._head);
        return *this;
    }

    ~List() {
        if (_head) {
            clear();
            delete _head;
            _head = nullptr;
        }
    }

    iterator begin() {
        return iterator(_head->_next);
    }

    iterator end() {
        return iterator(_head);
    }

    const_iterator begin() const {
        return const_iterator(_head->_next);
    }

    const_iterator end() const {
        return const_iterator(_head);
    }

    bool empty() const {
        return size() == 0;
    }

    size_t size() const {
        Node* cur = _head->_next;
        int size = 0;
        while (cur != _head) {
            ++size;
            cur = cur->_next;
        }
        return size;
    }

    void push_front(const T& val) {
        insert(begin(), val);
    }

    void pop_front() {
        erase(begin());
    }

    void push_back(const T& val) {
        insert(end(), val);
    }

    void pop_back() {
        erase(--end());
    }

    iterator insert(iterator pos, const T& val) {
        Node* cur = pos._node;
        Node* prev = cur->_prev;
        Node* newNode = new Node(val);

        newNode->_next = cur;
        newNode->_prev = prev;

        cur->_prev = newNode;
        prev->_next = newNode;
        return --pos;
    }

    // 删除：迭代器会失效，返回下一个元素的位置
    iterator erase(iterator pos) {
        assert(pos != end());
        Node* cur = pos._node;
        Node* prev = cur->_prev;
        Node* next = cur->_next;

        prev->_next = next;
        next->_prev = prev;

        delete cur;
        cur = nullptr;
        pos = iterator(next);
        return pos;
    }

    void swap(List& lst) {
        std::swap(_head, lst._head);
    }

    void clear() {
        if (_head) {
            Node* cur = _head->_next;
            while (cur != _head) {
                Node* next = cur->_next;
                delete cur;
                cur = next;
            }
            _head->_next = _head;
            _head->_prev = _head;
        }
    }
private:
    Node* _head;
};
```

