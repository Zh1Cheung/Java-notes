## 特性

- Redis 内置了复制（Replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），不同级别的磁盘持久化（Persistence），并通过 Redis哨兵（Sentinel）和自动分区（Cluster）提供高可用性（High Availability）的键值对(key-value)存储数据库

- > 首先，采用了多路复用io阻塞机制
  > 然后，数据结构简单，操作节省时间
  > 最后，运行在内存中，自然速度快

## redis-为什么这么高并发快？

Redis高并发快总结

1. Redis是纯内存数据库，一般都是简单的存取操作，线程占用的时间很多，时间的花费主要集中在IO上，所以读取速度快。
2. 再说一下IO，Redis使用的是非阻塞IO，IO多路复用，使用了单线程来轮询描述符，将数据库的开、关、读、写都转换成了事件，减少了线程切换时上下文的切换和竞争。
3. Redis采用了单线程的模型，保证了每个操作的原子性，也减少了线程的上下文切换和竞争。
4. 另外，数据结构也帮了不少忙，Redis全程使用hash结构，读取速度快，还有一些特殊的数据结构，对数据存储进行了优化，如压缩表，对短数据进行压缩存储，再如，跳表，使用有序的数据结构加快读取的速度。
5. 还有一点，Redis采用自己实现的事件分离器，效率比较高，内部采用非阻塞的执行方式，吞吐能力比较大。



## 应用场景

- 数据高速缓存,web会话缓存（Session Cache）
  - 由于redis访问速度块、支持的数据类型比较丰富
  - 结合expire（为 key 设置过期时间），我们可以设置过期时间然后再进行缓存更新操作
- 限时业务的运用
  - redis中可以使用expire命令设置一个键的生存时间，到时间后redis会删除它。
- 计数器相关问题
  - incrby命令可以实现原子性的递增，所以可以运用于高并发的秒杀活动、分布式序列号的生成
- 排行榜相关问题
  - redis的SortedSet进行热点数据的排序
- 分布式锁 
  - 利用redis的setnx命令（如果不存在则成功设置缓存同时返回1，否则返回0）进行
  - 因为我们服务器是集群的，定时任务可能在两台机器上都会运行，所以在定时任务中首先 通过setnx设置一个lock，如果成功设置则执行，如果没有成功设置，则表明该定时任务已执行
- 延时操作 
  - 我们在订单生产时，设置一个key，同时设置10分钟后过期， 我们在后台实现一个监听器，监听key的实效，监听到key失效时将后续逻辑加上。 当然我们也可以利用rabbitmq、activemq等消息中间件的延迟队列服务实现该需求
  - 比如在订单生产后我们占用了库存，10分钟后去检验用户是够真正购买，如果没有购买将该单据设置无效，同时还原库存。
- 分页、模糊搜索
  - 通过ZRANGEBYLEX zset - + LIMIT 0 10 可以进行分页数据查询，其中- +表示获取全部数据
- 点赞、好友等相互关系的存储
  - set是可以自动排重的
  - 在微博应用中，每个用户关注的人存在一个集合中，就很容易实现求两个人的共同好友功能

#### 1. String——字符串

String 数据结构是简单的 key-value 类型，value 不仅可以是 String，也可以是数字（当数字类型用 Long 可以表示的时候encoding 就是整型，其他都存储在 sdshdr 当做字符串）。使用 Strings 类型，可以完全实现目前 Memcached 的功能，并且效率更高。还可以享受 Redis 的定时持久化（可以选择 RDB 模式或者 AOF 模式），操作日志及 Replication 等功能。除了提供与 Memcached 一样的 get、set、incr、decr 等操作外，Redis 还提供了下面一些操作：

- `LEN niushuai`：O(1)获取字符串长度
- `APPEND niushuai redis`：往字符串 append 内容，而且采用智能分配内存（每次2倍）
- 设置和获取字符串的某一段内容
- 设置及获取字符串的某一位（bit）
- 批量设置一系列字符串的内容
- 原子计数器
- GETSET 命令的妙用，请于清空旧值的同时设置一个新值，配合原子计数器使用
- **应用场景：**存储key-value键值对

#### 2. Hash——字典

在 Memcached 中，我们经常将一些结构化的信息打包成 hashmap，在客户端序列化后存储为一个字符串的值（一般是 JSON 格式），比如用户的昵称、年龄、性别、积分等。这时候在需要修改其中某一项时，通常需要将字符串（JSON）取出来，然后进行反序列化，修改某一项的值，再序列化成字符串（JSON）存储回去。简单修改一个属性就干这么多事情，消耗必定是很大的，也不适用于一些可能并发操作的场合（比如两个并发的操作都需要修改积分）。而 Redis 的 Hash 结构可以使你像在数据库中 Update 一个属性一样只修改某一项属性值。

- 存储、读取、修改用户属性

- 购物车：`hset [key] [field] [value]` 命令， 可以实现以`用户Id`，`商品Id`为`field`，商品数量为`value`，恰好构成了购物车的3个要素。

  存储对象：`hash`类型的`(key, field, value)`的结构与对象的`(对象id, 属性, 值)`的结构相似，也可以用来存储对象

#### 3. List——列表

List 说白了就是链表（redis 使用双端链表实现的 List），相信学过数据结构知识的人都应该能理解其结构。使用 List 结构，我们可以轻松地实现最新消息排行等功能（比如新浪微博的 TimeLine ）。List 的另一个应用就是消息队列，可以利用 List 的 *PUSH 操作，将任务存在 List 中，然后工作线程再用 POP 操作将任务取出进行执行。Redis 还提供了操作 List 中某一段元素的 API，你可以直接查询，删除 List 中某一段的元素。

- 微博 TimeLine
- 消息队列
- 由于list它是一个按照插入顺序排序的列表，所以应用场景相对还较多的，例如：
  - 消息队列：`lpop`和`rpush`（或者反过来，`lpush`和`rpop`）能实现队列的功能
  - 朋友圈的点赞列表、评论列表、排行榜：`lpush`命令和`lrange`命令能实现最新列表的功能，每次通过`lpush`命令往列表里插入新的元素，然后通过`lrange`命令读取最新的元素列表。

#### 4. Set——集合

Set 就是一个集合，集合的概念就是一堆不重复值的组合。利用 Redis 提供的 Set 数据结构，可以存储一些集合性的数据。比如在微博应用中，可以将一个用户所有的关注人存在一个集合中，将其所有粉丝存在一个集合。因为 Redis 非常人性化的为集合提供了求交集、并集、差集等操作，那么就可以非常方便的实现如共同关注、共同喜好、二度好友等功能，对上面的所有集合操作，你还可以使用不同的命令选择将结果返回给客户端还是存集到一个新的集合中。

- 共同好友、二度好友

- 利用唯一性，可以统计访问网站的所有独立 IP

- 好友推荐的时候，根据 tag 求交集，大于某个 threshold 就可以推荐

- 好友、关注、粉丝、感兴趣的人集合： 
  1) `sinter`命令可以获得A和B两个用户的共同好友；
  2) `sismember`命令可以判断A是否是B的好友；
  3) `scard`命令可以获取好友数量；
  4) 关注时，`smove`命令可以将B从A的粉丝集合转移到A的好友集合

  首页展示随机：美团首页有很多推荐商家，但是并不能全部展示，set类型适合存放所有需要展示的内容，而`srandmember`命令则可以从中随机获取几个。

#### 5. Sorted Set——有序集合

和Sets相比，Sorted Sets是将 Set 中的元素增加了一个权重参数 score，使得集合中的元素能够按 score 进行有序排列，比如一个存储全班同学成绩的 Sorted Sets，其集合 value 可以是同学的学号，而 score 就可以是其考试得分，这样在数据插入集合的时候，就已经进行了天然的排序。另外还可以用 Sorted Sets 来做带权重的队列，比如普通消息的 score 为1，重要消息的 score 为2，然后工作线程可以选择按 score 的倒序来获取工作任务。让重要的任务优先执行。

- 带有权重的元素，比如一个游戏的用户得分排行榜

- 比较复杂的数据结构，一般用到的场景不算太多

- `zset` 可以用做排行榜，但是和`list`不同的是`zset`它能够实现动态的排序，例如： 可以用来存储粉丝列表，value 值是粉丝的用户 ID，score 是关注时间，我们可以对粉丝列表按关注时间进行排序。

  `zset` 还可以用来存储学生的成绩， `value` 值是学生的 ID, `score` 是他的考试成绩。 我们对成绩按分数进行排序就可以得到他的名次。



## Redis内存淘汰机制

redis 内存淘汰机制有以下几个：

1. noeviction：当内存使用超过配置的时候会返回错误，不会驱逐任何键

2. allkeys-lru：加入键的时候，如果过限，首先通过LRU算法驱逐最久没有使用的键

3. volatile-lru：加入键的时候如果过限，首先从设置了过期时间的键集合中驱逐最久没有使用的键

4. allkeys-random：加入键的时候如果过限，从所有key随机删除

5. volatile-random：加入键的时候如果过限，从过期键的集合中随机驱逐

6. volatile-ttl：从配置了过期时间的键中驱逐马上就要过期的键

7. volatile-lfu：从所有配置了过期时间的键中驱逐使用频率最少的键

8. allkeys-lfu：从所有键中驱逐使用频率最少的键



LFU (Least Frequently Used) ：最近最少使用，跟使用的次数有关，淘汰使用次数最少的。大体原理就是过去某个时间窗口内使用次数最少的记录替换掉。

LRU (Least Recently Used): 最近最不经常使用，跟使用的最后一次时间有关，淘汰最近使用时间离现在最久的。大体原理就是将使用时间距离现在最久的那条记录替换掉。



## 缓存

- 不要把 Redis 当作数据库
  - Redis 的特点是，处理请求很快，但无法保存超过内存大小的数据
  - 常用的数据淘汰策略
    - 其实，这些算法是 Key 范围 +Key 选择算法的搭配组合，其中范围有 allkeys 和 volatile 两种，算法有 LRU、TTL 和 LFU 三种
    - 首先，从算法角度来说，Redis 4.0 以后推出的 LFU 比 LRU 更“实用”。试想一下，如果一个 Key 访问频率是 1 天一次，但正好在 1 秒前刚访问过，那么 LRU 可能不会选择优先淘汰这个 Key，反而可能会淘汰一个 5 秒访问一次但最近 2 秒没有访问过的 Key，而 LFU 算法不会有这个问题。而 TTL 会比较“头脑简单”一点，优先删除即将过期的 Key，但有可能这个 Key 正在被大量访问。
    - 然后，从 Key 范围角度来说，allkeys 可以确保即使 Key 没有 TTL 也能回收，如果使用的时候客户端总是“忘记”设置缓存的过期时间，那么可以考虑使用这个系列的算法。而 volatile 会更稳妥一些，万一客户端把 Redis 当做了长效缓存使用，只是启动时候初始化一次缓存，那么一旦删除了此类没有 TTL 的数据，可能就会导致客户端出错。
- 注意缓存雪崩问题
  - 从广义上说，产生缓存雪崩的原因有两种
    - 第一种是，缓存系统本身不可用，导致大量请求直接回源到数据库；
    - 第二种是，应用设计层面大量的 Key 在同一时间过期，导致大量的数据回源。
  - 如何确保大量 Key 不在同一时间被动过期
    - 差异化缓存过期时间，不要让大量的 Key 在同一时间过期。比如，在初始化缓存的时候，设置缓存的过期时间是 30 秒 +10 秒以内的随机延迟（扰动值）。这样，这些Key 不会集中在 30 秒这个时刻过期，而是会分散在 30~40 秒之间过期
- 注意缓存击穿问题
  - 在某些 Key 属于极端热点数据，且并发量很大的情况下，如果这个 Key 过期，可能会在某个瞬间出现大量的并发请求同时回源，相当于大量的并发请求直接打到了数据库。
  - 使用 Redisson 来获取一个基于 Redis 的分布式锁，在查询数据库之前先尝试获取锁，这样，可以把回源到数据库的并发限制在 1
  - 在真实的业务场景下，不一定要这么严格地使用双重检查分布式锁进行全局的并发限制，因为这样虽然可以把数据库回源并发降到最低，但也限制了缓存失效时的并发。可以考虑的方式是
    - 方案一，使用进程内的锁进行限制，这样每一个节点都可以以一个并发回源数据库；
    - 方案二，不使用锁进行限制，而是使用类似 Semaphore 的工具限制并发数，比如限制为 10，这样既限制了回源并发数不至于太大，又能使得一定量的线程可以同时回源。
- 注意缓存穿透问题
  - 缓存中没有数据不一定代表数据没有缓存，还有一种可能是原始数据压根就不存在
  - 解决缓存穿透有以下两种方案
    - 方案一，对于不存在的数据，同样设置一个特殊的 Value 到缓存中，比如当数据库中查出的用户信息为空的时候，设置 NODATA 这样具有特殊含义的字符串到缓存中。这样下次请求缓存的时候还是可以命中缓存，即直接从缓存返回结果，不查询数据库
    - 方案二，即使用布隆过滤器做前置过滤
      - 布隆过滤器是一种概率型数据库结构，由一个很长的二进制向量和一系列随机映射函数组成。它的原理是，当一个元素被加入集合时，通过 k 个散列函数将这个元素映射成一个 m位 bit 数组中的 k 个点，并置为 1。
      - 要用上布隆过滤器，我们可以使用 Google 的 Guava 工具包提供的 BloomFilter 类改造一
        下程序：启动时，初始化一个具有所有有效用户 ID 的、10000 个元素的 BloomFilter，在
        从缓存查询数据之前调用其 mightContain 方法，来检测用户 ID 是否可能存在；如果布隆
        过滤器说值不存在，那么一定是不存在的
- 注意缓存数据同步策略
  - Cache Aside 更新模式
  - 前面提到的 3 个案例，其实都属于缓存数据过期后的被动删除。在实际情况下，修改了原始数据后，考虑到缓存数据更新的及时性，我们可能会采用主动更新缓存的策略
  - 先更新缓存，再更新数据库；
    - “先更新缓存再更新数据库”策略不可行。数据库设计复杂，压力集中，数据库因为超时等原因更新操作失败的可能性较大，此外还会涉及事务，很可能因为数据库更新失败，导致缓存和数据库的数据不一致。
  - 先更新数据库，再更新缓存；
    - “先更新数据库再更新缓存”策略不可行。一是，如果线程 A 和 B 先后完成数据库更新，但更新缓存时却是 B 和 A 的顺序，那很可能会把旧数据更新到缓存中引起数据不一致；二是，我们不确定缓存中的数据是否会被访问，不一定要把所有数据都更新到缓存中去。
  - 先删除缓存，再更新数据库，访问的时候按需加载数据到缓存；
    - “先删除缓存再更新数据库，访问的时候按需加载数据到缓存”策略也不可行。在并发的情况下，很可能删除缓存后还没来得及更新数据库，就有另一个线程先读取了旧值到缓存中，如果并发量很大的话这个概率也会很大。
  - 先更新数据库，再删除缓存，访问的时候按需加载数据到缓存。
    - “先更新数据库再删除缓存，访问的时候按需加载数据到缓存”策略是最好的。
    - 这种做法其实不能算是坑，在实际的系统中也推荐使用这种方式。但是这种方式理论上还是可能存在问题。以Redis和Mysql为例，查询操作没有命中缓存，然后查询出数据库的老数据。此时有一个并发的更新操作，更新操作在读操作之后更新了数据库中的数据并且删除了缓存中的数据。然而读操作将从数据库中读取出的老数据更新回了缓存。这样就会造成数据库和缓存中的数据不一致，应用程序中读取的都是原来的数据（脏数据）
    - 但是，仔细想一想，这种并发的概率极低。因为这个条件需要发生在读缓存时缓存失效，而且有一个并发的写操作。实际上数据库的写操作会比读操作慢得多，而且还要加锁，而读操作必需在写操作前进入数据库操作，又要晚于写操作更新缓存，所有这些条件都具备的概率并不大。但是为了避免这种极端情况造成脏数据所产生的影响，我们还是要为缓存设置过期时间。
    - 需要注意的是，更新数据库后删除缓存的操作可能失败，如果失败则考虑把任务加入延迟队列进行延迟重试，确保数据可以删除，缓存可以及时更新。因为删除操作是幂等的，所以即使重复删问题也不是太大，这又是删除比更新好的一个原因。
  - Write Behind Caching 更新模式
    - Write Behind Caching 更新模式就是在更新数据的时候，只更新缓存，不更新数据库，而我们的缓存会异步地批量更新数据库。这个设计的好处就是直接操作内存速度快。因为异步，Write Behind Caching 更新模式还可以合并对同一个数据的多次操作到数据库，所以性能的提高是相当可观的。
    - 但其带来的问题是，数据不是强一致性的，而且可能会丢失。另外，Write Behind Caching 更新模式实现逻辑比较复杂，因为它需要确认有哪些数据是被更新了的，哪些数据需要刷到持久层上。只有在缓存需要失效的时候，才会把它真正持久起来。





