# SPI

SPI 全称为 Service Provider Interface，是一种**服务发现机制**。SPI 的本质是将接口实现类的全限定名配置在文件中，并由服务加载器读取配置文件，加载实现类。这样可以在运行时，动态为接口替换实现类。正因此特性，我们可以很容易的通过 SPI 机制**为我们的程序提供拓展功能**。SPI 机制在第三方框架中也有所应用，比如 Dubbo 就是通过 SPI 机制加载所有的组件。不过，Dubbo 并未使用 Java 原生的 SPI 机制，而是对其进行了增强，使其能够更好的满足需求。在 Dubbo 中，SPI 是一个非常重要的模块。基于 SPI，我们可以很容易的对 Dubbo 进行拓展。

## 2.SPI 示例

### 2.1 Java SPI 示例

前面简单介绍了 SPI 机制的原理，本节通过一个示例演示 Java SPI 的使用方法。首先，我们定义一个接口，名称为 Robot。

```java
public interface Robot {
    void sayHello();
}
```

接下来定义两个实现类，分别为 OptimusPrime 和 Bumblebee。

```java
public class OptimusPrime implements Robot {
    
    @Override
    public void sayHello() {
        System.out.println("Hello, I am Optimus Prime.");
    }
}

public class Bumblebee implements Robot {

    @Override
    public void sayHello() {
        System.out.println("Hello, I am Bumblebee.");
    }
}
```

接下来 META-INF/services 文件夹下创建一个文件，名称为 Robot 的全限定名 org.apache.spi.Robot。文件内容为实现类的全限定的类名，如下：

```
org.apache.spi.OptimusPrime
org.apache.spi.Bumblebee
```

做好所需的准备工作，接下来编写代码进行测试。

```java
public class JavaSPITest {

    @Test
    public void sayHello() throws Exception {
        ServiceLoader<Robot> serviceLoader = ServiceLoader.load(Robot.class);
        System.out.println("Java SPI");
        serviceLoader.forEach(Robot::sayHello);
    }
}
```

最后来看一下测试结果，如下：

