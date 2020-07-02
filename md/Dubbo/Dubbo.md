# 背景

- 当垂直应用越来越多，应用之间交互不可避免，将核心业务抽取出来，作为独立的服务，逐渐形成稳定的服务中心，使前端应用能更快速的响应多变的市场需求。此时，用于提高业务复用及整合的分布式服务框架(RPC)是关键。
- 在大规模服务化之前，应用可能只是通过 RMI 或 Hessian 等工具，简单的暴露和引用远程服务，通过配置服务的URL地址进行调用，通过 F5 等硬件进行负载均衡
  - 当服务越来越多时，服务 URL 配置管理变得非常困难，F5 硬件负载均衡器的单点压力也越来越大
  - 当进一步发展，服务间依赖关系变得错踪复杂，甚至分不清哪个应用要在哪个应用之前启动
  - 接着，服务的调用量越来越大，服务的容量问题就暴露出来，这个服务需要多少机器支撑？什么时候该加机器？





# 成熟度

- 功能成熟度
  - 延迟暴露
    - 延迟暴露服务，用于等待应用加载warmup数据，或等待spring加载完成
  - 本地伪装
    - 用于服务降级
  - 隐式传参
    - 附加参数

- 策略成熟度
  - Zookeeper注册中心
  - Dubbo协议
    - 采用NIO复用单一长连接，并使用线程池并发处理请求，减少握手和加大并发效率，性能较好（推荐使用）
    - 在大文件传输时，单一连接会成为瓶颈
  - Hessian Serialization
  - Javassist ProxyFactory
    - 通过字节码生成代替反射，性能比较好（推荐使用）
    - 依赖于javassist.jar包，占用JVM的Perm内存，Perm可能要设大一些：java -XX:PermSize=128m
  - Failover Cluster
    - 失败自动切换，当出现失败，重试其它服务器，通常用于读操作（推荐使用）
  - Failfast Cluster
    - 快速失败，只发起一次调用，失败立即报错,通常用于非幂等性的写操作
  - Failsafe Cluster
    - 失败安全，出现异常时，直接忽略，通常用于写入审计日志等操作
  - Failback Cluster
    - 失败自动恢复，后台记录失败请求，定时重发，通常用于消息通知操作
  - Forking Cluster
    - 并行调用多个服务器，只要一个成功即返回，通常用于实时性要求较高的读操作
  - Random LoadBalance
    - 随机，按权重设置随机概率（推荐使用）
  - RoundRobin LoadBalance
    - 轮询，按公约后的权重设置轮询比率
  - LeastActive LoadBalance
    - 最少活跃调用数，相同活跃数的随机，活跃数指调用前后计数差，使慢的机器收到更少请求
  - ConsistentHash LoadBalance
    - 一致性Hash，相同参数的请求总是发到同一提供者，当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动
  - Spring Container
    - 自动加载META-INF/spring目录下的所有Spring配置





## 配置