## 

## **LFU 的问题**

LRU 实现上比较简单，最简单的只需要 链表和Map 就够了，移除元素时直接从链表队尾移除，增加时加到头部就可以了。

但实际上 Redis 的实现并不是这样， Redis 的实现非常直接，几乎就是 LRU 本身的意思。Redis 本身有全局的时钟 server.lruclock（单位为秒,24位，190多天会溢出），然后随机采样 N 个 key 的访问时间，离现在最久的，淘汰之。

redis 对象（简单理解为一个 key-value）定义如下：

```c
#define LRU_BITS 24
typedef struct redisObject {    // redis对象
    unsigned type:4;    // 类型,4bit
    unsigned encoding:4;    // 编码,4bit
    unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */ // 24bit
    int refcount;   // 引用计数
    void *ptr;  // 指向各种基础类型的指针
} robj;
```

**unsigned lru:LRU_BITS** 就是存放每个 key 的访问时间，为什么不用更大的数据类型？（是为了数据对齐？）

LFU 实现更为复杂，需要考虑几个问题：

1. 如果实现为链表，当对象被访问时按访问次数移动到链表的某个有序位置可能是低效的，因为可能存在大量访问次数相同的 key，最差情况是O(N) （链表无法直接用二分查找,可以用 跳表？ 哈哈）;
2. 某些 key 访问次数可能非常之大，理论上可以无限大，但实际上我们并不需要精确的访问次数；
3. 访问次数特别大的 key 可能以后都不再访问了，但是因为访问次数大而一直占用着内存不被淘汰，需要一个方法来逐步“驱除”（有点 LRU的意思），最简单的就是逐步衰减访问次数

- **LFU 实现**

本着能省则省的原则，Redis 只用了 24bit  （server.lruclock 也是24bit）来记录上述的信息，是的不是 24byte，连32位指针都放不下！

**16bit : 上一次递减时间 （解决第三个问题）**

**8bit : 访问次数 （解决第二个问题）**

**访问次数的计算如下：**

```c
  uint8_t LFULogIncr(uint8_t counter) {
      if (counter == 255) return 255;
      double r = (double)rand()/RAND_MAX;
      double baseval = counter - LFU_INIT_VAL;
      if (baseval < 0) baseval = 0;
      double p = 1.0/(baseval*server.lfu_log_factor+1);
      if (r < p) counter++;
      return counter;
  }
```

核心就是访问次数越大，访问次数被递增的可能性越小，最大 255，此外你可以在配置 redis.conf 中写明访问多少次递增多少。



**由于访问次数是有限的，所以第一个问题也被解决了，直接一个255数组或链表都可以。**

**16bit 部分怎么用呢？保存的是时间戳的后16位（分钟），表示上一次递减的时间，算法是这样执行，随机采样N个key(与原来的版本一样)，检查递减时间，如果距离现在超过 N 分钟（可配置），则递减或者减半（如果访问次数数值比较大）。**



此外，由于新加入的 key 访问次数很可能比不被访问的老 key小，为了不被马上淘汰，**新key访问次数设为 5**。

- LFU实现

力扣原题描述如下：

```java
请你为 最不经常使用（LFU）缓存算法设计并实现数据结构。它应该支持以下操作：get 和 put。

get(key) - 如果键存在于缓存中，则获取键的值（总是正数），否则返回 -1。
put(key, value) - 如果键不存在，请设置或插入值。当缓存达到其容量时，则应该在插入新项之前，使最不经常使用的项无效。在此问题中，当存在平局（即两个或更多个键具有相同使用频率）时，应该去除 最近 最少使用的键。
「项的使用次数」就是自插入该项以来对其调用 get 和 put 函数的次数之和。使用次数会在对应项被移除后置为 0 。

示例：

LFUCache cache = new LFUCache( 2 /* capacity (缓存容量) */ );

cache.put(1, 1);
cache.put(2, 2);
cache.get(1);       // 返回 1
cache.put(3, 3);    // 去除 key 2
cache.get(2);       // 返回 -1 (未找到key 2)
cache.get(3);       // 返回 3
cache.put(4, 4);    // 去除 key 1
cache.get(1);       // 返回 -1 (未找到 key 1)
cache.get(3);       // 返回 3
cache.get(4);       // 返回 4

来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/lfu-cache
```

就是要求我们设计一个 LFU 算法，根据访问次数（访问频次）大小来判断应该删除哪个元素，get和put操作都会增加访问频次。当访问频次相等时，就判断哪个元素是最久未使用过的，把它删除。

因此，这道题需要考虑两个方面，一个是访问频次，一个是访问时间的先后顺序。

### 方案一：使用优先队列

**思路：**

我们可以使用JDK提供的优先队列 PriorityQueue 来实现 。 因为优先队列内部维护了一个二叉堆，即可以保证每次 poll 元素的时候，都可以根据我们的要求，取出当前所有元素的最大值或是最小值。只需要我们的实体类实现 Comparable 接口就可以了。

因此，我们需要定义一个 Node 来保存当前元素的访问频次 freq，全局的自增的 index，用于比较大小。然后定义一个 Map<Integer,Node> cache ，用于存放元素的信息。 

当 cache 容量不足时，根据访问频次 freq 的大小来删除最小的 freq 。若相等，则删除 index 最小的，因为index是自增的，越大说明越是最近访问过的，越小说明越是很长时间没访问过的元素。

因本质是用二叉堆实现，故时间复杂度为O(logn)。

```java
public class LFUCache4 {

    public static void main(String[] args) {
        LFUCache4 cache = new LFUCache4(2);
        cache.put(1, 1);
        cache.put(2, 2);
        // 返回 1
        System.out.println(cache.get(1));
        cache.put(3, 3);    // 去除 key 2
        // 返回 -1 (未找到key 2)
        System.out.println(cache.get(2));
        // 返回 3
        System.out.println(cache.get(3));
        cache.put(4, 4);    // 去除 key 1
        // 返回 -1 (未找到 key 1)
        System.out.println(cache.get(1));
        // 返回 3
        System.out.println(cache.get(3));
        // 返回 4
        System.out.println(cache.get(4));
    }

    //缓存了所有元素的node
    Map<Integer,Node> cache;
    //优先队列
    Queue<Node> queue;
    //缓存cache 的容量
    int capacity;
    //当前缓存的元素个数
    int size;
    //全局自增
    int index = 0;

    //初始化
    public LFUCache4(int capacity){
        this.capacity = capacity;
        if(capacity > 0){
            queue = new PriorityQueue<>(capacity);
        }
        cache = new HashMap<>();
    }

    public int get(int key){
        Node node = cache.get(key);
        // node不存在，则返回 -1
        if(node == null) return -1;
        //每访问一次，频次和全局index都自增 1
        node.freq++;
        node.index = index++;
        // 每次都重新remove，再offer是为了让优先队列能够对当前Node重排序
        //不然的话，比较的 freq 和 index 就是不准确的
        queue.remove(node);
        queue.offer(node);
        return node.value;
    }

    public void put(int key, int value){
        //容量0，则直接返回
        if(capacity == 0) return;
        Node node = cache.get(key);
        //如果node存在，则更新它的value值
        if(node != null){
            node.value = value;
            node.freq++;
            node.index = index++;
            queue.remove(node);
            queue.offer(node);
        }else {
            //如果cache满了，则从优先队列中取出一个元素，这个元素一定是频次最小，最久未访问过的元素
            if(size == capacity){
                cache.remove(queue.poll().key);
                //取出元素后，size减 1
                size--;
            }
            //否则，说明可以添加元素，于是创建一个新的node，添加到优先队列中
            Node newNode = new Node(key, value, index++);
            queue.offer(newNode);
            cache.put(key,newNode);
            //同时，size加 1
            size++;
        }
    }


    //必须实现 Comparable 接口才可用于排序
    private class Node implements Comparable<Node>{
        int key;
        int value;
        int freq = 1;
        int index;

        public Node(){

        }

        public Node(int key, int value, int index){
            this.key = key;
            this.value = value;
            this.index = index;
        }

        @Override
        public int compareTo(Node o) {
            //优先比较频次 freq，频次相同再比较index
            int minus = this.freq - o.freq;
            return minus == 0? this.index - o.index : minus;
        }
    }
}
```

### 方案二：使用一条双向链表

**思路：**

只用一条双向链表，来维护频次和时间先后顺序。那么，可以这样想。把频次 freq 小的放前面，频次大的放后面。如果频次相等，就从当前节点往后遍历，直到找到第一个频次比它大的元素，并插入到它前面。（当然，如果遍历到了tail，则插入到tail前面）这样可以保证同频次的元素，最近访问的总是在最后边。

因此，总的来说，最低频次，并且最久未访问的元素肯定就是链表中最前面的那一个了。这样的话，当 cache容量满的时候，直接把头结点删除掉就可以了。但是，我们这里为了方便链表的插入和删除操作，用了两个哨兵节点，来表示头节点 head和尾结点tail。因此，删除头结点就相当于删除 head.next。

PS：哨兵节点只是为了占位，实际并不存储有效数据，只是为了链表插入和删除时，不用再判断当前节点的位置。不然的话，若当前节点占据了头结点或尾结点的位置，还需要重新赋值头尾节点元素，较麻烦。

为了便于理解新节点如何插入到链表中合适的位置，作图如下：



