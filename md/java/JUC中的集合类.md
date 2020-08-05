## List和Set

- JUC集合包中的List和Set实现类包括: CopyOnWriteArrayList, CopyOnWriteArraySet和ConcurrentSkipListSet

- CopyOnWriteArrayList相当于线程安全的ArrayList，它实现了List接口

  - “添加/修改/删除”数据时，都会新建一个数组，并将更新后的数据拷贝到新建的数组中，最后再将该数组赋值给“volatile数组”。这就是它叫做CopyOnWriteArrayList的原因
  - 如果被删除的是最后一个元素，则直接通过Arrays.copyOf()进行处理，而不需要新建数组。否则，新建数组，然后将”volatile数组中被删除元素之外的其它元素“拷贝到新数组中；最后，将新数组赋值给”volatile数组“。
  - 线程安全通过volatile和互斥锁来实现
    - 取volatile数组时，总能看到其它线程对该volatile变量最后的写入；就这样，通过volatile提供了“读取到的数据总是最新的”这个机制的保证。
    - 互斥锁来保护数据，添加/修改/删除”数据时，会先“获取互斥锁”，再修改完毕之后，先将数据更新到“volatile数组”中，然后再“释放互斥锁”
  - 遍历
    - CopyOnWriteArrayList返回迭代器不会抛出ConcurrentModificationException异常，即它不是fail-fast机制的
    - ArrayList的Iterator实现类中调用next()、 remove()时，会“调用checkForComodification()比较‘expectedModCount’和‘modCount’的大小”；但是，CopyOnWriteArrayList的Iterator实现类中，没有所谓的checkForComodification()，更不会抛出ConcurrentModificationException异常！
    - ArrayList的Iterator是在父类AbstractList.java中实现的，和ArrayList继承于AbstractList不同，CopyOnWriteArrayList没有继承于AbstractList，它仅仅只是实现了List接口
    - 无论是add()、remove()，还是clear()，只要涉及到修改集合中的元素个数时，都会改变modCount的值

- CopyOnWriteArraySet相当于线程安全的HashSet，它继承于AbstractSet类。

  - CopyOnWriteArraySet内部包含一个CopyOnWriteArrayList对象，它是通过CopyOnWriteArrayList实现的
  - HashSet是通过“散列表(HashMap)”实现的，而CopyOnWriteArraySet则是通过“动态数组(CopyOnWriteArrayList)”实现

  

  

  

## Map

- JUC集合包中Map的实现类包括: ConcurrentHashMap和ConcurrentSkipListMap
- ConcurrentHashMap(jdk1.7)是线程安全的哈希表(相当于线程安全的HashMap)，它继承于AbstractMap类，并且实现ConcurrentMap接口。ConcurrentHashMap是通过“锁分段”来实现的，它支持并发。
  - ConcurrentHashMap将哈希表分成许多片段(Segment)，每一个片段除了保存哈希表之外，本质上也是一个“可重入的互斥锁”(ReentrantLock)。多线程对同一个片段的访问，是互斥的；但是，对于不同片段的访问，却是可以同步进行的
    - Segment类继承于ReentrantLock类，所以Segment本质上是一个可重入的互斥锁
    - Segment包含了“HashEntry数组”，而“HashEntry数组”中的每一个HashEntry元素是一个单向链表
    - ConcurrentHashMap是链式哈希表，它是通过“拉链法”来解决哈希冲突的
  - 获取锁失败的话，则通过scanAndLockForPut()“自旋”获取锁，并返回”要插入的key-value“对应的”HashEntry链表。
    - while循环中，它会遍历”HashEntry链表(e)“，查找”要插入的key-value键值对“在”该HashEntry链表上对应的节点
    - 若找到的话，则不断的自旋，自旋期间，若通过tryLock()获取锁成功则返回；否则，在自旋MAX_SCAN_RETRIES次数之后，强制获取锁并退出
