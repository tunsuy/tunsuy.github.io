# memcached过期策略

## 前言
从 Memcached1.5 开始，实现了一个改良的 LRU 算法，也叫做分段 LRU（Segmented LRU）算法，新算法主要是为了更好的利用内存，并提升性能。包含了二个重要的线程:maintainer 线程、crawler 线程。	

### maintainer线程
每个 Slab-class 有一个 LRU，每个 LRU 又由四个子 LRU 组成，每个子 LRU 维护独立的锁（mutex lock），所有的 LRU 由一个独立的线程维护（这和旧的 LRU 算法有很大的不同），称之为 `LRU maintainer` 线程。

每个 item 有一个 flag，存储在其元数据中，标识其活跃程度：
* FETCHED：如果一个 item 有请求操作，其 flag 等于 FETCHED。
* ACTIVE：如果一个 item 第二次被请求则会标记为 ACTIVE；当一个 item 发生 bump 或被移动了，flag 会被清空。
* INACTIVE：不活跃状态。

这四个子 LRU 包含了四个独立的 queue，相关的 queue 可能会迁移到其他的 queue，这么设计就是为了减少 bump 的产生.
* （1）HOT queue：如果一个 item 的过期时间（TTL）很短，会进入该队列，在 HOT queue 中不会发生 bump，如果一个 item 到达了 queue 的 tail，那么会进入到 WARM 队列（如果 item 是 ACTIVE 状态）或者 COLD 队列（如果 item 处于不活跃状态）。
* （2）WARM queue：如果一个 item 不是 FETCHED，永远不会进入这个队列，该队列里面的 item TTL 时间相对较长，这个队列的 lock 竞争会很少。该队列 tail 处的一个 item 如果再一次被访问，会 bump 回到 head，否则移动到 COLD 队列。
* （3）COLD queue：包含了最不活跃的 item，一旦该队列内存满了，该队列 tail 处的 item 会被 evict。
  如果一个 item 被激活了，那么会异步移动到 WARM 队列，如果某个时间段内大量的 COLD item 被激活了，bump 操作可能会处于满负载，这个时候它会什么也不做（不移动到 WARM queue），避免影响工作线程的性能。
  *（4）TEMP queue：该队列中的 item TTL 通常只有几秒，该列队中的 item 永远不会发生 bump，也不会进入其他队列，节省了 CPU 时间，也避免了 lock 竞争。

HOT 和 WARM LAU queue 有内存使用的限制，而 COLD 和 TEMP 队列没有内存使用限制，这主要是为了避免一些不经常使用的 item 长期占据在相对活跃的队列中。

### crawler线程
虽然 LRU Maintainer解决了很多问题，但结合 Memcached 内存分配机制，它还有一些潜在的问题，比如说很难动态调整内存的大小；再比如某些 Slab-class 可能存储了很少的 item（和 item 的大小有关系）；再比如一个空间很大的过期 item 其实可以存储几百个小空间 item；还有 LRU Maintainer 并没有过期 item 回收的功能。

为了解决这些问题，memcached1.5 版本引进了 LRU crawler, 它是一个异步的后台线程，扫描 LRU 中的所有 item，然后回收过期 item，或者检查整个 Slab-class，进行相应的调整。

crawler 在每个 Slab-class 的每个子 LRU 的 tail 部插入一个特别的 crawler items，然后从子 LRU 的 tail 到 head 不断进行扫描，如果发现有过期的 item，就进行回收。

它在给每个子 LRU 进行扫描的时候，会构建一个直方图，通过直方图决定下一次何时扫描，举个例子：
* 假如 Slab-class 1 有 100 万个 item，过期时间都是 0（也就是不过期），那么最多每小时扫描一次（因为再扫描也回收不了多少内存）。
* 假如 Slab-class 5 有 10万个 item，其中 1% 的 item 5分钟后过期，那么 crawler 将智能的在五分钟后再一次扫描，因为能够回收很多内存。

crawler 还有很多的智能调度策略，比如 Slab-class 越高，代表存储的单个 item 空间更大，尽快回收能够释放更多的内存。

结合分段 LRU 机制，crawler 也有很多好的调度策略，比如 HOT queue 如果有很多 item （TTL 较短），那么应该频繁的扫描，同时避免频繁扫描 COLD queue。

这些调度策略都是为了减少不必要的 crawler 工作。

## LRU算法
在LRU高速缓存中，哈希映射使快速访问高速缓存的对象成为可能。 LRU通过标记过期的或所谓的最近最少使用的对象来避免缓存无限增长。接下来，我们从较高的角度来看LRU是如何工作的。

### 什么是LRU
LRU，Least Recently Used 最近最少使用的一种页面置换算法。算法根据数据的历史访问记录的时间来进行淘汰数据，其核心思想是 如果最近没有被访问过，那么将来被访问的概率也比较低，所以被删除的几率就更大

另外，除了 LRU 还有另外两种常用的缓存 页面置换算法：FIFO（先进先出，先来先服务）、LFU（最近最少使用算法，跟 LRU 的区别为 LFU是按照访问次数进行处理 而 LRU是访问时间）

LRU的实现原理比较简单：维护一个链表，INPUT操作的时候如果对应元素在链表已经存在，则把UPDATE后将该元素放到链表顶端，如果不存在则INSERT后将元素放到链表顶端；SELECT操作的后将查询到的元素移动到链表顶端；这样就能确保不常用的数据在链表底端。

memcached的LRU可没有这么简单。

### memcached的LRU
memcached 的 LRU 机制其实不止单纯的 LRU，它是由几种策略组成的一种机制：

* 惰性删除：memcached 一般不主动积极删除过期，当被访问的时候才根据时间判断是否过期

* flush_all： flush 命令专门用来清理所有数据，但是实际代码逻辑中也并不是一次清理了所有数据，一般在申请内存的时候或者查询的时候进行清理，这样保证了效率

* 创建的时候检查： 需要 set/add 的时候，需要申请一个新的 item，这个时候会检查同一个 slabs 里面的过期数据；另外一种情况，当没有内存分配给新的item，memcached 会从 LRU链表的尾部进行释放，即使还没有到 item 的过期时间

* LRU爬虫机制 LRU爬虫机制 实际是由多个爬虫联合组合而成的完整机制：item爬虫、lru爬虫、slab爬虫

* - item爬虫: memcached 是惰性删除机制的，但是如果有些 item 一直未被 get 呢，对应资源就只能一直被占用而无法释放，所以才有启动单独的 辅助线程，独立进行过期item 的清理

  - lru爬虫: 维护每个 slabclass 对应的 HOT_LRU( 热数据 ) 、WARM_LRU( 暖数据 ) 、COLD_LRU ( 冷数据 ) 三个队列，不断的调整三个队列下的item链表，当需要申请一个新的item的时候，如果没有内存可以分配，则从这三个队列里面进行淘汰item，所以需要维护队列数据，保证经常访问的不被淘汰，不经常访问或者过期的item优先被淘汰

  - ```
    - 新的item会被添加至 HOT_LRU 队列头部
    - 超过 HOT_LRU 队列长度阀值的，添加至 COLD_LRU 队列
    - 超过 WARM_LRU 队列长度阀值的，添加至 COLD_LRU 队列
    - 如果 COLD_LRU 队列数据被访问，则转移到 WARM_LRU 队列
    - 如果 HOT_LRU 队列 或者 WARM_LRU 队列 数据被访问，则转移到 WARM_LRU 队列头部
    - 如果内存不够需要淘汰 item，则优先回收 COLD_LRU 队列的内存
    ```


    以上三个队列都有可能 item 被 删除 或者 强制过期 而回收`

  - slab爬虫: 用来维护 slabclass 的空间，举个栗子,我们都知道存储 slabclass -> chunk -> item 的三级概念，每个 slabclass区域 ( slabclass[1] = 96K, slabclass[2] = 120K … ) 存放不同大小的 item，但是如果存储的一直都是 96K 以内的 item，一直存储在 slabclass[1] 这个内存空间，那么就会一直申请 chunk (每次1M ) ，直到内存申请完毕，但是万一后续需要存储 120K 规格的 item，则会出现无法申请内存的情况，那么就不能存储 120K 规格的item，所以 slab爬虫 就是用来处理这一尴尬情况的

  


接下来，我们阅读执行上述LRU操作的相关代码。

## 源码分析

### 惰性删除
```c++
item.c