![img](https://pic1.zhimg.com/v2-226742d9c85285d14525458d0be764e4_b.jpg)



代码如下：

```java
public class LFUCache {

    public static void main(String[] args) {
        LFUCache cache = new LFUCache(2);
        cache.put(1, 1);
        cache.put(2, 2);
        // 返回 1
        System.out.println(cache.get(1));
        cache.put(3, 3);    // 去除 key 2
        // 返回 -1 (未找到key 2)
        System.out.println(cache.get(2));
        // 返回 3
        System.out.println(cache.get(3));
        cache.put(4, 4);    // 去除 key 1
        // 返回 -1 (未找到 key 1)
        System.out.println(cache.get(1));
        // 返回 3
        System.out.println(cache.get(3));
        // 返回 4
        System.out.println(cache.get(4));

    }

    private Map<Integer,Node> cache;
    private Node head;
    private Node tail;
    private int capacity;
    private int size;

    public LFUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new HashMap<>();
        /**
         * 初始化头结点和尾结点，并作为哨兵节点
         */
        head = new Node();
        tail = new Node();
        head.next = tail;
        tail.pre = head;
    }

    public int get(int key) {
        Node node = cache.get(key);
        if(node == null) return -1;
        node.freq++;
        moveToPostion(node);
        return node.value;
    }

    public void put(int key, int value) {
        if(capacity == 0) return;
        Node node = cache.get(key);
        if(node != null){
            node.value = value;
            node.freq++;
            moveToPostion(node);
        }else{
            //如果元素满了
            if(size == capacity){
                //直接移除最前面的元素，因为这个节点就是频次最小，且最久未访问的节点
                cache.remove(head.next.key);
                removeNode(head.next);
                size--;
            }
            Node newNode = new Node(key, value);
            //把新元素添加进来
            addNode(newNode);
            cache.put(key,newNode);
            size++;
        }
    }

    //只要当前 node 的频次大于等于它后边的节点，就一直向后找，
    // 直到找到第一个比当前node频次大的节点，或者tail节点，然后插入到它前面
    private void moveToPostion(Node node){
        Node nextNode = node.next;
        //先把当前元素删除
        removeNode(node);
        //遍历到符合要求的节点
        while (node.freq >= nextNode.freq && nextNode != tail){
            nextNode = nextNode.next;
        }
        //把当前元素插入到nextNode前面
        node.pre = nextNode.pre;
        node.next = nextNode;
        nextNode.pre.next = node;
        nextNode.pre = node;

    }

    //添加元素（头插法），并移动到合适的位置
    private void addNode(Node node){
        node.pre = head;
        node.next = head.next;
        head.next.pre = node;
        head.next = node;
        moveToPostion(node);
    }

    //移除元素
    private void removeNode(Node node){
        node.pre.next = node.next;
        node.next.pre = node.pre;
    }

    class Node {
        int key;
        int value;
        int freq = 1;
        //当前节点的前一个节点
        Node pre;
        //当前节点的后一个节点
        Node next;

        public Node(){

        }

        public Node(int key ,int value){
            this.key = key;
            this.value = value;
        }
    }
}
```

可以看到不管是插入元素还是删除元素时，都不需要额外的判断，这就是设置哨兵节点的好处。

由于每次访问元素的时候，都需要按一定的规则把元素放置到合适的位置，因此，元素需要从前往后一直遍历。所以，时间复杂度 O(n)。

### 方案三：用 LinkedHashSet维护频次链表

**思路：**

我们不再使用一条链表，同时维护频次和访问时间了。此处，换为用 map 键值对来维护，用频次作为键，用当前频次对应的一条具有先后访问顺序的链表来作为值。它的结构如下：

```java
Map<Integer, LinkedHashSet<Node>> freqMap
```



![img](https://pic1.zhimg.com/v2-6df8b8f62869163888180ccc7405114e_b.jpg)



由于LinkedHashSet 的 iterator迭代方法是按插入顺序的，因此迭代到的第一个元素肯定是当前频次下，最久未访问的元素。这样的话，当缓存 cache满的时候，直接删除迭代到的第一个元素就可以了。

另外 freqMap，也需要在每次访问元素的时候，重新维护关系。从当前元素的频次对应的双向链表中移除当前元素，并加入到高频次的链表中。

```java
public class LFUCache1 {

    public static void main(String[] args) {
        LFUCache1 cache = new LFUCache1(2);
        cache.put(1, 1);
        cache.put(2, 2);
        // 返回 1
        System.out.println(cache.get(1));
        cache.put(3, 3);    // 去除 key 2
        // 返回 -1 (未找到key 2)
        System.out.println(cache.get(2));
        // 返回 3
        System.out.println(cache.get(3));
        cache.put(4, 4);    // 去除 key 1
        // 返回 -1 (未找到 key 1)
        System.out.println(cache.get(1));
        // 返回 3
        System.out.println(cache.get(3));
        // 返回 4
        System.out.println(cache.get(4));
    }

    //缓存 cache
    private Map<Integer,Node> cache;
    //存储频次和对应双向链表关系的map
    private Map<Integer, LinkedHashSet<Node>> freqMap;
    private int capacity;
    private int size;
    //存储最小频次值
    private int min;

    public LFUCache1(int capacity) {
        this.capacity = capacity;
        cache = new HashMap<>();
        freqMap = new HashMap<>();
    }

    public int get(int key) {
        Node node = cache.get(key);
        if(node == null) return -1;
        //若找到当前元素，则频次加1
        freqInc(node);
        return node.value;
    }

    public void put(int key, int value) {
        if(capacity == 0) return;
        Node node = cache.get(key);
        if(node != null){
            node.value = value;
            freqInc(node);
        }else{
            if(size == capacity){
                Node deadNode = removeNode();
                cache.remove(deadNode.key);
                size --;
            }
            Node newNode = new Node(key,value);
            cache.put(key,newNode);
            addNode(newNode);
            size++;
        }
    }

    //处理频次map
    private void freqInc(Node node){
        //从原来的频次对应的链表中删除当前node
        LinkedHashSet<Node> set = freqMap.get(node.freq);
        if(set != null)
            set.remove(node);
        //如果当前频次是最小频次，并且移除元素后，链表为空，则更新min值
        if(node.freq == min && set.size() == 0){
            min = node.freq + 1;
        }
        //添加到新的频次对应的链表
        node.freq ++;
        LinkedHashSet<Node> newSet = freqMap.get(node.freq);
        //如果高频次链表还未存在，则初始化一条
        if(newSet == null){
            newSet = new LinkedHashSet<Node>();
            freqMap.put(node.freq,newSet);
        }
        newSet.add(node);
    }

    //添加元素，更新频次
    private void addNode(Node node){
        //添加新元素，肯定是需要加入到频次为1的链表中的
        LinkedHashSet<Node> set = freqMap.get(1);
        if(set == null){
            set = new LinkedHashSet<>();
            freqMap.put(1,set);
        }
        set.add(node);
        //更新最小频次为1
        min = 1;
    }

    //删除频次最小，最久未访问的元素
    private Node removeNode(){
        //找到最小频次对应的 LinkedHashSet
        LinkedHashSet<Node> set = freqMap.get(min);
        //迭代到的第一个元素就是最久未访问的元素，移除之
        Node node = set.iterator().next();
        set.remove(node);
        //如果当前node的频次等于最小频次，并且移除元素之后，set为空，则 min 加1
        if(node.freq == min && set.size() == 0){
            min ++;
        }
        return node;
    }

    private class Node {
        int key;
        int value;
        int freq = 1;

        public Node(int key, int value){
            this.key = key;
            this.value = value;
        }

        public Node(){

        }
    }
}
```

### 方案四：手动实现一个频次链表

**思路：**

由于方案三用的是JDK自带的 LinkedHashSet ，其是实现了哈希表和双向链表的一个类，因此为了减少哈希相关的计算，提高效率，我们自己实现一条双向链表来替代它。

那么，这条双向链表，就需要维护当前频次下的所有元素的先后访问顺序。我们采用头插法，把新加入的元素添加到链表头部，这样的话，最久未访问的元素就在链表的尾部。

同样的，我们也用两个哨兵节点来代表头尾节点，以方便链表的操作。



![img](https://pic4.zhimg.com/v2-cb0d40abc4c9f5f9898acb241e120d3f_b.jpg)



代码如下：

```java
public class LFUCache2 {

    public static void main(String[] args) {
        LFUCache2 cache = new LFUCache2(2);
        cache.put(1, 1);
        cache.put(2, 2);
        // 返回 1
        System.out.println(cache.get(1));
        cache.put(3, 3);    // 去除 key 2
        // 返回 -1 (未找到key 2)
        System.out.println(cache.get(2));
        // 返回 3
        System.out.println(cache.get(3));
        cache.put(4, 4);    // 去除 key 1
        // 返回 -1 (未找到 key 1)
        System.out.println(cache.get(1));
        // 返回 3
        System.out.println(cache.get(3));
        // 返回 4
        System.out.println(cache.get(4));
    }

    private Map<Integer,Node> cache;
    private Map<Integer,DoubleLinkedList> freqMap;
    private int capacity;
    private int size;
    private int min;

    public LFUCache2(int capacity){
        this.capacity = capacity;
        cache = new HashMap<>();
        freqMap = new HashMap<>();
    }

    public int get(int key){
        Node node = cache.get(key);
        if(node == null) return -1;
        freqInc(node);
        return node.value;
    }

    public void put(int key, int value){
        if(capacity == 0) return;
        Node node = cache.get(key);
        if(node != null){
            node.value = value; //更新value值
            freqInc(node);
        }else{
            //若size达到最大值，则移除频次最小，最久未访问的元素
            if(size == capacity){
                //因链表是头插法，所以尾结点的前一个节点就是最久未访问的元素
                DoubleLinkedList list = freqMap.get(min);
                //需要移除的节点
                Node deadNode = list.tail.pre;
                cache.remove(deadNode.key);
                list.removeNode(deadNode);
                size--;
            }
            //新建一个node，并把node放到频次为 1 的 list 里面
            Node newNode = new Node(key,value);
            DoubleLinkedList newList = freqMap.get(1);
            if(newList == null){
                newList = new DoubleLinkedList();
                freqMap.put(1,newList);
            }
            newList.addNode(newNode);
            cache.put(key,newNode);
            size++;
            min = 1;//此时需要把min值重新设置为1
        }

    }

    //修改频次
    private void freqInc(Node node){
        //先删除node对应的频次list
        DoubleLinkedList list = freqMap.get(node.freq);
        if(list != null){
            list.removeNode(node);
        }
        //判断min是否等于当前node的频次，且当前频次的list为空，是的话更新min值
        if(min == node.freq && list.isEmpty()){
            min ++;
        }
        //然后把node频次加 1，并把它放到高频次list
        node.freq ++;
        DoubleLinkedList newList = freqMap.get(node.freq);
        if(newList == null){
            newList = new DoubleLinkedList();
            freqMap.put(node.freq, newList);
        }
        newList.addNode(node);
    }


    private class Node {
        int key;
        int value;
        int freq = 1;
        Node pre;
        Node next;

        public Node(){

        }

        public Node(int key, int value){
            this.key = key;
            this.value = value;
        }
    }

    //自实现的一个双向链表
    private class DoubleLinkedList {
        Node head;
        Node tail;

        // 设置两个哨兵节点，作为头、尾节点便于插入和删除操作
        public DoubleLinkedList(){
            head = new Node();
            tail = new Node();
            head.next = tail;
            tail.pre = head;
        }

        //采用头插法，每次都插入到链表的最前面，即 head 节点后边
        public void addNode(Node node){
            node.pre = head;
            node.next = head.next;
            //注意先把head的后节点的前节点设置为node
            head.next.pre = node;
            head.next = node;
        }

        //删除元素
        public void removeNode(Node node){
            node.pre.next = node.next;
            node.next.pre = node.pre;
        }

        //判断是否为空，即是否存在除了哨兵节点外的有效节点
        public boolean isEmpty(){
            //判断头结点的下一个节点是否是尾结点，是的话即为空
            return head.next == tail;
        }

    }

}
```

### 方案五：用双向链表嵌套

**思路：**

可以发现方案三和方案四，都是用 freqmap 来存储频次和它对应的链表之间的关系，它本身也是一个哈希表。这次，我们完全用自己实现的双向链表来代替 freqMap，进一步提高效率。

但是，结构有些复杂，它是一个双向链表中，每个元素又是双向链表。为了便于理解，我把它的结构作图如下：（为了方便，分别叫做外层链表，内层链表）



![img](https://pic1.zhimg.com/v2-6a32b8d9e8d6da00965b1943847d2f69_b.jpg)



我们把整体看成一个由 DoubleLinkedList组成的双向链表，然后，每一个 DoubleLinkedList 对象中又是一个由 Node 组成的双向链表。像极了 HashMap 数组加链表的形式。

但是，我们这里没有数组，也就不存在哈希碰撞的问题。并且都是双向链表，都有哨兵存在，便于灵活的从链表头部或者尾部开始操作元素。

这里，firstLinkedList 和 lastLinkedList 分别代表外层链表的头尾结点。链表中的元素 DoubleLinkedList 有一个字段 freq 记录了频次，并且按照前大后小的顺序组成外层链表，即图中的 DoubleLinkedList1.freq 大于它后面的 DoubleLinkedList2.freq。

每当有新频次的 DoubleLinkedList 需要添加进来的时候，直接插入到 lastLinkedList 这个哨兵前面，因此 lastLinkedList.pre 就是一个最小频次的内部链表。

内部链表中是由 Node组成的双向链表，也有两个哨兵代表头尾节点，并采用头插法。其实，可以看到内部链表和方案四，图中所示的双向链表结构是一样的，不用多说了。

这样的话，我们就可以找到频次最小，并且最久未访问的元素，即

```java
//频次最小，最久未访问的元素，cache满时需要删除
lastLinkedList.pre.tail.pre
```

于是，代码就好理解了：

```java
public class LFUCache3 {

    public static void main(String[] args) {
        LFUCache3 cache = new LFUCache3(2);
        cache.put(1, 1);
        cache.put(2, 2);
        // 返回 1
        System.out.println(cache.get(1));
        cache.put(3, 3);    // 去除 key 2
        // 返回 -1 (未找到key 2)
        System.out.println(cache.get(2));
        // 返回 3
        System.out.println(cache.get(3));
        cache.put(4, 4);    // 去除 key 1
        // 返回 -1 (未找到 key 1)
        System.out.println(cache.get(1));
        // 返回 3
        System.out.println(cache.get(3));
        // 返回 4
        System.out.println(cache.get(4));
    }

    Map<Integer,Node> cache;
    /**
     * 这两个代表的是以 DoubleLinkedList 连接成的双向链表的头尾节点，
     * 且为哨兵节点。每个list中，又包含一个由 node 组成的一个双向链表。
     * 最外层双向链表中，freq 频次较大的 list 在前面，较小的 list 在后面
     */
    DoubleLinkedList firstLinkedList, lastLinkedList;
    int capacity;
    int size;

    public LFUCache3(int capacity){
        this.capacity = capacity;
        cache = new HashMap<>();
        //初始化外层链表的头尾节点，作为哨兵节点
        firstLinkedList = new DoubleLinkedList();
        lastLinkedList = new DoubleLinkedList();
        firstLinkedList.next = lastLinkedList;
        lastLinkedList.pre = firstLinkedList;
    }

    //存储具体键值对信息的node
    private class Node {
        int key;
        int value;
        int freq = 1;
        Node pre;
        Node next;
        DoubleLinkedList doubleLinkedList;

        public Node(){

        }

        public Node(int key, int value){
            this.key = key;
            this.value = value;
        }
    }

    public int get(int key){
        Node node = cache.get(key);
        if(node == null) return -1;
        freqInc(node);
        return node.value;
    }

    public void put(int key, int value){
        if(capacity == 0) return;
        Node node = cache.get(key);
        if(node != null){
            node.value = value;
            freqInc(node);
        }else{
            if(size == capacity){
                /**
                 * 如果满了，则需要把频次最小的，且最久未访问的节点删除
                 * 由于list组成的链表频次从前往后依次减小，故最小的频次list是 lastLinkedList.pre
                 * list中的双向node链表采用的是头插法，因此最久未访问的元素是 lastLinkedList.pre.tail.pre
                 */
                //最小频次list
                DoubleLinkedList list = lastLinkedList.pre;
                //最久未访问的元素，需要删除
                Node deadNode = list.tail.pre;
                cache.remove(deadNode.key);
                list.removeNode(deadNode);
                size--;
                //如果删除deadNode之后，此list中的双向链表空了，则删除此list
                if(list.isEmpty()){
                    removeDoubleLinkedList(list);
                }
            }
            //没有满，则新建一个node
            Node newNode = new Node(key, value);
            cache.put(key,newNode);
            //判断频次为1的list是否存在，不存在则新建
            DoubleLinkedList list = lastLinkedList.pre;
            if(list.freq != 1){
                DoubleLinkedList newList = new DoubleLinkedList(1);
                addDoubleLinkedList(newList,list);
                newList.addNode(newNode);
            }else{
                list.addNode(newNode);
            }
            size++;
        }
    }

    //修改频次
    private void freqInc(Node node){
        //从当前频次的list中移除当前 node
        DoubleLinkedList list = node.doubleLinkedList;
        if(list != null){
            list.removeNode(node);
        }
        //如果当前list中的双向node链表空，则删除此list
        if(list.isEmpty()){
            removeDoubleLinkedList(list);
        }
        //当前node频次加1
        node.freq++;
        //找到当前list前面的list，并把当前node加入进去
        DoubleLinkedList preList = list.pre;
        //如果前面的list不存在，则新建一个，并插入到由list组成的双向链表中
        //前list的频次不等于当前node频次，则说明不存在
        if(preList.freq != node.freq){
            DoubleLinkedList newList = new DoubleLinkedList(node.freq);
            addDoubleLinkedList(newList,preList);
            newList.addNode(node);
        }else{
            preList.addNode(node);
        }

    }

    //从外层双向链表中删除当前list节点
    public void removeDoubleLinkedList(DoubleLinkedList list){
        list.pre.next = list.next;
        list.next.pre = list.pre;
    }

    //知道了它的前节点，即可把新的list节点插入到其后面
    public void addDoubleLinkedList(DoubleLinkedList newList, DoubleLinkedList preList){
        newList.pre = preList;
        newList.next = preList.next;
        preList.next.pre = newList;
        preList.next = newList;
    }

    //维护一个双向DoubleLinkedList链表 + 双向Node链表的结构
    private class DoubleLinkedList {
        //当前list中的双向Node链表所有频次都相同
        int freq;
        //当前list中的双向Node链表的头结点
        Node head;
        //当前list中的双向Node链表的尾结点
        Node tail;
        //当前list的前一个list
        DoubleLinkedList pre;
        //当前list的后一个list
        DoubleLinkedList next;

        public DoubleLinkedList(){
            //初始化内部链表的头尾节点，并作为哨兵节点
            head = new Node();
            tail = new Node();
            head.next = tail;
            tail.pre = head;
        }

        public DoubleLinkedList(int freq){
            head = new Node();
            tail = new Node();
            head.next = tail;
            tail.pre = head;
            this.freq = freq;
        }

        //删除当前list中的某个node节点
        public void removeNode(Node node){
            node.pre.next = node.next;
            node.next.pre = node.pre;
        }

        //头插法将新的node插入到当前list，并在新node中记录当前list的引用
        public void addNode(Node node){
            node.pre = head;
            node.next = head.next;
            head.next.pre = node;
            head.next = node;
            node.doubleLinkedList = this;
        }

        //当前list中的双向node链表是否存在有效节点
        public boolean isEmpty(){
            //只有头尾哨兵节点，则说明为空
            return head.next == tail;
        }
    }


}
```

由于，此方案全是链表的增删操作，因此时间复杂度可到 O(1)。





## Redis数据持久化

- 持久化策略

  - RDB持久化
    - 父进程在保存 RDB 文件时唯一要做的就是 fork 出一个子进程，然后这个子进程就会处理接下来的所有保存工作，父进程无须执行任何磁盘 I/O 操作。
      - 每次保存 RDB 的时候，Redis 都要 fork() 出一个子进程，在数据集比较庞大时， fork() 可能会非常耗时
      - 虽然 AOF 重写也需要进行 fork() ，但无论 AOF 重写的执行间隔有多长，数据的耐久性都不会有任何损失
    - RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快
    - 你可能会至少 5 分钟才保存一次 RDB 文件。 在这种情况下， 一旦发生故障停机， 你就可能会丢失好几分钟的数据
  - AOF持久化
    - 记录服务器执行的所有写操作命令，并在服务器启动时，通过重新执行这些命令来还原数据集
    - Redis 还可以在后台对 AOF 文件进行重写（rewrite），使得 AOF 文件的体积不会超出保存数据集状态所需的实际大小
    - fsync策略
      - 无fsync,每秒fsync,每次写的时候fsync.
      - fsync是由后台线程进行处理的,主线程会尽力处理客户端请求
    - 根据所使用的 fsync 策略，AOF 的速度可能会慢于 RDB 。
    - 处理巨大的写入载入时，RDB 可以提供更有保证的最大延迟时间（latency）

- 如果AOF文件损坏了怎么办

  - >  为现有的 AOF 文件创建一个备份。
    >
    >  使用 Redis 附带的 redis-check-aof 程序，对原来的 AOF 文件进行修复: $ redis-check-aof –fix 
    >
    >  使用 diff -u 对比修复后的 AOF 文件和原始 AOF 文件的备份，查看两个文件之间的不同之处。（可选）
    >
    >  重启 Redis 服务器，等待服务器载入修复后的 AOF 文件，并进行数据恢复。

- 当 Redis 启动时， 如果 RDB持久化和 AOF持久化都被打开了，那么程序会优先使用 AOF文件来恢复数据集， 因为 AOF 文件所保存的数据通常是最完整的

- 触发rdbSave过程的方式

  - > save命令：阻塞Redis服务器进程，直到RDB文件创建完毕为止。
    >
    > bgsave命令：派生出一个子进程，然后由子进程负责创建RDB文件，父进程继续处理命令请求。
    >
    > master接收到slave发来的sync命令
    >
    > 定时save(配置文件中制定）

- RDB文件结构

  - > REDIS长度为5个字节，程序在载入文件时，可以快速检查所载入的文件是否是RDB文件。
    >
    > db_version长度为4个字节，一个字符串表示的整数，记录了RDB文件的版本号。
    >
    > database包含零个或任意多个数据库，以及键值对的数据
    >
    > EOF常量的长度为1个字节，标志着RDB文件正文内容的结束。
    >
    > heck_sum是一个8字节长的无符号整数，保存着一个校验和，由前面四部分计算得出的。

- AOF配置文件

  - > Appendonly yes //启用AOF持久化方式
    >
    > Appendfsync always //收到写命令就立即写入磁盘（最慢）但是保证完全的持久化
    >
    > Appendfsync everysec //每秒写入磁盘一次，在性能和持久化方面做了很好的折中（默认）
    >
    > Appendfsync no //完全依赖os,性能最好，持久化没有保证。

  
  
- 由于Redis的数据都存放在内存中，如果没有配置持久化，redis重启后数据就全丢失了，于是需要开启redis的持久化功能，将数据保存到磁盘上，当redis重启后，可以从磁盘中恢复数据。redis提供两种方式进行持久化，一种是RDB持久化（原理是将Reids在内存中的数据库记录定时dump到磁盘上的RDB持久化），另外一种是AOF持久化（原理是将Reids的操作日志以追加的方式写入文件）。

  **2、二者的区别**

  RDB持久化是指在指定的时间间隔内将内存中的数据集快照写入磁盘，实际操作过程是fork一个子进程，先将数据集写入临时文件，写入成功后，再替换之前的文件，用二进制压缩存储。

  AOF持久化以日志的形式记录服务器所处理的每一个写、删除操作，查询操作不会记录，以文本的方式记录，可以打开文件看到详细的操作记录。

  

  **3、二者优缺点**

  RDB存在哪些优势呢？

  1). 一旦采用该方式，那么你的整个Redis数据库将只包含一个文件，这对于文件备份而言是非常完美的。比如，你可能打算每个小时归档一次最近24小时的数据，同时还要每天归档一次最近30天的数据。通过这样的备份策略，一旦系统出现灾难性故障，我们可以非常容易的进行恢复。

  2). 对于灾难恢复而言，RDB是非常不错的选择。因为我们可以非常轻松的将一个单独的文件压缩后再转移到其它存储介质上。

  3). 性能最大化。对于Redis的服务进程而言，在开始持久化时，它唯一需要做的只是fork出子进程，之后再由子进程完成这些持久化的工作，这样就可以极大的避免服务进程执行IO操作了。

  4). 相比于AOF机制，如果数据集很大，RDB的启动效率会更高。

  **RDB又存在哪些劣势呢？**

  1). 如果你想保证数据的高可用性，即最大限度的避免数据丢失，那么RDB将不是一个很好的选择。因为系统一旦在定时持久化之前出现宕机现象，此前没有来得及写入磁盘的数据都将丢失。

  2). 由于RDB是通过fork子进程来协助完成数据持久化工作的，因此，如果当数据集较大时，可能会导致整个服务器停止服务几百毫秒，甚至是1秒钟。

  **AOF的优势有哪些呢？**

  1). 该机制可以带来更高的数据安全性，即数据持久性。Redis中提供了3中同步策略，即每秒同步、每修改同步和不同步。事实上，每秒同步也是异步完成的，其效率也是非常高的，所差的是一旦系统出现宕机现象，那么这一秒钟之内修改的数据将会丢失。而每修改同步，我们可以将其视为同步持久化，即每次发生的数据变化都会被立即记录到磁盘中。可以预见，这种方式在效率上是最低的。至于无同步，无需多言，我想大家都能正确的理解它。

  2). 由于该机制对日志文件的写入操作采用的是append模式，因此在写入过程中即使出现宕机现象，也不会破坏日志文件中已经存在的内容。然而如果我们本次操作只是写入了一半数据就出现了系统崩溃问题，不用担心，在Redis下一次启动之前，我们可以通过redis-check-aof工具来帮助我们解决数据一致性的问题。

  3). 如果日志过大，Redis可以自动启用rewrite机制。即Redis以append模式不断的将修改数据写入到老的磁盘文件中，同时Redis还会创建一个新的文件用于记录此期间有哪些修改命令被执行。因此在进行rewrite切换时可以更好的保证数据安全性。

  4). AOF包含一个格式清晰、易于理解的日志文件用于记录所有的修改操作。事实上，我们也可以通过该文件完成数据的重建。

  **AOF的劣势有哪些呢？**

  1). 对于相同数量的数据集而言，AOF文件通常要大于RDB文件。RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快。

  2). 根据同步策略的不同，AOF在运行效率上往往会慢于RDB。总之，每秒同步策略的效率是比较高的，同步禁用策略的效率和RDB一样高效。

  二者选择的标准，就是看系统是愿意牺牲一些性能，换取更高的缓存一致性（aof），还是愿意写操作频繁的时候，不启用备份来换取更高的性能，待手动运行save的时候，再做备份（rdb）。rdb这个就更有些 eventually consistent的意思了。

  **4、常用配置**

  **RDB持久化配置**

  Redis会将数据集的快照dump到dump.rdb文件中。此外，我们也可以通过配置文件来修改Redis服务器dump快照的频率，在打开6379.conf文件之后，我们搜索save，可以看到下面的配置信息：

  save 900 1              #在900秒(15分钟)之后，如果至少有1个key发生变化，则dump内存快照。

  save 300 10            #在300秒(5分钟)之后，如果至少有10个key发生变化，则dump内存快照。

  save 60 10000        #在60秒(1分钟)之后，如果至少有10000个key发生变化，则dump内存快照。

  **AOF**持久化配置

  在Redis的配置文件中存在三种同步方式，它们分别是：

  appendfsync always     #每次有数据修改发生时都会写入AOF文件。

  appendfsync everysec  #每秒钟同步一次，该策略为AOF的缺省策略。

  appendfsync no          #从不同步。高效但是数据不会被持久化。