- ConcurrentHashMap(jdk1.8)
  - 介绍
    - 摒弃了Segment（锁段）的概念，而是启用了一种全新的方式实现,利用CAS算法
    - 底层依然由“数组”+链表+红黑树的方式思想
    - 锁更加细化，而不是像HashTable一样为几乎每个方法都添加了synchronized锁
  - ConcurrentHashMap定义了三个原子操作，用于对指定位置的节点进行操作。正是这些原子操作保证了ConcurrentHashMap的线程安全。
    - 利用volatile方法获得在i位置上的Node节点
    - 利用CAS算法设置i位置上的Node节点。之所以能实现并发是因为他指定了原来这个节点的值是多少 
      - 当前线程中的值并不是最新的值，这种修改可能会覆盖掉其他线程的修改结果
    - 利用volatile方法设置节点位置的值  
  - sizeCtl控制标识符
    - 默认为 0，用来控制 table 的初始化和扩容操作。-1 代表 table 正在初始化；-N 表示有 N-1 个线程正在进行扩容操作。
    - 初始化方法中的并发问题是通过对 sizeCtl 进行一个 CAS 操作来控制的。 
      - CAS 一下，将 sizeCtl 设置为 -1，代表抢到了锁
      - 可以看出ConcurrentHashMap的初始化只能由一个线程完成
      - 在任何一个构造方法中，都没有对存储Map元素Node的table变量进行初始化。而是在第一次put操作的时候在进行初始化。
  - put 方法
    - 判断存储的 key、value 是否为空，若为空，则抛出异常，否则，进入步骤 2。
    - 计算 key 的 hash 值，随后进入自旋，该自旋可以确保成功插入数据，若 table 表为空或者长度为 0，则初始化 table 表，否则，进入步骤 3。
    - 根据 key 的 hash 值取出 table 表中的结点元素，若取出的结点为空（该桶为空），则使用 CAS 将 key、value、hash 值生成的结点放入桶中。否则，进入步骤 4。
    - 若该结点的的 hash 值为 MOVED（-1），说明该链表正在进行transfer操作，返回扩容完成后的table。否则，进入步骤 5。
      - 只要看到当前主节点的hash值为-1时就会进入这里面的方法，我们看到它里面是helpTransfer()方法
      - 它对来帮忙的每一个线程都分配了一块区域，每个线程只能搬运自己所属区域内的元素，这样就互不干扰了。这些线程在帮助分配完元素之后，才会去做自己本来的操作
      - tryPresize方法
        - 当需要扩容的时候，调用的时候tryPresize方法
               * 扩容表为指可以容纳指定个数的大小（总是2的N次方）
                    *   *  假设原来的数组长度为16，则在调用tryPresize的时候，size参数的值为16<<1(32)，此时sizeCtl的值为12
                  * 计算出来c的值为64,则要扩容到sizeCtl≥为止
           
                  * 第一次扩容之后 数组长：32 sizeCtl：24
                  * 第二次扩容之后 数组长：64 sizeCtl：48
          
            	   * 第三次扩容之后 数组长：128 sizeCtl：94 --> 这个时候才会退出扩容
           - 初始化的时候，设置sizeCtrl为-1，初始化完成之后把sizeCtrl设置为数组长度的3/4
        - 在tryPresize方法中，并没有加锁，允许多个线程进入，如果数组正在扩张，则当前线程也去帮助扩容。
    - 对桶中的第一个结点（即 table 表中的结点）进行加锁，对该桶进行遍历，桶中的结点的 hash 值与 key 值与给定的 hash 值和 key 值相等，则根据标识选择是否进行更新操作（用给定的 value 值替换该结点的 value 值），若遍历完桶仍没有找到 hash 值与 key 值和指定的 hash 值与 key 值相等的结点，则直接新生一个结点并赋值为之前最后一个结点的下一个结点。进入步骤 6。
    - 若 binCount 值达到红黑树转化的阈值，则将桶中的结构转化为红黑树存储，最后，增加 binCount 的值
  - get 方法
    - 计算 hash 值。
    - 根据 hash 值找到数组对应位置: (n – 1) & h。
    - 根据该位置处结点性质进行相应查找。
      -  如果该位置为 null，那么直接返回 null。
      -  如果该位置处的结点刚好就是需要的，返回该结点的值即可。
      -  如果该位置结点的 hash 值小于 0，说明正在扩容，或者是红黑树。
      -  如果以上 3 条都不满足，那就是链表，进行遍历比对即可。
    - 读操作，从源码中可以看出来，在get操作中，根本没有使用同步机制，也没有使用unsafe方法，所以读操作是支持并发操作的。
  - 数组扩容的主要方法就是transfer方法
    - 步骤
      - ForwardingNode 一个特殊的 Node 结点，hash 值为 -1，其中存储 nextTable 的引用。有 table 发生扩容的时候，ForwardingNode 发挥作用，作为一个占位符放在 table 中表示当前结点为 null 或者已经被移动
      - 在扩容的时候每个线程都有处理的步长，最少为16，在这个步长范围内的数组节点只有自己一个线程来处理
      - 当对桶进行转移的时候，会将链表拆成两份，规则是根据节点的 hash 值取于 length，如果结果是 0，放在低位，否则放在高位。
    - tryPresize是在treeIfybin和putAll方法中调用，treeIfybin主要是在put添加元素完之后，判断该数组节点相关元素是不是已经超过8个的时候，如果超过则会调用这个方法来扩容数组或者把链表转为树。
    - helpTransfer是在当一个线程要对table中元素进行操作的时候，如果检测到节点的HASH值为MOVED的时候，就会调用helpTransfer方法，在helpTransfer中再调用transfer方法来帮助完成数组的扩容
    - addCount是在当对数组进行操作，使得数组中存储的元素个数发生了变化的时候会调用的方法
  - 链表转为红黑树的过程 
    - treeifyBin方法
    - 添加元素的时候，在同一个位置的个数又达到了8个以上，如果数组的长度还小于64的时候，则会扩容数组。如果数组的长度大于等于64了的话，则会将该节点的链表转换成树。
      - 首先将Node的链表转化为一个TreeNode的链表，然后将TreeNode链表的头结点来构造一个TreeBin。　　
      - 在TreeBin容器中，将链表转化为红黑树
      - 然后在每次添加完一个节点之后，都会调用balanceInsertion方法来维持这是一个红黑树的属性和平衡性。红黑树所有操作的复杂度都是O(logn)，所以当元素量比较大的时候，效率也很高。
  - ConcurrentHashmap 不支持 key 或者 value 为 null 的原因
    - ConcurrentHashmap 和 Hashtable 都是支持并发的，当通过 get(k) 获取对应的 value 时，如果获取到的是 null 时，无法判断是 put(k,v) 的时候 value 为 null，还是这个 key 从来没有做过映射。 
    - HashMap 是非并发的，可以通过 contains(key) 来做这个判断。
    - 支持并发的 Map 在调用 m.contains(key) 和 m.get(key) 时，m 可能已经发生了更改。
      因此 ConcurrentHashmap 和 Hashtable 都不支持 key 或者 value 为 null。
  - 在扩容的时候，可以不可以对数组进行读写操作呢？
    - 事实上是可以的。当在进行数组扩容的时候，如果当前节点还没有被处理（也就是说还没有设置为fwd节点），那就可以进行设置操作。　　
    - 如果该节点已经被处理了，则当前线程也会加入到扩容的操作中去。
  - 多个线程又是如何同步处理的呢
    - 在ConcurrentHashMap中，同步处理主要是通过Synchronized和unsafe两种方式来完成的。
    - 在取得sizeCtl、某个位置的Node的时候，使用的都是unsafe的方法，来达到并发安全的目的
    - 当需要在某个位置设置节点的时候，则会通过Synchronized的同步机制来锁定该位置的节点。
    - 当把某个位置的节点复制到扩张后的table的时候，也通过Synchronized的同步机制来保证现程安全
    - 在数组扩容的时候，则通过处理的步长和fwd节点来达到并发安全的目的，通过设置hash值为MOVED