![img](http://dubbo.apache.org/docs/zh-cn/source_code_guide/sources/images/java-spi-result.jpg)

从测试结果可以看出，我们的两个实现类被成功的加载，并输出了相应的内容。关于 Java SPI 的演示先到这里，接下来演示 Dubbo SPI。

### 2.2 Dubbo SPI 示例

Dubbo 并未使用 Java SPI，而是重新实现了一套功能更强的 SPI 机制。Dubbo SPI 的相关逻辑被封装在了 **ExtensionLoader 类**中，通过 ExtensionLoader，我们可以加载指定的实现类。Dubbo SPI 所需的配置文件需放置在 META-INF/dubbo 路径下，配置内容如下。

```
optimusPrime = org.apache.spi.OptimusPrime
bumblebee = org.apache.spi.Bumblebee
```

与 Java SPI 实现类配置不同，Dubbo SPI 是通过键值对的方式进行配置，这样我们可以按需加载指定的实现类。另外，在测试 Dubbo SPI 时，需要在 Robot 接口上标注 @SPI 注解。下面来演示 Dubbo SPI 的用法：

```java
public class DubboSPITest {

    @Test
    public void sayHello() throws Exception {
        ExtensionLoader<Robot> extensionLoader = 
            ExtensionLoader.getExtensionLoader(Robot.class);
        Robot optimusPrime = extensionLoader.getExtension("optimusPrime");
        optimusPrime.sayHello();
        Robot bumblebee = extensionLoader.getExtension("bumblebee");
        bumblebee.sayHello();
    }
}
```

测试结果如下：

![img](http://dubbo.apache.org/docs/zh-cn/source_code_guide/sources/images/dubbo-spi-result.jpg)

Dubbo SPI 除了支持按需加载接口实现类，还增加了 IOC 和 AOP 等特性，这些特性将会在接下来的源码分析章节中一一进行介绍。

## 3. Dubbo SPI 源码分析

上一章简单演示了 Dubbo SPI 的使用方法。我们首先通过 ExtensionLoader 的 getExtensionLoader 方法获取一个 ExtensionLoader 实例，然后再通过 ExtensionLoader 的 getExtension 方法获取拓展类对象。这其中，**getExtensionLoader 方法**用于从缓存中获取与拓展类对应的 ExtensionLoader，若缓存未命中，则创建一个新的实例。

- 下面我们从 ExtensionLoader 的 **getExtension 方法**作为入口，对拓展类对象的获取过程进行详细的分析。
  - 首先检查缓存，缓存未命中则创建拓展对象(双重检查单例模式创建)。	

    - ```java
          if (instance == null) {
              synchronized (holder) {
                  instance = holder.get();
                  if (instance == null) {
                      // 创建拓展实例
                      instance = createExtension(name);
                      // 设置实例到 holder 中
                      holder.set(instance);
                  }
              }
          }
      ```

  - 创建拓展对象的过程（**createExtension**(String name)）

    - createExtension 方法的逻辑稍复杂一下，包含了如下的步骤：

      1. 通过 getExtensionClasses 获取所有的拓展类
      2. 通过反射创建拓展对象
      3. 向拓展对象中注入依赖
      4. 将拓展对象包裹在相应的 Wrapper 对象中

      以上步骤中，第一个步骤是加载拓展类的关键，第三和第四个步骤是 Dubbo IOC 与 AOP 的具体实现。

### 3.1 获取所有的拓展类

- 我们在通过名称获取拓展类之前，首先需要根据配置文件解析出拓展项名称到拓展类的映射关系表（Map<名称, 拓展类>），之后再根据拓展项名称从映射关系表中取出相应的拓展类即可。

- 这里也是先检查缓存，若缓存未命中，则通过 synchronized 加锁。加锁后再次检查缓存，并判空。此时如果 classes 仍为 null，则通过 **loadExtensionClasses** 方法加载拓展类。

  - ```java
    private Map<String, Class<?>> getExtensionClasses() {
        // 从缓存中获取已加载的拓展类
        Map<String, Class<?>> classes = cachedClasses.get();
        // 双重检查
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    // 加载拓展类
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
    ```

- loadExtensionClasses 方法

  - loadExtensionClasses 方法总共做了两件事情，一是对 SPI 注解进行解析，二是调用 **loadDirectory 方法**加载指定文件夹配置文件。SPI 注解解析过程比较简单，无需多说。

- loadDirectory 方法
  - loadDirectory 方法先通过 classLoader 获取所有资源链接，然后再通过 loadResource 方法加载资源。
-  loadResource 方法
  - loadResource 方法用于读取和解析配置文件，并通过反射加载类，最后调用 loadClass 方法进行其他操作。loadClass 方法用于主要用于操作缓存
- loadClass 方法
  - loadClass 方法操作了不同的缓存，比如 cachedAdaptiveClass、cachedWrapperClasses 和 cachedNames 等等。除此之外，该方法没有其他什么逻辑了。

### 3.2 Dubbo IOC

Dubbo IOC 是通过 setter 方法注入依赖。Dubbo 首先会通过反射获取到实例的所有方法，然后再遍历方法列表，检测方法名是否具有 setter 方法特征。若有，则通过 ObjectFactory 获取依赖对象，最后通过反射调用 setter 方法将依赖设置到目标对象中。

bjectFactory 变量的类型为 AdaptiveExtensionFactory，AdaptiveExtensionFactory 内部维护了一个 ExtensionFactory 列表，用于存储其他类型的 ExtensionFactory。Dubbo 目前提供了两种 ExtensionFactory，分别是 SpiExtensionFactory 和 SpringExtensionFactory。前者用于创建自适应的拓展，后者是用于从 Spring 的 IOC 容器中获取所需的拓展

Dubbo IOC 目前仅支持 setter 方式注入，总的来说，逻辑比较简单易懂。



# 服务导出

Dubbo 服务导出过程始于 Spring 容器发布刷新事件，Dubbo 在接收到事件后，会立即执行服务导出逻辑。**整个逻辑大致可分为三个部分**，第一部分是前置工作，主要用于检查参数，组装 URL。第二部分是导出服务，包含导出服务到本地 (JVM)，和导出服务到远程两个过程。第三部分是向注册中心注册服务，用于服务发现。

- 服务导出的入口方法是 ServiceBean 的 **onApplicationEvent**。onApplicationEvent 是一个事件响应方法，该方法会在收到 Spring 上下文刷新事件后执行服务导出操作。
  - 这个方法首先会根据条件决定是否导出服务，比如有些服务设置了延时导出，那么此时就不应该在此处导出。还有一些服务已经被导出了，或者当前服务被取消导出了，此时也不能再次导出相关服务。注意这里的 **isDelay 方法**，这个方法字面意思是“是否延迟导出服务”，返回 true 表示延迟导出，false 表示不延迟导出。但是该方法真实意思却并非如此，当方法返回 true 时，表示无需延迟导出。返回 false 时，表示需要延迟导出。与字面意思恰恰相反，这个需要大家注意一下。

- isDelay()方法

  - 该方法目前已被重构
  - 现在解释一下 supportedApplicationListener 变量含义，该变量用于表示当前的 Spring 容器是否支持 ApplicationListener，这个值初始为 false。在 Spring 容器将自己设置到 ServiceBean 中时，ServiceBean 的 setApplicationContext 方法会检测 Spring 容器是否支持 ApplicationListener。若支持，则将 supportedApplicationListener 置为 true。ServiceBean 是 Dubbo 与 Spring 框架进行整合的关键，可以看做是两个框架之间的桥梁。具有同样作用的类还有 ReferenceBean。

  现在我们知道了 Dubbo 服务导出过程的起点，接下来对服务导出的前置逻辑进行分析。

### 	 2.1 前置工作

**前置工作主要包含两个部分**，分别是配置检查，以及 URL 装配。在导出服务之前，Dubbo 需要检查用户的配置是否合理，或者为用户补充缺省配置。配置检查完成后，接下来需要根据这些配置组装 URL。在 Dubbo 中，URL 的作用十分重要。Dubbo 使用 URL 作为配置载体，所有的拓展点都是通过 URL 获取配置。这一点，官方文档中有所说明。

> 采用 URL 作为配置信息的统一格式，所有扩展点都通过传递 URL 携带配置信息。

接下来，我们先来分析配置检查部分的源码，随后再来分析 URL 组装部分的源码。

#### 2.1.1 检查配置

本节我们接着前面的源码向下分析，前面说过 onApplicationEvent 方法在经过一些判断后，会决定是否调用 **export 方法**导出服务。那么下面我们从 export 方法开始进行分析

- **export 方法对两项配置进行了检查，并根据配置执行相应的动作。**首先是 export 配置，这个配置决定了是否导出服务。有时候我们只是想本地启动服务进行一些调试工作，我们并不希望把本地启动的服务暴露出去给别人调用。此时，我们可通过配置 export 禁止服务导出，比如：

```xml
<dubbo:provider export="false" />
```

- doExport 方法
  - 下面对配置检查的逻辑进行简单的总结，如下：
    1. 检测 <dubbo:service> 标签的 interface 属性合法性，不合法则抛出异常
    2. 检测 ProviderConfig、ApplicationConfig 等核心配置类对象是否为空，若为空，则尝试从其他配置类对象中获取相应的实例。
    3. 检测并处理泛化服务和普通服务类
    4. 检测本地存根配置，并进行相应的处理
    5. 对 ApplicationConfig、RegistryConfig 等配置类进行检测，为空则尝试创建，若无法创建则抛出异常

#### 2.1.2 多协议多注册中心导出服务

- Dubbo 允许我们使用不同的协议导出服务，也允许我们向多个注册中心注册服务。Dubbo 在 doExportUrls 方法中对**多协议，多注册中心**进行了支持。相关代码如下：

  ```java
  private void doExportUrls() {
      // 加载注册中心链接
      List<URL> registryURLs = loadRegistries(true);
      // 遍历 protocols，并在每个协议下导出服务
      for (ProtocolConfig protocolConfig : protocols) {
          doExportUrlsFor1Protocol(protocolConfig, registryURLs);
      }
  }
  ```

  上面代码首先是通过 **loadRegistries** 加载注册中心链接，然后再遍历 ProtocolConfig 集合导出每个服务。并在导出服务的过程中，将服务注册到注册中心。

-  loadRegistries 方法
  - loadRegistries 方法主要包含如下的逻辑：
    1. 检测是否存在注册中心配置类，不存在则抛出异常
    2. 构建参数映射集合，也就是 map
    3. 构建注册中心链接列表
    4. 遍历链接列表，并根据条件决定是否将其添加到 registryList 中

#### 2.1.3 组装 URL

配置检查完毕后，紧接着要做的事情是根据配置，以及其他一些信息组装 URL。前面说过，URL 是 Dubbo 配置的载体，通过 URL 可让 Dubbo 的各种配置在各个模块之间传递。URL 之于 Dubbo，犹如水之于鱼，非常重要。大家在阅读 Dubbo 服务导出相关源码的过程中，要注意 URL 内容的变化。既然 URL 如此重要，那么下面我们来了解一下 URL 组装的过程。

- **doExportUrlsFor1Protocol**方法
  - 首先是将一些信息，比如版本、时间戳、方法名以及各种配置对象的字段信息放入到 map 中，map 中的内容将作为 URL 的查询字符串。构建好 map 后，紧接着是获取上下文路径、主机名以及端口号等信息。最后将 map 和主机名等数据传给 URL 构造方法创建 URL 对象。需要注意的是，这里出现的 URL 并非 java.net.URL，而是 com.alibaba.dubbo.common.URL。



### 2.2 导出 Dubbo 服务

前置工作做完，接下来就可以进行服务导出了。服务导出分为导出到本地 (JVM)，和导出到远程。在深入分析服务导出的源码前，我们先来从宏观层面上看一下服务导出逻辑

- **doExportUrlsFor1Protocol**方法

  - 根据 url 中的 scope 参数决定服务导出方式，分别如下：

    - scope = none，不导出服务
    - scope != remote，导出到本地
    - scope != local，导出到远程

    不管是导出到本地，还是远程。进行服务导出之前，均需要先创建 Invoker，这是一个很重要的步骤。

  

### 2.2.1 Invoker 创建过程

- 在 Dubbo 中，Invoker 是一个非常重要的模型。在服务提供端，以及服务引用端均会出现 Invoker。Dubbo 官方文档中对 Invoker 进行了说明，这里引用一下。

  > Invoker 是实体域，它是 Dubbo 的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起 invoke 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。

  既然 Invoker 如此重要，那么我们很有必要搞清楚 Invoker 的用途。Invoker 是由 ProxyFactory 创建而来，Dubbo 默认的 ProxyFactory 实现类是 JavassistProxyFactory。下面我们到 JavassistProxyFactory 代码中，探索 Invoker 的创建过程。

- **getInvoker**方法

  - JavassistProxyFactory 创建了一个继承自 AbstractProxyInvoker 类的匿名对象，并覆写了抽象方法 doInvoke。覆写后的 doInvoke 逻辑比较简单，仅是将调用请求转发给了 Wrapper 类的 invokeMethod 方法。Wrapper 用于“包裹”目标类，Wrapper 是一个抽象类，仅可通过 **getWrapper(Class) 方法**创建子类。在创建 Wrapper 子类的过程中，子类代码生成逻辑会对 getWrapper 方法传入的 Class 对象进行解析，拿到诸如类方法，类成员变量等信息。以及生成 invokeMethod 方法代码和其他一些方法代码。代码生成完毕后，通过 Javassist 生成 Class 对象，最后再通过反射创建 Wrapper 实例。
  - getWrapper 方法仅包含一些缓存操作逻辑
    - 首先我们把目光移到分割线1之上的代码，这段代码主要用于进行一些**初始化操作**。比如创建 c1、c2、c3 以及 pts、ms、mns 等变量，以及向 c1、c2、c3 中添加方法定义和类型转换代码。接下来是分割线1到分割线2之间的代码，这段代码用于**为 public 级别的字段生成条件判断取值与赋值代码**。这段代码不是很难看懂，就不多说了。继续向下看，分割线2和分隔线3之间的代码用于**为定义在当前类中的方法生成判断语句，和方法调用语句**。因为需要对方法重载进行校验，因此到这这段代码看起来有点复杂。不过耐心看一下，也不是很难理解。接下来是分割线3和分隔线4之间的代码，这段代码用于**处理 getter、setter 以及以 is/has/can 开头的方法**。处理方式是通过正则表达式获取方法类型（get/set/is/...），以及属性名。之后为属性名生成判断语句，然后为方法生成调用语句。最后我们再来看一下分隔线4以下的代码，这段代码通过 ClassGenerator 为刚刚生成的代码构建 Class 类，并通过反射创建对象。ClassGenerator 是 Dubbo 自己封装的，该类的核心是 toClass() 的重载方法 toClass(ClassLoader, ProtectionDomain)，该方法通过 javassist 构建 Class

  ​	

### 2.2.2 导出服务到本地

按照代码执行顺序，本节先来分析导出服务到本地的过程。

- exportLocal 方法
  - exportLocal 方法比较简单，首先根据 URL 协议头决定是否导出服务。若需导出，则创建一个新的 URL 并将协议头、主机名以及端口设置成新的值。然后创建 Invoker，并调用 InjvmProtocol 的 export 方法导出服务。下面我们来看一下 InjvmProtocol 的 export 方法都做了哪些事情。
- InjvmProtocol 的 export 方法仅创建了一个 InjvmExporter，无其他逻辑。

### 2.2.3 导出服务到远程

与导出服务到本地相比，导出服务到远程的过程要复杂不少，其包含了服务导出与服务注册两个过程。这两个过程涉及到了大量的调用，比较复杂。按照代码执行顺序，本节先来分析服务导出逻辑，服务注册逻辑将在下一节进行分析。下面开始分析，我们把目光移动到 RegistryProtocol 的 export 方法上

主要做如下一些操作：

1. 调用 doLocalExport 导出服务
2. 向注册中心注册服务
3. 向注册中心进行订阅 override 数据
4. 创建并返回 DestroyableExporter

在以上操作中，除了创建并返回 DestroyableExporter 没什么难度外，其他几步操作都不是很简单。这其中，导出服务和注册服务是本章要重点分析的逻辑。 

- doLocalExport 方法
  - 上面的代码是典型的双重检查锁，大家在阅读 Dubbo 的源码中，会多次见到。接下来，我们把重点放在 Protocol 的 export 方法上。假设运行时协议为 dubbo，此处的 protocol 变量会在运行时加载 DubboProtocol，并调用 DubboProtocol 的 export 方法。
- export 方法
  - 重点关注 DubboExporter 的创建以及 openServer 方法
- openServer 方法
  - 如上，在同一台机器上（单网卡），同一个端口上仅允许启动一个服务器实例。若某个端口上已有服务器实例，此时则调用 reset 方法重置服务器的一些配置。考虑到篇幅问题，关于服务器实例重置的代码就不分析了。

### 2.2.4 服务注册

服务注册操作对于 Dubbo 来说不是必需的，通过服务直连的方式就可以绕过注册中心。但通常我们不会这么做，直连方式不利于服务治理，仅推荐在测试服务时使用。对于 Dubbo 来说，注册中心虽不是必需，但却是必要的。

本节内容以 Zookeeper 注册中心作为分析目标，其他类型注册中心大家可自行分析。下面从服务注册的入口方法开始分析，我们把目光再次移到 RegistryProtocol 的 export 方法上

- export 方法
  - RegistryProtocol 的 export 方法包含了服务导出，注册，以及数据订阅等逻辑。其中服务导出逻辑上一节已经分析过了，本节将分析服务注册逻辑
  - register 方法包含两步操作，第一步是获取注册中心实例，第二步是向注册中心注册服务。接下来分两节内容对这两步操作进行分析

#### 2.2.4.1 创建注册中心

getRegistry 方法的源码，这个方法由 AbstractRegistryFactory 实现

getRegistry 方法先访问缓存，缓存未命中则调用 createRegistry 创建 Registry，然后写入缓存。这里的 createRegistry 是一个模板方法，由具体的子类实现

#### 2.2.4.2 节点创建

//todo



# 服务引用

在 Dubbo 中，我们可以通过两种方式引用远程服务。第一种是使用服务直连的方式引用服务，第二种方式是基于注册中心进行引用。服务直连的方式仅适合在调试或测试服务的场景下使用，不适合在线上环境使用。因此，本文我将重点分析通过注册中心引用服务的过程。从注册中心中获取服务配置只是服务引用过程中的一环，除此之外，服务消费者还需要经历 Invoker 创建、代理类创建等步骤

## 2.服务引用原理

Dubbo 服务引用的**时机有两个**，第一个是在 Spring 容器调用 ReferenceBean 的 afterPropertiesSet 方法时引用服务，第二个是在 ReferenceBean 对应的服务被注入到其他类中时引用。这两个引用服务的时机区别在于，第一个是饿汉式的，第二个是懒汉式的。默认情况下，Dubbo 使用懒汉式引用服务。如果需要使用饿汉式，可通过配置 <dubbo:reference> 的 init 属性开启。下面我们按照 Dubbo 默认配置进行分析，整个分析过程从 ReferenceBean 的 getObject 方法开始。当我们的服务被注入到其他类中时，Spring 会第一时间调用 getObject 方法，并由该方法执行服务引用逻辑。按照惯例，在进行具体工作之前，需先进行配置检查与收集工作。接着根据收集到的信息决定服务用的方式，有三种，第一种是引用本地 (JVM) 服务，第二是通过直连方式引用远程服务，第三是通过注册中心引用远程服务。不管是哪种引用方式，最后都会得到一个 Invoker 实例。如果有多个注册中心，多个服务提供者，这个时候会得到一组 Invoker 实例，此时需要通过集群管理类 Cluster 将多个 Invoker 合并成一个实例。合并后的 Invoker 实例已经具备调用本地或远程服务的能力了，但并不能将此实例暴露给用户使用，这会对用户业务代码造成侵入。此时框架还需要通过代理工厂类 (ProxyFactory) 为服务接口生成代理类，并让代理类去调用 Invoker 逻辑。避免了 Dubbo 框架代码对业务代码的侵入，同时也让框架更容易使用。

## 3.源码分析

- 服务引用的入口方法为 ReferenceBean 的 getObject 方法，该方法定义在 Spring 的 FactoryBean 接口中，ReferenceBean 实现了这个方法。

### 3.1 处理配置

- Dubbo 提供了丰富的配置，用于调整和优化框架行为，性能等。Dubbo 在引用或导出服务时，首先会对这些配置进行检查和处理，以保证配置的正确性。配置解析逻辑封装在 ReferenceConfig 的 init 方法中
  - 首先是方法开始到分割线1之间的代码。这段代码主要用于**检测 ConsumerConfig 实例是否存在**，如不存在则创建一个新的实例，然后通过系统变量或 dubbo.properties 配置文件填充 ConsumerConfig 的字段。接着是检测泛化配置，并根据配置设置 interfaceClass 的值。接着来看分割线1到分割线2之间的逻辑。这段逻辑用于**从系统属性或配置文件中加载与接口名相对应的配置，并将解析结果赋值给 url 字段**。url 字段的作用一般是用于点对点调用。继续向下看，分割线2和分割线3之间的代码用于检测几个核心配置类是否为空，为空则尝试从其他配置类中获取。分割线3与分割线4之间的代码主要用于**收集各种配置，并将配置存储到 map 中**。分割线4和分割线5之间的代码用于**处理 MethodConfig 实例**。该实例包含了事件通知配置，比如 onreturn、onthrow、oninvoke 等。分割线5到方法结尾的代码主要用于**解析服务消费者 ip，以及调用 createProxy 创建代理对象**。

### 3.2 引用服务

- 本节我们要从 createProxy 开始看起。从字面意思上来看，createProxy 似乎只是用于创建代理对象的。但实际上并非如此，该方法还会调用其他方法构建以及合并 Invoker 实例。
  - 首先**根据配置检查是否为本地调用**，若是，则调用 InjvmProtocol 的 refer 方法生成 InjvmInvoker 实例。若不是，则读取直连配置项，或注册中心 url，并将读取到的 url 存储到 urls 中。然后根据 urls 元素数量进行后续操作。若 urls 元素数量为1，则直接通过 Protocol 自适应拓展类构建 Invoker 实例接口。若 urls 元素数量大于1，即存在多个注册中心或服务直连 url，此时先根据 url 构建 Invoker。然后再通过 Cluster 合并多个 Invoker，最后调用 ProxyFactory 生成代理类。

#### 3.2.1 创建 Invoker

- Invoker 是 Dubbo 的核心模型，代表一个可执行体。在服务提供方，Invoker 用于调用服务提供类。在服务消费方，Invoker 用于执行远程调用。Invoker 是由 Protocol 实现类构建而来。Protocol 实现类有很多，本节会分析最常用的两个，分别是 RegistryProtocol 和 DubboProtocol，其他的大家自行分析。下面先来分析 DubboProtocol 的 refer 方法源码
  - 上面方法看起来比较简单，不过这里有一个调用需要我们注意一下，即 **getClients**。这个方法用于获取客户端实例，实例类型为 ExchangeClient。ExchangeClient 实际上并不具备通信能力，它需要基于更底层的客户端实例进行通信。比如 NettyClient、MinaClient 等，默认情况下，Dubbo 使用 NettyClient 进行通信。
-  getClients 方法
  - 这里根据 connections 数量**决定是获取共享客户端还是创建新的客户端实例**，默认情况下，使用共享客户端实例。getSharedClient 方法中也会调用 initClient 方法，因此下面我们一起看一下这两个方法。
  - **getSharedClient**(URL url)
    - 上面方法先访问缓存，若缓存未命中，则通过 initClient 方法创建新的 ExchangeClient 实例，并将该实例传给 ReferenceCountExchangeClient 构造方法创建一个带有引用计数功能的 ExchangeClient 实例。
  - **initClient**(URL url)
    - initClient 方法首先获取用户配置的客户端类型，默认为 netty。然后检测用户配置的客户端类型是否存在，不存在则抛出异常。最后根据 lazy 配置决定创建什么类型的客户端。这里的 LazyConnectExchangeClient 代码并不是很复杂，该类会在 request 方法被调用时通过 Exchangers 的 connect 方法创建 ExchangeClient 客户端



#### 3.2.2 创建代理

Invoker 创建完毕后，接下来要做的事情是为服务接口生成代理对象。有了代理对象，即可进行远程调用。代理对象生成的入口方法为 ProxyFactory 的 getProxy

- getProxy(Invoker, Class<?>[]) 这个方法是一个抽象方法，下面我们到 JavassistProxyFactory 类中看一下该方法的实现代码
  - 首先是通过 Proxy 的 getProxy 方法获取 Proxy 子类，然后创建 InvokerInvocationHandler 对象，并将该对象传给 newInstance 生成 Proxy 实例。InvokerInvocationHandler 实现 JDK 的 InvocationHandler 接口，具体的用途是拦截接口类调用。
- Proxy 的 getProxy 方法
  - 比如我们有一个 DemoService 接口，这个接口代理类就是由 ccp 生成的。ccm 则是用于为 org.apache.dubbo.common.bytecode.Proxy 抽象类生成子类，主要是实现 Proxy 类的抽象方法。



# 集群容错

本篇文章，将开始分析 Dubbo **集群容错**方面的源码。集群容错源码**包含四个部分，分别是服务目录 Directory、服务路由 Router、集群 Cluster 和负载均衡 LoadBalance**。这几个部分的源码逻辑相对比较独立，我们将会分四篇文章进行分析。本篇文章作为集群容错的开篇文章，将和大家一起分析服务目录相关的源码。在进行深入分析之前，我们先来了解一下**服务目录**是什么。服务目录中存储了一些和服务提供者有关的信息，通过服务目录，服务消费者可获取到服务提供者的信息，比如 ip、端口、服务协议等。通过这些信息，服务消费者就可通过 Netty 等客户端进行远程调用。在一个服务集群中，服务提供者数量并不是一成不变的，如果集群中新增了一台机器，相应地在服务目录中就要新增一条服务提供者记录。或者，如果服务提供者的配置修改了，服务目录中的记录也要做相应的更新。如果这样说，服务目录和注册中心的功能不就雷同了吗？确实如此，这里这么说是为了方便大家理解。**实际上服务目录在获取注册中心的服务配置信息后，会为每条配置信息生成一个 Invoker 对象，并把这个 Invoker 对象存储起来，这个 Invoker 才是服务目录最终持有的对象。**Invoker 有什么用呢？看名字就知道了，这是一个具有远程调用功能的对象。讲到这大家应该知道了什么是服务目录了，它可以看做是 **Invoker 集合，且这个集合中的元素会随注册中心的变化而进行动态调整。**

## 2. 继承体系

服务目录目前内置的实现有两个，分别为 StaticDirectory 和 RegistryDirectory，它们均是 AbstractDirectory 的子类。AbstractDirectory 实现了 Directory 接口，这个接口包含了一个重要的方法定义，即 list(Invocation)，用于列举 Invoker。

## 3. 源码分析

本章将分析 AbstractDirectory 和它两个子类的源码。AbstractDirectory 封装了 Invoker 列举流程，具体的列举逻辑则由子类实现，这是典型的模板模式。接下来我们先来看一下 **AbstractDirectory 的源码**。

- list 方法源码，这个方法封装了 Invoker 的列举过程。如下：

  1. 调用 doList 获取 Invoker 列表
  2. 根据 Router 的 getUrl 返回值为空与否，以及 runtime 参数决定是否进行服务路由

  以上步骤中，doList 是模板方法，需由子类实现。Router 的 runtime 参数这里简单说明一下，这个参数决定了是否在每次调用服务时都执行路由规则。如果 runtime 为 true，那么每次调用服务前，都需要进行服务路由。这个对性能造成影响，配置时需要注意。

关于 AbstractDirectory 就分析这么多，下面开始分析子类的源码。

### 3.1 StaticDirectory

StaticDirectory 即静态服务目录，顾名思义，它内部存放的 Invoker 是不会变动的。所以，理论上它和不可变 List 的功能很相似。

### 3.2 RegistryDirectory

RegistryDirectory 是一种动态服务目录，实现了 NotifyListener 接口。当注册中心服务配置发生变化后，RegistryDirectory 可收到与当前服务相关的变化。收到变更通知后，RegistryDirectory 可根据配置变更信息刷新 Invoker 列表。**RegistryDirectory 中有几个比较重要的逻辑**，第一是 Invoker 的列举逻辑，第二是接收服务配置变更的逻辑，第三是 Invoker 列表的刷新逻辑。



# 服务路由

**服务目录在刷新 Invoker 列表的过程中，会通过 Router 进行服务路由，筛选出符合路由规则的服务提供者。**在详细分析服务路由的源码之前，先来介绍一下服务路由是什么。服务路由包含一条路由规则，路由规则决定了服务消费者的调用目标，即规定了服务消费者可调用哪些服务提供者。Dubbo 目前提供了三种服务路由实现，分别为条件路由 ConditionRouter、脚本路由 ScriptRouter 和标签路由 TagRouter。其中条件路由是我们最常使用的，标签路由是一个新的实现，暂时还未发布，该实现预计会在 2.7.x 版本中发布。



## 2. 源码分析

条件路由规则由两个条件组成，分别用于对服务消费者和提供者进行匹配。比如有这样一条规则：

```
host = 10.20.153.10 => host = 10.20.153.11
```

该条规则表示 IP 为 10.20.153.10 的服务消费者**只可**调用 IP 为 10.20.153.11 机器上的服务，不可调用其他机器上的服务。条件路由规则的格式如下：

```
[服务消费者匹配条件] => [服务提供者匹配条件]
```

如果服务消费者匹配条件为空，表示不对服务消费者进行限制。如果服务提供者匹配条件为空，表示对某些服务消费者禁用服务。官方文档中对条件路由进行了比较详细的介绍，大家可以参考下，这里就不过多说明了。

条件路由实现类 ConditionRouter 在进行工作前，需要先对用户配置的路由规则进行解析，得到一系列的条件。然后再根据这些条件对服务进行路由。本章将分两节进行说明，2.1节介绍表达式解析过程。2.2 节介绍服务路由的过程。下面，我们先从表达式解析过程看起。

### 2.1 表达式解析

条件路由规则是一条字符串，对于 Dubbo 来说，它并不能直接理解字符串的意思，需要将其解析成内部格式才行。条件表达式的解析过程始于 ConditionRouter 的构造方法

### 2.2 服务路由

服务路由的入口方法是 ConditionRouter 的 route 方法，该方法定义在 Router 接口中

- route 方法先是调用 matchWhen 对服务消费者进行匹配，如果匹配失败，直接返回 Invoker 列表。如果匹配成功，再对服务提供者进行匹配，匹配逻辑封装在了 matchThen 方法中



# 集群

## 1. 简介

为了避免单点故障，现在的应用通常至少会部署在两台服务器上。对于一些负载比较高的服务，会部署更多的服务器。这样，在同一环境下的服务提供者数量会大于1。对于服务消费者来说，同一环境下出现了多个服务提供者。**这时会出现一个问题**，服务消费者需要决定选择哪个服务提供者进行调用。另外服务调用失败时的处理措施也是需要考虑的，是重试呢，还是抛出异常，亦或是只打印异常等。为了处理这些问题，Dubbo 定义了集群接口 Cluster 以及 Cluster Invoker。集群 Cluster 用途是将多个服务提供者合并为一个 Cluster Invoker，并将这个 Invoker 暴露给服务消费者。这样一来，服务消费者只需通过这个 Invoker 进行远程调用即可，至于具体调用哪个服务提供者，以及调用失败后如何处理等问题，现在都交给集群模块去处理。集群模块是服务提供者和服务消费者的中间层，为服务消费者屏蔽了服务提供者的情况，这样服务消费者就可以专心处理远程调用相关事宜。比如发请求，接受服务提供者返回的数据等。这就是集群的作用。

集群容错对于 Dubbo 框架来说，是很重要的逻辑。集群模块处于服务提供者和消费者之间，对于服务消费者来说，集群可向其屏蔽服务提供者集群的情况，使其能够专心进行远程调用。除此之外，通过集群模块，我们还可以对服务之间的调用链路进行编排优化，治理服务。总的来说，对于 Dubbo 而言，集群容错相关逻辑是非常重要的。想要对 Dubbo 有比较深的理解，集群容错是必须要掌握的。

Dubbo 提供了多种集群实现，包含但不限于 Failover Cluster、Failfast Cluster 和 Failsafe Cluster 等。

## 2. 集群容错

在对集群相关代码进行分析之前，这里有必要先来介绍一下集群容错的所有组件。包含 Cluster、Cluster Invoker、Directory、Router 和 LoadBalance 等。

![img](http://dubbo.apache.org/docs/zh-cn/source_code_guide/sources/images/cluster.jpg)

集群工作过程可分为两个阶段，第一个阶段是在服务消费者初始化期间，集群 Cluster 实现类为服务消费者创建 Cluster Invoker 实例，即上图中的 merge 操作。第二个阶段是在服务消费者进行远程调用时。以 FailoverClusterInvoker 为例，该类型 Cluster Invoker 首先会调用 Directory 的 list 方法列举 Invoker 列表（可将 Invoker 简单理解为服务提供者）。Directory 的用途是保存 Invoker，可简单类比为 List<Invoker>。其实现类 RegistryDirectory 是一个动态服务目录，可感知注册中心配置的变化，它所持有的 Invoker 列表会随着注册中心内容的变化而变化。每次变化后，RegistryDirectory 会动态增删 Invoker，并调用 Router 的 route 方法进行路由，过滤掉不符合路由规则的 Invoker。当 FailoverClusterInvoker 拿到 Directory 返回的 Invoker 列表后，它会通过 LoadBalance 从 Invoker 列表中选择一个 Invoker。最后 FailoverClusterInvoker 会将参数传给 LoadBalance 选择出的 Invoker 实例的 invoke 方法，进行真正的远程调用。

以上就是集群工作的整个流程，这里并没介绍集群是如何容错的。Dubbo 主要提供了这样几种容错方式：

- Failover Cluster - 失败自动切换
- Failfast Cluster - 快速失败
- Failsafe Cluster - 失败安全
- Failback Cluster - 失败自动恢复
- Forking Cluster - 并行调用多个服务提供者

## 3.源码分析

### 3.1 Cluster 实现类分析

我们在上一章看到了两个概念，分别是集群接口 Cluster 和 Cluster Invoker，这两者是不同的。Cluster 是接口，而 Cluster Invoker 是一种 Invoker。服务提供者的选择逻辑，以及远程调用失败后的的处理逻辑均是封装在 Cluster Invoker 中。那么 Cluster 接口和相关实现类有什么用呢？用途比较简单，仅用于生成 Cluster Invoker。

- 如上，FailoverCluster 总共就包含这几行代码，用于创建 FailoverClusterInvoker 对象，很简单。
- 如上，FailbackCluster 的逻辑也是很简单，无需解释了。所以接下来，我们把重点放在各种 Cluster Invoker 上

### 3.2 Cluster Invoker 分析

我们首先从各种 Cluster Invoker 的父类 AbstractClusterInvoker 源码开始说起。前面说过，**集群工作过程可分为两个阶段**，第一个阶段是在服务消费者初始化期间，这个在[服务引用](http://dubbo.apache.org/zh-cn/docs/source_code_guide/cluster.html)那篇文章中分析过，就不赘述。第二个阶段是在服务消费者进行远程调用时，此时 AbstractClusterInvoker 的 invoke 方法会被调用。列举 Invoker，负载均衡等操作均会在此阶段被执行。因此下面先来看一下 invoke 方法的逻辑。

- AbstractClusterInvoker 的 invoke 方法主要用于列举 Invoker，以及加载 LoadBalance。最后再调用模板方法 doInvoke 进行后续操作。
- 下面我们来看一下 Invoker 列举方法 list(Invocation) 的逻辑
  - 如上，AbstractClusterInvoker 中的 list 方法做的事情很简单，只是简单的调用了 Directory 的 list 方法，没有其他更多的逻辑了。Directory 即相关实现类在前文已经分析过，这里就不多说了。
- 接下来，我们把目光转移到 AbstractClusterInvoker 的各种实现类上，来看一下这些实现类是如何实现 doInvoke 方法逻辑的。

#### 3.2.1 FailoverClusterInvoker

FailoverClusterInvoker 在调用失败时，会自动切换 Invoker 进行重试。默认配置下，Dubbo 会使用这个类作为缺省 Cluster Invoker。

- 如上，FailoverClusterInvoker 的 doInvoke 方法首先是获取重试次数，然后根据重试次数进行循环调用，失败后进行重试。在 for 循环内，首先是通过负载均衡组件选择一个 Invoker，然后再通过这个 Invoker 的 invoke 方法进行远程调用。如果失败了，记录下异常，并进行重试。重试时会再次调用父类的 list 方法列举 Invoker。整个流程大致如此，不是很难理解。
- 下面我们看一下 select 方法的逻辑
  - 如上，select 方法的主要逻辑集中在了对粘滞连接特性的支持上。首先是获取 sticky 配置，然后再检测 invokers 列表中是否包含 stickyInvoker，如果不包含，则认为该 stickyInvoker 不可用，此时将其置空。这里的 invokers 列表可以看做是**存活着的服务提供者**列表，如果这个列表不包含 stickyInvoker，那自然而然的认为 stickyInvoker 挂了，所以置空。如果 stickyInvoker 存在于 invokers 列表中，此时要进行下一项检测 — 检测 selected 中是否包含 stickyInvoker。如果包含的话，说明 stickyInvoker 在此之前没有成功提供服务（但其仍然处于存活状态）。此时我们认为这个服务不可靠，不应该在重试期间内再次被调用，因此这个时候不会返回该 stickyInvoker。如果 selected 不包含 stickyInvoker，此时还需要进行可用性检测，比如检测服务提供者网络连通性等。当可用性检测通过，才可返回 stickyInvoker，否则调用 doSelect 方法选择 Invoker。如果 sticky 为 true，此时会将 doSelect 方法选出的 Invoker 赋值给 stickyInvoker。

- 以上就是 select 方法的逻辑，这段逻辑看起来不是很复杂，但是信息量比较大。不搞懂 invokers 和 selected 两个入参的含义，以及粘滞连接特性，这段代码是不容易看懂的。
- **doSelect** 主要做了两件事，第一是通过负载均衡组件选择 Invoker。第二是，如果选出来的 Invoker 不稳定，或不可用，此时需要调用 reselect 方法进行重选。若 reselect 选出来的 Invoker 为空，此时定位 invoker 在 invokers 列表中的位置 index，然后获取 index + 1 处的 invoker，这也可以看做是重选逻辑的一部分
- reselect 方法
  - reselect 方法总结下来其实只做了两件事情，第一是查找可用的 Invoker，并将其添加到 reselectInvokers 集合中。第二，如果 reselectInvokers 不为空，则通过负载均衡组件再次进行选择。其中第一件事情又可进行细分，一开始，reselect 从 invokers 列表中查找有效可用的 Invoker，若未能找到，此时再到 selected 列表中继续查找。

#### 3.2.2 FailbackClusterInvoker

FailbackClusterInvoker 会在调用失败后，返回一个空结果给服务消费者。并通过定时任务对失败的调用进行重传，适合执行消息通知等操作。下面来看一下它的实现逻辑。

这个类主要由3个方法组成，首先是 doInvoker，该方法负责初次的远程调用。若远程调用失败，则通过 addFailed 方法将调用信息存入到 failed 中，等待定时重试。addFailed 在开始阶段会根据 retryFuture 为空与否，来决定是否开启定时任务。retryFailed 方法则是包含了失败重试的逻辑，该方法会对 failed 进行遍历，然后依次对 Invoker 进行调用。调用成功则将 Invoker 从 failed 中移除，调用失败则忽略失败原因。

#### 3.2.3 FailfastClusterInvoker

FailfastClusterInvoker 只会进行一次调用，失败后立即抛出异常。适用于幂等操作，比如新增记录。

首先是通过 select 方法选择 Invoker，然后进行远程调用。如果调用失败，则立即抛出异常。FailfastClusterInvoker 就先分析到这

#### 3.2.4 FailsafeClusterInvoker

FailsafeClusterInvoker 是一种失败安全的 Cluster Invoker。所谓的失败安全是指，当调用过程中出现异常时，FailsafeClusterInvoker 仅会打印异常，而不会抛出异常。适用于写入审计日志等操作。

FailsafeClusterInvoker 的逻辑和 FailfastClusterInvoker 的逻辑一样简单，无需过多说明。

#### 3.2.5 ForkingClusterInvoker

ForkingClusterInvoker 会在运行时通过线程池创建多个线程，并发调用多个服务提供者。只要有一个服务提供者成功返回了结果，doInvoke 方法就会立即结束运行。ForkingClusterInvoker 的应用场景是在一些对实时性要求比较高**读操作**（注意是读操作，并行写操作可能不安全）下使用，但这将会耗费更多的资源。

ForkingClusterInvoker 的 doInvoker 方法比较长，这里通过两个分割线将整个方法划分为三个逻辑块。从方法开始到分割线1之间的代码主要是用于选出 forks 个 Invoker，为接下来的并发调用提供输入。分割线1和分割线2之间的逻辑通过线程池并发调用多个 Invoker，并将结果存储在阻塞队列中。分割线2到方法结尾之间的逻辑主要用于从阻塞队列中获取返回结果，并对返回结果类型进行判断。如果为异常类型，则直接抛出，否则返回。

以上就是ForkingClusterInvoker 的 doInvoker 方法大致过程。我们在分割线1和分割线2之间的代码上留了一个问题，问题是这样的：为什么要在`value >= selected.size()`的情况下，才将异常对象添加到阻塞队列中？这里来解答一下。原因是这样的，在并行调用多个服务提供者的情况下，只要有一个服务提供者能够成功返回结果，而其他全部失败。此时 ForkingClusterInvoker 仍应该返回成功的结果，而非抛出异常。在`value >= selected.size()`时将异常对象放入阻塞队列中，可以保证异常对象不会出现在正常结果的前面，这样可从阻塞队列中优先取出正常的结果。

#### 3.2.6 BroadcastClusterInvoker

本章的最后，我们再来看一下 BroadcastClusterInvoker。BroadcastClusterInvoker 会逐个调用每个服务提供者，如果其中一台报错，在循环调用结束后，BroadcastClusterInvoker 会抛出异常。该类通常用于通知所有提供者更新缓存或日志等本地资源信息。



## 负载均衡

- LoadBalance 中文意思为负载均衡，它的职责是将网络请求，或者其他形式的负载“均摊”到不同的机器上。避免集群中部分服务器压力过大，而另一些服务器比较空闲的情况。
- RandomLoadBalance
  - RandomLoadBalance 是加权随机算法的具体实现，它的算法思想很简单。假设我们有一组服务器 servers = [A, B, C]，他们对应的权重为 weights = [5, 3, 2]，权重总和为10。现在把这些权重值平铺在一维坐标值上，[0, 5) 区间属于服务器 A，[5, 8) 区间属于服务器 B，[8, 10) 区间属于服务器 C。接下来通过随机数生成器生成一个范围在 [0, 10) 之间的随机数，然后计算这个随机数会落到哪个区间上。
  - 当调用次数比较少时，Random 产生的随机数可能会比较集中，此时多数请求会落到同一台服务器上。这个缺点并不是很严重，多数情况下可以忽略。RandomLoadBalance 是一个简单，高效的负载均衡实现，因此 Dubbo 选择它作为缺省实现。
- LeastActiveLoadBalance
  - 除了最小活跃数，LeastActiveLoadBalance 在实现上还引入了权重值。所以准确的来说，LeastActiveLoadBalance 是基于加权最小活跃数算法实现的。举个例子说明一下，在一个服务提供者集群中，有两个性能优异的服务提供者。某一时刻它们的活跃数相同，此时 Dubbo 会根据它们的权重去分配请求，权重越大，获取到新请求的概率就越大。如果两个服务提供者权重相同，此时随机选择一个即可。
- 2.6.4 版本中的 LeastActiveLoadBalance 的缺陷
  - 问题出在服务预热阶段，第一行代码直接从 url 中取权重值，未被降权过。第二行代码获取到的是经过降权后的权重。第一行代码获取到的权重值最终会被累加到权重总和 totalWeight 中，这个时候会导致一个问题。offsetWeight 是一个在 [0, totalWeight) 范围内的随机数，而它所减去的是经过降权的权重。很有可能在经过 leastCount 次运算后，offsetWeight 仍然是大于0的，导致无法选中 Invoker。
    - 修复后：*强调该变量经过了 warmup 降权处理*
  - 当一组 Invoker 具有相同的最小活跃数，且其中一个 Invoker 的权重值为1，此时这个 Invoker 无法被选中。
    - 问题出在了`offsetWeight <= 0`上，举例说明，假设有一组 Invoker  的权重为 5、2、1，offsetWeight 最大值为 7。假设 offsetWeight = 7，你会发现，当 for 循环进行第二次遍历后 offsetWeight = 7 - 5 - 2 = 0，提前返回了。此时，此时权重为1的 Invoker 就没有机会被选中了。
    - 修复后：int offsetWeight = random.nextInt(totalWeight) + 1;
- ConsistentHashLoadBalance
  - 它的工作过程是这样的，首先根据 ip 或者其他的信息为缓存节点生成一个 hash，并将这个 hash 投射到 [0, 232 - 1] 的圆环上。当有查询或写入请求时，则为缓存项的 key 生成一个 hash 值。然后查找第一个大于或等于该 hash 值的缓存节点，并到这个节点中查询或写入缓存项。如果当前节点挂了，则在下一次查询或写入缓存时，为缓存项查找另一个大于其 hash 值的缓存节点即可。
  - 引入虚拟节点，避免数据倾斜问题。所谓数据倾斜是指，由于节点不够分散，导致大量请求落到了同一个节点上，而其他节点只会接收到了少量请求的情况
- RoundRobinLoadBalance
  - 轮询是一种无状态负载均衡算法，实现简单，适用于每台服务器性能相近的场景下。但现实情况下，我们并不能保证每台服务器性能均相近。如果我们将等量的请求分配给性能较差的服务器，这显然是不合理的。
  - 因此，这个时候我们需要对轮询过程进行加权，以调控每台服务器的负载。经过加权后，每台服务器能够得到的请求数比例，接近或等于他们的权重比。比如服务器 A、B、C 权重比为 5:2:1。那么在8次请求中，服务器 A 将收到其中的5次请求，服务器 B 会收到其中的2次请求，服务器 C 则收到其中的1次请求。
  - 具体实现
    - *使用调用编号对权重总和进行取余操作*
      - *遍历 invokerToWeightMap*
      - *如果 mod = 0，且权重大于0，此时返回相应的 Invoker*
      - *mod != 0，且权重大于0，此时对权重和 mod 分别进行自减操作*
  - 2.6.4 版本的 缺陷
    - RoundRobinLoadBalance 需要在`mod == 0 && v.getValue() > 0` 条件成立的情况下才会被返回相应的 Invoker。假如 mod 很大，比如 10000，50000，甚至更大时，doSelect 方法需要进行很多次计算才能将 mod 减为0。由此可知，doSelect 的效率与 mod 有关，时间复杂度为 O(mod)。mod 又受最大权重 maxWeight 的影响，因此当某个服务提供者配置了非常大的权重，此时 RoundRobinLoadBalance 会产生比较严重的性能问题。
    - 修复后：每进行一轮循环，重新计算 currentWeight。如果当前 Invoker 权重大于 currentWeight，则返回该 Invoker
    - 修复后的缺陷：新的 RoundRobinLoadBalance 在某些情况下选出的服务器序列不够均匀。比如，服务器 [A, B, C] 对应权重 [5, 1, 1]。进行7次负载均衡后，选择出来的序列为 [A, A, A, A, A, B, C]。前5个请求全部都落在了服务器 A上，这将会使服务器 A 短时间内接收大量的请求，压力陡增。
  - 了增加负载均衡结果的平滑性，社区再次对 RoundRobinLoadBalance 的实现进行了重构，这次重构参考自 Nginx 的平滑加权轮询负载均衡
    - 每个服务器对应两个权重，分别为 weight 和 currentWeight。其中 weight 是固定的，currentWeight 会动态调整，初始值为0。当有新的请求进来时，遍历服务器列表，让它的 currentWeight 加上自身权重。遍历完成后，找到最大的 currentWeight，并将其减去权重总和，然后返回相应的服务器即可。

| 请求编号 | currentWeight 数组 | 选择结果 | 减去权重总和后的 currentWeight 数组 |
| -------- | ------------------ | -------- | :---------------------------------: |
| 1        | [5, 1, 1]          | A        |             [-2, 1, 1]              |
| 2        | [3, 2, 2]          | A        |             [-4, 2, 2]              |
| 3        | [1, 3, 3]          | B        |             [1, -4, 3]              |
| 4        | [6, -3, 4]         | A        |             [-1, -3, 4]             |
| 5        | [4, -2, 5]         | C        |             [4, -2, -2]             |
| 6        | [9, -1, -1]        | A        |             [2, -1, -1]             |
| 7        | [7, 0, 0]          | A        |              [0, 0, 0]              |

​	



## 服务调用过程

- Dubbo 服务调用过程比较复杂，包含众多步骤，比如发送请求、编解码、服务降级、过滤器链处理、序列化、线程派发以及响应请求等步骤
  - 本篇文章将会**重点分析请求的发送与接收、编解码、线程派发以及响应的发送与接收等过程**，至于服务降级、过滤器链和序列化大家自行进行分析，也可以将其当成一个黑盒，暂时忽略也没关系。
  
  ## 2. 源码分析
  
  在进行源码分析之前，我们先来通过一张图了解 Dubbo 服务调用过程。
  
  ![img](http://dubbo.apache.org/docs/zh-cn/source_code_guide/sources/images/send-request-process.jpg)
  
  首先服务消费者通过代理对象 Proxy 发起远程调用，接着通过网络客户端 Client 将编码后的请求发送给服务提供方的网络层上，也就是 Server。Server 在收到请求后，首先要做的事情是对数据包进行解码。然后将解码后的请求发送至分发器 Dispatcher，再由分发器将请求派发到指定的线程池上，最后由线程池调用具体的服务。这就是一个远程调用请求的发送与接收过程。至于响应的发送与接收过程，这张图中没有表现出来。对于这两个过程，我们也会进行详细分析。
  
  

### 2.1 服务调用方式

Dubbo 支持同步和异步两种调用方式，其中异步调用还可细分为“有返回值”的异步调用和“无返回值”的异步调用。所谓“无返回值”异步调用是指服务消费方只管调用，但不关心调用结果，此时 Dubbo 会直接返回一个空的 RpcResult。若要使用异步特性，需要服务消费方手动进行配置。默认情况下，Dubbo 使用同步调用方式。

Dubbo 默认使用 Javassist 框架为服务接口生成动态代理类，因此我们需要先将代理类进行反编译才能看到源码。这里使用阿里开源 Java 应用诊断工具 [Arthas](https://github.com/alibaba/arthas) 反编译代理类

```java
/**
 * Arthas 反编译步骤：
 * 1. 启动 Arthas
 *    java -jar arthas-boot.jar
 *
 * 2. 输入编号选择进程
 *    Arthas 启动后，会打印 Java 应用进程列表，如下：
 *    [1]: 11232 org.jetbrains.jps.cmdline.Launcher
 *    [2]: 22370 org.jetbrains.jps.cmdline.Launcher
 *    [3]: 22371 com.alibaba.dubbo.demo.consumer.Consumer
 *    [4]: 22362 com.alibaba.dubbo.demo.provider.Provider
 *    [5]: 2074 org.apache.zookeeper.server.quorum.QuorumPeerMain
 * 这里输入编号 3，让 Arthas 关联到启动类为 com.....Consumer 的 Java 进程上
 *
 * 3. 由于 Demo 项目中只有一个服务接口，因此此接口的代理类类名为 proxy0，此时使用 sc 命令搜索这个类名。
 *    $ sc *.proxy0
 *    com.alibaba.dubbo.common.bytecode.proxy0
 *
 * 4. 使用 jad 命令反编译 com.alibaba.dubbo.common.bytecode.proxy0
 *    $ jad com.alibaba.dubbo.common.bytecode.proxy0
 *
```



- 代理类的逻辑比较简单。首先将运行时参数存储到数组中，然后调用 InvocationHandler 接口实现类的 invoke 方法，得到调用结果，最后将结果转型并返回给调用方

  - InvokerInvocationHandler 中的 invoker 成员变量类型为 MockClusterInvoker，MockClusterInvoker 内部封装了服务降级逻辑
  - 服务降级不是本文重点，因此这里就不分析 doMockInvoke 方法了。考虑到前文已经详细分析过 FailoverClusterInvoker，因此本节略过 FailoverClusterInvoker，**直接分析 DubboInvoker。**

- DubboInvoker

  - 代码包含了 Dubbo 对同步和异步调用的处理逻辑
  - Dubbo 实现同步和异步调用比较关键的一点就在于由谁调用 ResponseFuture 的 get 方法。同步调用模式下，由框架自身调用 ResponseFuture 的 get 方法。异步调用模式下，则由用户调用该方法。

- ResponseFuture 是一个接口，下面我们来看一下它的默认实现类 DefaultFuture 的源码

  - 当服务消费者还未接收到调用结果时，用户线程调用 get 方法会被阻塞住。
  - 同步调用模式下，框架获得 DefaultFuture 对象后，会立即调用 get 方法进行等待。
  - 而异步模式下则是将该对象封装到 FutureAdapter 实例中，并将 FutureAdapter 实例设置到 RpcContext 中，供用户使用。
  - FutureAdapter 是一个适配器，用于将 Dubbo 中的 ResponseFuture 与 JDK 中的 Future 进行适配。这样当用户线程调用 Future 的 get 方法时，经过 FutureAdapter 适配，最终会调用 ResponseFuture 实现类对象的 get 方法，也就是 DefaultFuture 的 get 方法

  

  

### 2.2 服务消费方发送请求

#### 2.2.1 发送请求

服务消费方发送请求过程的部分调用栈，略为复杂。经过多次调用后，才将请求数据送至 Netty NioClientSocketChannel。这样做的原因是通过 Exchange 层为框架引入 Request 和 Response 语义

- ReferenceCountExchangeClient 
  - ReferenceCountExchangeClient 内部定义了一个引用计数变量 referenceCount，每当该对象被引用一次 referenceCount 都会进行自增。每当 close 方法被调用时，referenceCount 进行自减。ReferenceCountExchangeClient 内部仅实现了一个引用计数的功能，其他方法并无复杂逻辑，均是直接调用被装饰对象的相关方法。
- HeaderExchangeClient
  -  HeaderExchangeClient 封装了一些关于心跳检测的逻辑
- HeaderExchangeChannel 
  - 首先定义了一个 Request 对象，然后再将该对象传给 NettyClient 的 send 方法，进行后续的调用。需要说明的是，NettyClient 中并未实现 send 方法，该方法继承自父类 AbstractPeer
- AbstractPeer 
  - 默认情况下，Dubbo 使用 Netty 作为底层的通信框架
- NettyClient 类中看一下 getChannel 方法的实现逻辑
  - 获取到 NettyChannel 实例后，即可进行后续的调用。
  - 下面看一下 NettyChannel 的 send 方法。
    - 经历多次调用，到这里请求数据的发送过程就结束了
- 在 Netty 中，出站数据在发出之前还需要进行编码操作

#### 2.2.2 请求编码

在分析请求编码逻辑之前，我们先来看一下 Dubbo 数据包结构。

![img](http://dubbo.apache.org/docs/zh-cn/source_code_guide/sources/images/data-format.jpg)

Dubbo 数据包分为消息头和消息体，消息头用于存储一些元信息，比如魔数（Magic），数据包类型（Request/Response），消息体长度（Data Length）等。消息体中用于存储具体的调用消息，比如方法名称，参数列表等。

- 编码逻辑所在类(**ExchangeCodec** )
  - 该过程首先会通过位运算将消息头写入到 header 数组中。
  - 然后对 Request 对象的 data 字段执行序列化操作，序列化后的数据最终会存储到 ChannelBuffer 中。
  - 序列化操作执行完后，可得到数据序列化后的长度 len，紧接着将 len 写入到 header 指定位置处。
  - 最后再将消息头字节数组 header 写入到 ChannelBuffer 中，整个编码过程就结束了。

### 2.3 服务提供方接收请求

前面说过，默认情况下 Dubbo 使用 Netty 作为底层的通信框架。Netty 检测到有数据入站后，首先会通过解码器对数据进行解码，并将解码后的数据传递给下一个入站处理器的指定方法。所以在进行后续的分析之前，我们先来看一下数据解码过程。

#### 2.3.1 请求解码

这里直接分析请求数据的解码逻辑(**ExchangeCodec** )

- 上面方法通过检测消息头中的魔数是否与规定的魔数相等，提前拦截掉非常规数据包，比如通过 telnet 命令行发出的数据包。
- 接着再对消息体长度，以及可读字节数进行检测。
- 最后调用 decodeBody 方法进行后续的解码工作，ExchangeCodec 中实现了 decodeBody 方法，但因其子类 DubboCodec 覆写了该方法，所以在运行时 DubboCodec 中的 decodeBody 方法会被调用
  - decodeBody 对部分字段进行了解码，并将解码得到的字段封装到 Request 中。
  - 随后会调用 DecodeableRpcInvocation 的 decode 方法进行后续的解码工作
  - 此工作完成后，可将调用方法名、attachment、以及调用参数解析出来

上面的方法通过反序列化将诸如 path、version、调用方法名、参数列表等信息依次解析出来，并设置到相应的字段中，最终得到一个具有完整调用信息的 DecodeableRpcInvocation 对象。



#### 2.3.2 调用服务

解码器将数据包解析成 Request 对象后，NettyHandler 的 messageReceived 方法紧接着会收到这个对象，并将这个对象继续向下传递。这期间该对象会被依次传递给 NettyServer、MultiMessageHandler、HeartbeatHandler 以及 AllChannelHandler。最后由 AllChannelHandler 将该对象封装到 Runnable 实现类对象中，并将 Runnable 放入线程池中执行后续的调用逻辑

- NettyHandler 中的 messageReceived 逻辑比较简单。首先根据一些信息获取 NettyChannel 实例，然后将 NettyChannel 实例以及 Request 对象向下传递。

下面再来看看 AllChannelHandler 的逻辑，在详细分析代码之前，我们先来了解一下 Dubbo 中的线程派发模型

##### 2.3.2.1 线程派发模型

Dubbo 将底层通信框架中接收请求的线程称为 IO 线程。如果一些事件处理逻辑可以很快执行完，比如只在内存打一个标记，此时直接在 IO 线程上执行该段逻辑即可。但如果事件的处理逻辑比较耗时，比如该段逻辑会发起数据库查询或者 HTTP 请求。此时我们就不应该让事件处理逻辑在 IO 线程上执行，而是应该派发到线程池中去执行。原因也很简单，IO 线程主要用于接收请求，如果 IO 线程被占满，将导致它不能接收新的请求。

![img](http://dubbo.apache.org/docs/zh-cn/source_code_guide/sources/images/dispatcher-location.jpg)



如上图，红框中的 Dispatcher 就是线程派发器。需要说明的是，Dispatcher 真实的职责创建具有线程派发能力的 ChannelHandler，比如 AllChannelHandler、MessageOnlyChannelHandler 和 ExecutionChannelHandler 等，其本身并不具备线程派发能力。Dubbo 支持 5 种不同的线程派发策略，下面通过一个表格列举一下。

| 策略       | 用途                                                         |
| ---------- | ------------------------------------------------------------ |
| all        | 所有消息都派发到线程池，包括请求，响应，连接事件，断开事件等 |
| direct     | 所有消息都不派发到线程池，全部在 IO 线程上直接执行           |
| message    | 只有**请求**和**响应**消息派发到线程池，其它消息均在 IO 线程上执行 |
| execution  | 只有**请求**消息派发到线程池，不含响应。其它消息均在 IO 线程上执行 |
| connection | 在 IO 线程上，将连接断开事件放入队列，有序逐个执行，其它消息派发到线程池 |

默认配置下，Dubbo 使用 `all` 派发策略，即将所有的消息都派发到线程池中。



##### 2.3.2.2 调用服务

本小节，我们从 ChannelEventRunnable 开始分析

请求和响应消息出现频率明显比其他类型消息高，所以这里对该类型的消息进行了针对性判断。**ChannelEventRunnable** 仅是一个中转站，它的 run 方法中并不包含具体的调用逻辑，仅用于将参数传给其他 ChannelHandler 对象进行处理，该对象类型为 **DecodeHandler**

DecodeHandler 主要是包含了一些解码逻辑。2.2.1 节分析请求解码时说过，请求解码可在 IO 线程上执行，也可在线程池中执行，这个取决于运行时配置。DecodeHandler 存在的意义就是保证请求或响应对象可在线程池中被解码。解码完毕后，完全解码后的 Request 对象会继续向后传递，下一站是 **HeaderExchangeHandler**。

- **HeaderExchangeHandler** 

  - 到这里，我们看到了比较清晰的请求和响应逻辑。

  - 对于双向通信，HeaderExchangeHandler 首先向后进行调用，得到调用结果。

  - 然后将调用结果封装到 Response 对象中，最后再将该对象返回给服务消费方。

  - 如果请求不合法，或者调用失败，则将错误信息封装到 Response 对象中，并返回给服务消费方。

    

到这里，整个服务调用过程就分析完了

### 2.4 服务提供方返回调用结果

服务提供方调用指定服务后，会将调用结果封装到 Response 对象中，并将该对象返回给服务消费方。服务提供方也是通过 NettyChannel 的 send 方法将 Response 对象返回，这个方法在 2.2.1 节分析过，这里就不在重复分析了。本节我们仅需关注 Response 对象的编码过程即可，这里仍然省略一些中间调用，直接分析具体的编码逻辑。

和前面分析的 Request 对象编码过程很相似。如果大家能看 Request 对象的编码逻辑，那么这里的 Response 对象的编码逻辑也不难理解



### 2.5 服务消费方接收调用结果

服务消费方在收到响应数据后，首先要做的事情是对响应数据进行解码，得到 Response 对象。然后再将该对象传递给下一个入站处理器，这个入站处理器就是 NettyHandler。接下来 NettyHandler 会将这个对象继续向下传递，最后 AllChannelHandler 的 received 方法会收到这个对象，并将这个对象派发到线程池中。

这个过程和服务提供方接收请求的过程是一样的，因此这里就不重复分析了。本节我们重点分析两个方面的内容，一是响应数据的解码过程，二是 Dubbo 如何将调用结果传递给用户线程的

#### 2.5.1 响应数据解码

响应数据解码逻辑主要的逻辑封装在 DubboCodec 中

- DubboCodec 的 decodeBody 方法中关于请求数据的解码过程，该过程和响应数据的解码过程很相似

#### 2.5.2 向用户线程传递调用结果

响应数据解码完成后，Dubbo 会将响应对象派发到线程池上。要注意的是，线程池中的线程并非用户的调用线程，所以要想办法将响应对象从线程池线程传递到用户线程上。我们在 2.1 节分析过用户线程在发送完请求后的动作，即调用 DefaultFuture 的 get 方法等待响应对象的到来。当响应对象到来后，用户线程会被唤醒，并通过**调用编号**获取属于自己的响应对象。

- 将响应对象保存到相应的 DefaultFuture 实例中，然后再唤醒用户线程，随后用户线程即可从 DefaultFuture 实例中获取到相应结果



本篇文章在多个地方都强调过调用编号很重要。一般情况下，服务消费方会并发调用多个服务，每个用户线程发送请求后，会调用不同 DefaultFuture 对象的 get 方法进行等待。 一段时间后，服务消费方的线程池会收到多个响应对象。这个时候要考虑一个问题，如何将每个响应对象传递给相应的 DefaultFuture 对象，且不出错。答案是通过调用编号。DefaultFuture 被创建时，会要求传入一个 Request 对象。此时 DefaultFuture 可从 Request 对象中获取调用编号，并将 <调用编号, DefaultFuture 对象> 映射关系存入到静态 Map 中，即 FUTURES。线程池中的线程在收到 Response 对象后，会根据 Response 对象中的调用编号到 FUTURES 集合中取出相应的 DefaultFuture 对象，然后再将 Response 对象设置到 DefaultFuture 对象中。最后再唤醒用户线程，这样用户线程即可从 DefaultFuture 对象中获取调用结果了。整个过程大致如下图：

![img](http://dubbo.apache.org/docs/zh-cn/source_code_guide/sources/images/request-id-application.jpg)

## 3. 总结

本篇文章主要对 Dubbo 中的几种服务调用方式，以及从双向通信的角度对整个通信过程进行了详细的分析。按照通信顺序，通信过程包括服务消费方发送请求，服务提供方接收请求，服务提供方返回响应数据，服务消费方接收响应数据等过程。理解这些过程需要大家对网络编程，尤其是 Netty 有一定的了解。