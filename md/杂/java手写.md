# [纯手写实现HashMap](https://www.cnblogs.com/chenfei-java/p/10674341.html)



1.hashmap的实现

　　① 初始化

　　　　1）定义一个Node<K, V>的数组来存放元素，但不立即初始化，在使用的时候再加载

　　　　2）定义数组初始大小为16

　　　　3）定义负载因子，默认为0.75,

　　　　4）定义size用来记录容器存放的元素数量

　　② put的实现思路

　　　　1)  判断容器是否为空，为空则初始化。

　　　　2）判断容器的size是否大于阀值，是的话就扩容为以前长度的两倍，并重新计算其中元素的存放位置，进行重新存放

　　　　3）计算出key的index角标位置

　　　　4）判断计算出的index位置是否存在元素，存在的话则遍历链表，判断key是否存在，存在则更新，不存在则增加

　　③ get的实现思路

　　　　1）通过key计算出它所在的index

　　　　2）遍历index位置处的链表，并获取value返回。



```java
package com.test;

/**
 * 自定义hashMap
 * @author cf
 *
 * @param <K>
 * @param <V>
 */
public class MyHashMap<K, V> implements MyMap<K, V>{
    //1.定义一个容器用来存放元素， 但不立即初始化，使用懒加载方式
    Node<K, V>[] table = null;
    
    //2.定义容器的默认大小
    static int DEFAULT_INITIAL_CAPACITY = 16;
    
    //3.HashMap默认负载因子，负载因子越小，hash冲突机率越低，综合结论得出0.75最为合适
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    
    //4.记录当前容器实际大小
    static int size;
    
    @SuppressWarnings("unchecked")
    @Override
    public V put(K k, V v) {
        //1.判断容器是否为空为空则初始化。
        if (table == null) {
            table = new Node[DEFAULT_INITIAL_CAPACITY];
        }
        
        //如果size大于阈值则进行扩容
        if (size > DEFAULT_INITIAL_CAPACITY * DEFAULT_LOAD_FACTOR) {
            resize();
        }
        
        //2.计算出index角标
        int index = getIndex(k, DEFAULT_INITIAL_CAPACITY);
        
        //3.将k-v键值对放进相对应的角标，如果计算出角标相同则以链表的形势存放
        Node<K, V> node = table[index];
        if (node == null) {
            table[index] = new Node<>(k, v, null);
            size ++;
            return table[index].getValue();
        } else {
            Node<K, V> newNode = node;
            
            //循环遍历每个节点看看是否存在相同的key
            while (newNode != null) {
                //这里要用equals 和 == 因为key有可能是基本数据类型，也有可能是引用类型
                if (k.equals(newNode.getKey()) || k == newNode.getKey()) {
                    newNode.setValue(v);
                    size ++;
                    return v;
                } 
                newNode = node.getNextNode();
            }
            table[index] = new Node<K, V>(k, v, table[index]);
            size ++;
            
            return table[index].getValue();
        }
        
    }
    
    /**
     * 获取index
     * @param key
     * @param length
     * @return
     */
    public int getIndex(K key, int length) {
        int hashCode = key.hashCode();
        int index = hashCode % length;
        return index;
    }

    /**
     * 获取key
     */
    @Override
    public V get(K k) {
        int index = getIndex(k, DEFAULT_INITIAL_CAPACITY);
        Node<K, V> node = table[index];
        if (k.equals(node.getKey()) || k == node.getKey()) {
            return node.getValue();
        } else {
            Node<K, V> nextNode = node.getNextNode();
            while(nextNode != null) {
                if (k.equals(nextNode.getKey()) || k == nextNode.getKey()) {
                    return nextNode.getValue();
                }
            }
        }
        return null;
    }
    
    /**
     * 对size进行扩容
     */
    @SuppressWarnings("unchecked")
    public void resize() {
        //1.创建新的table长度扩展为以前的两倍
        int newLength = DEFAULT_INITIAL_CAPACITY * 2;
        Node<K, V>[] newtable = new Node[newLength];
        //2.将以前table中的取出，并重新计算index存入
        for (int i = 0; i < table.length; i++) {
            Node<K, V> oldtable = table[i];
            while (oldtable != null) {
                //将table[i]的位置赋值为空,
                table[i] = null;
                
                //方法1：重新计算index，然后按照put时候的方法进行放值，此种方法会不停的new 对象会造成效率比较低
                /*K key = oldtable.getKey();
                int index = getIndex(key, newLength);
                newtable[index] = new Node<K, V>(key, oldtable.getValue(), newtable[index]);
                oldtable = oldtable.getNextNode();*/
                
                //方法2：
                //计算新的index值
                K key = oldtable.getKey();
                int index = getIndex(key, newLength);
                
                //将以前的nextnode保存下来
                Node<K, V> nextNode = oldtable.getNextNode();
                
                //将newtable的值赋值在oldtable的nextnode上，如果以前是空，则nextnode也是空
                oldtable.setNextNode(newtable[index]);
                newtable[i] = oldtable;
                
                //将以前的nextcode赋值给oldtable以便继续遍历
                oldtable = nextNode;
            }
                
        }
        
        //3.将新的table赋值回老的table
        table = newtable;
        DEFAULT_INITIAL_CAPACITY = newLength;
        newtable = null;
        
    }

    @Override
    public int size() {
        return size;
    }

    @SuppressWarnings("hiding")
    class Node<K, V> implements Entry<K, V> {
        private K key;
        private V value;
        private Node<K, V> nextNode; //下一节点
        
        public Node(K key, V value, Node<K, V> nextNode) {
            super();
            this.key = key;
            this.value = value;
            this.nextNode = nextNode;
        }

        @Override
        public K getKey() {
            return this.key;
        }

        @Override
        public V getValue() {
            return this.value;
        }

        @Override
        public void setValue(V value) {
            this.value = value;
        }

        public Node<K, V> getNextNode() {
            return nextNode;
        }

        public void setNextNode(Node<K, V> nextNode) {
            this.nextNode = nextNode;
        }

        public void setKey(K key) {
            this.key = key;
        }
        
        //判断是否还有下一个节点
        /*private boolean hasNext() {
            return true;
        }*/
        
    }
    
    
    
    // 测试方法.打印所有的链表元素
    public void print() {
        for (int i = 0; i < table.length; i++) {
            Node<K, V> node = table[i];
            System.out.print("下标位置[" + i + "]");
            while (node != null) {
                System.out.print("[ key:" + node.getKey() + ",value:" + node.getValue() + "]");
                node = node.nextNode;
            }
            System.out.println();
        }

    }
    
}
```



