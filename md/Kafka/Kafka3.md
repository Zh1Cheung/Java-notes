## 认证机制

- 认证是指通过一定的手段，完成对用户身份的确认。授权一般是指对信息安全或计算机安全相关的资源定义与授予相应的访问权限

- SASL/SCRAM-SHA-256 配如示例的
  
- - 第 1 步：创建用户
    - 在本次测试中，我会创建 3 个用户，分别是 admin 用户、writer 用户和 reader 用户。admin 用户用于实现Broker 间通信，writer 用户用于生产消息，reader 用户用于消费消息
  - 第 2 步：创建 JAAS 文件
    - 在实际场景中，你需要为每台单独的物理 Broker 机器都创建一份 JAAS 文件
    - 配置 Broker 的server.properties 文件
  - 第 3 步：启动 Broker
    - 现在我们分别启动这两个 Broker。在启动时，你需要指定 JAAS 文件的位置
  - 第 4 步：发送消息
    - 由于启用了认证，客户端需要做一些相应的配置。我们创建一个名为 producer.conf 的配置文件
  - 第 5 步：消费消息
    - 接下来，我们使用 Console Consumer 程序来消费一下刚刚生产的消息。同样地，我们需要为 kafka-console-consumer 脚本创建一个名为 consumer.conf 的脚本
  
- > kafka-acls 脚本
  > 	如果我们要为用户 Alice 增加了集群级别的所有权限，那么我们可以使用下面这段命令。
  > 	$ kafka-acls --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:Alice --operation All --topic '\*' --cluster
  >
  > ​		在这个命令中，All 表示所有操作，topic 中的星号则表示所有主题，指定 --cluster 则说明我们要为 Alice 设置的是集群权限。
  >
  > 
  >
  > ​	这个命令的意思是，允许所有的用户使用任意的 IP 地址读取名为 test-topic 的主题数据，同时也禁止 BadUser 用户和 10.205.96.119 的 IP 地址访问 test-topic 下的消息。
  >
  > ​		 bin/kafka-acls --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:'\*' --allow-host '\*' --deny-principal User:BadUser --deny-host 10.205.96.119 --operation Read --topic test-topic
  >
  > User 后面的星号表示所有用户，allow-host 后面的星号则表示所有 IP 地址。

- ACL 权限列表
  - 通常情况下，Producer 程序还可能赋予写权限、创建主题、获取主题数据的权限，所以 Kafka 为 Producer 需要的这些常见权限创建了快捷方式，即 --producer。也就是说，在执行 kafka-acls 命令时，直接指定 --producer 就能同时获得这三个权限了。 
  - --consumer 也是类似的，指定 --consumer 可以同时获得 Consumer 端应用所需的权限。







## 监控Kafka

- 主机监控
  - 所谓主机监控，指的是监控 Kafka 集群Broker 所在的节点机器的性能，比如 CPU、内存或磁盘等
  - 运行 top 命令
- JVM 监控
  - 比如，Broker 进程的堆大小（HeapSize）是多少、各自的新生代和老年代是多大，用的是什么 GC 回收
  - 要做到 JVM 进程监控，有 3 个指标需要你时刻关注
    - Full GC 发生频率和时长。这个指标帮助你评估 Full GC 对 Broker 进程的影响。长时间的停顿会令 Broker 端抛出各种超时异常
    - 活跃对象大小。这个指标是你设定堆大小的重要依据，同时它还能帮助你细粒度地调优JVM 各个代的堆大小。
    - 应用线程总数。这个指标帮助你了解 Broker 进程对 CPU 的使用情况。
  - 谈到具体的监控，前两个都可以通过 GC 日志来查看
    - 自 0.9.0.0 版本起，社区将默认的 GC 收集器设置为 G1，而 G1 中的 Full GC 是由单线程执行的，速度非常慢。因此，你一定要监控你的Broker GC 日志，即以 kafkaServer-gc.log 开头的文件。注意不要出现 Full GC 的字样。一旦你发现 Broker 进程频繁 FullGC，可以开启 G1 的 -XX:+PrintAdaptiveSizePolicy 开关，让 JVM 告诉你到底是谁引发了 Full GC。

