## 幻读

- 幻读是什么？

  - 幻读指的是一个事务在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行。幻读仅专指“新插入的行”（我们给所有行加锁的时候，id=1 这一行还不存在，不存在也就加不上锁。）。
  - 在可重复读隔离级别下，普通的查询是快照读，是不会看到别的事务插入的数据的。因此，幻读在“当前读”下才会出现。

- 数据一致性的问题

  - 锁的设计是为了保证数据的一致性。而这个一致性，不止是数据库内部数据状态在此刻的一致性，还包含了数据和日志在逻辑上的一致性。

  - update 的加锁语义和 select …for update 是一致的

  - > T1时刻session A给（d=1，id=0）加锁，T2时刻session B update d=1 where id=1，T3时刻session A提交
    >
    > id=0 和 id=1 这两行，发生了数据不一致。这个问题很严重，是不行的。

  - > 由于 session A 把所有的行都加了写锁，所以 session B 在执行第一个 update 语句的时候就被锁住了。
    >
    > 
    >
    >  insert into t values(1,1,5); /*(1,1,5)*/
    >  update t set c=5 where id=1; /*(1,5,5)*/
    >  update t set d=100 where d=5;/* 所有 d=5 的行，d 改成 100*/
    >
    > 
    >
    > 也就是说，即使把所有的记录都加上锁，还是阻止不了新插入的记录，这也是为什么“幻读”会被单独拿出来解决的原因。

- 如何解决幻读？

  - 产生幻读的原因是，行锁只能锁住行，但是新插入记录这个动作，要更新的是记录之间的“间隙”。因此，为了解决幻读问题，InnoDB 只好引入新的锁，也就是间隙锁 (Gap Lock)。
  - 跟间隙锁存在冲突关系的，是“往这个间隙中插入一个记录”这个操作。间隙锁之间都不存在冲突关系。
  - 间隙锁的引入，可能会导致同样的语句锁住更大的范围，这其实是影响了并发度的。
  - 间隙锁是在可重复读隔离级别下才会生效的
  - 间隙锁和行锁合称 next-key lock，每个 next-key lock 是前开后闭区间







## 为什么我只改一行的语句，锁这么多

- 加锁规则里面，包含了两个“原则”、两个“优化”

  - 原则1：加锁的基本单位是 next-key lock

  - 原则2：查找过程中访问到的对象才会加锁。

    - > 查询使用覆盖索引，并不需要访问主键索引，所以主键索引上没有加任何锁，这就是为什么的 update 语句可以执行完成。
      >
      > update set d=d+1 where id =5;

    - lock in share mode 只锁覆盖索引，但是如果是 for update就不一样了。 执行 for update 时，系统会认为你接下来要更新数据，因此会顺便给主键索引上满足条件的行加上行锁。

    - 如果你要用 lock in share mode 来给行加读锁避免数据被更新的话，就必须得绕过覆盖索引的优化，在查询字段中加入索引中不存在的字段。比如，将 session  的查询语句改成 select d from t wherec=5 lock in share mode

  - 优化1：索引上的等值查询，给唯一索引、主键索引也算加锁的时候，next-key lock 退化为行锁

  - 优化2：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙锁。

- 锁是加在索引上的

- 案例一：等值查询间隙锁

- 案例二：非唯一索引等值锁

- 案例三：主键索引范围锁

- 案例四：非唯一索引范围锁

- 案例六：非唯一索引上存在"等值"的例子

- 案例七：limit 语句加锁

  - 在删除数据的时候尽量加 limit。这样不仅可以控制删除数据的条数，让操作更安全，还可以减小加锁的范围。

- 案例八：一个死锁的例子

  - session  A的“加 next-key lock(5,10] ”操作，实际上分成了两步，先是加 (5,10) 的间隙锁，加锁成功；然后加 c=10 的行锁，这时候才被锁住的。
    - 也就是说，我们在分析加锁规则的时候可以用 next-key lock 来分析。但是要知道，具体执行的时候，是要分成间隙锁和行锁两段来执行的。

- 共享锁 (lock in share mode)、排他锁 (for update)





