功能：

- 支持 `Dubbo`RPC框架进行分布式事务

- 支持事务异常回滚，超时异常恢复，防止事务悬挂

- 事务日志存储支持 `mysql`,, `mongodb`, `redis`, `zookeeper` 等方式

- 高性能，支持微服务集群部署

  



发展：

如何改进性能


 可以使用RedisTransactionRepository来存储事务日子, 另外可以设置confirm或cancel是异步执行，不会影响事务的一致性（可获取1.2.3.6版本，@Compensable属性增加了asyncConfirm,asyncCancel属性


serializer可以设置为KryoPoolSerializer，以优化序列化性能

采用disruptor框架进行事务日志的异步读

提供后台管理可视化,以及metrics相关性能监控

提供零侵入的`spring namespace`, `springboot` 快速集成方式, 简单易用





# **目录**

一、写在前面

二、业务场景介绍

三、进一步思考

四、落地实现TCC分布式事务

  (1)TCC实现阶段一：Try

  (2)TCC实现阶段二：Confirm

  (3)TCC实现阶段三：Cancel

五、总结与思考

 

## **一、写在前面**

 之前网上看到很多写分布式事务的文章，不过大多都是将分布式事务各种技术方案简单介绍一下。很多朋友看了不少文章，还是不知道分布式事务到底怎么回事，在项目里到底如何使用。

 所以咱们这篇文章，就用大白话+手工绘图，并结合一个电商系统的案例实践，来给大家讲清楚到底什么是TCC分布式事务。

 

## **二、业务场景介绍**

 咱们先来看看业务场景，假设你现在有一个电商系统，里面有一个支付订单的场景。

 那对一个订单支付之后，我们需要做下面的步骤：

