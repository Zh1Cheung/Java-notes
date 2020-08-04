-

# 背景

- 当垂直应用越来越多，应用之间交互不可避免，将核心业务抽取出来，作为独立的服务，逐渐形成稳定的服务中心，使前端应用能更快速的响应多变的市场需求。此时，用于提高业务复用及整合的分布式服务框架(RPC)是关键。
- 在大规模服务化之前，应用可能只是通过 RMI 或 Hessian 等工具，简单的暴露和引用远程服务，通过配置服务的URL地址进行调用，通过 F5 等硬件进行负载均衡
  - 当服务越来越多时，服务 URL 配置管理变得非常困难，F5 硬件负载均衡器的单点压力也越来越大
  - 当进一步发展，服务间依赖关系变得错踪复杂，甚至分不清哪个应用要在哪个应用之前启动
  - 接着，服务的调用量越来越大，服务的容量问题就暴露出来，这个服务需要多少机器支撑？什么时候该加机器？

- Dubbo 采用全 Spring 配置方式，透明化接入应用，对应用没有任何 API 侵入，只需用 Spring 加载 Dubbo 的配置即可，Dubbo 基于 [Spring 的 Schema 扩展](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/xsd-configuration.html) 进行加载。



# 架构

![dubbo-architucture](http://dubbo.apache.org/docs/zh-cn/user/sources/images/dubbo-architecture.jpg)

## 节点角色说明

| 节点        | 角色说明                               |
| ----------- | -------------------------------------- |
| `Provider`  | 暴露服务的服务提供方                   |
| `Consumer`  | 调用远程服务的服务消费方               |
| `Registry`  | 服务注册与发现的注册中心               |
| `Monitor`   | 统计服务的调用次数和调用时间的监控中心 |
| `Container` | 服务运行容器                           |

## 调用关系说明

1. 服务容器负责启动，加载，运行服务提供者。
2. 服务提供者在启动时，向注册中心注册自己提供的服务。
3. 服务消费者在启动时，向注册中心订阅自己所需的服务。
4. 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。
5. 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
6. 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。

Dubbo 架构具有以下几个特点，分别是连通性、健壮性、伸缩性、以及向未来架构的升级性。

## 连通性

- 注册中心负责服务地址的注册与查找，相当于目录服务，服务提供者和消费者只在启动时与注册中心交互，注册中心不转发请求，压力较小
- 监控中心负责统计各服务调用次数，调用时间等，统计先在内存汇总后每分钟一次发送到监控中心服务器，并以报表展示
- 服务提供者向注册中心注册其提供的服务，并汇报调用时间到监控中心，此时间不包含网络开销
- 服务消费者向注册中心获取服务提供者地址列表，并根据负载算法直接调用提供者，同时汇报调用时间到监控中心，此时间包含网络开销
- 注册中心，服务提供者，服务消费者三者之间均为长连接，监控中心除外
- 注册中心通过长连接感知服务提供者的存在，服务提供者宕机，注册中心将立即推送事件通知消费者
- 注册中心和监控中心全部宕机，不影响已运行的提供者和消费者，消费者在本地缓存了提供者列表
- 注册中心和监控中心都是可选的，服务消费者可以直连服务提供者

## 健壮性

- 监控中心宕掉不影响使用，只是丢失部分采样数据
- 数据库宕掉后，注册中心仍能通过缓存提供服务列表查询，但不能注册新服务
- 注册中心对等集群，任意一台宕掉后，将自动切换到另一台
- 注册中心全部宕掉后，服务提供者和服务消费者仍能通过本地缓存通讯
- 服务提供者无状态，任意一台宕掉后，不影响使用
- 服务提供者全部宕掉后，服务消费者应用将无法使用，并无限次重连等待服务提供者恢复

## 伸缩性

- 注册中心为对等集群，可动态增加机器部署实例，所有客户端将自动发现新的注册中心
- 服务提供者无状态，可动态增加机器部署实例，注册中心将推送新的服务提供者信息给消费者



# 用法

- 服务提供者

  完整安装步骤，请参见：[示例提供者安装](http://dubbo.apache.org/zh-cn/docs/admin/install/provider-demo.html)

  - 定义服务接口

  DemoService.java [[1\]](http://dubbo.apache.org/zh-cn/docs/user/quick-start.html#fn1)：

  ```java
  package org.apache.dubbo.demo;
  
  public interface DemoService {
      String sayHello(String name);
  }
  ```

  - 在服务提供方实现接口

  DemoServiceImpl.java [[2\]](http://dubbo.apache.org/zh-cn/docs/user/quick-start.html#fn2)：

  ```java
  package org.apache.dubbo.demo.provider;
   
  import org.apache.dubbo.demo.DemoService;
   
  public class DemoServiceImpl implements DemoService {
      public String sayHello(String name) {
          return "Hello " + name;
      }
  }
  ```

  - 用 Spring 配置声明暴露服务

  provider.xml：

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
      xsi:schemaLocation="http://www.springframework.org/schema/beans        http://www.springframework.org/schema/beans/spring-beans-4.3.xsd        http://dubbo.apache.org/schema/dubbo        http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
   
      <!-- 提供方应用信息，用于计算依赖关系 -->
      <dubbo:application name="hello-world-app"  />
   
      <!-- 使用multicast广播注册中心暴露服务地址 -->
      <dubbo:registry address="multicast://224.5.6.7:1234" />
   
      <!-- 用dubbo协议在20880端口暴露服务 -->
      <dubbo:protocol name="dubbo" port="20880" />
   
      <!-- 声明需要暴露的服务接口 -->
      <dubbo:service interface="org.apache.dubbo.demo.DemoService" ref="demoService" />
   
      <!-- 和本地bean一样实现服务 -->
      <bean id="demoService" class="org.apache.dubbo.demo.provider.DemoServiceImpl" />
  </beans>
  ```

- 加载 Spring 配置

  Provider.java：

  ```java
  import org.springframework.context.support.ClassPathXmlApplicationContext;
   
  public class Provider {
      public static void main(String[] args) throws Exception {
          ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[] {"http://10.20.160.198/wiki/display/dubbo/provider.xml"});
          context.start();
          System.in.read(); // 按任意键退出
      }
  }
  ```

- 服务消费者

  完整安装步骤，请参见：[示例消费者安装](http://dubbo.apache.org/zh-cn/docs/admin/install/consumer-demo.html)

  - 通过 Spring 配置引用远程服务

  consumer.xml：

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
      xsi:schemaLocation="http://www.springframework.org/schema/beans        http://www.springframework.org/schema/beans/spring-beans-4.3.xsd        http://dubbo.apache.org/schema/dubbo        http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
   
      <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->
      <dubbo:application name="consumer-of-helloworld-app"  />
   
      <!-- 使用multicast广播注册中心暴露发现服务地址 -->
      <dubbo:registry address="multicast://224.5.6.7:1234" />
   
      <!-- 生成远程服务代理，可以和本地bean一样使用demoService -->
      <dubbo:reference id="demoService" interface="org.apache.dubbo.demo.DemoService" />
  </beans>
  ```

  - 加载Spring配置，并调用远程服务

  Consumer.java [[3\]](http://dubbo.apache.org/zh-cn/docs/user/quick-start.html#fn3)：

  ```java
  import org.springframework.context.support.ClassPathXmlApplicationContext;
  import org.apache.dubbo.demo.DemoService;
   
  public class Consumer {
      public static void main(String[] args) throws Exception {
          ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[] {"http://10.20.160.198/wiki/display/dubbo/consumer.xml"});
          context.start();
          DemoService demoService = (DemoService)context.getBean("demoService"); // 获取远程服务代理
          String hello = demoService.sayHello("world"); // 执行远程方法
          System.out.println( hello ); // 显示调用结果
      }
  }
  ```

  ------

  



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



## 配置加载流程

- 在**应用启动阶段，Dubbo框架如何将所需要的配置采集起来**（包括应用配置、注册中心配置、服务配置等），以完成服务的暴露和引用流程。

- Dubbo的配置读取总体上遵循了以下几个原则：

  1. Dubbo支持了多层级的配置，并按预定优先级自动实现配置间的覆盖，最终所有配置汇总到数据总线URL后驱动后续的服务暴露、引用等流程。
  2. ApplicationConfig、ServiceConfig、ReferenceConfig可以被理解成配置来源的一种，是直接面向用户编程的配置采集方式。
  3. 配置格式以Properties为主，在配置内容上遵循约定的`path-based`的命名[规范](http://dubbo.apache.org/zh-cn/docs/user/configuration/configuration-load-process.html#配置格式)

- 首先，从Dubbo支持的配置来源说起，默认有四种配置来源（优先级依次降低）：

  - JVM System Properties，-D参数
  - Externalized Configuration，外部化配置
  - ServiceConfig、ReferenceConfig等编程接口采集的配置
  - 本地配置文件dubbo.properties

- > *# 应用级别*
  > **dubbo.{config-type}[.{config-id}].{config-item}**={config-item-value}
  >
  > *# 服务级别*
  > **dubbo.service.{interface-name}[.{method-name}].{config-item}**={config-item-value}
  > **dubbo.reference.{interface-name}[.{method-name}].{config-item}**={config-item-value}
  >
  > *# 多配置项*
  > **dubbo.{config-type}s.{config-id}.{config-item}**={config-item-value}
  >
  > 
  >
  > 









# 推荐用法

## 在 Provider 端尽量多配置 Consumer 端属性

原因如下：

- 作服务的提供方，比服务消费方更清楚服务的性能参数，如调用的超时时间、合理的重试次数等
- 在 Provider 端配置后，Consumer 端不配置则会使用 Provider 端的配置，即 Provider 端的配置可以作为 Consumer 的缺省值 [[1\]](http://dubbo.apache.org/zh-cn/docs/user/recommend.html#fn1)。否则，Consumer 会使用 Consumer 端的全局设置，这对于 Provider 是不可控的，并且往往是不合理的

Provider 端尽量多配置 Consumer 端的属性，让 Provider 的实现者一开始就思考 Provider 端的服务特点和服务质量等问题。

示例：

```xml
<dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService"
    timeout="300" retries="2" loadbalance="random" actives="0" />
 
<dubbo:service interface="com.alibaba.hello.api.WorldService" version="1.0.0" ref="helloService"
    timeout="300" retries="2" loadbalance="random" actives="0" >
    <dubbo:method name="findAllPerson" timeout="10000" retries="9" loadbalance="leastactive" actives="5" />
<dubbo:service/>
```

建议在 Provider 端配置的 Consumer 端属性有：

1. `timeout`：方法调用的超时时间
2. `retries`：失败重试次数，缺省是 2 [[2\]](http://dubbo.apache.org/zh-cn/docs/user/recommend.html#fn2)
3. `loadbalance`：负载均衡算法 [[3\]](http://dubbo.apache.org/zh-cn/docs/user/recommend.html#fn3)，缺省是随机 `random`。还可以配置轮询 `roundrobin`、最不活跃优先 [[4\]](http://dubbo.apache.org/zh-cn/docs/user/recommend.html#fn4) `leastactive` 和一致性哈希 `consistenthash` 等
4. `actives`：消费者端的最大并发调用限制，即当 Consumer 对一个服务的并发调用到上限后，新调用会阻塞直到超时，在方法上配置 `dubbo:method` 则针对该方法进行并发限制，在接口上配置 `dubbo:service`，则针对该服务进行并发限制

详细配置说明请参考：[Dubbo配置参考手册](http://dubbo.apache.org/zh-cn/docs/user/references/xml/introduction.html)

## 在 Provider 端配置合理的 Provider 端属性

```xml
<dubbo:protocol threads="200" /> 
<dubbo:service interface="com.alibaba.hello.api.HelloService" version="1.0.0" ref="helloService"
    executes="200" >
    <dubbo:method name="findAllPerson" executes="50" />
</dubbo:service>
```

建议在 Provider 端配置的 Provider 端属性有：

1. `threads`：服务线程池大小
2. `executes`：一个服务提供者并行执行请求上限，即当 Provider 对一个服务的并发调用达到上限后，新调用会阻塞，此时 Consumer 可能会超时。在方法上配置 `dubbo:method` 则针对该方法进行并发限制，在接口上配置 `dubbo:service`，则针对该服务进行并发限制

## 配置管理信息

目前有负责人信息和组织信息用于区分站点。以便于在发现问题时找到服务对应负责人，建议至少配置两个人以便备份。负责人和组织信息可以在运维平台 (Dubbo Ops) 上看到。

在应用层面配置负责人、组织信息：

```xml
<dubbo:application owner=”ding.lid,william.liangf” organization=”intl” />
```

在服务层面（服务端）配置负责人：

```xml
<dubbo:service owner=”ding.lid,william.liangf” />
```

在服务层面（消费端）配置负责人：

```xml
<dubbo:reference owner=”ding.lid,william.liangf” />
```

若没有配置服务层面的负责人，则默认使用 `dubbo:application` 设置的负责人。

## 配置 Dubbo 缓存文件

提供者列表缓存文件：

```xml
<dubbo:registry file=”${user.home}/output/dubbo.cache” />
```

注意：

1. 可以根据需要调整缓存文件的路径，保证这个文件不会在发布过程中被清除；
2. 如果有多个应用进程，请注意不要使用同一个文件，避免内容被覆盖；

该文件会缓存注册中心列表和服务提供者列表。配置缓存文件后，应用重启过程中，若注册中心不可用，应用会从该缓存文件读取服务提供者列表，进一步保证应用可靠性。

## 监控配置

1. 使用固定端口暴露服务，而不要使用随机端口

   这样在注册中心推送有延迟的情况下，消费者通过缓存列表也能调用到原地址，保证调用成功。

2. 使用 Dubbo Admin 监控注册中心上的服务提供方

   使用 [Dubbo Admin](https://github.com/apache/dubbo-admin) 监控服务在注册中心上的状态，确保注册中心上有该服务的存在。

3. 服务提供方可使用 Dubbo Qos 的 telnet 或 shell 监控项

   监控服务提供者端口状态：`echo status | nc -i 1 20880 | grep OK | wc -l`，其中的 20880 为服务端口

4. 服务消费方可通过将服务强制转型为 EchoService，并调用 `$echo()` 测试该服务的提供者是可用

   如 `assertEqauls(“OK”, ((EchoService)memberService).$echo(“OK”));`

## 不要使用 dubbo.properties 文件配置，推荐使用对应 XML 配置

Dubbo 中所有的配置项都可以配置在 Spring 配置文件中，并且可以针对单个服务配置。

如完全不配置则使用 Dubbo 缺省值，详情请参考 [Dubbo配置参考手册](http://dubbo.apache.org/zh-cn/docs/user/references/xml/introduction.html) 中的说明。

### dubbo.properties 中属性名与 XML 的对应关系

1. 应用名 `dubbo.application.name`

   ```xml
   <dubbo:application name="myalibaba" >
   ```

2. 注册中心地址 `dubbo.registry.address`

   ```xml
   <dubbo:registry address="11.22.33.44:9090" >
   ```

3. 调用超时 `dubbo.service.*.timeout`

   可以在多个配置项设置超时 `timeout`，由上至下覆盖（即上面的优先）[[5\]](http://dubbo.apache.org/zh-cn/docs/user/recommend.html#fn5)，其它的参数（`retries`、`loadbalance`、`actives`等）的覆盖策略与 `timeout` 相同。示例如下：

   提供者端特定方法的配置

   ```xml
   <dubbo:service interface="com.alibaba.xxx.XxxService" >
       <dubbo:method name="findPerson" timeout="1000" />
   </dubbo:service>
   ```

   提供者端特定接口的配置

   ```xml
   <dubbo:service interface="com.alibaba.xxx.XxxService" timeout="200" />
   ```

4. 服务提供者协议 `dubbo.service.protocol`、服务的监听端口 `dubbo.service.server.port`

   ```xml
   <dubbo:protocol name="dubbo" port="20880" />
   ```

5. 服务线程池大小 `dubbo.service.max.thread.threads.size`

   ```xml
   <dubbo:protocol threads="100" />
   ```

6. 消费者启动时，没有提供者是否抛异常 `alibaba.intl.commons.dubbo.service.allow.no.provider`

   ```xml
   <dubbo:reference interface="com.alibaba.xxx.XxxService" check="false" />
   ```

------

1. 配置的覆盖规则：1) 方法级别配置优于接口级别，即小 Scope 优先 2) Consumer 端配置优于 Provider 端配置，优于全局配置，最后是 Dubbo 硬编码的配置值（[Dubbo 配置参考手册](http://dubbo.apache.org/zh-cn/docs/user/configuration/properties.html)) [↩︎](http://dubbo.apache.org/zh-cn/docs/user/recommend.html#fnref1)
2. 表示加上第一次调用，会调用 3 次 [↩︎](http://dubbo.apache.org/zh-cn/docs/user/recommend.html#fnref2)
3. 有多个 Provider 时，如何挑选 Provider 调用 [↩︎](http://dubbo.apache.org/zh-cn/docs/user/recommend.html#fnref3)
4. 指从 Consumer 端并发调用最好的 Provider，可以减少对响应慢的 Provider 的调用，因为响应慢更容易累积并发调用 [↩︎](http://dubbo.apache.org/zh-cn/docs/user/recommend.html#fnref4)
5. `timeout` 可以在多处设置，配置项及覆盖规则请参考： [Dubbo 配置参考手册](http://dubbo.apache.org/zh-cn/docs/user/references/xml/introduction.html) [↩︎](http://dubbo.apache.org/zh-cn/docs/user/recommend.html#fnref5)



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

