---
title: 哈希冲突、unorde_map/set模拟实现
date: 2020-7-24 00:00:00
categories: C/C++
tags:
   - 哈希
   - unordered_map/set
mathjax: true
---

# 介绍

哈希是一种O(1)查找的数据结构，可以不经过任何比较，一次直接从表中得到要搜索的元素。通过hashFunc使元素的存储位置与它的关键码之间能够建立一一映射的关系
在插入元素时，由待插入元素的值根据一个特殊函数（哈希函数）计算出该元素的存储位置，并将该元素放置在此处。在搜索元素时，还是由搜索的元素值根据这个特殊函数计算存储位置，直接在该位置将元素取出即可
常见的哈希函数有：
直接定址法 Hash（Key）= A \* Key + B 直接取关键字本身或者他的线性函数来作为散列地址
除留余数法 Hash(key) = key % capacity 最常用的哈希函数，用一个数来对key取模，一般来说这个数都是容量
由于直接定址、除留余数法这些函数要求key必须为整形，而对于字符串类型就要别的方式，比如BKD对字符串每一个字符的ascii值或者字符串的大小进行计算，来推导出一个不容易产生冲突的key值，但是尽管如此，对与不同的数据根据哈希函数计算出来的值也是可能相同的，这就产生了哈希冲突，我们可以设计精妙的哈希函数来使产生哈希冲突的可能性尽可能的低，但不可能避免哈希冲突解决哈希冲突常见的办法有两种：闭散列和开散列

# 闭散列

当发生哈希冲突时，如果哈希表未被装满，说明在哈希表中必然还有空位置，那么可以把key存放到冲突位置中的下一个空位置中去，我们可以用线性探测或者二次探测的方法来寻找下一个空位置
**线性探测：** 从发生冲突的位置开始，依次向后探测，直到寻找到下一个空位置为止
插入：通过哈希函数获取待插入元素在哈希表中的位置，如果该位置中没有元素则直接插入新元素，如果该位置中有元素发生哈希冲突，使用线性探测找到下一个空位置，插入新元素
删除：采用闭散列处理哈希冲突时，不能随便物理删除哈希表中已有的元素，若直接删除元素会影响其他元素的搜索，因此线性探测采用标记的伪删除法来删除一个元素
线性探测缺点：一旦发生哈希冲突，所有的冲突连在一起，容易产生数据堆积，即：不同关键码占据了可利用的空位置，使得寻找某关键码的位置需要许多次比较，导致搜索效率降低
**二次探测：** 找下一个空位置的方法为$H_i = (H_0 + i^2) \% m$或者$H_i = (H_0 - i^2) \% m$其中：i = 1,2,3…，是通过散列函数Hash(x)对元素的关键码key进行计算得到的位置，m是表的大小
研究表明当表的长度为质数且表装载因子a不超过0.5时，新的表项一定能够插入，而且任何一个位置都不会被探查两次，因此只要表中有一半的空位置，就不会存在表满的问题。在搜索时可以不考虑表装满的情况，但在插入时必须确保表的装载因子a不超过0.5，如果超出必须考虑增容
二次探测缺点：空间利用率低
**提高性能：** (1)由于表中元素越多，产生冲突的概率越大，所以我们引入了负载因子，当容量到达百分之八十的时候就进行扩容操作。(2)除留余数法，最好模一个素数

模拟实现：