2.测试代码



```java
package com.test;

public class TestMap {
    public static void main(String[] args) {
        MyHashMap<String, String> extHashMap = new MyHashMap<String, String>();
        extHashMap.put("1号", "1号");// 0
        extHashMap.put("2号", "1号");// 1
        extHashMap.put("3号", "1号");// 2
        extHashMap.put("4号", "1号");// 3
        extHashMap.put("6号", "1号");// 4
        extHashMap.put("7号", "1号");
        extHashMap.put("14号", "1号");

        extHashMap.put("22号", "1号");
        extHashMap.put("26号", "1号");
        extHashMap.put("27号", "1号");
        extHashMap.put("28号", "1号");
        extHashMap.put("66号", "66");
        extHashMap.put("30号", "1号");
        System.out.println("扩容前数据....");
        extHashMap.print();
        System.out.println("扩容后数据....");
        extHashMap.put("31号", "1号");
        extHashMap.put("66号", "123466666");
        extHashMap.print();
        // 修改3号之后
        System.out.println(extHashMap.get("66号"));
        
    }
}
```

 



# 手写lru



LRU（Least recently used，最近最少使用）算法根据数据的历史访问记录来进行淘汰数据，其核心思想是“**如果数据最近被访问过，那么将来被访问的几率也更高**”。

------

## 什么是LRU

距离现在最早使用的会被我们替换掉。不够形象的话我们看下面的例子。

| 插入  | 1    | 2    | 3    | 4    | 2    | 3    | 1    |
| ----- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 位置1 | 1    | 1    | 1    | 2    | 3    | 4    | 2    |
| 位置2 | null | 2    | 2    | 3    | 4    | 2    | 3    |
| 位置3 | null | null | 3    | 4    | 2    | 3    | 1    |

…
位置1始终是最早进来的元素，是淘汰位置。新进来的元素如果是新元素直接放在位置3，然后将位置1弹出。如果是已有元素则将其放在位置3并删除之前位置上的已有元素，保持其他元素相对位置不变。

这里的例子就是一个size=3的缓存淘汰实现。

## 利用LinkedHashMap实现的简单LRU

对于

> java.util.LinkedHashMap

我们的认识仅仅只是停留在该map可以按照插入的顺序保存，那是不够的。
linkedHashMap还可以实现按照访问顺序保存元素。
先看看如何利用它实现LRU的吧

```java
public class UseLinkedHashMapCache<K,V> extends LinkedHashMap<K,V>{
    private int cacheSize;
    public UseLinkedHashMapCache(int cacheSize){
    //构造函数一定要放在第一行
     super(16,0.75f,true);    //那个f如果不加  就是double类型，然后该构造没有该类型的入参。 然后最为关键的就是那个入参 true
     this.cacheSize = cacheSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest){   //重写LinkedHashMap原方法
         return size()>cacheSize;  //临界条件不能有等于，否则会让缓存尺寸小1
    }   
}
123456789101112131415
```

关键点:

- 继承了LinkedHashMap并使用

```java
 public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }123456
```

构造函数

- 重写了

```java
 protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }123
```

看看如何使用

```java
public static void main(String[]args){
        UseLinkedHashMapCache<Integer,String> cache = new UseLinkedHashMapCache<Integer,String>(4);
        cache.put(1, "one");
        cache.put(2, "two");
        cache.put(3, "three");
        cache.put(4, "four");
        cache.put(2, "two");
        cache.put(3, "three");

        Iterator<Map.Entry<Integer,String>> it = cache.entrySet().iterator();
        while(it.hasNext()){
            Map.Entry<Integer, String> entry = it.next();
            Integer key = entry.getKey();
            System.out.print("Key:\t"+key);
            String Value = entry.getValue();  //这个无需打印...
            System.out.println();
        }
    }

```

结果是:

```console
Key:    1
Key:    4
Key:    2
Key:    3
12345
```

与我们表格中的结果一致。

## 手写LRU（利用数组）

```java
/**
 * 用数组写了一个
 * 
 * 有个疑问， 比如当缓存大小为5  这时候1、2、3、4、4  请问最后一个4是应该插入还是不处理呢？ 
 * 
 * 我个人觉得如果这里理解为缓存的key ，那么就应该是不插入  结果应该还是1、2、3、4、null 
 * */

public class HandMakeCache {
    //添加次数 计数器
    static int count =0;
    //数组元素 计数器
    static int size=0;
    //最大长度
    int maxSize;
    //对象数组
    int [] listArray;  //为了简略比较

    //顺序表的初始化方法
    public HandMakeCache(int maxSize)
    {
        listArray = new int [maxSize];
        this.maxSize = maxSize;
    }

    public int getSize(){
        return size;
    }

    public void insert(int obj) throws Exception {
        // 插入过程不应该指定下标，对于用户来讲这应该是透明的，只需要暴露插入的顺序
        boolean exist = false; // 每次insert校验一下是否存在
        int location = 0; // 对于已有元素，记录其已存在的位置
        for (int i = 0; i < maxSize; i++) {
            if (obj == listArray[i]) {
                exist = true;
                location = i; // 记录已存在的位置
            }
        } // 遍历看是否已有，每次插入都要遍历，感觉性能很差
        if (size < this.maxSize) { // 当插入次数小于缓存大小的时候随意插入
            if (exist) {
                if (location == 0) {
                    moveArrayElements(listArray,0,size-2);
                } else if (location < size - 1) { // 已存在元素不在最新的位置
                    moveArrayElements(listArray,location,size-2);
                }
                listArray[size - 1] = obj; // 由于已存在
            } else {
                listArray[size] = obj;
                size++; // 数组未满时才计数
            }
        } else { // 此时缓存为满，这时候要保留最末端元素先
            if (!exist || obj == listArray[0]) { // 新元素添加进来，和最远元素添加进来效果一样
                moveArrayElements(listArray,0,maxSize-2);
            } else if (obj != listArray[maxSize - 1]) {
                moveArrayElements(listArray,location,maxSize-2);
            } // 如果添加的是上次添加的元素，则不管了。。
            listArray[maxSize - 1] = obj;
        }
        count++; // 计数
    }

    public Object get(int index) throws Exception {
        return listArray[index];
    }

    /**
     * 平移数组的方法，start是要移动至的头位置，end为最后被移动的位置。
     * */
    public void moveArrayElements(int [] arr, int start, int end){
        for(int i=start;i<=end;i++){
            arr[i] = arr[i+1];
        }
    }


    public static void main(String[] args) {
        int cacheSize = 5;
        HandMakeCache list = new HandMakeCache(cacheSize);
        try
        {
            list.insert(1);
            list.insert(2);
            list.insert(3);
            list.insert(1);
            list.insert(3);
            list.insert(4);
            list.insert(4);
            list.insert(5);
//          list.insert(3);

            for(int i=0;i<cacheSize;i++)
            {
                System.out.println(list.get(i));
            }
            System.out.println("成功插入"+count+"次元素.");

        }
        catch(Exception ex)
        {
            ex.printStackTrace();
        }

    }
}

```

