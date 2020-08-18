## Kafka术语

- Apache Kafka 是消息引擎系统，也是一个分布式流处理平台

- 生产者和消费者统称为客户端（Clients）

- Kafka 的服务器端由被称为 Broker 的服务进程构成，即一个 Kafka 集群由多个 Broker 组成，Broker 负责接收和处理客户端发送过来的请求，以及对消息进行持久化。

  - 虽然多个 Broker 进程能够运行在同一台机器上，但更常见的做法是将不同的 Broker 分散运行在不同的机器上，这样如果集群中某一台机器宕机，即使在它上面运行的所有 Broker 进程都挂掉了，其他机器上的 Broker 也依然能够对外提供服务。这其实就是 Kafka 提供高可用的手段之一。

- 实现高可用的另一个手段就是备份机制（Replication）。

  - Kafka 定义了两类副本：领导者副本（Leader Replica）和追随者副本（Follower Replica）。前者对外提供服务，这里的对外指的是与客户端程序进行交互；而后者只是被动地追随领导者副本而已，不能与外界进行交互。
  - 副本的工作机制也很简单：生产者总是向领导者副本写消息；而消费者总是从领导者副本读消息。

- Kafka 的三层消息架构

  - 第一层是主题层，每个主题可以配置 M 个分区，而每个分区又可以配置 N 个副本。
  - 第二层是分区层，每个分区的 N 个副本中只能有一个充当领导者角色，对外提供服务；其他 N-1 个副本是追随者副本，只是提供数据冗余之用
  - 第三层是消息层，分区中包含若干条消息，每条消息的位移从 0 开始，依次递增。
  - 最后，客户端程序只能与分区的领导者副本进行交互。

- Kafka Broker 是如何持久化数据的。

  - 总的来说，Kafka 使用消息日志（Log）来保存数据，一个日志就是磁盘上一个只能追加写（Append-only）消息的物理文件。因为只能追加写入，故避免了缓慢的随机 I/O 操作，改为性能较好的顺序I/O 写操作，这也是实现 Kafka 高吞吐量特性的一个重要手段。
  - 日志段（Log Segment）机制。在 Kafka 底层，一个日志又近一步细分成多个日志段，消息被追加写到当前最新的日志段中

- 为什么kafka不支持主从分离

  - 对于那种读操作很多而写操作相对不频繁的负载类型而言，采用读写分离是非常不错的方案——我们可以添加很多follower横向扩展，提升读操作性能。反观Kafka，它的主要场景还是在消息引擎而不是以数据存储的方式对外提供读服务，通常涉及频繁地生产消息和消费消息，这不属于典型的读多写少场景，因此读写分离方案在这个场景下并不太适合。

  - Kafka中的领导者副本一般均匀分布在不同的broker中，已经起到了负载的作用。即：同一个topic的已经通过分区的形式负载到不同的broker上了，读写的时候针对的领导者副本

    





## Kafka线上集群部署方案

- Kafka 客户端底层使用了 Java的 selector，selector 在 Linux 上的实现机制是 epoll。将 Kafka 部署在 Linux 上是有优势的，因为能够获得更高效的I/O 性能。
- 在 Linux 部署 Kafka 能够享受到零拷贝技术所带来的快速数据传输特性
- 假设你所在公司有个业务每天需要向Kafka 集群发送 1 亿条消息，每条消息保存两份以防止数据丢失，另外消息默认保存两周时间。现在假设消息的平均大小是 1KB，那么你能说出你的 Kafka 集群需要为这个业务预留多少磁盘空间吗？
  - 我们来计算一下：每天 1 亿条 1KB 大小的消息，保存两份且留存两周的时间，那么总的空间大小就等于 1 亿 * 1KB * 2 / 1000 / 1000 = 200GB。一般情况下 Kafka 集群除了消息数据还有其他类型的数据，比如索引数据等，故我们再为这些数据预留出 10% 的磁盘空间，因此总的存储容量就是 220GB。既然要保存两周，那么整体容量即为 220GB * 14，大约 3TB 左右。Kafka 支持数据的压缩，假设压缩比是 0.75，那么最后你需要规划的存储空间就是 0.75 * 3 = 2.25TB。
- 真正要规划的是所需的 Kafka 服务器的数量。
  - 假设你公司的机房环境是千兆网络，即 1Gbps，现在你有个业务，其业务目标或 SLA 是在 1 小时内处理 1TB 的业务数据。那么问题来了，你到底需要多少台 Kafka 服务器来完成这个业务呢？
    - 通常情况下你只能假设 Kafka 会用到 70% 的带宽资源，因为总要为其他应用或进程留一些资源
    - 这只是它能使用的最大带宽资源，你不能让 Kafka 服务器常规性使用这么多资源，故通常要再额外预留出 2/3 的资源，即单台服务器使用带宽 700Mb / 3 ≈ 240Mbps。
    - 有了 240Mbps，我们就可以计算 1 小时内处理 1TB 数据所需的服务器数量了。根据这个目标，我们每秒需要处理 2336Mb 的数据，除以 240，约等于 10 台服务器。如果消息还需要额外复制两份，那么总的服务器台数还要乘以 3，即 30 台。







## 集群参数配置

- 针对存储信息的重要参数

  - log.dirs：这是非常重要的参数，指定了 Broker 需要使用的若干个文件目录路径。要知道这个参数是没有默认值的
  - 如果有条件的话你最好保证这些目录挂载到不同的物理磁盘上。
    - 提升读写性能
    - 能够实现故障转移：即 Failover。

- ZooKeeper 相关的设置

  - 分布式协调框架，负责协调管理并保存 Kafka 集群的所有元数据信息

  - Kafka 与 ZooKeeper 相关的最重要的参数当属zookeeper.connect

  - > 如果你有两套 Kafka 集群，假设分别叫它们 kafka1 和kafka2，那么两套集群的zookeeper.connect参数可以这样指定：zk1:2181,zk2:2181,zk3:2181/kafka1和zk1:2181,zk2:2181,zk3:2181/kafka2。

- Broker 连接相关

  - 客户端程序或其他Broker 如何与该 Broker 进行通信的设置
  - 监听器的概念，从构成上来说，它是若干个逗号分隔的三元组，每个三元组的格式为<协议名称，主机名，端口号>
  - advertised.listeners
    - 你的Kafka Broker机器上配置了双网卡，一块网卡用于内网访问（即我们常说的内网IP）；另一个块用于外网访问。那么你可以配置listeners为内网IP，advertised.listeners为外网IP。

- Topic 管理

  - auto.create.topics.enable参数我建议最好设置成false，即不允许自动创建 Topic。在我们的线上环境里面有很多名字稀奇古怪的 Topic，我想大概都是因为该参数被设置成了 true 的缘故。
  - unclean.leader.election.enable：是否允许定期进行 Leader 选举。
    - 这个参数在最新版的 Kafka 中默认就是 false
  - auto.leader.rebalance.enable
    - true 表示允许 Kafka 定期地对一些 Topic 分区进行Leader 重选举，当然这个重选举不是无脑进行的，它要满足一定的条件才会发生。严格来说它与上一个参数中Leader 选举的最大不同在于，它不是选 Leader，而是换Leader
    - 这种换 Leader本质上没有任何性能收益，因此我建议你在生产环境中把这个参数设置成 false。

