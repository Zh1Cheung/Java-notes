## 序列化

- 序列化是把对象转换为字节流的过程，以方便传输或存储。
- Spring 提供的 4 种 RedisSerializer
  - 默认情况下，RedisTemplate 使用 JdkSerializationRedisSerializer，也就是 JDK 序列化，容易产生 Redis 中保存了乱码的错觉。 StringRedisTemplate 对于 Key 和 Value，使用的是 String 序列化方式
    - 使用 RedisTemplate 读出的数据，由于是 Object 类型的，使用时可以先强制转换为 User 类型； 
    - 使用 StringRedisTemplate 读取出的字符串，需要手动将 JSON 反序列化为 User 类 型。 
    - StringRedisTemplate 和 RedisTemplate，使用这两种方式存取的数据完全无法通用。 
  - 通常考虑到易读性，可以设置 Key 的序列化器为 StringRedisSerializer。但直接使用RedisSerializer.string()，相当于使用了 UTF_8 编码的 StringRedisSerializer，需要注意字符集问题。
  - 如果希望 Value 也是使用 JSON 序列化的话，可以把 Value 序列化器设置为Jackson2JsonRedisSerializer。默认情况下，不会把类型信息保存在 Value 中，即使我们定义 RedisTemplate 的 Value 泛型为实际类型，查询出的 Value 也只能是LinkedHashMap 类型。如果希望直接获取真实的数据类型，你可以启用 JacksonObjectMapper 的 activateDefaultTyping 方法，把类型信息一起序列化保存在 Value中。
  - 如果希望 Value 以 JSON 保存并带上类型信息，更简单的方式是，直接使用RedisSerializer.json() 快捷方法来获取序列化器。
- 注意 Jackson JSON 反序列化对额外字段的处理
  - 自定义 ObjectMapper 启用 WRITE_ENUMS_USING_INDEX 序列化功能特性时，覆盖了 Spring Boot 自动创建的 ObjectMapper；而这个自动创建的ObjectMapper 设置过 FAIL_ON_UNKNOWN_PROPERTIES 反序列化特性为 false，以确保出现未知字段时不要抛出异常
  - 要修复这个问题，有三种方式
    - 第一种，同样禁用自定义的 ObjectMapper 的 FAIL_ON_UNKNOWN_PROPERTIES
    - 第二种，设置自定义类型，加上 @JsonIgnoreProperties 注解，开启 ignoreUnknown属性，以实现反序列化时忽略额外的数据
    - 第三种，不要自定义 ObjectMapper，而是直接在配置文件设置相关参数，来修改Spring 默认的 ObjectMapper 的功能
  - 忽略多余字段，是我们写业务代码时最容易遇到的一个配置项。Spring Boot 在自动配置时贴心地做了全局设置。如果需要设置更多的特性，可以直接修改配置文件spring.jackson.** 或设置 Jackson2ObjectMapperBuilderCustomizer 回调接口，来启用更多设置，无需重新定义 ObjectMapper Bean
- 反序列化时要小心类的构造方法
  - 默认情况下，在反序列化的时候，Jackson 框架只会调用无参构造方法创建对象。如果走自定义的构造方法创建对象，需要通过 @JsonCreator 来指定构造方法，并通过 @JsonProperty 设置构造方法中参数对应的 JSON 属性名
- 枚举作为 API 接口参数或返回值的两个大坑
  - 对于枚举，我建议尽量在程序内部使用，而不是作为 API 接口的参数或返回值，原因是枚举涉及序列化和反序列化时会有两个大坑。
  - 第一个坑是，客户端和服务端的枚举定义不一致时，会出异常。比如，客户端版本的枚举定
    义了 4 个枚举值，服务端定义了 5 个枚举值
    - 要解决这个问题，可以开启 Jackson 的
      read_unknown_enum_values_using_default_value 反序列化特性，也就是在枚举值未知
      的时候使用默认值
    - 并为枚举添加一个默认值，使用 @JsonEnumDefaultValue 注解注释
    - 需要注意的是，这个枚举值一定是添加在客户端 类中的，因为反序列化使用的是客户端枚举
    - 仅仅这样配置还不能让 RestTemplate 生效这个反序列化特性，还需要配置 RestTemplate，来使用 Spring Boot 的MappingJackson2HttpMessageConverter 才行
  - 第二个坑，也是更大的坑，枚举序列化反序列化实现自定义的字段非常麻烦，会涉及Jackson 的 Bug
