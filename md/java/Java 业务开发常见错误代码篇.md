## 并发工具类库

- 没有意识到线程重用导致用户信息错乱的 Bug

  - ThreadLocal 适用于变量在线程间隔离，而在方法或类间共享的场景
  - 程序运行在 Tomcat 中，执行程序的线程是 Tomcat 的工作线程，而 Tomcat 的工作线程是基于线程池的。顾名思义，线程池会重用固定的几个线程，一旦线程重用，那么很可能首次从 ThreadLocal 获取的值是之前其他用户的请求遗留的值。这时，ThreadLocal 中的用户信息就是其他用户的信息。
  - 这个例子告诉我们，在写业务代码时，首先要理解代码会跑在什么线程上
    - 在 Tomcat 这种 Web 服务器下跑的业务代码，本来就运行在一个多线程环境（否则接口也不可能支持这么高的并发），并不能认为没有显式开启多线程就不会有线程安全问题。
    - 理解了这个知识点后，我们修正这段代码的方案是，在代码的 finally 代码块中，显式清除ThreadLocal 中的数据（remove()）。这样一来，新的请求过来即使使用了之前的线程也不会获取到错误的用户信息了。

- 使用了线程安全的并发工具，并不代表解决了所有线程安全问题

  - “线程安全”这四个字特别容易让人误解，因为 ConcurrentHashMap 只能保证提供的原子性读写操作是线程安全的。

  - 我们需要注意 ConcurrentHashMap 对外提供的方法或能力的限制

    - 诸如 size、isEmpty 和 containsValue 等聚合方法，在并发情况下可能会反映ConcurrentHashMap 的中间状态。因此在并发情况下，这些方法的返回值只能用作参考，而不能用于流程控制。显然，利用 size 方法计算差异值，是一个流程控制。

    - 诸如 putAll 这样的聚合方法也不能确保原子性，在 putAll 的过程中去获取数据可能会获取到部分数据。

    - ```java
      // 代码的修改方案很简单，整段逻辑加锁即可     
      synchronized (concurrenthashMap){
                  int gap=concurrenthashMap.size();
                  concurrenthashMap.putAll(getData(gap));
              }
      ```

  - 有一个含 900 个元素的 Map，现在再补充 100 个元素进去，这个补充操作由 10 个线程并发进行。开发人员误以为使用了 ConcurrentHashMap 就不会有线程安全问题，于是不加思索地写出了下面的代码：在每一个线程的代码逻辑中先通过 size 方法拿到当前元素数量，计算 ConcurrentHashMap 目前还需要补充多少元素，并在日志中输出了这个值，然后通过 putAll 方法把缺少的元素添加进去。

    - ConcurrentHashMap 这个篮子本身，可以确保多个工人在装东西进去时，不会相互影响干扰，但无法确保工人 A 看到还需要装 100 个桔子但是还未装的时候，工人 B 就看不到篮子中的桔子数量。更值得注意的是，你往这个篮子装 100 个桔子的操作不是原子性的，在别人看来可能会有一个瞬间篮子里有 964 个桔子，还需要补 36 个桔子。

    - ```java
      // IntStream boxed（）返回一个由该流元素组成的Stream，每个元素装箱为一个Integer。
      // IntStream rangeClosed（int startInclusive，int endInclusive）以增量步长1返回一个从startInclusive（包括）到endInclusive（包括）的IntStream
      
      // Function是一个接口，Java 8允许在接口中加入具体方法。接口中的具体方法有两种，default方法和static方法，identity()就是Function接口的一个静态方法。Function.identity()返回一个输出跟输入一样的Lambda表达式对象，等价于形如t -> t形式的Lambda表达式。
      // int[] arrayProblem = list.stream().mapToInt(Function.identity()).toArray();
      // 运行的时候就会错误，因为mapToInt要求的参数是ToIntFunction类型，但是ToIntFunction类型和Function没有关系
      
      private ConcurrentHashMap<String, Long> getData(int count){
              return LongStream.rangeClosed(1,count)
                      .boxed()
                      .collect(Collectors.toConcurrentMap(i -> UUID.randomUUID().toString(), Function.identity(),
              (o1,o2)->o1,ConcurrentHashMap::new));
          }
      ```

