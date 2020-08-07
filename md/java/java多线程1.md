## 线程池

- 使用线程池可以带来一系列好处：

  - 降低资源消耗：通过池化技术重复利用已创建的线程，降低线程创建和销毁造成的损耗。
  - 提高响应速度：任务到达时，无需等待线程创建即可立即执行。
  - 提高线程的可管理性：线程是稀缺资源，如果无限制创建，不仅会消耗系统资源，还会因为线程的不合理分布导致资源调度失衡，降低系统的稳定性。使用线程池可以进行统一的分配、调优和监控。
  - 提供更多更强大的功能：线程池具备可拓展性，允许开发人员向其中增加更多的功能。比如延时定时线程池ScheduledThreadPoolExecutor，就允许任务延期执行或定期执行。

- 线程池可以通过ThreadPoolExecutor来创建

  - > corePoolSize： 线程池核心线程数最大值
    >
    > maximumPoolSize： 线程池最大线程数大小
    >
    > keepAliveTime： 线程池中非核心线程空闲的存活时间大小
    >
    > workQueue： 存放任务的阻塞队列
    >
    > RejectedExecutionHandler：拒绝策略
    >
    > ​			AbortPolicy：直接拒绝策略，也就是不会执行任务，直接抛出RejectedExecutionException，这是默认的拒绝策略。
    >
    > ​			DiscardPolicy：抛弃策略，也就是直接忽略提交的任务（通俗来说就是空实现）。
    > ​			DiscardOldestPolicy：抛弃最老任务策略，也就是通过poll()方法取出任务队列队头的任务抛弃，然后执行当前提交的任务。
    > ​			CallerRunsPolicy：调用者执行策略，也就是当前调用Executor#execute()的线程直接调用任务Runnable#run()，一般不希望任务丢失会选用这种策略，但从实际角度来看，原来的异步调用意图会退化为同步调用。

  - 提交一个任务，线程池里存活的核心线程数小于线程数corePoolSize时，线程池会创建一个核心线程去处理提交的任务。

  - 如果线程池核心线程数已满，即线程数已经等于corePoolSize，一个新提交的任务，会被放进任务队列workQueue排队等待执行。
    当线程池里面存活的线程数已经等于corePoolSize了,并且任务队列workQueue也满，判断线程数是否达到maximumPoolSize，即最大线程数是否已满，如果没到达，创建一个非核心线程执行提交的任务。
    如果当前的线程数达到了maximumPoolSize，还有新的任务过来的话，直接采用拒绝策略处理。

  - 非核心线程的回收周期（线程生命周期终结时刻）是keepAliveTime，线程生命周期终结的条件是：下一次通过任务队列获取任务的时候并且存活时间超过keepAliveTime。

- 工作队列（具体见JUC工具类）

  - ArrayBlockingQueue

    LinkedBlockingQueue

    SynchronousQueue

- 常用的线程池

  - newFixedThreadPool (固定数目线程的线程池)
    - 核心线程数和最大线程数大小一样
      没有非空闲时间，即keepAliveTime为0
      阻塞队列为无界队列LinkedBlockingQueue
    - newFixedThreadPool使用了无界的阻塞队列LinkedBlockingQueue，如果线程获取一个任务后，任务的执行时间比较长，会导致队列的任务越积越多，导致机器内存使用不停飙升
  - newSingleThreadExecutor(单线程的线程池)
    - 核心线程数为1
      最大线程数也为1
      阻塞队列是LinkedBlockingQueue
      keepAliveTime为0
  - newCachedThreadPool(可缓存线程的线程池)
    - 核心线程数为0
      最大线程数为Integer.MAX_VALUE
      阻塞队列是SynchronousQueue
      非核心线程空闲存活时间为60秒
    - 用来处理大量短时间工作任务的线程池
  - newScheduledThreadPool(定时及周期执行的线程池)
    - newSingleThreadScheduledExecutor() 和 newScheduledThreadPool(int corePoolSize)，创建的是个 ScheduledExecutorService，可以进行定时或周期性的工作调度，区别在于单一工作线程还是多个工作线程。

