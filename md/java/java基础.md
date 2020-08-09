## Java的简单类型及其封装器类

Java基本类型共有八种，基本类型可以分为三类，字符类型char，布尔类型boolean以及数值类型byte、short、int、long、float、double。数值类型又可以分为整数类型byte、short、int、long和浮点数类型float、double。JAVA中的数值类型不存在无符号的，它们的取值范围是固定的，不会随着机器硬件环境或者操作系统的改变而改变。实际上，JAVA中还存在另外一种基本类型void，它也有对应的包装类 java.lang.Void，不过我们无法直接对它们进行操作。8 中类型表示范围如下：

byte：8位，最大存储数据量是255，存放的数据范围是-128~127之间。

short：16位，最大数据存储量是65536，数据范围是-32768~32767之间。

int：32位，最大数据存储容量是2的32次方减1，数据范围是负的2的31次方到正的2的31次方减1。

long：64位，最大数据存储容量是2的64次方减1，数据范围为负的2的63次方到正的2的63次方减1。

float：32位，数据范围在3.4e-45~1.4e38，直接赋值时必须在数字后加上f或F。

double：64位，数据范围在4.9e-324~1.8e308，赋值时可以加d或D也可以不加。

boolean：只有true和false两个取值。

char：16位，存储Unicode码，用单引号赋值。

- JVM基本原理——java的基本类型



## 为什么等待和通知是在 Object 类而不是 Thread 中声明的

- notify(), wait()依赖于“同步锁”，而“同步锁”是对象锁持有，并且每个对象有且仅有一个





## 为什么 wait 方法需要在 synchronized 的方法中调用

- 如果我们不从同步上下文中调用 wait() 或 notify() 方法，我们将在 Java 中收到 IllegalMonitorStateException
- 调用wait()就是释放锁，释放锁的前提是必须要先获得锁，先获得锁才能释放锁。
  - 每个对象都可以被认为是一个"监视器monitor"，这个监视器由三部分组成（一个独占锁，一个入口队列，一个等待队列）。注意是一个对象只能有一个独占锁，但是任意线程线程都可以拥有这个独占锁。
  - 对于对象的同步方法而言，只有拥有这个对象的独占锁才能调用这个同步方法。如果这个独占锁被其他线程占用，那么另外一个调用该同步方法的线程就会处于阻塞状态，此线程进入入口队列。
  - 若一个拥有该独占锁的线程调用该对象同步方法的wait()方法，则该线程会释放独占锁，并加入对象的等待队列；某个线程调用notify(),notifyAll()方法是将等待队列的线程转移到入口队列，然后让他们竞争锁，所以这个调用线程本身必须拥有锁。
- 当两个线程竞争同一资源时，如果对资源的访问顺序敏感，就称存在竞态条件。导致竞态条件发生的代码区称作临界区。
  - 这个竞态条件通过使用  Java 提供的 synchronized 关键字和锁定来解决





## 序列化

- 序列化是把对象改成可以存到磁盘或通过网络发送到其他运行中的  Java 虚拟机的二进制格式的过程, 并可以通过反序列化恢复对象状态
- 在 Java 中的序列化和反序列化过程中使用哪些方法
  - readObject() 的用法、writeObject()、readExternal() 和  writeExternal()。
  - Java 序列化由java.io.ObjectOutputStream类完成
  - 如果被序列化对象实现了Serializable对象，则会调用writeOrdinaryObject()方法进行序列化
-  在 Java 序列化期间,哪些变量未序列化
  - 静态变量
  - 瞬态变量（transient ）
- serialVersionUID
  - Java的序列化机制是通过判断类的serialVersionUID来验证版本一致性的
  - 如果新类中实例变量的类型与序列化时类的类型不一致，则会反序列化失败，这时候需要更改serialVersionUID。如果只是新增了实例变量，则反序列化回来新增的是默认值；如果减少了实例变量，反序列化时会忽略掉减少的实例变量。
- 单例类序列化，需要重写readResolve()方法；否则会破坏单例原则。
  - 通过对Singleton的序列化与反序列化得到的对象是一个新的对象，这就破坏了Singleton的单例性。
- 序列化对象的引用类型成员变量，也必须是可序列化的，否则，会报错。
- 对象的类名、实例变量（包括基本类型，数组，对其他对象的引用）都会被序列化；方法、静态变量、transient实例变量都不会被序列化。





## char[]

- 在Java中，char类型占2个字节，而且Java默认采用Unicode编码，一个Unicode码是16位，所以一个Unicode码占两个字节，Java中无论汉子还是英文字母都是用Unicode编码来表示的。所以，在Java中，char类型变量可以存储一个中文汉字。



## string与stringbuilder的区别

