



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
-  使用 Kafka Java Consumer API 编程。
- 使用 Kafka 自带的 JMX 监控指标。
  - records-lag-max 和 records-lead-min
    - 这里的 Lead 值是指消费者最新消费消息的位移与分区当前第一条消息位移的差值
    - Lag 越大的话，Lead 就越小，反之也是同理
    - 一旦你监测到 Lead 越来越小，甚至是快接近于 0 了，你就一定要小心了，这可能预示着消费者端要丢消息了。因为已经逼近最早的消息了