item *do_item_get(const char *key, const size_t nkey, const uint32_t hv, conn *c, const bool do_update) {
    item *it = assoc_find(key, nkey, hv);
    
	...

    if (it != NULL) {
		was_found = 1;
		if (item_is_flushed(it)) {
		// 是否被 flush 命令标记过，这里具体的 item_is_flushed 函数后续进行介绍
			// LRU 和 hashtable 解绑
			do_item_unlink(it, hv);
			// 如果有 extstore 的话进行外部存储处理
			STORAGE_delete(c->thread->storage, it);
			// item 删除
			do_item_remove(it);
			it = NULL;
			pthread_mutex_lock(&c->thread->stats.mutex);
			c->thread->stats.get_flushed++;
			pthread_mutex_unlock(&c->thread->stats.mutex);
			if (settings.verbose > 2) {
				fprintf(stderr, " -nuked by flush");
			}
			was_found = 2;
		} else if (it->exptime != 0 && it->exptime <= current_time) {
		// 时间过期
			// LRU 和 hashtable 解绑
			do_item_unlink(it, hv);
			// 如果有 extstore 的话进行外部存储处理
			STORAGE_delete(c->thread->storage, it);
			// item 删除
			do_item_remove(it);
			it = NULL;
			pthread_mutex_lock(&c->thread->stats.mutex);
			c->thread->stats.get_expired++;
			pthread_mutex_unlock(&c->thread->stats.mutex);
			if (settings.verbose > 2) {
				fprintf(stderr, " -nuked by expire");
			}
			was_found = 3;
		} else {
		// 即没有过期，也不是 flush_all 命令执行前的数据
			// 更新lru相关队列
			if (do_update) {
				/* We update the hit markers only during fetches.
				 * An item needs to be hit twice overall to be considered
				 * ACTIVE, but only needs a single hit to maintain activity
				 * afterward.
				 * FETCHED tells if an item has ever been active.
				 */
				// 如果设置的lru分段处理，默认 true
				if (settings.lru_segmented) {
					// it->it_flags 在 item 建立以后一般默认存储的 ITEM_CAS，第一次访问标记为 ITEM_FETCHED，第二次置为 ITEM_ACTIVE
					if ((it->it_flags & ITEM_ACTIVE) == 0) {
						if ((it->it_flags & ITEM_FETCHED) == 0) {
							it->it_flags |= ITEM_FETCHED;
						} else {
							it->it_flags |= ITEM_ACTIVE;
							if (ITEM_lruid(it) != COLD_LRU) {
								// 更新 current_time
								do_item_update(it); // bump LA time
							} else if (!lru_bump_async(c->thread->lru_bump_buf, it, hv)) {
								// add flag before async bump to avoid race.
								it->it_flags &= ~ITEM_ACTIVE;
							}
						}
					}
				} else {
					it->it_flags |= ITEM_FETCHED;
					do_item_update(it);
				}
			}
			DEBUG_REFCNT(it, '+');
		} 
	}
}
```

## flush命令
当用户发送一个flush命令的时候，Memcached会将命令之前的所有的缓存都设置为失效。

Memcached不会主动去清除这些item，Memcached会在接受到flush命令的时候，将设置全局参数settings.oldest_live =current_time - 1。然后去调用item_flush_expired方法。

因为设置全局参数item_flush_expired到调用缓存锁方法之间会有一定的时间差，有可能这个过程中，会有新的item在操作。

然后Memcached调用do_item_flush_expired方法，去遍历所有的LRU链表。do_item_flush_expired不会将每一个在flush命令前的Item删除，因为这样会非常耗时，而是删除在设置全局变量到加上缓存锁这之间操作的item。这样就能加快flush的速度。

```c++
memcached.c

...

} else if (ntokens >= 2 && ntokens <= 4 && (strcmp(tokens[COMMAND_TOKEN].value, "flush_all") == 0)) {
    time_t exptime = 0;
    rel_time_t new_oldest = 0;
    set_noreply_maybe(c, tokens, ntokens);
    // 常规统计
    pthread_mutex_lock(&c->thread->stats.mutex);
    c->thread->stats.flush_cmds++;
    pthread_mutex_unlock(&c->thread->stats.mutex);
    if (!settings.flush_enabled) {
        // flush_all is not allowed but we log it on stats
        out_string(c, "CLIENT_ERROR flush_all not allowed");
        return;
    }
    // 获取 flush 命令后面的过期时间参数 exptime
    if (ntokens != (c->noreply ? 3 : 2)) {
        exptime = strtol(tokens[1].value, NULL, 10);
        if(errno == ERANGE) {
            out_string(c, "CLIENT_ERROR bad command line format");
            return;
        }
    }
    /*
      If exptime is zero realtime() would return zero too, and
      realtime(exptime) - 1 would overflow to the max unsigned
      value.  So we process exptime == 0 the same way we do when
      no delay is given at all.
    */
    // 如果有过期时间参数则根据参数计算时间，反之取当前时间
    if (exptime > 0) {
        new_oldest = realtime(exptime);
    } else { /* exptime == 0 */
        new_oldest = current_time;
    }
    // 确定过期时间点，一般会在预定时间点的基础上减一
    if (settings.use_cas) {
        settings.oldest_live = new_oldest - 1;
        if (settings.oldest_live <= current_time)
            settings.oldest_cas = get_cas_id();
    } else {
        settings.oldest_live = new_oldest;
    }
    out_string(c, "OK");
    return;
}
```

## 分配Item的时候去检查
Memcached在分配一个新的Item。步骤如下：
* 1. 先检查缓存存储空间大小。前几章我们讲到，memcached的命令中会将key的长度和value的长度带上，这样就可以计算出item总的占用空间的大小。
* 2. 通过缓存item的存储空间大小，就可以找到slabs class和slabs class的LRU双向链表。
* 3. 开始尝试分配内存，尝试次数为5次。
* 4. 尝试分配内存的过程中，会从LRU链表的尾部开始搜索，检查ITEM状态，如果item内容为空或者item被其它worker引用锁定等情况，则继续往LRU列表尾部搜索。
* 5. 如果尝试了5次，从LRU尾部搜索都没有找到符合预期的ITEM，则会slabs_alloc方法，申请创建一个新的内存块。
* 6. 如果从LRU尾部搜索找到符合预期的ITEM（没有锁定和有数据），首先会检查ITEM是否已经过了有效期，如果已经过了有效期，则将这个ITEM淘汰，占用该ITEM。
* 7. 如果ITEM还是有效的，则使用slabs_alloc分配一个新的ITEM，分配成功，则就用最新分配的ITEM
* 8. 如果使用slabs_alloc分配一个新的ITEM，分配失败，若开启了不使用LRU强制淘汰，返回ERROR；如果开启了强制淘汰，会将当前LRU链表尾部搜索到的ITEM强制进行淘汰（如果ITEM有效期还在或者设置了永久的也会被淘汰）

```c++
item.c