```cpp
enum STATE {
    EMPTY, EXIST, DELETE
};

template <class K, class V>
struct hashNode {
    std::pair<K, V> _data;
    STATE _state = EMPTY;
};

// 为了实现简单，将比较直接与元素绑定在一起
template <class K, class V>
class HashTable {
public:
    typedef hashNode<K, V> Node;

    HashTable(const int n = 10)
        : _ht(n)
        , _size(0) {}

    bool Insert(const std::pair<K, V>& data) {
        CheckCapacity();
        // 计算索引
        int index = data.first % _ht.size();
        // 判断当前位置是否有元素，若没有，则直接插入
        // 新的元素可以放在：EMPTY,DELETE
        while (_ht[index]._state == EXIST) {
            // 如果有：判读当前位置的元素的key是否和插入的数据相同
            if (_ht[index]._data.first == data.first) {
                return false;
            }
            // 如果有， 且key不同，继续向后遍历，找一个空的位置，进行插入（线性探测）
            ++index;
            if (index == _ht.size()) {
                index = 0;
            }
        }
        // 元素插入
        _ht[index]._data = data;
        _ht[index]._state = EXIST;
        ++_size;
    }

    void CheckCapacity() {
        if (_ht.size() == 0 || _size * 10 / _ht.size() >= 8) {
            // 负载因子大于阈值，进行增容
            // int newC = _ht.size() == 0 ? 10 : 2 * _ht.size();
            size_t newC = getNextPrime(_ht.size());
            HashTable<K, V> newHt(newC);
            // 旧表元素重新插入到新表
            for (int i = 0; i < _ht.size(); ++i) {
                if (_ht[i]._state == EXIST) {
                    newHt.Insert(_ht[i]._data);
                }
            }
            // 旧表新表交换
            std::swap(_ht, newHt._ht);
        }
    }

    Node* Find(const K& key) {
        int index = key % _ht.size();
        // 直到遇到空结束
        while (_ht[index]._state != EMPTY) {
            if (_ht[index]._state == EXIST) {
                if (_ht[index]._data.first == key) {
                    return &_ht[index];
                }
            }
            ++index;
            if (index == _ht.size()) {
                index = 0;
            }
        }
        return nullptr;
    }

    bool Erase(const K& key) {
        Node* pos = Find(key);
        if (pos) {
            pos->_state = DELETE;
            --_size;
            return true;
        }
        return false;
    }

    size_t getNextPrime(size_t sz) {
        const int PRIMECOUNT = 28;
        const static size_t primeList[PRIMECOUNT] = {
            53ul, 97ul, 193ul, 389ul, 769ul,
            1543ul, 3079ul, 6151ul, 12289ul, 24593ul,
            49157ul, 98317ul, 196613ul, 393241ul, 786433ul,
            1572869ul, 3145739ul, 6291469ul, 12582917ul, 25165843ul,
            50331653ul, 100663319ul, 201326611ul, 402653189ul, 805306457ul,
            1610612741ul, 3221225473ul, 4294967291ul
        };
        for (int i = 0; i < PRIMECOUNT; ++i) {
            if (primeList[i] > sz) {
                return primeList[i];
            }
        }
        return primeList[PRIMECOUNT - 1];
    }
private:
    std::vector<Node> _ht;
    size_t _size;
};
```

# 开散列

对关键码集合用散列函数计算散列地址，具有相同地址的关键码归于同一子集合，每一个子集合称为一个桶，各个桶中的元素通过一个单链表链接起来（数据过多时可能会转为建立红黑树），各链表的头结点存储在哈希表中，开散列最好的情况是每个哈希桶中刚好挂一个节点，再继续插入元素时，每一次都会发生哈希冲突，因此，在元素个数刚好等于桶的个数时，可以给哈希表增容

模拟实现：

