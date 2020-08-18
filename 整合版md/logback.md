## 日志

- 日志的统一管理就变得非常困难。为了解决这个问题，就有了 SLF4J（Simple Logging Facade For Java）

- SLF4J 实现了三种功能

  - 一是提供了统一的日志门面 API，实现了中立的日志记录 API。
  - 二是桥接功能，用来把各种日志框架的 API桥接到SLF4J API。这样一来，即便你的程序中使用了各种日志 API 记录日志，最终都可以桥接到 SLF4J 门面 API
  - 三是适配功能，可以实现 SLF4J API 和实际日志框架的绑定。SLF4J 只是日志标准，我们还是需要一个实际的日志框架。日志框架本身没有实现 SLF4J API，所以需要有一个前置转换。Logback 就是按照 SLF4J API 标准实现的，因此不需要绑定模块做转换

- 日志重复记录

  - 第一个案例是，logger 配置继承关系导致日志重复记录

    - 同一条日志既会通过 logger 记录，也会发送到 root 记录，因此应用 package 下的日志出现了重复记录。

    - 如此配置的初衷是实现自定义的 logger 配置，让应用内的日志暂时开启 DEBUG 级别的日志记录。其实，他完全不需要重复挂载 Appender，去掉挂载的 Appender 即可

    - ```xml
      <?xml version="1.0" encoding="UTF-8" ?>
      <configuration>
          <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
              <layout class="ch.qos.logback.classic.PatternLayout">
                  <pattern>[%d{yyyy-MM-dd HH:mm:ss.SSS}] [%thread] [%-5level] [%logger{40}:%line] - %msg%n</pattern>
              </layout>
          </appender>
          <logger name="org.abc.commonmistakes.logging" level="DEBUG">
              <appender-ref ref="CONSOLE"/>
          </logger>
          <root level="INFO">
              <appender-ref ref="CONSOLE"/>
          </root>
      </configuration>
      ```

    - 如果自定义的需要把日志输出到不同的 Appender，比如将应用的日志输出到文件app.log、把其他框架的日志输出到控制台，可以设置的 additivity 属性为 false，这样就不会继承的 Appender 了

    - ```xml
      <logger name="org.abc.commonmistakes.logging" level="DEBUG" additivity="false">
          <appender-ref ref="FILE"/>
      </logger>
      <root level="INFO">
          <appender-ref ref="CONSOLE" />
      </root>
      ```

  - 第二个案例是，错误配置 LevelFilter 造成日志重复记录

    - 分析 ThresholdFilter 的源码发现，当日志级别大于等于配置的级别时返回 NEUTRAL，继续调用过滤器链上的下一个过滤器；否则，返回 DENY 直接拒绝记录日志：

    - LevelFilter 用来比较日志级别，然后进行相应处理：如果匹配就调用 onMatch 定义的处理方式，默认是交给下一个过滤器处理（AbstractMatcherFilter 基类中定义的默认值）；否则，调用 onMismatch 定义的处理方式，默认也是交给下一个过滤器处理

    - 定位到问题后，修改方式就很明显了：配置 LevelFilter 的 onMatch 属性为 ACCEPT，表示接收 INFO 级别的日志；配置 onMismatch 属性为 DENY，表示除了 INFO 级别都不记录：

    - ```xml
      <filter class="ch.qos.logback.classic.filter.LevelFilter">
          <level>INFO</level>
          <onMatch>ACCEPT</onMatch>
          <onMismatch>DENY</onMismatch>
      </filter>
      ```