- string 对象时恒定不变的，stringBuider对象表示的字符串是可变的
- 对于简单的字符串连接操作，在性能上stringBuilder并不一定总是优于string。只有大量的或者无法预知次数的字符串操作，才考虑stringBuilder来实现
- 当修改字符串信息时，此时不许创建对象，可以使用stringBuilder对象
- 三个方面：常量变量， 执行效率，线程
  - String                 ---->     字符串常量
  - StringBuffer      ---->     字符串变量（线程安全）
  - StringBuilder    ---->     字符串变量（非线程安全）
- 执行效率问题（高  -->低）：
  - Java中对String对象进行的操作实际上是一个不断创建新的对象并且将旧的对象回收的一个过程，所以执行速度很慢。





## Collection

- Collections.sort()内部调用的Arrays.sort()方法

- Comparable和Comparator

  - 实现了Comparable接口的类的对象的列表或数组可以通过Collections.sort或Arrays.sort进行自动排序

  - Comparable相当于“内部比较器”，而Comparator相当于“外部比较器”

  - ```java
    class Point implements Comparable<Point> 类中实现
    
    Collections.sort(list,new Comparator<Point>()  内部类
    ```

- Collection和Collections的区别

  - Collection是集合类的上级接口，继承与他有关的接口主要有List和Set
  - Collections是针对集合类的一个帮助类



## Error与Exception

- 都是继承Throwable类
  - Error（错误）是系统中的错误
  - Exception（异常）表示程序可以处理的异常，可以捕获且可能恢复
    - CheckedException：（编译时异常） 需要用try——catch显示的捕获
    - UnCheckedException（RuntimeException）：（运行时异常）不需要捕获

- JVM基本原理1——JVM是如何处理异常的



## ClassNotFoundException 和NoClassDefFoundError

- 类装载的显式和隐式两种方式
  - 类装入发生在使用以下方法调用装入的类的时候
    - Class.forName()
    - cl.loadClass()（cl 是 java.lang.ClassLoader 的实例）
    - 当调用其中一个方法的时候，指定的类（以类名为参数）由类装入器装入
  - 类装入发生在由于引用、实例化或继承导致装入类的时候（不是通过显式方法调用）
  - 类装入器可能先显式地装入一个类，然后再隐式地装入它引用的所有类。
- ClassNotFoundException
  - 使用以下三种方法装入类，但却找不到指定名称的类定义时抛出该异常，是显式类装载的抛出的异常
    - 类 Class 中的 forName() 方法。
    - 类 ClassLoader 中的 findSystemClass() 方法。
    - 类 ClassLoader 中的 loadClass() 方法。
- NoClassDefFoundError
  - 如果 Java 虚拟机或 ClassLoader 实例试图装入类定义，但却没有找到类定义时抛出该异常。
  - NoClassDefFoundError 的抛出，是不成功的隐式类装入的结果。简单说来，就是引用的类在类路径中没有找到



## equals()&&hascode

- 在obj中的equals()和hashcode()是原始的，没有被重写的，且二者都与对象的地址有关，在String等包装类中，equals()和hashcode()是被重写了的，与对象的内容有关

- 重写equals()方法为什么要同时重写hashcode()方法

  - equals如果不重写，比较的其实就是stack里的引用
  - 只重写了equals方法而没有重写hashcode()方法，在我不需要使用集合的时候可能看不出什么问题，但是一旦我需要使用集合，问题就大了
    - 我只重写了equals(),重写的equals比较的是对象的内容，当有两个new Student(1,"zhangsan"))的时候，这是两个内容相同的不同地址的对象
    - 没有重写hashcode，而obj下的hashcode的取值与对象的地址有关，所以这两个对象的hashcode是不同的
    - 重写了hascode()方法，使得hashcode的取值只与对象的内容有关，而与对象的地址无关
  - 重写equals()方法同时重写hashcode()方法，就是为了保证当两个对象通过equals()方法比较相等时，那么他们的hashCode值也一定要保证相等

- 重写hashCode方法

  - ```java
    @Override
    public int hashCode(){
    	int result=17;
    	result=31*result+name.hashCode();
    	result=31*result+age.hashCode();
    	return result;
    }
    ```

  - 任何数n*31都可以被jvm优化为(n<<5)-n，移位和减法的操作效率比乘法的操作效率高很多

- 重写equals方法

  - ```java
    @Override
        public boolean equals(Object obj) {
            if (obj != null && obj.getClass() == this.getClass()) {
                Person person= (Person) obj;
                if (person.getName() == null || name == null) {
                    return false;
                }else{
                    return name.equalsIgnoreCase(person.getName());
                }
            }
            return false;
        }
    ```





##  equals和==的区别