- 数据留存

  - log.retention.{hour|minutes|ms}：这是个“三
    兄弟”，都是控制一条消息数据被保存多长时间。从优先级上来说 ms 设置最高、minutes 次之、hour 最低。
  - log.retention.bytes：这是指定 Broker 为消息保存的总磁盘容量大小。
  - max.message.bytes：控制 Broker 能够接收的最大消息大小。
    - 默认的 1000012 太少了，还不到 1MB。实际场景中突破 1MB 的消息都是屡见不鲜的，因此在线上环境中设置一个比较大的值还是比较保险的做法。毕竟它只是一个标尺而已，仅仅衡量 Broker 能够处理的最大消息大小，即使设置大一点也不会耗费什么磁盘空间的。

- Topic 级别参数

  - Topic 级别参数会覆盖全局Broker 参数的值，而每个 Topic 都能设置自己的参数值，这就是所谓的 Topic 级别参数。
  - 保存消息
    - retention.ms：规定了该 Topic 消息被保存的时长。
    - retention.bytes：规定了要为该 Topic 预留多大的磁盘空间。

- 将你的 JVM 堆大小设置成 6GB 吧，这是目前业界比较公认的一个合理值。

- 操作系统参数

  - 文件描述符限制
    - 通常情况下将它设置成一个超大的值是合理的做法，比如ulimit -n 1000000
  - 文件系统类型
    - 生产环境最好还是使用 XFS。
  - Swappiness
    - 网上很多文章都提到设置其为 0，将
      swap 完全禁掉以防止 Kafka 进程使用 swap 空间。我个人反倒觉得还是不要设置成 0 比较好，我们可以设置成一个较小的值。为什么呢？因为一旦设置成 0，当物理内存耗尽时，操作系统会触发 OOM killer 这个组件，它会随机挑选一个进程然后 kill 掉，即根本不给用户任何的预警。
    - 但如果设置成一个比较小的值，当开始使用 swap 空间时，你至少能够观测到 Broker 性能开始出现急剧下降，从而给你进一步调优和诊断问题的时间。基于这个考虑，我个人建议将swappniess 配置成一个接近 0 但不为 0 的值，比如 1。
  - 提交时间
    - 向 Kafka 发送数据并不是真要等数据被写入磁盘才会认为成功，而是只要数据被写入到操作系统的页缓存（Page Cache）上就可以了，随后操作系统根据 LRU 算法会定期将页缓存上的“脏”数据落盘到物理磁盘上。这个定期就是由提交时间来确定的，默认是 5 秒。







## 生产者消息分区机制原理

- 其实分区的作用就是提供负载均衡的能力，或者说对数据进行分区的主要原因，就是为了实现系统的高伸缩性（Scalability）。
  - 不同的分区能够被放置到不同节点的机器上，而数据的读写操作也都是针对分区这个粒度而进行的
- 所谓分区策略是决定生产者将消息发送到哪个分区的算法。
- kafka同一个topic是无法保证数据的顺序性的，但是同一个partition中的数据是有顺序的





## 生产者压缩算法

- Kafka 的消息层次都分为两层：消息集合（message set）以及消息（message）。一个消息集合中包含若干条日志项（record item），而日志项才是真正封装消息的地方。
  - Kafka 通常不会直接操作具体的一条条消息，它总是在消息集合这个层面上进行写入操作。
  - V2 版本的做法是对整个消息集合进行压缩。
- 在 Kafka 中，压缩可能发生在两个地方：生产者端和 Broker 端。
  - 生产者程序中配置 compression.type 参数即表示启用指定类型的压缩算法
  - 其实大部分情况下 Broker 从 Producer 端接收到消息后仅仅是原封不动地保存而不会对其进行任何修改，但这里的“大部分情况”也是要满足一定条件的。有两种例外情况就可能让Broker 重新压缩消息。
    - Broker 端指定了和 Producer 端不同的压缩算法
    - Broker 端发生了消息格式转换
      - 所谓的消息格式转换主要是为了兼容老版本的消费者程序。
      - 一般情况下这种消息格式转换对性能是有很大影响的，除了这里的压缩之外，它还让 Kafka 丧失了引以为豪的 Zero Copy 特性。
  - 每个压缩过的消息集合在 Broker 端写入时都要发生解压缩操作，目的就是为了对消息执行各种验证。这和前面提到消息格式转换时发生的解压缩是不同的场景。
- 从 2.1.0 开始，Kafka 正式支持 Zstandard 算法（简写为 zstd）。它是 Facebook 开源的一个压缩算法，能够提供超高的压缩比（compression ratio）。
  - 这年头，带宽可是比CPU 和内存还要珍贵的稀缺资源，毕竟万兆网络还不是普通公司的标配，因此千兆网络中Kafka 集群带宽资源耗尽这件事情就特别容易出现。如果你的客户端机器 CPU 资源有很多富余，我强烈建议你开启 zstd 压缩，这样能极大地节省网络资源消耗。







## 无消息丢失配置

- 一句话概括，Kafka 只对“已提交”的消息（committed message）做有限度的持久化保证。

  - Kafka 不丢消息是有前提条件的。假如你的消息保存在 N 个 Kafka Broker 上，那么这个前提条件就是这 N个 Broker 中至少有 1 个存活。只要这个条件成立，Kafka 就能保证你的这条消息永远不会丢失。

- “消息丢失”案例

  - 案例 1：生产者程序丢失数据
    - 目前 Kafka Producer 是异步发送消息的，也就是说如果你调用的是 producer.send(msg)这个 API，那么它通常会立即返回，但此时你不能认为消息发送已成功完成。
    - Producer 永远要使用带有回调通知的发送 API，不要使用producer.send(msg)，而要使用 producer.send(msg, callback)。不要小瞧这里的callback（回调），它能准确地告诉你消息是否真的提交成功了。
  - 案例 2：消费者程序丢失数据
    - Consumer 端丢失数据主要体现在 Consumer 端要消费的消息不见了。
    - 要对抗这种消息丢失，办法很简单：维持先消费消息（阅读），再更新位移（书签）的顺序即可
    - 当然，这种处理方式可能带来的问题是消息的重复处理，类似于同一页书被读了很多遍，但这不属于消息丢失的情形。

