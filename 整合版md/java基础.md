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



## String原理

### 定义：

​    从jdk1.5开始提供的新的封装字符串的类，**StringBuilder，其字符串拼接操作的效率远远高于 String。**

​    Java里面提供了String，StringBuffer和StringBuilder三个类来封装字符串

### 简介：

​    我们知道字符串其实就是由若干个字符线性排列而成的，可以理解为字符数组Array，那么既然是数组实现的，那就需要考虑到数组的特性，数组在内存中是一块连续的地址空间块，即在定义数组的时候需要指定数组的大小

​    换言之， 数组就分为可变数组和不可变数组。可变数组能够动态插入和删除，而不可变数组一旦分配好空间后则不能进行动态插入或删除操作。

​    在实际的字符串应用场景中，涉及到多种操作，比如字符串的插入，删除，修改，拼接，查询，替换...

### String:

   **不可变类**，属性value为***不可变数组***，即String初始化构造器没有初始容量为16的概念，你定义多少，String中字符数组的长度就是多少，不存在字符数组扩容一说。看下源码：



![img](https:////upload-images.jianshu.io/upload_images/5362354-078a3342b88f5fa0.png?imageMogr2/auto-orient/strip|imageView2/2/w/645/format/webp)

2





![img](https:////upload-images.jianshu.io/upload_images/5362354-d44d4ed593cfa30f.png?imageMogr2/auto-orient/strip|imageView2/2/w/965/format/webp)

3

final修饰的String 类，以及final修饰的char[] value,表示String类不可被继承，且value只能被初始化一次。这里的value变量其实就是存储了String字符串中的所有字符。

那既然String，不可变。我们再看下它的**截取方法subString（）实现**



![img](https:////upload-images.jianshu.io/upload_images/5362354-747b8021ae5ed379.png?imageMogr2/auto-orient/strip|imageView2/2/w/865/format/webp)

4

这里可以看到，在substring方法中，如果传入的参数为0，就返回自身原对象，否则就是***重新创建一个新的对象***。





![img](https:////upload-images.jianshu.io/upload_images/5362354-2054bb5c07573c13.png?imageMogr2/auto-orient/strip|imageView2/2/w/879/format/webp)

5





![img](https:////upload-images.jianshu.io/upload_images/5362354-357f1f7a43e156aa.png?imageMogr2/auto-orient/strip|imageView2/2/w/748/format/webp)

6

类似的我们可以看到，String类的concat方法，replace方法，都是内部重新生成一个String对象的。

这也就是为什么我们如果采用***String对象频繁的进行拼接，截取，替换操作效率很低下***的原因。

下面再看下StringBuilder对象的源码，分析为何其在做字符串的拼接，截取，替换方面效率远远高于String

### StringBuilder：

​    内部**可变数组**，存在初始化StringBuilder对象中**字符数组容量为16，存在扩容**。



![img](https:////upload-images.jianshu.io/upload_images/5362354-6541f26b3f361c0c.png?imageMogr2/auto-orient/strip|imageView2/2/w/716/format/webp)

7

StringBuilder类继承AbstractStringBuilder抽象类，其中StringBuilder的大部分方法都是直接调用的父类的实现。

首先看下StringBuilder的构造方法



![img](https:////upload-images.jianshu.io/upload_images/5362354-abcd880b2da6ffdc.png?imageMogr2/auto-orient/strip|imageView2/2/w/687/format/webp)

8

1：空参数的构造方法



![img](https:////upload-images.jianshu.io/upload_images/5362354-6f2592de6b3fe420.png?imageMogr2/auto-orient/strip|imageView2/2/w/711/format/webp)

9





![img](https:////upload-images.jianshu.io/upload_images/5362354-64ee89780b054ee0.png?imageMogr2/auto-orient/strip|imageView2/2/w/676/format/webp)

10

2：自定义初始容量-构造函数



![img](https:////upload-images.jianshu.io/upload_images/5362354-47d69e48629ff026.png?imageMogr2/auto-orient/strip|imageView2/2/w/741/format/webp)

11

3：以字符串String 作为参数的构造



![img](https:////upload-images.jianshu.io/upload_images/5362354-171507a17fda1ee3.png?imageMogr2/auto-orient/strip|imageView2/2/w/745/format/webp)

12

在参数Str 数组长度的基础上再增加16个字符长度，作为StringBuilder实例的初始数组容量，并将str字符串 append到StringBuilder的数组中。



![img](https:////upload-images.jianshu.io/upload_images/5362354-0be5e4014fdd6bd2.png?imageMogr2/auto-orient/strip|imageView2/2/w/751/format/webp)

13

具体看下父类AbstractStringBuilder的append方法



![img](https:////upload-images.jianshu.io/upload_images/5362354-f220c062ddf988f6.png?imageMogr2/auto-orient/strip|imageView2/2/w/1002/format/webp)

14

1：首先判断append的参数是否为null，如果为null的话，这里也是可以append进去的



![img](https:////upload-images.jianshu.io/upload_images/5362354-60e3d76ecdcc2cf4.png?imageMogr2/auto-orient/strip|imageView2/2/w/722/format/webp)

15

其中ensureCapacityInternal方法是确保这次append 的时候StringBuilder的内部数组容量是满足的，即这次要append的null字符长度为4，加上之前内部数组中已有的字符位数c之后作为参数执行。



![img](https:////upload-images.jianshu.io/upload_images/5362354-97d65df1b5a02c37.png?imageMogr2/auto-orient/strip|imageView2/2/w/712/format/webp)

16

2：如果不为null的话，就获取这次需要append的str的字符长度。紧接着执行是否需要扩容的方法

3：重点看下append方法的关键：String的 **getChars方法**（从str的0位开始，到str的长度，当前StringBuilder对象的字符数组，当前数组已有的字符长度）



![img](https:////upload-images.jianshu.io/upload_images/5362354-5bfb9b28ec893edf.png?imageMogr2/auto-orient/strip|imageView2/2/w/921/format/webp)

17





![img](https:////upload-images.jianshu.io/upload_images/5362354-87961884184a7fb1.png?imageMogr2/auto-orient/strip|imageView2/2/w/837/format/webp)

18

其实是调用了System的arraycopy方法 参数如下：

​    **value** 为str的内部不可变字符数组，

​    ***srcBegin*** 为从str 字符串数组的0下标开始，

​    **srcEnd** 为str字符串数组的长度，

​    ***dst*** 为StringBuilder对象的内部可变字符数组，

​    ***dstBegin*** 则为StringBuilder对象中已有的字符长度（char[] 已有的元素长度）

即整个StringBuilder的append方法，本质上是调用System的native方法，直接将String 类型的str字符串中的字符数组，拷贝到了StringBuilder的字符数组中



![img](https:////upload-images.jianshu.io/upload_images/5362354-874e3442e4873708.png?imageMogr2/auto-orient/strip|imageView2/2/w/839/format/webp)

19

#### toString()：

​    最后说下StringBuilder的toString方法，



![img](https:////upload-images.jianshu.io/upload_images/5362354-04dafc4f17b870c6.png?imageMogr2/auto-orient/strip|imageView2/2/w/649/format/webp)



​    这里的***toString方法直接new 一个String对象，将StringBuilder对象的value进行一个拷贝，重新生成一个对象，不共享之前StringBuilder的char[]***

以上就是StringBuilder的拼接字符串的原理分析，可以发现没有像String一样去重新new 对象，所以在频繁的拼接字符上，StringBuilder的效率远远高于String类。

### StringBuffer：

​    线程安全的高效字符串操作类，看下源码：



![img](https:////upload-images.jianshu.io/upload_images/5362354-921c369fcd91f88f.png?imageMogr2/auto-orient/strip|imageView2/2/w/683/format/webp)

19

类图和StringBuilder一样，不多说

构造函数：



![img](https:////upload-images.jianshu.io/upload_images/5362354-9ce422aefee16d0f.png?imageMogr2/auto-orient/strip|imageView2/2/w/668/format/webp)

20

和StringBuilder一样，也不用多说，重点看下其append方法：



![img](https:////upload-images.jianshu.io/upload_images/5362354-ccb4415469b38c58.png?imageMogr2/auto-orient/strip|imageView2/2/w/663/format/webp)

21

可以看到这里就是在append方法上加了同步锁，来实现多线程下的线程安全。其他的和StringBuilder一致。

这里比StringBuilder多了一个参数



![img](https:////upload-images.jianshu.io/upload_images/5362354-44b71dcbe5ee342f.png?imageMogr2/auto-orient/strip|imageView2/2/w/665/format/webp)

22

这里的作用简单介绍一下，就是去缓存toString的

可以看下StringBuffer的toString方法



![img](https:////upload-images.jianshu.io/upload_images/5362354-f84292e9ce021a53.png?imageMogr2/auto-orient/strip|imageView2/2/w/801/format/webp)

23

这里的作用就是如果StringBuffer对象此时存在toStringCache，在多次调用其toString方法时，其new出来的String对象是会共享同一个char[] 内存的，达到共享的目的。但是StringBuffer只要做了修改，其***toStringCache属性***值都会置null处理。**这也是StringBuffer和StringBuilder的一个区别点。**

### 总结：

​    String 类不可变，内部维护的char[] 数组长度不可变，为final修饰，String类也是final修饰，不存在扩容。字符串拼接，截取，都会生成一个新的对象。频繁操作字符串效率低下，因为每次都会生成新的对象。

​    StringBuilder 类内部维护可变长度char[] ， 初始化数组容量为16，存在扩容， 其append拼接字符串方法内部调用System的native方法，进行数组的拷贝，不会重新生成新的StringBuilder对象。非线程安全的字符串操作类， 其每次调用 toString方法而重新生成的String对象，不会共享StringBuilder对象内部的char[]，会进行一次char[]的copy操作。

​    StringBuffer 类内部维护可变长度char[]， 基本上与StringBuilder一致，但其为线程安全的字符串操作类，大部分方法都采用了Synchronized关键字修改，以此来实现在多线程下的操作字符串的安全性。其toString方法而重新生成的String对象，会共享StringBuffer对象中的toStringCache属性（char[]），但是每次的StringBuffer对象修改，都会置null该属性值。





## String字符串

- String 对象是如何实现的
  - 从 Java9 版本开始，工程师将 char[] 字段改为了 byte[] 字段，又维护了一个新的属性coder，它是一个编码格式的标识
  - 从 Java7 版本开始到 Java8 版本，Java 对 String 类做了一些改变。String 类中不再有offset 和 count 两个变量了。这样的好处是 String 对象占用的内存稍微少了些，同时，String.substring 方法也不再共享 char[]，从而解决了使用该方法可能导致的内存泄漏问题。
    - 在Java6中substring方法会调用new string构造函数，此时会复用原来的char数组
- String 对象的不可变性
  - String 类被 final 关键字修饰了，char[] 被 final+private 修饰，代表了String 对象不可被更改
  - 保证 String 对象的安全性
  - 可以实现字符串常量池
    - 当代码中使用String str= "abcdef"方式创建字符串对象时，JVM 首先会检查该对象是否在字符串常量池中，如果在，就返回该对象引用，否则新的字符串将在常量池中被创建。这种方式可以减少同一个值的字符串对象的重复创建，节约内存
    - String str = new String(“abc”) 这种方式，首先在编译类文件时，"abc"常量字符串将会放入到常量结构中，在类加载时，“abc"将会在常量池中创建；其次，在调用 new 时，JVM 命令将会调用 String 的构造函数，同时引用常量池中的"abc” 字符串，在堆内存中创建一个 String 对象；最后，str 将引用 String 对象
      - 具体的复制过程是先将常量池中的字符串压入栈中，在使用string的构造方法时，会拿到栈中的字符串作为构造方法的参数
      - 如果调用 intern 方法，会去查看字符串常量池中是否有等于该对象的字符串，如果没有，就在常量池中新增该对象，并返回该对象引用；如果有，就返回常量池中的字符串引用。堆内存中原有的对象由于没有引用指向它，将会通过垃圾回收器回收。
      - intern方法生成的引用或对象是在运行时常量池中。
  - 对象在内存中是一块内存地址，str 则是一个指向该内存地址的引用
- 静态常量池和运行时常量池，
  - 静态常量池是存放字符串字面量、符号引用以及类和方法的信息，而运行时常量池存放的是运行时一些直接引用。
  - 运行时常量池是在类加载完成之后，将静态常量池中的符号引用值转存到运行时常量池中，类在解析之后，将符号引用替换成直接引用。
  - 这两个常量池在JDK1.7版本之后，就移到堆内存中了，这里指的是物理空间，而逻辑上还是属于方法区（方法区是逻辑分区）。  



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









## int、Integer

- Java 语言虽然号称一切都是对象，但原始数据类型是例外。

- Integer 是 int 对应的包装类，它有一个 int 类型的字段存储数据，并且提供了基本操作，比如数学运算、int 和字符串之间转换等。

  - 在 Java 5 中新增了静态工厂方法 valueOf，在调用它的时候会利用一个缓存机制，带来了明显的性能改进。按照 Javadoc，这个值默认缓存是-128 到 127 之间。
  - 这种缓存机制并不是只有 Integer 才有，同样存在于其他的一些包装类，比如
    - Boolean，缓存了 true/false 对应实例，确切说，只会返回两个常量实例Boolean.TRUE/FALSE。
    - Short，同样是缓存了 -128 到 127 之间的数值。
    - Byte，数值有限，所以全部都被缓存。
    - Character，缓存范围’\u0000’ 到 ‘\u007F’。  

- 理解自动装箱、拆箱

  - 这种缓存机制并不是只有 Integer 才有，同样存在于其他的一些包装类，比如
  - 原则上，建议避免无意中的装箱、拆箱行为，尤其是在性能敏感的场合，创建 10 万个Java 对象和 10 万个整数的开销可不是一个数量级的，不管是内存使用还是处理速度，光是对象头的空间占用就已经是数量级的差距了

- Integer源码

  - Integer 的缓存范围虽然默认是 -128 到 127，但是在特别的应用场

    景，比如我们明确知道应用会频繁使用更大的数值，这时候应该怎么办呢

  - 缓存上限值实际是可以根据需要调整的，JVM 提供了参数设置：-XX:AutoBoxCacheMax=N

    - 这些实现，都体现在java.lang.Integer源码之中，并实现在 IntegerCache 的静态初始化块里

  - 字符串是不可变的，保证了基本的信息安全和并发编程中的线程安全。如果你去看包装类里存储数值的成员变量“value”，你会发现，不管是 Integer 还 Boolean 等，都被声明为“private final”，所以，它们同样是不可变类型

- 原始类型线程安全

  - 如果有线程安全的计算需要，建议考虑使用类似AtomicInteger、AtomicLong 这样的线程安全类
  - 特别的是，部分比较宽的数据类型，比如 float、double，甚至不能保证更新操作的原子性，可能出现程序读取到只更新了一半数据位的数值

- java 原始数据类型和引用类型局限性

  - 原始数据类型和 Java 泛型并不能配合使用
    - 这是因为 Java 的泛型某种程度上可以算作伪泛型，它完全是一种编译期的技巧，Java 编译期会自动将类型转换为对应的特定类型，这就决定了使用泛型，必须保证相应类型可以转换为 Object
  - 我们知道 Java 的对象都是引用类型，如果是一个原始数据类型数组，它在内存里是一段连续的内存，而对象数组则不然，数据存储的是引用，对象往往是分散地存储在堆的不同位置。这种设计虽然带来了极大灵活性，但是也导致了数据操作的低效，尤其是无法充分利用现代 CPU 缓存机制

- 自动装箱与自动拆箱

  - **之所以需要包装类型，是因为许多 Java 核心类库的 API 都是面向对象的**。举个例子，Java 核心类库中的容器类，就只支持引用类型
    - 调用 Integer.valueOf 方法，将 int 类型的值转换为 Integer 类型，再存储至容器类中
    - 调用 Integer.intValue 方法。返回 Integer 对象所存储的 int 值
  - 对于基本类型的数值来说，我们需要先将其转换为对应的包装类，再存入容器之中。这个转换可以是显式，也可以是隐式的，后者正是 Java 中的自动装箱
  - ArrayList 取出元素时，我们得到的实际上也是 Integer 对象。如果应用程序期待的是一个 int 值，那么就会发生自动拆箱



## JVM中Java的基本类型

- Java 虚拟机的 boolean 类型
  - 在 Java 虚拟机规范中，boolean 类型则被映射成 int 类型。具体来说，“true”被映射为整数 1，而“false”被映射为整数 0。
    - if（fag）比较时ifeq指令做是否为零判断
    - if（true == fag）比较时if_cmpne做整数比较
  - boolean 和 char 是唯二的无符号类型。
    - char 类型的取值范围则是 [0, 65535]。通常我们可以认定 char 类型的值为非负数。这种特性十分有用，比如说作为数组索引等。
- Java 虚拟机每调用一个 Java 方法，便会创建一个栈帧。我只讨论供解释器使用的解释栈帧
  - 这种栈帧有两个主要的组成部分，分别是局部变量区，以及字节码的操作数栈。
  - 这里的局部变量是广义的，除了普遍意义下的局部变量之外，它还包含实例方法的“this 指针”以及方法所接收的参数。
  - 在 Java 虚拟机规范中，局部变量区等价于一个数组
    - 除了long、double 值需要用两个数组单元来存储之外，其他基本类型以及引用类型的值均占用一个数组单元。
    - boolean、byte、char、short 这四种类型，在栈上占用的空间和 int 是一样的
    - 在 64 位的HotSpot 中，他们将占 8 个字节。
    - 这种情况仅存在于局部变量，而并不会出现在存储于堆中的字段或者数组元素上。对于byte、char 以及 short 这三种类型的字段或者数组单元，它们在堆上占用的空间分别为一字节、两字节，以及两字节，
    - 因此，当我们将一个 int 类型的值，存储到这些类型的字段或数组时，相当于做了一次隐式的掩码操作
      - 举例来说，当我们把 0xFFFFFFFF（-1）存储到一个声明为 char 类型的字段里时，由于该字段仅占两字节，所以高两位的字节便会被截取掉，最终存入“\uFFFF”。
    - boolean 字段和 boolean 数组则比较特殊
      - 为了保证堆中的 boolean 值是合法的，HotSpot 在存储时显式地进行掩码操作，也就是说，只取最后一位的值存入 boolean 字段或数组中。
  - Java 虚拟机的算数运算几乎全部依赖于操作数栈
    - 也就是说，我们需要将堆中的 boolean、byte、char 以及 short 加载到操作数栈上，而后将栈上的值当成 int 类型来运算。
    - 对于 boolean、char 这两个无符号类型来说，加载伴随着零扩展。举个例子，char 的大小为两个字节。在加载时 char 的值会被复制到 int 类型的低二字节，而高二字节则会用 0 来填充
    - 对于 byte、short 这两个类型来说，加载伴随着符号扩展。举个例子，short 的大小为两个字节。在加载时 short 的值同样会被复制到 int 类型的低二字节。如果该 short 值为非负数，即最高位为 0，那么该 int 类型的值的高二字节会用 0 来填充，否则用 1 来填充。
- Unsafe就是一些不被虚拟机控制的内存操作的合集。
  - CAS可以理解为原子性的写操作，这个概念来自于底层CPU指令。Unsafe提供了一些cas的Java接口，在即时编译器中我们会将对这些接口的调用替换成具体的CPU指令。



## 数值计算

- “危险”的 Double

  - 对 2.15-1.10 和 1.05 判等，结果判等不成立
    - 出现这种问题的主要原因是，计算机是以二进制存储数值的，浮点数也不例外
    - 对于计算机而言，0.1 无法精确表达，这是浮点数计算造成精度损失的根源
  - 我们大都听说过 BigDecimal 类型，浮点数精确表达和运算的场景，一定要使用这个类型。不过，在使用 BigDecimal 时有几个坑需要避开。
    - 浮点数运算避坑第一原则：使用 BigDecimal 表示和计算浮点数，且务必使用字符串的构造方法来初始化BigDecimal
    - 如果一定要用 Double 来初始化 BigDecimal 的话，可以使用 BigDecimal.valueOf 方法，以确保其表现和字符串形式的构造方法一致
    - BigDecimal 有 scale 和 precision 的概念，scale 表示小数点右边的位数，而 precision 表示精度，也就是有效数字的长度。
      - 调试一下可以发现，new BigDecimal(Double.toString(100)) 得到的 BigDecimal 的scale=1、precision=4；而 new BigDecimal(“100”) 得到的 BigDecimal 的 scale=0、precision=3。对于 BigDecimal 乘法操作，返回值的 scale 是两个数的 scale 相加。

- 考虑浮点数舍入和格式化的方式

  - String.format 采用四舍五入的方式进行舍入，取 1 位小数

    - 这就是由精度问题和舍入方式共同导致的，double 和 float 的 3.35 其实相当于 3.350xxx和 3.349xxx

    - 第二原则：浮点数的字符串格式化也要通过BigDecimal 进行。

      - ```java
        // 这次得到的结果是 3.3 和 3.4，符合预期
        BigDecimal num1 = new BigDecimal("3.35"); 
        BigDecimal num2 = num1.setScale(1, BigDecimal.ROUND_DOWN); 
        System.out.println(num2); 
        BigDecimal num3 = num1.setScale(1, BigDecimal.ROUND_HALF_UP); 
        System.out.println(num3);
        ```

- 用 equals 做判等，就一定是对的吗？

  - BigDecimal 的 equals 方法的注释中说明了原因，equals 比较的是 BigDecimal 的 value 和 scale，1.0 的 scale 是 1，1 的scale 是 0，所以结果一定是 false
  - 如果我们希望只比较 BigDecimal 的 value，可以使用 compareTo 方法
    - 你可能会意识到 BigDecimal 的 equals 和 hashCode 方法会同时考虑 value和 scale，如果结合 HashSet 或 HashMap 使用的话就可能会出现麻烦。比如，我们把值为 1.0 的 BigDecimal 加入 HashSet，然后判断其是否存在值为 1 的 BigDecimal，得到的结果是 false
    - 解决这个问题的办法有两个
      - 第一个方法是，使用 TreeSet 替换 HashSet。TreeSet 不使用 hashCode 方法，也不使用 equals 比较元素，而是使用 compareTo 方法，所以不会有问题	
      - 第二个方法是，把 BigDecimal 存入 HashSet 或 HashMap 前，先使用stripTrailingZeros 方法去掉尾部的零，比较的时候也去掉尾部的 0，确保 value 相同的BigDecimal，scale 也是一致的

- 小心数值溢出问题

  - 方法一是，考虑使用 Math 类的 addExact、subtractExact 等 xxExact 方法进行数值运算，这些方法可以在数值溢出时主动抛出异常。（执行后，可以得到 ArithmeticException，这是一个RuntimeException）
  - 方法二是，使用大数类 BigInteger。BigDecimal 是处理浮点数的专家，而 BigInteger 则是对大数进行科学计算的专家。





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



## 抽象类和接口

- 两者都是抽象类，都不能实例化。
- 使用抽象类是为了代码的复用，而使用接口的动机是为了实现多态性
- interface是完全抽象的，只能声明pulic的方法，实现类必须要实现不能定义方法体，也不能声明实例变量。abstract class的子类可以有选择地实现，abstract可以声明常量变量。
- abstract class的子类在继承它时，对非抽象方法既可以直接继承，也可以覆盖；而对抽象方法，可以选择实现，也可以通过再次声明其方法为抽象的方式，无需实现，留给其子类来实现，但此类必须也声明为抽象类。



## 序列化

- 两个服务之间要共享一个数据对象，就需要从对象转换成二进制流，通过网络传输，传送到对方服务，再转换回对象，供服务方法调用。这个编码和解码过程我们称之为序列化与反序列化。
- 在大量并发请求的情况下，如果序列化的速度慢，会导致请求响应时间增加；而序列化后的传输数据体积大，会导致网络吞吐量下降。所以一个优秀的序列化框架可以提高系统的整体性能
- 在 Java 中的序列化和反序列化过程中使用哪些方法
  - readObject() 的用法、writeObject()、readExternal() 和  writeExternal()。
  - Java 序列化由java.io.ObjectOutputStream类完成
  - 如果被序列化对象实现了Serializable对象，则会调用writeOrdinaryObject()方法进行序列化
- 在 Java 序列化期间,哪些变量未序列化
  - 静态变量
  - 瞬态变量（transient ）
- serialVersionUID
  - Java的序列化机制是通过判断类的serialVersionUID来验证版本一致性的
  - 如果新类中实例变量的类型与序列化时类的类型不一致，则会反序列化失败，这时候需要更改serialVersionUID。如果只是新增了实例变量，则反序列化回来新增的是默认值；如果减少了实例变量，反序列化时会忽略掉减少的实例变量。
- 单例类序列化，需要重写readResolve()方法；否则会破坏单例原则。
  - 通过对Singleton的序列化与反序列化得到的对象是一个新的对象，这就破坏了Singleton的单例性。
- 序列化对象的引用类型成员变量，也必须是可序列化的，否则，会报错。
- 对象的类名、实例变量（包括基本类型，数组，对其他对象的引用）都会被序列化；方法、静态变量、transient实例变量都不会被序列化。

- 一个使用单例模式实现的类，如果我们将该类实现 Java 的 Serializable 接口，它还是单例吗？如果要你来写一个实现了 Java 的 Serializable 接口的单例，你会怎么写呢？
  - 序列化会通过反射调用无参构造器返回一个新对象，破坏单例模式。反序列化得到的对象，和序列化之前的对象，不是同一个对象
  - 解决方法是重写readResolve，返回单例对象的方式来避免这个问题
    - Java的序列化机制提供了一个钩子方法，即私有的readresolve方法，允许我们来控制反序列化时得到的对象。
    - 序列化时，先执行了writeReplace方法，后执行了writeObject方法；在反序列化的时候，先执行了readObject方法，最后执行了readResolve方法
- Protobuf序列化
  - Protobuf 以一个 .proto 后缀的文件为基础，这个文件描述了字段以及字段类型，在序列化该数据对象的时候，Protobuf 通过.proto文件描述来生成 Protocol Buffers 格式的编码。 
  - 它使用 T-L-V（标识 - 长度 - 字段值）的数据格式来存储数据
- Spring 提供的 4 种 RedisSerializer
  - 默认情况下，RedisTemplate 使用 JdkSerializationRedisSerializer，也就是 JDK 序列化，容易产生 Redis 中保存了乱码的错觉。 StringRedisTemplate 对于 Key 和 Value，使用的是 String 序列化方式
    - 使用 RedisTemplate 读出的数据，由于是 Object 类型的，使用时可以先强制转换为 User 类型； 
    - 使用 StringRedisTemplate 读取出的字符串，需要手动将 JSON 反序列化为 User 类 型。 
    - StringRedisTemplate 和 RedisTemplate，使用这两种方式存取的数据完全无法通用。 
  - 通常考虑到易读性，可以设置 Key 的序列化器为 StringRedisSerializer。但直接使用RedisSerializer.string()，相当于使用了 UTF_8 编码的 StringRedisSerializer，需要注意字符集问题。
  - 如果希望 Value 也是使用 JSON 序列化的话，可以把 Value 序列化器设置为Jackson2JsonRedisSerializer。默认情况下，不会把类型信息保存在 Value 中，即使我们定义 RedisTemplate 的 Value 泛型为实际类型，查询出的 Value 也只能是LinkedHashMap 类型。如果希望直接获取真实的数据类型，你可以启用 JacksonObjectMapper 的 activateDefaultTyping 方法，把类型信息一起序列化保存在 Value中。
  - 如果希望 Value 以 JSON 保存并带上类型信息，更简单的方式是，直接使用RedisSerializer.json() 快捷方法来获取序列化器。
- 注意 Jackson JSON 反序列化对额外字段的处理
  - 自定义 ObjectMapper 启用 WRITE_ENUMS_USING_INDEX 序列化功能特性时，覆盖了 Spring Boot 自动创建的 ObjectMapper；而这个自动创建的ObjectMapper 设置过 FAIL_ON_UNKNOWN_PROPERTIES 反序列化特性为 false，以确保出现未知字段时不要抛出异常
  - 要修复这个问题，有三种方式
    - 第一种，同样禁用自定义的 ObjectMapper 的 FAIL_ON_UNKNOWN_PROPERTIES
    - 第二种，设置自定义类型，加上 @JsonIgnoreProperties 注解，开启 ignoreUnknown属性，以实现反序列化时忽略额外的数据
    - 第三种，不要自定义 ObjectMapper，而是直接在配置文件设置相关参数，来修改Spring 默认的 ObjectMapper 的功能
  - 忽略多余字段，是我们写业务代码时最容易遇到的一个配置项。Spring Boot 在自动配置时贴心地做了全局设置。如果需要设置更多的特性，可以直接修改配置文件spring.jackson.** 或设置 Jackson2ObjectMapperBuilderCustomizer 回调接口，来启用更多设置，无需重新定义 ObjectMapper Bean
- 反序列化时要小心类的构造方法
  - 默认情况下，在反序列化的时候，Jackson 框架只会调用无参构造方法创建对象。如果走自定义的构造方法创建对象，需要通过 @JsonCreator 来指定构造方法，并通过 @JsonProperty 设置构造方法中参数对应的 JSON 属性名
- 枚举作为 API 接口参数或返回值的两个大坑
  - 对于枚举，我建议尽量在程序内部使用，而不是作为 API 接口的参数或返回值，原因是枚举涉及序列化和反序列化时会有两个大坑。
  - 第一个坑是，客户端和服务端的枚举定义不一致时，会出异常。比如，客户端版本的枚举定
    义了 4 个枚举值，服务端定义了 5 个枚举值
    - 要解决这个问题，可以开启 Jackson 的
      read_unknown_enum_values_using_default_value 反序列化特性，也就是在枚举值未知
      的时候使用默认值
    - 并为枚举添加一个默认值，使用 @JsonEnumDefaultValue 注解注释
    - 需要注意的是，这个枚举值一定是添加在客户端 类中的，因为反序列化使用的是客户端枚举
    - 仅仅这样配置还不能让 RestTemplate 生效这个反序列化特性，还需要配置 RestTemplate，来使用 Spring Boot 的MappingJackson2HttpMessageConverter 才行
  - 第二个坑，也是更大的坑，枚举序列化反序列化实现自定义的字段非常麻烦，会涉及Jackson 的 Bug
- 使用RedisTemplate<String, Long> 能否存取 Value 是 Long的数据呢？这其中有什么坑吗？
  - 在Integer区间内返回的是Integer，超过这个区间返回Long



## char[]

- 在Java中，char类型占2个字节，而且Java默认采用Unicode编码，一个Unicode码是16位，所以一个Unicode码占两个字节，Java中无论汉子还是英文字母都是用Unicode编码来表示的。所以，在Java中，char类型变量可以存储一个中文汉字。







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

## JVM是如何处理异常的

- 在 Java 语言规范中，所有异常都是 Throwable 类或者其子类的实例。Throwable 有两大直接子类
  - 第一个是 Error，涵盖程序不应捕获的异常。
  - 第二子类则是 Exception，涵盖程序可能需要捕获并且处理的异常。
    - RuntimeException 和 Error 属于 Java 里的非检查异常（unchecked exception）。其他异常则属于检查异常（checked exception）。
    - 所有的检查异常都需要程序显式地捕获，或者在方法声明中用 throws 关键字标注。
- 异常实例的构造十分昂贵。这是由于在构造异常实例时，Java 虚拟机便需要生成该异常的栈轨迹。该操作会逐一访问当前线程的 Java 栈帧，并且记录下各种调试信息
  - 然而，该异常对应的栈轨迹并非 throw 语句的位置，而是新建异常的位置。
  - 这也是为什么在实践中，我们往往选择抛出新建异常实例的原因
- 在编译生成的字节码中，每个方法都附带一个异常表。
  - Java 代码中的 catch 代码块和 fnally 代码块都会生成异常表
  - 异常表不是声明这段代码所有有可能抛出的异常，而是声明会被捕获的异常
- fnally 代码块的编译比较复杂。当前版本 Java 编译器的做法，是复制 fnally 代码块的内容，分别放在 try-catch 代码块所有正常执行路径以及异常执行路径的出口中。
  - 编译结果包含三份 fnally 代码块。其中，前两份分别位于 try 代码块和 catch 代码块的正常执行路径出口。最后一份则作为异常处理器，监控 try 代码块以及 catch 代码块。它将捕获 try代码块触发的、未被 catch 代码块捕获的异常，以及 catch 代码块触发的异常。
  - 如果 catch 代码块捕获了异常，并且触发了另一个异常，那么 fnally 捕获并且重抛的异常是哪个呢？答案是后者。也就是说原本的异常便会被忽略掉，这对于代码调试来说十分不利。
  - Java 7 专门构造了一个名为 try-with-resources 的语法糖，在字节码层面自动使用Supressed 异常。
    - try 关键字后声明并实例化实现了 AutoCloseable 接口的类，编译器将自动添加对应的 close() 操作
    - try-with-resources 还会使用 Supressed 异常的功能，来避免原异常“被消失”。

## 异常处理

- 捕获和处理异常容易犯的错

  - “统一异常处理”方式正是我要说的第一个错：不在业务代码层面考虑异常处理，仅在框架层面粗犷捕获和处理异常。

  - 每层架构的工作性质不同，且从业务性质上异常可能分为业务异常和系统异常两大类，这就决定了很难进行统一的异常处理。我们从底向上看一下三层架构：

    - Repository 层出现异常或许可以忽略，或许可以降级，或许需要转化为一个友好的异常。如果一律捕获异常仅记录日志，很可能业务逻辑已经出错，而用户和程序本身完全感知不到
    - Service 层往往涉及数据库事务，出现异常同样不适合捕获，否则事务无法自动回滚。此外 Service 层涉及业务逻辑，有些业务逻辑执行中遇到业务异常，可能需要在异常后转入分支业务流程。如果业务异常都被框架捕获了，业务功能就会不正常。
    - 如果下层异常上升到 Controller 层还是无法处理的话，Controller 层往往会给予用户友好提示，或是根据每一个 API 的异常表返回指定的异常类型，同样无法对所有异常一视同仁。

  - 因此，我不建议在框架层面进行异常的自动、统一处理，尤其不要随意捕获异常。但，框架可以做兜底工作。如果异常上升到最上层逻辑还是无法处理的话，可以以统一的方式进行异常转换，比如通过 @RestControllerAdvice + @ExceptionHandler，来捕获这些“未处理”异常

  - 第二个错，捕获了异常后直接生吞。在任何时候，我们捕获了异常都不应该生吞，也就是直
    接丢弃异常不记录、不抛出。这样的处理方式还不如不捕获异常，因为被生吞掉的异常一旦
    导致 Bug，就很难在程序中找到蛛丝马迹，使得 Bug 排查工作难上加难。

  - 第三个错，丢弃异常的原始信息。我们来看两个不太合适的异常处理方式，虽然没有完全生吞异常，但也丢失了宝贵的异常信息。

    - 捕获异常后，完全不记录原始异常，直接抛出一个转换后异常，导致出了问题不知道 IOException 具体是哪里引起的

    - 只记录了异常消息，却丢失了异常的类型、栈等重要信息

      ```java
      // 这两种处理方式都不太合理，可以改为如下方式：	
      	@GetMapping("right1")
          public void right1() {
              try {
                  readFile();
              } catch (IOException e) {
                  log.error("文件读取错误", e);
                  throw new RuntimeException("系统忙请稍后再试");
              }
          }
      // 或者，把原始异常作为转换后新异常的 cause，原始异常信息同样不会丢：
          @GetMapping("right2")
          public void right2() {
              try {
                  readFile();
              } catch (IOException e) {
                  throw new RuntimeException("系统忙请稍后再试", e);
              }
          }
      ```

  - 第四个错，抛出异常时不指定任何消息，直接抛出没有message 的异常：

    - throw new RuntimeException();
    - java.lang.RuntimeException: null。这里的 null 非常容易引起误解。按照空指针问题排查半天才发现，其实是异常的 message为空。

- 小心 finally 中的异常

  - 在 finally 中抛出一个异常,最后在日志中只能看到 finally 中的异常，虽然 try 中的逻辑出现了异常，但却被 finally中的异常覆盖了。这是非常危险的，特别是 finally 中出现的异常是偶发的，就会在部分时候覆盖 try 中的异常，让问题更不明显
  - 至于异常为什么被覆盖，原因也很简单，因为一个方法无法出现两个异常。
    - 修复方式是，finally 代码块自己负责异常捕获和处理
    - 或者可以把 try 中的异常作为主异常抛出，使用 addSuppressed 方法把 finally 中的异常附加到主异常上
    - 其实这正是 try-with-resources 语句的做法，对于实现了 AutoCloseable 接口的资源，建议使用 try-with-resources 来释放资源，否则也可能会产生刚才提到的，释放资源时出现的异常覆盖主异常的问题

- 千万别把异常定义为静态变量

  - 把异常定义为静态变量会导致异常信息固化，这就和异常的栈一定是需要根据当前调用来动态获取相矛盾。

  - 修复方式很简单，改一下 Exceptions 类的实现，通过不同的方法把每一种异常都 new 出来抛出即可

  - ```java
        private void createOrderWrong() {
            throw Exceptions.ORDEREXISTS;
        }
    
        private void cancelOrderWrong() {
            throw Exceptions.ORDEREXISTS;
        }
        
      //   ----  
        private void createOrderRight() {
            throw Exceptions.orderExists();
        }
    
        private void cancelOrderRight() {
            throw Exceptions.orderExists();
        }
    
    public class Exceptions {
    
        public static BusinessException ORDEREXISTS = new BusinessException("订单已经存在", 3001);
    
        public static BusinessException orderExists() {
            return new BusinessException("订单已经存在", 3001);
        }
    }
    ```

- 提交线程池的任务出了异常会怎么样？

  - 线程池常用作异步处理或并行处理。那么，把任务提交到线程池处理，任务本身出现异常时会怎样呢？

    - 异常的抛出老线程退出了，线程池只能重新创建一个线程。如果每个异步任务都以异常结束，那么线程池可能完全起不到线程重用的作用

  - 因为没有手动捕获异常进行处理，显然，这种没有以统一的错误日志格式记录错误信息打印出来的形式，对生产级代码是不合适的

    - ```java
      // 设置自定义的异常处理程序作为保底，比如在声明线程池时自定义线程池的未捕获异常处理程序
      ExecutorService threadPool = Executors.newFixedThreadPool(1, new ThreadFactoryBuilder()
                      .setNameFormat(prefix + "%d")
                      .setUncaughtExceptionHandler((thread, throwable) -> log.error("ThreadPool {} got exception", thread, throwable))
                      .get());
                      
      // 或者设置全局的默认未捕获异常处理程序：
      	    static {
              Thread.setDefaultUncaughtExceptionHandler((thread, throwable) -> log.error("Thread {} got exception", thread, throwable));
          }
      
      ```

  - 通过线程池 ExecutorService 的 execute 方法提交任务到线程池处理，如果出现异常会导致线程退出，控制台输出中可以看到异常信息。那么，把 execute 方法改为 submit，线程还会退出吗，异常还能被处理程序捕获到吗？

    - 修改代码后重新执行程序，线程没退出，异常也没记录被生吞了
    - 查看 FutureTask 源码可以发现，在执行任务出现异常之后，异常存到了一个 outcome 字段中，只有在调用 get 方法获取 FutureTask 结果的时候，才会以 ExecutionException 的形式重新抛出异常

- 关于在 finally 代码块中抛出异常的坑，如果在 finally 代码块中返回值，你觉得程序会以 try 或 catch 中返回值为准，还是以 finally 中的返回值为准呢？

  - JVM采用异常表控制try-catch的跳转逻辑；
  - 对于finally中的代码块其实是复制到try和catch中的return和throw之前的方式来处理的。

## 空值处理

- 空指针问题

  - NullPointerException 是 Java 代码中最常见的异常，我将其最可能出现的场景归为以下5 种：

    - 参数值是 Integer 等包装类型，使用时因为自动拆箱出现了空指针异常；

      - 对于 Integer 的判空，可以使用 Optional.ofNullable 来构造一个 Optional，然后使用orElse(0) 把 null 替换为默认值再进行 +1 操作。
      - Optional.ofNullable(i).orElse(0) + 1

    - 字符串比较出现空指针异常；

      - 对于 String 和字面量的比较，可以把字面量放在前面，比如"OK".equals(s)，这样即使s 是 null 也不会出现空指针异常；而对于两个可能为 null 的字符串变量的 equals 比较，可以使用 Objects.equals，它会做判空处理。
      - "OK".equals(s)
      - Objects.equals(s, t)

    - 诸如 ConcurrentHashMap 这样的容器不支持 Key 和 Value 为 null，强行 put null 的Key 或 Value 会出现空指针异常；

    - A 对象包含了 B，在通过 A 对象的字段获得 B 之后，没有对字段判空就级联调用 B 的方法出现空指针异常；

      - ```java
        // 对于类似 fooService.getBarService().bar().equals(“OK”) 的级联调用，需要判空的地方有很多，包括 fooService、getBarService() 方法的返回值，以及 bar 方法返回的字符串。如果使用 if-else 来判空的话可能需要好几行代码，但使用 Optional 的话一行代码就够了。        
        Optional.ofNullable(fooService)
                        .map(FooService::getBarService)
                        .filter(barService -> "OK".equals(barService.bar()))
                        .ifPresent(result -> log.info("OK"));
        ```

    - 方法或远程服务返回的 List 不是空而是 null，没有进行判空就直接调用 List 的方法出现空指针异常。

      - 对于 rightMethod 返回的 List，由于不能确认其是否为 null，所以在调用 size 方法获得列表大小之前，同样可以使用 Optional.ofNullable 包装一下返回值，然后通过.orElse(Collections.emptyList()) 实现在 List 为 null 的时候获得一个空的 List，最后再调用 size 方法。

      - ```java
        return Optional.ofNullable(rightMethod(test.charAt(0) == '1' ? null : new FooService(),
                        test.charAt(1) == '1' ? null : 1,
                        test.charAt(2) == '1' ? null : "OK",
                        test.charAt(3) == '1' ? null : "OK"))
                        .orElse(Collections.emptyList()).size();
        ```

  - 推荐使用阿里开源的 Java 故障诊断神器Arthas。Arthas 简单易用功能强大，可以定位出大多数的 Java 生产问题。

    - 通过 watch 命令监控 wrongMethod 方法的入参
    - watch 命令的参数包括类名表达式、方法表达式和观察表达式。这里，我们设置观察类为
      AvoidNullPointerExceptionController，观察方法为 wrongMethod，观察表达式为params 表示观察入参

- POJO 中属性的 null 到底代表了什么？

  - 明确 DTO 中null 的含义。

    - 对于 JSON 到 DTO 的反序列化过程，null 的表达是有歧义的，客户端不传某个属性，或者传 null，这个属性在 DTO 中都是 null

    - 传了null，意味着客户端希望重置这个属性。因为 Java 中的 null 就是没有这个数据，无法区
      分这两种表达，或许我们可以借助Optional 来解决这个问题。

    - UserDto 中只保留 id、name 和 age 三个属性，且 name 和 age 使用 Optional 来包装，以区分客户端不传数据还是故意传 null。

      - ```Java
        userEntity.setName(user.getName().orElse(""));
        userEntity.setNickname("guest" + userEntity.getName());
        ```

  - POJO 中的字段有默认值。

    - 如果客户端不传值，就会赋值为默认值，导致创建时间也被更新到了数据库中

  - 注意字符串格式化时可能会把 null 值格式化为 null 字符串。

  - DTO 和 Entity 共用了一个 POJO

    - 对于用户昵称的设置是程序控制的，我们不应该把它们暴露在 DTO 中，否则很容易把客户端随意设置的值更新到数据库中。此外，创建时间最好让数据库设置为当前时间，不用程序控制，可以通过在字段上设置columnDefinition 来实现。
    - 使用 Hibernate 的 @DynamicUpdate 注解实现更新 SQL 的动态生成，实现只更新修改后的字段，不过需要先查询一次实体，让 Hibernate 可以“跟踪”实体属性的当前状态，以确保有效。
      - 对于MyBatis 框架如何实现类似的动态 SQL 功能，实现插入和修改 SQL 只包含 POJO 中的非空字段
      - if标签
    - 然后，由于 DTO 中已经巧妙使用了 Optional 来区分客户端不传值和传 null 值，那么业务逻辑实现上就可以按照客户端的意图来分别实现逻辑。如果不传值，那么 Optional 本身为null，直接跳过 Entity 字段的更新即可，这样动态生成的 SQL 就不会包含这个列；如果传了值，那么进一步判断传的是不是 null。

  - 数据库字段允许保存 null，会进一步增加出错的可能性和复杂度

    - 因为如果数据真正落地的时候也支持 NULL 的话，可能就有 NULL、空字符串和字符串 null 三种状态。
    - 在 UserEntity 的字段上使用 @Column 注解，把数据库字段 name、nickname、age和 createDate 都设置为 NOT NULL，并设置 createDate 的默认值为CURRENT_TIMESTAMP，由数据库来生成创建时间。

- 小心 MySQL 中有关 NULL 的三个坑

  - MySQL 中 sum 函数没统计到任何记录时，会返回 null 而不是 0，可以使用 IFNULL函数把 null 转换为 0；
  - MySQL 中 count 字段不统计 null 值，COUNT(*) 才是统计所有记录数量的正确方式。
  - MySQL 中 =NULL 并不是判断条件而是赋值，对 NULL 进行判断只能使用 IS NULL 或者 IS NOT NULL。

  




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
    - 散列表需要使用 hashCode 来定位元素放到哪个桶。如果自定义对象没有实现自定义的 hashCode 方法，就会使用 Object 超类的默认实现，得到的两个hashCode 是不同的，导致无法满足需求。
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

  - 考虑到性能，可以先进行指针判等，如果对象是同一个那么直接返回 true；

  - 需要对另一方进行判空，空对象和自身进行比较，结果一定是 fasle（if (this == o) return true）；

  - 需要判断两个对象的类型，如果类型都不同，那么直接返回 false；

  - 确保类型相同的情况下再进行类型强制转换，然后逐一判断所有字段。

  - 在实现 equals 时，我是先通过 getClass 方法判断两个对象的类型，你可能会想到还可以使用 instanceof 来判断。你能说说这两种实现方式的区别吗

    - getclass需要具体一种类型才能做比较
    - instanceof 涉及到继承的子类是都属于父类的判断

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

- 可以通过 HashSet 的 contains 方法判断元素是否在HashSet 中，同样是 Set 的 TreeSet 其 contains 方法和 HashSet 有什么区别吗
  - HashSet就是使用HashMap调用equals，判断两对象的HashCode是否相等
  - TreeSet因为是一个树形结构，则需要考虑树的左右。则需要通过compareTo计算正负值，看最后能否找到compareTo为0的值，找到则返回true
- 注意 compareTo 和 equals 的逻辑一致性
  - binarySearch 方法内部调用了元素的 compareTo 方法进行比较；
  - 修复方式很简单，确保 compareTo 的比较逻辑和 equals 的实现一致即可。通过 Comparator.comparing 这个便捷的方法来实现两个字段的比较
  - 其实，这个问题容易被忽略的原因在于两方面：
    - 我们使用了 Lombok 的 @Data 标记了 Student,其实包含了 @EqualsAndHashCode 注解的作用，也就是默认情况下使用类型所有的字段（不包括 static 和 transient 字段）参与到 equals 和 hashCode 方法的实现中。因为这两个方法的实现不是我们自己实现的，所以容易忽略其逻辑。
    - compareTo 方法需要返回数值，作为排序的依据，容易让人使用数值类型的字段随意实现
  - 对于自定义的类型，如果要实现 Comparable，请记得 equals、hashCode、compareTo 三者逻辑一致。



## equals和==的区别

- == 比较的是变量(栈)内存中存放的对象的(堆)内存地址，用来判断两个对象的地址是否相同，即是否是指相同一个对象。比较的是真正意义上的指针操作
- equals用来比较的是两个对象的内容是否相等，由于所有的类都是继承自java.lang.Object类的，所以适用于所有对象，如果没有对该方法进行覆盖的话，调用的仍然是Object类中的方法，而Object中的equals方法返回的却是==的判断。
  - quals比较的对象除了所谓的相等外，还有一个非常重要的因素，就是该对象的类加载器也必须是同一个，不然equals返回的肯定是false
    - 重启后，两个对象相等，结果是true，但是修改了某些东西后，热加载（不用重启即可生效）后，再次执行equals，返回就是false，因为热加载使用的类加载器和程序正常启动的类加载器不同
- String s="abce"是一种非常特殊的形式,和new 有本质的区别。它是java中唯一不需要new 就可以产生对象的途径。以String s="abce";形式赋值在java中叫常量,它是在常量池中而不是象new一样放在堆中。
  - 以这形式声明的字符串,只要值相等,任何多个引用都指向同一对象
  - 也可以这么理解: String str = "hello";  先在内存中找是不是有"hello"这个对象,如果有，就让str指向那个"hello".如果内存里没有"hello"，就创建一个新的对象保存"hello".  String str=new String ("hello")  就是不管内存里是不是已经有"hello"这个对象，都新建一个对象保存"hello"。
- equals和==的区别
  - 由equals的源码可以看出这里定义的equals与==是等效的（Object类中的equals没什么区别），不同的原因就在于有些类（像String、Integer等类）对equals进行了重写，但是没有对equals进行重写的类（比如我们自己写的类）就只能从Object类中继承equals方法，其equals方法与==就也是等效的，除非我们在此类中重写equals
  - "=="比"equals"运行速度快,因为"=="只是比较引用。可以看出，String类对equals方法进行了重写，用来比较指向的字符串对象所存储的字符串是否相等。其他的一些类诸如Double，Date，Integer等，都对equals方法进行了重写用来比较指向的对象所存储的内容是否相等
  - 对于equals方法，不能作用于基本数据类型的变量；对基本类型，比如 int、long，进行判等，只能使用 ==，比较的是直接值。因为基本类型的值就是其数值。
    - 对引用类型，比如 Integer、Long 和 String，进行判等，需要使用 equals 进行内容判等。因为引用类型的直接值是指针，使用 == 的话，比较的是指针，也就是两个对象在内存中的地址，即比较它们是不是同一个对象，而不是比较对象的内容。
      - Integer默认情况下会缓存[-128,127]的数值
      - 使用 == 对一个值为 128 的直接赋值的 Integer 对象和另一个值为 128 的 int 基本类型判等,我们把装箱的 Integer 和基本类型 int 比较，前者会先拆箱再比较，比较的肯定是数值而不是引用
      - 对两个 new 出来的值都为 2 的 String 使用 == 判等：new 出来的两个 String 是不同对象，引用当然不同，所以得到 false 的结果。
    - 业务代码中滥用 intern，可能会产生性能问题
      - 其实，原因在于字符串常量池是一个固定容量的 Map。如果容量太小（Number of buckets=60013）、字符串太多（1000 万个字符串），那么每一个桶中的字符串数量会非常多，所以搜索起来就很慢。输出结果中的 Average bucket size=167，代表了 Map中桶的平均长度是 167
  - 对于==如果作用于基本数据类型的变量，则直接比较其存储的 “值”是否相等；如果作用于引用类型的变量，则比较的是所指向的对象的地址





## Hello World 是如何运行的

- hello.c 源程序是由值0和1组成的位序列。
  - 一般来说，要将 hello.c 变成一个可执行的目标程序，必须要经过 预处理器、编译器、汇编器和链接器 的处理
- 名词解释
  - 位：最小的数据单位。每一位的状态只能是0或1。
  - 字节：8个二进制位构成1个"字节(Byte)"，它是存储空间的基本计量单位。
  - 字："字"由若干个字节构成，在32位操作系统当中，一个字是4个字节









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

  - 类通过暴露有限的访问接口，授权外部仅能通过类提供的方式（或者叫函数）来访问内部信息或者数据。暴露少许的几个必要的方法给调用者使用，调用者就不需要了解太多背后的业务细节。
  - 对于封装这个特性，我们需要编程语言本身提供一定的语法机制来支持。这个语法机制就是访问权限控制。

- 抽象（Abstraction）

  - 如何隐藏方法的具体实现，让调用者只需要关心方法提供了哪些功能，并不需要知道这些功能是如何实现的
  - 在面向对象编程中，我们常借助编程语言提供的接口类（比如 Java 中的 interface 关键字语法）或者抽象类（比如 Java 中的 abstract 关键字语法）这两种语法机制，来实现抽象这一特性。
  - 抽象作为一个非常宽泛的设计思想，在代码设计中，起到非常重要的指导作用。很多设计原则都体现了抽象这种设计思想，比如基于接口而非实现编程、开闭原则（对扩展开放、对修改关闭）、代码解耦（降低代码的耦合性）等

- 继承

  - 继承最大的一个好处就是代码复用。假如两个类有一些相同的属性和方法，我们就可以将这些相同的部分，抽取到父类中，让两个子类继承父类。不过，这一点也并不是继承所独有的，我们也可以通过其他方式来解决这个代码复用的问题，比如利用组合关系而不是继承关系。
  - 过度使用继承，继承层次过深过复杂，就会导致代码可读性、可维护性变差
  - 子类拥有父类非private的属性和方法、子类可以对父类进行扩展、子类可以用自己的方式实现父类的方法
  - 编译器会默认给子类调用父类的构造器

- 多态

  - 多态是指，子类可以替换父类，在实际的代码运行过程中，调用子类的方法实现。

  - 重写(Override)(运行时多态)

    - 重写是父类与子类之间多态性的表现，在运行时起作用（动态多态性，譬如实现动态绑定）
    - 重写是子类对父类的允许访问的方法的实现过程进行重新编写, 返回值和形参都不能改变
    - 子类可以根据需要，定义特定于自己的行为
    - 不能抛出新的检查异常或者比被重写方法申明更加宽泛的异常
    - 声明为final的方法不能被重写。
    - 声明为static的方法不能被重写，但是能够被再次声明。

  - 重载(Overload)(编译时多态)

    - 重载是一个类中多态性的表现，在编译时起作用（静态多态性，譬如实现静态绑定
    - 一个类里面，方法名字相同，而参数不同
    - 方法能够在同一个类中或者在一个子类中被重载

  - 我们用到了三个语法机制来实现多态。

    - ```java
      
      // 编程语言要支持父类对象可以引用子类对象，也就是可以将SortedDynamicArray 传递给 DynamicArray。
      DynamicArray dynamicArray = new SortedDynamicArray();
      
      // 编程语言要支持继承，也就是 SortedDynamicArray 继承了DynamicArray，才能将 SortedDyamicArray 传递给 DynamicArray
      public class SortedDynamicArray extends DynamicArray
      
      
      // 子类可以重写（override）父类中的方法
      
      ```

  - 如何利用接口类来实现多态特性

    - Iterator 是一个接口类，定义了一个可以遍历集合数据的迭代器。Array 和LinkedList 都实现了接口类 Iterator。我们通过传递不同类型的实现类（Array、LinkedList）到 print(Iterator iterator) 函数中，支持动态的调用不同的 next()、hasNext() 实现

  - 多态特性能提高代码的可扩展性和复用性

    - 我们利用多态的特性，仅用一个 print() 函数就可以实现遍历打印不同类型（Array、LinkedList）集合的数据。当再增加一种要遍历打印的类型的时候，比如HashMap，我们只需让 HashMap 实现 Iterator 接口，重新实现自己的 hasNext()、next() 等方法就可以了，完全不需要改动 print() 函数的代码。所以说，多态提高了代码的可扩展性。
    - 利用多态特性，我们只需要实现一个 print() 函数的打印逻辑，就能应对各种集合数据的打印操作，这显然提高了代码的复用性

  - 除此之外，多态也是很多设计模式、设计原则、编程技巧的代码实现基础，比如策略模式、基于接口而非实现编程、依赖倒置原则、里式替换原则、利用多态去掉冗长的 if-else 语句等等。

## 面向对象与面向过程

- 什么是面向过程编程与面向过程编程语言？
  - 面向对象编程是一种编程范式或编程风格。支持类或对象的语法机制，并有现成的语法机制，能方便地实现面向对象编程四大特性（封装、抽象、继承、多态）的编程语言。
  - 面向过程编程语言首先是一种编程语言。它最大的特点是不支持类和对象两个语法概念，不支持丰富的面向对象编程特性（比如继承、多态、封装），仅支持面向过程编程。
    面向过程和面向对象最基本的区别就是，代码的组织方式不同。面向过程风格的代码被组织成了一组
  - 方法集合及其数据结构（struct User），方法和数据结构的定义是分开的。面向对象风格的代码被组织成一组类，方法和数据结构被绑定一起，定义在类中



## JVM是如何执行方法调用的

- 通常来说，之所以不提倡可变长参数方法的重载，是因为 Java 编译器可能无法决定应该调用哪个目标方法。

- **重载**

  - 在 Java 程序里，如果同一个类中出现多个名字相同，并且参数类型相同的方法，那么它无法通过编译。也就是说，在正常情况下，如果我们想要在同一个类中定义名字相同的方法，那么它们的参数类型必须不同。

  - 这个限制可以通过字节码工具绕开。也就是说，在编译完成之后，我们可以再向 class 文件中添加方法名和参数类型相同，而返回类型不同的方法。

  - 重载的方法在编译过程中即可完成识别

  - Java 编译器会根据所传入参数的声明类型（注意与实际类型区分）来选取重载方法。

  - ```java
    // 选取的过程共分为三个阶段：
    // 		不考虑对基本类型自动装拆箱，以及可变长参数的情况下选取重载方法；
    // 		在允许自动装拆箱，但不允许可变长参数的情况下选取重载方法；
    // 		在允许自动装拆箱以及可变长参数的情况下选取重载方法。
    
    // 当传入 null 时，它既可以匹配第一个方法中声明为 Object 的形式参数，也可以匹配第二个方法中声明为 String 的形式参数。由于 String 是 Object 的子类，因此 Java 编译器会认为第二个方法更为贴切。
    
    void invoke(Object obj, Object... args) { ... }
    void invoke(String s, Object obj, Object... args) { ... }
    invoke(null, 1); // 调用第二个 invoke 方法
    invoke(null, 1, 2); // 调用第二个 invoke 方法
    invoke(null, new Object[]{1}); // 只有手动绕开可变长参数的语法糖，
     // 才能调用第一个 invoke 方法
    ```



- **重写**

  - 如果子类定义了与父类中非私有方法同名的方法，而且这两个方法的参数类型相同，那么这两个方法之间又是什么关系呢？
  - 如果这两个方法都是静态的，那么子类中的方法隐藏了父类中的方法。如果这两个方法都不是静态的，且都不是私有的，那么子类的方法重写了父类中的方法。
  - 方法重写，正是多态最重要的一种体现方式：它允许子类在继承父类部分功能的同时，拥有自己独特的行为。
  - 重写调用会根据调用者的动态类型，来选取实际的目标方法。

- **JVM 的静态绑定和动态绑定**

  - Java 虚拟机识别方法的关键在于类名、方法名以及方法描述符（方法描述符，它是由方法的参数类型以及返回类型所构成）
    - 同一个类中，如果同时出现多个名字相同且描述符也相同的方法，那么 Java 虚拟机会在类的验证阶段报错。
  - 由于对重载方法的区分在编译阶段已经完成，我们可以认为 Java 虚拟机不存在重载这一概念
  - Java 虚拟机中关于方法重写的判定同样基于方法描述符。
    - 如果子类定义了与父类中非私有、非静态方法同名的方法，那么只有当这两个方法的参数类型以及返回类型一致，Java 虚拟机才会判定为重写
  - 重载也被称为静态绑定，重写则被称为动态绑定
    - 并非完全正确。重载方法可能被它的子类所重写，因此 Java 编译器会将所有对非私有实例方法的调用编译为需要动态绑定的类型
    - 确切地说。静态绑定指的是在解析时便能够直接识别目标方法的情况，而动态绑定则指的是需要在运行过程中根据调用者的动态类型来识别目标方法的情况。

- **调用指令的符号引用**

  - > 1. invokestatic：用于调用静态方法。
    > 2. invokespecial：用于调用私有实例方法、构造器，以及使用 super 关键字调用父类的实例方法或构造器，和所实现接口的默认方法。
    > 3. invokevirtual：用于调用非私有实例方法。
    > 4. invokeinterface：用于调用接口方法。
    > 5. invokedynamic：用于调用动态方法。

  - 在执行使用了符号引用的字节码前，Java 虚拟机需要解析这些符号引用，并替换为实际引用。

    - 对于可以静态绑定的方法调用而言，实际引用是一个指向方法的指针。
    - 对于需要动态绑定的方法调用而言，实际引用则是一个方法表的索引。

  - Java识别方法是在java代码——>字节流class编译阶段的。

  - Jvm识别方法是在字节码class——>机器码阶段的。也就是加载，链接，初始化

- **虚方法调用**

  - Java 里所有非私有实例方法调用都会被编译成 invokevirtual 指令，而接口方法调用都会被编译成 invokeinterface 指令。这两种指令，均属于 Java 虚拟机中的虚方法调用
  - 在绝大多数情况下，Java 虚拟机需要根据调用者的动态类型，来确定虚方法调用的目标方法
  - 这个过程我们称之为动态绑定。Java 虚拟机中采取了一种用空间换取时间的策略来实现动态绑定，它为每个类生成一张方法表，用以快速定位目标方法

- **方法表**

  - 类的准备阶段，它除了为静态字段分配内存之外，还会构造与该类相关联的方法表。
  - 方法表本质上是一个数组，每个数组元素指向一个当前类及其祖先类中非私有的实例方法
  - 方法表满足两个特质：其一，子类方法表中包含父类方法表中的所有方法；其二，子类方法在方法表中的索引值，与它所重写的父类方法的索引值相同。
  - 在执行过程中，Java 虚拟机将获取调用者的实际类型，并在该实际类型的虚方法表中，根据索引值获得目标方法。这个过程便是动态绑定。

- **我们是否可以认为虚方法调用对性能没有太大影响呢？**

  - 但实际上仅存在于解释执行中，或者即时编译代码的最坏情况中。这是因为即时编译还拥有另外两种性能更好的优化手段：内联缓存（inlining cache）和方法内联（method inlining）
  - 内联缓存
    - 加快动态绑定的优化技术。能够缓存虚方法调用中调用者的动态类型，以及该类型所对应的目标方法。如果没有碰到已缓存的类型，内联缓存则会退化至使用基于方法表的动态绑定
    - 在实践中，大部分的虚方法调用均是单态的，也就是只有一种动态类型。为了节省内存空间，Java 虚拟机只采用单态内联缓存。
  - 在即时编译中，方法内联不仅仅能够消除方法调用的固定开销，而且还增加了进一步优化的可能性







## Object

- 在Java中，只有基本类型（int，boolean等）的值不是对象。其他类型，包括数组类型，不管是对象数组还是基本类型的数组都扩展于Object类
- protected Object clone() 创建并返回此对象的一个副本。 
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

    #### 8、长生命周期的对象持有短生命周期的引用，就很可能会出现内存泄露

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

    

- 太多份相同的对象导致 OOM

  - 我们虽然清楚数据总量，但却忽略了每一份数据在内存中可能有多份
    - 100M 的数据加载到程序内存中，变为 Java 的数据结构就已经占用了 200M 堆内存；这些数据经过 JDBC、MyBatis 等框架其实是加载了 2 份，然后领域模型、DTO 再进行转换可能又加载了 2 次；最终，占用的内存达到了 200M*4=800M。
  - 使用 WeakHashMap 不等于不会 OOM
    - WeakHashMap和 HashMap 的最大区别，是 Entry 对象的实现。
    - Entry 对象继承了 WeakReference，Entry 的构造函数调用了 super (key,queue)，这是父类的构造函数。其中，key 是我们执行 put 方法时的 key；queue 是一个ReferenceQueue，被 GC 的对象会被丢进这个 queue里面。
    - 每次调用 get、put、size 等方法时，都会从 queue 里拿出所有已经被 GC 掉的 key 并删除对应的 Entry 对象
    - WeakHashMap 的 Key 虽然是弱引用，但是其 Value 却持有 Key 中对象的强引用，Value 被 Entry 引用，Entry 被 WeakHashMap 引用，最终导致 Key 无法回收。解决方案就是让 Value 变为弱引用
  - Tomcat 参数配置不合理导致 OOM

- > 我建议你为生产系统的程序配置 JVM 参数启用详细的 GC 日志，方便观察垃圾收集器的行为，并开启HeapDumpOnOutOfMemoryError，以便在出现 OOM 时能自动Dump 留下第一问题现场。
  > XX:+HeapDumpOnOutOfMemoryError 
  > -XX:HeapDumpPath=. 
  > -XX:+PrintGCDateStamps
  > -XX:+PrintGCDetails

  

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

  





## 如何在项目中用上 Lambda 表达式和 Stream 操作？

- Lambda 表达式的初衷是，进一步简化匿名类的语法（不过实现上，Lambda 表达式并不是匿名类的语法糖），使 Java 走向函数式编程。

- 这里有个例子，分别使用匿名类和 Lambda 表达式创建一个线程打印字符串

  - ```java
    new Thread(new Runnable() {
    @Override
    	public void run() {
    		System.out.println(ss);
    }
    }).start();
    
    new Thread(() -> System.out.println("hello2")).start();
    ```

- Lambda 表达式如何匹配 Java 的类型系统呢？

  - 函数式接口是一种只有单一抽象方法的接口，使用 @FunctionalInterface 来描述，可以隐式地转换成 Lambda 表达式。使用 Lambda 表达式来实现函数式接口，不需要提供类名和方法定义，通过一行代码提供函数式接口的实例，就可以让函数成为程序中的头等公民，可以像普通数据一样作为参数传递，而不是作为一个固定的类中的固定方法。

  - ```java
    //Predicate接口是输入一个参数，返回布尔值。我们通过and方法组合两个Predicate条件
    Predicate<Integer> positiveNumber = i -> i > 0; 
    Predicate<Integer> evenNumber = i -> i % 2 == 0; 
    assertTrue(positiveNumber.and(evenNumber).test(2)); 
    
    //Consumer接口是消费一个数据。我们通过andThen方法组合调用两个Consumer，输出两行abcdefg 
    Consumer<String> println = System.out::println; 
    println.andThen(println).accept("abcdefg"); 
    
    //Function接口是输入一个数据，计算后输出一个数据。我们先把字符串转换为大写，然后通过andThen组合调用两个Function
    Function<String, String> upperCase = String::toUpperCase; 
    Function<String, String> duplicate = s -> s.concat(s); 
    assertThat(upperCase.andThen(duplicate).apply("test"), is("TESTTEST"));  
    
    //Supplier是提供一个数据的接口。这里我们实现获取一个随机数
    Supplier<Integer> random = ()->ThreadLocalRandom.current().nextInt(); 
    System.out.println(random.get()); 
    
    //BinaryOperator是输入两个同类型参数，输出一个同类型参数的接口。这里我们通过方法引用获得一个
    BinaryOperator<Integer> add = Integer::sum; 
    BinaryOperator<Integer> subtraction = (a, b) -> a - b; 
    assertThat(subtraction.apply(add.apply(1, 2), 3), is(0));
    ```

  - Predicate、Function 等函数式接口，还使用 default 关键字实现了几个默认方法。这样一来，它们既可以满足函数式接口只有一个抽象方法，又能为接口提供额外的功能

- 使用 Stream 简化集合操作

  - map 方法传入的是一个 Function，可以实现对象转换；
  - filter 方法传入一个 Predicate，实现对象的布尔判断，只保留返回 true 的数据；
  - mapToDouble 用于把对象转换为 double；
  - 通过 average 方法返回一个 OptionalDouble，代表可能包含值也可能不包含值的可空double。

- 使用 Optional 简化判空逻辑

  - 使用 Optional，不仅可以避免使用 Stream 进行级联调用的空指针问题；更重要的是，它提供了一些实用的方法帮我们避免判空逻辑。

- JDK8 结合 Lambda 和 Stream 对各种类的增强

  - 要通过 HashMap 实现一个缓存的操作，在 Java 8 之前我们可能会写出这样的getProductAndCache 方法：先判断缓存中是否有值；如果没有值，就从数据库搜索取值；最后，把数据加入缓存。

  - 而在 Java 8 中，我们利用 ConcurrentHashMap 的 computeIfAbsent 方法，用一行代码就可以实现这样的繁琐操作

  - ```java
    private Map<Long, Product> cache = new ConcurrentHashMap<>();
    private Product getProductAndCacheCool(Long id) { 
    	return cache.computeIfAbsent(id, i -> //当Key不存在的时候提供一个Function来代表
    		Product.getData().stream() // 拿入后存入cache
    			.filter(p -> p.getId().equals(i)) //过滤
    			.findFirst() //找第一个，得到Optional<Product> 
    			.orElse(null)); //如果找不到Product，则使用null 
    }
    ```

  - 并行流

    - 前面我们看到的 Stream 操作都是串行 Stream，操作只是在一个线程中执行，此外 Java 8还提供了并行流的功能：通过 parallel 方法，一键把 Stream 转换为并行操作提交到线程池处理。

      ```java
      IntStream.rangeClosed(1,100).parallel().forEach(xxxx)
      ```

- Stream 操作详解

  - flatMap:通过map把每一个元素转换为一个流，然后把所有流链接到一起扁平化展开

- 创建流

  - ```java
    // 通过 stream 方法把 List 或数组转换为流；
    Arrays.asList("a1", "a2", "a3").stream().forEach(System.out::println); 
    Arrays.stream(new int[]{1, 2, 3}).forEach(System.out::println);
    
    // 通过 Stream.of 方法直接传入多个元素构成一个流；
    String[] arr = {"a", "b", "c"};
    Stream.of(arr).forEach(System.out::println);
    Stream.of("a", "b", "c").forEach(System.out::println);
    Stream.of(1, 2, "a").map(item -> item.getClass().getName()).forEach(System.out::println);
    
    // 通过 Stream.iterate 方法使用迭代的方式构造一个无限流，然后使用 limit 限制流元素个数；
    Stream.iterate(2, item -> item * 2).limit(10).forEach(System.out::println);
    Stream.iterate(BigInteger.ZERO, n -> n.add(BigInteger.TEN)).limit(10).forEach(System.out::println);
    
    // 通过 Stream.generate 方法从外部传入一个提供元素的 Supplier 来构造无限流，然后使用 limit 限制流元素个数；
    Stream.generate(() -> "test").limit(3).forEach(System.out::println);
    Stream.generate(Math::random).limit(10).forEach(System.out::println);     
    
    // 通过 IntStream 或 DoubleStream 构造基本类型的流。
    IntStream.range(1, 3).forEach(System.out::println);
    IntStream.range(0, 3).mapToObj(i -> "x").forEach(System.out::println);
    IntStream.rangeClosed(1, 3).forEach(System.out::println);
    DoubleStream.of(1.1, 2.2, 3.3).forEach(System.out::println);
    
    //各种转换
    System.out.println(IntStream.of(1, 2).toArray().getClass()); //class [I
    System.out.println(Stream.of(1, 2).mapToInt(Integer::intValue).toArray().getClass()); //class [I
    System.out.println(IntStream.of(1, 2).boxed().toArray().getClass()); //class [Ljava.lang.Object;
    System.out.println(IntStream.of(1, 2).asDoubleStream().toArray().getClass()); //class [D
    System.out.println(IntStream.of(1, 2).asLongStream().toArray().getClass()); //class [J
    
    Arrays.asList("a", "b", "c").stream()   // Stream<String>
        .mapToInt(String::length)       // IntStream
        .asLongStream()                 // LongStream
        .mapToDouble(x -> x / 10.0)     // DoubleStream
        .boxed()                        // Stream<Double>
        .mapToLong(x -> 1L)             // LongStream
        .mapToObj(x -> "")              // Stream<String>
        .collect(Collectors.toList());
    }        
    ```

  - collect

    - toCollection 把流中的元素收集成指定的集合

    - joining 链接流中元素toString后的字符串

    - groupBy 根据元素的一个属性值对元素分组，属性值作为Key

    - partitionBy 用于分区，分区是特殊的分组，只有 true 和 false 两组。比如，我们把用户按照是否下单进行分区，给 partitioningBy 方法传入一个 Predicate 作为数据分区的区分，输出是 Map<Boolean, List>

      - ```java
        //根据是否有下单记录进行分区
        System.out.println(Customer.getData().stream().collect(
        	partitioningBy(customer -> orders.stream().mapToLong(Order::getCustomerId)
                                .anyMatch(id -> id == customer.getId()))));
        ```

  - 转化

    - ```java
      public class Main {
      
          public static void main(String[] args) {
      
              int[] data = {4, 5, 3, 6, 2, 5, 1};
              
              // int[] 转 List<Integer>
              List<Integer> list1 = Arrays.stream(data).boxed().collect(Collectors.toList());
      
              // Arrays.stream(arr) 可以替换成IntStream.of(arr)。
              // 1.使用Arrays.stream将int[]转换成IntStream。
              // 2.使用IntStream中的boxed()装箱。将IntStream转换成Stream<Integer>。
              // 3.使用Stream的collect()，将Stream<T>转换成List<T>，因此正是List<Integer>。
      
              // int[] 转 Integer[]
              Integer[] integers1 = Arrays.stream(data).boxed().toArray(Integer[]::new);
      
              // 前两步同上，此时是Stream<Integer>。
      
              // 然后使用Stream的toArray，传入IntFunction<A[]> generator。
      
              // 这样就可以返回Integer数组。
      
              // 不然默认是Object[]。
      
       
      
              // List<Integer> 转 Integer[]
      
              Integer[] integers2 = list1.toArray(new Integer[0]);
      
              //  调用toArray。传入参数T[] a。这种用法是目前推荐的。
      
              // List<String>转String[]也同理。
      
       
      
              // List<Integer> 转 int[]
      
              int[] arr1 = list1.stream().mapToInt(Integer::valueOf).toArray();
      
              // 想要转换成int[]类型，就得先转成IntStream。
      
              // 这里就通过mapToInt()把Stream<Integer>调用Integer::valueOf来转成IntStream
      
              // 而IntStream中默认toArray()转成int[]。
      
       
      
              // Integer[] 转 int[]
      
              int[] arr2 = Arrays.stream(integers1).mapToInt(Integer::valueOf).toArray();
      
              // 思路同上。先将Integer[]转成Stream<Integer>，再转成IntStream。
      
       
      
              // Integer[] 转 List<Integer>
      
              List<Integer> list2 = Arrays.asList(integers1);
      
              // 最简单的方式。String[]转List<String>也同理。
      
       
      
              // 同理
      
              String[] strings1 = {"a", "b", "c"};
      
              // String[] 转 List<String>
      
              List<String> list3 = Arrays.asList(strings1);
      
              // List<String> 转 String[]
      
              String[] strings2 = list3.toArray(new String[0]);
      
       
      
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

- - 