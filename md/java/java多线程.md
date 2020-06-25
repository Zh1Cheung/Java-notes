## Thread中start()和run()的区别

- Thread 类实现了 Runnable 接口，而 Runnable 接口定义了唯一的一个 run() 方法
  - 如果直接调用 run() 方法，那就等于调用了一个普通的同步方法，达不到多线程运行的异步执行
- start() : 它的作用是通过本地方法start0()启动一个新线程，新线程会执行相应的run()方法执行线程的运行时代码。run()可以重复调用，而start()只能调用一次。
  - start() 方法则是 Thread 类的方法，用来异步启动一个线程，然后主线程立刻返回。
  - 该启动的线程不会马上运行，会放到等待队列中等待 CPU 调度，只有线程真正被 CPU 调度时才会调用 run() 方法执行。
  - start() 方法被标识为 synchronized 的，即为了防止被多次启动的一个同步操作。







## synchronized关键字

- 原理
  - 在java中，每一个对象有且仅有一个同步锁。这也意味着，同步锁是依赖于对象而存在。当我们调用某对象的synchronized方法时，就获取了该对象的同步锁
  - 不同线程对同步锁的访问是互斥的，对象的同步锁只能被一个线程获取到
  - 在修饰代码块的时候需要一个reference对象作为锁的对象.
  - 在修饰方法的时候默认是当前对象作为锁的对象.
  - 在修饰类时候默认是当前类的Class对象作为锁的对象.
    - 在Java中一般有两种引用类型:Reference类型 类型和普通引用类型
- 32位的HotSpot虚拟机对象头存储结构
  - 对象头(Object Header)包括两部分信息:
    - "Mark Word":存储对象自身的运行时数据
      - 对象头中的Mark Word，synchronized源码实现就用了Mark Word来标识对象加锁状态
      - 不管是32/64位JVM，都是1bit偏向锁+2bit锁标志位
    - "Klass Pointer"：对象指向它的类的元数据的指针
  - 实例数据（Instance Data）
  - 对齐填充（Padding）
- 在Java虚拟机(HotSpot)中，monitor是由ObjectMonitor实现的
  - ObjectMonitor中有两个队列，_WaitSet 和 _EntryList
  - 当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1，若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSet集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)。
  - 因此，monitor对象存在于每个Java对象的对象头中(存储的指针的指向)，synchronized锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因，同时也是notify/notifyAll/wait等方法存在于顶级对象Object中的原因
- wait和notify为什么需要在synchronized里面？
  - monitor 存在于对象头的Mark Word 中(存储monitor引用指针)，而synchronized关键字可以获取 monitor ，这也就是为什么notify/notifyAll和wait方法必须在synchronized代码块或者synchronized方法调用的原因
  - synchronized 代码块通过javap生成的字节码中包含 monitorenter 和 monitorexit 指令。
  - 如果wait方法在synchronized代码中执行，该线程很显然已经持有了monitor。
  - wait方法的语义有两个，一个是释放当前的对象锁、另一个是使得当前线程进入阻塞队列，而这些操作都和monitor是相关的，所以wait必须要获得一个monitor。
  - WaitSet 主要是存放在同步块中执行 wait 方法的线程。配合 EntryList 就是 对象 的 wait 和 notify(notifyAll) 的底层实现。
- 那么这两类集合中的线程都是在什么条件下可以转变为RUNNABLE呢？
  - 对于Entry List中的线程，当获得对象锁的时候，JVM会唤醒处于Entry Set中的某一个线程，这个线程的状态就从BLOCKED转变为RUNNABLE。
  - 对于Wait Set中的线程，当对象的notify()方法被调用时，JVM会唤醒处于Wait Set中的某一个线程，这个线程的状态就从WAITING转变为BLOCKED；或者当notifyAll()方法被调用时，Wait Set中的全部线程会转变为BLOCKED状态。所有Wait Set中被唤醒的线程会被转移到Entry Set中。然后同上
-  notify执行之后立马唤醒线程吗
  - 其实hotspot里真正的实现是退出同步块的时候才会去真正唤醒对应的线程
