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

- jvm基本原理2——Java虚拟机是怎么实现 synchronized的





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

- 一个进程从代码到二进制到运行时的一个过程
  - 我们首先通过图右边的文件编译过程，生成 so 文件和可执行文件，放在硬盘上。下图左边的用户态的进程 A 执行 fork，创建进程 B，在进程 B 的处理逻辑中，执行 exec 系列系统调用。这个系统调用会通过 load_elf_binary 方法，将刚才生成的可执行文件，加载到进程 B 的内存中执行。
  - ![img](https://img2020.cnblogs.com/blog/1279115/202008/1279115-20200806105922849-668533800.jpg)
- 进行编译：程序的二进制格式
  - 程序写完了，这个文件只是文本文件，CPU 是不能执行文本文件里面的指令的，这些指令只有人能看懂，**CPU 能够执行的命令是二进制的**，比如“0101”这种，所以这些指令还需要翻译一下，这个翻译的过程就是**编译**（Compile）。
  - 在 Linux 下面，二进制的程序也要有严格的格式，这个格式我们称为**ELF**（Executeable and Linkable Format，可执行与可链接格式）。这个格式可以根据编译的结果不同，分为不同的格式。
    - 我们刚才说了可重定位，为啥叫**可重定位**呢？我们可以想象一下，这个编译好的代码和变量，将来加载到内存里面的时候，都是要加载到一定位置的。比如说，调用一个函数，其实就是跳到这个函数所在的代码位置执行；再比如修改一个全局变量，也是要到变量的位置那里去修改。但是现在这个时候，还是.o 文件，不是一个可以直接运行的程序，这里面只是部分代码片段。
    - 形成的二进制文件叫**可执行文件**，是 ELF 的第二种格式
    - 动态链接库，就是 ELF 的第三种类型，**共享对象文件**（Shared Object）。
      - **动态链接库**（Shared Libraries），不仅仅是一组对象文件的简单归档，而是多个对象文件的重新组合，可被多个程序共享。

- 运行程序为进程

  - 知道了 ELF 这个格式，这个时候它还是个程序，那**怎么把这个文件加载到内存里面呢**？

  - 学过了系统调用一节，你会发现，原理是 exec 这个系统调用最终调用的 load_elf_binary。

    exec 比较特殊，它是一组函数：

    - 包含 p 的函数（execvp, execlp）会在 PATH 路径下面寻找程序；
    - 不包含 p 的函数需要输入程序的全路径；
    - 包含 v 的函数（execv, execvp, execve）以数组的形式接收参数；
    - 包含 l 的函数（execl, execlp, execle）以列表的形式接收参数；
    - 包含 e 的函数（execve, execle）以数组的形式接收环境变量。

  - 创建 ls 进程，也是通过 exec。

- 进程树

  - 既然所有的进程都是从父进程 fork 过来的，那总归有一个祖宗进程，这就是咱们系统启动的 init 进程
  - 在解析 Linux 的启动过程的时候，1 号进程是 /sbin/init。如果在 centOS 7 里面，我们 ls 一下，可以看到，这个进程是被软链接到 systemd 的
  - 系统启动之后，init 进程会启动很多的 daemon 进程，为系统运行提供服务，然后就是启动 getty，让用户登录，登录后运行 shell，用户启动的进程都是通过 shell 运行的，从而形成了一棵进程树。
  - 我们可以通过 ps -ef 命令查看当前系统启动的进程，我们会发现有三类进程。
    - PID 1 的进程就是我们的 init 进程 systemd，PID 2 的进程是内核线程 kthreadd，这两个我们在内核启动的时候都见过。其中用户态的不带中括号，内核态的带中括号。
    - 接下来进程号依次增大，但是你会看所有带中括号的内核态的进程，祖先都是 2 号进程。而用户态的进程，祖先都是 1 号进程。tty 那一列，是问号的，说明不是前台启动的，一般都是后台的服务。
    - pts 的父进程是 sshd，bash 的父进程是 pts，ps -ef 这个命令的父进程是 bash。这样整个链条都比较清晰了。

- 对于任何一个进程来讲，即便我们没有主动去创建线程，进程也是默认有一个主线程的。 线程是负责执行二进制指令的，一行一行执行下去。进程要比线程管 的宽多了，除了执行指令之外，内存、文件系统等等都要它来管。 

- 一个普通线程的创建和运行过程
  - ![img](https://img2020.cnblogs.com/blog/1279115/202008/1279115-20200806110019812-342005156.jpg)
  - 线程的数据
    - 我们把线程访问的数据细分成三类。
    - 第一类是线程栈上的本地数据，比如函数执行过程中的局部变量。函数的调用会 使用栈的模型，这在线程里面是一样的。只不过每个线程都有自己的栈空间。
      - 栈的大小可以通过命令	ulimit	-a	查看，默认情况下线程栈大小为	8192（8MB）。我们可以使用 命令	ulimit	-s	修改。
      - 主线程在内存中有一个栈空间，其他线程栈也拥有独立的栈空间。为了避免线程之间的栈空间踩 踏，线程栈之间还会有小块区域，用来隔离保护各自的栈空间。一旦另一个线程踏入到这个隔离 区，就会引发段错误。 
    - 第二类数据就是**在整个进程里共享的全局数据**。例如全局变量，虽然在不同进程中是隔离的，但是在一个进程中是共享的。如果同一个全局变量，两个线程一起修改，那肯定会有问题，有可能把数据改的面目全非。这就需要有一种机制来保护他们
    - 第三类数据，**线程私有数据**（Thread Specific Data）
  - 数据的保护
    - **Mutex**，全称 Mutual Exclusion，中文叫**互斥**。
    - **条件变量和互斥锁是配合使用的**。

## 同步与异步

- 阻塞和非阻塞只是一种性质，而同步和异步是从宏观上对一种任务性质的描述
- 阻塞和非阻塞可以理解为，A请求B之后，在B端资源没准备好的时候，B是让A一直等待还是发出一个标志





## 死锁

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

  





## 进程间通信

- PIPE和FIFO用来实现进程间相互发送非常短小的、频率很高的消息；这两种方式通常适用于两个进程间的通信。
- 共享内存用来实现进程间共享的、非常庞大的、读写操作频率很高的数据（配合信号量使用）；这种方式通常适用于多进程间通信。（信号量是用于同步线程间的对象的使用的）
- 其他考虑用socket。这里的“其他情况”，其实是今天主要会碰到的情况：分布式开发。在多进程、多线程、多模块所构成的今天最常见的分布式系统开发中，socket是第一选择。
- 在面向对象的今天，我们更多的时候是多线程+锁+线程间共享数据。因此共享内存在今天使用的也越来越少了。

其中，最初 Unix IPC 包括：管道、FIFO、信号；

System V IPC 包括：System V 消息队列、System V 信号灯、System V 共享内存区；

Posix IPC 包括： Posix 消息队列、Posix 信号灯、Posix 共享内存区。

简单说明一下，现有大部分 Unix 和流行版本都是遵循 POSIX 标准的，而 Linux 从一开始就遵循 POSIX 标准。

图一给出了 Linux 所支持的各种 IPC 手段，在本文接下来的讨论中，

为了避免概念上的混淆，在尽可能少提及 Unix 的各个版本的情况下，所有问题的讨论最终都会归结到 Linux 环境下的进程间通信上来。

**进程间通信**

**进程间的七大通信方式**

**signal、file、pipe、shm、sem、msg、socket。**

下面分别介绍。

### **1、信号（Signal）**

**信号本质**

信号是在软件层次上对中断机制的一种模拟，在原理上，一个进程收到一个信号与处理器收到一个中断请求可以说是一样的。信号是异步的，一个进程不必通过任何操作来等待信号的到达，事实上，进程也不知道信号到底什么时候到达。

信号是**进程间通信机制中唯一的异步通信机制**，可以看作是异步通知，通知接收信号的进程有哪些事情发生了。

信号机制经过 POSIX 实时扩展后，功能更加强大，除了基本通知功能外，还可以传递附加信息。

**信号来源**

信号事件的发生有两个来源：**硬件来源(比如我们按下了键盘或者其它硬件故障)；\****软件来源**，

最常用发送信号的系统函数是 kill, raise, alarm 和 setitimer 以及 sigqueue 函数，软件来源还包括一些非法运算等操作。

**进程对信号的响应**

进程可以通过三种方式来响应一个信号：

（1）忽略信号，即对信号不做任何处理，其中，有两个信号不能忽略：SIGKILL 及 SIGSTOP；

（2）捕捉信号。定义信号处理函数，当信号发生时，执行相应的处理函数；

（3）执行缺省操作，Linux 对每种信号都规定了默认操作。

**信号的发送**

发送信号的主要函数有：

kill()、raise()、 sigqueue()、alarm()、setitimer() 以及abort()。

1、 kill 函数

对指定的进程发送什么信息。

pid>0 进程 ID 为 pid 的进程；

pid=0 同一个进程组的进程；

pid<0 pid!=-1进程组 ID 为 -pid 的所有进程；

pid=-1 除发送进程自身外，所有进程 ID 大于1的进程。

```
#include <sys/types.h> 
```

2、raise 函数

向进程本身发送信号，参数为即将发送的信号值。

调用成功返回 0；否则，返回 -1。

```
#include <signal.h> 
```

3、sigqueue 函数

调用成功返回 0；否则，返回 -1。

第一个参数是指定接收信号的进程 ID，第二个参数确定即将发送的信号，第三个参数是一个联合数据结构 union sigval，指定了信号传递的参数，sigqueue() 比 kill() 传递了更多的附加信息，但 sigqueue() 只能向一个进程发送信号，而不能发送信号给一个进程组。

```
#include <sys/types.h> 
```

4、 alarm 函数

专门为 SIGALRM 信号而设，在指定的时间 seconds 秒后，将向进程本身发送 SIGALRM 信号，又称为闹钟时间。

进程调用 alarm 后，任何以前的 alarm() 调用都将无效。

如果参数 seconds 为零，那么进程内将不再包含任何闹钟时间。 返回值，如果调用 alarm 前，进程中已经设置了闹钟时间，则返回上一个闹钟时间的剩余时间，否则返回 0。

```
#include <unistd.h> 
```

5、setitimer 函数

比 alarm功能强大，支持3种类型的定时器：

```
ITIMER_REAL：  设定绝对时间；经过指定的时间后，内核将发送SIGALRM信号给本进程；

#include <sys/time.h> 
```

6、abort 函数

```
#include <stdlib.h> 
```

向进程发送 SIGABORT 信号，默认情况下进程会异常退出，当然可定义自己的信号处理函数。

即使 SIGABORT 被进程设置为阻塞信号，调用 abort() 后，SIGABORT 仍然能被进程接收。该函数无返回值。

**信号处理**

如果进程要处理某一信号，那么就要在进程中安装该信号。

**安装信号主要用来确定信号值及进程针对该信号值的动作之间的映射关系，即进程将要处理哪个信号；该信号被传递给进程时，将执行何种操作。**

1、signal()

```
#include <signal.h> 
```

如果该函数原型不容易理解的话，可以参考下面的分解方式来理解：

```
#include <signal.h> 
```

第一个参数指定信号的值，第二个参数指定针对前面信号值的处理，可以忽略该信号（参数设为 SIG_IGN）；

可以采用系统默认方式处理信号(参数设为 SIG_DFL)；

也可以自己实现处理方式(参数指定一个函数地址)。

如果 signal() 调用成功，返回最后一次为安装信号 signum 而调用signal() 时的 handler值；失败则返回 SIG_ERR。

2、sigaction()

```
#include <signal.h>
```

sigaction函数 用于改变进程接收到特定信号后的行为。

该函数的第一个参数为信号的值，可以为除SIGKILL及SIGSTOP外的任何一个特定有效的信号（为这两个信号定义自己的处理函数，将导致信号安装错误）。

第二个参数是指向结构 sigaction 的一个实例的指针，在结构sigaction 的实例中，指定了对特定信号的处理，可以为空，进程会以缺省方式对信号处理；

第三个参数 oldact 指向的对象用来保存原来对相应信号的处理，可指定 oldact 为 NULL。如果把第二、第三个参数都设为NULL，那么该函数可用于检查信号的有效性。

**第二个参数最为重要，其中包含了对指定信号的处理、信号所传递的信息、信号处理函数执行过程中应屏蔽掉哪些函数等等。**

**信号通信方式的局限性**

不能够传递复杂的、有效的、具体的数据。

### **2、文件（file）**

使用文件进行进程间通信应该是最先学会的一种 IPC 方式。任何编程语言中，文件 IO 都是很重要的知识。

在 Linux 中，每打开一个文件，就会产生一个文件控制块，而文件控制块与文件描述符是一一对应的，因此可以通过对文件描述符的操作进而对文件进行操作。

文件描述符的分配原则：**编号的连续性(节省编号的资源)。**

文件系统对文件描述符的读/写控制，进程间一方对文件写，一方对文件读，达到文件之间的通信，可以是不相关进程间的通信。

使用的 API：read() 和 write()

为了能够实现两个进程通过文件进行有序的数据交流，**还得借助于信号的处理机制。**

- 通过 pause() 等待对方发起一个信号，已确认可以开始执行下一次读/写操作；pause()：只要接受到任何的信号，立马就可以往下执行。
- 通过kill() 方法向对方发出明确的信号：可以开始下一步执行(读、写)。

**缺点：**

- 文件通信没有访问规则。
- 访问速度慢。

### **3、管道（Pipe）及有名管道（named pipe）**

**相关概念**

管道是 Linux 支持的最初 Unix IPC 形式之一，具有以下特点：

（1）管道是半双工的，数据只能向一个方向流动；需要双方通信时，需要建立起两个管道；

（2）只能用于父子进程或者兄弟进程之间（具有亲缘关系的进程）；

（3）单独构成一种独立的文件系统：管道对于管道两端的进程而言，就是一个文件，但它不是普通的文件，它不属于某种文件系统，而是自立门户，单独构成一种文件系统，并且只存在与内存中。

（4）数据的读出和写入：一个进程向管道中写的内容被管道另一端的进程读出。写入的内容每次都添加在管道缓冲区的末尾，并且每次都是从缓冲区的头部读出数据。

**创建无名管道 API**

```
#include <unistd.h>
```

管道两端可分别用描述字 fd[0] 以及 fd[1] 来描述，需要注意的是，管道的两端是固定了任务的。

即一端只能用于读，由描述字 fd[0] 表示，称其为管道读端；

另一端则只能用于写，由描述字 fd[1] 来表示，称其为管道写端。

如果试图从管道写端读取数据，或者向管道读端写入数据都将导致错误发生。一般文件的 I/O 函数都可以用于管道，如close、read、write 等等。

**创建有名管道 API**

```
#include <sys/types.h>
```

该函数的第一个参数是一个普通的路径名，也就是创建后 FIFO 的名字。

第二个参数与打开普通文件的 open() 函数中的mode 参数相同。

如果 mkfifo 的第一个参数是一个已经存在的路径名时，会返回EEXIST 错误，所以一般典型的调用代码首先会检查是否返回该错误，如果确实返回该错误，那么只要调用打开 FIFO 的函数就可以了。一般文件的 I/O 函数都可以用于 FIFO，如 close、read、write 等等。

**mkfifo 会在文件系统中创建一个管道文件，然后使其映射内存的一个特殊区域，凡是能够打开 mkfifo 创建的管道文件进程(通过这个文件描述符)，都可以使用该文件实现 FIFO 的数据流动。**

### **4、共享内存（shm）**

**概念**

使得多个进程可以访问同一块内存空间，**是最快的可用 IPC 形式。**

各个进程都能够共同访问的共享的内存区域；是独立于所有的进程空间之外的地址区域；

进程对于共享内存的操作与管理主要是：

（1）申请创建一个共享内存区域(操作系统内核是不可能主动为进程创建共享内存)，操作系统内核得到申请然后创建。

（2）申请使用一个已存在的共享内存区域。

（3）申请释放共享内存区域(操作系统内核也是不可能主动释放共享内存区域)，操作系统内核得到申请然后释放。

共享内存允许两个或多个进程共享一给定的存储区，因为数据不需要来回复制，所以是最快的一种进程间通信机制。

**共享内存可以通过mmap()映射普通文件（特殊情况下还可以采用匿名映射）机制实现，也可以通过系统V共享内存机制实现。**

应用接口和原理很简单，内部机制复杂。为了实现更安全通信，往往还与信号灯等同步机制共同使用。

系统调用 mmap() 通过映射一个普通文件实现共享内存。

系统 V 则是通过映射特殊文件系统 shm 中的文件实现进程间的共享内存通信。

也就是说，每个共享内存区域对应特殊文件系统 shm 中的一个文件（这是通过 shmid_kernel 结构联系起来的）。

对于系统 V 共享内存，主要有以下几个 API：shmget()、shmat()、shmdt() 及 shmctl()。

```
int shmget(key_t key, size_t size, int shmflg); //返回值是共享内存的标号shmid
```

**说明 **

key_t 是一个 long 类型，是 IPC 资源外部约定的 key (关键)值，通过 key 值映射对应的唯一存在的某一个 IPC 资源

通过 key_t 的值就能够判断某一个对应的共享内存区域在哪，是否已经创建等等。

一个 key 值只能映射一个共享内存区域，但同时还可以映射一个信号量，一个消息队列资源，于是就可以使用一个 key 值管理三种不同的资源。

**共享内存的控制**

共享内存的控制信息可以通过 shmctl() 方法获取，会保存在struct_shmid_ds 结构体中。

共享内存的控制主要是 shmid_ds，即就是共享内存的控制信息

```
int shmctl(int shmid, int cmd, struct shmid_ds *buf)
```

cmd：看执行什么操作(1、获取共享内存信息；2、设置共享内存信息；3、删除共享内存)。

**总结**

**共享内存涉及到了存储管理以及文件系统等方面的知识，深入理解其内部机制有一定的难度，关键还要紧紧抓住内核使用的重要数据结构。**

**系统 V 共享内存是以文件的形式组织在特殊文件系统 shm 中的。\**** 通过 shmget 可以创建或获得共享内存的标识符。***\*取得共享内存标识符后，要通过 shmat 将这个内存区映射到本进程的虚拟地址空间。**

### **5、信号量（semaphore）**

**概念**

主要作为进程间以及同一进程不同线程之间的同步手段。

**解决：**进程在访问共享资源的时候存在冲突的问题，必须有一种强制手段说明这些共享资源的访问规则。

sem：表示的是一种共享资源的个数，对共享资源的访问规则。

**访问规则**

（1）用一种数量单位去标识某一种共享资源的个数。

（2）当有进程需要访问对应的共享资源的时候，则需要先查看申请，根据当前资源对应的可用数量进行申请。

（3）资源的管理者（也就是操作系统内核）就使用当前的资源个数减去要申请的资源的个数。如果结果 >=0 表示有可用资源，允许该进程的继续访问；否则表示资源不可用，通知进程（暂停或立即返回）。

（4）资源数量的变化就表示资源的占用和释放。占用：使得可用资源减少；释放：使得可用资源增加。

**相关 API**

```
 //创建信号量集
```

初始化信号量：

**信号量 ID 事实上是信号量集合的 ID，一个 ID 对应的是一组信号量，此时就使用信号量 ID 设置整个信号量集合。**

这个时候操作分两种：

（1）**针对信号量集合中的一个信号量进行设置；信号量集合中的信号量是按照数组的方式被管理起来的，从而可以直接使用信号的数组下标来进行访问。**

（2）针对整个信号量集和进行统一的设置。

```
 int semctl(int semid, int semnum, int cmd, ...)
```

初始化信号量：

如果 cmd 是 GETALL、SETALL、GETVAL、SETVAL...的话，则需要提供第四个参数。第四个参数是一个共用体，这个共用体在程序中必须的自己定义（作用：初始化资源个数），定义格式如下：

```
 union semun{
```

信号量的操作 API

```
 //semop()方法。(op：operator操作)
```

第二个参数需要借助结构体 struct sembuf：

```
 struct sembuf{
```

通过下标直接对其信号量 sem_op 进行加减即可。

**信号量特征**

如果有进程通过信号量申请共享资源，而且此时资源个数已经小于0，则此时对于该进程有两种可能性：等待资源，不等待。

如果此时进程选择等待资源，则操作系统内核会针对该信号量构建进程等待队列，将等待的进程加入到该队列之中。

如果此时有进程释放资源则会：

(1)、先将资源个数增加；

(2)、从等待队列中抽取第一个进程；

(3)、根据此时资源个数和第一个进程需要申请的资源个数进行比较，结果大于 0，则唤醒该进程；结果小于 0，则让该进程继续等待。

所以一般结合信号量的操作和共享内存使用来达到进程间的通信。

### **6、消息队列（Message）**

**概念**

消息队列就是一个消息的链表。可以把消息看作一个记录，具有特定的格式以及特定的优先级。

对消息队列有写权限的进程可以向中按照一定的规则添加新消息；对消息队列有读权限的进程则可以从消息队列中读走消息。

消息队列是随内核持续的；克服了信号承载信息量少，管道只能承载无格式字节流以及缓冲区大小受限等缺点。

系统 V 消息队列是随内核持续的，只有在内核重起或者显示删除一个消息队列时，该消息队列才会真正被删除。因此系统中记录消息队列的数据结构（struct ipc_ids msg_ids）位于内核中，系统中的所有消息队列都可以在结构msg_ids中找到访问入口。

**结构**

消息队列就是一个消息的链表。每个消息队列都有一个队列头，用结构struct msg_queue来描述。队列头中包含了该消息队列的大量信息，包括消息队列键值、用户ID、组ID、消息队列中消息数目等等，甚至记录了最近对消息队列读写进程的ID。读者可以访问这些信息，也可以设置其中的某些信息。

下图说明了内核与消息队列是怎样建立起联系的：

![img](https://mmbiz.qpic.cn/mmbiz_png/US10Gcd0tQHhDWgI17QEtzm1G4nFuxDFRpfqXAZ5QrlfVYs5ib31welzWLeic6iatzniarKUr1LyGngLmtYcRX4m1w/640?tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

其中：struct ipc_ids msg_ids是内核中记录消息队列的全局数据结构；struct msg_queue是每个消息队列的队列头。

从上图可以看出，全局数据结构 struct ipc_ids msg_ids 可以访问到每个消息队列头的第一个成员：struct kern_ipc_perm；

而每个 struct kern_ipc_perm 能够与具体的消息队列对应起来是因为在该结构中，有一个 key_t 类型成员 key，而 key 则唯一确定一个消息队列。kern_ipc_perm 结构如下：

```
struct kern_ipc_perm{   //内核中记录消息队列的全局数据结构msg_ids能够访问到该结构；
```

**API **

消息队列与管道不同的地方在于：管道中的数据并没有分割为一个一个的数据独立单位，在字节流上是连续的。然而，消息队列却将数据分成了一个一个独立的数据单位，每一个数据单位被称为消息体。每一个消息体都是固定大小的存储块儿，在字节流上是不连续的。

**创建消息队列**

```
int msgget(key_t key, int msgflg); 
```

在发送消息的时候动态的创建消息队列；

```
int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);
```

（1）msgsnd() 方法在发送消息的时候，是在消息体结构体中指定，当前的消息发送到消息队列集合中的哪一个消息队列上。

（2）消息体结构体中就必须包含一个 type 值，type 值是long类型，而且还必须是结构体的第一个成员。而结构体中的其他成员都被认为是要发送的消息体数据。

（3）无论是 msgsnd() 发送还是 msgrcv()接收时，只要操作系统内核发现新提供的 type 值对应的消息队列集合中的消息队列不存在，则立即为其创建该消息队列。

**总结**

为了能够顺利的发送与接收，发送方与接收方需要约定规则

（1）同样的消息体结构体；

（2）发送方与接收方在发送和接收的数据块儿大小上要与消息结构体的具体数据部分保持一致， 否则将不会读出正确的数据。

重点注意：

**消息结构体被发送的时候，只是发送了消息结构体中成员的值，如果结构体成员是指针，并不会将指针所指向的空间的值发送，而只是发送了指针变量所保存的地址值。数组作为消息体结构体成员是可以的。因为整个数组空间都在消息体结构体中。**

```
struct msgbuf{
```

long mtype 指定的消息队列编号，后面的数组才是要发送的数据，计算大小，也是这个数组所申请的空间大小。

接收方倒数第二个参数为：mtype的值(指定的消息队列编号)。

在接收的时候，必须指明是哪个消息队列进行接收。

### **7、套接字（Socket）**

更为一般的进程间通信机制，可用于不同机器之间的进程间通信。

一个套接口可以看作是进程间通信的端点（endpoint），每个套接口的名字都是唯一的（唯一的含义是不言而喻的），其他进程可以发现、连接并且与之通信。

通信域用来说明套接口通信的协议，不同的通信域有不同的通信协议以及套接口的地址结构等等，因此，创建一个套接口时，要指明它的通信域。比较常见的是unix域套接口（采用套接口机制实现单机内的进程间通信）及网际通信域。

- 基于TCP协议的Socket程序函数调用过程
  - TCP的服务端要先监听一个端口，一般是先调用bind函数
  - 在内核中，为每个Socket维护两个队列。一个是已经建立了连接的队列，这时候连接三次握手已经完毕，处于established状态；一个是还没有完全建立连接的队列，这个时候三次握手还没完成，处于syn_rcvd的状态。
  - 当服务端有了IP和端口号，就可以调用listen函数进行监听
  - 接下来，服务端调用accept函数，拿出一个已经完成的连接进行处理。如果还没有完成，就要等着。
  - 在服务端等待的时候，客户端可以通过connect函数发起连接，内核会给客户端分配一个临时的端口。一旦握手成功，服务端的accept就会返回另一个Socket。
  - 连接建立成功之后，双方开始通过read和write函数来读写数据
- 基于UDP协议的Socket程序函数调用过程
  - UDP是没有连接的，所以不需要三次握手，也就不需要调用listen和connect，但是，UDP的的交互仍然需要IP和端口号，因而也需要bind。
  - UDP是没有维护连接状态的，因而不需要每对连接建立一组Socket，而是只要有一个Socket，就能够和多个客户端通信
  - 每次通信的时候，都调用sendto和recvfrom，都可以传入IP地址和端口。

![img](https://picb.zhimg.com/v2-496a4ad707a95f37a003edec362b3dc5_b.jpg)

### 简单实现进程间的Socket的通信

实现步骤如下：

（1）使用socket套接字，并选择TCP/IP协议，端口号为6000

（2）客户端发送“this is a test”到服务端

（3）服务端收到后打印字符串，并回复“test ok”到客户端

（4）通信结束，断开连接

客户端socket通信的实现步骤

- socket()创建socket
- 设置socket的属性
- connect()向服务端发起连接
- send()向服务端发送消息
- recv()等待服务端的消息
- close()关闭socket，回收资源

服务端socket通信的实现步骤

- socket()创建socket
- bind()对socket进行绑定
- listen()监听绑定的socket，用于监听来自客户端的消息
- accept()用于接收客户端的连接请求和消息
- recv()函数接收客户端的消息，并将其保存到设置好的buffer中
- send()函数用于服务端向客户端发送消息
- close()关闭socket，回收资源



## 线程通信

线程通信主要可以分为三种方式，分别为**共享内存**、**消息传递**和**管道流**。每种方式有不同的方法来实现

- 共享内存：线程之间共享程序的公共状态，线程之间通过读-写内存中的公共状态来隐式通信。

> volatile共享内存

```java


public class TestVolatile {
    private static volatile boolean flag=true;
    public static void main(String[] args){
        new Thread(new Runnable() {
            public void run() {
                while (true){
                    if(flag){
                        System.out.println("线程A");
                        flag=false;
                    }
                }
            }
        }).start();


        new Thread(new Runnable() {
            public void run() {
                while (true){
                    if(!flag){
                        System.out.println("线程B");
                        flag=true;
                    }
                }
            }
        }).start();
    }
}

```



- 消息传递：线程之间没有公共的状态，线程之间必须通过明确的发送信息来显示的进行通信。

> wait/notify等待通知方式
> join方式

```java

public class WaitNotify {
    static boolean flag=true;
    static Object lock=new Object();
    public static void main(String[] args) throws InterruptedException {
        Thread waitThread=new Thread(new WaitThread(),"WaitThread");
        waitThread.start();
        TimeUnit.SECONDS.sleep(1);
        Thread notifyThread=new Thread(new NotifyThread(),"NotifyThread");
        notifyThread.start();
    }
    //等待线程
    static class WaitThread implements Runnable{
        public void run() {
            //加锁
            synchronized (lock){
                //条件不满足时，继续等待，同时释放lock锁
                while (flag){
                    System.out.println("flag为true，不满足条件，继续等待");
                    try {
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                //条件满足
                System.out.println("flag为false，我要从wait状态返回继续执行了");

            }

        }
    }
    //通知线程
    static class NotifyThread implements Runnable{

        public void run() {
            //加锁
            synchronized (lock){
                //获取lock锁，然后进行通知，但不会立即释放lock锁，需要该线程执行完毕
                lock.notifyAll();
                System.out.println("设置flag为false,我发出通知了，但是我不会立马释放锁");
                flag=false;
            }
        }
    }
 }
```

```java

public class TestJoin {
    public static void main(String[] args){
        Thread thread=new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("线程0开始执行了");
            }
        });
        thread.start();
        for (int i=0;i<10;i++){
            JoinThread jt=new JoinThread(thread,i);
            jt.start();
            thread=jt;
        }

    }

    static class JoinThread extends Thread{
        private Thread thread;
        private int i;

        public JoinThread(Thread thread,int i){
            this.thread=thread;
            this.i=i;
        }

        @Override
        public void run() {
            try {
                thread.join();
                System.out.println("线程"+(i+1)+"执行了");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```



- 管道流

> 管道输入/输出流的形式





## 线上某台虚机CPU Load过高

- 造成cpu load过高的原因： Full gc次数的增大、代码中存在Bug（例如死循环、正则的不恰当使用等）都有可能造成cpu load 增高。
  - jps -v：查看java进程号
  - top -Hp [java进程号]：查看当前进程下最耗费CPU的线程
  - printf "%x\n" [步骤2中的线程号]：得到线程的16进制表示
  - jstack [java进程号] | grep -A100 [步骤3的结果]：查看线程堆栈，定位代码行。





