## 异常处理

- 捕获和处理异常容易犯的错

  - “统一异常处理”方式正是我要说的第一个错：不在业务代码层面考虑异常处理，仅在框架层面粗犷捕获和处理异常。

  - 每层架构的工作性质不同，且从业务性质上异常可能分为业务异常和系统异常两大类，这就决定了很难进行统一的异常处理。我们从底向上看一下三层架构：

    - Repository 层出现异常或许可以忽略，或许可以降级，或许需要转化为一个友好的异常。如果一律捕获异常仅记录日志，很可能业务逻辑已经出错，而用户和程序本身完全感知不到
    - Service 层往往涉及数据库事务，出现异常同样不适合捕获，否则事务无法自动回滚。此外 Service 层涉及业务逻辑，有些业务逻辑执行中遇到业务异常，可能需要在异常后转入分支业务流程。如果业务异常都被框架捕获了，业务功能就会不正常。
    - 如果下层异常上升到 Controller 层还是无法处理的话，Controller 层往往会给予用户友好提示，或是根据每一个 API 的异常表返回指定的异常类型，同样无法对所有异常一视同仁。

  - 因此，我不建议在框架层面进行异常的自动、统一处理，尤其不要随意捕获异常。但，框架可以做兜底工作。如果异常上升到最上层逻辑还是无法处理的话，可以以统一的方式进行异常转换，比如通过 @RestControllerAdvice + @ExceptionHandler，来捕获这些“未处理”异常

  - 第二个错，捕获了异常后直接生吞。在任何时候，我们捕获了异常都不应该生吞，也就是直
    接丢弃异常不记录、不抛出。这样的处理方式还不如不捕获异常，因为被生吞掉的异常一旦
    导致 Bug，就很难在程序中找到蛛丝马迹，使得 Bug 排查工作难上加难。

  - 第三个错，丢弃异常的原始信息。我们来看两个不太合适的异常处理方式，虽然没有完全生吞异常，但也丢失了宝贵的异常信息。

    - 捕获异常后，完全不记录原始异常，直接抛出一个转换后异常，导致出了问题不知道 IOException 具体是哪里引起的

    - 只记录了异常消息，却丢失了异常的类型、栈等重要信息

      ```java
      // 这两种处理方式都不太合理，可以改为如下方式：	
      	@GetMapping("right1")
          public void right1() {
              try {
                  readFile();
              } catch (IOException e) {
                  log.error("文件读取错误", e);
                  throw new RuntimeException("系统忙请稍后再试");
              }
          }
      // 或者，把原始异常作为转换后新异常的 cause，原始异常信息同样不会丢：
          @GetMapping("right2")
          public void right2() {
              try {
                  readFile();
              } catch (IOException e) {
                  throw new RuntimeException("系统忙请稍后再试", e);
              }
          }
      ```

  - 第四个错，抛出异常时不指定任何消息，直接抛出没有message 的异常：

    - throw new RuntimeException();
    - java.lang.RuntimeException: null。这里的 null 非常容易引起误解。按照空指针问题排查半天才发现，其实是异常的 message为空。

- 小心 finally 中的异常

  - 在 finally 中抛出一个异常,最后在日志中只能看到 finally 中的异常，虽然 try 中的逻辑出现了异常，但却被 finally中的异常覆盖了。这是非常危险的，特别是 finally 中出现的异常是偶发的，就会在部分时候覆盖 try 中的异常，让问题更不明显
  - 至于异常为什么被覆盖，原因也很简单，因为一个方法无法出现两个异常。
    - 修复方式是，finally 代码块自己负责异常捕获和处理
    - 或者可以把 try 中的异常作为主异常抛出，使用 addSuppressed 方法把 finally 中的异常附加到主异常上
    - 其实这正是 try-with-resources 语句的做法，对于实现了 AutoCloseable 接口的资源，建议使用 try-with-resources 来释放资源，否则也可能会产生刚才提到的，释放资源时出现的异常覆盖主异常的问题

- 千万别把异常定义为静态变量

  - 把异常定义为静态变量会导致异常信息固化，这就和异常的栈一定是需要根据当前调用来动态获取相矛盾。

  - 修复方式很简单，改一下 Exceptions 类的实现，通过不同的方法把每一种异常都 new 出来抛出即可

  - ```java
        private void createOrderWrong() {
            throw Exceptions.ORDEREXISTS;
        }
    
        private void cancelOrderWrong() {
            throw Exceptions.ORDEREXISTS;
        }
        
      //   ----  
        private void createOrderRight() {
            throw Exceptions.orderExists();
        }
    
        private void cancelOrderRight() {
            throw Exceptions.orderExists();
        }
    
    public class Exceptions {
    
        public static BusinessException ORDEREXISTS = new BusinessException("订单已经存在", 3001);
    
        public static BusinessException orderExists() {
            return new BusinessException("订单已经存在", 3001);
        }
    }
    ```

