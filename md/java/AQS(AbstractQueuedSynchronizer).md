## 介绍

- AQS是用来构建锁和其他同步组件的基础框架

- 每个类内部都包含一个如下的内部类定义abstract static class Sync extends AbstractQueuedSynchronizer        

  - 每个方法都是一个风格，就是换个名直接调用sync的对应方法
  - 几种同步类提供的功能其实都是委托sync来完成

- AQS中主要维护了state（锁状态的表示）和一个可阻塞的等待队列。

  - ```java
    private transient volatile Node head;
    private transient volatile Node tail;
    private volatile int state;
    ```

## 方法

-  acquire(int arg)-- 获取排他锁

  - ReentrantLock.lock()中调用了这个方法
    - 共性：都属于独占锁的实现，任意一个时刻只有一个线程能够获取到锁。都支持可重入。
    - 区别：a.synchronized属于JVM层面的管程实现，ReentrantLock属于Java语言层面实现的管程。
  - ReentrantLock有synchronized不具备的特性：响应中断、支持超时、支持非阻塞式地获取锁，公平锁（在构造方法中传参true），支持多个等待队列。

- acquire()的实现逻辑

  - tryAcquire(int arg)：//再次尝试获取同步状态。成功，方法直接退出，失败调用addWaier(Node.EXCLUSIVE)。

  - addWaiter(Node.EXCLUSIVE,arg)：将获取同步状态失败的当前线程以指定的模式（独占式/共享式）封装为Node节点后置入同步队列。

  - 同步队列为空或者尾插失败执行：enq(Node node):当前队列为空或者CAS尾插失败调用此方法来初始化队列或者不断尾插。尾插失败：V不等于O，多个线程同时进行尾插。

  - 节点入队后排队获取同步状态：acquireQueued(addWaiter(Node.EXCLUSIVE), arg))：将当前线程以指定模式（独占式、共享式）封装为Node节点后置入同步队列。

  - 节点获取到同步状态的前置条件：当前节点的前驱节点为头节点并且调用tryAcquire获取到了同步状态->前驱头节点出队，当前节点作为头节点。

  - acquiredQueued()获取锁成功条件：当前节点前驱为头节点并且获取同步状态成功。

    acquireQueued():

    - 如果当前节点的前驱节点为头节点并且能够成功获取同步状态，当前线程获得锁成功，调用setHead(Node node)： 将当前节点设置为头节点，删除原有头节点，方法返回。
    - 如果获取锁失败，先不断自旋将前驱节点状态置为SIGNAL，而后调用LockSupport.park()方法将当前线程阻塞。

  - shouldParkAfterFailedAcquire(Node pred, Node node)：

    - 节点在同步队列中获取锁失败后调用shouldParkAfterFailedAcquire(Node prev,Node node)。
    - 不断自旋设置前驱节点状态为SIGNAL，表示当前节点需要阻塞。
    - 如果CAS失败，不断自旋直到将前驱节点状态设置为SIGNAL(-1)为止。

  - parkAndCheckInterrupt()：将当前节点调用LockSupport.park()阻塞。

- 独占锁获取总结

  - 线程获取锁失败，将线程调用addWaiter()，封装成Node，进行入队操作。
    - addWaiter()中方法enq()完成对同步队列的头节点初始化以及CAS尾插失败后的重试处理。
  - 入队之后，排队获取锁的核心方法acquireQueued()，节点排队获取锁是一个自旋过程。
    - 当且仅当当前节点的前驱节点为头节点并且成功获取同步状态时，节点出队并且该节点引用的线程获取到锁。
    - 否则，不满足条件时会不断自旋将前驱节点的状态设置为SIGNAL，而后调用LockSupport.park()将当前线程阻塞。

- doAcquireNanos()--独占模式下在规定时间内获取锁

  - ReentrantLock.tryLock()过程中被调用
  - doAcquireNanos()：这个方法只工作于独占模式，自旋获取资源超时后则返回false；如果有必要挂起，且未超时则挂起
  - 该方法在三种情况下会返回
    - 在超时时间内，当前线程成功获取到锁。
    - 当前线程在超时时间内，被中断。
    - 超时时间结束，仍未获取到锁，线程退出，返回false。

- release(int arg)--释放排他锁 

  - 每一次锁释放后就会唤醒队列中该节点的后继节点所引用的线程，从而进一步可以佐证获得锁的过程是一个FIFO（先进先出）的过程

- acquireShared(int arg)--获取共享锁





## Condition

- 介绍

  - Object的wait和notify/notify是与对象监视器配合完成线程间的等待/通知机制，Condition与Lock配合完成等待/通知机制
  - 前者是java底层级别的，后者是语言级别的，具有更高的可控制性和扩展性
  - Condition能够支持不响应中断，而通过使用Object方式不支持；
  2. Condition能够支持多个等待队列（new 多个Condition对象），而Object方式只能支持一个；
  3. Condition能够支持超时时间的设置，而Object不支持
  3. 等待/通知机制，通过使用condition提供的await和signal/signalAll方法就可以实现这种机制，而这种机制能够解决最经典的问题就是“生产者与消费者问题”

- 参照Object的wait和notify/notifyAll方法，Condition也提供了同样的方法：

  3. await() ，当前线程进入等待状态，如果其他线程调用condition的signal或者signalAll方法并且当前线程获取Lock从await方法返回，如果在等待状态中被中断会抛出被中断异常；
  3. awaitNanos(long nanosTimeout)：当前线程进入等待状态直到被通知，中断或者超时；
  3. await(long time, TimeUnit unit)：同第二种，支持自定义时间单位
  3. awaitUntil(Date deadline) ：当前线程进入等待状态直到被通知，中断或者到了某个时间

- 针对Object的notify/notifyAll方法

  3. signal()：唤醒一个等待在condition上的线程，将该线程从等待队列中转移到同步队列中，如果在同步队列中能够竞争到Lock则可以从等待方法中返回。
  3. signalAll()：与1的区别在于能够唤醒所有等待在condition上的线程

- await实现原理

  - await()：当前线程处于阻塞状态，直到调用signal()或中断才能被唤醒。

    1）将当前线程封装成node且等待状态为CONDITION。
    2）释放当前线程持有的所有锁，让下一个线程能获取锁。
    3）加入到等待队列后，则阻塞当前线程，等待被唤醒。
    4）如果是因signal被唤醒，则节点会从等待队列转移到同步队列；如果是因中断被唤醒，则记录中断状态。两种情况都会跳出循环。
    5）若是因signal被唤醒，就自旋获取锁；否则处理中断异常。

- signal/signalAll实现原理

  - doSignal方法：将等待队列的头节点转移到同步队列