- 使用RedisTemplate<String, Long> 能否存取 Value 是 Long的数据呢？这其中有什么坑吗？
  - 在Integer区间内返回的是Integer，超过这个区间返回Long





## OOM

- 太多份相同的对象导致 OOM

  - 我们虽然清楚数据总量，但却忽略了每一份数据在内存中可能有多份
    - 100M 的数据加载到程序内存中，变为 Java 的数据结构就已经占用了 200M 堆内存；这些数据经过 JDBC、MyBatis 等框架其实是加载了 2 份，然后领域模型、DTO 再进行转换可能又加载了 2 次；最终，占用的内存达到了 200M*4=800M。
  - 使用 WeakHashMap 不等于不会 OOM
    - WeakHashMap和 HashMap 的最大区别，是 Entry 对象的实现。
    - Entry 对象继承了 WeakReference，Entry 的构造函数调用了 super (key,queue)，这是父类的构造函数。其中，key 是我们执行 put 方法时的 key；queue 是一个ReferenceQueue，被 GC 的对象会被丢进这个 queue里面。
    - 每次调用 get、put、size 等方法时，都会从 queue 里拿出所有已经被 GC 掉的 key 并删除对应的 Entry 对象
    - WeakHashMap 的 Key 虽然是弱引用，但是其 Value 却持有 Key 中对象的强引用，Value 被 Entry 引用，Entry 被 WeakHashMap 引用，最终导致 Key 无法回收。解决方案就是让 Value 变为弱引用
  - Tomcat 参数配置不合理导致 OOM

- > 我建议你为生产系统的程序配置 JVM 参数启用详细的 GC 日志，方便观察垃圾收集器的行为，并开启HeapDumpOnOutOfMemoryError，以便在出现 OOM 时能自动Dump 留下第一问题现场。
  >  XX:+HeapDumpOnOutOfMemoryError 
  > -XX:HeapDumpPath=. 
  > -XX:+PrintGCDateStamps
  > -XX:+PrintGCDetails





##  反射、注解和泛型

- 反射调用方法不是以传参决定重载

  - 反射的功能包括，在运行时动态获取类和类成员定义，以及动态读取属性调用方法。也就是说，针对类动态调用方法，不管类中字段和方法怎么变动，我们都可以用相同的规则来读取信息和执行方法。因此，几乎所有的 ORM（对象关系映射）、对象映射、MVC 框架都使用了反射。
  - getDeclaredMethod 传入的参数类型 Integer.TYPE 代表的是 int，所以实际执行方法时，无论传的是包装类型还是基本类型，都会调用 int 入参的 age 方法。
  - 把 Integer.TYPE 改为 Integer.class，执行的参数类型就是包装类型的 Integer。这时，无论传入的是 Integer.valueOf(“36”) 还是基本类型的 36

- 泛型经过类型擦除多出桥接方法的坑

  - 子类没有指定 String 泛型参数，父类的泛型方法 setValue(T value) 在泛型擦除后是 setValue(Object value)，子类中入参是 String 的 setValue 方法被当作了新方法；

  - 使用 javap 命令来反编译编译后的 Child类的 class 字节码

    - 如果子类 Child 的 setValue 方法要覆盖父类的 setValue 方法，那 入参也必须是 Object。所以，编译器会为我们生成一个所谓的 bridge 桥接方法

    - 入参为 Object 的 setValue 方法在内部调用了入参为 String 的 setValue 方法

    - 如果编译器没有帮我们实现这个桥接方法，那么 Child 子类重写的是父类经过泛型类型擦除后、入参是 Object 的 setValue 方法。这两个方法的参数，一个是 String 一个是 Object，明显不符合 Java 的语义

    - 使用 jclasslib 工具打开 Child类，同样可以看到入参为 Object 的桥接方法上标记了public + synthetic + bridge 三个属性。synthetic 代表由编译器生成的不可见代码，bridge 代表这是泛型类型擦除后生成的桥接代码

    - 通过 getDeclaredMethods 方法获取到所有方法后，必须同时根据方法名 setValue 和 

      非 isBridge 两个条件过滤

  - getMethods 和 getDeclaredMethods 是有区别的，前者可以查询到父类方法，后者只能查询到当前类。

