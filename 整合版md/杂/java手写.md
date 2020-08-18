

```java
import java.util.Arrays;

public class IStringBuffer {
    private char[] value;
    private int count;

    //default no-arg constructor
    public MyStringBuffer() {
        value = new char[16];
    }

    //constructor with capacity
    public MyStringBuffer(int capacity) {
        value = new char[capacity];
    }

    //constructor with string
    public MyStringBuffer(String str) {
        value = new char[str.length() + 16];
        append(str);
    }

    public static void main(String[] args) {
        MyStringBuffer msb = new MyStringBuffer("the");

        msb.append(" redpig is writting java programs for fun");
        System.out.println("msb is: " + msb);
        System.out.println("length is: " + msb.length());
        System.out.println("capacity is: " + msb.capacity());

        msb.append('!');
        System.out.println("msb is: " + msb);
        System.out.println("length is: " + msb.length());
        System.out.println("capacity is: " + msb.capacity());

        msb.insert(23, "funny ");
        System.out.println("msb is: " + msb);
        System.out.println("length is: " + msb.length());
        System.out.println("capacity is: " + msb.capacity());

        msb.insert(23, ' ');
        msb.insert(23, 'a');
        System.out.println("msb is: " + msb);
        System.out.println("length is: " + msb.length());
        System.out.println("capacity is: " + msb.capacity());

        msb.delete(23, 31);
        System.out.println("msb is: " + msb);
        System.out.println("length is: " + msb.length());
        System.out.println("capacity is: " + msb.capacity());

        msb.delete(36);
        System.out.println("msb is: " + msb);
        System.out.println("length is: " + msb.length());
        System.out.println("capacity is: " + msb.capacity());

        msb.reverse();
        System.out.println("msb is: " + msb);
        System.out.println("length is: " + msb.length());
        System.out.println("capacity is: " + msb.capacity());
    }

    //追加字符串
    public void append(String str) {
        int newCapacity;
        if ((str.length() + count) - value.length > 0) {
            newCapacity = (value.length << 1) + 2;
            if (newCapacity - (str.length() + count) < 0)
                newCapacity = str.length() + count;
            value = Arrays.copyOf(value, newCapacity);
        }
        str.getChars(0, str.length(), value, count);
        count += str.length();
    }

    //追加字符
    public void append(char c) {
        this.append(String.valueOf(c));
    }

    public void insert(int pos, String str) {
        if ((str.length() + count) - value.length > 0) {
            int newCapacity;
            newCapacity = (value.length << 1) + 2;
            if (newCapacity - (str.length() + count) < 0)
                newCapacity = str.length() + count;
            value = Arrays.copyOf(value, newCapacity);
        }
        System.arraycopy(value, pos, value, pos + str.length(), count - pos);
        str.getChars(0, str.length(), value, pos);
        count += str.length();
    }

    public void insert(int pos, char c) {
        this.insert(pos, String.valueOf(c));
    }

    public void delete(int start, int end) {
        System.arraycopy(value, end, value, start, capacity() - end);
        count -= (end - start);
    }

    public void delete(int start) {
        this.delete(start, count);
    }

    public void reverse() {
        int n = count - 1;
        for (int j = (n - 1) >> 1; j >= 0; j--) {
            int k = n - j;
            char cj = value[j];
            char ck = value[k];
            value[j] = ck;
            value[k] = cj;
        }
    }

    public int length() {
        return count;
    }

    public int capacity() {
        return value.length;
    }

    public String toString() {
        return String.valueOf(value);
    }
}
```







## keyset()方法

```java
        Map<String, Integer> map = new HashMap<String, Integer>();
        map.put("a", 1);
        map.put("b", 2);
        map.put("c", 3);
        Set<String> ks = map.keySet();
        for (String s: ks){
            System.out.println(s);
        }
```

再看HashMap中keySet()方法：

```
    public Set<K> keySet() {
        Set<K> ks = keySet;
        return (ks != null ? ks : (keySet = new KeySet()));
    }
```

 

