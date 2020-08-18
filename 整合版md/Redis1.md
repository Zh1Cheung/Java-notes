

# Redis分布式锁

分布式锁在很多场景中是非常有用的原语， 不同的进程必须以独占资源的方式实现资源共享就是一个典型的例子。

**Redlock**

## 安全和活性失效保障

按照我们的思路和设计方案，算法只需具备3个特性就可以实现一个最低保障的分布式锁。

1. 安全属性（Safety property）: 独享（相互排斥）。在任意一个时刻，只有一个客户端持有锁。
2. 活性A(Liveness property A): 无死锁。即便持有锁的客户端崩溃（crashed)或者网络被分裂（gets partitioned)，锁仍然可以被获取。
3. 活性B(Liveness property B): 容错。 只要大部分Redis节点都活着，客户端就可以获取和释放锁.



## 为什么基于故障转移的实现还不够

为了更好的理解我们想要改进的方面，我们先分析一下当前大多数基于Redis的分布式锁现状和实现方法.

实现Redis分布式锁的最简单的方法就是在Redis中创建一个key，这个key有一个失效时间（TTL)，以保证锁最终会被自动释放掉（这个对应特性2）。当客户端释放资源(解锁）的时候，会删除掉这个key。

从表面上看，似乎效果还不错，但是这里有一个问题：这个架构中存在一个严重的单点失败问题。如果Redis挂了怎么办？你可能会说，可以通过增加一个slave节点解决这个问题。但这通常是行不通的。这样做，我们不能实现资源的独享,因为Redis的主从同步通常是异步的。

在这种场景（主从结构）中存在明显的竞态:

1. 客户端A从master获取到锁
2. 在master将锁同步到slave之前，master宕掉了。
3. slave节点被晋级为master节点
4. 客户端B取得了同一个资源被客户端A已经获取到的另外一个锁。**安全失效！**

有时候程序就是这么巧，比如说正好一个节点挂掉的时候，多个客户端同时取到了锁。如果你可以接受这种小概率错误，那用这个基于复制的方案就完全没有问题。否则的话，我们建议你实现下面描述的解决方案。

## 单Redis实例实现分布式锁的正确方法

在尝试克服上述单实例设置的限制之前，让我们先讨论一下在这种简单情况下实现分布式锁的正确做法，实际上这是一种可行的方案，尽管存在竞态，结果仍然是可接受的，另外，这里讨论的单实例加锁方法也是分布式加锁算法的基础。

获取锁使用命令:

```
    SET resource_name my_random_value NX PX 30000
```

这个命令仅在不存在key的时候才能被执行成功（NX选项），并且这个key有一个30秒的自动失效时间（PX属性）。这个key的值是“my_random_value”(一个随机值），这个值在所有的客户端必须是唯一的，所有同一key的获取者（竞争者）这个值都不能一样。

value的值必须是随机数主要是为了更安全的释放锁，释放锁的时候使用脚本告诉Redis:只有key存在并且存储的值和我指定的值一样才能告诉我删除成功。可以通过以下Lua脚本实现：

```
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

使用这种方式释放锁可以避免删除别的客户端获取成功的锁。举个例子：客户端A取得资源锁，但是紧接着被一个其他操作阻塞了，当客户端A运行完毕其他操作后要释放锁时，原来的锁早已超时并且被Redis自动释放，并且在这期间资源锁又被客户端B再次获取到。如果仅使用DEL命令将key删除，那么这种情况就会把客户端B的锁给删除掉。使用Lua脚本就不会存在这种情况，因为脚本仅会删除value等于客户端A的value的key（value相当于客户端的一个签名）。

这个随机字符串应该怎么设置？我认为它应该是从/dev/urandom产生的一个20字节随机数，但是我想你可以找到比这种方法代价更小的方法，只要这个数在你的任务中是唯一的就行。例如一种安全可行的方法是使用/dev/urandom作为RC4的种子和源产生一个伪随机流;一种更简单的方法是把以毫秒为单位的unix时间和客户端ID拼接起来，理论上不是完全安全，但是在多数情况下可以满足需求.

key的失效时间，被称作“锁定有效期”。它不仅是key自动失效时间，而且还是一个客户端持有锁多长时间后可以被另外一个客户端重新获得。

截至到目前，我们已经有较好的方法获取锁和释放锁。基于Redis单实例，假设这个单实例总是可用，这种方法已经足够安全。现在让我们扩展一下，假设Redis没有总是可用的保障。

## Redlock算法

在Redis的分布式环境中，我们假设有N个Redis master。这些节点完全互相独立，不存在主从复制或者其他集群协调机制。之前我们已经描述了在Redis单实例下怎么安全地获取和释放锁。我们确保将在每（N)个实例上使用此方法获取和释放锁。在这个样例中，我们假设有5个Redis master节点，这是一个比较合理的设置，所以我们需要在5台机器上面或者5台虚拟机上面运行这些实例，这样保证他们不会同时都宕掉。

为了取到锁，客户端应该执行以下操作:

1. 获取当前Unix时间，以毫秒为单位。
2. 依次尝试从N个实例，使用相同的key和随机值获取锁。在步骤2，当向Redis设置锁时,客户端应该设置一个网络连接和响应超时时间，这个超时时间应该小于锁的失效时间。例如你的锁自动失效时间为10秒，则超时时间应该在5-50毫秒之间。这样可以避免服务器端Redis已经挂掉的情况下，客户端还在死死地等待响应结果。如果服务器端没有在规定时间内响应，客户端应该尽快尝试另外一个Redis实例。
3. 客户端使用当前时间减去开始获取锁时间（步骤1记录的时间）就得到获取锁使用的时间。当且仅当从大多数（这里是3个节点）的Redis节点都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功。
4. 如果取到了锁，key的真正有效时间等于有效时间减去获取锁所使用的时间（步骤3计算的结果）。
5. 如果因为某些原因，获取锁失败（*没有*在至少N/2+1个Redis实例取到锁或者取锁时间已经超过了有效时间），客户端应该在所有的Redis实例上进行解锁（即便某些Redis实例根本就没有加锁成功）。

## 这个算法是异步的么?

算法基于这样一个假设：虽然多个进程之间没有时钟同步，但每个进程都以相同的时钟频率前进，时间差相对于失效时间来说几乎可以忽略不计。这种假设和我们的真实世界非常接近：每个计算机都有一个本地时钟，我们可以容忍多个计算机之间有较小的时钟漂移。

从这点来说，我们必须再次强调我们的互相排斥规则：只有在锁的有效时间（在步骤3计算的结果）范围内客户端能够做完它的工作，锁的安全性才能得到保证（锁的实际有效时间通常要比设置的短，因为计算机之间有时钟漂移的现象）。.

想要了解更多关于需要*时钟漂移*间隙的相似系统, 这里有一个非常有趣的参考: [Leases: an efficient fault-tolerant mechanism for distributed file cache consistency](http://dl.acm.org/citation.cfm?id=74870).

## 失败时重试

当客户端无法取到锁时，应该在一个*随机*延迟后重试,防止多个客户端在*同时*抢夺同一资源的锁（这样会导致脑裂，没有人会取到锁）。同样，客户端取得大部分Redis实例锁所花费的时间越短，脑裂出现的概率就会越低（必要的重试），所以，理想情况一下，客户端应该同时（并发地）向所有Redis发送SET命令。

需要强调，当客户端从大多数Redis实例获取锁失败时，应该尽快地释放（部分）已经成功取到的锁，这样其他的客户端就不必非得等到锁过完“有效时间”才能取到（然而，如果已经存在网络分裂，客户端已经无法和Redis实例通信，此时就只能等待key的自动释放了，等于被惩罚了）。

## 释放锁

释放锁比较简单，向所有的Redis实例发送释放锁命令即可，不用关心之前有没有从Redis实例成功获取到锁.

## 安全争议

这个算法安全么？我们可以从不同的场景讨论一下。

让我们假设客户端从大多数Redis实例取到了锁。所有的实例都包含同样的key，并且key的有效时间也一样。然而，key肯定是在不同的时间被设置上的，所以key的失效时间也不是精确的相同。我们假设第一个设置的key时间是T1(开始向第一个server发送命令前时间），最后一个设置的key时间是T2(得到最后一台server的答复后的时间），我们可以确认，第一个server的key至少会存活 `MIN_VALIDITY=TTL-(T2-T1)-CLOCK_DRIFT`。所有其他的key的存活时间，都会比这个key时间晚，所以可以肯定，所有key的失效时间至少是MIN_VALIDITY。

当大部分实例的key被设置后，其他的客户端将不能再取到锁，因为至少N/2+1个实例已经存在key。所以，如果一个锁被（客户端）获取后，客户端自己也不能再次申请到锁(违反互相排斥属性）。

然而我们也想确保，当多个客户端同时抢夺一个锁时不能两个都成功。

如果客户端在获取到大多数redis实例锁，使用的时间接近或者已经大于失效时间，客户端将认为锁是失效的锁，并且将释放掉已经获取到的锁，所以我们只需要在有效时间范围内获取到大部分锁这种情况。在上面已经讨论过有争议的地方，在`MIN_VALIDITY`时间内，将没有客户端再次取得锁。所以只有一种情况，多个客户端会在相同时间取得N/2+1实例的锁，那就是取得锁的时间大于失效时间（TTL time)，这样取到的锁也是无效的.

如果你能提供关于现有的类似算法的一个正式证明（指出正确性），或者是发现这个算法的bug？ 我们将非常感激.

## 活性争议

系统的活性安全基于三个主要特性:

1. 锁的自动释放（因为key失效了）：最终锁可以再次被使用.
2. 客户端通常会将没有获取到的锁删除，或者锁被取到后，使用完后，客户端会主动（提前）释放锁，而不是等到锁失效另外的客户端才能取到锁。.
3. 当客户端重试获取锁时，需要等待一段时间，这个时间必须大于从大多数Redis实例成功获取锁使用的时间，以最大限度地避免脑裂。.

然而，当网络出现问题时系统在`失效时间(TTL)`内就无法服务，这种情况下我们的程序就会为此付出代价。如果网络持续的有问题，可能就会出现死循环了。 这种情况发生在当客户端刚取到一个锁还没有来得及释放锁就被网络隔离.

如果网络一直没有恢复，这个算法会导致系统不可用.

## 性能，崩溃恢复和Redis同步

很多用户把Redis当做分布式锁服务器，使用获取锁和释放锁的响应时间，每秒钟可用执行多少次 acquire / release 操作作为性能指标。为了达到这一要求，增加Redis实例当然可用降低响应延迟（没有钱买硬件的”穷人”,也可以在网络方面做优化，使用非阻塞模型，一次发送所有的命令，然后异步的读取响应结果，假设客户端和redis服务器之间的RTT都差不多。

然而，如果我们想使用可以从备份中恢复的redis模式，有另外一种持久化情况你需要考虑，.

我们考虑这样一种场景，假设我们的redis没用使用备份。一个客户端获取到了3个实例的锁。此时，其中一个已经被客户端取到锁的redis实例被重启，在这个时间点，就可能出现3个节点没有设置锁，此时如果有另外一个客户端来设置锁，锁就可能被再次获取到，这样锁的互相排斥的特性就被破坏掉了。

如果我们启用了AOF持久化，情况会好很多。我们可用使用SHUTDOWN命令关闭然后再次重启。因为Redis到期是语义上实现的，所以当服务器关闭时，实际上还是经过了时间，所有（保持锁）需要的条件都没有受到影响. 没有受到影响的前提是redis优雅的关闭。停电了怎么办？如果redis是每秒执行一次fsync，那么很有可能在redis重启之后，key已经丢弃。理论上，如果我们想在Redis重启地任何情况下都保证锁的安全，我们必须开启fsync=always的配置。这反过来将完全破坏与传统上用于以安全的方式实现分布式锁的同一级别的CP系统的性能.

然而情况总比一开始想象的好一些。当一个redis节点重启后，只要它不参与到任意**当前活动**的锁，没有被当做“当前存活”节点被客户端重新获取到,算法的安全性仍然是有保障的。

为了达到这种效果，我们只需要将新的redis实例，在一个`TTL`时间内，对客户端不可用即可，在这个时间内，所有客户端锁将被失效或者自动释放.

使用*延迟重启*可以在不采用持久化策略的情况下达到同样的安全，然而这样做有时会让系统转化为彻底不可用。比如大部分的redis实例都崩溃了，系统在`TTL`时间内任何锁都将无法加锁成功。

## 使算法更加可靠：锁的扩展

如果你的工作可以拆分为许多小步骤，可以将有效时间设置的小一些，使用锁的一些扩展机制。在工作进行的过程中，当发现锁剩下的有效时间很短时，可以再次向redis的所有实例发送一个Lua脚本，让key的有效时间延长一点（前提还是key存在并且value是之前设置的value)。

客户端扩展TTL时必须像首次取得锁一样在大多数实例上扩展成功才算再次取到锁，并且是在有效时间内再次取到锁（算法和获取锁是非常相似的）。

这样做从技术上将并不会改变算法的正确性，所以扩展锁的过程中仍然需要达到获取到N/2+1个实例这个要求，否则活性特性之一就会失效。



# Redis协议

- 我们都知道调用Redis的命令是这样的：`set username afei`，`hmset Person:1 username afei password 123456`，那么Redis真正接收的请求是什么样的呢？即Redis定义的协议是怎么样的，让我们一探究竟。

  Redis协议官方有定义，[参考地址](https://link.jianshu.com?t=http://redis.cn/topics/protocol.html)，定义如下：

  ```xml
  *<number of arguments> CR LF
  $<number of bytes of argument 1> CR LF
  <argument data> CR LF
  ...
  $<number of bytes of argument N> CR LF
  <argument data> CR LF
  ```

  > **重要申明**：CR LF事实上就是\r\n，即window平台的换行。我们在构造redis协议文件时只需要按照正常的方式编写完后，在linux服务器上通过unix2dos转码即可；

- Redis协议文件样例

  - 通过Redis对协议的定义，我们可以自己写出redis协议文件, 如下所示：

    **String类型**之`set username afei`对应的redis协议文本：

    ```bash
    *3
    $3
    set
    $8
    username
    $4
    afei
    ```

    ### Redis协议文件解读:

    `*3` 表示这个命令有3个参数；
     `$3` 表示第一个参数的长度是3
     `set` 表示定义长度为$3的参数
     `$8` 表示第二个参数的长度是8
     `usename` 表示定义长度为$8的参数
     `$4` 表示第三个参数的长度是4
     `afei` 表示定义长度为$4的参数

# AOF文件全量重写源码阅读

- AOF文件什么时候完全重写：
  - **1** AOF文件超过64M且增长一定比例(最后一次AOF文件重写后增长了aof_rewrite_perc，默认是100%，在redis.h中有定义：REDIS_AOF_REWRITE_PERC，可以通过config get/set auto-aof-rewrite-percentage热修改)
  - **2** 有AOF重写的调度任务（例如执行BGREWRITEAOF命令）

### rewriteAppendOnlyFileBackground(void)

​	这个方法的注释说明了后台AOF重写是如何工作的--主要是全量重新AOF文件业务逻辑

- ```cpp
  /* This is how rewriting of the append only file in background works:
   *
   * 1) The user calls BGREWRITEAOF
   * 2) Redis calls this function, that forks():
   *    2a) the child rewrite the append only file in a temp file.
   *    2b) the parent accumulates differences in server.aof_rewrite_buf.
   * 3) When the child finished '2a' exists.
   * 4) The parent will trap the exit code, if it's OK, will append the
   *    data accumulated into server.aof_rewrite_buf into the temp file, and
   *    finally will rename(2) the temp file in the actual file name.
   *    The the new file is reopened as the new append only file. Profit!
   */
  int rewriteAppendOnlyFileBackground(void) {
      pid_t childpid;
      long long start;
  
      // 如果已经有AOF重写任务，那么退出；
      if (server.aof_child_pid != -1) return REDIS_ERR;
      start = ustime();
  
      // 调用fork()，如果返回值childpid==0那么表示当前处于fork的子进程中；
      if ((childpid = fork()) == 0) {
          char tmpfile[256];
  
          /* Child */
          closeListeningSockets(0);
          redisSetProcTitle("redis-aof-rewrite");
          // 如果getpid()的结果为1976，即当前进程id为1976，那么tmpfile=‘temp-rewriteaof-bg-1976.aof’，即AOF文件重写临时文件名
          snprintf(tmpfile,256,"temp-rewriteaof-bg-%d.aof", (int) getpid());
          // 调用rewriteAppendOnlyFile重写aof文件到tmpfile中[后面会解读]；
          if (rewriteAppendOnlyFile(tmpfile) == REDIS_OK) {
              size_t private_dirty = zmalloc_get_private_dirty();
  
              if (private_dirty) {
                  redisLog(REDIS_NOTICE,
                      "AOF rewrite: %zu MB of memory used by copy-on-write",
                      private_dirty/(1024*1024));
              }
              exitFromChild(0);
          } else {
              exitFromChild(1);
          }
      } else {
          // 调用fork()，如果返回值childpid!=0那么表示当前处于父进程中；
          /* Parent */
          server.stat_fork_time = ustime()-start;
          server.stat_fork_rate = (double) zmalloc_used_memory() * 1000000 / server.stat_fork_time / (1024*1024*1024); /* GB per second. */
          latencyAddSampleIfNeeded("fork",server.stat_fork_time/1000);
          if (childpid == -1) {
              redisLog(REDIS_WARNING,
                  "Can't rewrite append only file in background: fork: %s",
                  strerror(errno));
              return REDIS_ERR;
          }
          redisLog(REDIS_NOTICE,
              "Background append only file rewriting started by pid %d",childpid);
          server.aof_rewrite_scheduled = 0;
          server.aof_rewrite_time_start = time(NULL);
          server.aof_child_pid = childpid;
          updateDictResizePolicy();
          /* We set appendseldb to -1 in order to force the next call to the
           * feedAppendOnlyFile() to issue a SELECT command, so the differences
           * accumulated by the parent into server.aof_rewrite_buf will start
           * with a SELECT statement and it will be safe to merge. */
          server.aof_selected_db = -1;
          replicationScriptCacheFlush();
          return REDIS_OK;
      }
      return REDIS_OK; /* unreached */
  }
  ```

### rewriteAppendOnlyFile

调用rewriteAppendOnlyFile重写AOF文件（增量重写AOF文件，重新生成AOF文件）

```java
/* Write a sequence of commands able to fully rebuild the dataset into
 * "filename". Used both by REWRITEAOF and BGREWRITEAOF.
 *
 * In order to minimize the number of commands needed in the rewritten
 * log Redis uses variadic commands when possible, such as RPUSH, SADD
 * and ZADD. However at max REDIS_AOF_REWRITE_ITEMS_PER_CMD items per time
 * are inserted using a single command. */
int rewriteAppendOnlyFile(char *filename) {
    dictIterator *di = NULL;
    dictEntry *de;
    rio aof;
    FILE *fp;
    char tmpfile[256];
    int j;
    long long now = mstime();

    /* Note that we have to use a different temp name here compared to the
     * one used by rewriteAppendOnlyFileBackground() function. */
    snprintf(tmpfile,256,"temp-rewriteaof-%d.aof", (int) getpid());
    fp = fopen(tmpfile,"w");
    if (!fp) {
        redisLog(REDIS_WARNING, "Opening the temp file for AOF rewrite in rewriteAppendOnlyFile(): %s", strerror(errno));
        return REDIS_ERR;
    }

    rioInitWithFile(&aof,fp);

    // 如果开启了AOF重写增量模式--即配置appendonly yes然后执行set,lpush等引起内存数据变化的命令；
    if (server.aof_rewrite_incremental_fsync)
        rioSetAutoSync(&aof,REDIS_AOF_AUTOSYNC_BYTES);
    // 遍历redis中所有db重新生成AOF文件
    for (j = 0; j < server.dbnum; j++) {
        //写入AOF文件中的第一行内容就是selectcmd，即*2\r\n$6\r\nSELECT\r\n，这个内容是根据redis协议定义的：
        // *2
        // $6
        // SELECT
        // *2 表示这条命名有两个参数(SELECT dbnum)
        // $6 表示接下来参数的长度是6
        // SELECT表示长度是6的参数，后面还会写入dbnum；
        char selectcmd[] = "*2\r\n$6\r\nSELECT\r\n";
        redisDb *db = server.db+j;
        // redis中每个db里保存key的数据结构是一个dict；
        dict *d = db->dict;
        // 如果遍历当前db的dict(保存所有key的数据结构)是空，那么遍历下一次db
        if (dictSize(d) == 0) continue;
        // 如果遍历当前db的dict有值，那么迭代这个dict；
        di = dictGetSafeIterator(d);
        if (!di) {
            fclose(fp);
            return REDIS_ERR;
        }

        // 把selectcmd这个char[]以及当前遍历的db编号即j写入aof文件中（接着写在上面的SELECT之后）；
        /* SELECT the new DB */
        if (rioWrite(&aof,selectcmd,sizeof(selectcmd)-1) == 0) goto werr;
        if (rioWriteBulkLongLong(&aof,j) == 0) goto werr;

        // 迭代dictIterator *di，迭代过程中得到的de就是一个dictEntry  :
        /* Iterate this DB writing every entry */
        while((de = dictNext(di)) != NULL) {
            sds keystr;
            robj key, *o;
            long long expiretime;

            // 根据dictEntry得到key和value，value是一个redisObject类型指针；
            keystr = dictGetKey(de);
            o = dictGetVal(de);
            initStaticStringObject(key,keystr);

            // 从存放所有设置了过期时间的dict中查询这个key是否设置了过期时间；
            expiretime = getExpire(db,&key);

            // 如果已经过期，那么跳过，不保存到aof文件中
            /* If this key is already expired skip it */
            if (expiretime != -1 && expiretime < now) continue;

            // 接下来根据值的类型不同处理方式也不同；
            /* Save the key and associated value */

            // 如果当前key的值的类型是REDIS_STRING，即set命令生成的，假设当前遍历的是set username afei，那么写入aof文件大概内容如下（\r\n就是window格式的换行符）：
            // *3
            // $3
            // SET
            // $8
            // username
            // $4
            // afei
            // 其他的list，set，zset，hash处理类似；
            if (o->type == REDIS_STRING) {
                /* Emit a SET command */
                char cmd[]="*3\r\n$3\r\nSET\r\n";
                if (rioWrite(&aof,cmd,sizeof(cmd)-1) == 0) goto werr;
                /* Key and value */
                if (rioWriteBulkObject(&aof,&key) == 0) goto werr;
                if (rioWriteBulkObject(&aof,o) == 0) goto werr;
            } else if (o->type == REDIS_LIST) {
                if (rewriteListObject(&aof,&key,o) == 0) goto werr;
            } else if (o->type == REDIS_SET) {
                if (rewriteSetObject(&aof,&key,o) == 0) goto werr;
            } else if (o->type == REDIS_ZSET) {
                if (rewriteSortedSetObject(&aof,&key,o) == 0) goto werr;
            } else if (o->type == REDIS_HASH) {
                if (rewriteHashObject(&aof,&key,o) == 0) goto werr;
            } else {
                redisPanic("Unknown object type");
            }
            /* Save the expire time */
            // 如果key有过期属性，那么还需要单独保存过期属性到aof文件中，格式大概如下：
            // *3
            // $9
            // PEXPIREAT
            // $8
            // username
            // $13
            // 1506405235055
            if (expiretime != -1) {
                char cmd[]="*3\r\n$9\r\nPEXPIREAT\r\n";
                if (rioWrite(&aof,cmd,sizeof(cmd)-1) == 0) goto werr;
                if (rioWriteBulkObject(&aof,&key) == 0) goto werr;
                if (rioWriteBulkLongLong(&aof,expiretime) == 0) goto werr;
            }
        }
        dictReleaseIterator(di);
        di = NULL;
    }

    /* Make sure data will not remain on the OS's output buffers */
    if (fflush(fp) == EOF) goto werr;
    if (fsync(fileno(fp)) == -1) goto werr;
    if (fclose(fp) == EOF) goto werr;

    // 最后重命名这个AOF文件；用rename能保证重命名的原子性；
    /* Use RENAME to make sure the DB file is changed atomically only
     * if the generate DB file is ok. */
    if (rename(tmpfile,filename) == -1) {
        redisLog(REDIS_WARNING,"Error moving temp append only file on the final destination: %s", strerror(errno));
        unlink(tmpfile);
        return REDIS_ERR;
    }
    redisLog(REDIS_NOTICE,"SYNC append only file rewrite performed");
    return REDIS_OK;

werr:
    redisLog(REDIS_WARNING,"Write error writing append only file on disk: %s", strerror(errno));
    fclose(fp);
    unlink(tmpfile);
    if (di) dictReleaseIterator(di);
    return REDIS_ERR;
}
```





# AOF文件增量追写源码阅读

这个方法主要是实时追写AOF文件的业务逻辑，比如配置了appendonly yes的场景下，执行set ，hset，lpush等（导致内存数据变化）命令，就会调用这个方法实时刷新AOF文件

```cpp
/* Write the append only file buffer on disk.
 *
 * Since we are required to write the AOF before replying to the client,
 * and the only way the client socket can get a write is entering when the
 * the event loop, we accumulate all the AOF writes in a memory
 * buffer and write it on disk using this function just before entering
 * the event loop again.
 *
 * About the 'force' argument:
 *
 * When the fsync policy is set to 'everysec' we may delay the flush if there
 * is still an fsync() going on in the background thread, since for instance
 * on Linux write(2) will be blocked by the background fsync anyway.
 * When this happens we remember that there is some aof buffer to be
 * flushed ASAP, and will try to do that in the serverCron() function.
 *
 * However if force is set to 1 we'll write regardless of the background
 * fsync. */
#define AOF_WRITE_LOG_ERROR_RATE 30 /* Seconds between errors logging. */

// force：是否强制刷新，只有从appendonly yes切换到appendonly（通过config set）时force才为0，其他情况都是0；
void flushAppendOnlyFile(int force) {
    ssize_t nwritten;
    int sync_in_progress = 0;
    mstime_t latency;
    // 如果AOF buffer中没有任何数据（非读的redis命令操作都会记录到aof_buf中），那么不需要flush AOF文件；
    if (sdslen(server.aof_buf) == 0) return;

    if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
        // 判断是否有正在进行中的AOF fsync任务
        sync_in_progress = bioPendingJobsOfType(REDIS_BIO_AOF_FSYNC) != 0;

    // 如果刷新策略是EVERYSEC，默认策略，即每秒刷新，且force为0
    if (server.aof_fsync == AOF_FSYNC_EVERYSEC && !force) {
        /* With this append fsync policy we do background fsyncing.
         * If the fsync is still in progress we can try to delay
         * the write for a couple of seconds. */
        // 如果AOF fsync任务正在进行中
        if (sync_in_progress) {
            // 如果以前从来没有推迟aof flush，那么设置aof_flush_postponed_start 的值为当前时间并退出；
            if (server.aof_flush_postponed_start == 0) {
                /* No previous write postponing, remember that we are
                 * postponing the flush and return. */
                server.aof_flush_postponed_start = server.unixtime;
                return;
            // 如果以前有推迟aof flush，但是与当前时间间隔不超过2s，那么认为还OK，继续推迟，可以退出；即两次aof flush的时间间隔要超过2s，否则推迟aof flush，让redis使用者通过日志排查是否服务器有问题；
            } else if (server.unixtime - server.aof_flush_postponed_start < 2) {
                /* We were already waiting for fsync to finish, but for less
                 * than two seconds this is still ok. Postpone again. */
                return;
            }
            // 否则（即两次AOF flush的任务时间间隔超过2s）输出日志提示，disk is busy? ....this may slow down Redis；即AOF flush的速度太慢了；
            /* Otherwise fall trough, and go write since we can't wait
             * over two seconds. */
            server.aof_delayed_fsync++;
            redisLog(REDIS_NOTICE,"Asynchronous AOF fsync is taking too long (disk is busy?). Writing the AOF buffer without waiting for fsync to complete, this may slow down Redis.");
        }
    }
    /* We want to perform a single write. This should be guaranteed atomic
     * at least if the filesystem we are writing is a real physical one.
     * While this will save us against the server being killed I don't think
     * there is much to do about the whole server stopping for power problems
     * or alike */

    latencyStartMonitor(latency);
    // 将AOF buffer中的内容写入aof文件中；并返回写入内容长度nwritten 
    nwritten = write(server.aof_fd,server.aof_buf,sdslen(server.aof_buf));
    latencyEndMonitor(latency);
    /* We want to capture different events for delayed writes:
     * when the delay happens with a pending fsync, or with a saving child
     * active, and when the above two conditions are missing.
     * We also use an additional event name to save all samples which is
     * useful for graphing / monitoring purposes. */
    if (sync_in_progress) {
        latencyAddSampleIfNeeded("aof-write-pending-fsync",latency);
    } else if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) {
        latencyAddSampleIfNeeded("aof-write-active-child",latency);
    } else {
        latencyAddSampleIfNeeded("aof-write-alone",latency);
    }
    latencyAddSampleIfNeeded("aof-write",latency);

    /* We performed the write so reset the postponed flush sentinel to zero. */
    server.aof_flush_postponed_start = 0;
    // 写入内容长度nwritten与AOF buf不一致，即aof flush失败
    if (nwritten != (signed)sdslen(server.aof_buf)) {
        static time_t last_write_error_log = 0;
        int can_log = 0;
        // 限制aof flush失败的日志输出，即每两次aof flush的warning日志要超过30s（AOF_WRITE_LOG_ERROR_RATE定义），否则can_log=0，即不能输出日志
        /* Limit logging rate to 1 line per AOF_WRITE_LOG_ERROR_RATE seconds. */
        if ((server.unixtime - last_write_error_log) > AOF_WRITE_LOG_ERROR_RATE) {
            can_log = 1;
            last_write_error_log = server.unixtime;
        }

        // nwritten为-1表示写入aof文件失败，输出warnings日志；
        /* Log the AOF write error and record the error code. */
        if (nwritten == -1) {
            if (can_log) {
                redisLog(REDIS_WARNING,"Error writing to the AOF file: %s",
                    strerror(errno));
                server.aof_last_write_errno = errno;
            }
        // 如果nwritten不为-1，表示写入aof文件的内容与期望的内容不一致，输出warnings日志；
        } else {
            if (can_log) {
                redisLog(REDIS_WARNING,"Short write while writing to "
                                       "the AOF file: (nwritten=%lld, "
                                       "expected=%lld)",
                                       (long long)nwritten,
                                       (long long)sdslen(server.aof_buf));
            }

            // 由于只是AOF文件没有写完整，所以尝试通过ftruncate()函数修复AOF文件（server.aof_current_size就是最后一次AOF成功的文件大小）
            if (ftruncate(server.aof_fd, server.aof_current_size) == -1) {
                if (can_log) {
                    redisLog(REDIS_WARNING, "Could not remove short write "
                             "from the append-only file.  Redis may refuse "
                             "to load the AOF the next time it starts.  "
                             "ftruncate: %s", strerror(errno));
                }
            } else {
                /* If the ftruncate() succeeded we can set nwritten to
                 * -1 since there is no longer partial data into the AOF. */
                nwritten = -1;
            }
            server.aof_last_write_errno = ENOSPC;
        }

        // 如果aof flush出错，且AOF flush的策略为AOF_FSYNC_ALWAYS，即总是刷新，这种情况下不能恢复aof文件，只能通过warnings日志告知用户；
        /* Handle the AOF write error. */
        if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
            /* We can't recover when the fsync policy is ALWAYS since the
             * reply for the client is already in the output buffers, and we
             * have the contract with the user that on acknowledged write data
             * is synced on disk. */
            redisLog(REDIS_WARNING,"Can't recover from AOF write error when the AOF fsync policy is 'always'. Exiting...");
            exit(1);
        } else {
            /* Recover from failed write leaving data into the buffer. However
             * set an error to stop accepting writes as long as the error
             * condition is not cleared. */
            server.aof_last_write_status = REDIS_ERR;

            /* Trim the sds buffer if there was a partial write, and there
             * was no way to undo it with ftruncate(2). */
            if (nwritten > 0) {
                server.aof_current_size += nwritten;
                sdsrange(server.aof_buf,nwritten,-1);
            }
            return; /* We'll try again on the next call... */
        }
    } else {
        /* Successful write(2). If AOF was in error state, restore the
         * OK state and log the event. */
        if (server.aof_last_write_status == REDIS_ERR) {
            redisLog(REDIS_WARNING,
                "AOF write error looks solved, Redis can write again.");
            server.aof_last_write_status = REDIS_OK;
        }
    }
    // aof flush成功后更新aof文件size，即增加此次写入内容长度nwritten
    server.aof_current_size += nwritten;

    /* Re-use AOF buffer when it is small enough. The maximum comes from the
     * arena size of 4k minus some overhead (but is otherwise arbitrary). */
    if ((sdslen(server.aof_buf)+sdsavail(server.aof_buf)) < 4000) {
        sdsclear(server.aof_buf);
    } else {
        sdsfree(server.aof_buf);
        server.aof_buf = sdsempty();
    }

    // 如果有正在执行中的RDB或者AOF持久化任务，且no-appendfsync-on-rewrite配置为true（可以通过config配置，或者配置文件），那么不执行fsync；
    /* Don't fsync if no-appendfsync-on-rewrite is set to yes and there are
     * children doing I/O in the background. */
    if (server.aof_no_fsync_on_rewrite &&
        (server.aof_child_pid != -1 || server.rdb_child_pid != -1))
            return;

    // 如果aof flush的策略是AOF_FSYNC_ALWAYS，那么调用aof_fsync()，即调用fdatasync进行数据同步；如果aof flush的策略是AOF_FSYNC_EVERYSEC ，那么调用aof_background_fsync()即创建一个job任务进行fsync；无论哪种策略都记录最后一次fsync的时间到server.aof_last_fsync中；
    /* Perform the fsync if needed. */
    if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
        /* aof_fsync is defined as fdatasync() for Linux in order to avoid
         * flushing metadata. */
        latencyStartMonitor(latency);
        aof_fsync(server.aof_fd); /* Let's try to get this data on the disk */
        latencyEndMonitor(latency);
        latencyAddSampleIfNeeded("aof-fsync-always",latency);
        server.aof_last_fsync = server.unixtime;
    } else if ((server.aof_fsync == AOF_FSYNC_EVERYSEC &&
                server.unixtime > server.aof_last_fsync)) {
        if (!sync_in_progress) aof_background_fsync(server.aof_fd);
        server.aof_last_fsync = server.unixtime;
    }
}
```





# RDB实现源码阅读

RDB相关源码在**rdb.c**中；通过saveCommand(redisClient *c) 和bgsaveCommand(redisClient *c) 两个方法可知，RDB持久化业务逻辑在rdbSave(server.rdb_filename)和rdbSaveBackground(server.rdb_filename这两个方法中；一个通过执行"save"触发，另一个通过执行"bgsave"或者save seconds changes条件满足时(在redis.c的serverCron中)触发：



redis.c里serverCron中通过调用rdbSaveBackground(server.rdb_filename)触发bgsave的部分代码里面会调用rdbSaveBackground(char *filename)方法

rdbSaveBackground(char *filename)的源码可知，其最终的实现还是调用rdbSave(char *filename)，只不过是通过fork()出的子进程来执行罢了，所以bgsave和save的实现是殊途同归

```cpp
int rdbSaveBackground(char *filename) {
    pid_t childpid;
    long long start;

    // 如果已经有RDB持久化任务，那么rdb_child_pid的值就不是-1，那么返回REDIS_ERR；
    if (server.rdb_child_pid != -1) return REDIS_ERR;

    server.dirty_before_bgsave = server.dirty;
    server.lastbgsave_try = time(NULL);

    // 记录RDB持久化开始时间
    start = ustime();
    //fork一个子进程，
    if ((childpid = fork()) == 0) {
        // 如果fork()的结果childpid为0，即当前进程为fork的子进程，那么接下来调用rdbSave()进程持久化；
        int retval;

        /* Child */
        closeListeningSockets(0);
        redisSetProcTitle("redis-rdb-bgsave");
        // bgsave事实上就是通过fork的子进程调用rdbSave()实现, rdbSave()就是save命令业务实现；
        retval = rdbSave(filename);
        if (retval == REDIS_OK) {
            size_t private_dirty = zmalloc_get_private_dirty();

            if (private_dirty) {
                // RDB持久化成功后，如果是notice级别的日志，那么log输出RDB过程中copy-on-write使用的内存
                redisLog(REDIS_NOTICE,
                    "RDB: %zu MB of memory used by copy-on-write",
                    private_dirty/(1024*1024));
            }
        }
        exitFromChild((retval == REDIS_OK) ? 0 : 1);
    } else {
        // 父进程更新redisServer记录一些信息，例如：fork进程消耗的时间stat_fork_time, 
        /* Parent */
        server.stat_fork_time = ustime()-start;
       // 更新redisServer记录fork速率：每秒多少G；zmalloc_used_memory()的单位是字节，所以通过除以(1024*1024*1024),得到GB；由于记录的fork_time即fork时间是微妙，所以*1000000，得到每秒钟fork多少GB的速度；
        server.stat_fork_rate = (double) zmalloc_used_memory() * 1000000 / server.stat_fork_time / (1024*1024*1024); /* GB per second. */
        latencyAddSampleIfNeeded("fork",server.stat_fork_time/1000);
        // 如果fork子进程出错，即childpid为-1，更新redisServer，记录最后一次bgsave状态是REDIS_ERR;
        if (childpid == -1) {
            server.lastbgsave_status = REDIS_ERR;
            redisLog(REDIS_WARNING,"Can't save in background: fork: %s",
                strerror(errno));
            return REDIS_ERR;
        }
        redisLog(REDIS_NOTICE,"Background saving started by pid %d",childpid);
        // 最后在redisServer中记录的save开始时间重置为空，并记录执行bgsave的子进程id，即child_pid；
        server.rdb_save_time_start = time(NULL);
        server.rdb_child_pid = childpid;
        server.rdb_child_type = REDIS_RDB_CHILD_TYPE_DISK;
        updateDictResizePolicy();
        return REDIS_OK;
    }
    return REDIS_OK; /* unreached */
}
```



## RDB持久化实现：

```cpp
/* Save the DB on disk. Return REDIS_ERR on error, REDIS_OK on success. */
int rdbSave(char *filename) {
    char tmpfile[256];
    FILE *fp;
    rio rdb;
    int error;
    // 文件临时文件名为temp-${pid}.rdb
    snprintf(tmpfile,256,"temp-%d.rdb", (int) getpid());
    fp = fopen(tmpfile,"w");
    if (!fp) {
        redisLog(REDIS_WARNING, "Failed opening .rdb for saving: %s",
            strerror(errno));
        return REDIS_ERR;
    }

    rioInitWithFile(&rdb,fp);
    // RDB持久化的核心实现；
    if (rdbSaveRio(&rdb,&error) == REDIS_ERR) {
        errno = error;
        goto werr;
    }

    /* Make sure data will not remain on the OS's output buffers */
    if (fflush(fp) == EOF) goto werr;
    if (fsync(fileno(fp)) == -1) goto werr;
    if (fclose(fp) == EOF) goto werr;

    // 重命名rdb文件的命名；
    /* Use RENAME to make sure the DB file is changed atomically only
     * if the generate DB file is ok. */
    if (rename(tmpfile,filename) == -1) {
        redisLog(REDIS_WARNING,"Error moving temp DB file on the final destination: %s", strerror(errno));
        unlink(tmpfile);
        return REDIS_ERR;
    }
    redisLog(REDIS_NOTICE,"DB saved on disk");
    server.dirty = 0;
    server.lastsave = time(NULL);
    server.lastbgsave_status = REDIS_OK;
    return REDIS_OK;

werr:
    redisLog(REDIS_WARNING,"Write error saving DB on disk: %s", strerror(errno));
    fclose(fp);
    unlink(tmpfile);
    return REDIS_ERR;
}
```

## rdbSaveRio--RDB持久化实现的核心代码

根据RDB文件协议将所有redis中的key-value写入rdb文件中:

```cpp
/* Produces a dump of the database in RDB format sending it to the specified
 * Redis I/O channel. On success REDIS_OK is returned, otherwise REDIS_ERR
 * is returned and part of the output, or all the output, can be
 * missing because of I/O errors.
 *
 * When the function returns REDIS_ERR and if 'error' is not NULL, the
 * integer pointed by 'error' is set to the value of errno just after the I/O
 * error. */