- 提交线程池的任务出了异常会怎么样？

  - 线程池常用作异步处理或并行处理。那么，把任务提交到线程池处理，任务本身出现异常时会怎样呢？

    - 异常的抛出老线程退出了，线程池只能重新创建一个线程。如果每个异步任务都以异常结束，那么线程池可能完全起不到线程重用的作用

  - 因为没有手动捕获异常进行处理，显然，这种没有以统一的错误日志格式记录错误信息打印出来的形式，对生产级代码是不合适的

    - ```java
      // 设置自定义的异常处理程序作为保底，比如在声明线程池时自定义线程池的未捕获异常处理程序
      ExecutorService threadPool = Executors.newFixedThreadPool(1, new ThreadFactoryBuilder()
                      .setNameFormat(prefix + "%d")
                      .setUncaughtExceptionHandler((thread, throwable) -> log.error("ThreadPool {} got exception", thread, throwable))
                      .get());
                      
      // 或者设置全局的默认未捕获异常处理程序：
      	    static {
              Thread.setDefaultUncaughtExceptionHandler((thread, throwable) -> log.error("Thread {} got exception", thread, throwable));
          }
      
      ```

  - 通过线程池 ExecutorService 的 execute 方法提交任务到线程池处理，如果出现异常会导致线程退出，控制台输出中可以看到异常信息。那么，把 execute 方法改为 submit，线程还会退出吗，异常还能被处理程序捕获到吗？

    - 修改代码后重新执行程序，线程没退出，异常也没记录被生吞了
    - 查看 FutureTask 源码可以发现，在执行任务出现异常之后，异常存到了一个 outcome 字段中，只有在调用 get 方法获取 FutureTask 结果的时候，才会以 ExecutionException 的形式重新抛出异常

- 关于在 finally 代码块中抛出异常的坑，如果在 finally 代码块中返回值，你觉得程序会以 try 或 catch 中返回值为准，还是以 finally 中的返回值为准呢？

  - JVM采用异常表控制try-catch的跳转逻辑；
  - 对于finally中的代码块其实是复制到try和catch中的return和throw之前的方式来处理的。





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

  -  EvaluatorFilter（求值过滤器），用于判断日志是否符合某个条件

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

-  生产级项目的文件日志肯定需要按时间和日期进行分割和归档处理，以避免单个文件太大，同时保留一定天数的历史日志，你知道如何配置吗？

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

    



## 文件IO

- 文件读写需要确保字符编码一致

  - UTF-8 编码的“你好”的十六进制是 E4BDA0E5A5BD，每一个汉字需要三个字节；而 GBK 编码的汉字，每一个汉字两个字节。字节长度都不一样，以 GBK 编码后保存的汉字，以 UTF8 进行解码读取，必然不会成功。
  - 修复：直接使用 FileInputStream 拿文件流，然后使用 InputStreamReader 读取字符流，并指定字符集为 GBK
  - 如果你觉得这种方式比较麻烦的话，使用 JDK1.7 推出的 Files 类的 readAllLines 方法，可以很方便地用一行代码完成文件内容读取
    - 但这种方式有个问题是，读取超出内存大小的大文件时会出现 OOM
    - 打开 readAllLines 方法的源码可以看到，readAllLines 读取文件所有内容后，放到一个List<String> 中返回，如果内存无法容纳这个 List，就会 OOM
    - File 类的 lines 方法：实现按需的流式读取。比如，需要消费某行数据时再读取，而不是把整个文件一次性读取到内存

- 使用 Files 类静态方法进行文件操作注意释放文件句柄

  - 读取完文件后没有关闭。我们通常会认为静态方法的调用不涉及资源释放，因为方法调用结束自然代表资源使用完成，由 API 释放资源，但对于 Files 类的一些返回 Stream的方法并不是这样。这，是一个很容易被忽略的严重问题
  - 在JDK 文档中有提到，注意使用 try-with-resources 方式来配合，确保流的close 方法可以调用释放资源。
  - 修复方式很简单，使用 try 来包裹 Stream 即可

- 注意读写文件要考虑设置缓冲区

  - ```java
    // 在使用BufferedInputStream 和 BufferedOutputStream 时，我还是建议你再使用一个缓冲进行读写，不要因为它们实现了内部缓冲就进行逐字节的操作
    byte[] buffer = new byte[100]; 
    int len = 0; 
    while ((len = fileInputStream.read(buffer)) != -1) { 
    	fileOutputStream.write(buffer, 0, len);
    ```

- Java 的 File 类和 Files 类提供的文件复制、重命名、删除等操作，不是原子性的

- Files.lines 方法进行流式处理，需要使用 try-with-resources 进行资源释放。那么，使用 Files 类中其他返回 Stream 包装对象的方法进行流式处理，比如newDirectoryStream 方法返回 DirectoryStream<Path>，list、walk 和 find 方法返回 Stream<Path>，也同样有资源释放问题吗

  - 都间接实现了autoCloseable接口，所以都可以使用try-with-resources进行释放。



