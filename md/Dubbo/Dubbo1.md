



# 示例

### **启动时检查**

- 概念

  - Dubbo 缺省会在启动时检查依赖的服务是否可用，不可用时会抛出异常，阻止 Spring 初始化完成，以便上线时，能及早发现问题，默认  `check="true"`。
  - 可以通过 `check="false"` 关闭检查，比如，测试时，有些服务不关心，或者出现了循环依赖，必须有一方先启动。
  - 另外，如果你的 Spring 容器是懒加载的，或者通过 API 编程延迟引用服务，请关闭 check，否则服务临时不可用时，会抛出异常，拿到 null 引用，如果 `check="false"`，总是会返回引用，当服务恢复时，能自动连上。

- 通过 spring 配置文件

  关闭某个服务的启动时检查 (没有提供者时报错)：

  ```xml
  <dubbo:reference interface="com.foo.BarService" check="false" />
  ```

  关闭所有服务的启动时检查 (没有提供者时报错)：

  ```xml
  <dubbo:consumer check="false" />
  ```

  关闭注册中心启动时检查 (注册订阅失败时报错)：

  ```xml
  <dubbo:registry check="false" />
  ```

### **集群容错**

- 在集群调用失败时，Dubbo 提供了多种容错方案，缺省为 failover 重试。

