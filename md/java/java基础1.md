

## 反射

- 反射最大的作用之一就在于我们可以不在编译时知道某个对象的类型，而在运行时通过提供完整的”包名+类名.class”得到。注意：不是在编译时，而是在运行时。

- 功能：

  > •在运行时能判断任意一个对象所属的类。
  > •在运行时能构造任意一个类的对象。
  > •在运行时判断任意一个类所具有的成员变量和方法。
  > •在运行时调用任意一个对象的方法。
  > 说大白话就是，利用Java反射机制我们可以加载一个运行时才得知名称的class，获悉其构造方法，并生成其对象实体，能对其fields设值并唤起其methods。

- 反射的主要应用场景有：

  - 开发通用框架 - 反射最重要的用途就是开发各种通用框架。很多框架（比如 Spring）都是配置化的（比如通过 XML 文件配置 JavaBean、Filter 等），为了保证框架的通用性，它们可能需要根据配置文件加载不同的对象或类，调用不同的方法，这个时候就必须用到反射——运行时动态加载需要加载的对象。
  - 动态代理 - 在切面编程（AOP）中，需要拦截特定的方法，通常，会选择动态代理方式。这时，就需要反射技术来实现了。
  - 注解 - 注解本身仅仅是起到标记作用，它需要利用反射机制，根据注解标记去调用注解解释器，执行行为。如果没有反射机制，注解并不比注释更有用。
  - 可扩展性功能 - 应用程序可以通过使用完全限定名称创建可扩展性对象实例来使用外部的用户定义类。

- ### 反射的缺点

  - **性能开销** - 由于反射涉及动态解析的类型，因此无法执行某些 Java 虚拟机优化。因此，反射操作的性能要比非反射操作的性能要差，应该在性能敏感的应用程序中频繁调用的代码段中避免。
  - **破坏封装性** - 反射调用方法时可以忽略权限检查，因此可能会破坏封装性而导致安全问题。
  - **内部曝光** - 由于反射允许代码执行在非反射代码中非法的操作，例如访问私有字段和方法，所以反射的使用可能会导致意想不到的副作用，这可能会导致代码功能失常并可能破坏可移植性。反射代码打破了抽象，因此可能会随着平台的升级而改变行为。

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

- JVM基本原理1——JVM是如何实现反射的



动态代理是反射的一个非常重要的应用场景。动态代理常被用于一些 Java 框架中。例如 Spring 的 AOP ，Dubbo 的 SPI 接口，就是基于 Java 动态代理实现的。





## 动态代理

- 动态代理是反射的一个非常重要的应用场景。动态代理常被用于一些 Java 框架中。例如 Spring 的 AOP ，Dubbo 的 SPI 接口，就是基于 Java 动态代理实现的。

- 动态代理的原理解析

  - 所谓动态代理（Dynamic Proxy），就是我们不事先为每个原始类编写代理类，而是在运行的时候，动态地创建原始类对应的代理类，然后在系统中用代理类替换掉原始类。那如何实现动态代理呢？
  - 动态代理中所说的"动态",是针对使用Java代码实际编写了代理类的"静态"代理而言的,它的优势不在于省去了编写代理类那一点编码工作量,而是实现了可以在原始类和接口还未知的时候,就确定了代理类的行为,当代理类与原始类脱离直接联系后,就可以很灵活的重用于不同的应用场景之中

- 静态代理的实现比较简单：编写一个代理类，实现与目标对象相同的接口，并在内部维护一个目标对象的引用。通过构造器塞入目标对象，在代理对象中调用目标对象的同名方法，并添加前拦截，后拦截等所需的业务功能。

- 动态代理步骤：

  1. 获取 RealSubject 上的所有接口列表；
  2. 确定要生成的代理类的类名，默认为：com.sun.proxy.$ProxyXXXX；
  3. 根据需要实现的接口信息，在代码中动态创建 该 Proxy 类的字节码；
  4. 将对应的字节码转换为对应的 class 对象；
  5. 创建 InvocationHandler 实例 handler，用来处理 Proxy 所有方法调用；
  6. Proxy 的 class 对象 以创建的 handler 对象为参数，实例化一个 proxy 对象。

- 要得到一个类的实例，关键是先得到该类的Class对象

  - 我们无法根据接口直接创建对象
  - Proxy.getProxyClass()：返回代理类的Class对象。
    - 只要传入目标类实现的接口的Class对象，getProxyClass()方法即可返回代理Class对象，而不用实际编写代理类。
    - 不过实际编程中，一般不用getProxyClass()，而是使用Proxy类的另一个静态方法：Proxy.newProxyInstance()，直接返回代理实例，连中间得到代理Class对象的过程都帮你隐藏

