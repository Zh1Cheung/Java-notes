## 分布式事务的解决方案（三种）

- 全局事务（DTP模型）

  - 三种角色
    - AP：Application 应用系统
      - 开发的业务系统
    - TM：Transaction Manager 事务管理器
      - 提供分布式事务的操作接口
      - 调度资源管理器
      - DTP只是一套实现分布式事务的规范，并没有定义具体如何实现分布式事务，TM可以采用2PC、3PC、Paxos等协议实现分布式事务
    - RM：Resource Manager 资源管理器
      - 能够提供数据服务的对象都可以是资源管理器，比如：数据库、消息中间件、缓存等。大部分场景下，数据库即为分布式事务中的资源管理器。
      - 源管理器能够提供单数据库的事务能力，将本数据库的提交、回滚等能力提供给事务管理器调用，以帮助事务管理器实现分布式的事务管理
      - RM具体的实现是由数据库厂商来完成的

- **可靠消息最终一致**

  - 业务处理服务在业务事务提交前，向实时消息服务请求发送消息，实时消息服务只记
    录消息数据，而不真正发送。业务处理服务在业务事务提交后，向实时消息服务确认
    发送。只有在得到确认发送指令后，实时消息服务才真正发送
  - 需要通过消息中间件来实现
    - 消息中间件扮演者分布式事务协调者的角色
    - Commit和Rollback都属于理想情况，但在实际系统中，Commit和Rollback指令都有可能在传输途中丢失。那么当出现这种情况的时候，消息中间件是如何保证数据一致性呢？——答案就是超时询问机制。
      - 上游系统只要发出Commit/Rollback指令后便可以处理其他任务，无需等待确认应答
      - 当上游系统执行完任务并向消息中间件提交了Commit指令后，便可以处理其他任务了，此时它可以认为事务已经完成，接下来消息中间件一定会保证消息被下游系统成功消费掉！那么这是怎么做到的呢？这由消息中间件的投递流程来保证
        - 上游系统和消息中间件之间采用异步通信是为了提高系统并发度。业务系统直接和用户打交道，用户体验尤为重要，因此这种异步通信方式能够极大程度地降低用户等待时间
        - 异步通信相对于同步通信而言，没有了长时间的阻塞等待，因此系统的并发性也大大增加
        - 异步通信可能会引起Commit/Rollback指令丢失的问题，这就由消息中间件的超时询问机制来弥补
      - 消息中间件向下游系统投递完消息后便进入阻塞等待状态，消息中间件收到确认应答后便认为该事务处理完毕
        - 消息中间件和下游系统之间为什么要采用同步通信呢
        - 异步能提升系统性能，但随之会增加系统复杂度；而同步虽然降低系统并发度，但实现成本较低
  - 步骤
    - 在系统A处理任务A前，首先向消息中间件发送一条消息
      - 消息中间件收到后将该条消息持久化，但并不投递。此时下游系统B仍然不知道该条消息的存在。
      - 消息中间件持久化成功后，便向系统A返回一个确认应答；
      - 系统A收到确认应答后，则可以开始处理任务A；
    - 任务A处理完成后，向消息中间件发送Commit请求。该请求发送完成后，对系统A而言，该事务的处理过程就结束了，此时它可以处理别的任务了。  
      - 但commit消息可能会在传输途中丢失，从而消息中间件并不会向系统B投递这条消息，从而系统就会出现不一致性。这个问题由消息中间件的事务回查机制完成
    - 消息中间件收到Commit指令后，便向系统B投递该消息，从而触发任务B的执行；
    - 当任务B执行完成后，系统B向消息中间件返回一个确认应答，告诉消息中间件该消息已经成功消费，此时，这个分布式事务完成。
  - 适用范围
    - 对最终一致性时间敏感度较高，降低业务被动方实现成本

- **最大努力通知**

  - 最大努力通知也被称为定期校对

    - 业务活动的主动方，在完成业务处理之后，向业务活动的被动方发送消息，
      允许消息丢失。
      业务活动的被动方根据定时策略，向业务活动主动方查询，恢复丢失的业
      务消息
    - 这种方案也需要消息中间件的参与

  - 意外情况

    - 1. 消息中间件向下游系统投递消息失败

      - 消息中间件具有重试机制
      - 设置消息的重试次数和重试时间间隔
      - 消息中间件需要提供失败消息的查询接口，下游系统会定期查询失败消息，并将其消费，这就是所谓的“定期校对”
      - 如果重复投递和定期校对都不能解决问题，往往是因为下游系统出现了严重的错误，此时就需要人工干预

    - 2. 上游系统向消息中间件发送消息失败

      - 需要在上游系统中建立消息重发机制
        - 可以在上游系统建立一张本地消息表，并将 任务处理过程和 向本地消息表中插入消息这两个步骤放在一个本地事务中完成
        - 如果向本地消息表插入消息失败，那么就会触发回滚，之前的任务处理结果就会被取消。如果这两步都执行成功，那么该本地事务就完成了
        - 接下来会有一个专门的消息发送者不断地发送本地消息表中的消息，如果发送失败它会返回重试。

  - 针对异步调用

  - 适用范围

    - • 对业务最终一致性的时间敏感度低
    - • 跨企业的业务活动
    - 银行通知、商户通知等（各大交易业务平台间的商户通知：多次通知、查询校对、对账文件）

- TCC（两阶段型、补偿型）

  - 概述
    - 对于常见的微服务系统，大部分接口调用是同步的，也就是一个服务直接调用另外一个服务的接口
    - • 一个完整的业务活动由一个主业务服务与若干从业务服务组成
      • 主业务服务负责发起并完成整个业务活动
      • 从业务服务提供TCC型业务操作
      • 业务活动管理器控制业务活动的一致性，它登记业务活动中的操作， 并在
      业务活动提交时确认所有的TCC型操作的confirm操作，在业务活动取消
      时调用所有TCC型操作的cancel操作
    - Try：尝试待执行的业务
    - Confirm：执行业务
    - Cancel：取消执行的业务
  - 适用范围
    - 强隔离性、严格一致性要求的业务活动
    - 适用于执行时间较短的业务（比如处理账户、收费等业务）



## 分布式事务

- **背景1：**
  - 如今，大部分公司的服务基本都实现了**微服务化**，首先是业务需求，为了解耦业务；其次是为了减少业务与业务之间的相互影响。
  - 电商系统亦是如此，大部分公司的电商系统都是分为了不同服务模块，例如商品模块、订单模块、库存模块等等。事实上，分解服务是一把双刃剑，可以带来一些开发、性能以及运维上的优势，但同时也会增加业务开发的逻辑复杂度。其中最为突出的就是分布式事务了。
  - **有很多用例会跨多个子系统才能完成，比较典型的是电子商务网站的下单支付流程，至少会涉及交易系统和支付系统，而且这个过程中会涉及到事务的概念**，即保证交易系统和支付系统的数据一致性，此处我们称这种**跨系统的事务为分布式事务**，具体一点而言，分布式事务是指事务的参与者、支持事务的服务器、资源服务器以及事务管理器分别位于不同的分布式系统的不同节点之上。
- **背景2**
  - 国内最早关于TCC的报道，应该是InfoQ上对阿里程立博士的一篇采访



通常，存在分布式事务的服务架构部署有以下两种：同服务不同数据库，不同服务不同数据库。我们以商城为例，用图示说明下这两种部署：