##  MySQL是怎么保证数据不丢的

- 只要 redo log 和 binlog 保证持久化到磁盘，就能确保 MySQL 异常重启后，数据可以恢复。
- binlog 的写入机制
  - 事务执行过程中，先把日志写到 binlog cache，事务提交的时候，再把 binlog cache 写到binlog 文件中。
    - write，指的就是指把日志写入到文件系统的 binlog cache，并没有把数据持久化到磁盘，所以速度比较快。
    - fsync，才是将数据持久化到磁盘的操作。一般情况下，我们认为 fsync 才占磁盘的 IOPS。
  - write 和 fsync 的时机，是由参数 sync_binlog 控制的：
    - sync_binlog=0 的时候，表示每次提交事务都只 write，不 fsync；
    2. sync_binlog=1 的时候，表示每次提交事务都会执行 fsync；
    3. sync_binlog=N(N>1) 的时候，表示每次提交事务都 write，但累积 N 个事务后才fsync。
  - 一个事务的 binlog 是不能被拆开的，因此不论这个事务多大，也要确保一次性写入。这就涉及到了 binlog cache 的保存问题。
    3. 分配了一片内存，每个线程一个，参数 binlog_cache_size 用于控制单个线程内 binlog cache 所占内存的大小。如果超过了这个参数规定的大小，就要暂存到磁盘。
- redo log 的写入机制
  3. redo log buffer。事务在执行过程中，生成的 redo log 是要先写到 redo log buffer 的。
  - 事务还没提交的时候，redo log buffer 中的部分日志有没有可能被持久化到磁盘呢？答案是，确实会有。
  - 为了控制 redo log 的写入策略，InnoDB 提供了 innodb_flush_log_at_trx_commit 参数，它有三种可能取值：
    3. 设置为 0 的时候，表示每次事务提交时都只是把 redo log 留在 redo log buffer 中
    2. 设置为 1 的时候，表示每次事务提交时都将 redo log 直接持久化到磁盘；
    3. 设置为 2 的时候，表示每次事务提交时都只是把 redo log 写到 page cache。
  3. InnoDB 有一个后台线程，每隔 1 秒，就会把 redo log buffer 中的日志，调用 write 写到文件系统的 page cache，然后调用 fsync 持久化到磁盘。
  - 实际上，除了后台线程每秒一次的轮询操作外，还有两种场景会让一个没有提交的事务的redo log 写入到磁盘中。
    3. 一种是，redo log buffer 占用的空间即将达到 innodb_log_buffer_size 一半的时候，后台线程会主动写盘	由于这个事务并没有提交，所以这个写盘动作只是write，而没有调用 fsync，也就是只留在了文件系统的 page cache。
    3. 另一种是，并行的事务提交的时候，顺带将这个事务的 redo log buffer 持久化到磁盘

- 时序上 redo log 先 prepare， 再写binlog，最后再把 redo log commit。

  - 如果把 innodb_flush_log_at_trx_commit 设置成 1，那么 redo log 在 prepare 阶段就要持久化一次，因为有一个崩溃恢复逻辑是要依赖于 prepare 的 redo log，再加上 binlog来恢复的。

- 组提交（group commit）机制

  - 日志逻辑序列号（log sequence number，LSN）

    - > 三个并发事务 (trx1, trx2, trx3) 在 prepare 阶段，都写完 redo log buffer，持久化到磁盘的过程，对应的 LSN 分别是 50、120 和 160。
      > 1. trx1 是第一个到达的，会被选为这组的 leader；
      > 2. 等 trx1 要开始写盘的时候，这个组里面已经有了三个事务，这时候 LSN 也变成了160；
      > 3. trx1 去写盘的时候，带的就是 LSN=160，因此等 trx1 返回时所有 LSN 小于等于160 的 redo log，都已经被持久化到磁盘；
      > 4. 这时候 trx2 和 trx3 就可以直接返回了。

  - WAL 机制主要得益于两个方面：

    - redo log 和 binlog 都是顺序写，磁盘的顺序写比随机写速度要快；
    - 组提交机制，可以大幅度降低磁盘的 IOPS 消耗。

