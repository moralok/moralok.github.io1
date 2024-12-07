---
title: Redis 的内存淘汰
date: 2024-12-04 15:55:30
tags: [memory eviction, redis, algorithm]
mathjax: true
---

本文记录了 `Redis` 的**内存淘汰策略**，并介绍了其中 `LRU` 和 `LFU` 算法实现相比于普通版本的**优化思路**，最后简单探究了 `Redis` 关于内存淘汰的**具体实现细节**，重点关注 `Redis` 是如何**选取**和**评估**哪些 `key` 是更应该优先被淘汰的 `key`。
在探究的过程中，再次体会到 Chat GPT 带来的便捷和它在细微之处的胡说八道，以及网上的资料很有帮助但是通过阅读源码加以验证同样重要。

<!-- more -->

前置补充说明：

- 以下提到的代码文件可在 [GitHub 的 redis 代码库](https://github.com/redis/redis/)中搜索。
- 引用代码的分支版本为 `4.0`。
- 阅读代码并不需要对 `C` 语言有多深入的了解，也不需要配置相关环境，通过在 `Github` 代码库中搜索符号跳转即可完成。

## Redis 的内存淘汰策略

Redis 的内存淘汰策略可以通过配置项 `maxmemory-policy` 设置，最大内存阈值可以通过配置项 `maxmemory` 设置：

- `volatile-lru`：从设置了过期时间的数据集中挑选最近最少使用的数据进行淘汰。
- `allkeys-lru`：从所有数据集中挑选最近最少使用的数据进行淘汰。
- `volatile-lfu`：从设置了过期时间的数据集中挑选最近最不经常使用的数据进行淘汰。
- `allkeys-lfu`：从所有数据集中挑选最近最不经常使用的数据进行淘汰。
- `volatile-random`：从设置了过期时间的数据集中随机挑选数据进行淘汰。
- `allkeys-random`：从所有数据集中随机挑选数据进行淘汰。
- `volatile-ttl`：从设置了过期时间的数据集中随机最近要过期的数据进行淘汰。
- `noeviction`(默认)：不淘汰数据，当内存不足时，新写入操作将报错。

根据面向的数据集分为两类，**设置了过期时间的数据集**指 `server.db[i].expires`，**所有数据集**指 `server.db[i].dict`。两者都是 `dict` 类型，在后续的代码分析中，我们会看到 `Redis` 根据策略是 `volatile` 还是 `allkeys` 类型选择引用哪一个 `dict`。

根据算法类型分为五类，`LRU`、`LFU`、`Random`、`TTL` 和 `NO`。

> `LFU` 算法是在 `4.0` 版本后新增。其实截至目前（2024年），`Redis` 的版本已经来到了 `8.0`，接触到 `4.0` 以前的版本的可能性已经很低了。

关于内存淘汰策略的最新定义可以在 `config.c` 中查看：

```c
configEnum maxmemory_policy_enum[] = {
    {"volatile-lru", MAXMEMORY_VOLATILE_LRU},
    {"volatile-lfu", MAXMEMORY_VOLATILE_LFU},
    {"volatile-random",MAXMEMORY_VOLATILE_RANDOM},
    {"volatile-ttl",MAXMEMORY_VOLATILE_TTL},
    {"allkeys-lru",MAXMEMORY_ALLKEYS_LRU},
    {"allkeys-lfu",MAXMEMORY_ALLKEYS_LFU},
    {"allkeys-random",MAXMEMORY_ALLKEYS_RANDOM},
    {"noeviction",MAXMEMORY_NO_EVICTION},
    {NULL, 0}
};
```

上述常量定义在 `server.h` 中：

```c
/* Redis maxmemory strategies. Instead of using just incremental number
 * for this defines, we use a set of flags so that testing for certain
 * properties common to multiple policies is faster. */
#define MAXMEMORY_FLAG_LRU (1<<0)
#define MAXMEMORY_FLAG_LFU (1<<1)
#define MAXMEMORY_FLAG_ALLKEYS (1<<2)
#define MAXMEMORY_FLAG_NO_SHARED_INTEGERS \
    (MAXMEMORY_FLAG_LRU|MAXMEMORY_FLAG_LFU)
#define MAXMEMORY_VOLATILE_LRU ((0<<8)|MAXMEMORY_FLAG_LRU)
#define MAXMEMORY_VOLATILE_LFU ((1<<8)|MAXMEMORY_FLAG_LFU)
#define MAXMEMORY_VOLATILE_TTL (2<<8)
#define MAXMEMORY_VOLATILE_RANDOM (3<<8)
#define MAXMEMORY_ALLKEYS_LRU ((4<<8)|MAXMEMORY_FLAG_LRU|MAXMEMORY_FLAG_ALLKEYS)
#define MAXMEMORY_ALLKEYS_LFU ((5<<8)|MAXMEMORY_FLAG_LFU|MAXMEMORY_FLAG_ALLKEYS)
#define MAXMEMORY_ALLKEYS_RANDOM ((6<<8)|MAXMEMORY_FLAG_ALLKEYS)
#define MAXMEMORY_NO_EVICTION (7<<8)
```

由自身经验或注释均可知，`Redis` 没有使用增量数字来定义这些常量，而是使用了一组标志，以便**更快地测试多个策略的某些共同属性**。在后续的代码分析中，我们会看到 `Redis` 是如何通过**位运算**判断内存淘汰策略是否具有某一属性，比如是否属于 `LRU` 类型就可以通过 `server.maxmemory_policy & MAXMEMORY_FLAG_LRU` 来判断。

## Redis 的内存淘汰优化

和普通版本的 `LRU` 以及 `LFU` 算法实现并不完全相同，`Redis` 的 `LRU` 和 `LFU` 做了一些“优化”，**主要目的是降低内存和时间的开销**。

### 近似 LRU

> 代码注释中使用了 approximated LRU algorithm 作为表述。

`Redis` 的 `LRU` 并非严格的 `LRU`，而是使用了“近似 `LRU`”。在具体实现中，`Redis` 为了节省内存，并没有维护一个完整的 `LRU` 链表，而是为每个 `key` 维护一个“**最后访问时间**”，然后通过采样机制检查一部分 `key`，在通过筛选获得的候选 `key` 中选出其中最近最少使用的 `key` 进行淘汰。

- **访问时间戳**：`Redis` 为每个键维护一个“最后访问时间”，用于实现 `LRU`。
- **采样机制**：当已用内存超出 `maxmemory` 时，`Redis` 会通过采样机制获取一部分 `key` 加以筛选，然后在通过筛选获得的候选 `key` 中选取“最后访问时间”最久远的 `key` 进行淘汰。这个采样机制降低了内存的开销，也减少了计算的复杂度。
- **采样精度**：用户可以通过配置参数 `maxmemory-samples` 调整采样数量来影响采样的精度。采样数量越多，近似 `LRU` 越接近于真实的 `LRU`（回头再看这句陈述，个人并不能严谨的证明）。

> 事实上，
“最后访问时间”并不是简单地直接使用我们常用的时间戳，而是为了节省内存对时间戳做了点小处理。
`Redis` 的采样机制也不是简单地在每次需要进行内存淘汰时获取一部分 `key`，然后从这部分 `key` 中选择要淘汰的目标。如果是这样，并不能选出更应该优先被淘汰的 `key`。
回头再看“采样数量越多，越接近于真实的 `LRU`”这句话，个人虽然予以保留，但心里是存疑的，个人只能确认增加采样的数量在一定程度上可以**加快**接近真实 LRU 的速度。

### 近似 LFU

`Redis` 的 `LFU` 同样并非严格的 `LFU`，也是使用了“近似 `LFU`”。`Redis` 在使用 `LFU` 策略时，会为每个 `key` 维护一个“频率计数器”，然后通过和 `LRU` 相同的采样机制，在通过筛选获得的候选 `key` 中选出其中最近最不常使用的 `key` 进行淘汰，若访问频率相同，则选择最近最少使用的 `key`。

- **计数器设计**：`Redis` 为每个 `key` 维护一个 `8` 位的“频率计数器”，其取值范围是 `0-255`，用于统计访问频率。**注意**：这个“频率计数器”不直接记录访问次数，而是记录访问次数的对数 `logc`（logistic counter）。
- **随时间衰减**：`Redis` 的 `LFU` 还使用了频率衰减机制，以避免陈旧的高频 `key` 长期占据内存。在使用 `LFU` 策略时，`Redis` 不仅为每个 `key` 维护一个“频率计数器”，还会维护一个 `ldt`（last decrement time），用于记录上一次 `logc` 的更新时间。`Redis` 会根据当前的时间和 `ldt` 的差值以及配置的 `lfu-decay-time` 对 `logc` 进行衰减。
- **衰减时间**：用户可以通过配置项 `lfu-decay-time` 控制衰减的速率，默认是 `1`。
- **采样精度**：和 `LRU` 一样，用户可以通过配置参数 `maxmemory-samples` 调整采样数量来影响采样的精度。事实上 `LRU`、`LFU` 和 `TTL` 共用了同一个采样机制。

> 事实上，`logc` 并不是直接对访问次数求对数而得，因为求对数是一个较为复杂的操作。`Redis` 通过**加权衰减**机制模拟对数增长，在访问时它会根据 `logc` 的数值决定增加“频率计数器”的概率，公式可以简化为：
$$
p = \frac{1}{logc + 1}
$$
真正的公式有少许调整，我们会在后续代码分析中进一步探究。这个公式使得“频率计数器”的增长是非线性的——计数越高，增加的概率越小，从而避免计数器快速增长导致溢出。对于高频访问的 `key`，“频率计数器”仍能达到较高的值；对于低频访问的 `key`，“频率计数器”增长缓慢，在内存淘汰时也更容易被淘汰。

> 关于“随时间衰减”机制，其实个人在刚开始对具体细节就有些好奇，对一些不够清晰的表述也有些困惑，而探究的过程也确实“一波三折”。有些资料将其称为“定期衰减”，个人感觉有点容易让人误解，随意取了一个名字。

## Redis 的内存淘汰实现细节

> 在了解 `Redis` 内存淘汰算法的优化思路时，情不自禁会设想如何处理脑海中浮现的一些问题。有时候并不是不知道“看了源代码就知道了”，而是对不熟悉的项目可能不知道从何开始看，以至于“开始”成了最难的一个步骤。

### freeMemoryIfNeeded

`Redis` 处理内存淘汰的过程是定义在 `evict.c` 中的 `freeMemoryIfNeeded` 方法，它会先判断是否真正需要进行内存淘汰，如果需要则根据设置的内存淘汰策略执行相应的操作。

1. 内存淘汰前判断和处理。
    1. 比较已用内存和 `maxmemory` 判断是否需要进行内存淘汰（由此可见**一些配置项会被转换为 `server` 的属性**在代码中使用）。
        ```c
        /* Check if we are over the memory usage limit. If we are not, no need
        * to subtract the slaves output buffers. We can just return ASAP. */
        mem_reported = zmalloc_used_memory();
        if (mem_reported <= server.maxmemory) return C_OK;
        ```
    2. 在真正淘汰 `key` 前会先尝试释放主从同步和 `AOF` 的缓冲区占用的内存（由此可见这部分的内存也被计算在已用内存中，也就是说已用内存**不单单只包含存储数据占用的内存**），如果在释放后内存已经够用，则直接返回。
        ```c
        /* Remove the size of slaves output buffers and AOF buffer from the
         * count of used memory. */
        mem_used = mem_reported;
        size_t overhead = freeMemoryGetNotCountedMemory();
        mem_used = (mem_used > overhead) ? mem_used-overhead : 0;

        /* Check if we are still over the memory limit. */
        if (mem_used <= server.maxmemory) return C_OK;
        ```
    3. 计算需要释放的内存（按这个计算方式，释放到 `maxmemory` 即可写入数据？那不是有机会在最后放入一个超大 `key` 从而使已用内存超过阈值很多？）。
        ```c
        mem_tofree = mem_used - server.maxmemory;
        mem_freed = 0;
        ```
2. 根据设置的内存淘汰策略，执行相应的操作。
    1. 如果策略设置为 `noeviction`，则直接跳转到不能释放内存的处理逻辑。
        ```c
        if (server.maxmemory_policy == MAXMEMORY_NO_EVICTION)
            goto cant_free;
        ```
        在 `cant_free` 代码块中，如果异步释放内存的线程正在执行任务，则等待（**对于 if 判断条件不理解**）。
        ```c
        cant_free:
        /* We are here if we are not able to reclaim memory. There is only one
        * last thing we can try: check if the lazyfree thread has jobs in queue
        * and wait... */
        while(bioPendingJobsOfType(BIO_LAZY_FREE)) {
            if (((mem_reported - zmalloc_used_memory()) + mem_freed) >= mem_tofree)
                break;
            usleep(1000);
        }
        return C_ERR;
        ```
    2. 否则，循环“选取“最佳”的淘汰目标 `key` 然后释放内存”直到释放的内存大于需要释放的内存。
        ```c
        while (mem_freed < mem_tofree) {
        }
        ```
        在 `while` 循环的内部，主要分为两个步骤：
            1. 选取“最佳”的淘汰目标 `key`
            2. 释放内存

### 选取“最佳”的淘汰目标 key

选取“最佳”的淘汰目标 `key` 的处理分为两种情形：
1. 在使用 `LRU`、`LFU` 和 `TTL` 策略时，需要通过采样机制
2. 在使用 `Random` 策略时，则较为简单

相关的分支判断代码如下，判断过程通过位运算提高效率：

```c
if (server.maxmemory_policy & (MAXMEMORY_FLAG_LRU|MAXMEMORY_FLAG_LFU) ||
    server.maxmemory_policy == MAXMEMORY_VOLATILE_TTL)
{
}
else if (server.maxmemory_policy == MAXMEMORY_ALLKEYS_RANDOM ||
            server.maxmemory_policy == MAXMEMORY_VOLATILE_RANDOM)
{
}
```

#### 在使用 Random 策略时选择 key

`Redis` 在使用 `Random` 策略时选择 `key` 的过程较为简单，所以我们先看这部分。

```c
/* When evicting a random key, we try to evict a key for
 * each DB, so we use the static 'next_db' variable to
 * incrementally visit all DBs. */
for (i = 0; i < server.dbnum; i++) {
    j = (++next_db) % server.dbnum;
    db = server.db+j;   // 指针运算
    dict = (server.maxmemory_policy == MAXMEMORY_ALLKEYS_RANDOM) ?
            db->dict : db->expires;
    if (dictSize(dict) != 0) {
        de = dictGetRandomKey(dict);
        bestkey = dictGetKey(de);
        bestdbid = j;
        break;
    }
}
```

1. `Redis` 首先使用一个静态全局的变量 `next_db` 用于遍历所有 `DB`。
2. 然后根据是否为 `allkeys` 类型的策略，选择不同的 `dict`。通过指针运算，选择 `DB` 和 `dict` 的代码显得很优雅。
3. `dictGetRandomKey` 方法定义在 `dict.c` 中，顾名思义，用于从哈希表 `dict` 中获得一个随机的条目 `dictEntry`。

> `db->dict` 是主字典，存储了 `key-value`；`db->expires` 存储了 `key-expire_time`。

`dictGetRandomKey` 方法并不复杂，即使不了解“重哈希”的实现细节，也很容易理解，主要分为两个步骤：
1. 用随机数作为数组索引选择一个非空 `bucket`。
2. 在选择到一个非空 `bucket` 后，就得到了存储在那的链表。遍历并计算链表的大小，并再次用一个随机数选择一个 `dictEntry`。

```c
dictEntry *dictGetRandomKey(dict *d)
{
    dictEntry *he, *orighe;
    unsigned long h;
    int listlen, listele;

    // 如果 dict 大小为 0，直接返回
    if (dictSize(d) == 0) return NULL;
    if (dictIsRehashing(d)) _dictRehashStep(d);
    // 1. 用随机数作为数组索引选择一个非空 bucket
    if (dictIsRehashing(d)) {
        // 如果正在重哈希，需要用到 ht[0] 和 ht[1]
        do {
            /* We are sure there are no elements in indexes from 0
             * to rehashidx-1 */
            h = d->rehashidx + (random() % (d->ht[0].size +
                                            d->ht[1].size -
                                            d->rehashidx));
            he = (h >= d->ht[0].size) ? d->ht[1].table[h - d->ht[0].size] :
                                      d->ht[0].table[h];
        } while(he == NULL);
    } else {
        // 正常情况，只需要用到 ht[0]
        do {
            h = random() & d->ht[0].sizemask;
            he = d->ht[0].table[h];
        } while(he == NULL);
    }

    // 2. 在选择到一个非空 bucket 后，用随机数在链表上选择一个 dictEntry
    /* Now we found a non empty bucket, but it is a linked
     * list and we need to get a random element from the list.
     * The only sane way to do so is counting the elements and
     * select a random index. */
    listlen = 0;
    orighe = he;
    while(he) {
        he = he->next;
        listlen++;
    }
    listele = random() % listlen;
    he = orighe;
    while(listele--) he = he->next;
    return he;
}
```

`dictSize` 方法定义在 `dict.h` 中，可以快速获取 `dict` 的大小。

```c
#define dictSize(d) ((d)->ht[0].used+(d)->ht[1].used)
```

> `ChatGPT` 的胡言乱语中曾提到 `Redis` 需要遍历所有的 `key` 进行统计才可以知道 `key` 的数量，因此在使用 `Random` 策略进行内存淘汰时需要注意效率问题。不知道这个说法的诞生是否是因为考虑到获取随机的一个 `key` 需要遍历链表统计数量，而如果链表过长，可能会影响性能。

#### 在使用 LRU、LFU 和 TTL 策略时选择 key

1. 采样机制的核心是一个保存了“更应该优先被淘汰”的 `key` 的候选池 `pool`，默认大小为 `16`。
    ```c
    struct evictionPoolEntry *pool = EvictionPoolLRU;
    ```
2. `Redis` 遍历每个非空 `DB`，从中获取一部分 `key` **尝试**填充到 `pool` 中。
    ```c
    /* We don't want to make local-db choices when expiring keys,
     * so to start populate the eviction pool sampling keys from
     * every DB. */
    for (i = 0; i < server.dbnum; i++) {
        db = server.db+i;
        dict = (server.maxmemory_policy & MAXMEMORY_FLAG_ALLKEYS) ?
                db->dict : db->expires;
        if ((keys = dictSize(dict)) != 0) {
            // 填充 pool
            evictionPoolPopulate(i, dict, db->dict, pool);
            total_keys += keys;
        }
    }
    if (!total_keys) break; /* No keys to evict. */
    ```
3. 填充后的 `pool` 中的 `key` 已经按“更不应该优先被淘汰”到“更应该优先被淘汰”的顺序排序，在 `pool` 中从“最优先被淘汰”开始选择，直到选择到一个存在的 `key`。
    ```c
    /* Go backward from best to worst element to evict. */
    for (k = EVPOOL_SIZE-1; k >= 0; k--) {
        // 如果 key 不存在，跳过（pool 可能没有满）
        if (pool[k].key == NULL) continue;
        bestdbid = pool[k].dbid;
        // 根据策略的不同，从对应的 dict 中获取 dictEntry
        if (server.maxmemory_policy & MAXMEMORY_FLAG_ALLKEYS) {
            de = dictFind(server.db[pool[k].dbid].dict,
                pool[k].key);
        } else {
            de = dictFind(server.db[pool[k].dbid].expires,
                pool[k].key);
        }

        /* Remove the entry from the pool. */
        if (pool[k].key != pool[k].cached)
            sdsfree(pool[k].key);
        pool[k].key = NULL;
        pool[k].idle = 0;

        /* If the key exists, is our pick. Otherwise it is
         * a ghost and we need to try the next element. */
        if (de) {
            // 如果 key 存在
            bestkey = dictGetKey(de);
            break;
        } else {
            // 如果 key 不存在，可能已经过期或删除了？
            /* Ghost... Iterate again. */
        }
    }
    ```
4. 若在 `pool` 中找不到可以淘汰的 `key`，会循环上述过程直到选到一个可以淘汰的 `key` 为止
    ```c
    while(bestkey == NULL) {   
    }
    ```

#### 填充 EvictionPoolLRU

填充 `EvictionPoolLRU`（`pool`） 的过程，就是所谓采样的过程，`LRU`、`LFU` 和 `TTL` 共享了这部分处理逻辑。之所以命名中携带了 `LRU`，是因为这部分代码原先就是为了处理 `LRU` 而写的。

> `Redis` 的代码中也会留下这种不修正命名的历史代码呢。代码洁癖既有好的一面，也有不好的一面，得取舍。

1. 首先通过 `dictGetSomeKeys` 获取（最多） `maxmemory_samples` 个 `key`，该方法并不保证返回指定个数的结果。
    ```c
    int j, k, count;
    dictEntry *samples[server.maxmemory_samples];
    count = dictGetSomeKeys(sampledict,samples,server.maxmemory_samples);
    ```
2. 循环处理实际获得的 `dictEntry`，将它们与之前留在 `pool` 中的 `key` 进行比较，留下更应该优先被淘汰的 `key`
    ```c
    for (j = 0; j < count; j++) {
    }
    ```
    1. 获取 `key` 以及视情况从主字典获取 `value`。这是因为 `db.expires` 中存储的 `value` 是过期时间，在使用 `TTL` 类型的策略时，使用过期时间就足以实现算法；在使用非 `TTL` 策略时，需要用到主字典存储的 `value` 对象头中的 `lru` 字段。
        ```c
        unsigned long long idle;
        sds key;
        robj *o;
        dictEntry *de;

        de = samples[j];
        // 获取 key
        key = dictGetKey(de);

        /* If the dictionary we are sampling from is not the main
         * dictionary (but the expires one) we need to lookup the key
         * again in the key dictionary to obtain the value object. */
        if (server.maxmemory_policy != MAXMEMORY_VOLATILE_TTL) {
            // 非 TTL 策略中，需要从主字典中获取 value
            if (sampledict != keydict) de = dictFind(keydict, key);
            o = dictGetVal(de);
        }
        ```
    2. 计算 `idle` 作为评估 `key` 是否应该优先被淘汰的“得分”，分数越高表示越应该优先被淘汰。之所以叫做 `idle`，也是因为这部分代码原先就是为了处理 `LRU` 而写的。`idle` 在使用不同的内存淘汰策略时有不同的计算方式。
        ```c
        /* Calculate the idle time according to the policy. This is called
         * idle just because the code initially handled LRU, but is in fact
         * just a score where an higher score means better candidate. */
        if (server.maxmemory_policy & MAXMEMORY_FLAG_LRU) {
            // LRU
            idle = estimateObjectIdleTime(o);
        } else if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
            /* When we use an LRU policy, we sort the keys by idle time
             * so that we expire keys starting from greater idle time.
             * However when the policy is an LFU one, we have a frequency
             * estimation, and we want to evict keys with lower frequency
             * first. So inside the pool we put objects using the inverted
             * frequency subtracting the actual frequency to the maximum
             * frequency of 255. */
            // LFU
            idle = 255-LFUDecrAndReturn(o);
        } else if (server.maxmemory_policy == MAXMEMORY_VOLATILE_TTL) {
            /* In this case the sooner the expire the better. */
            // TTL
            idle = ULLONG_MAX - (long)dictGetVal(de);
        } else {
            serverPanic("Unknown eviction policy in evictionPoolPopulate()");
        }
        ```
    3. 将当前 `key` 的“得分”和 `pool` 中已存在的 `key` 的得分进行比较，判断它是否应该被放入 `pool` 中，如果需要则做些前置处理。
        ```c
        /* Insert the element inside the pool.
         * First, find the first empty bucket or the first populated
         * bucket that has an idle time smaller than our idle time. */
        k = 0;
        while (k < EVPOOL_SIZE &&
               pool[k].key &&
               pool[k].idle < idle) k++;
        if (k == 0 && pool[EVPOOL_SIZE-1].key != NULL) {
            /* Can't insert if the element is < the worst element we have
             * and there are no empty buckets. */
            // 如果 key 的得分低于 pool 中最差的 key，则跳过
            continue;
        } else if (k < EVPOOL_SIZE && pool[k].key == NULL) {
            /* Inserting into empty position. No setup needed before insert. */
            // 如果目标位置为空，无需前置处理
        } else {
            /* Inserting in the middle. Now k points to the first element
             * greater than the element to insert.  */
            // 如果目标位置位于中间
            if (pool[EVPOOL_SIZE-1].key == NULL) {
                /* Free space on the right? Insert at k shifting
                 * all the elements from k to end to the right. */

                /* Save SDS before overwriting. */
                // 如果还有空位置，则将剩余的 key 右移一个位置
                sds cached = pool[EVPOOL_SIZE-1].cached;
                memmove(pool+k+1,pool+k,
                    sizeof(pool[0])*(EVPOOL_SIZE-k-1));
                pool[k].cached = cached;
            } else {
                /* No free space on right? Insert at k-1 */
                // 如果没有空位置，只需要淘汰掉最低分，将跳过的 key 左移一个位置
                k--;
                /* Shift all elements on the left of k (included) to the
                 * left, so we discard the element with smaller idle time. */
                sds cached = pool[0].cached; /* Save SDS before overwriting. */
                if (pool[0].key != pool[0].cached) sdsfree(pool[0].key);
                memmove(pool,pool+1,sizeof(pool[0])*k);
                pool[k].cached = cached;
            }
        }
        ```
    4. 将评估为更应该优先被淘汰的 `key` 放入 `pool` 的相应位置
        ```c
        /* Try to reuse the cached SDS string allocated in the pool entry,
         * because allocating and deallocating this object is costly
         * (according to the profiler, not my fantasy. Remember:
         * premature optimizbla bla bla bla. */
        int klen = sdslen(key);
        if (klen > EVPOOL_CACHED_SDS_SIZE) {
            pool[k].key = sdsdup(key);
        } else {
            memcpy(pool[k].cached,key,klen+1);
            sdssetlen(pool[k].cached,klen);
            pool[k].key = pool[k].cached;
        }
        pool[k].idle = idle;
        pool[k].dbid = dbid;
        ```
    

总的来说，`Redis` 维护了一个保存了更应该优先被淘汰的 `key` 的 `pool`（类似于排行榜前 100），大小为 `16`；每次采样，从每个非空 `DB` 中获取最多 `maxmemory_samples` 个 `key`，根据当前使用的内存淘汰策略对其进行评估；将当前 `key` 的评估结果和原先留在 `pool` 中的 `key` 的进行比较，留下更应该优先被淘汰的 `key`；随着内存淘汰的不断进行，`pool` 中的 `key` 将越来越具有“代表性”。

> `dictGetSomeKeys` 像裹脚布一样，跳过算了。。。

### 淘汰 key

在选取到“最佳”的淘汰目标 `key` 后，释放内存。

> 无话可说，主要是对于内存分配和释放的代码，个人也没深入阅读。

```c
if (bestkey) {
    // 确定 DB
    db = server.db+bestdbid;
    robj *keyobj = createStringObject(bestkey,sdslen(bestkey));
    propagateExpire(db,keyobj,server.lazyfree_lazy_eviction);
    // 计算释放的内存
    delta = (long long) zmalloc_used_memory();
    latencyStartMonitor(eviction_latency);
    // 删除
    if (server.lazyfree_lazy_eviction)
        dbAsyncDelete(db,keyobj);
    else
        dbSyncDelete(db,keyobj);
    latencyEndMonitor(eviction_latency);
    latencyAddSampleIfNeeded("eviction-del",eviction_latency);
    latencyRemoveNestedEvent(latency,eviction_latency);
    delta -= (long long) zmalloc_used_memory();
    mem_freed += delta;
    // 统计淘汰的 key
    server.stat_evictedkeys++;
    notifyKeyspaceEvent(NOTIFY_EVICTED, "evicted",
        keyobj, db->id);
    decrRefCount(keyobj);
    keys_freed++;

    /* When the memory to free starts to be big enough, we may
     * start spending so much time here that is impossible to
     * deliver data to the slaves fast enough, so we force the
     * transmission here inside the loop. */
    if (slaves) flushSlavesOutputBuffers();

    /* Normally our stop condition is the ability to release
     * a fixed, pre-computed amount of memory. However when we
     * are deleting objects in another thread, it's better to
     * check, from time to time, if we already reached our target
     * memory, since the "mem_freed" amount is computed only
     * across the dbAsyncDelete() call, while the thread can
     * release the memory all the time. */
    if (server.lazyfree_lazy_eviction && !(keys_freed % 16)) {
        overhead = freeMemoryGetNotCountedMemory();
        mem_used = zmalloc_used_memory();
        mem_used = (mem_used > overhead) ? mem_used-overhead : 0;
        if (mem_used <= server.maxmemory) {
            mem_freed = mem_tofree;
        }
    }
}

if (!keys_freed) {
    latencyEndMonitor(latency);
    latencyAddSampleIfNeeded("eviction-cycle",latency);
    goto cant_free; /* nothing to free... */
}
```

### 如何评估“更应该优先被淘汰”

在这部分我们会介绍 `Redis` 具体如何评估一个 `key` 更应该优先被淘汰。

#### Redis 的对象头

我们先看 `Redis` 的对象头，对象头中存储了评估所需要的信息，对象头的定义在 `server.h` 中：

```c
#define LRU_BITS 24
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

**重点关注** `lru` 字段，该字段为 `24` 位，在使用 `LRU` 策略时，存储 `server.lruclock`；在使用 `LFU` 策略时，存储两个值，分别是 `ldt` 和 `logc`。

在每次访问 `key` 时，`Redis` 都会更新该字段，相关处理逻辑在 `db.c` 的 `lookupKey` 方法中：

```c
robj *lookupKey(redisDb *db, robj *key, int flags, dictEntry **deref) {
    // ...
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        // 如果使用 LFU 策略，则调用 updateLFU 方法
        updateLFU(val);
    } else {
        // 否则，将 lru 赋值为 LRU_CLOCK 的返回值
        val->lru = LRU_CLOCK();
    }
    // ...
}
```

#### 在使用 LRU 策略时的评估机制

在使用 `LRU` 策略时，每次访问 `key`，都会更新 `lru` 字段为 `LRU_CLOCK` 方法的返回值。如果 `Redis` 刷新 `lruclock` 的频率高于一定程度，则获取缓存的 `lru_clock`；否则获取系统时间计算 `lruclock`。

> 由于这个分辨率（resolution）的概念有点抽象，感觉描述起来有点奇怪，但意会一下并不困难。我们可以先往下看看 `lruclock` 的定义和更新。

```c
/* This function is used to obtain the current LRU clock.
 * If the current resolution is lower than the frequency we refresh the
 * LRU clock (as it should be in production servers) we return the
 * precomputed value, otherwise we need to resort to a system call. */