- 使用异步日志改善性能的坑

  - EvaluatorFilter（求值过滤器），用于判断日志是否符合某个条件

    - 在这个案例中，我们给输出测试结果的那条日志上做了 time 标记

      ```java
      Marker timeMarker = MarkerFactory.getMarker("time");
      log.info(timeMarker, "took {} ms", System.currentTimeMillis() - begin);
      ```

    - 配合使用标记和 EvaluatorFilter，实现日志的按标签过滤，是一个不错的小技巧

      ```xml
      <filter class="ch.qos.logback.core.filter.EvaluatorFilter">
          <evaluator class="ch.qos.logback.classic.boolex.OnMarkerEvaluator">
              <marker>time</marker>
      ```

  - 使用 Logback 提供的 AsyncAppender 即可实现异步的日志记录。AsyncAppende 类似装饰模式，也就是在不改变类原有基本功能的情况下为其增添新功能。这样，我们就可以把 AsyncAppender 附加在其他的 Appender 上，将其变为异步的。

    - ```xml
      <appender name="ASYNCFILE" class="ch.qos.logback.classic.AsyncAppender">
      	<appender-ref ref="FILE"/> 
      </appender>
      ```

  - 很多关于 AsyncAppender 异步日志的坑，这些坑可以归结为三类

    - 记录异步日志撑爆内存；
    - 记录异步日志出现日志丢失；
    - 记录异步日志出现阻塞。

  - 出现这个问题的原因在于，AsyncAppender 提供了一些配置参数，而我们没用对。我们结合相关源码分析一下：

    - includeCallerData 用于控制是否收集调用方数据，默认是 false，此时方法行号、方法名等信息将不能显示
    - queueSize 用于控制阻塞队列大小，使用的 ArrayBlockingQueue 阻塞队列，默认大小是 256，即内存中最多保存 256 条日志。
    - discardingThreshold 是控制丢弃日志的阈值，主要是防止队列满后阻塞。默认情况下，队列剩余量低于队列长度的 20%，就会丢弃 TRACE、DEBUG 和 INFO 级别的日志
    - neverBlock 用于控制队列满的时候，加入的数据是否直接丢弃，不会阻塞等待，默认是false）。这里需要注意一下 offer 方法和 put 方法的区别，当队列满的时候 offer 方法不阻塞，而 put 方法会阻塞；neverBlock 为 true 时，使用 offer方法。

  - 我们可以继续分析下异步记录日志出现坑的原因

    - queueSize 设置得特别大，就可能会导致 OOM。
    - queueSize 设置得比较小（默认值就非常小），且 discardingThreshold 设置为大于 0的值（或者为默认值），队列剩余容量少于 discardingThreshold 的配置就会丢弃<=INFO 的日志。这里的坑点有两个。
    - neverBlock 默认为 false，意味着总可能会出现阻塞。如果 discardingThreshold 为0，那么队列满时再有日志写入就会阻塞；如果 discardingThreshold 不为 0，也只会丢弃 <=INFO 级别的日志，那么出现大量错误日志时，还是会阻塞程序。

  - 可以看出 queueSize、discardingThreshold 和 neverBlock 这三个参数息息相关，务必按需进行设置和取舍，到底是性能为先，还是数据不丢为先：

    - 如果考虑绝对性能为先，那就设置 neverBlock 为 true，永不阻塞。
    - 如果考虑绝对不丢数据为先，那就设置 discardingThreshold 为 0，即使是 <=INFO 的级别日志也不会丢，但最好把 queueSize 设置大一点，毕竟默认的 queueSize 显然太小，太容易阻塞。
    - 如果希望兼顾两者，可以丢弃不重要的日志，把 queueSize 设置大一点，再设置一个合理的 discardingThreshold。

- 生产级项目的文件日志肯定需要按时间和日期进行分割和归档处理，以避免单个文件太大，同时保留一定天数的历史日志，你知道如何配置吗？

  - ```xml
    <!-- 按照每天和固定大小(5MB)生成日志文件【最新的日志，是没有日期没有数字的】 -->
        <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>${LOG_HOME}/项目名.log</file>
            <append>true</append>
            <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
                <fileNamePattern>${LOG_HOME}/项目名_%d{yyyy-MM-dd}.%i.log</fileNamePattern>
                <!--日志文件保留天数-->
                <MaxHistory>30</MaxHistory>
                <!--日志文件最大的大小-->
                <MaxFileSize>5MB</MaxFileSize>
            </rollingPolicy>
            <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
                <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            </encoder>
           <!-- <filter class="ch.qos.logback.classic.filter.LevelFilter">
                <level>TRACE</level>
                <onMatch>ACCEPT</onMatch>
                <onMismatch>DENY</onMismatch>
            </filter>-->
        </appender>
     
    ```

    





## 简介

- `springboot`默认使用的日志框架是`logback`

- 三个模块组成

  - `logback-core` 是其它模块的基础设施，其它模块基于它构建，显然，`logback-core` 提供了一些关键的通用机制。
  - `logback-classic` 的地位和作用等同于 `Log4J`，它也被认为是 `Log4J` 的一个改进版，并且它实现了简单日志门面 `SLF4J`；
  -  `logback-access` 主要作为一个与 `Servlet` 容器交互的模块，比如说`tomcat`或者 `jetty`，提供一些与 `HTTP` 访问相关的功能。

  

  

## logback.xml