- ![cluster](http://dubbo.apache.org/docs/zh-cn/user/sources/images/cluster.jpg)

- 各节点关系：

  - 这里的 `Invoker` 是 `Provider` 的一个可调用 `Service` 的抽象，`Invoker` 封装了 `Provider` 地址及 `Service` 接口信息
  - `Directory` 代表多个 `Invoker`，可以把它看成 `List<Invoker>` ，但与 `List` 不同的是，它的值可能是动态变化的，比如注册中心推送变更
  - `Cluster` 将 `Directory` 中的多个 `Invoker` 伪装成一个 `Invoker`，对上层透明，伪装过程包含了容错逻辑，调用失败后，重试另一个
  - `Router` 负责从多个 `Invoker` 中按路由规则选出子集，比如读写分离，应用隔离等
  - `LoadBalance` 负责从多个 `Invoker` 中选出具体的一个用于本次调用，选的过程包含了负载均衡算法，调用失败后，需要重选

- 集群容错模式

  可以自行扩展集群容错策略，参见：[集群扩展](http://dubbo.apache.org/zh-cn/docs/dev/impls/cluster.html)

  Failover Cluster

  失败自动切换，当出现失败，重试其它服务器 [[1\]](http://dubbo.apache.org/zh-cn/docs/user/demos/fault-tolerent-strategy.html#fn1)。通常用于读操作，但重试会带来更长延迟。可通过 `retries="2"` 来设置重试次数(不含第一次)。

  重试次数配置如下：

  ```xml
  <dubbo:service retries="2" />
  ```

  或

  ```xml
  <dubbo:reference retries="2" />
  ```

  或

  ```xml
  <dubbo:reference>
      <dubbo:method name="findFoo" retries="2" />
  </dubbo:reference>
  ```

  Failfast Cluster

  快速失败，只发起一次调用，失败立即报错。通常用于非幂等性的写操作，比如新增记录。

  Failsafe Cluster

  失败安全，出现异常时，直接忽略。通常用于写入审计日志等操作。

  Failback Cluster

  失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作。

  Forking Cluster

  并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多服务资源。可通过 `forks="2"` 来设置最大并行数。

  Broadcast Cluster

  广播调用所有提供者，逐个调用，任意一台报错则报错 [[2\]](http://dubbo.apache.org/zh-cn/docs/user/demos/fault-tolerent-strategy.html#fn2)。通常用于通知所有提供者更新缓存或日志等本地资源信息。

- 集群模式配置

  按照以下示例在服务提供方和消费方配置集群模式

  ```xml
  <dubbo:service cluster="failsafe" />
  ```

  或

  ```xml
  <dubbo:reference cluster="failsafe" />
  ```

### **负载均衡**

- 在集群负载均衡时，Dubbo 提供了多种均衡策略，缺省为 `random` 随机调用。

- 负载均衡策略

  Random LoadBalance

  - **随机**，按权重设置随机概率。
  - 在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重。

  RoundRobin LoadBalance

  - **轮询**，按公约后的权重设置轮询比率。
  - 存在慢的提供者累积请求的问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上。

  LeastActive LoadBalance

  - **最少活跃调用数**，相同活跃数的随机，活跃数指调用前后计数差。
  - 使慢的提供者收到更少请求，因为越慢的提供者的调用前后计数差会越大。

  ConsistentHash LoadBalance

  - **一致性 Hash**，相同参数的请求总是发到同一提供者。
  - 当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动。
  - 算法参见：<http://en.wikipedia.org/wiki/Consistent_hashing>
  - 缺省只对第一个参数 Hash，如果要修改，请配置 `<dubbo:parameter key="hash.arguments" value="0,1" />`
  - 缺省用 160 份虚拟节点，如果要修改，请配置 `<dubbo:parameter key="hash.nodes" value="320" />`

- 配置

  服务端服务级别

  ```xml
  <dubbo:service interface="..." loadbalance="roundrobin" />
  ```

  客户端服务级别

  ```xml
  <dubbo:reference interface="..." loadbalance="roundrobin" />
  ```

  服务端方法级别

  ```xml
  <dubbo:service interface="...">
      <dubbo:method name="..." loadbalance="roundrobin"/>
  </dubbo:service>
  ```

  客户端方法级别

  ```xml
  <dubbo:reference interface="...">
      <dubbo:method name="..." loadbalance="roundrobin"/>
  </dubbo:reference>
  ```

### **线程模型**

- 需要通过不同的派发策略和不同的线程池配置的组合来应对不同的场景

  - ![dubbo-protocol](http://dubbo.apache.org/docs/zh-cn/user/sources/images/dubbo-protocol.jpg)
  - 如果事件处理的逻辑能迅速完成，并且不会发起新的 IO 请求，比如只是在内存中记个标识，则直接在 IO 线程上处理更快，因为减少了线程池调度。
  - 但如果事件处理逻辑较慢，或者需要发起新的 IO 请求，比如需要查询数据库，则必须派发到线程池，否则 IO 线程阻塞，将导致不能接收其它请求。
  - 如果用 IO 线程处理事件，又在事件处理过程中发起新的 IO 请求，比如在连接事件中发起登录请求，会报“可能引发死锁”异常，但不会真死锁。

- ```xml
  <dubbo:protocol name="dubbo" dispatcher="all" threadpool="fixed" threads="100" />
  
  Dispatcher
  all 所有消息都派发到线程池，包括请求，响应，连接事件，断开事件，心跳等。
  direct 所有消息都不派发到线程池，全部在 IO 线程上直接执行。
  message 只有请求响应消息派发到线程池，其它连接断开事件，心跳等消息，直接在 IO 线程上执行。
  execution 只有请求消息派发到线程池，不含响应，响应和其它连接断开事件，心跳等消息，直接在 IO 线程上执行。
  connection 在 IO 线程上，将连接断开事件放入队列，有序逐个执行，其它消息派发到线程池。
  
  ThreadPool
  fixed 固定大小线程池，启动时建立线程，不关闭，一直持有。(缺省)
  cached 缓存线程池，空闲一分钟自动删除，需要时重建。
  limited 可伸缩线程池，但池中的线程数只会增长不会收缩。只增长不收缩的目的是为了避免收缩时突然来了大流量引起的性能问题。
  
  ```

### 直连提供者

- 在开发及测试环境下，经常需要绕过注册中心，只测试指定服务提供者，这时候可能需要点对点直连，点对点直连方式，将以服务接口为单位，忽略注册中心的提供者列表，A 接口配置点对点，不影响 B 接口从注册中心获取列表。
- 通过 XML 配置

如果是线上需求需要点对点，可在 `<dubbo:reference>` 中配置 url 指向提供者，将绕过注册中心，多个地址用分号隔开，配置如下 [[1\]](http://dubbo.apache.org/zh-cn/docs/user/demos/explicit-target.html#fn1)：

```xml
<dubbo:reference id="xxxService" interface="com.alibaba.xxx.XxxService" url="dubbo://localhost:20890" />
```



### 只订阅

- 为方便开发测试，经常会在线下共用一个所有服务可用的注册中心，这时，如果一个正在开发中的服务提供者注册，可能会影响消费者不能正常运行。

- 可以让服务提供者开发方，只订阅服务(开发的服务可能依赖其它服务)，而不注册正在开发的服务，通过直连测试正在开发的服务。

  - 禁用注册配置

  ```xml
  <dubbo:registry address="10.20.153.10:9090" register="false" />
  ```

  ​	或者

  ```xml
  <dubbo:registry address="10.20.153.10:9090?register=false" />
  ```



### 只注册

- 如果有两个镜像环境，两个注册中心，有一个服务只在其中一个注册中心有部署，另一个注册中心还没来得及部署，而两个注册中心的其它应用都需要依赖此服务。这个时候，可以让服务提供者方只注册服务到另一注册中心，而不从另一注册中心订阅服务。

禁用订阅配置

```xml
<dubbo:registry id="hzRegistry" address="10.20.153.10:9090" />
<dubbo:registry id="qdRegistry" address="10.20.141.150:9090" subscribe="false" />
```

或者

```xml
<dubbo:registry id="hzRegistry" address="10.20.153.10:9090" />
<dubbo:registry id="qdRegistry" address="10.20.141.150:9090?subscribe=false" />
```

### **多协议**

- Dubbo 允许配置多协议，在不同服务上支持不同协议或者同一服务上同时支持多种协议。

- 不同服务不同协议

  不同服务在性能上适用不同协议进行传输，比如大数据用短连接协议，小数据大并发用长连接协议

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
      xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd"> 
      <dubbo:application name="world"  />
      <dubbo:registry id="registry" address="10.20.141.150:9090" username="admin" password="hello1234" />
      <!-- 多协议配置 -->
      <dubbo:protocol name="dubbo" port="20880" />
      <dubbo:protocol name="rmi" port="1099" />
      <!-- 使用dubbo协议暴露服务 -->
      <dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService" protocol="dubbo" />
      <!-- 使用rmi协议暴露服务 -->
      <dubbo:service interface="com.alibaba.hello.api.DemoService" version="1.0.0" ref="demoService" protocol="rmi" /> 
  </beans>
  ```

- 多协议暴露服务

  需要与 http 客户端互操作

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
      xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
      <dubbo:application name="world"  />
      <dubbo:registry id="registry" address="10.20.141.150:9090" username="admin" password="hello1234" />
      <!-- 多协议配置 -->
      <dubbo:protocol name="dubbo" port="20880" />
      <dubbo:protocol name="hessian" port="8080" />
      <!-- 使用多个协议暴露服务 -->
      <dubbo:service id="helloService" interface="com.alibaba.hello.api.HelloService" version="1.0.0" protocol="dubbo,hessian" />
  </beans>
  ```

### **多注册中心**

- Dubbo 支持同一服务向多注册中心同时注册，或者不同服务分别注册到不同的注册中心上去，甚至可以同时引用注册在不同注册中心上的同名服务。另外，注册中心是支持自定义扩展的

- 多注册中心注册

  比如：中文站有些服务来不及在青岛部署，只在杭州部署，而青岛的其它应用需要引用此服务，就可以将服务同时注册到两个注册中心。

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
      xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
      <dubbo:application name="world"  />
      <!-- 多注册中心配置 -->
      <dubbo:registry id="hangzhouRegistry" address="10.20.141.150:9090" />
      <dubbo:registry id="qingdaoRegistry" address="10.20.141.151:9010" default="false" />
      <!-- 向多个注册中心注册 -->
      <dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService" registry="hangzhouRegistry,qingdaoRegistry" />
  </beans>
  ```

- 不同服务使用不同注册中心

  比如：CRM 有些服务是专门为国际站设计的，有些服务是专门为中文站设计的。

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
      xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
      <dubbo:application name="world"  />
      <!-- 多注册中心配置 -->
      <dubbo:registry id="chinaRegistry" address="10.20.141.150:9090" />
      <dubbo:registry id="intlRegistry" address="10.20.154.177:9010" default="false" />
      <!-- 向中文站注册中心注册 -->
      <dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService" registry="chinaRegistry" />
      <!-- 向国际站注册中心注册 -->
      <dubbo:service interface="com.alibaba.hello.api.DemoService" version="1.0.0" ref="demoService" registry="intlRegistry" />
  </beans>
  ```

- 多注册中心引用

  比如：CRM 需同时调用中文站和国际站的 PC2 服务，PC2 在中文站和国际站均有部署，接口及版本号都一样，但连的数据库不一样。

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
      xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
      <dubbo:application name="world"  />
      <!-- 多注册中心配置 -->
      <dubbo:registry id="chinaRegistry" address="10.20.141.150:9090" />
      <dubbo:registry id="intlRegistry" address="10.20.154.177:9010" default="false" />
      <!-- 引用中文站服务 -->
      <dubbo:reference id="chinaHelloService" interface="com.alibaba.hello.api.HelloService" version="1.0.0" registry="chinaRegistry" />
      <!-- 引用国际站站服务 -->
      <dubbo:reference id="intlHelloService" interface="com.alibaba.hello.api.HelloService" version="1.0.0" registry="intlRegistry" />
  </beans>
  ```

  如果只是测试环境临时需要连接两个不同注册中心，使用竖号分隔多个不同注册中心地址：

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
      xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
      <dubbo:application name="world"  />
      <!-- 多注册中心配置，竖号分隔表示同时连接多个不同注册中心，同一注册中心的多个集群地址用逗号分隔 -->
      <dubbo:registry address="10.20.141.150:9090|10.20.154.177:9010" />
      <!-- 引用服务 -->
      <dubbo:reference id="helloService" interface="com.alibaba.hello.api.HelloService" version="1.0.0" />
  </beans>
  ```

### **服务分组**

- 当一个接口有多种实现时，可以用 group 区分。

  ```xml
  服务
  <dubbo:service group="feedback" interface="com.xxx.IndexService" />
  <dubbo:service group="member" interface="com.xxx.IndexService" />
  
  引用
  <dubbo:reference id="feedbackIndexService" group="feedback" interface="com.xxx.IndexService" />
  <dubbo:reference id="memberIndexService" group="member" interface="com.xxx.
  ```

### **多版本**

- 当一个接口实现，出现不兼容升级时，可以用版本号过渡，版本号不同的服务相互间不引用。

  可以按照以下的步骤进行版本迁移：

  1. 在低压力时间段，先升级一半提供者为新版本
  2. 再将所有消费者升级为新版本
  3. 然后将剩下的一半提供者升级为新版本

  老版本服务提供者配置：

  ```xml
  <dubbo:service interface="com.foo.BarService" version="1.0.0" />
  ```

  新版本服务提供者配置：

  ```xml
  <dubbo:service interface="com.foo.BarService" version="2.0.0" />
  ```

  老版本服务消费者配置：

  ```xml
  <dubbo:reference id="barService" interface="com.foo.BarService" version="1.0.0" />
  ```

  新版本服务消费者配置：

  ```xml
  <dubbo:reference id="barService" interface="com.foo.BarService" version="2.0.0" />
  ```

  如果不需要区分版本，可以按照以下的方式配置 [[1\]](http://dubbo.apache.org/zh-cn/docs/user/demos/multi-versions.html#fn1)：

  ```xml
  <dubbo:reference id="barService" interface="com.foo.BarService" version="*" />
  ```

### 隐式参数

- 可以通过 `RpcContext` 上的 `setAttachment` 和 `getAttachment` 在服务消费方和提供方之间进行参数的隐式传递。 [[1\]](http://dubbo.apache.org/zh-cn/docs/user/demos/attachment.html#fn1)

  ![/user-guide/images/context.png](http://dubbo.apache.org/docs/zh-cn/user/sources/images/context.png)

- 在服务消费方端设置隐式参数

  `setAttachment` 设置的 KV 对，在完成下面一次远程调用会被清空，即多次远程调用要多次设置。

  ```xml
  RpcContext.getContext().setAttachment("index", "1"); // 隐式传参，后面的远程调用都会隐式将这些参数发送到服务器端，类似cookie，用于框架集成，不建议常规业务使用
  xxxService.xxx(); // 远程调用
  // ...
  ```

- 在服务提供方端获取隐式参数

  ```java
  public class XxxServiceImpl implements XxxService {
   
      public void xxx() {
          // 获取客户端隐式传入的参数，用于框架集成，不建议常规业务使用
          String index = RpcContext.getContext().getAttachment("index"); 
      }
  }
  ```

### **Consumer异步调用**

- 从v2.7.0开始，Dubbo的所有异步编程接口开始以[CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)为基础

  基于 NIO 的非阻塞实现并行调用，客户端不需要启动多线程即可完成并行调用多个远程服务，相对多线程开销较小。

  ![/user-guide/images/future.jpg](http://dubbo.apache.org/docs/zh-cn/user/sources/images/future.jpg)

- **使用CompletableFuture签名的接口**

  需要服务提供者事先定义CompletableFuture签名的服务，具体参见[服务端异步执行](http://dubbo.apache.org/zh-cn/docs/user/demos/async-execute-on-provider.html)接口定义：

  ```java
  public interface AsyncService {
      CompletableFuture<String> sayHello(String name);
  }
  ```

  注意接口的返回类型是`CompletableFuture<String>`。

  XML引用服务：

  ```xml
  <dubbo:reference id="asyncService" timeout="10000" interface="com.alibaba.dubbo.samples.async.api.AsyncService"/>
  ```

  调用远程服务：

  ```java
  // 调用直接返回CompletableFuture
  CompletableFuture<String> future = asyncService.sayHello("async call request");
  // 增加回调
  future.whenComplete((v, t) -> {
      if (t != null) {
          t.printStackTrace();
      } else {
          System.out.println("Response: " + v);
      }
  });
  // 早于结果输出
  System.out.println("Executed before response return.");
  ```

- **使用RpcContext**

  在 consumer.xml 中配置：

  ```xml
  <dubbo:reference id="asyncService" interface="org.apache.dubbo.samples.governance.api.AsyncService">
        <dubbo:method name="sayHello" async="true" />
  </dubbo:reference>
  ```

  调用代码:

  ```java
  // 此调用会立即返回null
  asyncService.sayHello("world");
  // 拿到调用的Future引用，当结果返回后，会被通知和设置到此Future
  CompletableFuture<String> helloFuture = RpcContext.getContext().getCompletableFuture();
  // 为Future添加回调
  helloFuture.whenComplete((retValue, exception) -> {
      if (exception == null) {
          System.out.println(retValue);
      } else {
          exception.printStackTrace();
      }
  });
  ```

  或者，你也可以这样做异步调用:

  ```java
  CompletableFuture<String> future = RpcContext.getContext().asyncCall(
      () -> {
          asyncService.sayHello("oneway call request1");
      }
  );
  
  future.get();
  ```

- **重载服务接口**

  如果你只有这样的同步服务定义，而又不喜欢RpcContext的异步使用方式。

  ```java
  public interface GreetingsService {
      String sayHi(String name);
  }
  ```

  那还有一种方式，就是利用Java 8提供的default接口实现，重载一个带有带有CompletableFuture签名的方法。

  有两种方式来实现：

  1. 提供方或消费方自己修改接口签名

  ```java
  public interface GreetingsService {
      String sayHi(String name);
      
      // AsyncSignal is totally optional, you can use any parameter type as long as java allows your to do that.
      default CompletableFuture<String> sayHi(String name, AsyncSignal signal) {
          return CompletableFuture.completedFuture(sayHi(name));
      }
  }
  ```

  你也可以设置是否等待消息发出： [[1\]](http://dubbo.apache.org/zh-cn/docs/user/demos/async-call.html#fn1)

  - `sent="true"` 等待消息发出，消息发送失败将抛出异常。
  - `sent="false"` 不等待消息发出，将消息放入 IO 队列，即刻返回。

  ```xml
  <dubbo:method name="findFoo" async="true" sent="true" />
  ```

  如果你只是想异步，完全忽略返回值，可以配置 `return="false"`，以减少 Future 对象的创建和管理成本：

  ```xml
  <dubbo:method name="findFoo" async="true" return="false" />
  ```

  ------

  1. 异步总是不等待返回



### Provider异步执行

Provider端异步执行将阻塞的业务从Dubbo内部线程池切换到业务自定义线程，避免Dubbo线程池的过度占用，有助于避免不同服务间的互相影响。异步执行无益于节省资源或提升RPC响应性能，因为如果业务执行需要阻塞，则始终还是要有线程来负责执行。

> 注意：Provider端异步执行和Consumer端异步调用是相互独立的，你可以任意正交组合两端配置
>
> - Consumer同步 - Provider同步
> - Consumer异步 - Provider同步
> - Consumer同步 - Provider异步
> - Consumer异步 - Provider异步

- **定义CompletableFuture签名的接口**
  - 服务接口定义：

```java
public interface AsyncService {
    CompletableFuture<String> sayHello(String name);
}
```

-  服务实现：	

```java
public class AsyncServiceImpl implements AsyncService {
    @Override
    public CompletableFuture<String> sayHello(String name) {
        RpcContext savedContext = RpcContext.getContext();
        // 建议为supplyAsync提供自定义线程池，避免使用JDK公用线程池
        return CompletableFuture.supplyAsync(() -> {
            System.out.println(savedContext.getAttachment("consumer-key1"));
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "async response from provider.";
        });
    }
}
```

通过`return CompletableFuture.supplyAsync()`，业务执行已从Dubbo线程切换到业务线程，避免了对Dubbo线程池的阻塞。

- **使用AsyncContext**

Dubbo提供了一个类似Serverlet 3.0的异步接口`AsyncContext`，在没有CompletableFuture签名接口的情况下，也可以实现Provider端的异步执行。

- 服务接口定义：

```java
public interface AsyncService {
    String sayHello(String name);
}
```

- 服务暴露，和普通服务完全一致：

```xml
<bean id="asyncService" class="org.apache.dubbo.samples.governance.impl.AsyncServiceImpl"/>
<dubbo:service interface="org.apache.dubbo.samples.governance.api.AsyncService" ref="asyncService"/>
```

服务实现：

```java
public class AsyncServiceImpl implements AsyncService {
    public String sayHello(String name) {
        final AsyncContext asyncContext = RpcContext.startAsync();
        new Thread(() -> {
            // 如果要使用上下文，则必须要放在第一句执行
            asyncContext.signalContextSwitch();
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            // 写回响应
            asyncContext.write("Hello " + name + ", response from provider.");
        }).start();
        return null;
    }
}
```



### **服务降级**

- 可以通过服务降级功能 [[1\]](http://dubbo.apache.org/zh-cn/docs/user/demos/service-downgrade.html#fn1) 临时屏蔽某个出错的非关键服务，并定义降级后的返回策略。

  向注册中心写入动态配置覆盖规则：

  ```java
  RegistryFactory registryFactory = ExtensionLoader.getExtensionLoader(RegistryFactory.class).getAdaptiveExtension();
  Registry registry = registryFactory.getRegistry(URL.valueOf("zookeeper://10.20.153.10:2181"));
  registry.register(URL.valueOf("override://0.0.0.0/com.foo.BarService?category=configurators&dynamic=false&application=foo&mock=force:return+null"));
  ```

  其中：

  - `mock=force:return+null` 表示消费方对该服务的方法调用都直接返回 null 值，不发起远程调用。用来屏蔽不重要服务不可用时对调用方的影响。
  - 还可以改为 `mock=fail:return+null` 表示消费方对该服务的方法调用在失败后，再返回 null 值，不抛异常。用来容忍不重要服务不稳定时对调用方的影响。

### 优雅停机

Dubbo 是通过 JDK 的 ShutdownHook 来完成优雅停机的，所以如果用户使用 `kill -9 PID` 等强制关闭指令，是不会执行优雅停机的，只有通过 `kill PID` 时，才会执行。

- 原理

  - 服务提供方
    - 停止时，先标记为不接收新请求，新请求过来时直接报错，让客户端重试其它机器。
    - 然后，检测线程池中的线程是否正在运行，如果有，等待所有线程执行完成，除非超时，则强制关闭。

  - 服务消费方
    - 停止时，不再发起新的调用请求，所有新的调用在客户端即报错。
    - 然后，检测有没有请求的响应还没有返回，等待响应返回，除非超时，则强制关闭。

  - 设置方式

    设置优雅停机超时时间，缺省超时时间是 10 秒，如果超时则强制关闭。

    ```properties
    # dubbo.properties
    dubbo.service.shutdown.wait=15000
    ```

    如果 ShutdownHook 不能生效，可以自行调用，**使用tomcat等容器部署的场景，建议通过扩展ContextListener等自行调用以下代码实现优雅停机**：

    ```java
    DubboShutdownHook.destroyAll();
    ```

    

### 延迟暴露

- 如果你的服务需要预热时间，比如初始化缓存，等待相关资源就位等，可以使用 delay 进行延迟暴露。我们在 Dubbo 2.6.5 版本中对服务延迟暴露逻辑进行了细微的调整，将需要延迟暴露（delay > 0）服务的倒计时动作推迟到了 Spring 初始化完成后进行。你在使用 Dubbo 的过程中，并不会感知到此变化，因此请放心使用。

- Dubbo-2.6.5 之前版本

  延迟到 Spring 初始化完成后，再暴露服务[[1\]](http://dubbo.apache.org/zh-cn/docs/user/demos/delay-publish.html#fn1)

  ```xml
  <dubbo:service delay="-1" />
  ```

  延迟 5 秒暴露服务

  ```xml
  <dubbo:service delay="5000" />
  ```

- Dubbo-2.6.5 及以后版本

  所有服务都将在 Spring 初始化完成后进行暴露，如果你不需要延迟暴露服务，无需配置 delay。

  延迟 5 秒暴露服务

  ```xml
  <dubbo:service delay="5000" />
  ```

- Spring 2.x 初始化死锁问题

  - 触发条件

  在 Spring 解析到 `<dubbo:service />` 时，就已经向外暴露了服务，而 Spring 还在接着初始化其它 Bean。如果这时有请求进来，并且服务的实现类里有调用 `applicationContext.getBean()` 的用法。

  1. 请求线程的 applicationContext.getBean() 调用，先同步 singletonObjects 判断 Bean 是否存在，不存在就同步 beanDefinitionMap 进行初始化，并再次同步 singletonObjects 写入 Bean 实例缓存。

     ![deadlock](http://dubbo.apache.org/docs/zh-cn/user/sources/images/lock-get-bean.jpg)

  2. 而 Spring 初始化线程，因不需要判断 Bean 的存在，直接同步 beanDefinitionMap 进行初始化，并同步 singletonObjects 写入 Bean 实例缓存。

     ![/user-guide/images/lock-init-context.jpg](http://dubbo.apache.org/docs/zh-cn/user/sources/images/lock-init-context.jpg)

     这样就导致 getBean 线程，先锁 singletonObjects，再锁 beanDefinitionMap，再次锁 singletonObjects。
     而 Spring 初始化线程，先锁 beanDefinitionMap，再锁 singletonObjects。反向锁导致线程死锁，不能提供服务，启动不了。

  - 规避办法

  1. 强烈建议不要在服务的实现类中有 applicationContext.getBean() 的调用，全部采用 IoC 注入的方式使用 Spring的Bean。
  2. 如果实在要调 getBean()，可以将 Dubbo 的配置放在 Spring 的最后加载。
  3. 如果不想依赖配置顺序，可以使用 `<dubbo:provider delay=”-1” />`，使 Dubbo 在 Spring 容器初始化完后，再暴露服务。
  4. 如果大量使用 getBean()，相当于已经把 Spring 退化为工厂模式在用，可以将 Dubbo 的服务隔离单独的 Spring 容器。

  

### 路由规则

路由规则在发起一次RPC调用前起到过滤目标服务器地址的作用，过滤后的地址列表，将作为消费端最终发起RPC调用的备选地址。

- 条件路由。支持以服务或Consumer应用为粒度配置路由规则。
- 标签路由。以Provider应用为粒度配置路由规则。

后续我们计划在2.6.x版本的基础上继续增强脚本路由功能，老版本脚本路由规则配置方式请参见开篇链接。

#### **条件路由**

您可以随时在服务治理控制台[Dubbo-Admin](https://github.com/apache/dubbo-admin)写入路由规则

简介

- 应用粒度

  ```yaml
  # app1的消费者只能消费所有端口为20880的服务实例
  # app2的消费者只能消费所有端口为20881的服务实例
  ---
  scope: application
  force: true
  runtime: true
  enabled: true
  key: governance-conditionrouter-consumer
  conditions:
    - application=app1 => address=*:20880
    - application=app2 => address=*:20881
  ...
  ```

- 服务粒度

  ```yaml
  # DemoService的sayHello方法只能消费所有端口为20880的服务实例
  # DemoService的sayHi方法只能消费所有端口为20881的服务实例
  ---
  scope: service
  force: true
  runtime: true
  enabled: true
  key: org.apache.dubbo.samples.governance.api.DemoService
  conditions:
    - method=sayHello => address=*:20880
    - method=sayHi => address=*:20881
  ...
  ```

**规则详解**

各字段含义

- `scope`

  表示路由规则的作用粒度，scope的取值会决定key的取值。必填

  - service 服务粒度
  - application 应用粒度

- `Key`明确规则体作用在哪个服务或应用。必填

  - scope=service时，key取值为[{group}:]{service}[:{version}]的组合
  - scope=application时，key取值为application名称

- `enabled=true` 当前路由规则是否生效，可不填，缺省生效。

- `force=false` 当路由结果为空时，是否强制执行，如果不强制执行，路由结果为空的路由规则将自动失效，可不填，缺省为 `false`。

- `runtime=false` 是否在每次调用时执行路由规则，否则只在提供者地址列表变更时预先执行并缓存结果，调用时直接从缓存中获取路由结果。如果用了参数路由，必须设为 `true`，需要注意设置会影响调用的性能，可不填，缺省为 `false`。

- `priority=1` 路由规则的优先级，用于排序，优先级越大越靠前执行，可不填，缺省为 `0`。

- `conditions` 定义具体的路由规则内容。**必填**。

Conditions规则体

```
`conditions`部分是规则的主体，由1到任意多条规则组成，下面我们就每个规则的配置语法做详细说明：
```

1. **格式**

- `=>` 之前的为消费者匹配条件，所有参数和消费者的 URL 进行对比，当消费者满足匹配条件时，对该消费者执行后面的过滤规则。
- `=>` 之后为提供者地址列表的过滤条件，所有参数和提供者的 URL 进行对比，消费者最终只拿到过滤后的地址列表。
- 如果匹配条件为空，表示对所有消费方应用，如：`=> host != 10.20.153.11`
- 如果过滤条件为空，表示禁止访问，如：`host = 10.20.153.10 =>`

1. **表达式**

参数支持：

- 服务调用信息，如：method, argument 等，暂不支持参数路由
- URL 本身的字段，如：protocol, host, port 等
- 以及 URL 上的所有参数，如：application, organization 等

条件支持：

- 等号 `=` 表示"匹配"，如：`host = 10.20.153.10`
- 不等号 `!=` 表示"不匹配"，如：`host != 10.20.153.10`

值支持：

- 以逗号 `,` 分隔多个值，如：`host != 10.20.153.10,10.20.153.11`
- 以星号 `*` 结尾，表示通配，如：`host != 10.20.*`
- 以美元符 `$` 开头，表示引用消费者参数，如：`host = $host`

1. **Condition示例**

- 排除预发布机：

```
=> host != 172.22.3.91
```

- 白名单 [[1\]](http://dubbo.apache.org/zh-cn/docs/user/demos/routing-rule.html#fn1)：

```
host != 10.20.153.10,10.20.153.11 =>
```

- 黑名单：

```
host = 10.20.153.10,10.20.153.11 =>
```

- 服务寄宿在应用上，只暴露一部分的机器，防止整个集群挂掉：

```
=> host = 172.22.3.1*,172.22.3.2*
```

- 为重要应用提供额外的机器：

```
application != kylin => host != 172.22.3.95,172.22.3.96
```

- 读写分离：

```
method = find*,list*,get*,is* => host = 172.22.3.94,172.22.3.95,172.22.3.96
method != find*,list*,get*,is* => host = 172.22.3.97,172.22.3.98
```

- 前后台分离：

```
application = bops => host = 172.22.3.91,172.22.3.92,172.22.3.93
application != bops => host = 172.22.3.94,172.22.3.95,172.22.3.96
```

- 隔离不同机房网段：

```
host != 172.22.3.* => host != 172.22.3.*
```

- 提供者与消费者部署在同集群内，本机只访问本机的服务：

```
=> host = $host
```

#### **标签路由规则**

简介

标签路由通过将某一个或多个服务的提供者划分到同一个分组，约束流量只在指定分组中流转，从而实现流量隔离的目的，可以作为蓝绿发布、灰度发布等场景的能力基础。

Provider

标签主要是指对Provider端应用实例的分组，目前有两种方式可以完成实例分组，分别是`动态规则打标`和`静态规则打标`，其中动态规则相较于静态规则优先级更高，而当两种规则同时存在且出现冲突时，将以动态规则为准。

- 动态规则打标，可随时在[服务治理控制台](http://dubbo.apache.org/zh-cn/docs/user/demos/routing-rule.html)下发标签归组规则

  ```yaml
  # governance-tagrouter-provider应用增加了两个标签分组tag1和tag2
  # tag1包含一个实例 127.0.0.1:20880
  # tag2包含一个实例 127.0.0.1:20881
  ---
    force: false
    runtime: true
    enabled: true
    key: governance-tagrouter-provider
    tags:
      - name: tag1
        addresses: ["127.0.0.1:20880"]
      - name: tag2
        addresses: ["127.0.0.1:20881"]
   ...
  ```

- 静态打标

  ```xml
  <dubbo:provider tag="tag1"/>
  ```

  or

  ```xml
  <dubbo:service tag="tag1"/>
  ```

  or

  ```properties
  java -jar xxx-provider.jar -Ddubbo.provider.tag={the tag you want, may come from OS ENV}
  ```

Consumer

```java
RpcContext.getContext().setAttachment(Constants.REQUEST_TAG_KEY,"tag1");
```

请求标签的作用域为每一次 invocation，使用 attachment 来传递请求标签，注意保存在 attachment 中的值将会在一次完整的远程调用中持续传递，得益于这样的特性，我们只需要在起始调用时，通过一行代码的设置，达到标签的持续传递。

> 目前仅仅支持 hardcoding 的方式设置 requestTag。注意到 RpcContext 是线程绑定的，优雅的使用 TagRouter 特性，建议通过 servlet 过滤器(在 web 环境下)，或者定制的 SPI 过滤器设置 requestTag。



**规则详解**

格式

- `Key`明确规则体作用到哪个应用。**必填**。

- `enabled=true` 当前路由规则是否生效，可不填，缺省生效。

- `force=false` 当路由结果为空时，是否强制执行，如果不强制执行，路由结果为空的路由规则将自动失效，可不填，缺省为 `false`。

- `runtime=false` 是否在每次调用时执行路由规则，否则只在提供者地址列表变更时预先执行并缓存结果，调用时直接从缓存中获取路由结果。如果用了参数路由，必须设为 `true`，需要注意设置会影响调用的性能，可不填，缺省为 `false`。

- `priority=1` 路由规则的优先级，用于排序，优先级越大越靠前执行，可不填，缺省为 `0`。

- ```
  tags
  ```

   

  定义具体的标签分组内容，可定义任意n（n>=1）个标签并为每个标签指定实例列表。

  必填

  。

  - name， 标签名称

- addresses， 当前标签包含的实例列表

#### 降级约定

1. `request.tag=tag1` 时优先选择 标记了`tag=tag1` 的 provider。若集群中不存在与请求标记对应的服务，默认将降级请求 tag为空的provider；如果要改变这种默认行为，即找不到匹配tag1的provider返回异常，需设置`request.tag.force=true`。
2. `request.tag`未设置时，只会匹配tag为空的provider。即使集群中存在可用的服务，若 tag 不匹配也就无法调用，这与约定1不同，携带标签的请求可以降级访问到无标签的服务，但不携带标签/携带其他种类标签的请求永远无法访问到其他标签的服务。

------

1. 注意：一个服务只能有一条白名单规则，否则两条规则交叉，就都被筛选掉了 [↩︎](http://dubbo.apache.org/zh-cn/docs/user/demos/routing-rule.html#fnref1)







### **并发控制**

- 配置样例

- 样例 1

限制 `com.foo.BarService` 的每个方法，服务器端并发执行（或占用线程池线程数）不能超过 10 个：

```xml
<dubbo:service interface="com.foo.BarService" executes="10" />
```

- 样例 2

限制 `com.foo.BarService` 的 `sayHello` 方法，服务器端并发执行（或占用线程池线程数）不能超过 10 个：

```xml
<dubbo:service interface="com.foo.BarService">
    <dubbo:method name="sayHello" executes="10" />
</dubbo:service>
```

- 样例 3

限制 `com.foo.BarService` 的每个方法，每客户端并发执行（或占用连接的请求数）不能超过 10 个：

```xml
<dubbo:service interface="com.foo.BarService" actives="10" />
```

或

```xml
<dubbo:reference interface="com.foo.BarService" actives="10" />
```

- 样例 4

限制 `com.foo.BarService` 的 `sayHello` 方法，每客户端并发执行（或占用连接的请求数）不能超过 10 个：

```xml
<dubbo:service interface="com.foo.BarService">
    <dubbo:method name="sayHello" actives="10" />
</dubbo:service>
```

或

```xml
<dubbo:reference interface="com.foo.BarService">
    <dubbo:method name="sayHello" actives="10" />
</dubbo:service>
```

如果 `<dubbo:service>` 和 `<dubbo:reference>` 都配了actives，`<dubbo:reference>` 优先，参见：[配置的覆盖策略](http://dubbo.apache.org/zh-cn/docs/user/configuration/xml.html)。

- Load Balance 均衡

  配置服务的客户端的 `loadbalance` 属性为 `leastactive`，此 Loadbalance 会调用并发数最小的 Provider（Consumer端并发数）。

  ```xml
  <dubbo:reference interface="com.foo.BarService" loadbalance="leastactive" />
  ```

  或

  ```xml
  <dubbo:service interface="com.foo.BarService" loadbalance="leastactive" />
  ```

  

### **连接控制**

- 服务端连接控制

  限制服务器端接受的连接不能超过 10 个 [[1\]](http://dubbo.apache.org/zh-cn/docs/user/demos/config-connections.html#fn1)：

  ```xml
  <dubbo:provider protocol="dubbo" accepts="10" />
  ```

  或

  ```xml
  <dubbo:protocol name="dubbo" accepts="10" />
  ```

- 客户端连接控制

  限制客户端服务使用连接不能超过 10 个 [[2\]](http://dubbo.apache.org/zh-cn/docs/user/demos/config-connections.html#fn2)：

  ```xml
  <dubbo:reference interface="com.foo.BarService" connections="10" />
  ```

  或

  ```xml
  <dubbo:service interface="com.foo.BarService" connections="10" />
  ```

  如果 `<dubbo:service>` 和 `<dubbo:reference>` 都配了 connections，`<dubbo:reference>` 优先，参见：[配置的覆盖策略](http://dubbo.apache.org/zh-cn/docs/user/configuration/xml.html)

------

1. 因为连接在 Server上，所以配置在 Provider 上 [↩︎](http://dubbo.apache.org/zh-cn/docs/user/demos/config-connections.html#fnref1)
2. 如果是长连接，比如 Dubbo 协议，connections 表示该服务对每个提供者建立的长连接数 [↩︎](http://dubbo.apache.org/zh-cn/docs/user/demos/config-connections.html#fnref2)





