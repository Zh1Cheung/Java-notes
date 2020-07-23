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
    -  该远程 Follower 副本 LEO 值落后于 Leader 副本 LEO 值的时间，不超过 Broker 端参数 replica.lag.time.max.ms 的值。如果使用默认值的话，就是不超过 10 秒。

  - Leader 副本

    > 处理生产者请求的逻辑如下：
    >
    > 1. 写入消息到本地磁盘。
    > 2. 更新分区高水位值。
    > i. 获取 Leader 副本所在 Broker 端保存的所有远程副本 LEO 值{LEO-1，LEO-2，……，LEO-n}。
    > ii. 获取 Leader 副本高水位值：currentHW。
    > iii. 更新 currentHW = max{currentHW,min(LEO-1，LEO-2，……，LEO-n)}。
>
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
    -  严格来说，这个场景发生的前提是Broker 端参数 min.insync.replicas 设置为 1。
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
    >
    > 创建一个 json 文件，显式提供 50 个分区对应的副本数。注意，replicas 中的 3 台 Broker 排列顺序不同，目的是将 Leader 副本均匀地分散在 Broker 上。
    >
    > 
    >
    > {"version":1, "partitions":[
    >  {"topic":"\__consumer_offsets","partition":0,"replicas":[0,1,2]}, 
    >   {"topic":"\__consumer_offsets","partition":1,"replicas":[0,2,1]},
    >   {"topic":"\__consumer_offsets","partition":2,"replicas":[1,0,2]},
    >   {"topic":"\__consumer_offsets","partition":3,"replicas":[1,2,0]},
    >   ...
    >   {"topic":"__consumer_offsets","partition":49,"replicas":[0,1,2]}
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

-  __consumer_offsets 的副本数问题
  
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
    -  Specified-Offset 策略则是比较通用的策略，表示消费者把位移值调整到你指定的位移处。这个策略的典型使用场景是，消费者程序在处理某条错误消息时，你可以手动地“跳过”此消息的处理。
    - 如果说 Specified-Offset 策略要求你指定位移的绝对数值的话，那么 Shift-By-N 策略指定的就是位移的相对数值，即你给出要跳过的一段消息的距离即可。这里的“跳”是双向的，你既可以向前“跳”，也可以向后“跳”。
  - 时间维度
    - 我们可以给定一个时间，让消费者把位移调整成大于该时间的最小位移；也可以给出一段时间间隔，比如 30 分钟前，然后让消费者直接将位移调回 30 分钟之前的位移值。
      - DateTime 允许你指定一个时间，然后将位移重置到该时间之后的最早位移处
      - Duration 策略则是指给定相对的时间间隔，然后将位移调整到距离当前给定时间间隔的位移处，具体格式是 PnDTnHnMnS。