- notifyAll是怎么实现全唤起所有线程
  - JVM里没实现这么简单，而是借助了monitorexit，上面提到了当某个线程从wait状态恢复出来的时候，要先获取锁，然后再退出同步块，所以notifyAll的实现是调用notify的线程在退出其同步块的时候唤醒起最后一个进入wait状态的线程，然后这个线程退出同步块的时候继续唤醒其倒数第二个进入wait状态的线程，依次类推







##  interrupt()和线程终止方式

- 终止处于“阻塞状态”的线程
  - 若线程在阻塞状态时，调用了它的interrupt()方法，那么它的“中断状态”会被清除并且会收到一个InterruptedException异常
- 终止处于“运行状态”的线程
  - 通过“标记”方式终止处于“运行状态”的线程
    - isInterrupted()是判断线程的中断标记是不是为true
    - interrupt()并不会终止处于“运行状态”的线程！它会将线程的中断标记设为true
    - interrupted()除了返回中断标记之外，它还会清除中断标记(即将中断标记设为false)；而isInterrupted()仅仅返回中断标记





## 生产消费者问题

- 使用Object的wait/notify的消息通知机制

  - ```java
    /*
     * 生产者线程
     */
    class ProducerDemo implements Runnable{
        private List list;
        public ProducerDemo(List list){
            this.list=list;
        }
        @Override
        public void run() {
            while(true){
                Random random=new Random();
                synchronized(list){
                    while(list.size()>0){ 
                        //因为生产者线程有多个，当本线程wait之后，假如一个生产者线程得到锁（本该消费者得到），
                        // 如果是if，那么此线程就会继续执行，会导致数据错乱。
                        //如果是while则会继续等待。
                        try {
                            list.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    list.add(random.nextInt(100));//0-99的随机数；
                    System.out.println(Thread.currentThread().getName()+"  "+list.get(0));
                    list.notifyAll();  //唤醒此对象锁所有等待线程（消费者和生产者线程均有）
                }
            }
        }
    }
    ```

- 使用Lock的Condition的await/signal的消息通知机制

  - wait--->await，notify---->Signal

- 使用BlockingQueue实现

  - ```java
    //生产者
    class ProducerDemo1 implements Runnable{
        ReentrantLock put;
        ReentrantLock out;
        Condition notFull;
        Condition notEmpty;
        Queue<Integer> queue;
        public ProducerDemo1(ReentrantLock put,ReentrantLock out,Condition notFull,Condition notEmpty,Queue queue){
            this.put=put;
            this.out=out;
            this.notFull=notFull;
            this.notEmpty=notEmpty;
            this.queue=queue;
        }
    
        @Override
        public void run() {
            Random random=new Random();
            while(true){
                put.lock();
                while(queue.size()==10){
                    try {
                        notFull.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                //延迟打印速度
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                Integer integer=random.nextInt(100);
                System.out.println(Thread.currentThread().getName()+"........."+integer);
                queue.add(integer);
                //队列没有满，通知更多生产者来生产
                if(queue.size()<10){
                    notFull.signal();
                }
                put.unlock();
                //只要生产出一个就通知消费者消费，后续不需要通知
                //因为消费者内部有唤醒更多消费者的机制
                if(queue.size()==1){
                    out.lock();
                    notEmpty.signal();
                    out.unlock();
                }
            }
        }
    }
    ```

    





## 进程与线程

- 进程，是并发执行的程序在执行过程中分配和管理资源的基本单位。进程是程序在计算机上的一次执行活动
- 线程，是进程的一部分，一个没有线程的进程可以被看作是单线程的，也是CPU 调度的一个基本单位，多线程意味着单个程序可以并发执行两个或者多个任务
  - 进程拥有一个完整的虚拟地址空间，不依赖于线程而独立存在；反之，线程是进程的一部分，没有自己的地址空间，与进程内的其他线程一起共享分配给该进程的所有资源
  - 线程的改变只代表了 CPU 执行过程的改变，而没有发生进程所拥有的资源变化
  - 多线程的意义在于一个应用程序中，有多个执行部分可以同时执行。但操作系统并没有将多个线程看做多个独立的应用，来实现进程的调度和管理以及资源分配。这就是进程和线程的重要区别。