- 最佳实践

  - 不要使用 producer.send(msg)，而要使用 producer.send(msg, callback)。记住，一定要使用带有回调通知的 send 方法。
  - 设置 acks = all。acks 是 Producer 的一个参数，代表了你对“已提交”消息的定义。如果设置成 all，则表明所有副本 Broker 都要接收到消息，该消息才算是“已提交”。这是最高等级的“已提交”定义。
  - 设置 retries 为一个较大的值。这里的 retries 同样是 Producer 的参数，对应前面提到的 Producer 自动重试。当出现网络的瞬时抖动时，消息发送可能会失败，此时配置了retries > 0 的 Producer 能够自动重试消息发送，避免消息丢失。
  - 设置 unclean.leader.election.enable = false。这是 Broker 端的参数，它控制的是哪些 Broker 有资格竞选分区的 Leader。如果一个 Broker 落后原先的 Leader 太多，那么它一旦成为新的 Leader，必然会造成消息的丢失。故一般都要将该参数设置成 false，即不允许这种情况的发生。
  - 设置 replication.factor >= 3。这也是 Broker 端的参数。其实这里想表述的是，最好将消息多保存几份，毕竟目前防止消息丢失的主要机制就是冗余
  - 设置 min.insync.replicas > 1。这依然是Broker 端参数，控制的是消息至少要被写入到多少个副本才算是“已提交”。设置成大于 1 可以提升消息持久性。在实际环境中千万不要使用默认值 1。
    - 推荐设置成 replication.factor = min.insync.replicas +1。
  - 确保消息消费完成再提交。Consumer 端有个参数 enable.auto.commit，最好把它设置成 false，并采用手动提交位移的方式。就像前面说的，这对于单 Consumer 多线程处理的场景而言是至关重要的

  





## 幂等生产者和事务生产者

- Kafka 消息交付可靠性保障以及精确处理一次语义的实现。

  - > 所谓的消息交付可靠性保障，是指 Kafka 对 Producer 和Consumer 要处理的消息提供什么样的承诺。
    > 	最多一次
    > 	至少一次
    > 	精确一次

- 在 Kafka 中，Producer 默认不是幂等性的，但我们可以创建幂等性 Producer。

  - > props.put(“enable.idempotence”,ture)，或props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG， true)。

  - 幂等性 Producer 的作用范围

    - 只能保证单分区上的幂等性
    - 单会话上的幂等性，不能实现跨会话的幂等性。这里的会话，你可以理解为 Producer 进程的一次运行。当你重启了Producer 进程之后，这种幂等性保证就丧失了。
    - 如果我想实现多分区以及多会话上的消息无重复，应该怎么做呢？答案就是事务（transaction）或者依赖事务型 Producer。

- 事务更多用在Kafka Streams中。如果要实现流处理中的精确一次语义，事务是不可少的。





## 消费者组

- Consumer Group 是 Kafka 提供的可扩展且具有容错性的消费者机制
  - Consumer Group 下所有实例订阅的主题的单个分区，只能分配给组内的某个 Consumer 实例消费。这个分区当然也可以被其他的 Group 消费。
  - Consumer Group 这一种机制，同时实现了传统消息引擎系统的两大模型：如果所有实例都属于同一个 Group，那么它实现的就是消息队列模型；如果所有实例分别属于不同的 Group，那么它实现的就是发布 / 订阅模型。
  - 理想情况下，Consumer 实例的数量应该等于该 Group 订阅主题的分区总数。
- 位移（Offset）
  - 对于 ConsumerGroup 而言，它是一组 KV 对，Key 是分区，V 对应Consumer 消费该分区的最新位移
  - 在新版本的 Consumer Group 中，Kafka 社区重新设计了 Consumer Group 的位移管理方式，采用了将位移保存在 Kafka 内部主题的方法。这个内部主题就是让人既爱又恨的 __consumer_offsets
- 重平衡
  - Rebalance 本质上是一种协议，规定了一个 ConsumerGroup 下的所有 Consumer 如何达成一致，来分配订阅Topic 的每个分区。
    - 比如某个 Group 下有 20 个Consumer 实例，它订阅了一个具有 100 个分区的Topic。正常情况下，Kafka 平均会为每个 Consumer 分配5 个分区。这个分配的过程就叫 Rebalance。
  - Rebalance 的触发条件有 3 个
    - 组成员数发生变更
    - 订阅主题数发生变更
    - 订阅主题的分区数发生变更
  - Rebalance 过程对 Consumer Group 消费过程有极大的影响
    - 在 Rebalance 过程中，所有 Consumer 实例都会停止消费，等待 Rebalance 完成。这是 Rebalance 为人诟病的一个方面。





## 位移主题

- 新版本 Consumer 的位移管理机制其实也很简单，就是将 Consumer 的位移数据作为一条条普通的 Kafka 消息，提交到 __consumer_offsets 中。可以这么说，\__consumer_offsets 的主要作用是保存 Kafka 消费者的位移信息。
  - 位移主题的 Key 中应该保存 3 部分内容：<GroupID，主题名，分区号 >
  - 位移主题就是普通的 Kafka 主题，那么它自然也有对应的分区数。但如果是 Kafka 自动创建的，分区数是怎么设置的呢？这就要看 Broker 端参数 offsets.topic.num.partitions 的取值了。它的默认值是 50，因此 Kafka 会自动创建一个 50 分区的位移主题。
  - 副本数，Broker 端另一个参数 offsets.topic.replication.factor 要做的事情了。它的默认值是 3。
- Kafka 是怎么删除位移主题中的过期消息的呢？答案就是 Compaction（整理）
  - 对于同一个 Key 的两条消息 M1和 M2，如果 M1 的发送时间早于 M2，那么 M1 就是过期消息。Compact 的过程就是扫描日志的所有消息，剔除那些过期的消息，然后把剩下的消息整理在一起。
  - Kafka 提供了专门的后台线程定期地巡检待 Compact 的主题，看看是否存在满足条件的可删除数据。这个后台线程叫 Log Cleaner。很多实际生产环境中都出现过位移主题无限膨胀占用过多磁盘空间的问题，如果你的环境中也有这个问题，我建议你去检查一下 LogCleaner 线程的状态，通常都是这个线程挂掉了导致的。







## 消费者组重平衡能避免吗

- 所谓协调者，在 Kafka 中对应的术语是 Coordinator，它专门为 Consumer Group 服务，负责为 Group 执行Rebalance 以及提供位移管理和组成员管理等。

- Rebalance 的弊端

  - Rebalance 影响 Consumer 端 TPS
  - Rebalance 很慢
  - Rebalance 效率不高
  - 在真实的业务场景中，很多 Rebalance 都是计划外的或者说是不必要的。

- 组成员数量变化而引发的 Rebalance 该如何避免

  - 99% 的 Rebalance，都是这个原因导致的
  - session.timout.ms 决定了 Consumer 存活性的时间间隔。
  - 除了这个参数，Consumer 还提供了一个允许你控制发送心跳请求频率的参数，就是heartbeat.interval.ms。这个值设置得越小，Consumer 实例发送心跳请求的频率就越高。
    - 目前 Coordinator 通知各个 Consumer 实例开启 Rebalance 的方法，就是将 REBALANCE_NEEDED 标志封装进心跳请求的响应体中。
  - 除了以上两个参数，Consumer 端还有一个参数，用于控制 Consumer 实际消费能力对Rebalance 的影响，即 max.poll.interval.ms 参数。它限定了 Consumer 端应用程序两次调用 poll 方法的最大时间间隔。它的默认值是 5 分钟，表示你的 Consumer 程序如果在 5分钟之内无法消费完 poll 方法返回的消息，那么 Consumer 会主动发起“离开组”的请求，Coordinator 也会开启新一轮 Rebalance。