- 线程池状态

  - RUNNING,SHUTDOWN,STOP,TIDYING,TERMINATED
  - RUNNING
    - 该状态的线程池会接收新任务，并处理阻塞队列中的任务;
  - SHUTDOWN
    - 该状态的线程池不会接收新任务，但会处理阻塞队列中的任务；
  - STOP
    - 该状态的线程不会接收新任务，也不会处理阻塞队列中的任务，而且会中断正在运行的任务；
  - TIDYING
    - 所有任务已经终结，工作线程数为0，过渡到此状态的工作线程会调用钩子方法terminated()
  - TERMINATED
    - 钩子方法terminated()执行完毕

- 线程池中submit()和execute()方法有什么区别

  - execute()：只能执行Runnable类型的任务。submit()：可以执行Runnable和Callnable类型的任务。
  - Callable类型的任务可以获取执行的返回值，而Runnable执行无返回值。

- 场景

  - 快速响应用户请求
    - 这种场景最重要的就是获取最大的响应速度去满足用户，所以应该不设置队列去缓冲并发任务，调高corePoolSize和maxPoolSize去尽可能创造多的线程快速执行任务。
  - 快速处理批量任务
    - 这类场景任务量巨大，并不需要瞬时的完成，而是关注如何使用有限的资源，尽可能在单位时间内处理更多的任务，也就是吞吐量优先的问题。所以应该设置队列去缓冲并发任务，调整合适的corePoolSize去设置处理任务的线程数。
    - 在这里，设置的线程数过多可能还会引发线程上下文切换频繁的问题，也会降低处理任务的速度，降低吞吐量。

- 线程池使用面临的核心的问题

  - 接口大量调用降级
    - 事故原因：该服务展示接口内部逻辑使用线程池做并行计算，由于没有预估好调用的流量，导致最大核心数设置偏小，大量抛出RejectedExecutionException，触发接口降级条件
  - 服务不可用
    - 事故原因：该服务处理请求内部逻辑使用线程池做资源隔离，由于队列设置过长，最大线程数设置失效，导致请求数量增加时，大量任务堆积在队列中，任务执行时间过长，最终导致下游服务的大量调用超时失败。



## ThreadPoolExecutor#execute()的实现

- ThreadPoolExecutor里面使用到JUC同步器框架AbstractQueuedSynchronizer（俗称AQS）、大量的位操作、CAS操作。ThreadPoolExecutor提供了固定活跃线程（核心线程）、额外的线程（线程池容量 - 核心线程数这部分额外创建的线程，下面称为非核心线程）、任务队列以及拒绝策略这几个重要的功能。

- JUC同步器框架

  - 全局锁mainLock成员属性，是可重入锁ReentrantLock类型，主要是用于访问工作线程Worker集合和进行数据统计记录时候的加锁操作。
  - 条件变量termination，Condition类型，主要用于线程进行等待终结awaitTermination()方法时的带期限阻塞。
  - 任务队列workQueue，BlockingQueue<Runnable>类型，任务队列，用于存放待执行的任务。
  - 工作线程，内部类Worker类型，是线程池中真正的工作线程对象。

- 实现一个只有核心线程的线程池

  - ```java
    public class CoreThreadPool implements Executor {
    
        private BlockingQueue<Runnable> workQueue;
        private static final AtomicInteger COUNTER = new AtomicInteger();
        private int coreSize;
        private int threadCount = 0;
    
        public CoreThreadPool(int coreSize) {
            this.coreSize = coreSize;
            this.workQueue = new LinkedBlockingQueue<>();
        }
    
        @Override
        public void execute(Runnable command) {
            if (++threadCount <= coreSize) {
                new Worker(command).start();
            } else {
                try {
                    workQueue.put(command);
                } catch (InterruptedException e) {
                    throw new IllegalStateException(e);
                }
            }
        }
    
        private class Worker extends Thread {
            private Runnable firstTask;
    
            public Worker(Runnable runnable) {
                super(String.format("Worker-%d", COUNTER.getAndIncrement()));
                this.firstTask = runnable;
            }
    
            @Override
            public void run() {
                Runnable task = this.firstTask;
                while (null != task || null != (task = getTask())) {
                    try {
                        task.run();
                    } finally {
                        task = null;
                    }
                }
            }
        }
    
        private Runnable getTask() {
            try {
                return workQueue.take();
            } catch (InterruptedException e) {
                throw new IllegalStateException(e);
            }
        }
    
        public static void main(String[] args) throws Exception {
            CoreThreadPool pool = new CoreThreadPool(5);
            IntStream.range(0, 10)
                    .forEach(i -> pool.execute(() ->
                            System.out.println(String.format("Thread:%s,value:%d", Thread.currentThread().getName(), i))));
            Thread.sleep(Integer.MAX_VALUE);
        }
    }
    ```