int rdbSaveRio(rio *rdb, int *error) {
    dictIterator *di = NULL;
    dictEntry *de;
    char magic[10];
    int j;
    long long now = mstime();
    uint64_t cksum;

    if (server.rdb_checksum)
        rdb->update_cksum = rioGenericUpdateChecksum;
    // rdb文件中最先写入的内容就是magic，magic就是REDIS这个字符串+4位版本号
    snprintf(magic,sizeof(magic),"REDIS%04d",REDIS_RDB_VERSION);
    if (rdbWriteRaw(rdb,magic,9) == -1) goto werr;

    // 遍历所有db重写rdb文件；
    for (j = 0; j < server.dbnum; j++) {
        redisDb *db = server.db+j;
        dict *d = db->dict;
        // 如果db的size为0，即没有任何key，那么跳过，遍历下一个db；
        if (dictSize(d) == 0) continue;
        di = dictGetSafeIterator(d);
        if (!di) return REDIS_ERR;

        // 写入REDIS_RDB_OPCODE_SELECTDB，这个值redis定义为254，即FE，再通过rdbSaveLen合入当前dbnum，例如当前db为0，那么写入FE 00
        /* Write the SELECT DB opcode */
        if (rdbSaveType(rdb,REDIS_RDB_OPCODE_SELECTDB) == -1) goto werr;
        if (rdbSaveLen(rdb,j) == -1) goto werr;

        // 如注释所表达的，迭代遍历db这个dict的每一个entry；
        /* Iterate this DB writing every entry */
        while((de = dictNext(di)) != NULL) {
            // 先得到当前entry的key(sds类型)和value（redisObject类型）；
            sds keystr = dictGetKey(de);
            robj key, *o = dictGetVal(de);
            long long expire;

            initStaticStringObject(key,keystr);
            // 从redisDb的expire这个dict中查询过期时间属性值；
            expire = getExpire(db,&key);
            // 每个entry（redis中的key和其value）rdb持久化的核心代码
            if (rdbSaveKeyValuePair(rdb,&key,o,expire,now) == -1) goto werr;
        }
        dictReleaseIterator(di);
    }
    di = NULL; /* So that we don't release it again on error. */

    // 遍历所有db后，写入EOF这个opcode，REDIS_RDB_OPCODE_EOF申明为255，即FF，所以是写入FF到rdb文件中；FF是redis对rdb文件结束的定义；
    /* EOF opcode */
    if (rdbSaveType(rdb,REDIS_RDB_OPCODE_EOF) == -1) goto werr;

    // 最后写入8个字节长度的checksum值到rdb文件尾部；
    /* CRC64 checksum. It will be zero if checksum computation is disabled, the
     * loading code skips the check in this case. */
    cksum = rdb->cksum;
    memrev64ifbe(&cksum);
    if (rioWrite(rdb,&cksum,8) == 0) goto werr;
    return REDIS_OK;

werr:
    if (error) *error = errno;
    if (di) dictReleaseIterator(di);
    return REDIS_ERR;
}
```

## 每个entry（key-value）rdb持久化的核心代码

```cpp
/* Save a key-value pair, with expire time, type, key, value.
 * On error -1 is returned.
 * On success if the key was actually saved 1 is returned, otherwise 0
 * is returned (the key was already expired). */
