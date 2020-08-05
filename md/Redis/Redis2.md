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
  -  **1** AOF文件超过64M且增长一定比例(最后一次AOF文件重写后增长了aof_rewrite_perc，默认是100%，在redis.h中有定义：REDIS_AOF_REWRITE_PERC，可以通过config get/set auto-aof-rewrite-percentage热修改)
  -  **2** 有AOF重写的调度任务（例如执行BGREWRITEAOF命令）

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
>  0006        // redis的版本号
>  FE 00       // 表示当前接下来的key都是db=0中的key；
>  FC 1506327609  // 表示key失效时间点为1506327609
>  0 // 表示key的属性是string类型；
>  username // key
>  afei           // value
>  FF            // 表示遍历完成
>  y73e9iq1  // checksum值