- 更改订单的状态为“已支付”
- 扣减商品库存
- 给会员增加积分
- 创建销售出库单通知仓库发货

 这是一系列比较真实的步骤，无论大家有没有做过电商系统，应该都能理解。

 ![img](https://img2018.cnblogs.com/blog/918692/201811/918692-20181123154323784-1697034217.png)

 

##  **三、进一步思考**

 好，业务场景有了，现在我们要更进一步，实现一个TCC分布式事务的效果。

 什么意思呢？也就是说，订单服务-修改订单状态，库存服务-扣减库存，积分服务-增加积分，仓储服务-创建销售出库单。

 上述这几个步骤，要么一起成功，要么一起失败，**必须是一个整体性的事务**。

 举个例子，现在订单的状态都修改为“已支付”了，结果库存服务扣减库存失败。那个商品的库存原来是100件，现在卖掉了2件，本来应该是98件了。

 结果呢？由于库存服务操作数据库异常，导致库存数量还是100。这不是在坑人么，当然不能允许这种情况发生了！

 

但是如果你不用TCC分布式事务方案的话，就用个Spring Cloud开发这么一个微服务系统，很有可能会干出这种事儿来。

 我们来看看下面的这个图，直观的表达了上述的过程。

![img](https://img2018.cnblogs.com/blog/918692/201811/918692-20181123154419908-1904290158.png)

 

 

所以说，我们有必要使用TCC分布式事务机制来保证各个服务形成一个整体性的事务。

 上面那几个步骤，要么全部成功，如果任何一个服务的操作失败了，就全部一起回滚，撤销已经完成的操作。

 比如说库存服务要是扣减库存失败了，那么订单服务就得撤销那个修改订单状态的操作，然后得停止执行增加积分和通知出库两个操作。

 说了那么多，老规矩，给大家上一张图，大伙儿顺着图来直观的感受一下。

![img](https://img2018.cnblogs.com/blog/918692/201811/918692-20181123154449362-184004902.png)

 

##  **四、落地实现TCC分布式事务**

 那么现在到底要如何来实现一个TCC分布式事务，使得各个服务，要么一起成功？要么一起失败呢？

 大家稍安勿躁，我们这就来一步一步的分析一下。咱们就以一个Spring Cloud开发系统作为背景来解释。

 

###  **1、TCC实现阶段一：Try**

 首先，订单服务那儿，他的代码大致来说应该是这样子的：

 ![img](https://img2018.cnblogs.com/blog/918692/201811/918692-20181123154530434-790705908.png)

 

如果你之前看过Spring Cloud架构原理那篇文章，同时对Spring Cloud有一定的了解的话，应该是可以理解上面那段代码的。

 其实就是订单服务完成本地数据库操作之后，通过Spring Cloud的Feign来调用其他的各个服务罢了。

 但是光是凭借这段代码，是不足以实现TCC分布式事务的啊？！兄弟们，别着急，我们对这个订单服务修改点儿代码好不好。

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

 总结上述过程，如果你要实现一个TCC分布式事务，首先你的业务的主流程以及各个接口提供的业务含义，不是说直接完成那个业务操作，而是完成一个Try的操作。

 这个操作，一般都是锁定某个资源，设置一个预备类的状态，冻结部分数据，等等，大概都是这类操作。

 咱们来一起看看下面这张图，结合上面的文字，再来捋一捋这整个过程。

 

![img](https://img2018.cnblogs.com/blog/918692/201811/918692-20181123154627825-1811880105.png)

 

 

### **2、TCC实现阶段二：Confirm**

 

然后就分成两种情况了，第一种情况是比较理想的，那就是各个服务执行自己的那个Try操作，都执行成功了，bingo！

 这个时候，就需要依靠**TCC分布式事务框架**来推动后续的执行了。

 这里简单提一句，如果你要玩儿TCC分布式事务，必须引入一款TCC分布式事务框架，比如国内开源的**ByteTCC、himly、tcc-transaction。**

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

 好了，上面各种服务的Confirm的逻辑都实现好了，一旦订单服务里面的TCC分布式事务框架感知到各个服务的Try阶段都成功了以后，就会执行各个服务的Confirm逻辑。

 订单服务内的TCC事务框架会负责跟其他各个服务内的TCC事务框架进行通信，依次调用各个服务的Confirm逻辑。然后，正式完成各个服务的所有业务逻辑的执行。

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

 

 

 

 

## **五、总结与思考**

 好了，兄弟们，聊到这儿，基本上大家应该都知道TCC分布式事务具体是怎么回事了！

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

 

写到这里，本文差不多该结束了。等一等，你有没有想到一个问题？

 

## 问题

如果有一些意外的情况发生了，比如说订单服务突然挂了，然后再次重启，TCC分布式事务框架是**如何保证之前没执行完的分布式事务继续执行的呢？**

 所以，TCC事务框架都是要记录一些分布式事务的活动日志的，可以在磁盘上的日志文件里记录，也可以在数据库里记录。保存下来分布式事务运行的各个阶段和状态。

 问题还没完，万一某个服务的Cancel或者Confirm逻辑执行一直失败怎么办呢？

 那也很简单，TCC事务框架会通过活动日志记录各个服务的状态。

 举个例子，比如发现某个服务的Cancel或者Confirm一直没成功，会不停的重试调用他的Cancel或者Confirm逻辑，务必要他成功！

 当然了，如果你的代码没有写什么bug，有充足的测试，而且Try阶段都基本尝试了一下，那么其实一般Confirm、Cancel都是可以成功的！

 最后，再给大家来一张图，来看看给我们的业务，加上分布式事务之后的整个执行流程：

![img](https://img2018.cnblogs.com/blog/918692/201811/918692-20181123154939317-290045834.png)

 

 

 不少大公司里，其实都是自己研发TCC分布式事务框架的，专门在公司内部使用，比如我们就是这样。

 不过如果自己公司没有研发TCC分布式事务框架的话，那一般就会选用开源的框架。

 这里笔者给大家推荐几个比较不错的框架，都是咱们国内自己开源出去的：**ByteTCC，tcc-transaction，himly**。

 大家有兴趣的可以去他们的github地址，学习一下如何使用，以及如何跟Spring Cloud、Dubbo等服务框架整合使用。

 只要把那些框架整合到你的系统里，很容易就可以实现上面那种奇妙的TCC分布式事务的效果了。

 

## 这种方案的应用场景：

这种方案说实话几乎很少用人使用，我们用的也比较少，但是也有使用的场景。因为这个事务回滚实际上是严重依赖于你自己写代码来回滚和补偿了，会造成补偿代码巨大，非常之恶心。

 比如说我们，一般来说跟钱相关的，跟钱打交道的，支付、交易相关的场景，我们会用TCC，严格严格保证分布式事务要么全部成功，要么全部自动回滚，严格保证资金的正确性，在资金上出现问题

 比较适合的场景：这个就是除非你是真的一致性要求太高，是你系统中核心之核心的场景，比如常见的就是资金类的场景，那你可以用TCC方案了，自己编写大量的业务逻辑，自己判断一个事务中的各个环节是否ok，不ok就执行补偿/回滚代码。

 而且最好是你的各个业务执行的时间都比较短。

 但是说实话，一般尽量别这么搞，自己手写回滚逻辑，或者是补偿逻辑，实在太恶心了，那个业务代码很难维护。





**本文主要基于 TCC-Transaction 1.2.3.3 正式版**

- [1. 概述](http://www.iocoder.cn/TCC-Transaction/http-sample/)
- \2. 实体结构
  - [2.1 商城服务](http://www.iocoder.cn/TCC-Transaction/http-sample/)
  - [2.2 资金服务](http://www.iocoder.cn/TCC-Transaction/http-sample/)
  - [2.3 红包服务](http://www.iocoder.cn/TCC-Transaction/http-sample/)
- [3. 服务调用](http://www.iocoder.cn/TCC-Transaction/http-sample/)
- \4. 下单支付流程
  - [4.1 Try 阶段](http://www.iocoder.cn/TCC-Transaction/http-sample/)
  - 4.2 Confirm / Cancel 阶段
    - [4.2.1 Confirm](http://www.iocoder.cn/TCC-Transaction/http-sample/)
    - [4.2.2 Cancel](http://www.iocoder.cn/TCC-Transaction/http-sample/)
- [666. 彩蛋](http://www.iocoder.cn/TCC-Transaction/http-sample/)

------

# 详细场景

# 1. 概述

本文分享 **TCC 项目实战**。以官方 Maven项目 `tcc-transaction-http-sample` 为例子( `tcc-transaction-dubbo-sample` 类似 )。



OK，首先我们简单了解下这个项目。

![img](http://static.iocoder.cn/images/TCC-Transaction/2018_03_15/01.png)

- 首页 => 商品列表 => 确认支付页 => 支付结果页
- 使用账户余额 + 红包余额**联合**支付购买商品，并账户之间**转账**。

项目拆分三个子 Maven 项目：

- `tcc-transaction-http-order` ：商城服务，提供商品和商品订单逻辑。
- `tcc-transaction-http-capital` ：资金服务，提供账户余额逻辑。
- `tcc-transaction-http-redpacket` ：红包服务，提供红包余额逻辑。

![img](http://static.iocoder.cn/images/TCC-Transaction/2018_03_15/03.png)

> 你行好事会因为得到赞赏而愉悦
> 同理，开源项目贡献者会因为 Star 而更加有动力
> 为 TCC-Transaction 点赞！[传送门](https://github.com/changmingxie/tcc-transaction)

# 2. 实体结构

## 2.1 商城服务

![img](http://static.iocoder.cn/images/TCC-Transaction/2018_03_15/02.png)

- Shop，商店表。实体代码如下：

  

  ```
  public class Shop {
  
      /**
       * 商店编号
       */
      private long id;
      /**
       * 所有者用户编号
       */
      private long ownerUserId;
  }
  ```

  

- Product，商品表。实体代码如下：

  

  ```
  public class Product implements Serializable {
  
      /**
       * 商品编号
       */
      private long productId;
      /**
       * 商店编号
       */
      private long shopId;
      /**
       * 商品名
       */
      private String productName;
      /**
       * 单价
       */
      private BigDecimal price;
  }
  ```

  

- Order，订单表。实现代码如下：

  

  ```
  public class Order implements Serializable {
  
      private static final long serialVersionUID = -5908730245224893590L;
  
      /**
       * 订单编号
       */
      private long id;
      /**
       * 支付( 下单 )用户编号
       */
      private long payerUserId;
      /**
       * 收款( 商店拥有者 )用户编号
       */
      private long payeeUserId;
      /**
       * 红包支付金额
       */
      private BigDecimal redPacketPayAmount;
      /**
       * 账户余额支付金额
       */
      private BigDecimal capitalPayAmount;
      /**
       * 订单状态
       * - DRAFT ：草稿
       * - PAYING ：支付中
       * - CONFIRMED ：支付成功
       * - PAY_FAILED ：支付失败
       */
      private String status = "DRAFT";
      /**
       * 商户订单号，使用 UUID 生成
       */
      private String merchantOrderNo;
  
      /**
       * 订单明细数组
       * 非存储字段
       */
      private List<OrderLine> orderLines = new ArrayList<OrderLine>();
  }
  ```

  

- OrderLine，订单明细。实体代码如下：

  

  ```
  public class OrderLine implements Serializable {
  
      private static final long serialVersionUID = 2300754647209250837L;
  
      /**
       * 订单编号
       */
      private long id;
      /**
       * 商品编号
       */
      private long productId;
      /**
       * 数量
       */
      private int quantity;
      /**
       * 单价
       */
      private BigDecimal unitPrice;
  }
  ```

  

**业务逻辑**：

下单时，插入订单状态为 `"DRAFT"` 的订单( Order )记录，并插入购买的商品订单明细( OrderLine )记录。支付时，更新订单状态为 `"PAYING"`。

- 订单支付成功，更新订单状态为 `"CONFIRMED"`。
- 订单支付失败，更新订单状体为 `"PAY_FAILED"`。

## 2.2 资金服务

关系较为简单，有两个实体：

- CapitalAccount，资金账户余额。实体代码如下：

  

  ```
  public class CapitalAccount {
  
      /**
       * 账户编号
       */
      private long id;
      /**
       * 用户编号
       */
      private long userId;
      /**
       * 余额
       */
      private BigDecimal balanceAmount;
  }
  ```

  

- TradeOrder，交易订单表。实体代码如下：

  

  ```
  public class TradeOrder {
  
      /**
       * 交易订单编号
       */
      private long id;
      /**
       * 转出用户编号
       */
      private long selfUserId;
      /**
       * 转入用户编号
       */
      private long oppositeUserId;
      /**
       * 商户订单号
       */
      private String merchantOrderNo;
      /**
       * 金额
       */
      private BigDecimal amount;
      /**
       * 交易订单状态
       * - DRAFT ：草稿
       * - CONFIRM ：交易成功
       * - CANCEL ：交易取消
       */
      private String status = "DRAFT";
  }
  ```

  

**业务逻辑**：

订单支付支付中，插入交易订单状态为 `"DRAFT"` 的订单( TradeOrder )记录，并更新**减少**下单用户的资金账户余额。

- 订单支付成功，更新交易订单状态为 `"CONFIRM"`，并更新**增加**商店拥有用户的资金账户余额。
- 订单支付失败，更新交易订单状态为 `"CANCEL"`，并更新**增加( 恢复 )**下单用户的资金账户余额。

## 2.3 红包服务

关系较为简单，**和资金服务 99.99% 相同**，有两个实体：

- RedPacketAccount，红包账户余额。实体代码如下：

  

  ```
  public class RedPacketAccount {
  
      /**
       * 账户编号
       */
      private long id;
      /**
       * 用户编号
       */
      private long userId;
      /**
       * 余额
       */
      private BigDecimal balanceAmount;
  }
  ```

  

- TradeOrder，交易订单表。实体代码如下：

  

  ```
  public class TradeOrder {
  
      /**
       * 交易订单编号
       */
      private long id;
      /**
       * 转出用户编号
       */
      private long selfUserId;
      /**
       * 转入用户编号
       */
      private long oppositeUserId;
      /**
       * 商户订单号
       */
      private String merchantOrderNo;
      /**
       * 金额
       */
      private BigDecimal amount;
      /**
       * 交易订单状态
       * - DRAFT ：草稿
       * - CONFIRM ：交易成功
       * - CANCEL ：交易取消
       */
      private String status = "DRAFT";
  }
  ```

  

**业务逻辑**：

订单支付支付中，插入交易订单状态为 `"DRAFT"` 的订单( TradeOrder )记录，并更新**减少**下单用户的红包账户余额。

- 订单支付成功，更新交易订单状态为 `"CONFIRM"`，并更新**增加**商店拥有用户的红包账户余额。
- 订单支付失败，更新交易订单状态为 `"CANCEL"`，并更新**增加( 恢复 )**下单用户的红包账户余额。

# 3. 服务调用

服务之间，通过 **HTTP** 进行调用。

**红包服务和资金服务为商城服务提供调用( 以资金服务为例子 )**：

- XML 配置如下 ：

  

  ```
  // appcontext-service-provider.xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:util="http://www.springframework.org/schema/util"
         xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd">
  
      <bean name="capitalAccountRepository"
            class="org.mengyun.tcctransaction.sample.http.capital.domain.repository.CapitalAccountRepository"/>
  
      <bean name="tradeOrderRepository"
            class="org.mengyun.tcctransaction.sample.http.capital.domain.repository.TradeOrderRepository"/>
  
      <bean name="capitalTradeOrderService"
            class="org.mengyun.tcctransaction.sample.http.capital.service.CapitalTradeOrderServiceImpl"/>
  
      <bean name="capitalAccountService"
            class="org.mengyun.tcctransaction.sample.http.capital.service.CapitalAccountServiceImpl"/>
  
      <bean name="capitalTradeOrderServiceExporter"
            class="org.springframework.remoting.httpinvoker.SimpleHttpInvokerServiceExporter">
          <property name="service" ref="capitalTradeOrderService"/>
          <property name="serviceInterface"
                    value="org.mengyun.tcctransaction.sample.http.capital.api.CapitalTradeOrderService"/>
      </bean>
  
      <bean name="capitalAccountServiceExporter"
            class="org.springframework.remoting.httpinvoker.SimpleHttpInvokerServiceExporter">
          <property name="service" ref="capitalAccountService"/>
          <property name="serviceInterface"
                    value="org.mengyun.tcctransaction.sample.http.capital.api.CapitalAccountService"/>
      </bean>
  
  
      <bean id="httpServer"
            class="org.springframework.remoting.support.SimpleHttpServerFactoryBean">
          <property name="contexts">
              <util:map>
                  <entry key="/remoting/CapitalTradeOrderService" value-ref="capitalTradeOrderServiceExporter"/>
                  <entry key="/remoting/CapitalAccountService" value-ref="capitalAccountServiceExporter"/>
              </util:map>
          </property>
          <property name="port" value="8081"/>
      </bean>
  
  </beans>
  ```

  

- Java 代码实现如下 ：

  

  ```
  public class CapitalAccountServiceImpl implements CapitalAccountService {
      
      @Autowired
      CapitalAccountRepository capitalAccountRepository;
  
      @Override
      public BigDecimal getCapitalAccountByUserId(long userId) {
          return capitalAccountRepository.findByUserId(userId).getBalanceAmount();
      }
  
  }
  
  public class CapitalAccountServiceImpl implements CapitalAccountService {
  
      @Autowired
      CapitalAccountRepository capitalAccountRepository;
  
      @Override
      public BigDecimal getCapitalAccountByUserId(long userId) {
          return capitalAccountRepository.findByUserId(userId).getBalanceAmount();
      }
  
  }
  ```

  

------

**商城服务调用**

- XML 配置如下：

  

  ```
  // appcontext-service-consumer.xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
  
      <bean id="httpInvokerRequestExecutor"
            class="org.springframework.remoting.httpinvoker.CommonsHttpInvokerRequestExecutor">
          <property name="httpClient">
              <bean class="org.apache.commons.httpclient.HttpClient">
                  <property name="httpConnectionManager">
                      <ref bean="multiThreadHttpConnectionManager"/>
                  </property>
              </bean>
          </property>
      </bean>
  
      <bean id="multiThreadHttpConnectionManager"
            class="org.apache.commons.httpclient.MultiThreadedHttpConnectionManager">
          <property name="params">
              <bean class="org.apache.commons.httpclient.params.HttpConnectionManagerParams">
                  <property name="connectionTimeout" value="200000"/>
                  <property name="maxTotalConnections" value="600"/>
                  <property name="defaultMaxConnectionsPerHost" value="512"/>
                  <property name="soTimeout" value="5000"/>
              </bean>
          </property>
      </bean>
  
      <bean id="captialTradeOrderService" class="org.springframework.remoting.httpinvoker.HttpInvokerProxyFactoryBean">
          <property name="serviceUrl" value="http://localhost:8081/remoting/CapitalTradeOrderService"/>
          <property name="serviceInterface"
                    value="org.mengyun.tcctransaction.sample.http.capital.api.CapitalTradeOrderService"/>
          <property name="httpInvokerRequestExecutor" ref="httpInvokerRequestExecutor"/>
      </bean>
  
      <bean id="capitalAccountService" class="org.springframework.remoting.httpinvoker.HttpInvokerProxyFactoryBean">
          <property name="serviceUrl" value="http://localhost:8081/remoting/CapitalAccountService"/>
          <property name="serviceInterface"
                    value="org.mengyun.tcctransaction.sample.http.capital.api.CapitalAccountService"/>
          <property name="httpInvokerRequestExecutor" ref="httpInvokerRequestExecutor"/>
      </bean>
  
      <bean id="redPacketAccountService" class="org.springframework.remoting.httpinvoker.HttpInvokerProxyFactoryBean">
          <property name="serviceUrl" value="http://localhost:8082/remoting/RedPacketAccountService"/>
          <property name="serviceInterface"
                    value="org.mengyun.tcctransaction.sample.http.redpacket.api.RedPacketAccountService"/>
          <property name="httpInvokerRequestExecutor" ref="httpInvokerRequestExecutor"/>
      </bean>
  
      <bean id="redPacketTradeOrderService" class="org.springframework.remoting.httpinvoker.HttpInvokerProxyFactoryBean">
          <property name="serviceUrl" value="http://localhost:8082/remoting/RedPacketTradeOrderService"/>
          <property name="serviceInterface"
                    value="org.mengyun.tcctransaction.sample.http.redpacket.api.RedPacketTradeOrderService"/>
          <property name="httpInvokerRequestExecutor" ref="httpInvokerRequestExecutor"/>
      </bean>
  
  </beans>
  ```

  

- Java 接口接口如下：

  

  ```
  public interface CapitalAccountService {
      BigDecimal getCapitalAccountByUserId(long userId);
  }
  
  public interface CapitalTradeOrderService {
      String record(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto);
  }
  
  public interface RedPacketAccountService {
      BigDecimal getRedPacketAccountByUserId(long userId);
  }
  
  public interface RedPacketTradeOrderService {
      String record(TransactionContext transactionContext, RedPacketTradeOrderDto tradeOrderDto);
  }
  ```

  

# 4. 下单支付流程

**ps**：数据访问的方法，请自己拉取代码，使用 IDE 查看。谢谢。🙂

下单支付流程，整体流程如下图( [打开大图](http://www.iocoder.cn/images/TCC-Transaction/2018_03_15/04.png) )：

![img](http://static.iocoder.cn/images/TCC-Transaction/2018_03_15/04.png)

点击**【支付】**按钮，下单支付流程。实现代码如下：



```
@Controller
@RequestMapping("")
public class OrderController {
    
        @RequestMapping(value = "/placeorder", method = RequestMethod.POST)
    public ModelAndView placeOrder(@RequestParam String redPacketPayAmount,
                                   @RequestParam long shopId,
                                   @RequestParam long payerUserId,
                                   @RequestParam long productId) {
        PlaceOrderRequest request = buildRequest(redPacketPayAmount, shopId, payerUserId, productId);
        // 下单并支付订单
        String merchantOrderNo = placeOrderService.placeOrder(request.getPayerUserId(), request.getShopId(),
                request.getProductQuantities(), request.getRedPacketPayAmount());
        // 返回
        ModelAndView mv = new ModelAndView("pay_success");
        // 查询订单状态
        String status = orderService.getOrderStatusByMerchantOrderNo(merchantOrderNo);
        // 支付结果提示
        String payResultTip = null;
        if ("CONFIRMED".equals(status)) {
            payResultTip = "支付成功";
        } else if ("PAY_FAILED".equals(status)) {
            payResultTip = "支付失败";
        }
        mv.addObject("payResult", payResultTip);
        // 商品信息
        mv.addObject("product", productRepository.findById(productId));
        // 资金账户金额 和 红包账户金额
        mv.addObject("capitalAmount", accountService.getCapitalAccountByUserId(payerUserId));
        mv.addObject("redPacketAmount", accountService.getRedPacketAccountByUserId(payerUserId));
        return mv;
    }

}
```



- 调用 `PlaceOrderService#placeOrder(...)` 方法，下单并支付订单。
- 调用 [`OrderService#getOrderStatusByMerchantOrderNo(...)`](https://github.com/YunaiV/tcc-transaction/blob/ff443d9e6d39bd798ed042034bad123b83675922/tcc-transaction-tutorial-sample/tcc-transaction-http-sample/tcc-transaction-http-order/src/main/java/org/mengyun/tcctransaction/sample/http/order/domain/service/OrderServiceImpl.java) 方法，查询订单状态。

------

调用 `PlaceOrderService#placeOrder(...)` 方法，下单并支付订单。实现代码如下：



```
@Service
public class PlaceOrderServiceImpl {

    public String placeOrder(long payerUserId, long shopId, List<Pair<Long, Integer>> productQuantities, BigDecimal redPacketPayAmount) {
        // 获取商店
        Shop shop = shopRepository.findById(shopId);
        // 创建订单
        Order order = orderService.createOrder(payerUserId, shop.getOwnerUserId(), productQuantities);
        // 发起支付
        Boolean result = false;
        try {
            paymentService.makePayment(order, redPacketPayAmount, order.getTotalAmount().subtract(redPacketPayAmount));
        } catch (ConfirmingException confirmingException) {
            // exception throws with the tcc transaction status is CONFIRMING,
            // when tcc transaction is confirming status,
            // the tcc transaction recovery will try to confirm the whole transaction to ensure eventually consistent.
            result = true;
        } catch (CancellingException cancellingException) {
            // exception throws with the tcc transaction status is CANCELLING,
            // when tcc transaction is under CANCELLING status,
            // the tcc transaction recovery will try to cancel the whole transaction to ensure eventually consistent.
        } catch (Throwable e) {
            // other exceptions throws at TRYING stage.
            // you can retry or cancel the operation.
            e.printStackTrace();
        }
        return order.getMerchantOrderNo();
    }

}
```



- 调用 `ShopRepository#findById(...)` 方法，查询商店。

- 调用 `OrderService#createOrder(...)` 方法，创建订单状态为 `"DRAFT"` 的**商城**订单。实际业务不会这么做，此处仅仅是例子，简化流程。实现代码如下：

  

  ```
  @Service
  public class OrderServiceImpl {
  
      @Transactional
      public Order createOrder(long payerUserId, long payeeUserId, List<Pair<Long, Integer>> productQuantities) {
          Order order = orderFactory.buildOrder(payerUserId, payeeUserId, productQuantities);
          orderRepository.createOrder(order);
          return order;
      }
  
  }
  ```

  

- 调用 `PaymentService#makePayment(...)` 方法，发起支付，**TCC 流程**。
- **生产代码对于异常需要进一步处理**。
- **生产代码对于异常需要进一步处理**。
- **生产代码对于异常需要进一步处理**。

## 4.1 Try 阶段

**商城服务**

调用 `PaymentService#makePayment(...)` 方法，发起 Try 流程，实现代码如下：



```
@Compensable(confirmMethod = "confirmMakePayment", cancelMethod = "cancelMakePayment")
@Transactional
public void makePayment(Order order, BigDecimal redPacketPayAmount, BigDecimal capitalPayAmount) {
   System.out.println("order try make payment called.time seq:" + DateFormatUtils.format(Calendar.getInstance(), "yyyy-MM-dd HH:mm:ss"));
   // 更新订单状态为支付中
   order.pay(redPacketPayAmount, capitalPayAmount);
   orderRepository.updateOrder(order);
   // 资金账户余额支付订单
   String result = tradeOrderServiceProxy.record(null, buildCapitalTradeOrderDto(order));
   // 红包账户余额支付订单
   String result2 = tradeOrderServiceProxy.record(null, buildRedPacketTradeOrderDto(order));
}
```



- 设置方法注解 @Compensable

  - 事务传播级别 Propagation.REQUIRED ( **默认值** )
  - 设置 `confirmMethod` / `cancelMethod` 方法名
  - 事务上下文编辑类 DefaultTransactionContextEditor ( **默认值** )

- 设置方法注解 @Transactional，保证方法操作原子性。

- 调用 `OrderRepository#updateOrder(...)` 方法，更新订单状态为**支付中**。实现代码如下：

  

  ```
  // Order.java
  public void pay(BigDecimal redPacketPayAmount, BigDecimal capitalPayAmount) {
     this.redPacketPayAmount = redPacketPayAmount;
     this.capitalPayAmount = capitalPayAmount;
     this.status = "PAYING";
  }
  ```

  

- 调用 `TradeOrderServiceProxy#record(...)` 方法，**资金**账户余额支付订单。实现代码如下：

  

  ```
  // TradeOrderServiceProxy.java
  @Compensable(propagation = Propagation.SUPPORTS, confirmMethod = "record", cancelMethod = "record", transactionContextEditor = Compensable.DefaultTransactionContextEditor.class)
  public String record(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto) {
     return capitalTradeOrderService.record(transactionContext, tradeOrderDto);
  }
  
  // CapitalTradeOrderService.java
  public interface CapitalTradeOrderService {
      String record(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto);
  }
  ```

  

  - 设置方法注解 @Compensable
    - `propagation=Propagation.SUPPORTS` ：支持当前事务，如果当前没有事务，就以非事务方式执行。**为什么不使用 REQUIRED** ？如果使用 REQUIRED 事务传播级别，事务恢复重试时，会发起新的事务。
    - `confirmMethod`、`cancelMethod` 使用和 try 方法**相同方法名**：**本地发起**远程服务 TCC confirm / cancel 阶段，调用相同方法进行事务的提交或回滚。远程服务的 CompensableTransactionInterceptor 会根据事务的状态是 CONFIRMING / CANCELLING 来调用对应方法。
  - 调用 `CapitalTradeOrderService#record(...)` 方法，远程调用，发起**资金**账户余额支付订单。
    - 本地方法调用时，参数 `transactionContext` 传递 `null` 即可，TransactionContextEditor 会设置。在[《TCC-Transaction 源码分析 —— TCC 实现》「6.3 资源协调者拦截器」](http://www.iocoder.cn/TCC-Transaction/tcc-core/?self)有详细解析。
    - 远程方法调用时，参数 `transactionContext` 需要传递。Dubbo 远程方法调用实际也进行了传递，传递方式较为特殊，通过隐式船舱，在[《TCC-Transaction 源码分析 —— Dubbo 支持》「3. Dubbo 事务上下文编辑器」](http://www.iocoder.cn/TCC-Transaction/dubbo-support/?self)有详细解析。

- 调用 `TradeOrderServiceProxy#record(...)` 方法，**红包**账户余额支付订单。和**资金**账户余额支付订单 99.99% 类似，不重复“复制粘贴”。

------

**资金服务**

调用 `CapitalTradeOrderServiceImpl#record(...)` 方法，**红包**账户余额支付订单。实现代码如下：



```
@Override
@Compensable(confirmMethod = "confirmRecord", cancelMethod = "cancelRecord", transactionContextEditor = Compensable.DefaultTransactionContextEditor.class)
@Transactional
public String record(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto) {
   // 调试用
   try {
       Thread.sleep(1000l);
//            Thread.sleep(10000000L);
   } catch (InterruptedException e) {
       throw new RuntimeException(e);
   }
   System.out.println("capital try record called. time seq:" + DateFormatUtils.format(Calendar.getInstance(), "yyyy-MM-dd HH:mm:ss"));
   // 生成交易订单
   TradeOrder tradeOrder = new TradeOrder(
           tradeOrderDto.getSelfUserId(),
           tradeOrderDto.getOppositeUserId(),
           tradeOrderDto.getMerchantOrderNo(),
           tradeOrderDto.getAmount()
   );
   tradeOrderRepository.insert(tradeOrder);
   // 更新减少下单用户的资金账户余额
   CapitalAccount transferFromAccount = capitalAccountRepository.findByUserId(tradeOrderDto.getSelfUserId());
   transferFromAccount.transferFrom(tradeOrderDto.getAmount());
   capitalAccountRepository.save(transferFromAccount);
   return "success";
}
```



- 设置方法注解 @Compensable
  - 事务传播级别 Propagation.REQUIRED ( **默认值** )
  - 设置 `confirmMethod` / `cancelMethod` 方法名
  - 事务上下文编辑类 DefaultTransactionContextEditor ( **默认值** )
- 设置方法注解 @Transactional，保证方法操作原子性。
- 调用 [`TradeOrderRepository#insert(...)`](https://github.com/YunaiV/tcc-transaction/blob/ff443d9e6d39bd798ed042034bad123b83675922/tcc-transaction-tutorial-sample/tcc-transaction-http-sample/tcc-transaction-http-capital/src/main/java/org/mengyun/tcctransaction/sample/http/capital/domain/repository/TradeOrderRepository.java) 方法，生成订单状态为 `"DRAFT"` 的交易订单。
- 调用 [`CapitalAccountRepository#save(...)`](https://github.com/YunaiV/tcc-transaction/blob/ff443d9e6d39bd798ed042034bad123b83675922/tcc-transaction-tutorial-sample/tcc-transaction-http-sample/tcc-transaction-http-capital/src/main/java/org/mengyun/tcctransaction/sample/http/capital/domain/repository/CapitalAccountRepository.java) 方法，更新减少下单用户的资金账户余额。**Try 阶段锁定资源时，一定要先扣。TCC 是最终事务一致性，如果先添加，可能被使用**。

## 4.2 Confirm / Cancel 阶段

当 Try 操作**全部**成功时，发起 Confirm 操作。
当 Try 操作存在**任务**失败时，发起 Cancel 操作。

### 4.2.1 Confirm

**商城服务**

调用 `PaymentServiceImpl#confirmMakePayment(...)` 方法，更新订单状态为支付**成功**。实现代码如下：



```
public void confirmMakePayment(Order order, BigDecimal redPacketPayAmount, BigDecimal capitalPayAmount) {
   // 调试用
   try {
       Thread.sleep(1000l);
   } catch (InterruptedException e) {
       throw new RuntimeException(e);
   }
   System.out.println("order confirm make payment called. time seq:" + DateFormatUtils.format(Calendar.getInstance(), "yyyy-MM-dd HH:mm:ss"));
   // 更新订单状态为支付成功
   order.confirm();
   orderRepository.updateOrder(order);
}
```



- **生产代码该方法需要加下 @Transactional 注解，保证原子性**。

- 调用 `OrderRepository#updateOrder(...)` 方法，更新订单状态为支付成功。实现代码如下：

  

  ```
  // Order.java
  public void confirm() {
     this.status = "CONFIRMED";
  }
  ```

  

------

**资金服务**

调用 `CapitalTradeOrderServiceImpl#confirmRecord(...)` 方法，更新交易订单状态为交易**成功**。



```
@Transactional
public void confirmRecord(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto) {
   // 调试用
   try {
       Thread.sleep(1000l);
   } catch (InterruptedException e) {
       throw new RuntimeException(e);
   }
   System.out.println("capital confirm record called. time seq:" + DateFormatUtils.format(Calendar.getInstance(), "yyyy-MM-dd HH:mm:ss"));
   // 查询交易记录
   TradeOrder tradeOrder = tradeOrderRepository.findByMerchantOrderNo(tradeOrderDto.getMerchantOrderNo());
   // 判断交易记录状态。因为 `#record()` 方法，可能事务回滚，记录不存在 / 状态不对
   if (null != tradeOrder && "DRAFT".equals(tradeOrder.getStatus())) {
       // 更新订单状态为交易成功
       tradeOrder.confirm();
       tradeOrderRepository.update(tradeOrder);
       // 更新增加商店拥有者用户的资金账户余额
       CapitalAccount transferToAccount = capitalAccountRepository.findByUserId(tradeOrderDto.getOppositeUserId());
       transferToAccount.transferTo(tradeOrderDto.getAmount());
       capitalAccountRepository.save(transferToAccount);
   }
}
```



- 设置方法注解 @Transactional，保证方法操作原子性。

- **判断交易记录状态**。因为 `#record()` 方法，可能事务回滚，记录不存在 / 状态不对。

- 调用 [`TradeOrderRepository#update(...)`](https://github.com/YunaiV/tcc-transaction/blob/ff443d9e6d39bd798ed042034bad123b83675922/tcc-transaction-tutorial-sample/tcc-transaction-http-sample/tcc-transaction-http-capital/src/main/java/org/mengyun/tcctransaction/sample/http/capital/domain/repository/TradeOrderRepository.java) 方法，更新交易订单状态为交易**成功**。

- 调用 [`CapitalAccountRepository#save(...)`](https://github.com/YunaiV/tcc-transaction/blob/ff443d9e6d39bd798ed042034bad123b83675922/tcc-transaction-tutorial-sample/tcc-transaction-http-sample/tcc-transaction-http-capital/src/main/java/org/mengyun/tcctransaction/sample/http/capital/domain/repository/CapitalAccountRepository.java) 方法，更新增加商店拥有者用户的资金账户余额。实现代码如下：

  

  ```
  // CapitalAccount.java
  public void transferTo(BigDecimal amount) {
     this.balanceAmount = this.balanceAmount.add(amount);
  }
  ```

  

------

**红包服务**

和**资源服务** 99.99% 相同，不重复“复制粘贴”。

### 4.2.2 Cancel

**商城服务**

调用 `PaymentServiceImpl#cancelMakePayment(...)` 方法，更新订单状态为支付**失败**。实现代码如下：



```
public void cancelMakePayment(Order order, BigDecimal redPacketPayAmount, BigDecimal capitalPayAmount) {
   // 调试用
   try {
       Thread.sleep(1000l);
   } catch (InterruptedException e) {
       throw new RuntimeException(e);
   }
   System.out.println("order cancel make payment called.time seq:" + DateFormatUtils.format(Calendar.getInstance(), "yyyy-MM-dd HH:mm:ss"));
   // 更新订单状态为支付失败
   order.cancelPayment();
   orderRepository.updateOrder(order);
}
```



- **生产代码该方法需要加下 @Transactional 注解，保证原子性**。

- 调用 `OrderRepository#updateOrder(...)` 方法，更新订单状态为支付失败。实现代码如下：

  

  ```
  // Order.java
  public void cancelPayment() {
      this.status = "PAY_FAILED";
  }
  ```

  

------

**资金服务**

调用 `CapitalTradeOrderServiceImpl#cancelRecord(...)` 方法，更新交易订单状态为交易**失败**。



```
@Transactional
public void cancelRecord(TransactionContext transactionContext, CapitalTradeOrderDto tradeOrderDto) {
   // 调试用
   try {
       Thread.sleep(1000l);
   } catch (InterruptedException e) {
       throw new RuntimeException(e);
   }
   System.out.println("capital cancel record called. time seq:" + DateFormatUtils.format(Calendar.getInstance(), "yyyy-MM-dd HH:mm:ss"));
   // 查询交易记录
   TradeOrder tradeOrder = tradeOrderRepository.findByMerchantOrderNo(tradeOrderDto.getMerchantOrderNo());
   // 判断交易记录状态。因为 `#record()` 方法，可能事务回滚，记录不存在 / 状态不对
   if (null != tradeOrder && "DRAFT".equals(tradeOrder.getStatus())) {
       // / 更新订单状态为交易失败
       tradeOrder.cancel();
       tradeOrderRepository.update(tradeOrder);
       // 更新增加( 恢复 )下单用户的资金账户余额
       CapitalAccount capitalAccount = capitalAccountRepository.findByUserId(tradeOrderDto.getSelfUserId());
       capitalAccount.cancelTransfer(tradeOrderDto.getAmount());
       capitalAccountRepository.save(capitalAccount);
   }
}
```



- 设置方法注解 @Transactional，保证方法操作原子性。

- **判断交易记录状态**。因为 `#record()` 方法，可能事务回滚，记录不存在 / 状态不对。

- 调用 [`TradeOrderRepository#update(...)`](https://github.com/YunaiV/tcc-transaction/blob/ff443d9e6d39bd798ed042034bad123b83675922/tcc-transaction-tutorial-sample/tcc-transaction-http-sample/tcc-transaction-http-capital/src/main/java/org/mengyun/tcctransaction/sample/http/capital/domain/repository/TradeOrderRepository.java) 方法，更新交易订单状态为交易**失败**。

- 调用 [`CapitalAccountRepository#save(...)`](https://github.com/YunaiV/tcc-transaction/blob/ff443d9e6d39bd798ed042034bad123b83675922/tcc-transaction-tutorial-sample/tcc-transaction-http-sample/tcc-transaction-http-capital/src/main/java/org/mengyun/tcctransaction/sample/http/capital/domain/repository/CapitalAccountRepository.java) 方法，更新增加( 恢复 )下单用户的资金账户余额。实现代码如下：

  

  ```
  // CapitalAccount.java
  public void cancelTransfer(BigDecimal amount) {
      transferTo(amount);
  }
  ```

  

------

**红包服务**

和**资源服务** 99.99% 相同，不重复“复制粘贴”。