int rdbSaveKeyValuePair(rio *rdb, robj *key, robj *val,
                        long long expiretime, long long now)
{
    /* Save the expire time */
    if (expiretime != -1) {
        // 如果过期时间少于当前时间，那么表示该key已经失效，返回不做任何保存；
        /* If this key is already expired skip it */
        if (expiretime < now) return 0;
        // 如果当前遍历的entry有失效时间属性，那么保存REDIS_RDB_OPCODE_EXPIRETIME_MS即252，即"FC"以及失效时间到rdb文件中，
        if (rdbSaveType(rdb,REDIS_RDB_OPCODE_EXPIRETIME_MS) == -1) return -1;
        if (rdbSaveMillisecondTime(rdb,expiretime) == -1) return -1;
    }

    // 接下来保存redis key的类型，key，以及value到rdb文件中；
    /* Save type, key, value */
    if (rdbSaveObjectType(rdb,val) == -1) return -1;
    if (rdbSaveStringObject(rdb,key) == -1) return -1;
    if (rdbSaveObject(rdb,val) == -1) return -1;
    return 1;
}

```

通过上面的源码分析得到最终rdb文件的格式如下：

> REDIS     // RDB协议约束的固定字符串
> 0006        // redis的版本号
> FE 00       // 表示当前接下来的key都是db=0中的key；
> FC 1506327609  // 表示key失效时间点为1506327609
> 0 // 表示key的属性是string类型；
> username // key
> afei           // value
> FF            // 表示遍历完成
> y73e9iq1  // checksum值