- 明确一下到底哪些 Rebalance 是“不必要的”

  - 第一类非必要 Rebalance 是因为未能及时发送心跳，导致 Consumer 被“踢出”Group而引发的。

    - > 设置 session.timeout.ms = 6s。
      > 设置 heartbeat.interval.ms = 2s。
      > 要保证 Consumer 实例在被判定为“dead”之前，能够发送至少 3 轮的心跳请求，即session.timeout.ms >= 3 * heartbeat.interval.ms。

  - 第二类非必要 Rebalance 是 Consumer 消费时间过长导致的。

  - 如果你按照上面的推荐数值恰当地设置了这几个参数，却发现还是出现了 Rebalance，那么我建议你去排查一下Consumer 端的 GC 表现，比如是否出现了频繁的 Full GC 导致的长时间停顿，从而引发了 Rebalance。





## 位移提交

- 从用户的角度来说，位移提交分为自动提交和手动提交；从 Consumer 端的角度来说，位移提交分为同步提交和异步提交
  - 手动提交
    - 最简单的 API 就是KafkaConsumer#commitSync()。该方法会提交KafkaConsumer#poll() 返回的最新位移。从名字上来看，它是一个同步操作
    - 可见，调用 consumer.commitSync() 方法的时机，是在你处理完了 poll() 方法返回的所有消息之后。
    - Kafka 社区为手动提交位移提供了另一个 API 方法：KafkaConsumer#commitAsync()。由于它是异步的，Kafka 提供了回调函数（callback）	
    - commitAsync 是否能够替代 commitSync 呢？答案是不能。commitAsync 的问题在于，出现问题时它不会自动重试。因为它是异步操作，倘若提交失败后自动重试，那么它重试时提交的位移值可能早已经“过期”或不是最新值了。
- 直接提交最新一条消息的位移
  - 设想这样一个场景：你的 poll 方法返回的不是 500 条消息，而是 5000 条。那么，你肯定不想把这 5000 条消息都处理完之后再提交位移，因为一旦中间出现差错，之前处理的全部都要重来一遍。这类似于我们数据库中的事务处理。很多时候，我们希望将一个大事务分割成若干个小事务分别提交，这能够有效减少错误恢复的时间。
  - 在 Kafka 中也是相同的道理。对于一次要处理很多消息的 Consumer 而言，它会关心社区有没有方法允许它在消费的中间进行位移提交。比如前面这个 5000 条消息的例子，你可能希望每处理完 100 条消息就提交一次位移，这样能够避免大批量的消息重新消费。
  - 庆幸的是，Kafka Consumer API 为手动提交提供了这样的方法：commitSync(Map<TopicPartition, OffsetAndMetadata>) 和commitAsync(Map<TopicPartition, OffsetAndMetadata>)。它们的参数是一个 Map对象，键就是 TopicPartition，即消费的分区，而值是一个 OffsetAndMetadata 对象，
    保存的主要是位移数据。





## CommitFailedException异常

- 所谓 CommitFailedException，顾名思义就是 Consumer 客户端在提交位移时出现了错误或异常，而且还是那种不可恢复的严重异常。
- 注释
  - 本次提交位移失败了，原因是消费者组已经开启了 Rebalance过程，并且将要提交位移的分区分配给了另一个消费者实例。出现这个情况的原因是，你的消费者实例连续两次调用 poll 方法的时间间隔超过了期望的 max.poll.interval.ms 参数
    值。这通常表明，你的消费者实例花费了太长的时间进行消息处理，耽误了调用 poll 方法。
  - 两个相应的解决办法
    - 增加期望的时间间隔 max.poll.interval.ms 参数值。
      - max.poll.interval.ms是指两次poll()的最大间隔时间
    - 减少 poll 方法一次性返回的消息数量，即减少 max.poll.records 参数值。
- 当消息处理的总时间超过预设的 max.poll.interval.ms 参数值
  - 你需要简化你的消息处理逻辑。具体来说有 4 种方法。
    - 缩短单条消息处理的时间。
    - 增加 Consumer 端允许下游系统消费一批消息的最大时长。
    - 减少下游系统一次性消费的消息总数
    - 下游系统使用多线程来加速消费





## 多线程开发消费者

- 从 Kafka 0.10.1.0 版本开始，KafkaConsumer 就变为了双线程的设计，即用户主线程和心跳线程。
- KafkaConsumer 类不是线程安全的 (thread-safe)。
  - 所有的网络I/O 处理都是发生在用户主线程中，因此，你在使用过程中必须要确保线程安全。简单来说，就是你不能在多个线程中共享同一个 KafkaConsumer 实例，否则程序会抛出ConcurrentModificationException 异常。
- 鉴于 KafkaConsumer 不是线程安全的事实，我们能够制定两套多线程方案。
  - 消费者程序启动多个线程，每个线程维护专属的 KafkaConsumer 实例，负责完整的消息获取、消息处理流程。
    - 由于每个线程使用专属的 KafkaConsumer 实例来执行消息获取和消息处理逻辑，因此，Kafka 主题中的每个分区都能保证只被一个线程处理，这样就很容易实现分区内的消息消费顺序
    - 每个线程完整地执行消息获取和消息处理逻辑。一旦消息处理逻辑很重，造成消息处理速度慢，就很容易出现不必要的 Rebalance，从而引发整个消费者组的消费停滞。
  - 消费者程序使用单或多线程获取消息，同时创建多个消费线程执行消息处理逻辑
    - 获取消息的线程可以是一个，也可以是多个，每个线程维护专属的 KafkaConsumer 实例，处理消息则交由特定的线程池来做，从而实现消息获取与消息处理的真正解耦。
    - 方案 2 引入了多组线程，使得整个消息消费链路被拉长，最终导致正确位移提交会变得异常困难，结果就是可能会出现消息的重复消费。





## 消费者组消费进度监控

- 对于 Kafka 消费者来说，最重要的事情就是监控它们的消费进度了，或者说是监控它们消费的滞后程度。这个滞后程度有个专门的名称：消费者 Lag 或 Consumer Lag。
  - Kafka 监控 Lag 的层级是在分区上的
  - 更可怕的是，由于消费者的速度无法匹及生产者的速度，极有可能导致它消费的数据已经不在操作系统的页缓存中了，那么这些数据就会失去享有 Zero Copy 技术的资格。这样的话，消费者就不得不从磁盘上读取它们
- 使用 Kafka 自带的命令行工具 kafka-consumer-groups 脚本。
- 使用 Kafka Java Consumer API 编程。
- 使用 Kafka 自带的 JMX 监控指标。
  - records-lag-max 和 records-lead-min
    - 这里的 Lead 值是指消费者最新消费消息的位移与分区当前第一条消息位移的差值
    - Lag 越大的话，Lead 就越小，反之也是同理
    - 一旦你监测到 Lead 越来越小，甚至是快接近于 0 了，你就一定要小心了，这可能预示着消费者端要丢消息了。因为已经逼近最早的消息了



## 副本机制

- 副本机制有什么好处
  - 提供数据冗余
    - 即使系统部分组件失效，系统依然能够继续运转，因而增加了整体可用性以及数据持久性
  - 提供高伸缩性
    - 支持横向扩展，能够通过增加机器的方式来提升读性能，进而提高读操作吞吐量
  - 改善数据局部性
    - 允许将数据放入与用户地理位置相近的地方，从而降低系统延时。
  - 对于 ApacheKafka 而言，目前只能享受到副本机制带来的第 1 个好处