unsigned int LRU_CLOCK(void) {
    unsigned int lruclock;
    if (1000/server.hz <= LRU_CLOCK_RESOLUTION) {
        // 获取缓存的 lruclock
        atomicGet(server.lruclock,lruclock);
    } else {
        // 系统调用，获得当下的 lruclock
        lruclock = getLRUClock();
    }
    return lruclock;
}
```

##### lruclock 的定义和更新

在 `Redis` 中，`server.lruclock` 是用于 `LRU` 的时钟，我们通过它来计算 `key` 的空闲时间 `idle`。`lruclock` 并不直接使用时间戳，为了节省内存，它使用了秒级时间戳对常量 `LRU_CLOCK_MAX` 取余的结果，这样就可以只用 `24` 位存储。计算 `lruclock` 的方法 `getLRUClock` 定义在 `evict.c` 中：

```c
/* Return the LRU clock, based on the clock resolution. This is a time
 * in a reduced-bits format that can be used to set and check the
 * object->lru field of redisObject structures. */
unsigned int getLRUClock(void) {
    return (mstime()/LRU_CLOCK_RESOLUTION) & LRU_CLOCK_MAX;
}
```

相关常量定义在 `server.h` 中：

```c
#define LRU_BITS 24
#define LRU_CLOCK_MAX ((1<<LRU_BITS)-1) /* Max value of obj->lru */
#define LRU_CLOCK_RESOLUTION 1000 /* LRU clock resolution in ms */
```

- `LRU_CLOCK_RESOLUTION` 表示时钟分辨率（感觉这个概念有点抽象，好像在不同代码中有多个概念重叠在里面了，不过大概明白它的用处）。
- 最大值 `LRU_CLOCK_MAX` 就是 $2^{24} - 1$。
- `lruclock` 是一个 `24` 位表示的整数，大约 `194` 天“清零”一次。

`server.lruclock` 是 `lruclock` 的缓存，会定期更新。相关处理逻辑在 `server.c` 的 `serverCron` 防范中，该方法中的操作每秒钟执行 `server.hz` 次，`server.hz` 的默认值为 `10`。

```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    // ...
    server.lruclock = getLRUClock();
}
```

> 之所以需要缓存 `lruclock` 是因为 `mstime` 方法是系统调用，消耗较高。也就是说在 `LRU` 时钟分辨率为 `1000ms` 时，只要每秒钟更新至少一次，我们就可以使用缓存的 `lruclock`。

##### 空闲时间的计算

计算 `key` 的空闲时间 `idle` 的方法 `estimateObjectIdleTime` 定义在 `evict.c` 中：
1. 获取 `lruclock`
2. 计算空闲时间 `idle`，考虑 `lruclock` 折返分两种情况

> Redis 并没有考虑时钟多次折返的情况，它假设时钟折返发生的频率非常低，认为一次折返已经覆盖了大部分情况。

```c
/* Given an object returns the min number of milliseconds the object was never
 * requested, using an approximated LRU algorithm. */