int lru_pull_tail(const int orig_id, const int cur_lru,
        const uint64_t total_bytes, const uint8_t flags, const rel_time_t max_age,
        struct lru_pull_tail_return *ret_it) {
    item *it = NULL;
    int id = orig_id;
    int removed = 0;
    if (id == 0)
        return 0;
    // 5次尝试机会
    int tries = 5;
    item *search;
    item *next_it;
    void *hold_lock = NULL;
    unsigned int move_to_lru = 0;
    uint64_t limit = 0;
    // 通过 slab id 计算出当前 lru队列 id
    id |= cur_lru;
    // lru队列 加锁
    pthread_mutex_lock(&lru_locks[id]);
    // 当前队列中的最后一位
    search = tails[id];
    /* We walk up *only* for locked items, and if bottom is expired. */
    for (; tries > 0 && search != NULL; tries--, search=next_it) {
        /* we might relink search mid-loop, so search->prev isn't reliable */
        // 底端法查找
        next_it = search->prev;
        // 这是一个爬虫item，直接跳过，不计入 tries 的次数，所以 tries++
        if (search->nbytes == 0 && search->nkey == 0 && search->it_flags == 1) {
            /* We are a crawler, ignore it. */
            if (flags & LRU_PULL_CRAWL_BLOCKS) {
                pthread_mutex_unlock(&lru_locks[id]);
                return 0;
            }
            tries++;
            continue;
        }
        // 计算 item 的 hv 值，用于计算所处 hashtable 位置
        uint32_t hv = hash(ITEM_key(search), search->nkey);
        /* Attempt to hash item lock the "search" item. If locked, no
         * other callers can incr the refcount. Also skip ourselves. */
        // 尝试抢占锁，如果抢占失败，证明有其他锁事务，继续寻找
        if ((hold_lock = item_trylock(hv)) == NULL)
            continue;
        /* Now see if the item is refcount locked */
        // 引用次数 >= 3，证明已经有线程正在使用则认怂
        if (refcount_incr(search) != 2) {
            /* Note pathological case with ref'ed items in tail.
             * Can still unlink the item, but it won't be reusable yet */
            // item 被其他线程引用的次数 加1
            itemstats[id].lrutail_reflocked++;
            /* In case of refcount leaks, enable for quick workaround. */
            /* WARNING: This can cause terrible corruption */
            // item 是否出现异常线程引用错误，tail_repair_time 是用来设置是否检测引用错误的配置(单位是秒)，对 item 最后一次访问时间进行判断，如果异常引用，则删除这个 item
            if (settings.tail_repair_time &&
                    search->time + settings.tail_repair_time < current_time) {
                // item统计中 需要修复的次数 +1
                itemstats[id].tailrepairs++;
                // 引用 = 1
                search->refcount = 1;
                /* This will call item_remove -> item_free since refcnt is 1 */
                STORAGE_delete(ext_storage, search);
                // 删除
                do_item_unlink_nolock(search, hv);
                // 解除占用锁（如果有的话）
                item_trylock_unlock(hold_lock);
                continue;
            }
        }
        /* Expired or flushed */
        // item 过期了或者根据 item_is_flushed 判断是否执行过 flush 命令（实际是判断 settings.oldest_live 变量）
        if ((search->exptime != 0 && search->exptime < current_time)
            || item_is_flushed(search)) {
            // item 被重复使用的次数 +1
            itemstats[id].reclaimed++;
            if ((search->it_flags & ITEM_FETCHED) == 0) {
                // 被访问过并且超时的item 个数 +1
                itemstats[id].expired_unfetched++;
            }
            /* refcnt 2 -> 1 */
            // 删除
            do_item_unlink_nolock(search, hv);
            STORAGE_delete(ext_storage, search);
            /* refcnt 1 -> 0 -> item_free */
            // item free掉
            do_item_remove(search);
            // 解除占用锁（如果有的话）
            item_trylock_unlock(hold_lock);
            removed++;
            /* If all we're finding are expired, can keep going */
            continue;
        }
        /* If we're HOT_LRU or WARM_LRU and over size limit, send to COLD_LRU.
         * If we're COLD_LRU, send to WARM_LRU unless we need to evict
         */
        // 如果当前为 HOT_LRU 或者 WARM_LRU 列表，则更新列表尺寸，如果为 COLD_LRU，除非是需要 踢出去，则更新至 WARM_LRU 列表
        switch (cur_lru) {
            case HOT_LRU:
                // HOT_LRU队列的 item 比例阀值
                limit = total_bytes * settings.hot_lru_pct / 100;
            case WARM_LRU:
                if (limit == 0)
                    // WARM_LRU队列的 item 比例阀值
                    limit = total_bytes * settings.warm_lru_pct / 100;
                /* Rescue ACTIVE items aggressively */
                // 如果item为 ITEM_ACTIVE 活动状态
                if ((search->it_flags & ITEM_ACTIVE) != 0) {
                    // 更新为 ITEM_FETCHED 状态
                    search->it_flags &= ~ITEM_ACTIVE;
                    // 移动+1
                    removed++;
                    // 如果当前在 WARM_LRU 队列
                    if (cur_lru == WARM_LRU) {
                        // 统计
                        itemstats[id].moves_within_lru++;
                        // 更新current_time，同时将item移动到 WARM_LRU 队列头部，当然头部也可以避免因为阀值达到了而强制剔除
                        do_item_update_nolock(search);
                        // 引用 -1，之前引用 +1 了，所以这里不会被释放掉
                        do_item_remove(search);
                        item_trylock_unlock(hold_lock);
                    } else {
                        /* Active HOT_LRU items flow to WARM */
                        // 如果是在 HOT_LRU 队列中，则移动到 WARM_LRU 队列头部
                        itemstats[id].moves_to_warm++;
                        move_to_lru = WARM_LRU;
                        do_item_unlink_q(search);
                        it = search;
                    }
                } else if (sizes_bytes[id] > limit ||
                           current_time - search->time > max_age) {
                    // 如果 WARM_LRU队列的 阀值已经超过了，则移动到 COLD_LRU队列 中
                    itemstats[id].moves_to_cold++;
                    move_to_lru = COLD_LRU;
                    do_item_unlink_q(search);
                    it = search;
                    removed++;
                    break;
                } else {
                    /* Don't want to move to COLD, not active, bail out */
                    it = search;
                }
                break;
            case COLD_LRU:
                it = search; /* No matter what, we're stopping */
                if (flags & LRU_PULL_EVICT) {
                // 强制剔除模式，这个只有在 do_item_alloc_pull 里面 item 不能通过正常 slabs_alloc 申请的时候，进行 剔除
                    if (settings.evict_to_free == 0) {
                        /* Don't think we need a counter for this. It'll OOM.  */
                        break;
                    }
                    // 统计
                    itemstats[id].evicted++;
                    itemstats[id].evicted_time = current_time - search->time;
                    if (search->exptime != 0)
                        itemstats[id].evicted_nonzero++;
                    if ((search->it_flags & ITEM_FETCHED) == 0) {
                        itemstats[id].evicted_unfetched++;
                    }
                    if ((search->it_flags & ITEM_ACTIVE)) {
                        itemstats[id].evicted_active++;
                    }
                    LOGGER_LOG(NULL, LOG_EVICTIONS, LOGGER_EVICTION, search);
                    // 存储介质删除
                    STORAGE_delete(ext_storage, search);
                    // 删除
                    do_item_unlink_nolock(search, hv);
                    removed++;
                    if (settings.slab_automove == 2) {
                        // 如果配置允许则 执行回收 slab 逻辑
                        slabs_reassign(-1, orig_id);
                    }
                } else if (flags & LRU_PULL_RETURN_ITEM) {
                // LRU_PULL_RETURN_ITEM 这个状态传递 只有外部存储介质的时候使用
                    /* Keep a reference to this item and return it. */
                    ret_it->it = it;
                    ret_it->hv = hv;
                } else if ((search->it_flags & ITEM_ACTIVE) != 0
                        && settings.lru_segmented) {
                // 如果为活跃资源，移动至 WARM_LRU 队列
                    itemstats[id].moves_to_warm++;
                    search->it_flags &= ~ITEM_ACTIVE;
                    move_to_lru = WARM_LRU;
                    do_item_unlink_q(search);
                    removed++;
                }
                break;
            case TEMP_LRU:
                it = search; /* Kill the loop. Parent only interested in reclaims */
                break;
        }
        if (it != NULL)
            break;
    }
    pthread_mutex_unlock(&lru_locks[id]);
    if (it != NULL) {
        // 执行 LRU 队列移动逻辑
        if (move_to_lru) {
            it->slabs_clsid = ITEM_clsid(it);
            it->slabs_clsid |= move_to_lru;
            item_link_q(it);
        }
        // LRU_PULL_RETURN_ITEM 这个状态传递 只有外部存储介质的时候使用，直接原样返回，这里 remove 的时候 引用 -1，之前引用 +1 了，所以这里不会被释放掉
        if ((flags & LRU_PULL_RETURN_ITEM) == 0) {
            do_item_remove(it);
            item_trylock_unlock(hold_lock);
        }
    }
    return removed;
}
```

## LRU爬虫
当前版本 LRU爬虫机制 item爬虫、lru爬虫、slab爬虫 是默认开启的，由 main主线程 初始化逻辑中进行控制

```c++
memcached.c

// 启动 item爬虫 线程
if (start_lru_crawler && start_item_crawler_thread() != 0) {
    fprintf(stderr, "Failed to enable LRU crawler thread\n");
    exit(EXIT_FAILURE);
}
#ifdef EXTSTORE
// ....省略了一部分代码
#else
// 启动 lru爬虫 线程
if (start_lru_maintainer && start_lru_maintainer_thread(NULL) != 0) {
#endif
    fprintf(stderr, "Failed to enable LRU maintainer thread\n");
    return 1;
}
// 启动 slab爬虫 线程
if (settings.slab_reassign &&
    start_slab_maintenance_thread() == -1) {
    exit(EXIT_FAILURE);
}
```
而实际中 虽然 爬虫线程都被启动，但是刚开始处于挂起状态，等待激活，启动线程的方法比较简单我们就不进行查看了，我们更多关注它们分别的主要控制逻辑：item_crawler_thread、lru_maintainer_thread、slab_rebalance_thread

刚才我们说起 这三个 爬虫线程实际都是处于 挂起状态，而这个激活信号就是 LRU维护线程函数里面处理逻辑的时候根据一些条件触发的，也可以说相当于有 LRU线程去 统一调度，所以我们先来看看 lru_maintainer_thread LRU维护线程的逻辑：
```c++
item.c