- 所谓副本（Replica），本质就是一个只能追加写消息的提交日志
  - 根据 Kafka 副本机制的定义，同一个分区下的所有副本保存有相同的消息序列，这些副本分散保存在不同的Broker 上，从而能够对抗部分 Broker 宕机带来的数据不可用。
- 基于领导者（Leader-based）的副本机制
  - 在 Kafka 中，追随者副本是不对外提供服务的。追随者副本不处理客户端请求，它唯一的任务就是从领导者副本异步拉取消息，并写入到自己的提交日志中，从而实现与领导者副本的同步。
  - 方便实现“Read-your-writes”。
    - 当你使用生产者 API 向 Kafka 成功写入消息后，马上使用消费者 API 去读取刚才生产的消息。
    - 如果允许追随者副本对外提供服务，由于副本同步是异步的，因此有可能出现追随者副本还没有从领导者副本那里拉取到最新的消息，从而使得客户端看不到最新写入的消息。
  - 方便实现单调读（Monotonic Reads）
    - 如果允许追随者副本提供读服务，那么假设当前有 2 个追随者副本 F1 和 F2，它们异步地拉取领导者副本数据。倘若 F1 拉取了 Leader 的最新消息而 F2 还未及时拉取，那么，此时如果有一个消费者先从 F1 读取消息之后又从 F2 拉取消息，它可能会看到这样的现象：第一次消费时看到的最新消息在第二次消费时不见了，这就不是单调读一致性。
- In-sync Replicas（ISR）
  - Kafka 要明确地告诉我们，追随者副本到底在什么条件下才算与 Leader 同步
    - ISR 是一个动态调整的集合，而非静态不变的。
    - ISR 中的副本都是与 Leader 同步的副本，相反，不在 ISR 中的追随者副本就被认为是与 Leader 不同步的。
  - 能够进入到 ISR 的追随者副本要满足一定的条件，这个标准就是 Broker 端参数 replica.lag.time.max.ms 参数值。
    - 这个参数的含义是Follower 副本能够落后 Leader 副本的最长时间间隔，当前默认值是 10 秒。这就是说，只要一个 Follower 副本落后 Leader 副本的时间不连续超过 10 秒，那么 Kafka 就认为该Follower 副本与 Leader 是同步的，即使此时 Follower 副本中保存的消息明显少于Leader 副本中的消息
    - Kafka在启动的时候会开启两个任务，一个任务用来定期地检查是否需要缩减或者扩大ISR集合，这个周期是replica.lag.time.max.ms的一半，默认5000ms。当检测到ISR集合中有失效副本时，就会收缩ISR集合，当检查到有Follower的HighWatermark追赶上Leader时，就会扩充ISR。
    - 除此之外，当ISR集合发生变更的时候还会将变更后的记录缓存到isrChangeSet中，另外一个任务会周期性地检查这个Set,如果发现这个Set中有ISR集合的变更记录，那么它会在zk中持久化一个节点。然后因为Controllr在这个节点的路径上注册了一个Watcher，所以它就能够感知到ISR的变化，并向它所管理的broker发送更新元数据的请求。最后删除该路径下已经处理过的节点。
- producer生产消息ack=all的时候，消息是怎么保证到follower的，因为看到follower是异步拉取数据的
  - 通过HW（ High watermark）机制。leader处的HW要等所有follower LEO（Log end offset）都越过了才会前移
    - 一个分区有3个副本，一个leader，2个follower。producer向leader写了10条消息，follower1从leader处拷贝了5条消息，follower2从leader处拷贝了3条消息，那么leader副本的LEO就是10，HW=3；follower1副本的LEO是5。
- 假设一个分区有5个副本，Broker的min.insync.replicas设置为2，生产者设置acks=all，这时是有2个副本同步了就可以，还是必须是5个副本都同步，他们是什么关系。
  - min.insync.replicas是保证下限的
  - acks=all保证ISR中的所有副本都要同步
  - Kafka只对已提交消息做持久化保证。如果你设置了最高等级的持久化需求（比如acks=all），那么follower副本没有同步完成前这条消息就不算已提交，也就不算丢失了。





## 请求是怎么被处理的

- Apache Kafka 自己定义了一组请求协议，用于实现各种各样的交互操作。
  - 比如常见的PRODUCE 请求是用于生产消息的，FETCH 请求是用于消费消息的，METADATA 请求是用于请求 Kafka 集群元数据信息的。
  - 所有的请求都是通过 TCP 网络以 Socket 的方式进行通讯的
- Kafka Broker 端处理请求的全流程
  - Kafka 使用的是Reactor 模式
    - Reactor 模式是事件驱动架构的一种实现方式，特别适合应用于处理多个客户端并发向服务器端发送请求的场景
  - 多个客户端会发送请求给到 Reactor。Reactor 有个请求分发线程 Dispatcher，也就是 Acceptor，它会将不同的请求下发到多个工作线程中处理。
    - Acceptor 线程只是用于请求分发，不涉及具体的逻辑处理，非常得轻量级，因此有很高的吞吐量表现
    - Kafka 的 Broker 端有个 SocketServer 组件，类似于Reactor 模式中的 Dispatcher，它也有对应的 Acceptor 线程和一个工作线程池，只不过在 Kafka 中，这个工作线程池有个专属的名字，叫网络线程池。
    - Kafka 提供了 Broker 端参数 num.network.threads，用于调整该网络线程池的线程数。其默认值是 3，表示每台Broker 启动时会创建 3 个网络线程，专门处理客户端发送的请求。
  - 当网络线程拿到请求后，它不是自己处理，而是将请求放入到一个共享请求队列中。
    - Broker 端还有个 IO 线程池，负责从该队列中取出请求，执行真正的处理。
    - Broker 端参数num.io.threads控制了这个线程池中的线程数。目前该参数默认值是 8，表示每台 Broker 启动后自动创建 8 个 IO线程处理请求。
  - 请求队列是所有网络线程共享的，而响应队列则是每个网络线程专属的。
    - Dispatcher 只是用于请求分发而不负责响应回传，因此只能让每个网络线程自己发送 Response 给客户端，所以这些Response 也就没必要放在一个公共的地方。
  - Purgatory 组件用来缓存延时请求
    - 所谓延时请求，就是那些一时未满足条件不能立刻处理的请求。
    - 比如设置了 acks=all 的 PRODUCE 请求，一旦设置了acks=all，那么该请求就必须等待 ISR 中所有副本都接收了消息后才能返回，此时处理该请求的 IO 线程就必须等待其他 Broker 的写入结果。
    - 当请求不能立刻处理时，它就会暂存在 Purgatory 中。稍后一旦满足了完成条件，IO 线程会继续处理该请求，并将 Response放入对应网络线程的响应队列中。
- 在 Kafka 内部，除了客户端发送的 PRODUCE 请求和FETCH 请求之外，还有很多执行其他操作的请求类型。这些请求有个明显的不同：它们不是数据类的请求，而是控制类的请求。也就是说，它们并不是操作消息数据的，而是用来执行特定的Kafka 内部动作的。
  - 数据类请求和控制类请求的分离
  - Kafka Broker 启动后，会在后台分别创建网络线程池和 IO 线程池，它们分别处理数据类请求和控制类请求。至于所用的Socket 端口，自然是使用不同的端口了，你需要提供不同的listeners 配置，显式地指定哪套端口用于处理哪类请求。





