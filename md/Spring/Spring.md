## 介绍

- 轻量级的 DI / IoC 和 AOP 容器的开源框架
  - AOP其实只是OOP的补充而已。OOP从横向上区分出一个个的类来，而AOP则从纵向上向对象中加入特定的代码。
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
- Bean的生命周期
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
  - 调用Bean的初始化方法   init-method属性
    - 调用Bean的初始化方法之前  将Bean实例传递给Bean的前置处理器的postProcessBeforeInitialization
    - 调用Bean的初始化方法之后  将Bean实例传递给Bean的后置处理器的postProcessAfterInitialization
  - 使用Bean容器关闭之前，调用Bean的销毁方法







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

  















































## Spring Session