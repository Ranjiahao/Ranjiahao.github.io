---
title: SGI-STL空间配置器模拟实现
date: 2020-10-07 00:00:00
categories: C/C++
tags:
   - STL
---

STL为了各个容器高效的管理空间，设计一块高效的内存管理机制，避免了频繁向系统申请小块内存块造成内存碎片、影响程序运行效率，和直接使用malloc和new申请额外空间浪费、申请空间失败的问题，还考虑了线程安全问题。虽然内核中已经有了一个类似的slab分配器来管理小块内存但是内核是针对操作系统中所有程序的，他申请和释放内存的消耗会非常大，STL需要的全部都是小块内存，并且需求比较集中，如果自己设计一个效率更高，还能顺带解决内存碎片的问题

SGI-STL的空间配置器分为分为两级结构，一级空间配置器处理大块内存（大于128字节），二级空间配置器处理小块内存

一级空间配置器对malloc和free进行了封装，同时加上了对内存不足的处理方法每当通过malloc或者realloc申请空间时，如果出现了内存不足的情况，就会循环调用oom来进行处理，然后继续判断是否申请成功，失败则继续尝试。若用户不自己设定oom则内存不足时抛出bad_alloc异常

二级空间配置器专门负责处理小于128字节的小块内存。采用了内存池的技术来提高申请空间的速度以及减少额外空间的浪费，采用哈希桶的方式来提高用户获取空间的速度与高效管理，避免外碎片问题（但还是存在内碎片问题）
首先申请一大块内存，并且用类似哈希桶的结构来维护这个内存池，每一个桶中装载一个free-list单链表。总共维护16个free-lists，各自管理大小以8字节为倍数，依次是8，16，24，32，40，48，56，64，72，80，88，96，104，112，120，128字节，SGI-STL将用户申请的内存块向上对齐到了8的整数倍
当用户申请的空间小于128字节则在桶中寻找可利用的内存

1. 如果有则取出分给用户
2. 如果没有则需要调用refill向内存池申请空间，refill中调用chunk_alloc一次申请对应字节的20块内存
    2.1 若内存池内存足够，则将一个分配给用户，其余内存挂到哈希桶中
    2.2 若内存池中内存小于20块大于1块，则同样将一个分配给用户，其余内存挂到哈希桶中
    2.3 若内存不足1块，则把内存池中剩余的空间挂到对应哈希桶中，然后调用malloc向系统申请
        2.3.1 若申请成功则放入内存池，重复调用chunk_alloc
        2.3.2 若申请失败则从哈希桶中找更大的内存块
            2.3.2.1 如果有则从哈希桶中取出给用户
            2.3.2.2 如果没有则调用一级空间配置器，成功则放入内存重复调用chunk_alloc，失败抛异常