static void *lru_maintainer_thread(void *arg) {
    // slab 自动回收配置的 初始方法
    slab_automove_reg_t *sam = &slab_automove_default;
#ifdef EXTSTORE
    void *storage = arg;
    if (storage != NULL)
        sam = &slab_automove_extstore;
    int x;
#endif
    int i;
    // 循环休眠时间设置
    useconds_t to_sleep = MIN_LRU_MAINTAINER_SLEEP;
    useconds_t last_sleep = MIN_LRU_MAINTAINER_SLEEP;
    rel_time_t last_crawler_check = 0;
    rel_time_t last_automove_check = 0;
    useconds_t next_juggles[MAX_NUMBER_OF_SLAB_CLASSES] = {0};
    useconds_t backoff_juggles[MAX_NUMBER_OF_SLAB_CLASSES] = {0};
    struct crawler_expired_data *cdata =
        calloc(1, sizeof(struct crawler_expired_data));
    if (cdata == NULL) {
        fprintf(stderr, "Failed to allocate crawler data for LRU maintainer thread\n");
        abort();
    }
    pthread_mutex_init(&cdata->lock, NULL);
    cdata->crawl_complete = true; // kick off the crawler.
    logger *l = logger_create();
    if (l == NULL) {
        fprintf(stderr, "Failed to allocate logger for LRU maintainer thread\n");
        abort();
    }
    double last_ratio = settings.slab_automove_ratio;
    // 初始化自动迁移逻辑
    void *am = sam->init(&settings);
    pthread_mutex_lock(&lru_maintainer_lock);
    if (settings.verbose > 2)
        fprintf(stderr, "Starting LRU maintainer background thread\n");
    while (do_run_lru_maintainer_thread) {
        pthread_mutex_unlock(&lru_maintainer_lock);
        if (to_sleep)
            usleep(to_sleep);
        pthread_mutex_lock(&lru_maintainer_lock);
        /* A sleep of zero counts as a minimum of a 1ms wait */
        last_sleep = to_sleep > 1000 ? to_sleep : 1000;
        to_sleep = MAX_LRU_MAINTAINER_SLEEP;
        STATS_LOCK();
        stats.lru_maintainer_juggles++;
        STATS_UNLOCK();
        /* Each slab class gets its own sleep to avoid hammering locks */
        // 循环每一个 slab
        for (i = POWER_SMALLEST; i < MAX_NUMBER_OF_SLAB_CLASSES; i++) {
            // next_juggles 变量的意义在于 在后续的运算逻辑中根据获取的移除数量确定下次while循环的延迟时间
            next_juggles[i] = next_juggles[i] > last_sleep ? next_juggles[i] - last_sleep : 0;
            if (next_juggles[i] > 0) {
                // Sleep the thread just for the minimum amount (or not at all)
                if (next_juggles[i] < to_sleep)
                    to_sleep = next_juggles[i];
                continue;
            }
            // HOT_LRU、WARM_LRU、COLD_LRU 列表的维护，具体逻辑可以查看下方 lru_maintainer_juggle 函数
            int did_moves = lru_maintainer_juggle(i);
#ifdef EXTSTORE
            // Deeper loop to speed up pushing to storage.
            if (storage) {
                for (x = 0; x < 500; x++) {
                    int found;
                    found = lru_maintainer_store(storage, i);
                    if (found) {
                        did_moves += found;
                    } else {
                        break;
                    }
                }
            }
#endif
            // 根据转移个数计算 while 下次循环的 延迟时间
            if (did_moves == 0) {
                if (backoff_juggles[i] != 0) {
                    backoff_juggles[i] += backoff_juggles[i] / 8;
                } else {
                    backoff_juggles[i] = MIN_LRU_MAINTAINER_SLEEP;
                }
                if (backoff_juggles[i] > MAX_LRU_MAINTAINER_SLEEP)
                    backoff_juggles[i] = MAX_LRU_MAINTAINER_SLEEP;
            } else if (backoff_juggles[i] > 0) {
                backoff_juggles[i] /= 2;
                if (backoff_juggles[i] < MIN_LRU_MAINTAINER_SLEEP) {
                    backoff_juggles[i] = 0;
                }
            }
            next_juggles[i] = backoff_juggles[i];
            if (next_juggles[i] < to_sleep)
                to_sleep = next_juggles[i];
        }
        /* Minimize the sleep if we had async LRU bumps to process */
        // 优化内存空间，lru_maintainer_bumps 的作用在于 当队列从零到非零的时候可能需要微调队列数据，特别是 COLD_LRU 队列，以节省LRU维护线程的额外内存消耗
        if (settings.lru_segmented && lru_maintainer_bumps() && to_sleep > 1000) {
            to_sleep = 1000;
        }
        /* Once per second at most */
        // 判断是否开启了item爬虫线程
        if (settings.lru_crawler && last_crawler_check != current_time) {
            // 如果开启了则调用该函数执行,判断是否符合触发item爬虫线程条件
            // 如果符合条件则触发信号
            lru_maintainer_crawler_check(cdata, l);
            last_crawler_check = current_time;
        }
        // item 内存自动调整判断，默认 settings 开启，这部分逻辑主要用于调用 slabs_reassign 函数 启动 slab 线程
        if (settings.slab_automove == 1 && last_automove_check != current_time) {
            // 如果调整比例跟自定义配置不一致，则重新初始化
            if (last_ratio != settings.slab_automove_ratio) {
                sam->free(am);
                am = sam->init(&settings);
                last_ratio = settings.slab_automove_ratio;
            }
            int src, dst;
            // 这里为执行 slab_automove_run 函数，slab_automove_run 旨在选出 最贫穷和最富有的 slab，判断是否需要进行 slab 自从划分处理
            sam->run(am, &src, &dst);
            // 当 最贫穷 和 最富有 的slab 都存在的时候，执行 slab爬虫进行 slab自动划分逻辑
            if (src != -1 && dst != -1) {
                // 设置需要进行内存页转移的源和目的slab-class，并唤醒rebalance线程进行内存页转移操作
                slabs_reassign(src, dst);
                LOGGER_LOG(l, LOG_SYSEVENTS, LOGGER_SLAB_MOVE, NULL,
                        src, dst);
            }
            // dst == 0 means reclaim to global pool, be more aggressive
            // 等于0 代表成功，需要 休眠一段时间，不能太过频繁操作，而不成功则记录下时间点，进入下一个 逻辑循环再进行处理
            if (dst != 0) {
                last_automove_check = current_time;
            } else if (dst == 0) {
                // also ensure we minimize the thread sleep
                to_sleep = 1000;
            }
        }
    }
    pthread_mutex_unlock(&lru_maintainer_lock);
    sam->free(am);
    // LRU crawler *must* be stopped.
    free(cdata);
    if (settings.verbose > 2)
        fprintf(stderr, "LRU maintainer thread stopping\n");
    return NULL;
}
```
lru_maintainer_juggle: HOT_LRU、WARM_LRU、COLD_LRU 列表的维护函数
```c++
items.c

static int lru_maintainer_juggle(const int slabs_clsid) {
    int i;
    int did_moves = 0;
    uint64_t total_bytes = 0;
    unsigned int chunks_perslab = 0;
    //unsigned int chunks_free = 0;
    /* TODO: if free_chunks below high watermark, increase aggressiveness */
    // 获取指定id的slabs的使用情况
    slabs_available_chunks(slabs_clsid, NULL,
            &total_bytes, &chunks_perslab);
    // 如果开启了 temp_lru 队列，则进行维护，但是默认关闭的
    if (settings.temp_lru) {
        /* Only looking for reclaims. Run before we size the LRU. */
        for (i = 0; i < 500; i++) {
            if (lru_pull_tail(slabs_clsid, TEMP_LRU, 0, 0, 0, NULL) <= 0) {
                break;
            } else {
                did_moves++;
            }
        }
        total_bytes -= temp_lru_size(slabs_clsid);
    }
    // item 存在于队列中的时长，根据源码逻辑，目前一般起作用于 WARM_LRU 里item 呆了太久，就移动到 COLD_LRU 中，所以这里都是以 cold_age 为基准计算的
    rel_time_t cold_age = 0;
    rel_time_t hot_age = 0;
    rel_time_t warm_age = 0;
    /* If LRU is in flat mode, force items to drain into COLD via max age */
    if (settings.lru_segmented) {
        pthread_mutex_lock(&lru_locks[slabs_clsid|COLD_LRU]);
        if (tails[slabs_clsid|COLD_LRU]) {
            cold_age = current_time - tails[slabs_clsid|COLD_LRU]->time;
        }
        pthread_mutex_unlock(&lru_locks[slabs_clsid|COLD_LRU]);
        hot_age = cold_age * settings.hot_max_factor;
        warm_age = cold_age * settings.warm_max_factor;
    }
    /* Juggle HOT/WARM up to N times */
    for (i = 0; i < 500; i++) {
        int do_more = 0;
        // 转移 HOT_LRU、WARM_LRU 队列中 item，具体逻辑可以查看之前的 lru_pull_tail 函数
        if (lru_pull_tail(slabs_clsid, HOT_LRU, total_bytes, LRU_PULL_CRAWL_BLOCKS, hot_age, NULL) ||
            lru_pull_tail(slabs_clsid, WARM_LRU, total_bytes, LRU_PULL_CRAWL_BLOCKS, warm_age, NULL)) {
            do_more++;
        }
        if (settings.lru_segmented) {
            // 转移 COLD_LRU 队列中的 item，具体逻辑可以查看之前的 lru_pull_tail 函数
            do_more += lru_pull_tail(slabs_clsid, COLD_LRU, total_bytes, LRU_PULL_CRAWL_BLOCKS, 0, NULL);
        }
        if (do_more == 0)
            break;
        did_moves++;
    }
    return did_moves;
}
```

lru_maintainer_crawler_check 实际就相当于开始 item爬虫逻辑了，检查每个slab lass中的 lru 是否需要进行爬虫操作，如果是则将需要进行爬虫的 slab class 的 id 等信息传递给 lru_crawler_start 用来启动爬虫功能
```c++
item.c