- 在注解上标记 @Inherited 元注解可以实现注解的继承

  - 子类可以获得父类类上的注解；子类 foo 方法虽然是重写父类方法，并且注解本身也支持继承，但还是无法获得方法上的注解
  -  AnnotatedElementUtils 类，来方便我们处理注解的继承问题
    - findMergedAnnotation 工具方法，可以帮助我们找出父类和接口、父类方法和接口方法上的注解，并可以处理桥接方法，实现一键找到继承链的注解

- 泛型类型擦除后会生成一个 bridge 方法，这个方法同时又是 synthetic 方法。除了泛型类型擦除，你知道还有什么情况编译器会生成 synthetic 方法吗？

  - 编译器通过生成一些在源代码中不存在的synthetic方法和类的方式，实现了对private级别的字段和类的访问，从而绕开了语言限制
  - 如果同时用到了Enum和switch，如先定义一个enum枚举，然后用switch遍历这个枚举，java编译器会偷偷生成一个synthetic的数组，数组内容是enum的实例。

- @Controller

  - 子类XXXController 继承了YYYController 连带requestMapping 也设为一样，requestMapping 可以理解为请求派发的命名空间，相同的空间里，子类享有父类protected及其以上的访问权限，当容器启动做方法请求映射时，就会发现父类的一般请求方法，全部都没有明确派发地址了。





##  Spring

- 如果以容器为依托来管理所有的框架、业务对象，我们不仅可以无侵入地调整对象的关系，还可以无侵入地随时调整对象的属性，甚至是实现对象的替换。

- 单例的 Bean 如何注入 Prototype 的 Bean

  - 定义了这么一个SayService 抽象类，其中维护了一个类型是 ArrayList 的字段 data，用于保存方法处理的中间数据。每次调用 say 方法都会往 data 加入新数据，可以认为 SayService 是有状态，如果 SayService 是单例的话必然会 OOM

    - 正确的方式是，在为类标记上 @Service 注解把类型交由容器管理前，首先评估一下类是否有状态，然后为 Bean 设置合适的 Scope
    - 但，上线后还是出现了内存泄漏，证明修改是无效的。

  - Bean 默认是单例的，所以单例的 Controller 注入的 Service 也是一次性创建的，即使Service 本身标识了 prototype 的范围也没用。

    - 修复方式是，让 Service 以代理方式注入。这样虽然 Controller 本身是单例的，但每次都能从代理获取 Service。这样一来，prototype 范围的配置才能真正生效

    - ```java
      @Scope(value = WebApplicationContext.SCOPE_SESSION,proxyMode = ScopedProxyMode.INTERFACES)
      ```

    - 当然，如果不希望走代理的话还有一种方式是，每次直接从 ApplicationContext 中获取Bean

- 监控切面因为顺序问题导致 Spring 事务失效

  - 我们知道，切面本身是一个 Bean，Spring 对不同切面增强的执行顺序是由 Bean 优先级决定的，具体规则是
    - 入操作（Around（连接点执行前）Before），切面优先级越高，越先执行。一个切面的入操作执行完，才轮到下一切面，所有切面入操作执行完，才开始执行连接点（方法）。
    - 出操作（Around（连接点执行后）、After、AfterReturning、AfterThrowing），切面优先级越低，越先执行。一个切面的出操作执行完，才轮到下一切面，直到返回到调用点。
    - 同一切面的 Around 比 After、Before 先执行。
    - 对于 Bean 可以通过 @Order 注解来设置优先级，值越大优先级反而越低

