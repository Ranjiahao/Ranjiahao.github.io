---
title: 红黑树与map、set模拟实现
date: 2020-7-23 00:00:00
categories: C/C++
tags:
   - 红黑树
   - map/set
---

由于AVL树不适用一些经常修改的结构，这是因为AVL树追求绝对的平衡，所以就有了一种近似平衡的二叉搜索树红黑树，相对而言，降低了插入和旋转的次数，所以在经常进行增删的结构中性能比AVL树更优，而且红黑树实现比较简单，所以实际运用中红黑树更多。

红黑树在每个结点上增加一个存储位表示结点的颜色，可以是Red或Black，它有这些特点：

1. 每个结点不是红色就是黑色
2. 根节点是黑色的
3. 如果一个节点是红色的，则它的两个孩子结点是黑色的
4. 对于每个结点，从该结点到其所有后代叶结点的简单路径上，均包含相同数目的黑色结点
5. 每个叶子结点都是黑色的(此处的叶子结点指的是空结点)

由于红黑树的这些特点，所以红黑树能保证其最长路径中节点个数不会超过最短路径节点个数的两倍

红黑树的插入：
首先按照二叉搜索树的规则插入新节点，如果新节点设为黑色，就会导致根到叶子的路径上有一条路上，多一个额外的黑节点，这个是很难调整的。但是设为红色节点后，可能会导致出现两个连续红色节点的冲突，那么可以通过颜色调换和树旋转来调整
插入红色节点后，分情况讨论当前红黑树的状态