## 数据结构

- String（字符串）

  - > SET key value
    >
    > 设置指定 key 的值
    >
    > GET key
    >
    > 获取指定 key 的值。
    >
    > GETRANGE key start end
    >
    > 返回 key 中字符串值的子字符
    >
    > GETSET key value
    >
    > 将给定 key 的值设为 value ，并返回 key 的旧值(old value)。
    >
    > GETBIT key offset对 key
    >
    > 所储存的字符串值，获取指定偏移量上的位(bit)。
    >
    > MGET key1 \[key2..\]
    >
    > 获取所有(一个或多个)给定 key 的值。
    >
    > SETBIT key offset value
    >
    > 对 key 所储存的字符串值，设置或清除指定偏移量上的位(bit)。
    >
    > SETEX key seconds value
    >
    > 将值 value 关联到 key ，并将 key 的过期时间设为 seconds (以秒为单位)。
    >
    > SETNX key value
    >
    > 只有在 key 不存在时设置 key 的值。
    >
    > SETRANGE key offset value
    >
    > 用 value 参数覆写给定 key 所储存的字符串值，从偏移量 offset 开始。
    >
    > STRLEN key
    >
    > 返回 key 所储存的字符串值的长度。
    >
    > MSET key value \[key value ...\]
    >
    > 同时设置一个或多个 key-value 对。
    >
    > MSETNX key value \[key value ...\]
    >
    > 同时设置一个或多个 key-value 对，当且仅当所有给定 key 都不存在。
    >
    > PSETEX key milliseconds value
    >
    > 这个命令和 SETEX 命令相似，但它以毫秒为单位设置 key 的生存时间，而不是像 SETEX 命令那样，以秒为单位。
    >
    > INCR key
    >
    > 将 key 中储存的数字值增一。
    >
    > INCRBY key increment
    >
    > 将 key 所储存的值加上给定的增量值（increment） 。
    >
    > INCRBYFLOAT key increment
    >
    > 将 key 所储存的值加上给定的浮点增量值（increment） 。
    >
    > DECR key
    >
    > 将 key 中储存的数字值减一。
    >
    > DECRBY key decrementkey
    >
    > 所储存的值减去给定的减量值（decrement） 。
    >
    > APPEND key value
    >
    > 如果 key 已经存在并且是一个字符串， APPEND 命令将 指定value 追加到改 key 原来的值（value）的末尾。

- Hash（字典）

  - > HDEL key field1 \[field2\]
    >
    > 删除一个或多个哈希表字段
    >
    > HEXISTS key field
    >
    > 查看哈希表 key 中，指定的字段是否存在。
    >
    > HGET key field
    >
    > 获取存储在哈希表中指定字段的值。
    >
    > HGETALL key
    >
    > 获取在哈希表中指定 key 的所有字段和值
    >
    > HINCRBY key field increment
    >
    > 为哈希表 key 中的指定字段的整数值加上增量 increment 。
    >
    > HINCRBYFLOAT key field increment
    >
    >  为哈希表 key 中的指定字段的浮点数值加上增量 increment 。
    >
    > HKEYS key
    >
    > 获取所有哈希表中的字段
    >
    > HLEN key
    >
    > 获取哈希表中字段的数量
    >
    > HMGET key field1 \[field2\]
    >
    > 获取所有给定字段的值
    >
    > HMSET key field1 value1 \[field2 value2 \]
    >
    > 同时将多个 field-value (域-值)对设置到哈希表 key 中。
    >
    > HSET key field value
    >
    > 将哈希表 key 中的字段 field 的值设为 value 。
    >
    > HSETNX key field value
    >
    > 只有在字段 field 不存在时，设置哈希表字段的值。
    >
    > HVALS key
    >
    > 获取哈希表中所有值
    >
    > HSCAN key cursor \[MATCH pattern\] \[COUNT count\]
    >
    > 迭代哈希表中的键值对。

- LIST（列表）

  - > BLPOP key1 \[key2 \] timeout
    >
    > 移出并获取列表的第一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
    >
    > BRPOP key1 \[key2 \] timeout
    >
    > 移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
    >
    > BRPOPLPUSH source destination timeout
    >
    > 从列表中弹出一个值，将弹出的元素插入到另外一个列表中并返回它； 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。
    >
    > LINDEX key index
    >
    > 通过索引获取列表中的元素
    >
    > LINSERT key BEFORE|AFTER pivot value
    >
    > 在列表的元素前或者后插入元素
    >
    > LLEN key
    >
    > 获取列表长度
    >
    > LPOP key
    >
    > 移出并获取列表的第一个元素
    >
    > LPUSH key value1 \[value2\]
    >
    > 将一个或多个值插入到列表头部
    >
    > LPUSHX key value
    >
    > 将一个值插入到已存在的列表头部
    >
    > LRANGE key start stop
    >
    > 获取列表指定范围内的元素
    >
    > LREM key count value
    >
    > 移除列表元素
    >
    > LSET key index value
    >
    > 通过索引设置列表元素的值
    >
    > LTRIM key start stop
    >
    > 对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。
    >
    > RPOP key
    >
    > 移除并获取列表最后一个元素
    >
    > RPOPLPUSH source destination
    >
    > 移除列表的最后一个元素，并将该元素添加到另一个列表并返回
    >
    > RPUSH key value1 \[value2\]
    >
    > 在列表中添加一个或多个值
    >
    > RPUSHX key value
    >
    > 为已存在的列表添加值

- SET(集合)

  - > SADD key member1 \[member2\]
    >
    > 向集合添加一个或多个成员
    >
    > SCARD key
    >
    > 获取集合的成员数
    >
    > SDIFF key1 \[key2\]
    >
    > 返回给定所有集合的差集
    >
    > SDIFFSTORE destination key1 \[key2\]
    >
    > 返回给定所有集合的差集并存储在 destination 中
    >
    > SINTER key1 \[key2\]
    >
    > 返回给定所有集合的交集
    >
    > SINTERSTORE destination key1 \[key2\]
    >
    > 返回给定所有集合的交集并存储在 destination 中
    >
    > SISMEMBER key member
    >
    > 判断 member 元素是否是集合 key 的成员
    >
    > SMEMBERS key
    >
    > 返回集合中的所有成员
    >
    > SMOVE source destination member
    >
    > 将 member 元素从 source 集合移动到 destination 集合
    >
    > SPOP key
    >
    > 移除并返回集合中的一个随机元素
    >
    > SRANDMEMBER key \[count\]
    >
    > 返回集合中一个或多个随机数
    >
    > SREM key member1 \[member2\]
    >
    > 移除集合中一个或多个成员
    >
    > SUNION key1 \[key2\]
    >
    > 返回所有给定集合的并集
    >
    > SUNIONSTORE destination key1 \[key2\]
    >
    > 所有给定集合的并集存储在 destination 集合中
    >
    > SSCAN key cursor \[MATCH pattern\] \[COUNT count\]
    >
    > 迭代集合中的元素

- SortedSet（有序集合）

  - > ZADD key score1 member1 \[score2 member2\]
    >
    > 向有序集合添加一个或多个成员，或者更新已存在成员的分数
    >
    > ZCARD key
    >
    > 获取有序集合的成员数
    >
    > ZCOUNT key min max
    >
    > 计算在有序集合中指定区间分数的成员数
    >
    > ZINCRBY key increment member
    >
    > 有序集合中对指定成员的分数加上增量 increment
    >
    > ZINTERSTORE destination numkeys key \[key ...\]
    >
    > 计算给定的一个或多个有序集的交集并将结果集存储在新的有序集合 key 中
    >
    > ZLEXCOUNT key min max
    >
    > 在有序集合中计算指定字典区间内成员数量
    >
    > ZRANGE key start stop \[WITHSCORES\]
    >
    > 通过索引区间返回有序集合成指定区间内的成员
    >
    > ZRANGEBYLEX key min max \[LIMIT offset count\]
    >
    > 通过字典区间返回有序集合的成员
    >
    > ZRANGEBYSCORE key min max \[WITHSCORES\] \[LIMIT\]
    >
    > 通过分数返回有序集合指定区间内的成员
    >
    > ZRANK key member
    >
    > 返回有序集合中指定成员的索引
    >
    > ZREM key member \[member ...\]
    >
    > 移除有序集合中的一个或多个成员
    >
    > ZREMRANGEBYLEX key min max
    >
    > 移除有序集合中给定的字典区间的所有成员
    >
    > ZREMRANGEBYRANK key start stop
    >
    > 移除有序集合中给定的排名区间的所有成员
    >
    > ZREMRANGEBYSCORE key min max
    >
    > 移除有序集合中给定的分数区间的所有成员
    >
    > ZREVRANGE key start stop \[WITHSCORES\]
    >
    > 返回有序集中指定区间内的成员，通过索引，分数从高到底
    >
    > ZREVRANGEBYSCORE key max min \[WITHSCORES\]
    >
    > 返回有序集中指定分数区间内的成员，分数从高到低排序
    >
    > ZREVRANK key member
    >
    > 返回有序集合中指定成员的排名，有序集成员按分数值递减(从大到小)排序
    >
    > ZSCORE key member
    >
    > 返回有序集中，成员的分数值
    >
    > ZUNIONSTORE destination numkeys key \[key ...\]
    >
    > 计算给定的一个或多个有序集的并集，并存储在新的 key 中
    >
    > ZSCAN key cursor \[MATCH pattern\] \[COUNT count\]
    >
    > 迭代有序集合中的元素（包括元素成员和元素分值）

  - > WITHSCORES ：显示score