- Spring 程序配置的优先级问题

  - Spring 通过环境 Environment 抽象出的 Property 和 Profile
    - 针对 Property，又抽象出各种 PropertySource 类代表配置源。一个环境下可能有多个配置源，每个配置源中有诸多配置项。在查询配置信息时，需要按照配置源优先级进行查询。
    - Profile 定义了场景的概念。通常，我们会定义类似 dev、test、stage 和 prod 等环境作为不同的 Profile
  - 对于非 Web 应用，Spring 对于 Environment 接口的实现是 StandardEnvironment 类
    - MutablePropertySources 类型的字段 propertySources，看起来代表了所有配置源；
      - propertySourceList 字段用来真正保存 PropertySource 的 List，且这个 List 是一个CopyOnWriteArrayList
      - 类中定义了 addFirst、addLast、addBefore、addAfter 等方法，来精确控制PropertySource 加入 propertySourceList 的顺序。这也说明了顺序的重要性。
    - getProperty 方法，通过 PropertySourcesPropertyResolver 类进行查询配置；
    - 实例化 PropertySourcesPropertyResolver 的时候，传入了当前的MutablePropertySources。





## 代码重复

- 利用工厂模式 + 模板方法模式，消除 if…else 和重复代码
  - 模板方法模式
    - 在父类中实现了购物车处理的流程模板，然后把需要特殊处理的地方留空白也就是留抽象方法定义，让子类去实现其中的逻辑。由于父类的逻辑不完整无法单独工作，因此需要定义为抽象类。
  - 工厂模式	
    - 定义三个购物车子类时，我们在 @Service 注解中对 Bean 进行了命名。既然三个购物车都叫 XXXUserCart，那我们就可以把用户类型字符串拼接 UserCart构成购物车 Bean 的名称，然后利用 Spring 的 IoC 容器，通过 Bean 的名称直接获取到AbstractCart，调用其 process 方法即可实现通用
    - 其实，这就是工厂模式，只不过是借助 Spring 容器实现罢了
  - 观察者模式是一种很常见的解耦方式，多数应用在了事件发布订阅这种业务场景下，有名的当属guava的EventBus了。
- 利用注解 + 反射消除重复代码
  - 许多涉及类结构性的通用处理，都可以按照这个模式来减少重复代码
  - 反射配合注解实现动态的接口参数组装
- 利用属性拷贝工具消除重复代码
  - 对于三层架构的系统，考虑到层之间的解耦隔离以及每一层对数据的不同需求，通常每一层都会有自己的 POJO 作为数据实体。比如，数据访问层的实体一般叫作 DataObject 或DO，业务逻辑层的实体一般叫作 Domain，表现层的实体一般叫作 Data Transfer Object或 DTO
  - 可以使用类似 BeanUtils 这种 Mapping 工具来做 Bean 的转换，copyProperties 方法还允许我们提供需要忽略的属性
  - 对于属性的copy，无论是spring，guava，apache commons都有涉及，hutool支持各种参数来调整属性的拷贝





## 接口设计

- 为了简化服务端代码，我们可以把包装 API 响应体 APIResponse的工作交由框架自动完成，这样直接返回 DTO即可。对于业务逻辑错误，可以抛出一个自定义异常

  - 在 APIException 中包含错误码和错误消息

    - ```java
      public class APIException extends RuntimeException
      ```

  - 然后，定义一个 @RestControllerAdvice 来完成自动包装响应体的工作

    - ```java
      // 通过实现 ResponseBodyAdvice 接口的 beforeBodyWrite 方法，来处理成功请求的响应体转换
      // 自动包装外层APIResposne响应
          @Override
          public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
              APIResponse apiResponse = new APIResponse();
              apiResponse.setSuccess(true);
              apiResponse.setMessage("OK");
              apiResponse.setCode(2000);
              apiResponse.setData(body);
              return apiResponse;
          }
      }
      
      // 实现一个 @ExceptionHandler 来处理业务异常时，APIException 到 APIResponse 的转换。
      // 自动处理APIException，包装为APIResponse
          @ExceptionHandler(APIException.class)
          public APIResponse handleApiException(HttpServletRequest request, APIException ex) {
              log.error("process url {} failed", request.getRequestURL().toString(), ex);
              APIResponse apiResponse = new APIResponse();
              apiResponse.setSuccess(false);
              apiResponse.setCode(ex.getErrorCode());
              apiResponse.setMessage(ex.getErrorMessage());
              return apiResponse;
          }
          
      ```

      



