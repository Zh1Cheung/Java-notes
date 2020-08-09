## 介绍

- 轻量级的 DI / IoC 和 AOP 容器的开源框架
  
  - 要了解**控制反转( Inversion of Control )**, 我觉得有必要先了解软件设计的一个重要思想：**依赖倒置原则（Dependency Inversion Principle ）**
    
    - 假设我们设计一辆汽车：先设计轮子，然后根据轮子大小设计底盘，接着根据底盘设计车身，最后根据车身设计好整个汽车。这里就出现了一个“依赖”关系：汽车依赖车身，车身依赖底盘，底盘依赖轮子。
    
    - 我们现在换一种思路。我们先设计汽车的大概样子，然后根据汽车的样子来设计车身，根据车身来设计底盘，最后根据底盘来设计轮子。这时候，依赖关系就倒置过来了：轮子依赖底盘， 底盘依赖车身， 车身依赖汽车。
    
    - 这时候，上司再说要改动轮子的设计，我们就只需要改动轮子的设计，而不需要动底盘，车身，汽车的设计了。
    
      这就是依赖倒置原则——把原本的高层建筑依赖底层建筑“倒置”过来，变成底层建筑依赖高层建筑。高层建筑决定需要什么，底层去实现这样的需求，但是高层并不用管底层是怎么实现的。这样就不会出现前面的“牵一发动全身”的情况。
    
    - ![img](https://pic2.zhimg.com/80/v2-ee924f8693cff51785ad6637ac5b21c1_720w.png)
    
    - **所谓依赖注入，就是把底层类作为参数传入上层类，实现上层类对下层类的“控制**”。这里我们用**构造方法传递的依赖注入方式**
    
      - ![img](https://pic2.zhimg.com/80/v2-99ad2cd809fcb86dd791ff7f65fb1779_720w.png)
    
    - 在实际的工程中，这种设计模式还有利于**不同组的协同合作和单元测试：**比如开发这四个类的分别是四个不同的组，那么只要定义好了接口，四个不同的组可以同时进行开发而不相互受限制；而对于单元测试，如果我们要写Car类的单元测试，就只需要Mock一下Framework类传入Car就行了，而不用把Framework, Bottom, Tire全部new一遍再来构造Car。
    
    - 这里我们是采用的**构造函数传入**的方式进行的依赖注入。其实还有另外两种方法：**Setter传递**和**接口传递**。这里就不多讲了，核心思路都是一样的，都是为了实现**控制反转**。
    
      - ![img](https://pic2.zhimg.com/80/v2-861683acac47577c81f2b7493dd05649_720w.png)
    
  - **控制反转容器(IoC Container)**
  
    - 显然你也应该观察到了，因为采用了依赖注入，在初始化的过程中就不可避免的会写大量的new。这里IoC容器就解决了这个问题。**这个容器可以自动对你的代码进行初始化，你只需要维护一个Configuration（可以是xml可以是一段代码），而不用每次初始化一辆车都要亲手去写那一大段初始化的代码**。这是引入IoC Container的第一个好处。
  
    - IoC Container的第二个好处是：**我们在创建实例的时候不需要了解其中的细节。**
  
      - 这个过程中，我们需要了解整个Car/Framework/Bottom/Tire类构造函数是怎么定义的，才能一步一步new/注入。
      - 而IoC Container在进行这个工作的时候是反过来的，它先从最上层开始往下找依赖关系，到达最底层之后再往上一步一步new
      - DI的资料，里面就大概是这样解释DI的。我还记得当时的我读这一段的心路历程：
      - “高层不应该依赖底层”这个还好理解，然后“它们都应该依赖于抽象” WTF? “抽象不应该依赖于具体细节” Whaaat? “具体细节应该依赖于抽象” Fuck me...犹如在读哲学课文。
      - 我举得那个例子里面说到了“轮胎‘依赖’于底盘”   这个概念，其实就是为了尽量避免使用什么“抽象”“具象”“实例”这些让初学者费解的词汇。其实底盘定义了轮胎接口之后，轮胎必须按照接口来写它的成员函数，不就可以理解为一种“高层‘控制’底层，底层‘依赖’于高层”么。
  
      
  
  - IoC**控制反转( Inversion of Control )**，其实就是一种设计思想。使用 Spring 来实现 IoC，意味着将你设计好的对象交给Spring 容器控制，而不是直接在对象内部控制。
  
    - 或许你能想到的是，使用 IoC 方便、可以实现解耦。但在我看来，相比于这两个原因，更重要的是 IoC 带来了更多的可能性。 
  
    - 如果以容器为依托来管理所有的框架、业务对象，我们不仅可以无侵入地调整对象的关系，还可以无侵入地随时调整对象的属性，甚至是实现对象的替换。这就使得框架开发者在程序 
  
    - 背后实现一些扩展不再是问题，带来的可能性是无限的。比如我们要监控的对象如果是Bean，实现就会非常简单。所以，这套容器体系，不仅被 Spring Core 和 Spring Boot 大量依赖，还实现了一些外部框架和 Spring 的无缝整合。
  
    - DI 是 Spring 使用的方式，容器负责组件的装配。
  
    - Spring 的 IoC 设计支持以下功能：
  
      1. 依赖注入
      2. 依赖检查
      3. 自动装配
      4. 支持集合
      5. 指定初始化方法和销毁方法
      6. 支持回调某些方法（但是需要实现 Spring 接口，略有侵入）
      7. **对于 IoC 来说，最重要的就是容器。容器管理着 Bean 的生命周期，控制着 Bean 的依赖注入。**
  
    - 这里小结一下：IoC 在 Spring 里，只需要低级容器就可以实现，2 个步骤：
  
      1. 加载配置文件，解析成 BeanDefinition 放在 Map 里。
      2. 调用 getBean 的时候，从 BeanDefinition 所属的 Map 里，拿出 Class 对象进行实例化，同时，如果有依赖关系，将递归调用 getBean 方法 —— 完成依赖注入。
  
      上面就是 Spring 低级容器（BeanFactory）的 IoC。
  
    - 至于高级容器 ApplicationContext，他包含了低级容器的功能，当他执行 refresh 模板方法的时候，将刷新整个容器的 Bean。同时其作为高级容器，包含了太多的功能。一句话，他不仅仅是 IoC。他支持不同信息源头，支持 BeanFactory 工具类，支持层级容器，支持访问文件资源，支持事件发布通知，支持接口回调等等。
  
    - 但是请注意，实现 Spring 接口代表着你这个应用就绑定死 Spring 了！代表 Spring 具有侵入性！要知道，Spring 发布时，无侵入性就是他最大的宣传点之一 —— 即 IoC 容器可以随便更换，代码无需变动。而现如今，Spring 已然成为 J2EE 社区准官方解决方案，也没有了所谓的侵入性这个说法。因为他就是标准，和 Servlet 一样，你能不实现 Servlet 的接口吗？: -)
  
  - AOP其实只是OOP的补充而已。OOP从横向上区分出一个个的类来，而AOP则从纵向上向对象中加入特定的代码。
  
    - AOP，体现了松耦合、高内聚的精髓，在切面集中实现横切关注点（缓存、权限、日志等），然后通过切点配置把代码注入合适的地方。切面、切点、增强、连接点，是 AOP 中非常重要的概念
  
  - ioc
    - BeanFactory
      - BeanFactory是个Factory，也就是IOC容器或对象工厂，FactoryBean是个Bean
      - BeanFactory是接口，给具体的IOC容器的实现提供了规范
      -  FactoryBean也是接口，为IOC容器中Bean的实现加上了一个简单工厂模式
    -  ApplicationContext
      - ApplicationContext 是 BeanFactory 的子接口之一，对 BeanFactory 功能做了许多的扩展
  
- 过滤器和拦截器
  - 过滤器和拦截器最大的区别在于：过滤器是servlet的规范规定的，只能用于过滤请求，而interceptor是Spring里面面向切面编程的一种实现
    - 拦截器是基于java的反射机制的，而过滤器是基于函数回调。
    - 拦截器不依赖与servlet容器，过滤器依赖与servlet容器。
    - 拦截器可以多次被调用，而过滤器只能在容器初始化时被调用一次。
    - 拦截器可以获取IOC容器中的各个bean，而过滤器不行
  



## Bean的生命周期

- ```
  * bean的生命周期：
  *     bean创建---初始化----销毁的过程
  * 容器管理bean的生命周期；
  * 我们可以自定义初始化和销毁方法；容器在bean进行到当前生命周期的时候来调用我们自定义的初始化和销毁方法
  ```

- 调用postProcessBeanFactory()方法

- 实例化bean对象
  - 实例化Bean之前调用postProcessBeforeInstantiation()方法
  - 实例化Bean之后调用该接口的postProcessAfterInstantiation()方法
  - 声明，只生成对象不赋值的过程。
    初始化，是给对象赋值的过程。
    实例化，是使用new为对象分配内存的过程。
  
- 设置对象属性

- 如果Bean实现了BeanNameAware，工厂调用Bean的setBeanName()方法传递Bean的ID

- 如果Bean实现了BeanFactoryAware，工厂调用setBeanFactory()方法传入工厂自身

- 如果Bean实现了ApplicationContextAware，调用接口的setApplicationContext()方法，将ApplicationContext实例设置到Bean中

- 如果实现了 InitializingBean 接口，则会调用 afterPropertiesSet 方法

- 调用Bean的初始化方法   init-method属性
  - 调用Bean的初始化方法之前  将Bean实例传递给Bean的前置处理器的postProcessBeforeInitialization
  - 调用Bean的初始化方法之后  将Bean实例传递给Bean的后置处理器的postProcessAfterInitialization
  
- 使用Bean容器关闭之前，调用Bean的销毁方法





## 作用域

- Spring Bean 有五个作用域，其中最基础的有下面两种
  - Singleton，这是 Spring 的默认作用域，也就是为每个 IOC 容器创建唯一的一个 Bean实例。
  - Prototype，针对每个 getBean 请求，容器都会单独创建一个 Bean 实例
  - 从 Bean 的特点来看，Prototype 适合有状态的 Bean，而 Singleton 则更适合无状态的情况。另外，使用 Prototype 作用域需要经过仔细思考，毕竟频繁创建和销毁 Bean 是有明显开销的
- 如果是 Web 容器，则支持另外三种作用域
  - Request，为每个 HTTP 请求创建单独的 Bean 实例。
  - Session，很显然 Bean 实例的作用域是 Session 范围。
  - GlobalSession，用于 Portlet 容器，因为每个 Portlet 有单独的 Session，GlobalSession 提供一个全局性的 HTTP Session

- 他们是什么时候创建的:
  - 一个单例的bean,而且lazy-init属性为false(默认),在Application Context创建的时候构造
  - 一个单例的bean,lazy-init属性设置为true,那么,它在第一次需要的时候被构造.
  - 其他scope的bean,都是在第一次需要使用的时候创建

- 他们是什么时候销毁的:
  - 单例的bean始终 存在与application context中, 只有当 application 终结的时候,才会销毁
  - 和其他scope相比,Spring并没有管理prototype实例完整的生命周期,在实例化,配置,组装对象交给应用后,spring不再管理.只要bean本身不持有对另一个资源（如数据库连接或会话对象）的引用，只要删除了对该对象的所有引用或对象超出范围，就会立即收集垃圾.
  - Request: 每次客户端请求都会创建一个新的bean实例,一旦这个请求结束,实例就会离开scope,被垃圾回收.
  - Session: 如果用户结束了他的会话,那么这个bean实例会被GC.





## Spring事务

- 事务属性包含了5个方面
- 一、传播行为：当事务方法被另一个事务方法调用时，必须指定事务应该如何传播
  - PROPAGATION_REQUIRED ：required , 必须。默认值，A如果有事务，B将使用该事务；如果A没有事务，B将创建一个新的事务。
  - PROPAGATION_SUPPORTS：supports ，支持。A如果有事务，B将使用该事务；如果A没有事务，B将以非事务执行。　
  - PROPAGATION_MANDATORY：mandatory ，强制。A如果有事务，B将使用该事务；如果A没有事务，B将抛异常。　
  - PROPAGATION_REQUIRES_NEW ：requires_new，必须新的。如果A有事务，将A的事务挂起，B创建一个新的事务；如果A没有事务，B创建一个新的事务。　
  - PROPAGATION_NOT_SUPPORTED ：not_supported ,不支持。如果A有事务，将A的事务挂起，B将以非事务执行；如果A没有事务，B将以非事务执行。　
  - PROPAGATION_NEVER ：never，从不。如果A有事务，B将抛异常；如果A没有事务，B将以非事务执行。
  - PROPAGATION_NESTED ：nested ，嵌套。A和B底层采用保存点机制，形成嵌套事务。

- 二、隔离级别：定义了一个事务可能受其他并发事务影响的程度。
  - ISOLATION_DEFAULT：使用后端数据库默认的隔离级别
  - ISOLATION\_READ\_UNCOMMITTED：最低的隔离级别，允许读取尚未提交的数据变更，可能会导致脏读、幻读或不可重复读
  - ISOLATION\_READ\_COMMITTED：允许读取并发事务已经提交的数据，可以阻止脏读，但是幻读或不可重复读仍有可能发生
  - ISOLATION\_REPEATABLE\_READ：对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，可以阻止脏读和不可重复读，但幻读仍有可能发生
  - ISOLATION_SERIALIZABLE：最高的隔离级别，完全服从ACID的隔离级别，确保阻止脏读、不可重复读以及幻读，也是最慢的事务隔离级别，因为它通常是通过完全锁定事务相关的数据库表来实现的
- 三、是否只读
- 四、事务超时
- 五、回滚规则
  - 默认情况下，事务只有遇到运行期异常时才会回滚，而在遇到检查型异常时不会回滚

- 声明式事务其实说白了是一种特殊的aop应用，它其实包括两种advice，一种是around，另外一种是after-throwing。利用around advice在方法执行前，先关闭数据库的自动提交功能，然后设定一个标志符。根据业务代码实际的情况，对标志符赋不同的值，如果数据更新成功赋true，否则false。在业务方法执行完之后的部分对标志符进行处理。如为true，则提交数据库操作，否则就进行回滚。
  另外还会使用after-throwing，对出错的信息进行记录。然后再将错误抛出至上层。



## 循环依赖

- 关于Spring bean的创建，其本质上还是一个对象的创建，一个完整的对象包含两部分：当前对象实例化和对象属性的实例化。

  - 在Spring中，对象的实例化是通过反射实现的，而对象的属性则是在对象实例化之后通过一定的方式设置的。
  - Spring实例化bean是通过ApplicationContext.getBean()方法来进行的。
  - 如果要获取的对象依赖了另一个对象，那么其首先会创建当前对象，然后通过递归的调用ApplicationContext.getBean()方法来获取所依赖的对象，最后将获取到的对象注入到当前对象中。
  - Spring是通过递归的方式获取目标bean及其所依赖的bean的；

- 源码讲解

  - 需要注意的一个点，Spring是如何标记开始生成的A对象是一个半成品，并且是如何保存A对象的。
    - 这里的标记工作Spring是使用ApplicationContext的属性SetsingletonsCurrentlyInCreation来保存的，而半成品的A对象则是通过MapsingletonFactories来保存的
  - AbstractBeanFactory.doGetBean()方法中获取对象的方法如下：
    - 两个getSingleton()方法
      - 第一个步骤的getSingleton()方法的作用是尝试从缓存中获取目标对象，如果没有获取到，则尝试获取半成品的目标对象；如果第一个步骤没有获取到目标对象的实例，那么就进入第二个步骤
      - 第二个步骤的getSingleton()方法的作用是尝试创建目标对象（实例化当前尝试获取的bean对象），并且为该对象注入其所依赖的属性。
  - 在整个过程中会调用三次doGetBean()方法
    - 第一次调用的时候会尝试获取A对象实例，此时走的是第一个getSingleton()方法，由于没有已经创建的A对象的成品或半成品，因而这里得到的是null
    - 然后就会调用第二个getSingleton()方法，创建A对象的实例，然后递归的调用doGetBean()方法，尝试获取B对象的实例以注入到A对象中
      - 此时由于Spring容器中也没有B对象的成品或半成品，因而还是会走到第二个getSingleton()方法，在该方法中创建B对象的实例
      - 在该方法中会得到一个半成品的A对象的实例，然后将该实例返回，并且将其注入到B对象的属性a中，此时B对象实例化完成。
    - 创建完成之后，尝试获取其所依赖的A的实例作为其属性，因而还是会递归的调用doGetBean()方法，此时需要注意的是，在前面由于已经有了一个半成品的A对象的实例，因而这个时候，再尝试获取A对象的实例的时候，会走第一个getSingleton()方法
    - 然后，将实例化完成的B对象递归的返回，此时就会将该实例注入到A对象中，这样就得到了一个成品的A对象。

  

## Spring MVC 过程

![img](https://pic4.zhimg.com/v2-11b8c43869683d5fe9f50efb019f0df5_b.png)

1. 设置属性
2. 根据 Request 请求的 URL 得到对应的 handler 执行链，其实就是拦截器和 Controller 代理对象。
3. 得到 handler 的适配器
4. 循环执行 handler 的 pre 拦截器
5. 执行真正的 handler，并返回 ModelAndView(Handler 是个代理对象，可能会执行 AOP )
6.  循环执行 handler 的 post 拦截器
7.  根据 ModelAndView 信息得到 View 实例
8. 渲染 View 返回



## Springboot

- 概述
  - Spring Boot提供的只是一些starters，这些Starter依赖了(maven dependence)对应的框架或技术
  - springboot 只是提供了一套默认配置，快速开发。采用约定大于配置的方式，简化了大量的xml配置，真正做到了开箱即用。
  - Spring Boot本身不是脚手架，也不是框架，Spring Boot只是Spring本身的扩展
- 源码
  - 代码中只有⼀个@SpringBootApplication 注解 和 调用了SpringApplication#run 方法.
    - @SpringbootApplication注解
      - @SpringBootConfiguration：springboot配置类
      - @EnableAutoConfiguration：实现自动配置
      - @ComponentScan ：扫描指定的包下的类，加载符合条件的组件
  - run⽅法
    - 第一步：获取并启动监听器
      - 通过指定的classloader 从META-INF/spring.factories获取指定的Spring的工厂实例
      - 获取Spring容器的启动监听器：EventPublishingRunListener
    - 第二步：构造应用上下文环境
      - 调用prepareEnvironment方法
      - 首先是创建并配置相应的环境，然后根据用户的配置，配置系统环境，然后启动加载项目配置文件的监听器，并加载系统配置文件。
    - 第三步：初始化应用上下文
      - 调用createApplicationContext()
      - 应用上下文的beanFactory属性就是IoC容器（DefaultListableBeanFactory）
    - 第四步：刷新应用上下文前的准备阶段
      - 调用prepareContext()获取启动类，并且将启动类注入容器
      - 将前面创建的上下文强转为BeanDefinitionRegistry
      - 最终来到DefaultListableBeanFactory类的registerBeanDefinition()方法，将启动类的BeanDefinition注册进DefaultListableBeanFactory容器
    - 第五步：刷新应用上下文
      - 调用refresh()
        - Resource定位
          - 主类所在包的路径定位
          - SpringBoot的自动装配（各种starter）定位
          - 在SpringBoot中有很多的@EnableXXX注解，其底层是@Import注解，方法中也实现了对该注解指定的配置类的定位
        - 载入BeanDefinition
        - 注册BeanDefinition

- 自动装配

  - @SpringBootApplication注解上的@EnableAutoConfiguration注解

    - @Import 把所有符合条件的 Bean 加载到Ioc 容器中。
    - 通过@ConditionalOnClass对@Configuration相应的bean进行有选择的实例化

  - ```java
    @Configuration
    @ConditionalOnClass({ RabbitTemplate.class, Channel.class })
    @EnableConfigurationProperties(RabbitProperties.class)
    @Import(RabbitAnnotationDrivenConfiguration.class)
    public class RabbitAutoConfiguration {}
    
    @ConfigurationProperties(prefix = "spring.rabbitmq")
    public class RabbitProperties {}
    
    //@ConfigurationProperties(perfix = "spring.rabbitmq")作用是从配置文件中加载指定前缀的配置，并自动设置到对应的成员变量上。也正是通过这种方式，真正实现了配置的自动注入。
    @EnableAutoConfiguration->AutoConfigurationImportSelector->spring.factories->EnableAutoConfiguration全类名->RabbitAutoConfiguration(自动配置类)->@EnableConfigurationProperties->RabbitProperties(具体的配置类)->@ConfigurationProperties(prefix = "spring.rabbitmq")
    ```

  - @EnableAutoConfiguration注解

    - 借助@Import（AutoConfigurationImportSelector）来收集所有符合自动配置条件的bean，并加载到容器中
    - @AutoConfigurationPackage 自动配置包，将主程序类所在包及所有子包下的组件到扫描到容器中

  - AutoConfigurationImportSelector

    - 将所有符合条件的@Configuration自动配置类都加载到当前容器(ApplicationContext)中

    - 实现了selectImports() 方法，用来筛选@Configuration自动配置类
      selectImports() 方法核心功能就是调用SpringFactoriesLoader获取spring.factories中

    - EnableAutoConfiguration所对应的Configuration类列表，由@ComponentScan注解中的exclude/excludeName参数筛选一遍，再由AutoConfigurationImportFilter类筛选一遍，得到最终的用于Import的@configuration注解的自动配置类。

      







## Spring Session

- 简介

  - Spring Session解决Session共享	
  - spring-session的核心思想在于此：将session从web容器中剥离，存储在独立的存储服务器(Redis)中
  - 当request进入web容器，根据request获取session时，由spring-session负责从存储器中获取session，如果存在则返回，如果不存在则创建并持久化至存储器中

- 工作原理

  - SessionRepositoryFilter：Servlet规范中Filter的实现，包装请求和响应

    - 负责包装切换HttpSession至Spring Session的请求和响应
      - 包装原始HttpServletRequest至SessionRepositoryRequestWrapper
      - 包装原始HttpServletResponse响应至SessionRepositoryResponseWrapper
    - 将session持久化至存储器中

  - SessionHttpServerletRequestWrapper

    - SessionRepositoryRequestWrapper继承Servlet规范中定义的包装器HttpServletRequestWrapper
      - 在构造方法中将原有的HttpServletRequest通过调用super完成对HttpServletRequestWrapper中持有的HttpServletRequest初始化赋值，然后重写和session相关的方法。这样就保证SessionRepositoryRequestWrapper的其他方法调用都是使用原有的HttpServletRequest的数据，只有session相关的是重写的逻辑
      - 重写HttpServletRequest的getSession()方法
        - 获取当前request的sessionId
        - 根据sessionId获取spring session
        - 如果spring session不为空，则将spring session包装成HttpSession并设置到当前Request的attribute中
        - 持久化只持久spring session，并不是将spring session包装后的HttpSession持久化，因为HttpSession不过是包装器，持久化没有意义。
    - SessionRepositoryRequestWrapper用来包装原始的HttpServletRequest。实现了HttpSession切换至Spring Sessio

  - SessionRepositoryResponseWrapper

    - 我们需要在response被提交之前确保session被创建。如果response已经被提交，将没有办法追踪session（例如：无法将cookie写入response以跟踪哪个session id）

  - Session：Spring Session模块

    - spring-session单独抽象出Session层接口，可以应对多种场景下不同的session的实现，然后通过适配器模式将Session适配成HttpSession的接口

    - Session层是spring-session对session的抽象，定义了Session的基本行为

      - > getId：获取sessionId
        > setAttribute：设置session属性
        > getAttribte：获取session属性

    - ExipringSession：提供Session额外的过期特性

      - > setLastAccessedTime：设置最近Session会话过程中最近的访问时间
        > getLastAccessedTime：获取最近的访问时间
        > setMaxInactiveIntervalInSeconds：设置Session的最大闲置时间
        > getMaxInactiveIntervalInSeconds：获取最大闲置时间
        > isExpired：判断Session是否过期

    - RedisSession：提供Session的持久化能力

  - SessionRepository：管理Spring Session的模块

    - 创建、保存、获取、删除Session的接口行为。根据Session的不同，分为很多种Session操作仓库
    - RedisOperationsSessionRepository
      - Redis会为每个RedisSession存储三个k-v
        - 第一个k-v用来存储Session的详细信息
        - 第二个k-v用来表示Session在Redis中的过期
        - 第三个k-v存储这个Session的id
      - 对于Session的实现，需要支持HttpSessionEvent，即Session创建、过期、销毁等事件。当应用用监听器设置监听相应事件，Session发生上述行为时，监听器能够做出相应的处理。
      - Redis的强大之处在于支持KeySpace Notifiction——键空间通知。即可以监视某个key的变化，如删除、更新、过期

  - HttpSessionStrategy：映射HttpRequst和HttpResponse到Session的策略整合实战 

  