static void lru_maintainer_crawler_check(struct crawler_expired_data *cdata, logger *l) {
    int i;
    // 状态记录
    static rel_time_t next_crawls[POWER_LARGEST];
    static rel_time_t next_crawl_wait[POWER_LARGEST];
    uint8_t todo[POWER_LARGEST];
    memset(todo, 0, sizeof(uint8_t) * POWER_LARGEST);
    bool do_run = false;
    unsigned int tocrawl_limit = 0;
    // TODO: If not segmented LRU, skip non-cold
    
    for (i = POWER_SMALLEST; i < POWER_LARGEST; i++) {
        // 爬虫状态
        crawlerstats_t *s = &cdata->crawlerstats[i];
        /* We've not successfully kicked off a crawl yet. */
        // 如果该 LRU 下的爬虫爬取完毕,会将s->run_complete设置为true，而里面的代码就是确认该 LRU 是否可以重新再添加爬虫，但是第一次进入这个逻辑的时候全是 false，通过后面的操作唤起 item 爬虫，item_crawler_thread 将这个变量置为 true
        if (s->run_complete) {
            char *lru_name = "na";
            pthread_mutex_lock(&cdata->lock);
            int x;
            /* Should we crawl again? */
            // 检查的未过期的item数量
            // seen 未过期的item数量，noexp 永不过期的 item数量
            uint64_t possible_reclaims = s->seen - s->noexp;
            uint64_t available_reclaims = 0;
            /* Need to think we can free at least 1% of the items before
             * crawling. */
            /* FIXME: Configurable? */
            // 未过期item百分比
            uint64_t low_watermark = (possible_reclaims / 100) + 1;
            // 当前时间距离爬虫结束时间有多少秒
            rel_time_t since_run = current_time - s->end_time;
            /* Don't bother if the payoff is too low. */
            for (x = 0; x < 60; x++) {
                // 计算当前时间距离结束时间内有多少个item将要过期
                available_reclaims += s->histo[x];
                // 判断过期的item数量是否大于未过期的item数量百分之一
                // 这里使用 x * 60 的原因就是保证时间控制在整60s、120s、180s、240s等 60s整数，如果不符合则减去60s，下次运行
                if (available_reclaims > low_watermark) { 
                    if (next_crawl_wait[i] < (x * 60)) {
                        next_crawl_wait[i] += 60;
                    } else if (next_crawl_wait[i] >= 60) {
                        next_crawl_wait[i] -= 60;
                    }
                    break;
                }
            }
            // 最低延迟60
            if (available_reclaims == 0) {
                next_crawl_wait[i] += 60;
            }
            // 限制最大延迟
            if (next_crawl_wait[i] > MAX_MAINTCRAWL_WAIT) {
                next_crawl_wait[i] = MAX_MAINTCRAWL_WAIT;
            }
            // 计算出下个运行的绝对时间
            next_crawls[i] = current_time + next_crawl_wait[i] + 5;
            // LRU队列类型，用于日志
            switch (GET_LRU(i)) {
                case HOT_LRU:
                    lru_name = "hot";
                    break;
                case WARM_LRU:
                    lru_name = "warm";
                    break;
                case COLD_LRU:
                    lru_name = "cold";
                    break;
                case TEMP_LRU:
                    lru_name = "temp";
                    break;
            }
            LOGGER_LOG(l, LOG_SYSEVENTS, LOGGER_CRAWLER_STATUS, NULL,
                    CLEAR_LRU(i),
                    lru_name,
                    (unsigned long long)low_watermark,
                    (unsigned long long)available_reclaims,
                    (unsigned int)since_run,
                    next_crawls[i] - current_time,
                    s->end_time - s->start_time,
                    s->seen,
                    s->reclaimed);
            // Got our calculation, avoid running until next actual run.
            s->run_complete = false;
            pthread_mutex_unlock(&cdata->lock);
        }
        // 如果当前时间符合下次运行时间
        if (current_time > next_crawls[i]) {
            pthread_mutex_lock(&lru_locks[i]);
            // 每个LRU的item数量是否大于爬虫限制，取最大item数量值
            if (sizes[i] > tocrawl_limit) {
                tocrawl_limit = sizes[i];
            }
            pthread_mutex_unlock(&lru_locks[i]);
            todo[i] = 1;
            do_run = true;
            // 最少延迟 5
            next_crawls[i] = current_time + 5; // minimum retry wait.
        }
    }
    // 如果有任何一个符合运行时间条件即运行
    if (do_run) {
        // 如果有设置 LRU爬虫队列大小则按照配置来，如果没有则 按照最大 item 数量
        if (settings.lru_crawler_tocrawl && settings.lru_crawler_tocrawl < tocrawl_limit) {
            tocrawl_limit = settings.lru_crawler_tocrawl;
        }
        // 开启 item 爬虫运行逻辑
        lru_crawler_start(todo, tocrawl_limit, CRAWLER_AUTOEXPIRE, cdata, NULL, 0);
    }
}
```

lru_crawler_start 函数，为需要进行爬虫操作的 slab 的每个 LRU 链表 后插入用于 爬虫的item,　这个 爬虫item 的主要目的是记录当前在 LRU链表 中的位置，并检查 item 是否已经过期或者已经被 flushed，如果是则将其删除，这就是所谓的惰性删除。函数的最后是唤醒用于爬虫的线程 item_crawler_thread
```c++
crawler.c

int lru_crawler_start(uint8_t *ids, uint32_t remaining,
                             const enum crawler_run_type type, void *data,
                             void *c, const int sfd) {
    int starts = 0;
    bool is_running;
    static rel_time_t block_ae_until = 0;
    pthread_mutex_lock(&lru_crawler_lock);
    STATS_LOCK();
    // 是否运行状态
    is_running = stats_state.lru_crawler_running;
    STATS_UNLOCK();
    // 如果是运行状态，阻止时间往后推
    if (is_running &&
            !(type == CRAWLER_AUTOEXPIRE && active_crawler_type == CRAWLER_AUTOEXPIRE)) {
        pthread_mutex_unlock(&lru_crawler_lock);
        block_ae_until = current_time + 60;
        return -1;
    }
    if (type == CRAWLER_AUTOEXPIRE && block_ae_until > current_time) {
        pthread_mutex_unlock(&lru_crawler_lock);
        return -1;
    }
    /* Configure the module */
    if (!is_running) {
        assert(crawler_mod_regs[type] != NULL);
        // 配置爬虫激活模块，默认 type 为 CRAWLER_AUTOEXPIRE，自动过期
        active_crawler_mod.mod = crawler_mod_regs[type];
        active_crawler_type = type;
        if (active_crawler_mod.mod->init != NULL) {
            // 调用 crawler_expired_init 函数，主要为重置 start_time 以及 run_complete 初始为 false
            active_crawler_mod.mod->init(&active_crawler_mod, data);
        }
        if (active_crawler_mod.mod->needs_client) {
            if (c == NULL || sfd == 0) {
                pthread_mutex_unlock(&lru_crawler_lock);
                return -2;
            }
            if (lru_crawler_set_client(&active_crawler_mod, c, sfd) != 0) {
                pthread_mutex_unlock(&lru_crawler_lock);
                return -2;
            }
        }
    }
    /* we allow the autocrawler to restart sub-LRU's before completion */
    for (int sid = POWER_SMALLEST; sid < POWER_LARGEST; sid++) {
        // 判断是否为需要运行的id，这个在 lru_maintainer_crawler_check 函数里面为 todo 变量控制
        if (ids[sid])
            // do_lru_crawler_start 主要为初始化爬虫信息，并判断开启每个LRU的爬虫状态，返回 开启的个数，而且这个函数里面会调用 do_item_linktail_q函数 对LRU队列安装 item爬虫（伪item，跟item结构体基本相同）
            starts += do_lru_crawler_start(sid, remaining);
    }
    if (starts) {
        // 唤醒 item 爬虫线程
        pthread_cond_signal(&lru_crawler_cond);
    }
    pthread_mutex_unlock(&lru_crawler_lock);
    return starts;
}
```

item_crawler_thread 对所有需要进行爬虫的 LRU 依次进行爬虫，每次检查当前 LRU 中的一个 item，接着检查下一个LRU。但是为了防止爬虫时间过长影响正常的处理，每处理一个 item，则将相关的锁释放，源码中会每处理 crawls_persleep 个 item 就进入睡眠。当所有的 LRU 都处理完毕，则爬虫线程将会进入睡眠，等待 lru_maintainer_thread 线程将其唤醒

item 爬虫的做法比较经典 对每个需要爬虫的 LRU队列安装一个 item爬虫，item爬虫 和 item结构体 除了最后一个属性 remaining 其他都一样，所以二者可以互换，不影响原有的 item 属性值。这个是为了解决一个问题： 每次对 LRU 进行爬取的时候，由于 LRU 队列仅仅是一个链表队列，不支持随机存储，每次访问一个 item 都必须从 head 开始，一个一个访问，所以有了这个 item爬虫，下次对 对应LRU进行爬取的时候，就可以从这个 item爬虫结点 进行继续往下走
```c++
crawler.c

