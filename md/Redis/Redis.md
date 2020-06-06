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

  - 







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
  - redis3.0上加入了cluster模式，实现的redis的分布式存储，也就是说每台redis节点上存储不同的内容
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



