```cpp
// hash.hpp
// 开散列
template <class V>
struct HashNode {
    HashNode(const V& data = V())
        : _data(data)
        , _next(nullptr) {}
    V _data;
    HashNode<V>* _next;
};

// 迭代器前置声明
template <class K, class V, class KeyOfValue, class HFun>
struct _HashIterator;

template <class K, class V, class KeyOfValue, class HFun>
class HashTable {
public:
    typedef HashNode<V> Node;
    typedef Node* pNode;

    typedef _HashIterator<K, V, KeyOfValue, HFun> iterator;

    // 模板友元类声明
    template <class K, class V, class KeyOfValue, class HFun>
    friend struct _HashIterator;

    iterator begin() {
        // 找到非空链表的头节点
        for (size_t i = 0; i < _ht.size(); ++i) {
            // this: 当前调用函数的对象hastTbale
            if (_ht[i]) {
                return iterator(_ht[i], this);
            }
        }
        return iterator(nullptr, this);
    }

    iterator end() {
        return iterator(nullptr, this);
    }

    HashTable(size_t N = 10) {
        _ht.resize(N);
        _size = 0;
    }

    size_t hashIndex(const K& key, size_t sz) {
        HFun hf;
        // 哈希函数把key转成整数
        return hf(key) % sz;
    }

    std::pair<iterator, bool> insert(const V& data) {
        CheckCapacity();
        // 计算位置
        KeyOfValue kov;
        int index = hashIndex(kov(data), _ht.size());

        // 遍历单链表
        pNode cur = _ht[index];
        while (cur) {
            if (kov(cur->_data) == kov(data)) {
                return std::make_pair(iterator(cur, this), false);
            }
            cur = cur->_next;
        }

        // 插入：头插
        cur = new Node(data);
        cur->_next = _ht[index];
        _ht[index] = cur;
        ++_size;
        return std::make_pair(iterator(cur, this), true);
    }

    size_t getNextPrime(size_t sz) {
        const int PRIMECOUNT = 28;
        const static size_t primeList[PRIMECOUNT] = {
            53ul, 97ul, 193ul, 389ul, 769ul,
            1543ul, 3079ul, 6151ul, 12289ul, 24593ul,
            49157ul, 98317ul, 196613ul, 393241ul, 786433ul,
            1572869ul, 3145739ul, 6291469ul, 12582917ul, 25165843ul,
            50331653ul, 100663319ul, 201326611ul, 402653189ul, 805306457ul,
            1610612741ul, 3221225473ul, 4294967291ul
        };
        for (int i = 0; i < PRIMECOUNT; ++i) {
            if (primeList[i] > sz) {
                return primeList[i];
            }
        }
        return primeList[PRIMECOUNT - 1];
    }

    void CheckCapacity() {
        if (_size == _ht.size()) {
            size_t newC = getNextPrime(_ht.size());
            std::vector<pNode> newHt;
            newHt.resize(newC);
            KeyOfValue kov;
            // 移动旧表的元素到新表
            for (size_t i = 0; i < _ht.size(); ++i) {
                // 单链表头节点
                pNode cur = _ht[i];
                // 遍历单链表
                while (cur) {
                    pNode next = cur->_next;
                    // 重新计算在新表中的位置
                    int index = hashIndex(kov(cur->_data), newHt.size());
                    // 头插
                    cur->_next = newHt[index];
                    newHt[index] = cur;
                    cur = next;
                }
                _ht[i] = nullptr;
            }
            std::swap(_ht, newHt); // start, finish, endOfStorage
        }
    }

    pNode Find(const K& key) {
        int index = key % _ht.size();
        pNode cur = _ht[index];
        KeyOfValue kov;
        while (cur) {
            if (kov(cur->_data) == key) {
                return cur;
            }
            cur = cur->_next;
        }
        return nullptr;
    }

    bool Erase(const K& key) {
        int index = hashIndex(key, _ht.size());
        pNode cur = _ht[index];
        pNode parent = nullptr;
        KeyOfValue kov;
        while (cur) {
            // 删除
            if (kov(cur->_data) == key) {
                // 判断是否为头
                if (parent == nullptr) {
                    _ht[index] = cur->_next;
                } else {
                    parent->_next = cur->_next;
                }
                delete cur;
                --_size;
                return true;
            }
            parent = cur;
            cur = cur->_next;
        }
        return false;
    }
private:
    // 指针数组
    std::vector<pNode> _ht;
    size_t _size;
};

template <class K, class V, class KeyOfValue, class HFun>
struct _HashIterator {
    typedef HashNode<V> Node;
    typedef Node* pNode;

    typedef _HashIterator<K, V, KeyOfValue, HFun> Self;
    typedef HashTable<K, V, KeyOfValue, HFun> HTable;

    _HashIterator(pNode node, HTable* pht)
        : _node(node)
        , _pht(pht) {}

    V& operator*() {
        return _node->_data;
    }

    V* operator->() {
        return &_node->_data;
    }

    bool operator!=(const Self& it) {
        return _node != it._node;
    }

    Self& operator++() {
        if (_node->_next) {
            _node = _node->_next;
        } else {
            KeyOfValue kov;
            // 下一个非空链表的头节点
            size_t index = _pht->hashIndex(kov(_node->_data), _pht->_ht.size());
            ++index;
            while (index < _pht->_ht.size()) {
                if (_pht->_ht[index]) {
                    _node = _pht->_ht[index];
                    break;
                }
                ++index;
            }
            if (index == _pht->_ht.size()) {
                _node = nullptr;
            }
        }
        return *this;
    }
private:
    pNode _node;
    HTable* _pht;
};
```

# unordered_map、set模拟实现

```cpp
#include "hash.hpp"

template <class K>
struct hashFun {
    K& operator()(const K& key) {
        return key;
    }
};

// 模板特化
template <>
struct hashFun<std::string> {
    size_t operator()(const std::string& str) {
        // BKD
        size_t hash = 0;
        for (auto& e : str) {
            hash = hash * 131 + e;
        }
        return hash;
    }
};

//template <>
//struct hashfun<自定义类型> {
//	size_t operator()(const 自定义类型& key) {
//        // ...
//	}
//};


template <class K, class V, class HFun = hashFun<K>>
class UMap {
    struct MapKeyOfValue {
        const K& operator()(const std::pair<K, V>& data) {
            return data.first;
        }
    };
public:
    typedef typename HashTable<K, std::pair<K, V>, MapKeyOfValue, HFun>::iterator iterator;

    iterator begin() {
        return _ht.begin();
    }

    iterator end() {
        return _ht.end();
    }

    std::pair<iterator, bool> insert(const std::pair<K, V>& data) {
        return _ht.insert(data);
    }

    V& operator[](const K& key) {
        // pair<iterator, bool>
        std::pair<iterator, bool> ret = _ht.insert(std::make_pair(key, V()));
        return ret.first->second;
    }
private:
    HashTable<K, std::pair<K, V>, MapKeyOfValue, HFun> _ht;
};

template <class K, class HFun>
class USet {
    struct SetKeyOfValue {
        const K& operator()(const K& data) {
            return data;
        }
    };

public:
    typedef typename HashTable<K, K, SetKeyOfValue, HFun>::iterator iterator;

    iterator begin() {
        return _ht.begin();
    }

    iterator end() {
        return _ht.end();
    }

    std::pair<iterator, bool> Insert(const K& data) {
        return _ht.insert(data);
    }
private:
    HashTable<K, K, SetKeyOfValue, HFun> _ht;
};
```