而且keySet成员初始为null，且并没有在构造函数中初始化过

```
transient volatile Set<K> keySet = null;
```

所以初次调用keySet()方法时会new KeySet()，而KeySet()是一个内部类

```
    private final class KeySet extends AbstractSet<K> {
        public Iterator<K> iterator() {
            return newKeyIterator();
        }
        public int size() {
            return size;
        }
        public boolean contains(Object o) {
            return containsKey(o);
        }
        public boolean remove(Object o) {
            return HashMap.this.removeEntryForKey(o) != null;
        }
        public void clear() {
            HashMap.this.clear();
        }
    }
```

这个时候我们只是新建了一个KeySet 内部类对象，并没有调用其他方法，而且内部类KeySet 的父类无参构造函数也并没有做啥，那么问题来了，我们是怎么获取的HashMap中的key值的，之前对这个问题一直没明白。 
其实，Set ks = map.keySet(); 这里的 ks 就仅仅只是一个Set引用，指向HashMap内部类KeySet的一个实例，重点在于该实例拥有自己的迭代器，当我们在使用增强for循环时才会调用该迭代器，也才会输出我们想要的东西。 
现在看看迭代器是如何工作的：

```
private final class KeySet extends AbstractSet<K> {
        public Iterator<K> iterator() {
            return newKeyIterator();
        }
}
```

newKeyIterator()是指向外部类的一个函数

```
Iterator<K> newKeyIterator()   {
    return new KeyIterator();
}
```

现在又指向创建的另一个内部类对象KeyIterator()，该类继承HashIterator也是HashMap的内部类

```
    private final class KeyIterator extends HashIterator<K> {
        public K next() {
            return nextEntry().getKey();
        }
    }
```

当我们在增强for循环时会调用该next()方法，它指向的是nextEntry().getKey()，Entry中不仅存放了key，value，也存放了next，指向下一个Entry对象，我们知道，HashMap的数据层实现是数组+链表，nextEntry会先遍历链表，然后再继续遍历下一个数组位置的链表，直至全部遍历完成，其部分源码如下：

```
if ((next = e.next) == null) {
    Entry[] t = table;
    while (index < t.length && (next = t[index++]) == null)
        ;
}
```

总结：keySet()方法返回一个内部引用，并指向一个内部类对象，该内部类重写了迭代器方法，当在增强for循环时才调用，并从外部类的table中取值。





## 纯手写实现HashMap



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



## 手写LRU(LinkedHashMap）

注：get()和put()方法都算是对数据的一次访问，我们要淘汰的是最长时间没有被访问过的数据。

所以，想要实现LRU，需保证以下几点：

- 支持插入key-value形式的数据。
- 可根据键值获取数据。
- 可删除最久未访问的数据。



**方法一（直接使用LinkedHashMap）**

Java中的LinkedHashMap不仅满足前两点要求，还提供了一个removeEldestEntry(Map.Entry<K,V> eldest)方法，可以删除最久未访问的结点，非常适合用来解这道题。

代码：

```java
class LRUCache extends LinkedHashMap<Integer,Integer>{
    private int capacity;
    public LRUCache(int capacity) {
        //调用父类中的构造方法创建一个LinkedHashMap，设置其容量为capacity，loadFactor为0.75，并开启accessOrder为true
        super(capacity, 0.75F, true);
        this.capacity = capacity;
    }

    public int get(int key) {
        //若key存在,返回对应value值;若key不存在,返回-1
        return super.getOrDefault(key,-1);
    }
    
    public void put(int key, int value) {
        super.put(key,value);
    }
    protected boolean removeEldestEntry(Map.Entry<Integer,Integer> eldest){
        //若返回的结果为true，则执行removeEntryForKey方法删除eldest entry
        return size() > capacity;
    }
}
```

说明：

创建LinkedHashMap时，用到了loadFactor和accessOrder。loadFactor（负载因子），是用来控制数组存放数据的疏密程度的参数。