unsigned long long estimateObjectIdleTime(robj *o) {
    unsigned long long lruclock = LRU_CLOCK();
    if (lruclock >= o->lru) {
        return (lruclock - o->lru) * LRU_CLOCK_RESOLUTION;
    } else {
        return (lruclock + (LRU_CLOCK_MAX - o->lru)) *
                    LRU_CLOCK_RESOLUTION;
    }
}
```

#### 在使用 LFU 策略时的评估机制

在使用 `LFU` 策略时，每次访问 `key`，都会调用 `updateLFU(val)` 更新 `lru` 字段，该字段保存了两个值，分别是 `ldt`（last decrement time，16 bit） 和 `logc`（logistic counter，8 bit）。

```c
/* Update LFU when an object is accessed.
 * Firstly, decrement the counter if the decrement time is reached.
 * Then logarithmically increment the counter, and update the access time. */
void updateLFU(robj *val) {
    // 计算衰减后的 logc
    unsigned long counter = LFUDecrAndReturn(val);
    // 根据计算得到的概率处理 logc 的自增
    counter = LFULogIncr(counter);
    // 更新 lru 字段
    val->lru = (LFUGetTimeInMinutes()<<8) | counter;
}
```

##### logc 的定义和更新

显然，`8 bit` 的 `logc` 虽然节省了内存空间，但是不足以存储真正的访问次数，在设计上它存储的是访问次数的对数，在实现上 `Redis` 通过“加权衰减”机制模拟了 `logc` 的对数增长。计算的方法定义在 `evict.c` 中

```c
/* Logarithmically increment a counter. The greater is the current counter value
 * the less likely is that it gets really implemented. Saturate it at 255. */