- == 比较的是变量(栈)内存中存放的对象的(堆)内存地址，用来判断两个对象的地址是否相同，即是否是指相同一个对象。比较的是真正意义上的指针操作
- equals用来比较的是两个对象的内容是否相等，由于所有的类都是继承自java.lang.Object类的，所以适用于所有对象，如果没有对该方法进行覆盖的话，调用的仍然是Object类中的方法，而Object中的equals方法返回的却是==的判断。
- String s="abce"是一种非常特殊的形式,和new 有本质的区别。它是java中唯一不需要new 就可以产生对象的途径。以String s="abce";形式赋值在java中叫常量,它是在常量池中而不是象new一样放在堆中。
  - 以这形式声明的字符串,只要值相等,任何多个引用都指向同一对象
  - 也可以这么理解: String str = "hello";  先在内存中找是不是有"hello"这个对象,如果有，就让str指向那个"hello".如果内存里没有"hello"，就创建一个新的对象保存"hello".  String str=new String ("hello")  就是不管内存里是不是已经有"hello"这个对象，都新建一个对象保存"hello"。

- equals和==的区别
  - 由equals的源码可以看出这里定义的equals与==是等效的（Object类中的equals没什么区别），不同的原因就在于有些类（像String、Integer等类）对equals进行了重写，但是没有对equals进行重写的类（比如我们自己写的类）就只能从Object类中继承equals方法，其equals方法与==就也是等效的，除非我们在此类中重写equals
  - "=="比"equals"运行速度快,因为"=="只是比较引用。可以看出，String类对equals方法进行了重写，用来比较指向的字符串对象所存储的字符串是否相等。其他的一些类诸如Double，Date，Integer等，都对equals方法进行了重写用来比较指向的对象所存储的内容是否相等
  - 对于equals方法，不能作用于基本数据类型的变量；
  - 对于==如果作用于基本数据类型的变量，则直接比较其存储的 “值”是否相等；如果作用于引用类型的变量，则比较的是所指向的对象的地址





## Hello World 是如何运行的

-  hello.c 源程序是由值0和1组成的位序列。
  - 一般来说，要将 hello.c 变成一个可执行的目标程序，必须要经过 预处理器、编译器、汇编器和链接器 的处理
- 名词解释
  - 位：最小的数据单位。每一位的状态只能是0或1。
  - 字节：8个二进制位构成1个"字节(Byte)"，它是存储空间的基本计量单位。
  - 　字："字"由若干个字节构成，在32位操作系统当中，一个字是4个字节





## 浅拷贝和深拷贝

- 对于数据类型是基本数据类型的成员变量，浅拷贝会直接进行值传递
- 对于数据类型是引用数据类型的成员变量,浅拷贝会进行引用传递
  - String类型属于引用数据类型，不属于基本数据类型，但是String类型的数据是存放在常量池中的，也就是无法修改的
  - 当我将name属性从“耶稣”改为“大傻子"后，并不是修改了这个数据的值，而是把这个数据的引用从指向”耶稣“这个常量改为了指向”大傻子“这个常量。
- 深拷贝对引用数据类型的成员变量的对象图中所有的对象都开辟了内存空间；而浅拷贝只是传递地址指向，新的对象并没有对引用数据类型创建内存空间。





## 修饰符

- 访问修饰符
  - public:可以被所有类访问
  - protected:除了其他类，其他都可以访问（不能修饰类，内部类除外）
  - default:同一个包里的都可以访问
  - private:最严格的访问权限，仅同一个类下的可以访问
- 非访问修饰符
  - static ，静态修饰符，修饰类方法和类变量。
  - final 最终修饰符，修饰类、方法和变量，修饰的类不能够被继承，修饰的方法不能被重新定义，修饰的变量表示为不可修改的常量。
    - 当final修饰一个变量时，已经为该变量指定了初始值，那么这个变量在编译时就可以确定下来，那么这个final变量实质上就是一个“宏变量”
  - abstract ，抽象修饰符，用来创建抽象类和抽象方法。
  - synchronized 修饰符，用于线程编程。
  - transient 修饰符,用于跳过序列化对象中特定的敏感变量
  - volatile 修饰符，用于线程编程。





## 三大特性

- 封装
  - 将数据和基于数据的操作封装在一起，尽可能地隐藏内部的细节，只保留一些对外接口使之与外部发生联系
- 继承
  - 子类拥有父类非private的属性和方法、子类可以对父类进行扩展、子类可以用自己的方式实现父类的方法
  - 编译器会默认给子类调用父类的构造器