- Proxy.newProxyInstance（）

  - 直接返回代理对象，而不是代理对象Class

  - InvocationHandler对象成了代理对象和目标对象的桥梁

  - ```java
    public class ProxyTest {
    	public static void main(String[] args) throws Throwable {
    		CalculatorImpl target = new CalculatorImpl();
    		Calculator calculatorProxy = (Calculator) getProxy(target);
    		calculatorProxy.add(1, 2);
    		calculatorProxy.subtract(2, 1);
    	}
    
    	private static Object getProxy(final Object target) throws Exception {
    		Object proxy = Proxy.newProxyInstance(
    				target.getClass().getClassLoader(),/*类加载器*/
    				target.getClass().getInterfaces(),/*让代理对象和目标对象实现相同接口*/
    				new InvocationHandler(){/*代理对象的方法最终都会被JVM导向它的invoke方法*/
                        // Object proxy：是代理对象本身，而不是目标对象（不要调用，会无限递归）
    					// Method method：本次被调用的代理对象的方法
    					// Obeject[] args：本次被调用的代理对象的方法参数
    					public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    						System.out.println(method.getName() + "方法开始执行...");
    						Object result = method.invoke(target, args);
    						System.out.println(result);
    						System.out.println(method.getName() + "方法执行结束...");
    						return result;
    					}
    				}
    		);
    		return proxy;
    	}
    }
    ```

- Cglib实现动态代理

  - 概念

    - jdk动态代理生成的代理类是继承自Proxy，实现你的被代理类所实现的接口，要求必须有接口。
    - cglib动态代理生成的代理类是被代理者的子类，并且会重写父类的所有方法，要求该父类必须有空的构造方法,否则会报错:Superclass has no null constructors but no arguments were given,还有，private和final修饰的方法不会被子类重写。
    - CGLIB 代理的生成原理是生成目标类的子类，而子类是增强过的，这个子类对象就是代理对象。所以，使用CGLIB 生成动态代理，要求目标类必须能够被继承，即不能是 final 的类。

  - 代理类要实现cglib包下的MethodInterceptor接口，从而实现对intercept的调用。

    - ```java
      // 需要实现MethodInterceptor, 当前这个类的对象就是一个回调对象
      // MyCglibFactory 是 类A，它调用了Enhancer(类B)的方法: setCallback(this)，而且将类A对象传给了类B
      // 而类A 的 方法intercept会被类B的 setCallback调用，这就是回调设计模式
      
      // 我们可以从上面的代码示例中看到，代理类是由enhancer.create()创建的。Enhancer是CGLIB的字节码增强器，可以很方便的对类进行拓展。
      
      // 创建代理类的过程：
      //   1. 生成代理类的二进制字节码文件；
      //   2. 加载二进制字节码，生成Class对象；
      //   3. 通过反射机制获得实例构造，并创建代理类对象。
      
      
      public Object getInstance(Object object){
              Enhancer enhancer = new Enhancer();
              enhancer.setSuperclass(object.getClass());
              System.out.println("生成代理对象前对象是:"+object.getClass());
              enhancer.setCallback(this);
              return enhancer.create();
          }
       
          @Override
          public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
              System.out.println("代理中...");
              methodProxy.invokeSuper(o, objects);
      //        methodProxy.invoke(o, objects);
              System.out.println("代理处理完毕,OK,请查收");
              return null;
          }
      }
      ```







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

  

  



## 抽象类和接口

- 两者都是抽象类，都不能实例化。
- 使用抽象类是为了代码的复用，而使用接口的动机是为了实现多态性
- interface是完全抽象的，只能声明pulic的方法，实现类必须要实现不能定义方法体，也不能声明实例变量。abstract class的子类可以有选择地实现，abstract可以声明常量变量。
- abstract class的子类在继承它时，对非抽象方法既可以直接继承，也可以覆盖；而对抽象方法，可以选择实现，也可以通过再次声明其方法为抽象的方式，无需实现，留给其子类来实现，但此类必须也声明为抽象类。



## RESTful

- URL定位资源，用HTTP动词（GET,POST,DELETE,DETC）描述操作。
- URL中只使用名词来指定资源
- 这个风格太理想化了
  - REST要求要将接口以资源的形式呈现
  - 我们之所以要定义接口，本身的动机是做一个抽象，把复杂性隐藏起来，而绝对不是把内部的实现细节给暴露出去。REST却反其道而行之，要求实现应该是“资源”并且这个实现细节要暴露在接口的形式上。
  - 有些时候，用REST完成CRUD已经能完成任务了。此时，用REST没有什么不好的。但是，现实当中，真正的业务领域一般都会比资源的CRUD复杂的多。这时REST“基本上没解决太多实际问题”的缺点就会体现出来。



