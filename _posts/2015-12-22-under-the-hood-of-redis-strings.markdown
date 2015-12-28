---
layout: post
title: "Under the Hood of Redis: Strings"
modified:
categories: Under the Hood of Redis
description: 
tags: [nosql, redis, algorithms, big data]
image:
  feature:
  credit:
  creditlink:
comments:
share:
date: 2015-12-22T10:37:01+00:00
---

You know why the simple string `strings` in Redis will occupy 56 bytes of RAM? I will try to tell why this so in Redis. 
Why it is so important to the developer to understand how Redis work under the cover. This knowledge is especially important 
if you try to calculate the actual consumption of memory or you planing to build high loaded system. 
Or, as often happens, you try to understand urgently why your Redis instance began to consume unexpectedly a lot of memory.

Content in few lines:

* How strings are stored in Redis
* What internal structures are used for storage of lines
* What types of optimization are used by Redis under a cowl.
* How effectively to store big structures and in what situations you shouldn't use strings or structures constructed on their basis. 

Strings is a key structure of Redis, HSET/ZSET/LIST are constructed on their base adding a small overhead on representation of the internal structures. 
Over a year I read and actively answer on stackoverflow in Redis tag. On an extent of this time I constantly see not stopping stream of questions
 which is shown that much of developers don't understand RAM features of Redis and that price Redis pays for the extremely high speed.

The answer to a question how many memory it will be used actually depends on an operating system, the compiler, CPU and
 the used memory allocator (jemalloc used in Redis by default). I give all further calculations for Redis 3.0.5 compiled 
 on the 64th bit server under control of centos 7.
 
It seems to me that here the small interlude is absolutely necessary for those who doesn't write on c/c++ or 
isn't really well familiar with that as everything works at a low level. Let's simplify terribly strongly some concepts 
that it was simpler to you to understand calculations. When in the program on with/with ++ you declare structure, and 
in it you have field unsigned int (without sign whole on 4 bytes) the compiler will carefully level their size to 8 bytes
in real random access memory (for h64 architecture). In it to article will be periodically it is told about a memory
allokator — this such piece which marks out memory "on clever". For example, jemalloc tries to optimize for you the
speed of search of new blocks of memory, staking on alignment of the allocated fragments. Strategy of allocation and 
alignment of memory in jemalloc is well described, however I think that we should use simplification that any size of
the allocated fragment will be rounded to the next degree 2. You ask 24 bytes — will allocate 32. You ask 61 — will
allocate 64. I strongly simplify and I hope to you it will be a little more clear. These are those things which in
the interpreted languages, logically, shouldn't excite you, however here very much I ask you to pay to them the 
attention.
 