## 消费者组重平衡全流程

- 重平衡过程是如何通知到其他消费者实例的？答案就是，靠消费者端的心跳线程
  - 当协调者决定开启新一轮重平衡后，它会将“REBALANCE_IN_PROGRESS”封装进心跳请求的响应中，发还给消费者实例。
  - 参数 heartbeat.interval.ms 的真实用途，从字面上看，它就是设置了心跳的间隔时间，但这个参数的真正作用是控制重平衡通知的频率。
- 消费者组状态机
  - Kafka 为消费者组定义了 5 种状态，它们分别是：Empty、Dead、PreparingRebalance、CompletingRebalance 和 Stable。
  - 一个消费者组最开始是 Empty 状态，当重平衡过程开启后，它会被置于 PreparingRebalance 状态等待成员加入，之后变更到CompletingRebalance 状态等待分配方案，最后流转到 Stable 状态完成重平衡。
  - Kafka 定期自动删除过期位移的条件就是，组要处于 Empty 状态。
- 消费者端重平衡流程
  - 在消费者端，重平衡分为两个步骤：分别是加入组和等待领导者消费者分配方案。这两个步骤分别对应两类特定的请求：JoinGroup 请求和SyncGroup 请求。
    - 领导者消费者的任务是收集所有成员的订阅信息，然后根据这些信息，制定具体的分区消费分配方案。
    - 领导者向协调者发送 SyncGroup 请求，将刚刚做出的分配方案发给协调者。其他成员也会向协调者发送 SyncGroup 请求，只不过请求体中并没有实际的内容。这一步的主要目的是让协调者接收分配方案，然后统一以 SyncGroup 响应的方式分发给所有成员，这样组内所有成员就都知道自己该消费哪些分区了







## Kafka控制器

- 控制器组件（Controller），是 Apache Kafka 的核心组件。它的主要作用是在 ApacheZooKeeper 的帮助下管理和协调整个 Kafka 集群。
  - 控制器是重度依赖ZooKeeper 的
    - Apache ZooKeeper 是一个提供高可靠性的分布式协调服务框架。它使用的数据模型类似于文件系统的树形结构
    - 如果以 znode 持久性来划分，znode 可分为持久性 znode 和临时 znode
    - ZooKeeper 赋予客户端监控 znode 变更的能力，即所谓的 Watch 通知功能。
    - 依托于这些功能，ZooKeeper 常被用来实现集群成员管理、分布式锁、领导者选举等功能
  - 在运行过程中，只能有一个 Broker 成为控制器
- 控制器是做什么的
  - 主题管理（创建、删除、增加分区）
  - 分区重分配
  - Preferred 领导者选举
  - 集群成员管理（新增 Broker、Broker 主动关闭、Broker 宕机）
  - 数据服务
    - 控制器的最后一大类工作，就是向其他 Broker 提供数据服务。控制器上保存了最全的集群元数据信息，其他所有 Broker 会定期接收控制器发来的元数据更新请求，从而更新其内存中的缓存数据。
- 当你觉得控制器组件出现问题时，比如主题无法删除了，或者重分区 hang 住了，你不用重启Kafka Broker 或控制器。
  - 有一个简单快速的方式是，去 ZooKeeper 中手动删除/controller 节点。具体命令是 rmr /controller。这样做的好处是，既可以引发控制器的重选举，又可以避免重启 Broker 导致的消息处理中断。
  - 如何看出重分区被hang住了
    - 执行reassign命令总提示分区在reassign中，或者ZooKeeper中的/admin/reassign_partitions下相应节点未被删除





## 高水位和Leader Epoch

- 在 Kafka 的世界中，水位的概念有一点不同。Kafka 的水位不是时间戳，更与时间无关。它是和位置信息绑定的，具体来说，它是用消息位移来表征的。

- 在 Kafka 中，高水位的作用主要有 2 个

  - 定义消息可见性，即用来标识分区下的哪些消息是可以被消费者消费的。 
    - 在分区高水位以下的消息被认为是已提交消息，反之就是未提交消息。消费者只能消费已提交消息
  - 帮助 Kafka 完成副本同步

- Broker 0 上保存了某分区的 Leader 副本和所有 Follower 副 本的 LEO 值，而 Broker 1 上仅仅保存了该分区的某个 Follower 副本。Kafka 把 Broker 0上保存的这些 Follower 副本又称为**远程副本**

- 高水位和 LEO 是副本对象的两个重要属性

  - 日志末端位移的概念，即 Log End Offset，简写是 LEO。它表示副本写入下一条消息的位移值
  - 分区的高水位就是其 Leader 副本的高水位

- 高水位更新机制

  - 使用高水位、LEO来执行副本消息同步

    - > Follow副本LEO、高水位
      > Leader副本LEO、高水位
      > min(LEO,高水位)

  - Follow副本从Leader副本拉取消息或Leader副本接收到生产者发送的消息会更新LEO值

  - 与 Leader 副本保持同步。判断的条件有两个

    - 该远程 Follower 副本在 ISR 中。
    - 该远程 Follower 副本 LEO 值落后于 Leader 副本 LEO 值的时间，不超过 Broker 端参数 replica.lag.time.max.ms 的值。如果使用默认值的话，就是不超过 10 秒。

  - Leader 副本

    > 处理生产者请求的逻辑如下：
    >
    > 1. 写入消息到本地磁盘。
    > 2. 更新分区高水位值。
    >    i. 获取 Leader 副本所在 Broker 端保存的所有远程副本 LEO 值{LEO-1，LEO-2，……，LEO-n}。
    >    ii. 获取 Leader 副本高水位值：currentHW。
    >    iii. 更新 currentHW = max{currentHW,min(LEO-1，LEO-2，……，LEO-n)}。

> 
>
> 处理 Follower 副本拉取消息的逻辑如下：
>
> 1. 读取磁盘（或页缓存）中的消息数据。
> 2. 使用 Follower 副本发送请求中的位移值更新远程副本 LEO 值。
> 3. 更新分区高水位值（具体步骤与处理生产者请求的步骤相同）。

- Follower 副本

  > 从 Leader 拉取消息的处理逻辑如下： 
  >
  > 1. 写入消息到本地磁盘。 
  > 2. 更新 LEO 值。 
  > 3. 更新高水位值。 
  >
  > i. 获取 Leader 发送的高水位值：currentHW。 
  >
  > ii. 获取步骤 2 中更新过的 LEO 值：currentLEO。 
  >
  > iii. 更新高水位为 min(currentHW, currentLEO)。