- 如果你的 MySQL 现在出现了性能瓶颈，而且瓶颈在 IO 上，可以通过哪些方法来提升性能呢？

  - 设置 binlog_group_commit_sync_delay 和binlog_group_commit_sync_no_delay_count 参数，减少 binlog 的写盘次数。可能会增加语句的响应时间，但没有丢失数据的风险
  - 将 sync_binlog 设置为大于 1 的值（比较常见是 100~1000）。这样做的风险是，主机掉电时会丢 binlog 日志。
  -  将 innodb_flush_log_at_trx_commit 设置为 2。这样做的风险是，主机掉电的时候会丢数据。

- binlog cache 是每个线程自己维护的，而 redo log buffer 是全局共用的
  - binlog 是不能“被打断的”，一个事务的 binlog必须连续写，因此要整个事务完成后，再一起写到文件里
  -  redo log 并没有这个要求，中间有生成的日志可以写到 redo log buffer 中。redo log buffer 中的内容还能“搭便车”，其他事务提交的时候可以被一起写到磁盘中







## MySQL是怎么保证主备一致的

-  binlog 可以用来归档，也可以用来做主备同步，但它的内容是什么样的呢？为什么备库执行了 binlog 就可以跟主库保持一致了呢
- MySQL 主备的基本原理
  - 备库设置成只读（readonly）模式。这样做，有以下几个考虑：
    - 有时候一些运营类的查询语句会被放到备库上去查，设置为只读可以防止误操作
    2. 防止切换逻辑有 bug，比如切换过程中出现双写，造成主备不一致；
    3. 可以用 readonly 状态，来判断节点的角色。
  - 主备流程
    3. 主库接收到客户端的更新请求后，执行内部事务的更新逻辑，同时写 binlog。
    3. 备库 B 跟主库 A 之间维持了一个长连接。主库 A 内部有一个线程，专门用于服务备库 B的这个长连接。
    3. 备库 B  change master 命令
    3. 在备库 B 上执行 start slave 命令，这时候备库会启动两个线程，就是 io_thread和 sql_thread。其中 io_thread 负责与主库建立连接。
    3. 主库 A从本地读取 binlog，发给 B
    3. 备库 B 拿到 binlog 后，写到本地文件，称为中转日志（relay log）。
    3. sql_thread 读取中转日志，解析出日志里的命令，并执行
    3. 后来由于多线程复制方案的引入，sql_thread 演化成为了多个线程

- binlog 的三种格式对比

  - statement

    - 当 binlog_format=statement 时，binlog 里面记录的就是 SQL 语句的原文。

    - 由于 statement 格式下，记录到 binlog 里的是语句原文，因此可能会出现这样一种情况：在主库执行这条 SQL 语句的时候，用的是索引 a；而在备库执行这条 SQL 语句的时候，却使用了索引 t_modified。因此，MySQL 认为这样写是有风险的

    - >  delete from t  where a>=4 and t_modified<='2018-11-10' limit 1;
      >
      > 
      >
      > 如果 delete 语句使用的是索引 a，那么会根据索引 a 找到第一个满足条件的行，也就是说删除的是 a=4 这一行；
      >
      > 但如果使用的是索引 t_modified，那么删除的就是 t_modified='2018-11-09’也就是a=5 这一行。

  - row

    - row 格式的 binlog 里没有了 SQL 语句的原文，而是替换成了两个 event：Table_map 和 Delete_rows。
    - 借助 mysqlbinlog 工具  解析和查看 binlog 中的内容。 mysqlbinlog -vv data/master.000001 --start-position=8900;
    - 你可以看到，当 binlog_format 使用 row 格式的时候，binlog 里面记录了真实删除行的主键 id，这样 binlog 传到备库去的时候，就肯定会删除 id=4 的行，不会有主备删除不同行的问题。

  -  mixed

    - 有些 statement 格式的 binlog 可能会导致主备不一致，所以要使用 row 格式。
    - row 格式的缺点是，很占空间
      - 比如你用一个 delete 语句删掉 10 万行数据，用statement 的话就是一个 SQL 语句被记录到 binlog 中，占用几十个字节的空间。但如果用 row 格式的 binlog，就要把这 10 万条记录都写到 binlog 中

  - 现在越来越多的场景要求把 MySQL 的 binlog 格式设置成 row。这么做的理由有很多，一个可以直接看出来的好处：恢复数据。