- 整个logback.xml配置文件的结构

  - ```xml
    <!--scan:当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true。
    scanPeriod:设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当scan为true时，此属性生效。默认的时间间隔为1分钟。
    debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。-->
    <configuration scan="true" scanPeriod="60 seconds" debug="false">  
        <!--用来定义变量值的标签，property标签有两个属性，name和value；其中name的值是变量的名称，value的值时变量定义的值。通过property定义的值会被插入到logger上下文中。定义变量后，可以使“${name}”来使用变量。-->
        <property name="xxx" value="yyy" /> 
         <!-- 属性文件:在properties文件中找到对应的配置项 -->
        <springProperty scope="context" name="logging.path"  source="logging.path"/>
        <springProperty scope="context" name="logging.level" source="logging.level.com.xxx.spring.boot"/>
        <!--每个logger都关联到logger上下文，默认上下文名称为“default”。-->
        <contextName>${xxx}</contextName> 
        
        <!--负责写日志的组件appender 有两个属性 name和class;name指定appender名称，class指定appender的全限定名-->
        <appender>
            
            <!--filter其实是appender里面的子元素。它作为过滤器存在，执行一个过滤器会有返回DENY，NEUTRAL，ACCEPT三个枚举值中的一个。
        DENY：日志将立即被抛弃不再经过其他过滤器
        NEUTRAL：有序列表里的下个过滤器接着处理日志
        ACCEPT：日志会被立即处理，不再经过剩余过滤器-->
            
            <!--临界值过滤器，过滤掉低于指定临界值的日志。当日志级别等于或高于临界值时，过滤器返回NEUTRAL；当日志级别低于临界值时，日志会被拒绝。-->
            <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        		<level>INFO</level>
    		</filter>
            <!--级别过滤器，根据日志级别进行过滤。如果日志级别等于配置级别，过滤器会根据onMath(用于配置符合过滤条件的操作) 和 onMismatch(用于配置不符合过滤条件的操作)接收或拒绝日志。-->
            <filter class="ch.qos.logback.classic.filter.LevelFilter">   
                <level>INFO</level>   
                <onMatch>ACCEPT</onMatch>   
                <onMismatch>DENY</onMismatch>   
            </filter> 
            
            <!--这个子标签用来描述滚动策略的。这个只有appender的class是RollingFileAppender时才需要配置。这个也会涉及文件的移动和重命名（a.log->a.log.2018.07.22）。-->
            <!--TimeBasedRollingPolicy:最常用的滚动策略，它根据时间来制定滚动策略，既负责滚动也负责触发滚动-->
            <rollingPolicy                 class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <!--日志文件输出的文件名:按天回滚 daily -->
                <FileNamePattern>
                    ${logging.path}/xxx-spring-boot/xxx-loggerone.log.%d{yyyy-MM-dd}
                </FileNamePattern>
                <!--日志文件保留天数-->
                <MaxHistory>30</MaxHistory>
            </rollingPolicy>
    
            <!--encoder 子标签:对记录事件进行格式化。-->
            <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50}
        - %msg%n</pattern>
        <charset>UTF-8</charset>
    </encoder>
    
            //xxxx
        </appender>   
        
        <!--用来设置某一个包或者具体的某一个类的日志打印级别以及指定appender
    	ppender让我们的应用知道怎么打、打印到哪里、打印成什么样；而logger则是告诉应用哪些可以这么打。
        name:用来指定受此logger约束的某一个包或者具体的某一个类。
        level:用来设置打印级别（TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF），还有一个值INHERITED或者同义词NULL，代表强制执行上级的级别。如果没有设置此属性，那么当前logger将会继承上级的级别。
        addtivity:用来描述是否向上级logger传递打印信息。默认是true。
    	-->
    <logger name="com.xxx.spring.boot.controller"
            level="${logging.level}" additivity="false">
        <appender-ref ref="GLMAPPER-LOGGERONE" />
    </logger>
    
        <!--根logger，也是一种logger，且只有一个level属性-->
        <root>             
           //xxxx
        </root>  
    </configuration>  
    ```

  - 想使用spring扩展profile支持，要以logback-spring.xml命名，其他如property需要改为springProperty

  - application.properties中指定了日志的打印级别和日志的输出位置

    - ```yaml
      #设置应用的日志级别
      logging.level.com.xxx.spring.boot=INFO
      #路径
      logging.path=./logs
      ```

  - java

    - ```java
      // 在getLogger中我们是将当前对象的class作为参数的，这个是为了打印时获取其全限定名的
      private static final Logger LOGGER =
       LoggerFactory.getLogger(TestLogTask.class);
      
      private static final Logger LOGGER =
          LoggerFactory.getLogger("TEST-LOG");
      
      <!--这里的name和业务类中的getLogger中的字符串是一样的-->
      <logger name="TEST-LOG" level="${logging.level}" additivity="true">
              <appender-ref ref="ROOT-APPENDER" />
              <appender-ref ref="ERROR-APPENDER" />
      </logger>
      ```

      

  