- 副本同步机制

  - 使用一个单分区且有两个副本的主题
  - 当生产者给主题分区发送一条消息后，Leader 副本成功将消息写入了本地磁盘，故 LEO 值被更新为 1。
  - Follower 再次尝试从 Leader 拉取消息，这时，Follower 副本也成功地更新 LEO 为 1。此时，Leader 和 Follower 副本的 LEO 都是 1，但各自的高水位依然是 0，还没有被更新。它们需要在下一轮的拉取中被更新
  - 在新一轮的拉取请求中，由于位移值是 0 的消息已经拉取成功，因此 Follower 副本这次请求拉取的是位移值 =1 的消息。Leader 副本接收到此请求后，更新远程副本 LEO 为 1，然后更新 Leader 高水位为 1
  - 做完这些之后，它会将当前已更新过的高水位值 1 发送给Follower 副本。Follower 副本接收到以后，也将自己的高水位值更新成 1。至此，一次完整的消息同步周期就结束了。事实上，Kafka 就是利用这样的机制，实现了 Leader 和Follower 副本之间的同步。

- Leader Epoch

  - 从刚才的分析中，我们知道，Follower 副本的高水位更新需要一轮额外的拉取请求才能实 现。如果把上面那个例子扩展到多个 Follower 副本，情况可能更糟，也许需要多轮拉取请求。也就是说，Leader 副本高水位更新和 Follower 副本高水位更新在时间上是存在错配的。这种错配是很多“数据丢失”或“数据不一致”问题的根源。
  - 所谓 Leader Epoch，我们大致可以认为是 Leader 版本。它由两部分数据组成
    - Epoch。一个单调增加的版本号。每当副本领导权发生变更时，都会增加该版本号。小版本号的 Leader 被认为是过期 Leader，不能再行使 Leader 权力。 
    - 起始位移（Start Offset）。Leader 副本在该 Epoch 值上写入的首条消息的位移
  - 开始时，副本 A 和副本 B 都处 于正常状态，A 是 Leader 副本。某个使用了默认 acks 设置的生产者程序向 A 发送了两条消息，A 全部写入成功，此时 Kafka 会通知生产者说两条消息全部发送成功。
    - 现在我们假设 Leader 和 Follower 都写入了这两条消息，而且 Leader 副本的高水位也已经更新了，但 Follower 副本高水位还未更新——这是可能出现的。还记得吧，Follower端高水位的更新与 Leader 端有时间错配。倘若此时副本 B 所在的 Broker 宕机，当它重启回来后，副本 B 会执行日志截断操作，将 LEO 值调整为之前的高水位值，也就是 1。这就是说，位移值为 1 的那条消息被副本 B 从磁盘中删除，此时副本 B 的底层磁盘文件中只保存有 1 条消息，即位移值为 0 的那条消息。
    - 当执行完截断操作后，副本 B 开始从 A 拉取消息，执行正常的消息同步。如果就在这个节骨眼上，副本 A 所在的 Broker 宕机了，那么 Kafka 就别无选择，只能让副本 B 成为新的Leader，此时，当 A 回来后，需要执行相同的日志截断操作，即将高水位调整为与 B 相同的值，也就是 1。这样操作之后，位移值为 1 的那条消息就从这两个副本中被永远地抹掉了。这就是这张图要展示的数据丢失场景。
    - 严格来说，这个场景发生的前提是Broker 端参数 min.insync.replicas 设置为 1。
  - 利用 Leader Epoch 机制来规避这种数据丢失
    - 场景和之前大致是类似的，只不过引用 Leader Epoch 机制后，Follower 副本 B 重启回来后，需要向 A 发送一个特殊的请求去获取 Leader 的 LEO 值。在这个例子中，该值为 2。 当获知到 Leader LEO=2 后，B 发现该 LEO 值不比它自己的 LEO 值小，而且缓存中也没 有保存任何起始位移值 > 2 的 Epoch 条目，因此 B 无需执行任何日志截断操作。这是对高水位机制的一个明显改进，即副本是否执行日志截断不再依赖于高水位进行判断。
    - 现在，副本 A 宕机了，B 成为 Leader。同样地，当 A 重启回来后，执行与 B 相同的逻辑判断，发现也不用执行日志截断，至此位移值为 1 的那条消息在两个副本中均得到保留。后面当生产者程序向 B 写入新消息时，副本 B 所在的 Broker 缓存中，会生成新的 LeaderEpoch 条目：[Epoch=1, Offset=2]。之后，副本 B 会使用这个条目帮助判断后续是否执行日志截断操作。这样，通过 Leader Epoch 机制，Kafka 完美地规避了这种数据丢失场景。 









## 主题管理

- Kafka 提供了自带的 kafka-topics 脚本，用于帮助用户创建主题。

  - create 表明我们要创建主题，而 partitions 和 replication factor 分别设置了主题的分区数以及每个分区下的副本数

  - > bin/kafka-topics.sh --bootstrap-server broker_host:port --create --topic my_topic_name  --partitions 1 --replication-factor 1

  - 社区推荐使用 --bootstrap-server 而非 --zookeeper 的原因主要有两个

    - 使用 --zookeeper 会绕过 Kafka 的安全体系
    - 使用 --bootstrap-server 与集群进行交互，越来越成为使用 Kafka 的标准姿势

- 查询主题

  - > bin/kafka-topics.sh --bootstrap-server broker_host:port --list
    >
    > bin/kafka-topics.sh --bootstrap-server broker_host:port --describe --topic <topic_name>

- 修改主题分区

  - > 其实就是增加分区，目前 Kafka 不允许减少某个主题的分区数。
    >
    > bin/kafka-topics.sh --bootstrap-server broker_host:port --alter --topic <topic_name> --partitions < 新分区数 >

- 修改主题级别参数

  - > 在主题创建之后，我们可以使用 kafka-configs 脚本修改对应的参数。
    >
    > 假设我们要设置主题级别参数 max.message.bytes，那么命令如下：
    >
    > bin/kafka-configs.sh --zookeeper zookeeper_host:port --entity-type topics --entity-name <topic_name> --alter --add-config max.message.bytes=10485760
    >
    > 其实，这个脚本也能指定 --bootstrap-server 参数，只是它是用来设置动态参数的

- 变更副本数

  - 使用自带的 kafka-reassign-partitions 脚本，帮助我们增加主题的副本数。

  - > 假设kafka的内部主题 \__consumer_offsets 只有 1 个副本，现在我们想要增加至 3 个副本_
    >
    > 
    >
    > 创建一个 json 文件，显式提供 50 个分区对应的副本数。注意，replicas 中的 3 台 Broker 排列顺序不同，目的是将 Leader 副本均匀地分散在 Broker 上。
    >
    > 
    >
    > {"version":1, "partitions":[
    > {"topic":"\__consumer_offsets","partition":0,"replicas":[0,1,2]}, 
    > {"topic":"\__consumer_offsets","partition":1,"replicas":[0,2,1]},
    > {"topic":"\__consumer_offsets","partition":2,"replicas":[1,0,2]},
    > {"topic":"\__consumer_offsets","partition":3,"replicas":[1,2,0]},
    > ...
    > {"topic":"__consumer_offsets","partition":49,"replicas":[0,1,2]}
    > ]}
    >
    > ​	
    >
    > bin/kafka-reassign-partitions.sh --zookeeper zookeeper_host:port --reassignment-json-file reassign.json --execute

