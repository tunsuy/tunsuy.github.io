# memcache内存分配机制

## Slab Allocator
memcached 默认情况下采用了 Slab Allocator 的机制分配和管理内存. 在该机制出现之前内存分配简单的通过 malloc 和 free 来管理所有的记录, 旧的方式会导致产生很多内存碎片, 加重机器管理内存的负担, 甚至有可能导致操作系统比 memcached 进程本身还慢, Slab Allocator 则解决了该问题. 

 Slab 的基本原理是按照预先规定的大小, 将分配的内存分割成特定长度的块(chunk), 以解决内存碎片的问题. 这也意味着存取记录的时候可以减少内存分配的次数, 有点类似线程池/内存池的感觉. Slab 的原理也比较简单, 是将分配的内存分割成各种尺寸的块(chunk), 且把尺寸相同的 chunk 分成组(chunk 集合), 一个组称为 slab class. Slab class 的主要术语包括以下:
* page: 分配给 Slab 的内存空间, 默认是 1MB, 分配给 slab 之后根据 slab 大小分成 chunk.
* chunk: 用于缓存记录的内存空间.
* slab class: 特定大小的 chunk 的组.

每个chunk的占用的大小不仅是存储的数据的大小，还有chunk的数据结构也需要占用48B
由此可以看出, 三者之间在内存分配上的关系为 slab class -> page -> chunk, 


## Slab Allocator 的缺点
尽管 slab 很好的解决了内存碎片的问题, 但该机制也给 memcached 带来了新的问题. 比如如果chunk 都是固定的长度 120 字节, 将 100 字节存到该 chunk 中之后, 剩余的 20 字节就会被浪费,下面介绍的增长因子则能较好的减少内存浪费.

## 增长因子(growth factor)
```
Item_Size  Max_age   Pages   Count   Full?  Evicted Evict_Time OOM

  1      96B   2327514s       1       2      no        0        0    0
  2     120B   2071695s       3    6981      no        0        0    0
  3     152B   1038377s      23   90100      no        0        0    0
  4     192B         0s       1       0      no        0        0    0
  5     240B   2327378s       1      28      no        0        0    0
  6     304B   2313765s       1       1      no        0        0    0
  7     384B   2326507s       1       3      no        0        0    0
  8     480B   2310076s       1       1      no        0        0    0
  9     600B   2327669s       1       8      no        0        0    0
 10     752B   2327425s       1      47      no        0        0    0
```

从 class 1 的 chunk 大小 96 字节开始, 增长因子为 1.25， 后续的 class 2 的 chunk 大小即为 96*1.25 = 120, class 3 的即为 120*1.25 = 152 等等. 从示例中我们看到, memcached 中的记录都保存在于 class 2 和 class 3 中. 这意味着大部分数据在 96 ~ 152 字节之间. 假如这里有很多 100 字节和 125 字节的记录的话, 可想而知会有很多的内存被浪费掉, 这个时候, 如果我们调节增长因子为 1.1, 则会减少很多内存的浪费
调整为 1.1 的因子后, 一个 100 字节的记录比调整前少浪费了 8 字节, 一个 125 字节的记录比调整前少浪费了 24 字节. 由此可见在内存使用很紧张的情况下, 调整增长因子也能节省相当多的内存.

## Item 缓存数据存储的基本单元
1. Item是Memcached存储的最小单位
2. 每一个缓存都会有自己的一个Item数据结构
3. Item主要存储缓存的key、value、key的长度、value的长度、缓存的时间等信息。
4. HashTable和LRU链表结构都是依赖Item结构中的元素的。
5. 在Memcached中，Item扮演着重要的角色。
```c++
//item的具体结构
typedef struct _stritem {
    //记录下一个item的地址,主要用于LRU链和freelist链
    struct _stritem *next;
    //记录下一个item的地址,主要用于LRU链和freelist链
    struct _stritem *prev;
    //记录HashTable的下一个Item的地址
    struct _stritem *h_next;
    //最近访问的时间，只有set/add/replace等操作才会更新这个字段
    //当执行flush命令的时候，需要用这个时间和执行flush命令的时间相比较，来判断是否失效
    rel_time_t      time;       /* least recent access */
    //缓存的过期时间。设置为0的时候，则永久有效。
    //如果Memcached不能分配新的item的时候，设置为0的item也有可能被LRU淘汰
    rel_time_t      exptime;    /* expire time */
    //value数据大小
    int             nbytes;     /* size of data */
    //引用的次数。通过这个引用的次数，可以判断item是否被其它的线程在操作中。
    //也可以通过refcount来判断当前的item是否可以被删除，只有refcount -1 = 0的时候才能被删除
    unsigned short  refcount;
    uint8_t         nsuffix;    /* length of flags-and-length string */
    uint8_t         it_flags;   /* ITEM_* above */
    //slabs_class的ID。
    uint8_t         slabs_clsid;/* which slab class we're in */
    uint8_t         nkey;       /* key length, w/terminating null and padding */
    /* this odd type prevents type-punning issues when we do
     * the little shuffle to save space when not using CAS. */
    //数据存储结构
    union {
        uint64_t cas;
        char end;
    } data[];
    /* if it_flags & ITEM_CAS we have 8 bytes CAS */
    /* then null-terminated key */
    /* then " flags length\r\n" (no terminating null) */
    /* then data with terminating \r\n (no terminating null; it's binary!) */
} item;
```