uint8_t LFULogIncr(uint8_t counter) {
    // 1. 以达到最大值则直接返回
    if (counter == 255) return 255;
    // 2. 计算一个随机概率值
    double r = (double)rand()/RAND_MAX;
    // LFU_INIT_VAL 默认值是 5
    double baseval = counter - LFU_INIT_VAL;
    if (baseval < 0) baseval = 0;
    // 3. 计算增加计数器的概率，其中 lfu_log_factor 默认值是 10
    double p = 1.0/(baseval*server.lfu_log_factor+1);
    // 4. 比较决定是否增加概率值
    if (r < p) counter++;
    return counter;
}
```

根据代码，我们可以知道，增加“频率计数器”的概率的计算公式如下：

$$
p = \begin{cases}
    0 & \text{if } logc = 255 \\\\
    1 & \text{if } logc \le lfu\\_init\\_val \\\\
    \frac{1}{(logc - lfu\\_init\\_val) * lfu\\_log\\_factor + 1} & \text{if } lfu\\_init\\_val < logc < 255
    \end{cases}
$$

其中，`LFU_INIT_VAL` 是在使用 `LFU` 策略时用于初始化的默认值，默认为 `5`；`lfu_log_factor` 是 `LFU` 的频率计数器的对数增长系数，默认为 `10`。

##### ldt 的定义和更新

`ldt` 的全称是 last decrement time，意思是 `logc` 最后衰减的时间。由于每次在访问时，都会进行衰减处理，因此 `ldt` 也具备“最后访问时间”的含义。

> 在和 Chat GPT 吧啦吧啦的时候，对于衰减的时机一度产生了困惑，到底是访问时、定期的还是进行内存淘汰时？看代码到底是最靠谱的。

但是和使用 `LRU` 策略时不同，由于存储 `ldt` 的位数只有 `16` 位，计算的方式变成以下这样，取分钟级时间戳对 `65535` 取余。

```c
/* Return the current time in minutes, just taking the least significant
 * 16 bits. The returned time is suitable to be stored as LDT (last decrement
 * time) for the LFU implementation. */