- 进程和线程都是一个时间段的描述，是CPU工作时间段的描述。
  - 进程就是包含上下文切换的程序执行时间总和 = CPU加载上下文+CPU执行+CPU保存上下文
  - 程序A得到CPU =》CPU加载上下文，开始执行程序A的a小段，然后执行A的b小段，然后再执行A的c小段，最后CPU保存A的上下文。
    - 这里a，b，c的执行是共享了A的上下文，CPU在执行的时候没有进行上下文切换的。这里的a，b，c就是线程，也就是说线程是共享了进程的上下文环境的更为细小的CPU时间段。
- 线程和进程是两个相对独立的概念，线程更多是对执行序列的抽象，进程更多是运行空间的抽象，它们是交叉的，按OS实现的方便，有时可以切换执行序列而不切换运行空间（例如Linux的进程内线程切换），有时可以切换运行空间而不切换执行序列





## 同步与异步

- 阻塞和非阻塞只是一种性质，而同步和异步是从宏观上对一种任务性质的描述
- 阻塞和非阻塞可以理解为，A请求B之后，在B端资源没准备好的时候，B是让A一直等待还是发出一个标志





## 死锁

- 产生死锁的原因
  - 竞争不可剥夺资源
    - CPU属于可剥夺性资源,但一个进程已获得的资源，在未使用完之前，另一个进程要使用可以把它的资源剥夺过来，所以不会产生死锁。只有不可剥夺资源才会因为竞争资源而产生死锁
    - 系统中只有一台打印机，可供进程P1使用，假定P1已占用了打印机，若P2继续要求打印机打印将阻塞
  - 竞争临时资源
    - 临时资源包括硬件中断、信号、消息、缓冲区内的消息等，通常消息通信顺序进行不当，则会产生死锁
  - 进程间推进顺序非法
    - 例如，当P1运行到P1：Request（R2）时，将因R2已被P2占用而阻塞；当P2运行到P2：Request（R1）时，也将因R1已被P1占用而阻塞，于是发生进程死锁
- 防止死锁
  - 尽量使用tryLock(long timeout,TimeUnit unit)的方法(ReentrantLock、ReentrantReadWriteLock)，设置超时时间，超时可以退出防止死锁。
  - 尽量使用java.util.concurrent并发类代替自己手写锁。
  - 尽量降低锁的使用粒度，尽量不要几个功能用同一把锁。
  - 尽量减少同步的代码块。





## ThreadLocal

- 同一个 ThreadLocal 所包含的对象，在不同的 Thread 中有不同的副本，实际是不同的实例

  - ThreadLocal提供了线程本地的实例。它与普通变量的区别在于，每个使用该变量的线程都会初始化一个完全独立的实例副本

  - 当一个线程结束时，它所使用的所有 ThreadLocal 相对的实例副本都可被回收。

    - 每个线程通过 ThreadLocal 的 get() 方法拿到的是不同的 StringBuilder 实例

    - ```java
      private static ThreadLocal<StringBuilder> counter = new ThreadLocal<StringBuilder>()
      ```

- Thread维护ThreadLocal与实例的映射

  - 如果 Map 由 Thread 维护，从而使得每个 Thread 只访问自己的 Map，那就不存在多线程写的问题，也就不需要锁
  - 由于每个线程访问某 ThreadLocal 变量后，都会在自己的 Map 内维护该 ThreadLocal 变量与具体实例的映射，如果不删除这些引用（映射），则这些 ThreadLocal 不能被回收，可能会造成内存泄漏

- ThreadLocal 在 JDK 8 中的实现

  - Map 由 ThreadLocal 类的静态内部类 ThreadLocalMap 提供。
  - 与 HashMap 不同的是，ThreadLocalMap 的每个 Entry 都是一个对 键 的弱引用，每个 Entry 都包含了一个对 值 的强引用。
    - 使用弱引用的原因在于，当没有强引用指向 ThreadLocal 变量时，它可被回收，从而避免上文所述 ThreadLocal 不能被回收而造成的内存泄漏的问题
    - Entry虽然是弱引用，但它是 ThreadLocal 类型（键）的弱引用，非具体实例的的弱引用，所以无法避免具体实例相关的内存泄漏
    - 当 ThreadLocal 变量被回收后，该映射的键变为 null，该 Entry 无法被移除。从而使得实例被该 Entry 引用而无法被回收造成内存泄漏。
  - ThreadLocalMap 的 set 方法中，通过 replaceStaleEntry 方法将所有键为 null 的 Entry 的值设置为 null，从而使得该值可被回收