- 集群监控
  - 查看 Broker 进程是否启动，端口是否建立
  - 查看 Broker 端关键日志
    - 这里的关键日志，主要涉及 Broker 端服务器日志 server.log，控制器日志 controller.log以及主题分区状态变更日志 state-change.log。
    - 其中，server.log 是最重要的，你最好时刻对它保持关注。很多 Broker 端的严重错误都会在这个文件中被展示出来
  - 查看 Broker 端关键线程的运行状态
    - 比方说，Broker 后台有个专属的线程执行 Log Compaction 操作，由于源代码的 Bug，这个线程有时会无缘无故地“死掉”，社区中很多 Jira 都曾报出过这个问题。当这个线程挂掉之后，作为用户的你不会得到任何通知，Kafka 集群依然会正常运转，只是所有的 Compaction 操作都不能继续了，这会导致 Kafka 内部的位移主题所占用的磁盘空间越来越大。因此，我们有必要对这些关键线程的状态进行监控。
  - 查看 Broker 端的关键 JMX 指标
    - BytesIn/BytesOut：即 Broker 端每秒入站和出站字节数。
    - NetworkProcessorAvgIdlePercent：即网络线程池线程平均的空闲比例。
    - RequestHandlerAvgIdlePercent：即 I/O 线程池线程平均的空闲比例。
    - UnderReplicatedPartitions：即未充分备份的分区数。
      - 所谓未充分备份，是指并非所有的 Follower 副本都和 Leader 副本保持同步。一旦出现了这种情况，通常都表明该分区有可能会出现数据丢失
    - ISRShrink/ISRExpand：即 ISR 收缩和扩容的频次指标。
    - ActiveControllerCount：即当前处于激活状态的控制器的数量。
      - 正常情况下，Controller 所在 Broker 上的这个 JMX 指标值应该是 1，其他 Broker 上的这个值是 0。如果你发现存在多台 Broker 上该值都是 1 的情况，一定要赶快处理，处理方式主要是查看网络连通性。这种情况通常表明集群出现了脑裂。
  - 监控 Kafka 客户端
    - 不管是生产者还是消费者，我们首先要关心的是客户端所在的机器与 Kafka Broker 机器之间的网络往返时延
      - 通俗点说，就是你要在客户端机器上 ping 一下 Broker 主机 IP，看看 RTT 是多少
    - 对于生产者而言，有一个以kafka-producer-network-thread 开头的线程是你要实时监控的。它是负责实际消息发送的线程。
    - 对于消费者而言，心跳线程事关 Rebalance，也是必须要监控的一个线程。它的名字以 kafka-coordinator-heartbeat-thread 开头。
  - 除此之外，客户端有一些很重要的 JMX 指标，可以实时告诉你它们的运行情况
    - 从 Producer 角度，你需要关注的 JMX 指标是 request-latency，即消息生产请求的延时。这个 JMX 最直接地表征了 Producer 程序的 TPS；
    - 而从 Consumer 角度来说，records-lag 和 records-lead 是两个重要的 JMX 指标。它们直接反映了 Consumer 的消费进度。
    - 如果你使用了 Consumer Group，那么有两个额外的 JMX 指标需要你关注下，一个是 join rate，另一个是 sync rate。它们说明了 Rebalance 的频繁程度。







## Kafka调优

- 操作系统调优
  - 在操作系统层面，你最好在挂载（Mount）文件系统时禁掉 atime 更新。atime 的全称是 access time，记录的是文件最后被访问的时间。记录atime 需要操作系统访问 inode 资源
    - 执行mount -o noatime 命令进行设置
  - 至于文件系统，我建议你至少选择 ext4 或 XFS。尤其是 XFS 文件系统，它具有高性能、高伸缩性等特点，特别适用于生产服务器。
  - 另外就是 swap 空间的设置。我个人建议将 swappiness 设置成一个很小的值，比如 1～10 之间，以防止 Linux 的 OOM Killer 开启随意杀掉进程。你可以执行 sudo sysctlvm.swappiness=N 来临时设置该值，如果要永久生效，可以修改 /etc/sysctl.conf 文件，增加 vm.swappiness=N，然后重启机器即可。
  - 操作系统层面还有两个参数也很重要，它们分别是ulimit -n 和 vm.max_map_count。前者如果设置得太小，你会碰到 Too Many File Open 这类的错误，而后者的值如果太小，在一个主题数超多的 Broker 机器上，你会碰到OutOfMemoryError：Map failed的严重错误，因此，我建议在生产环境中适当调大此值，比如将其设置为 655360。
    - 修改 /etc/sysctl.conf 文件，增加 vm.max_map_count=655360，保存之后，执行sysctl -p 命令使它生效。
  - 最后，不得不提的就是操作系统页缓存大小了，这对 Kafka 而言至关重要。在某种程度上，我们可以这样说：给 Kafka 预留的页缓存越大越好，最小值至少要容纳一个日志段的大小，也就是 Broker 端参数 log.segment.bytes 的值。该参数的默认值是 1GB。预留出一个日志段大小，至少能保证 Kafka 可以将整个日志段全部放入页缓存，这样，消费者程序在消费时能直接命中页缓存，从而避免昂贵的物理磁盘 I/O 操作。
