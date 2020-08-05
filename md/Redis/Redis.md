## 特性

- Redis 内置了复制（Replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），不同级别的磁盘持久化（Persistence），并通过 Redis哨兵（Sentinel）和自动分区（Cluster）提供高可用性（High Availability）的键值对(key-value)存储数据库

- > 首先，采用了多路复用io阻塞机制
  > 然后，数据结构简单，操作节省时间
  > 最后，运行在内存中，自然速度快





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





## Redis主从复制

- 复制的过程

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

  





## Redis cluster

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

- Master调用BGREWRITEAOF重写AOF文件，AOF在重写的时候会占大量的CPU和内存资源，导致服务load过高，出现短暂服务暂停现象
  - 将no-appendfsync-on-rewrite的配置设为yes可以缓解这个问题，设置为yes表示rewrite期间对新写操作不fsync，暂时存在内存中，等rewrite完成后再写入。
  - 这就相当于将appendfsync设置为no，这说明并没有执行磁盘操作，只是写入了缓冲区，因此这样并不会造成阻塞（因为没有竞争磁盘），但是如果这个时候redis挂掉，就会丢失数据。

- Redis主从复制的性能问题
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



