## Java 到底是值传递还是引用传递

- 实参与形参

  - ```java
    public static void main(String[] args) {
        ParamTest pt = new ParamTest();
        pt.sout("Hollis");//实际参数为 Hollis
    }
    
    public void sout(String name) { //形式参数为 name
        System.out.println(name);
    }
    
    ```

- 值传递与引用传递

  - 值传递（pass by value）是指在调用函数时将实际参数复制一份传递到函数中，这样在函数中如果对参数进行修改，将不会影响到实际参数。

  - 引用传递（pass by reference）是指在调用函数时将实际参数的地址直接传递到函数中，那么在函数中对参数所进行的修改，将影响到实际参数。

  - 值传递和引用传递的区别并不是传递的内容。而是实参到底有没有被复制一份给形参。

  - Java中的传递，是值传递，而这个值，实际上是对象的引用。

  - ```java
    // 当我们在main中创建一个User对象的时候，在堆中开辟一块内存，其中保存了name和gender等数据。然后hollis持有该内存的地址0x123456。
    // 当尝试调用pass方法，并且hollis作为实际参数传递给形式参数user的时候，会把这个地址0x123456交给user，这时，user也指向了这个地址。
    // 然后在pass方法内对参数进行修改的时候，即user = new User();，会重新开辟一块0X456789的内存，赋值给user。后面对user的任何修改都不会改变内存0X123456的内容。
    
    public static void main(String[] args) {
        ParamTest pt = new ParamTest();
        User hollis = new User();
        hollis.setName("Hollis");
        hollis.setGender("Male");
        pt.pass(hollis);
        System.out.println("print in main , user is " + hollis);
    }
    
    public void pass(User user) {
        user = new User();
        user.setName("hollischuang");
        user.setGender("Male");
        System.out.println("print in pass , user is " + user);
    }
    
    // 上面这种传递是什么传递？肯定不是引用传递，如果是引用传递的话，在user=new User()的时候，实际参数的引用也应该改为指向new User()的地址，但是实际上并没有。
    
    // 也能知道，这里是把实际参数的引用的地址复制了一份，传递给了形式参数。所以，上面的参数其实是值传递，把实参对象引用的地址当做值传递给了形式参数。
    ```

- 局部变量/方法参数

  - 局部变量和方法参数在jvm中的储存方法是相同的，都是在栈上开辟空间来储存的，随着进入方法开辟，退出方法回收
  - 以32位JVM为例，boolean/byte/short/char/int/float以及引用都是分配4字节空间，long/double分配8字节空间。对于每个方法来说，最多占用多少空间是一定的，这在编译时就可以计算好。
  - 当我们在方法中声明一个 int i = 0，或者 Object obj = null 时，仅仅涉及stack，不影响到heap，当我们 new Object() 时，会在heap中开辟一段内存并初始化Object对象。当我们将这个对象赋予obj变量时，仅仅是stack中代表obj的那4个字节变更为这个对象的地址。

  





## JDK中注解的底层实现

- 我们先定义一个十分简单的Counter注解

  - ```java
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Target(ElementType.TYPE)
    public @interface Counter {
    
        int count() default 0;
    }
    ```

  - @Counter实例从Debug过程中观察发现是JDK的一个代理类（并且InvocationHandler的实例是sun.reflect.annotation.AnnotationInvocationHandler，它是一个修饰符为default的sun包内可用的类）

- @Counter反编译后的字节码

  - 注解是一个接口，它继承自java.lang.annotation.Annotation父接口。
  - @Counter对应的接口接口除了继承了java.lang.annotation.Annotation中的抽象方法，自身定义了一个抽象方法public abstract int count();。
  - 既然注解最后转化为一个接口，注解中定义的注解成员属性会转化为抽象方法，那么最后这些注解成员属性怎么进行赋值的呢？
  - 直接点说就是：Java通过动态代理的方式生成了一个实现了"注解对应接口"的实例，该代理类实例实现了"注解成员属性对应的方法"，这个步骤类似于"注解成员属性"的赋值过程，这样子就可以在程序运行的时候通过反射获取到注解的成员属性（这里注解必须是运行时可见的，也就是使用了@Retention(RetentionPolicy.RUNTIME)