非常重要的一点~ 写LRU之前你一定要知道LRU的正确的含义。。
这里分为几种情况吧..
1. 当数组未满的情况下，随便插
2. 数组满了之后，插入介于头和尾的元素，需要记录其之前存在的下标，然后将大于该下标的元素整体前移。
3. 数组满了之后，插入最新的元素等于什么操作也没有。保持原样
3. 数组满了之后，插入一个不存在的元素 等同于 插入数组最开始的元素。
比如 1、2、3、4 之后插入5 和 1、2、3、4 之后插入1 结果分别为 2、3、4、5和 2、3、4、1。

**缺点:**
如果利用数组来存储的话，当我们缓存的大小非常大的时候。比如10W，那么假设我们需要淘汰最远的元素，就需要将99999个元素整体往前移一位，这样还仅仅只是替换一次。大量这样的操作是非常低效的，所以我们还是考虑用链表来实现↓。



## 手写LRU（LinkedList）

```java
 public class LruLinkedList<T> extends LinkedList<T> {
        /**
         * lru算法
         * 新增的数据放在头部，添加的时候超过了内存大小，就删除尾部
         * 缓存命中，即缓存数据被访问时移至头部，
         */
        public static final int defaultSize = 5;
        public int lruSize;

        public LruLinkedList() {
            this(defaultSize);
        }
        public LruLinkedList(int size) {
            this.lruSize = size;
        }
        //增
        public void lruPut(T data) {
            if (size >= lruSize) {
                removeLast();
                put(data);
            } else {
                put(data);
            }
        }
        //删
        public void lruRemove() {
            removeLast();
        }
        //改
        public void lruSet(int index, T data) {
            testingIndex(index);
            Node head = list;
            Node cur = list;
            for (int i = 0; i < index; i++) {
                head = cur;
                cur = cur.next;
            }
            head.next = cur.next;
            cur.next=list;
            cur.data=data;
            list=cur;
        }
        //查
        public void lruGet(int index) {
            testingIndex(index);
            Node head = list;
            Node cur = list;
            for (int i = 0; i < index; i++) {
                head = cur;
                cur = cur.next;
            }
            head.next = cur.next;
            cur.next=list;
            list=cur;
        }
    }

```



# 生产者消费者

作者：flyingcr

链接：https://zhuanlan.zhihu.com/p/73442055

来源：知乎

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

## **什么是生产者-消费者模式**

比如有两个进程A和B，它们共享一个**固定大小的缓冲区**，A进程产生数据放入缓冲区，B进程从缓冲区中取出数据进行计算，那么这里其实就是一个生产者和消费者的模式，A相当于生产者，B相当于消费者