static void *item_crawler_thread(void *arg) {
    int i;
    // 爬虫休眠时间
    int crawls_persleep = settings.crawls_persleep;
    pthread_mutex_lock(&lru_crawler_lock);
    pthread_cond_signal(&lru_crawler_cond);
    settings.lru_crawler = true;
    if (settings.verbose > 2)
        fprintf(stderr, "Starting LRU crawler background thread\n");
    // 挂起等待 LRU爬虫线程 进行唤醒
    while (do_run_lru_crawler_thread) {
    pthread_cond_wait(&lru_crawler_cond, &lru_crawler_lock);
    // crawler_count 开启=1，完成后赋为0
    while (crawler_count) {
        item *search = NULL;
        void *hold_lock = NULL;
        // 循环所有所有 LRU队列
        for (i = POWER_SMALLEST; i < LARGEST_ID; i++) {
            // 是否开启爬虫，it_flag = 1 开启
            if (crawlers[i].it_flags != 1) {
                continue;
            }
            /* Get memory from bipbuf, if client has no space, flush. */
            if (active_crawler_mod.c.c != NULL) {
                int ret = lru_crawler_client_getbuf(&active_crawler_mod.c);
                if (ret != 0) {
                    lru_crawler_class_done(i);
                    continue;
                }
            } else if (active_crawler_mod.mod->needs_client) {
                lru_crawler_class_done(i);
                continue;
            }
            pthread_mutex_lock(&lru_locks[i]);
            // 移动 爬虫item，就是把当前 爬虫item 往上移动一位，然后把 爬虫item 下面的 item 返回
            // item_1 -> item_2 -> crawler_item
            // item_1 -> crawler_item -> item_2
            search = do_item_crawl_q((item *)&crawlers[i]);
            // 空代表移动到头部了，或者 代表剩余的属性remaining 显示为最后一个的
            if (search == NULL ||
                (crawlers[i].remaining && --crawlers[i].remaining < 1)) {
                if (settings.verbose > 2)
                    fprintf(stderr, "Nothing left to crawl for %d\n", i);
                // 结束此队列爬虫，主要为逻辑为 it_flags状态置为0、crawler_count减1、爬虫item 移除、更新下次爬虫结束时间 等
                lru_crawler_class_done(i);
                continue;
            }
            uint32_t hv = hash(ITEM_key(search), search->nkey);
            /* Attempt to hash item lock the "search" item. If locked, no
             * other callers can incr the refcount
             */
            // 对当前hashtable尝试加段锁，如果加锁失败，就暂时跳过
            if ((hold_lock = item_trylock(hv)) == NULL) {
                pthread_mutex_unlock(&lru_locks[i]);
                continue;
            }
            /* Now see if the item is refcount locked */
            // 引用+1 还不等于2，意味着item正在被使用，则引用-1（维持原状），就暂时跳过
            if (refcount_incr(search) != 2) {
                refcount_decr(search);
                if (hold_lock)
                    item_trylock_unlock(hold_lock);
                pthread_mutex_unlock(&lru_locks[i]);
                continue;
            }
            
            crawlers[i].checked++;
            /* Frees the item or decrements the refcount. */
            /* Interface for this could improve: do the free/decr here
             * instead? */
            if (!active_crawler_mod.mod->needs_lock) {
                pthread_mutex_unlock(&lru_locks[i]);
            }
            // 执行对应 item 的具体甄别，执行函数为 crawler_expired_eval，主要内容为 是否 过期、是否执行过flush命令，如果需要淘汰则进行回收item动作
            active_crawler_mod.mod->eval(&active_crawler_mod, search, hv, i);
            if (hold_lock)
                item_trylock_unlock(hold_lock);
            if (active_crawler_mod.mod->needs_lock) {
                pthread_mutex_unlock(&lru_locks[i]);
            }
            // 解锁以及休眠
            if (crawls_persleep-- <= 0 && settings.lru_crawler_sleep) {
                pthread_mutex_unlock(&lru_crawler_lock);
                usleep(settings.lru_crawler_sleep);
                pthread_mutex_lock(&lru_crawler_lock);
                crawls_persleep = settings.crawls_persleep;
            } else if (!settings.lru_crawler_sleep) {
                // TODO: only cycle lock every N?
                pthread_mutex_unlock(&lru_crawler_lock);
                pthread_mutex_lock(&lru_crawler_lock);
            }
        }
    }
    
    if (active_crawler_mod.mod != NULL) {
        // 爬虫模型里面的 结束标识 是否结束
        if (active_crawler_mod.mod->finalize != NULL)
            active_crawler_mod.mod->finalize(&active_crawler_mod);
        // 处理大的消息报问题
        while (active_crawler_mod.c.c != NULL && bipbuf_used(active_crawler_mod.c.buf)) {
            lru_crawler_poll(&active_crawler_mod.c);
        }
        // Double checking in case the client closed during the poll
        if (active_crawler_mod.c.c != NULL) {
            lru_crawler_release_client(&active_crawler_mod.c);
        }
        active_crawler_mod.mod = NULL;
    }
    if (settings.verbose > 2)
        fprintf(stderr, "LRU crawler thread sleeping\n");
    STATS_LOCK();
    stats_state.lru_crawler_running = false;
    STATS_UNLOCK();
    }
    pthread_mutex_unlock(&lru_crawler_lock);
    if (settings.verbose > 2)
        fprintf(stderr, "LRU crawler thread stopping\n");
    return NULL;
}
```

slabs_reassign 这个函数的作用是设置需要进行内存页转移的源和目的slab，并唤醒 rebalance线程 进行内存页转移操作，具体操作 函数为 do_slabs_reassign，唤醒执行的线程函数为 slab_rebalance_thread
```c++
slabs.c

static enum reassign_result_type do_slabs_reassign(int src, int dst) {
    bool nospare = false;
    // 是否 slab 线程正在工作 如果正在工作则不在通知该线程进行处理了
    if (slab_rebalance_signal != 0)
        return REASSIGN_RUNNING;
    if (src == dst)
        return REASSIGN_SRC_DST_SAME;
    /* Special indicator to choose ourselves. */
    if (src == -1) {
        src = slabs_reassign_pick_any(dst);
        /* TODO: If we end up back at -1, return a new error type */
    }
    if (src < SLAB_GLOBAL_PAGE_POOL || src > power_largest ||
        dst < SLAB_GLOBAL_PAGE_POOL || dst > power_largest)
        return REASSIGN_BADCLASS;
    pthread_mutex_lock(&slabs_lock);
    // 如果该 slab id 下的 chunk 小于2块则不回收了
    if (slabclass[src].slabs < 2)
        nospare = true;
    pthread_mutex_unlock(&slabs_lock);
    if (nospare)
        return REASSIGN_NOSPARE;
    // 赋值全局变量，在slab爬虫线程中会获取
    // s_clsid 要回收的 slab id
    // d_clsid 回收之后移动到该 slab id 下
    slab_rebal.s_clsid = src;
    slab_rebal.d_clsid = dst;
    // 修改状态,跟上面判断对应,代表已经通知slab爬虫线程了
    slab_rebalance_signal = 1;
    // 通知slab爬虫线程信号
    pthread_cond_signal(&slab_rebalance_cond);
    return REASSIGN_OK;
}
```

slab_rebalance_thread: 这是 rebalance 线程，首先调用 slab_rebalance_start 确定要转移的源 slab class 中的 slab，然后调用 slab_rebalance_move 将要转移的 slab 中的 item 全部回收，最后如果回收完毕，则会调用slab_rebalance_finish 将 slab 从源 slab class 转移到目的 slab class 中
```c++
slabs.c

static void *slab_rebalance_thread(void *arg) {
    int was_busy = 0;
    /* So we first pass into cond_wait with the mutex held */
    mutex_lock(&slabs_rebalance_lock);
    while (do_run_slab_rebalance_thread) {
        // 开始信号
        if (slab_rebalance_signal == 1) {
            // 确认转移 slab
            if (slab_rebalance_start() < 0) {
                /* Handle errors with more specificity as required. */
                slab_rebalance_signal = 0;
            }
            was_busy = 0;
        } else if (slab_rebalance_signal && slab_rebal.slab_start != NULL) {
            // 开始转移
            was_busy = slab_rebalance_move();
        }
        // 转移完毕
        if (slab_rebal.done) {
            slab_rebalance_finish();
        } else if (was_busy) {
            /* Stuck waiting for some items to unlock, so slow down a bit
             * to give them a chance to free up */
            usleep(1000);
        }
        // 转移信号结束，继续挂起等待
        if (slab_rebalance_signal == 0) {
            /* always hold this lock while we're running */
            pthread_cond_wait(&slab_rebalance_cond, &slabs_rebalance_lock);
        }
    }
    return NULL;
}
```

slab_rebalance_start 这个函数的功能主要的作用是选取需要转出内存页的 slab class 的内存页，源码中选取的是 slab class 中的第一个内存页，因为第一个内存页中的 items 最先分配出去，其中的 items 存在的时间一般较大，确定这个内存页的起始位置和终止位置
```c++
slabs.c

static int slab_rebalance_start(void) {
    slabclass_t *s_cls;
    int no_go = 0;
    pthread_mutex_lock(&slabs_lock);
    // src 和 dst 是否为有效 id
    if (slab_rebal.s_clsid < SLAB_GLOBAL_PAGE_POOL ||
        slab_rebal.s_clsid > power_largest  ||
        slab_rebal.d_clsid < SLAB_GLOBAL_PAGE_POOL ||
        slab_rebal.d_clsid > power_largest  ||
        slab_rebal.s_clsid == slab_rebal.d_clsid)
        no_go = -2;
    // 源 slabclass
    s_cls = &slabclass[slab_rebal.s_clsid];
    // 申请一块内存来保存 chunk 地址，而这个就是对 d_clsid 这个区申请内存来保存 chunk 地址，因为最后 chunk 回收完都会移动 d_clsid 这个区
    if (!grow_slab_list(slab_rebal.d_clsid)) {
        no_go = -1;
    }
    // 在判断一次当前slab下的 chunk 是否小于2块
    if (s_cls->slabs < 2)
        no_go = -3;
    if (no_go != 0) {
        pthread_mutex_unlock(&slabs_lock);
        return no_go; /* Should use a wrapper function... */
    }
    /* Always kill the first available slab page as it is most likely to
     * contain the oldest items
     */
    // 获取s_cls下第一块chunk的开始地址
    slab_rebal.slab_start = s_cls->slab_list[0];
    // 获取s_cls下第一块chunk的结束地址
    slab_rebal.slab_end   = (char *)slab_rebal.slab_start +
        (s_cls->size * s_cls->perslab);
    // 移动下标，因为在回收 chunk 的时候，需要把这个 chunk 里面已使用的 item ，全部复制到其他的 chunk 内，好把当前的 chunk 腾出来，所以就需要靠这个，下标根据 item 的大小，不断的往后移动指针以达到获取每一个 item 的作用，直到结束
    slab_rebal.slab_pos   = slab_rebal.slab_start;
    // 回收状态 1:已回收 0:未回收
    slab_rebal.done       = 0;
    // Don't need to do chunk move work if page is in global pool.
    if (slab_rebal.s_clsid == SLAB_GLOBAL_PAGE_POOL) {
        slab_rebal.done = 1;
    }
    // 更改状态为2
    slab_rebalance_signal = 2;
    if (settings.verbose > 1) {
        fprintf(stderr, "Started a slab rebalance\n");
    }
    pthread_mutex_unlock(&slabs_lock);
    STATS_LOCK();
    stats_state.slab_reassign_running = true;
    STATS_UNLOCK();
    return 0;
}