- 修改主题限速

  - 这里主要是指设置 Leader 副本和 Follower 副本使用的带宽

  - > 假设我有个主题，名为 test，我想让该主题各个分区的 Leader 副本和 Follower 副本在处理副本同步时，不得占用超过 100MBps 的带宽。注意是大写 B，即每秒不超过 100MB。
    >
    > 
    >
    > bin/kafka-configs.sh --zookeeper zookeeper_host:port --alter --add-config 'leader.replication.throttled.rate=104857600,follower.replication.throttled.rate=104857600' --entity-type brokers --entity-name 0
    >
    > 
    >
    > 这条命令结尾处的 --entity-name 就是 Broker ID。倘若该主题的副本分别在 0、1、2、3 多个 Broker 上，那么你还要依次为 Broker 1、2、3 执行这条命令。
    >
    > 
    >
    > 设置好这个参数之后，我们还需要为该主题设置要限速的副本。在这个例子中，我们想要为所有副本都设置限速，因此统一使用通配符 * 来表示
    >
    > 
    >
    > bin/kafka-configs.sh --zookeeper zookeeper_host:port --alter --add-config 'leader.replication.throttled.replicas=*,follower.replication.throttled.replicas=*' --entity-type topics --entity-name test

- 删除主题

  - > bin/kafka-topics.sh --bootstrap-server broker_host:port --delete  --topic <topic_name>
    >
    > 删除主题的命令并不复杂，关键是删除操作是异步的，执行完这条命令不代表主题立即就被删除了。它仅仅是被标记成 “已删除” 状态而已。Kafka 会在后台默默地开启主题删除操作。因此，通常情况下，你都需要耐心地等待一段时间。

- __consumer_offsets 的副本数问题

- 0.11 之后，Kafka 会严格遵守offsets.topic.replication.factor 值。如果当前运行的 Broker 数量小于offsets.topic.replication.factor 值，Kafka 会创建主题失败，并显式抛出异常

- __consumer_offsets 占用太多的磁盘

  - 一旦你发现这个主题消耗了过多的磁盘空间，那么，你一定要显式地用jstack 命令查看一下 kafka-log-cleaner-thread 前缀的线程状态。通常情况下，这都是因为该线程挂掉了，无法及时清理此内部主题。
  - 倘若真是这个原因导致的，那我们就只能重启相应的 Broker了。另外，请你注意保留出错日志，因为这通常都是 Bug 导致的，最好提交到社区看一下。





## 动态配置

- 社区于 1.1.0 版本中正式引入了动态 Broker 参数

  - 如果你打开 1.1 版本之后（含 1.1）的 Kafka 官网，你会发现Broker Configs表中增加了Dynamic Update Mode 列。该列有 3 类值，分别是 read-only、per-broker 和 cluster-wide。
    - read-only。被标记为 read-only 的参数和原来的参数行为一样，只有重启 Broker，才能令修改生效。
    - per-broker。被标记为 per-broker 的参数属于动态参数，修改它之后，只会在对应的Broker 上生效。
    - cluster-wide。被标记为 cluster-wide 的参数也属于动态参数，修改它之后，会在整个集群范围内生效，也就是说，对所有 Broker 都生效。

- 动态 Broker 参数的使用场景非常广泛，通常包括但不限于以下几种

  - 动态调整 Broker 端各种线程池大小，实时应对突发流量。
  - 动态调整 Broker 端连接信息或安全配置信息。
  - 动态更新 SSL Keystore 有效期。
  - 动态调整 Broker 端 Compact 操作性能。
  - 实时变更 JMX 指标收集器 

- 目前，设置动态参数的工具行命令只有一个，那就是 Kafka 自带的 kafka-configs 脚本

  - 如果要设置 cluster-wide 范围的动态参数，需要显式指定 entity-default。

    - > $ bin/kafka-configs.sh --bootstrap-server kafka-host:port --entity-type brokers --entity-default --alter --add-config unclean.leader.election.enable=true

  - 设置 per-broker 范围参数，为 ID 为 1 的 Broker 设置一个不同的值。

    - > $ bin/kafka-configs.sh --bootstrap-server kafka-host:port --entity-type brokers --entity-name 1 --alter --add-config unclean.leader.election.enable=false

  - 删除 cluster-wide 范围参数或 per-broker 范围参数

  - > 删除动态参数要指定 delete-config
    >
    > 
    >
    > ​	$ bin/kafka-configs.sh --bootstrap-server kafka-host:port --entity-type brokers --entity-default --alter --delete-config unclean.leader.election.enable
    >
    > 
    >
    > ​	$ bin/kafka-configs.sh --bootstrap-server kafka-host:port --entity-type brokers --entity-name 1 --alter --delete-config unclean.leader.election.enable







## 重设消费者组位移

- 像 RabbitMQ 或 ActiveMQ 这样的传统消息中间件，它们处理和响应消息的方式是破坏性的（destructive），即一旦消息被成功处理，就会被从 Broker 上删除。
  - 反观 Kafka，由于它是基于日志结构（log-based）的消息引擎，消费者在消费消息时，仅仅是从磁盘文件上读取数据而已，是只读的操作，因此消费者不会删除消息数据。
  - 同时，由于位移数据是由消费者控制的，因此它能够很容易地修改位移的值，实现重复消费历史数据的功能
  - 如果在你的场景中，消息处理逻辑非常复杂，处理代价很高，同时你又不关心消息之间的顺序，那么传统的消息中间件是比较合适的；反之，如果你的场景需要较高的吞吐量，但每条消息的处理时间很短，同时你又很在意消息的顺序，此时，Kafka 就是你的首选
- 重设位移策略
  - 位移维度
    - Earliest 策略表示将位移调整到主题当前最早位移处。如果你想要重新消费主题的所有消息，那么可以使用 Earliest 策略
    - Latest 策略表示把位移重设成最新末端位移。如果你想跳过所有历史消息，打算从最新的消息处开始消费的话，可以使用 Latest 策略。
    - Current 策略表示将位移调整成消费者当前提交的最新位移。有时候你可能会碰到这样的场景：你修改了消费者程序代码，并重启了消费者，结果发现代码有问题，你需要回滚之前的代码变更，同时也要把位移重设到消费者重启时的位置，那么，Current 策略就可以帮你实现这个功能。
    - Specified-Offset 策略则是比较通用的策略，表示消费者把位移值调整到你指定的位移处。这个策略的典型使用场景是，消费者程序在处理某条错误消息时，你可以手动地“跳过”此消息的处理。
    - 如果说 Specified-Offset 策略要求你指定位移的绝对数值的话，那么 Shift-By-N 策略指定的就是位移的相对数值，即你给出要跳过的一段消息的距离即可。这里的“跳”是双向的，你既可以向前“跳”，也可以向后“跳”。
  - 时间维度
    - 我们可以给定一个时间，让消费者把位移调整成大于该时间的最小位移；也可以给出一段时间间隔，比如 30 分钟前，然后让消费者直接将位移调回 30 分钟之前的位移值。
      - DateTime 允许你指定一个时间，然后将位移重置到该时间之后的最早位移处
      - Duration 策略则是指给定相对的时间间隔，然后将位移调整到距离当前给定时间间隔的位移处，具体格式是 PnDTnHnMnS。



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

































