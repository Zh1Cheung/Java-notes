

## 可见性、原子性和有序性问题

- 原子性：一个或者多个操作在 CPU 执行的过程中被中断

- 可见性：一个线程对共享变量的修改，另外一个线程不能立刻看到

- 有序性：程序执行的顺序没有按照代码的先后顺序执行

- 源头之一：缓存导致的可见性问题

  - 多核 CPU  count+=1

- 源头之二：线程切换带来的原子性问题

  - “时间片”
  - CPU 的使用率
  - IO 的使用率
  - 支持多进程分时复用在操作系统的发展史上却具有里程碑意义，现代的操作系统都基于更轻量的线程来调度，现在我们提到的“任务切换”都是指“线程切换”。
  - 操作系统做任务切换，可以发生在任何一条CPU 指令执行完
  - count += 1，至少需要三条 CPU 指令。
  - 把一个或者多个操作在 CPU 执行的过程中不被中断的特性称为原子性

- 源头之三：编译优化带来的有序性问题

  - 双重检查创建单例对象

    - 空指针异常

    - > instance = new Singleton();
      >
      > 1. 分配一块内存 M；
      > 2. 在内存 M 上初始化 Singleton 对象；
      > 3. 然后 M 的地址赋值给 instance 变量。
      >    但是实际上优化后的执行路径却是这样的：
      >
      > 
      >
      > 1. 分配一块内存 M；
      > 2. 将 M 的地址赋值给 instance 变量；
      > 3. 最后在内存 M 上初始化 Singleton 对象。
      >
      > 
      >
      > 优化后会导致什么问题呢？我们假设线程 A 先执行 getInstance() 方法，当执行完指令 2时恰好发生了线程切换，切换到了线程 B 上；如果此时线程 B 也执行 getInstance() 方法，那么线程 B 在执行第一个判断时会发现 instance != null ，所以直接返回instance，而此时的 instance 是没有初始化过的，如果我们这个时候访问 instance 的成员变量就可能触发空指针异常。

    - 如果对instance进行volatile语义声明，就可以禁止指令重排序，避免该情况发生

  - **下面简单谈谈针对以上的三个问题，java程序如何保证线程安全呢？**
    **针对问题1**：JDK里面提供了很多atomic类，比如AtomicInteger, AtomicLong, AtomicBoolean等等，这些类本身可以通过CAS来保证操作的原子性；另外Java也提供了各种锁机制，来保证锁内的代码块在同一时刻只能有一个线程执行，比如刚刚的例子我们就可以加锁，如下：
    `java      synchronized (Test.class){             count ++;         } `
    这样，就能够保证一个线程在多count值进行读、改、写操作时，其他线程不可对count进行操作，从而保证了线程的安全性。
     **针对问题2**：同样可以通过synchronized关键字加锁来解决。与此同时，java还提供了一种轻量级的锁，即volatile关键字，要优于synchronized的性能，同样可以保证修改对其他线程的可见性。volatile一般用于对变量的写操作不依赖于当前值的场景中，比如状态标记量等。
     **针对问题3**：可以通过synchronized关键字定义同步代码块或者同步方法保障有序性，另外也可以通过Lock接口保障有序性。

## 安全性、活跃性以及性能问题



- 安全性问题

  - 理论上线程安全的程序，就要避免出现原子性问题、可见性问题和有序性问题。

  - 数据竞争：当多个线程同时访问同一数据，并且至少有一个线程会写这个数据的时候，如果我们不采取防护措施，那么就会导致并发 Bug

  - 竞态条件：程序的执行结果依赖线程执行的顺序

    - 转账操作里面有个判断条件——转出金额不能大于账户余额，但在并发环境里面，如果不加控制，当多个线程同时对一个账号执行转出操作时，就有可能出现超额转出问题。

    - ```java
      // contains和add之间不是原子操作，有可能重复添加。
      void addIfNotExist(Vector v, Object o){
      	if(!v.contains(o)) {
      		v.add(o);
      	}
      }
      ```

    - ```java
      // 假设 count=0，当两个线程同时执行 get() 方法时，get() 方法会返回相同的值 0，两个线程执行 get()+1 操作，结果都是 1
      
      // get虽然有锁，只能保证多个线程不能同一时刻执行。但是出现不安全的可能是线程a调用get后线程b调用get,这时两个get返回的值是一样的。然后都加一后再分别set.这样两个线程就出现并发问题了。问题在于同时执行get，而在于get和set是两个方法，这两个方法组合不是原子的，就可能两个方法中间的时间也有其它线程分别调用，出现并发问题。
      
      1 public class Test {
      2 	private long count = 0;
      3 	synchronized long get(){
      4 		return count； 
      5 	}
      6 	synchronized void set(long v){
      7 		count = v;
      8 	} 
      9 	void add10K() {
      10 		int idx = 0;
      11 		while(idx++ < 10000) {
      12 			set(get()+1) 
      13 		}
      14 	}
      15 }
      ```

  - 那面对数据竞争和竞态条件问题，又该如何保证线程的安全性呢？其实这两类问题，都可以用互斥这个技术方案，而实现互斥的方案有很多，从逻辑上来看，我们可以统一归为：锁

- 活跃性问题

  - 除了死锁外，还有两种情况，分别是“活锁”和“饥饿”
    - 解决“活锁”的方案很简单，谦让时，尝试等待一个随机的时间就可以了
      - “等待一个随机时间”的方案虽然很简单，却非常有效，Raft 这样知名的分布式一致性算法中也用到了它。
    - 解决“饥饿”问题的方案很简单，有三种方案：一是保证资源充足，二是公平地分配资源，三就是避免持有锁的线程长时间执行。

- 性能问题

  - 第一，既然使用锁会带来性能问题，那最好的方案自然就是使用无锁的算法和数据结构
    - 线程本地存储 (Thread Local Storage, TLS)、写入时复制 (Copy-on-write)、乐观锁等；
    - Java 并发包里面的原子类也是一种无锁的数据结构；Disruptor 则是一个无锁的内存队列
  - 第二，减少锁持有的时间
    - 例如使用细粒度的锁，一个典型的例子就是 Java 并发包里的 ConcurrentHashMap，它使用了所谓分段锁的技术；还可以使用读写锁，也就是读是无锁的，只有写的时候才会互斥





## Java内存模型

- happens-before 

  - happens-before 关系是用来描述两个操作的内存可见性的。如果操作 X happens-before 操作 Y，那么 X 的结果对于 Y 可见。

  - 实际上，如果后者没有观测前者的运行结果，即后者没有数据依赖于前者，那么它们可能会被重排序。

  - 在同一个线程中，字节码的先后顺序（program order）也暗含了 happens-before 关系

  - 线程间的 happens-before关系

    - 解锁操作 happens-before 之后（这里指时钟顺序先后）对同一把锁的加锁操作
    - volatile 字段的写操作 happens-before 之后（这里指时钟顺序先后）对同一字段的读操作
    - 线程的启动操作（即 Thread.starts()） happens-before 该线程的第一个操作
    - 线程的最后一个操作 happens-before 它的终止事件
      - （即其他线程通过 Thread.isAlive() 或Thread.join() 判断该线程是否中止）。
    - 线程对其他线程的中断操作 happens-before 被中断线程所收到的中断事件
      - 即被中断线程的InterruptedException 异常，或者第三个线程针对被中断线程的 Thread.interrupted 或者Thread.isInterrupted 调用

  - happens-before 关系还具备传递性

    - ```java
      // 如何解决这个问题呢？答案是，将 a 或者 b 设置为 volatile 字段
      
      int a=0, b=0;
      
      public void method1() {
       	int r2 = a;
       	b = 1;
      }
      
      public void method2() {
       	int r1 = b;
       	a = 2;
      }
      ```

    - 解决这种数据竞争问题的关键在于构造一个跨线程的 happens-before 关系 ：操作X happens-before 操作 Y，使得操作 X 之前的字节码的结果对操作 Y 之后的字节码可见。

- Java 内存模型的底层实现

  - Java 内存模型是通过内存屏障来禁止重排序的。对于即时编译器来说，内存屏障将限制它所能做的重排序优化。对于处理器来说，内存屏障会导致缓存的刷新操作。
    - 对于即时编译器来说，它会针对前面提到的每一个 happens-before 关系，向正在编译的目标方法中插入相应的读读、读写、写读以及写写内存屏障。
    - X86_64 架构上读读、读写以及写写内存屏障是空操作（no-op）
    - 这些内存屏障会限制即时编译器的重排序操作
      - 以 volatile 字段访问为例，所插入的内存屏障将不允许 volatile 字段写操作之前的内存访问被重排序至其之后；也将不允许 volatile 字段读操作之后的内存访问被重排序至其之前（写前读后）
    - 即时编译器将根据具体的底层体系架构，将这些内存屏障替换成具体的 CPU 指令
  - X86_64 架构的处理器不能将读操作重排序至写操作之后
  - 只有 volatile 字段写操作之后的写读内存屏障需要用具体指令来替代
    - 该具体指令的效果，可以简单理解为强制刷新处理器的写缓存
    - 强制刷新写缓存，将使得当前线程写入 volatile 字段的值（以及写缓存中已有的其他内存修改），同步至主内存之中。

