---
title: Redis数据结构与对象
date: 2019-12-24 21:04:07
tags:
category: Redis
---

# 简单动态字符串

Redis使用一个抽象类型来保存字符串，称为简单动态字符串（SDS）。

每个sds.h/sdshdr结构表示一个SDS值：

```c
struct sdshdr {
	// 记录buf数组中已经使用字节的数量，也就是字符串的长度
    int len;
    // 记录buf数组中未使用子节的数量
    int free;
    // 字节数组，用于保存字符串
    char buf[];
}
```

SDS与C字符串的区别：

- C：获取字符串长度的复杂度为O(N)，SDS获取字符串长度的复杂度为O(1)。
- C：API不安全，可能会造成缓冲区溢出，SDS：API是安全的，不会造成缓冲区溢出。
- C：修改字符串长度N次必然执行N次内存重分配，SDS：修改字符串长度N次最多执行N次内存重分配
- C：只能保存文本数据，SDS：可以保存二进制数据
- C：可以使用所有<string.h>库中的函数，SDS：可以使用一部分<string.h>库中的函数。

# 链表

链表被广泛用于实现Redis的各种功能，比如列表键、发布与订阅、慢查询、监视器等。

每个链表节点用一个adlist.h/listNode结构来表示：

```c
typedef struct listNode {
    struct listNode *perv;
    struct listNode *next;
    void *value;
}listNode;
```

另外使用adlist.h/list来持有链表：

```c
typedef struct list {
    listNode *head;
    listNode *tail;
    unsigned long len;
    void *(*dup) (void *ptr);
    void (*free) (void *ptr);
    int (*match) (void *ptr, void *key);
}list;
```

通过为链表设置不同的类型特定函数，Redis的链表可以用于保存各种不同类型的值。

# 字典

字典在Redis中的应用相当广泛，比如Redis的数据库就是使用字典来作为底层实现的，对数据库的增、删、查、改操作也是构建在对字典的操作之上的。

## 哈希表

Reids的字典在底层使用哈希表dict.h/dictht结构定义：

```c
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;
    
    // 哈希表大小
    unsigned long size;
    
    // 哈希表大小掩码，用于计算索引值
    // 总是等于size-1
    unsigned long sizemask;
    
    // 该哈希表已有节点的数量
    unsigned long used;
}dictht;
```

## 哈希表节点

哈希表节点用dictEntry结构表示，每个dictEntry结构都保存着一个键值对

```c
typedef struct dictEntry {
    // 键
    void *key;
    
    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    }v;
    
    // 指向下一个哈希表节点，形成链表
    struct dictEntry *next;
}dictEntry
```

## 字典

字典由dict.h/dict结构表示：

```c
typedef struct dict {
    
    // 类型特定函数
    dictType *type;
    
    // 私有数据
    void *privdata;
    
    // 哈希表
    dictht ht[2];
    
    // rehash索引
    // 当rehash不在进行时，值为-1
    int rehashidx;
}
```

- type属性时一个指向dicType结构的指针，每个dictType结构保存了一簇用于操作特定类型键值对的函数，Redis会为不同的字典设置不同的类型特定函数
- privdata保存了需要传给那些类型热定函数的可选参数
- ht是包含两个项的数组，数组中的每个项都是一个dictht哈希表，一般情况下只使用ht[0]哈希表，ht[1]哈希表只会在对ht[0]哈希表进行rehash时使用。
- rehashidx记录了rehash目前的进度

## 哈希算法

Redis计算哈希值和索引值的方法如下：

```c
hash = dict->type->hashFunction(key);
index = hash & dict->ht[x].sizemask;
```

计算哈希值的函数使用的是MurmurHash2算法

## 解决键冲突

使用next指针建立链表，每次讲新的冲突键值对插入到表头。

## rehash

rehash步骤为：

1）为字典的ht[1]哈希表分配空间，这个哈希表的空间大小取决于要执行的操作，以及ht[0]当前包含的键值对数量

- 如果执行的是扩展操作，那么ht[1]的大小为第一个大于等于ht[0].used*2的2^n（2

  的n次幂）