static int slab_rebalance_move(void) {
    slabclass_t *s_cls;
    int x;
    int was_busy = 0;
    int refcount = 0;
    uint32_t hv;
    void *hold_lock;
    enum move_status status = MOVE_PASS;
    pthread_mutex_lock(&slabs_lock);
    // 取出slab信息
    s_cls = &slabclass[slab_rebal.s_clsid];
    
    // 这里默认一次只循环一次，也就是在移除 chunk 里面 item 的时候，一次只移除一个 item，不过可以在启动的时候设定这个 slab_bulk_check 循环次数，尽可能的还是把这个变量设置小一些，保证只循环一次就退出循环，因为这样可以减小锁的颗粒度，可以看到上面 slab 是全局锁，如果我们当前这个 slab id 一直占用锁，会导致其它的 slab id，也都无法操作，所以这里每次循环处理完一个 item 之后马上释放锁，然后在获取锁在进行处理
    for (x = 0; x < slab_bulk_check; x++) {
        hv = 0;
        hold_lock = NULL;
        // 获取item
        item *it = slab_rebal.slab_pos;
        item_chunk *ch = NULL;
        status = MOVE_PASS;
        
        if (it->it_flags & ITEM_CHUNK) {
            /* This chunk is a chained part of a larger item. */
            ch = (item_chunk *) it;
            /* Instead, we use the head chunk to find the item and effectively
             * lock the entire structure. If a chunk has ITEM_CHUNK flag, its
             * head cannot be slabbed, so the normal routine is safe. */
            it = ch->head;
            assert(it->it_flags & ITEM_CHUNKED);
        }
        /* ITEM_FETCHED when ITEM_SLABBED is overloaded to mean we've cleared
         * the chunk for move. Only these two flags should exist.
         */
        // 判断 it_flags 是不是不等于这两个状态组合，默认情况不会等于，会等到该item移除完毕之后才会赋值成这两个状态组合
        if (it->it_flags != (ITEM_SLABBED|ITEM_FETCHED)) {
            /* ITEM_SLABBED can only be added/removed under the slabs_lock */
            // 是否空闲的item
            if (it->it_flags & ITEM_SLABBED) {
                // 把当前item从空闲 s_cls->slots 链表移动出来，因为我们要回收这个 chunk，所以这个 chunk 里面的item 就不能在被当前 slab 引用了
                assert(ch == NULL);
                slab_rebalance_cut_free(s_cls, it);
                status = MOVE_FROM_SLAB;
            } else if ((it->it_flags & ITEM_LINKED) != 0) {
                // 是否被使用的item
                /* If it doesn't have ITEM_SLABBED, the item could be in any
                 * state on its way to being freed or written to. If no
                 * ITEM_SLABBED, but it's had ITEM_LINKED, it must be active
                 * and have the key written to it already.
                 */
                hv = hash(ITEM_key(it), it->nkey);
                if ((hold_lock = item_trylock(hv)) == NULL) {
                    // 如果没有抢到锁，则代表这个item正在忙
                    status = MOVE_LOCKED;
                } else {
                    bool is_linked = (it->it_flags & ITEM_LINKED);
                    // 引用+1
                    refcount = refcount_incr(it);
                    // 如果等于 2 则代表目前没有其他线程在使用这个item
                    if (refcount == 2) { /* item is linked but not busy */
                        /* Double check ITEM_LINKED flag here, since we're
                         * past a memory barrier from the mutex. */
                        // 在判断一次是否被使用的item
                        if (is_linked) {
                            // 把这个被使用的item复制到其他chunk下
                            status = MOVE_FROM_LRU;
                        } else {
                            /* refcount == 1 + !ITEM_LINKED means the item is being
                             * uploaded to, or was just unlinked but hasn't been freed
                             * yet. Let it bleed off on its own and try again later */
                            // 如果不是则可能刚巧同一时间被删除了,所以改成正在忙的状态,下次循环再看一次
                            status = MOVE_BUSY;
                        }
                    } else if (refcount > 2 && is_linked) {
                        // 如果引用+1不等于2则代表其他线程正在操作该item,正在忙
                        // TODO: Mark items for delete/rescue and process
                        // outside of the main loop.
                        if (slab_rebal.busy_loops > SLAB_MOVE_MAX_LOOPS) {
                            slab_rebal.busy_deletes++;
                            // Only safe to hold slabs lock because refcount
                            // can't drop to 0 until we release item lock.
                            STORAGE_delete(storage, it);
                            pthread_mutex_unlock(&slabs_lock);
                            do_item_unlink(it, hv);
                            pthread_mutex_lock(&slabs_lock);
                        }
                        status = MOVE_BUSY;
                    } else {
                        if (settings.verbose > 2) {
                            fprintf(stderr, "Slab reassign hit a busy item: refcount: %d (%d -> %d)\n",
                                it->refcount, slab_rebal.s_clsid, slab_rebal.d_clsid);
                        }
                        status = MOVE_BUSY;
                    }
                    /* Item lock must be held while modifying refcount */
                    if (status == MOVE_BUSY) {
                        // 引用-1
                        refcount_decr(it);
                        item_trylock_unlock(hold_lock);
                    }
                }
            } else {
                /* See above comment. No ITEM_SLABBED or ITEM_LINKED. Mark
                 * busy and wait for item to complete its upload. */
                status = MOVE_BUSY;
            }
        }
        int save_item = 0;
        item *new_it = NULL;
        size_t ntotal = 0;
        switch (status) {
            case MOVE_FROM_LRU:
                /* Lock order is LRU locks -> slabs_lock. unlink uses LRU lock.
                 * We only need to hold the slabs_lock while initially looking
                 * at an item, and at this point we have an exclusive refcount
                 * (2) + the item is locked. Drop slabs lock, drop item to
                 * refcount 1 (just our own, then fall through and wipe it
                 */
                /* Check if expired or flushed */
                // 当前item总占用字节数
                ntotal = ITEM_ntotal(it);
#ifdef EXTSTORE
                if (it->it_flags & ITEM_HDR) {
                    ntotal = (ntotal - it->nbytes) + sizeof(item_hdr);
                }
#endif
                /* REQUIRES slabs_lock: CHECK FOR cls->sl_curr > 0 */
                if (ch == NULL && (it->it_flags & ITEM_CHUNKED)) {
                    /* Chunked should be identical to non-chunked, except we need
                     * to swap out ntotal for the head-chunk-total. */
                    ntotal = s_cls->size;
                }
                // 判断是否过期了
                if ((it->exptime != 0 && it->exptime < current_time)
                    || item_is_flushed(it)) {
                    /* Expired, don't save. */
                    save_item = 0;
                } else if (ch == NULL &&
                        (new_it = slab_rebalance_alloc(ntotal, slab_rebal.s_clsid)) == NULL) {
                    // 去其他chunk下获取一个空闲的item
                    /* Not a chunk of an item, and nomem. */
                    save_item = 0;
                    slab_rebal.evictions_nomem++;
                } else if (ch != NULL &&
                        (new_it = slab_rebalance_alloc(s_cls->size, slab_rebal.s_clsid)) == NULL) {
                    /* Is a chunk of an item, and nomem. */
                    save_item = 0;
                    slab_rebal.evictions_nomem++;
                } else {
                    /* Was whatever it was, and we have memory for it. */
                    save_item = 1;
                }
                pthread_mutex_unlock(&slabs_lock);
                unsigned int requested_adjust = 0;
                if (save_item) {
                    if (ch == NULL) {
                        assert((new_it->it_flags & ITEM_CHUNKED) == 0);
                        /* if free memory, memcpy. clear prev/next/h_bucket */
                        // 把当前的 item 内容 copy 到新的 new_it 下
                        memcpy(new_it, it, ntotal);
                        new_it->prev = 0;
                        new_it->next = 0;
                        new_it->h_next = 0;
                        /* These are definitely required. else fails assert */
                        new_it->it_flags &= ~ITEM_LINKED;
                        new_it->refcount = 0;
                         // 把当前 item 引用全部释放掉，在把新的 new_it 全部引用上，这样就相当于把当前 item 移动copy 到其他 chunk 下了,把当前 item 位置腾出来了
                        do_item_replace(it, new_it, hv);
                        /* Need to walk the chunks and repoint head  */
                        if (new_it->it_flags & ITEM_CHUNKED) {
                            item_chunk *fch = (item_chunk *) ITEM_data(new_it);
                            fch->next->prev = fch;
                            while (fch) {
                                fch->head = new_it;
                                fch = fch->next;
                            }
                        }
                        it->refcount = 0;
                        it->it_flags = ITEM_SLABBED|ITEM_FETCHED;
#ifdef DEBUG_SLAB_MOVER
                        memcpy(ITEM_key(it), "deadbeef", 8);
#endif
                        slab_rebal.rescues++;
                        requested_adjust = ntotal;
                    } else {
                        item_chunk *nch = (item_chunk *) new_it;
                        /* Chunks always have head chunk (the main it) */
                        ch->prev->next = nch;
                        if (ch->next)
                            ch->next->prev = nch;
                        memcpy(nch, ch, ch->used + sizeof(item_chunk));
                        ch->refcount = 0;
                        ch->it_flags = ITEM_SLABBED|ITEM_FETCHED;
                        slab_rebal.chunk_rescues++;
#ifdef DEBUG_SLAB_MOVER
                        memcpy(ITEM_key((item *)ch), "deadbeef", 8);
#endif
                        refcount_decr(it);
                        requested_adjust = s_cls->size;
                    }
                } else {
                    /* restore ntotal in case we tried saving a head chunk. */
                    ntotal = ITEM_ntotal(it);
                    STORAGE_delete(storage, it);
                    do_item_unlink(it, hv);
                    slabs_free(it, ntotal, slab_rebal.s_clsid);
                    /* Swing around again later to remove it from the freelist. */
                    slab_rebal.busy_items++;
                    was_busy++;
                }
                item_trylock_unlock(hold_lock);
                pthread_mutex_lock(&slabs_lock);
                /* Always remove the ntotal, as we added it in during
                 * do_slabs_alloc() when copying the item.
                 */
                s_cls->requested -= requested_adjust;
                break;
            case MOVE_FROM_SLAB:
                it->refcount = 0;
                // 更新flags状态
                it->it_flags = ITEM_SLABBED|ITEM_FETCHED;
#ifdef DEBUG_SLAB_MOVER
                memcpy(ITEM_key(it), "deadbeef", 8);
#endif
                break;
            case MOVE_BUSY:
            case MOVE_LOCKED:
                // 记录一下正在忙的item数量
                slab_rebal.busy_items++;
                was_busy++;
                break;
            case MOVE_PASS:
                break;
        }
        // 如果本次 item 正在忙没有移动走, 那么也会把指针移动到下个 item 的位置, 处理下个 item 等这一圈全部循环完之后,回过头来发现刚才有正在忙的 item 没有移动走,那么会再继续循环一轮，直到把 chunk 内所有 item 全部移除完毕
        slab_rebal.slab_pos = (char *)slab_rebal.slab_pos + s_cls->size;
        if (slab_rebal.slab_pos >= slab_rebal.slab_end)
            break;
    }
    // 判断是否处理完所有item
    if (slab_rebal.slab_pos >= slab_rebal.slab_end) {
        /* Some items were busy, start again from the top */
        // 判断是否有正在忙的item
        if (slab_rebal.busy_items) {
            // 如果有则重新把slab_pos指针重置到开始位置,然后重新一轮循环处理
            slab_rebal.slab_pos = slab_rebal.slab_start;
            STATS_LOCK();
            stats.slab_reassign_busy_items += slab_rebal.busy_items;
            STATS_UNLOCK();
            slab_rebal.busy_items = 0;
            slab_rebal.busy_loops++;
        } else {
            // 当前chunk内所有item移除完毕
            slab_rebal.done++;
        }
    }
    pthread_mutex_unlock(&slabs_lock);
    return was_busy;
}