- 注解的最底层实现就是一个JDK的动态代理类

  - AnnotationInvocationHandler的成员变量Map<String, Object> memberValues存放着注解的成员属性的名称和值的映射，注解成员属性的名称实际上就对应着接口中抽象方法的名称，例如上面我们定义的@Counter注解生成代理类后，它的AnnotationInvocationHandler实例中的memberValues属性存放着键值对count=1。
  - 既然知道了注解底层使用了JDK原生的Proxy，那么我们可以直接输出代理类到指定目录去分析代理类的源码
  - 其中$Proxy0是@Retention注解对应的动态代理类，而$Proxy1才是我们的@Counter对应的动态代理类，当然如果有更多的注解，那么有可能生成$ProxyN。显然，$Proxy1实现了Counter接口，它在代码的最后部分使用了静态代码块实例化了成员方法的Method实例
  - 我们在分析AnnotationInvocationHandler的时候看到，它只用到了Method的名称从Map从匹配出成员方法的结果，效率近似于通过Key从Map实例中获取Value一样，是十分高效的。





## Javassist

- Javassist是用于处理Java字节码的类库。Java字节码存储在称为class文件的二进制文件中。每个class文件包含一个Java类或接口。

- Javassist.CtClass是类文件的抽象表示，一个CtClass（编译时类）对象是处理一类文件的句柄。

  - ```java
    // 该程序首先获得一个ClassPool对象，该对象使用Javassist控制字节码的修改
    // 该ClassPool对象是CtClass 表示类文件的 对象的容器。它按需读取用于构造CtClass对象的类文件，并记录构造的对象以响应以后的访问。
    // 要修改类的定义，用户必须首先从ClassPool对象get()获得对CtClass表示该类的对象的引用。
    ClassPool pool = ClassPool.getDefault();
    CtClass cc = pool.get("test.Rectangle");
    cc.setSuperclass(pool.get("test.Point"));
    cc.writeFile();
    
    
    // toClass()请求当前线程的上下文类加载器加载由表示的类文件CtClass。它返回一个java.lang.Class代表加载的类的对象。
    Class clazz = cc.toClass();
    
    ```

    





## 设计模式在外卖营销业务中的实践

- 设计模式与领域驱动设计

  - 将业务需求映射为领域上下文以及上下文间的映射关系
  - 站在业务建模的立场上，DDD的模式解决的是如何进行领域建模。而站在代码实践的立场上，设计模式主要关注于代码的设计与实现。既然本质都是模式，那么它们天然就具有一定的共通之处。

- “邀请下单”业务中设计模式的实践

  - 邀请下单后台主要涉及两个技术要点
    - 返奖金额的计算，涉及到不同的计算规则。
    - 从邀请开始到返奖结束的整个流程。
  - 业务建模
    - 新用户
      - 奖励金额
    - 老用户
      - 奖励金额