- 没有充分了解并发工具的特性，从而无法发挥其威力

  - AtomicLong 的缺陷

    - AtomicLong 的 Add() 是依赖自旋不断的 CAS 去累加一个 Long 值。如果在竞争激烈的情况下，CAS 操作不断的失败，就会有大量的线程不断的自旋尝试 CAS 会造成 CPU 的极大的消耗

  - LongAdder 解决方案

    - AtomicLong 只对一个 Long 值进行 CAS 操作。而 LongAdder 是针对 Cell 数组的某个 Cell 进行 CAS 操作 ，把线程的名字的 hash 值，作为 Cell 数组的下标，然后对 Cell[i] 的 long 进行 CAS 操作。简单粗暴的分散了高并发下的竞争压力。
    - 其实一个 Cell 的本质就是一个 volatile 修饰的 long 值，且这个值能够进行 cas 操作
    - 其实我们可以发现，LongAdder 使用了一个 cell 列表去承接并发的 cas，以提升性能，但是 LongAdder 在统计的时候如果有并发更新，可能导致统计的数据有误差。

  - 

  - ```java
    // 使用 ConcurrentHashMap 的原子性方法 computeIfAbsent 来做复合逻辑操作，判断Key 是否存在 Value，如果不存在则把 Lambda 表达式运行后的结果放入 Map 作为Value，也就是新创建一个 LongAdder 对象，最后返回 Value。
    // 由于 computeIfAbsent 方法返回的 Value 是 LongAdder，是一个线程安全的累加器，因此可以直接调用其 increment 方法进行累加。
    ConcurrentHashMap<String, LongAdder> freqs=new ConcurrentHashMap<>();
    ForkJoinPool forkJoinPool =new ForkJoinPool(10);
    forkJoinPool.execute(()-> IntStream.rangeClosed(1,1000000).parallel().forEach(i->{
                String key="item"+ ThreadLocalRandom.current().nextInt(1000);
                //利用computeIfAbsent()方法来实例化LongAdder，然后利用LongAdder来进行线程安全计数
                freqs.computeIfAbsent(key, k -> new LongAdder()).increment();
            }));
            forkJoinPool.shutdown();
            forkJoinPool.awaitTermination(1, TimeUnit.HOURS);
            return freqs.entrySet().stream()
                    .collect(Collectors.toMap(
                            e->e.getKey(),
                            e->e.getValue().longValue())
                    );
    ```

- 没有认清并发工具的使用场景，因而导致性能问题

  - 在 Java 中，CopyOnWriteArrayList 虽然是一个线程安全的 ArrayList，但因为其实现方式是，每次修改数据时都会复制一份数据出来，所以有明显的适用场景，即读多写少或者说希望无锁读的场景。