## Redis慢日志查询

- 可以通过改写 redis.conf 文件或者用 CONFIG GET 和 CONFIG SET 命令对它们动态地进行修改、

  - > slowlog-log-slower-than 10000 超过多少微秒
    >
    > CONFIG SET slowlog-log-slower-than 100 
    >
    > CONFIG SET slowlog-max-len 1000 保存多少条慢日志
    > CONFIG GET slow*                             
    > SLOWLOG GET
    > SLOWLOG RESET



## Redis高可用

Redis 中为了实现高可用采用了如下两个方式：

- 主从复制数据。
- 采用哨兵监控数据节点的运行情况，一旦主节点出现问题由从节点顶上继续进行服务。



## Redis哨兵

**哨兵模式介绍**

在将哨兵模式之前，先来说说**主从复制**的缺点吧。

如果主节点出了问题，那么主节点不在提供服务，需要手动的将从节点切换成主节点。

所以这个时候哨兵模式就出现啦，当主节出现故障时，Redis Sentinel 会自动的发现主节点的故障并转移，并通知应用方，实现高可用。

**下面是 Redis 官方文档对于哨兵功能的描述：**

监控（Monitoring）：哨兵会不断地检查主节点和从节点是否运作正常。

自动故障转移（Automatic failover）：当主节点不能正常工作时，哨兵会开始自动故障转移操作，它会将失效主节点的其中一个从节点升级为新的主节点，并让其他从节点改为复制新的主节点。

配置提供者（Configuration provider）：客户端在初始化时，通过连接哨兵来获得当前 Redis 服务的主节点地址。

通知（Notification）：哨兵可以将故障转移的结果发送给客户端。

哨兵模式的结构拓扑图大概如下\