unsigned long LFUGetTimeInMinutes(void) {
    return (server.unixtime/60) & 65535;
}
```

“随时间衰减”的处理逻辑定义在 `evict.c` 中的 `LFUDecrAndReturn` 方法，它会根据当前时间和 `ldt` 的差值以及配置项 `server.lfu_decay_time` 决定是否对 `logc` 进行衰减计算，然后返回最终的结果。`server.lfu_decay_time` 的含义是每个 `n` 分钟衰减 `1` 吗？Chat GPT 又迷惑到我了。当取值为 `0` 时不衰减。

```c
unsigned long LFUDecrAndReturn(robj *o) {
    // 获取 ldt 和 counter
    unsigned long ldt = o->lru >> 8;
    unsigned long counter = o->lru & 255;
    // 计算衰减周期数
    unsigned long num_periods = server.lfu_decay_time ? LFUTimeElapsed(ldt) / server.lfu_decay_time : 0;
    if (num_periods)
        counter = (num_periods > counter) ? 0 : counter - num_periods;
    return counter;
}
```

计算过去了多长时间，以分钟为单位，最多考虑一次时钟折返。

```c
/* Given an object last access time, compute the minimum number of minutes
 * that elapsed since the last access. Handle overflow (ldt greater than
 * the current 16 bits minutes time) considering the time as wrapping
 * exactly once. */