- 循环复制问题

  - binlog 的特性确保了在备库执行相同的 binlog，可以得到与主库相同的状态。因此，我们可以认为正常情况下主备的数据是一致的。
  - 双 M 结构（各为主备）还有一个问题需要解决。
    - MySQL 在 binlog 中记录了这个命令第一次执行时所在实例的server id。因此，我们可以用下面的逻辑，来解决两个节点间的循环复制的问题：
      - 规定两个库的 server id 必须不同
      - 一个备库接到 binlog 并在重放的过程中，生成与原 binlog 的 server id 相同的新的binlog；
      - 每个库在收到从自己的主库发过来的日志后，先判断 server id，如果跟自己的相同，表示这个日志是自己生成的，就直接丢弃这个日志。







## MySQL是怎么保证高可用的

- 正常情况下，只要主库执行更新生成的所有 binlog，都可以传到备库并被正确地执行，备库就能达到跟主库一致的状态，这就是最终一致性。
  - 你可以在备库上执行 show slave status 命令，它的返回结果里面会显示seconds_behind_master，用于表示当前备库延迟了多少秒。
  - 主备延迟最直接的表现是，备库消费中转日志（relay log）的速度，比主库生产binlog 的速度要慢。
- 主备延迟的来源
  - 备库的压力大	
    - 一主多从。除了备库外，可以多接几个从库，让这些从库来分担读的压力。
    - 其中，一主多从的方式大都会被采用。因为作为数据库系统，还必须保证有定期全量备份的能力。而从库，就很适合用来做备份。
  - 大事务
  - 还有一个大方向的原因，就是备库的并行复制能力

- 可靠性优先策略

  - 双 M 结构下
  - 判断备库 B 的 seconds_behind_master 的值，直到这个值变成 0 为止；
  - 这个切换流程中是有不可用时间的

- 可用性优先策略

  - 直接切换
  - 使用 row 格式的 binlog 时，数据不一致的问题更容易被发现。而使用 mixed 或者statement 格式的 binlog 时，数据很可能悄悄地就不一致了
    - 因为 row 格式在记录 binlog 的时候，会记录新插入的行的所有字段值，所以最后只会有一行不一致。而且，两边的主备同步的应用线程会报错 duplicate key error 并停止。也就是说，这种情况下，备库 B 的 (5,4) 和主库 A 的 (5,5) 这两行数据，都不会被对方执行
  - 有没有哪种情况数据的可用性优先级更高呢
    - 有一个库的作用是记录操作日志。这时候，如果数据不一致可以通过 binlog 来修补，而这个短暂的不一致也不会引发业务问题。同时，业务系统依赖于这个日志写入逻辑，如果这个库不可写，会导致线上的业务操作无法执行。这时候，你可能就需要选择先强行切换，事后再补数据的策略。
    - 改进措施
      - 让业务逻辑不要依赖于这类日志的写入。也就是说，日志写入这个逻辑模块应该可以降级，比如写到本地文件，或者写到另外一个临时库里面。

- 什么情况下双 M 结构会出现循环复制

  - 在一个主库更新事务后，用命令 set global server_id=x 修改了 server_id。等日志再传回来的时候，发现 server_id 跟自己的 server_id 不同

  - 三节点复制的场景，做数据库迁移的时候会出现。

    - 有三个节点的时候，trx1 是在节点 B 执行的，因此 binlog上的 server_id 就是 B，binlog 传给节点 A，然后 A 和 A’搭建了双 M 结构，就会出现循环复制。

    - > 如果出现了循环复制，可以在 A 或者 A’上，执行如下命令：
      > 1 stop slave； 
      >
      > 2 CHANGE MASTER TO IGNORE_SERVER_IDS=(server_id_of_B);
      >
      > 3 start slave;
      >
      > 
      >
      > 这样这个节点收到日志后就不会再执行。过一段时间后，再执行下面的命令把这个值改回来。
      > 1 stop slave；
      > 2 CHANGE MASTER TO IGNORE_SERVER_IDS=();
      > 3 start slave;