## 缓存

- 不要把 Redis 当作数据库
  - Redis 的特点是，处理请求很快，但无法保存超过内存大小的数据
  - 常用的数据淘汰策略
    - 其实，这些算法是 Key 范围 +Key 选择算法的搭配组合，其中范围有 allkeys 和 volatile 两种，算法有 LRU、TTL 和 LFU 三种
    - 首先，从算法角度来说，Redis 4.0 以后推出的 LFU 比 LRU 更“实用”。试想一下，如果一个 Key 访问频率是 1 天一次，但正好在 1 秒前刚访问过，那么 LRU 可能不会选择优先淘汰这个 Key，反而可能会淘汰一个 5 秒访问一次但最近 2 秒没有访问过的 Key，而 LFU 算法不会有这个问题。而 TTL 会比较“头脑简单”一点，优先删除即将过期的 Key，但有可能这个 Key 正在被大量访问。
    - 然后，从 Key 范围角度来说，allkeys 可以确保即使 Key 没有 TTL 也能回收，如果使用的时候客户端总是“忘记”设置缓存的过期时间，那么可以考虑使用这个系列的算法。而 volatile 会更稳妥一些，万一客户端把 Redis 当做了长效缓存使用，只是启动时候初始化一次缓存，那么一旦删除了此类没有 TTL 的数据，可能就会导致客户端出错。
- 注意缓存雪崩问题
  - 从广义上说，产生缓存雪崩的原因有两种
    - 第一种是，缓存系统本身不可用，导致大量请求直接回源到数据库；
    - 第二种是，应用设计层面大量的 Key 在同一时间过期，导致大量的数据回源。
  - 如何确保大量 Key 不在同一时间被动过期
    - 差异化缓存过期时间，不要让大量的 Key 在同一时间过期。比如，在初始化缓存的时候，设置缓存的过期时间是 30 秒 +10 秒以内的随机延迟（扰动值）。这样，这些Key 不会集中在 30 秒这个时刻过期，而是会分散在 30~40 秒之间过期
- 注意缓存击穿问题
  - 在某些 Key 属于极端热点数据，且并发量很大的情况下，如果这个 Key 过期，可能会在某个瞬间出现大量的并发请求同时回源，相当于大量的并发请求直接打到了数据库。
  - 使用 Redisson 来获取一个基于 Redis 的分布式锁，在查询数据库之前先尝试获取锁，这样，可以把回源到数据库的并发限制在 1
  - 在真实的业务场景下，不一定要这么严格地使用双重检查分布式锁进行全局的并发限制，因为这样虽然可以把数据库回源并发降到最低，但也限制了缓存失效时的并发。可以考虑的方式是
    - 方案一，使用进程内的锁进行限制，这样每一个节点都可以以一个并发回源数据库；
    - 方案二，不使用锁进行限制，而是使用类似 Semaphore 的工具限制并发数，比如限制为 10，这样既限制了回源并发数不至于太大，又能使得一定量的线程可以同时回源。