- 多态
  - 同一操作作用于不同的对象，可以有不同的解释，产生不同的执行结果，这就是多态性
  - 重写(Override)(运行时多态)
    - 重写是子类对父类的允许访问的方法的实现过程进行重新编写, 返回值和形参都不能改变
    - 子类可以根据需要，定义特定于自己的行为
    - 不能抛出新的检查异常或者比被重写方法申明更加宽泛的异常
    - 声明为final的方法不能被重写。
    - 声明为static的方法不能被重写，但是能够被再次声明。
  - 重载(Overload)(编译时多态)
    - 一个类里面，方法名字相同，而参数不同
    - 方法能够在同一个类中或者在一个子类中被重载
  - 重写是父类与子类之间多态性的表现，在运行时起作用（动态多态性，譬如实现动态绑定）
  - 重载是一个类中多态性的表现，在编译时起作用（静态多态性，譬如实现静态绑定）
- jvm基本原理——JVM是如何执行方法调用的
- 设计模式





## 创建对象（new）

- 使用new关键字
  	→ 调用了构造函数
  
- 使用Class类的newInstance方法
  	 → 调用了构造函数
  
- 使用Constructor类的newInstance方法
  	 → 调用了构造函数
  
- 使用clone方法
  	 → 没有调用构造函数
  
- 使用反序列化
  	 → 没有调用构造函数
  
- Java在new一个对象的时候，会先查看对象所属的类有没有加载到内存中，如果没有，就会先通过类的全限定名来加载。加载并初始化完成后，再进行对象的创建工作。
  	 
  	 ①加载和初始化类
    	 
    	 通过**双亲委派模型**进行类的加载，先将请求传送给父类加载器，如果父类无法完成这个加载请求，子加载器才会尝试自己去加载。
    	 
    	 初始化也是先加载父类后加载子类。最终方法区会存储静态变量、类初始化代码、实例变量、实例初始化代码、实例方法等。
    	 
    	 ②创建对象
    	 
    	 在堆中开辟对象的所需的内存。然后对实例变量和初始化方法进行执行。还需要在栈中定义了类引用变量，然后将堆内对象的地址赋值给引用变量。

- 

- jvm基本原理1——Java对象的内存布局



## Object

- 在Java中，只有基本类型（int，boolean等）的值不是对象。其他类型，包括数组类型，不管是对象数组还是基本类型的数组都扩展于Object类
-  protected Object clone() 创建并返回此对象的一个副本。 
   boolean equals(Object obj) 指示某个其他对象是否与此对象“相等”。 
   protected void finalize() 当垃圾回收器确定不存在对该对象的更多引用时，由对象的垃圾回收器调用此方法。 
   Class<? extendsObject> getClass() 返回一个对象的运行时类。 
   int hashCode() 返回该对象的哈希码值。 
   void notify() 唤醒在此对象监视器上等待的单个线程。 
   void notifyAll() 唤醒在此对象监视器上等待的所有线程。 
   String toString() 返回该对象的字符串表示。 
   void wait() 导致当前的线程等待，直到其他线程调用此对象的 notify() 方法或 notifyAll() 方法。 
   void wait(long timeout) 导致当前的线程等待，直到其他线程调用此对象的 notify() 方法或 notifyAll() 方法，或者超过指定的时间量。 
   void wait(long timeout, int nanos) 导致当前的线程等待，直到其他线程调用此对象的 notify()





## 匿名内部类

- 匿名内部类必须继承一个抽象类或者实现一个接口。

- 匿名内部类没有类名，因此没有构造方法。

- 只能使用一次，它通常用来简化代码编写

- 匿名内部类如何访问在其外面定义的变量：外部类局部变量必须是final

- ```java
   * 匿名内部类的格式：
   * 		new 类名或者接口名(){
   * 			重写方法；
   * 		}
   * 		本质：是该类或者接口的子类。
  ```

  



## 内存泄漏 内存溢出

- 内存泄漏的两种情况：一是堆中申请的内存没释放；二是对象已不再使用，但还在内存中保留着。

- 内存泄露的症状：

  ​    应用程序长时间连续运行时性能严重下降；

  ​    应用程序中的OutOfMemoryError堆错误；

  ​    自发且奇怪的应用程序崩溃；

  ​    应用程序偶尔会耗尽连接对象。