- JVM 层调优
  - 设置堆大小
    - 将你的 JVM 堆大小设置成 6～8GB。
    - 如果你想精确调整的话，我建议你可以查看 GC log，特别是关注 Full GC 之后堆上存活对象的总大小，然后把堆大小设置为该值的 1.5～2 倍。如果你发现 Full GC 没有被执行过，手动运行jmap -histo:live < pid > 就能人为触发 Full GC。
  - GC 收集器的选择
    - 你一定要尽力避免 Full GC 的出现。其实，不论使用哪种收集器，都要竭力避免Full GC。在 G1 中，Full GC 是单线程运行的，它真的非常慢。如果你的 Kafka 环境中经常出现 Full GC，你可以配置 JVM 参数 -XX:+PrintAdaptiveSizePolicy，来探查一下到底是谁导致的 Full GC。
      使用 G1 还很容易碰到的一个问题，就是大对象（Large Object）
    - 要解决这个问题，除了增加堆大小之外，你还可以适当地增加区域大小，设置方法是增加 JVM 启动参数 -XX:+G1HeapRegionSize=N。
- Broker 端调优
  - 尽力保持客户端版本和 Broker 端版本一致
    - 不要小看版本间的不一致问题，它会令 Kafka 丧失很多性能收益，比如 Zero Copy。
    - Producer、Consumer 和 Broker 的版本是相同的，它们之间的通信可以享受Zero Copy 的快速通道；相反，一个低版本的 Consumer 程序想要与 Producer、Broker交互的话，就只能依靠 JVM 堆中转一下，丢掉了快捷通道，就只能走慢速通道了。

- 应用层调优
  - 不要频繁地创建 Producer 和 Consumer 对象实例
  - 用完及时关闭
  - 合理利用多线程来改善性能	
    - Kafka 的 Java Producer 是线程安全的，你可以放心地在多个线程中共享同一个实例；而 Java Consumer 虽不是线程安全的，但有多线程的方案

- 调优吞吐量
  - Broker 端参数 num.replica.fetchers 表示的是 Follower 副本用多少个线程来拉取消息，默认使用 1 个线程。如果你的 Broker 端 CPU 资源很充足，不妨适当调大该参数值，加快Follower 副本的同步速度。因为在实际生产环境中，配置了 acks=all 的 Producer 程序吞吐量被拖累的首要因素，就是副本同步性能。增加这个值后，你通常可以看到 Producer端程序的吞吐量增加。
  - 在 Producer 端，如果要改善吞吐量，通常的标配是增加消息批次的大小以及批次缓存时间，即 batch.size 和 linger.ms
    - 除了这两个，你最好把压缩算法也配置上，以减少网络 I/O 传输量，从而间接提升吞吐量。当前，和 Kafka 适配最好的两个压缩算法是LZ4 和 zstd
    - 同时，由于我们的优化目标是吞吐量，最好不要设置 acks=all 以及开启重试
    - 最后，如果你在多个线程中共享一个 Producer 实例，就可能会碰到缓冲区不够用的情形。倘若频繁地遭遇 TimeoutException：Failed to allocate memory within the configured max blocking time 这样的异常，那么你就必须显式地增加buffer.memory参数值，确保缓冲区总是有空间可以申请的。
  - Consumer 端提升吞吐量的手段是有限的，你可以利用多线程方案增加整体吞吐量，也可以增加 fetch.min.bytes 参数值。默认是1 字节，表示只要 Kafka Broker 端积攒了 1 字节的数据，就可以返回给 Consumer 端，这实在是太小了。我们还是让 Broker 端一次性多返回点数据吧。
- 调优延时
  - 在 Broker 端，我们依然要增加 num.replica.fetchers 值以加快 Follower 副本的拉取速度，减少整个消息处理的延时。
  - 在 Producer 端，我们希望消息尽快地被发送出去，因此不要有过多停留，所以必须设置linger.ms=0，同时不要启用压缩。因为压缩操作本身要消耗 CPU 时间，会增加消息发送的延时。另外，最好不要设置 acks=all。我们刚刚在前面说过，Follower 副本同步往往是降低 Producer 端吞吐量和增加延时的首要原因。
  - 在 Consumer 端，我们保持 fetch.min.bytes=1 即可，也就是说，只要 Broker 端有能返回的数据，立即令其返回给 Consumer，缩短 Consumer 消费延时











