### Redis底层 哈希表和字典

首先简单介绍几个概念：哈希表（散列表）、映射、冲突、链地址、哈希函数。

 哈希表（Hash table）的初衷是**为了将数据映射到数组中的某个位置**，这样就能够通过数组下标访问该数据，提高数据的查找速度，这样的查找的平均期望时间复杂度是O(1)的。

采用哈希表的话，我们可以只申请一个长度为4的数组，如下图所示：



​    例如四个整数 6、7、9、12 需要映射到数组中，我们可以开一个长度为13（C语言下标从0开始）的数组，然后将对应值放到对应的下标，但是这样做，就会浪费没有被映射到的位置的空间。

![img](https://img-blog.csdn.net/20180628093901157?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1doZXJlSXNIZXJvRnJvbQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)





采用哈希表的话，我们可以只申请一个长度为4的数组，如下图所示：





![img](https://img-blog.csdn.net/20180628094836110?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1doZXJlSXNIZXJvRnJvbQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)



       将每个数的值对数组长度4取模，然后放到对应的数组槽位中，这样就把离散的数据映射到了连续的空间，所以哈希表又称为散列表。这样做，最大限度上提高空间了利用率，并且查找效率还很高。
    
       那么问题来了，如果这四个数据是6、7、8、11呢？继续看图：

![img](https://img-blog.csdn.net/20180628094836110?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1doZXJlSXNIZXJvRnJvbQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

       7 和 11 对4取模的值都是 3，所以占据了同一个槽位，这种情况我们称为冲突 （collision）。一般遇到冲突后，有很多方法解决冲突，包括但不限于 开放地址法、再散列法、链地址法 等等。 Redis采用的是链地址法，所以这里只介绍链地址法，其它的方法如果想了解请自行百度。
    
      链地址法就是将有冲突的数据用一个链表串联起来，如图所示：

![img](https://img-blog.csdn.net/20180628100122405?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1doZXJlSXNIZXJvRnJvbQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

   这样一来，就算有冲突，也可以将有冲突的数据存储在一起了。存储结构需要稍加变化，哈希表的每个元素将变成一个指针，指向数据链表的链表头，每次有新数据来时从链表头插入，可以达到插入的时间复杂度保持O(1)。

再将问题进行变形，如果4个数据是 "are",  "you",  "OK",  "？" 这样的字符串，**如何进行映射呢？没错，我们需要通过一个哈希函数将字符串变成整数，**哈希函数的概念会在接下来详细讲述，这里只需要知道它可以把一个值变成另一个值即可，比如哈希函数f(x)，调用 f("are") 就可以得到一个整数，f("you") 也可以得到一个整数。

一个简易的大小写不敏感的字符串哈希函数如下：

```cc

unsigned int hashFunction(const unsigned char *buf, int len) {
    unsigned int hash = (unsigned int)5381;                       // hash初始种子，实验值
    while (len--)
        hash = ((hash << 5) + hash) + (tolower(*buf++));          // hash * 33 + c
    return hash;


```

 我们看到，**哈希函数的作用就是把*非数字的对象*通过一系列的算法转化成数字（下标），得到的数字可能是哈希表数组无法承载的，所以还需要通过取模才能映射到连续的数组空间中。**对于这个取模，我们知道取模的效率相比位运算来说是很低的，那么有没有什么办法可以把取模用位运算来代替呢？



 **ht** 是两个哈希表，一般情况下，只使用ht[0]，只有当哈希表的键值对数量超过负载(元素过多)时，才会将键值对迁移到ht[1]，这一步迁移被称为 rehash (重哈希)，rehash 会在下文进行详细介绍；



rehash
      千呼万唤始出来，提到了这么多次的 rehash 终于要开讲了。其实没有想象中的那么复杂，随着字典操作的不断执行，哈希表保存的键值对会不断增多（或者减少），为了让哈希表的负载因子维持在一个合理的范围之内，当哈希表保存的键值对数量太多或者太少时，需要对哈希表大小进行扩展或者收缩。

 1、负载因子

   这里提到了一个负载因子，其实就是当前已使用结点数量除上哈希表的大小，即：

```
load_factor = ht[0].used / ht[0].size
```

 2、哈希表扩展

   **1、当哈希表的负载因子大于5时，为 ht[1] 分配空间，大小为第一个大于等于 ht[0].used * 2 的 2 的幂；**

   **2、将保存在 ht[0] 上的键值对 rehash 到 ht[1] 上，rehash 就是重新计算哈希值和索引，并且重新插入到 ht[1] 中，插入一个删除一个；**

   **3、当 ht[0] 包含的所有键值对全部 rehash 到 ht[1] 上后，释放 ht[0] 的控件， 将 ht[1] 设置为 ht[0]，并且在 ht[1] 上新创件一个空的哈希表，为下一次 rehash 做准备；**

   Redis 中 实现哈希表扩展调用的是 dict.c/_dictExpandIfNeeded 函数：



```java
static int _dictExpandIfNeeded(dict *d)
{
    if (dictIsRehashing(d)) return DICT_OK;
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);          // 大小为0需要设置初始哈希表大小为4
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))                 // 负载因子超过5，执行 dictExpand
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;

}
```

  3、哈希表收缩

   哈希表的收缩，同样是为 ht[1] 分配空间， 大小等于 max( ht[0].used, DICT_HT_INITIAL_SIZE )，然后和扩展做同样的处理即可。

## 渐进式rehash

扩展或者收缩哈希表的时候，需要将 ht[0] 里面所有的键值对 rehash 到 ht[1] 里，**当键值对数量非常多的时候，这个操作如果在一帧内完成，大量的计算很可能导致服务器宕机，所以不能一次性完成，需要渐进式的完成。**
       渐进式 rehash 的详细步骤如下：
       1、为 ht[1] 分配指定空间，让字典同时持有 ht[0] 和 ht[1] 两个哈希表；
       2、将 rehashidx 设置为0，表示正式开始 rehash，前两步是在 dict.c/dictExpand 中实现的

```java
int dictExpand(dict *d, unsigned long size)
{
    dictht n;
    unsigned long realsize = _dictNextPower(size);                      // 找到比size大的最小的2的幂
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;
    if (realsize == d->ht[0].size) return DICT_ERR;
 
    n.size = realsize;                                                 // 给ht[1]分配 realsize 的空间
    n.sizemask = realsize-1;
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;
    if (d->ht[0].table == NULL) {                                      // 处于初始化阶段
        d->ht[0] = n;
        return DICT_OK;
    }
    d->ht[1] = n;
    d->rehashidx = 0;                                                  // rehashidx 设置为0，开始渐进式 rehash
    return DICT_OK;
}
```

  3、在进行 rehash 期间，每次对字典执行 增、删、改、查操作时，程序除了执行指定的操作外，还会将 哈希表 ht[0].table中下标为 rehashidx 位置上的所有的键值对 全部迁移到 ht[1].table 上，完成后 rehashidx 自增。这一步就是 rehash 的关键一步。为了防止 ht[0] 是个稀疏表 （遍历很久遇到的都是NULL），从而导致函数阻塞时间太长，这里引入了一个 “最大空格访问数”，也即代码中的 enmty_visits，初始值为 n*10。当遇到NULL的数量超过这个初始值直接返回。

       这一步实现在 dict.c/dictRehash 中：
```java
int dictRehash(dict *d, int n) {
    int empty_visits = n*10;
    if (!dictIsRehashing(d)) return 0;
 
    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;
 
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;                                      // 设置一个空访问数量 为 n*10
        }
        de = d->ht[0].table[d->rehashidx];                                          // dictEntry的迁移
        while(de) {
            unsigned int h;
            nextde = de->next;
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;                                                            // 完成一次 rehash
    }
 
    if (d->ht[0].used == 0) {                                                      // 迁移完毕，rehashdix 置为 -1
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }
    return 1;
}
```