## slabclass 划分数据空间
```c++
//slabclass的结构
typedef struct {
	//当前的slabclass存储最大多大的item
    unsigned int size;
    //每一个slab上可以存储多少个item.每个slab大小为1M， 可以存储的item个数根据size决定。
    unsigned int perslab;
 
    //当前slabclass的（空闲item列表）freelist 的链表头部地址
    //freelist的链表是通过item结构中的item->next和item->prev连建立链表结构关系
    void *slots;           /* list of item ptrs */
    //当前总共剩余多少个空闲的item
    //当sl_curr=0的时候，说明已经没有空闲的item，需要分配一个新的slab（每个1M，可以切割成N多个Item结构）
    unsigned int sl_curr;   /* total free items in list */
 
    //总共分配多少个slabs
    unsigned int slabs;     /* how many slabs were allocated for this class */
    //分配的slab链表
    void **slab_list;       /* array of slab pointers */
    unsigned int list_size; /* size of prev array */
 
    unsigned int killing;  /* index+1 of dying slab, or zero if none */
    //总共请求的总bytes
    size_t requested; /* The number of requested bytes */
} slabclass_t;
//定义一个slabclass数组，用于存储最大200个的slabclass_t的结构。
static slabclass_t slabclass[MAX_NUMBER_OF_SLAB_CLASSES];
```

## slabclass初始化
```c++
//slabclass初始化
void slabs_init(const size_t limit, const double factor, const bool prealloc) {
    int i = POWER_SMALLEST - 1;
    unsigned int size = sizeof(item) + settings.chunk_size;
 
    mem_limit = limit;
 
    //这边是否初始化的时候，就给每一个slabclass_t结构分配一个slab内存块
    //默认都会分配
    if (prealloc) {
        /* Allocate everything in a big chunk with malloc */
        mem_base = malloc(mem_limit);
        if (mem_base != NULL) {
            mem_current = mem_base;
            mem_avail = mem_limit;
        } else {
            fprintf(stderr, "Warning: Failed to allocate requested memory in"
                    " one large chunk.\nWill allocate in smaller chunks\n");
        }
    }
 
    memset(slabclass, 0, sizeof(slabclass));
    //factor 默认等于1.25 ，也就是说前一个slabclass允许存储96byte大小的数据，
    //则下一个slabclass可以存储120byte
    while (++i < POWER_LARGEST && size <= settings.item_size_max / factor) {
        /* Make sure items are always n-byte aligned */
        if (size % CHUNK_ALIGN_BYTES)
            size += CHUNK_ALIGN_BYTES - (size % CHUNK_ALIGN_BYTES);
 
        //每个slabclass[i]存储最大多大的item
        slabclass[i].size = size;
        slabclass[i].perslab = settings.item_size_max / slabclass[i].size;
        size *= factor;
        if (settings.verbose > 1) {
            fprintf(stderr, "slab class %3d: chunk size %9u perslab %7u\n",
                    i, slabclass[i].size, slabclass[i].perslab);
        }
    }
 
    power_largest = i;
    slabclass[power_largest].size = settings.item_size_max;
    slabclass[power_largest].perslab = 1;
    if (settings.verbose > 1) {
        fprintf(stderr, "slab class %3d: chunk size %9u perslab %7u\n",
                i, slabclass[i].size, slabclass[i].perslab);
    }
 
    /* for the test suite:  faking of how much we've already malloc'd */
    {
        char *t_initial_malloc = getenv("T_MEMD_INITIAL_MALLOC");
        if (t_initial_malloc) {
            mem_malloced = (size_t)atol(t_initial_malloc);
        }
 
    }
 
    if (prealloc) {
        slabs_preallocate(power_largest);
    }
}

```
根据size值和每个内存页1mb大小算出当前slabclass_t的每个slab能够存放多少个item放到perslab属性上
按照factor这个缩放值增大size(size=size*factor)继续初始化下一个slabclass_t

