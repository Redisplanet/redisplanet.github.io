---
layout: post
title: "Under the Hood of Redis: Strings"
modified:
categories: Redis
description: How strings are stored in Redis. What internal structures are used strings. What types of optimization are used by Redis under a cowl.
tags: ["nosql", "redis", "algorithms", "big data", "strings"]
image:
  feature:
  credit:
  creditlink:
comments:
share:
date: 2015-12-22T10:37:01+00:00
---

Do you know why the simple string `strings` in Redis will occupy 56 bytes of RAM?

I wil try to tell you why this so in Redis. And why it is so important to understand how Redis work under the cover. This is especially important if you try to calculate the actual consumption of memory or planing to build high loaded application. Or, as often happens, you try to understand urgently why your Redis instance began to consume unexpectedly a lot of memory.

Content in few lines:

* How strings are stored in Redis.
* What internal structures are used strings.
* What types of optimization are used by Redis under a cowl.
* How effectively to store big structures and in what situations you shouldn't use strings or structures constructed on their basis.

Strings is a most used structure in Redis. HSET/ZSET/LIST are constructed on their base adding a small overhead on representation of the internal structures. Over a year I read and actively answer on [stackoverflow](http://stackoverflow.com/) in Redis tag. On an extent of this time I constantly see not stopping stream of questions which is shown that much of developers don't understand RAM features of Redis and that price Redis pays for the extremely high speed. This is the first article in the series, explaining how Redis works inside.
<!--more-->
> The answer to a question how many memory it will be used actually depends on an operating system, the compiler, CPU and the used memory allocator (jemalloc used in Redis by default). I give all further calculations for Redis 3.0.5 compiled on the 64th bit server under control of centos 7.

It seems to me that here the small interlude is absolutely necessary for those who doesn't write on c/c++ or isn't really well familiar with that as everything works at a low level. Let's simplify terribly strongly some concepts that it was simpler to you to understand calculations. When in the program on c/c++ you declare structure, and in it you have field unsigned int (4 bytes) the compiler will carefully align their size to 8 bytes in RAM (for x64 architecture). In it to article will be periodically it is told about a memory allocator — this such piece which marks out memory "on clever". For example, jemalloc tries to optimize for you the speed of search of new blocks of memory, staking on alignment of the allocated fragments. Strategy of allocation and alignment of memory in jemalloc is well described, however I think that we should use simplification that any size of the allocated fragment will be rounded to the next degree 2. You ask 24 bytes — will allocate 32. You ask 61 — will allocate 64. I strongly simplify and I hope to you it will be a little more clear. These are those things which in the interpreted languages, logically, shouldn't excite you, however here very much I ask you to pay to them the attention.

The concept and realization of strings belongs to Salvatore Sanfilippo's (aka antirez) and placed in Redis sub project under the name SDS (Simple Dynamic String, [github.com/antirez/sds](http://github.com/antirez/sds)):


    +--------+-------------------------------+-----------+
    | Header | Binary safe C alike string... | Null term |
    +--------+-------------------------------+-----------+
             |
             `-> Pointer returned to the user.

It is a simple `c` structure, whose header stores the actual size and the empty space in already allocated memory, the string data itself and terminating zero. The sds strings we are most interested in is the cost of a header, the resize strategy and structures alignment penalty when allocating memory.

July 4th 2015 ended with [a long history with the optimization of sds strings](https://github.com/antirez/redis/pull/2509), which is to get to the Redis 3.2. This optimization will bring big RAM savings in sds headers (from 16% to 200% for synthetic tests). Remove restrictions in the 512MB to a maximum length of string in Redis. All this will be possible thanks to the dynamic length of the header when changing the length of the string. So the header will take only 3 bytes for strings with a length up to 256 bytes, 5 bytes for strings less than 65 kb, 9 bytes (as now) for strings up to 512 MB, and 17 bytes for strings whose size "intermeddle" in uint64_t (64 bit unsigned integer). By the way, this change in our Redis server farm will save about 19.3% of memory (~ 42 GB). However, in Redis 3.0.x everything is simple - 8 bytes + 1 byte to the terminating zero. Let's estimate how much memory takes a string «strings»:

    16 (header) + 7 (string length) + 1(trailing zero) = 24 bytes (16 bytes in the header, because the compiler will align 2 unsigned int for you).

And jemalloc allocate 32 bytes for you. Let's take as long as it will not be taken into account (I hope it will be understood later why).

What happens when a string changes the size? Whenever you increase the size of the string and has already allocated memory is not enough, Redis compares the new length with constant `SDS_MAX_PREALLOC` (defined in sds.h and is 1,048,576 bytes).
 * If the new length is less than this value will be allocated memory twice requested.
 * If the string length exceeds `SDS_MAX_PREALLOC` - the new requested length will be added to the value of this constant.

This feature is important in the story "about the disappearing memory at the use of bitmaps». By the way, when allocating memory for the bitmap it will always be allocated to 2 times more than requested, because of the peculiarities of realization SETBIT (see setbitCommand in bitops.c).

Now you could say that our line will take in 32 bytes of RAM (including alignment). Those who read the tips from hashedin.com ([redis memory optimization guide](https://github.com/sripathikrishnan/redis-rdb-tools/wiki/Redis-Memory-Optimization)) might recall that they are strongly advised not to use strings length of less than 100 bytes, as to hold short line, for example by using the command `set foo bar` you spend ~ 96 bytes of which 90 bytes of overhead is (64 bit machine). Telling the whole story? Let's deal further.

All values in Redis stored in structure named `redisObject`. This allows Redis to know the type of value, its internal representation (named encoding), the data for the LRU, the refCount and the value itself:

    +------+----------+-----+----------+-------------------------+
    | Type | Encoding | LRU | RefCount | Pointer to  data (ptr*) |
    +------+----------+-----+----------+-------------------------+

Later we calculate its size for our string, taking into account compiler alignment and  jemalloc features. It is very important to know what the encoding used to store strings. Right now Redis uses three different storage strategies:

- **REDIS_ENCODING_INT**. Strings can be stored in this form, if the value is cast to long value in the range **LONG_MIN**, **LONG_MAX**. For example, the string «dict» it will be stored in the form of this encoding, and will be the number 1952672100 (0x74636964). This encoding is also used for pre-selected range of special values in the range **REDIS_SHARED_INTEGERS** (defined in redis.h and the default is 10000). The values of this range are allocated immediately at the start of Redis.
- **REDIS_ENCODING_EMBSTR** used for strings with a length up to 39 bytes (the value from constant **REDIS_ENCODING_EMBSTR_SIZE_LIMIT** object.c). This means that redisObject structure and sds string structure are placed in a single area of memory allocated by allocator. With this in mind, we will be able to calculate the correct alignment. However, it is equally important to understand the problem of memory fragmentation in the Redis and how to live with it.
- **REDIS_ENCODING_RAW** used for all strings whose length exceeds **REDIS_ENCODING_EMBSTR_SIZE_LIMIT**. In this case our ptr * stores a pointer to the memory area with sds string.

> EMBSTR appeared in 2012 and brought a 60-70% increase in performance with short strings, but serious research on the impact on memory and its fragmentation is not so far.

The length of our string "strings" of 7 bytes, ie type its internal representation - EMBSTR. Thus created a string placed in the memory like this:

    +--------------+--------------+------------+--------+----+
    | robj data... | robj->ptr    | sds header | string | \0 |
    +--------------+-----+--------+------------+--------+----+
                         |                       ^
                         +-----------------------+


Now we are ready to calculate how much memory is required to store our string "strings" in Redis:

    (4 + 4)* + 8(encoding) + 8 (lru) + 8 (refcount) + 8 (ptr) + 16 (sds header) + 7(strig itself) + 1 (terminating zero) = 56 bytes.

>*The type and value in redisObject uses only the 4 lower and higher bits in the same number, so these two aligned fields will take 8 bytes.

Let's check that I'm not kidding. Let's the encoding and meaning. We use a DEBUG SDSLEN command to debug sds string. By the way, this command [No official documentation] (http://redis.io/commands/debug-object) was added to the [redis 2.6] (https://raw.githubusercontent.com/antirez/redis/2.6 / 00-RELEASENOTES) and can be very useful:

    set key strings
    +OK
    debug object key
    +Value at:0x7fa037c35dc0 refcount:1 encoding:embstr serializedlength:8 lru:3802212 lru_seconds_idle:14
    debug sdslen key
    +key_sds_len:3, key_sds_avail:0, val_sds_len:7, val_sds_avail:0

Encoding used - embstr, string length 7 bytes (val_sds_len). What about those 96 bytes are talking about the guys from hashedin.com? In my understanding, they are a little wrong, they are an example of a `set foo bar` require the allocation of 112 bytes of RAM (56 bytes to the value and the same number on the key), of which 106 - overhead.

I promised a story about a vanishing memory when using BITMAP. The feature that I want to tell you, is constantly flowing out of the attention of the developers using it. The family of «Bit and byte level operations» commands appeared in Redis 2.2 and immediately positioned as a stick wand for real time counter (for example, the article by [Spool] (http://blog.getspool.com/2011/11/29/fast-easy-realtime-metrics-using-redis- bitmaps /)), which saves memory. The official guide on memory optimization also has a slogan on the use of this family of data for online storage "for 100 million users, this data will occupy a total of 12 megabytes of RAM."

The description [SETBIT](http://redis.io/commands/setbit)/[SETRANGE](http://redis.io/commands/setrange) warn of possible lags the server memory allocation, omitting important, I think, see "When you do not use BITMAP» or "When to use SET instead BITMAP».

Armed with an understanding of how to work strings in Redis can be seen that the bitmap:
* Should not be used for sparse data.
* Understand the ratio of useful and actual load (this in the example below).
* Take into account the dynamics of filling up your bitmap.

Consider an example. Suppose that you have recorded up to 10 million people and your ten million users went online:

    setbit online 10000000 1
    :0
    debug sdslen online
    +key_sds_len:6, key_sds_avail:0, val_sds_len:1250001, val_sds_avail:1048576

Your consumption amounted to 2,298,577 bytes of RAM, with "useful" to you 1,250,001 bytes. Storage of your one user cost you ~ 2.3 Mb. Using SET you would need ~ 64 bytes (4 bytes in the payload). It is necessary to choose the aggregation intervals so that reduce the sparseness of data and try to fill bitmap lies in the range of 30% - in this case, you really will efficiently use this memory for the data structure. I say this to the fact that if you have a multi-million audience and watch online say 10,000 - 100,000 is used for this purpose bitmap can be overlaid on the memory.

Finally, resizing strings in Redis - a constant reallocation of memory blocks. Memory fragmentation is another specificity of Redis, which the developers give little thought.

    info memory
    $222
    # Memory
    used_memory:506920
    used_memory_human:495.04K
    used_memory_rss:7565312
    used_memory_peak:2810024
    used_memory_peak_human:2.68M
    used_memory_lua:36864
    mem_fragmentation_ratio:14.92
    mem_allocator:jemalloc-3.6.0


`mem_fragmentation_ratio` metric shows the RAM allocation by operating system (`used_memory_rss`) and the memory used by Redis (`used_memory`). This `used_memory` and `used_memory_rss` already will include both the data and the cost of storing the internal structures of Redis for storage and presentation. Redis RSS (`Resident Set Size`) - RAM allocated by the operating system, which in addition to the user data (and the costs of their internal representation) accounted for the cost of fragmentation during the physical allocation of the operating system.

How to understand `mem_fragmentation_ratio`? The value of 2.1 tells us that we use 210% more memory for storing data than we need. A value less than 1 indicates that the memory is ended and the operating system swapping.

In practice, if the values fall outside the boundaries of mem_fragmentation_ratio 1 - 1.5 says that you have something wrong. Try this:

- Restart your Redis. The longer you actively write to Redis without rebooting, the higher you will `mem_fragmentation_ratio`. In many ways, "thanks to" allocator. In particular, it is guaranteed to help you if you have a large difference between `used_memory` and `used_memory_peak`. Last indicator says what is the maximum amount of memory that is ever needed your copy of Redis since its launch.
- See what data and for how much you plan to store. For example, if your data is enough 4 GB - use the 32-bit build of Redis. At least, if you are using a 64 bit assembly of Redis try to expand your dump on a 32-bit version of Redis (rdb is independent of Redis build and you can easily run rdb created on 64 bit instance to 32-bit). Almost guaranteed it reduces fragmentation (and memory usage) on a ~ 7% (due to economies of alignment).
- If you understand the difference, and features try to change the allocator. Redis can be compiled with the glibc malloc, jemalloc (read [facebook research about jemalloc](https://www.facebook.com/notes/facebook-%C2%AD%E2%80%90engineering/scalable-%C2%AD%E2%80%90memory-%C2%AD%E2%80%90allocation-%C2%AD%E2%80%90using-%C2%AD%E2%80%90jemalloc/480222803919)), tcmalloc.

In speaking of fragmentation, I do not consider the specifics of Redis when the LRU or other challenges when a large number of ordinary string key - all drawn to a separate article. I would be grateful if you share your suggestions about whether to write about it and what else you think is important when working with Redis.

Additional interesting materials:

- http://redis.io/topics/memory-optimization
- http://redis.io/topics/internals-sds
- http://redislabs.com/blog/redis-ram-ramifications
- http://github.com/sripathikrishnan/redis-rdb-tools/wiki/Redis-Memory-Optimization