![img](https://static001.geekbang.org/resource/image/11/5a/111f44892deb9919a1310d636a538f5a.jpg)



![img](https://static001.geekbang.org/resource/image/48/6c/48d448543aeac5eba4b9edd24e1bcf6c.jpg)



通常，我们都是基于第二种架构部署实现的，那我们应该如何实现在这种服务架构下，有关订单提交业务的分布式事务呢？

## 分布式事务解决方案（TCC）

我们讲过，在单个数据库的情况下，数据事务操作具有 ACID 四个特性，但如果在一个事务中操作多个数据库，则无法使用数据库事务来保证一致性。

也就是说，当两个数据库操作数据时，可能存在一个数据库操作成功，而另一个数据库操作失败的情况，我们无法通过单个数据库事务来回滚两个数据操作。

而分布式事务就是为了解决在同一个事务下，不同节点的数据库操作数据不一致的问题。在一个事务操作请求多个服务或多个数据库节点时，要么所有请求成功，要么所有请求都失败回滚回去。通常，分布式事务的实现有多种方式，例如 XA 协议实现的二阶提交（2PC）、三阶提交 (3PC)，以及 TCC 补偿性事务。

在了解 2PC 和 3PC 之前，我们有必要先来了解下 XA 协议。XA 协议是由 X/Open 组织提出的一个分布式事务处理规范，目前 MySQL 中只有 InnoDB 存储引擎支持 XA 协议。

### 1. XA 规范

在 XA 规范之前，存在着一个 DTP 模型，该模型规范了分布式事务的模型设计。

DTP 规范中主要包含了 AP、RM、TM 三个部分，其中 AP 是应用程序，是事务发起和结束的地方；RM 是资源管理器，主要负责管理每个数据库的连接数据源；TM 是事务管理器，负责事务的全局管理，包括事务的生命周期管理和资源的分配协调等。

![img](https://static001.geekbang.org/resource/image/dc/67/dcbb483b62b1e0a51d03c7edfcf89767.jpg)

XA 则规范了 TM 与 RM 之间的通信接口，在 TM 与多个 RM 之间形成一个双向通信桥梁，从而在多个数据库资源下保证 ACID 四个特性。

这里强调一下，JTA 是基于 XA 规范实现的一套 Java 事务编程接口，是一种两阶段提交事务。

### 2. 二阶提交和三阶提交

XA 规范实现的分布式事务属于二阶提交事务，顾名思义就是通过两个阶段来实现事务的提交。

在第一阶段，应用程序向事务管理器（TM）发起事务请求，而事务管理器则会分别向参与的各个资源管理器（RM）发送事务预处理请求（Prepare），此时这些资源管理器会打开本地数据库事务，然后开始执行数据库事务，但执行完成后并不会立刻提交事务，而是向事务管理器返回已就绪（Ready）或未就绪（Not Ready）状态。如果各个参与节点都返回状态了，就会进入第二阶段。

![img](https://static001.geekbang.org/resource/image/2a/95/2a1cf8f45675acac6fe07c172a36ec95.jpg)

到了第二阶段，如果资源管理器返回的都是就绪状态，事务管理器则会向各个资源管理器发送提交（Commit）通知，资源管理器则会完成本地数据库的事务提交，最终返回提交结果给事务管理器。

![img](https://static001.geekbang.org/resource/image/59/d5/59734e1a229ceee9df4295d0901ce2d5.jpg)

在第二阶段中，如果任意资源管理器返回了未就绪状态，此时事务管理器会向所有资源管理器发送事务回滚（Rollback）通知，此时各个资源管理器就会回滚本地数据库事务，释放资源，并返回结果通知。



![img](https://static001.geekbang.org/resource/image/87/2f/8791dfe19fce916f77b6c5740bc32e2f.jpg)

但事实上，二阶事务提交也存在一些缺陷。

第一，在整个流程中，我们会发现各个资源管理器节点存在阻塞，只有当所有的节点都准备完成之后，事务管理器才会发出进行全局事务提交的通知，这个过程如果很长，则会有很多节点长时间占用资源，从而影响整个节点的性能。

一旦资源管理器挂了，就会出现一直阻塞等待的情况。类似问题，我们可以通过设置事务超时时间来解决。

第二，仍然存在数据不一致的可能性，例如，在最后通知提交全局事务时，由于网络故障，部分节点有可能收不到通知，由于这部分节点没有提交事务，就会导致数据不一致的情况出现。



**而三阶事务（3PC）的出现就是为了减少此类问题的发生。**

3PC 把 2PC 的准备阶段分为了准备阶段和预处理阶段，在第一阶段只是询问各个资源节点是否可以执行事务，而在第二阶段，所有的节点反馈可以执行事务，才开始执行事务操作，最后在第三阶段执行提交或回滚操作。并且在事务管理器和资源管理器中都引入了超时机制，如果在第三阶段，资源节点一直无法收到来自资源管理器的提交或回滚请求，它就会在超时之后，继续提交事务。

所以 3PC 可以通过超时机制，避免管理器挂掉所造成的长时间阻塞问题，但其实这样还是无法解决在最后提交全局事务时，由于网络故障无法通知到一些节点的问题，特别是回滚通知，这样会导致事务等待超时从而默认提交。

### 3. 事务补偿机制（TCC）

以上这种基于 XA 规范实现的事务提交，由于阻塞等性能问题，有着比较明显的低性能、低吞吐的特性。所以在抢购活动中使用该事务，很难满足系统的并发性能。

除了性能问题，JTA 只能解决同一服务下操作多数据源的分布式事务问题，换到微服务架构下，可能存在同一个事务操作，分别在不同服务上连接数据源，提交数据库操作。

而 TCC 正是为了解决以上问题而出现的一种分布式事务解决方案。TCC 采用最终一致性的方式实现了一种柔性分布式事务，与 XA 规范实现的二阶事务不同的是，TCC 的实现是基于服务层实现的一种二阶事务提交。

TCC 分为三个阶段，即 Try、Confirm、Cancel 三个阶段。

![img](https://static001.geekbang.org/resource/image/23/a9/23f68980870465ba6c00c0f2619fcfa9.jpg)

- Try 阶段：主要尝试执行业务，执行各个服务中的 Try 方法，主要包括预留操作；
- Confirm 阶段：确认 Try 中的各个方法执行成功，然后通过 TM 调用各个服务的 Confirm 方法，这个阶段是提交阶段；
- Cancel 阶段：当在 Try 阶段发现其中一个 Try 方法失败，例如预留资源失败、代码异常等，则会触发 TM 调用各个服务的 Cancel 方法，对全局事务进行回滚，取消执行业务。

以上执行只是保证 Try 阶段执行时成功或失败的提交和回滚操作，你肯定会想到，如果在 Confirm 和 Cancel 阶段出现异常情况，那 TCC 该如何处理呢？此时 TCC 会不停地重试调用失败的 Confirm 或 Cancel 方法，直到成功为止。

### 柔性事务的概念

在电商等互联网场景下，传统的事务在数据库性能和处理能力上都暴露出了瓶颈。在分布式领域基于CAP理论以及BASE理论，有人就提出了**柔性事务**的概念。

基于BASE理论的设计思想，柔性事务下，在不影响系统整体可用性的情况下(Basically Available 基本可用)，允许系统存在数据不一致的中间状态(Soft State 软状态)，在经过数据同步的延时之后，最终数据能够达到一致。**并不是完全放弃了ACID，而是通过放宽一致性要求，借助本地事务来实现最终分布式事务一致性的同时也保证系统的吞吐**。

但 TCC 补偿性事务也有比较明显的缺点，那就是对业务的侵入性非常大。

首先，我们需要在业务设计的时候考虑预留资源；然后，我们需要编写大量业务性代码，例如 Try、Confirm、Cancel 方法；最后，我们还需要为每个方法考虑幂等性。这种事务的实现和维护成本非常高，但综合来看，这种实现是目前大家最常用的分布式事务解决方案。



## 优缺点

**优点：**

- 性能提升：具体业务来实现控制资源锁的粒度变小，不会锁定整个资源。
- 数据最终一致性：基于 Confirm 和 Cancel 的幂等性，保证事务最终完成确认或者取消，保证数据的一致性。
- 可靠性：解决了 XA 协议的协调者单点故障问题，由主业务方发起并控制整个业务活动，业务活动管理器也变成多点，引入集群。



**TCC 方式虽然提高了分布式事务的整体性能，但也给业务层带来了非常大的工作量，对应用服务的侵入性非常强，但这是大多数公司目前所采用的分布式事务解决方案。**





# Tyloo（概念介绍）



## 功能

- 支持 `Dubbo`RPC框架进行分布式事务

- 支持事务异常回滚，超时异常恢复，防止事务悬挂

- 事务日志存储支持 `mysql`,, `mongodb`, `redis`, `zookeeper` 等方式

- 高性能，支持微服务集群部署

  



## 改进性能


 可以使用RedisTransactionRepository来存储事务日子, 另外可以设置confirm或cancel是异步执行，不会影响事务的一致性（可获取1.2.3.6版本，@Compensable属性增加了asyncConfirm,asyncCancel属性


serializer可以设置为KryoPoolSerializer，以优化序列化性能

采用disruptor框架进行事务日志的异步读

提供后台管理可视化,以及metrics相关性能监控

提供零侵入的`spring namespace`, `springboot` 快速集成方式, 简单易用





## 项目概念

- 一个完整的业务活动由一个主业务服务与若干从业务服务组成
- 从业务服务提供TCC型业务操作
- 业务活动管理器控制业务活动的一致性，它登记业务活动中的操作， 并在业务活动提交时确认所有的TCC型操作的confirm操作，在业务活动取消时调用所有TCC型操作的cancel操作
- 分布式事务是一种由若干修改本地状态的子事务组合而形成的全局事务. 

  - 子事务具有ACID特性

  - 分布式事务的本质就是多个分布式节点达成共识。
- 三个阶段
- **框架本身很简单，主要是扫描 TCC 接口，注册资源，拦截接口调用，注册分支事务，最后回调二阶段接口。最核心的实际上是 TCC 接口的实现逻辑。**
- 希望你明确这样几个重点。
  - TCC 是个业务层面的分布式事务协议，而 XA 规范是数据层面的分布式事务协议，这也是 TCC 和 XA 规范最大的区别。
  - TCC 与业务耦合紧密，在实际场景使用时，需要我们根据场景特点和业务逻辑来设计相应的预留、确认、撤销操作，相比 MySQL XA，有一定的编程开发工作量。
  - 本质上而言，TCC 是一种设计模式，也就是一种理念，它没有与任何技术（或实现）耦合，也不受限于任何技术，对所有的技术方案都是适用的。
  - 最后，我想补充的是，因为 TCC 是在业务代码中编码实现的，所以，TCC 可以跨数据库、跨业务系统实现资源管理，满足复杂业务场景下的事务需求，比如，TCC 可以将对不同的数据库、不同业务系统的多个操作通过编码方式，转换为一个原子操作，实现事务。
  - 另外，因为 TCC 的每一个操作对于数据库来讲，都是一个本地数据库事务，那么当操作结束时，本地数据库事务的执行也就完成了，所以相关的数据库资源也就被释放了，这就能避免数据库层面的二阶段提交协议长时间锁定资源，导致系统性能低下的问题。



## **业务场景介绍**

 咱们先来看看业务场景，假设你现在有一个电商系统，里面有一个支付订单的场景。

 那对一个订单支付之后，我们需要做下面的步骤：

- 更改订单的状态为“已支付”
- 扣减商品库存
- 给会员增加积分
- 创建销售出库单通知仓库发货

 这是一系列比较真实的步骤，无论大家有没有做过电商系统，应该都能理解。

 ![img](https://img2018.cnblogs.com/blog/918692/201811/918692-20181123154323784-1697034217.png)

 



 上述这几个步骤，要么一起成功，要么一起失败，**必须是一个整体性的事务**。 

但是如果你不用TCC分布式事务方案的话，就用个Spring Cloud开发这么一个微服务系统，很有可能会干出这种事儿来。 

所以说，**我们有必要使用TCC分布式事务机制来保证各个服务形成一个整体性的事务。**

 上面那几个步骤，要么全部成功，如果任何一个服务的操作失败了，就全部一起回滚，撤销已经完成的操作。

 比如说库存服务要是扣减库存失败了，那么订单服务就得撤销那个修改订单状态的操作，然后得停止执行增加积分和通知出库两个操作。

 说了那么多，老规矩，给大家上一张图，大伙儿顺着图来直观的感受一下。

![img](https://img2018.cnblogs.com/blog/918692/201811/918692-20181123154449362-184004902.png)

 

##  **落地实现TCC分布式事务**

 那么现在到底要如何来实现一个TCC分布式事务，使得各个服务，要么一起成功？要么一起失败呢？

 大家稍安勿躁，我们这就来一步一步的分析一下。咱们就以一个Spring Cloud开发系统作为背景来解释。

 

###  **1、TCC实现阶段一：Try**

 首先，订单服务那儿，他的代码大致来说应该是这样子的：

 ![img](https://img2018.cnblogs.com/blog/918692/201811/918692-20181123154530434-790705908.png)

 

如果你之前看过Spring Cloud架构原理那篇文章，同时对Spring Cloud有一定的了解的话，应该是可以理解上面那段代码的。

 **其实就是订单服务完成本地数据库操作之后，通过Spring Cloud的Feign来调用其他的各个服务罢了。**

 **但是光是凭借这段代码，是不足以实现TCC分布式事务的啊**？我们对这个订单服务修改点儿代码

 首先，上面那个订单服务先把自己的状态修改为：**OrderStatus.UPDATING**。

 这是啥意思呢？也就是说，在pay()那个方法里，你别直接把订单状态修改为已支付啊！你先把订单状态修改为**UPDATING**，也就是修改中的意思。

 这个状态是个没有任何含义的这么一个状态，代表有人正在修改这个状态罢了。

 然后呢，库存服务直接提供的那个reduceStock()接口里，也别直接扣减库存啊，你可以是**冻结掉库存**。

 举个例子，本来你的库存数量是100，你别直接100 - 2 = 98，扣减这个库存！

 你可以把可销售的库存：100 - 2 = 98，设置为98没问题，然后在一个单独的冻结库存的字段里，设置一个2。也就是说，有2个库存是给冻结了。

 积分服务的addCredit()接口也是同理，别直接给用户增加会员积分。你可以先在积分表里的一个**预增加积分字段**加入积分。

 比如：用户积分原本是1190，现在要增加10个积分，别直接1190 + 10 = 1200个积分啊！

 你可以保持积分为1190不变，在一个预增加字段里，比如说prepare_add_credit字段，设置一个10，表示有10个积分准备增加。

 仓储服务的saleDelivery()接口也是同理啊，你可以先创建一个销售出库单，但是这个销售出库单的状态是“**UNKNOWN**”。

 也就是说，刚刚创建这个销售出库单，此时还不确定他的状态是什么呢！

 上面这套改造接口的过程，其实就是所谓的TCC分布式事务中的第一个T字母代表的阶段，也就是**Try阶段**。

 **总结上述过程，如果你要实现一个TCC分布式事务，首先你的业务的主流程以及各个接口提供的业务含义，不是说直接完成那个业务操作，而是完成一个Try的操作。**

 **这个操作，一般都是锁定某个资源，设置一个预备类的状态，冻结部分数据，等等，大概都是这类操作。**

 咱们来一起看看下面这张图，结合上面的文字，再来捋一捋这整个过程。

 

![img](https://img2018.cnblogs.com/blog/918692/201811/918692-20181123154627825-1811880105.png)

 

 

### **2、TCC实现阶段二：Confirm**

 

然后就分成两种情况了，第一种情况是比较理想的，那就是各个服务执行自己的那个Try操作，都执行成功了，bingo！

 这个时候，就需要依靠**TCC分布式事务框架**来推动后续的执行了。

 否则的话，感知各个阶段的执行情况以及推进执行下一个阶段的这些事情，不太可能自己手写实现，太复杂了。

 如果你在各个服务里引入了一个TCC分布式事务的框架，**订单服务里内嵌的那个TCC分布式事务框架可以感知到**，各个服务的Try操作都成功了。

 此时，TCC分布式事务框架会控制进入TCC下一个阶段，第一个C阶段，也就是**Confirm阶段**。

 为了实现这个阶段，你需要在各个服务里再加入一些代码。

 比如说，**订单服务**里，你可以加入一个Confirm的逻辑，就是正式把订单的状态设置为“已支付”了，大概是类似下面这样子：

 ![img](https://img2018.cnblogs.com/blog/918692/201811/918692-20181123154705929-276762080.png) 

**库存服务**也是类似的，你可以有一个InventoryServiceConfirm类，里面提供一个reduceStock()接口的Confirm逻辑，这里就是将之前冻结库存字段的2个库存扣掉变为0。

 这样的话，可销售库存之前就已经变为98了，现在冻结的2个库存也没了，那就正式完成了库存的扣减。

 **积分服务**也是类似的，可以在积分服务里提供一个CreditServiceConfirm类，里面有一个addCredit()接口的Confirm逻辑，就是将预增加字段的10个积分扣掉，然后加入实际的会员积分字段中，从1190变为1120。

 **仓储服务**也是类似，可以在仓储服务中提供一个WmsServiceConfirm类，提供一个saleDelivery()接口的Confirm逻辑，将销售出库单的状态正式修改为“已创建”，可以供仓储管理人员查看和使用，而不是停留在之前的中间状态“UNKNOWN”了。

 **好了，上面各种服务的Confirm的逻辑都实现好了，一旦订单服务里面的TCC分布式事务框架感知到各个服务的Try阶段都成功了以后，就会执行各个服务的Confirm逻辑。**

 **订单服务内的TCC事务框架会负责跟其他各个服务内的TCC事务框架进行通信，依次调用各个服务的Confirm逻辑。然后，正式完成各个服务的所有业务逻辑的执行。**

 同样，给大家来一张图，顺着图一起来看看整个过程。

 ![img](https://img2018.cnblogs.com/blog/918692/201811/918692-20181123154740015-430603121.png)

 

### **3、TCC实现阶段三：Cancel**

 好，这是比较正常的一种情况，那如果是异常的一种情况呢？

 举个例子：在Try阶段，比如积分服务吧，他执行出错了，此时会怎么样？

 那订单服务内的TCC事务框架是可以感知到的，然后他会决定对整个TCC分布式事务进行回滚。

 也就是说，会执行各个服务的**第二个C阶段，Cancel阶段**。

 同样，为了实现这个Cancel阶段，各个服务还得加一些代码。

 首先**订单服务**，他得提供一个OrderServiceCancel的类，在里面有一个pay()接口的Cancel逻辑，就是可以将订单的状态设置为“CANCELED”，也就是这个订单的状态是已取消。

 **库存服务**也是同理，可以提供reduceStock()的Cancel逻辑，就是将冻结库存扣减掉2，加回到可销售库存里去，98 + 2 = 100。

 **积分服务**也需要提供addCredit()接口的Cancel逻辑，将预增加积分字段的10个积分扣减掉。

 **仓储服务**也需要提供一个saleDelivery()接口的Cancel逻辑，将销售出库单的状态修改为“CANCELED”设置为已取消。

 然后这个时候，订单服务的TCC分布式事务框架只要感知到了任何一个服务的Try逻辑失败了，就会跟各个服务内的TCC分布式事务框架进行通信，然后调用各个服务的Cancel逻辑。

 大家看看下面的图，直观的感受一下。

![img](https://img2018.cnblogs.com/blog/918692/201811/918692-20181123154820466-2112179693.png)

  

## **总结与思考**

 总结一下，你要玩儿TCC分布式事务的话：

 **首先需要选择某种TCC分布式事务框架**，各个服务里就会有这个TCC分布式事务框架在运行。

 **然后你原本的一个接口，要改造为3个逻辑，Try-Confirm-Cancel**。

 

- 先是服务调用链路依次执行Try逻辑

- 如果都正常的话，TCC分布式事务框架推进执行Confirm逻辑，完成整个事务

- 如果某个服务的Try逻辑有问题，TCC分布式事务框架感知到之后就会推进执行各个服务的Cancel逻辑，撤销之前执行的各种操作

 这就是所谓的**TCC分布式事务。**

 

 

TCC分布式事务的核心思想，说白了，就是当遇到下面这些情况时，

 

- 某个服务的数据库宕机了

- 某个服务自己挂了

- 那个服务的redis、elasticsearch、MQ等基础设施故障了

- 某些资源不足了，比如说库存不够这些

 

先来Try一下，不要把业务逻辑完成，先试试看，看各个服务能不能基本正常运转，能不能先冻结我需要的资源。

 如果Try都ok，也就是说，底层的数据库、redis、elasticsearch、MQ都是可以写入数据的，并且你保留好了需要使用的一些资源（比如冻结了一部分库存）。

 接着，再执行各个服务的Confirm逻辑，基本上Confirm就可以很大概率保证一个分布式事务的完成了。

 那如果Try阶段某个服务就失败了，比如说底层的数据库挂了，或者redis挂了，等等。

 此时就自动执行各个服务的Cancel逻辑，把之前的Try逻辑都回滚，所有服务都不要执行任何设计的业务逻辑。**保证大家要么一起成功，要么一起失败**。



### seata

Seata是通过TM管理全局事务，所有用Seata的AP都可以实现写的隔离，也就是对同一行数据有影响的时候，并不存在分布的A事务没有操作完毕，B事务就开始操作的情况。

Seata 设计通过事务协调器维护的全局写排它锁，来保证事务间的写隔离，而读写隔离级别则默认为未提交读的隔离级别。Seata默认采用了很乐观的分布式策略，CAP里面优先保证了A，并没有彻底解决脏读的问题。

而如果设置为读“已提交”，那就要Seata在内存记录额外的数据，用于返回已提交的"正确数据"？而这个就又扯出内存管理或崩溃时这些"正确数据"持久化的问题，导致系统复杂度上升



## TCC 设计原则

设计一套 TCC 接口最重要的是什么？主要有两点，**第一点，需要将操作分成两阶段完成。**TCC（Try-Confirm-Cancel）分布式事务模型相对于 XA 等传统模型，其特征在于它不依赖 RM 对分布式事务的支持，而是通过对业务逻辑的分解来实现分布式事务。

TCC 模型认为对于业务系统中一个特定的业务逻辑 ，其对外提供服务时，必须接受一些不确定性，即对业务逻辑初步操作的调用仅是一个临时性操作，调用它的主业务服务保留了后续的取消权。如果主业务服务认为全局事务应该回滚，它会要求取消之前的临时性操作，这就对应从业务服务的取消操作。而当主业务服务认为全局事务应该提交时，它会放弃之前临时性操作的取消权，这对应从业务服务的确认操作。针对一个具体的业务服务，TCC 分布式事务模型需要业务系统提供三段业务逻辑。

**第二点，就是要根据自身的业务模型控制并发，这个对应 ACID 中的隔离性。**

在第一阶段需要检查并预留业务资源，因此，我们在扣钱 TCC 资源的 Try 接口里先检查 A 账户余额是否足够，然后预留余额里的业务资源，即扣除 30 元。在 Confirm 接口，由于业务资源已经在 Try 接口里扣除掉了，那么在第二阶段的 Confirm 接口里，可以什么都不用做。

而在加钱的 TCC 资源里。在第一阶段 Try 接口里不能直接给账户加钱，如果这个时候给账户增加了可用余额，那么在一阶段执行完后，账户里的钱就可以被使用了。但是一阶段执行完以后，有可能是要回滚的。因此，真正加钱的动作需要放在 Confirm 接口里。对于加钱这个动作，第一阶段 Try 接口里不需要预留任何资源，可以设计为空操作。那相应的，Cancel 接口没有资源需要释放，也是一个空操作。只有真正需要提交时，再在 Confirm 接口里给账户增加可用余额。

框架本身仅提供两阶段原子提交协议，保证分布式事务原子性。事务的隔离需要交给业务逻辑来实现。隔离的本质就是控制并发，防止并发事务操作相同资源而引起的结果错乱。

可以发现，并发控制是业务逻辑执行正确的保证，但是像两阶段锁这样的并发访问控制技术要求一直持有数据库资源锁直到整个事务执行结束，特别是在分布式事务架构下，要求持有锁到分布式事务第二阶段执行结束，也就是说，分布式事务会加长资源锁的持有时间，导致并发性能进一步下降。

因此，TCC 模型的隔离性思想就是通过业务的改造，在第一阶段结束之后，从底层数据库资源层面的加锁过渡为上层业务层面的加锁，从而释放底层数据库锁资源，放宽分布式事务锁协议，将锁的粒度降到最低，以最大限度提高业务并发性能。

“账户 A 上有 100 元，事务 T1 要扣除其中的 30 元，事务 T2 也要扣除 30 元，出现并发”。在第一阶段 Try 操作中，需要先利用数据库资源层面的加锁，检查账户可用余额，如果余额充足，则预留业务资源，扣除本次交易金额，一阶段结束后，虽然数据库层面资源锁被释放了，但这笔资金被业务隔离，不允许除本事务之外的其它并发事务动用。

事务 T1 和 T2 分别扣除的那一部分资金，相互之间无干扰。这样在分布式事务的二阶段，无论 T1 是提交还是回滚，都不会对 T2 产生影响，这样 T1 和 T2 可以在同一个账户上并发执行。

一阶段结束以后，实际上采用业务加锁的方式，隔离账户资金，在第一阶段结束后直接释放底层资源锁，该用户和卖家的其他交易都可以立刻并发执行，而不用等到整个分布式事务结束，可以获得更高的并发交易能力。



TCC是try-confirm-cancel的单词首字母缩写，是一个类2PC的柔性事务解决方案，由支付宝提出后得到广泛的实践。其核心思想是：`针对每个操作，都要注册一个与其对应的确认和补偿（撤销）操作`。

**补偿事务（TCC）有三个阶段：**

- Try 阶段，对业务系统做检测和资源预留
- Confirm 阶段对业务系统做确认提交，默认：Try执行成功，Confirm一定成功
- Cancel 阶段在业务执行失败，需要回滚的情况下执行的业务取消，预留资源释放。

**优点**

跟2PC比起来，实现以及流程相对简单了一些，但数据的一致性比2PC也要差一些。

**缺点**

缺点还是比较明显的，在2,3步中都有可能失败。TCC属于应用层的一种补偿方式，所以需要程序员在实现的时候多写很多补偿的代码，在一些场景中，一些业务流程可能用TCC不太好定义及处理。

**首先我们看它的一个原理图：**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190118161821956.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0owODA2MjQ=,size_16,color_FFFFFF,t_70)
图中的主服务调用两个从服务，这两个从服务属于不同的进程，各自操作不同的数据库表。主服务A调用从服务B后，继续调用从服务C，这个过程要保证调用B、C同时成功，同时失败。如果任何一个服务的操作失败了，就全部一起回滚，撤销已经完成的操作。

那么如何保证多个服务的调用是同时成功、同时失败呢。TCC帮我们实现了这个目标。

------

## **【1】TRY阶段**

首先进行TRY阶段，该阶段主要做资源的锁定/预留，设置一个预备的状态，冻结部分数据等等。

场景：电商平台先在订单模块做下单操作，下单成功后调用库存模块做扣减库存，扣减成功调用支付接口进行支付，然后调用积分模块做积分的增加，最后调用发货模块做发货处理。

**这个过程中的Try阶段描述如下：**

订单服务先做下单操作，这是个本地事务，能够保证ACID的事务特性。下单成功后，订单服务将当前订单状态由初始化改为处理中进行扣库存操作，这里不能直接将库存扣除，应当冻结库存，将库存减去后，将减去的值保存在已冻结的字段中。

例如：本来库存数量是100，要减去5个库存，不能直接100 - 5 = 95，而是要把可销售的库存设置为：100 - 5 = 95，接着在一个单独的库存冻结的字段里，设置一个5。也就是说，有5个库存是给冻结了。

此时订单状态为OrderStatus.DEALING。

接着进行支付操作。那么为什么不直接进行支付，然后改为支付完成呢？因为存在支付失败甚至支付未知的风险，只要进行了支付操作，订单状态就不是初始化了。也就是说，不能直接把订单状态修改为已支付的确认状态！而是应当先把订单状态修改为DEALING，也就是处理中状态。该状态是个没有任何含义的中间状态，代表分布式事务正在进行中。

积分服务的增加积分接口也是同理，不能直接给用户增加会员积分。可以先在积分表里的一个预增加积分字段加入积分。

比如：用户积分原本是1000，现在要增加100个积分，可以保持积分为1000不变，在一个预增加字段里，设置一个100，表示有100个积分准备增加。

发货服务的发货接口也是同理，可以先创建一个发货订单，并设置这个销售出库单的状态是“DEALING”。

也就是说，刚刚创建这个发货订单，此时不能确定他的状态是什么。需要等真实发货之后再进行状态的修改。

这整个过程也就是所谓的TCC分布式事务中的TRY阶段。

简而言之，TRY阶段的业务的主流程以及各个接口提供的业务含义，不是直接完成那个业务操作，而是完成一个资源的预准备的操作，状态均为过渡态。

------

## **【2】CONFIRM阶段**

常见的TCC框架，如：ByteTCC、tcc-transaction 均为我们实现了事务管理器，用来执行CONFIRM阶段。他们能够对各个子模块的try阶段执行结果有所感知。

感知各个阶段的执行情况以及推进执行下一个阶段的操作较为复杂，不太可能自己手写实现，我们最好是借助开源框架实现。

为了实现这个阶段，我们需要加入CONFIRM操作相关的代码做事务的提交操作。

接着上述的情景来说：

- 订单服务中的CONFIRM操作，是将订单状态更新为支付成功这样的确定状态。
- 库存服务中，我们要加入正式扣除库存的操作，将临时冻结的库存真正的扣除，更新冻结字段为0，并修改库存字段为减去库存后的值。
- 同时积分服务将积分变更为增加积分之后的值，修改预增加的值为0，积分值修改为原值+预增加的100分的和。
- 发货服务也类似，真实发货后，修改DEALING为已发货。

当TCC框架感知到各个服务的TRY阶段都成功了以后，就会执行各个服务的CONFIRM逻辑。

各个模块内的TCC事务框架会负责跟其他服务内的TCC事务框架进行通信，依次调用各服务的CONFIRM逻辑。正式完成各服务的完整的业务逻辑的执行。

------

## **【3】CANCEL阶段**

CONFIRM是业务正常执行的阶段，那么异常分支自然交给CANCEL阶段执行了。

接着TRY阶段的业务情景来说。

- 订单服务中，当支付失败，CANCEL操作需要更改订单状态为支付失败
- 库存服务中的CANCEL操作要将预扣减的库存加回到原库存，也就是可用库存=90+10=100
- 积分服务要将预增加的100个积分扣除
- 发货服务的CANCEL操作将发货订单的状态修改为发货取消

当TCC框架感知到任何一个服务的TRY阶段执行失败，就会在和各服务内的TCC分布式事务框架进行通信的过程中，调用各个服务的CANCEL逻辑，将事务进行回滚

------

## **【4】总结**

TCC分布式事务的核心思想，就是当系统出现异常时，比如某服务的数据库宕机了、某个服务自己挂了、系统使用的第三方服务如redis、elasticsearch、MQ等基础设施出现故障了或者某些资源不足了比如说库存不够等等情况下，先执行TRY操作，而不是一次性把业务逻辑做完，进行预操作，看各个服务能不能基本正常运转，能不能预留出需要的资源。

如果TRY阶段均执行ok，即，数据库、redis、elasticsearch、MQ都是可以写入数据的，并且保留成功需要使用的一些资源（比如库存冻结成功、积分预添加完成）。

接着，再执行各个服务的CONFIRM逻辑，基本上CONFIRM执行完成之后就可以很大概率保证一个分布式事务的完成了。

那如果TRY阶段某个服务就执行失败了，比如说底层的数据库挂了，或者redis挂了，那么此时就自动执行各个服务的CANCEL逻辑，把之前的TRY逻辑都回滚，所有服务都不执行任何设计的业务逻辑。从而保证各个服务模块一起成功，或者一起失败。

到这里还是不能保证完全的事务一致性，试想，如果真的发生服务突发性宕机，比如订单服务挂了，那么重启之后，TCC框架如何保证之前的事务继续执行呢？

这个其实不必担心，成熟的TCC框架比如TCC-transaction中引入了事务的活动日志，它们保存了分布式事务运行的各个阶段的状态。后台会启动一个定时任务，周期性的扫描未执行完成的事务进行重试，保证最终一定会成功或失败。

这里也体现了TCC解决方案是一个保证最终一致性的柔性事务解决方案。



# **账务系统模型优化**（加钱扣钱流程）

在扣钱的 TCC 资源里。Try 接口不再是直接扣除账户的可用余额，而是真正的预留资源，冻结部分可用余额，即减少可用余额，增加冻结金额。Confirm 接口也不再是空操作，而是使用 Try 接口预留的业务资源，即将该部分冻结金额扣除；最后在 Cancel 接口里，就是释放预留资源，把 Try 接口的冻结金额扣除，增加账户可用余额。加钱的 TCC资源由于不涉及冻结金额的使用，所以无需更改。

通过这样的优化，可以更直观的感受到 TCC 接口的预留资源、使用资源、释放资源的过程。

并发控制中，在事务 T1 的第一阶段 Try 操作中，先锁定账户，检查账户可用余额，如果余额充足，则预留业务资源，减少可用余额，增加冻结金额。并发的事务 T2 类似，加锁，检查余额，减少可用余额金额，增加冻结金额。

这里可以发现，事务 T1 和T2 在一阶段执行完成后，都释放了数据库层面的资源锁，但是在各自二阶段的时候，相互之间并无干扰，各自使用本事务内第一阶段 Try 接口内冻结金额即可。这里大家就可以直观感受到，在每个事务的第一阶段，先通过数据库层面的资源锁，预留业务资源，即冻结金额。虽然在一阶段结束以后，数据库层面的资源锁被释放了，但是第二阶段的执行并不会被干扰，这是因为数据库层面资源锁释放以后通过业务隔离的方式为这部分资源加锁，不允许除本事务之外的其它并发事务动用，从而保证该事务的第二阶段能够正确顺利的执行

通过这两个例子，为大家讲解了**怎么去设计一套完备的 TCC 接口。最主要的有两点，一点是将业务逻辑拆分成两个阶段完成，即 Try、Confirm、Cancel 接口。其中 Try 接口检查资源、预留资源、Confirm 使用资源、Cancel 接口释放预留资源。另外一点就是并发控制，采用数据库锁与业务加锁的方式结合。**由于业务加锁的特性不影响性能，因此，尽可能降低数据库锁粒度，过渡为业务加锁，从而提高业务并发能力。

# **业务流程使用记录**（模拟流程）：

前提：用户下单，建立订单，创建支付记录，支付记录状态为待支付

try：

​    用户金额冻结

​    调用积分处理TCC：

​    try：预增加积分

​    confirm：更新增加积分状态

​    cancel：取消增加的积分

 confirm：

​    订单支付状态更新为已支付

​    订单支付记录支付状态更新为已支付 

​    用户金额扣款（以上三个操作在同一本地事务）

cancel：

​    判断订单支付状态与订单记录支付状态为未支付
​    用户冻结金额释放



## 执行流程

约定：

根事务/本地事务（P1=Order），分支事务/远程事务：P2=Capital，P3=Red。
由于远程事务参与到本地事务是通过Proxy的方式，而远程事务本身也有事务。
所以以P2'表示远程的Capital，P3'表示远程的Red。P2和P3表示Proxy。
P1、P2、P3、P2'、P3'的@Conpensable注解都定义了confirm和cancel方法
P2和P3 Proxy上的注解的confirm和cancel都是record方法
而且P2和P3的事务传播属性为SUPPORTS，即normal类型
TCC中的try方法有时候也叫做record方法，confirm方法也叫做commit

具体流程（重点是步骤20）：

P1 add to Transaction's participants, => P1
调用 P1的真正方法，分别调用两个proxy的record方法
P2.try no transaction
P2 add to Transaction(1)'s participants, now: P1|P2
由于P2 Proxy调用了远程P2'的record方法，所以P2'远程事务开始工作
P2' try propogate new Transaction(2)
P2' add to his Transaction(2)'s participants, now P2'
调用P2'的真正方法... P2'远程事务的try方法结束
P3.try no transaction
P3 add to Transaction(1)'s participants, now P1|P2|P3
由于P3 Proxy调用了远程P3'的record方法，所以P3'远程事务开始工作
P3' try propogate new Transaction(3)
P3' add to his Transaction(3)'s participants, now P3'
调用P3'的真正方法... P3'远程事务的try方法结束
P1所在的事务Transaction(1)有三个参与者，分别是P1|P2|P3，并且P2和P3的try方法都结束了，注意这里的方法结束，表示P2'和P3'的try方法也结束了
P1 ROOT Trasaction调用commit方法，分别调用P1,P2,P3的commit方法
P1的commit方法会调用confirmMakePayment
再来看P2和P3 Proxy的commit方法
P2的commit方法仍然是record方法（约定5），所以继续调用P2'的record方法，只不过这时P2'
try方法的状态是CONFIRMING，所以还会调用它自己的commit方法，即调用P2'的confirm方法
P3的commit方法仍然是record方法（约定5），所以继续调用P3'的record方法，只不过这时P3'
try方法的状态是CONFIRMING，所以还会调用它自己的commit方法，即调用P3'的confirm方法
整个事务完成

我的理解是本地事务和远程事务都有一个Transaction对象。
远程事务要加入到本地事务中是通过Proxy的方式，这样本地事务有三个Participants（包括自己还有两个远程事务的代理）。而远程事务的Transaction对象中只有自己的Participant：
Transactions 	Transaction1 	Transaction2 	Transaction3
Participants 	[P1, P2, P3] 	[P2'] 	[P3']

这里对约定5的解释是：在本地事务中，要远程调用远程事务，只能通过Proxy，而Proxy的confirm和cancel方法都是record方法。这样远程事务在本地事务的不同阶段（CONFIRMING或CANCEL），要进入不同的分支。比如TRY阶段是propogate new transaction，而CONFIRMING阶段则是propogate exist transaction。
同时在CONFIRMING时，还要调用提交方法，这里的提交是远程事务的提交，即调用远程事务的confirm方法。





## **TCC 异常控制**



在微服务架构下，很有可能出现网络超时、重发，机器宕机等一系列的异常 Case。一旦遇到这些 Case，就会导致我们的分布式事务执行过程出现异常。根据蚂蚁金服内部多年的使用来看，最常见的主要是这三种异常，分别是空回滚、幂等、悬挂。



### **空回滚**

**首先是空回滚。**什么是空回滚？空回滚就是对于一个分布式事务，在没有调用 TCC 资源 Try 方法的情况下，调用了二阶段的 Cancel 方法，Cancel 方法需要识别出这是一个空回滚，然后直接返回成功。

注册分支事务是在调用 RPC 时，Seata 框架的切面会拦截到该次调用请求，先向 TC 注册一个分支事务，然后才去执行 RPC 调用逻辑。如果 RPC 调用逻辑有问题，比如调用方机器宕机、网络异常，都会造成 RPC 调用失败，即未执行 Try 方法。但是分布式事务已经开启了，需要推进到终态，因此，TC 会回调参与者二阶段 Cancel 接口，从而形成空回滚。

那怎么解决空回滚呢？前面提到，Cancel 要识别出空回滚，直接返回成功。那关键就是要识别出这个空回滚。思路很简单就是需要知道一阶段是否执行，如果执行了，那就是正常回滚；如果没执行，那就是空回滚。因此，需要一张额外的事务控制表，其中有分布式事务 ID 和分支事务 ID，第一阶段 Try 方法里会插入一条记录，表示一阶段执行了。Cancel 接口里读取该记录，如果该记录存在，则正常回滚；如果该记录不存在，则是空回滚。



### **幂等**

**接下来是幂等。**幂等就是对于同一个分布式事务的同一个分支事务，重复去调用该分支事务的第二阶段接口，因此，要求 TCC 的二阶段 Confirm 和 Cancel 接口保证幂等，不会重复使用或者释放资源。如果幂等控制没有做好，很有可能导致资损等严重问题。

提交或回滚是一次 TC 到参与者的网络调用。因此，网络故障、参与者宕机等都有可能造成参与者 TCC 资源实际执行了二阶段防范，但是 TC 没有收到返回结果的情况，这时，TC 就会重复调用，直至调用成功，整个分布式事务结束。

怎么解决重复执行的幂等问题呢？一个简单的思路就是记录每个分支事务的执行状态。在执行前状态，如果已执行，那就不再执行；否则，正常执行。前面在讲空回滚的时候，已经有一张事务控制表了，事务控制表的每条记录关联一个分支事务，那我们完全可以在这张事务控制表上加一个状态字段，用来记录每个分支事务的执行状态。

该状态字段有三个值，分别是初始化、已提交、已回滚。Try 方法插入时，是初始化状态。二阶段 Confirm 和 Cancel 方法执行后修改为已提交或已回滚状态。当重复调用二阶段接口时，先获取该事务控制表对应记录，检查状态，如果已执行，则直接返回成功；否则正常执行。



### **悬挂**

**最后是防悬挂。**按照惯例，咱们来先讲讲什么是悬挂。悬挂就是对于一个分布式事务，其二阶段 Cancel 接口比 Try 接口先执行。因为允许空回滚的原因，Cancel 接口认为 Try 接口没执行，空回滚直接返回成功，对于 Seata 框架来说，认为分布式事务的二阶段接口已经执行成功，整个分布式事务就结束了。但是这之后 Try 方法才真正开始执行，预留业务资源，前面提到事务并发控制的业务加锁，对于一个 Try 方法预留的业务资源，只有该分布式事务才能使用，然而 Seata 框架认为该分布式事务已经结束，也就是说，当出现这种情况时，该分布式事务第一阶段预留的业务资源就再也没有人能够处理了，对于这种情况，我们就称为悬挂，即业务资源预留后没法继续处理。

什么样的情况会造成悬挂呢？按照前面所讲，在 RPC 调用时，先注册分支事务，再执行 RPC 调用，如果此时 RPC 调用的网络发生拥堵，通常 RPC 调用是有超时时间的，RPC 超时以后，发起方就会通知 TC 回滚该分布式事务，可能回滚完成后，RPC 请求才到达参与者，真正执行，从而造成悬挂。

怎么实现才能做到防悬挂呢？根据悬挂出现的条件先来分析下，悬挂是指二阶段 Cancel 执行完后，一阶段才执行。也就是说，为了避免悬挂，如果二阶段执行完成，那一阶段就不能再继续执行。因此，当一阶段执行时，需要先检查二阶段是否已经执行完成，如果已经执行，则一阶段不再执行；否则可以正常执行。那怎么检查二阶段是否已经执行呢？大家是否想到了刚才解决空回滚和幂等时用到的事务控制表，可以在二阶段执行时插入一条事务控制记录，状态为已回滚，这样当一阶段执行时，先读取该记录，如果记录存在，就认为二阶段已经执行；否则二阶段没执行。





## **异常控制实现**

一个 TCC 接口如何完整的解决这三个问题。



**首先是** **Try** **方法。**结合前面讲到空回滚和悬挂异常，Try 方法主要需要考虑两个问题，一个是 Try 方法需要能够告诉二阶段接口，已经预留业务资源成功。第二个是需要检查第二阶段是否已经执行完成，如果已完成，则不再执行。

先插入事务控制表记录，如果插入成功，说明第二阶段还没有执行，可以继续执行第一阶段。如果插入失败，则说明第二阶段已经执行或正在执行，则抛出异常，终止即可。



**接下来是** **Confirm** **方法。**因为 Confirm 方法不允许空回滚，也就是说，Confirm 方法一定要在 Try 方法之后执行。因此，Confirm 方法只需要关注重复提交的问题。可以先锁定事务记录，如果事务记录为空，则说明是一个空提交，不允许，终止执行。如果事务记录不为空，则继续检查状态是否为初始化，如果是，则说明一阶段正确执行，那二阶段正常执行即可。如果状态是已提交，则认为是重复提交，直接返回成功即可；如果状态是已回滚，也是一个异常，一个已回滚的事务，不能重新提交，需要能够拦截到这种异常情况，并报警。



**最后是** **Cancel** **方法。**因为 Cancel 方法允许空回滚，并且要在先执行的情况下，让 Try 方法感知到 Cancel 已经执行，所以和 Confirm 方法略有不同。首先依然是锁定事务记录。

如果事务记录为空，则认为 Try 方法还没执行，即是空回滚。空回滚的情况下，应该先插入一条事务记录，确保后续的 Try 方法不会再执行。如果插入成功，则说明 Try 方法还没有执行，空回滚继续执行。如果插入失败，则认为Try 方法正再执行，等待 TC 的重试即可。

如果一开始读取事务记录不为空，则说明 Try 方法已经执行完毕，再检查状态是否为初始化，如果是，则还没有执行过其他二阶段方法，正常执行 Cancel 逻辑。如果状态为已回滚，则说明这是重复调用，允许幂等，直接返回成功即可。如果状态为已提交，则同样是一个异常，一个已提交的事务，不能再次回滚。



## **TCC 性能优化**



### **同库模式**

第一个优化方案是改为同库模式。同库模式简单来说，就是分支事务记录与业务数据在相同的库中。什么意思呢？之前提到，在注册分支事务记录的时候，框架的调用方切面会先向 TC 注册一个分支事务记录，注册成功后，才会继续往下执行 RPC 调用。TC 在收到分支事务记录注册请求后，会往自己的数据库里插入一条分支事务记录，从而保证事务数据的持久化存储。那同库模式就是调用方切面不再向 TC 注册了，而是直接往业务的数据库里插入一条事务记录。



在讲解同库模式的性能优化点之前，先给大家简单讲讲同库模式的恢复逻辑。一个分布式事务的提交或回滚还是由发起方通知 TC，但是由于分支事务记录保存在业务数据库，而不是 TC 端。因此，TC 不知道有哪些分支事务记录，在收到提交或回滚的通知后，仅仅是记录一下该分布式事务的状态。那分支事务记录怎么真正执行第二阶段呢？需要在各个参与者内部启动一个异步任务，定期捞取业务数据库中未结束的分支事务记录，然后向 TC 检查整个分布式事务的状态。TC 在收到这个请求后，会根据之前保存的分布式事务的状态，告诉参与者是提交还是回滚，从而完成分支事务记录。



那这样做有什么好处呢？采用同库模式前，在每次调用一个参与者的时候，都是先向 TC 注册一个分布式事务记录，TC 再持久化存储在自己的数据库中，也就是说，一个分支事务记录的注册，包含一次 RPC 和一次持久化存储。

优化后，每次调用一个参与者的时候，都是直接保存在业务的数据库中，从而减少与 TC 之间的 RPC 调用。优化后，有多少个参与者，就节约多少次 RPC 调用。



这就是同库模式的性能方案。把分支事务记录保存在业务数据库中，从而减少与 TC 的 RPC 调用。



### **异步化**

TCC 模型的一个作用就是把两阶段拆分成了两个独立的阶段，通过资源业务锁定的方式进行关联。资源业务锁定方式的好处在于，既不会阻塞其他事务在第一阶段对于相同资源的继续使用，也不会影响本事务第二阶段的正确执行。从理论上来说，只要业务允许，事务的第二阶段什么时候执行都可以，反正资源已经业务锁定，不会有其他事务动用该事务锁定的资源。

假设只有一个中间账户的情况下，每次调用支付服务的 Commit 接口，都会锁定中间账户，中间账户存在热点性能问题。

但是，在担保交易场景中，七天以后才需要将资金从中间账户划拨给商户，中间账户并不需要对外展示。因此，在执行完支付服务的第一阶段后，就可以认为本次交易的支付环节已经完成，并向用户和商户返回支付成功的结果，并不需要马上执行支付服务二阶段的 Commit 接口，等到低锋期时，再慢慢消化，异步地执行。





## 主要模块

### **两个拦截器**

- 通过对 @Tyloo AOP 切面( 参与者 try 方法 )进行拦截，透明化对参与者confirm / cancel 方法调用，从而实现 TCC
- **第一个拦截器，可补偿事务拦截器**
- 在 Try 阶段，对事务的发起、传播。
	- 在 Confirm / Cancel 阶段，对事务提交或回滚。
	- 具体方法
	
	- 			发起根事务 
		提供 begin() 方法，发起根事务。该方法在调用方法类型为 MethodType.ROOT 并且 事务处于 Try 阶段被调用
	- 			传播发起分支事务
		该方法在调用方法类型为MethodType.PROVIDER 并且 事务处于 Try 阶段被调用
	- 			传播获取分支事务
		该方法在调用方法类型为MethodType.PROVIDER 并且 事务处于 Confirm / Cancel 阶段被调用
	- 			提交事务
		该方法在事务处于 Confirm / Cancel 阶段被调用。
	- 			回滚事务
		该方法在事务处于 Confirm / Cancel 阶段被调用。
	
- **第二个拦截器，资源协调者拦截器**
- 在 Try 阶段，添加参与者到事务中。当事务上下文不存在时，进行创建。
	- 具体方法
	
	- 			添加参与者到事务
		该方法在事务处于 Try 阶段被调用

### **事务与参与者**

- TCC 通过多个参与者的 try / confirm / cancel 方法，实现事务的最终一致性
- Tyloo 将每个业务操作抽象成事务参与者

### **事务管理器**

- 提供事务的获取、发起、提交、回滚，参与者的新增等等方法。

### **事务存储器**

- 提供对事务对象的持久化

### **事务注解**

- 传播级别

	- 默认default Propagation.REQUIRED

- 确认执行业务方法
- 取消执行业务方法
- 事务上下文编辑

	- 通过对目标对象的方法上的参数获取事务上下文
	- 事务上下文

		- 事务编号
		- 事务状态
		- 附加属性

			- Attachment

		- public class DubboTransactionContextLoader implements TylooTransactionContextLoader 

			- Dubbo隐式传参方式
			-         RpcContext.getContext().setAttachment(TransactionContextConstants.TRANSACTION_CONTEXT, JSON.toJSONString(transactionContext));
			-         String context = RpcContext.getContext().getAttachment(TransactionContextConstants.TRANSACTION_CONTEXT);

## **技术概念**

### **Spring AOP**



### Dubbo 代理

- @SPI指定默认使用javassist字节码技术来生成代理对象

  - 比如我们有几个接口Car:

    Dubbo首先生成Car的一个实现类proxy0，每个方法都代理给InvocationHandler处理：

    然后生成一个Proxy的子类Proxy0，可以用于创建代理类的实例（过newInstance可以创建代理类proxy0的实例）

- `confirmMethod`、`cancelMethod` 使用和 try 方法**相同方法名**：**本地发起**远程服务 TCC confirm / cancel 阶段，调用相同方法进行事务的提交或回滚。远程服务的 CompensableTransactionInterceptor 会根据事务的状态是 CONFIRMING / CANCELLING 来调用对应方法。

- `org.mengyun.tcctransaction.dubbo.proxy.javassist.TccJavassistProxyFactory`，TCC Javassist 代理工厂。实现代码如下：

  

  ```
  public class TccJavassistProxyFactory extends JavassistProxyFactory {
  
      @SuppressWarnings("unchecked")
      public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
          return (T) TccProxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
      }
  
  }
  ```

  

  - **项目启动时**，调用 `TccJavassistProxyFactory#getProxy(...)` 方法，生成 Dubbo Service 调用 Proxy。
  - `com.alibaba.dubbo.rpc.proxy.InvokerInvocationHandler`，Dubbo 调用处理器，点击[连接](https://github.com/alibaba/dubbo/blob/17619dfa974457b00fe27cf68ae3f9d266709666/dubbo-rpc/dubbo-rpc-api/src/main/java/com/alibaba/dubbo/rpc/proxy/InvokerInvocationHandler.java)查看代码。

- ### TccProxy & TccClassGenerator

  - `TccProxy`，TCC Proxy 工厂，生成 Dubbo Service 调用 Proxy 。
  - `TccClassGenerator`，TCC 类代码生成器，基于 Javassist 实现。
    - ClassGenerator是dubbo提供的基于javassist之上的封装，方便dubbo用于生成字节码操作，ClassGenerator主要用来收集java的类信息如接口，字段，方法，构造器等等信息，

- `TccProxy#getProxy(...)` 方法获得 **TCC Proxy 工厂**

  - **生成 Dubbo Service 调用 Proxy 的代码**
    - 调用 `TccClassGenerator#newInstance(loader)` 方法， 创建生成 Dubbo Service 调用 **Proxy** 的代码生成器。
    - 处理接口。
    - 添加方法签名到已处理方法签名集合。多个接口可能存在相同的接口方法，跳过相同的方法，避免冲突。
    - 生成 Dubbo Service 调用实现代码
    - 调用 `TccClassGenerator#addMethod(...)` 方法，添加生成的方法
    - 调用 `TccClassGenerator#toClass()` 方法，**生成类**
      - 基于 Javassist 生成类
      - 设置 @Compensable 默认属性。
  - 创建 Proxy

- **以上三个类参考了 Dubbo 自带的实现：**

  - com.alibaba.dubbo.common.bytecode.Proxy
  - com.alibaba.dubbo.common.bytecode.ClassGenerator
  - com.alibaba.dubbo.common.bytecode.Wrapper

- Dubbo Service Proxy 提供了两种生成方式
  - 将 Dubbo Service 方法上的注解 @Tyloo ，自动生成注解的 confirmMethod、cancelMethod、TylooTransactionContextLoader 属性
  - 如果是使用动态代理的方式实现aop(默认方式）,则confirmMethod和cancelMethod需在接口类中声明
  - 如果使用动态字节码技术实现aop（如指定aspectj-autoproxy的proxy-target-class属性为true,在1.2.1版本中，默认已设置为true),则无需在接口类中声明。

- TccJavassistProxyFactory extends JavassistProxyFactory

	-     public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) TccProxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
}

- **TccJavassistProxyFactory：TCC Javassist 代理工厂**

	- 项目启动时，调用 TccJavassistProxyFactory#getProxy(...) 方法，生成 Dubbo Service 接口的代理对象。
	- TccProxy & TccClassGenerator

		- TCC Proxy 工厂，生成 Dubbo Service 调用 Proxy
		- TCC 类代码生成器，基于 Javassist 实现。

			- ClassGenerator是dubbo提供的基于javassist之上的封装，方便dubbo用于生成字节码操作，ClassGenerator主要用来收集java的类信息如接口，字段，方法，构造器等等信息，

	- **调用 TccProxy#getProxy(...) 方法，获得 TCC Proxy 工厂**
	
	

### **隐式传参**

- 通过 Dubbo 的隐式传参的方式，避免在 Dubbo Service 接口上声明 TylooTylooTransactionContext 参数，对接口产生一定的入侵
- 				get方法
		{
			Object obj = RpcContext.getContext().getAttachments();
			获取客户端隐式传入的参数，用于框架集成
		}
		set方法
		{	
			RpcContext.getContext().setAttachment("index", "1"); // 隐式传参，后面的远程调用都会隐式将这些参数发送到服务器端，类似cookie
		}





# 项目流程细节

## 主业务执行

### **拦截器**

- 补偿事务拦截器
- 资源协调员拦截器 

### **两个切面**

#### **第一个拦截器**

- 因为贴有@Compensable注解，所以进入第一个切面，进行环绕增强，调用第一个拦截器

	- public Object interceptTylooMethod(ProceedingJoinPoint pjp)

- 从拦截方法的参数中获取事务上下文

	-  TylooMethodContext tylooMethodContext = new TylooMethodContext(pjp);

- 获取方法类型

	- 通过该方法事务传播级别获取方法类型
	- ROOT

		- 无事务上下文并且传播级别为REQUIRED

			- 如果本来有事务，则加入该事务，如果没有事务，则创建新的事务

		- 传播级别为REQUIRES_NEW

			- 如果当前存在事务，那么将当前的事务挂起，并开启一个新事务去执行REQUIRES_NEW标志的方法

	- PROVIDE

		- 传播级别为REQUIRED

	- NORMAL

		- 动态代理对象专用
		- 传播级别为SUPPORT

- 为ROOT，调用rootMethodProceed方法

	- TransactionManager事务管理器
	- begin(tylooMethodContext.getUniqueIdentity())

		- getUniqueIdentity()

			- 遍历事务注解参数获取唯一标识
			- 标识注解定义在主业务方法上

		-         TylooTransaction tylooTransaction = new TylooTransaction(uniqueIdentify, TransactionType.ROOT);
transactionRepository.create(tylooTransaction);
registerTransaction(tylooTransaction);
return tylooTransaction;
		- 先创建一个根环境的事务对象

			- 把唯一标识和ROOT事务类型传进去

				-             this.globalTransactionId = CUSTOMIZED_TRANSACTION_ID;
    this.branchQualifier = uniqueIdentity.toString().getBytes();

			-         this.xid = new TylooTransactionXid(uniqueIdentity);
this.transactionStatus = TransactionStatus.TRYING;
this.transactionType = transactionType;

		- 将创建的事务对象保存在本地

			- transactionRepository对象通过注入的方式获得，其实这个就是连接tcc数据库的dao
			- 真实对象在本地项目tyloo-dubbo-order中的config.spring.local中appcontext-service-tcc.xml有配置，其真实类型是SpringJdbcTransactionRepository
			- transactionRepository对象调用create方法将创建的事务对象保存在本地，其最终调用的是JdbcTransactionRepository类中的doCreate方法，实际上就是将序列化后的事务对象和一些相关信息保存在数据库中

				- 用SpringBuilder拼接

		- 注册事务到当前线程事务队列

			- CURRENT.get().push(tylooTransaction);
- **try{}**
	-  // Try (开始执行被拦截的方法)
	returnValue = tylooMethodContext.proceed();
	
	- 实际上调用的还是 return this.pjp.proceed();
	
	- TylooMethodContext
	
		- 通过pjp封装了注解方法上下文
	
			- 切入点
			- 注解方法
				- 注解
				- 传播级别
				- 事务上下文
	- **catch**
	- tylooTransactionManager.rollback();
	
		-  // 回退 事务
	 tylooTransaction.rollback();
	
	-         tylooTransaction.changeStatus(TransactionStatus.CANCELLING);
		transactionRepository.update(tylooTransaction);
- 更改事务状态为CANCELLING，并更新事务
		
		-  public void rollback() {
	    for (Participant participant : participants) {
	    participant.rollback();
	    }
      - 遍历每个参与者，通过反射机制调用各自的cancel方法
	    -    public void rollback() {
	    terminator.invoke(new TylooTransactionContext(xid, TransactionStatus.CANCELLING.getId()), cancelInvocationContext, tylooContextLoaderClass);
              		    }
	-   // 删除 事务    transactionRepository.delete(tylooTransaction);
- **finally**
	- tylooTransactionManager.cleanAfterCompletion(tylooTransaction);
		
	- 线程池的一个线程使用完ThreadLocal对象之后，再也不用，由于线程池中的线程不会退出，线程池中的线程的存在，同时ThreadLocal变量也会存在，占用内存！造成OOM溢出！
	
		-  ExecutorService executorService = Executors.newFixedThreadPool(THREAD_LOOP_SIZE);
		for (int i = 0; i < THREAD_LOOP_SIZE; i++) {
executorService.execute(() -> {
	      threadLocal.set(new ThreadLocalOOMDemo().addBigList());
  
		- 由于ThreadLocalMap的生命周期跟Thread一样长，如果没有手动删除对应key就会导致内存泄漏
		- 每次使用完ThreadLocal，都调用它的remove()方法，清除数据
		
		-                 CURRENT.get().pop();
		    if (CURRENT.get().size() == 0) {
          CURRENT.remove();
        }
    
	- 采用ThreadLocal保存线程
		
			- 因为多个线程不能共用一个事务
			- 比如线程池调用多个线程执行不同的主业务方法（全局事务ID不同），不能共享事务，使用ThreadLocal为每个线程创建一个事务队列副本
				- 在Spring中，bean可以被定义为两种模式：prototype（多例）和singleton（单例）
  	
  			- singleton（单例）：只有一个共享的实例存在，所有对这个bean的请求都会返回这个唯一的实例。
  				- prototype（多例）：对这个bean的每次请求都会创建一个新的bean实例，类似于new。
	
	- 将事务从当前线程事务队列移除
	
	-             // Try检验正常后提交(事务管理器在控制提交)         tylooTransactionManager.commit(asyncConfirm);
	
		- 跟回退事务一样

#### **pjp.proceed()方法继续执行这条执行链**

- 当使用环绕通知时，这个方法必须调用，否则拦截到的方法就不会再执行了

	- 环绕通知要注意的是方法的参数及抛出异常类型的固定写法（方法名可以是任意得），另在该方法中必须执行pjp.proceed()才能让环绕通知中的两处打印代码得以执行。即是说要想环绕通知的拦截处理代码起作用必须调用pjp.proceed方法。 补充：环绕通知通常可以用来测试方法的执行时间，在pjp.proceed前获取一个时间，在pjp.proceed方法后再获取一个时间。最后两个时间相减即可得方法执行时间。

- **接着被第二个拦截器拦截**

  

#### **第二个拦截器**

-         // 获取当前事务
TylooTransaction tylooTransaction = tylooTransactionManager.getCurrentTransaction();
- switch判断事务状态，此时为TRYING
- 添加根环境参与者

	-    //获取到调用的真实方法的签名
MethodSignature signature = (MethodSignature) pjp.getSignature();
	- 然后获取事务对象，创建事务id，获取目标对象方法，参数，注解等
	-     //获取注解上的confirmMethodName和cancelMethodName参数值(对应的是业务的确认方法的方法)
String confirmMethodName = compensable.confirmMethod();
String cancelMethodName = compensable.cancelMethod();
	-     //通过pjp参数反射获取到真实方法调用对象（其实就是PlaceOrderServiceImpl对象字节码）
Class targetClass = ReflectionUtils.getDeclaringType(pjp.getTarget().getClass(), method.getName(), method.getParameterTypes());
	-    封装InvocationContext（ 构建确认/取消方法的提交上下文），把confirmMethod和cancelMethod需要通过反射进行调用的信息存储起来，后期通过反射的方式去调用confirmMethod，实际上就可以看做是Method

	  - 目标类
	  - confirm/cancel方法
	
- 创建事务参与者
	
	- 事务id
		- confirmInvocation,cancelInvocation
		- 事务上下文编辑加载器
	
- 把transaction更新到数据库中
	
	- 将新的transaction更新到tcc数据库，这时的transaction对象已经添加了根环境(Order)的参与者

### 此时正常执行业务逻辑

- 主业务俩注解 一个@Tyloo一个@Transactional本地事务注解
-     //修改订单状态
order.pay(redPacketPayAmount, capitalPayAmount);
//更新本地数据库(注意此时还在事务中，还未提交,相当于try操作)
orderRepository.updateOrder(order);
- 创建当前主环境(Order)的其他消费参与者

	- capitalTradeOrderService和redPacketTradeOrderService是dubbo动态代理出来本地代理类      
String result = capitalTradeOrderService.record(null, buildCapitalTradeOrderDto(order));
	- 由于此时的capitalTradeOrderService是本地的一个代理类，所以这个record方法实际上是本地代理对象中的一个方法，在这个方法里中才通过dubbo(RPC)调用远程的record业务方法返回结果,真正的业务方法的实现是在tyloo-dubbo-capital模块内

## 代理对象执行

- **调用的从业务的方法为从业务的代理对象，代理对象去调用真正的方法**

#### 第一个拦截器

- switch

	-            default:
          return pjp.proceed();

#### 第二个拦截器

-         // 获取当前事务
  TylooTransaction tylooTransaction = tylooTransactionManager.getCurrentTransaction();
- switch (tylooTransaction.getTransactionStatus()) 

	- switch中为TRING

- 跟上面一样，还是添加参与者，此时当前线程有两个参与者

#### 调用真正的业务方法

- 调用远程try方法

## 从业务执行

### 调用前被拦截

- 第一个拦截器

	- 事务上下文传过来了
	- 计算出事务类型为PROVIDER
	- 走到switch

		- case TRYING

			- 传播发起分支事务

				-  TylooTransaction(TylooTransactionContext tylooTransactionContext) 参数为事务上下文

					- ROOT时走的是TylooTransaction(TransactionType transactionType) 参数为事务类型
					- 设置事务id，设置事务状态为TRYING，设置事务类型为BRANCH

			- 持久化当前事务
			- 注册事务到当前线程事务队列

	- return tylooMethodContext.proceed();

- 第二个拦截器

	- case TRYING

		- 添加事务参与者

			- 跟ROOT事务时一样

		- 此时当前线程只有一个参与者

### 执行业务逻辑



## 所有从业务执行完后

- **此时远程事务的TRYING阶段执行完毕**
- 此时参与者已经提交
	
- **此时又回到了根事务类型下TylooInterceptor类**
- returnValue = tylooMethodContext.proceed();这一步执行完了 接着往下执行
- **tylooTransactionManager.commit();**
- 获取事务（获取的是根事务当前的事务）
	
	- 此时队列中有三个事务：根事务（主业务）对象，从业务代理对象的事务对象A,从业务代理对象的事务对象B
	
- 设置 事务状态 为 CONFIRMING
	- 更新 事务
	- 提交事务
	
	- for循环各个参与者调用commit方法
	
		- invoke方法（不是反射的那个方法）
	
			- 参数为事务上下文，事务上下文编辑器，提交方法的invocationHandler（confirmInvocationContext）
	
				- InvocationContext类
	
					- 目标对象
						- 方法名字
						- 方法类型
						- 方法参数
	
			- 获得目标类的一个实例 target
				- 获取目标方法 method
				- 通过事务上下文编辑器注入事务上下文，目标对象，目标方法，参数
				- invoke反射调用目标方法
	
				- method.invoke(target, invocationContext.getArgs());
					- 调用confirm方法
					- 调用后再被拦截
	
			- 所有confirm方法均为反射调用
	
- **tylooTransactionManager.cleanAfterCompletion(tylooTransaction);**
- 最后从队列中移除当前事务（根事务）
	- 要等到所有参与者都执行完comiit方法后才执行

## 主业务执行comfirm方法后

- 走到第二个循环，参与者提交
- 此时参与者是代理对象

## 代理对象执行comfirm方法

- 此时因为代理对象@Tyloo注解中confirmMethod/cancelMethod中均和try方法的名字相同，所以此时依旧调代理对象的try方法
- 第一个拦截器

	- swith

		-             default:
            return pjp.proceed(); 

- 第二个拦截器

	- 因为此时状态为COMFIRMING 直接break了

- 调用业务方法

	- 还是从业务的try方法，接着被拦截，但是此时try方法的事务类型已经为CONFIRMING了

## 然后从业务接着执行confirm方法

- 从业务try方法保证幂等，如果执行过了直接跳过
- 第一个拦截器

	- switch PROVIDER

		- 传播获取分支事务

			- 根据全局事务ID和分支事务id获取分支事务
			- 设置事务状态为发来的事务上下文中的状态

				- 此时为CONFIRMING

			- 注册事务到事务队列中

				- 此时队列中只有一个事务

	- propagationExistBegin

		- 根据代理对象发来的分支事务上下文获取分支事务
		- 更新事务状态为CONFIRMING
		- 更新事务
		- 提交后删除事务

	- commit(asyncConfirm)

		- 最终调用invoke反射cofirmmethod中的方法，也就是从业务真实的confirm方法

- 第二个拦截器

	- break；

## 最终主业务删除根事务和移除事务队列中的事务

- 分支事务在事务队列中相继移除，最后移除根事务





# 事务存储器

- 将事务信息添加到内存中的同时，会使用外部存储进行持久化

	- TylooTransaction 是一个比较复杂的对象，内嵌 Participant 数组，而 Participant 本身也是复杂的对象，内嵌了更多的其他对象，
	- 因此，存储器在持久化 TylooTransaction 时，需要序列化后才能存储。

- TylooTylooTransactionRepository，事务存储器接口。不同的存储器通过实现该接口，提供事务的增删改查功能

	- CachableTylooTylooTransactionRepository，可缓存的事务存储器抽象类，实现增删改查事务时，同时缓存事务信息

		- 使用 Guava Cache 内存缓存事务信息，设置最大缓存个数为 1000 个，缓存过期时间为最后访问时间 120 秒
		- 若更新成功后，调用 #putToCache(...) 方法，添加事务到缓存。
		- 若更新失败后，抛出 OptimisticLockException 异常

			- 有两种情况会导致更新失败：

				- (1) 该事务已经被提交，被删除；
				- (2) 乐观锁更新时，缓存的事务的版本号( TylooTransaction.version )和存储器里的事务的版本号不同，更新失败。

					- sql脚本是有对当前的version做判断的，如果version不匹配是，update不成功的，这个就是版本控制的逻辑

				- 更新失败，意味着缓存已经不不一致，调用 #removeFromCache(...) 方法，移除事务从缓存中。

		- 添加和更新 TylooTransaction 时，使用 Redis HSETNX，不存在当前版本的值时，进行设置，重而实现类似乐观锁的更新。
		- 读取 TylooTransaction 时，使用 Redis HGETALL，将 TylooTransaction 所有 version 对应的值读取到内存后，取 version 值最大的对应的值。

- **JDBC**
- JdbcTransactionRepository extends CachableTransactionRepository 
	- 字段
	
	-     domain，领域，或者也可以称为模块名，应用名，用于唯一标识一个资源。例如，Maven 模块 xxx-order，我们可以配置该属性为 ORDER。
		-     tbSuffix，表后缀。默认存储表名为 TCC_TRANSACTION，配置表名后，为 TCC_TRANSACTION${tbSuffix}。
		-     dataSource，存储数据的数据源。
		-     serializer，序列化。当数据库里已经有数据的情况下，不要更换别的序列化，否则会导致反序列化报错。建议：存储时，新增字段，记录序列化的方式。
	
- 表结构
	
	-     TRANSACTION_ID，仅仅数据库自增，无实际用途。
	CONTENT，Transaction 序列化。
  
- **Redis**
- 字段
	
	- 
	keyPrefix，key 前缀。类似 JdbcTransactionRepository 的 domain 属性。

一个事务存储到 Reids，使用 Redis 的数据结构为 HASHES。
			- key : 使用 keyPrefix + xid

		- getRedisKey(String keyPrefix, Xid xid)
		- HASHES 的 key ：使用 version
	
			-     添加和更新 Transaction 时，使用 Redis HSETNX，不存在当前版本的值时，进行设置，重而实现类似乐观锁的更新。
			-     读取 Transaction 时，使用 Redis HGETALL，将 Transaction 所有 version 对应的值读取到内存后，取 version 值最大的对应的值。
	
		- HASHES 的 value ：调用 TransactionSerializer#serialize(...) 方法，序列化 Transaction。
	
			- 属性都存到MAP，key为数据库列名，value为实际事务属性

- ```java
  create
  Object result = jedis.eval("if redis.call('exists', KEYS[1]) == 0 then redis.call('hmset', KEYS[1], unpack(ARGV)); return 1; end; return 0;".getBytes(),Arrays.asList(RedisHelper.getRedisKey(keyPrefix, transaction.getXid())), params);
  
  
  update
  Object result = jedis.eval(String.format("if redis.call('hget',KEYS[1],'VERSION') == '%s' then redis.call('hmset', KEYS[1], unpack(ARGV)); return 1; end; return 0;",transaction.getVersion() - 1).getBytes(),Arrays.asList(RedisHelper.getRedisKey(keyPrefix, transaction.getXid())), params);
  
                  return (Long) result;
  ```

# 事务恢复

- **两个类**
- recover
	
	-     RecoverConfig，事务恢复配置接口
	-     TransactionRecovery，事务恢复逻辑
	
- spring.recover
	
	-     DefaultRecoverConfig，默认事务恢复配置实现
	-     RecoverScheduledJob，事务恢复定时任务
	
- **概念**
- 事务信息被持久化到外部的存储器中。事务存储是事务恢复的基础。通过读取外部存储器中的异常事务，定时任务会按照一定频率对事务进行重试，直到事务完成或超过最大重试次数。
	
- **事务重试配置**
- RecoverConfig，事务恢复配置接口
	
	- 最大重试次数
		- 单个事务恢复重试的间隔时间，单位：秒
		- cron 表达式
		- 延迟取消异常集合
		- 设置延迟取消异常集合
	
- DefaultRecoverConfig，默认事务恢复配置实现
	
	-     maxRetryCount，单个事务恢复最大重试次数 为 30。
		-     recoverDuration，单个事务恢复重试的间隔时间为 120 秒。
		-     cronExpression，定时任务 cron 表达式为 "0 */1 * * * ?"，每分钟执行一次。如果你希望定时任务执行的更频繁，可以修改 cron 表达式，例如 0/30 * * * * ?，每 30 秒执行一次。
		- **delayCancelExceptions，延迟取消异常集合**。在 DefaultRecoverConfig 构造方法里，预先添加了 OptimisticLockException / SocketTimeoutException 。 
	
		-         delayCancelExceptions.add(OptimisticLockException.class);
	    delayCancelExceptions.add(SocketTimeoutException.class);
  
- 针对 SocketTimeoutException 

  - try 阶段，本地参与者调用远程参与者( 远程服务，例如 Dubbo，Http 服务)，远程参与者 try 阶段的方法逻辑执行时间较长，超过 Socket 等待时长，发生 SocketTimeoutException，如果立刻执行事务回滚，远程参与者 try 的方法未执行完成，可能导致 cancel 的方法实际未执行( try 的方法未执行完成，数据库事务【非 TCC 事务】未提交，cancel 的方法读取数据时发现未变更，导致方法实际未执行，最终 try 的方法执行完后，提交数据库事务【非 TCC 事务】，较为极端 )，最终引起数据不一致。
    - 在事务恢复时，会对这种情况的事务进行取消回滚，如果此时远程参与者的 try 的方法还未结束，还是可能发生数据不一致。 

  -     为什么tcc事务切面中对乐观锁与socket超时异常不做回滚处理，只抛异常
        - 不立即回滚，主要考虑是被调用服务方存在一直在正常执行的可能，只是执行的慢，导致了调用方超时，此时如果立即回滚，在被调用方执行cancel操作的同时，被调用方的try方法还在执行，甚至cancel操作执行完了，try方法还没结束，这种情况下业务数据存在不一致的可能。目前解决办法是这类异常不立即回滚，而是由恢复job执行回滚，恢复job会在一段时间后再去调用该被调用方的cancel方法，这个时间可在RecoverConfig中设置，默认120s。

- 针对 OptimisticLockException

	- 还是 SocketTimeoutException 的情况，事务恢复间隔时间小于 Socket 超时时间，此时事务恢复调用远程参与者取消回滚事务，远程参与者下次更新事务时，会因为乐观锁更新失败，抛出 OptimisticLockException。如果 CompensableTransactionInterceptor 此时立刻取消回滚，可能会和定时任务的取消回滚冲突，因此统一交给定时任务处理。
		- 使用版本号控制，事务超时了，恢复机制将尝试进行事务恢复，恢复时会更新版本号（乐观锁）以获得事务控制权，原来的事务因为失去了控制权，则在更新事务状态时失败，并终止事务继续进行。
	
- 在资源准备过程中 和 资源确认、回滚、或者调用调度任务都会走 CachableTransactionRepository的update方法。
		
		- 先doUpdate 改状态，乐观锁控制，改成功了 返回1 正常执行流程，改失败了，会抛出OptimisticLockException异常
		
	- 并发其实利用由数据库 update 乐观锁来控制的。同一时刻，只能有一个成功更新状态；
		update后，即使再去调用 confirm 去提交，幂等性保证不会出现问题，即 confirm 多次也能保证数据的一致性。
		
	- - ```java
		  在资源准备过程中 和 资源确认、回滚、或者调用调度任务都会走 CachableTransactionRepository的update方法。下面是方法体。
		  int result = 0;
		  try {
		  result = doUpdate(transaction);
		  if (result > 0) {
		  putToCache(transaction);
		  } else {
		  throw new OptimisticLockException();
		  }
		  } finally {
		  if (result <= 0) {
		  removeFromCache(transaction);
		  }
		  }
		  return result;
		  
		  先doUpdate 改状态，就是作者所得乐观锁，改成功了 返回1 正常执行流程，改失败了，会抛出OptimisticLockException异常
		  ```
		
		  
	
- 事务重试定时任务
	
	- RecoverScheduledJob，事务恢复定时任务，基于 Quartz 实现调度，不断不断不断执行事务恢复
		- 用 MethodInvokingJobDetailFactoryBean#setConcurrent(false) 方法，禁用任务并发执行。
		- 调用 MethodInvokingJobDetailFactoryBean#setTargetObject(...) + MethodInvokingJobDetailFactoryBean#setTargetMethod(...) 方法，设置任务调用 TransactionRecovery#startRecover(...) 方法执行。
		
	- **如果应用集群部署，会不会相同事务被多个定时任务同时重试**
	
	  - 答案是不会，事务在重试时会乐观锁更新，同时只有一个应用节点能更新成功。
	
	  - 通过乐观锁解决，只有一个节点能获得控制权来处理，其它节点不能处理则忽略
	
	  - 当然极端情况下，Socket 调用超时时间大于事务重试间隔，第一个节点在重试某个事务，一直未执行完成，第二个节点已经可以重试。
	
	    ps：建议，Socket 调用超时时间小于事务重试间隔。
	
- 是否定时任务和应用服务器解耦？
	
	- 蚂蚁金服的分布式事务服务 DTS 采用 client-server 模式：
	
		-     xts-client ：负责事务的创建、提交、回滚、记录。
			-     xts-server ：负责异常事务的恢复。
	
- **异常事务恢复**
- TransactionRecovery，异常事务恢复 
	- startRecover(...) 方法
	- 调用 `#loadErrorTransactions()` 方法，加载异常事务集合。
	
		- 异常事务的定义：当前时间 - 事务变更时间( 最后执行时间 ) >= 事务恢复间隔( RecoverConfig#getRecoverDuration() )。
	
			- 这里有一点要注意，已完成的事务会从事务存储器删除。事务完成/回滚后会删除事务日志，所以取出的都是没完成的和根事务
				- 一个事务日志当超过一定时间间隔后没有更新就会被认为是发生了异常，需要恢复
				- 恢复Job将扫描超过这个时间间隔依旧没有更新的事务日志，并对这些事务进行恢复，时间单位是秒，默认是120秒
	
		-         return transactionRepository.findAllUnmodifiedSince(new Date(currentTimeInMillis - recoverConfig.getRecoverDuration() * 1000));
	
			- 查询出了所有的超时服务
	- 恢复错误的事务集合
	- 当单个事务超过最大重试次数时，不再重试，只打印异常，此时需要**人工介入**解决。可以接入 ELK 收集日志监控报警。
		- 当**分支事务**超过最大可重试时间时，不再重试。可能有同学和我一开始理解的是相同的，实际**分支事务**对应的应用服务器也可以重试**分支事务**，不是必须**根事务**发起重试，从而一起重试**分支事务**。这点要注意下。
	- 当事务处于 TransactionStatus.CONFIRMING 状态时，提交事务，逻辑和 `TransactionManager#commit()` 类似。
		- 当事务处于 TransactionStatus.CONFIRMING 状态，或者**事务类型为根事务**，回滚事务，逻辑和 `TransactionManager#rollback()` 类似。这里加判断的**事务类型为根事务**，用于处理延迟回滚异常的事务的回滚。
	- 分支事务&&当前时间超过最大可重试时间，则跳过
		
	  - 当前事务创建时间+最大次数*间隔>当前时间
		- 根事务或分支事务
	
		  - 重试次数+1
	  	- 如果是CONFIRMING
		
		  	- 更改事务状态为COMFIRMING
	  		- commit
		  		- 删除事务
		
		  - 如果是CANCELING或为根事务
	
		  	- 更改事务状态为CANCELING
	  		- rollback
		  		- 删除事务



#### 延迟取消异常集合

​		delayCancelExceptions表示系统发生了设置的异常时，主事务不立即rollback，而是由恢复job来执行事务恢复。通常需要将超时异常设置为delayCancelExceptions，这样可以避免因为服务调用时发生了超时异常，主事务如果立刻rollback, 但是从事务还没执行完，从而造成主事务rollback失败。
​			

​		在 DefaultRecoverConfig 构造方法里，预先添加了 OptimisticLockException
​		针对 OptimisticLockException ：事务恢复间隔时间小于 Socket 超时时间，此时事务恢复调用远程参与者取消回滚事务，
​		远程参与者下次更新事务时，会因为乐观锁更新失败，抛出 OptimisticLockException。
​		如果 TylooTransactionInterceptor 此时立刻回滚，可能会和定时任务的回滚冲突，因此统一交给定时任务处理。
​	为什么tcc事务切面中对乐观锁与socket超时异常不做回滚处理，只抛异常? 
​			不立即回滚，主要考虑是被调用服务方存在一直在正常执行的可能，只是执行的慢，
​			导致了调用方超时，此时如果立即回滚，在被调用方执行cancel操作的同时，被调用方的try方法还在执行，
​			甚至cancel操作执行完了，try方法还没结束，这种情况下业务数据存在不一致的可能。
​			目前解决办法是这类异常不立即回滚，而是由恢复job执行回滚，恢复job会在一段时间后再去调用该被调用方的cancel方法，这个时间可在RecoverConfig中设置，默认120s。
​	

#### 事务重试定时任务

​	RecoverScheduledJob，事务恢复定时任务，基于 Quartz 实现调度，不断不断不断执行事务恢复。
​		事务在重试时会乐观锁更新，同时只有一个应用节点能更新成功。
​	TransactionRecovery对象中定时执行的方法 startRecover 在TransactionRecovery类中编写，主要是将超过设置存活时间的transaction对象从数据库查出

#### 异常事务恢复

​	恢复异常事务集合
​		当单个事务超过最大重试次数时，不再重试，只打印异常，此时需要人工介入解决。可以接入 ELK 收集日志监控报警。
​		当分支事务超过最大可重试时间时，不再重试。实际分支事务对应的应用服务器也可以重试分支事务，不是必须根事务发起重试，从而一起重试分支事务。这点要注意下。
​		当事务处于 TransactionStatus.CONFIRMING 状态时，提交事务，逻辑和 TylooTransactionManager#commit() 类似。
​		当事务处于 TransactionStatus.CONFIRMING 状态，或者事务类型为根事务，回滚事务，逻辑和 TylooTransactionManager#rollback() 类似。这里加判断的事务类型为根事务，用于处理延迟回滚异常的事务的回滚。

#### 若中途出现异常

​	如果是TRY的阶段出现了异常
​		在 rollback 方法中，类似 commit 方法，拿到当前事务的所有参与者，调用所有参与者 cancelMethod 方法，程序运行到参与者的根分布式事务环境时，也会去遍历本地参与者执行cancleMethod方法，最后回到根事务，删除根事务，数据回滚完毕

```
如果是confirm出现异常
	只要TRY通过了，那么数据一定可以操作，所以就一定会执行confirm，所以confirm不会出现操作数据库异常，那么这里的异常就有可能是服务器宕机，此时有对应的定时器来管理，confirm方法要实现幂等
	重试次数
	
如果是rollback异常	

	执行回滚 重试次数
```

try阶段执行完毕, root节点执行了commit, 那么所有节点将进入 commit阶段
commit失败会重试
如果try阶段实行失败, 所有paticipate会进入cancel, 失败后会重试

#### 事务恢复

使用版本号控制，事务超时了，恢复机制将尝试进行事务恢复，恢复时会更新版本号（乐观锁）以获得事务控制权，原来的事务因为失去了控制权，则在更新事务状态时失败，并终止事务继续进行。

version的问题，在doUpdate的时候，sql脚本是有对当前的version做判断的，如果version不匹配是，update不成功的，这个就是版本控制的逻辑。1.2.x的Repository都提供了Cache的功能，在更改之后，都会进行Cache。

# 整体架构（流程、超时、幂等）

- **正常流程**
- 因为对远程业务的调用需要用到代理对象，代理对象由dubbo service生成，TRY阶段在进行远程调用前需要调用代理对象（代理对象的confrimmethod和cancelmethod均和try方法名字相同），此时拦截器会进行拦截代理对象，拦截后调用远程业务，远程业务（远程业务本地有个本地事务保证执行成功）也会被拦截，然后远程业务执行。COMMIT阶段时主业务执行完comfirm方法后代理对象执行comfirm方法（confirm方法均为反射调用），但是代理对象的confirm方法还是代理对象的try方法，此时再被拦截器拦截，因为此时代理对象的事务状态已经改为CONFIRMING，由于事务类型为defalut（因为代理对象的传播级别默认为SUPPORT），所以直接过了。此时再调用远程业务的try方法，走拦截，invoke反射的时候反射远程业务的confrim方法，因为try方法做了幂等所以直接过了，此时远程业务的commit阶段完成，然后继续下一个远程业务。最后根事务提交，完成。
	
- **出现异常**
- **超时**
	- try执行时间慢，导致超时回滚
	
		- 设置delayCancelExcepiton，当遇到delayCancelException时将不会立即执行cancel，而是在其后恢复job再处理
		- 恢复周期内(默认2分钟)try过程一定结束了的，这个是需要约定保证的，比如try阶段数据库操作，超时时间得设置小于恢复周期
			- @Tyloo注解中加入delayCancelExceptions = {SocketTimeoutException.class, TimeoutException.class}
		
			- 在第一个拦截器中对Throwable tryingException做处理
	
				- 如果是非超时问题，则进行回滚
				- 如果是超时问题，先不回滚，由后台定时恢复job进行回滚（默认每30s执行一次）
		
	- **commit断电**
- 如果是Root事务更新状态时断电 没有更新状态成功，则恢复job获取事务状态时还是Trying阶段，会执行Cancel操作；
		- 如果时分支事务更新状态时断电，这个时候Root事务已经将状态改成了Confirming，恢复job获取事务状态时会拿到Confirming状态，则会继续将执行Confirm操作

- **幂等性**
- 约束：被动方应用对于消息的业务处理要实现幂等
	- try阶段为insert方法，comfirm阶段为update方法
	
	- update方法由xxxRepository的乐观锁保证幂等性
		- 这个幂等性是保证的tcc数据库的幂等性，防止多次confirm/cancel操作
		- 业务数据库幂等性由业务方控制
	
- **注意事项**
- 全局事务id 分支id
	
	- 传播获取分支事务时获取的是全局事务id+分支事务id
		- 传播新建分支事务时xid为全局事务id+分支事务id
		-     public TylooTransactionXid(byte[] globalTransactionId) {
	    this.globalTransactionId = globalTransactionId;
      this.branchQualifier = uuidToByteArray(UUID.randomUUID());
  }
  
- ThreadLocal
	
	- 调用远程try 和远程cofirm是两个不同线程
	本地事务（根事务带着俩代理对象）一个
远程事务各一个
远程事务根据代理对象发来的分支事务事务上下文获得参与者集合（为1个）

- @Transaction
	
	- 因为贴有@Transactional标签，所以由本地事务管理，因为tradeOrderDto和transferFromAccount是同一个数据库的数据，也就是连接池一样，这里可以保证tradeOrderDto(之前的数据)和transferFromAccount(修改后的数据)一致的保存在数据库中
	
- proceedingjoinpoint
	
	- 获取当前方法和参数
		- 意思是 ： 主业务对象一个pjp 俩代理对象各一个pjp 俩远程业务各一个pjp
	pjp设置当前方法的xid 事务状态 事务类型到事务上下文中

- 并发问题
	
	- 恢复job对每个日志 通过乐观锁来处理并发问题，只有一个节点可以操作成功。
		- 并发其实利用由数据库 update 乐观锁来控制的。同一时刻，只能有一个成功更新状态；
		- update后，即使再去调用 confirm 去提交，幂等性保证不会出现问题，即 confirm 多次也能保证数据的一致性。
		- 并发其实利用由数据库 update 乐观锁来控制的。同一时刻，只能有一个成功更新状态；
		  update后，即使再去调用 confirm 去提交，幂等性保证不会出现问题，即 confirm 多次也能保证数据的一致性。



## 难点

### 主事务到分支事务xid传参错误问题

场景：dubbo嵌套调用，或者非嵌套调用，但是主事务方法入口注解Compensable使用了属性transactionContextEditor = DubboTransactionContextEditor.class

原因：dubbo服务rpc调用内部的事务上下文传递，由于ResourceCoordinatorInterceptor添加参与者时，主事务自己添加为参与者，会更新RpcContext的事务上下文，但是在添加第一个分支事务作为参与者时，由于对是否存在事务上下文的判断（此时事务上下文Xid的branchQualifier为主事务参与者的branchQualifier），导致并未刷新分支事务参与者的branchQualifier进去，造成对应分支事务创建事务记录时的branchQualifier为主事务参与者的branchQualifier，然后在分支事务确认或回滚时又拿着分支事务参与者的Xid查询记录查不出来，无法确认和回滚。建议去掉此处的判断(下面注释掉了去掉的判断)，ResourceCoordinatorInterceptor.enlistParticipant()方法：代码如下：

```java
//        if (FactoryBuilder.factoryOf(compensable.transactionContextEditor()).getInstance().get(pjp.getTarget(), method, pjp.getArgs()) == null) {
            FactoryBuilder.factoryOf(compensable.transactionContextEditor()).getInstance().set(new TransactionContext(xid, TransactionStatus.TRYING.getId()), pjp.getTarget(), ((MethodSignature) pjp.getSignature()).getMethod(), pjp.getArgs());
//        }
```

是由于DubboTransactionContextEditor的get方法 暂时并未用到那些入参，所以这里的判断有了问题。可能后续对于这块会有处理。



### 针对 SocketTimeoutException 

- try 阶段，本地参与者调用远程参与者( 远程服务，例如 Dubbo，Http 服务)，远程参与者 try 阶段的方法逻辑执行时间较长，超过 Socket 等待时长，发生 SocketTimeoutException，如果立刻执行事务回滚，远程参与者 try 的方法未执行完成，可能导致 cancel 的方法实际未执行( try 的方法未执行完成，数据库事务【非 TCC 事务】未提交，cancel 的方法读取数据时发现未变更，导致方法实际未执行，最终 try 的方法执行完后，提交数据库事务【非 TCC 事务】，较为极端 )，最终引起数据不一致。
  - 在事务恢复时，会对这种情况的事务进行取消回滚，如果此时远程参与者的 try 的方法还未结束，还是可能发生数据不一致。 

- 为什么tcc事务切面中对乐观锁与socket超时异常不做回滚处理，只抛异常
  - 不立即回滚，主要考虑是被调用服务方存在一直在正常执行的可能，只是执行的慢，导致了调用方超时，此时如果立即回滚，在被调用方执行cancel操作的同时，被调用方的try方法还在执行，甚至cancel操作执行完了，try方法还没结束，这种情况下业务数据存在不一致的可能。目前解决办法是这类异常不立即回滚，而是由恢复job执行回滚，恢复job会在一段时间后再去调用该被调用方的cancel方法，这个时间可在RecoverConfig中设置，默认120s。

### 针对 OptimisticLockException

- 还是 SocketTimeoutException 的情况，事务恢复间隔时间小于 Socket 超时时间，此时事务恢复调用远程参与者取消回滚事务，远程参与者下次更新事务时，会因为乐观锁更新失败，抛出 OptimisticLockException。如果 CompensableTransactionInterceptor 此时立刻取消回滚，可能会和定时任务的取消回滚冲突，因此统一交给定时任务处理。

  - 使用版本号控制，事务超时了，恢复机制将尝试进行事务恢复，恢复时会更新版本号（乐观锁）以获得事务控制权，原来的事务因为失去了控制权，则在更新事务状态时失败，并终止事务继续进行。

  - 在资源准备过程中 和 资源确认、回滚、或者调用调度任务都会走 CachableTransactionRepository的update方法。

  - 先doUpdate 改状态，乐观锁控制，改成功了 返回1 正常执行流程，改失败了，会抛出OptimisticLockException异常

  - 并发其实利用由数据库 update 乐观锁来控制的。同一时刻，只能有一个成功更新状态；update后，即使再去调用 confirm 去提交，幂等性保证不会出现问题，即 confirm 多次也能保证数据的一致性。
  
  - ```java
    在资源准备过程中 和 资源确认、回滚、或者调用调度任务都会走 CachableTransactionRepository的update方法。下面是方法体。
    int result = 0;
    try {
    result = doUpdate(transaction);
    if (result > 0) {
    putToCache(transaction);
    } else {
    throw new OptimisticLockException();
    }
    } finally {
    if (result <= 0) {
    removeFromCache(transaction);
    }
    }
    return result;
    
    ```
  
  先doUpdate 改状态，就是作者所得乐观锁，改成功了 返回1 正常执行流程，改失败了，会抛出OptimisticLockException异常
  
- 严重BUG:主服务调用从服务，主服务出现异常，从服务为什么不回滚

  -  看看是不是配置有问题，主服务异常，在主服务内从服务如果都没被执行，那就是说从服务没有被加入到整个事务中，也就不会回滚了。

- 在调confirm及cancle方法时，如果判断事务不存在，即从未做过或已经处理完了，则直接跳过

  - 在confirm或是cancel时，框架是先执行服务的confirm或是cancel，执行成功后删除事务日志，这里存在服务的confirm或cancel执行完了，但是删除事务日志失败了，这种情况下恢复job将会重新执行整个confirm或是cancel，这时需要业务代码保证服务的幂等性

- ThreadLocal的使用存在内存泄漏

  在TransactionManager中使用了ThreadLocal，如下：
  private static final ThreadLocal<Deque> CURRENT = new ThreadLocal<Deque>();

  在CompensableTransactionInterceptor拦截器主事务方法执行完毕后，
  transactionManager.cleanAfterCompletion(transaction);
  未执行ThreadLocal的remove方法，会导致内存泄漏。



