---
title: Redis Hash的实现
date: 2019-04-06
tags: redis
---

字典(hash）在Redis中使用非常广泛，Redis的数据库就是使用字典作为底层实现的，对数据库的增删改查也是建立在对字典的操作上的。Redis的字典使用哈希表作为底层实现。

### 哈希表节点
哈希表中的一个节点用来存储一个键值对，下面是哈希表节点结构的定义：

```c
typedef struct dictEntry {
    void *key; //键
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v; //值
    struct dictEntry *next; // 下一个节点，节点相连组成链表
} dictEntry;
```
key是键，v是值，值可以是一个指针、uint64_t整数、int64_t整数或者double类型浮点数。next属性指向下一个节点，在键的哈希值相同时，使用链表来解决冲突。

### 哈希表

```c
typedef struct dictht {
    dictEntry **table; // 哈希表数组
    unsigned long size; // 哈希表大小
    unsigned long sizemask; //哈希表掩码，用于计算索引值，等于size-1
    unsigned long used; // 哈希表已使用节点的数量
} dictht;
```
table属性是一个数组，数组中的元素是指向dictEntry的指针，size是table数组的大小，sizemask是哈希表掩码，计算索引时使用，used保存已有键值对的数量。

### 字典
```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int iterators; /* number of iterators currently running */
} dict;
```
type和privdata属性是为实现多态字典而设置的。

ht属性是一个包含两个dictht的数组，一般只使用ht[0], 需要rehash时再使用ht[1]。

rehashidx记录rehash当前的进度，没有rehash的话值为-1.

没有rehash状态下的字典：

![](https://i.loli.net/2019/04/16/5cb59b50df439.png)

### 哈希算法
添加新的键值对时，需要先根据键计算出哈希值哈索引值，然后将包含新键值对的节点放在哈希表数组的指定索引上。

```c
h = dictHashKey(d, key); // 计算哈希值
...
idx = h & d->ht[table].sizemask; //根据哈希值和掩码计算索引
```
对不同类型的key提供了不同的hash算法，好的hash算法需要满足两个条件：

1. 性能高，运算足够快
2. 相邻的数据hash后分布广；即使输入的键是有规律的，算法仍然能给出一个很好的随机分布性

利用哈希函数计算出一个数值，然后对这个数值和掩码进行&操作来得到在哈希表的存放索引。
### 重哈希
哈希表在满足一定条件时会进行扩展或收缩操作，主要是根据哈希表的负载因子来判断。

负载因子计算公式：

```
load_factor = ht[0].used / ht[0].size // 哈希表已保存的节点数量/ 哈希表大小
```
当负载因子过大时，会进行扩展，过小时会收缩，扩展和收缩就是使用重哈希(rehash)来进行的。

重哈希将ht[0]的所有键值对

```
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; // 最多访问的空bucket的个数， 为n*10
    if (!dictIsRehashing(d)) return 0; 

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        
        assert(d->ht[0].size > (unsigned long)d->rehashidx); // 确保rehashindex没有超出哈希表的大小
        while(d->ht[0].table[d->rehashidx] == NULL) { //rehashindex上面是空的，则继续访问下一个，并将rehashidx加1， 剩余的空节点访问个数减1
            d->rehashidx++;
            if (--empty_visits == 0) return 1;   // 已经访问了n*10个空节点，则这次rehash结束
        }
        de = d->ht[0].table[d->rehashidx];
        // 将老的节点从ht[0]移动到ht[1], 并且插入作为链表的头节点
        while(de) {
            unsigned int h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    // 检查是否已经完成整个哈希表的重哈希
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0; // 全部完成，则让ht[0]等于ht[1], 然后回收ht[1], 并且将rehashidx设为-1，返回0表示重哈希已全部完成
    }

    // 返回1表示重哈希还没有全部完成
    return 1;
}
```