- ThreadLocalRandom是否可以把它的实例设置到静态变量中，在多线程情况下重用呢？

  - ThreadLocalRandom中存放一个单例的instance，调用current()方法返回这个instance，每个线程首次调用current()方法时，会在各个线程中初始化seed和probe。
  - nextX(）方法会调用nextSeed()，在其中使用各个线程中的种子，计算下一个种子并保存（UNSAFE.getLong(t, SEED) + GAMMA）。
  - 所以，如果使用静态变量，直接调用nextX()方法就跳过了各个线程初始化的步骤，只会在每次调用nextSeed()时来更新种子。

- computeIfAbsent 和 putIfAbsent 方法的区别

  - 当Key存在的时候，如果Value获取比较昂贵的话，putIfAbsent就白白浪费时间在获取这个昂贵的Value上（这个点特别注意）
  - Key不存在的时候，putIfAbsent返回null，小心空指针，而computeIfAbsent返回计算后的值
  - 当Key不存在的时候，putIfAbsent允许put null进去，而computeIfAbsent不能（当然了，此条针对HashMap，ConcurrentHashMap不允许put null value进去）

- ConcurrentHashMap 只能保证提供的原子性读写操作是线程安全的

  - 线程安全是指多线程访问的操作ConcurrentHashMap，并不会出现状态不一致，数据错乱，异常等问题。
  - 第一，ConcurrentHashMap提供的那些针对单一Key读写的API可以认为是线程安全的，但是诸如putAll这种涉及到多个Key的操作，并发读取可能无法确保读取到完整的数据。
  - 第二，ConcurrentHashMap只能确保提供的API是线程安全的，但是使用者组合使用多个API，ConcurrentHashMap无法从内部确保使用过程中的状态一致。

  

  

## 代码加锁

- 没有分析清线程、业务逻辑和锁三者之间的关系

  - a<b 这种比较操作在字节码层面是加载 a、加载 b 和比较三步，代码虽然是一行但也不是原子性的。

- 加锁前要清楚锁和被保护的对象是不是一个层面的

  - 静态字段属于类，类级别的锁才能保护；而非静态字段属于类实例，实例级别的锁就可以保护
  - 在非静态的 wrong 方法上加锁，只能确保多个线程无法执行同一个实例的 wrong 方法，却不能保证不会执行不同实例的 wrong 方法。而静态的 counter 在多个实例中共享，所以必然会出现线程安全问题。

- 加锁要考虑锁的粒度和场景问题

  - 在业务代码中，有一个 ArrayList 因为会被多个线程操作而需要保护，又有一段比较耗时的操作（代码中的 slow 方法）不涉及线程安全问题，应该如何加锁呢？
    - 把加锁的粒度降到最低，只在操作 ArrayList 的时候给这个 Array
  - 如果精细化考虑了锁应用范围后，性能还无法满足需求的话，我们就要考虑另一个维度的粒度问题了，即：区分读写场景以及资源的访问冲突，考虑使用悲观方式的锁还是乐观方式的锁。
    - JDK 里 ReentrantLock 和 ReentrantReadWriteLock 都提供了公平锁的版本，在没有明确需求的情况下不要轻易开启公平锁特性，在任务很轻的情况下开启公平锁可能会让性能下降上百倍。

- 多把锁要小心死锁问题

  - 避免死锁的方案很简单，为购物车中的商品排一下序，让所有的线程一定是先获取item1 的锁然后获取 item2 的锁，就不会有问题了。

    - ```java
      private List<Item> createCart() { 
       	return IntStream.rangeClosed(1, 3) 
       	.mapToObj(i -> "item" + ThreadLocalRandom.current().nextInt(items.size()))
       	.map(name -> items.get(name)).collect(Collectors.toList()); 
      }
      
      
      IntStream.rangeClosed(1, 100).parallel()
                  .mapToObj(i -> {
                      List<Item> cart = createCart().stream()
                              .sorted(Comparator.comparing(Item::getName))
                              .collect(Collectors.toList());
                      return createOrder(cart);});
      ```

- 可见性问题和禁止指令重排序优化

  - 可见性问题：本质上是cpu缓存失效，必须从主内存读取数据；
  - 禁止指令重排序优化：x86处理器下，只实现了volatile的读写内存屏障，也就是store load，也就是写读，本质上也就是读写可见性，happen-before原则

- 两个坑，一是加锁和释放没有配对的问题，二是锁自动释放导致的重复逻辑执行的问题

  - 锁建议使用synchronized
  - 加锁解锁没有配对可以用一些代码质量工具协助排插，如Sonar，集成到ide和代码仓库，在编码阶段发现，加上超时自动释放，避免长期占有锁
  - 重复执行有两个解决方法
    - 避免超时，单独开一个线程给锁延长有效期。比如设置锁有效期30s，有个线程每隔10s重新设置下锁的有效期。
      - 默认情况下,加锁的时间是30秒.如果加锁的业务没有执行完,那么到 30-10 = 20秒的时候,就会进行一次续期,把锁重置成30秒
    - 避免重复，业务上增加一个标记是否被处理的字段。或者开一张新表，保存已经处理过的流水号。
      - 业务层面一定要做好幂等



## 线程池

- 线程池的声明需要手动进行

  - newFixedThreadPool 和 newCachedThreadPool，可能因为资源耗尽导致OOM 问题。

    - ```java
      // 虽然使用 newFixedThreadPool 可以把工作线程控制在固定的数量上，但任务队列是无界的。如果任务较多并且执行较慢的话，队列可能会快速积压，撑爆内存导致 OOM。
      public void oom() throws InterruptedException {
              ThreadPoolExecutor threadpool =(ThreadPoolExecutor) Executors.newFixedThreadPool(1);
              for (int i = 0; i < 10000000; i++) {
                  threadpool.execute(()->{
                      String payload = IntStream.rangeClosed(1,10000)
                              .mapToObj(__->"A")
                              .collect(Collectors.joining(""))+ UUID.randomUUID().toString();
                      try{
                          TimeUnit.HOURS.sleep(1);
                      }catch (InterruptedException e){
                          
                      }
                  });
                  threadpool.shutdown();
                  threadpool.awaitTermination(1,TimeUnit.HOURS);
      
              }
      ```

    - newCachedThreadPool 

      - 从日志中可以看到，这次 OOM 的原因是无法创建线程
      - 翻看 newCachedThreadPool 的源码可以看到，这种线程池的最大线程数是 Integer.MAX_VALUE，可以认为是没有上限的，而其工作队列 SynchronousQueue 是一个没有存储空间的阻塞队列
      - 这意味着，只要有请求到来，就必须找到一条工作线程来处理，如果当前没有空闲的线程就再创建一条新的。由于我们的任务需要 1 小时才能执行完成，大量的任务进来后会创建大量的线程。我们知道线程是需要分配一定的内存空间作为线程栈的，比如 1MB，因此无限制创建线程必然会
        导致 OOM

  - 应该手动 new ThreadPoolExecutor 来创建线程池。

- 线程池线程管理策略

  - ```java
    // 通过java在做定时任务的时候最好使用scheduleThreadPoolExecutor的方式
    // scheduleAtFixedRate(commod,initialDelay,period,unit)
    // initialDelay是说系统启动后，需要等待多久才开始执行。
    // period为固定周期时间，按照一定频率来重复执行任务。
    // 如果period设置的是3秒，系统执行要5秒；那么等上一次任务执行完就立即执行，也就是任务与任务之间的差异是5s；
    // 如果period设置的是3s，系统执行要2s；那么需要等到3S后再次执行下一次任务。
    
    // 最简陋的监控，每秒输出一次线程池的基本内部信息，包括线程数、活跃线程数、完成了多少任务，以及队列中还有多少积压任务等信息
    ThreadPoolExecutor threadpool =(ThreadPoolExecutor) Executors.newFixedThreadPool(1);
            Executors.newSingleThreadScheduledExecutor().scheduleAtFixedRate(()-> {
                threadpool.getPoolSize();
                threadpool.getActiveCount();
                threadpool.getCompletedTaskCount();
                threadpool.getQueue().size();
            },0,1,TimeUnit.SECONDS);
    ```

- 让线程池更激进一点，优先开启更多的线程，而把队列当成一个后备方案呢？

  - 不知道你有没有想过：Java 线程池是先用工作队列来存放来不及处理的任务，满了之后再扩容线程池。当我们的工作队列设置得很大时，最大线程数这个参数显得没有意义，因为队列很难满，或者到满的时候再去扩容线程池已经于事无补了。

  - 

  - ```java
    // 使用Java 7 LinkedTransferQueue并进行offer()调用tryTransfer()。当有一个正在等待的使用者线程时，任务将被传递给该线程。否则，offer()将返回false ThreadPoolExecutor并将产生一个新线程。
    // submit()方法是调用了workQueue的offer()方法来塞入task，而offer()方法是非阻塞的，当workQueue已经满的时候，offer()方法会立即返回false，并不会阻塞在那里等待workQueue有空出位置，所以要让submit()阻塞，关键在于改变向workQueue添加task的行为
    // 这就达到了想要的效果：当workQueue满时，submit()一个task会导致调用我们自定义的RejectedExecutionHandler，而我们自定义的RejectedExecutionHandler会保证该task继续被尝试用阻塞式的put()到workQueue中。
    BlockingQueue<Runnable> queue = new LinkedTransferQueue<Runnable>() {
            @Override
            public boolean offer(Runnable e) {
                return tryTransfer(e);
            }
        };
        ThreadPoolExecutor threadPool = new ThreadPoolExecutor(1, 50, 60, TimeUnit.SECONDS, queue);
        threadPool.setRejectedExecutionHandler(new RejectedExecutionHandler() {
            @Override
            public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
                try {
                    executor.getQueue().put(r);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });
    ```

- 务必确认清楚线程池本身是不是复用的

  - 这样一个事故：某项目生产环境时不时有报警提示线程数过多，超过2000 个，收到报警后查看监控发现，瞬时线程数比较多但过一会儿又会降下来，线程数抖动很厉害，而应用的访问量变化不大。
    - 但是，来到 ThreadPoolHelper 的实现让人大跌眼镜，getThreadPool 方法居然是每次都使用 Executors.newCachedThreadPool 来创建一个线程池。
    - 我们可以想到 newCachedThreadPool 会在需要时创建必要多的线程，业务代码的一次业务操作会向线程池提交多个慢任务，这样执行一次业务操作就会开启多个线程。如果业务操作并发量较大的话，的确有可能一下子开启几千个线程。
  - 那，为什么我们能在监控中看到线程数量会下降，而不会撑爆内存呢？
    - 回到 newCachedThreadPool 的定义就会发现，它的核心线程数是 0，而 keepAliveTime是 60 秒，也就是在 60 秒之后所有的线程都是可以回收的。好吧，就因为这个特性，我们的业务程序死得没太难看
    - 要修复这个 Bug 也很简单，使用一个静态字段来存放线程池的引用，返回线程池的代码直接返回这个静态字段即可。这里一定要记得我们的最佳实践，手动创建线程池。

- 需要仔细斟酌线程池的混用策略

  - 对于执行比较慢、数量不大的 IO 任务，或许要考虑更多的线程数，而不需要太大的队列。
  - 而对于吞吐量较大的计算型任务，线程数量不宜过多，可以是 CPU 核数或核数 *2（理由是，线程一定调度到某个 CPU 进行执行，那么过多的线程只会增加线程切换的开销，并不能提升吞吐量），但可能需要较长的队列来做缓冲。
  - 遇到过这么一个问题，业务代码使用了线程池异步处理一些内存中的数据，但通过监控发现处理得非常慢，整个处理过程都是内存中的计算不涉及 IO 操作，也需要数秒的处理时间，应用程序 CPU 占用也不是特别高，有点不可思议。
    - 经排查发现，业务代码使用的线程池，还被一个后台的文件批处理任务用到了
    - 或许是够用就好的原则，这个线程池只有 2 个核心线程，最大线程也是 2，使用了容量为100 的 ArrayBlockingQueue 作为工作队列，使用了 CallerRunsPolicy 拒绝策略
    - CallerRunsPolicy    -- 当任务添加到线程池中被拒绝时，会在线程池当前正在运行的Thread线程池中处理被拒绝的任务。
    - 因为开启了CallerRunsPolicy 拒绝处理策略，所以当线程满载队列也满的情况下，任务会在提交任务的线程，或者说调用 execute 方法的线程执行，也就是说不能认为提交到线程池的任务就一定是异步处理的。如果使用了CallerRunsPolicy 策略，那么有可能异步任务变为同步执行。
    - 细想一下，问题其实没有这么简单。因为原来执行 IO 任务的线程池使用的是CallerRunsPolicy 策略，所以直接使用这个线程池进行异步计算的话，当线程池饱和的时候，计算任务会在执行 Web 请求的 Tomcat 线程执行，这时就会进一步影响到其他同步处理的线程，甚至造成整个应用程序崩溃。

- Java 8 的 parallel stream 功能

  - 可以让我们很方便地并行处理集合中的元素，其背后是共享同一个 ForkJoinPool，默认并行度是CPU 核数 -1
  - 对于 CPU 绑定的任务来说，使用这样的配置比较合适，但如果集合操作涉及同步 IO 操作的话（比如数据库操作、外部服务调用等），建议自定义一个ForkJoinPool（或普通线程池）

- 我们改进了 ThreadPoolHelper 使其能够返回复用的线程池。如果我们不小心每次都创建了这样一个自定义的线程池（10 核心线程，50 最大线程，2 秒回收的），反复执行测试接口线程，最终可以被回收吗？会出现 OOM 问题吗？

  - 每次请求都新建线程池，每个线程池的核心数都是10, 虽然自定义线程池设置2秒回收，但是没超过线程池核心数10是不会被回收的, 不间断的请求过来导致创建大量线程，最终OOM
  - ThreadPoolExecutor回收不了，工作线程Worker是内部类，只要它活着，换句话说线程在跑，就会阻止ThreadPoolExecutor回收，所以其实ThreadPoolExecutor是无法回收的，并不能认为ThreadPoolExecutor没有引用就能回收

- 

  



