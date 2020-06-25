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





## 反射

- 在Java运行环境中，对于任意一个类，可以知道这个类有哪些属性和方法

- 获取Class对象的三种方式

  - 通过类名获取 —类名.class  
  - 通过对象获取 —对象名.getClass()
  - 通过全类名获取— Class.forName(全类名)

- newInstance() 创建了一个实例，调用的哪一个构造方法呢

  - 我们在定义一个类的时候，定义一个有参数的构造器，作用是对属性进行初始化，还要写一个无参数的构造器，作用就是反射时候用

- ClassLoader

  - 类加载器是用来把类(class)加载进 JVM 的
  - 反射的作用就是对Class对象在运行出结果之前动态的修改

- 反射的常用类和函数:Java反射机制的实现要借助于4个类：Class，Constructor，Field，Method；

  - ```java
    Class clazz = Class.forName(classname);
    Method m = clazz.getMethod(methodname);
    Constructor c = clazz.getConstructor();
    Object service = c.newInstance();
    m.invoke(service);
    ```

- getMethods()与getDeclaredMethods()区别

  - getMethods(),该方法是获取本类以及父类或者父接口中所有的公共方法(public修饰符修饰的)
  - getDeclaredMethods(),该方法是获取本类中的所有方法，包括私有的(private、protected、默认以及public)的方法。





## 创建对象

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
- 场景
  - 长生命周期的对象持有短生命周期对象的引用
  - 变量不合理的作用域
  - 没有及时的将对象设置为null
  - 各种连接没显示关闭
  - 释放对象时没有删除监听器。
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



