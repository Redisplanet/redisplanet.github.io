---
layout: post
title: "Under the Hood of Redis: Hash [Part 1]"
modified:
categories: Redis
description: What ziplist and dict in Redis. When and for what purpose. How much RAM occupied by these structures. When hash is stored in ziplist, when dicth and that it gives us. How this internals works.
tags: ["nosql", "redis", "algorithms", "big data", "hash", "dictionary"]
image:
  feature:
  credit:
  creditlink:
comments:
share:
date: 2016-01-02T10:44:42+00:00
---
Do you know why after `hset mySet foo bar` we spend not less than 296 bytes of RAM in Redis? Why instagramm engineers do not use string keys? Why change the `hash-max-ziplist-entries`/`hash-max-ziplist-val` and why the type of data that underlies HASH this part of the LIST, SORTED SET, SET? Understanding the design of hash tables in Redis is critical when writing systems where RAM saving is important.

Content in few lines:

- Any costs on Redis key storage.
- What ziplist and dict. When and for what purpose. How much RAM occupied by these structures. When hash is stored in ziplist, when dicth and that it gives us.
- What advice from fashion articles about Redis optimizing should not be taken seriously, and why.

First, we need to deal with how to operate the individual structures that underlie Hashes. Then calculate the cost of data storage. You will be easier to understand the material, if you know what is redisObject and [how Redis stores strings](http://redisplanet.com/redis/under-the-hood-of-redis-strings/).

<!--more-->
There is a certain confusion that hashes is perceived as a hash table. Under the hood there is a separate type dict, which is a real hash table. The very same structure hashes hybrid of dict and ziplist. We need a big entry on how to construct and work a hash table.

In Redis you control the way in which internal data type is used in hash key. Setting `hash-max-ziplist-entries` defines the maximum number of elements in the hash in which encoding **REDIS_ENCODING_ZIPLIST** is used. For example, in the hash-max-ziplist-entries = 100 your hash will be presented as a ziplist, until there is less than 100 members. As soon as the elements become larger, it will be converted to **REDIS_ENCODING_HT** (dict). Similarly running `hash-max-ziplist-val` is the length of any value of any particular value in the hash does not exceed` hash-max-ziplist-val` be used ziplist. Settings work in pairs - an internal representation from the zip list to dict happen as soon exceeded any of them. By default, any hash always has a coding ziplist.

Dictionary
---

Dictionary (dict) - special structure of Redis. Not only for hashes, but also for the list, zset, set. This basic structure, which is used in Redis to store any kind of data bundles key - value. Dictionaries in Redis - a classic hash table that supports insert, replace, delete, search, and get a random item. The whole implementation can be found in dict.h and disc.c in source Redis.

At any hash table has a number of parameters that are critical for understanding the effectiveness of its work and the choice of the correct implementation. Not only the **algorithmic complexity** but also number of factors that are directly dependent on the usage conditions. In this vein, the **fill factor** and **resizing strategy** are especially interesting. The strategy of resizing important in any system operating in real time, as any of the available theoretical implementations always puts you in front of a choice - spend a lot of memory or CPU time. Redis uses a [increment resizing](https://en.wikibooks.org/wiki/Data_Structures/Hash_Tables).

>It would be interesting to be able to choose from a linear hashing, monotonic keys, consistent hashing, resizing by copying depending on the specific hash.

The Redis implementation (function dictRehash Milliseconds, dict Rehash in dict.c), is to ensure that:

1. When you change the size of the table, we allocate a new hash table clearly greater than the already existing (and do not change the old). As with strings, Redis will double the number of slots in the new table. However, the size (including the original) will always be aligned up to the nearest power of two. For example, you need 9 slots been allocated - 16. The minimum slot count for the new table is defined by a compile time constant **DICT_HT_INITIAL_SIZE** (default 4, defined in dict.h).
2. During each read / write operations look in both tables.
3. All insert operations are carried out only in the new table.
4. When any operation move `n` items from the old table into the new.
5. Remove the old table if all the elements transferred.

In the case of Redis, `n` - is the number of keys that the server should transfer for 1 millisecond (in increments of 1000). In other words, one step of rehashing - is at least 1000 elements. This is important because when you use the keys with long string values such operation can significantly exceed 1 ms, causing the server freeze.

> An additional aspect - large peak memory consumption when rehashing table. This is particularly noticeable on heshes with a large number of "long" values. When the calculations we will consider the table outside of rehashing state. Remember, however, that if a node in our Redis has a large table that has a high probability of failure of the service if you can not resize. For example, if you are renting on a budget Redis instance on heroku (only 25 MB of memory).

The fill factor (constant `dict_force_resize_ratio` default is 5) determines the sparse factor in the dictionary and then when Redis will begin the process of doubling the size of the current dictionary. The current value of the fill factor is defined as the ratio of the total number of elements in the hash table to the number of slots. For example, if we only have 4 slots and 24 elements (distributed among them) Redis decides to double the size (24/4> 5). This check occurs for each access to key to any dictionary. If the number of elements is equal to or exceeds the number of slots - Redis also try to double the number of slots.

![dict]({{ site.url }}/images/redis/dict.png)

{% highlight c %}
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int iterators; /* number of iterators currently running */
} dict;

typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
{% endhighlight %}

Each dictionary is built around the use of the three structures - `dict`, `dictht` and `dictEntry`:

* `Dict` - root structure in its field `ht` kept 1 or 2 (during rehashing) `dictht` table with a list of slots.
* In turn, `dictht` keeps a linked list of `dictEntry`.
* Each `dictEntry` keeps a pointer to the key and value (in the form of pointer or a double / uint64_t / int64_t).

> If you use a number as the value - `dictEntry` will store the number, store string - will be kept a pointer to a `redisObject` with `sds` string on board.

When Redis refers to a specific key in a hash table:

1. The required slot is located using a hash function (currently used [MurMur2] (https://ru.wikipedia.org/wiki/Murmur2) hash function).
2. All values of given slot compating one by one (like in linked list).
3. Redis performs the requested operation on finded `dictEntry` (with the key and value).

In reference to [HSET] (http://redis.io/commands/hset) or [HGET] (http://redis.io/commands/hget) says that this operation is performed in **O(1)**. Believe it with caution. Large tables (more than 1000 elements) with an active recording hget also will occupy most **O(n)**.

> The answer to a question how many memory it will be used actually depends on an operating system, the compiler, CPU and the used memory allocator (jemalloc used in Redis by default). I give all further calculations for Redis 3.0.5 compiled on the 64th bit server under control of centos 7.

Now you can calculate how much memory is wasted when you call `set mySet foo bar`. Overhead to create an empty dictionary:

    dictEntry: 3 * size_of(pointer)
    dictht: 3 * size_of(unsigned longs) + size_of(pointer) = 24 + size_of(pointer)
    dict: 2 * size_of(dictht) + 2 * size_of(int) + 2 * size_of(pointer) = 56 + 4 * size_of(pointer)

Putting it all together, we get the formula for calculating the approximate overhead for storing `n` elements:

	56 + 2 * size_of(pointer) + 3 * next_power(n) * size_of(pointer)
	-------------------------    -----------------------------------
	      |	                                      |
          ` dict + dictht                         `dictEntry


This formula does not account for the cost of storage of keys and values itself. The key is always a `sds string` in the `redisObject`. The value according to the type may be a sds string or numbers (integer or float). Check this for `hset mySet foo bar`, remembering about the fact that the minimum number of slots (`dictEntry`) in the new dictionary still **DICT_HT_INITIAL_SIZE** (4):

    56 + 2 * 8 + 3 * 4 * 8 = 168 + 2 * 56 = 280 bytes (would be aligned up to 296 bytes).

Check it:

    config set hash-max-ziplist-entries 0
    +OK
    config set hash-max-ziplist-value 1
    +OK
    hset mySet foo bar
    :1
    debug object mySet
    +Value at:0x7f44894d6340 refcount:1 encoding:hashtable serializedlength:9 lru:5614464 lru_seconds_idle:6

What does the info memory? It will show you that spent 320 bytes. On 24 bytes more than we expected. This memory - the cost of alignment when allocating memory.

Dict in redisDb internals
---

As saved key `mySet` itself? How did Redis find our hash by this name?

In the heart of Redis is located the structure named `redisDb` (redis database representation, in redis.h). It contains a single dictionary for all keys in Redis instance. Just the same, as discussed above.

This is important as gives an idea of the costs of storage of key database and will give us a basis for understanding the tips that in one instance it is not necessary to store a lot of simple keys.

Let's see what is written in [storing hundreds of millions of simple key value](http://instagram-engineering.tumblr.com/post/12202313862/storing-hundreds-of-millions-of-simple-key-value). If you need to store a lot of key-value pairs do not use string keys, use a hash table.

[Gist with the test] (https://gist.github.com/mikeyk/1329319) of Article instagram, we will write to the LUA, was not required to test for anything except running Redis:

    flushall
    +OK
    info memory
    $225
    # Memory
    used_memory:508408

    eval "for i=0,1000000,1 do redis.call('set', i, i) end" 0
    $-1
    info memory
    $226
    # Memory
    used_memory:88578952

For storage of **1,000,000** integer keys we needed a little more than **88 MB**. Now save the same data in the hash table, evenly distributing keys between 2000 hash tables:

    flushall
    +OK
    info memory
    $227
    # Memory
    used_memory:518496

    eval "for i=0,1000000,1 do local bucket=math.floor(i/500); redis.call('hset', bucket, i, i) end" 0
    $-1
    info memory
    $229
    # Memory
    used_memory:104407616

For storage of the same data with integer fields, we spent a little more than **103 Mb**. However, we should try to keep the key 10,000,000 say the situation has changed: **~970 mb** (simple keys) and **~165 mb**(hash). All of difference in the overhead  of redisDb key hash table.

> On the whole, the rule of **"if you have hundreds of millions of keys, do not use string keys"** - the truth.

Let's look at another aspect. Very often describing the optimization of this kind when it is implied that you have a lot of key species essence of identification (such as `set username: 4566 RedisMan`) and offer to upgrade to a hash table `bucket:id` ID value (eg `hset usernames: 300 4566 RedisMan`).

It is important to understand that there is a **partial substitution of concepts** - sds string `username: 4566` turns in the key 4566 - encoded as *REDIS_ENCODING_INT*. This means that instead of a sds string the redisObject started using the numbers thus minimal for sds 32 bytes per string (after alignment jemalloc) turned into 4 bytes.

Let we force the encoding of hash tables in **ziplist**:

    config set hash-max-ziplist-entries 1000
    +OK
    config set hash-max-ziplist-value 1000000
    +OK
    flushall
    +OK
    eval "for i=0,1000000,1 do local b=math.floor(i/500); redis.call('hset', 'usernames:' ..b, i, i) end" 0
    $-1
    info memory
    $228
    # Memory
    used_memory:16816432

On the storage it took only **16 MB** or **72 MB** economy (5 times less for 1,000,000 keys). Tempting? This will be discussed in the second part of the article.


The interim conclusion
---

The interim conclusion is to formulate some important conclusions for the loaded system to save memory:

* Using the numeric key names, the values of fields in the hash tables wherever possible. Do not use the prefix / postfix.
* When designing systems that actively use Redis, proceeding from the principle: one set of data requirements - one instance of Redis. Keep heterogeneous data is difficult because of per instance settings `hash-max-ziplist-entries` / `hash-max-ziplist-value` and differentiation of keys without prefixes.
* Replacing the simple keys to hash tables, remember that your optimization work when the number of keys from a million or more.
* If your instance of Redis stores hundreds of millions of keys - you carry the huge cost of keeping them in the system dictionary and a memory overrun it. For example, for 100 million keys it will be about 2.5 GB for the redisDb dict excluding your data.
* If you pass a value of hash-max-ziplist-entries / hash-max-ziplist-value and your data is stored in ziplist, instead of dict you can get savings of memory, expressed in hundreds of percent. And pay for it using a high CPU usage.

The materials used in writing this article:

- [Redis 3.0.5](https://github.com/antirez/redis/tree/3.0.5/src) sources
- http://redis.io/topics/memory-optimization
- [Back To Basics: Hashtables Part 2](http://openmymind.net/Back-To-Basics-Hasthables-Part-2/) by [Karl Seguin](https://twitter.com/karlseguin)
- https://en.m.wikipedia.org/wiki/Hash_tablefor 
- [Redis Internal Data Structure: Dictionary](http://blog.wjin.org/posts/redis-internal-data-structure--dictionary.html) by [Wei Jin](https://www.linkedin.com/in/wjin0)
- https://github.com/sripathikrishnan/redis-rdb-tools/wiki/Redis-Memory-Optimization
- http://instagram-engineering.tumblr.com/post/12202313862/storing-hundreds-of-millions-of-simple-key-value
- [Great answer about speed and problems with collisions in different hashing algorithms](http://programmers.stackexchange.com/questions/49550/which-hashing-algorithm-is-best-for-uniqueness-and-speed/145633#145633)