## 在Java程序中怎么保证多线程的运行安全

- 安全类、自动锁、手动锁
  - 方法一：使用安全类，比如JUC下的类
  - 方法二：使用自动锁synchronized
  - 方法三：使用互斥锁Lock
- ThreadLocal
- final
- 局部变量
  - 被分配在线程栈内存中
  - 线程的栈内存只能自己访问，所以栈内存中的变量只属于自己，其它线程根本就不知道。





## semaphore和mutex的区别

- mutex，一句话：保护共享资源。

- semaphore的用途，一句话：调度线程。

  - 调度线程，就是：一些线程生产（increase）同时另一些线程消费（decrease），semaphore可以让生产和消费保持合乎逻辑的执行顺序。
  - 线程池是通过用固定数量的线程去执行任务队列里的任务来达到避免反复创建和销毁线程而造成的资源浪费；而semaphore并没有直接提供这种机制 

- 锁是服务于共享资源的；而semaphore是服务于多个线程间的执行的逻辑顺序的。

  





## Java中线程同步锁和互斥锁的区别

- 锁的目的就是避免多个线程对同一个共享的数据并发修改带来的数据混乱。

  - 互斥就是线程A访问了一组数据，线程BCD就不能同时访问这些数据，直到A停止访问了
  - 同步就是ABCD这些线程要约定一个执行的协调顺序。比如D要执行，B和C必须都得做完，而B和C要开始，A必须先得做完
  - 互斥是通过竞争对资源的独占使用，彼此之间不需要知道对方的存在，执行顺序是一个乱序。
    同步是协调多个相互关联线程合作完成任务，彼此之间知道对方存在，执行顺序往往是有序的。
  - Mutex是专门被设计来解决互斥的；Barrier，Semaphore是专门来解决同步的
  - Java里面所说的"同步块"其实应该叫互斥块，而没有同步锁这一说法，同步是线程间的协作，Java中通过wait,notify实现

- 锁的实现要处理的大概就只有这4个问题：

  - “谁拿到了锁“这个信息存哪里（可以是当前class，当前instance的markword，还可以是某个具体的Lock的实例）
  - 谁能抢到锁的规则（只能一个人抢到 - Mutex；能抢有限多个数量 - Semaphore；自己可以反复抢 - 重入锁；读可以反复抢到但是写独占 - 读写锁……）
  - 抢不到时怎么办（抢不到玩命抢；抢不到暂时睡着，等一段时间再试/等通知再试；或者二者的结合，先玩命抢几次，还没抢到就睡着）
  - 如果锁被释放了还有其他等待锁的怎么办（不管，让等的线程通过超时机制自己抢；按照一定规则通知某一个等待的线程；通知所有线程唤醒他们，让他们一起抢……）

  





## 进程间通信

- PIPE和FIFO用来实现进程间相互发送非常短小的、频率很高的消息；这两种方式通常适用于两个进程间的通信。
- 共享内存用来实现进程间共享的、非常庞大的、读写操作频率很高的数据（配合信号量使用）；这种方式通常适用于多进程间通信。（信号量是用于同步线程间的对象的使用的）
- 其他考虑用socket。这里的“其他情况”，其实是今天主要会碰到的情况：分布式开发。在多进程、多线程、多模块所构成的今天最常见的分布式系统开发中，socket是第一选择。
- 在面向对象的今天，我们更多的时候是多线程+锁+线程间共享数据。因此共享内存在今天使用的也越来越少了。





## 线上某台虚机CPU Load过高

- 造成cpu load过高的原因： Full gc次数的增大、代码中存在Bug（例如死循环、正则的不恰当使用等）都有可能造成cpu load 增高。
  - jps -v：查看java进程号
  - top -Hp [java进程号]：查看当前进程下最耗费CPU的线程
  - printf "%x\n" [步骤2中的线程号]：得到线程的16进制表示
  - jstack [java进程号] | grep -A100 [步骤3的结果]：查看线程堆栈，定位代码行。