![img](https://pic3.zhimg.com/v2-1a026b35fb94b42b79429fa3c850059d_b.jpg)

## **为什么要使用生产者消费者模式**

在多线程开发中，如果生产者生产数据的速度很快，而消费者消费数据的速度很慢，那么生产者就必须等待消费者消费完了数据才能够继续生产数据，因为生产那么多也没有地方放啊；同理如果消费者的速度大于生产者那么消费者就会经常处理等待状态，所以为了达到生产者和消费者生产数据和消费数据之间的**平衡**，那么就需要一个缓冲区用来存储生产者生产的数据，所以就引入了生产者-消费者模式

简单来说这里的缓冲区的作用就是为了平衡生产者和消费者的处理能力，起到一个数据缓存的作用，同时也达到了一个解耦的作用

## **生产者-消费者模式的特点**

- 保证生产者不会在缓冲区满的时候继续向缓冲区放入数据，而消费者也不会在缓冲区空的时候，消耗数据
- 当缓冲区满的时候，生产者会进入休眠状态，当下次消费者开始消耗缓冲区的数据时，生产者才会被唤醒，开始往缓冲区中添加数据；当缓冲区空的时候，消费者也会进入休眠状态，直到生产者往缓冲区中添加数据时才会被唤醒

![img](https://pic3.zhimg.com/v2-6e19e7ed5fafca7d87ffd236f6fb7a5e_b.jpg)

## **生产者-消费者模式的应用场景**

生产者-消费者模式一般用于将生产数据的一方和消费数据的一方分割开来，将生产数据与消费数据的过程解耦开来

- Excutor任务执行框架：

- - 通过将任务的提交和任务的执行解耦开来，提交任务的操作相当于生产者，执行任务的操作相当于消费者
  - 例如使用Excutor构建web服务器，用于处理线程的请求：生产者将任务提交给线程池，线程池创建线程处理任务，如果需要运行的任务数大于线程池的基本线程数，那么就把任务扔到阻塞队列（通过线程池+阻塞队列的方式比只使用一个阻塞队列的效率高很多，因为消费者能够处理就直接处理掉了，不用每个消费者都要先从阻塞队列中取出任务再执行）



- 消息中间件activeMQ: 

- - 双十一的时候，会产生大量的订单，那么不可能同时处理那么多的订单，需要将订单放入一个队列里面，然后由专门的线程处理订单。这里用户下单就是生产者，处理订单的线程就是消费者；再比如12306的抢票功能，先由一个容器存储用户提交的订单，然后再由专门处理订单的线程慢慢处理，这样可以在短时间内支持高并发服务



- 任务的处理时间比较长的情况下：

- - 比如上传附近并处理，那么这个时候可以将用户上传和处理附件分成两个过程，用一个队列暂时存储用户上传的附近，然后立刻返回用户上传成功，然后有专门的线程处理队列中的附近



## **生产者-消费者模式的优点**

- 解耦：将生产者类和消费者类进行解耦，消除代码之间的依赖性，简化工作负载的管理
- 复用：通过将生产者类和消费者类独立开来，那么可以对生产者类和消费者类进行独立的复用与扩展
- 调整并发数：由于生产者和消费者的处理速度是不一样的，可以调整并发数，给予慢的一方多的并发数，来提高任务的处理速度
- 异步：对于生产者和消费者来说能够各司其职，生产者只需要关心缓冲区是否还有数据，不需要等待消费者处理完；同样的对于消费者来说，也只需要关注缓冲区的内容，不需要关注生产者，通过异步的方式支持高并发，将一个耗时的流程拆成生产和消费两个阶段，这样生产者因为执行put()的时间比较短，而支持高并发
- 支持分布式：生产者和消费者通过队列进行通讯，所以不需要运行在同一台机器上，在分布式环境中可以通过redis的list作为队列，而消费者只需要轮询队列中是否有数据。同时还能支持集群的伸缩性，当某台机器宕掉的时候，不会导致整个集群宕掉



## **生产者-消费者模式的实现**

首先我们从最简单的开始，假设只有一个生产者线程执行put操作，向缓冲区中添加数据，同时也只有一个消费者线程从缓冲区中取出数据

![img](https://pic4.zhimg.com/v2-cb47f33c3a2e5e3091013b945cf661c3_b.png)

UML实体关系图,从UML类图中可以看出，我们的producer和consumer类都持有一个对container对象的引用，这样的设计模式实际上在很多设计模式都有用到，比如我们的装饰者模式等等，它们共同的目的都是为了达到解耦和复用的效果

![img](https://pic1.zhimg.com/v2-a152000c78260a6b3eb513e93ebfa12c_b.jpg)

在实现生产者-消费者模式之前我们需要搞清两个问题：

- 如何保证容器中数据状态的一致性
- 如何保证消费者和生产者之间的同步和协作关系

1）容器中数据状态的一致性：当一个consumer执行了take()方法之后，此时容器为空，但是还没来得及更新容器的size,那么另外一个consumer来了之后以为size不等于0，那么继续执行take(),从而造成了了状态的不一致性

2）为了保证当容器里面没有数据的时候，消费者不会继续take，此时消费者释放锁，处于阻塞状态；并且一旦生产者添加了一条数据之后，此时重新唤醒消费者，消费者重新获取到容器的锁，继续执行take();

 当容器里面满的时候，生产者也不会继续put, 此时生产者释放锁，处于阻塞状态；一旦消费者take了一条数据，此时应该唤醒生产者重新获取到容器的锁，继续put

![img](https://pic3.zhimg.com/v2-5fec414c4cc1a40e5ec9608b5e7af4c6_b.jpg)

所以对于该容器的任何访问都需要进行同步，也就是说在获取容器的数据之前，需要先获取到容器的锁。

而这里对于容器状态的同步可以参考如下几种方法：

- **Object**的wait() / notify()方法
- **Semaphore**的acquire()/release()方法
- **BlockingQueue**阻塞队列方法
- **Lock和Condition**的await() / signal()方法
- **PipedInputStream**/ **PipedOutputStream**

要构建一个生产者消费者模式，那么首先就需要构建一个固定大小的缓冲区，并且该缓冲区具有可阻塞的put方法和take方法

### **1、利用内部线程之间的通信：Object的wait() / notify()方法**

接下来我们采用第一种方法来实现该模型：使用Object的wait() / notify()方法实现生产者-消费者模型

ps:采用wait()/notify()方法的缺点是不能实现单生产者单消费者模式，因为要是用notify()就必须使用同步代码块

### **创建Container容器类**

```java
package test1;

import java.util.LinkedList;

public class Container {
    LinkedList<Integer> list = new LinkedList<Integer>();
    int capacity = 10;

    public void put(int value){
        while (true){
            try {
                //sleep不能放在同步代码块里面，因为sleep不会释放锁，
                // 当前线程会一直占有produce线程，直到达到容量，调用wait()方法主动释放锁
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (this){
                //当容器满的时候，producer处于等待状态
                while (list.size() == capacity){
                    System.out.println("container is full,waiting ....");
                    try {
                        wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                //没有满，则继续produce
                System.out.println("producer--"+ Thread.currentThread().getName()+"--put:" + value);
                list.add(value++);
                //唤醒其他所有处于wait()的线程，包括消费者和生产者
                notifyAll();
            }
        }
    }

    public Integer take(){
        Integer val = 0;
        while (true){
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (this){
                //如果容器中没有数据，consumer处于等待状态
                while (list.size() == 0){
                    System.out.println("container is empty,waiting ...");
                    try {
                        wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                //如果有数据，继续consume
                val = list.removeFirst();
                System.out.println("consumer--"+ Thread.currentThread().getName()+"--take:" + val);

                //唤醒其他所有处于wait()的线程，包括消费者和生产者
                //notify必须放在同步代码块里面
                notifyAll();
            }
        }


    }

}

```

ps: 

- sleep()的位置

这里需要注意的是sleep()不能放在synchronized代码块里面，因为我们知道sleep()执行之后是不会释放锁的，也就是说当前线程仍然持有对container对象的互斥锁，这个时候当前线程继续判断list.size是否等于capacity，不等于就继续put,然后又sleep一会，然后又继续，直到当list.size == capacity,这个时候终于进入wait()方法，我们知道wait()方法会释放锁，这个时候其他线程才有机会获取到container的互斥锁，

- notifyAll()不能单独放在producer类里面，因为notifyAll（）必须放在同步代码块里面
- 弊端：这里由于不能区分哪些是not empty或者not full或者is full/empty线程，所以需要唤醒所有其他等待的线程，但实际上我们需要的是唤醒那些not empty或者not full的线程就够了

### **创建生产者类**

```java
package test1;
import test1.Container;
import java.util.Random;

public class Producer implements Runnable{
    private Container container;
    public Producer(Container container) {
        this.container = container;
    }
    @Override
    public void run() {
        container.put(new Random().nextInt(100));
    }
}

```

### **创建消费者类**

```java
package test1;
import java.util.Random;

public class Consumer implements Runnable{
    private Container container;
    public Consumer(Container container) {
        this.container = container;
    }

    @Override
    public void run() {
        Integer val = container.take();
    }
}

```

### **测试类**

```java
package test1;

import test1.Consumer;
import test1.Container;
import test1.Producer;

public class Main {
    public static void main(String[] args){
        Container container = new Container();

        Thread producer1 = new Thread(new Producer(container));
        Thread producer2 = new Thread(new Producer(container));
        Thread producer3 = new Thread(new Producer(container));
        Thread producer4 = new Thread(new Producer(container));
        producer1.start();
        producer2.start();
        producer3.start();
        producer4.start();

        Thread consumer1 = new Thread(new Consumer(container));
        Thread consumer2 = new Thread(new Consumer(container));
        Thread consumer3 = new Thread(new Consumer(container));
        Thread consumer4 = new Thread(new Consumer(container));
        Thread consumer5 = new Thread(new Consumer(container));
        Thread consumer6 = new Thread(new Consumer(container));
        consumer1.start();
        consumer2.start();
        consumer3.start();
        consumer4.start();
        consumer5.start();
        consumer6.start();
    }
}

```

运行结果

```java
producer--Thread-1--put:80
producer--Thread-2--put:19
producer--Thread-3--put:8
producer--Thread-0--put:74
consumer--Thread-8--take:80
consumer--Thread-4--take:19
consumer--Thread-6--take:8
consumer--Thread-9--take:74
container is empty,waiting ...
container is empty,waiting ...
producer--Thread-2--put:20
consumer--Thread-7--take:20
container is empty,waiting ...
producer--Thread-3--put:9
producer--Thread-1--put:81
producer--Thread-0--put:75
consumer--Thread-5--take:9
consumer--Thread-6--take:81
consumer--Thread-8--take:75
container is empty,waiting ...
container is empty,waiting ...
container is empty,waiting ...
```

### **2、利用信号量实现生产者-消费者模型**

### **思路**

生产者消费者模型中的共享资源是一个固定大小的缓冲区，该模式需要当缓冲区满的时候，生产者不再生产数据，直到消费者消费了一个数据之后，才继续生产；同理当缓冲区空的时候，消费者不再消费数据，直到生产者生产了一个数据之后，才继续消费

如果要通过信号量来解决这个问题：关键在于找到能够跟踪缓冲区的size大小变化，并根据缓冲区的数量变化来控制消费者和生产者线程之间的协作和运行

那么很容易很够想到用两个信号量：empytyCount和fullCount分别来表示缓冲区满或者空的状态，进而能够更加容易控制消费者和生产者到底什么时候处于阻塞状态，什么时候处于运行状态

- emptyCount = N ; fullCount = 0 ; useQueue = 1

同时为了使得程序更加具有健壮性，我们还添加一个二进制信号量useQueue,确保队列的状态的完整性不受损害。例如当两个生产者同时向空队列添加数据时，从而破坏了队列内部的状态，使得其他计数信号量或者返回的缓冲区的size大小不具有一致性。（当然这里也可以使用mutex来代替二进制信号量）

```text
produce:
    P(emptyCount)//信号量emptyCount减一
    P(useQueue)//二值信号量useQueue减一，变为0（其他线程不能进入缓冲区，阻塞状态）
    putItemIntoQueue(item)//执行put操作
    V(useQueue)//二值信号量useQueue加一，变为1（其他线程可以进入缓冲区）
    V(fullCount)//信号量fullCount加一
consume:
    P(fullCount)//fullCount -= 1
    P(useQueue)//useQueue -= 1(useQueue = 0)
    item ← getItemFromQueue()
    V(useQueue)//useQueue += 1 (useQueue = 1)
    V(emptyCount)//emptyCount += 1
```

ps:  这里的两个PV操作是否可以颠倒

- P操作不可以
  首先生产者获取到信号量emptyCount，执行P(emptyCount)，确保emptyCount不等于0，也就是还有空间添加数据，从而才能够进入临界区container
  然后执行put操作，执行put操作之前需要为缓冲区加把锁，防止在put的过程中，其他线程对缓冲区进行修改，所以这个时候需要获取另外一个信号量useQueue
  相反，如果先执行了 P(useQueue)，并且此时的emptyCount = 0，那么生产者就会一直阻塞，直到消费者消费了一个数据；但是此时消费者又无法获取到互斥信号量useQueue，也会一直阻塞，所以就形成了一个死锁
  所以这两个p操作是不能交换顺序的，信号量emptyCount是useQueue的基础和前提条件
- V操作可以
  此时如果生产者已经执行完put操作，那么可以先释放互斥信号量，再执行 V(fullCount)；或者先执行 V(fullCount)再释放互斥信号量都没有关系。不会对其他的生产者消费者的状态产生影响；但是最好的还是先释放互斥锁，再执行V(fullCount)，这样可以保证当容器满的时候，消费者能够及时的获取到互斥锁

### **代码实现**

Container

```java
package test3;

import java.util.LinkedList;
import java.util.List;
import java.util.concurrent.Semaphore;

public class Container {
    Semaphore fullCount = new Semaphore(0);
    Semaphore emptyCount = new Semaphore(10);
    Semaphore isUse = new Semaphore(1);

    List list = new LinkedList<Integer>();

    public void  put(Integer val){

        try {
            emptyCount.acquire();
            isUse.acquire();

            list.add(val);
            System.out.println("producer--"+ Thread.currentThread().getName()+"--put:" + val+"===size:"+list.size());

        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            isUse.release();
            fullCount.release();
        }
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }

    public Integer get(){
        Integer val1 = 0;
        try {
            fullCount.acquire();
            isUse.acquire();

             val1 = (Integer) list.remove(0);
            System.out.println("consumer--"+ Thread.currentThread().getName()+"--take:" + val1+"===size:"+list.size());

        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            isUse.release();
            emptyCount.release();
        }

       return val1;

    }
}

```

生产者

```java
package test3;

import java.util.Random;

public class Producer implements Runnable{
    private Container container;

    public Producer(Container container) {
        this.container = container;
    }

    @Override
    public void run() {
        while (true){

            container.put(new Random().nextInt(100));
        }
    }
}

```

消费者

```java
package test3;

public class Consumer implements Runnable{
    private Container container;

    public Consumer(Container container) {
        this.container = container;
    }

    @Override
    public void run() {
        while (true){
            Integer val = container.get();

        }
    }
}

```

测试

```java
package test3;

public class Test {
    public static void main(String[] args){

        Container container = new Container();

        Thread producer1 = new Thread(new Producer(container));
        Thread producer2 = new Thread(new Producer(container));
        Thread producer3 = new Thread(new Producer(container));

        Thread consumer1 = new Thread(new Consumer(container));
        Thread consumer2 = new Thread(new Consumer(container));
        Thread consumer3 = new Thread(new Consumer(container));
        Thread consumer4 = new Thread(new Consumer(container));

        producer1.start();
        producer2.start();
        producer3.start();

        consumer1.start();
        consumer2.start();
        consumer3.start();
        consumer4.start();

    }
}

producer--Thread-0--put:74===size:1
producer--Thread-4--put:16===size:2
producer--Thread-2--put:51===size:3
producer--Thread-1--put:77===size:4
producer--Thread-3--put:93===size:5
consumer--Thread-6--take:74===size:4
consumer--Thread-6--take:16===size:3
consumer--Thread-6--take:51===size:2
consumer--Thread-6--take:77===size:1
consumer--Thread-5--take:93===size:0
producer--Thread-4--put:19===size:1
producer--Thread-3--put:68===size:2
producer--Thread-0--put:72===size:3
consumer--Thread-6--take:19===size:2
consumer--Thread-6--take:68===size:1
consumer--Thread-5--take:72===size:0
producer--Thread-1--put:82===size:1
producer--Thread-2--put:32===size:2
consumer--Thread-5--take:82===size:1

```

### **3、基于阻塞队列的生产者消费者模型**

由于这里的缓冲区由BlockingQueue容器代替，那么这里我们就不需要重新创建一个容器类了，直接创建生产者类和消费者类，并且同样的都需要拥有一个容器类BlockingQueue的实例应用

### **创建生产者类**

```java
package test;

import java.util.Random;
import java.util.concurrent.ArrayBlockingQueue;

public class Producer implements Runnable{
    private ArrayBlockingQueue<Integer> queue ;

    public Producer(ArrayBlockingQueue<Integer> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        Random random = new Random();
        while (true){
           try {
               Thread.sleep(100);
               if(queue.size() == 10) System.out.println("================the queue is full,the producer thread is waiting..................");
               int item = random.nextInt(100);
               queue.put(item);
               System.out.println("producer:" + Thread.currentThread().getName() + " produce:" + item+";the size of the queue:" + queue.size());
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
       }
    }
}

```

### **创建消费者类**

```java
package test;

import java.util.concurrent.ArrayBlockingQueue;

public class Consumer implements Runnable {
    private ArrayBlockingQueue<Integer> queue;

    public Consumer(ArrayBlockingQueue<Integer> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
       while (true){
           try {
               Thread.sleep(100);
               if(queue.size() == 0) System.out.println("=============the queue is empty,the consumer thread is waiting................");
               Integer item = queue.take();
               System.out.println("consumer:" + Thread.currentThread().getName() + " consume:" + item+";the size of the queue:" + queue.size());
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
       }

    }
}

```

### **测试类**

```java
package test;

import java.util.concurrent.ArrayBlockingQueue;

public class Test {
    public static void main(String[] args){

        ArrayBlockingQueue<Integer> queue = new ArrayBlockingQueue<Integer>(10);
        Thread producer1 = new Thread(new Producer(queue));
        Thread producer2 = new Thread(new Producer(queue));
        Thread producer3 = new Thread(new Producer(queue));
        Thread producer4 = new Thread(new Producer(queue));
        Thread producer5 = new Thread(new Producer(queue));
        producer1.start();
        producer2.start();
        producer3.start();
        producer4.start();
        producer5.start();

        Thread consumer1 = new Thread(new Consumer(queue));
        Thread consumer2 = new Thread(new Consumer(queue));
        consumer1.start();
        consumer2.start();

        try {
            producer1.join();
            producer2.join();
            producer3.join();
            producer4.join();
            producer5.join();
            consumer1.join();
            consumer2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

=============the queue is empty,the consumer thread is waiting................
consumer:Thread-5 consume:64;the size of the queue:0
producer:Thread-3 produce:64;the size of the queue:1
consumer:Thread-6 consume:87;the size of the queue:0
producer:Thread-1 produce:1;the size of the queue:3
producer:Thread-4 produce:87;the size of the queue:2
producer:Thread-2 produce:71;the size of the queue:2
producer:Thread-0 produce:76;the size of the queue:1
consumer:Thread-6 consume:71;the size of the queue:2
producer:Thread-1 produce:26;the size of the queue:6
producer:Thread-3 produce:6;the size of the queue:6
producer:Thread-0 produce:76;the size of the queue:5
producer:Thread-2 produce:37;the size of the queue:6

```

### **4、Lock和Condition的await() / signal()方法**

在用Lock和Condition的await()/signal()方法实现生产者消费者之前，我们先来了解一下Lock和synchronized都是基于锁有哪些区别，以及Condition的await()/signal()方法和Object的wait()/notify()方法都是等待和唤醒又有哪些区别

### **Lock和synchronized的区别**

**锁机制Locksynchronized**所属层次**java.util.concurrent** package中的一个接口是一个关键字，JVM内置的语言实现释放锁与加锁通过lock()/unlock()进行手动释放与加锁不需要，进入synchronized同步代码块就自动获取锁，退出同步代码块自动释放锁设置超时时间trylock(timeout)没有超时时间，线程会一直阻塞，直到获取锁公平机制设置true,为公平锁，等待时间最长的先获取没有阻塞线程列表可以查看正处于等待状态的线程列表不可以遇到异常时释放当遇到异常时在finally中执行unlock()遇到异常时释放锁底层实现乐观锁方式（cas），每次不加锁而是假设没有冲突而去完成某项操作CPU悲观锁机制，即线程获得的是独占锁，只能依靠阻塞来等待线程释放锁具体唤醒某一个线程ReentrantLock里面的Condition应用，能够控制signal哪个线程不能控制具体notify哪个线程，notifyall()唤醒所有线程灵活性比synchronized更加灵活不是那么灵活响应中断等待的线程可以响应中断不能响应中断应用场景资源竞争激烈的情况下，是synchronized的几十倍资源竞争不激烈时，优于Lock

### **Condition的await()/signal()方法和Object的wait()/notify()方法**

**方法ConditionObject**阻塞等待await()wait()唤醒其他线程signal()notify()/notifyall()使用的锁互斥锁/共享锁，如Lock同步锁:如synchronized一个锁对应可以创建多个condition对应一个Object唤醒指定的线程明确的指定线程只能通过notifyAll唤醒所有线程；或者notify()随机唤醒

### **lock和condition实现生产者消费者**

该实现方式相比较synchronized于object的wait()/notify()方法具有更加的灵活性，可以唤醒具体的消费者线程或者生产者线程，达到当缓冲区满的时候，唤醒消费者线程，此时生产者线程都将被阻塞，而不是向notifyall()那样唤醒所有的线程。

### **容器类**

```java
package test8;

import java.util.LinkedList;
import java.util.List;
import java.util.Vector;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Container{
    private final Lock lock = new ReentrantLock();
    //表示生产者线程
    private final Condition notFull = lock.newCondition();
    //表示消费者线程
    private final Condition notEmpty = lock.newCondition();
    private int capacity;
    private List<Integer> list = new LinkedList<>();

    public Container(int capacity) {
        this.capacity = capacity;

    }

    public Integer take(){
        lock.lock();
       try {
           while (list.size() == 0)
               try {
                   System.out.println("the list is empty........");
                   notEmpty.await();//阻塞消费者线程
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
           Integer val = list.remove(0);
           System.out.println("consumer--"+ Thread.currentThread().getName()+"--take:" + val+"===size:"+list.size());

           notFull.signalAll();//唤醒所有生产者线程
           return val;
       }finally {
           lock.unlock();
       }
    }

    public void put(Integer val){
        lock.lock();
       try {
           while (list.size() == capacity){
               try {
                   System.out.println("the list is full........");

                   notFull.await();//阻塞生产者线程
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
           }
           list.add(val);
           System.out.println("producer--"+ Thread.currentThread().getName()+"--put:" + val+"===size:"+ list.size());

           notEmpty.signalAll();//唤醒所有消费者线程
       }finally {
           lock.unlock();
       }

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }



}

```

### **生产者**

```java
package test8;

import java.util.Random;
import java.util.TreeMap;
import java.util.concurrent.locks.Condition;

public class Producer implements Runnable {

    private Container container;

    public Producer(Container container) {
        this.container = container;
    }

    @Override
    public void run() {
        while (true){

            container.put(new Random().nextInt(100));
        }
    }
}

```

### **消费者**

```java
package test8;

public class Consumer implements Runnable {

    private Container container;

    public Consumer(Container container) {
        this.container = container;
    }

    @Override
    public void run() {
        while (true){
            Integer val = container.take();
        }
    }
}

```

### **测试类**

```java
package test8;

public class Test {
    public static void main(String[] args){

        Container container = new Container(5);

        Thread producer1 = new Thread(new Producer(container));
        Thread producer2 = new Thread(new Producer(container));
        Thread producer3 = new Thread(new Producer(container));
        Thread producer4 = new Thread(new Producer(container));
        Thread producer5 = new Thread(new Producer(container));

        Thread consumer1 = new Thread(new Consumer(container));
        Thread consumer2 = new Thread(new Consumer(container));


        producer1.start();
        producer2.start();
        producer3.start();
        producer4.start();
        producer5.start();

        consumer1.start();
        consumer2.start();
    }
}

the list is empty........
producer--Thread-3--put:77===size:1
consumer--Thread-6--take:77===size:0
the list is empty........
producer--Thread-4--put:55===size:1
producer--Thread-0--put:62===size:2
producer--Thread-1--put:90===size:3
producer--Thread-2--put:57===size:4
consumer--Thread-5--take:55===size:3
consumer--Thread-5--take:62===size:2
consumer--Thread-5--take:90===size:1
consumer--Thread-5--take:57===size:0
the list is empty........
the list is empty........
producer--Thread-0--put:10===size:1
producer--Thread-1--put:21===size:2
producer--Thread-3--put:3===size:3
producer--Thread-4--put:75===size:4
producer--Thread-2--put:94===size:5
consumer--Thread-5--take:10===size:4
```

## **不同模式的生产者消费者模型**

### **1、单生产者单消费者(SPSC)（只有同步没有互斥）**

### **使用信号量实现**

对于单生产者单消费者，只用保证缓冲区满的时候，生产者不会继续向缓冲区放数据，缓冲区空的时候，消费者不会继续从缓冲区取数据，而不存在同时有两个生产者使用缓冲区资源，造成数据不一致的状态。

所以对于单生产者单消费者，如果采用信号量模型来实现的话，那么只需要两个信号量：empytyCount和fullCount分别来表示缓冲区满或者空的状态，进而能够更加容易控制消费者和生产者到底什么时候处于阻塞状态，什么时候处于运行状态;  而不需要使用互斥信号量了

- emptyCount = N ; fullCount = 0 ; 

```text
produce:
    P(emptyCount)//信号量emptyCount减一
    putItemIntoQueue(item)//执行put操作
    V(fullCount)//信号量fullCount加一
consume:
    P(fullCount)//fullCount -= 1
    item ← getItemFromQueue()
    V(emptyCount)//emptyCount += 1
```

实现

缓冲区容器类

```java
package test9;

import java.time.temporal.ValueRange;
import java.util.LinkedList;
import java.util.List;
import java.util.concurrent.Semaphore;

public class Container_spsc {
    Semaphore emptyCount = new Semaphore(10);
    Semaphore fullCount = new Semaphore(0);

    List<Integer> list = new LinkedList<Integer>();

    public void put(int val){
        try {
            emptyCount.acquire();
            list.add(val);
            System.out.println("producer--"+ Thread.currentThread().getName()+"--put:" + val+"===size:"+list.size());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            fullCount.release();
        }
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public Integer take(){
        Integer val = 0;

        try {
            fullCount.acquire();
             val = list.remove(0);
            System.out.println("consumer--"+ Thread.currentThread().getName()+"--take:" + val+"===size:"+list.size());

        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            emptyCount.release();
        }
        return val;
    }

}

```

生产者

```java
package test9;

import test8.Container;

import java.util.Random;

public class Producer implements Runnable {

    private Container_spsc container;

    public Producer(Container_spsc container) {
        this.container = container;
    }

    @Override
    public void run() {
        while (true){
            container.put(new Random().nextInt(100));
        }
    }
}

```

消费者类

```java
package test9;

import test8.Container;

public class Consumer implements Runnable {

    private Container_spsc container;

    public Consumer(Container_spsc container) {
        this.container = container;
    }

    @Override
    public void run() {
        while (true){
            Integer take = container.take();
        }
    }
}

```

测试

```java
package test9;

public class Test {
    public static void main(String[] args){

        Container_spsc container = new Container_spsc();
        Thread producer = new Thread(new Producer(container));
        Thread consumer = new Thread(new Consumer(container));

        producer.start();
        consumer.start();

    }
}

producer--Thread-0--put:62===size:1
consumer--Thread-1--take:62===size:0
producer--Thread-0--put:40===size:1
consumer--Thread-1--take:40===size:0
producer--Thread-0--put:86===size:1
consumer--Thread-1--take:86===size:0
producer--Thread-0--put:15===size:1
consumer--Thread-1--take:15===size:0
producer--Thread-0--put:83===size:1
consumer--Thread-1--take:83===size:0
producer--Thread-0--put:13===size:1
consumer--Thread-1--take:13===size:0
```

### **2、多生产者单消费者（MPSC）**

对于多生产者单消费者来说，多生产者之间具有互斥关系，所以这里需要一个互斥锁来实现缓冲区的互斥访问，那么具体的实现方式就是在单生产者单消费者的基础之上，加一个互斥信号量useQueue

如果采用信号量来实现的话可以如下：

- emptyCount = N ; fullCount = 0 ; useQueue = 1

```text
produce:
    P(emptyCount)//信号量emptyCount减一
    P(useQueue)//二值信号量useQueue减一，变为0（其他线程不能进入缓冲区，阻塞状态）
    putItemIntoQueue(item)//执行put操作
    V(useQueue)//二值信号量useQueue加一，变为1（其他线程可以进入缓冲区）
    V(fullCount)//信号量fullCount加一
consume:
    P(fullCount)//fullCount -= 1   
    item ← getItemFromQueue()
    V(emptyCount)//emptyCount += 1
```

具体的实现和单生产者单消费者差不多，只不过在生产者类里面多加了一个互斥信号量useQueue

### **3、单生产者多消费者（SPMC）**

对于单生产者多消费者同多生产者多消费者

- emptyCount = N ; fullCount = 0 ; useQueue = 1

```text
produce:
    P(emptyCount)//信号量emptyCount减一
    putItemIntoQueue(item)//执行put操作
    V(fullCount)//信号量fullCount加一
consume:
    P(fullCount)//fullCount -= 1   
    P(useQueue)//二值信号量useQueue减一，变为0（其他线程不能进入缓冲区，阻塞状态）
    item ← getItemFromQueue()
    V(useQueue)//二值信号量useQueue加一，变为1（其他线程可以进入缓冲区）
    V(emptyCount)//emptyCount += 1
```

具体的实现和单生产者单消费者差不多，只不过在消费者类里面多加了一个互斥信号量useQueue

### **4、多生产者多消费者（MPMC）-单缓冲区(SB)**

对于多生产者多消费者问题，是一个同步+互斥问题，不仅需要生产者和消费者之间的同步协作，还需要实现对缓冲区资源的互斥访问；这个可以参考前面对生产者消费者4种实现方式

采用信号量

- emptyCount = N ; fullCount = 0 ; useQueue = 1

```text
produce:
    P(emptyCount)//信号量emptyCount减一
    P(useQueue)//二值信号量useQueue减一，变为0（其他线程不能进入缓冲区，阻塞状态）
    putItemIntoQueue(item)//执行put操作
    V(useQueue)//二值信号量useQueue加一，变为1（其他线程可以进入缓冲区）
    V(fullCount)//信号量fullCount加一
consume:
    P(fullCount)//fullCount -= 1   
    P(useQueue)//二值信号量useQueue减一，变为0（其他线程不能进入缓冲区，阻塞状态）
    item ← getItemFromQueue()
    V(useQueue)//二值信号量useQueue加一，变为1（其他线程可以进入缓冲区）
    V(emptyCount)//emptyCount += 1
```

### **5、多生产者多消费者（MPMC）-双缓冲区(MB)**

- **为什么要用双缓冲区**：**读写分离减少释放锁和获取锁的开销**

用一个缓冲区，生产者和消费者需要先获取到缓冲区的锁才能进行put和take操作，每一次put和take都需要获取一次锁，这需要大量的同步与互斥操作，十分损耗性能。

所以如果采用双缓冲区的话，一个缓冲区bufferA用于生产者执行put操作，一个缓冲区bufferB用于消费者执行take操作；生产者线程和消费者线程在使用各自的缓冲区之前都需要先获取到缓冲区对应的锁，才能进行操作；

生产者和消费者各自使用自己独立的缓冲区，那么就不存在同一个缓冲区被put的同时进行take操作

所以一旦生产者和消费者一旦获取到了对应缓冲区的锁，那么每一次执行put/take操作时就不用再次重新获取锁了，从而减少了很多获取锁、释放锁的性能开销

- **缓冲区的切换**

如果bufferA被put满了，那么生产者释放bufferA的锁，并等待消费者释放bufferB的锁；当bufferB被take空了，消费者释放bufferB的锁，此时生产者获取到bufferB的锁，对bufferB进行put;消费者获取到bufferA的锁，对bufferA进行take,那么就完成了一次缓冲区的切换

- **双缓冲区的状态**

- - **并发读写**
    bufferA和bufferB都处于工作状态，一个读一个写
  - **单个缓冲区空闲**
    假设bufferA已经满了，那么生产者就会释放bufferA的锁，尝试获取bufferB，而此时bufferB还在执行take操作，消费者还没释放bufferB的锁，那么生产者进入等待状态
  - **缓冲区的切换**
    当bufferB为空，那么此时消费者释放bufferB的锁，尝试获取bufferA的锁，此时消费者被唤醒，重新尝试获取bufferB的锁



- **双缓冲区的死锁问题**
  如果操作完当前的缓冲区之后，先获取另外一个缓冲区的锁，再释放当前缓冲区的锁，就会发生死锁问题。如果bufferA和bufferB的线程同时尝试获取对方的锁，那么就会一直循环等待下去

- **需要注意的问题**
  由于双缓冲区是为了避免每次读写的时候不用进行同步与互斥操作，所以对于一些本来就是线程安全的类例如arrayblockingqueue就不适合作为双缓冲区，因为他们内部已经实现了每次读写操作的时候进行加锁和释放

- 应用场景：

- - **共享内存和共享文件**
  - **逻辑处理线程和IO处理线程分离**。 I/0处理线程负责网络数据的发送和接收，连接的建立和维护。 逻辑处理线程处理从IO线程接收到的包。



### **6、多生产者多消费者（MPMC）-多缓冲区(MB)**

多个缓冲区构成一个缓冲池，同样需要两个同步信号量emtpyCount和fullCount，还有一个互斥信号量useQueue,同时还需要两个变量指示哪些是空缓冲区哪些是有数据的缓冲区，多缓冲区和双缓冲区一样，同样是以空间换时间，减少单个读写操作的同步与互斥操作，对于同一个缓冲区而言，不可能同时会put和take

### **7、多生产者多消费者(MPMC)-环形缓冲区（Ring buffer）**

### **为什么要引入环形缓冲区**

讨论为什么要引入环形缓冲区，其实也就是在讨论队列缓冲区有什么弊端，而环形缓冲区是如何解决这种弊端的=

那么我们先认识一下什么是环形缓冲区

- 循环缓冲区的有用特性是，当使用一个循环缓冲区时，它不需要将其元素打乱。
- FIFO
- 所有的 push/pop 操作都是在一个固定的存储空间内进行，少掉了对于缓冲区元素所用存储空间的分配、释放

队列缓冲区

- 如果使用非循环缓冲区，那么在使用一个缓冲区时，需要移动所有元素
- LIFO
- 在执行push和pop操作时，涉及到内存的分配与释放开销大



了解了如何使用Java通过简单的synchronized与object的wait()/notify()、Lock与Condition的await()/signal()方法、BlockingQueue、信号量semaphore四种方法来实现生产者消费者模型，以后有机会我们在研究研究Linux和windows分别又是如何实现生产者消费者模型的





