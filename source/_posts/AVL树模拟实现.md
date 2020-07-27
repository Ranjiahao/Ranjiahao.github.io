---
title: AVL树模拟实现
date: 2020-7-18 00:00:00
categories: C/C++
tags:
   - AVL树
---

AVL树是一种高度平衡的二叉树，它改进了普通二叉搜索树在极端情况下会退化成单支效率低下的问题，AVL树中在每个节点中加入了平衡因子，使得左右子树的高度差不超过1，这样可以使得高度保持在log n，从而避免的极端情况下的查找效率低下问题，但是如果要对AVL树做一些结构修改的操作，性能非常低下，比如：插入时要维护其绝对平衡，旋转的次数比较多，更差的是在删除时，有可能一直要让旋转持续到根的位置。因此：如果需要一种查询高效且有序的数据结构，而且数据的个数为静态的(即不会改变)，可以考虑AVL树，但一个结构经常修改，就不太适合

插入：
首先根据二叉搜索树的规则插入新的节点，然后调整平衡因子即可
如果新节点插入到Parent的左侧，则Parent的平衡因子-1
如果新节点插入到Parent的右侧，则Parent的平衡因子+1
这时Parent的平衡因子有三种情况-1、0、1
若为0，则说明之前的平衡因子为-1/1，插入后调整为0，则插入成功返回即可
若为-1/1，则说明之前的平衡因子为0，插入后变为-1/1，则AVL树高度增加需要向上寻找，看看有没有别的平衡被破坏的节点
若为2，则说明违反了AVL树的性质，需要旋转处理

旋转：根据节点插入位置的不同，AVL树的旋转分为四种
1. Parent的平衡因子为2，说明Parent的右子树高，设Parent的右子树的根为SubR
    当SubR的平衡因子为1时，执行左单旋
    当SubR的平衡因子为-1时，执行右左双旋
2. Parent的平衡因子为-2，说明Parent的左子树高，设Parent的左子树的根为SubL
    当SubL的平衡因子为-1是，执行右单旋
    当SubL的平衡因子为1时，执行左右双旋

旋转完成后，原Parent为根的子树个高度降低，已经平衡，不需要再向上更新

单旋：

![img1](img1.png)

双旋：

![img2](img2.png)

![img3](img3.png)



```cpp
// AVLTree.hpp

#include <iostream>

template <class K, class V>
struct AVLNode {
    AVLNode(const std::pair<K, V>& data)
        : _data(data)
        , _left(nullptr)
        , _right(nullptr)
        , _parent(nullptr)
        , _bf(0) {}

    std::pair<K, V> _data;
    AVLNode<K, V>* _left;
    AVLNode<K, V>* _right;
    AVLNode<K, V>* _parent;
    int _bf;
};

template <class K, class V>
class AVLTree {
public:
    typedef AVLNode<K, V> Node;
    typedef Node* pNode;

    bool Insert(const std::pair<K, V>& data) {
        if (_root == nullptr) {
            _root = new Node(data);
            return true;
        }

        // 搜索
        pNode cur = _root;
        pNode parent = nullptr;
        while (cur) {
            parent = cur;
            if (cur->_data.first == data.first) {
                return false;
            } else if (cur->_data.first > data.first) {
                cur = cur->_left;
            } else {
                cur = cur->_right;
            }
        }
        cur = new Node(data);
        if (parent->_data.first > data.first) {
            parent->_left = cur;
        } else {
            parent->_right = cur;
        }
        cur->_parent = parent;

        // 调整
        while (parent) {
            // 更新平衡因子
            if (parent->_right == cur) {
                ++parent->_bf;
            } else {
                --parent->_bf;
            }
            if (parent->_bf == -1 || parent->_bf == 1) {
                // 向上继续更新
                cur = parent;
                parent = parent->_parent;
            } else if (parent->_bf == 0) {
                break;
            } else {
                if (parent->_bf == -2 && cur->_bf == -1) {
                    // 左边的左边高
                    RotateRight(parent);
                } else if (parent->_bf == 2 && cur->_bf == 1) {
                    // 右边的右边高
                    RotateLeft(parent);
                } else if (parent->_bf == 2 && cur->_bf == -1) {
                    // 右边的左边高
                    int bf = cur->_left->_bf;
                    RotateRight(cur);
                    RotateLeft(parent);
                    if (bf == -1) {
                        cur->_bf = 1;
                    } else if (bf == 1) {
                        parent->_bf = -1;
                    } else {
                        // cur->_left 没有子树
                        parent->_bf = 0;
                    }
                } else if (parent->_bf == -2 && cur->_bf == 1) {
                    // 左边的右边高
                    int bf = cur->_right->_bf;
                    RotateLeft(cur);
                    RotateRight(parent);
                    if (bf == -1) {
                        parent->_bf = 1;
                    } else if (bf == 1) {
                        cur->_bf = -1;
                    } else {
                        // cur->_right 没有子树
                        parent->_bf = 0;
                    }
                }
                break;
            }
        }
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
        if (parent != _root) {
            if (gParent->_left == parent) {
                gParent->_left = subL;
            } else {
                gParent->_right = subL;
            }
            subL->_parent = gParent;
        } else {
            _root = subL;
            subL->_parent = nullptr;
        }

        // 更新平衡因子
        subL->_bf = parent->_bf = 0;
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
        if (parent != _root) {
            if (parent->_parent->_left == parent) {
                parent->_parent->_left = subR;
            } else {
                parent->_parent->_right = subR;
            }
            subR->_parent = parent->_parent;
        } else {
            _root = subR;
            subR->_parent = nullptr;
        }
        parent->_parent = subR;

        // 更新平衡因子
        subR->_bf = parent->_bf = 0;
    }

    void Inorder() {
        _Inorder(_root);
        std::cout << std::endl;
    }

    int Height(pNode root) {
        if (root == nullptr) {
            return 0;
        }
        int leftH = Height(root->_left);
        int rightH = Height(root->_right);
        return leftH > rightH ? leftH + 1 : rightH + 1;
    }

    bool isBalance() {
        return _isBalance(_root);
    }
private:
    void _Inorder(pNode root) {
        if (root) {
            _Inorder(root->_left);
            std::cout << root->_data.first << ":" << root->_data.second << " ";
            _Inorder(root->_right);
        }
    }

    bool _isBalance(pNode root) {
    // 1. 验证其为二叉搜索树
    //     如果中序遍历可得到一个有序的序列，就说明为二叉搜索树
    // 2. 验证其为平衡树
    //     每个节点子树高度差的绝对值不超过1(注意节点中如果没有平衡因子)
    //     节点的平衡因子是否计算正确
        if (root == nullptr) {
            return true;
        }

        int rightH = Height(root->_right);
        int leftH = Height(root->_left);

        if (root->_bf != rightH - leftH) {
            std::cout << root->_data.first << ":" << root->_data.second << "不平衡" << std::endl;
            std::cout << "平衡因子:" << root->_bf << "高度差：" << rightH - leftH << std::endl;
            return false;
        }

        return abs(root->_bf) < 2
            && _isBalance(root->_left)
            && _isBalance(root->_right);
    }

    pNode _root = nullptr;
};
```