![img](https://pic2.zhimg.com/v2-b1947399efbe475689bf7b9c0122ef00_b.jpg)



**大意就是：**

每一个哨兵节点会监听其他的哨兵节点以及 Master 和所有的 Slave

所有哨兵节点会定期的 ping 主节点，监控是否正常

如果认为主节点出现故障的哨兵数量达到阈值，就判定主节点死掉，主节点就会客观下线

主节点客观下线后，哨兵节点通过选举模式在 Slave 中选择出一个升级为主节点

其他的 salve 指向新的主节点

原来的 Master 变成 Slave，并且指向新的主节点



作者：华为云开发者社区

链接：https://zhuanlan.zhihu.com/p/65949577

来源：知乎

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

**哨兵机制概述**

Redis 使用哨兵机制来实现高可用的大概工作原理是：

- Redis 使用一组哨兵（Sentinel）节点来监控主从 Redis 服务的可用性。
- 一旦发现 Redis 主节点失效，将选举出一个哨兵节点作为领导者（Leader）。
- 哨兵领导者再从剩余的从 Redis 节点中选出一个 Redis 节点作为新的主 Redis 节点对外服务。

以上将 Redis 节点分为两类：

- 哨兵节点（Sentinel）：负责监控节点的运行情况。
- 数据节点：即正常服务客户端请求的 Redis 节点，有主从之分。

以上是大体的流程，这个流程需要解决以下几个问题：

- 如何对 Redis 数据节点进行监控？
- 如何确定一个 Redis 数据节点失效？
- 如何选择出一个哨兵领导者节点？
- 哨兵节点选择新的主 Redis 节点的依据是什么？

以下来逐个回答这些问题。

#### **三个监控任务**

哨兵节点通过三个定时监控任务监控 Redis 数据节点的服务可用性。

##### **①info 命令**

![img](https://pic2.zhimg.com/v2-304608ed545318fde6ffeaaf9ae7e668_b.jpg)

每隔 10 秒，每个哨兵节点都会向主、从 Redis 数据节点发送 info 命令，获取新的拓扑结构信息。

Redis 拓扑结构信息包括了：

- **本节点角色：主或从。**
- **主从节点的地址、端口信息。**

这样，哨兵节点就能从 info 命令中自动获取到从节点信息，因此那些后续才加入的从节点信息不需要显式配置就能自动感知。

##### **②向 __sentinel__:hello 频道同步信息**

每隔 2 秒，每个哨兵节点将会向 Redis 数据节点的 __sentinel__:hello 频道同步自身得到的主节点信息以及当前哨兵节点的信息。

由于其他哨兵节点也订阅了这个频道，因此实际上这个操作可以交换哨兵节点之间关于主节点以及哨兵节点的信息。

![img](https://pic1.zhimg.com/v2-c411d102cfa7c7cb9d73ccdb74ae4e9e_b.png)

这一操作实际上完成了两件事情：

- **发现新的哨兵节点：**如果有新的哨兵节点加入，此时保存下来这个新哨兵节点的信息，后续与该哨兵节点建立连接。
- **交换主节点的状态信息，**作为后续客观判断主节点下线的依据。

**③向数据节点做心跳探测**

![img](https://picb.zhimg.com/v2-b38805200c647e0add75f743c10c1443_b.jpg)

每隔 1 秒，每个哨兵节点向主、从数据节点以及其他 Sentinel 节点发送 Ping 命令做心跳探测，这个心跳探测是后续主观判断数据节点下线的依据。

#### **主观下线和客观下线**

##### **①主观下线**

上面三个监控任务中的第三个探测心跳任务，如果在配置的 down-after-milliseconds 之后没有收到有效回复，那么就认为该数据节点“主观下线（sdown）”。

![img](https://pic4.zhimg.com/v2-caef27138ac2642566ceb2342b2d16df_b.jpg)

为什么称为“主观下线”？因为在一个分布式系统中，有多个机器在一起联动工作，网络可能出现各种状况，仅凭一个节点的判断还不足以认为一个数据节点下线了，这就需要后面的“客观下线”。

##### **②客观下线**

![img](https://pic4.zhimg.com/v2-d3a7221c945dd5824d942ffa19bd29f9_b.jpg)



当一个哨兵节点认为主节点主观下线时，该哨兵节点需要通过”sentinel is-master-down-by addr”命令向其他哨兵节点咨询该主节点是否下线了，如果有超过半数的哨兵节点都回答了下线，此时认为主节点“客观下线”。

#### **选举哨兵领导者**

当主节点客观下线时，需要选举出一个哨兵节点做为哨兵领导者，以完成后续选出新的主节点的工作。

![img](https://pic1.zhimg.com/v2-19cec01c23a7b119298680badcab1922_b.jpg)

这个选举的大体思路是：

- 每个哨兵节点通过向其他哨兵节点发送”sentinel is-master-down-by addr”命令来申请成为哨兵领导者。
- 而每个哨兵节点在收到一个”sentinel is-master-down-by addr”命令时，只允许给第一个节点投票，其他节点的该命令都会被拒绝。
- 如果一个哨兵节点收到了半数以上的同意票，则成为哨兵领导者。
- 如果前面三步在一定时间内都没有选出一个哨兵领导者，将重新开始下一次选举。

可以看到，这个选举领导者的流程很像 Raft 中选举 Leader 的流程。

#### **选出新的主节点**

![img](https://pic4.zhimg.com/v2-78fb6129277e438283c0a44da78c7621_b.jpg)

在剩下的 Redis 从节点中，按照以下顺序来选择新的主节点：

- 过滤掉“不健康”的数据节点：比如主观下线、断线的从节点、五秒内没有回复过哨兵节点 Ping 命令的节点、与主节点失联的从节点。
- 选择 Slave-Priority（从节点优先级）最高的从节点，如果存在则返回，不存在则继续后面的流程。
- 选择复制偏移量最大的从节点，这意味着这个从节点上面的数据最完整，如果存在则返回，不存在则继续后面的流程。
- 到了这里，所有剩余从节点的状态都是一样的，选择 runid 最小的从节点。

#### **提升新的主节点**

![img](https://pic3.zhimg.com/v2-c015dff886575a136e7269565da980af_b.jpg)

选择了新的主节点之后，还需要最后的流程让该节点成为新的主节点：

- 哨兵领导者向上一步选出的从节点发出“slaveof no one”命令，让该节点成为主节点。
- 哨兵领导者向剩余的从节点发送命令，让它们成为新主节点的从节点。
- 哨兵节点集合会将原来的主节点更新为从节点，当其恢复之后命令它去复制新的主节点的数据。



## Redis主从复制

- 复制的过程

  - ![img](https://pic4.zhimg.com/v2-5f291b42b56630c401b83bb90a05f35f_b.jpg)
  - 从节点执行 slaveof 命令
  - 从节点只是保存了 slaveof 命令中主节点的信息，并没有立即发起复制
  - 从节点内部的定时任务发现有主节点的信息，开始使用 socket 连接主节点
  - 连接建立成功后，发送 ping 命令，希望得到 pong 命令响应，否则会进行重连
  - 如果主节点设置了权限，那么就需要进行权限验证；如果验证失败，复制终止。
  - 权限验证通过后，进行数据同步，这是耗时最长的操作，主节点将把所有的数据全部发送给从节点。
  - 当主节点把当前的数据同步给从节点后，便完成了复制的建立流程。接下来，主节点就会持续的把写命令发送给从节点，保证主从数据一致性。

- 数据间的同步

  - psync 命令需要 3 个组件支持：

    - 主从节点各自复制偏移量
      - 主节点在处理完写入命令后，会把命令的字节长度做累加记录，统计信息在 info replication  中的 master_repl_offset 指标中。 
      -  从节点每秒钟上报自身的的复制偏移量给主节点，因此主节点也会保存从节点的复制偏移量。  
      - 从节点在接收到主节点发送的命令后，也会累加自身的偏移量，统计信息在 info replication 中。 
      -  通过对比主从节点的复制偏移量，可以判断主从节点数据是否一致
    - 主节点复制积压缓冲区
      - 复制积压缓冲区是一个保存在主节点的一个固定长度的先进先出的队列。默认大小 1MB。 
      -  这个队列在 slave 连接时创建。这时主节点响应写命令时，不但会把命令发送给从节点，也会写入复制缓冲区。  
      - 他的作用就是用于部分复制和复制命令丢失的数据补救。
      -  通过 info replication 可以看到相关信息。
    - 主节点运行 ID
      - 每个 redis 启动的时候，都会生成一个 40 位的运行 ID。  
      - 运行 ID 的主要作用是用来识别 Redis 节点。如果使用 ip+port 的方式，那么如果主节点重启修改 了 RDB/AOF 数据，从节点再基于偏移量进行复制将是不安全的。所以，当运行 id 变化后，从节点将 进行全量复制。也就是说，redis 重启后，默认从节点会进行全量复制。  
      - 如果在重启时不改变运行 ID 呢？ 可以通过 debug reload 命令重新加载 RDB 并保持运行 ID 不变。从而有效的避免不必要的全量复制。 他的缺点则是：debug reload 命令会阻塞当前 Redis 节点主线程，因此对于大数据量的主节点或者 无法容忍阻塞的节点，需要谨慎使用。
      -   一般通过故障转移机制可以解决这个问题。

  - psync 命令的使用方式

    - > 命令格式为 psync {runId} {offset}
      > runId : 从节点所复制主节点的运行 id
      > offset：当前从节点已复制的数据偏移量

    - 从节点发送 psync 命令给主节点，runId 就是目标主节点的 ID，如果没有默认为 -1，offset 是从节点保存的复制偏移量，如果是第一次复制则为 -1.
    - 主节点会根据 runid 和 offset 决定返回结果：
      - 如果回复 +FULLRESYNC {runId} {offset} ，那么从节点将触发全量复制流程。
      - 如果回复 +CONTINUE，从节点将触发部分复制。
      - 如果回复 +ERR，说明主节点不支持 2.8 的 psync  命令，将使用 sync 执行全量复制。

- 全量复制

  - > 1. 发送 psync 命令（spync ？ -1）
    > 2. 主节点根据命令返回  FULLRESYNC
    > 3. 从节点记录主节点 ID 和 offset
    > 4. **主节点 bgsave 并保存 RDB 到本地**
    > 5. **主节点发送 RBD 文件到从节点**
    > 6. **从节点收到 RDB 文件并加载到内存中**
    > 7. 主节点在从节点接受数据的期间，将新数据保存到“复制客户端缓冲区”，当从节点加载 RDB 完毕，再发送过去。（如果从节点花费时间过长，将导致缓冲区溢出，最后全量同步失败）
    > 8. **从节点清空数据后加载 RDB 文件，如果 RDB 文件很大，这一步操作仍然耗时，如果此时客户端访问，将导致数据不一致，可以使用配置slave-server-stale-data 关闭**.
    > 9. **从节点成功加载完 RBD 后，如果开启了 AOF，会立刻做 bgrewriteaof**。
    >
    > **以上加粗的部分是整个全量同步耗时的地方。**

- 部分复制

  - 当从节点正在复制主节点时，如果出现网络闪断和其他异常，从节点会让主节点补发丢失的命令数据，主节点只需要将复制缓冲区的数据发送到从节点就能够保证数据的一致性，相比较全量复制，成本小很多。

  - > 当从节点出现网络中断，超过了 repl-timeout 时间，主节点就会中断复制连接。
    >
    > 主节点会将请求的数据写入到“复制积压缓冲区”，默认 1MB。
    >
    > 当从节点恢复，重新连接上主节点，从节点会将 offset 和主节点 id 发送到主节点
    >
    > 主节点校验后，如果偏移量的数后的数据在缓冲区中，就发送 continue 响应 —— 表示可以进行部分复制
    >
    > 主节点将缓冲区的数据发送到从节点，保证主从复制进行正常状态。

- 心跳

  - 心跳检测机制，各自模拟成对方的客户端进行通信，通过 client list 命令查看复制相关客户端信息，主节点的连接状态为 flags = M，从节点的连接状态是 flags = S。
  - 主节点默认每隔  10 秒对从节点发送  ping 命令，可修改配置 repl-ping-slave-period 控制发送频率。
  - 从节点在主线程每隔一秒发送 replconf ack{offset} 命令，给主节点上报自身当前的复制偏移量。
  - 主节点收到 replconf 信息后，判断从节点超时时间，如果超过 repl-timeout 60 秒，则判断节点下线。
  - 为了降低主从延迟，一般把 redis 主从节点部署在相同的机房/同城机房，避免网络延迟带来的网络分区造成的心跳中断等情况。

- 异步复制

  - 主节点不但负责数据读写，还负责把写命令同步给从节点，写命令的发送过程是异步完成，也就是说主节点处理完写命令后立即返回客户度，并不等待从节点复制完成。

  





## Redis 集群

- Redis 集群
  - Redis 集群是一个可以在多个 Redis 节点之间进行数据共享的设施
  - 不同的master可以拥有不同数量的slave，且集群中任意一个节点都与其它所有节点建立了连接，每一个节点都在称为cluster port的端口监听其它节点的集群通信连接。
  - redis3.0上加入了cluster模式，实现的redis的分布式存储，也就是说每台redis节点上存储不同的内容
  
- Redis集群数据结构

  -   与集群相关的数据结构主要有**clusterState与clusterNode两个**(在cluster.h源文件中声明)：每一个redis实例拥有唯一一个clusterState实例，即server.cluster；而clusterNode的实例与集群中的节点数目n对应，每一个节点上都拥有n个clusterNode实例表示它知道的n个节点的信息，存储在clusterState结构的nodes成员中。 clusterState中有2个与集群结构相关的成员

    - nodes成员是一个字典，以节点nameid为关键字，指向clusterNode的指针为值。 集群中的节点都有一个nameid，以nameid为索引即可在nodes字典中找到描述该节点信息的clusterNode实例

    ![img](https://img2018.cnblogs.com/blog/1766634/201910/1766634-20191007150900326-80135328.png)

    - 而slots是一个clusterNode结构的指针数组，CLUSTER_SLOTS是redis中定义的支持的slot最大值，所有的key计算得到的slot都小于该值。slots[slot]存储着负责该slot的master节点的clusterNode结构的指针。每一个节点上都拥有该slots数组，因此在任意节点上都可以查找到负责某个slot的主节点的信息。

    ![img](https://img2018.cnblogs.com/blog/1766634/201910/1766634-20191007151048799-1878638016.png)

  - clusterNode结构描述了一个节点的基本信息，如ip，port, cluster port等

  - name即是nodes字典中用作关键字的节点nameid

    slots与clusterState中的slots有所不同，这里以bit的索引作为slot值，以该bit的状态标识该clusterNode对应的节点是否负责该slot。

    slaves与slaveof代表了节点之间的master-slave关系。如果这是一个master节点，那么它的slave节点的clusterNode指针存储在slaves数组中；如果这是一个slave节点，那么slaveof指向了它的master节点的clusterNode。

    link，指向当前节点与该clusterNode代表的节点之间的连接的相关信息，节点之间通过该link定期发送ping/pong消息。

- Redis集群通信结构

  - Redis集群中的节点没有namenode与datanode的区别，每一个节点都维护了所有节点的信息。clusterNode结构中的link指向了当前节点与clusterNode所代表的节点之间的连接，Redis中每一个节点都与它所知道的所有节点之间维护了一个连接，通过这些连接发ping/pong消息，同步集群信息。集群中任意两个节点之间都建立了两个tcp连接，例如有nodeA与nodeB，那么nodeA中代表nodeB的clusterNode中有一个link维护了A主动与B建立的连接，而nodeB中代表nodeA的clusterNode中也有一个link维护了B主动与A建立的连接，即构建了一个全双工的通信链路。假设集群中存在3个节点，那么它们之间的通信结构如下所示：

    ![img](https://img2018.cnblogs.com/blog/1766634/201910/1766634-20191007151629019-940324846.png)

- link的建立与节点发现
  
  -  每一个节点都会在cluster port端口监听tcp连接请求，参见clusterInit函数，并且每个节点都有一个定时任务clusterCron，其中会遍历nodes字典，检测其中的clusterNode的link是否建立，如果没有建立连接，那么会主动连接该clusterNode所代表的节点建立连接。如果nodes字典中没有某个节点clusterNode结构，那么便不会与它建立连接。
    
      建立clusterNode的时机大致有如下几处：
    
    1. 从文件中加载节点信息建立 clusterNode结构，在函数clusterLoadConfig中。
    2. 客户端执行meet命令告知节点信息，建立相应的clusterNode结构，由函数clusterCommand调用clusterStartHandshake完成。
    3. 接收到meet类消息，建立与发送方对应的clusterNode结构，在函数clusterProcessPacket中。
    4. 接收到的ping/pong/meet消息中带有其它不知道的节点信息，建立相应的clusterNode结构，同样在clusterProcessPacket函数中，调用clusterStartHandshake完成。
    
      新建立的clusterNode的nameid是随机的，并且此时的clusterNode中flag设置为CLUSTER_NODE_HANDSHAKE状态，表示尚未首次通信。当clusterCron中建立相应的link，并发送ping/meet消息，收到响应消息(Pong)时去除CLUSTER_NODE_HANDSHAKE状态，并将clusterNode的nameid修改为响应消息中附带的nameid，至此成功建立一个方向的连接，反方向的连接由对方主动发起建立。 
  
- Redis处理通信的函数结构

  - clusterInit中监听端口cport，注册读事件，响应函数为clusterAcceptHandler。
  - clusterCron中主动建立连接，并将连接结构保存到clusterNode中的link指针中。注册读事件，响应函数为clusterReadHandler，并主动发送ping/meet消息，若先前未注册写事件，则为该link注册写事件，响应函数为clusterWriteHandler。
  - clusterAcceptHandler中接受连接后建立link结构(未保存)，并注册读事件，响应函数为clusterReadHandler。
  -  clusterReadHandler中接收数据，当接收到一个完整的消息后，调用clusterProcessPacket函数处理。

  Redis中定义了集群通信消息的结构，每一个消息至少包含一个消息头，而消息头中包含整个消息的长度，因此clusterReadHandler中可以判断是否接收到一个完整的数据包。

- Redis 集群数据共享
  
  - Redis 集群使用数据分片（sharding）而非一致性哈希（consistency hashing）来实现
    - 通常的做法是获取 key 的哈希值，然后根据节点数来求模，但这种做法有其明显的弊端，当我们需要增加或减少一个节点时，会造成大量的 key 无法命中，这种比例是相当高的，所以就有人提出了一致性哈希的概念
    - 一致性哈希有四个重要特征：
      - 均衡性：是指哈希的结果能够尽可能分布到所有的节点中去，这样可以有效的利用每个节点上的资源。
      - 单调性：当节点数量变化时哈希的结果应尽可能的保护已分配的内容不会被重新分派到新的节点。
      - 分散性和负载：这两个其实是差不多的意思，就是要求一致性哈希算法对 key 哈希应尽可能的避免重复。
  - 一个 Redis 集群包含 16384 个哈希槽（hash slot），数据库中的每个键都属于这 16384 个哈希槽的其中一个
    - 使用哈希槽的好处就在于可以方便的添加或移除节点
      - 当需要增加节点时，只需要把其他节点的某些哈希槽挪到新节点就可以了；
      - 当需要移除节点时，只需要把移除节点上的哈希槽挪到其他节点就行了；
      - 集群中的每个主节点（Master）都负责处理16384个哈希槽中的一部分，当集群处于稳定状态时，每个哈希槽都只由一个主节点进行处理，每个主节点可以有一个到N个从节点（Slave），当主节点出现宕机或网络断线等不可用时，从节点能自动提升为主节点进行处理。   
    - 集群使用公式CRC16(key) % 16384 来计算键 key 属于哪个槽， 其中 CRC16(key) 语句用于计算键 key 的 CRC16 校验和 
  - 不足
    - 数据通过异步复制,不保证数据的强一致性
    - 多个业务使用同一套集群时，无法根据统计区分冷热数据，资源隔离性较差，容易出现相互影响的情况
    - slave在集群中充当“冷备”，不能缓解读压力







## Redis性能问题

- **Master调用BGREWRITEAOF重写AOF文件，AOF在重写的时候会占大量的CPU和内存资源，导致服务load过高，出现短暂服务暂停现象**
  - 将no-appendfsync-on-rewrite的配置设为yes可以缓解这个问题，设置为yes表示rewrite期间对新写操作不fsync，暂时存在内存中，等rewrite完成后再写入。
  - 这就相当于将appendfsync设置为no，这说明并没有执行磁盘操作，只是写入了缓冲区，因此这样并不会造成阻塞（因为没有竞争磁盘），但是如果这个时候redis挂掉，就会丢失数据。

- **Redis主从复制的性能问题**
  - Redis的主从复制是建立在内存快照的持久化基础上，只要有Slave就一定会有内存快照发生
  - 由于磁盘io的限制，如果Master快照文件比较大，那么dump会耗费比较长的时间，这个过程中Master可能无法响应请求，也就是说服务会中断
- 根本问题的原因都离不开系统io瓶颈问题，也就是硬盘读写速度不够快，主进程 fsync()/write() 操作被阻塞





## 9个redis命令

### keys

我把这个命令放在第一位，是因为笔者曾经做过的项目，以及一些朋友的项目，都因为使用`keys`这个命令，导致出现性能毛刺。这个命令的时间复杂度是O(N)，而且redis又是单线程执行，在执行keys时即使是时间复杂度只有O(1)例如SET或者GET这种简单命令也会堵塞，从而导致这个时间点性能抖动，甚至可能出现timeout。

> **强烈建议生产环境屏蔽keys命令**（后面会介绍如何屏蔽）。

### scan

既然keys命令不允许使用，那么有什么代替方案呢？有！那就是`scan`命令。如果把keys命令比作类似`select * from users where username like '%afei%'`这种SQL，那么scan应该是`select * from users where id>? limit 10`这种命令。

官方文档用法如下：

```css
SCAN cursor [MATCH pattern] [COUNT count]
```

初始执行scan命令例如`scan 0`。SCAN命令是一个基于游标的迭代器。这意味着命令每次被调用都需要使用上一次这个调用返回的游标作为该次调用的游标参数，以此来延续之前的迭代过程。当SCAN命令的游标参数被设置为0时，服务器将开始一次新的迭代，而**当redis服务器向用户返回值为0的游标时，表示迭代已结束**，这是唯一迭代结束的判定方式，而不能通过返回结果集是否为空判断迭代结束。

使用方式：

```ruby
127.0.0.1:6380> scan 0
1) "22"
2)  1) "23"
    2) "20"
    3) "14"
    4) "2"
    5) "19"
    6) "9"
    7) "3"
    8) "21"
    9) "12"
   10) "25"
   11) "7"
```

返回结果分为两个部分：第一部分即1)就是下一次迭代游标，第二部分即2)就是本次迭代结果集。

### slowlog

上面提到不能使用keys命令，如果就有开发这么做了呢，我们如何得知？与其他任意存储系统例如mysql，mongodb可以查看慢日志一样，redis也可以，即通过命令`slowlog`。用法如下：

```css
SLOWLOG subcommand [argument]
```

subcommand主要有：

-  **get**，用法：slowlog get [argument]，获取argument参数指定数量的慢日志。
-  **len**，用法：slowlog len，总慢日志数量。
-  **reset**，用法：slowlog reset，清空慢日志。

执行结果如下：

```bash
127.0.0.1:6380> slowlog get 5
1) 1) (integer) 2
   2) (integer) 1532656201
   3) (integer) 2033
   4) 1) "flushddbb"
2) 1) (integer) 1  ----  慢日志编码，一般不用care
   2) (integer) 1532646897  ----  导致慢日志的命令执行的时间点，如果api有timeout，可以通过对比这个时间，判断可能是慢日志命令执行导致的
   3) (integer) 26424  ----  导致慢日志执行的redis命令，通过4)可知，执行config rewrite导致慢日志，总耗时26ms+
   4) 1) "config"
      2) "rewrite"
```

> 命令耗时超过多少才会保存到slowlog中，可以通过命令`config set slowlog-log-slower-than 2000`配置并且不需要重启redis。注意：单位是微妙，2000微妙即2毫秒。

### rename-command

为了防止把问题带到生产环境，我们可以通过配置文件重命名一些危险命令，例如`keys`等一些高危命令。操作非常简单，只需要在conf配置文件增加如下所示配置即可：

```undefined
rename-command flushdb flushddbb
rename-command flushall flushallall
rename-command keys keysys
```

### bigkeys

随着项目越做越大，缓存使用越来越不规范。我们如何检查生产环境上一些有问题的数据。`bigkeys`就派上用场了，用法如下：

```undefined
redis-cli -p 6380 --bigkeys
```

执行结果如下：

```python
... ...
-------- summary -------

Sampled 526 keys in the keyspace!
Total key length in bytes is 1524 (avg len 2.90)

Biggest string found 'test' has 10005 bytes
Biggest   list found 'commentlist' has 13 items

524 strings with 15181 bytes (99.62% of keys, avg size 28.97)
2 lists with 19 items (00.38% of keys, avg size 9.50)
0 sets with 0 members (00.00% of keys, avg size 0.00)
0 hashs with 0 fields (00.00% of keys, avg size 0.00)
0 zsets with 0 members (00.00% of keys, avg size 0.00)
```

最后5行可知，没有set,hash,zset几种数据结构的数据。string类型有524个，list类型有两个；通过`Biggest ... ...`可知，最大string结构的key是`test`，最大list结构的key是`commentlist`。

需要注意的是，这个**bigkeys得到的最大，不一定是最大**。说明原因前，首先说明`bigkeys`的原理，非常简单，通过scan命令遍历，各种不同数据结构的key，分别通过不同的命令得到最大的key：

- 如果是string结构，通过`strlen`判断；
- 如果是list结构，通过`llen`判断；
- 如果是hash结构，通过`hlen`判断；
- 如果是set结构，通过`scard`判断；
- 如果是sorted set结构，通过`zcard`判断。

> 正因为这样的判断方式，虽然string结构肯定可以正确的筛选出最占用缓存，也可以说最大的key。但是list不一定，例如，现在有两个list类型的key，分别是：numberlist--[0,1,2]，stringlist--["123456789123456789"]，由于通过llen判断，所以numberlist要大于stringlist。而事实上stringlist更占用内存。其他三种数据结构hash，set，sorted set都会存在这个问题。使用bigkeys一定要注意这一点。

### monitor

假设生产环境没有屏蔽keys等一些高危命令，并且slowlog中还不断有新的keys导致慢日志。那我们如何揪出这些命令是由谁执行的呢？这就是`monitor`的用处，用法如下：

```undefined
redis-cli -p 6380 monitor
```

如果当前redis环境OPS比较高，那么建议结合linux管道命令优化，只输出keys命令的执行情况：

```csharp
[afei@redis ~]# redis-cli -p 6380 monitor | grep keys 
1532645266.656525 [0 10.0.0.1:43544] "keyss" "*"
1532645287.257657 [0 10.0.0.1:43544] "keyss" "44*"
```

执行结果中很清楚的看到keys命名执行来源。通过输出的IP和端口信息，就能在目标服务器上找到执行这条命令的进程，揪出元凶，勒令整改。

### info

如果说哪个命令能最全面反映当前redis运行情况，那么非info莫属。用法如下：

```css
INFO [section]
```

section可选值有：

-  **Server**：运行的redis实例一些信息，包括：redis版本，操作系统信息，端口，GCC版本，配置文件路径等；
-  **Clients**：redis客户端信息，包括：已连接客户端数量，阻塞客户端数量等；
-  **Memory**：使用内存，峰值内存，内存碎片率，内存分配方式。这几个参数都非常重要；
-  **Persistence**：AOF和RDB持久化信息；
-  **Stats**：一些统计信息，最重要三个参数：OPS(`instantaneous_ops_per_sec`)，`keyspace_hits`和`keyspace_misses`两个参数反应缓存命中率；
-  **Replication**：redis集群信息；
-  **CPU**：CPU相关信息；
-  **Keyspace**：redis中各个DB里key的信息；

### config

config是一个非常有价值的命令，主要体现在对redis的运维。因为生产环境一般是不允许随意重启的，不能因为需要调优一些参数就修改conf配置文件并重启。redis作者早就想到了这一点，通过config命令能热修改一些配置，不需要重启redis实例，可以通过如下命令查看哪些参数可以热修改：

```csharp
config get *
```

热修改就比较容易了，执行如下命令即可：

```bash
config set 
```

例如：`config set slowlog-max-len 100`，`config set maxclients 1024`

这样修改的话，如果以后由于某些原因redis实例故障需要重启，那通过config热修改的参数就会被配置文件中的参数覆盖，所以我们需要通过一个命令将config热修改的参数刷到redis配置文件中持久化，通过执行如下命令即可：

```undefined
config rewrite
```

执行该命令后，我们能在config文件中看到类似这种信息：

```kotlin
# 如果conf中本来就有这个参数，通过执行config set，那么redis直接原地修改配置文件
maxclients 1024
# 如果conf中没有这个参数，通过执行config set，那么redis会追加在Generated by CONFIG REWRITE字样后面
# Generated by CONFIG REWRITE
save 600 60
slowlog-max-len 100
```

### set

set命令也能提升逼格？是的，我本不打算写这个命令，但是我见过太多人没有完全掌握这个命令，官方文档介绍的用法如下：

```css
SET key value [EX seconds] [PX milliseconds] [NX|XX]
```

你可能用的比较多的就是`set key value`，或者`SETEX key seconds value`，所以很多同学用redis实现分布式锁分为两步：首先执行`SETNX key value`，然后执行`EXPIRE key seconds`。很明显，这种实现有很严重的问题，因为两步执行不具备原子性，如果执行第一个命令后出现某些未知异常导致无法执行`EXPIRE key seconds`，那么分布式锁就会一直无法得到释放。

通过`SET`命令实现分布式锁的正式姿势应该是`SET key value EX seconds NX`（EX和PX任选，取决于对过期时间精度要求）。另外，value也有要求，最好是一个类似UUID这种具备唯一性的字符串。当然如果问你redis是否还有其他实现分布式锁的方案。你能说出redlock，那对方一定眼前一亮，心里对你竖起大拇指，但嘴上不会说。













## 单线程/并发竞争

- 为什么Redis是单线程的

  - CPU不是Redis的瓶颈，Redis的瓶颈最有可能是机器内存的大小或者网络带宽
  - 这里我们一直在强调的单线程，只是在处理我们的网络请求的时候只有一个线程来处理
    - Redis进行持久化的时候会以子进程或者子线程的方式执行
  - 因为是单一线程，所以同一时刻只有一个操作在进行，所以，耗时的命令会导致并发的下降，不只是读并发，写并发也会下降。而单一线程也只能用到一个CPU核心
    - 我们使用单线程的方式是无法发挥多核CPU 性能，不过我们可以通过在单机开多个Redis 实例来完善
    - 在多处理器情况下，不能充分利用其他CPU。可以的解决方法是开启多个redis服务实例，通过复制和修改配置文件，可以在多个端口上开启多个redis服务实例，这样就可以利用其他CPU来处理连接流
    - 所以可以在同一个多核的服务器中，可以启动多个实例，组成master-master或者master-slave的形式，耗时的读命令可以完全在slave进行
    - 由于是单线程模型，Redis 更喜欢大缓存快速 CPU， 而不是多核

- 并发竞争

  - 多客户端同时并发写一个key，可能本来应该先到的数据后到了，导致数据版本错了。或者是多客户端同时获取一个key，修改值之后再写回去，只要顺序错了，数据就错了。

  - 并发写竞争解决方案

    - 利用redis自带的incr命令

      - 数字值在 Redis 中以字符串的形式保存

      - 可以通过组合使用 INCR 和 EXPIRE，来达到只在规定的生存时间内进行计数(counting)的目的。

      - 客户端可以通过使用 GETSET命令原子性地获取计数器的当前值并将计数器清零

      - 使用其他自增/自减操作，比如 DECR 和 INCRBY ，用户可以通过执行不同的操作增加

        或减少计数器的值，比如在游戏中的记分器就可能用到这些命令

    - 独占锁的方式，类似操作系统的mutex机制

    - 乐观锁的方式进行解决（成本较低，非阻塞，性能较高）

      - 使用redis的命令watch进行构造条件

      - watch这里表示监控该key值，后面的事务是有条件的执行，如果watch的key对应的value值被修改了，则事务不会执行

      - > T1
        > set key1 value1
        > 初始化key1
        > T2
        > watch key1
        > 监控 key1 的键值对
        > T3
        > multi
        > 开启事务
        > T4
        > set key2 value2
        > 设置 key2 的值
        > T5
        > exec
        > 提交事务，Redis 会在这个时间点检测 key1 的值在 T2 时刻后，有没有被其他命令修改过，如果没有，则提交事务去执行

    - 针对客户端来的，在代码里要对redis操作的时候，针对同一key的资源，就先进行加锁（java里的synchronized或lock）。

    - 利用redis的set（使用set来获取锁, Lua 脚本来释放锁）

      - 考虑可以使用SETNX，将 key 的值设为 value ，当且仅当 key 不存在。

      - ```java
        /* 第一个为key，我们使用key来当锁名
           第二个为value，我们传的是uid，唯一随机数，也可以使用本机mac地址 + uuid
           第三个为NX，意思是SET IF NOT EXIST，即当key不存在时，我们进行set操作；若key已经存在，则不做任何操作 
        	第四个为PX，意思是我们要给这个key加一个过期的设置，具体时间由第五个参数决定 
        	第五个为time，代表key的过期时间，对应第四个参数 PX毫秒，EX秒
        */
        String result = jedis.set(key, value, "NX", "PX", expireMillis);
        if (result != null && result.equalsIgnoreCase("OK")) {
        	flag = true;
        }
        
        
        // ---
        
        // 执行脚本的常用命令为 EVAL。 
        // 原子操作：Redis会将整个脚本作为一个整体执行，中间不会被其他命令插入
        // redis.call 函数的返回值就是redis命令的执行结果
        
        
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Object result = jedis.eval(script,Collections.singletonList(fullKey),Collections.singletonList(value));
        if (Objects.equals(UNLOCK_SUCCESS, result)) {
            flag = true;
        }
        
        
        ```

        



## 多路I/O复用模型

- 复用是指一个线程可以服务多条IO流，I/O multiplexing 这里面的 multiplexing 指的其实是在单个线程通过记录跟踪每一个Socket(I/O流)的状态来同时管理多个I/O流。
  - IO多路复用的优势在于，当处理的消耗对比IO几乎可以忽略不计时，可以处理大量的并发IO，而不用消耗太多CPU/内存
  - IO多路复用 + 单进（线）程有个额外的好处，就不会有并发编程的各种坑问题，比如在nginx里，redis里，编程实现都会很简单很多。
  - Java世界里，因为JDBC这个东西是BIO的，所以在我们常见的Java服务里没有办法做到全部NIO化，必须得弄成多线程模型。如果要在做Java web服务这个大场景下享受IO多路复用的好处，要不就是不需要DB的，要不就是得用Vert.X一类的纯NIO框架把DB IO访问也得框进来。
- 进\线程并不是在socket上阻塞  而是在select/epoll上阻塞   socket是否阻塞是交给内核来判断然后给进程发送信号
- Epoll代理
  - 没有最大并发连接的限制，能打开的fd上限远大于1024（1G的内存能监听约10万个端口）
  - 采用回调的方式，效率提升。只有活跃可用的fd才会调用callback函数，也就是说 epoll 只管你“活跃”的连接，而跟连接总数无关，因此在实际的网络环境中，epoll的效率就会远远高于select和poll。只轮询发出了事件的流，哪个流发生了怎样的IO事件会通知处理线程，因此对这些流的操作都是有意义的，复杂度降低到了O(1)
  - 内存拷贝。使用mmap()文件映射内存来加速与内核空间的消息传递，减少复制开销
- CPU本来就是线性的不论什么都需要顺序处理并行只能是多核CPU
- io多路复用本来就是用来解决对多个I/O监听时,一个I/O阻塞影响其他I/O的问题,跟多线程没关系.
- 跟多线程相比较,线程切换需要切换到内核进行线程切换,需要消耗时间和资源.而I/O多路复用不需要切换线/进程,效率相对较高,特别是对高并发的应用nginx就是用I/O多路复用,故而性能极佳.但多线程编程逻辑和处理上比I/O多路复用简单.而I/O多路复用处理起来较为复杂







## 线程模型、工作模式(Reactor模式)

- Redis基于Reactor模式开发了网络事件处理器，这个处理器被称为文件事件处理器。它的组成结构为4部分：多个套接字、IO多路复用、文件事件分派器、事件处理器。因为文件事件分派器队列的消费是单线程的，所以Redis才叫单线程模型。
- 大体上可以说 **Redis 的工作模式**是：reactor 模式 + 一个队列，用一个 serverAccept 线程来处理建立请求的链接，并且**通过 IO 多路复用模型**，让内核来监听这些 socket，一旦某些 socket 的读写事件准备就绪后就对应的事件压入队列中，然后 worker 工作，由 文件事件分派器 从中获取事件交于对应的 处理器去执行，当某个事件执行完成后 文件事件分派器 才会从队列中获取下一个事件进行处理。

### 一、阻塞 IO 之 BIO

　　![img](https://img2020.cnblogs.com/blog/980882/202006/980882-20200618173021144-1673525710.png)

 

　　解释：fd 表示的是文件描述符，因为 linux 环境下一切结尾文件，它可以理解为 java 对象的引用，当 redis 服务启动的时候，会建立 socket 链接，如图中的 fd6，假设外部有两个客户端来连接 redis，则会与 redis 建立 socket 连接，分别为 fd7 和 fd8，当其中一个客户端发起读取操作，会将读取操作交给内核 kernel 来发起系统调用命令，系统调用命令 read 发起 read 请求将数据读取返后返回给客户端。后续基本上都是这个逻辑，就不会再重复解释了。

　　在这种 IO 模型的场景下，我们是给每一个客户端连接创建一个线程去处理它。不管这个客户端建立了连接有没有在做事，都要去维护这个连接，直到连接断开为止。创建过多的线程就会消耗过高的资源，以 Java BIO 为例

- BIO 是一个同步阻塞 IO
- 一个线程映射到一个轻量级进程（用户态中）然后去调用内核线程执行操作
- 对线程的调度，用户态和内核态切换以及上下文和现场存储等等都要消耗很多 CPU 和缓存资源
- 同步：客户端请求服务端后，服务端开始处理假设处理1秒钟，这一秒钟就算客户端再发送很多请求过来，服务端也忙不过来，它必须等到之前的请求处理完毕后再去处理下一个请求，当然我们可以使用伪异步 IO 来实现，也就是实现一个线程池，客户端请求过来后就丢给线程池处理，那么就能够继续处理下一个请求了
- 阻塞：inputStream.read(data) 会通过 recvfrom 去接收数据，如果内核数据还没有准备好就会一直处于阻塞状态

　　由此可见阻塞 I/O 难以支持高并发的场景，具体代码如下：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(9999);
        // 新建一个线程用于接收客户端连接。伪异步 IO
        new Thread(() -> {
            while (true) {
                System.out.println("开始阻塞, 等待客户端连接");
                try {
                    Socket socket = serverSocket.accept();
                    // 每一个新来的连接给其创建一个线程去处理
                    new Thread(() -> {
                        byte[] data = new byte[1024];
                        int len = 0;
                        System.out.println("客户端连接成功，阻塞等待客户端传入数据");
                        try {
                            InputStream inputStream = socket.getInputStream();
                            // 阻塞式获取数据直到客户端断开连接
                            while ((len = inputStream.read(data)) != -1) {
                                // 或取到数据
                                System.out.println(new String(data, 0, len));
                                // 处理数据
                            }
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }).start();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　见最上面的图，相应的问题就显而易见了：

　　文件描述符 fd7 读取的时候 fd8 一直在那等着，什么时候fd7 处理完了 fd8 才能开始执行。那如果 fd7 耗时很长，那后面即使耗时很短的指令也无法执行了，如何解？

### 二、非阻塞 IO 之 NIO

　　![img](https://img2020.cnblogs.com/blog/980882/202006/980882-20200618175208890-711162630.png)

 

　　在 BIO 中只能监控一个 socket 且只能阻塞的问题，如何解决？简单易想的方案就是：我们改写一下，不让整个链路阻塞，那怎么实现呢？

　　让所有客户端排好队，内核来逐个循环来取，有指令执行了就直接执行就可以了，没有就接着下一个扫描，这样看起来就解决了每个客户端需要单独一个线程来阻塞等待执行了。

　　那么问题由来了：

　　虽然 NIO 解决了 BIO 中的阻塞问题，但是如果有 1000 个客户端连接，那么 NIO 采用轮询的方式就有问题了。因为从 用户态 到 内核态 频繁的切换，也会很耗性能，如果第一次循环第一个连接没有相应的读取指令，过去之后读取指令马上就来了，那还需要等到剩余的 999 次循环以及用户态和内核态的切换才能再次来到当前连接执行。如何才能降低这种资源的浪费与性能的提升呢？

 

### 三、多路复用 IO 之 select

　　![img](https://img2020.cnblogs.com/blog/980882/202006/980882-20200618180930021-365021007.png)

 

 

 　　NIO 中我们知道频繁的 用户态 到 内核态 切换，导致性能和资源浪费严重，那我们是否可以不这么来回切换状态，那就有了多路复用之 select 了，即：

　　还是原来的 1000 个客户端连接，我们将用户态所有的连接打包统一交给内核，之后再由内核来统一循环，内核循环一圈后得到 要执行的 fds，再发送给 server，由 server 发起 用户态-->内核态的调用。这就是基本的多路复用策略。用一段伪代码来实现：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
// 假设现目前获得了很多 serverSocket.accept(); 后的客户端连接 List<Socket> sockets;
sockets = getSockets(); 
while (true) {
    // 阻塞，将所有的 sockets 传入内核让它帮我们检测是否有数据准备就绪
    // n 表示有多少个 socket 准备就绪了
    int n = select(sockets);
    for (int i = 0; i < sockets.length; i++) {
        // FD_ISSET 挨个检查 sockets 查看下内核数据是否准备就绪
        if (FD_ISSET(sockets[i]) {
            // 准备就绪了，挨个处理就绪的 socket
            doSomething();
        }
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　由此也能看出 select 的一些缺陷：

- 单进程能打开的最大文件描述符为 1024
- 监视 sockets 的时候需要将所有的 sockets 的文件描述符传入内核并且设置对应的进程，传入的东西太大了
- 内核同样也会一直在重复循环，然而其中也可能只有几个 fds 可用，导致内核一直很忙，这样的效率仍然不高。　　

### 四、多路复用 IO 之 poll

　　poll 跟 select 相似对其进行了部分优化，比如单进程能打开的文件描述符不受限制，底层是采用的链表实现。

### 五、多路复用 IO 终极 epoll

　　epoll 的出现相较于 select 晚了几年，它对 select，poll 进行了大幅度的优化。

　　![img](https://img2020.cnblogs.com/blog/980882/202006/980882-20200618182548493-886635332.png)    ![img](https://img2020.cnblogs.com/blog/980882/202006/980882-20200618182751230-174168666.jpg)

 

 　　就上图说明，相较于 select 可以发现主要是多了一个 eventpoll（rdlist），之前的需要监视的 socket 都需要绑定一个进程，现在都改为指向了 eventpoll

　　它是什么呢，我们看下 epoll 实现的伪代码：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
// 假设现目前获得了很多 serverSocket.accept(); 后的客户端连接 List<Socket> sockets;
sockets = getSockets(); 
// 这里就是在创建 eventpoll
int epfd = epoll_create();
// 将所有需要监视的 socket 都加入到 eventpoll 中
epoll_ctl(epfd, sockets);
while (true) {
    // 阻塞返回准备好了的 sockets
    int n = epoll_wait();
    // 这里就直接对收到数据的 socket 进行遍历不需要再遍历所有的 sockets
    // 是怎么做到的呢，下面继续分析
    for (遍历接收到数据的 socket) {
        
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　解释：

　　epoll_create：当某个进程调用 epoll_create 方法时，内核会创建一个 eventpoll 对象。eventpoll 对象也是文件系统中的一员，和 socket 一样，它也会有等待队列。

　　　　创建一个代表该epoll的eventpoll对象是必须的，因为内核要维护“就绪列表”等数据，“就绪列表”可以作为eventpoll的成员。

　　epoll_ctl：创建epoll对象后，可以用 epoll_ctl 添加或删除所要监听的socket，内核会将 eventpoll 添加到这些 socket 的等待队列中。

　　　　当socket收到数据后，中断程序会操作eventpoll对象，而不是直接操作进程。eventpoll对象相当于是socket和进程之间的中介，socket的数据接收并不直接影响进程，而是通过改变eventpoll的就绪列表来改变进程状态。

　　epoll_wait：阻塞返回准备好了的 sockets

### 六、就绪队列

 　　就绪队列就是下图的 rdlist 它是 eventpoll 的一个成员，指的是内核中有哪些数据已经准备就绪。这个是怎么做到的呢，当我们调用 epoll_ctl() 的时候会为每一个 socket 注册一个 回调函数，当某个 socket 准备好了就会 回调 然后加入 rdlist 中的，rdlist 的数据结构是一个双向链表。

　　![img](https://img2020.cnblogs.com/blog/980882/202006/980882-20200618232208188-25235765.jpg)

　　这下我们就可以直接从 rdlist 中通过一次系统调用直接获取数据了而不需要再去遍历所有的 sockets 了。

### 总结

　　epoll 提升了系统的并发，有限的资源提供更多的服务较于 select、poll 优势总结如下：

- 内核监视 sockets 的时候不再需要每次传入所有的 sockets 文件描述符，然后又全部断开（反复）的操作了，它只需通过一次 epoll_ctl 即可
- select、poll 模型下进程收到了 sockets 准备就绪的指令执行后，它不知道到底是哪个 socket 就绪了，需要去遍历所有的 sockets，而 epoll 维护了一个 rdlist 通过回调的方式将就绪的 socket 插入到 rdlist 链表中，我们可以直接获取 rdlist 即可，无需遍历其它的 socket 提升效率（注意需记住：多路复用 + 消息回调 的方式）

### 考虑下 epoll 的适用场景：

　　　　只要同一时间就绪列表不要太长都适合。比如 Nginx 它的处理都是及其快速的，如果它为每一个请求还创建一个线程，这个开销情况下它还如何支持高并发。

### netty

　　最后我们来看下 **netty**：

　　　netty 也是采用的多路复用模型我们讨论在 linux 情况下的 epoll 使用情况，netty 要如何使用才能更加高效呢？

　　　如果某一个 socket 请求时间相对较长比如 100MS 会大幅度降低模型对应的并发性，该如何处理呢，java 代码如下。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public class NIOServer {
    public static void main(String[] args) throws IOException {
        Selector serverSelector = Selector.open();
        Selector clientSelector = Selector.open();

        new Thread(() -> {
            try {
                // 对应IO编程中服务端启动
                ServerSocketChannel listenerChannel = ServerSocketChannel.open();
                listenerChannel.socket().bind(new InetSocketAddress(8000));
                listenerChannel.configureBlocking(false);
                listenerChannel.register(serverSelector, SelectionKey.OP_ACCEPT);

                while (true) {
                    // 一致处于阻塞直到有 socket 数据准备就绪
                    if (serverSelector.select() > 0) {
                        Set<SelectionKey> set = serverSelector.selectedKeys();
                        Iterator<SelectionKey> keyIterator = set.iterator();

                        while (keyIterator.hasNext()) {
                            SelectionKey key = keyIterator.next();
                            if (key.isAcceptable()) {
                                try {
                                    // (1) 每来一个新连接，不需要创建一个线程，而是直接注册到clientSelector
                                    SocketChannel clientChannel = ((ServerSocketChannel) key.channel()).accept();
                                    clientChannel.configureBlocking(false);
                                    clientChannel.register(clientSelector, SelectionKey.OP_READ);
                                } finally {
                                    keyIterator.remove();
                                }
                            }
                        }
                    }
                }
            } catch (IOException ignored) {
            }
        }).start();

        new Thread(() -> {
            try {
                while (true) {
                    // 阻塞等待读事件准备就绪
                    if (clientSelector.select() > 0) {
                        Set<SelectionKey> set = clientSelector.selectedKeys();
                        Iterator<SelectionKey> keyIterator = set.iterator();
                        while (keyIterator.hasNext()) {
                            SelectionKey key = keyIterator.next();

                            if (key.isReadable()) {
                                try {
                                    SocketChannel clientChannel = (SocketChannel) key.channel();
                                    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                                    // (3) 面向 Buffer
                                    clientChannel.read(byteBuffer);
                                    byteBuffer.flip();
       System.out.println(Charset.defaultCharset().newDecoder().decode(byteBuffer)
                                            .toString());
                                } finally {
                                    keyIterator.remove();
                                    key.interestOps(SelectionKey.OP_READ);
                                }
                            }
                        }
                    }
                }
            } catch (IOException ignored) {
            }
        }).start();
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

来分析下上面这段代码

- 用 serverSelector 来处理所有客户端的连接请求
- 用 clientSelector 来处理所有客户端连接成功后的读操作
- \1. 将 SelectionKey.OP_ACCEPT 这个操作注册到了 serverSelector 上面

> 　　*相当于上述将的将我们去创建 eventpoll 并且将当前 serverSocket 进行监视并且注册的是 ACCEPT 建立连接这个事件，将当前 Thread 移除工作队列挂入 eventpoll 的等待队列*

- \2. serverSelector.select() > 0 就是有 socket 数据准备就绪这里也就是有连接建立准备就绪

> 　　*相当于 epoll_wait 返回了可读数量（建立连接的数量），然后我们通过 clientSelector.selectedKeys(); 拿到了就绪队列里面的 socket*

- \3. 我们知道建立连接这个操作是很快的，建立成功后给 socket 注册到 clientSelector 上并且注册 READ 事件

> *就相当于我们又建立了一个 eventpoll 传入的就是需要监视读取事件的 socket（这其实就是之前讲的列子 sockets = getSockets()），然后 eventpoll 从工作队列中移除，需要监视的 sockets 全部指向 eventpoll ，eventpoll 的等待队列就是当前 new Thread 这个线程。*

- \4. 一旦某个 socket 读准备就绪，那么 eventpoll 的 rdlist 数据就会准备好，同时会唤醒当前等待的线程来处理数据

　　这里思考下由于建立连接的那个线程非常快速只有绑定读取事件给 clientSelector，所以时间可以忽略。但是在 clientSelector 中获取到数据后一般需要进行业务逻辑操作，可能耗时会比较长。

　　如果出现这种情况由于是单线程的，那么其它 socket 的读就绪事件可能就无法得到及时的响应，所以一般的做法是，不要在这个线程中处理过于耗时的操作，因为会极大的降低其并发性，对于那种可能相对较慢的操作我们就丢给线程池去处理。

```
if (key.isReadable()) {
    // 耗时就扔进线程池中
    executor.execute(task);
}
```

　　其实这也就是 netty 的处理方式，我们默认使用 netty 的时候，会创建 `serverBootstrap.group(boosGroup, workerGroup)` 其中默认情况 boosGroup 是一个线程在处理，workerGroup 是 n * cup 个线程在处理这样就能大幅度的提升并发性了。

　　另外有的小伙伴会说，netty 这样处理，最终又将客户端的操作去建立一个线程又丢给线程池了，这和我们使用阻塞式 I/O 每个请求建立一个连接一样扔进线程池有撒区别。

区别就在于，对于阻塞I/O每一个请求过来会创建一个连接（就算有线程池一样有很多线程创建维护的开销），而对于多路复用来说建立连接只是一个线程在处理，并且它会将对于的 read 事件注入到其它 selector 中，对于用户来说，肯定不会建立了连接那我就时时刻刻我不停的在发送请求了，多路复用的好处就体现出来了，连接你建立 OK linux 内核维护，我不去创建线程开销。当你真正有读的请求来的时候，我再给你取分配资源执行（如果耗时就走线程池），这里真正的请求过来的数量是远远低于建立成功的 sockets 数目的。那么对于的线程池线程开销也会远远低于每个请求建立一个线程的开销。

　　但是如果对于那种每次获取就绪队列的时候都是接近满负荷的话就不太适用于了多路复用的场景了。

## 事件机制

　　redis 客户端与 redis 服务端建立连接，发送命令，redis 服务器响应命令都是需要通过事件机制来做的，

- 首先 redis 服务器运行，监听套接字的 AE_READABLE 事件处于监听的状态下，此时 **连接应答处理器** 工作，
- 客户端 与 redis 服务器发起建立连接，监听套接字产生 AE_READABLE 事件，当 IO 多路复用程序监听到其准备就绪后，将该事件压入队列中，由 文件事件分派器 获取队列中的事件交于 **连接应答处理器**工作处理，应答客户端建立连接成功，同时将客户端 socket 的 AE_READABLE 事件压入队列由 文件事件分派器 获取队列中的事件交 **命令请求处理器关联**
- 客户端发送 set key value 请求，客户端 socket 的 AE_READABLE 事件，当 IO 多路复用程序监听到其准备就绪后，将该事件压入队列中，由 文件事件分派器 获取队列中的事件交于 **命令请求处理器关联** 处理
- **命令请求处理器关联** 处理完成后，需要响应客户端操作完成，此时将产生 socket 的 AE_WRITEABLE 事件压入队列，由 文件事件分派器 获取队列中的事件交于 **命令恢复处理器** 处理，返回操作结果，完成后将解除 AE_WRITEABLE 事件 与 **命令恢复处理器** 的关联





## rehash

- 在Redis中，键值对（Key-Value Pair）存储方式是由字典（Dict）保存的，而字典底层是通过哈希表来实现的。通过哈希表中的节点保存字典中的键值对。类似Java中的HashMap，将Key通过哈希函数映射到哈希表节点位置。

- rehash的步骤

  - 为字典的ht[1]哈希表分配空间

    - > 该哈希表已有节点的数量
      > unsigned long used;
      >
      > 
      >
      > 扩展
      > 	ht[1]的大小为>=ht[0].used*2>=2^n
      > 收缩
      > 	ht[1]的大小为>=ht[0].used>=2^n

    - 将保存在ht[0]中的所有键值对rehash到ht[1]中，rehash指重新计算键的哈希值和索引值，然后将键值对放置到ht[1]哈希表的指定位置上

    - 释放ht[0]，将ht[1]设置为ht[0],新建空白的哈希表ht[1]，以备下次rehash使用

- 渐进式 rehash 执行期间的哈希表操作

  - 因为在进行渐进式 rehash 的过程中， 字典会同时使用 ht[0] 和 ht[1] 两个哈希表， 所以在渐进式 rehash 进行期间， 字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行： 比如说， 要在字典里面查找一个键的话， 程序会先在 ht[0] 里面进行查找， 如果没找到的话， 就会继续到 ht[1] 里面进行查找， 诸如此类。
  - 另外， 在渐进式 rehash 执行期间， 新添加到字典的键值对一律会被保存到 ht[1] 里面， 而 ht[0] 则不再进行任何添加操作： 这一措施保证了 ht[0] 包含的键值对数量会只减不增， 并随着 rehash 操作的执行而最终变成空表。
  - 渐进式rehash避免了redis阻塞，可以说非常完美，但是由于在rehash时，需要分配一个新的hash表，在rehash期间，同时有两个hash表在使用，会使得redis内存使用量瞬间突增，在Redis 满容状态下由于Rehash会导致大量Key驱逐。
  - 除了导致满容驱逐淘汰，Redis Rehash还会引起其他一些问题：
    - 在tablesize级别与现有Keys数量不在同一个区间内，主从切换后，由于Redis全量同步，从库tablesize降为与现有Key匹配值，导致内存倾斜；
    - Redis Cluster下的某个分片由于Key数量相对较多提前Resize，导致集群分片内存不均

- Redis使用Scan清理Key由于Rehash导致清理数据不彻底

  - 为了高效地匹配出数据库中所有符合给定模式的Key，Redis提供了Scan命令。

  - Redis官方定义Scan特点如下：

    - 整个遍历从开始到结束期间， 一直存在于Redis数据集内的且符合匹配模式的所有Key都会被返回；
    - 如果发生了rehash，同一个元素可能会被返回多次，遍历过程中新增或者删除的Key可能会被返回，也可能不会。

  - 那么在Dict非稳定状态，即发生Rehash的情况下，Scan要如何保证原有的Key都能遍历出来，又尽少可能重复扫描呢？Redis Scan通过Hash桶掩码的高位顺序访问来解决。

    - Scan采用高位序访问的原因，就是为了实现Redis Dict在Rehash时尽可能少重复扫描返回Key。

      - > 000-100-010-110-001-101-011-111

    - 举个例子，如果Dict的tablesize从8扩展到了16，梳理一下Scan扫描方式:

      - > Dict(8) 从Cursor 0开始扫描；
        >
        > 准备扫描Cursor 6时发生Resize，扩展为之前的2倍，并完成Rehash；
        >
        > 客户端这时开始从Dict(16)的Cursor 6继续迭代；
        >
        > 这时按照 6→14→1→9→5→13→3→11→7→15 Scan完成。





## 跳表

- 它不要求上下相邻两层链表之间的节点个数有严格的对应关系，而是为每个节点随机出一个层数(level)

  - > 如果一个节点有第i层(i>=1)指针，么它有第(i+1)层指针的概率为p
    > 节点最大的层数不允许超过一个最大值，记为MaxLevel
    >
    > 
    >
    > level := 1    // random()返回一个[0...1)的随机数    
    > while random() < p and level < MaxLevel 
    > do level := level + 1 return level 
    >
    > 
    >
    > 在Redis的skiplist实现中，这两个参数的取值为：
    > 	p = 1/4
    > MaxLevel = 32

- skiplist与平衡树、哈希表的比较

  - skiplist和各种平衡树（如AVL、红黑树等）的元素是有序排列的，而哈希表不是有序的，哈希表上只能做单个key的查找
  - 在做范围查找的时候，平衡树比skiplist操作要复杂
    - 平衡树上，我们找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点
    - skiplist上进行范围查找就非常简单，只需要在找到小值之后，对第1层链表进行若干步的遍历就可以实现。
  - 平衡树的插入和删除操作可能引发子树的调整，逻辑复杂
  - 从内存占用上来说，skiplist比平衡树更灵活一些
    - 平衡树每个节点包含2个指针（分别指向左右子树）
    - Redis里的实现一样，取p=1/4，那么平均每个节点包含1.33个指针，比平衡树更有优势
  - 查找单个key，skiplist和平衡树的时间复杂度都为O(log n)。哈希表在保持较低的哈希值冲突概率的前提下，查找时间复杂度接近O(1)，性能更高一些。所以我们平常使用的各种Map或dictionary结构，大都是基于哈希表实现的
  - 从算法实现难度上来比较，skiplist比平衡树要简单得多

- Redis中的skiplist实现

  - sorted set底层不仅仅使用了skiplist，还使用了ziplist和dict

  - skiplist的数据结构定义

    - zskiplistNode定义了skiplist的节点结构

      - > backward字段是指向链表前一个节点的指针（前向指针）。节点只有1个前向指针，所以只有第1层链表是一个双向链表。
        >
        > level[]存放指向各层链表后一个节点的指针（后向指针）
        >
        > 每层对应1个后向指针，用forward字段表示。另外，每个后向指针还对应了一个span值，它表示当前的指针跨越了多少个节点。span用于计算元素排名(rank)

    - zskiplist定义了真正的skiplist结构

      - > 头指针header和尾指针tail
        > 链表长度length
        > level表示skiplist的总层数，即所有节点层数的最大值

  - Redis中skiplist实现的特殊性

    - 当数据较少时，sorted set是  由一个ziplist来实现的
    - 当数据多的时候，sorted set是由一个dict + 一个skiplist来实现的
      - dict用来查询数据到分数的对应关系，而skiplist用来根据分数查询数据（可能是范围查找）
    - 在如下两个条件之一满足的时候，ziplist会转成zset
      - 元素个数，即(数据, score)对的数目超过128的时候，也就是ziplist数据项超过256的时候
      - 当sorted set中插入的任意一个数据的长度超过了64的时候

- Redis中的skiplist跟前面介绍的经典的skiplist相比，有如下不同

  - 分数(score)允许重复，即skiplist的key允许重复。这在最开始介绍的经典skiplist中是不允许的
  - 在比较时，不仅比较分数（相当于skiplist的key），还比较数据本身。在Redis的skiplist实现中，数据本身的内容唯一标识这份数据，而不是由key来唯一标识
  - 第1层链表不是一个单向链表，而是一个双向链表。这是为了方便以倒序方式获取一个范围内的元素