-  ConcurrentSkipListMap
  - 是线程安全的有序的哈希表(相当于线程安全的TreeMap);它继承于AbstractMap类，并且实现ConcurrentNavigableMap接口。ConcurrentSkipListMap是通过“跳表”来实现的，它支持并发
  - Node，Index， HeadIndex 这三个类。Node表示最底层的链表节点，Index类表示基于Node类的索引层，而HeadIndex则是用来维护索引的层次。
    - index类作为索引节点，共包含了三个属性，一个是Node节点的引用，一个是指向下一层的索引节点的引用，一个是指向右侧索引节点的引用
    - HeadIndex继承自Index，扩展了一个level属性，表示当前索引节点Index的层级
  - Put方法
    - 根据key从跳跃表的左上方开始，向右或者向下查找到需要插入位置的前驱Node节点，查找过程中会删除一些已经标记为删除状态的节点；
    - 然后根据随机值来判断是否生成索引层以及生成索引层的层次；
  - remove方法
    - 查询到要删除的节点，通过CAS操作把value设置为null（这样其他线程可以感知到这个节点状态，协助完成删除工作），然后在该节点后面添加一个marker节点作为删除标志位，若添加成功，则将该结点的前驱的后继设置为该结点之前的后继（也就是删除该节点操作），这样可以避免丢失数据；
    - 这里可能需要说明下，因为ConcurrentSkipListMap是支持并发操作的，因此在删除的时候可能有其他线程在该位置上进行插入，这样有可能导致数据的丢失。在ConcurrentSkipListMap中，会在要删除的节点后面添加一个特殊的节点进行标记，然后再进行整体的删除，如果不进行标记，那么如果正在删除的节点，可能其它线程正在此节点后面添加数据，造成数据丢失。
    
    