- 工厂模式和策略模式的实际应用

  - 我们可以使用工厂模式生产出不同的策略，同时使用策略模式来进行不同的策略执行。首先确定我们需要生成出n种不同的返奖策略

  - 我们可以使用工厂模式生产出不同的策略，同时使用策略模式来进行不同的策略执行。首先确定我们需要生成出n种不同的返奖策略

    - ```java
      //抽象策略
      public abstract class RewardStrategy {
          public abstract void reward(long userId);
        
          public void insertRewardAndSettlement(long userId, int reward) {} ; //更新用户信息以及结算
      }
      //新用户返奖具体策略A
      public class newUserRewardStrategyA extends RewardStrategy {
          @Override
          public void reward(long userId) {}  //具体的计算逻辑，...
      }
      
      //老用户返奖具体策略A
      public class OldUserRewardStrategyA extends RewardStrategy {
          @Override
          public void reward(long userId) {}  //具体的计算逻辑，...
      }
      
      //抽象工厂
      public abstract class StrategyFactory<T> {
          abstract RewardStrategy createStrategy(Class<T> c);
      }
      
      //具体工厂创建具体的策略
      public class FactorRewardStrategyFactory extends StrategyFactory {
          @Override
          RewardStrategy createStrategy(Class c) {
              RewardStrategy product = null;
              try {
                  product = (RewardStrategy) Class.forName(c.getName()).newInstance();
              } catch (Exception e) {}
              return product;
          }
      }
      ```

  - 通过工厂模式生产出具体的策略之后，很容易就可以想到使用策略模式来执行我们的策略

    - ```java
      public class RewardContext {
          private RewardStrategy strategy;
      
          public RewardContext(RewardStrategy strategy) {
              this.strategy = strategy;
          }
      
          public void doStrategy(long userId) { 
              int rewardMoney = strategy.reward(userId);
              insertRewardAndSettlement(long userId, int reward) {
                insertReward(userId, rewardMoney);
                settlement(userId);
             }  
          }
      }
      ```

  - 接下来我们将工厂模式和策略模式结合在一起，就完成了整个返奖的过程

    - ```java
      public class InviteRewardImpl {
          //返奖主流程
          public void sendReward(long userId) {
              FactorRewardStrategyFactory strategyFactory = new FactorRewardStrategyFactory();  //创建工厂
              Invitee invitee = getInviteeByUserId(userId);  //根据用户id查询用户信息
              if (invitee.userType == UserTypeEnum.NEW_USER) {  //新用户返奖策略
                  NewUserBasicReward newUserBasicReward = (NewUserBasicReward) strategyFactory.createStrategy(NewUserBasicReward.class);
                  RewardContext rewardContext = new RewardContext(newUserBasicReward);
                  rewardContext.doStrategy(userId); //执行返奖策略
              }if(invitee.userType == UserTypeEnum.OLD_USER){}  //老用户返奖策略，... 
          }
      }
      ```

  - 点评外卖投放系统中设计模式的实践

    - 点评App的外卖频道中会预留多个资源位为营销使用，向用户展示一些比较精品美味的外卖食品，为了增加用户点外卖的意向。当用户点击点评首页的“美团外卖”入口时，资源位开始加载，会通过一些规则来筛选出合适的展示Banner。

    - 我们资源的过滤规则相对灵活多变，这里体现为三点：

      - 过滤规则大部分可重用，但也会有扩展和变更。
      - 不同资源位的过滤规则和过滤顺序是不同的。
      - 同一个资源位由于业务所处的不同阶段，过滤规则可能不同。

    - 责任链模式最重要的优点就是解耦，将客户端与处理者分开，客户端不需要了解是哪个处理者对事件进行处理，处理者也不需要知道处理的整个流程。在我们的系统中，后台的过滤规则会经常变动，规则和规则之间可能也会存在传递关系，通过责任链模式，我们将规则与规则分开，将规则与规则之间的传递关系通过Spring注入到List中，形成一个链的关系。当增加一个规则时，只需要实现BasicRule接口，然后将新增的规则按照顺序加入Spring中即可。当删除时，只需删除相关规则即可，不需要考虑代码的其他逻辑。

      - ```java
        //定义一个抽象的规则
        
        public abstract class BasicRule<CORE_ITEM, T extends RuleContext<CORE_ITEM>>{
            //有两个方法，evaluate用于判断是否经过规则执行，execute用于执行具体的规则内容。
            public abstract boolean evaluate(T context);
            public abstract void execute(T context) {
        }
        
        //定义所有的规则具体实现
            
        //规则1：判断服务可用性
        public class ServiceAvailableRule extends BasicRule<UserPortrait, UserPortraitRuleContext> {}
        
        //规则2：判断当前用户属性是否符合当前资源位投放的用户属性要求
        public class UserGroupRule extends BasicRule<UserPortrait, UserPortraitRuleContext> {}
          
        //规则3：判断当前用户是否在投放城市
        public class CityInfoRule extends BasicRule<UserPortrait, UserPortraitRuleContext> {}
        //规则4：根据用户的活跃度进行资源过滤
        public class UserPortraitRule extends BasicRule<UserPortrait, UserPortraitRuleContext> {} 
        
        //我们通过spring将这些规则串起来组成一个一个请求链
            
            <bean name="serviceAvailableRule" class="com.dianping.takeaway.ServiceAvailableRule"/>
            <bean name="userGroupValidRule" class="com.dianping.takeaway.UserGroupRule"/>
            <bean name="cityInfoValidRule" class="com.dianping.takeaway.CityInfoRule"/>
            <bean name="userPortraitRule" class="com.dianping.takeaway.UserPortraitRule"/>
              
            <util:list id="userPortraitRuleChain" value-type="com.dianping.takeaway.Rule">
                <ref bean="serviceAvailableRule"/>
                <ref bean="userGroupValidRule"/>
                <ref bean="cityInfoValidRule"/>
                <ref bean="userPortraitRule"/>
            </util:list>
              
        //规则执行
        public class DefaultRuleEngine{
            @Autowired
            List<BasicRule> userPortraitRuleChain;
        
            public void invokeAll(RuleContext ruleContext) {
                for(Rule rule : userPortraitRuleChain) {
                    rule.evaluate(ruleContext)
                }
            }
        }
        ```

        