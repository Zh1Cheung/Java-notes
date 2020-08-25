

# 并行和并发

并发是两个任务共享时间段，并行则是两个任务在同一时间发生，比如运行在多核CPU上。
如果一个程序要运行两个任务，并且只有一个CPU给它们分配了不同的时间片，那么这就是并发，而不是并行。

> 并行化是指为缩短任务执行时间，将一个任务分解成几部分，然后并行执行。

这和顺序执行的任务量是一样的，区别就像用更多的马来拉车，花费的时间自然减少了。
实际上，和顺序执行相比，并行化执行任务时，CPU承载的工作量更大。

> 数据并行化是指将数据分成块，为每块数据分配单独的处理单元。

还是拿马拉车那个例子打比方，就像从车里取出一些货物，放到另一辆车上，两辆马车都沿着同样的路径到达目的地。

当需要在大量数据上执行同样的操作时，数据并行化很管用。
它将问题分解为可在多块数据上求解的形式，然后对每块数据执行运算，最后将各数据块上得到的结果汇总，从而获得最终答案。

人们经常拿任务并行化和数据并行化做比较，在任务并行化中，线程不同，工作各异。
我们最常遇到的JavaEE应用容器便是任务并行化的例子之一，每个线程不光可以为不同用户服务，
还可以为同一个用户执行不同的任务，比如登录或往购物车添加商品。

- 概念
  - 【并发】：你有一套方案，可以让多个人一起把事情做的更高效
  - 但如果你真的雇了10个人，就可以很容易的让第1个人干第1人份和第2人份的活，第2个人干第3和第4人份的活…… 而这10个人同时在工地上干活，就是【并行】（Parallelism）
- 【并行】的上限是由【并发】的方法的设计决定的
  - 如果想让一个事情变得容易【并行】，先得让制定一个【并发】的方法。
  - 倘若一个事情压根就没有【并发】的方法，那么无论有多少个可以干活的人，也不能【并行】。比如你让20个人不铺路，而是一起去拧同一个灯泡，也只能有一个人踩在梯子上去拧，其他19个人只能看着，啥也干不了。
- 常见的误解
  - 并发是多个任务交替使用CPU，同一时刻只有一个任务在跑；并行是多个任务同时跑
    - 这个错误的理解就是：并发和并行是两个并列的，非此即彼的概念，一个状态要不就是并行的，要不就是并发的。这是完全错误的
    - 实际上【并发】和【并行】描述的是两个频道的事情
  - 并发是多个任务交替使用CPU，同一时刻只有一个任务在跑
    - 其实正确的理解是：针对一个问题，想到了一个可以拆解为多个【并发】的任务，这些任务执行时因为只有一个CPU只能“切换”的跑。
  - 并行是多个任务同时跑
    - 其实的意思是：如果这些并行执行的任务是解决同一个问题的，那么他们既是【并发】的，同时也是【并行】的。
- 为什么要这么理解并发
  - 将大的任务拆解为许许多多小的可以并发的任务是重要的编程思想。
- 高并发
  - 不像【并发】说的是“处理”，【并行】说的是“执行”，【高并发】说的是最终效果。
  - 【高并发】是指可以让软件系统在一段时间内能够处理大量的请求。比如每秒钟可以完成10万个请求。这是互联网系统的一个重要的特征。
  - 极端一点【高并发】甚至并不一定需要【并行】，只要处理速度快的足够满足要求就可以。如启动一个nginx的【OS进程】，它只能用到一个CPU核心，也就不可能【并行】。但是他如果能每秒能处理10万个请求，而业务需求只要求8万个请求就可以了，那么这个单进程的nginx本身就算【高并发】了。
- 并发这个词其实可以分两种方向来理解，一个是宏观，一个是微观。宏观就顾名思义就行了，即同时发生的请求或事件，就叫并发。微观理解就是，多个线程在同一个物理资源下的具体操作，不管单核也好双核也好





# 创建线程

Java中创建线程主要有三种方式：

一、继承Thread类创建线程类

（1）定义Thread类的子类，并重写该类的run方法，该run方法的方法体就代表了线程要完成的任务。因此把run()方法称为执行体。

（2）创建Thread子类的实例，即创建了线程对象。

（3）调用线程对象的start()方法来启动该线程。

```
package com.nf147.Constroller;

public class FirstThreadTest extends Thread {

    int i = 0;

    //重写run方法，run方法的方法体就是现场执行体
    public void run() {
        for (; i < 100; i++) {
            System.out.println(getName() + "  " + i);
        }
    }

    public static void main(String[] args) {

        for (int i = 0; i < 100; i++) {
            System.out.println(Thread.currentThread().getName() + "  : " + i);
            if (i == 50) {
                new FirstThreadTest().start();
                new FirstThreadTest().start();
            }
        }
    }


}
```



上述代码中Thread.currentThread()方法返回当前正在执行的线程对象。GetName()方法返回调用该方法的线程的名字。



二、通过Runnable接口创建线程类

 

（1）定义runnable接口的实现类，并重写该接口的run()方法，该run()方法的方法体同样是该线程的线程执行体。

 

（2）创建 Runnable实现类的实例，并依此实例作为Thread的target来创建Thread对象，该Thread对象才是真正的线程对象。

 

（3）调用线程对象的start()方法来启动该线程。

```
package com.nf147.Constroller;

public class RunnableThreadTest implements Runnable{
        private int i;
        public void run()
        {
            for(i = 0;i <100;i++)
            {
                System.out.println(Thread.currentThread().getName()+" "+i);
            }
        }
        public static void main(String[] args)
        {
            for(int i = 0;i < 100;i++)
            {
                System.out.println(Thread.currentThread().getName()+" "+i);
                if(i==20)
                {
                    RunnableThreadTest rtt = new RunnableThreadTest();
                    new Thread(rtt,"新线程1").start();
                    new Thread(rtt,"新线程2").start();
                }
            }

        }
}
```



 

三、通过Callable和Future创建线程

（1）创建Callable接口的实现类，并实现call()方法，该call()方法将作为线程执行体，并且有返回值。

（2）创建Callable实现类的实例，使用FutureTask类来包装Callable对象，该FutureTask对象封装了该Callable对象的call()方法的返回值。

（3）使用FutureTask对象作为Thread对象的target创建并启动新线程。

（4）调用FutureTask对象的get()方法来获得子线程执行结束后的返回值

 

实例代码：

```
package com.nf147.Constroller;

import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;

public class CallableThreadTest implements Callable<Integer> {


    public static void main(String[] args) {
        CallableThreadTest ctt = new CallableThreadTest();
        FutureTask<Integer> ft = new FutureTask<>(ctt);
        for (int i = 0; i < 100; i++) {
            System.out.println(Thread.currentThread().getName() + " 的循环变量i的值" + i);
            if (i == 20) {
                new Thread(ft, "有返回值的线程").start();
            }
        }
        try {
            System.out.println("子线程的返回值：" + ft.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

    }

    @Override
    public Integer call() throws Exception {
        int i = 0;
        for (; i < 100; i++) {
            System.out.println(Thread.currentThread().getName() + " " + i);
        }
        return i;
    }


}
```



二、创建线程的三种方式的对比

 

采用实现Runnable、Callable接口的方式创见多线程时，优势是：

 

线程类只是实现了Runnable接口或Callable接口，还可以继承其他类。

 

在这种方式下，多个线程可以共享同一个target对象，所以非常适合多个相同线程来处理同一份资源的情况，从而可以将CPU、代码和数据分开，形成清晰的模型，较好地体现了面向对象的思想。

 

劣势是：

 

编程稍微复杂，如果要访问当前线程，则必须使用Thread.currentThread()方法。

 

使用继承Thread类的方式创建多线程时优势是：

 

编写简单，如果需要访问当前线程，则无需使用Thread.currentThread()方法，直接使用this即可获得当前线程。

 

劣势是：



线程类已经继承了Thread类，所以不能再继承其他父类。