## Queue

- 阻塞队列接口BlockingQueue继承自Queue接口
  - 插入方法：
    - add(E e) : 添加成功返回true，失败抛IllegalStateException异常
    - offer(E e) : 成功返回 true，如果此队列已满，则返回 false。
    - put(E e) :将元素插入此队列的尾部，如果该队列已满，则一直阻塞
  - 删除方法:
    - remove(Object o) :移除指定元素,成功返回true，失败返回false
    - poll() : 获取并移除此队列的头元素，若队列为空，则返回 null
    - take()：获取并移除此队列头元素，若没有元素则一直阻塞。
  - 检查方法
    - element() ：获取但不移除此队列的头元素，没有元素则抛异常
    - peek() :获取但不移除此队列的头；若队列为空，则返回 null。
- ArrayBlockingQueue是数组实现的线程安全的有界的阻塞队列。
  - 概述
    - 支持公平锁和非公平锁（ArrayBlockingQueue内部的阻塞队列是通过重入锁ReenterLock和Condition条件队列实现的）
    - 通过这种方式便实现生产者消费者模式。比直接使用等待唤醒机制或者Condition条件队列来得更加简单
    - 通过一个ReentrantLock来同时控制添加线程与移除线程的并发访问，与LinkedBlockingQueue区别很大
  - 添加
    - add方法、offer方法底层调用enqueue(E x)方法
      - putIndex索引大小等于数组长度时，需要将putIndex重新设置为0
        - 这是因为当前队列执行元素获取时总是从队列头部获取，而添加元素是从队列尾部添加。所以当队列索引（从0开始）与数组长度相等时，需要从数组头部开始添加。
      - 添加完数据后，说明数组中有数据了，所以可以唤醒 notEmpty 条件对象等待队列(链表)中第一个可用线程去 take 数据
    - put方法（阻塞添加的方法）
      - 当队列已满的时候，线程会挂起，然后将该线程加入到 notFull 条件对象的等待队列(链表)中
      - 如果队列没有满，那么就直接调用enqueue(e)方法
  - 移除
    - poll方法 底层调用dequeue()方法
      - 提取 takeIndex 位置上的元素， 然后 takeIndex 前进一位，
       - 同时唤醒 notFull 等待队列(链表)中的第一个可用线程去 put 数据。
       - 这些操作都是在当前线程获取到锁的前提下进行的，
       - 同时也说明了 dequeue 方法线程安全的。
     - take()方法，是一个阻塞方法，直接获取队列头元素并删除。
        - 队列中有元素，直接提取，没有元素则线程阻塞(可中断的阻塞)，将该线程加入到 notEmpty 条件对象的等待队列中；等有新的 put 线程添加了数据，分析发现，会在 put 操作中唤醒 notEmpty 条件对象的等待队列中的 take 线程，去执行 take 操作
 - LinkedBlockingQueue是单向链表实现的有界(指定大小)阻塞队列，该队列按 FIFO（先进先出）排序元素。
    - LinkedBlockingQueue内部分别使用了takeLock 和 putLock 对并发进行控制，也就是说，添加和删除操作并不是互斥操作，可以同时进行，这样也就可以大大提高吞吐量
    - 添加
       - add方法和offer方法
          - 判断队列是否满，满了就直接释放锁，没满就将节点封装成Node入队，然后再次判断队列添加完成后是否已满，不满就继续唤醒等到在条件对象notFull上的添加线程
             - 在ArrayBlockingQueue内部完成添加操作后，会直接唤醒消费线程对元素进行获取，这是因为ArrayBlockingQueue只用了一个ReenterLock同时对添加线程和消费线程进行控制，这样如果在添加完成后再次唤醒添加线程的话，消费线程可能永远无法执行
          - 判断是否需要唤醒等到在notEmpty条件对象上的消费线程
             - 为什么要判断`if (c == 0)`时才去唤醒消费线程呢，这是因为消费线程一旦被唤醒是一直在消费的
             - 所以c值是一直在变化的，c值是添加元素前队列的大小
             - 只可能是0或`c>0`，如果是`c=0`，那么说明之前消费线程已停止
             - 如果`c>0`那么消费线程就不会被唤醒，只能等待下一个消费操作（poll、take、remove）的调用
             - c == 0说明在执行释放锁的时候队列里面至少有一个元素
    - 移除
       - remove方法
          - 同时对putLock和takeLock加锁
          - remove方法删除的数据的位置不确定，为了避免造成并发安全问题，所以需要对2个锁同时加锁
       - poll方法
          - 如果队列没有数据就返回null，如果队列有数据，那么就取出来
          - 如果队列还有数据那么唤醒等待在条件对象notEmpty上的消费线程
          - 尝试唤醒条件对象notFull上等待队列中的添加线程(判断if (c == capacity))
       - take方法是一个可阻塞可中断的移除方法
          - 获取当前队列头部元素并从队列里面移除，如果队列为空则阻塞当前线程知道队列不为空，然后返回元素，如果在阻塞的时候被其他线程设置了中断标志，则被阻塞线程会抛出InterruptedException 异常而返回。
          - 如果队列还有数据那么唤醒等待在条件对象notEmpty上的消费线程
          - 尝试唤醒条件对象notFull上等待队列中的添加线程(判断if (c == capacity))
 - LinkedBlockingQueue和ArrayBlockingQueue迥异
    - 队列大小有所不同，ArrayBlockingQueue是有界的初始化必须指定大小，而LinkedBlockingQueue可以是有界的也可以是无界的(Integer.MAX_VALUE)，对于后者而言，当添加速度大于移除速度时，在无界的情况下，可能会造成内存溢出等问题。
    - 数据存储容器不同，ArrayBlockingQueue采用的是数组作为数据存储容器，而LinkedBlockingQueue采用的则是以Node节点作为连接对象的链表。 
    - 由于ArrayBlockingQueue采用的是数组的存储容器，因此在插入或删除元素时不会产生或销毁任何额外的对象实例，而LinkedBlockingQueue则会生成一个额外的Node对象。这可能在长时间内需要高效并发地处理大批量数据的时，对于GC可能存在较大影响。 
    - 两者的实现队列添加或移除的锁不一样，ArrayBlockingQueue实现的队列中的锁是没有分离的，即添加操作和移除操作采用的同一个ReenterLock锁，而LinkedBlockingQueue实现的队列中的锁是分离的，其添加采用的是putLock，移除采用的则是takeLock，这样能大大提高队列的吞吐量，也意味着在高并发的情况下生产者和消费者可以并行地操作队列中的数据，以此来提高整个队列的并发性能。