```cpp
#ifdef USE_MALLOC
// 使用一级空间配置器
typedef malloc_alloc_template malloc_alloc;
typedef malloc_alloc alloc;
#else
// 使用二级空间配置器，默认使用二级空间配置器
typedef default_alloc_tmplate alloc;
#endif // USE_MALLOC



///////////////////////////////////////////////
//               一级空间配置器               //
///////////////////////////////////////////////

class malloc_alloc_template {
private:
    static void* oom_malloc(size_t);
    static void(*malloc_alloc_oom_handler)();
public:
    // 对malloc的封装
    static void* allocate(size_t n) {
        // 申请空间成功，直接返回，失败交由oom_malloc处理
        void* result = malloc(n);
        if (result == 0) {
            result = oom_malloc(n);
        }
        return result;
    }

    // 对free的封装
    static void deallocate(void* p) {
        free(p);
    }

    // 不可直接使用new-handler机制，因为它并非使用::operator new来配置内存
    // new-handler机制是一旦::operator new无法完成任务，再丢出std::bad_alloc之前会调用用户指定的函数
    // 模拟set_new_handle()
    // 该函数的参数为函数指针，返回值类型也为函数指针
    //                  函数名            参数
    // void (*    set_malloc_handler( void (*f)() )    )()
    // 返回值类型void(*)()
    static void(*set_malloc_handler(void(*f)()))() {
        void(*old)() = malloc_alloc_oom_handler;
        malloc_alloc_oom_handler = f;
        return old;
    }
};

// 默认用户处理函数为nullptr
void(*malloc_alloc_template::malloc_alloc_oom_handler)() = nullptr;

// malloc申请空间失败时代用该函数
void* malloc_alloc_template::oom_malloc(size_t n) {
    void(*my_malloc_handler)() = nullptr;
    void* result = nullptr;
    while (1) {
        my_malloc_handler = malloc_alloc_oom_handler;
        // 检测用户是否设置空间不足应对措施，如果没有设置，则抛异常
        if (my_malloc_handler == nullptr) {
            throw std::bad_alloc();
        }
        // 如果设置，执行用户提供的空间不足应对措施
        (*my_malloc_handler)();
        // 继续申请空间，可能就会申请成功，若失败则继续尝试
        result = malloc(n);
        if (result) {
            return result;
        }
    }
}



///////////////////////////////////////////////
//               二级空间配置器               //
///////////////////////////////////////////////

enum { ALIGN = 8 };                      // 小型区块的上调边界
enum { MAX_BYTES = 128 };                // 小型区块的上限
enum { NFREELISTS = MAX_BYTES / ALIGN }; // 哈希桶freelist个数

// 暂时不考虑多线程情况
class default_alloc_tmplate {
private:
    // 如果用户所需内存块不是8的整数倍，则向上对齐到8的整数倍
    static size_t ROUND_UP(size_t bytes) {
        return (bytes + ALIGN - 1) & ~(ALIGN - 1);
    }

    // 采用联合体的方式管理内存碎片，分配与回收小内存区块
    union obj {
        union obj* free_list_link; // 指向下一个区块
        char client_data[1];       // 储存本块内存的首地址
    };

    // 16个freelist
    static obj* free_list[NFREELISTS];

    // 哈希函数，根据用户提供字节数找到对应的桶号
    static size_t FREELIST_INDEX(size_t bytes) {
        return (bytes + ALIGN - 1) / ALIGN - 1;
    }

    // 向内存池申请，往哈希桶中补充空间，返回首个小块内存的首地址
    static void* refill(size_t n);

    // 配置nobjs个大小为size的区块，nobjs可能减少
    static char* chunk_alloc(size_t size, int& nobjs);

    static char* start_free; // 内存池起始位置
    static char* end_free;   // 内存池结束位置
    static size_t heap_size; // 记录该空间配置器已经向系统索要了多少的内存块
public:
    static void* allocate(size_t n);
    static void deallocate(void* p, size_t n);
};

char* default_alloc_tmplate::start_free = nullptr;
char* default_alloc_tmplate::end_free = nullptr;
size_t default_alloc_tmplate::heap_size = 0;
default_alloc_tmplate::obj* default_alloc_tmplate::free_list[NFREELISTS] = {
    nullptr, nullptr, nullptr, nullptr, nullptr, nullptr, nullptr, nullptr,
    nullptr, nullptr, nullptr, nullptr, nullptr, nullptr, nullptr, nullptr
};

// 假设n已经调至8的倍数
void* default_alloc_tmplate::refill(size_t n) {
    // 一次性向内存池索要20个n字节的小块内存
    int nobjs = 20;
    char* chunk = chunk_alloc(n, nobjs);
    // 如果只要了一块，直接返回给用户使用
    if (nobjs == 1) {
        return chunk;
    }
    // 将第一块返回给用户，其他块连接在对应的桶中
    obj* result = (obj*)chunk;
    // 找到对应的桶号
    obj** my_free_list = free_list + FREELIST_INDEX(n);
    // 从1开始，因为第0个将返回给用户
    obj* next_obj = *my_free_list = (obj*)(chunk + n);
    for (int i = 1; i < nobjs ; i++) {
        obj* current_obj = next_obj;
        next_obj = (obj*)((char*)next_obj + n);
        if (i == nobjs - 1) {
            current_obj->free_list_link = nullptr;
        } else {
            current_obj->free_list_link = next_obj;
        }
    }
    return result;
}

// 假设size已经调至8的倍数
char* default_alloc_tmplate::chunk_alloc(size_t size, int& nobjs) {
    // 计算nobjs个size字节内存块的总大小以及内存池中剩余空间总大小
    char* result;
    size_t total_bytes = size * nobjs;
    size_t bytes_left = end_free - start_free; // 内存池剩余空间

    if (bytes_left >= total_bytes) {
        // 内存池剩余空间完全满足需求量
        result = start_free;
        start_free += total_bytes;
        return result;
    } else if (bytes_left >= size) {
        // 内存池不能完全满足需求量，但是可以供应一个区块
        nobjs = bytes_left / size;
        total_bytes = size * nobjs;
        result = start_free;
        start_free += total_bytes;
        return result;
    } else {
        // 内存池剩余空间连一个区块的大小都无法提供，向系统堆求助，往内存池中补充空间
        // 向内存池中补充空间大小：本次空间总大小两倍+向系统申请总大小/16
        size_t bytes_to_get = 2 * total_bytes + ROUND_UP(heap_size >> 4);
        // 如果内存池有剩余空间，该空间一定是8的整数倍，将该空间挂到对应哈希桶中
        if (bytes_left > 0) {
            // 找对用哈希桶，将剩余空间挂在其上
            obj** my_free_list = free_list + FREELIST_INDEX(bytes_left);
            ((obj*)start_free)->free_list_link = *my_free_list;
            *my_free_list = (obj*)start_free;
        }

        // 通过系统堆向内存池补充空间，如果补充成功，递归继续分配
        start_free = (char*)malloc(bytes_to_get);
        if (start_free == 0) {
            // 通过系统堆补充空间失败，在哈希桶中找是否有没有使用的较大的内存块
            for (int i = size; i <= MAX_BYTES; i += ALIGN) {
                obj** my_free_list = free_list + FREELIST_INDEX(i);
                obj* p = *my_free_list;
                // 如果有，将该内存块补充进内存池，递归继续分配
                if (p) {
                    *my_free_list = p->free_list_link;
                    start_free = (char*)p;
                    end_free = start_free + i;
                    return chunk_alloc(size, nobjs);
                }
            }
            end_free = nullptr;
            // 调用一级空间配置器
            start_free = (char*)malloc_alloc_template::allocate(bytes_to_get);
        }
        // 通过系统堆向内存池补充空间成功，更新信息并继续分配
        heap_size += bytes_to_get;
        end_free = start_free + bytes_to_get;
        return chunk_alloc(size, nobjs);
    }
}

void* default_alloc_tmplate::allocate(size_t n) {
    // 大于128字节则调用一级空间配置器
    if (n > (size_t)MAX_BYTES) {
        return malloc_alloc_template::allocate(n);
    }
    // 根据用户所需字节找到对应的桶号
    obj** my_free_list = free_list + FREELIST_INDEX(n);
    obj* result = *my_free_list;
    // 如果该桶中没有内存块时，向该桶中补充空间
    if (result == 0) {
        // 将n向上对齐到8的整数被，保证向桶中补充内存块时，内存块一定是8的整数倍
        void* r = refill(ROUND_UP(n));
        return r;
    }
    // 维护桶中剩余内存块的链式关系
    *my_free_list = result->free_list_link;
    return result;
}

void default_alloc_tmplate::deallocate(void* p, size_t n) {
    // 大于128字节则调用一级空间配置器
    if (n > (size_t)MAX_BYTES) {
        malloc_alloc_template::deallocate(p);
        return;
    }
    // 找到对应的哈希桶，将内存挂在哈希桶中
    obj** my_free_list = free_list + FREELIST_INDEX(n);
    obj* q = (obj*)p;
    q->free_list_link = *my_free_list;
    *my_free_list = q;
}



// 对空间配置器进行封装
// 只负责申请与归还对象的空间，不负责空间中对象的构造与析构
// T: 元素类型    Alloc: 空间配置器
template <class T, class Alloc>
class simple_alloc {
public:
    // 申请n个T类型对象大小的空间
    static T* allocate(size_t n) {
        return n ? (T*)Alloc::allocate(n * sizeof(T)) : nullptr;
    }

    // 申请一个T类型对象大小的空间
    static T* allocate(void) {
        return (T*)Alloc::allocate(sizeof(T));
    }

    // 释放n个T类型对象大小的空间
    static void deallocate(T *p, size_t n) {
        if (n) {
            Alloc::deallocate(p, n * sizeof(T));
        }
    }

    // 释放一个T类型对象大小的空间
    static void deallocate(T *p) {
        Alloc::deallocate(p, sizeof(T));
    }
};

// 归还空间时，先先调用该函数将对象中资源清理掉
template <class T>
void destroy(T* pointer) {
    pointer->~T();
}

// 空间申请好后调用该函数：利用placement-new完成对象的构造
template <class T1, class T2>
void construct(T1* p, const T2& value) {
    new(p) T1(value);
}

//// 与容器结合
//template <class T, class Alloc = alloc>
//class list {
//    // 实例化空间配置器
//    typedef simple_alloc<list_node, Alloc> list_node_allocator;
//protected:
//    link_type get_node() {
//        // 调用空间配置器接口先申请节点的空间
//        return list_node_allocator::allocate();
//    }
//    // 将节点归还给空间配置器
//    void put_node(link_type p) {
//        list_node_allocator::deallocate(p);
//    }
//    // 创建节点：1. 申请空间 2. 完成节点构造
//    link_type create_node(const T& x) {
//        link_type p = get_node();
//        construct(&p->data, x);
//        return p;
//    }
//    // 销毁节点：1. 调用析构函数清理节点中资源 2. 将节点空间归还给空间配置器
//    void destroy_node(link_type p) {
//        destroy(&p->data);
//        put_node(p);
//    }
//};
```