```
package com.nf147.Constroller;

public class FirstThreadTest extends Thread {

    int i = 0;

    //重写run方法，run方法的方法体就是现场执行体
public void run() {
        for (; i < 100; i++) {
            System.out.println(getName() + "  " + i);
        }
    }

    public static void main(String[] args) {

        for (int i = 0; i < 100; i++) {
            System.out.println(Thread.currentThread().getName() + "  : " + i);
            if (i == 50) {
                new FirstThreadTest().start();
                new FirstThreadTest().start();
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

- **通用的线程生命周期**

  - “五态模型”来描述。这五态分别是：初始状态、可运行状态、运行状态、休眠状态和终止状态

- **Java 中线程的生命周期**

  - BLOCKED、WAITING、TIMED_WAITING 是一种状态，即前面我们提到的休眠状态
  - RUNNABLE 与 BLOCKED 的状态转换
    - 只有一种场景会触发这种转换，就是线程等待 synchronized 的隐式锁
    - 平时所谓的 Java 在调用阻塞式 API 时，线程会阻塞，指的是操作系统线程的状态，并不是 Java 线程的状态。
    - JVM 层面并不关心操作系统调度相关的状态，因为在 JVM 看来，等待CPU 使用权（操作系统层面此时处于可执行状态）与等待 I/O（操作系统层面此时处于 休眠状态）没有区别，都是在等待某个资源，所以都归入了 RUNNABLE 状态。
  - RUNNABLE 与 WAITING 的状态转换
    - 三种场景会触发这种转换
      - 第一种场景，获得 synchronized 隐式锁的线程，调用无参数的 Object.wait() 方法
      - 第二种场景，调用无参数的 Thread.join() 方法。
      - 第三种场景，调用 LockSupport.park() 方法
  - RUNNABLE 与 TIMED_WAITING 的状态转换
    - 五种场景会触发这种转换：
      - Thread.sleep(long millis)
      - 获得 synchronized 隐式锁的线程，调用带超时参数的 Object.wait(long timeout) 方法；
      - Thread.join(long millis)
      - LockSupport.parkNanos(Object blocker, long deadline)
      - LockSupport.parkUntil(long deadline)
  - NEW 到 RUNNABLE 状态
    - 只要调用线程对象的start() 方法就可以了
  - RUNNABLE 到 TERMINATED 状态
    - 线程执行完 run() 方法后，会自动转换到 TERMINATED 状态，当然如果执行 run() 方法的时候异常抛出，也会导致线程终止。
    - 有时候我们需要强制中断 run() 方法的执行： interrupt() 方法
      - 被 interrupt 的线程，是怎么收到通知的呢？一种是异常，另一种是主动检测。
        - 当线程 A 处于 WAITING、TIMED_WAITING 状态时，如果其他线程调用线程 A 的interrupt() 方法，会使线程 A 返回到 RUNNABLE 状态，同时线程 A 的代码会触发InterruptedException 异常。
        - wait()、join()、sleep() 这样的方法，我们看这些方法的签名，发现都会 throws InterruptedException 这个异常。这个异常的触发条件就是：其他线程调用了该线程的 interrupt() 方法。
        - 如果线程处于 RUNNABLE 状态，并且没有阻塞在某个 I/O 操作上，例如中断计算圆周率的线程 A，这时就得依赖线程 A 主动检测中断状态了。如果其他线程调用线程 A 的 interrupt()方法，那么线程 A 可以通过 isInterrupted() 方法，检测是不是自己被中断了
        - Interrupter:线程在sleep期间被打断了，抛出一个InterruptedException异常，抛出异常后，中断标示会自动清除掉，导致th.isInterrupted()一直都是返回false的。
          - 正确的处理方式是在执行线程的run()方法中捕获到InterruptedException异常，并重新设置中断标志位（也就是在捕获InterruptedException异常的catch代码块中，重新调用当前线程的interrupt()方法）。

- I/O 密集型程序和 CPU 密集型程序

  - 对于 CPU 密集型的计算场景，线程的数量一般会设置为“CPU 核数 +1”

  - I/O 密集型计算场景，最佳线程数 =CPU 核数 * [ 1 +（I/O 耗时 / CPU 耗时）]

    - 当I/O 耗时远远大于CPU耗时时，"2 * CPU 的核数 + 1"会导致所有线程在长时间下都处于 

      等待I/O操作的状态，而无法合理利用CPU

    - Nginx作为反向代理服务器，那么它会通过负载均衡策略调用后端的服务器，而远程调用属于IO操作，所以此处Nginx作为IO密集型的操作。但因为它 采用的是非阻塞IO模型，所以工作的方式又类似于CPU密集型，所以设置的最佳线程数为CPU的核数。

  - 一般来讲，随着线程数的增加，吞吐量会增加，延迟也会缓慢增加；但是当线程数增加到一定程度，吞吐量就会开始下降，延迟会迅速增加。这个时候基本上就是线程能够设置的最大值了。

- 每个方法在调用栈里都有自己的独立空间，称为栈帧

  - 每个栈帧里都有对应方法需要的参数和返回地址。当调用方法时，会创建新的栈帧，并压入调用栈；当方法返回时，对应的栈帧就会被自动弹出。也就是说，栈帧和方法是同生共死的。
  - 局部变量放到了调用栈里。不会有并发问题。
  - 每个线程都有自己独立的调用栈
  - 采用线程封闭技术的案例非常多，例如从数据库连接池里获取的连接 Connection，在JDBC 规范里并没有要求这个 Connection 必须是线程安全的。数据库连接池通过线程封闭技术，保证一个 Connection 一旦被一个线程获取之后，在这个线程关闭 Connection 之前的这段时间里，不会再分配给其他线程，从而保证了 Connection 不会有并发问题。

- 对共享变量进行封装，要避免“逸出”，所谓“逸出”简单讲就是共享变量逃逸到对象的外面

  - 构造函数里的this“逸出”。这些都是必须要避免的





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

- 假设我们有一个线程池，核心线程数为10，最大线程数也为20，任务队列为100。现在来了100个任务，线程池里现在有几个线程运行
  - 先进队列，到最大值,再起线程
    - ThreadPoolExecutor的execute方法中
    - 有100个任务添加进来时，剩下先起10个核心线程，剩下90个任务都丢进队列里，因此线程池里只有10个线程在执行
  - 先起线程，到最大值,再进队列
    - 在dubbo中，有一种线程池叫EagerThreadPoolExecutor线程池。该线程池的execute方法
    - 它调的还是父类的execute方法，也还是ThreadPoolExecutor中的execute方法！但是，它的队列！是一种自定义队列，叫TaskQueue,它的offer方法如下
      - 当前线程数小于最大线程数时，则直接返回false。
      - ThreadPoolExecutor中的execute方法中的第二步的条件中，如果workQueue.offer返回为fasle,则直接进入第三步，创建新任务
      - EagerThreadPoolExecutor线程池通过自定义队列的这么一种形式，改写了线程池的机制。这种线程池的机制是核心线程数不够了，先起线程，当线程达到最大值后，后面的任务就丢进队列！
      - 因此，如果按照这么一套机制，当100个任务添加进来时，直接会起20个线程，剩下80个任务都丢进队列！



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





## 线程池的坑

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

- 不建议使用 Executors 的最重要的原因是：Executors 提供的很多方法默认使用的都是无界的 LinkedBlockingQueue，高负载情境下，无界队列很容易导致 OOM，而 OOM 会导致所有请求都无法处理，这是致命问题。所以强烈建议使用有界队列。

  - 使用有界队列，当任务过多时，线程池会触发执行拒绝策略.线程池默认的拒绝策略会throw RejectedExecutionException 这是个运行时异常，对于运行时异常编译器并不强制catch 它，所以开发人员很容易忽略。因此默认拒绝策略要慎重使用。
  - 最致命的是任务虽然异常了，但是你却获取不到任何通知，这会让你误以为任务都执行得很正常。虽然线程池提供了很多用于异常处理的方法，但是最稳妥和简单的方案还是捕获所有异常并按需处理，

- Java 8 的 parallel stream 功能

  - 可以让我们很方便地并行处理集合中的元素，其背后是共享同一个 ForkJoinPool，默认并行度是CPU 核数 -1
  - 对于 CPU 绑定的任务来说，使用这样的配置比较合适，但如果集合操作涉及同步 IO 操作的话（比如数据库操作、外部服务调用等），建议自定义一个ForkJoinPool（或普通线程池）

- 我们改进了 ThreadPoolHelper 使其能够返回复用的线程池。如果我们不小心每次都创建了这样一个自定义的线程池（10 核心线程，50 最大线程，2 秒回收的），反复执行测试接口线程，最终可以被回收吗？会出现 OOM 问题吗？

  - 每次请求都新建线程池，每个线程池的核心数都是10, 虽然自定义线程池设置2秒回收，但是没超过线程池核心数10是不会被回收的, 不间断的请求过来导致创建大量线程，最终OOM
  - ThreadPoolExecutor回收不了，工作线程Worker是内部类，只要它活着，换句话说线程在跑，就会阻止ThreadPoolExecutor回收，所以其实ThreadPoolExecutor是无法回收的，并不能认为ThreadPoolExecutor没有引用就能回





## BIO

- 同步异步、阻塞非阻塞
  - 同步或异步（synchronous/asynchronous）。简单来说，同步是一种可靠的有序运行机制，当我们进行同步操作时，后续的任务是等待当前调用返回，才会进行下一步；而异步则相反，其他任务不需要等待当前调用返回，通常依靠事件、回调等机制来实现任务间次序关系。
  - 区分阻塞与非阻塞（blocking/non-blocking）。在进行阻塞操作时，当前线程会处于阻塞状态，无法从事其他任务，只有当条件就绪才能继续，比如 ServerSocket 新连接建立完毕，或数据读取、写入操作完成；而非阻塞则是不管 IO 操作是否结束，直接返回，相应操作在后台继续处理
- io
  - 首先，传统的 java.io 包，它基于流模型实现，提供了我们最熟知的一些 IO 功能，比如File 抽象、输入输出流等。交互方式是同步、阻塞的方式，也就是说，在读取输入流或者写入输出流时，在读、写动作完成之前，线程会一直阻塞在那里，它们之间的调用是可靠的线性顺序
  - 线程上下文切换开销会在高并发时变得很明显，这是同步阻塞方式的低扩展性劣势
- 传统 I/O 的性能问题
  - 磁盘 I/O 操作和网络 I/O 操作
  - 多次内存复制，导致不必要的数据拷贝和上下文切换，从而降低 I/O的性能
  - 大量连接请求时，创建大量监听线程，这时如果线程没有数据就绪就会被挂起，然后进入阻塞状态。
    - 阻塞线程在阻塞状态是不会占用CPU资源的，但是会被唤醒争夺CPU资源。操作系统将CPU轮流分配给线程任务，当线程数量越多的时候，当某个线程在规定的时间片运行完之后，会被其他线程抢夺CPU资源，此时会导致上下文切换。抢夺越激烈，上下文切换就越频繁
  - 在传统 I/O 中，InputStream 的 read() 是一个 while 循环操作，它会一直等待数据读取直到数据就绪才会返回。这就意味着如果没有数据就绪，这个读取操作将会一直被挂起，用线程将会处于阻塞状态。

#### 传统BIO模型分析

让我们先回忆一下传统的服务器端同步阻塞I/O处理（也就是BIO，Blocking I/O）的经典编程模型：

```java
{
 ExecutorService executor = Excutors.newFixedThreadPollExecutor(100);//线程池

 ServerSocket serverSocket = new ServerSocket();
 serverSocket.bind(8088);
 while(!Thread.currentThread.isInturrupted()){//主线程死循环等待新连接到来
 Socket socket = serverSocket.accept();
 executor.submit(new ConnectIOnHandler(socket));//为新的连接创建新的线程
}

class ConnectIOnHandler extends Thread{
    private Socket socket;
    public ConnectIOnHandler(Socket socket){
       this.socket = socket;
    }
    public void run(){
      while(!Thread.currentThread.isInturrupted()&&!socket.isClosed()){死循环处理读写事件
          String someThing = socket.read()....//读取数据
          if(someThing!=null){
             ......//处理数据
             socket.write()....//写数据
          }

      }
    }
}
```

这是一个经典的每连接每线程的模型，之所以使用多线程，主要原因在于socket.accept()、socket.read()、socket.write()三个主要函数都是同步阻塞的，当一个连接在处理I/O的时候，系统是阻塞的，如果是单线程的话必然就挂死在那里；但CPU是被释放出来的，开启多线程，就可以让CPU去处理更多的事情。**其实这也是所有使用多线程的本质：**

1. 利用多核。
2. 当I/O阻塞系统，但CPU空闲的时候，可以利用多线程使用CPU资源。

现在的多线程一般都使用线程池，可以让线程的创建和回收成本相对较低。在活动连接数不是特别高（小于单机1000）的情况下，这种模型是比较不错的，可以让每一个连接专注于自己的I/O并且编程模型简单，也不用过多考虑系统的过载、限流等问题。线程池本身就是一个天然的漏斗，可以缓冲一些系统处理不了的连接或请求。

不过，这个模型最本质的问题在于，严重依赖于线程。但线程是很"贵"的资源，主要表现在：

1. 线程的创建和销毁成本很高，在Linux这样的操作系统中，线程本质上就是一个进程。创建和销毁都是重量级的系统函数。
2. 线程本身占用较大内存，像Java的线程栈，一般至少分配512K～1M的空间，如果系统中的线程数过千，恐怕整个JVM的内存都会被吃掉一半。
3. 线程的切换成本是很高的。操作系统发生线程切换的时候，需要保留线程的上下文，然后执行系统调用。如果线程数过高，可能执行线程切换的时间甚至会大于线程执行的时间，这时候带来的表现往往是系统load偏高、CPU sy使用率特别高（超过20%以上)，导致系统几乎陷入不可用的状态。
4. 容易造成锯齿状的系统负载。因为系统负载是用活动线程数或CPU核心数，一旦线程数量高但外部网络环境不是很稳定，就很容易造成大量请求的结果同时返回，激活大量阻塞线程从而使系统负载压力过大。

**所以，当面对十万甚至百万级连接的时候，传统的BIO模型是无能为力的。随着移动端应用的兴起和各种网络游戏的盛行，百万级长连接日趋普遍，此时，必然需要一种更高效的I/O处理模型。**



#### BIO、NIO、AIO

传统的BIO里面socket.read()，如果TCP RecvBuffer里没有数据，函数会一直阻塞，直到收到数据，返回读到的数据。

对于NIO，如果TCP RecvBuffer有数据，就把数据从网卡读到内存，并且返回给用户；反之则直接返回0，永远不会阻塞。

最新的AIO(Async I/O)里面会更进一步：不但等待就绪是非阻塞的，就连数据从网卡到内存的过程也是异步的。

换句话说，BIO里用户最关心“我要读”，NIO里用户最关心"我可以读了"，在AIO模型里用户更需要关注的是“读完了”。

NIO一个重要的特点是：socket主要的读、写、注册和接收函数，在等待就绪阶段都是非阻塞的，真正的I/O操作是同步阻塞的（消耗CPU但性能非常高）。





#### 常见I/O模型对比

**所有的系统I/O都分为两个阶段：等待就绪和操作。**举例来说，读函数，分为等待系统可读和真正的读；同理，写函数分为等待网卡可以写和真正的写。

需要说明的是等待就绪的阻塞是不使用CPU的，是在“空等”；而真正的读写操作的阻塞是使用CPU的，真正在"干活"，而且这个过程非常快，属于memory copy，带宽通常在1GB/s级别以上，可以理解为基本不耗时。

下图是几种常见I/O模型的对比：

![img](https://pic3.zhimg.com/v2-f47206d5b5e64448744b85eaf568f92d_b.jpg)



以socket.read()为例子：



#### 如何结合事件模型使用NIO同步非阻塞特性

socket.read()和socket.write()函数中返回，这两个函数无法进行有效的中断。所以除了多开线程另起炉灶，没有好的办法利用CPU。

NIO的读写函数可以立刻返回，这就给了我们不开线程利用CPU的最好机会：如果一个连接不能读写（socket.read()返回0或者socket.write()返回0），我们可以把这件事记下来，记录的方式通常是在Selector上注册标记位，然后切换到其它就绪的连接（channel）继续进行读写。

下面具体看下如何利用事件模型单线程处理所有I/O请求：

**NIO的主要事件有几个：读就绪、写就绪、有新连接到来。**

我们首先需要注册当这几个事件到来的时候所对应的处理器。然后在合适的时机告诉事件选择器：我对这个事件感兴趣。对于写操作，就是写不出去的时候对写事件感兴趣；对于读操作，就是完成连接和系统没有办法承载新读入的数据的时；对于accept，一般是服务器刚启动的时候；而对于connect，一般是connect失败需要重连或者直接异步调用connect的时候。

其次，用一个死循环选择就绪的事件，会执行系统调用（Linux 2.6之前是select、poll，2.6之后是epoll，Windows是IOCP），还会阻塞的等待新事件的到来。**新事件到来的时候，会在selector上注册标记位，标示可读、可写或者有连接到来。**

注意，select是阻塞的，无论是通过操作系统的通知（epoll）还是不停的轮询(select，poll)，这个函数是阻塞的。所以你可以放心大胆地在一个while(true)里面调用这个函数而不用担心CPU空转。

所以我们的程序大概的模样是：

```java
   interface ChannelHandler{
      void channelReadable(Channel channel);
      void channelWritable(Channel channel);
   }
   class Channel{
     Socket socket;
     Event event;//读，写或者连接
   }

   //IO线程主循环:
   class IoThread extends Thread{
   public void run(){
   Channel channel;
   while(channel=Selector.select()){//选择就绪的事件和对应的连接
      if(channel.event==accept){
         registerNewChannelHandler(channel);//如果是新连接，则注册一个新的读写处理器
      }
      if(channel.event==write){
         getChannelHandler(channel).channelWritable(channel);//如果可以写，则执行写事件
      }
      if(channel.event==read){
          getChannelHandler(channel).channelReadable(channel);//如果可以读，则执行读事件
      }
    }
   }
   Map<Channel，ChannelHandler> handlerMap;//所有channel的对应事件处理器
  }
```

这个程序很简短，也是最简单的Reactor模式：注册所有感兴趣的事件处理器，单线程轮询选择就绪事件，执行事件处理器。

**NIO由原来的阻塞读写（占用线程）变成了单线程轮询事件，找到可以进行读写的网络描述符进行读写。除了事件的轮询是阻塞的（没有可干的事情必须要阻塞），剩余的I/O操作都是纯CPU操作，没有必要开启多线程。**

**并且由于线程的节约，连接数大的时候因为线程切换带来的问题也随之解决，进而为处理海量连接提供了可能。**



单线程处理I/O的效率确实非常高，没有线程切换，只是拼命的读、写、选择事件。**但现在的服务器，一般都是多核处理器，如果能够利用多核心进行I/O，无疑对效率会有更大的提高。**

仔细分析一下我们需要的线程，其实主要包括以下几种：

1. 事件分发器，单线程选择就绪的事件。
2. I/O处理器，包括connect、read、write等，这种纯CPU操作，一般开启CPU核心个线程就可以。
3. 业务线程，在处理完I/O后，业务一般还会有自己的业务逻辑，有的还会有其他的阻塞I/O，如DB操作，RPC等。只要有阻塞，就需要单独的线程。

Java的Selector对于Linux系统来说，有一个致命限制：同一个channel的select不能被并发的调用。因此，如果有多个I/O线程，必须保证：一个socket只能属于一个IoThread，而一个IoThread可以管理多个socket。

另外连接的处理和读写的处理通常可以选择分开，这样对于海量连接的注册和读写就可以分发。虽然read()和write()是比较高效无阻塞的函数，但毕竟会占用CPU，如果面对更高的并发则无能为力。

#### NIO在客户端的魔力

通过上面的分析，可以看出**NIO在服务端对于解放线程，优化I/O和处理海量连接方面，确实有自己的用武之地。那么在客户端上，NIO又有什么使用场景呢**?

常见的客户端BIO+连接池模型，可以建立n个连接，然后当某一个连接被I/O占用的时候，可以使用其他连接来提高性能。

但多线程的模型面临和服务端相同的问题：如果指望增加连接数来提高性能，则连接数又受制于线程数、线程很贵、无法建立很多线程，则性能遇到瓶颈



## NIO高级主题

## Proactor与Reactor

一般情况下，I/O 复用机制需要**事件分发器（event dispatcher）**。 事件分发器的作用，即将那些读写事件源分发给各读写事件的处理者，在开始的时候需要在分发器那里注册感兴趣的事件，并提供相应的处理者（event handler)，或者是回调函数；事件分发器在适当的时候，会将请求的事件分发给这些handler或者回调函数。

**涉及到事件分发器的两种模式称为：Reactor和Proactor**。 Reactor模式是基于同步I/O的，而Proactor模式是和异步I/O相关的。在Reactor模式中，事件分发器等待某个事件或者可应用或个操作的状态发生（比如文件描述符可读写，或者是socket可读写），事件分发器就把这个事件传给事先注册的事件处理函数或者回调函数，由后者来做实际的读写操作。

而在Proactor模式中，事件处理者（或者代由事件分发器发起）直接发起一个异步读写操作（相当于请求），而实际的工作是由操作系统来完成的。发起时，需要提供的参数包括用于存放读到数据的缓存区、读的数据大小或用于存放外发数据的缓存区，以及这个请求完后的回调函数等信息。事件分发器得知了这个请求，它默默等待这个请求的完成，然后转发完成事件给相应的事件处理者或者回调。举例来说，在Windows上事件处理者投递了一个异步IO操作（称为overlapped技术），事件分发器等IO Complete事件完成。这种异步模式的典型实现是基于操作系统底层异步API的，所以我们可称之为“系统级别”的或者“真正意义上”的异步，因为具体的读写是由操作系统代劳的。

举个例子，将有助于理解Reactor与Proactor二者的差异，以读操作为例（写操作类似）。

#### 在Reactor中实现读

- 注册读就绪事件和相应的事件处理器。
- 事件分发器等待事件。
- 事件到来，激活分发器，分发器调用事件对应的处理器。
- 事件处理器完成实际的读操作，处理读到的数据，注册新的事件，然后返还控制权。

#### 在Proactor中实现读：

- 处理器发起异步读操作（注意：操作系统必须支持异步IO）。在这种情况下，处理器无视IO就绪事件，它关注的是完成事件。
- 事件分发器等待操作完成事件。
- 在分发器等待过程中，操作系统利用并行的内核线程执行实际的读操作，并将结果数据存入用户自定义缓冲区，最后通知事件分发器读操作完成。
- 事件分发器呼唤处理器。
- 事件处理器处理用户自定义缓冲区中的数据，然后启动一个新的异步操作，并将控制权返回事件分发器。

可以看出，两个模式的相同点，都是对某个I/O事件的事件通知（即告诉某个模块，这个I/O操作可以进行或已经完成)。在结构上，两者也有相同点：事件分发器负责提交IO操作（异步)、查询设备是否可操作（同步)，然后当条件满足时，就回调handler；不同点在于，异步情况下（Proactor)，当回调handler时，表示I/O操作已经完成；同步情况下（Reactor)，回调handler时，表示I/O设备可以进行某个操作（can read 或 can write)。

**下面，我们将尝试应对为Proactor和Reactor模式建立可移植框架的挑战。**在改进方案中，我们将Reactor原来位于事件处理器内的Read/Write操作移至分发器（不妨将这个思路称为“模拟异步”），以此寻求将Reactor多路同步I/O转化为模拟异步I/O。以读操作为例子，改进过程如下：

- 注册读就绪事件和相应的事件处理器。并为分发器提供数据缓冲区地址，需要读取数据量等信息。
- 分发器等待事件（如在select()上等待）。
- 事件到来，激活分发器。分发器执行一个非阻塞读操作（它有完成这个操作所需的全部信息），最后调用对应处理器。
- 事件处理器处理用户自定义缓冲区的数据，注册新的事件（当然同样要给出数据缓冲区地址，需要读取的数据量等信息），最后将控制权返还分发器。
  如我们所见，通过对多路I/O模式功能结构的改造，可将Reactor转化为Proactor模式。改造前后，模型实际完成的工作量没有增加，只不过参与者间对工作职责稍加调换。没有工作量的改变，自然不会造成性能的削弱。对如下各步骤的比较，可以证明工作量的恒定：

#### 标准/典型的Reactor：

- 步骤1：等待事件到来（Reactor负责）。
- 步骤2：将读就绪事件分发给用户定义的处理器（Reactor负责）。
- 步骤3：读数据（用户处理器负责）。
- 步骤4：处理数据（用户处理器负责）。

#### 改进实现的模拟Proactor：

- 步骤1：等待事件到来（Proactor负责）。

- 步骤2：得到读就绪事件，执行读数据（现在由Proactor负责）。

- 步骤3：将读完成事件分发给用户处理器（Proactor负责）。

- 步骤4：处理数据（用户处理器负责）。

  对于不提供异步I/O API的操作系统来说，这种办法可以隐藏Socket API的交互细节，从而对外暴露一个完整的异步接口。借此，我们就可以进一步构建完全可移植的，平台无关的，有通用对外接口的解决方案。



### Buffer的选择

通常情况下，操作系统的一次写操作分为两步：

1. 将数据从用户空间拷贝到系统空间。
2. 从系统空间往网卡写。同理，读操作也分为两步：
   ① 将数据从网卡拷贝到系统空间；
   ② 将数据从系统空间拷贝到用户空间。

对于NIO来说，缓存的使用可以使用DirectByteBuffer和HeapByteBuffer。如果使用了DirectByteBuffer，一般来说可以减少一次系统空间到用户空间的拷贝。但Buffer创建和销毁的成本更高，更不宜维护，通常会用内存池来提高性能。

如果数据量比较小的中小应用情况下，可以考虑使用heapBuffer；反之可以用directBuffer。

### NIO存在的问题

使用NIO != 高性能，当连接数<1000，并发程度不高或者局域网环境下NIO并没有显著的性能优势。

NIO并没有完全屏蔽平台差异，它仍然是基于各个操作系统的I/O系统实现的，差异仍然存在。使用NIO做网络编程构建事件驱动模型并不容易，陷阱重重。

推荐大家使用成熟的NIO框架，如Netty，MINA等。解决了很多NIO的陷阱，并屏蔽了操作系统的差异，有较好的性能和编程模型。



### NIO给我们带来了些什么

> - 事件驱动模型
> - 避免多线程
> - 单线程处理多任务
> - 非阻塞I/O，I/O读写不再阻塞，而是返回0
> - 基于block的传输，通常比基于流的传输更高效
> - 更高级的IO函数，zero-copy
> - IO多路复用大大提高了Java网络应用的可伸缩性和实用性

- 网络 I/O 模型优化

  - B和N通常是针对数据是否就绪的处理方式来
    sync和async是对阻塞进行更深一层次的阐释，区别在于数据拷贝由用户线程完成还是内核完成，讨论范围一定是两个线程及以上了。

    > 同步阻塞，从数据是否准备就绪到数据拷贝都是由用户线程完成
    >
    > 同步非阻塞，数据是否准备就绪由内核判断，数据拷贝还是用户线程完成
    >
    > 异步非阻塞，数据是否准备就绪到数据拷贝都是内核来完成
    >
    > 所以真正的异步IO一定是非阻塞的。
    >
    > 多路复用IO即使有Reactor通知用户线程也是同步IO范畴，因为数据拷贝期间仍然是用户线程完成。
    >
    > 所以假如我们没有内核支持数据拷贝的情况下，讨论的非阻塞并不是彻底的非阻塞，也就没有引入sync和async讨论的必要了  

- **线程模型优化**

  - Reactor 模型是同步 I/O 事件处理的一种常见模型，其核心思想是将 I/O 事件注册到多路复用器上，一旦有 I/O 事件触发，多路复用器就会将事件分发到事件处理器中，执行就绪的 I/O 事件操作。
  - 单线程 Reactor 线程模型
    - 最开始 NIO 是基于单线程实现的，所有的 I/O 操作都是在一个 NIO 线程上完成。由于NIO 是非阻塞 I/O，理论上一个线程可以完成所有的 I/O 操作。
  - 多线程 Reactor 线程模型
    - Acceptor 线程来监听连接请求事件，当连接成功之后，会将建立的连接注册到多路复用器中，一旦监听到事件，将交给 Worker 线程池来负责处理
  - 主从 Reactor 线程模型
    - Acceptor 不再是一个单独的 NIO 线程，而是一个线程池。Acceptor 接收到客户端的 TCP 连接请求，建立连接之后，后续的 I/O 操作将交给 Worker I/O 线程。

- 基于线程模型的 Tomcat 参数调优

  - 在 NIO 中，Tomcat 新增了一个 Poller 线程池，Acceptor 监听到连接后，不是直接使用Worker 中的线程处理请求，而是先将请求发送给了 Poller 缓冲队列。在 Poller 中，维护了一个 Selector 对象，通过遍历 Selector，找出其中就绪的 I/O 操作，并使用 Worker 中的线程处理相应的请求
    - acceptorThreadCount：该参数代表 Acceptor 的线程数量
    - maxThreads：专门处理 I/O 操作的 Worker 线程数量
    - acceptCount ：这里的 acceptCount 指的是 accept 队列的大小。
      - Tomcat 的 Acceptor 线程是负责从 accept 队列中取出该 connection，然后交给工作线程去执行相关操作
      - 当 Http 关闭 keep alive，在并发量比较大时，可以适当地调大这个值。而在 Http 开启keep alive 时，因为 Worker 线程数量有限，Worker 线程就可能因长时间被占用，而连接在 accept 队列中等待超时。如果 accept 队列过大，就容易浪费连接。
    - maxConnections：表示有多少个 socket 连接到 Tomcat 上。





## NIO

- 同步非阻塞的I/O模型，也是I/O多路复用的基础，已经被越来越多地应用到大型应用服务器，成为解决高并发与大量连接、I/O处理问题的有效方式。

- 如何优化 I/O 操作

  - NIO 的发布优化了内存复制以及阻塞导致的严重性能问题
  - 使用缓冲区优化读写流操作
    - 传统 I/O 和 NIO 的最大区别就是传统 I/O 是面向流，NIO 是面向 Buffer。buffer 可以文件一次性读入内存再做后续处理，而传统的方式是边读文件边处理数据
    - 缓冲区（Buffer）和通道（Channel）
      - Buffer 是一块连续的块，是 NIO 读写数据的中转地。
        - 在没有bytebuff缓存的情况下，一旦读取数据的SO_RCVBUF满了，将会通知对端TCP协议中的窗口关闭（滑动窗口），将影响TCP发送端，这也就影响到了整个TCP通信的速度。而有了bytebuff，我们可以先将读取的数据缓存在bytebuff中，提高TCP的通信能力。
      - Channel 表示缓冲数据的源头或者目的地，它用于读取缓冲或者写入数据，是访问缓冲的接口。
  - 使用 DirectBuffer 减少内存复制
    - NIO 的 Buffer 除了做了缓冲块优化之外，还提供了一个可以直接访问物理内存的类DirectBuffer。普通的 Buffer 分配的是 JVM 堆内存，而 DirectBuffer 是直接分配物理内存。
    - 直接将步骤简化为从内核空间复制到外部设备，减少了数据拷贝
    - 由于 DirectBuffer 申请的是非 JVM 的物理内存，所以创建和销毁的代价很小。DirectBuffer 申请的内存并不是直接由 JVM 负责垃圾回收，但在 DirectBuffer 包类被回收时，会通过 Java Reference 机制来释放该内存块

- NIO主要组成部分

  - Buffer，高效的数据容器，除了布尔类型，所有原始数据类型都有相应的 Buffer 实现。
  - Channel，类似在 Linux 之类操作系统上看到的文件描述符，是 NIO 中被用来支持批量式 IO 操作的一种抽象。
    - 外部设备(磁盘)通过DMA控制器（DMAC），向CPU提出接管总线控制权的总线请求
  - CPU对某个设备接口响应DMA请求时，会让出总线控制权。于是在DMA控制器的管理下，磁盘和存储器直接进行数据交换，而不需CPU干预。 
    - 通道则是在DMA的基础上增加了能执行有限通道指令的I/O控制器，代替CPU管理控制外设
  - File 或者 Socket，通常被认为是比较高层次的抽象，而 Channel 则是更加操作系统底层的一种抽象，这也使得 NIO 得以充分利用现代操作系统底层机制，获得特定场景的性能优化。
  - Selector，是 NIO 实现多路复用的基础，它提供了一种高效的机制，可以检测到注册在Selector 上的多个 Channel 中，是否有 Channel 处于就绪状态，进而实现了单线程对多 Channel 的高效管理
    - selector 是 Java NIO 编程的基础。用于检查一个或多个 NIO Channel 的状态是否处于可读、可写。
    - selector 是基于事件驱动实现的，一个线程使用一个 Selector，通过轮询的方式，可以监听多个 Channel 上的事件。
    - 目前操作系统的 I/O 多路复用机制都使用了 epoll，相比传统的 select 机制，epoll 没有最大连接句柄 1024 的限制。所以 Selector 在理论上可以轮询成千上万的客户端。

- NIO 多路复用

  - 首先，通过 Selector.open() 创建一个 Selector，作为类似调度员的角色。
  - 然后，创建一个 ServerSocketChannel，并且向 Selector 注册，通过指定SelectionKey.OP_ACCEPT，告诉调度员，它关注的是新的连接请求。注意，为什么我们要明确配置非阻塞模式呢？这是因为阻塞模式下，注册操作是不允许的，会抛出 IllegalBlockingModeException 异常。
  - Selector 阻塞在 select 操作，当有 Channel 发生接入请求，就会被唤醒

- 对于多路复用IO，当出现有的IO请求在数据拷贝阶段，会出现由于资源类型过份庞大而导致线程长期阻塞，最后造成性能瓶颈的情况




## IO多路复用

- 复用是指一个线程可以服务多条IO流，I/O multiplexing 这里面的 multiplexing 指的其实是在单个线程通过记录跟踪每一个Socket(I/O流)的状态来同时管理多个I/O流。
  - IO多路复用的优势在于，当处理的消耗对比IO几乎可以忽略不计时，可以处理大量的并发IO，而不用消耗太多CPU/内存
  - IO多路复用 + 单进（线）程有个额外的好处，就不会有并发编程的各种坑问题，比如在nginx里，redis里，编程实现都会很简单很多。
  - Java世界里，因为JDBC这个东西是BIO的，所以在我们常见的Java服务里没有办法做到全部NIO化，必须得弄成多线程模型。如果要在做Java web服务这个大场景下享受IO多路复用的好处，要不就是不需要DB的，要不就是得用Vert.X一类的纯NIO框架把DB IO访问也得框进来。
- I/O 复用
  - Linux 提供了 I/O 复用函数 select/poll/epoll，进程将一个或多个读操作通过系统调用函数，阻塞在函数操作上。这样，系统内核就可以帮我们侦测多个读操作是否处于就绪状态。
  - select() 函数调用后会阻塞，直到有描述符就绪或者超时，函数返回。当 select 函数返回后，可以通过函数 FD_ISSET 遍历 fdset，来找到就绪的描述符
  - select() 函数监视的文件描述符分 3 类，分别是 writefds（写文件描述符）、readfds（读文件描述符）以及 exceptfds（异常事件文件描述符）
  - select 会修改传入的参数数组，这个对于一个需要调用很多次的函数，是非常不友好的。
  - select 只能监视1024个链接， ，linux 定义在头文件中的，参见*FD_SETSIZE。*
  - select 不是线程安全的，如果你把一个sock加入到select, 然后突然另外一个线程发现，尼玛，这个sock不用，要收回。对不起，这个select 不支持的，如果你丧心病狂的竟然关掉这个sock, select的标准行为是。。呃。。不可预测的， 这个可是写在文档中的哦.
    - “如果在另一个线程中关闭了由select（）监视的文件描述符，则结果未指定”
  - poll() 和 select() 存在一个相同的缺点，那就是包含大量文件描述符的数组被整体复制到用户态和内核的地址空间之间，而无论这些文件描述符是否就绪，他们的开销都会随着文件描述符数量的增加而线性增大
  - 在2002, 大神 Davide Libenzi 实现了epoll.
  - epoll 使用事件驱动的方式代替轮询扫描fd。
    - 一旦某个文件描述符就绪时，内核会采用类似 callback 的回调机制，迅速激活这个文件描述符，当进程调用 epoll_wait() 时便得到通知，之后进程将完成相关 I/O 操作
    - 进\线程并不是在socket上阻塞  而是在select/epoll上阻塞   socket是否阻塞是交给内核来判断然后给进程发送信号
- Epoll代理
  - 没有最大并发连接的限制，能打开的fd上限远大于1024（1G的内存能监听约10万个端口）
  - 采用回调的方式，效率提升。只有活跃可用的fd才会调用callback函数，也就是说 epoll 只管你“活跃”的连接，而跟连接总数无关，因此在实际的网络环境中，epoll的效率就会远远高于select和poll。只轮询发出了事件的流，哪个流发生了怎样的IO事件会通知处理线程，因此对这些流的操作都是有意义的，复杂度降低到了O(1)
  - 内存拷贝。使用mmap()文件映射内存来加速与内核空间的消息传递，减少复制开销
- CPU本来就是线性的不论什么都需要顺序处理并行只能是多核CPU
- io多路复用本来就是用来解决对多个I/O监听时,一个I/O阻塞影响其他I/O的问题,跟多线程没关系.
- 跟多线程相比较,线程切换需要切换到内核进行线程切换,需要消耗时间和资源.而I/O多路复用不需要切换线/进程,效率相对较高,特别是对高并发的应用nginx就是用I/O多路复用,故而性能极佳.但多线程编程逻辑和处理上比I/O多路复用简单.而I/O多路复用处理起来较为复杂







## epoll

- 进程阻塞为什么不占用cpu资源

  - 了解epoll本质的**第三步**，要从**操作系统进程调度**的角度来看数据接收。阻塞是进程调度的关键一环，指的是进程在等待某事件（如接收到网络数据）发生之前的等待状态，recv、select和epoll都是阻塞方法。

  - 为简单起见，我们从普通的recv接收开始分析，先看看下面代码：

    ```c++
    //创建socket
    int s = socket(AF_INET, SOCK_STREAM, 0);   
    //绑定
    bind(s, ...)
    //监听
    listen(s, ...)
    //接受客户端连接
    int c = accept(s, ...)
    //接收客户端数据
    recv(c, ...);
    //将数据打印出来
    printf(...)
    ```

    这是一段最基础的网络编程代码，先新建socket对象，依次调用bind、listen、accept，最后调用recv接收数据。recv是个阻塞方法，当程序运行到recv时，它会一直等待，直到接收到数据才往下执行。

- 阻塞的原理

  - 操作系统为了支持多任务，实现了进程调度的功能，会把进程分为“运行”和“等待”等几种状态。运行状态是进程获得cpu使用权，正在执行代码的状态；等待状态是阻塞状态，比如上述程序运行到recv时，程序会从运行状态变为等待状态，接收到数据后又变回运行状态。操作系统会分时执行各个运行状态的进程，由于速度很快，看上去就像是同时执行多个任务。
  - 一开始，这3个进程都被操作系统的**工作队列**所引用，处于运行状态，会分时执行。
  - 当进程A执行到创建socket的语句时，操作系统会创建一个由文件系统管理的**socket对象**。这个socket对象包含了发送缓冲区、接收缓冲区、等待队列等成员。等待队列是个非常重要的结构，它指向所有需要等待该socket事件的进程。
  - 当程序执行到recv时，操作系统会将进程A从工作队列移动到该socket的**等待队列**中。由于工作队列只剩下了进程B和C，依据进程调度，cpu会轮流执行这两个进程的程序，不会执行进程A的程序。**所以进程A被阻塞，不会往下执行代码，也不会占用cpu资源**。
  - 当socket接收到数据后，操作系统将该socket等待队列上的进程重新放回到工作队列，该进程变成运行状态，继续执行代码。也由于socket的接收缓冲区已经有了数据，recv可以返回接收到的数据。

- 内核接收网络数据全过程

  - 进程在recv阻塞期间，计算机收到了对端传送的数据（步骤①）。数据经由网卡传送到内存（步骤②），然后网卡通过中断信号通知cpu有数据到达，cpu执行中断程序（步骤③）。此处的中断程序主要有两项功能，先将网络数据写入到对应socket的接收缓冲区里面（步骤④），再唤醒进程A（步骤⑤），重新将进程A放入工作队列中。

- 操作系统如何知道网络数据对应于哪个socket？

  - 因为一个socket对应着一个端口号，而网络数据包中包含了ip和端口的信息，内核可以通过端口号找到对应的socket。当然，为了提高处理速度，操作系统会维护端口号到socket的索引结构，以快速读取。

- 如何同时监视多个socket的数据？

  - 服务端需要管理多个客户端连接，而recv只能监视单个socket，这种矛盾下，人们开始寻找监视多个socket的方法。epoll的要义是**高效**的监视多个socket。**从历史发展角度看，必然先出现一种不太高效的方法**，人们再加以改进。只有先理解了不太高效的方法，才能够理解epoll的本质。

  - 如能够预先传入一个socket列表，**如果列表中的socket都没有数据，挂起进程，直到有一个socket收到数据，唤醒进程**。这种方法很直接，也是**select**的设计思想。

  - 为方便理解，我们先复习select的用法。在如下的代码中，先准备一个数组（下面代码中的fds），让fds存放着所有需要监视的socket。然后调用select，如果fds中的所有socket都没有数据，select会阻塞，直到有一个socket接收到数据，select返回，唤醒进程。用户可以遍历fds，通过FD_ISSET判断具体哪个socket收到数据，然后做出处理。

    ```c++
    int s = socket(AF_INET, SOCK_STREAM, 0);  
    bind(s, ...)
    listen(s, ...)
    
    int fds[] =  存放需要监听的socket
    
    while(1){
        int n = select(..., fds, ...)
        for(int i=0; i < fds.count; i++){
            if(FD_ISSET(fds[i], ...)){
                //fds[i]的数据处理
            }
        }
    }
    ```

  - select的实现思路很直接。假如程序同时监视sock1、sock2和sock3三个socket，那么在调用select之后，操作系统把进程A分别加入这三个socket的等待队列中

  - 当任何一个socket收到数据后，中断程序将唤起进程。谓唤起进程，就是将进程从所有的等待队列中移除，加入到工作队列里面。

  - 经由这些步骤，当进程A被唤醒后，它知道至少有一个socket接收了数据。程序只需遍历一遍socket列表，就可以得到就绪的socket。

- 缺点

  - 其一，每次调用select都需要将进程加入到所有监视socket的等待队列，每次唤醒都需要从每个队列中移除。这里涉及了两次遍历，而且每次都要将整个fds列表传递给内核，有一定的开销。正是因为遍历操作开销大，出于效率的考量，才会规定select的最大监视数量，默认只能监视1024个socket。
  - 其二，进程被唤醒后，程序并不知道哪些socket收到数据，还需要遍历一次。
  - 本节只解释了select的一种情形。当程序调用select时，内核会先遍历一遍socket，如果有一个以上的socket接收缓冲区有数据，那么select直接返回，不会阻塞。这也是为什么select的返回值有可能大于1的原因之一。如果没有socket有数据，进程才会阻塞。

- **epoll的设计思路**

  - epoll通过以下一些措施来改进效率。

  - select低效的原因之一是将“维护等待队列”和“阻塞进程”两个步骤合二为一。

    - 如下图所示，每次调用select都需要这两步操作，然而大多数应用场景中，需要监视的socket相对固定，并不需要每次都修改。epoll将这两个操作分开，先用epoll_ctl维护等待队列，再调用epoll_wait阻塞进程。显而易见的，效率就能得到提升。

    - ![img](https://pic2.zhimg.com/80/v2-5ce040484bbe61df5b484730c4cf56cd_720w.jpg)

    - 为方便理解后续的内容，我们先复习下epoll的用法。如下的代码中，先用epoll_create创建一个epoll对象epfd，再通过epoll_ctl将需要监视的socket添加到epfd中，最后调用epoll_wait等待数据。

      ```c++
      int s = socket(AF_INET, SOCK_STREAM, 0);   
      bind(s, ...)
      listen(s, ...)
      
      int epfd = epoll_create(...);
      epoll_ctl(epfd, ...); //将所有需要监听的socket添加到epfd中
      
      while(1){
          int n = epoll_wait(...)
          for(接收到数据的socket){
              //处理
          }
      }
      ```

      功能分离，使得epoll有了优化的可能。

  - **措施二：就绪列表**

    select低效的另一个原因在于程序不知道哪些socket收到数据，只能一个个遍历。如果内核维护一个“就绪列表”，引用收到数据的socket，就能避免遍历。假设计算机共有三个socket，收到数据的sock2和sock3被rdlist（就绪列表）所引用。当进程被唤醒后，只要获取rdlist的内容，就能够知道哪些socket收到数据。

- epoll的原理和流程

  - **创建epoll对象**
    - 当某个进程调用epoll_create方法时，内核会创建一个eventpoll对象（也就是程序中epfd所代表的对象）。eventpoll对象也是文件系统中的一员，和socket一样，它也会有等待队列。
    - 创建一个代表该epoll的eventpoll对象是必须的，因为内核要维护“就绪列表”等数据，“就绪列表”可以作为eventpoll的成员。
  - **维护监视列表**
    - 创建epoll对象后，可以用epoll_ctl添加或删除所要监听的socket。以添加socket为例，如果通过epoll_ctl添加sock1、sock2和sock3的监视，内核会将eventpoll添加到这三个socket的等待队列中。
    - ![img](https://picb.zhimg.com/v2-b49bb08a6a1b7159073b71c4d6591185_b.jpg)
    - 当socket收到数据后，中断程序会操作eventpoll对象，而不是直接操作进程。
  - **接收数据**
    - 当socket收到数据后，中断程序会给eventpoll的“就绪列表”添加socket引用。如下图展示的是sock2和sock3收到数据后，中断程序让rdlist引用这两个socket。
    - ![img](https://pic2.zhimg.com/v2-18b89b221d5db3b5456ab6a0f6dc5784_b.jpg)
    - eventpoll对象相当于是socket和进程之间的中介，socket的数据接收并不直接影响进程，而是通过改变eventpoll的就绪列表来改变进程状态。
    - 当程序执行到epoll_wait时，如果rdlist已经引用了socket，那么epoll_wait直接返回，如果rdlist为空，阻塞进程。
  - **阻塞和唤醒进程**
    - 假设计算机中正在运行进程A和进程B，在某时刻进程A运行到了epoll_wait语句。内核会将进程A放入eventpoll的等待队列中，阻塞进程。
    - ![img](https://pic1.zhimg.com/v2-90632d0dc3ded7f91379b848ab53974c_b.jpg)
    - 当socket接收到数据，中断程序一方面修改rdlist，另一方面唤醒eventpoll等待队列中的进程，进程A再次进入运行状态。也因为rdlist的存在，进程A可以知道哪些socket发生了变化。
    - ![img](https://pic3.zhimg.com/v2-40bd5825e27cf49b7fd9a59dfcbe4d6f_b.jpg)

- eventpoll的数据结构

  - **就绪列表的数据结构**

    就绪列表引用着就绪的socket，所以它应能够快速的插入数据。

    程序可能随时调用epoll_ctl添加监视socket，也可能随时删除。当删除时，若该socket已经存放在就绪列表中，它也应该被移除。

    所以就绪列表应是一种能够快速插入和删除的数据结构。双向链表就是这样一种数据结构，epoll使用双向链表来实现就绪队列（对应上图的rdllist）。

  - **索引结构**

    既然epoll将“维护监视队列”和“进程阻塞”分离，也意味着需要有个数据结构来保存监视的socket。至少要方便的添加和移除，还要便于搜索，以避免重复添加。红黑树是一种自平衡二叉查找树，搜索、插入和删除时间复杂度都是O(log(N))，效率较好。epoll使用了红黑树作为索引结构（对应上图的rbr）。

    > ps：因为操作系统要兼顾多种功能，以及由更多需要保存的数据，rdlist并非直接引用socket，而是通过epitem间接引用，红黑树的节点也是epitem对象。同样，文件系统也并非直接引用着socket。

- select、poll和epoll的区别

  - ![img](https://pic2.zhimg.com/80/v2-14e0536d872474b0851b62572b732e39_720w.jpg)



一、ET模式（边沿触发）的文件描述符(fd)：

​         当epoll_wait检测到fd上有事件发生并将此事件通知应用程序后，应用程序必须立即处理该事件，因为后续的epoll_wait调用将不再向应用程序通知这一事件。

​         epoll_wait只有在客户端第一次发数据是才会返回,以后即使缓冲区里还有数据，也不会返回了。epoll_wait是否返回，是看客户端是否发数据，客户端发数据了就会返回，且只返回一次。

​         eg：客户端发送数据，I/O函数只会提醒一次服务端fd上有数据，以后将不会再提醒

所以要求服务端必须一次把数据读完--->循环读数据 (读完数据后，可能会阻塞)  --->将描述符设置成非阻塞模式

二、LT模式（水平触发）的文件描述符(fd)：

​         当epoll_wait检测到fd上有事件发生并将此事件通知应用程序后，应用程序可以不立即处理该事件，这样，当应用程序下一次调用epoll_wait时，epoll_wait还会再次向应用程序通知此事件，直到此事件被处理。

eg：客户端发送数据，I/O函数会提醒描述符fd有数据---->recv读数据，若一次没有读完，I/O函数会一直提醒服务端fd上有数据，直到recv缓冲区里的数据读完

 

三、可见ET模式在很大程度上降低了同一个epoll事件被重复触发的次数，因此ET模式效率比LT模式高

​         原因：ET模式下事件被触发的次数比LT模式下少很多

注意：每个使用ET模式的文件描述符都应该是非阻塞的。 如果描述符是阻塞的，那么读或写操作将会因没有后续事件而一直处于阻塞状态 ( 饥渴状态 )。







## 零拷贝

- "零拷贝"中的"拷贝"是操作系统在I/O操作中,将数据从一个内存区域复制到另外一个内存区域. 而"零"并不是指0次复制, 更多的是指在用户态和内核态之间的复制是0次.

  - CPU COPY
    - 在"拷贝"发生的时候,往往需要CPU暂停现有的处理逻辑,来协助内存的读写.这种我们称为CPU COPY
  - DMA COPY
    - 当需要与外设进行数据交换时, CPU只需要初始化这个动作便可以继续执行其他指令,剩下的数据传输的动作完全由DMA来完成
    - DMA COPY是可以避免大量的CPU中断的
  - 存在多次拷贝的原因
    - 操作系统为了保护系统不被应用程序有意或无意地破坏,为操作系统设置了用户态和内核态两种状态.用户态想要获取系统资源(例如访问硬盘), 必须通过系统调用进入到内核态, 由内核态获取到系统资源,再切换回用户态返回应用程序.
    - 操作系统在内核态中也增加了一个"内核缓冲区"(kernel buffer). 读取数据时并不是直接把数据读取到应用程序的buffer, 而先读取到kernel buffer, 再由kernel buffer复制到应用程序的buffer. 因此,数据在被应用程序使用之前,可能需要被多次拷贝
  - 从硬盘上读取文件数据, 发送到网络上去.
    - ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190807155340609.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppODI4MTk1NzAyNA==,size_16,color_FFFFFF,t_70)
    - 一次read-send涉及到了四次拷贝
      - 其中涉及到2次cpu中断, 还有4次的上下文切换
      - 很明显,第2次和第3次的的copy只是把数据复制到app buffer又原封不动的复制回来, 为此带来了两次的cpu copy和两次上下文切换, 是完全没有必要的
      - linux的零拷贝技术就是为了优化掉这两次不必要的拷贝
  - sendFile
    - ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190807155602428.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppODI4MTk1NzAyNA==,size_16,color_FFFFFF,t_70)
    - 这个系统调用可以在内核态内把数据从内核缓冲区直接复制到套接字(SOCKET)缓冲区内, 从而可以减少上下文的切换和不必要数据的复制
  - mmap和sendFile
    - 虚拟内存
      - 一个用户虚拟地址和内核虚拟地址可以指向同一个物理内存地址
      - 虚拟内存空间可大于实际可用的物理地址；
    - 利用第一条特性可以把内核空间地址和用户空间的虚拟地址映射到同一个物理地址，这样DMA就可以填充对内核和用户空间进程同时可见的缓冲区了
    - java也利用操作系统的此特性来提升性能
    - MMAP(内存映射文件), 是指将文件映射到进程的地址空间去, 实现硬盘上的物理地址跟进程空间的虚拟地址的一一对应关系.
    - MMAP是另外一个用于实现零拷贝的系统调用.跟sendFile不一样的地方是, 它是利用共享内存空间的方式, 避免app buf和kernel buf之间的数据拷贝(两个buf共享同一段内存)
  - 在 Linux 中零拷贝技术主要有 3 个实现思路：用户态直接 I/O、减少数据拷贝次数以及写时复制技术。
    - 用户态直接 I/O：应用程序可以直接访问硬件存储，操作系统内核只是辅助数据传输。这种方式依旧存在用户空间和内核空间的上下文切换，硬件上的数据直接拷贝至了用户空间，不经过内核空间。因此，直接 I/O 不存在内核空间缓冲区和用户空间缓冲区之间的数据拷贝。
    - 减少数据拷贝次数：在数据传输过程中，避免数据在用户空间缓冲区和系统内核空间缓冲区之间的CPU拷贝，以及数据在系统内核空间内的CPU拷贝，这也是当前主流零拷贝技术的实现思路。
    - 写时复制技术：写时复制指的是当多个进程共享同一块数据时，如果其中一个进程需要对这份数据进行修改，那么将其拷贝到自己的进程地址空间中，如果只是数据读取操作则不需要进行拷贝操作。
  - Java零拷贝
    - ByteBuffer byteBuffer = ByteBuffer.allocate(512);
      - 堆内存：返回 HeapByteBuffer
      - HeapByteBuffer是在jvm的内存范围之内，然后在调io的操作时会将数据区域拷贝一份到os的内存区域
      - 只能拷贝jvm的那一份到os的内存空间，即使jvm那边的数据区域被改变，但是os里边的不会受到影响，等os使用io结束后会对这块区域进行回收，因为这是os的管理范围之内
    - DirectByteBuffer
      - 开辟了一段直接的内存，并不会占用jvm的内存空间
      - DirectByteBuffer使用的是直接的堆外内存，这块内存直接与io设备进行交互

  