1. 其双亲节点的颜色是黑色，没有违反红黑树任何性质，则不需要调整
2. 其双亲节点的颜色是红色，约定n为当前节点，p为父节点，g为祖父节点，u为叔叔节点
    (2.1) n为红，p为红，g为黑，u存在且为红，则需要将p、u改为黑，g改为红，然后把g当成n，继续向上调整
     ![2.1](https://upload.wikimedia.org/wikipedia/commons/c/c8/Red-black_tree_insert_case_3.png)
    (2.2) n为红，p为红，g为黑，u不存在/u为黑
            (2.2.1) p为g的左孩子，n为p的左孩子，进行右单旋转，同理p为g的右孩子，n为p的右孩子，进行左单旋转
             ![2.2.1](https://upload.wikimedia.org/wikipedia/commons/6/66/Red-black_tree_insert_case_5.png)
            (2.2.2) p为g的左孩子，n为p的右孩子，进行左旋转后变为情况2.2.1，再进行右旋
                        同理p为g的右孩子，n为p的左孩子，进行右旋转后变为情况2.2.1，再进行左旋
             ![2.2.2](https://upload.wikimedia.org/wikipedia/commons/5/56/Red-black_tree_insert_case_4.png)

```cpp
//RBTree.hpp

#include <iostream>

enum COLOR {
    RED, BLACK
};

// 红黑树节点
template <class V>
struct RBTNode {
    RBTNode(const V& data = V())
        : _data(data)
        , _left(nullptr)
        , _right(nullptr)
        , _parent(nullptr)
        , _color(RED) {}

    V _data;
    RBTNode<V>* _left;
    RBTNode<V>* _right;
    RBTNode<V>* _parent;
    COLOR _color;
};

// 红黑树迭代器
template <class V>
class _RBTreeIterator {
public:
    typedef RBTNode<V> Node;
    typedef Node* pNode;
    typedef _RBTreeIterator<V> Self;

    _RBTreeIterator(pNode node)
        : _node(node) {}

    V& operator*() {
        return _node->_data;
    }

    V* operator->() {
        return &_node->_data;
    }

    bool operator!=(const Self& it) {
        return _node != it._node;
    }

    bool operator==(const Self& it) {
        return _node == it._node;
    }

    Self& operator++() {
        if (_node->_right) {
            // 找到右子树的最左节点
            _node = _node->_right;
            while (_node->_left) {
                _node = _node->_left;
            }
        } else {
            // 向上回溯
            pNode parent = _node->_parent;
            while (parent->_right == _node) {
                _node = parent;
                parent = parent->_parent;
            }
            // 特殊情况：迭代器在根节点位置，并且根节点无右孩子
            if (_node->_right != parent) {
                _node = parent;
            }
        }
        return *this;
    }

    Self& operator--() {
        // 迭代器在head位置
        // _node->color == RED 排除根节点
        if (_node->_parent->_parent == _node && _node->_color == RED) {
            _node = _node->_right;
        } else if (_node->_left) {
            // 找到左子树的最右节点
            _node = _node->_left;
            while (_node->_right) {
                _node = _node->_right;
            }
        } else {
            // 向上回溯
            pNode parent = _node->_parent;
            while (parent->_left == _node) {
                _node = parent;
                parent = parent->_parent;
            }
            _node = parent;
        }
    }
private:
    pNode _node;
};

// 红黑树结构：为了后续实现关联式容器简单，红黑树的实现中增加一个头结点
// 因为跟节点必须为黑色，为了与根节点进行区分，将头结点给成黑色
// 并且让头结点的Parent指向红黑树的根节点，Left指向红黑树中最小的节点，Right指向红黑树中最大的节点
template <class K, class V, class KeyOfValue>
class RBTree {
public:
    typedef RBTNode<V> Node;
    typedef Node* pNode;
    typedef _RBTreeIterator<V> iterator;

    RBTree(const V& data = V()) {
        _header = new Node(data);
        _header->_left = _header;
        _header->_right = _header;
        _header->_parent = nullptr;
    }

    iterator begin() {
        return iterator(_header->_left);
    }

    iterator end() {
        return iterator(_header);
    }

    iterator Find(const V& data) {
        KeyOfValue kov;
        if (_header == nullptr) {
            return nullptr;
        }
        pNode cur = _header->_parent;
        while (cur) {
            if (kov(cur->_data) > kov(data)) {
                cur = cur->_left;
            } else if (kov(cur->_data) < kov(data)) {
                cur = cur->_right;
            } else {
                return iterator(cur);
            }
        }
        return iterator(nullptr);
    }

    std::pair<iterator, bool> Insert(const V& data) {
        // 判断是否是空树
        if (_header->_parent == nullptr) {
            pNode root = new Node(data);
            // 根节点的颜色必须为黑的
            root->_color = BLACK;
            root->_parent = _header;

            _header->_left = root;
            _header->_right = root;
            _header->_parent = root;

            return std::make_pair(iterator(root), true);
        }

        // 从根节点开始搜索合适的叶子
        pNode cur = _header->_parent;
        pNode parent = nullptr;
        KeyOfValue kov;
        while (cur) {
            parent = cur;
            if (kov(cur->_data) > kov(data)) {
                cur = cur->_left;
            } else if (kov(cur->_data) < kov(data)) {
                cur = cur->_right;
            } else {
                return std::make_pair(iterator(cur), false);
            }
        }
        cur = new Node(data);
        pNode newNode = cur;
        if (kov(parent->_data) > kov(data)) {
            parent->_left = cur;
        } else {
            parent->_right = cur;
        }
        cur->_parent = parent;

        // 调整：修改颜色，旋转
        // cur和parent的颜色都为红色, 需要调整
        while (cur != _header->_parent && cur->_parent->_color == RED) {
            pNode parent = cur->_parent;
            pNode gParent = parent->_parent;

            if (gParent->_left == parent) {
                pNode uncle = gParent->_right;
                if (uncle && uncle->_color == RED) {
                    // 更新颜色
                    parent->_color = uncle->_color = BLACK;
                    gParent->_color = RED;
                    // 向上继续更新
                    cur = gParent;
                } else {
                    // u不存在/u存在且为黑
                    // 判断是否存在双旋的场景
                    if (parent->_right == cur) {
                        RotateLeft(parent);
                        std::swap(cur, parent);
                    }
                    // 右旋
                    RotateRight(gParent);
                    // 修改颜色
                    parent->_color = BLACK;
                    gParent->_color = RED;
                    // 结束调整
                    break;
                }
            } else {
                pNode uncle = gParent->_left;
                if (uncle && uncle->_color == RED) {
                    uncle->_color = parent->_color = BLACK;
                    gParent->_color = RED;
                    cur = gParent;
                } else {
                    // u不存在/u存在且为黑
                    // 判断是否存在双旋的场景
                    if (parent->_left == cur) {
                        RotateRight(parent);
                        std::swap(cur, parent);
                    }
                    RotateLeft(gParent);
                    parent->_color = BLACK;
                    gParent->_color = RED;
                    break;
                }
            }
        }
        // 根节点的颜色必须为黑色
        _header->_parent->_color = BLACK;

        // 调整之后，根可能发生变化
        _header->_left = leftMost();
        _header->_right = rightMost();

        return std::make_pair(iterator(newNode), true);
    }

    pNode leftMost() {
        pNode cur = _header->_parent;
        while (cur && cur->_left) {
            cur = cur->_left;
        }
        return cur;
    }

    pNode rightMost() {
        pNode cur = _header->_parent;
        while (cur && cur->_right) {
            cur = cur->_right;
        }
        return cur;
    }

    void RotateRight(pNode parent) {
        pNode subL = parent->_left;
        pNode subLR = subL->_right;

        // 更新left/right
        subL->_right = parent;
        parent->_left = subLR;

        // 更新parent
        pNode gParent = parent->_parent;
        parent->_parent = subL;
        if (subLR) {
            subLR->_parent = parent;
        }
        if (parent != _header->_parent) {
            if (gParent->_left == parent) {
                gParent->_left = subL;
            } else {
                gParent->_right = subL;
            }
            subL->_parent = gParent;
        } else {
            _header->_parent = subL;
            subL->_parent = _header;
        }
    }

    void RotateLeft(pNode parent) {
        pNode subR = parent->_right;
        pNode subRL = subR->_left;

        // 更新left/right
        subR->_left = parent;
        parent->_right = subRL;

        // 更新parent
        if (subRL) {
            subRL->_parent = parent;
        }
        if (parent != _header->_parent) {
            if (parent->_parent->_left == parent) {
                parent->_parent->_left = subR;
            } else {
                parent->_parent->_right = subR;
            }
            subR->_parent = parent->_parent;
        } else {
            _header->_parent = subR;
            subR->_parent = _header;
        }
        parent->_parent = subR;
    }

    void _Inorder(pNode root) {
        if (root) {
            _Inorder(root->_left);
            std::cout << root->_data.first << ":" << root->_data.second << " ";
            _Inorder(root->_right);
        }
    }

    void Inorder() {
        _Inorder(_header->_parent);
        std::cout << std::endl;
    }

    bool isRBTree() {
        // 红黑树的检测分为两步：
        // 1. 检测其是否满足二叉搜索树(中序遍历是否为有序序列)
        // 2. 检测其是否满足红黑树的性质
        pNode root = _header->_parent;
        if (root == nullptr) {
            return true;
        }
        if (root->_color == RED) {
            return false;
        }
        // 统计一条路径黑色的数量
        int blackCount = 0;
        pNode cur = root;
        while (cur) {
            if (cur->_color == BLACK) {
                ++blackCount;
            }
            cur = cur->_left;
        }
        // 前序遍历
        return _isRBTree(root, blackCount, 0);
    }

    bool _isRBTree(pNode root, int blackCount, int curBlackCount) {
        if (root == nullptr) {
            if (curBlackCount != blackCount) {
                return false;
            }
            return true;
        }

        // 累加黑节点
        if (root->_color == BLACK) {
            ++curBlackCount;
        }

        // 判断是否有红色连续
        if (root->_parent->_color == RED && root->_color == RED) {
            return false;
        }

        return _isRBTree(root->_left, blackCount, curBlackCount)
            && _isRBTree(root->_right, blackCount, curBlackCount);
    }
private:
    pNode _header;
};
```

```cpp
#include "RBTree.hpp"

template <class K, class V>
class Map {
    struct MapKeyOfValue {
        const K& operator() (const std::pair<K, V>& data) {
            return data.first;
        }
    };
public:
    typedef typename RBTree<K, std::pair<K, V>, MapKeyOfValue>::iterator iterator;
    // typedef _RBTreeIterator<pair<K, V>> iterator;

    iterator Find(const K& key) {
        return _rbt.Find(std::make_pair(key, V()));
    }

    iterator begin() {
        return _rbt.begin();
    }

    iterator end() {
        return _rbt.end();
    }

    std::pair<iterator, bool> Insert(const std::pair<K, V>& data) {
        return _rbt.Insert(data);
    }

    V& operator[](const K& key) {
        std::pair<iterator, bool> ret = _rbt.Insert(std::make_pair(key, V()));
        return ret.first->second;
    }
private:
    RBTree<K, std::pair<K, V>, MapKeyOfValue> _rbt;
};

template <class K>
class Set {
    struct SetKeyOfValue {
        const K& operator()(const K& data) {
            return data;
        }
    };
public:
    // typedef typename RBTree<K, K, SetKeyOfValue>::iterator iterator;
    typedef _RBTreeIterator<K> iterator;

    iterator Find(const K& key) {
        return _rbt.Find(key);
    }

    iterator begin() {
        return _rbt.begin();
    }

    iterator end() {
        return _rbt.end();
    }

    std::pair<iterator, bool> Insert(const K& data) {
        return _rbt.Insert(data);
    }
private:
    RBTree<K, K, SetKeyOfValue> _rbt;
};
```