- 如果执行的是收缩操作，那么ht[1]的大小为第一个大于等于ht[0].used的2^n（2的n次幂）

2）将保存在ht[0]中的所有键值对重新计算键的哈希值和索引值，然后将键值对放置到ht[1]哈希表指定位置上

3）所有键值对迁移完成后，释放ht[0]，将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表，为下一次rehash做准备

## 渐进式rehash

不是一次性完成rehash，而是分多次、渐进式的完成。

1）让字典同时持有ht[0]和ht[1]两个哈希表

2）在字典中维持一个索引计数器变量rehashidx，将它设置为0，表示rehash工作正式开始

3）在rehash进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作外，顺带将ht[0]在rehashidx索引上的所有键值对rehash到ht[1]，并将rehashidx加一

4）所有键值对迁移完成后，rehashidx设为-1

# 跳跃表

跳跃表支持平均O(logN)、最坏O(N)复杂度的节点查找，还可以通过顺序性操作来批量处理节点。

## 跳跃表节点

redis.h/zskiplistNode

```c
typedef struct zskiplistNode {
   // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
    // 后退指针
    struct zskiplistNode *backward;
    // 分值
    double score;
    // 成员对象
    robj *obj;
} zskiplistNode;
```

## 跳跃表

redis.h/zskiplist

```c
typedef struct zskiplist {
    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;
    // 表中节点的数量
    unsigned long length;
    // 表中层数最大的节点的层数
    int level;
} zskiplist;
```

# 整数集合

整数集合时集合键的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis就会使用整数集合作为集合键的底层实现。

## 整数集合的实现

intset.h/intset

```c
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    //集合包含的元素数量
    uint32_t length;
    //保存元素的数组
    int8_t contents[];
}
```

## 升级

当要添加的新元素比整数集合现有所有元素的类型都要长时，需要对整数集合进行升级，才能将新元素添加到整数集合里。

升级整数集合并添加新元素共分为三步进行：

- 根据新元素的类型，扩展整数集合底层数组的空间大小，并为新元素分配空间。
- 将底层数组现有的所有元素都转换为新元素相同的类型，并放置到正确的位置上，且保持有序性不变。
- 将新元素添加到底层数组里面。

# 压缩列表

压缩列表是列表键和哈希键的底层实现之一。

## 压缩列表的构成

压缩列表的各个组成部分：

- zlbytes：记录整个压缩列表占用的内存字节数
- zltail：记录压缩列表表尾节点距离起始地址有多少字节
- zllen：记录压缩列表包含的节点数量
- entryX：压缩列表的各个节点
- zlend：特殊值0xFF（十进制255），用于标记压缩列表的末端

## 压缩列表节点的构成

每个压缩列表节点可以保存一个字节数组或者一个整数值。每个节点都由previous_entry_length、encoding、content三个部分组成

# 对象

Redis并没有直接使用前面的数据结构来实现键值对数据库，而是基于这些数据结构创建了一个对象系统，这个系统包含字符串对象、列表对象、哈希对象、集合对象和有序集合对象这五种类型的对象，每个对象都用到了至少一种我们前面所介绍的数据结构。

## 对象的类型与编码

每次当我们在Redis的数据库中新创建一个键值对时，我们至少会创建两个对象：键对象和值对象。

Redis中的每个对象都由一个redisObject结构表示，该结构中和保存数据有关的三个属性分别是type属性、encoding属性和ptr属性：

```c
typedef struct redisObject {
    \\ 类型
	unsigned type:4;
    \\ 编码
    unsigned encoding:4;
    \\ 指向底层实现数据结构的指针
    void *ptr;
    \\ ...
} robj;
```

### 类型

对象的type属性记录了对象的类型，这个属性的值可以是REDIS_STRING、REDIS_LIST、REDIS_HASH、REDIS_SET、REDIS_ZSET

### 编码和底层实现

对象的ptr指针指向对象的底层实现数据结构，而这些数据结构由对象的encoding属性决定。