#### 状态控制

- ```java
  // 状态控制主要围绕原子整型成员变量ctl
  private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
  // 工作线程上限数量位的长度是COUNT_BITS
  // 我们知道，整型包装类型Integer实例的大小是4 byte，一共32 bit，也就是一共有32个位用于存放0或者1。
  // 在ThreadPoolExecutor实现中，使用32位的整型包装类型存放工作线程数和线程池状态。
  private static final int COUNT_BITS = Integer.SIZE - 3;
  // 其中，低29位用于存放工作线程数，而高3位用于存放线程池状态，所以线程池的状态最多只能有2^3种。
  private static final int COUNT_MASK = (1 << COUNT_BITS) - 1;
  
  // -1的补码为：111-11111111111111111111111111111
  // 左移29位后：111-00000000000000000000000000000
  // 高3位111的值就是表示线程池正在处于运行状态
  private static final int RUNNING    = -1 << COUNT_BITS;
  private static final int SHUTDOWN   =  0 << COUNT_BITS;
  private static final int STOP       =  1 << COUNT_BITS;
  private static final int TIDYING    =  2 << COUNT_BITS;
  private static final int TERMINATED =  3 << COUNT_BITS;
  
  // 线程池源码中有很多中间变量用了简单的单字母表示，例如c就是表示ctl、wc就是表示worker count、rs就是表示running status。
  
  // 从ctl中取出高3位的线程池状态
  // 先把COUNT_MASK取反(~COUNT_MASK)，得到：111-00000000000000000000000000000
  // ctl位图特点是：xxx-yyyyyyyyyyyyyyyyyyyyyyyyyyyyyy
  // 两者做一次与运算即可得到高3位xxx
  private static int runStateOf(int c)     { return c & ~COUNT_MASK; }
  
  // 通过ctl值获取工作线程数
  private static int workerCountOf(int c)  { return c & COUNT_MASK; }
  
  // 通过运行状态和工作线程数计算ctl的值，或运算
  // 控制变量ctl的组成就是通过线程池运行状态rs和工作线程数wc通过或运算得到的：
  // rs=RUNNING值为：111-00000000000000000000000000000
  // wc的值为0：000-00000000000000000000000000000
  // rs | wc的结果为：111-00000000000000000000000000000
  private static int ctlOf(int rs, int wc) { return rs | wc; }
  
  
  // 工作线程数为0的前提下：RUNNING(-536870912) < SHUTDOWN(0) < STOP(536870912) < TIDYING(1073741824) < TERMINATED(1610612736)
  // ctl和状态常量比较，判断是否小于
  private static boolean runStateLessThan(int c, int s) {
      return c < s;
  }
  // ctl和状态常量比较，判断是否小于或等于
  private static boolean runStateAtLeast(int c, int s) {
      return c >= s;
  }
  
  // 这里有一个比较特殊的技巧，由于运行状态值存放在高3位，所以可以直接通过十进制值（甚至可以忽略低29位，直接用ctl进行比较，或者使用ctl和线程池状态常量进行比较）来比较和判断线程池的状态：
  
  // ctl和状态常量SHUTDOWN比较，判断是否处于RUNNING状态
  private static boolean isRunning(int c) {
      return c < SHUTDOWN;
  }
  
  // CAS操作线程数增加1
  private boolean compareAndIncrementWorkerCount(int expect) {
      return ctl.compareAndSet(expect, expect + 1);
  }
  
  // CAS操作线程数减少1
  private boolean compareAndDecrementWorkerCount(int expect) {
      return ctl.compareAndSet(expect, expect - 1);
  }
  
  // 线程数直接减少1
  private void decrementWorkerCount() {
      ctl.addAndGet(-1);
  }
  ```