- **锁，volatile 字段，fnal 字段与安全发布**

  - 在解锁时，Java 虚拟机同样需要强制刷新缓存，使得当前线程所修改的内存对其他线程可见。

    - 锁操作的 happens-before 规则的关键字是同一把锁。也就意味着，如果编译器能够（通过逃逸分析）证明某把锁仅被同一线程持有，那么它可以移除相应的加锁解锁操作。
    - 因此也就不再强制刷新缓存。举个例子，即时编译后的 synchronized (new Object()) {}，可能等同于空操作，而不会强制刷新缓存。

  - **volatile**

    - volatile 是一个类型修饰符。volatile 的作用是作为指令关键字，确保本条指令不会因编译器的优化而省略
    - 保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。（实现可见性）禁止进行指令重排序。（实现有序性）volatile 只能保证对单次读/写的原子性。i++ 这种操作不能保证原子性。关于volatile 原子性可以理解为把对volatile变量的单个读/写，看成是使用同一个锁对这些单个读/写操作做了同步

  - volatile 字段可以看成一种轻量级的、不保证原子性的同步，其性能往往优于锁操作。

    - 频繁地访问 volatile 字段也会因为不断地强制刷新缓存而严重影响程序的性能。
      - volatile 字段的另一个特性是即时编译器无法将其分配到寄存器里。换句话说，volatile 字段的每次访问均需要直接从内存中读写。
      - 所谓的分配到寄存器中，可以理解为编译器将内存中的值缓存在寄存器中，之后一直用访问寄存器来代表对这个内存的访问的。
        - 遍历一个数组，数组的长度是内存中的值。由于我们每次循环都要比较一次，因此编译器决定把它放在寄存器中，免得每次比较都要读一次内存
        - 对于会更改的内存值，编译器也可以先缓存至寄存器，最后更新回内存即可。
      - 非volatile的
        - jvm不保证何时变量的值会写回内存。假如另一个线程加锁访问这个变量，jvm也不保证它能拿到最新数据。
        - 如果即时编译器把那个变量放在寄存器里维护，那么另一个线程也没办法
      - Volatile会禁止上述优化。

  - fnal 实例字段则涉及新建对象的发布问题。当一个对象包含 fnal 实例字段时，我们希望其他线程只能看到已初始化的 fnal 实例字段。

    - final在Java中是一个保留的关键字，可以声明成员变量、方法、类以及本地变量。一旦你将引用声明作final，你将不能改变这个引用了，编译器会检查代码，如果你试图将变量再次初始化的话，编译器会报编译错误

      - 1.修饰变量  

        凡是对成员变量或者局部变量(在方法中的或者代码块中的变量称为本地变量)声明为final的都叫作final变量。final变量经常和static关键字一起使用，作为常量。

        final修饰基本数据类型的变量时，必须赋予初始值且不能被改变，修饰引用变量时，该引用变量不能再指向其他对象

        例如：

        ![img](https://img-blog.csdn.net/20180807202158422?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0Mzc1NDcz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

        当final修饰基本数据类型变量时不赋予初始值以及引用变量指向其他对象时就会报错

        当final修饰基本数据类型变量被改变时，就会报错

        ![img](https://img-blog.csdn.net/20180807202158422?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0Mzc1NDcz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

        2.修饰方法

        final也可以声明方法。方法前面加上final关键字，代表这个方法不可以被子类的方法重写。如果你认为一个方法的功能已经足够完整了，子类中不需要改变的话，你可以声明此方法为final。final方法比非final方法要快，因为在编译的时候已经静态绑定了，不需要在运行时再动态绑定。

        ![img](https://img-blog.csdn.net/2018080720335941?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0Mzc1NDcz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

        3.修饰类

        使用final来修饰的类叫作final类。final类通常功能是完整的，它们不能被继承。Java中有许多类是final的，譬如String, Interger以及其他包装类。

        ![img](https://img-blog.csdn.net/20180807203825459?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0Mzc1NDcz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

         

      - 深入分析final关键字

        1.被final修饰的对象内容是可变的

        虽然对象被final修饰对象不可被继承，但其内容依然可以被改变

        ![img](https://img-blog.csdn.net/20180807204510390?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0Mzc1NDcz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

        2.final关键字与static对比

        static关键字修饰变量时，会使该变量在类加载时就会被初始化，不会因为对象的创建再次被加载，当变量被static 修饰时就代表该变量只会被初始化一次

        ![img](https://img-blog.csdn.net/20180807205225735?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0Mzc1NDcz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

        例如图中所示，被static修饰的变量j，虽然创建两个对象，对值并没有变化。

    - 即时编译器会在 fnal 字段的写操作后插入一个写写屏障，以防某些优化将新建对象的发布重排序至 fnal 字段的写操作之前。



## Thread中start()和run()的区别

- Thread 类实现了 Runnable 接口，而 Runnable 接口定义了唯一的一个 run() 方法

  - 如果直接调用 run() 方法，那就等于调用了一个普通的同步方法，达不到多线程运行的异步执行

- start() : 它的作用是通过本地方法start0()启动一个新线程，新线程会执行相应的run()方法执行线程的运行时代码。run()可以重复调用，而start()只能调用一次。

  - start() 方法则是 Thread 类的方法，用来异步启动一个线程，然后主线程立刻返回。
  - 该启动的线程不会马上运行，会放到等待队列中等待 CPU 调度，只有线程真正被 CPU 调度时才会调用 run() 方法执行。
  - start() 方法被标识为 synchronized 的，即为了防止被多次启动的一个同步操作。

  

## **用锁的最佳实践**

- 核心问题有两点：一个是锁有可能会变化，另一个是Integer 和 String 类型的对象不适合做锁。

  - Integer 和 String 类型的对象在 JVM 里面是可能被重用的，除此之外，JVM 里可能被重用的对象还有 Boolean，那重用意味着什么呢？意味着你的锁可能被其他代码使用，如果其他代码 synchronized(你的锁)，而且不释放，那你的程序就永远拿不到锁，这是隐藏的风险。

  - Integer会缓存-128～127这个范围内的数值，String对象同样会缓存字符串常量到字符串常量池，可供重复使用，所以不能用来用作锁对象

    - 如果100个人的项目都用这个缓存的对象做锁，还有人一直不释放，那整个系统都不了用了，锁也要隔离的

  - 锁，应是私有的、不可变的、不可重用的

    - ```java
      // 普通对象锁
      private final Object  lock = new Object();
      // 静态对象锁
      private static final Object lock = new Object();
      ```

      





## synchronized关键字

- 原理
  - 在java中，每一个对象有且仅有一个同步锁。这也意味着，同步锁是依赖于对象而存在。当我们调用某对象的synchronized方法时，就获取了该对象的同步锁
  - Synchronized 是 JVM 实现的一种内置锁，锁的获取和释放是由 JVM 隐式实现。
  - 不同线程对同步锁的访问是互斥的，对象的同步锁只能被一个线程获取到
  - Lock 同步锁是基于 Java 实现的，而 Synchronized 是基于底层操作系统的 Mutex Lock实现的，每次获取和释放锁操作都会带来用户态和内核态的切换，从而增加系统性能开销
  - Synchronized 同步锁对普通方法和静态方法的修饰有什么区别？
    - 加在普通方法锁对象是当前对象，其ObjectMonitor就是对象的，而静态方法上，锁对象就是字节码对象，静态方法是所有对象共享的，锁粒度比较大
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
  
- notify执行之后立马唤醒线程吗
  
  - 其实hotspot里真正的实现是退出同步块的时候才会去真正唤醒对应的线程
  
- notifyAll是怎么实现全唤起所有线程
  
  - JVM里没实现这么简单，而是借助了monitorexit，上面提到了当某个线程从wait状态恢复出来的时候，要先获取锁，然后再退出同步块，所以notifyAll的实现是调用notify的线程在退出其同步块的时候唤醒起最后一个进入wait状态的线程，然后这个线程退出同步块的时候继续唤醒其倒数第二个进入wait状态的线程，依次类推
  
- **在多线程中使用 Synchronized 还会发生进程间的上下文切换吗？具体又会发生在哪些环节呢？**
  - 如果一旦Synchronized锁资源竞争激烈，线程将会被阻塞，阻塞的线程将会从用户态调用内核态，尝试获取mutex，这个过程就是进程上下文切换。
  - CAS乐观锁只是一个原子操作，为CPU指令实现，不需要进入内核或者切换线程。而lock竞争锁资源是基于用户态完成，所以竞争锁资源时不会发生进程上下文切换。
  - 使用Synchronized获得锁失败，进入等待队列会发生上下文切换。如果竞争锁时锁是其他线程的偏向锁，需要升级，这时需要stop the world也会发生上下文切换
  
- **在多线程编程中，锁其实不是性能开销的根源，竞争锁才是。**
  - 减少锁的持有时间
  - 降低锁的粒度
  - 非阻塞乐观锁替代竞争锁
  - JVM 内部其实也对Synchronized 同步锁做了优化
  - 如果有多个消费者线程同时被阻塞，notifyAll() 方法，将会唤醒所有阻塞的线程。而某些商品依然没有库存，过早地唤醒这些没有库存的商品的消费线程，可能会导致线程再次进入阻塞状态，从而引起不必要的上下文切换。
    - 为了避免长时间等待，我们常会使用 Object.wait (long）设置等待超时时间
    - 建议使用 Lock 锁结合 Condition 接口替代 Synchronized 内部锁中的 wait notify，实现等待／通知。
  
- **synchronized 是重入锁吗？**

  那么问题来了，synchronized 是重入锁吗？

  你可能会说不是，因为 ReentrantLock 既然是重入锁，根据推理，相反，那 synchronized 肯定就不是重入锁，那你就错了。

  答案是：yes，为什么？看下面的例子：

  ```text
  public synchronized void operation(){
      add();
  }
  
  public synchronized void add(){
  
  }
  ```

  operation 方法调用了 add 方法，两个方法都是用 synchronized 修饰的，add()  方法可以成功获取当前线程 operation() 方法已经获取到的锁，说明 synchronized 就是可重入锁怎么实现 synchronized的

- **当声明 synchronized 代码块时**

  - 编译而成的字节码将包含 monitorenter 和 monitorexit 指令。这两种指令均会消耗操作数栈上的一个 synchronized 关键字括号里的引用( synchronized (lock))，作为所要加锁解锁的锁对象。
  - 字节码中包含一个 monitorenter 指令以及多个 monitorexit 指令。这是因为 Java虚拟机需要确保所获得的锁在正常执行路径，以及异常执行路径上都能够被解锁。

- **当用 synchronized 标记方法时**

  - 字节码中方法的访问标记包括 ACC_SYNCHRONIZED
  - 表示在进入该方法时，Java 虚拟机需要进行 monitorenter 操作。而在退出该方法时，不管是正常返回，还是向调用者抛异常，Java 虚拟机均需要进行 monitorexit 操作。
  - 这里 monitorenter 和 monitorexit 操作所对应的锁对象是隐式的。对于实例方法来说，这两个操作对应的锁对象是 this；对于静态方法来说，这两个操作对应的锁对象是则是所在类的 Class 实例。

- **关于 monitorenter 和 monitorexit 的作用，我们可以抽象地理解为每个锁对象拥有一个锁计数器和一个指向持有该锁的线程的指针**。

  - 当执行 monitorenter 时，如果目标锁对象的计数器为 0，Java 虚拟机会将该锁对象的持有线程设置为当前线程，并且将其计数器加 1。
  - 不为 0 ，如果锁对象的持有线程是当前线程，那么 Java 虚拟机可以将其计数器加 1，否则需要等待，直至持有线程释放该锁
  - 之所以采用这种计数器的方式，是为了允许同一个线程重复获取同一把锁
  - 举个例子，如果一个Java 类中拥有多个 synchronized 方法，那么这些方法之间的相互调用，不管是直接的还是间接的，都会涉及对同一把锁的重复加锁操作。因此，我们需要设计这么一个可重入的特性，来避免编程里的隐式约束。

- 重量级锁

  - Java 线程的阻塞以及唤醒，都是依靠操作系统来完成的
    - 这些操作将涉及系统调用，需要从操作系统的用户态切换至内核态，其开销非常之大
  - 为了尽量避免昂贵的线程阻塞、唤醒操作，Java 虚拟机会在线程进入阻塞状态之前，以及被唤醒后竞争不到锁的情况下，进入自旋状态，在处理器上空跑并且轮询锁是否被释放
    - Java 虚拟机给出的方案是自适应自旋，根据以往自旋等待时是否能够获得锁，来动态调整自旋的时间（循环数目）
    - 自旋状态还带来另外一个副作用，那便是不公平的锁机制。处于阻塞状态的线程，并没有办法立刻竞争被释放的锁。然而，处于自旋状态的线程，则很有可能优先获得这把锁。

- 轻量级锁

  - 多个线程在不同的时间段请求同一把锁，也就是说没有锁竞争。针对这种情形，Java 虚拟机采用了轻量级锁，来避免重量级锁的阻塞以及唤醒。

- Java 虚拟机是怎么区分轻量级锁和重量级锁的。

  - 对象头中的标记字段（mark word）。它的最后两位便被用来表示该对象的锁状态。
    - 00 代表轻量级锁，01 代表无锁（或偏向锁），10 代表重量级锁，11 则跟垃圾回收算法的标记有关。
  - 加锁操作时
    - Java 虚拟机会判断是否已经是重量级锁。如果不是，它会在当前线程的当前栈桢中划出一块空间，作为该锁的锁记录，并且将锁对象的标记字段复制到该锁记录中（假设当前锁对象的标记字段为 X…XYZ）
    - 然后，Java 虚拟机会尝试用 CAS操作替换锁对象的标记字段
      - 如果是X…X01，则替换为刚才分配的锁记录的地址。由于内存对齐的缘故，它的最后两位为 00。此时，该线程已成功获得这把锁，可以继续执行了。
      - 如果不是 X…X01，那么有两种可能。
        - 第一，该线程重复获取同一把锁。此时，Java 虚拟机会将锁记录清零，以代表该锁被重复获取
        - 第二，其他线程持有该锁。此时，Java 虚拟机会将这把锁膨胀为重量级锁，并且阻塞当前线程。
  - 解锁操作时
    - 如果当前锁记录，值为 0，则代表重复进入同一把锁，直接返回即可。
    - 否则，Java 虚拟机会尝试用 CAS 操作，比较锁对象的标记字段的值是否为当前锁记录的地址
      - 如果是，则替换为锁记录中的值，也就是锁对象原本的标记字段。此时，该线程已经成功释放这把锁。
      - 如果不是，则意味着这把锁已经被膨胀为重量级锁。此时，Java 虚拟机会进入重量级锁的释放过程，唤醒因竞争该锁而被阻塞了的线程。

- 偏向锁

  - 偏向锁针对的情况则更加乐观：从始至终只有一个线程请求某一把锁。
    - 具体来说，在线程进行加锁时，如果该锁对象支持偏向锁，那么 Java 虚拟机会通过 CAS 操作，将当前线程的地址记录在锁对象的标记字段之中，并且将标记字段的最后三位设置为 101
    - 在接下来的运行过程中，每当有线程请求这把锁，Java 虚拟机只需判断锁对象标记字段中：最后三位是否为 101，是否包含当前线程的地址，以及 epoch 值是否和锁对象的类的 epoch 值相同。如果都满足，那么当前线程持有该偏向锁，可以直接返回
  - epoch 值
    - 偏向锁的撤销
      - 当请求加锁的线程和锁对象标记字段保持的线程地址不匹配时，Java 虚拟机需要撤销该偏向锁。这个撤销过程非常麻烦，它要求持有偏向锁的线程到达安全点，再将偏向锁替换成轻量锁。
      - 总撤销数超过了一个阈值 20，Java 虚拟机会宣布这个类的偏向锁失效	
      - 如果总撤销数超过另一个阈值40， Java 虚拟机会认为这个类已经不再适合偏向锁，之后的加锁过程中直接为该类实例设置轻量级锁
    - 每个类中维护一个 epoch 值，你可以理解为第几代偏向锁，当设置偏向锁时，Java 虚拟机需要将该 epoch 值复制到锁对象的标记字段中。
    - 在宣布某个类的偏向锁失效时，Java 虚拟机实则将该类的 epoch 值加 1，表示之前那一代的偏向锁已经失效。而新设置的偏向锁则需要复制新的 epoch 值。
  - 流程
    - 访问同步方法
      - 无锁-MarkWord是否存储线程ID，
        - 是就获取偏向锁并执行方法
        - 不是就CAS操作替换线程ID
          - 成功就执行方法
          - 失败就开始撤销偏向锁
            - 原持有偏向锁的线程到达安全点，发生stw
            - 检查持有偏向锁的线程状态
              - 已退出执行方法——返回MarkWord是否存储线程ID
              - 执行方法中——升级为轻量级锁，原持有偏向锁的线程获得轻量级锁
                - CAS操作，若失败，自旋
                - 自选到达一定次数没有成功，升级为重量级锁
                  - 挂起当前线程，进入阻塞
  - 动态编译实现锁消除 / 锁粗化
    - JIT 编译器在动态编译同步块的时候，借助了一种被称为逃逸分析的技术，来判断同步块使用的锁对象是否只能够被一个线程访问，而没有被发布到其它线程。
    - 如果发现几个相邻的同步块使用的是同一个锁实例，那么 JIT 编译器将会把这几个同步块合并为一个大的同步块，从而避免一个线程“反复申请、释放同一个锁“所带来的性能开销。
  - 减小锁粒度
    - 当我们的锁对象是一个数组或队列时，集中竞争一个对象的话会非常激烈，锁也会升级为重量级锁。我们可以考虑将一个数组和队列对象拆成多个小对象，来降低锁竞争，提升并行度。
    - 最经典的减小锁粒度的案例就是 JDK1.8 之前实现的 ConcurrentHashMap 版本
    - 1.8后，可以尽量采用并发包中的无锁或则称乐观锁来实现

  

  

## 上下文切换

- 处理器给每个线程分配 CPU 时间片（Time Slice），线程在分配获得的时间片内执行任务。一个线程被暂停剥夺使用权，另外一个线程被选中开始或者继续运行的过程就叫做上下文切换（Context Switch）。

- 在这种切出切入的过程中，操作系统需要保存和恢复相应的进度信息，这个进度信息就是“上下文”了。它包括了寄存器的存储内容以及程序计数器存储的指令内容。

- 一种是程序本身触发的切换，这种我们称为自发性上下文切换，另一种是由系统或者虚拟机诱发的非自发性上下文切换。

  - 自发性上下文切换指线程由 Java 程序调用导致切出
  - 非自发性上下文切换指线程由于调度器的原因被迫切出。常见的有：线程被分配的时间片用完，虚拟机垃圾回收导致或者执行优先级的问题导致
    - 垃圾回收机制的使用有可能会导致 stop-the-world 事件的发生，这其实就是一种线程暂停行为。

- 系统开销具体发生在切换过程中的哪些具体环节

  - 操作系统保存和恢复上下文；

  - 调度器进行线程调度；

  - 处理器高速缓存重新加载

    

- **在多线程中使用 Synchronized 还会发生进程间的上下文切换吗？具体又会发生在哪些环节呢？**

  - 如果一旦Synchronized锁资源竞争激烈，线程将会被阻塞，阻塞的线程将会从用户态调用内核态，尝试获取mutex，这个过程就是进程上下文切换。
  - CAS乐观锁只是一个原子操作，为CPU指令实现，不需要进入内核或者切换线程。而lock竞争锁资源是基于用户态完成，所以竞争锁资源时不会发生进程上下文切换。
  - 使用Synchronized获得锁失败，进入等待队列会发生上下文切换。如果竞争锁时锁是其他线程的偏向锁，需要升级，这时需要stop the world也会发生上下文切换

- **在多线程编程中，锁其实不是性能开销的根源，竞争锁才是。**

  - 减少锁的持有时间
  - 降低锁的粒度
  - 非阻塞乐观锁替代竞争锁
  - JVM 内部其实也对Synchronized 同步锁做了优化
  - 如果有多个消费者线程同时被阻塞，notifyAll() 方法，将会唤醒所有阻塞的线程。而某些商品依然没有库存，过早地唤醒这些没有库存的商品的消费线程，可能会导致线程再次进入阻塞状态，从而引起不必要的上下文切换。
    - 为了避免长时间等待，我们常会使用 Object.wait (long）设置等待超时时间
    - 建议使用 Lock 锁结合 Condition 接口替代 Synchronized 内部锁中的 wait notify，实现等待／通知。

- 合理地设置线程池大小，避免创建过多线程

- 减少 Java 虚拟机的垃圾回收

  - 很多 JVM 垃圾回收器（serial 收集器、ParNew 收集器）在回收旧对象时，会产生内存碎片，从而需要进行内存整理，在这个过程中就需要移动存活的对象。
  - 而移动内存对象就意味着这些对象所在的内存地址会发生变化，因此在移动对象前需要暂停线程，在移动完成后需要再次唤醒该线程。因此减少 JVM 垃圾回收的频率可以有效地减少上下文切换。

- 竞争锁、线程间的通信以及过多地创建线程等多线程编程操作，都会给系统带来上下文切换。除此之外，I/O 阻塞以及 JVM 的垃圾回收也会增加上下文切换。

- 本质上java目前都是利用内核线程，所以都会有上下文切换**

  - Lock是通过AQS的state以及CAS操作判断是否持有锁，AQS中，阻塞线程再次获取锁时，是通过state以及CAS操作判断，只有没有竞争成功时，才会再次被挂起，这样可以尽量减少上下文切换。
  - AQS挂起是通过LockSupport中的park进入阻塞状态，这个过程也是存在进程上下文切换的。但被阻塞的线程再次获取锁时，不会产生进程上下文切换，而synchronized阻塞的线程每次获取锁资源都要通过系统调用内核来完成，这样就比AQS阻塞的线程更消耗系统资源了。
  - Doug Lea在设计AQS的线程阻塞策略使用了自旋等待和挂起两种方式，通过挂起线程前的低频自旋保证了AQS阻塞线程上下文切换开销及CUP时间片占用的最优化选择。保证在等待时间短通过自旋去占有锁而不需要挂起，而在等待时间长时将线程挂起。实现锁性能的最大化。





## interrupt()和线程终止方式

- 终止处于“阻塞状态”的线程
  - 若线程在阻塞状态时，调用了它的interrupt()方法，那么它的“中断状态”会被清除并且会收到一个InterruptedException异常
- 终止处于“运行状态”的线程
  - 通过“标记”方式终止处于“运行状态”的线程
    - isInterrupted()是判断线程的中断标记是不是为true
    - interrupt()并不会终止处于“运行状态”的线程！它会将线程的中断标记设为true
    - interrupted()除了返回中断标记之外，它还会清除中断标记(即将中断标记设为false)；而isInterrupted()仅仅返回中断标记







## 同步与异步

- 阻塞和非阻塞只是一种性质，而同步和异步是从宏观上对一种任务性质的描述
- 阻塞和非阻塞可以理解为，A请求B之后，在B端资源没准备好的时候，B是让A一直等待还是发出一个标志

## 用“等待-通知”机制优化循环等待



- 用线程阻塞的方式就能避免循环等待消耗 CPU 的问题

- 如果在 synchronized{}外部调用wait()、notify()、notifyAll() ：java.lang.IllegalMonitorStateException

- 等待队列和互斥锁是一对一的关系，每个互斥锁都有自己独立的等待队列。

- notify() 只能保证在通知时间点，条件是满足的

  - 而被通知线程的执行时间点和通知的时间点基本上不会重合，所以当线程执行的时候，很可能条件已经不满足了（保不齐有其他线程插队）。
  - 被通知的线程要想重新执行，仍然需要获取到互斥锁
  - 当 wait() 返回时，有可能条件已经发生变化了，曾经条件满足，但是现在已经不满足了，所以要重新检验条件是否满足。

  

## **wait与sleep区别**

- sleep是Thread的方法，而wait是Object类的方法
- wait会释放所有锁而sleep不会释放锁资源.
- wait只能在同步方法和同步块中使用，而sleep任何地方都可以.
- wait无需捕捉异常，而sleep需要.（都抛出InterruptedException ，wait也需要捕获异常）
- wait()无参数需要唤醒，线程状态WAITING；wait(1000L);到时间自己醒过来或者到时间之前被其他线程唤醒，状态和sleep都是TIME_WAITING
- 两者相同点：都会让步CPU执行时间，等待再次调度



## 乐观锁、悲观锁

**一、悲观锁**

总是假设最坏的情况，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻塞直到它拿到锁（共享资源每次只给一个线程使用，其它线程阻塞，用完后再把资源转让给其它线程）。传统的关系型数据库里边就用到了很多这种锁机制，比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁。Java中`synchronized`和`ReentrantLock`等独占锁就是悲观锁思想的实现。

**二、乐观锁**

总是假设最好的情况，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号机制和CAS算法实现。乐观锁适用于多读的应用类型，这样可以提高吞吐量，像数据库提供的类似于write_condition机制，其实都是提供的乐观锁。在Java中`java.util.concurrent.atomic`包下面的原子变量类就是使用了乐观锁的一种实现方式CAS实现的。

**三、两种锁的使用场景**

从上面对两种锁的介绍，我们知道两种锁各有优缺点，不可认为一种好于另一种，像乐观锁适用于写比较少的情况下（多读场景），即冲突真的很少发生的时候，这样可以省去了锁的开销，加大了系统的整个吞吐量。但如果是多写的情况，一般会经常产生冲突，这就会导致上层应用会不断的进行retry，这样反倒是降低了性能，所以一般多写的场景下用悲观锁就比较合适。

**四、乐观锁常见的两种实现方式**

乐观锁一般会使用版本号机制或CAS（Compare-and-Swap，即比较并替换）算法实现。

4.1 版本号机制

一般是在数据表中加上一个数据版本号version字段，表示数据被修改的次数，当数据被修改时，version值会加一。当线程A要更新数据值时，在读取数据的同时也会读取version值，在提交更新时，若刚才读取到的version值为当前数据库中的version值相等时才更新，否则重试更新操作，直到更新成功。

举一个简单的例子： 假设数据库中帐户信息表中有一个 version 字段，当前值为 1 ；而当前帐户余额字段（ balance ）为 $100 。

1. 操作员 A 此时将其读出（ version=1 ），并从其帐户余额中扣除 $50（ $100-$50 ）。
2. 在操作员 A 操作的过程中，操作员B 也读入此用户信息（ version=1 ），并从其帐户余额中扣除 $20 （ $100-$20 ）。
3. 操作员 A 完成了修改工作，将数据版本号加一（ version=2 ），连同帐户扣除后余额（ balance=$50 ），提交至数据库更新，此时由于提交数据版本大于数据库记录当前版本，数据被更新，数据库记录 version 更新为 2 。
4. 操作员 B 完成了操作，也将版本号加一（ version=2 ）试图向数据库提交数据（ balance=$80 ），但此时比对数据库记录版本时发现，操作员 B 提交的数据版本号为 2 ，数据库记录当前版本也为 2 ，不满足 “ 提交版本必须大于记录当前版本才能执行更新 “ 的乐观锁策略，因此，操作员 B 的提交被驳回。

这样，就避免了操作员 B 用基于 version=1 的旧数据修改的结果覆盖操作员A 的操作结果的可能。

4.2 CAS算法

即 compare and swap（比较与交换），是一种有名的无锁算法。无锁编程，即不使用锁的情况下实现多线程之间的变量同步，也就是在没有线程被阻塞的情况下实现变量的同步，所以也叫非阻塞同步（Non-blocking Synchronization）。CAS算法涉及到三个操作数

- 需要读写的内存值 V
- 进行比较的值 A
- 拟写入的新值 B

当且仅当 V 的值等于 A时，CAS通过原子方式用新值B来更新V的值，否则不会执行任何操作（比较和替换是一个原子操作）。一般情况下是一个自旋操作，即不断的重试。



**五、乐观锁的缺点**

ABA 问题是乐观锁一个常见的问题。

1. ABA 问题

如果一个变量V初次读取的时候是A值，并且在准备赋值的时候检查到它仍然是A值，那我们就能说明它的值没有被其他线程修改过了吗？很明显是不能的，因为在这段时间它的值可能被改为其他值，然后又改回A，那CAS操作就会误认为它从来没有被修改过。这个问题被称为CAS操作的 "ABA"问题。

JDK 1.5 以后的 `AtomicStampedReference 类`就提供了此种能力，其中的 `compareAndSet 方法`就是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。

2. 循环时间长开销大

自旋CAS（也就是不成功就一直循环执行直到成功）如果长时间不成功，会给CPU带来非常大的执行开销。 如果JVM能支持处理器提供的pause指令那么效率会有一定的提升，pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。第二它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起CPU流水线被清空（CPU pipeline flush），从而提高CPU的执行效率。

3. 只能保证一个共享变量的原子操作

CAS 只对单个共享变量有效，当操作涉及跨多个共享变量时 CAS 无效。但是从 JDK 1.5开始，提供了`AtomicReference类`来保证引用对象之间的原子性，你可以把多个变量放在一个对象里来进行 CAS 操作.所以我们可以使用锁或者利用`AtomicReference类`把多个共享变量合并成一个共享变量来操作。

### **六、CAS与synchronized的使用情景**

简单的来说CAS适用于写比较少的情况下（多读场景，冲突一般较少），synchronized适用于写比较多的情况下（多写场景，冲突一般较多）

- 对于资源竞争较少（线程冲突较轻）的情况，使用synchronized同步锁进行线程阻塞和唤醒切换以及用户态内核态间的切换操作额外浪费消耗cpu资源；而CAS基于硬件实现，不需要进入内核，不需要切换线程，操作自旋几率较少，因此可以获得更高的性能。
- 对于资源竞争严重（线程冲突严重）的情况，CAS自旋的概率会比较大，从而浪费更多的CPU资源，效率低于synchronized。

补充： Java并发编程这个领域中synchronized关键字一直都是元老级的角色，很久之前很多人都会称它为 “重量级锁” 。但是，在JavaSE 1.6之后进行了主要包括为了减少获得锁和释放锁带来的性能消耗而引入的 偏向锁 和 轻量级锁 以及其它各种优化之后变得在某些情况下并不是那么重了。synchronized的底层实现主要依靠 Lock-Free 的队列，基本思路是 自旋后阻塞，竞争切换后继续竞争锁，稍微牺牲了公平性，但获得了高吞吐量。在线程冲突较少的情况下，可以获得和CAS类似的性能；而线程冲突严重的情况下，性能远高于CAS。



## 死锁

- 使用细粒度锁是有代价的，这个代价就是可能会导致死锁。

- **产生死锁的的四个条件如下**：

  1、**互斥条件**：一个资源每次只能被一个进程使用；

  2、**请求与保持条件**：一个进程因请求资源而阻塞时，对已获得的资源保持不放；

  3、**不剥夺条件**：进程已获得的资源，在没使用完之前，不能强行剥夺；

  4、**循环等待条件**：多个进程之间形成一种互相循环等待资源的关系。

- 产生死锁的原因

  - 竞争不可剥夺资源

    - CPU属于可剥夺性资源,但一个进程已获得的资源，在未使用完之前，另一个进程要使用可以把它的资源剥夺过来，所以不会产生死锁。只有不可剥夺资源才会因为竞争资源而产生死锁
    - 系统中只有一台打印机，可供进程P1使用，假定P1已占用了打印机，若P2继续要求打印机打印将阻塞

  - 竞争临时资源

    - 临时资源包括硬件中断、信号、消息、缓冲区内的消息等，通常消息通信顺序进行不当，则会产生死锁

  - 进程间推进顺序非法

    - 例如，当P1运行到P1：Request（R2）时，将因R2已被P2占用而阻塞；当P2运行到P2：Request（R1）时，也将因R1已被P1占用而阻塞，于是发生进程死锁

    - ```java
      public class JavaTest {
          @Test
          public void test() {
              final Object lockA = new Object();
              final Object lockB = new Object();
              new Thread(new Runnable() {
                  @Override
                  public void run() {
                      synchronized (lockA) {
                          try {
                              Thread.sleep(1000);
                          } catch (InterruptedException e) {
                              e.printStackTrace();
                          }
                          synchronized (lockB) {
                          }
                          System.out.println("finish A");
                      }
                  }
              }).start();
              new Thread(new Runnable() {
                  @Override
                  public void run() {
                      synchronized (lockB) {
                          try {
                              Thread.sleep(1000);
                          } catch (InterruptedException e) {
                              e.printStackTrace();
                          }
                          synchronized (lockA) {
                          }
                          System.out.println("finish B");
                      }
                  }
              }).start();
              try {
                  Thread.sleep(10000);
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
          }
      }
      // 此程序中，线程 A 持有 lockA 对象，并请求 lockB 对象；线程 B 持有 lockB 对象，并请求 lockA 对象。由于他们都在等待对方释放资源，所以会产生死锁。运行程序，将发现控制台无法打印出 "finish A" 和 "finish B" 消息。
      ```

  - **2.2动态锁顺序死锁**

    我们看一下下面的例子，你认为会发生死锁吗？

    ![img](https://pic2.zhimg.com/v2-0971621be38efda732a94482a17d5afd_b.jpg)

    上面的代码**看起来是没有问题的**：锁定两个账户来判断余额是否充足才进行转账！

    但是，同样**有可能会发生死锁**：

    - 如果两个线程**同时**调用`transferMoney()`
    - 线程A从X账户向Y账户转账
    - 线程B从账户Y向账户X转账
    - 那么就会发生死锁。

    ![img](https://pic2.zhimg.com/v2-84bc1dc014f28328570a48818a84949f_b.jpg)

    **2.3协作对象之间发生死锁**

    我们来看一下下面的例子：

    ![img](https://pic3.zhimg.com/v2-ee30d364410113b21076fe01b7cf8b0d_b.jpg)接着下面代码

    ![img](https://pic4.zhimg.com/v2-375f991d8300dac414d3a50ee3863c8e_b.jpg)

    上面的`getImage()`和`setLocation(Point location)`都需要获取两个锁的

    - 并且在操作**途中是没有释放锁的**

    这就是**隐式获取两个锁**(对象之间协作)..

    这种方式**也很容易就造成死锁**.....

- 防止死锁

  - 尽量使用tryLock(long timeout,TimeUnit unit)的方法(ReentrantLock、ReentrantReadWriteLock)，设置超时时间，超时可以退出防止死锁。

  - 尽量使用java.util.concurrent并发类代替自己手写锁。

  - 尽量降低锁的使用粒度，尽量不要几个功能用同一把锁。

  - 尽量减少同步的代码块。

  - **加锁时限**：加上一个超时时间，若一个线程没有在给定的时限内成功获得所有需要的锁，则会进行回退并释放所有已经获得的锁，然后等待一段随机的时间再重试。但是如果有非常多的线程同一时间去竞争同一批资源，就算有超时和回退机制，还是可能会导致这些线程重复地尝试但却始终得不到锁。

  - **死锁检测**：死锁检测即每当一个线程获得了锁，会在线程和锁相关的数据结构中（ map 、 graph 等）将其记下。除此之外，每当有线程请求锁，也需要记录在这个数据结构中。死锁检测是一个更好的死锁预防机制，它主要是针对那些不可能实现按序加锁并且锁超时也不可行的场景。

  - **3.1固定锁顺序避免死锁**

    ![img](https://pic2.zhimg.com/v2-10abe7e842330f53c7048b05963f7d01_b.jpg)

    上面`transferMoney()`发生死锁的原因是因为**加锁顺序**不一致而出现的~

    - 如果所有线程**以固定的顺序来获得锁**，那么程序中就不会出现锁顺序死锁问题！

    针对两个特定的锁，开发者可以尝试按照锁对象的**hashCode值大小的顺序**，分别获得两个锁，这样锁总是会以特定的顺序获得锁，那么死锁也不会发生。问题变得更加复杂一些，如果此时有多个线程，都在竞争不同的锁，简单按照锁对象的hashCode进行排序（单纯按照hashCode顺序排序会出现“环路等待”），可能就无法满足要求了，这个时候开发者可以使用银行家算法，所有的锁都按照特定的顺序获取，同样可以防止死锁的发生，该算法在这里就

    那么上面的例子我们就可以**改造**成这样子：

    ![img](https://pic4.zhimg.com/v2-ce17ddce68e197027937a840291e3560_b.jpg)

    **3.2开放调用避免死锁**

    在协作对象之间发生死锁的例子中，主要是因为在**调用某个方法时就需要持有锁**，并且在方法内部也调用了其他带锁的方法！

    - **如果在调用某个方法时不需要持有锁，那么这种调用被称为开放调用**！

    我们可以这样来改造：

    - 同步代码块最好**仅被用于保护那些涉及共享状态的操作**！

    ![img](https://pic4.zhimg.com/v2-83f71d98f89a2c5d7c778a63b325c6a5_b.jpg)

    ![img](https://picb.zhimg.com/v2-f0df50fcc272fd3efc26ea19f3facdd1_b.jpg)

    使用开放调用是**非常好的一种方式**，应该尽量使用它~

- 银行家算法

  - 于是我们定义这样一个概念： **如果一组进程，按照的次序执行，并为它们分配资源，如果都每个进程都能顺利执行，那么就称这个序列为一个安全序列。否则就是一个不安全序列。**， 

    注意，这里说的是 **“一个“** 而不是说 **就是**，因为安全序列有可能不止一个。 **安全序列一定不会产生死锁，但是不安全序列只是有产生死锁的危险，并不是一定就会死锁。但是如果死锁了，那么一定是在不安全的序列下执行的。**

  - 算法

    

    ![img](https://pic1.zhimg.com/v2-86f1592b1233b2e6b3329152135110e4_b.jpg)

    

    

    ![img](https://pic1.zhimg.com/v2-1d62c9878622d69f369a3c3d3e774ee1_b.jpg)

    ![img](https://pic3.zhimg.com/v2-42d9548f2b2a4315bcd15ce93242aee2_b.jpg)

    

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

  

## AtomicInteger底层实现原理

- AtomicIntger 是对 int 类型的一个封装，提供原子性的访问和更新操作，其原子性操作的实现是基于 CAS（compare-and-swap）技术
- 所谓 CAS，表征的是一些列操作的集合，获取当前数值，进行一些运算，利用 CAS 指令试图进行更新。如果当前数值未变，代表没有其他线程进行并发修改，则成功更新。否则，可能出现不同的选择，要么进行重试，要么就返回一个成功或者失败的结果
  - 从 AtomicInteger 的内部属性可以看出，它依赖于 Unsafe 提供的一些底层能力，进行底层操作。以 volatile 的 value 字段，记录数值，以保证可见性
  - 具体的原子操作细节，可以参考任意一个原子更新方法，比如 getAndIncrement（）
    - Unsafe 会利用 value 字段的内存地址偏移，直接完成操作
  - CAS 是 Java 并发中所谓 lock-free 机制的基础
  - CAS 更加底层是如何实现的，这依赖于 CPU 提供的特定指令，具体根据体系结构的不同还存在着明显区别。比如，x86 CPU 提供 cmpxchg 指令；而在精简指令集的体系架构中，则通常是靠一对儿指令（如“load and reserve”和“store conditional”）实现的，在大多数处理器上 CAS 都是个非常轻量级的操作，这也是其优势所在
  - CAS 也并不是没有副作用，试想，其常用的失败重试机制，隐含着一个假设，即竞争情况是短暂的。大多数应用场景中，确实大部分重试只会发生一次就获得了成功，但是总是有意外情况，所以在有需要的时候，还是要考虑限制自旋的次数，以免过度消耗 CPU。
  - 另外一个就是著名的ABA问题，这是通常只在 lock-free 算法下暴露的问题。我前面说过CAS 是在更新时比较前值，如果对方只是恰好相同，例如期间发生了 A -> B -> A 的更新，仅仅判断数值是 A，可能导致不合理的修改操作。针对这种情况，Java 提供了AtomicStampedReference 工具类，通过为引用建立类似版本号（stamp）的方式，来保证 CAS 的正确性
  - **大多数情况下，Java 开发者并不需要直接利用CAS 代码去实现线程安全容器等，更多是通过并发包等间接享受到 lock-free 机制在扩展性上的好处。**



## AQS

### 介绍

- AQS是用来构建锁和其他同步组件的基础框架（ Java 并发包中，实现各种同步结构和部分其他组成单元（如线程池中的 Worker）的基础）

- 每个类内部都包含一个如下的内部类定义abstract static class Sync extends AbstractQueuedSynchronizer        

  - 每个方法都是一个风格，就是换个名直接调用sync的对应方法
  - 几种同步类提供的功能其实都是委托sync来完成

- AQS中主要维护了state（锁状态的表示）和一个可阻塞的等待队列。

  - ```java
    // 1. 一个 volatile 的整数成员表征状态，同时提供了 setState 和 getState 方法
    // 2. 一个先入先出（FIFO）的等待线程队列，以实现多线程间竞争和等待，这是 AQS 机制的核心之一。
    // 3. 各种基于 CAS 的基础操作方法，以及各种期望具体同步结构去实现的 acquire/release方法
    private transient volatile Node head;
    private transient volatile Node tail;
    private volatile int state;
    ```

- **本质上java目前都是利用内核线程，所以都会有上下文切换**

  - Lock是通过AQS的state以及CAS操作判断是否持有锁，AQS中，阻塞线程再次获取锁时，是通过state以及CAS操作判断，只有没有竞争成功时，才会再次被挂起，这样可以尽量减少上下文切换。
  - AQS挂起是通过LockSupport中的park进入阻塞状态，这个过程也是存在进程上下文切换的。但被阻塞的线程再次获取锁时，不会产生进程上下文切换，而synchronized阻塞的线程每次获取锁资源都要通过系统调用内核来完成，这样就比AQS阻塞的线程更消耗系统资源了。
  - Doug Lea在设计AQS的线程阻塞策略使用了自旋等待和挂起两种方式，通过挂起线程前的低频自旋保证了AQS阻塞线程上下文切换开销及CUP时间片占用的最优化选择。保证在等待时间短通过自旋去占有锁而不需要挂起，而在等待时间长时将线程挂起。实现锁性能的最大化。

### 方法

- 利用 AQS 实现一个同步结构，至少要实现两个基本类型的方法，分别是 acquire 操作，获取资源的独占权；还有就是 release 操作，释放对某个资源的独占
- acquire(int arg)-- 获取排他锁
  - ReentrantLock.lock()中调用了这个方法
    - 共性：都属于独占锁的实现，任意一个时刻只有一个线程能够获取到锁。都支持可重入。
    - 区别：a.synchronized属于JVM层面的管程实现，ReentrantLock属于Java语言层面实现的管程。
  - ReentrantLock有synchronized不具备的特性：响应中断、支持超时、支持非阻塞式地获取锁，公平锁（在构造方法中传参true），支持多个等待队列。
- 以非公平的 tryAcquire 为例，其内部实现了如何配合状态与 CAS 获取锁
  - state=0 表示无人占有，则直接用 CAS 修改状态位，
  - 即使状态不是 0，也可能当前线程是锁持有者，因为这是重入锁
- 再来分析 acquireQueued，如果前面的 tryAcquire 失败，代表着锁争抢失败，进入排队竞争阶段。这里就是我们所说的，利用 FIFO 队列，实现线程间对锁的竞争的部分，算是是 AQS 的核心逻辑。
  - 当前线程会被包装成为一个排他模式的节点（EXCLUSIVE），通过 addWaiter 方法添加到队列中。acquireQueued 的逻辑，简要来说，就是如果当前节点的前面是头节点，则试图获取锁，一切顺利则成为新的头节点；否则，有必要则等待
  - 到这里线程试图获取锁的过程基本展现出来了，tryAcquire 是按照特定场景需要开发者去实现的部分，而线程间竞争则是 AQS 通过 Waiter 队列与 acquireQueued 提供的，在 release 方法中，同样会对队列进行对应操作。
- AQS 中 Node 的 waitStatus 有什么作用？
  - CANCELLED 1 因为超时或中断设置为此状态，标志节点不可用
  - SIGNAL -1 处于此状态的节点释放资源时会唤醒后面的节点
  - CONDITION -2 处于条件队列里，等待条件成立(signal signalall) 条件成立后会置入获取资源的队列里
  - PROPAGATE -3 共享模式下使用，头节点获取资源时将后面节点设置为此状态，如果头节点获取资源后还有足够的资源，则后面节点会尝试获取，这个状态主要是为了共享状态下队列里足够多的节点同时获取资源
  - 0 初始状态



## Condition

- 介绍

  - Object的wait和notify/notify是与对象监视器配合完成线程间的等待/通知机制，Condition与Lock配合完成等待/通知机制
  - 前者是java底层级别的，后者是语言级别的，具有更高的可控制性和扩展性
  - Condition能够支持不响应中断，而通过使用Object方式不支持；
  - Condition能够支持多个等待队列（new 多个Condition对象），而Object方式只能支持一个；
  - Condition能够支持超时时间的设置，而Object不支持
  - 等待/通知机制，通过使用condition提供的await和signal/signalAll方法就可以实现这种机制，而这种机制能够解决最经典的问题就是“生产者与消费者问题”

- 参照Object的wait和notify/notifyAll方法，Condition也提供了同样的方法：

  - await() ，当前线程进入等待状态，如果其他线程调用condition的signal或者signalAll方法并且当前线程获取Lock从await方法返回，如果在等待状态中被中断会抛出被中断异常；
  - awaitNanos(long nanosTimeout)：当前线程进入等待状态直到被通知，中断或者超时；
  - await(long time, TimeUnit unit)：同第二种，支持自定义时间单位
  - awaitUntil(Date deadline) ：当前线程进入等待状态直到被通知，中断或者到了某个时间

- 针对Object的notify/notifyAll方法

  - signal()：唤醒一个等待在condition上的线程，将该线程从等待队列中转移到同步队列中，如果在同步队列中能够竞争到Lock则可以从等待方法中返回。
  - signalAll()：与1的区别在于能够唤醒所有等待在condition上的线程

- await实现原理

  - await()：当前线程处于阻塞状态，直到调用signal()或中断才能被唤醒。

    1）将当前线程封装成node且等待状态为CONDITION。
    2）释放当前线程持有的所有锁，让下一个线程能获取锁。
    3）加入到等待队列后，则阻塞当前线程，等待被唤醒。
    4）如果是因signal被唤醒，则节点会从等待队列转移到同步队列；如果是因中断被唤醒，则记录中断状态。两种情况都会跳出循环。
    5）若是因signal被唤醒，就自旋获取锁；否则处理中断异常。

- signal/signalAll实现原理

  - doSignal方法：将等待队列的头节点转移到同步队列

## AQS的源码分析

清楚了AQS的基本架构以后，我们来分析一下AQS的源码，仍然以ReentrantLock为模型。

**ReentrantLock的时序图**

调用ReentrantLock中的lock()方法，源码的调用过程我使用了时序图来展现
![ReentrantLock中lock方法的时序图](https://segmentfault.com/img/remote/1460000017372074?w=915&h=409)
从图上可以看出来，当锁获取失败时，会调用addWaiter()方法将当前线程封装成Node节点加入到AQS队列，基于这个思路，我们来分析AQS的源码实现

### 分析源码

#### ReentrantLock.lock()

```
public void lock() {
    sync.lock();
}
```

这个是获取锁的入口，调用sync这个类里面的方法，sync是什么呢？

```
abstract static class Sync extends AbstractQueuedSynchronizer
```

sync是一个静态内部类，它继承了AQS这个抽象类，前面说过AQS是一个同步工具，主要用来实现同步控制。我们在利用这个工具的时候，会继承它来实现同步控制功能。
通过进一步分析，发现Sync这个类有两个具体的实现，分别是`NofairSync(非公平锁)`,`FailSync(公平锁)`.

- 公平锁 表示所有线程严格按照FIFO来获取锁
- 非公平锁 表示可以存在抢占锁的功能，也就是说不管当前队列上是否存在其他线程等待，新线程都有机会抢占锁

公平锁和非公平锁的实现上的差异，我会在文章后面做一个解释，接下来的分析仍然以`非公平锁`作为主要分析逻辑。

#### NonfairSync.lock

```
final void lock() {
    if (compareAndSetState(0, 1)) //通过cas操作来修改state状态，表示争抢锁的操作
      setExclusiveOwnerThread(Thread.currentThread());//设置当前获得锁状态的线程
    else
      acquire(1); //尝试去获取锁
}
```

这段代码简单解释一下

- 由于这里是非公平锁，所以调用lock方法时，先去通过cas去抢占锁
- 如果抢占锁成功，保存获得锁成功的当前线程
- 抢占锁失败，调用acquire来走锁竞争逻辑

> **compareAndSetState**
> compareAndSetState的代码实现逻辑如下

```
// See below for intrinsics setup to support this
return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
```

}

> ```
> 这段代码其实逻辑很简单，就是通过cas乐观锁的方式来做比较并替换。上面这段代码的意思是，如果当前内存中的state的值和预期值expect相等，则替换为update。更新成功返回true，否则返回false.
> 这个操作是原子的，不会出现线程安全问题，这里面涉及到Unsafe这个类的操作，一级涉及到state这个属性的意义。
> **state**
> ```

- 当state=0时，表示无锁状态
- 当state>0时，表示已经有线程获得了锁，也就是state=1，但是因为ReentrantLock允许重入，所以同一个线程多次获得同步锁的时候，state会递增，比如重入5次，那么state=5。 而在释放锁的时候，同样需要释放5次直到state=0其他线程才有资格获得锁

> ```
> private volatile int state;
> ```
>
> 需要注意的是：不同的AQS实现，state所表达的含义是不一样的。
> **Unsafe**
> Unsafe类是在sun.misc包下，不属于Java标准。但是很多Java的基础类库，包括一些被广泛使用的高性能开发库都是基于Unsafe类开发的，比如Netty、Hadoop、Kafka等；Unsafe可认为是Java中留下的后门，提供了一些低层次操作，如直接内存访问、线程调度等
>
> ```
> public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
> ```
>
> 这个是一个native方法， 第一个参数为需要改变的对象，第二个为偏移量(即之前求出来的headOffset的值)，第三个参数为期待的值，第四个为更新后的值
> 整个方法的作用是如果当前时刻的值等于预期值var4相等，则更新为新的期望值 var5，如果更新成功，则返回true，否则返回false；

#### acquire

acquire是AQS中的方法，如果CAS操作未能成功，说明state已经不为0，此时继续acquire(1)操作,这里大家思考一下，acquire方法中的1的参数是用来做什么呢？如果没猜中，往前面回顾一下state这个概念

```
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

这个方法的主要逻辑是

- 通过tryAcquire尝试获取独占锁，如果成功返回true，失败返回false
- 如果tryAcquire失败，则会通过addWaiter方法将当前线程封装成Node添加到AQS队列尾部
- acquireQueued，将Node作为参数，通过自旋去尝试获取锁。

> 如果大家看过我写的[Synchronized源码分析](https://mp.weixin.qq.com/s?__biz=MzI0MzI1Mjg5Nw==&mid=2247483699&idx=1&sn=9e51113bbbb3ae94d6b7273f3ee1b00f&chksm=e96eaafdde1923eb6d3f721c902335c54037b503d5a3d7693e30246efa8356c41ea17bcfacc5&token=1402731013&lang=zh_CN#rd)的文章，就应该能够明白自旋存在的意义

#### NonfairSync.tryAcquire

这个方法的作用是尝试获取锁，如果成功返回true，不成功返回false
它是重写AQS类中的tryAcquire方法，并且大家仔细看一下AQS中tryAcquire方法的定义，并没有实现，而是抛出异常。按照一般的思维模式，既然是一个不实现的模版方法，那应该定义成abstract，让子类来实现呀？大家想想为什么

```
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```

#### nonfairTryAcquire

tryAcquire(1)在NonfairSync中的实现代码如下

```
ffinal boolean nonfairTryAcquire(int acquires) {
    //获得当前执行的线程
    final Thread current = Thread.currentThread();
    int c = getState(); //获得state的值
    if (c == 0) { //state=0说明当前是无锁状态
        //通过cas操作来替换state的值改为1，大家想想为什么要用cas呢？
        //理由是，在多线程环境中，直接修改state=1会存在线程安全问题，你猜到了吗？
        if (compareAndSetState(0, acquires)) {
             //保存当前获得锁的线程
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //这段逻辑就很简单了。如果是同一个线程来获得锁，则直接增加重入次数
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires; //增加重入次数
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

- 获取当前线程，判断当前的锁的状态
- 如果state=0表示当前是无锁状态，通过cas更新state状态的值
- 如果当前线程是属于重入，则增加重入次数

#### addWaiter

当tryAcquire方法获取锁失败以后，则会先调用addWaiter将当前线程封装成Node，然后添加到AQS队列

```
private Node addWaiter(Node mode) { //mode=Node.EXCLUSIVE
        //将当前线程封装成Node，并且mode为独占锁
        Node node = new Node(Thread.currentThread(), mode); 
        // Try the fast path of enq; backup to full enq on failure
        // tail是AQS的中表示同步队列队尾的属性，刚开始为null，所以进行enq(node)方法
        Node pred = tail;
        if (pred != null) { //tail不为空的情况，说明队列中存在节点数据
            node.prev = pred;  //讲当前线程的Node的prev节点指向tail
            if (compareAndSetTail(pred, node)) {//通过cas讲node添加到AQS队列
                pred.next = node;//cas成功，把旧的tail的next指针指向新的tail
                return node;
            }
        }
        enq(node); //tail=null，将node添加到同步队列中
        return node;
    }
```

- 将当前线程封装成Node
- 判断当前链表中的tail节点是否为空，如果不为空，则通过cas操作把当前线程的node添加到AQS队列
- 如果为空或者cas失败，调用enq将节点添加到AQS队列

#### enq

enq就是通过自旋操作把当前节点加入到队列中

```
private Node enq(final Node node) {
        //自旋，不做过多解释，不清楚的关注公众号[架构师修炼宝典]
        for (;;) {
            Node t = tail; //如果是第一次添加到队列，那么tail=null
            if (t == null) { // Must initialize
                //CAS的方式创建一个空的Node作为头结点
                if (compareAndSetHead(new Node()))
                   //此时队列中只一个头结点，所以tail也指向它
                    tail = head;
            } else {
//进行第二次循环时，tail不为null，进入else区域。将当前线程的Node结点的prev指向tail，然后使用CAS将tail指向Node
                node.prev = t;
                if (compareAndSetTail(t, node)) {
//t此时指向tail,所以可以CAS成功，将tail重新指向Node。此时t为更新前的tail的值，即指向空的头结点，t.next=node，就将头结点的后续结点指向Node，返回头结点
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

假如有两个线程t1,t2同时进入enq方法，t==null表示队列是首次使用，需要先初始化
另外一个线程cas失败，则进入下次循环，通过cas操作将node添加到队尾

> 到目前为止，通过addwaiter方法构造了一个AQS队列，并且将线程添加到了队列的节点中

#### acquireQueued

将添加到队列中的Node作为参数传入acquireQueued方法，这里面会做抢占锁的操作

```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();// 获取prev节点,若为null即刻抛出NullPointException
            if (p == head && tryAcquire(arg)) {// 如果前驱为head才有资格进行锁的抢夺
                setHead(node); // 获取锁成功后就不需要再进行同步操作了,获取锁成功的线程作为新的head节点
//凡是head节点,head.thread与head.prev永远为null, 但是head.next不为null
                p.next = null; // help GC
                failed = false; //获取锁成功
                return interrupted;
            }
//如果获取锁失败，则根据节点的waitStatus决定是否需要挂起线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())// 若前面为true,则执行挂起,待下次唤醒的时候检测中断的标志
                interrupted = true;
        }
    } finally {
        if (failed) // 如果抛出异常则取消锁的获取,进行出队(sync queue)操作
            cancelAcquire(node);
    }
}
```

- 获取当前节点的prev节点
- 如果prev节点为head节点，那么它就有资格去争抢锁，调用tryAcquire抢占锁
- 抢占锁成功以后，把获得锁的节点设置为head，并且移除原来的初始化head节点
- 如果获得锁失败，则根据waitStatus决定是否需要挂起线程
- 最后，通过cancelAcquire取消获得锁的操作

前面的逻辑都很好理解，主要看一下shouldParkAfterFailedAcquire这个方法和parkAndCheckInterrupt的作用

#### shouldParkAfterFailedAcquire

从上面的分析可以看出，只有队列的第二个节点可以有机会争用锁，如果成功获取锁，则此节点晋升为头节点。对于第三个及以后的节点，if (p == head)条件不成立，首先进行shouldParkAfterFailedAcquire(p, node)操作
shouldParkAfterFailedAcquire方法是判断一个争用锁的线程是否应该被阻塞。它首先判断一个节点的前置节点的状态是否为Node.SIGNAL，如果是，是说明此节点已经将状态设置-如果锁释放，则应当通知它，所以它可以安全的阻塞了，返回true。

```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus; //前继节点的状态
    if (ws == Node.SIGNAL)//如果是SIGNAL状态，意味着当前线程需要被unpark唤醒
               return true;
如果前节点的状态大于0，即为CANCELLED状态时，则会从前节点开始逐步循环找到一个没有被“CANCELLED”节点设置为当前节点的前节点，返回false。在下次循环执行shouldParkAfterFailedAcquire时，返回true。这个操作实际是把队列中CANCELLED的节点剔除掉。
    if (ws > 0) {// 如果前继节点是“取消”状态，则设置 “当前节点”的 “当前前继节点” 为 “‘原前继节点'的前继节点”。
       
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else { // 如果前继节点为“0”或者“共享锁”状态，则设置前继节点为SIGNAL状态。
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

#### parkAndCheckInterrupt

如果shouldParkAfterFailedAcquire返回了true，则会执行：`parkAndCheckInterrupt()`方法，它是通过LockSupport.park(this)将当前线程挂起到WATING状态，它需要等待一个中断、unpark方法来唤醒它，通过这样一种FIFO的机制的等待，来实现了Lock的操作。

```
private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
}
```

> **LockSupport**
> LockSupport类是Java6引入的一个类，提供了基本的线程同步原语。LockSupport实际上是调用了Unsafe类里的函数，归结到Unsafe里，只有两个函数：
>
> ```
> public native void unpark(Thread jthread);  
> public native void park(boolean isAbsolute, long time);  
> ```
>
> unpark函数为线程提供“许可(permit)”，线程调用park函数则等待“许可”。这个有点像信号量，但是这个“许可”是不能叠加的，“许可”是一次性的。
> permit相当于0/1的开关，默认是0，调用一次unpark就加1变成了1.调用一次park会消费permit，又会变成0。 如果再调用一次park会阻塞，因为permit已经是0了。直到permit变成1.这时调用unpark会把permit设置为1.每个线程都有一个相关的permit，permit最多只有一个，重复调用unpark不会累积

## 锁的释放

### ReentrantLock.unlock

加锁的过程分析完以后，再来分析一下释放锁的过程，调用release方法，这个方法里面做两件事，1，释放锁 ；2，唤醒park的线程

```
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

### tryRelease

这个动作可以认为就是一个设置锁状态的操作，而且是将状态减掉传入的参数值（参数是1），如果结果状态为0，就将排它锁的Owner设置为null，以使得其它的线程有机会进行执行。
在排它锁中，加锁的时候状态会增加1（当然可以自己修改这个值），在解锁的时候减掉1，同一个锁，在可以重入后，可能会被叠加为2、3、4这些值，只有unlock()的次数与lock()的次数对应才会将Owner线程设置为空，而且也只有这种情况下才会返回true。

```
protected final boolean tryRelease(int releases) {
    int c = getState() - releases; // 这里是将锁的数量减1
    if (Thread.currentThread() != getExclusiveOwnerThread())// 如果释放的线程和获取锁的线程不是同一个，抛出非法监视器状态异常
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) { 
// 由于重入的关系，不是每次释放锁c都等于0，
    // 直到最后一次释放锁时，才会把当前线程释放
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

### unparkSuccessor

在方法unparkSuccessor(Node)中，就意味着真正要释放锁了，它传入的是head节点（head节点是占用锁的节点），当前线程被释放之后，需要唤醒下一个节点的线程

```
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {//判断后继节点是否为空或者是否是取消状态,
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0) //然后从队列尾部向前遍历找到最前面的一个waitStatus小于0的节点, 至于为什么从尾部开始向前遍历，因为在doAcquireInterruptibly.cancelAcquire方法的处理过程中只设置了next的变化，没有设置prev的变化，在最后有这样一行代码：node.next = node，如果这时执行了unparkSuccessor方法，并且向后遍历的话，就成了死循环了，所以这时只有prev是稳定的
                s = t;
    }
//内部首先会发生的动作是获取head节点的next节点，如果获取到的节点不为空，则直接通过：“LockSupport.unpark()”方法来释放对应的被挂起的线程，这样一来将会有一个节点唤醒后继续进入循环进一步尝试tryAcquire()方法来获取锁
    if (s != null)
        LockSupport.unpark(s.thread); //释放许可
}
```

## 总结

通过这篇文章基本将AQS队列的实现过程做了比较清晰的分析，主要是基于非公平锁的独占锁实现。在获得同步锁时，同步器维护一个同步队列，获取状态失败的线程都会被加入到队列中并在队列中进行自旋；移出队列（或停止自旋）的条件是前驱节点为头节点且成功获取了同步状态。在释放同步状态时，同步器调用tryRelease(int arg)方法释放同步状态，然后唤醒头节点的后继节点。









## Lock和Condition

- Lock 用于解决互斥问题，Condition 用于解决同步问题
- 对于“不可抢占”这个条件，占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占有的资源，这样不可抢占这个条件就破坏掉了。
  - 能够响应中断
    - 持有锁 A 后，如果尝试获取锁 B 失败，那么线程就进入阻塞状态，如果阻塞状态的线程能够响应中断信号，也就是说当我们给阻塞的线程发送中断信号的时候，能够唤醒它，那它就有机会释放曾经持有的锁 A。这样就破坏了不可抢占条件了。
  - 支持超时
  - 非阻塞地获取锁
    - 如果尝试获取锁失败，并不进入阻塞状态，而是直接返回，那这个线程也有机会释放曾经持有的锁。这样也能破坏不可抢占条件
      - 用非阻塞的方式去获取锁，破坏了产生死锁的四个条件之一的“不可抢占”。所以不会产生死锁。
      - 有可能活锁，A，B两账户相互转账，各自持有自己lock的锁，都一直在尝试获取对方的锁，形成了活锁。加个随机重试时间避免活锁

```java
   // 支持中断的 API
   void lockInterruptibly() throws InterruptedException;
   // 支持超时的 API
   boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
   // 支持非阻塞获取锁的 API
   boolean tryLock();
```

- Java SD里面的 ReentrantLock，内部持有一个 volatile 的成员变量 state，获取锁的时候，会读写state 的值；解锁的时候，也会读写 state 的值

  ```java
  // 根据相关的 Happens-Before 规则：
  //  顺序性规则：对于线程 T1，value+=1 Happens-Before 释放锁的操作 unlock()；
  //  volatile 变量规则：由于 state = 1 会先读取 state，所以线程 T1 的 unlock() 操作Happens-Before 线程 T2 的 lock() 操作；
  //  传递性规则：线程 T1 的 value+=1 Happens-Before 线程 T2 的 lock() 操作。
  
  1 class X {
  2 	private final Lock rtl = new ReentrantLock();
  4 	int value;
  5 	public void addOne() {
  6 	// 获取锁
  7 		rtl.lock(); 
  8 		try {
  9 			value+=1;
  10 		} finally {
  11 // 保证锁能释放
  12 			rtl.unlock();
  13 }
  
  ```

- 可重入锁

- 公平锁与非公平锁

- 减少锁的持有时间、减小锁的粒度

  - 永远只在更新对象的成员变量时加锁
  - 永远只在访问可变的成员变量时加锁
  - 永远不在调用其他对象的方法时加锁

- Lock&Condition 实现的管程是支持多个条件变量的

- signal唤醒任意一个线程竞争锁，signalAll唤醒同一个条件变量的所有线程竞争锁。但都只有一个线程获得锁执行。区别只是被唤醒线程的数量。 所以用signalall可以避免极端情况线程只能等待超时

- 同步与异步

  - 调用方是否需要等待结果，如果需要等待结果，就是同步；如果不需要等待结果，就是异步。
  - 调用方创建一个子线程，在子线程中执行方法调用，这种调用我们称为异步调用；
  - 方法实现的时候，创建一个新的线程执行主要逻辑，主线程直接 return，这种方法我们一般称为异步方法。

- ReentrantLock**对比**Synchronized 

  - 性能比较

    - ReentrantLock是Lock的实现类，是一个互斥的同步器，在多线程高竞争条件下，ReentrantLock比synchronized有更加优异的性能表现。
    - Lock使用起来比较灵活，但是必须有释放锁的配合动作
      Lock必须手动获取与释放锁，而synchronized不需要手动释放和开启锁。Lock只适用于代码块锁，而synchronized可用于修饰方法、代码块等 
    - 从性能方面上来说，在并发量不高、竞争不激烈的情况下，Synchronized 同步锁由于具有分级锁的优势，性能上与Lock 锁差不多；但在高负载、高并发的情况下，Synchronized 同步锁由于竞争激烈会升级到重量级锁，性能则没有 Lock 锁稳定。
    
  -  特性比较

    - ReentrantLock的优势体现在：
      具备尝试非阻塞地获取锁的特性：当前线程尝试获取锁，如果这一时刻锁没有被其他线程获取到，则成功获取并持有锁
    - 能被中断地获取锁的特性：与synchronized不同，获取到锁的线程能够响应中断，当获取到锁的线程被中断时，中断异常将会被抛出，同时锁会被释放
    - 超时获取锁的特性：在指定的时间范围内获取锁；如果截止时间到了仍然无法获取锁，则返回

    - 3 注意事项
      在使用ReentrantLock类的时，一定要注意三点：
      在finally中释放锁，目的是保证在获取锁之后，最终能够被释放
      不要将获取锁的过程写在try块内，因为如果在获取锁时发生了异常，异常抛出的同时，也会导致锁无故被释放。
      ReentrantLock提供了一个newCondition的方法，以便用户在同一锁的情况下可以根据不同的情况执行等待或唤醒的动作。  

- 所有的Lock都是基于AQS来实现了。

  - AQS和Condition各自维护了不同的队列，在使用lock和condition的时候，其实就是两个队列的互相移动。
  - 如果我们想自定义一个同步器，可以实现AQS。它提供了获取共享锁和互斥锁的方式，都是基于对state操作而言的。
  - ReentranLock这个是可重入的。它内部自定义了同步器Sync，这个又实现了AQS，同时又实现了AOS，而后者就提供了一种互斥锁持有的方式。其实就是每次获取锁的时候，看下当前维护的那个线程和当前请求的线程是否一样，一样就可重入了



## Semaphore

- 简单概括为：一个计数器，一个等待队列，三个方法
  - init()：设置计数器的初始值。
  - down()：计数器的值减 1；如果此时计数器的值小于 0，则当前线程将被阻塞
  - up()：计数器的值加 1；如果此时计数器的值小于或者等于 0，则唤醒等待队列中的一个线程，并将其从等待队列中移除。
  - init()、down() 和 up() 三个方法都是原子性的，并且这个原子性是由信号量模型的实现方保证的。Semaphore 这个类能够保证这三个方法都是原子操作。
  - 在 Java SDK 并发包里，down() 和 up() 对应的则是 acquire() 和 release()。 
- 和管程相比，信号量可以实现的独特功能就是同时允许多个线程进入临界区
  - 我们把计数器的值设置成对象池里对象的个数 N，就能完美解决对象池的限流问题了。
  - 但是信号量不能做的就是同时唤醒多个线程去争抢锁，只能唤醒一个阻塞中的线程
  - 而且信号量模型是没有Condition的概念的，即阻塞线程被醒了直接就运行了而不会去检查此时临界条件是否已经不满足了
- Semaphore 可以允许多个线程访问一个临界区，那就意味着可能存在多个线程同时访问 ArrayList，而 ArrayList 不是线程安全的
- Semaphore 允许多个线程访问一个临界区，这也是一把双刃剑，当多个线程进入临界区时，如果需要访问共享变量就会存在并发问题，所以必须加锁，也就是说 Semaphore 需要锁中锁。



## ReadWriteLock

- 读写锁

  - > 1. 允许多个线程同时读共享变量；
    > 2. 只允许一个线程写共享变量；
    > 3. 如果一个写线程正在执行写操作，此时禁止读线程读共享变量。
    >
    > 
    >
    > ReadWriteLock 是一个接口，它的实现类是 ReentrantReadWriteLock

- 非常普遍的并发场景：读多写少场景

- 读写锁 ReentrantReadWriteLock是如何实现锁分离来保证共享资源的原子性的

  - RRW 也是基于 AQS 实现的，它的自定义同步器（继承AQS）需要在同步状态 state 上维护多个读线程和一个写线程的状态，该状态的设计成为实现读写锁的关键
  - 高低位，来实现一个整型控制两种状态的功能，读写锁将变量切分成了两个部分，高 16 位表示读，低 16位表示写。
  - 一个线程尝试获取写锁时
    - 会先判断同步状态 state 是否为0。如果 state 等于 0，说明暂时没有其它线程获取锁；如果 state 不等于 0，则说明有其它线程获取了锁。
      - 再判断同步状态 state 的低 16 位（w）是否为 0，为 0，则说明其它线程获取了读锁，此时进入 CLH 队列进行阻塞等待
      - 如果 w 不为 0，则说明其它线程获取了写锁，此时要判断获取了写锁的是不是当前线程，若不是就进入 CLH 队列进行阻塞等待；若是，就应该判断当前线程获取写锁是否超过了最大次数，若超过，抛异常，反之更新同步状态。
  - StampLock不支持重入，不支持条件变量，线程被中断时可能导致CPU暴涨
  - StampedLock在写多读少的时候性能会很差
  - 进程上下文切换，是指用户态和内核态的来回切换。我们知道，如果一旦Synchronized锁资源竞争激烈，线程将会被阻塞，阻塞的线程将会从用户态调用内核态，尝试获取mutex，这个过程就是进程上下文切换。

  

- 实现缓存的按需加载

  - 在高并发的场景下，有可能会有多线程竞争写锁
  - 在获取写锁之后，我们并没有直接去查询数据库，而是重新验证了一次缓存中是否存在，再次验证如果还是不存在
  - 再次验证的方式，能够避免高并发场景下重复查询数据的问题。

- 读写锁的升级与降级

  - 写锁可以降级为读锁：也好理解，本线程在释放写锁之前，获取读锁一定是可以立刻获取到的，不存在其他线程持有读锁或者写锁（读写锁互斥），所以java允许锁降级
  - 本线程在释放读锁之前，想要获取写锁是不一定能获取到的，因为其他线程可能持有读锁（读锁共享），可能导致阻塞较长的时间，所以java干脆直接不支持读锁升级为写锁。

- 解决数据同步问题的一个最简单的方案就是超时机制

- 系统停止了响应,cpu利用率低大概率是死锁了

  - 如果线上是Web应用，应用服务器比如说是Tomcat，并且开启了JMX，则可以通过JConsole等工具远程查看下线上死锁的具体情况
  - ①ps -ef | grep java查看pid②top -p查看java中的线程③使用jstack将其堆栈信息保存下来，查看是否是锁升级导致的阻塞问题





## StampedLock

- 读多写少的场景中，更快的技术方案

- 三种锁模式

  - 写锁、悲观读锁和乐观读

  - StampedLock 里的写锁和悲观读锁加锁成功之后，都会返回一个 stamp；然后解锁的时候，需要传入这个 stamp

  - StampedLock 提供的乐观读，是允许一个线程获取写锁的，也就是说不是所有的写操作都被阻塞。

    - 由于 tryOptimisticRead() 是无锁的，所以共享变量 x 和 y读入方法局部变量时，x 和 y 有可能被其他线程修改了。因此最后读完之后，还需要再次验证一下是否存在写操作，这个验证操作是通过调用 validate(stamp) 来实现的。

    - 如果执行乐观读操作的期间，存在写操作，会把乐观读升级为悲观读锁

    - ```java
      if (!sl.validate(stamp)){
      // 升级为悲观读锁
      stamp = sl.readLock();
      
      ```

  - version 字段就类似于 StampedLock 里面的 stamp

    - ```sql
      update product_doc set version=version+1，...where id=777 and version=9
      
      ```

- StampedLock 在命名上并没有增加 Reentrant，StampedLock 不支持重入。

- StampedLock 的悲观读锁、写锁都不支持条件变量，这个也需要你注意。

- 如果线程阻塞在 StampedLock 的 readLock() 或者writeLock() 上时，此时调用该阻塞线程的 interrupt() 方法，会导致 CPU 飙升。

- 使用 StampedLock 一定不要调用中断操作，如果需要支持中断功能，一定使用可中断的悲观读锁 readLockInterruptibly() 和写锁 writeLockInterruptibly()。这个规则一定要记清楚。

- 锁的申请和释放要成对出现，对此我们有一个最佳实践，就是使用try{}finally{}，但是 try{}finally{}并不能解决所有锁的释放问题。锁的升级会生成新的stamp ，而 finally 中释放锁用的是锁升级前的 stamp，本质上这也属于锁的申请和释放没有成对出现，只是它隐藏得有点深。解决这个问题倒也很简单，只需要对 stamp 重新赋值就可以了



## CountDownLatch和CyclicBarrier

- 用 CountDownLatch 实现线程等待

  - 最直接的办法是弄一个计数器
    - 首先创建了一个 CountDownLatch，计数器的初始值等于 2
    - 对计数器减 1 的操作是通过调用 latch.countDown(); 来实现的。
    - 调用 latch.await() 来实现对计数器等于 0 的等待。

- 用 CyclicBarrier 实现线程同步

  - 调用 barrier.await() 来将计数器减 1

  - CyclicBarrier 的计数器有自动重置的功能，当减到 0 的时候，会自动重置你设置的初始值

  - 首先创建了一个计数器初始值为 2 的CyclicBarrier，你需要注意的是创建 CyclicBarrier 的时候，我们还传入了一个回调函数，当计数器减到 0 的时候，会调用这个回调函数。回调函数总是在计数器归0时候执行,但是线程T1 T2要等回调函数执行结束之后才会再次执行。

  - - ```java
      // 执行回调的线程池
      Executor executor = Executors.newFixedThreadPool(1);
      final CyclicBarrier barrier = new CyclicBarrier(2, ()->{
      	executor.execute(()->check());
      });
       
      
      ```

    - CyclicBarrier的回调函数执行在一个回合里最后执行await()的线程上，而且同步调用回调函数check()，调用完check()之后，才会开始第二回合。所以check如果不另开一线程异步执行，就起不到性能优化的作用了。

    - CyclicBarrier 是同步调用回调函数之后才唤醒等待的线程，如果我们在回调函数里直接调用 check() 方法，那就意味着在执行 check()的时候，是不能同时执行 xxx() 和 yyy() 的，这样就起不到提升性能的作用。

    - 当看到回调函数的时候，一定问一问执行回调函数的线程是谁。

  - 如果生产者速度比消费者快的情况下，放入一个双向的阻塞队列尾部，每次从双向队列头部取两个对象，根据对象属性来区别订单类型，也能开多个线程进行check操作。





## 并发容器

- 组合操作需要注意竞态条件问题。组合操作往往隐藏着竞态条件问题，即便每个操作都能保证原子性，也并不能保证组合操作的原子性，这个一定要注意。
- 迭代器遍历容器
  - 通过迭代器遍历容器 list，对每个元素调用方法，这就存在并发问题
  - 迭代器遍历不安全是因为hasNext(size)和next()存在的竞态条件
  - 正确做法是锁住 list 之后再执行遍历操作。
- List
  - 使用 CopyOnWriteArrayList 需要注意的“坑”主要有两个方面。一个是应用场景，CopyOnWriteArrayList 仅适用于写操作非常少的场景，而且能够容忍读写的短暂不一致。写入的新元素并不能立刻被遍历到
- Map
  - 跳表插入、删除、查询操作平均的时间复杂度是 O(log n)，理论上和并发线程数没有关系，所以在并发度非常高的情况下，若你对 ConcurrentHashMap 的性能还不满意，可以尝试一下 ConcurrentSkipListMap。
- Queue
  - 你可以从以下两个维度来分类
    - 一个维度是阻塞与非阻塞
    - 另一个维度是单端与双端
    - Java 并发包里阻塞队列都用 Blocking 关键字标识，单端队列使用 Queue 标识，双端队列使用 Deque 标识。
    - 实际工作中，一般都不建议使用无界的队列，因为数据量大了之后很容易导致OOM。上面我们提到的这些 Queue 中，只有 ArrayBlockingQueue 和LinkedBlockingQueue 是支持有界的，所以在使用其他无界队列时，一定要充分考虑是否存在导致 OOM 的隐患。





## 原子类

- CAS

  - 作为一条 CPU 指令，CAS 指令本身是能够保证原子性的。
  - 只有当内存中 count 的值等于期望值 A 时，才能将内存中count 的值更新
  - 采用自旋方案，可以重新读 count 最新的值来计算 newValue 并尝试再次更新，直到成功。
  - ABA 问题
  - Java 如何实现原子化的 count += 1
    - 在 Java 1.8 版本中，getAndIncrement() 方法会转调 unsafe.getAndAddXxx() 方法
    - 该方法首先会在内存中读取共享变量的值，之后循环调用 compareAndSwapXxx() 方法来尝试设置共享变量的值，直到成功为止。
    - compareAndSwapXxx() 是一个 native 方法，只有当内存中共享变量的值等于expected 时，才会将共享变量的值更新为 x，并且返回 true；否则返回 fasle。compareAndSwapXxx 的语义和 CAS 指令的语义的差别仅仅是返回值不同而已。
    - getAndAddXxx() 方法的实现，基本上就是 CAS 使用的经典范例

- 原子类概览

  - 原子化的基本数据类型

    - AtomicBoolean、AtomicInteger 和 AtomicLong

  - 原子化的对象引用类型

    - 对象引用的更新需要重点关注 ABA 问题，AtomicStampedReference 和AtomicMarkableReference 这两个原子类可以解决 ABA 问题。
    - AtomicStampedReference 实现的 CAS 方法就增加了版本号参数
    - AtomicMarkableReference 的实现机制则更简单，将版本号简化成了一个 Boolean 值

  - 原子化数组

    - AtomicIntegerArray、AtomicLongArray 和 AtomicReferenceArray
    - 我们可以原子化地更新数组里面的每一个元素。这些类提供的方法和原子化的 基本数据类型的区别仅仅是：每个方法多了一个数组的索引参数

  - 原子化对象属性更新器

    - AtomicIntegerFieldUpdater、AtomicLongFieldUpdater 和 AtomicReferenceFieldUpdater，利用它们可以原子化地更新对象的属性，这三个方法都 是利用反射机制实现的

    - 对象属性必须是 volatile 类型的，只有这样才能保证可见性；如果对象属性不是 volatile 类型的，newUpdater() 方法会抛出 IllegalArgumentException 这个运行时异常。

    - ```java
      // newUpdater() 的方法参数只有类的信息，没有对象的引用，而更新对象的属性，一定需要对象的引用，那这个参数是在哪里传入的呢？是在原子操作的方法参数中传入的。
      public static <U> AtomicXXXFieldUpdater<U>  newUpdater(Class<U> tclass, String fieldName)
          
      // compareAndSet() 这个原子操作，相比原子化的基本数据类型多了一个对象引用obj。    
      boolean compareAndSet(T obj,  int expect, int update)   
      ```

      

  - 原子化的累加

    - DoubleAccumulator、DoubleAdder、LongAccumulator 和 LongAdder，这四个类仅仅用来执行累加操作，相比原子化的基本数据类型，速度更快，但是不支持compareAndSet() 方法。如果你仅仅需要累加操作，使用原子化的累加器性能会更好。





## Future

- ThreadPoolExecutor 的 void execute(Runnable command) 方法，利用这个方法虽然可以提交任务，但是却没有办法获取任务的执行结果（execute() 方法没有返回值）

- 如果任务之间有依赖关系，比如当前任务依赖前一个任务的执行结果，这种问题基本上都可以用 Future 来解决

- future是阻塞的等待。发起任务后，做其他的工作。做完后，从future获取处理结果，继续进行后面的任务

- 如何获取任务执行结果

  - 3 个 submit() 方法

    - ```java
      class Task implements Runnable{
           // 通过构造函数传入 result
      }
      Future<Result> future = executor.submit(new Task(r), r); 
      Result fr = future.get();
      
      ```

    - 提交 Runnable 任务 submit(Runnable task)

    - 提交 Callable 任务 submit(Callable<T> task)：

    - 提交 Runnable 任务及结果引用 submit(Runnable task, T result)：

    - 返回值都是 Future 接口，Future 接口有 5 个方法

      - 取消任务的方法 cancel()、判断任务是否已取消的方法 isCancelled()、判断任务是否已结束的方法 isDone()以及2 个获得任务执行结果的 get() 和 get(timeout, unit)
      - 这两个 get() 方法都是阻塞式的，如果被调用的时候，任务还没有执行完，那么调用 get() 方法的线程会阻塞，直到任务执行完才会被唤醒。

- FutureTask 工具类

  - 这个工具类有两个构造函数，它们的参数和前面介绍的 submit() 方法类似

  - FutureTask 实现了 Runnable 和 Future 接口

  - ```java
    // 创建 FutureTask
    FutureTask<Integer> futureTask= new FutureTask<>(()-> 1+2);
    // 创建线程池
    ExecutorService es = Executors.newCachedThreadPool();
    // 提交 FutureTask 
    es.submit(futureTask);
    // 获取计算结果
    Integer result = futureTask.get();
    
    
    ```

  - 利用FutureTask 对象可以很容易获取子线程的执行结果。

  - ```java
    // 创建 FutureTask
    FutureTask<Integer> futureTask= new FutureTask<>(()-> 1+2);
    // 创建并启动线程
    Thread T1 = new Thread(futureTask);
    T1.start();
    // 获取计算结果
    Integer result = futureTask.get();
    
    
    ```



- 实现最优的“烧水泡茶”程序

  - ft1 这个任务在执行泡茶任务前，需要等待 ft2 把茶叶拿来，所以 ft1 内部需要引用 ft2，并在执行泡茶之前，调用 ft2 的 get() 方法实现等待

  - ```java
    // 创建任务 T1 的 FutureTask
    FutureTask<String> ft1= new FutureTask<>(new T1Task(ft2));
    // 线程 T1 执行任务 ft1
    Thread T1 = new Thread(ft1);
    
    class T1Task implements Callable<String>
    class T2Task implements Callable<String>
    
    
    ```

    

  

## CompletableFuture

- 创建 CompletableFuture 对象
  - runAsync(Runnable runnable)和supplyAsync(Supplier<U> supplier)，它们之间的区别是：Runnable 接口的run() 方法没有返回值，而 Supplier 接口的 get() 方法是有返回值的。
  - 可以指定线程池 
    - 默认情况下 CompletableFuture 会使用公共的 ForkJoinPool 线程池，这个线程池默认创建的线程数是 CPU 的核数
- CompletionStage接口
  - 串行关系
    - thenApply、thenAccept、thenRun和 thenCompose 这四个系列的接口。 
    - thenApply 系列方法参数 fn 的类型是接口 Function<T, R>，这个接口里与CompletionStage 相关的方法是 R apply(T t)，这个方法既能接收参数也支持返回值
    - thenAccept 系列方法参数 consumer 的类型是接口Consumer<T>，这个接口里与 CompletionStage 相关的方法是 void accept(T t)，这个方法虽然支持参数，但却不支持回值
    - thenRun 系列方法里 action 的参数是 Runnable，所以 action 既不能接收参数也不支持返回值
    - thenCompose 系列方法，这个系列的方法会新创建出一个子流程，最终结果和thenApply 系列是相同的
  - AND 聚合关系
    - thenCombine、 thenAcceptBoth 和 runAfterBoth 系列的接口，这些接口的区别也是源自 fn、consumer、action 这三个核心参数不同。
  - OR聚合关系
    - applyToEither、acceptEither 和runAfterEither 系列的接口，这些接口的区别也是源自 fn、consumer、action 这三个核心参数不同。
    - f1.applyToEither(f2,s -> s);
  - 异常处理
    - 非异步编程里面，我们可以使用 try{}catch{}来捕获并处理异常，那在异步编程里面，异常该如何处理呢？
    - 用 exceptionally() 方法来处理异常，非常类似于 try{}catch{}中的 catch{}，
    - whenComplete() 和 handle() 系列方法就类似于 try{}finally{}中的 finally{}，
      - whenComplete() 和 handle() 的区别在于whenComplete() 不支持返回结果，而 handle() 是支持返回结果的。
- 注意事项
  - 查数据库属于io操作，用定制线程池 
  - 查出来的结果做为下一步处理的条件，若结果为空呢，没有对应处理 
  - 缺少异常处理机制





## **CompletionService**

- CompletionService 的实现原理是内部维护了一个阻塞队列

  - CompletionService 是把任务执行结果的 Future对象加入到阻塞队列中

  - ```java
    // CompletionService 接口的实现类是 ExecutorCompletionService
    // 如果不指定 completionQueue，那么默认会使用无界的 LinkedBlockingQueue。任务执行结果的 Future 对象就是加入到completionQueue 中。
    ExecutorCompletionService(Executor executor,BlockingQueue<Future<V>> completionQueue)。
    
    //  submit() 方法将会被 CompletionService 异步执行
    //  通过 CompletionService 接口提供的 take() 方法获取一个 Future 对象
    //  调用 Future 对象的 get() 方法就能返回询价操作的执行结果了
    
    
    ```

- CompletionService 接口提供的方法有 5 个

  - submit() 相关的方法有两个
    - 一个方法参数是Callable<V> task
    - 另外一个方法有两个参数，分别是Runnable task和V result，这个方法类似于ThreadPoolExecutor 的 <T> Future<T> submit(Runnable task, T result）
  - CompletionService 接口其余的 3 个方法，都是和阻塞队列相关的，take()、poll() 都是从阻塞队列中获取并移除一个元素；
    - 它们的区别在于如果阻塞队列是空的，那么调用 take()方法的线程会被阻塞，而 poll() 方法会返回 null 值。
    - poll(long timeout, TimeUnitunit) 方法支持以超时的方式获取并移除阻塞队列头部的一个元素

- 假设要等三个线程都执行完才能执行主线程的的return m，但是代码无法保证三个线程都执行完，和主线程执行return的顺序，因此，m的值不是准确的，可以加个线程栅栏，线程执行完计数器，来达到这效果







------



## 高性能限流器Guava RateLimiter

- Guava 实现令牌桶算法，用了一个很简单的办法，其关键是记录并动态计算下一令牌发放的时间

  - 下一令牌产生时间之前请求令牌
  - 下一令牌产生时间之后请求令牌
  - 只需要记录一个下一令牌产生的时间，并动态更新它，就能够轻松完成限流功能

- 令牌桶的容量是 1

  - ```java
    class SimpleLimiter {
        // 下一令牌产生时间
        long next = System.nanoTime();
        // 发放令牌间隔：纳秒
        long interval = 1000_000_000;
    
        // 预占令牌，返回能够获取令牌的时间
        synchronized long reserve(long now) {
    	// 请求时间在下一令牌产生时间之后
    	// 重新计算下一令牌产生时间
            if (now > next) {
                // 将下一令牌产生时间重置为当前时间
                next = now;
            }
            // 能够获取令牌的时间
            long at = next;
            // 设置下一令牌产生时间
            next += interval;
            // 返回线程需要等待的时间
            return max(at, 0L);
        }
    
        // 申请令牌
        public static void main(String[] args) {
            // 申请令牌时的时间
            long now = System.nanoTime();
            SimpleLimiter simpleLimiter = new SimpleLimiter();
            // 预占令牌
            long at = simpleLimiter.reserve(now);
            long waitTime = max(at - now, 0);
            // 按照条件等待
            if (waitTime > 0) {
                try {
                    TimeUnit.NANOSECONDS
                            .sleep(waitTime);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
    
    
    ```

    

- 令牌桶的容量大于 1

  - 如果线程请求令牌的时间在下一令牌产生时间之后，会重新计算令牌桶中的令牌数

  - 如果令牌是从令牌桶中出的

  - ```java
    class SimpleLimiter {
        // 当前令牌桶中的令牌数量
        long storedPermits = 0;
        // 令牌桶的容量
        long maxPermits = 3;
        // 下一令牌产生时间
        long next = System.nanoTime();
        // 发放令牌间隔：纳秒
        long interval = 1000_000_000;
    
        // 请求时间在下一令牌产生时间之后, 则
        // 1. 重新计算令牌桶中的令牌数
        // 2. 将下一个令牌发放时间重置为当前时间
        void resync(long now) {
            if (now > next) {
                // 新产生的令牌数
                long newPermits = (now - next) / interval;
                // 新令牌增加到令牌桶
                storedPermits = min(maxPermits,
                        storedPermits + newPermits);
                // 将下一个令牌发放时间重置为当前时间
                next = now;
            }
        }
    
        // 预占令牌，返回能够获取令牌的时间
        synchronized long reserve(long now) {
            resync(now);
            // 能够获取令牌的时间
            long at = next;
            // 令牌桶中能提供的令牌
            long fb = min(1, storedPermits);
            // 令牌净需求：首先减掉令牌桶中的令牌
            long nr = 1 - fb;
            if (fb != 1) {
                // 重新计算下一令牌产生时间
                next = next + nr * interval;
                at = next;
                // 重新计算令牌桶中的令牌
                this.storedPermits -= fb;
            }
            return at;
        }
    
        // 申请令牌
        void acquire() {
            // 申请令牌时的时间
            long now = System.nanoTime();
            // 预占令牌
            long at = reserve(now);
            long waitTime = max(at - now, 0);
            // 按照条件等待
            if (waitTime > 0) {
                try {
                    TimeUnit.NANOSECONDS
                            .sleep(waitTime);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
    
    ```

  
  



# 高并发限流花式炫技

# 1 场景

> 在分布式领域当中，大型的项目都被拆分单独的众多微服务，而一个优秀的项目就是组织调用各个服务，来达到聚合用户资源，因为是分布式的，每个服务器单独部署各个微服务，这样我们就很容易水平扩展微服务的资源，比如增加服务器的数量，动态的发送请求，通过算法，发送可供选择的服务器应用，但是系统的处理能力有限，大量的请求对系统继续施压，导致服务崩溃，不可用，比如接口恶意攻击，大量的请求打到服务接口，导致该服务其他接口不能正常工作，首先想到的就是服务隔离。但是本章节主要讲的不是服务隔离，而是另一个**限流**，阻止计划之外的请求对系统继续施压。

# 2 限流

> 限流,除了控制流量，限流还有一个目的是控制用户行为，避免垃圾请求。比如在贴吧的用户发帖，回复，点赞等行为都要严格控制，严格限定某一行为在指定的时间内只能发生N次

# 2.1 限流方案

- **应用层限流**：NGINX，网关
- **限流算法**：令牌桶、漏桶，计数器也可以进行粗暴限流实现

## 2.2 应用层限流

- **NGINX**

> 大家都很熟悉，不仅能做轻量级的web服务器，本身也可以做负载均衡，限流

服务器每天都在接受上亿次的请求,如何控制请求的单位时间内的访问？

在NGINX中通过geo,map,limit_req来达到限流的目的，请看如下配置:

```
##IP白名单
geo $whiteiplist {
     default 1;
     127.0.0.1 0;
     10.0.0.0/8 0;
  
}
map $whiteiplist $limit {
     1 $limit_key;
     0 "";
}
## 接口白名单
map $uri $limit2{
     default $limit;
     /api/sample "";

}
limit_req_status 406;
### 频率控制
limit_req_zone $limit2 zone=freq_controll:100m rate=10r/s;
limit_req_zone $limit2 zone=freq_controll_2:100m rate=500r/m;
复制代码
```

在location中使用上面定义好的限流算法

```
location / {
	limit_req zone=freq_controll burst=5 nodelay;
	limit_req zone=freq_controll_2 burst=10 nodelay;
	error_page 406 =406  @f406;
	location @f406 {
		access_log syslog:server=127.0.0.1:12301;
		return 406;
		}
	}
复制代码
```

注意到limit_req中两个奇怪的参数 burst ,nodelay 开始的时候我也感到疑惑,了解了背后的逻辑和算法,才理解了其中的奥义.

limit_req 模块的算法属于**令牌桶算法**.可以应对某些突发的负载 而burst参数的作用是：：假设一秒内同时有120个请求发到服务器.按照传统的漏斗算法,多出的这20个请求会被直接拒绝 或者是放到队列中等待.而在令牌桶算法中又是另外一种景象了.令牌桶中实际上有100个令牌.但是允许并发10个请求.那么多出来10个请求会被拒绝.

1. 在没有配置nodelay的情况下,这10个请求会被放到队列.以0.001秒的速率被取出,共计消耗0.1秒.处理110个请求用了1.1秒.实际上这个等待是没有必要的.
2. 配置了nodelay,这多出的十个请求会被正常处理,只是burst的数量会被清空.等待令牌重新补充,才会重新接收请求.处理110个请求用了1秒.但上面的情况一样,都要等令牌补充才能接收请求.

- **网关限流**

> 这里所说的网关是客户端网关，具有代表性的就是zuul以及spring cloud gateway，统一入口处限流，防止各个服务器之间高频请求导致服务崩溃

在spring cloud gateway当中使用**RequestRateLimiter**过滤器可以用于限流，使用RateLimiter实现来确定是否允许当前请求继续进行，如果请求太大默认会返回HTTP 429-太多请求状态。

1. 在pom.xml中添加相关依赖：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
复制代码
```

1. 添加限流策略的配置类，这里有两种策略一种是根据请求参数中的username进行限流，另一种是根据访问IP进行限流；

```
@Configuration
public class RedisRateLimiterConfig {
    @Bean
    KeyResolver userKeyResolver() {
        return exchange -> Mono.just(exchange.getRequest().getQueryParams().getFirst("username"));
    }

    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(exchange.getRequest().getRemoteAddress().getHostName());
    }
}
复制代码
```

1. 我们使用Redis来进行限流，所以需要添加Redis和RequestRateLimiter的配置，这里对所有的GET请求都进行了按IP来限流的操作

```
server:
  port: 9201
spring:
  redis:
    host: localhost
    password: 123456
    port: 6379
  cloud:
    gateway:
      routes:
        - id: requestratelimiter_route
          uri: http://localhost:8201
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 1 #每秒允许处理的请求数量
                redis-rate-limiter.burstCapacity: 2 #每秒最大处理的请求数量
                key-resolver: "#{@ipKeyResolver}" #限流策略，对应策略的Bean
          predicates:
            - Method=GET
logging:
  level:
    org.springframework.cloud.gateway: debug
复制代码
```

## 2.2 限流算法

### 2.2.1 计数器

> 它是限流算法中最简单最容易的一种算法，比如我们要求某一个接口，**1分钟内的请求不能超过10次**，我们可以在开始时设置一个计数器，**每次请求，该计数器+1**；如果该计数器的值大于10并且与第一次请求的时间间隔在1分钟内，那么说明请求过多，如果该请求与第一次请求的时间间隔大于1分钟，并且该计数器的值还在限流范围内，那么重置该计数器



![img](https://user-gold-cdn.xitu.io/2020/6/14/172b2e65aed3b716?imageslim)



手写计数器代码示例：

```
/**
 * 功能说明: 手写计数器
 */
public class LimitService {
	private int limtCount = 60;// 限制最大访问的容量
	AtomicInteger atomicInteger = new AtomicInteger(0); // 每秒钟 实际请求的数量
	private long start = System.currentTimeMillis();// 获取当前系统时间
	private int interval = 60;// 间隔时间60秒
	public boolean acquire() {
		long newTime = System.currentTimeMillis();
		if (newTime > (start + interval)) {
			// 判断是否是一个周期
			start = newTime;
			atomicInteger.set(0); // 清理为0
			return true;
		}
		atomicInteger.incrementAndGet();// i++;
		return atomicInteger.get() <= limtCount;
	}
	static LimitService limitService = new LimitService();
	public static void main(String[] args) {
		ExecutorService newCachedThreadPool = Executors.newCachedThreadPool();
		for (int i = 1; i < 100; i++) {
			final int tempI = i;
			newCachedThreadPool.execute(new Runnable() {
				public void run() {
					if (limitService.acquire()) {
						System.out.println("你没有被限流,可以正常访问逻辑 i:" + tempI);
					} else {
						System.out.println("你已经被限流了呢  i:" + tempI);
					}
				}
			});
		}
	}
}
复制代码
```

**问题**： 计数器可能产生**临界**的问题，如果大量的流量，在临界的时候聚集，比如59秒访问10个请求，61秒的时候访问10个请求。这样2秒内出现了个20个请求，这样就违背了我们设定的60秒之内允有10个请求。

这个问题我们可以**采用滑动窗口方式**解决计数器临界值的问题

### 2.2.2 滑动窗口计数

> 滑动窗口原理:每次有访问进来时，先判断前 N 个单位时间内的总访问量是否超过了设置的阈值，并对当前时间片上的请求数 +1。

滑动窗口计数有很多使用场景，比如说**限流防止系统雪崩**。相比计数实现，滑动窗口实现会更加平滑，能自动消除毛刺。

滑动窗口示例：在一分钟之内分为6个格子，每个格子访问时间在10秒，每个格子中有自己的独立计数器。60秒之内只能允许1000个请求

![img](https://user-gold-cdn.xitu.io/2020/6/14/172b2e65ae45ab1a?imageslim)



思考：**为什么滑动窗口能够解决临界高并发问题？** 回答：因为滑动窗口是计算单位时间内走过的窗口内的请求总数，圈起来的狂口就是单位时间，这样就能控制单位时间内的请求总数

### 2.2.3 令牌桶算法

> 令牌桶算法是一个存放固定容量令牌的桶，按照固定速率往桶里添加令牌。

**令牌桶算法原理**：假设限制2r/s，则按照500毫秒的固定速率往桶中添加令牌；桶中最多存放b个令牌，**当桶满时，新添加的令牌被丢弃或拒绝**；当一个n个字节大小的数据包到达，将从桶中删除n个令牌，接着数据包被发送到网络上；如果桶中的令牌不足n个，则不会删除令牌，且该数据包将被限流（要么丢弃，要么缓冲区等待）。





![img](https://user-gold-cdn.xitu.io/2020/6/14/172b2e65af182b02?imageslim)

**示例**

- 使用RateLimiter实现令牌桶限流

> RateLimiter是guava提供的基于令牌桶算法的实现类，可以非常简单的完成限流特技，并且根据系统的实际情况来调整生成token的速率。 通常可应用于抢购限流防止冲垮系统；限制某接口、服务单位时间内的访问量，譬如一些第三方服务会对用户访问量进行限制；限制网速，单位时间内只允许上传下载多少字节等。

1. 引入guava的maven依赖。

```
<dependency>
  <groupId>com.google.guava</groupId>
  <artifactId>guava</artifactId>
  <version>25.1-jre</version>
</dependency>
复制代码
```

1. 使用RateLimiter 实现令牌桶算法

```
@RestController
public class IndexController {
	@Autowired
	private OrderService orderService;
	// 解释：1.0 表示 每秒中生成1个令牌存放在桶中
	RateLimiter rateLimiter = RateLimiter.create(1.0);
	// 下单请求
	@RequestMapping("/order")
	public String order() {
		// 1.限流判断
		// 如果在500秒内 没有获取不到令牌的话，则会一直等待
		System.out.println("生成令牌等待时间:" + rateLimiter.acquire());
		boolean acquire = rateLimiter.tryAcquire(500, TimeUnit.MILLISECONDS);
		if (!acquire) {
			System.out.println("你在怎么抢，也抢不到，因为会一直等待的，你先放弃吧！");
			return "你在怎么抢，也抢不到，因为会一直等待的，你先放弃吧！";
		}
		// 2.如果没有达到限流的要求,直接调用订单接口
		boolean isOrderAdd = orderService.addOrder();
		if (isOrderAdd) {
			return "恭喜您,抢购成功!";
		}
		return "抢购失败!";
	}
}
复制代码
```

这是简单的单机限流，这个也可以改造成通过注解加上AOP的方式平滑的注入接口当中 3. 封装RateLimiter

- 自定义注解

```
@Target(value = ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ExtRateLimiter {
	double value();
	long timeOut();
}
复制代码
```

- 编写AOP

```
@Aspect
@Component
@Slf4j
public class RateLimiterAspect {

    /**
     * 存放接口是否已经存在
     */
    private static ConcurrentHashMap<String, RateLimiter> rateLimiterMap = new ConcurrentHashMap<>();

    @Pointcut("@annotation(extRateLimiter)")
    public void pointcut(ExtRateLimiter extRateLimiter) {

    }

    @Around(value = "pointcut(extRateLimiter)", argNames = "proceedingJoinPoint,extRateLimiter")
    public Object doBefore(ProceedingJoinPoint proceedingJoinPoint, ExtRateLimiter extRateLimiter) throws Throwable {

        // 获取配置的速率
        double permitsPerSecond = extRateLimiter.permitsPerSecond();
        // 获取等待令牌等待时间
        long timeOut = extRateLimiter.timeOut();
        RateLimiter rateLimiter = getRateLimiter(permitsPerSecond);
        boolean acquire = rateLimiter.tryAcquire(timeOut, TimeUnit.MILLISECONDS);
        if (acquire) {
            return proceedingJoinPoint.proceed();
        }
        HttpServletUtil.write2ExceptionResult("执行降级方法,亲,服务器忙！请稍后重试!");
        return null;
    }

    private RateLimiter getRateLimiter(double permitsPerSecond) {
        // 获取当前URL
        HttpServletRequest servletRequest = HttpServletUtil.getRequest();
        String requestURI = servletRequest.getRequestURI();
        RateLimiter rateLimiter = rateLimiterMap.get(requestURI);
        if (ObjectUtils.isEmpty(rateLimiter)) {
            rateLimiter = RateLimiter.create(permitsPerSecond);
            rateLimiterMap.put(requestURI, rateLimiter);
        }
        return rateLimiter;
    }
}
复制代码
```

- 使用示例

```
@RequestMapping("/myOrder")
@ExtRateLimiter(value = 10.0, timeOut = 500)
public String myOrder() throws InterruptedException {
		System.out.println("myOrder");
		return "SUCCESS";
}
复制代码
```

### 2.2.4 漏桶算法

> 漏桶作为计量工具（The Leaky Bucket Algorithm as a Meter）时，可以用于流量整形（Traffic Shaping）和流量控制（TrafficPolicing）

- 漏桶算法原理: 一个固定容量的漏桶，按照常量固定速率流出水滴；如果桶是空的，则不需流出水滴；可以以任意速率流入水滴到漏桶；如果流入水滴超出了桶的容量，则流入的水滴溢出了（被丢弃），而漏桶容量是不变的。



![img](https://user-gold-cdn.xitu.io/2020/6/14/172b2e65afb5d3a5?imageslim)



- 令牌桶和漏桶对比：

1. 方式区别： **令牌桶**是按照固定速率往桶中添加令牌，请求是否被处理需要看桶中令牌是否足够，当令牌数减为零时则拒绝新的请求； **漏桶**则是按照常量固定速率流出请求，流入请求速率任意，当流入的请求数累积到漏桶容量时，则新流入的请求被拒绝；
2. 优缺点： **令牌桶**限制的是平均流入速率（允许突发请求，只要有令牌就可以处理，支持一次拿3个令牌，4个令牌），并允许一定程度突发流量； **漏桶**限制的是常量流出速率（即流出速率是一个固定常量值，比如都是1的速率流出，而不能一次是1，下次又是2），从而平滑突发流入速率；

> 注意：令牌桶允许一定程度的突发，而漏桶主要目的是平滑流入速率；

- 应用场景： “漏桶算法”能够**强行限制数据的传输速率**，而“令牌桶算法”在能够限制数据的平均传输速率外，还允许某种程度的突发传输。在“令牌桶算法”中，只要令牌桶中存在令牌，那么就允许突发地传输数据直到达到用户配置的门限，因此它适合于具有**突发特性**的流量
- 实现

```
@Slf4j
public class LeakyBucketLimiter {
    private ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(5);
 
    // 桶的容量
    public int capacity = 10;
    // 当前水量
    public int water = 0;
    //水流速度/s
    public int rate = 4;
    // 最后一次加水时间
    public long lastTime = System.currentTimeMillis();
 
    public void acquire() {
        scheduledExecutorService.scheduleWithFixedDelay(() -> {
            long now = System.currentTimeMillis();
            //计算当前水量
            water = Math.max(0, (int) (water - (now - lastTime) * rate /1000));
            int permits = (int) (Math.random() * 8) + 1;
            log.info("请求数：" + permits + "，当前桶余量：" + (capacity - water));
            lastTime = now;
            if (capacity - water < permits) {
                // 若桶满,则拒绝
                log.info("限流了");
            } else {
                // 还有容量
                water += permits;
                log.info("剩余容量=" + (capacity - water));
            }
        }, 0, 500, TimeUnit.MILLISECONDS);
    }
 
    public static void main(String[] args) {
        LeakyBucketLimiter limiter = new LeakyBucketLimiter();
        limiter.acquire();
    }
}
复制代码
```

- 运行







# 3 总结

> 在分布式场景，高并发大流量的实际业务中，非常有必要对接口进行限流，具体得看业务场景，选择合适的限流算法，阿里巴巴也出了限流组件，后期读者可以体验一下[Sentinel](https://github.com/alibaba/Sentinel/wiki/介绍)



![img](https://user-gold-cdn.xitu.io/2020/6/14/172b2e65aefe299d?imageslim)