- 注意缓存穿透问题
  - 缓存中没有数据不一定代表数据没有缓存，还有一种可能是原始数据压根就不存在
  - 解决缓存穿透有以下两种方案
    - 方案一，对于不存在的数据，同样设置一个特殊的 Value 到缓存中，比如当数据库中查出的用户信息为空的时候，设置 NODATA 这样具有特殊含义的字符串到缓存中。这样下次请求缓存的时候还是可以命中缓存，即直接从缓存返回结果，不查询数据库
    - 方案二，即使用布隆过滤器做前置过滤
      - 布隆过滤器是一种概率型数据库结构，由一个很长的二进制向量和一系列随机映射函数组成。它的原理是，当一个元素被加入集合时，通过 k 个散列函数将这个元素映射成一个 m位 bit 数组中的 k 个点，并置为 1。
      - 要用上布隆过滤器，我们可以使用 Google 的 Guava 工具包提供的 BloomFilter 类改造一
        下程序：启动时，初始化一个具有所有有效用户 ID 的、10000 个元素的 BloomFilter，在
        从缓存查询数据之前调用其 mightContain 方法，来检测用户 ID 是否可能存在；如果布隆
        过滤器说值不存在，那么一定是不存在的
- 注意缓存数据同步策略
  - Cache Aside 更新模式
  - 前面提到的 3 个案例，其实都属于缓存数据过期后的被动删除。在实际情况下，修改了原始数据后，考虑到缓存数据更新的及时性，我们可能会采用主动更新缓存的策略
  - 先更新缓存，再更新数据库；
    - “先更新缓存再更新数据库”策略不可行。数据库设计复杂，压力集中，数据库因为超时等原因更新操作失败的可能性较大，此外还会涉及事务，很可能因为数据库更新失败，导致缓存和数据库的数据不一致。
  - 先更新数据库，再更新缓存；
    - “先更新数据库再更新缓存”策略不可行。一是，如果线程 A 和 B 先后完成数据库更新，但更新缓存时却是 B 和 A 的顺序，那很可能会把旧数据更新到缓存中引起数据不一致；二是，我们不确定缓存中的数据是否会被访问，不一定要把所有数据都更新到缓存中去。
  - 先删除缓存，再更新数据库，访问的时候按需加载数据到缓存；
    - “先删除缓存再更新数据库，访问的时候按需加载数据到缓存”策略也不可行。在并发的情况下，很可能删除缓存后还没来得及更新数据库，就有另一个线程先读取了旧值到缓存中，如果并发量很大的话这个概率也会很大。
  - 先更新数据库，再删除缓存，访问的时候按需加载数据到缓存。
    - “先更新数据库再删除缓存，访问的时候按需加载数据到缓存”策略是最好的。
    - 这种做法其实不能算是坑，在实际的系统中也推荐使用这种方式。但是这种方式理论上还是可能存在问题。以Redis和Mysql为例，查询操作没有命中缓存，然后查询出数据库的老数据。此时有一个并发的更新操作，更新操作在读操作之后更新了数据库中的数据并且删除了缓存中的数据。然而读操作将从数据库中读取出的老数据更新回了缓存。这样就会造成数据库和缓存中的数据不一致，应用程序中读取的都是原来的数据（脏数据）
    - 但是，仔细想一想，这种并发的概率极低。因为这个条件需要发生在读缓存时缓存失效，而且有一个并发的写操作。实际上数据库的写操作会比读操作慢得多，而且还要加锁，而读操作必需在写操作前进入数据库操作，又要晚于写操作更新缓存，所有这些条件都具备的概率并不大。但是为了避免这种极端情况造成脏数据所产生的影响，我们还是要为缓存设置过期时间。
    - 需要注意的是，更新数据库后删除缓存的操作可能失败，如果失败则考虑把任务加入延迟队列进行延迟重试，确保数据可以删除，缓存可以及时更新。因为删除操作是幂等的，所以即使重复删问题也不是太大，这又是删除比更新好的一个原因。
  - Write Behind Caching 更新模式
    - Write Behind Caching 更新模式就是在更新数据的时候，只更新缓存，不更新数据库，而我们的缓存会异步地批量更新数据库。这个设计的好处就是直接操作内存速度快。因为异步，Write Behind Caching 更新模式还可以合并对同一个数据的多次操作到数据库，所以性能的提高是相当可观的。
    - 但其带来的问题是，数据不是强一致性的，而且可能会丢失。另外，Write Behind Caching 更新模式实现逻辑比较复杂，因为它需要确认有哪些数据是被更新了的，哪些数据需要刷到持久层上。只有在缓存需要失效的时候，才会把它真正持久起来。