- xml

  - | 标签                                                         | 用途         | 解释                                                         |
    | ------------------------------------------------------------ | ------------ | ------------------------------------------------------------ |
    | `<dubbo:service/>`                                           | 服务配置     | 用于暴露一个服务，定义服务的元信息，一个服务可以用多个协议暴露，一个服务也可以注册到多个注册中心 |
    | `<dubbo:reference/>` [[2\]](http://dubbo.apache.org/zh-cn/docs/user/configuration/xml.html#fn2) | 引用配置     | 用于创建一个远程服务代理，一个引用可以指向多个注册中心       |
    | `<dubbo:protocol/>`                                          | 协议配置     | 用于配置提供服务的协议信息，协议由提供方指定，消费方被动接受 |
    | `<dubbo:application/>`                                       | 应用配置     | 用于配置当前应用信息，不管该应用是提供者还是消费者           |
    | `<dubbo:module/>`                                            | 模块配置     | 用于配置当前模块信息，可选                                   |
    | `<dubbo:registry/>`                                          | 注册中心配置 | 用于配置连接注册中心相关信息                                 |
    | `<dubbo:monitor/>`                                           | 监控中心配置 | 用于配置连接监控中心相关信息，可选                           |
    | `<dubbo:provider/>`                                          | 提供方配置   | 当 ProtocolConfig 和 ServiceConfig 某属性没有配置时，采用此缺省值，可选 |
    | `<dubbo:consumer/>`                                          | 消费方配置   | 当 ReferenceConfig 某属性没有配置时，采用此缺省值，可选      |
    | `<dubbo:method/>`                                            | 方法配置     | 用于 ServiceConfig 和 ReferenceConfig 指定方法级的配置信息   |
    | `<dubbo:argument/>`                                          | 参数配置     | 用于指定方法参数配置                                         |

  - 方法级优先，接口级次之，全局配置再次之。

  - 如果级别一样，则消费方优先，提供方次之。

- 属性

  - Dubbo可以自动加载classpath根目录下的dubbo.properties，但是你同样可以使用JVM参数来指定路径：`-Ddubbo.properties.file=xxx.properties`。
  - 优先级从高到低：
    - JVM -D参数，当你部署或者启动应用时，它可以轻易地重写配置，比如，改变dubbo协议端口；
    - XML, XML中的当前配置会重写dubbo.properties中的；
    - Properties，默认配置，仅仅作用于以上两者没有配置时。

- API

  - API 属性与配置项一对一，比如：`ApplicationConfig.setName("xxx")` 对应  `<dubbo:application name="xxx" />` 

- 注解

  - ```java
    @Configuration
    @EnableDubbo(scanBasePackages = "org.apache.dubbo.samples.simple.annotation.action")
    @PropertySource("classpath:/spring/dubbo-consumer.properties")
    @ComponentScan(value = {"org.apache.dubbo.samples.simple.annotation.action"})
    static public class ConsumerConfiguration {
    
    }
    
    @Component("annotationAction")
    public class AnnotationAction {
    
        @Reference
        private AnnotationService annotationService;
        
        public String doSayHello(String name) {
            return annotationService.sayHello(name);
        }
    }
    
    
    
    @Configuration
    @EnableDubbo(scanBasePackages = "org.apache.dubbo.samples.simple.annotation.impl")
    @PropertySource("classpath:/spring/dubbo-provider.properties")
    static public class ProviderConfiguration {
           
    }
    
    @Service
    public class AnnotationServiceImpl implements AnnotationService {
        @Override
        public String sayHello(String name) {
            return "annotation: hello, " + name;
        }
    }
    
    ```

- 动态配置中心

  - Zookeeper
  - Apollo







## 示例

- 启动时检查

  - Dubbo 缺省会在启动时检查依赖的服务是否可用，不可用时会抛出异常，阻止 Spring 初始化完成，以便上线时，能及早发现问题，默认  `check="true"`。
  - 另外，如果你的 Spring 容器是懒加载的，或者通过 API 编程延迟引用服务，请关闭 check，否则服务临时不可用时，会抛出异常，拿到 null 引用，如果 `check="false"`，总是会返回引用，当服务恢复时，能自动连上。

- 集群容错

  - 在集群调用失败时，Dubbo 提供了多种容错方案，缺省为 failover 重试。

- 负载均衡

  - 在集群负载均衡时，Dubbo 提供了多种均衡策略，缺省为 `random` 随机调用。

- 线程模型

  - 需要通过不同的派发策略和不同的线程池配置的组合来应对不同的场景

  - ```xml
    <dubbo:protocol name="dubbo" dispatcher="all" threadpool="fixed" threads="100" />
    
    all 所有消息都派发到线程池，包括请求，响应，连接事件，断开事件，心跳等。
    direct 所有消息都不派发到线程池，全部在 IO 线程上直接执行。
    message 只有请求响应消息派发到线程池，其它连接断开事件，心跳等消息，直接在 IO 线程上执行。
    execution 只有请求消息派发到线程池，不含响应，响应和其它连接断开事件，心跳等消息，直接在 IO 线程上执行。
    connection 在 IO 线程上，将连接断开事件放入队列，有序逐个执行，其它消息派发到线程池。
    
    fixed 固定大小线程池，启动时建立线程，不关闭，一直持有。(缺省)
    cached 缓存线程池，空闲一分钟自动删除，需要时重建。
    limited 可伸缩线程池，但池中的线程数只会增长不会收缩。只增长不收缩的目的是为了避免收缩时突然来了大流量引起的性能问题。
    
    ```

- 多协议

  - Dubbo 允许配置多协议，在不同服务上支持不同协议或者同一服务上同时支持多种协议。

- 多注册中心

  - Dubbo 支持同一服务向多注册中心同时注册，或者不同服务分别注册到不同的注册中心上去，甚至可以同时引用注册在不同注册中心上的同名服务。另外，注册中心是支持自定义扩展的

- 服务分组

  - 当一个接口有多种实现时，可以用 group 区分。

    ```xml
    服务
    <dubbo:service group="feedback" interface="com.xxx.IndexService" />
    <dubbo:service group="member" interface="com.xxx.IndexService" />
    
    引用
    <dubbo:reference id="feedbackIndexService" group="feedback" interface="com.xxx.IndexService" />
    <dubbo:reference id="memberIndexService" group="member" interface="com.xxx.
    ```

- 多版本

  - 当一个接口实现，出现不兼容升级时，可以用版本号过渡，版本号不同的服务相互间不引用。

- 隐式参数

  - 可以通过 `RpcContext` 上的 `setAttachment` 和 `getAttachment` 在服务消费方和提供方之间进行参数的隐式传递

- 异步调用

  - 从v2.7.0开始，Dubbo的所有异步编程接口开始以[CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)为基础
  - 基于 NIO 的非阻塞实现并行调用，Consumer不需要启动多线程即可完成并行调用多个远程服务，相对多线程开销较小。
  - Provider端异步执行将阻塞的业务从Dubbo内部线程池切换到业务自定义线程，避免Dubbo线程池的过度占用，有助于避免不同服务间的互相影响。异步执行无益于节省资源或提升RPC响应性能，因为如果业务执行需要阻塞，则始终还是要有线程来负责执行。

- 结果缓存

  - `lru` 基于最近最少使用原则删除多余缓存，保持最热的数据被缓存。
  - `threadlocal` 当前线程缓存，比如一个页面渲染，用到很多 portal，每个 portal 都要去查用户信息，通过线程缓存，可以减少这种多余访问。
  - `jcache` 与 [JSR107](http://jcp.org/en/jsr/detail?id=107') 集成，可以桥接各种缓存实现。

- 服务降级

  - 

- 延迟暴露

  - 如果你的服务需要预热时间，比如初始化缓存，等待相关资源就位等，可以使用 delay 进行延迟暴露。我们在 Dubbo 2.6.5 版本中对服务延迟暴露逻辑进行了细微的调整，将需要延迟暴露（delay > 0）服务的倒计时动作推迟到了 Spring 初始化完成后进行。你在使用 Dubbo 的过程中，并不会感知到此变化，因此请放心使用。

- 路由规则

  - 路由规则在发起一次RPC调用前起到过滤目标服务器地址的作用，过滤后的地址列表，将作为消费端最终发起RPC调用的备选地址。

  - 条件路由。支持以服务或Consumer应用为粒度配置路由规则。

    - ```yaml
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
      
      # Key明确规则体作用在哪个服务或应用
      # force=false 当路由结果为空时，是否强制执行，如果不强制执行，路由结果为空的路由规则将自动失效，可不填，缺省为 false。
      
      ---
      
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

      

  - 标签路由通过将某一个或多个服务的提供者划分到同一个分组，约束流量只在指定分组中流转，从而实现流量隔离的目的

- 并发控制

  - 服务器端、客户端
    - executes、actives
  - Load Balance 均衡
    - 配置服务的客户端的 `loadbalance` 属性为 `leastactive`，此 Loadbalance 会调用并发数最小的 Provider（Consumer端并发数）

- 连接控制

  - 服务端、客户端连接控制
    - accepts、connections







## 推荐用法

- 在 Provider 端尽量多配置 Consumer 端属性

  - 作服务的提供方，比服务消费方更清楚服务的性能参数，如调用的超时时间、合理的重试次数等

  - 在 Provider 端配置后，Consumer 端不配置则会使用 Provider 端的配置，即 Provider 端的配置可以作为 Consumer 的缺省值。否则，Consumer 会使用 Consumer 端的全局设置，这对于 Provider 是不可控的，并且往往是不合理的

  - 建议在 Provider 端配置的 Consumer 端属性有

    - > 1. `timeout`：方法调用的超时时间
      > 2. `retries`：失败重试次数，缺省是 2
      > 3. `loadbalance`：负载均衡算法，缺省是随机 `random`。还可以配置轮询 `roundrobin`、最不活跃优先 `leastactive` 和一致性哈希 `consistenthash` 等
      > 4. `actives`：消费者端的最大并发调用限制，即当 Consumer 对一个服务的并发调用到上限后，新调用会阻塞直到超时，在方法上配置 `dubbo:method` 则针对该方法进行并发限制，在接口上配置 `dubbo:service`，则针对该服务进行并发限制

- 在 Provider 端配置合理的 Provider 端属性

  - 建议在 Provider 端配置的 Provider 端属性有

    - > 1. `threads`：服务线程池大小
      > 2. `executes`：一个服务提供者并行执行请求上限，即当 Provider 对一个服务的并发调用达到上限后，新调用会阻塞，此时 Consumer 可能会超时。在方法上配置 `dubbo:method` 则针对该方法进行并发限制，在接口上配置 `dubbo:service`，则针对该服务进行并发限制

- 不要使用 dubbo.properties 文件配置，推荐使用对应 XML 配置

  





## 框架设计

- **config 配置层**：对外配置接口，以 `ServiceConfig`, `ReferenceConfig` 为中心，可以直接初始化配置类，也可以通过 spring 解析配置生成配置类
- **proxy 服务代理层**：服务接口透明代理，生成服务的客户端 Stub 和服务器端 Skeleton, 以 `ServiceProxy` 为中心，扩展接口为 `ProxyFactory`
- **registry 注册中心层**：封装服务地址的注册与发现，以服务 URL 为中心，扩展接口为 `RegistryFactory`, `Registry`, `RegistryService`
- **cluster 路由层**：封装多个提供者的路由及负载均衡，并桥接注册中心，以 `Invoker` 为中心，扩展接口为 `Cluster`, `Directory`, `Router`, `LoadBalance`
- **monitor 监控层**：RPC 调用次数和调用时间监控，以 `Statistics` 为中心，扩展接口为 `MonitorFactory`, `Monitor`, `MonitorService`
- **protocol 远程调用层**：封装 RPC 调用，以 `Invocation`, `Result` 为中心，扩展接口为 `Protocol`, `Invoker`, `Exporter`
- **exchange 信息交换层**：封装请求响应模式，同步转异步，以 `Request`, `Response` 为中心，扩展接口为 `Exchanger`, `ExchangeChannel`, `ExchangeClient`, `ExchangeServer`
- **transport 网络传输层**：抽象 mina 和 netty 为统一接口，以 `Message` 为中心，扩展接口为 `Channel`, `Transporter`, `Client`, `Server`, `Codec`
- **serialize 数据序列化层**：可复用的一些工具，扩展接口为 `Serialization`, `ObjectInput`, `ObjectOutput`, `ThreadPool`

