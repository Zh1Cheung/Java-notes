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

      

  