## 备库为什么会延迟好几个小时

- 备库并行复制能力

  - coordinator 就是原来的 sql_thread, 不过现在它不再直接更新数据了，只负责读取中转日志和分发事务
  - 真正更新日志的，变成了 worker 线程。而 work 线程的个数，就是由参数 slave_parallel_workers 决定的
  - coordinator 在分发的时候，需要满足以下这两个基本要求：
    - 更新同一行的两个事务，必须被分发到同一个 worker中。
    - 同一个事务不能被拆开，必须放到同一个 worker 中。

- MariaDB 的并行复制策略

  - > 1. 在一组里面一起提交的事务，有一个相同的 commit_id，下一组就是 commit_id+1；
    > 2. commit_id 直接写到 binlog 里面；
    > 3. 传到备库应用的时候，相同 commit_id 的事务分发到多个 worker 执行；
    > 4. 这一组全部执行完成后，coordinator 再去取下一批。

  - 这个方案很容易被大事务拖后腿。假设 trx2 是一个超大事务，那么在备库应用的时候，trx1 和 trx3 执行完成后，就只能等 trx2 完全执行完成，下一组才能开始执行。

- MySQL 5.7 的并行复制策略

  - 由参数slave-parallel-type 来控制并行复制策略
  - MySQL 5.7 并行复制策略的思想是
    - 更新同一行的事务是不可能同时进入 commit 状态的
    - 同时处于 prepare 状态的事务，在备库执行时是可以并行的；
    - 处于 prepare 状态的事务，与处于 commit 状态的事务之间，在备库执行时也是可以并行的。
  - 这两个参数是用于故意拉长 binlog 从 write 到 fsync 的时间，以此减少 binlog 的写盘次数。在 MySQL 5.7 的并行复制策略里，它们可以用来制造更多的“同时处于 prepare 阶段的事务”。这样就增加了备库复制的并行度。
    - binlog_group_commit_sync_delay 参数，表示延迟多少微秒后才调用 fsync;.
    - binlog_group_commit_sync_no_delay_count 参数，表示累积多少次以后才调用fsync。

- MySQL 5.7.22 的并行复制策略

  - 新增了一个参数 binlog-transaction-dependency-tracking，用来控制是否启用这个新策略。

    - COMMIT_ORDER，表示的就是前面介绍的，根据同时进入 prepare 和 commit 来判断是否可以并行的策略。

    - WRITESET，表示的是对于事务涉及更新的每一行，计算出这一行的 hash 值，组成集合writeset。如果两个事务没有操作相同的行，也就是说它们的 writeset 没有交集，就可以并行。

    -  WRITESET_SESSION，是在 WRITESET 的基础上多了一个约束，即在主库上同一个线程先

      后执行的两个事务，在备库执行的时候，要保证相同的先后顺序。

    - 当然为了唯一标识，这个 hash 值是通过“库名 + 表名 + 索引名 + 值”计算出来的。如果一个表上除了有主键索引外，还有其他唯一索引，那么对于每个唯一索引，insert 语句对应的 writeset 就要多增加一个 hash 值







## 读写分离

- 判断主备无延迟方案

  - 第一种

    - show slave status 结果里的seconds_behind_master 参数的值，可以用来衡量主备延迟时间的长短。
    - 每次从库执行查询请求前，先判断seconds_behind_master 是否已经等于 0。如果还不等于 0 ，那就必须等到这个参数变为0 才能执行查询请求。

  - 第二种

    - > 对比位点确保主备无延迟：
      > Master_Log_File 和 Read_Master_Log_Pos，表示的是读到的主库的最新位点；
      > Relay_Master_Log_File 和 Exec_Master_Log_Pos，表示的是备库执行的最新位点。

  - 第三种

    - > 对比 GTID 集合确保主备无延迟：
      > Auto_Position=1 ，表示这对主备关系使用了 GTID 协议。
      > Retrieved_Gtid_Set，是备库收到的所有日志的 GTID 集合；
      > Executed_Gtid_Set，是备库所有已经执行完成的 GTID 集合。
      > 如果这两个集合相同，也表示备库接收到的日志都已经同步完成。

