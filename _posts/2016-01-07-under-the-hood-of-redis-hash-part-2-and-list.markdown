---
layout: post
title: "Under the Hood of Redis: Hash [Part 2] & List"
modified:
categories: Redis
description: Hash table is a bit LIST, SET and SORTED SET in Redis. Judge for yourself - LIST is made up of ziplist/linkedlist, SET consists of dict/intset, and SORTED SET is ziplist/skiplist. We have already discussed Dictionary (dict), and in the second part of the article we consider the structure of ziplist and linkedlist.
tags: ["nosql", "redis", "algorithms", "big data", "ziplist", "skiplist"]
image:
  feature:
  credit:
  creditlink:
comments:
share:
date: 2016-01-07T10:10:11+00:00
---

In the [first part](http://redisplanet.com/redis/under-the-hood-of-redis-hash-part-1/){:target="_blank"} I said that the hash table is a bit **LIST**, **SET** and **SORTED SET**. Judge for yourself - LIST is composed of *ziplist* / *linkedlist*, SET consists of *dict* / *intset*, and SORTED SET is *ziplist* / *skiplist*. We have already discussed Dictionary (dict), and in the second part of the article we consider the structure of ziplist - the second most common structure is applicable under the hood of Redis. Look at LIST - the second part of his "kitchen" is a simple implementation of a linked list. It is useful to us to carefully consider the often mentioned advice about the optimization of hash table by replacing them on the list. Calculate how much memory is required for overhead costs by using these structures, what price you pay for saving memory. To sum up when working with hash tables, using the ziplist encoding.

The last time we ended on that saved using ziplist **1,000,000*** keys took *16 MB* of RAM, while the dict the same data required *104 mb* (**ziplist 6 times smaller**). Let's understand what the price:
![crazy]({{ site.url }}/images/redis/crz.jpg){: .image-center}
<!--more-->
Ziplist
---

So, ziplist - is doubly linked list. References in each node point to a previous and for the next node in the list. In the doubly linked list can be effectively move in any direction - both to the head and to the tail. In this list easier to make removal and rearrangement of the elements, as readily available addresses of the elements of the list, the pointers are aimed to required element.

Redis developers positioning their implementation as an effective in terms of RAM. List can store strings and numbers. When this number is stored in the form of numbers, rather than the value in redisObject. And if you want to save the string "123", it will save the number 123 instead of the character sequence `1`,` 2`, `3`.

> In the 1.x branch of Redis instead of `dict` in the dictionary used `zipmap` - uncomplicated (~ 140 lines) implementation of a linked list that is optimized to save memory, where all the operations take **O(n)**. This structure is not used in Redis (though it is in the source codes, and kept up to date), the part `zipmap` ideas form the basis `ziplist`.

Keys and values are stored as arranged one after the other list items. Operations on the list - is the search key by brute force and work with a value that is located in the next list item. Theoretically, inserts and updates are performed in constant **O(1)**. In fact, any such operation in the implementation of Redis requires allocation and reallocation of memory and a real difficulty depends on the amount already used RAM. In this case, we remember about the **O(1)** as an ideal case and **O(n)** - as the worst.

    +-------------+------+-----+-------+-----+-------+-----+
    | Total Bytes | Tail | Len | Entry | ... | Entry | End |
    +-------------+------+-----+-------+-----+-------+-----+
                                    ^
                                    |
    +----------------+------------+---------+-----+------------+----------+-------+
    | PrevRawLenSize | PrevRawLen | LenSize | Len | HeaderSize | Encoding | Value |
    +----------------+------------+---------+-----+------------+----------+-------+

{% highlight c %}
#define ZIPLIST_HEADER_SIZE  (sizeof(uint32_t)*2+sizeof(uint16_t))

typedef struct zlentry {
    unsigned int prevrawlensize, prevrawlen;
    unsigned int lensize, len;
    unsigned int headersize;
    unsigned char encoding;
    unsigned char *p;
} zlentry;
{% endhighlight %}

Calculating ziplist size
---

> The answer to a question how many memory it will be used actually depends on an operating system, the compiler, CPU and the used memory allocator (jemalloc used in Redis by default). I give all further calculations for Redis 3.0.5 compiled on the 64th bit server under control of centos 7.

With the header all is simple - *ZIPLIST HEADER_SIZE* (a constant defined in ziplist.c and equal to **sizeof (uint32_t) * 2 + sizeof (uint16_t) + 1**). Cell Size - **5 * size_of (unsigned int) + 1 + 1** (not less than 1 byte value). The total overhead for storage of data in a coding for n items (without values):

    12 + 21 * n

Each item begins with a fixed header which consists of several pieces of information. The first - the size of the previous element and is used to move backward through the doubly linked list. Second, with optional encoding length of the structure in bytes. The length of the previous item is as follows: if the length does not exceed 254 bytes (the same constant *ZIP_BIGLEN*) will be used 1 byte to store the length value. If the length is greater than or equal to 254, 5 bytes will be used. At the same time the first byte is set to a constant value *ZIP_BIGLEN*, that we understand that we have a long value. The other 4 bytes store the length of the previous value. The third - the header, which depends on the values contained in the item. If the string value - the first 2 bytes of header will store the encoding type for the length of the string, followed by a number with a length of string. If the digit - integer, the first 2 bits will be on display in the unit. For numbers 2 more bits used to determine which dimension it is stored in the unit.

Here is encoding table from Redis sources:

|Internals |Bytes | About |
|:---------|:-----:|------|
|0pppppp |	1  |	String length <= 63 bytes (ie. **REDIS_NCODING_EMBSTR** encoded sds string, [see this post](http://redisplanet.com/redis/under-the-hood-of-redis-strings/){:target="_blank"}) |
|01pppppp&#124;B<sup>1&nbsp;byte</sup> |	2	| String length >= 16383 bytes (14 bit)|
|10______&#124;BBBBB<sup>4&nbsp;bytes</sup>|	5 |	String length >= 16384 bytes
|11000000|	1 |	Integer, int16_t (2 bytes) |
|11010000|	1 |	Integer, int32_t (4 bytes) |
|11100000|	1 |	Integer, int64_t (8 byte) |
|11110000|	1 |	Signed integer, «fit» in 24 bits (3 bytes)
|11111110|	1 |	Signed integer, «fit» in 8 bit (1 byte)
|1111xxxx|	1 |	Where xxxx is between 0000 and 1101 is an integer, that "fit" in 4 bits. Unsigned integer from 0 to 12. The decoded value on the fact of from 1 to 13, for 0000 and 1111 are already occupied and we always subtract 1 from the decoded values to get the desired result. |
|11111111|	1 |	Marker "end of list"
{: rules="groups"}

Cost of victory
---

Check and look at the reverse side of the medal. We will only use LUA, so you do not need anything other than Redis, to replicate the test on their own. First, look what happens when you use `dict`.

Executing

    multi
    time
    eval "for i=0,10000,1 do redis.call('hset', 'test', i, i) end" 0
    time

and

    multi
    time
    eval "for i=0,10000,1 do redis.call('hget', 'test', i) end" 0
    time

> Gist with [full log here](https://gist.github.com/misterion/ee59980b852db2c63f48){:target="_blank"}.

In the case of encoding:hashtable (dict) spent **~516 kb** of memory and **18.9 ms** to save **10,000** items, **7.5 ms** read them. With encoding:ziplist get **~81 kb** of memory, **710 ms** to save and **705 ms** to read. For the test with even 10,000 entries received:

> **The decrease of RAM 6 times the price of write speed subsidence of 37.5 times and 94 times in reading.**

It is important to understand that the  performance degradation is non-linear and even for 1,000,000 you risk not await for the results.

Who will put 10,000 or 1,000,000 items in ziplist? It is, unfortunately, one of the first recommendations from many consultants which i know. When the game is worth the candle? I would say that as long as the number of elem ents in the range 1 - 3500 items you can choose. RAM with ziplist always wins by 6 times higher then dict. Anything more ~3500 elements - measure on the real data. But  it will not have any relation with high loaded real-time systems.

That's what happens with the read / write performance, depending on the size of the hash in the dict and ziplist([gist with test](https://gist.github.com/misterion/78793fe1f536332dc0e0){:target="_blank"}):
![Dict and ziplist results]({{ site.url }}/images/redis/dict-read-write.png){: .image-center}

Why? Price of the insertion, resizing of the elements and removal from ziplist is monstrous - it or realloc (all work to go on the shoulders of allocator) or complete rebuild of the list of n + 1 from the modified element to the end of the list. Rebuilding - a lot of low fragment realloc, memmove, memcpy (see. ziplistCascadeUpdate function in the ziplist.c).

List
---

Why in the article about HASH important to talk about the LIST? The point is one very important council about optimizing structured data. The first I heard it from the [DataDog](https://www.datadoghq.com/){:target="_blank"}, but find it difficult to say exactly. I really like the explanation [Sripathi Krishnan](https://github.com/sripathikrishnan){:target="_blank"} from [HashedIn](http://hashedin.com/){:target="_blank"}:

> Lets say you want to store user details in Redis. The logical data structure is a Hash. For example, you would do this - `hmset user:123 id 123 firstname Sripathi lastname Krishnan location Mumbai`.
>
> Now, Redis 2.6 will store this internally as a Zip List; you can confirm by running debug object `user:123` and look at the encoding field. In this encoding, key value pairs are stored sequentially, so the user object we created above would roughly look like this:
>
> `["firstname", "Sripathi", "lastname", "Krishnan", "location", "Mumbai", "twitter", "srithedabbler"]`
>
> Now, if you create a second user, the **keys will be duplicated**. If you have a million users, well, its a big waste repeating the keys again.
>
> To get around this, we can borrow a concept from Python - [NamedTuples](http://docs.python.org/library/collections.html#collections.namedtuple){:target="_blank"}. A NamedTuple is simply a read-only list, but with some magic to make that list look like a dictionary.
>
> Your application needs to maintain a mapping from field names to indexes. So, "firstname" => 0, "lastname" => 1 and so on. Then, you simply create a list instead of a hash, like this - `lpush user:123 Sripathi Krishnan Mumbai srithedabbler`. With the right abstractions in your application, you can save significant memory.
>
> Don't use this technique if:
>
> * You have less than 50,000 objects
> * Your objects are not regular i.e. some users have lots of information, others very little.

**And this is very good advice**. Obviously, the list (LIST) will help you greatly save on memory - at least 2 times.

As HASH - using the `list-max-ziplist-entries` / `list-max-ziplist-val` you control how list type key will be stored in Redis internals. For example, if list-max-ziplist-entries = 100 yours LIST will be encoded as a *REDIS_ENCODING_ZIPLIST*, until there is less than 100 members. As soon as the number of elements will be more, it will be converted to *REDIS_ENCODING_LINKEDLIST*. Setting `list-max-ziplist-val` works similar to `hash-max-ziplist-val` (see. [The first part](http://redisplanet.com/redis/under-the-hood-of-redis-hash-part-1/){:target="_blank"}).

Linked list
---

Let's look *REDIS_ENCODING_LINKEDLIST*. In the implementation of Redis it is a very simple (~ 270 lines of code) not sorted linked list ([everything as it in Wikipedia](https://en.wikipedia.org/wiki/Linked_list){:target="_blank"}). With all its advantages and disadvantages:

{% highlight c %}
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
{% endhighlight %}

Nothing complicated - each LIST in what encoding - a 5 pointers and 1 unsigned long. So that the overhead is **5 * size_of (pointer) + 8 bytes**. And each node is a 3 pointer. Overhead on the storage of data of **n** elements are:

    5 * size_of(pointer) + 8 + n * 3 * size_of(pointer)

In the implementation of a linked list is not used realloc - only memory allocation under the desired fragment. In the values - redisObejct without any tricks and optimizations. By an overhead of internal structure - the difference between ziplist and linkedlist about 15%. If we consider this overhead and data value - at times (2 and higher) depending on the type of the value.

So for a list of **10,000** items, consisting only of numbers in the `ziplist` you spend about **41 KB** of memory and in `linkedlist` - **360 KB** real memory:

    config set list-max-ziplist-entries 1
    +OK
    flushall
    +OK
    info memory
    $226
    # Memory
    used_memory:511544
    eval "for i=0,10000,1 do redis.call('lpush', 'test', i) end" 0
    $-1
    debug object test
    +Value at:0x7fd9864d6530 refcount:1 encoding:linkedlist serializedlength:29877 lru:6387397 lru_seconds_idle:5
    info memory
    $226
    # Memory
    used_memory:832024
    config set list-max-ziplist-entries 10000000
    +OK
    flushall
    +OK
    info memory
    $226
    # Memory
    used_memory:553144
    eval "for i=0,10000,1 do redis.call('lpush', 'test', i) end" 0
    $-1
    info memory
    $226
    # Memory
    used_memory:594376
    debug object test
    +Value at:0x7fd9864d65e0 refcount:1 encoding:ziplist serializedlength:79681 lru:6387467 lru_seconds_idle:16

Let's look at the difference in performance when writing and receiving element through [LPOP](http://redis.io/commands/lpop){:target="_blank"}(to read and delete). Why not [LRANGE](http://redis.io/commands/lrange){:target="_blank"} - its complexity is const as **O(S + N)** for both implementations (enconding) of the list. With LPOP it is still not quite as written documentation - the complexity is designated as **O(1)**. But what happens if I need to read sequentially to read everything:
![Linkedlist and ziplist results]({{ site.url }}/images/redis/linked-read-write.png){: .image-center}

What's wrong with the reading speed when using ziplist? Each `LPOP` - removing the element from the beginning of the list, with its complete rebuilding. By the way, if we use on reading RPOP, instead LPOP - the situation will not change much (hello realloc from function of updating the list in ziplist.c).

Why is it important?

Commands [RPOPLPUSH](http://redis.io/commands/RPOPLPUSH){:target="_blank"}/[BRPOPLPUSH](http://redis.io/commands/BRPOPLPUSH){:target="_blank"} popular solution for queues based on Redis (eg Sidekiq, Resque). When a queue encoded ziplist has a large number of value (thousands) - getting a single element is not a const, and the system begins to "fever".

Conclusions
---

* You will not use ziplist encoding with hash tables with a large number of values (1000), if you are still need high performance with a large amount of items.
* If the data in your hash table have a regular structure - forget about the hash table and go to store data in lists.
* Whatever kind of the encoding you use - Redis is ideal for numbers, suitable for strings with a length less then 63 bytes, and is ambiguous when stored strings larger.
* If your system has a lot of lists, remember that as long as they are small (up list-max-ziplist-entries) - you spend little memory and has good performance, but as soon as they begin to grow - memory can rise sharply by 2 times or more and the process of changing the encoding will take a long time (rebuilding with sequential insert + delete).
* Be careful with the representation of the list (setting list-max-\*), if you use the list to build a queue or the active read / write with the removal. Or else, if you are using Redis for the construction of queues based on lists - set list-max-ziplist-entries = 1 (memory still spend only a little more).
* Redis never gives the memory already allocated by the system. Consider the overhead of service information and strategy resizing. If you write a lot, you can greatly increase the memory fragmentation because of this aspect and spend up to 2 times more memory than expected. This is especially important when you run N instances of Redis on a single physical server.
* If you need to store data heterogeneous by size and access speed on a single Redis - think about that a little bit to add Redis sources and go to the setup list-max-\* parameters on each key, instead of the server.
* Encoding of the same type of data in the master / slave can be different, allowing you more flexible approach to the requirements. For example, quickly and with a high consumption of memory read on master, slower and more economical for memory on the slave or vice versa.
* Overhead using ziplist is minimal. Store strings in ziplist cheaper than any other structure (overhead of zlentry is only 21 bytes per string, while the traditional redisObject + sds string - 56 bytes).

The materials used in writing this article:

- [Redis 3.0.5](https://github.com/antirez/redis/tree/3.0.5/src){:target="_blank"} sources
- [http://redis.io/topics/memory-optimization](http://redis.io/topics/memory-optimization){:target="_blank"}
- [Redis Memory Optimization from Sripathi Krishnan](https://github.com/sripathikrishnan/redis-rdb-tools/wiki/Redis-Memory-Optimization){:target="_blank"}