unsigned long LFUTimeElapsed(unsigned long ldt) {
    unsigned long now = LFUGetTimeInMinutes();
    if (now >= ldt) return now-ldt;
    return 65535-ldt+now;
}
```

##### 不常访问程度的计算

在使用 `LFU` 策略中，不常访问程度（尽管变量名仍为 `idle`）的计算方式如下。注意：在内存淘汰时，尽管计算了衰减后的 `logc`，但是并未更新 `lru` 字段。

```c
idle = 255-LFUDecrAndReturn(o);
```

#### 在使用 TTL 策略时的评估机制

在使用 `TTL` 策略时，计算距离过期远近程度的计算方式如下，只需要 `db->expires` 中存储的过期时间，

```c
idle = ULLONG_MAX - (long)dictGetVal(de);
```

## 总结和感受

常见的内存淘汰算法并不难记忆和理解，`Redis` 的优化思路也并不复杂，但是我们难免会对黑盒中的一些细节产生好奇心，又会因为网上一些简化且不太准确的信息而产生困扰。这些问题，有时候如同鸡肋食之无味又弃之可惜，不探索无伤大雅但想起来又如鲠在喉，但是查看并理解相应源代码后豁然开朗的感受却是非常真实的。
深入探究 `Redis` 内存淘汰的具体实现，既可以帮助我们了解一个工业界成熟的内存淘汰算法实现方案是如何的，也可以了解到一些资料中一笔带过的信息背后的细节。