- 场景
  - #### 1、static字段引起的内存泄露
  
    大量使用static字段会潜在的导致内存泄露，在Java中，静态字段通常拥有与整个应用程序相匹配的生命周期。
  
    解决办法：最大限度的减少静态变量的使用；单例模式时，依赖于延迟加载对象而不是立即加载方式。
  
    #### 2、未关闭的资源导致内存泄露
  
    每当创建连接或者打开流时，JVM都会为这些资源分配内存。如果没有关闭连接，会导致持续占有内存。在任意情况下，资源留下的开放连接都会消耗内存，如果我们不处理，就会降低性能，甚至OOM。
  
    解决办法：使用finally块关闭资源；关闭资源的代码，不应该有异常；jdk1.7后，可以使用try-with-resource块。
  
    #### 3、不正确的equals()和hashCode()
  
    在HashMap和HashSet这种集合中，常常用到equal()和hashCode()来比较对象，如果重写不合理，将会成为潜在的内存泄露问题。
  
    解决办法：用最佳的方式重写equals()和hashCode。
  
    #### 4、引用了外部类的内部类
  
    非静态内部内的初始化，总是需要外部类的实例；默认情况下，每个非静态内部类都包含对其包含内的隐式引用，如果我们在应用程序中使用这个内部类对象，那么即使在我们的包含类对象超出范围后，它也不会被垃圾收集。
  
    解决办法：如果内部类不需要访问包含的类成员，考虑转换为静态类。
  
    #### 5、finalize()方法造成的内存泄露
  
    重写finalize()方法时，该类的对象不会立即被垃圾收集器收集，如果finalize()方法的代码有问题，那么会潜在的引发OOM；
  
    解决办法：避免重写finalize()。
  
    #### 6、常量字符串造成的内存泄露
  
    如果我们读取一个很大的String对象，并调用了inter(），那么它将放到字符串池中，位于PermGen中，只要应用程序运行，该字符串就会保留，这就会占用内存，可能造成OOM。
  
    解决办法：增加PermGen的大小，-XX:MaxPermSize=512m；升级Java版本，JDK1.7后字符串池转移到了堆中。
  
    ### 7、使用ThreadLocal造成内存泄露
  
    使用ThreadLocal时，每个线程只要处于存货状态就可保留对其ThreadLocal变量副本的隐式调用，且将保留其自己的副本。使用不当，就会引起内存泄露。
  
    一旦线程不在存在，ThreadLocals就应该被垃圾收集，而现在线程的创建都是使用线程池，线程池有线程重用的功能，因此线程就不会被垃圾回收器回收。所以使用到ThreadLocals来保留线程池中线程的变量副本时，ThreadLocals没有显示的删除时，就会一直保留在内存中，不会被垃圾回收。
  
    解决办法：不在使用ThreadLocal时，调用remove()方法，该方法删除了此变量的当前线程值。不要使用ThreadLocal.set(null)，它只是查找与当前线程关联的Map并将键值对设置为当前线程为null。
  
    ####  8、长生命周期的对象持有短生命周期的引用，就很可能会出现内存泄露
  
    [![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)
  
    ```
    public class Test {
        Object object;
        public void test(){
            object = new Object();
            //...其他代码
        }
    }
    ```
  
    [![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)
  
    这个例子中，Test类中的test方法结束后，创建出来的object所占用的内存不会马上被认为是可以被释放，严格意义上已经导致了垃圾回收。有两种解决办法：在test方法结束处，显示给object ==null，将其打上可被回收的标志；将object作为test方法内部的局部变量。
  
    2.静态集合类像HashMap、Vector等的使用最容易出现内存泄露，这些静态变量的生命周期和应用程序一致，所有的对象Object也不能被释放，因为他们也将一直被Vector等应用着。
  
    static Vector v = new Vector(); 
    for (int i = 1; i<100; i++) 
    { 
        Object o = new Object(); 
        v.add(o); 
        o = null; 
    }
  
    在这个例子中，代码栈中存在Vector 对象的引用 v 和 Object 对象的引用 o 。在 For 循环，我们不断的生成新的对象，然后将其添加到 Vector 对象中，之后将 o 引用置空。问题是当 o 引用被置空后，如果发生 GC，我们创建的 Object 对象是否能够被 GC 回收呢？答案是否定的。因为， GC 在跟踪代码栈中的引用时，会发现 v 引用，而继续往下跟踪，就会发现 v 引用指向的内存空间中又存在指向 Object 对象的引用。也就是说尽管o 引用已经被置空，但是 Object 对象仍然存在其他的引用，是可以被访问到的，所以 GC 无法将其释放掉。如果在此循环之后， Object 对象对程序已经没有任何作用，那么我们就认为此 Java 程序发生了内存泄漏。
  
    **3.当集合里面的对象属性被修改后，再调用remove()方法时不起作用。**
  
    [![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)
  
    ```java
    public static void main(String[] args) 
    { 
        Set<Person> set = new HashSet<Person>(); 
        Person p1 = new Person("唐僧","pwd1",25); 
        Person p2 = new Person("孙悟空","pwd2",26); 
        Person p3 = new Person("猪八戒","pwd3",27); 
        set.add(p1); 
        set.add(p2); 
        set.add(p3); 
        System.out.println("总共有:"+set.size()+" 个元素!"); //结果：总共有:3 个元素! 
        p3.setAge(2); //修改p3的年龄,此时p3元素对应的hashcode值发生改变 
        set.remove(p3); //此时remove不掉，造成内存泄漏
        set.add(p3); //重新添加，居然添加成功 
        System.out.println("总共有:"+set.size()+" 个元素!"); //结果：总共有:4 个元素! 
        for (Person person : set) 
        { 
            System.out.println(person); 
        } 
    }    
    ```
  
    [![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)
  
    **4.各种连接**，数据库连接，网络连接，IO连接等没有显示调用close关闭，不被GC回收导致内存泄露，对于Resultset 和Statement 对象可以不进行显式回收，但Connection 一定要显式回收，因为Connection 在任何时候都无法自动回收，而Connection一旦回收，Resultset 和Statement 对象就会立即为NULL；
  
- 避免内存泄漏
  - 尽早释放无用对象的引用
  - 使用StringBuffer，避免使用String
  - 尽量少用静态变量，因为静态变量存放在永久代（方法区）
  - 避免在循环中创建对象

- 内存溢出：程序要求的内存超出了系统所能分配的范围

- 场景
  - 堆内存溢出
    - 堆中的内存是用来生成对象实例和数组的
    - 例子：申请了很多内存，没释放
    
  - 方法区内存溢出
    - 方法区主要存放的是类信息、常量、静态变量等
    - 如果程序加载的类过多，或者使用反射、cglib等这种动态代理生成类的技术，就可能导致该区发生内存溢出
    
  - 线程栈溢出
    - 线程栈发生问题必定是某个线程运行时产生的错误
    - 一般线程栈溢出是由于递归太深或方法调用层级过多导致的
    
  - > Jprofile exe:https://www.ej-technologies.com/download/jprofiler/version_92
  
    ## 2.4.1 Java堆溢出
  
    ```java
    import java.util.ArrayList;
    
    /**
     * @Description HeapOOM
     * @Author Zerah
     * @Date 2019/12/20 13:04
     *  VM args: -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
     */
    public class HeapOOM {
        static class OOMObject{
    
        }
    
        public static void main(String[] args) {
            ArrayList<OOMObject> list = new ArrayList<>();
            while (true){
                list.add(new OOMObject());
            }
        }
        /** 运行结果
            java.lang.OutOfMemoryError: Java heap space
            Dumping heap to java_pid7472.hprof ...
            Heap dump file created [28238887 bytes in 0.175 secs]
            Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
        **/
    }
    
    
    ```
  
    ## 2.4.2 虚拟机栈和本地方法栈溢出
  
    ```java
    /**
     * @Description JavaVMStackSOF 虚拟机栈和本地方法栈溢出OOM测试
     * @Author Zerah
     * @Date 2019/12/20 13:35
     * VM args: -Xss128k
     */
    public class JavaVMStackSOF {
        private int stackLength =1;
        public void stackLeak(){
            stackLength ++;
            stackLeak();
        }
    
        public static void main(String[] args) throws Throwable{
            JavaVMStackSOF stackSOF = new JavaVMStackSOF();
            try {
                stackSOF.stackLeak();
            } catch (Throwable e) {
                System.out.println("stack length:"+ stackSOF.stackLength);
                throw e;
            }
        }
        /** 运行结果：
         * stack length:1611
         * Exception in thread "main" java.lang.StackOverflowError
         * 	at com.zerah.concurrent.jvm.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:13)
         * 	at com.zerah.concurrent.jvm.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:13)
         * 	at com.zerah.concurrent.jvm.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:13)
         * 	省略。。。。。。
         */
    }
    
    ----------
    
    
    /**
     * @Description JavaVMStackOOM 创建线程导致内存溢出异常, 别跑了，会把机子搞死机，别问我是怎么知道的，非要跑把其他软件能保存得保存下
     * @Author Zerah
     * @Date 2019/12/23 14:13
     * VM args: -Xss2M
     */
    public class JavaVMStackOOM {
        private void dontStop(){
            while (true){
    
            }
        }
        public void stackLeakByThread(){
            while (true){
                Thread thread = new Thread(new Runnable() {
                    @Override
                    public void run() {
                        dontStop();
                    }
                });
                thread.start();
            }
        }
    
        public static void main(String[] args) {
            JavaVMStackOOM oom = new JavaVMStackOOM();
            oom.stackLeakByThread();
        }
    
    }
    
    
    ```
  
    ## 2.4.3 方法区和运行时常量池溢出
  
    ```java
    import java.util.ArrayList;
    
    /**
     * @Description RuntimeConstantPoolOOM 运行时常量池导致的内存溢出异常      JDK1.6及之前版本可测试，1.6之前常量池在永久代中分配
     * String.intern() 是一个Native方法： 如果字符串常量池中已经包含一个等于此String对象得字符串，则返回常量池中代表这个字符串得【String对象】，
     *  否则，将此String对象包含的字符串添加到常量池中，并且返回此【String对象的引用】
     *
     *  如果使用JDK1.7 + 测试，如果不限制堆内存大小，while循环将一直进行下去，JDK1.7字符串常量池由永久代转移到堆中，JDK1.8之后移除永久代由元空间替代
     *  关于元空间的测试可以看：https://blog.csdn.net/qq_16681169/article/details/70471010
     * @Author Zerah
     * @Date 2019/12/23 14:25
     *
     * VM args: -XX:PermSize=10M -XX:MaxPermSize=10M
     * 如果限制对内存大小：-XX:PermSize=10M -XX:MaxPermSize=10M -Xmx15M
     */
    public class RuntimeConstantPoolOOM {
        public static void main(String[] args) {
            // 使用List保持着常量池的引用，避免Full GC回收常量池
            ArrayList<String> list = new ArrayList<>();
            // 10MB的PermSize在Integer 范围内足够产生OOM了
            int i= 0;
            while (true){
                list.add(String.valueOf(i++).intern());
            }
        }
    }
    
    
    ```
  
    > 异常输出
  
    ```java
    Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
    	at java.util.Arrays.copyOf(Arrays.java:3210)
    	at java.util.Arrays.copyOf(Arrays.java:3181)
    	at java.util.ArrayList.grow(ArrayList.java:265)
    	at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:239)
    	at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:231)
    	at java.util.ArrayList.add(ArrayList.java:462)
    	at com.zerah.concurrent.jvm.RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java:23)
    Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=10M; support was removed in 8.0
    Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=10M; support was removed in 8.0
    1234567891011
    import java.lang.reflect.Method;
    
    --------
    /**
     * @Description JavaMethodAreaOOM 借助CGLib 使方法区出现内存溢出异常 JDK1.6
     * @Author Zerah
     * @Date 2019/12/23 15:04
     * VM args: -XX:PermSize=10M -XX:MaxPermSize=10M
     */
    public class JavaMethodAreaOOM {
        public static void main(String[] args) {
            while (true){
                Enhancer enhancer = new Enhancer();
                enhancer.setSuperclass(OOMObject.class);
                enhancer.setUseCache(false);
                enhancer.setCallback(new MethodInterceptor() {
                    @Override
                    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
                        return proxy.invokeSuper(obj,args);
                    }
                });
                enhancer.create();
            }
        }
        static class OOMObject{}
    
    }
    
    ```
  
    ## 2.4.4 本机直接内存溢出
  
    ```java
    import sun.misc.Unsafe;
    
    import java.lang.reflect.Field;
    
    /**
     * @Description DirectMemorryOOM
     * @Author Zerah
     * @Date 2019/12/23 15:49
     * VM args: -Xmx20M -XX:MaxDirectMemorySize=10M  如果不指定，默认与Java堆最大值（-Xmx）一样
     */
    public class DirectMemoryOOM {
        private static final int _1MB = 1024*1024;
    
        public static void main(String[] args) throws Exception{
            Field field = Unsafe.class.getDeclaredFields()[0];
            field.setAccessible(true);
            Unsafe unsafe = (Unsafe) field.get(null);
            while (true){
                unsafe.allocateMemory(_1MB);
            }
        }
    }
    
    
    Exception in thread "main" java.lang.OutOfMemoryError
    	at sun.misc.Unsafe.allocateMemory(Native Method)
    	at com.zerah.concur
    ```
  
    

### 一个Java内存泄漏的排查案例

**某个业务系统在一段时间突然变慢**，我们怀疑是因为出现内存泄露问题导致的，于是踏上排查之路。

#### 2.1 确定频繁Full GC现象

首先通过“虚拟机进程状况工具：jps”找出正在运行的虚拟机进程，最主要是找出这个进程在本地虚拟机的唯一ID（LVMID，Local Virtual Machine Identifier），因为在后面的排查过程中都是需要这个LVMID来确定要监控的是哪一个虚拟机进程。  同时，对于本地虚拟机进程来说，LVMID与操作系统的进程ID（PID，Process Identifier）是一致的，使用Windows的任务管理器或Unix的ps命令也可以查询到虚拟机进程的LVMID。  jps命令格式为：  `jps [ options ] [ hostid ]`  使用命令如下：  使用jps：`jps -l`  使用ps：`ps aux | grep tomat`

找到你需要监控的ID（假设为20954），再利用“虚拟机统计信息监视工具：jstat”监视虚拟机各种运行状态信息。  jstat命令格式为：  `jstat [ option vmid [interval[s|ms] [count]] ]`  使用命令如下：  `jstat -gcutil 20954 1000`  意思是每1000毫秒查询一次，一直查。gcutil的意思是已使用空间站总空间的百分比。  结果如下图：



查询结果表明：这台服务器的新生代Eden区（E，表示Eden）使用了28.30%（最后）的空间，两个Survivor区（S0、S1，表示Survivor0、Survivor1）分别是0和8.93%，老年代（O，表示Old）使用了87.33%。程序运行以来共发生Minor GC（YGC，表示Young GC）101次，总耗时1.961秒，发生Full GC（FGC，表示Full GC）7次，Full GC总耗时3.022秒，总的耗时（GCT，表示GC Time）为4.983秒。

#### 2.2 找出导致频繁Full GC的原因

分析方法通常有两种：  1）把堆dump下来再用MAT等工具进行分析，但dump堆要花较长的时间，并且文件巨大，再从服务器上拖回本地导入工具，这个过程有些折腾，不到万不得已最好别这么干。  2）更轻量级的在线分析，使用“Java内存影像工具：jmap”生成堆转储快照（一般称为headdump或dump文件）。  jmap命令格式：  `jmap [ option ] vmid`  使用命令如下：  `jmap -histo:live 20954`  查看存活的对象情况，



按照一位IT友的说法，数据不正常，十有八九就是泄露的。在我这个图上对象还是挺正常的。

#### 2.3 定位到代码

定位带代码，有很多种方法，比如前面提到的通过MAT查看Histogram即可找出是哪块代码。——我以前是使用这个方法。  也可以使用BTrace，我没有使用过。



## 枚举

- 大量实际使用枚举替代常量
  
  - 常量使用枚举定义使代码可读性增强，实现编译时检查，避免因传入无效值导致的异常行为。
- 使用 == 比较枚举类型
  - “ ==”运算符可提供编译时和运行时的安全性。
  - 运行时安全性，如果两个值均为null 都不会引发 NullPointerException。相反，如果使用equals方法，将抛出 NullPointerException：
  - 编译时安全性，两个不同枚举类型进行比较，使用equal方法比较结果确定为true，因为getXxx方法的枚举值与另一个类型枚举值一致，但逻辑上应该为false。这个问题可以使用==操作符避免
- 使用枚举实现设计模式
  - 单例模式
  - 策略模式
- 枚举类型的构造函数、属性和方法

- ```java
  
  public class Pizza {
  
      private PizzaStatus status;
      public enum PizzaStatus {
          ORDERED (5){
              @Override
              public boolean isOrdered() {
                  return true;
              }
          },
          READY (2){
              @Override
              public boolean isReady() {
                  return true;
              }
          },
          DELIVERED (0){
              @Override
              public boolean isDelivered() {
                  return true;
              }
          };
          
  @Test
  public void givenPizaOrder_whenDelivered_thenPizzaGetsDeliveredAndStatusChanges() {
      Pizza pz = new Pizza();
      pz.setStatus(Pizza.PizzaStatus.READY);
      pz.deliver();
      assertTrue(pz.getStatus() == Pizza.PizzaStatus.DELIVERED);
  }
  ```

  ```java
  public enum PizzaDeliverySystemConfiguration {
      INSTANCE;
      PizzaDeliverySystemConfiguration() {
          // Initialization configuration which involves
          // overriding defaults like delivery strategy
      }
   
      private PizzaDeliveryStrategy deliveryStrategy = PizzaDeliveryStrategy.NORMAL;
   
      public static PizzaDeliverySystemConfiguration getInstance() {
          return INSTANCE;
      }
   
      public PizzaDeliveryStrategy getDeliveryStrategy() {
          return deliveryStrategy;
      }
  }
  
  
  public enum PizzaDeliveryStrategy {
      EXPRESS {
          @Override
          public void deliver(Pizza pz) {
              System.out.println("Pizza will be delivered in express mode");
          }
      },
      NORMAL {
          @Override
          public void deliver(Pizza pz) {
              System.out.println("Pizza will be delivered in normal mode");
          }
      };
   
      public abstract void deliver(Pizza pz);
  }
  
  
  
  public void deliver() {
      if (isDeliverable()) {
          PizzaDeliverySystemConfiguration.getInstance().getDeliveryStrategy() .deliver(this);
          this.setStatus(PizzaStatus.DELIVERED);
      }
  }
  ```

  





## JDK1.8流

- 构造流的几种方式

```JAVA
// 1. Individual values
Stream stream = Stream.of("a", "b", "c");
// 2. Arrays
String [] strArray = new String[] {"a", "b", "c"};
stream = Stream.of(strArray);
stream = Arrays.stream(strArray);
// 3. Collections
List<String> list = Arrays.asList(strArray);
stream = list.stream();
```

- collect(Collectors.toList())
  - 将流转换为 list。还有 toSet()，toMap() 等
- flatMap
  - 将多个 Stream 合并为一个 Stream
- reduce
  - reduce 操作可以实现从一组值中生成一个值
- joining 
  - 接收三个参数，第一个是分界符，第二个是前缀符，第三个是结束符
  - Collectors.joining(",","[","]")