-  semi-sync （半同步复制）方案

  - semi-sync 做了这样的设计：

    > 	1. 事务提交的时候，主库把 binlog 发给从库；
    >
    > 2. 从库收到 binlog 以后，发回给主库一个 ack，表示收到了；
    > 3. 主库收到这个 ack 以后，才能给客户端返回“事务完成”的确认。

  - 判断同步位点的方案还有另外一个潜在的问题，即：如果在业务更新的高峰期，主库的位点或者 GTID 集合更新很快，那么上面的两个位点等值判断就会一直不成立，很可能出现从库上迟迟无法响应查询请求的情况。

    - 备库 B 一直都和主库 A 存在延迟，如果用上面必须等到无延迟才能查询的方案，select 语句一直都不能被执行。
    - 其实客户端是在发完 trx1 更新后发起的 select 语句，我们只需要确保 trx1 已经执行完成就可以执行 select 语句了

  - semi-sync 配合判断主备无延迟的方案，存在两个问题：

    - 一主多从的时候，在某些从库执行查询请求会存在过期读的现象；
      - semi-sync+ 位点判断的方案，只对一主一备的场景是成立的。在一主多从场景中，主库只要等到一个从库的 ack，就开始给客户端返回确认。
    - 在持续延迟的情况下，可能出现过度等待的问题。

- 等主库位点方案

  - > select master_pos_wait(file, pos[, timeout])
    >
    > 	1. 它是在从库执行的；
    >
    > 2. 参数 file 和 pos 指的是主库上的文件名和位置；
    > 3. timeout 可选，设置为正整数 N 表示这个函数最多等待 N 秒。
    > 	这个命令正常返回的结果是一个正整数 M，表示从命令开始执行，到应用完 file 和 pos 表示的 binlog 位置，执行了多少事务。

  - 对于先执行 trx1，再执行一个查询请求的逻辑，要保证能够查到正确的数据，我们可以使用这个逻辑：

    - > 1. trx1 事务更新完成后，马上执行 show master status 得到当前主库执行到的 File 和Position；
      > 2. 选定一个从库执行查询语句；
      > 3. 在从库上执行 select master_pos_wait(File, Position, 1)；
      > 4. 如果返回值是 >=0 的正整数，则在这个从库执行查询语句；
      > 5. 否则，到主库执行查询语句。

    - 这里我们假设，这条 select 查询最多在从库上等待 1 秒。那么，如果 1 秒内master_pos_wait 返回一个大于等于 0 的整数，就确保了从库上执行的这个查询结果一定包含了 trx1 的数据。

- 等 GTID 方案

  - > select wait_for_executed_gtid_set(gtid_set, 1);
    >
    > 1. 等待，直到这个库执行的事务中包含传入的 gtid_set，返回 0；
    > 2. 超时返回 1。

  - 这时，等 GTID 的执行流程就变成了：

    - > 1. trx1 事务更新完成后，从返回包直接获取这个事务的 GTID，记为 gtid1；
      > 2. 选定一个从库执行查询语句；
      > 3. 在从库上执行 select wait_for_executed_gtid_set(gtid1, 1)；
      > 4. 如果返回值是 0，则在这个从库执行查询语句；
      > 5. 否则，到主库执行查询语句。

  - MySQL 5.7.6 版本开始，允许在执行完更新类事务后，把这个事务的 GTID 返回给客户端，这样等 GTID 的方案就可以减少一次查询。

    - 问题是，怎么能够让 MySQL 在执行事务后，返回包中带上 GTID 呢？
      - 你只需要将参数 session_track_gtids 设置为 OWN_GTID，然后通过 API 接口mysql_session_track_get_first 从返回包解析出 GTID 的值即可。

  

