loadFactor越趋近于1，数组中存放的数据就越多、越密，会导致查找元素效率降低。而loadFactor越趋近于0，数组中存放的数据就越少、越疏，会导致数组的利用率降低。loadFactor的默认值为0.75f，是官方给出的一个比较好的临界值。

至于accessOrder，介绍它之前，先来看一下LinkedHashMap的底层构造：

![img](https://pic1.zhimg.com/v2-8ff7f8a5830278f92835c1f81ad10c6b_b.jpg)linkedHashMap的底层结构是数组+单向链表，此外还维护一条逻辑上的双向链表

LinkedHashMap的底层数据结构和HashMap一样，采用数组+单向链表（JDK1.8之前）的形式，只是在节点Entry中增加了before和after变量，用于维护一个双向链表来保存LinkedHashMap的存储顺序。

而这个双向链表又提供了两种排序方法：插入顺序和访问顺序。

当accessOrder为false时（默认情况），linkedHashMap只会按插入顺序维护双向链表。而开启了accessOrder之后，linkedHashMap就会把每一次对结点的访问也作为标准来进行排序。也就是说，在每次插入结点/访问结点的时候，都会将相应结点移动到双向链表的尾部，从而达到按访问顺序进行排序的目的。

所以这里需将accessOrder参数开启为true。

可以看到，用LinkedHashMap实现LRU非常方便，它内部提供了我们需要的所有操作。但走这条捷径并不是很有利于我们深刻理解LRU的缓存机制，接下来我们就尝试使用最简单的数据结构来实现。



**方法二（哈希表+双向链表）**

思路：不依赖LinkedHashMap，只使用最基础的哈希表，再维护一个双向链表，提供以下几种方法即可：

- 插入结点到头部
- 移动结点到头部
- 删除结点
- 删除尾部结点

代码：

```java
import java.util.Hashtable;
class LRUCache {
    class DLinkedNode{
        int key;
        int value;
        DLinkedNode prev;
        DLinkedNode next;
    }
    //插入结点到头部
    private void addNode(DLinkedNode node){
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }
    //删除结点
    private void removeNode(DLinkedNode node){
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
    //删除尾部结点
    private void removeLastNode(){
        removeNode(tail.prev);
    }
    //移动结点到头部
    private void removeToHead(DLinkedNode node){
        removeNode(node);
        addNode(node);
    }
    private Hashtable<Integer,DLinkedNode> cache = new Hashtable<Integer,DLinkedNode>();
    private int capacity,size;
    private DLinkedNode head,tail;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.size = 0;
        head = new DLinkedNode();
        tail = new DLinkedNode();
        //将头结点和尾结点相连
        head.next = tail;
        tail.prev = head;
    }
    
    public int get(int key) {
        DLinkedNode node = cache.get(key);
        if(node==null){
            return -1;
        }
        //若key值存在，移动该结点到头部
        removeToHead(node);
        return node.value;
    }
    
    public void put(int key, int value) {
        DLinkedNode node = cache.get(key);
        //若key值不存在，直接插入结点到头部，再判断当前容量是否大于capacity，如果是，就删除尾部结点
        if(node==null){
            DLinkedNode newNode = new DLinkedNode();
            newNode.key = key;
            newNode.value = value;
            cache.put(key,newNode);
            addNode(newNode);
            size++;
            if(size>capacity){
                DLinkedNode last = tail.prev;
                cache.remove(last.key);
                removeLastNode();
                size--;
            }
        }else{
            //若key值存在，则更新value值，并移动该结点到链表头部
            node.value = value;
            removeToHead(node);
        }
    }
}
```

总结：这就是LRU的底层实现了，get和put方法的时间复杂度都是O(1)，执行效率非常高。





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



## 什么是生产者和消费者模式：

生产者和消费者模式是通过一个容器来解决生产者和消费者的强耦合问题。生产者和消费者彼此并不直接通信，而是通过阻塞队列进行通信，所以生产者生产完数据后不用等待消费者进行处理，而是直接扔给阻塞队列，消费者不找生产者要数据，而是直接从阻塞队列中获取数据，阻塞队列就相当于一个缓冲区，平衡生产者和消费者的处理能力。

### wait/notify和synchronized配合实现：

**生产者和消费者线程各一条：**

   代码实现：   

```java
package ThreadDemo.ThreadExercise;
import java.util.ArrayList;
import java.util.List;
import java.util.Random;

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
                if(list.size()>0){ //表明集合中有元素，此线程等待
                //可以是while
                    try {
                        list.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                list.add(random.nextInt(100));//0-99的随机数；
                System.out.println(Thread.currentThread().getName()+"  "+list.get(0));
                list.notify(); //通知消费者，集合中已有元素。
            }
        }
    }
}
/*
 *  消费者线程
 */
class ConsumerDemo implements Runnable{
    private List list;
    public ConsumerDemo(List list){
         this.list=list;
    }
    @Override
    public void run() {
        while(true){
            synchronized (list){
                if(list.size()<1){//可以是while
                    try {
                        list.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println(Thread.currentThread().getName()+"  "+list.remove(0));
                list.notify();
            }
        }
    }
}
public class ProducerAndConsumer {
    public static void main(String[] args) {
        List list=new ArrayList();
        Thread thread1=new Thread(new ProducerDemo(list));
        thread1.setName("生产者线程_");
        Thread thread2=new Thread(new ConsumerDemo(list));
        thread2.setName("......消费者线程_");
        thread2.start();
        thread1.start();
    }
}
```

   测试结果：      

```text
生产者线程_  88
......消费者线程_  88
生产者线程_  77
......消费者线程_  77
生产者线程_  49
......消费者线程_  49
生产者线程_  62
......消费者线程_  62
生产者线程_  94
......消费者线程_  94
生产者线程_  64
......消费者线程_  64
生产者线程_  33
......消费者线程_  33
生产者线程_  41
......消费者线程_  41
生产者线程_  7
......消费者线程_  7
生产者线程_  21
......消费者线程_  21
```

**多条生产者和消费者线程：**

   注意事项：   

多条线程需要注意是：

- 我们唤醒的时候需要使用notifyAll()，如果使用notify（）随机唤醒的可能是同一类线程，这样会导致死锁； 
- 需要将if改为while，比如生产者线程有多个，当本生产者线程wait之后，假如另一个生产者线程得到锁（本该消费者得到），如果是if，那么此线程就会继续执行，会导致数据错乱。如果是while则会继续等待。 

   代码实现：      

```java
package ThreadDemo.ThreadExercise;
import java.util.ArrayList;
import java.util.List;
import java.util.Random;

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
/*
 *  消费者线程
 */
class ConsumerDemo implements Runnable{
    private List list;
    public ConsumerDemo(List list){
         this.list=list;
    }
    @Override
    public void run() {
        while(true){
            synchronized (list){
                while(list.size()<1){
                    try {
                        list.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println(Thread.currentThread().getName()+"  "+list.remove(0));
                list.notifyAll(); //唤醒此对象锁所有等待线程（消费者和生产者线程均有）
            }
        }
    }
}
public class ProducerAndConsumer {
    public static void main(String[] args) {
        List list=new ArrayList();
        for (int i = 1; i <4 ; i++) {
            Thread thread1=new Thread(new ProducerDemo(list));
            thread1.setName("生产者线程_"+i+"_");
            Thread thread2=new Thread(new ConsumerDemo(list));
            thread2.setName("......消费者线程_"+i+"_");
            thread2.start();
            thread1.start();
        }
    }
}
```

   测试结果：      

```text
生产者线程_3_  83
......消费者线程_1_  83
生产者线程_1_  74
......消费者线程_2_  74
生产者线程_3_  18
......消费者线程_3_  18
生产者线程_1_  81
......消费者线程_2_  81
生产者线程_3_  56
......消费者线程_1_  56
生产者线程_1_  3
......消费者线程_2_  3
生产者线程_3_  0
......消费者线程_3_  0
生产者线程_1_  18
......消费者线程_2_  18
生产者线程_3_  18
......消费者线程_1_  18
生产者线程_1_  35
......消费者线程_2_  35
生产者线程_3_  81
......消费者线程_3_  81
生产者线程_1_  15
......消费者线程_2_  15
生产者线程_3_  95
......消费者线程_1_  95
生产者线程_1_  2
......消费者线程_2_  2
```

## ReentrantLock+BlockingQueue实现：

采用两把锁，实现生产者和消费者同时作业。需要注意的是，生产者的一个锁对象，消费者一个锁对象。分别用来唤醒消费者和生产者

#### 代码实现1：

**多条生产者和消费者随机交替，也就是说只要队列没有满那一直生产，只要队列没有空那就一直消费。**

```java
package ThreadDemo.ThreadExercise;

import java.util.Queue;
import java.util.Random;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;
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
class ConsumerDemo1 implements Runnable{
    ReentrantLock put;
    ReentrantLock out;
    Condition notFull;
    Condition notEmpty;
    Queue<Integer> queue;
    public ConsumerDemo1(ReentrantLock put,ReentrantLock out,Condition notFull,Condition notEmpty,Queue queue){
        this.put=put;
        this.out=out;
        this.notFull=notFull;
        this.notEmpty=notEmpty;
        this.queue=queue;
    }

    @Override
    public void run() {
        while(true){
            out.lock();
            while(queue.size()==0){
                try {
                    notEmpty.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName()+"   "+queue.poll());
            //只要队列里还有元素，就通知更多消费者来消费
            if(queue.size()>0){
                notEmpty.signal();
            }
            out.unlock();
            //只要队列没有满，就通知生产者进行生产
            if(queue.size()==9){
                put.lock();
                notFull.signal();
                put.unlock();
            }
        }
    }
}
public class ProducerAndConsumerDemo {
    public static void main(String[] args) {
        LinkedBlockingQueue<Integer> queue=new LinkedBlockingQueue<>();
        ReentrantLock put=new ReentrantLock();
        Condition notFull=put.newCondition();
        ReentrantLock out=new ReentrantLock();
        Condition notEmpty=out.newCondition();
        for (int i = 0; i <3 ; i++) {
            new Thread(new ProducerDemo1(put,out,notFull,notEmpty,queue),"生产者"+i).start();
            new Thread(new ConsumerDemo1(put,out,notFull,notEmpty,queue),"消费者"+i).start();
        }
    }
}
```

   测试结果：      

```text
生产者0.........91
生产者0.........99
消费者0   91
消费者0   99
生产者1.........99
生产者1.........12
消费者1   99
生产者2.........79
消费者1   12
生产者0.........22
消费者1   79
消费者1   22
生产者1.........23
消费者1   23
生产者1.........19
生产者1.........79
消费者2   19
消费者2   79
生产者2.........39
消费者1   39
生产者0.........48
生产者0.........43
消费者0   48
生产者1.........64
消费者0   43
生产者2.........65
消费者0   64
消费者0   65
生产者0.........24
生产者0.........72
消费者0   24
生产者1.........79
消费者0   72
生产者2.........49
消费者0   79
消费者0   49
生产者0.........41
消费者0   41
生产者0.........25
消费者1   25
```

#### 代码实现2：

**多条生产者和消费者，实现生产满了在消费，消费完了在生产。**

```java
package ThreadDemo.ThreadExercise;

import java.util.Queue;
import java.util.Random;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;
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
            // 当队列已满时才通知消费者进行消费
            if(queue.size()==10){
                out.lock();
                notEmpty.signal();
                out.unlock();
            }
        }
    }
}
class ConsumerDemo1 implements Runnable{
    ReentrantLock put;
    ReentrantLock out;
    Condition notFull;
    Condition notEmpty;
    Queue<Integer> queue;
    public ConsumerDemo1(ReentrantLock put,ReentrantLock out,Condition notFull,Condition notEmpty,Queue queue){
        this.put=put;
        this.out=out;
        this.notFull=notFull;
        this.notEmpty=notEmpty;
        this.queue=queue;
    }

    @Override
    public void run() {
        while(true){
            out.lock();
            while(queue.size()==0){
                try {
                    notEmpty.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName()+"   "+queue.poll());
            //只要队列里还有元素，就通知更多消费者来消费
            if(queue.size()>0){
                notEmpty.signal();
            }
            out.unlock();
            //当队列为空时，通知生产者进行生产
            if(queue.size()==0){
                put.lock();
                notFull.signal();
                put.unlock();
            }
        }
    }
}
public class ProducerAndConsumerDemo {
    public static void main(String[] args) {
        LinkedBlockingQueue<Integer> queue=new LinkedBlockingQueue<>();
        ReentrantLock put=new ReentrantLock();
        Condition notFull=put.newCondition();
        ReentrantLock out=new ReentrantLock();
        Condition notEmpty=out.newCondition();
        for (int i = 0; i <3 ; i++) {
            new Thread(new ProducerDemo1(put,out,notFull,notEmpty,queue),"生产者"+i).start();
            new Thread(new ConsumerDemo1(put,out,notFull,notEmpty,queue),"消费者"+i).start();
        }
    }
}
```

   测试结果      

```text
生产者0.........19
生产者0.........28
生产者0.........76
生产者1.........56
生产者1.........69
生产者1.........44
生产者1.........77
生产者1.........51
生产者1.........56
生产者1.........99
消费者0   19
消费者0   28
消费者0   76
消费者1   56
消费者1   69
消费者1   44
消费者1   77
消费者1   51
消费者1   56
消费者1   99
生产者2.........20
生产者2.........42
生产者2.........56
生产者2.........47
生产者2.........38
生产者2.........90
生产者2.........54
生产者2.........66
生产者2.........79
生产者2.........96
消费者2   20
消费者2   42
消费者2   56
消费者2   47
消费者2   38
消费者2   90
消费者2   54
消费者2   66
消费者2   79
消费者2   96
```

## BlockingQueue实现

```java
生产者消费者问题
生产者生成1~100的随机整数，消费者消费这个整数并打印。
生产者有三个，每秒生成一个随机整数。
消费者有两个，每秒消费一个整数
仓库里有20个数则生产者停止生产，小于则继续生产。

Producer
public class Producer implements Runnable {

    private String name;
    private BlockingQueue<Integer> queue;

    public Producer(String name, BlockingQueue<Integer> queue) {
        this.name = name;
        this.queue = queue;
    }

    @Override
    public void run() {
        while (true) {
            try {
                Thread.sleep(1000);
                int value = (int)(Math.random() * 100);
                queue.put(value);
                System.out.println(name + " 生产数字 -> " + value + "，当前队列有" + queue.size() + "个数");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

Consumer
public class Consumer implements Runnable {

    String name;
    BlockingQueue<Integer> queue;

    public Consumer(String name, BlockingQueue<Integer> queue) {
        this.name = name;
        this.queue = queue;
    }

    @Override
    public void run() {
        try {
            int value = 0;
            while (true) {
                Thread.sleep(1000);
                value = queue.take();
                System.out.println("holylogs " + name + " 消费数字 -> " + value + "，当前队列有" + queue.size() + "个数");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

main
// 大小为20的队列
BlockingQueue<Integer> producerQueue = new LinkedBlockingQueue<>(20);

Producer producer1 = new Producer("生产1号", producerQueue);
Producer producer2 = new Producer("生产2号", producerQueue);
Producer producer3 = new Producer("生产3号", producerQueue);

Consumer consumer1 = new Consumer("消费1号", producerQueue);
Consumer consumer2 = new Consumer("消费2号", producerQueue);

// 开始producer线程进行生产
new Thread(producer1).start();
new Thread(producer2).start();
new Thread(producer3).start();

// 开始consumer线程进行消费。
new Thread(consumer1).start();
new Thread(consumer2).start();


```

　　

## 使用Object的wait/notify的\生产消费者问题



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

  
  

## ReentrantLock实现一个商品库存的生产和消费

- ```java
  public class LockConditionTest {
   
  	private LinkedList<String> product = new LinkedList<String>();
   
  	private int maxInventory = 10; // 最大库存
   
  	private Lock lock = new ReentrantLock();// 资源锁
   
  	private Condition condition = lock.newCondition();// 库存非满和非空条件
   
  	/**
  	 * 新增商品库存
  	 * @param e
  	 */
  	public void produce(String e) {
  		lock.lock();
  		try {
  			while (product.size() == maxInventory) {
  				condition.await();
  			}
   
  			product.add(e);
  			System.out.println(" 放入一个商品库存，总库存为：" + product.size());
  			condition.signalAll();
   
  		} catch (Exception ex) {
  			ex.printStackTrace();
  		} finally {
  			lock.unlock();
  		}
  	}
   
  	/**
  	 * 消费商品
  	 * @return
  	 */
  	public String consume() {
  		String result = null;
  		lock.lock();
  		try {
  			while (product.size() == 0) {
  				condition.await();
  			}
   
  			result = product.removeLast();
  			System.out.println(" 消费一个商品，总库存为：" + product.size());
  			condition.signalAll();
   
  		} catch (Exception e) {
  			e.printStackTrace();
  		} finally {
  			lock.unlock();
  		}
   
  		return result;
  	}
   
  	/**
  	 * 生产者
  	 * @author admin
  	 *
  	 */
  	private class Producer implements Runnable {
   
  		public void run() {
  			for (int i = 0; i < 20; i++) {
  				produce(" 商品 " + i);
  			}
  		}
   
  	}
   
  	/**
  	 * 消费者
  	 * @author admin
  	 *
  	 */
  	private class Customer implements Runnable {
   
  		public void run() {
  			for (int i = 0; i < 20; i++) {
  				consume();
  			}
  		}
  	}
   
  	public static void main(String[] args) {
   
  		LockConditionTest lc = new LockConditionTest();
  		new Thread(lc.new Producer()).start();
  		new Thread(lc.new Customer()).start();
  		new Thread(lc.new Producer()).start();
  		new Thread(lc.new Customer()).start();
   
  	}
  }
  ```

  从代码中应该不难发现，生产者和消费者都在竞争同一把锁，而实际上两者没有同步关系，由于 Condition 能够支持多个等待队列以及不响应中断， 所以我们可以将生产者和消费者的等待条件和锁资源分离，从而进一步优化系统并发性能

- ```java
  	private LinkedList<String> product = new LinkedList<String>();
    	private AtomicInteger inventory = new AtomicInteger(0);// 实时库存
   
    	private int maxInventory = 10; // 最大库存
   
    	private Lock consumerLock = new ReentrantLock();// 资源锁
    	private Lock productLock = new ReentrantLock();// 资源锁
   
    	private Condition notEmptyCondition = consumerLock.newCondition();// 库存满和空条件
    	private Condition notFullCondition = productLock.newCondition();// 库存满和空条件
   
    	/**
    	 * 新增商品库存
    	 * @param e
    	 */
    	public void produce(String e) {
    		productLock.lock();
    		try {
    			while (inventory.get() == maxInventory) {
    				notFullCondition.await();
    			}
      
    			product.add(e);
    			
    			System.out.println(" 放入一个商品库存，总库存为：" + inventory.incrementAndGet());
    			
    			if(inventory.get()<maxInventory) {
    				notFullCondition.signalAll();
    			}
      
    		} catch (Exception ex) {
    			ex.printStackTrace();
    		} finally {
    			productLock.unlock();
    		}
    		
    		if(inventory.get()>0) {
    			try {
    				consumerLock.lockInterruptibly();
    				notEmptyCondition.signalAll();
    			} catch (InterruptedException e1) {
    				// TODO Auto-generated catch block
    				e1.printStackTrace();
    			}finally {
    				consumerLock.unlock();
    			}
    		}
    		
    	}
   
    	/**
    	 * 消费商品
    	 * @return
    	 */
    	public String consume() {
    		String result = null;
    		consumerLock.lock();
    		try {
    			while (inventory.get() == 0) {
    				notEmptyCondition.await();
    			}
      
    			result = product.removeLast();
    			System.out.println(" 消费一个商品，总库存为：" + inventory.decrementAndGet());
    			
    			if(inventory.get()>0) {
    				notEmptyCondition.signalAll();
    			}
    		} catch (Exception e) {
    			e.printStackTrace();
    		} finally {
    			consumerLock.unlock();
    		}
    		
    		if(inventory.get()<maxInventory) {
    			
    			try {
    				productLock.lockInterruptibly();
    				notFullCondition.signalAll();
    			} catch (InterruptedException e1) {
    				// TODO Auto-generated catch block
    				e1.printStackTrace();
    			}finally {
    				productLock.unlock();
    			}
    		}
    		return result;
    	}
    	/**
    	 * 生产者
    	 * @author admin
    	 *
    	 */
    	private class Producer implements Runnable {
   
    		public void run() {
    			for (int i = 0; i < 20; i++) {
    				produce(" 商品 " + i);
    			}
    		}
    	}
   
    	/**
    	 * 消费者
    	 * @author admin
    	 *
    	 */
    	private class Customer implements Runnable {
   
    		public void run() {
    			for (int i = 0; i < 20; i++) {
    				consume();
    			}
    		}
    	}
   
    	public static void main(String[] args) {
   
    		LockConditionTest2 lc = new LockConditionTest2();
    		new Thread(lc.new Producer()).start();
    		new Thread(lc.new Customer()).start();
      
    	}
  }
  ```

  

## BlockingQueue 实现生产者消费者

因为 BlockingQueue 是线程安全的，且从队列中获取或者移除元素时，如果队列为空，获取或移除操作则需要等待，直到队列不为空；同时，如果向队列中添加元素，假设此时队列无可用空间，添加操作也需要等待。所以 BlockingQueue 非常适合用来实现生产者消费者模式。还是以一个案例来看下它的优化，

```java
public class BlockingQueueTest {
 
	private int maxInventory = 10; // 最大库存
 
	private BlockingQueue<String> product = new LinkedBlockingQueue<>(maxInventory);// 缓存队列
 
	/**
	 * 新增商品库存
	 * @param e
	 */
	public void produce(String e) {
		try {
			product.put(e);
			System.out.println(" 放入一个商品库存，总库存为：" + product.size());
		} catch (InterruptedException e1) {
			// TODO Auto-generated catch block
			e1.printStackTrace();
		}
	}
 
	/**
	 * 消费商品
	 * @return
	 */
	public String consume() {
		String result = null;
		try {
			result = product.take();
			System.out.println(" 消费一个商品，总库存为：" + product.size());
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
 
		return result;
	}
 
	/**
	 * 生产者
	 * @author admin
	 *
	 */
	private class Producer implements Runnable {
 
		public void run() {
			for (int i = 0; i < 20; i++) {
				produce(" 商品 " + i);
			}
		}
 
	}
 
	/**
	 * 消费者
	 * @author admin
	 *
	 */
	private class Customer implements Runnable {
 
		public void run() {
			for (int i = 0; i < 20; i++) {
				consume();
			}
		}
	}
 
	public static void main(String[] args) {
 
		BlockingQueueTest lc = new BlockingQueueTest();
		new Thread(lc.new Producer()).start();
		new Thread(lc.new Customer()).start();
		new Thread(lc.new Producer()).start();
		new Thread(lc.new Customer()).start();
 
	}
}
```

在这个案例中，我们创建了一个 LinkedBlockingQueue，并设置队列大小。之后我们创建一个消费方法 consume()，方法里面调用 LinkedBlockingQueue 中的 take() 方法，消费者通过该方法获取商品，当队列中商品数量为零时，消费者将进入等待状态；我们再创建一个生产方法 produce()，方法里面调用 LinkedBlockingQueue 中的 put() 方法，生产方通过该方法往队列中放商品，如果队列满了，生产者就将进入等待状态。