## 分配一个item
```c++
//分配一个Item
static void *do_slabs_alloc(const size_t size, unsigned int id) {
    slabclass_t *p;
    void *ret = NULL;
    item *it = NULL;
 
    if (id < POWER_SMALLEST || id > power_largest) {
        MEMCACHED_SLABS_ALLOCATE_FAILED(size, 0);
        return NULL;
    }
 
    //获取slabclass
    p = &slabclass[id];
    assert(p->sl_curr == 0 || ((item *)p->slots)->slabs_clsid == 0);
 
    /* fail unless we have space at the end of a recently allocated page,
       we have something on our freelist, or we could allocate a new page */
    //p->sl_curr 说明是否有空闲的item list
    //如果没有空闲的item list，则取分配一个新的slab，如果分配失败，返回NULL
    if (! (p->sl_curr != 0 || do_slabs_newslab(id) != 0)) {
        /* We don't have more memory available */
        ret = NULL;
    //如果有free item lits，则从空闲的列表中取一个Item
    } else if (p->sl_curr != 0) {
        /* return off our freelist */
        it = (item *)p->slots;
        p->slots = it->next;
        if (it->next) it->next->prev = 0;
        p->sl_curr--;
        ret = (void *)it;
    }
 
    if (ret) {
        p->requested += size;
        MEMCACHED_SLABS_ALLOCATE(size, id, p->size, ret);
    } else {
        MEMCACHED_SLABS_ALLOCATE_FAILED(size, id);
    }
 
    return ret;
}
分配一个新的slab：
//分配一块新的item块
static int do_slabs_newslab(const unsigned int id) {
	//获取slabclass
    slabclass_t *p = &slabclass[id];
    //分配一个slab，默认是1M
    //分配的slab也可以根据 该slabclass存储的item的大小 * 可以存储的item的个数 来计算出内存块长度
    int len = settings.slab_reassign ? settings.item_size_max
        : p->size * p->perslab;
    char *ptr;
 
    //这边回去分配一块slab内存块
    if ((mem_limit && mem_malloced + len > mem_limit && p->slabs > 0) ||
        (grow_slab_list(id) == 0) ||
        ((ptr = memory_allocate((size_t)len)) == 0)) {
 
        MEMCACHED_SLABS_SLABCLASS_ALLOCATE_FAILED(id);
        return 0;
    }
 
    //将slab内存内存块切割成N个item，放进freelist中
    memset(ptr, 0, (size_t)len);
    split_slab_page_into_freelist(ptr, id);
 
    p->slab_list[p->slabs++] = ptr;
    mem_malloced += len;
    MEMCACHED_SLABS_SLABCLASS_ALLOCATE(id);
 
    return 1;
}
```
1. Memcached分配一个item，会先检查freelist空闲的列表中是否有空闲的item，如果有的话就用空闲列表中的item。
2. 如果空闲列表没有空闲的item可以分配，则Memcached会去申请一个slab（默认大小为1M）的内存块，如果申请失败，则返回NULL，表明分配失败。
3. 如果申请成功，则会去将这个1M大小的内存块，根据slabclass_t可以存储的最大的item的size，将slab切割成N个item，然后放进freelist（空闲列表中）
4. 然后去freelist（空闲列表）中取出一个item来使用。

## 释放一个item
```c++
//释放一个item
static void do_slabs_free(void *ptr, const size_t size, unsigned int id) {
    slabclass_t *p;
    item *it;
 
    assert(((item *)ptr)->slabs_clsid == 0);
    assert(id >= POWER_SMALLEST && id <= power_largest);
    if (id < POWER_SMALLEST || id > power_largest)
        return;
 
    MEMCACHED_SLABS_FREE(size, id, ptr);
    p = &slabclass[id];
 
    it = (item *)ptr;
    it->it_flags |= ITEM_SLABBED;
    //放进空闲列表  freelist
    it->prev = 0;
    it->next = p->slots;
    if (it->next) it->next->prev = it;
    p->slots = it;
 
    p->sl_curr++;
    p->requested -= size;
    return;
}
```
主要就是把删除的item放进slabclass_t的slots数组中，申请内存时，优先从这个slots中获取，达到这个memcached解决内存碎片的目的