static void slab_rebalance_finish(void) {
    slabclass_t *s_cls;
    slabclass_t *d_cls;
    int x;
    uint32_t rescues;
    uint32_t evictions_nomem;
    uint32_t inline_reclaim;
    uint32_t chunk_rescues;
    uint32_t busy_deletes;
    pthread_mutex_lock(&slabs_lock);
    s_cls = &slabclass[slab_rebal.s_clsid];
    d_cls = &slabclass[slab_rebal.d_clsid];
#ifdef DEBUG_SLAB_MOVER
    /* If the algorithm is broken, live items can sneak in. */
    slab_rebal.slab_pos = slab_rebal.slab_start;
    while (1) {
        item *it = slab_rebal.slab_pos;
        assert(it->it_flags == (ITEM_SLABBED|ITEM_FETCHED));
        assert(memcmp(ITEM_key(it), "deadbeef", 8) == 0);
        it->it_flags = ITEM_SLABBED|ITEM_FETCHED;
        slab_rebal.slab_pos = (char *)slab_rebal.slab_pos + s_cls->size;
        if (slab_rebal.slab_pos >= slab_rebal.slab_end)
            break;
    }
#endif
    /* At this point the stolen slab is completely clear.
     * We always kill the "first"/"oldest" slab page in the slab_list, so
     * shuffle the page list backwards and decrement.
     */
    // 因为我们回收了一个chunk所以把chunk数量减一
    s_cls->slabs--;
    // 更新保存 chunk 地址，由于第一个 chunk 地址被回收，所以第二个 chunk 地址挪到数组的第一个位置，以此类推
    for (x = 0; x < s_cls->slabs; x++) {
        s_cls->slab_list[x] = s_cls->slab_list[x+1];
    }
    // 把刚才回收的 chunk 地址保存到 d_clsid 下面
    d_cls->slab_list[d_cls->slabs++] = slab_rebal.slab_start;
    /* Don't need to split the page into chunks if we're just storing it */
    if (slab_rebal.d_clsid > SLAB_GLOBAL_PAGE_POOL) {
        memset(slab_rebal.slab_start, 0, (size_t)settings.slab_page_size);
        split_slab_page_into_freelist(slab_rebal.slab_start,
            slab_rebal.d_clsid);
    } else if (slab_rebal.d_clsid == SLAB_GLOBAL_PAGE_POOL) {
        /* mem_malloc'ed might be higher than mem_limit. */
        mem_limit_reached = false;
        memory_release();
    }
    // 因为已经回收完毕一个chunk 所以重置下
    slab_rebal.busy_loops = 0;
    slab_rebal.done       = 0;
    slab_rebal.s_clsid    = 0;
    slab_rebal.d_clsid    = 0;
    slab_rebal.slab_start = NULL;
    slab_rebal.slab_end   = NULL;
    slab_rebal.slab_pos   = NULL;
    evictions_nomem    = slab_rebal.evictions_nomem;
    inline_reclaim = slab_rebal.inline_reclaim;
    rescues   = slab_rebal.rescues;
    chunk_rescues = slab_rebal.chunk_rescues;
    busy_deletes = slab_rebal.busy_deletes;
    slab_rebal.evictions_nomem    = 0;
    slab_rebal.inline_reclaim = 0;
    slab_rebal.rescues  = 0;
    slab_rebal.chunk_rescues = 0;
    slab_rebal.busy_deletes = 0;
    slab_rebalance_signal = 0;
    pthread_mutex_unlock(&slabs_lock);
    // 统计
    STATS_LOCK();
    stats.slabs_moved++;
    stats.slab_reassign_rescues += rescues;
    stats.slab_reassign_evictions_nomem += evictions_nomem;
    stats.slab_reassign_inline_reclaim += inline_reclaim;
    stats.slab_reassign_chunk_rescues += chunk_rescues;
    stats.slab_reassign_busy_deletes += busy_deletes;
    stats_state.slab_reassign_running = false;
    STATS_UNLOCK();
    if (settings.verbose > 1) {
        fprintf(stderr, "finished a slab move\n");
    }
}
```


## 老版本的LRU算法
在 memcached 中，每个 item 对象被创建的时候，它维护一个计数器，item 对象计数器的值就是 unix 当前时间戳，当一个 item 被 FETCHED 的时候（get、set、replace），这个计数器的值就会更新为当前时间（表示被使用了）。

如果 memcached 在遇到 set 操作的时候，发现内存不够，就会淘汰计数器值最小的 item（过期的优先淘汰），本质上就是这么简单：如果某个 item 没被使用就优先淘汰。

听上去是不是很简单？我们再稍微深入一点，每个 Slab-class 的 LRU 由一个双向链表维护：
* 当一个新 item 被 set 的时候，它会进入链表的 head，如果发生 evict，那么在链表 tail 尾的 item 会从内存中释放。
* 当 get 一个 item，它会从链表中 unlink，然后重新 link 到链表的 head，这个过程叫做 bump。
  由于 bump 会有锁（mutex locks and mutations），频繁发生对性能有非常大的影响，所以 memcached 做了一个优化，在 60 秒内同一个 item 只会产生一次 bump。

活跃的 item 即使没有频繁产生 bump，但如果 get 操作非常多，也会产生很多的锁竞争，导致某些 get 延时不一致，甚至导致 cpu 负载在某个时间过高，也就是多线程的扩展性受制于 memcached 的 LRU lock。什么意思呢？对于老的 URL 实现来说，memcached 开启的工作线程建议不要超过 8 个。