The concept and realization of strings belongs to Salvatore Sanfilippo's (aka antirez) and placed in Redis sub project 
under the name SDS (Simple Dynamic String, [github.com/antirez/sds](http://github.com/antirez/sds)):


    +--------+-------------------------------+-----------+
    | Header | Binary safe C alike string... | Null term |
    +--------+-------------------------------+-----------+
             |
             `-> Pointer returned to the user.

It is a simple `c` structure, whose header stores the actual size and the empty space in already allocated memory,
the string data itself and terminating zero. The sds strings we are most interested in is the cost of a header,
the resize strategy and structures alignment penalty when allocating memory.

July 4th 2015 ended with [a long history with the optimization of sds strings] (https://github.com/antirez/redis/pull/2509),
which is to get to the Redis 3.1. This optimization will bring big savings in sds headers (from 16% to 200% for synthetic tests).
Remove restrictions in the 512MB to a maximum length of strings in Redis. All this will be possible thanks to the 
dynamic length of the header when changing the length of the string. 
So the header will take only 3 bytes for strings with a length up to 256 bytes, 5 bytes for strings less than 65 kb, 
9 bytes (as now) for strings up to 512 MB, and 17 bytes for strings whose size "intermeddle" in uint64_t (64 bit unsigned integer). 
By the way, this change in our Redis server farm will save about 19.3% of memory (~ 42 GB). 
However, in Redis 3.0.x everything is simple - 8 bytes + 1 byte to the terminating zero. Let's estimate how much memory 
takes a string «strings»: 

    16 (title): + 7 (string length) +1 (trailing zero) = 24 bytes (16 bytes in the header, because the compiler will align for you 2 unsigned int).
     
And jemalloc allocate 32 bytes for you. Let's take as long as it will not be taken into account (I hope it will be understood later why).

What happens when a string changes the size? Whenever you increase the size of the string and has already allocated memory is not enough, 
Redis compares the new length with constant `SDS_MAX_PREALLOC` (defined in sds.h and is 1,048,576 bytes). 
 * If the new length is less than this value will be allocated memory twice requested. 
 * If the string length exceeds `SDS_MAX_PREALLOC` - the new requested length will be added to the value of this constant. 
 
This feature is important in the story "about the disappearing memory at the use of bitmaps». By the way, when allocating
 memory for the bitmap it will always be allocated to 2 times more than requested, because of the peculiarities 
 of realization SETBIT (see setbitCommand in bitops.c).



Теперь можно было бы сказать, что наша строка займёт в оперативной памяти 32 байта (с учетом выравнивания). Те, кто читал советы ребят из hashedin.com ([redis memory optimization guide](https://github.com/sripathikrishnan/redis-rdb-tools/wiki/Redis-Memory-Optimization)) могут вспомнить, что они настоятельно рекомендуют не использовать строки длиной менее 100 байт, т.к. для хранения короткой строки, скажем при использовании команды `set foo bar` вы потратите ~96 байт из которых 90 байт это оверхед (на 64 битной машине). Лукавят? Давайте разбираться дальше.


Все значения в редис хранятся в структуре типа redisObject. Это позволяет редису знать тип значения, его внутреннее представление (в редисе это называется кодировкой), данные для LRU, количество ссылающихся на значение объектов и непосредственно само значение:
+------+----------+-----+----------+-------------------------+
| Type | Encoding | LRU | RefCount | Pointer to  data (ptr*) |
+------+----------+-----+----------+-------------------------+


Чуть позже посчитаем её размер для нашей строки с учётом выравнивания компилятора и особенностей jemalloc. В разрезе строк нам очень важно знать, какие кодировки используются для хранения строк. Прямо сейчас редис использует три различные стратегии хранения:


- REDIS_ENCODING_INT достаточно прост. Строки могут хранится в таком виде, если значение приведённое к long значение находится в диапазоне LONG_MIN, LONG_MAX. Так, строка «dict» будет хранится именно в виде этой кодировки и будет представлять собой число 1952672100 (0x74636964). Эта же кодировка используется для предварительно выделенного диапазона специальных значений в диапазоне REDIS_SHARED_INTEGERS (определён в redis.h и равен по умолчанию 10000). Значения их этого диапазона выделяются сразу при старте редиса.
- REDIS_ENCODING_EMBSTR используется для строк с длиной до 39 байт (значение константы REDIS_ENCODING_EMBSTR_SIZE_LIMIT из object.c). Это означает, что redisObject и структура с sds строкой будут размещена в единой области памяти выделенной аллокатором. Помня это, мы правильно сможем посчитать выравнивание. Впрочем, это не менее важно для понимания проблемы фрагментации памяти в редисе и то, как с этим жить.
- REDIS_ENCODING_RAW используется для всех строк, чья длина превышает REDIS_ENCODING_EMBSTR_SIZE_LIMIT. В этом случае наш ptr* хранит обычный указатель на область памяти с sds строкой.

> EMBSTR появились в 2012 году и привнесли 60-70% увеличение производительности при работе с короткими строками, однако серьезных изысканий на тему влияния на память и её фрагментацию нет до сих пор.


Длина нашей строки "strings" всего 7 байт, т.е. тип её внутреннего представления — EMBSTR. Созданная таким образом строка размещена в памяти вот так:


+--------------+--------------+------------+--------+----+
| robj data... | robj->ptr    | sds header | string | \0 |
+--------------+-----+--------+------------+--------+----+
                     |                       ^
                     +-----------------------+


Теперь мы готовы посчитать сколько оперативной памяти потребуется redis для хранения нашей строки "strings":
    
    (4 + 4)* + 8(encoding) + 8 (lru) + 8 (refcount) + 8 (ptr) + 16 (sds header) + 7(сама строка) + 1 (завершающий ноль) = 56 байт.

Тип и значение в redisObject используют только 4 младших и старших бита одного числа, поэтому эти два поля после выравнивания займут 8 байт.


Давайте проверим, что я не вожу вас за нос. Посмотрим кодировку и значение. Воспользуемся одной мало известной командой для отладки строк — DEBUG SDSLEN. К слову, команды [нет в официальной документации](http://redis.io/commands/debug-object), она была добавлена в [redis 2.6](https://raw.githubusercontent.com/antirez/redis/2.6/00-RELEASENOTES) и бывает очень полезна:

set key strings
+OK
debug object key
+Value at:0x7fa037c35dc0 refcount:1 encoding:embstr serializedlength:8 lru:3802212 lru_seconds_idle:14
debug sdslen key
+key_sds_len:3, key_sds_avail:0, val_sds_len:7, val_sds_avail:0

Используемая кодировка — embstr, длина строки 7 байт (val_sds_len). Что насчёт тех 96 байт, про которые говорили парни из hashedin.com? В моём понимании, они немного ошиблись, их пример с `set foo bar` потребует выделение 112 байт оперативной памяти (56 байт на значение и столько же на ключ), из которых 106 — оверхед. 


Чуть выше я обещал историю про исчезающую память при использование BITMAP. Особенность о которой я хочу рассказать, 
постоянно утекает из внимания части разработчиков, её использующих. Парни, на этом регулярно зарабатывают консультанты
 по оптимизации памяти. Такие как redis-labs или datadog. Семейство команда «Bit and byte level operations» появились
  в redis 2.2 и сразу позиционировались как палочка выручалочка для счётчиков реального времени (например, статья 
  от [Spool](http://blog.getspool.com/2011/11/29/fast-easy-realtime-metrics-using-redis-bitmaps/)), которые позволяют 
  экономить память. В официальном гайде по оптимизации памяти тоже есть рекламный слоган об использовании этого семейства 
  данных для хранения онлайна «Для 100 миллионов пользователей эта данные займут всего 12 мегабайт оперативной памяти».
   В описании SETBIT и SETRANGE предупреждают о возможных лагах работы сервера при выделении памяти, опуская при этом
    важный, как мне кажется, раздел «Когда вам не стоит использовать BITMAP» или «Когда лучше использовать SET вместо BITMAP».
     

Вооружившись пониманием того, как растут строки в редисе можно заметить, что bitmap:
 * не стоит использовать для разреженных данных.
 * понимать отношение полезной и реальной нагрузки (об этом на примере ниже).
 * учитывать динамику заполнения вашего bitmap.

Рассмотрим на примере. Предположим, что у вас зарегистрировано до 10 млн человек и ваш десяти миллионный пользователь вышел онлайн:

setbit online 10000000 1
:0
debug sdslen online
+key_sds_len:6, key_sds_avail:0, val_sds_len:1250001, val_sds_avail:1048576

Ваш фактический расход памяти составил 2,298,577 байт, при «полезной» для вас 1,250,001 байтах. Хранение одного вашего пользователя обошлось вам в ~2,3 мб. Используя SET вам потребовалось бы ~64 байта (при 4 байтах полезной нагрузки). Нужно правильно выбирать интервалы агрегации так, чтобы снижать разреженность данных и стараться, чтобы заполнение bitmap лежало в диапазоне от 30% — в этом случае вы на самом деле будете эффективно использовать память под эту структуру данных. Я говорю это к тому, что если у вас многомиллионная аудитория, а часовой онлайн скажем 10,000 — 100,000 человек то использовать для этой цели bitmap может быть накладным по памяти.

Наконец, изменение размеров строк в редисе — это постоянное перераспределение блоков памяти. Фрагментации памяти ещё одна специфика редис, о котором разработчики мало задумываются.
 
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
 
 
 Метрика mem_fragmentation_ratio показывает отношение выделенной операционной системой памятью (used_memory_rss) и памятью, используемой редисом (used_memory). При этом used_memory и used_memory_rss уже включат в себя как сами данные так и затраты на хранение внутренних структур редиса для их хранения и представления. Редис рассматривает RSS (Resident Set Size) как выделенное операционной системой количество памяти, в котором помимо пользовательских данных (и расходов на их внутреннее представление)учитываются расходы на фрагментацию при физическом выделение памяти самой операционной системой.
 
 Как понимать mem_fragmentation_ratio? Значение 2,1 говорит нам что мы используем на 210% больше памяти под хранение данных, чем нам нужно. А значения меньше 1 говорит о том, что память кончилась и операционная система свапится. 
 
 На практике, если значения для mem_fragmentation_ratio выпадают за границы 1 — 1,5 говорит о том, что у вас что-то не так. Попробуйте: 
 
- Перезагрузить ваш редис. Чем дольше редис, в который вы активно пишите работал без перезагрузки, тем выше у вас будет mem_fragmentation_ratio. Во многом «благодаря» особенности аллокатора. В том числе, это гарантировано поможет, если у вас большая разница между used_memory и used_memory_peak. Последний показатель говорит какой максимальный объем памяти который когда либо требовался вашему экземпляру редиса с момента его старта.
- Посмотреть, что за данные и сколько вы планируете хранить. Так, если для хранения ваших данных достаточно 4 гб — используйте 32 битные сборки редиса. По крайне мере, если вы используете 64 битную сборку хотя бы попробуйте развернуть ваш дам на 32 битной версии (rdb не зависит от битности редиса и вы легко можете запустить rdb созданный 64 битным экземпляром на 32 битном). Практически гарантировано это снижает фрагментацию (и объем использованной памяти) на ~7% (за счёт экономии на выравнивании).
- Если вы понимаете разницу и особенности, попробуйте сменить аллокатор. Редис можно собрать с glibс malloc, jemalloc (почитайте что [думают об этом инженеры facebook](https://www.facebook.com/notes/facebook-%C2%AD%E2%80%90engineering/scalable-%C2%AD%E2%80%90memory-%C2%AD%E2%80%90allocation-%C2%AD%E2%80%90using-%C2%AD%E2%80%90jemalloc/480222803919), tcmalloc.

В разговоре о фрагментации, я не учитываю специфике редиса при включенном LRU или дополнительных сложностях при большом количество обычный строковых ключей — всё это тянет на отдельную статью. Буду признателен, если вы поделитесь предложениями, стоит ли об этом писать и что ещё вам кажется важным при работе с редисом. 

Дополнительные материалы для ознакомления:
- http://redis.io/topics/memory-optimization
- http://redis.io/topics/internals-sds
- http://redislabs.com/blog/redis-ram-ramifications
- http://github.com/sripathikrishnan/redis-rdb-tools/wiki/Redis-Memory-Optimization
 






 