#### 线程池状态的跃迁

- ![img](https://throwable-blog-1256189093.cos.ap-guangzhou.myqcloud.com/201907/j-u-c-t-p-e-2.png)

#### execute方法源码分析

- 如果当前工作线程总数小于corePoolSize，则直接创建核心线程执行任务（任务实例会传入直接用于构造工作线程实例）。
- 如果当前工作线程总数大于等于corePoolSize，判断线程池是否处于运行中状态，同时尝试用非阻塞方法向任务队列放入任务，这里会二次检查线程池运行状态，如果当前工作线程数量为0，则创建一个非核心线程并且传入的任务对象为null。
  - 这里是一个疑惑点：为什么需要二次检查线程池的运行状态，当前工作线程数量为0，尝试创建一个非核心线程并且传入的任务对象为null？这个可以看API注释：
  - 如果一个任务成功加入任务队列，我们依然需要二次检查是否需要添加一个工作线程（因为所有存活的工作线程有可能在最后一次检查之后已经终结）或者执行当前方法的时候线程池是否已经shutdown了。所以我们需要二次检查线程池的状态，必须时把任务从任务队列中移除或者在没有可用的工作线程的前提下新建一个工作线程。
- 如果向任务队列投放任务失败（任务队列已经满了），则会尝试创建非核心线程传入任务实例执行。
- 如果创建非核心线程失败，此时需要拒绝执行任务，调用拒绝策略处理任务
- addWorker方法源码分析

  - boolean addWorker(Runnable firstTask, boolean core)方法的第一的参数可以用于直接传入任务实例，第二个参数用于标识将要创建的工作线程是否核心线程。
  - 需要注意一点，Worker实例创建的同时，在其构造函数中会通过ThreadFactory创建一个Java线程Thread实例，后面会加锁后二次检查是否需要把Worker实例添加到工作线程集合workers中和是否需要启动Worker中持有的Thread实例，只有启动了Thread实例实例，Worker才真正开始运作，否则只是一个无用的临时对象。Worker本身也实现了Runnable接口，它可以看成是一个Runnable的适配器。

#### 工作线程内部类Worker源码分析

- 线程池中的每一个具体的工作线程被包装为内部类Worker实例，Worker继承于AbstractQueuedSynchronizer(AQS)，实现了Runnable接口
- Worker的构造函数里面的逻辑十分重要，通过ThreadFactory创建的Thread实例同时传入Worker实例，因为Worker本身实现了Runnable，所以可以作为任务提交到线程中执行。只要Worker持有的线程实例w调用Thread#start()方法就能在合适时机执行Worker#run()。

#### 核心方法ThreadPoolExecutor#runWorker()：

- Worker类的run方法实际上调用的还是ThreadPoolExecutor的runworker方法

- 核心流程

  - 通过while循环调用getTask()方法从任务队列中获取任务（当然，首轮循环也有可能是外部传入的firstTask任务实例）。

  - 如果线程池更变为STOP状态，则需要确保工作线程是中断状态并且进行中断处理，否则要保证工作线程必须不是中断状态。

    - Thread.interrupted()方法获取线程的中断状态同时会清空该中断状态，这里之所以会调用这个方法是因为在执行上面这个if逻辑同时外部有可能调用shutdownNow()方法，shutdownNow()方法中也存在中断所有Worker线程的逻辑，但是由于shutdownNow()方法中会遍历所有Worker做线程中断，有可能无法及时在任务提交到Worker执行之前进行中断，所以这个中断逻辑会在Worker内部执行，就是if代码块的逻辑。这里还要注意的是：STOP状态下会拒绝所有新提交的任务，不会再执行任务队列中的任务，同时会中断所有Worker线程。也就是，即使任务Runnable已经runWorker()中前半段逻辑取出，只要还没走到调用其Runnable#run()，都有可能被中断。假设刚好发生了进入if代码块的逻辑同时外部调用了shutdownNow()方法，那么if逻辑内会判断线程中断状态并且重置，那么shutdownNow()方法中调用的interruptWorkers()就不会因为中断状态判断出现问题导致二次中断线程（会导致异常）。

    - ```java
      if ((runStateAtLeast(ctl.get(), STOP) ||
              (Thread.interrupted() &&
              runStateAtLeast(ctl.get(), STOP))) &&
          !wt.isInterrupted())
          wt.interrupt();
      // 先简化一下判断逻辑，如下
      // 判断线程池状态是否至少为STOP，rs >= STOP(1)
      boolean atLeastStop = runStateAtLeast(ctl.get(), STOP);
      // 判断线程池状态是否至少为STOP，同时判断当前线程的中断状态并且清空当前线程的中断状态
      boolean interruptedAndAtLeastStop = Thread.interrupted() && runStateAtLeast(ctl.get(), STOP);
      if (atLeastStop || interruptedAndAtLeastStop && !wt.isInterrupted()){
          wt.interrupt();
      }
      ```

  - 执行任务实例Runnale#run()方法，任务实例执行之前和之后（包括正常执行完毕和异常执行情况）分别会调用钩子方法beforeExecute()和afterExecute()。

  - while循环跳出意味着runWorker()方法结束和工作线程生命周期结束（Worker#run()生命周期完结），会调用processWorkerExit()处理工作线程退出的后续工作。

#### getTask方法源码分析

- getTask()方法是工作线程在while死循环中获取任务队列中的任务对象的方法
- 这个方法中，有两处十分庞大的if逻辑，对于第一处if可能导致工作线程数减去1直接返回null的场景有
  - 线程池状态为SHUTDOWN，一般是调用了shutdown()方法，并且任务队列为空。
  - 线程池状态为STOP
- 对于第二处if
  - 这段逻辑大多数情况下是针对非核心线程。在execute()方法中，当线程池总数已经超过了corePoolSize并且还小于maximumPoolSize时，当任务队列已经满了的时候，会通过addWorker(task,false)添加非核心线程。而这里的逻辑恰好类似于addWorker(task,false)的反向操作，用于减少非核心线程，使得工作线程总数趋向于corePoolSize。
  - 如果对于非核心线程，上一轮循环获取任务对象为null，这一轮循环很容易满足timed && timedOut为true，这个时候getTask()返回null会导致Worker#runWorker()方法跳出死循环，之后执行processWorkerExit()方法处理后续工作，而该非核心线程对应的Worker则变成“游离对象”，等待被JVM回收。

#### keepAliveTime的意义：

- 当允许核心线程超时，也就是allowCoreThreadTimeOut设置为true的时候，此时keepAliveTime表示空闲的工作线程的存活周期。
- 默认情况下不允许核心线程超时，此时keepAliveTime表示空闲的非核心线程的存活周期。

#### processWorkerExit方法源码分析

- processWorkerExit()方法是为将要终结的Worker做一次清理和数据记录工作（因为processWorkerExit()方法也包裹在runWorker()方法finally代码块中，其实工作线程在执行完processWorkerExit()方法才算真正的终结）。
- 代码的后面部分区域，会判断线程池的状态，如果线程池是RUNNING或者SHUTDOWN状态的前提下，如果当前的工作线程由于抛出用户异常被终结，那么会新创建一个非核心线程。
- 如果当前的工作线程并不是抛出用户异常被终结（正常情况下的终结）
  - allowCoreThreadTimeOut为true，也就是允许核心线程超时的前提下，如果任务队列空，则会通过创建一个非核心线程保持线程池中至少有一个工作线程。
  - allowCoreThreadTimeOut为false，如果工作线程总数大于corePoolSize则直接返回，否则创建一个非核心线程，也就是会趋向于保持线程池中的工作线程数量趋向于corePoolSize。



