

## 反射、泛型

- 反射调用方法不是以传参决定重载

  - 反射的功能包括，在运行时动态获取类和类成员定义，以及动态读取属性调用方法。也就是说，针对类动态调用方法，不管类中字段和方法怎么变动，我们都可以用相同的规则来读取信息和执行方法。因此，几乎所有的 ORM（对象关系映射）、对象映射、MVC 框架都使用了反射。
  - getDeclaredMethod 传入的参数类型 Integer.TYPE 代表的是 int，所以实际执行方法时，无论传的是包装类型还是基本类型，都会调用 int 入参的 age 方法。
  - 把 Integer.TYPE 改为 Integer.class，执行的参数类型就是包装类型的 Integer。这时，无论传入的是 Integer.valueOf(“36”) 还是基本类型的 36

- 泛型经过类型擦除多出桥接方法的坑

  - 子类没有指定 String 泛型参数，父类的泛型方法 setValue(T value) 在泛型擦除后是 setValue(Object value)，子类中入参是 String 的 setValue 方法被当作了新方法；

  - 使用 javap 命令来反编译编译后的 Child类的 class 字节码

    - 如果子类 Child 的 setValue 方法要覆盖父类的 setValue 方法，那 入参也必须是 Object。所以，编译器会为我们生成一个所谓的 bridge 桥接方法

    - 入参为 Object 的 setValue 方法在内部调用了入参为 String 的 setValue 方法

    - 如果编译器没有帮我们实现这个桥接方法，那么 Child 子类重写的是父类经过泛型类型擦除后、入参是 Object 的 setValue 方法。这两个方法的参数，一个是 String 一个是 Object，明显不符合 Java 的语义

    - 使用 jclasslib 工具打开 Child类，同样可以看到入参为 Object 的桥接方法上标记了public + synthetic + bridge 三个属性。synthetic 代表由编译器生成的不可见代码，bridge 代表这是泛型类型擦除后生成的桥接代码

    - 通过 getDeclaredMethods 方法获取到所有方法后，必须同时根据方法名 setValue 和 

      非 isBridge 两个条件过滤

  - getMethods 和 getDeclaredMethods 是有区别的，前者可以查询到父类方法，后者只能查询到当前类。

- 泛型类型擦除后会生成一个 bridge 方法，这个方法同时又是 synthetic 方法。除了泛型类型擦除，你知道还有什么情况编译器会生成 synthetic 方法吗？

  - 编译器通过生成一些在源代码中不存在的synthetic方法和类的方式，实现了对private级别的字段和类的访问，从而绕开了语言限制
  - 如果同时用到了Enum和switch，如先定义一个enum枚举，然后用switch遍历这个枚举，java编译器会偷偷生成一个synthetic的数组，数组内容是enum的实例。

- @Controller

  - 子类XXXController 继承了YYYController 连带requestMapping 也设为一样，requestMapping 可以理解为请求派发的命名空间，相同的空间里，子类享有父类protected及其以上的访问权限，当容器启动做方法请求映射时，就会发现父类的一般请求方法，全部都没有明确派发地址了。



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

  

## Java对象的内存布局（new）

- **以 new 语句为例，它编译而成的字节码将包含用来请求内存的 new 指令，以及用来调用构造器的invokespecial 指令。**
- 构造器
  - 如果一个类没有定义任何构造器的话， Java 编译器会自动添加一个无参数的构造器。
  - 子类的构造器需要调用父类的构造器。如果父类存在无参数构造器的话，该调用可以是隐式的，也就是说 Java 编译器会自动添加对父类构造器的调用
  - 如果父类没有无参数构造器，那么子类的构造器则需要显式地调用父类带参数的构造器。
  - 通过 new 指令新建出来的对象，它的内存其实涵盖了所有父类中的实例字段。
  - 子类的实例还是会为这些父类实例字段分配内存的。
- 压缩指针
  - 在 Java 虚拟机中，每个 Java 对象都有一个对象头（object header）
    - 由标记字段和类型指针所构成
    - 标记字段用以存储 Java 虚拟机有关该对象的运行数据，如哈希码、GC 信息以及锁信息，而类型指针则指向该对象的类。
    - 为了尽量较少对象的内存使用量，64 位 Java 虚拟机引入了压缩指针的概念，将堆中原本 64 位的 Java 对象指针压缩成 32 位的，32 位压缩指针最多可以标记 2 的 32 次。这样一来，对象头中的类型指针也会被压缩成 32 位，使得对象头的大小从 16 字节降至 12 字节。
  - 默认情况下，Java 虚拟机堆中对象的起始地址需要对齐至 8 的倍数。如果一个对象用不到 8N 个字节，那么空白的那部分空间就浪费掉了。这些浪费掉的空间我们称之为对象间的填充
    - 内存对齐不仅存在于对象与对象之间，也存在于对象中的字段之间。
    - 字段内存对齐的其中一个原因，是让字段只出现在同一 CPU 的缓存行中。如果字段不是对齐的， 那么就有可能出现跨缓存行的字段。也就是说，该字段的读取可能需要替换两个缓存行，而该字段 的存储也会同时污染两个缓存行。
- 字段重排列
  - java 虚拟机重新分配字段的先后顺序，以达到内存对齐的目的
  - Java 8 还引入了一个新的注释 @Contended，用来解决对象字段之间的伪共享问题 。这个注释也会影响到字段的排列。Java 虚拟机会让不同的 @Contended 字段处于独立的缓存行中，因此你会看到大量的空间被浪费掉。
  - 伪共享
    - 假设两个线程分别访问同一对象中不同的 volatile 字段，逻辑上它们并没有共享内容，因此不需要同步。
    - 如果这两个字段恰好在同一个缓存行中，那么对这些字段的写操作会导致缓存行的写回，也就造成了实质上的共享。







## 反射

- 反射最大的作用之一就在于我们可以不在编译时知道某个对象的类型，而在运行时通过提供完整的”包名+类名.class”得到。注意：不是在编译时，而是在运行时。

- 允许正在运行的 Java 程序观测，甚至是修改程序的动态行为。

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

动态代理是反射的一个非常重要的应用场景。动态代理常被用于一些 Java 框架中。例如 Spring 的 AOP ，Dubbo 的 SPI 接口，就是基于 Java 动态代理实现的。

## JVM是如何实现反射的

- 反射调用的实现

  - 方法的反射调用，也就是 Method.invoke

    - 实际上委派给 MethodAccessor 来处理。MethodAccessor 是一个接口，它有两个已有的具体实现：一个通过本地方法来实现反射调用，另一个则使用了委派模式。
    - 每个 Method 实例的第一次反射调用都会生成一个委派实现，它所委派的具体实现便是一个本地实现。

  - Java 的反射调用机制还设立了另一种动态生成字节码的实现（下称动态实现），直接使用invoke 指令来调用目标方法。之所以采用委派实现，便是为了能够在本地实现以及动态实现中切换。

  - 考虑到许多反射调用仅会执行一次，Java 虚拟机设置了一个阈值15	

    - 动态实现无需经过 Java 到 C++再到 Java 的切换，但由于生成字节码十分耗时，仅调用一次的话，反而是本地实现要快上 3 到 4倍

    - 当某个反射调用的调用次数在 15 之下时，采用本地实现；当达到 15 时，便开始动态生成字节码，并将委派实现的委派对象切换至动态实现，这个过程我们称之为 Infation。

    - 可以关闭反射调用的 Infation 机制，从而取消委派实现，并且直接使用动态实现

    - 此外，每次反射调用都会检查目标方法的权限，而这个检查同样可以在 Java 代码里关闭

    - > -Dsun.refect.noInfation=true
      >
      > method.setAccessible(true);

- 反射调用的开销

  - 以 getMethod 为代表的查找方法操作，会返回查找得到结果的一份拷贝。因此，我们应当避免在热点代码中使用返回 Method 数组的 getMethods 或者 getDeclaredMethods 方法，以减少不必要的堆空间消耗。
  - 在实践中，我们往往会在应用程序中缓存 Class.forName 和 Class.getMethod 的结果。

- 反射调用之前字节码都做了什么

  - 由于 Method.invoke 是一个变长参数方法，在字节码层面它的最后一个参数会是 Object 数组。Java 编译器会在方法调用处生成一个长度为传入参数 数量的 Object 数组，并将传入参数一一存储进该数组中。
    - 由于 Object 数组不能存储基本类型，Java 编译器会对传入的基本类型参数进行自动装箱
    - 这两个操作除了带来性能开销外，还可能占用堆内存，使得 GC 更加频繁。
    - 如果一个对象不逃逸，那么即时编译器可以选择栈分配甚至是虚拟分配，也就是不占用堆空间。
  - 之所以反射调用能够变得这么快，主要是因为即时编译器中的方法内联。在关闭了 Infation 的情况下，内联的瓶颈在于 Method.invoke 方法中对 MethodAccessor.invoke 方法的调用。
  - 在生产环境中，我们往往拥有多个不同的反射调用，对应多个 GeneratedMethodAccessor，也就是动态实现。
    - Java 虚拟机的关于上述调用点的调用者的具体类型（ invokevirtual 或者invokeinterface），Java 虚拟机会记录下调用者的具体类型，我们称之为类型 profle）无法同时记录这么多个类，因此可能造成所测试的反射调用没有被内联的情况。
    - 之所以这么慢，除了没有内联之外，另外一个原因是逃逸分析不再起效
    - 只要没有完全内联，就会将看似不逃逸的对象通过参数传递出去。即时编译器不知道所调用的方法对该对象有没有副作用，所以会将其判定为逃逸。

- 方法的反射调用会带来不少性能开销，原因主要有三个：变长参数方法导致的 Object 数组，基本类型的自动装箱、拆箱，还有最重要的方法内联。







## 动态代理

- 动态代理是反射的一个非常重要的应用场景。动态代理常被用于一些 Java 框架中。例如 Spring 的 AOP ，Dubbo 的 SPI 接口，就是基于 Java 动态代理实现的。

- 动态代理的原理解析

  - 所谓动态代理（Dynamic Proxy），就是我们不事先为每个原始类编写代理类，而是在运行的时候，动态地创建原始类对应的代理类，然后在系统中用代理类替换掉原始类。那如何实现动态代理呢？
  - 动态代理中所说的"动态",是针对使用Java代码实际编写了代理类的"静态"代理而言的,它的优势不在于省去了编写代理类那一点编码工作量,而是实现了可以在原始类和接口还未知的时候,就确定了代理类的行为,当代理类与原始类脱离直接联系后,就可以很灵活的重用于不同的应用场景之中

- **静态代理的实现比较简单**：编写一个代理类，实现与目标对象相同的接口，并在内部维护一个目标对象的引用。通过构造器塞入目标对象，在代理对象中调用目标对象的同名方法，并添加前拦截，后拦截等所需的业务功能。

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





## RESTful

- URL定位资源，用HTTP动词（GET,POST,DELETE,DETC）描述操作。
- URL中只使用名词来指定资源
- 这个风格太理想化了
  - REST要求要将接口以资源的形式呈现
  - 我们之所以要定义接口，本身的动机是做一个抽象，把复杂性隐藏起来，而绝对不是把内部的实现细节给暴露出去。REST却反其道而行之，要求实现应该是“资源”并且这个实现细节要暴露在接口的形式上。
  - 有些时候，用REST完成CRUD已经能完成任务了。此时，用REST没有什么不好的。但是，现实当中，真正的业务领域一般都会比资源的CRUD复杂的多。这时REST“基本上没解决太多实际问题”的缺点就会体现出来。



## 浅拷贝和深拷贝

- 对于数据类型是基本数据类型的成员变量，浅拷贝会直接进行值传递
- 对于数据类型是引用数据类型的成员变量,浅拷贝会进行引用传递
  - String类型属于引用数据类型，不属于基本数据类型，但是String类型的数据是存放在常量池中的，也就是无法修改的
  - 当我将name属性从“耶稣”改为“大傻子"后，并不是修改了这个数据的值，而是把这个数据的引用从指向”耶稣“这个常量改为了指向”大傻子“这个常量。
- 深拷贝对引用数据类型的成员变量的对象图中所有的对象都开辟了内存空间；而浅拷贝只是传递地址指向，新的对象并没有对引用数据类型创建内存空间。





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

- 在注解上标记 @Inherited 元注解可以实现注解的继承

  - 子类可以获得父类类上的注解；子类 foo 方法虽然是重写父类方法，并且注解本身也支持继承，但还是无法获得方法上的注解
  - AnnotatedElementUtils 类，来方便我们处理注解的继承问题
    - findMergedAnnotation 工具方法，可以帮助我们找出父类和接口、父类方法和接口方法上的注解，并可以处理桥接方法，实现一键找到继承链的注解



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

        

## 集合类概述

- List , Set,  Map都是接口，前两个继承至collection接口，Map为独立接口
- collection接口下还有个Queue接口，有PriorityQueue类
- Set下有HashSet，LinkedHashSet，TreeSet
- List下有ArrayList，Vector，LinkedList
- Map下有Hashtable，LinkedHashMap，HashMap，TreeMap                     



## ArrayList	

- 介绍

  - ArrayList的底层数据结构为数组（数组是一组连续的内存空间），默认容量为10，线程不安全，可以存储null值。

  - arrayList由于本质是数组，所以它在数据的查询方面会很快，而在插入删除这些方面，性能下降很多，有移动很多数据才能达到应有的效果（需要扩容）

  - ```java
    ArrayList extends AbstractList
    AbstractList extends AbstractCollection 
    
    ```

  - ArrayList实现了List<E>接口、RandomAccess接口、Cloneable接口、Serializable接口

- add(E)

  - ensureCapacityInternal(size + 1);确定内部容量的方法。判断size+1的这个个数数组能否放得下，就在这个方法中去判断是否数组.length是否够用了
    - ensureCapacityInternal方法，主要是有两个目的：1.如果没初始化则进行初始化；2.校验添加元素后是否需要扩容
    - ArrayList中储存数据的其实就是一个数组，这个数组就是elementData
  - calculateCapacity
    - 如果elementData是空的话,将minCapacity变成10，也就是默认大小，但是在这里，还没有真正的初始化这个elementData的大小。
  - ensureExplicitCapacity(xxx)；确认实际的容量，这个方法就是真正的判断elementData是否够用
    - minCapacity如果大于了实际elementData的长度，那么就说明elementData数组的长度不够用，不够用那么就要增加elementData的length
  - grow(xxx); 扩展数组大小
    - newCapacity就是1.5倍的oldCapacity
    - elementData为空数组的时候，真正的初始化elementData的大小了，就是为10.前面的工作都是准备工作。
    - 新的容量大小已经确定好了，就copy数组，改变容量大小

- 循环遍历并删除元素的陷阱

  - ```java
    // 在遍历第二个元素字符串bb时因为符合删除条件，所以将该元素从数组中删除，并且将后一个元素移动（也是字符串bb）至当前位置，导致下一次循环遍历时后一个字符串bb并没有遍历到，所以无法删除
    public static void remove(ArrayList<String> list) {  
        for (int i = 0; i < list.size(); i++) {  
            String s = list.get(i);  
            if (s.equals("bb")) {  
                list.remove(s);  
            }  
        }  
    }  
    
    list.add("a");
    list.add("bb");
    list.add("bb");
    list.add("ccc");
    list.add("ccc");
    list.add("ccc");
    
    // 针对这种情况可以倒序删除的方式来避免
    
    ```

- ConcurrentModificationException

  - 在单线程环境下的解决办法
    - 在迭代器中如果要删除元素的话，需要调用Itr类的remove方法（iterator.remove()）
    - 在这个方法中，删除元素实际上调用的就是list.remove()方法，但是多了一个操作：expectedModCount = modCount;
  - 在多线程环境下的解决方法
    - 通过Iterator访问的情况下，每个线程里面返回的是不同的iterator，也就是说expectedModCount是每个线程私有
    - 在使用iterator迭代的时候使用synchronized或者Lock进行同步；
    - 使用并发容器CopyOnWriteArrayList代替ArrayList和Vector。





## LinkedList

- 介绍
  - LinkedList是一种可以在任何位置进行高效地插入和移除操作的有序序列，它是基于双向链表实现的
  - linkedList在执行任何操作的时候，都必须先遍历此列表来靠近通过index查找我们所需要的的值。（顺序存取）（注意和随机存取结构两个概念搞清楚）
  - 非线程安全的(异步)，其中在操作Interator时，如果改变列表结构(add\delete等)，会发生fail-fast
- 在addAll函数中，传入一个集合参数和插入位置，然后将集合转化为数组，然后再遍历数组，挨个添加数组的元素，但是问题来了，为什么要先转化为数组再进行遍历，而不是直接遍历集合呢？
  - 如果直接遍历集合的话，那么在遍历过程中需要插入元素，在堆上分配内存空间，修改指针域，这个过程中就会一直占用着这个集合，考虑正确同步的话，其他线程只能一直等待。
  - 如果转化为数组，只需要遍历集合，而遍历集合过程中不需要额外的操作，所以占用的时间相对是较短的，这样就利于其他线程尽快的使用这个集合。说白了，就是有利于提高多线程访问该集合的效率，尽可能短时间的阻塞。
- arrayList和LinkedList区别
  - arrayList底层是用数组实现的顺序表，是随机存取类型，可自动扩增，并且在初始化时，数组的长度是0，只有在增加元素时，长度才会增加。默认是10，不能无限扩增，有上限，在查询操作的时候性能更好
  - LinkedList底层是用链表来实现的，是一个双向链表，注意这里不是双向循环链表,顺序存取类型。在源码中，似乎没有元素个数的限制。应该能无限增加下去，直到内存满了在进行删除，增加操作时性能更好。
  - 两个都是线程不安全的，在iterator时，会发生fail-fast。

## ArrayList还是LinkedList

- ArrayList
  - 数组是一块连续的内存空间，并且实现了RandomAccess 接口标志，意味着 ArrayList 可以实现快速随机访问，所以 for 循环效率非常高。
  - ArrayList 的实现类源码中对象数组 elementData 使用了transient 修饰，防止对象数组被其他外部方法序列化。
    - 由于 ArrayList 的数组是基于动态扩增的，所以并不是所有被分配的内存空间都存储了数据。
    - 如果采用外部序列化法实现数组的序列化，会序列化整个数组。ArrayList 为了避免这些没有存储数据的内存空间被序列化，内部提供了两个私有方法 writeObject 以及 readObject来自我完成序列化与反序列化，从而在序列化与反序列化数组时节省了空间和时间。
  - 如果我们在初始化时就比较清楚存储数据的大小，就可以在 ArrayList 初始化时指定数组容量大小，并且在添加元素时，只在数组末尾添加元素，那么 ArrayList 在大量新增元素的场景下，性能并不会变差
  - 两个方法也有不同之处，添加元素到任意位置，会导致在该位置后的所有元素都需要重新排列，而将元素添加到数组的末尾，在没有发生扩容的前提下，是不会有元素复制排序过程的。
  - ArrayList 在每一次有效的删除元素操作之后，都要进行数组的重组，并且删除的元素位置越前，数组重组的开销就越大。
- LinkedList
  - LinkedList 的两个重要属性 first/last 属性，其实还有一个 size 属性。我们可以看到这三个属性都被 transient 修饰了，原因很简单，我们在序列化的时候不会只对头尾进行序列化，所以 LinkedList 也是自行实现 readObject 和writeObject 进行序列化与反序列化
  - LinkedList 的获取元素操作实现跟 LinkedList 的删除元素操作基本类似，通过分前后半段来循环查找到对应的元素。但是通过这种方式来查询元素是非常低效的，特别是在 for 循环遍历的情况下，每一次循环都会去遍历半个 List。以使用 iterator 方式迭代循环，直接拿到我们的元素，而不需要通过循环查找 List。
- 在添加元素到尾部的操作中，我们发现，在没有扩容的情况下，ArrayList 的效率要高于LinkedList。这是因为 ArrayList 在添加元素到尾部的时候，不需要复制重排数据，效率非常高。而 LinkedList 虽然也不用循环查找元素，但 LinkedList 中多了 new 对象以及变换指针指向对象的过程，所以效率要低于 ArrayList。
- for迭代遍历
  - 当再次遍历时，会先调用内部类iteator中的hasNext(),再调用next(),在调用next()方法时，会对modCount和expectedModCount进行比较（checkForComodification()），此时两者不一致，就抛出了ConcurrentModificationException异常
  - 在foreach循环中调用list中的remove()方法，会走到fastRemove()方法，该方法不是iterator中的方法，而是ArrayList中的方法，在该方法只做了modCount++，而没有同步到expectedModCount。
  - 在iterator中的remove方法中，删除元素实际上调用的就是list.remove()方法，但是它多了一个操作：expectedModCount = modCount;
- 为什么 ArrayList 不像 HashMap 一样在扩容时需要一个负载因子
  - HashMap有负载因子是既要考虑数组太短，因哈希冲突导致链表过长而导致查询性能下降，也考虑了数组过长，新增数据时性能下降。这个负载因子是综合了数组和链表两者的长度，不能太大也不能太小。而ArrayList不需要这种考虑。

## 集合类的坑

- 使用 Arrays.asList 把数据转换为 List 的三个坑

  - 第一个坑是，不能直接使用 Arrays.asList 来转换基本类型数组

    - ```java
      int[] arr1 = {1, 2, 3};
      List list1 = Arrays.stream(arr1).boxed().collect(Collectors.toList());
      ```

  - 第二个坑，Arrays.asList 返回的 List 不支持增删操作

    - Arrays.asList 返回的 List 并不是我们期望的 java.util.ArrayList，而是 Arrays 的内部类 ArrayList。ArrayList 内部类继承自AbstractList 类，并没有覆写父类的 add 方法，而父类中 add 方法的实现，就是抛出UnsupportedOperationException

  - 第三个坑，对原始数组的修改会影响到我们获得的那个 List

    - 修复方式比较简单，重新 new 一个 ArrayList 初始化 Arrays.asList 返回的 List 即可

- 使用 List.subList 进行切片操作会导致 OOM

  - List.subList 返回的子List 不是一个普通的 ArrayList。这个子 List 可以认为是原始 List 的视图，会和原始 List 相互影响。如果不注意，很可能会因此产生 OOM 问题。
  - 既然 SubList 相当于原始 List 的视图，那么避免相互影响的修复方式有两种
    - 一种是，不直接使用 subList 方法返回的 SubList，而是重新使用 new ArrayList，在构造方法传入 SubList，来构建一个独立的 ArrayList；
    - 另一种是，对于 Java 8 使用 Stream 的 skip 和 limit API 来跳过流中的元素，以及限制流中元素的个数，同样可以达到 SubList 切片的目的。

- 一定要让合适的数据结构做合适的事情

  - 第一个误区是，使用数据结构不考虑平衡时间和空间。
    - ArrayList 在内存占用上性价比很高
    - 要对大 List 进行单值搜索的话，可以考虑使用 HashMap，其中 Key 是要搜索的值，Value 是原始对象，会比使用 ArrayList 有非常明显的性能优势。
  - 第二个误区是，过于迷信教科书的大 O 时间复杂度
    - 对于数组，随机元素访问的时间复杂度是 O(1)，元素插入操作是 O(n)； 
    - 对于链表，随机元素访问的时间复杂度是 O(n)，元素插入操作是 O(1)。 
    - 在大量的元素插入、很少的随机访问的业务场景下，
      - 在随机访问方面，我们看到了 ArrayList 的绝对优势，
      - 随机插入操作居然也是 LinkedList 落败
    - 翻看 LinkedList 源码发现，插入操作的时间复杂度是 O(1) 的前提是，你已经有了那个要插入节点的指针。但，在实现的时候，我们需要先通过循环获取到那个节点的 Node，然后再执行插入操作。前者也是有开销的，不可能只考虑插入操作本身的代价

- 与 ArrayList 在删除元素方面的坑有关

  - 调用类型是 Integer 的 ArrayList 的 remove 方法删除元素，传入一个 Integer 包装类的数字和传入一个 int 基本类型的数字，结果一样吗？
    - remove包装类数字是删除对象，基本类型的int数字是删除下标。



## Hashtable

- 函数都是同步的，意味着它是线程安全的
- Hashtable的主要对外接口
  - 根据entrySet()获取Hashtable的“键值对”的Set集合
  - 根据keySet()获取Hashtable的“键”的Set集合。
  - 根据value()获取Hashtable的“值”的集合。
  - 根据keys()获取Hashtable“键”的Set集合。
  - 根据elements()获取Hashtable“值”的集合。





## HashMap

- 介绍

  - 根据键的hashCode值存储数据，大多数情况下可以直接定位到它的值，因而具有很快的访问速度，但遍历顺序却是不确定的。 
  - HashMap最多只允许一条记录的键为null，允许多条记录的值为null。
  - HashMap非线程安全，即任一时刻可以有多个线程同时写HashMap，可能会导致数据的不一致。如果需要满足线程安全，可以用 Collections的synchronizedMap方法使HashMap具有线程安全的能力，或者使用ConcurrentHashMap。

- HashMap的属性

  - loadFactor加载因子

    - 计算HashMap的实时装载因子的方法为：size/capacity，而不是占用桶的数量去除以capacity
    - 在hashMap中loadFactor的初始值就是0.75，一般情况下不需要更改它

  - 桶

    - 数组中每一个位置上都放有一个桶，每个桶里就是装一个链表，链表中可以有很多个元素(entry)，这就是桶的意思。也就相当于把元素都放在桶中。

  - capacity

    - capacity译为容量代表的数组的容量，也就是数组的长度，同时也是HashMap中桶的个数，默认值是16

  - size

    - size就是在该HashMap的实例中实际存储的元素的个数

  - threshold

    - 当哈希表中条目数超出了当前容量*加载因子threshold = capacity * loadFactor(其实就是HashMap的实际容量)时，则对该哈希表进行rehash操作，将哈希表扩充至两倍的桶数。
    - 什么时候会扩增数组的大小？在put一个元素时先size>=threshold并且还要在对应数组位置上有元素，这才能扩增数组

  - 通过key的值来计算entry的hash值，通过entry的hash值和数组的长度length来计算出entry放在数组中的哪个位置上面

    - 用hash算法求得这个位置(三步：取key的hashCode值、高位运算、取模运算)

    - ```java
      // 方法一：
      // 在JDK1.8的实现中，优化了高位运算的算法，通过hashCode()的高16位异或低16位实现的
      static final int hash(Object key) { //jdk1.8 & jdk1.7
      int h;
      // h = key.hashCode() 为第一步 取hashCode值
      // h ^ (h >>> 16) 为第二步 高位参与运算
      return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
      
      }
      
      // 方法二：
      static int indexFor(int h, int length) { //jdk1.7的源码，jdk1.8没有这个方法，但是实现原理一样的
      // 它通过h & (table.length -1)来得到该对象的保存位，而HashMap底层数组的长度总是2的n次方，这是HashMap在速度上的优化。当length总是2的n次方时，h& (length-1)运算等价于对length取模，也就是h%length，但是&比%具有更高的效率
          return h & (length-1); //第三步 取模运算
      
      }
      
      
      ```

- hashmap为什么是二倍扩容

  - 容量n为2的幂次方，n-1的二进制会全为1，位运算时可以充分散列，避免不必要的哈希冲突。所以扩容必须2倍就是为了维持容量始终为2的幂次方。

    > /*
    > HashMap的容量为16转化成二进制为10000，length-1得出的二进制为01111 
    >
    > h & (length-1)
    > 哈希值为1111 
    >
    > 
    >
    > 可以得出索引的位置为15
    >
    > 假设 
    >
    > HashMap的容量为15转化成二进制为1111，length-1得出的二进制为1110 
    >
    > h & (length-1)
    > 哈希值为1111和1110 
    >
    > 
    >
    > 那么两个索引的位置都是14，就会造成分布不均匀了，
    >
    > 增加了碰撞的几率，
    >
    > 减慢了查询的效率，
    >
    > 造成空间的浪费。 
    >
    > 总结：
    >
    > 因为2的幂-1都是11111结尾的，所以碰撞几率小。
    >
    > */

- put方法

  ①.判断键值对数组table[i]是否为空或为null，否则执行resize()进行扩容；

  ②.根据键值key计算hash值得到插入的数组索引i，如果table[i]==null，直接新建节点添加，转向⑥，如果table[i]不为空，转向③；

  ③.判断table[i]的首个元素是否和key一样，如果相同直接覆盖value，否则转向④，这里的相同指的是hashCode以及equals；

  ④.判断table[i] 是否为treeNode，即table[i] 是否是红黑树，如果是红黑树，则直接在树中插入键值对，否则转向⑤；

  ⑤.遍历table[i]，判断链表长度是否大于8，大于8的话把链表转换为红黑树，在红黑树中执行插入操作，否则进行链表的插入操作；遍历过程中若发现key已经存在直接覆盖value即可；

  ⑥.插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。

- resize方法

  - resize用于以下两种情况之一
    - 初始化table
      - 初始化HashMap时,  如果构造函数没有指定initialCapacity, 则table大小为16;
      - 如果构造函数指定了initialCapacity, 则table大小为threshold, 即大于指定initialCapacity的最小的2的整数次幂
    - 在table大小超过threshold之后进行扩容
  - resize时的链表拆分
    - 明确三点
      - oldCap一定是2的整数次幂, 这里假设是2^m
      - newCap是oldCap的两倍, 则会是2^(m+1)
      - hash对数组大小取模`(n - 1) & hash` 其实就是取hash的低`m`位
        假设 oldCap = 16, 即 2^4, 
          		则 `(16-1) & hash` 自然就是取hash值的低4位,我们假设它为 `abcd`.
    - oldCap扩大两倍后, 新的index的位置就变成了 `(32-1) & hash`, 其实就是取 hash值的低5位
    - 新旧index是否一致就体现在hash值的第4位(我们把最低为称作第0位)
      - 如果 `(e.hash & oldCap) == 0` 则该节点在新表的下标位置与旧表一致都为 `j`  
      - 如果 `(e.hash & oldCap) == 1` 则该节点在新表的下标位置 `j + oldCap` 
      - 根据这个条件, 我们将原位置的链表拆分成两个链表, 然后一次性将整个链表放到新的Table对应的位置上.

- 头插法导致的死循环

  - 老版本HashMap扩容代码，其中，重点在于transfer()

  - ```java
    void transfer(Entry[] newTable)
    {
    　　//复制一个原数组src，Entry是一个静态内部类，有K，V，next三个成员变量
        Entry[] src = table;
    　　//数组新容量
        int newCapacity = newTable.length;：
        //  从OldTable里摘一个元素出来，然后放到NewTable中
        for (int j = 0; j < src.length; j++) {
            Entry<K,V> e = src[j];//取出原数组一个元素
            if (e != null) {//判断原数组该位置有元素
                src[j] = null;//原数组位置置为空
                do {//对原数组某一位置下的一串元素进行操作
                    Entry<K,V> next = e.next;//next是当前元素下一个
                    int i = indexFor(e.hash, newCapacity);//i是元素在新数组的位置
                    e.next = newTable[i];//此处体现了头插法，当前元素的下一个是新数组的头元素
                    newTable[i] = e;//将原数组元素加入新数组
                    e = next;//遍历到原数组某一位置下的一串元素的下一个
    　　　　　　} while (e != null); 
    　　　　} 
    　　} 
    }
    ```

    

    - 接下来图示**单线程**情况下，do循环内的情况：

      初始：当前数组容量为2，有三个元素3、7、5，此处的hash算法是简化处理（对容量取模）。因此，3、7、5都在数组索引1对应的链表上。

      　　扩容新容量为2*2=4。

      　　第一步：当前Entry e对应3，next对应7，新位置i为3，然后将3插入新数组对应位置。

      　　第二步：当前Entry e对应7，next对应5，新位置i为3，然后将新数组对应索引处的元素3添加到7的尾巴后（头插），然后将7插入新数组对应位置。

      　　第三步：当前Entry e对应5，next对应null，新位置i为1, 然后将5插入新数组对应位置。

      ![img](https://img2020.cnblogs.com/blog/1384439/202006/1384439-20200606103906885-46047020.png)

       

       

    - 接下来图示多线程情况下死循环场景：初始条件相同。

      如果有两个线程：

      　　　　线程一执行到 Entry<K,V> next = e.next; 便挂起了，即此时Entry e是3，next是7，3是在7前面的。

      　　　　线程二执行完成。

      　　此时如下图所示，线程一的3的next是7，而线程二的7的next是3。（此处是Entry里的next成员变量，在多个线程中相同Entry不冲突）。此时可以看出出现了死循环问题。

      ![img](https://img2020.cnblogs.com/blog/1384439/202006/1384439-20200606111846055-839726988.png)

       

       

       　　如果此时线程一继续往下执行：

       　　第一步：当前Entry e对应3，next对应7，新位置i为3，然后将3插入新数组对应位置。

      　　 第二步：当前Entry e对应7，next对应3（单线程情况下是5），新位置i为3，然后将7插入新数组对应位置。

      　　 第三步：当前Entry e对应3，next对应7，此处死循环，永远不会跳出while循环。

       ![img](https://img2020.cnblogs.com/blog/1384439/202006/1384439-20200606120547060-2090940084.png)

       

       

      总结归纳：多线程情况下，使用头插法会导致链表节点之间的关系混乱，出现倒排现象，例如原本3->7->5变成7->3，其他线程此时再进行扩容是会出现死循环。 

  - 线程A先执行，执行完Entry<K,V> next = e.next;这行代码后挂起，然后线程B完整的执行完整个扩容流程，接着线程A唤醒，继续之前的往下执行，当while循环执行3次后会形成环形链表

  - newTable[i]的引用赋给了e.next，也就是使用了单链表的头插入方式，同一位置上新元素总会被放在链表的头部位置

  - jdk1.8在扩充HashMap的时候，不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了

    - 我们使用的是2次幂的扩展(指长度扩为原来2倍)，所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置
    - 这个设计确实非常的巧妙，既省去了重新计算hash值的时间，而且同时，由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的bucket了。这一块就是JDK1.8新增的优化点
    - 减少哈希冲突，均匀分布元素

- HashMap在jdk1.8之后引入了红黑树的概念，表示若桶中链表元素超过8时，会自动转化成红黑树；若桶中元素小于等于6时，树结构还原成链表形式。

  原因：

  　　红黑树的平均查找长度是log(n)，长度为8，查找长度为log(8)=3，链表的平均查找长度为n/2，当长度为8时，平均查找长度为8/2=4，这才有转换成树的必要；链表长度如果是小于等于6，6/2=3，虽然速度也很快的，但是转化为树结构和生成树的时间并不会太短。

  还有选择6和8的原因是：

  　　中间有个差值7可以防止链表和树之间频繁的转换。假设一下，如果设计成链表个数超过8则链表转换成树结构，链表个数小于8则树结构转换成链表，如果一个HashMap不停的插入、删除元素，链表个数在8左右徘徊，就会频繁的发生树转链表、链表转树，效率会很低。



## TreeMap

- TreeMap实现了SotredMap接口，它是有序的集合。而且是一个红黑树结构，每个key-value都作为一个红黑树的节点
- 如果在调用TreeMap的构造函数时没有指定比较器，则根据key执行自然排序
  - 自然排序：TreeMap的所有key必须实现Comparable接口，所有的key都是同一个类的对象
  - 定制排序：创建TreeMap对象传入了一个Comparator对象，该对象负责对TreeMap中所有的key进行排序，采用定制排序不要求Map的key实现Comparable接口
  - 默认排序方式：对key升序排序。
- 二叉-查找树
  - 概念
    - 若任意节点的左子树不空，则左子树上所有结点的值均小于它的根结点的值；
    - 若任意节点的右子树不空，则右子树上所有结点的值均大于它的根结点的值；
    - 任意节点的左、右子树也分别为二叉查找树；
    - 没有键值相等的节点。
  - 对二叉查找树进行中序遍历，即可得到有序的数列
  - 删除操作
    - 如果删除的是叶节点，可以直接删除；
    - 如果被删除的元素有一个子节点，可以将子节点直接移到被删除元素的位置；
    - 如果有两个子节点，这时候就采用中序遍历，找到待删除的节点的后继节点，将其与待删除的节点互换，此时待删除节点的位置已经是叶子节点，可以直接删除
- 平衡二叉树
  - 它是一棵空树或它的左右两个子树的高度差的绝对值不超过1，并且左右两个子树都是一棵平衡二叉树
  - 在构建一棵平衡二叉树的过程中，当有新的节点要插入时，检查是否因插入后而破坏了树的平衡，如果是，则需要做旋转去改变树的结构
  - 旋转
    - 旋转的目的都是将节点多的一支出让节点给另一个节点少的一支
    - 左旋
      - 将节点的右支往左拉，右子节点变成父节点，并把晋升之后多余的左子节点出让给降级节点的右子节点
    - 右旋
      - 将节点的左支往右拉，左子节点变成了父节点，并把晋升之后多余的右子节点出让给降级节点的左子节点
  - 插入
    - 在节点的左子树的左子树下，有新节点插入
      - 只需要对节点进行右旋即可
    - 在节点的左子树的右子树下，有新节点插入
      - 第一次旋转，将左右先调整成左左
      - 然后再对左左进行调整，从而使得二叉树平衡
      - 先左旋再右旋
    - 左右跟右左互为镜像，左左跟右右也互为镜像
- 平衡二叉树（AVL树）
  - 平衡二叉树(Balance Binary Tree)又称AVL树。它或者是一颗空树，或者是具有下列性质的二叉树：它的左子树和右子树都是平衡二叉树，且左子树和右子树的深度之差的绝对值不超过1。



## 从2-3树到 红黑树

- 红黑树的基本思想是用标准的二叉查找树（完全由2-结点构成）和一些额外的信息（替换3-结点）来表示2-3树。
  - 2-结点：含有一个键(及值)和两条链接，左链接指向的2-3树中的键都小于该结点，右链接指向的2-3树中的键都大于该结点。
  - 3-结点：含有两个键(及值)和三条链接，左链接指向的2-3树中的键都小于该结点，中链接指向的2-3树中的键都位于该结点的两个键之间，右链接指向的2-3树中的键都大于该结点。
- 插入
  - 要在2-3树中插入一个新结点，我们可以和二叉查找树一样先进行一次未命中的查找，然后把新结点挂在树的底部。但这样的话树无法保持完美平衡性。我们使用2-3树的主要原因就在于它能够在插入之后继续保持平衡。
  - 如果未命中的查找结束于一个2-结点，我们只要把这个2-结点替换为一个3-结点，将要插入的键保存在其中即可。如果未命中的查找结束于一个3-结点，事情就要麻烦一些。
- 总结
  - 先考虑最简单的例子：只有一个3-结点的树，向其插入一个新键。
    - 这棵树唯一的结点中已经没有可插入的空间了。我们又不能把新键插在其空结点上（破坏了完美平衡）。为了将新键插入，我们先临时将新键存入该结点中，使之成为一个4-结点。创建一个4-结点很方便，因为很容易将它转换为一颗由3个2-结点组成的2-3树，这棵树既是一颗含有3个结点的二叉查找树，同时也是一颗完美平衡的2-3树，其中所有空链接到根结点的距离都相等。
  - 先找插入结点，若结点有空(即2-结点)，则直接插入。如结点没空(即3-结点)，则插入使其临时容纳这个元素，然后分裂此结点，把中间元素移到其父结点中。对父结点亦如此处理
  - 2-3树插入算法的根本在于这些变换都是局部的：除了相关的结点和链接之外不必修改或者检查树的其他部分。每次变换中，变更的链接数量不会超过一个很小的常数。所有局部变换都不会影响整棵树的有序性和平衡性。
- 优缺点
  - 2-3树在最坏情况下仍有较好的性能。每个操作中处理每个结点的时间都不会超过一个很小的常数，且这两个操作都只会访问一条路径上的结点，所以任何查找或者插入的成本都肯定不会超过对数级别。
  - 完美平衡的2-3树要平展的多。例如，含有10亿个结点的一颗2-3树的高度仅在19到30之间。我们最多只需要访问30个结点就能在10亿个键中进行任意查找和插入操作。
  - 我们需要维护两种不同类型的结点，查找和插入操作的实现需要大量的代码，而且它们所产生的额外开销可能会使算法比标准的二叉查找树更慢。
  - 平衡一棵树的初衷是为了消除最坏情况，但我们希望这种保障所需的代码能够越少越好。
- 理解红黑树一句话就够了：红黑树就是用红链接表示3-结点的2-3树。
  - 那么红黑树的插入、构造就可转化为2-3树的问题，即：在脑中用2-3树来操作，得到结果，再把结果中的3-结点转化为红链接即可
  - 我们将树中的链接分为两种类型：红链接将两个2-结点连接起来构成一个3-结点，黑链接则是2-3树中的普通链接。确切地说，我们将3-结点表示为由一条左斜的红色链接相连的两个2-结点。
  - 这种表示法的一个优点是，我们无需修改就可以直接使用标准二叉查找树的get()方法。对于任意的2-3树，只要对结点进行转换，我们都可以立即派生出一颗对应的二叉查找树。我们将用这种方式表示2-3树的二叉查找树称为红黑树。
- 红黑树的另一种定义是满足下列条件的二叉查找树
  - 红链接均为左链接。
  - 没有任何一个结点同时和两条红链接相连。
  - 该树是完美黑色平衡的，即任意空链接到根结点的路径上的黑链接数量相同。
  - 无论我们用何种方式去定义它们，红黑树都既是二叉查找树，也是2-3树。
- 颜色表示
  - 因为每个结点都只会有一条指向自己的链接（从它的父结点指向它），我们将链接的颜色保存在表示结点的Node数据类型的布尔变量color中（若指向它的链接是红色的，那么该变量为true，黑色则为false）。
  - 当我们提到一个结点颜色时，我们指的是指向该结点的链接的颜色。



## 红黑树

- 红黑树之所以难是难在它是自平衡的二叉查找树，在进行插入和删除等可能会破坏树的平衡的操作时，需要重新自处理达到平衡状态。

- 定义和性质

  - 每个节点要么是黑色，要么是红色。
    - 根节点是黑色。
    - 每个叶子节点(Nil)是黑色。
    - 每个红色结点的两个子结点一定都是黑色。
    - 任意一结点到每个叶子结点的路径都包含数量相同的黑结点。
      左子树和右子树的黑结点的层数是相等的
    - 从性质 5 又可以推出：如果一个结点存在黑子结点，那么该结点肯定有两个子结点。
      - ![img](http://s2.51cto.com/oss/201901/22/c7d65c349191f5e5a7efdb6fc33f0ff6.jpg)

- 时间复杂度

- 最坏的情况下也可以保证O(logN)的，这是要好于二叉查找树的。因为二叉查找树最坏情况可以让查找达到O(N)。

  - 红黑树是牺牲了严格的高度平衡的优越条件为代价，它只要求部分地达到平衡要求，降低了对旋转的要求，从而提高了性能。能够以O(log2 n)的时间复杂度进行搜索、插入、删除操作。由于它的设计，任何不平衡都会在三次旋转之内解决

- 红黑树操作

  - 红黑树能自平衡，它靠的是三种操作：
    - 左旋：以某个结点作为支点(旋转结点)，其右子结点变为旋转结点的父结点，右子结点的左子结点变为旋转结点的右子结点，左子结点保持不变。
    - 右旋：以某个结点作为支点(旋转结点)，其左子结点变为旋转结点的父结点，左子结点的右子结点变为旋转结点的左子结点，右子结点保持不变。
    - 变色：结点的颜色由红变黑或由黑变红。
    - 旋转操作不会影响旋转结点的父结点，父结点以上的结构还是保持不变的。
    - 但要保持红黑树的性质，结点不能乱挪，还得靠变色了
  - 红黑树查找
    - ![img](http://s2.51cto.com/oss/201901/22/995e3eb18c0ecd0d25ec4d4f995cadb8.jpg)
    - 简单不代表它效率不好。正由于红黑树总保持黑色完美平衡，所以它的查找最坏时间复杂度为 O(2lgN)，也即整颗树刚好红黑相隔的时候。
    - 能有这么好的查找效率得益于红黑树自平衡的特性，而这背后的付出，红黑树的插入操作功不可没。
  - 红黑树插入
    - 只表示一半的情景，另一半为相反操作
    - ![img](http://s3.51cto.com/oss/201901/22/d3a336a58aad16c18d9a198b367c037d.jpg)
    - 插入操作包括两部分工作：一是查找插入的位置;二是插入后自平衡查找插入的父结点很简单，跟查找操作区别不大：
      - 插入结点红色
        - 红色在父结点(如果存在)为黑色结点时，红黑树的黑色平衡没被破坏，不需要做自平衡操作
      - 如果插入结点是黑色，那么插入位置所在的子树黑色结点总是多 1，必须做自平衡
    - I 表示插入结点，P 表示插入结点的父结点，S 表示插入结点的叔叔结点，PP 表示插入结点的祖父结点。
    - 情景 1：红黑树为空树
      - 把插入结点作为根结点，并把结点设置为黑色。
    - 情景 2：插入结点的 Key 已存在
      - 把 I 设为当前结点的颜色，更新当前结点的值为插入结点的值。
    - 情景 3：插入结点的父结点为黑结点
      - 直接插入
    - 情景 4：插入结点的父结点为红结点
      - 回想下红黑树的性质  2：根结点是黑色。如果插入的父结点为红结点，那么该父结点不可能为根结点，所以插入结点总是存在祖父结点
      - 插入情景 4.1：叔叔结点存在并且为红结点
        - ![img](http://s4.51cto.com/oss/201901/22/ab9a8511442810355d23d47052036c11.jpg)
        - 处理：将 P 和 S 设置为黑色，将 PP 设置为红色，把 PP 设置为当前插入结点。
        - 如果 PP 的父结点是红色，根据性质 4，此时红黑树已不平衡了，所以还需要把 PP 当作新的插入结点，继续做插入操作自平衡处理，直到平衡为止
        - PP 刚好为根结点时，那么根据性质 2，我们必须把 PP 重新设为黑色，那么树的红黑结构变为：黑黑红
          - 从根结点到叶子结点的路径中，黑色结点增加了。这也是唯一一种会增加红黑树黑色结点层数的插入情景
          - 总结出另外一个经验：红黑树的生长是自底向上的。这点不同于普通的二叉查找树，普通的二叉查找树的生长是自顶向下的
      - 插入情景 4.2：叔叔结点不存在或为黑结点，并且插入结点的父亲结点是祖父结点的左子结点。
        - 插入情景 4.2.1：插入结点是其父结点的左子结点。
          - 将 P 设为黑色，将 PP 设为红色，对 PP 进行右旋。
          - ![img](http://s2.51cto.com/oss/201901/22/1babcc6b055fd252e6f553688ec5d770.jpg)
        - 插入情景 4.2.2：插入结点是其父结点的右子结点
          - 对 P 进行左旋，把 P 设置为插入结点，得到情景 4.2.1，进行情景 4.2.1 的处理
          - ![img](http://s2.51cto.com/oss/201901/22/42bacde6f4c67cca3f7c83d356a146ab.jpg)
  - 红黑树删除情景
    - 红黑树的删除操作也包括两部分工作：一是查找目标结点;二是删除后自平衡
    - 前继和后继结点的直观的方法
      - 若删除结点无子结点，直接删除。
      - 若删除结点只有一个子结点，用子结点替换删除结点。
        - 情景 2：删除结点用其唯一的子结点替换，子结点替换为删除结点后，可以认为删除的是子结点，若子结点又有两个子结点，那么相当于转换为情景  3，一直自顶向下转换，总是能转换为情景 1。
      - 若删除结点有两个子结点，用后继结点(大于删除结点的最小结点)替换删除结点。
        - 情景 3：删除结点用后继结点(肯定不存在左结点)，如果后继结点有右子结点，那么相当于转换为情景 2，否则转为情景 1
    - 删除操作结点的叫法约定
      - R  是即将被替换到删除结点的位置的替代结点，在删除前，它还在原来所在位置参与树的子平衡，平衡后再替换到删除结点的位置，才算删除完成
      - ![img](http://s4.51cto.com/oss/201901/22/d36a4e855a65bbdf2d5ec07b9b2dd440.jpg)
    - 删除情景 1：替换结点是红色结点
      - 颜色变为删除结点的颜色。
        - 将 S 设为黑色，将 P 设为红色，对 P 进行左旋，得到情景 2.1.2.3，进行情景 2.1.2.3 的处理
    - 删除情景 2：替换结点是黑结点。
      - 删除情景 2.1：替换结点是其父结点的左子结点
        - 删除情景 2.1.1：替换结点的兄弟结点是红结点
        - 删除情景 2.1.2：替换结点的兄弟结点是黑结点。
          - 父结点和子结点的具体颜色也无法确定
          - 删除情景 2.1.2.1：替换结点的兄弟结点的右子结点是红结点，左子结点任意颜色。
            - 将 S 的颜色设为 P 的颜色，将 P 设为黑色，将 SR 设为黑色，对 P 进行左旋。
            - ![img](http://s2.51cto.com/oss/201901/22/df57bc49670d2b35ef933d6ffb2d04f7.jpg)
          - 删除情景 2.1.2.2：替换结点的兄弟结点的右子结点为黑结点，左子结点为红结点
            - 将 S 设为红色，将 SL 设为黑色，对 S 进行右旋，得到情景 2.1.2.1，进行情景 2.1.2.1 的处理
            - ![img](http://s4.51cto.com/oss/201901/22/39b77c7e02f8824eb4aaa9bb355c7b2f.jpg)
          - 删除情景 2.1.2.3：替换结点的兄弟结点的子结点都为黑结点。
            - 将 S 设为红色，把 P 作为新的替换结点，重新进行删除结点情景处理
            - 这种情景我们把兄弟结点设为红色，再把父结点当作替代结点，自底向上处理，去找父结点的兄弟结点去“借”。
            - 为什么需要把兄弟结点设为红色呢?显然是为了在 P 所在的子树中保证平衡(R  即将删除，少了一个黑色结点，子树也需要少一个)，后续的平衡工作交给父辈们考虑了
            - ![img](http://s5.51cto.com/oss/201901/22/8c230394c67174c0b2a6de0bb8928ca1.jpg)

- 相对于哈希表，在选择使用的时候有什么依据

  - hash查找速度会比map快，而且查找速度基本和数据量大小无关，属于常数级别;而map的查找速度是log(n)级别
  - 如果你考虑效率，特别是在元素达到一定数量级时，考虑考虑hash
  - 希望程序尽可能少消耗内存，那么一定要小心，hash可能会让你陷入尴尬，hash的构造速度较慢
  - 如果数据基本上是静态的，那么让他们待在他们能够插入，并且不影响平衡的地方会具有更好的性能
  - 如果数据完全是静态的，例如，做一个哈希表，性能可能会更好一些。
  - 需要使用动态规则的防火墙系统，使用红黑树而不是散列表被实践证明具有更好的伸缩性
  - Linux内核在管理vm_area_struct时就是采用了红黑树来维护内存块的。

- 为什么一般hashtable的桶数会取一个素数

  - ```java
    设有一个哈希函数
    H( c ) = c % N;
    当N取一个合数时，最简单的例子是取2^n，比如说取2^3=8,这时候
    H( 11100(二进制） ) = H( 28 ) = 4
    H( 10100(二进制) ) = H( 20 ）= 4
    
    这时候c的二进制第4位（从右向左数）就”失效”了，
    
    取质数，基本可以保证c的每一位都参与H( c )的运算，从而在常见应用中减小冲突几率．
    ```

  

  

- 之前我们介绍了[二叉搜索树](https://blog.csdn.net/weixin_42562387/article/details/106091122), 理想情况下二叉搜索树的性能不错, 但是在极端情况下(数据有序, 或者接近有序)二叉搜索树会退化为单支树, 从而导致其操作效率很低, 时间复杂度为O(N), 于是俄罗斯数学家提出了[平衡二叉搜索树,又称为AVL树](https://blog.csdn.net/weixin_42562387/article/details/106176426). AVL树引入了平衡因子,保证了树的平衡, 从而避免了二叉搜索树会退化为单支树的情况, 进而使得AVL操作时间复杂度为log(N), 但是AVL的应用却比较少, **因为AVL的维护,成本太高,对AVL树进行插入或者删除时, 需要不停地通过左旋,右旋操作来维持其平衡,这些操作的成本太高,甚至抵消了AVL树带来的性能提升**.
  还有另外一种平衡树2-3树,红黑树也借鉴了其思想.2-3树就不介绍了, 感兴趣的同学请查看[2-3树](https://blog.csdn.net/qq_31728311/article/details/79252189)

  **红黑树同时借鉴了AVL树和2-3树的思想, 将AVL的完美平衡改进为局部平衡, 通过改变颜色来区分不同的结点类型, 从而降低了维护平衡的成本和实现的复杂度.**



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

          ```
             * 第一次扩容之后 数组长：32 sizeCtl：24
             * 第二次扩容之后 数组长：64 sizeCtl：48
          ```

          - 第三次扩容之后 数组长：128 sizeCtl：94 --> 这个时候才会退出扩容
          - 初始化的时候，设置sizeCtrl为-1，初始化完成之后把sizeCtrl设置为数组长度的3/4

        - 在tryPresize方法中，并没有加锁，允许多个线程进入，如果数组正在扩张，则当前线程也去帮助扩容。

    - 对桶中的第一个结点（即 table 表中的结点）进行加锁，对该桶进行遍历，桶中的结点的 hash 值与 key 值与给定的 hash 值和 key 值相等，则根据标识选择是否进行更新操作（用给定的 value 值替换该结点的 value 值），若遍历完桶仍没有找到 hash 值与 key 值和指定的 hash 值与 key 值相等的结点，则直接新生一个结点并赋值为之前最后一个结点的下一个结点。进入步骤 6。

    - 若 binCount 值达到红黑树转化的阈值，则将桶中的结构转化为红黑树存储，最后，增加 binCount 的值

  - get 方法

    - 计算 hash 值。
    - 根据 hash 值找到数组对应位置: (n – 1) & h。
    - 根据该位置处结点性质进行相应查找。
      - 如果该位置为 null，那么直接返回 null。
      - 如果该位置处的结点刚好就是需要的，返回该结点的值即可。
      - 如果该位置结点的 hash 值小于 0，说明正在扩容，或者是红黑树。
      - 如果以上 3 条都不满足，那就是链表，进行遍历比对即可。
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

    - concurrenthashmap 链表超过8个为什么要转成红黑树
      - 红黑树插入为O(lgn),查询为O(lgn)，链表插入为O(1)，查询为O(n)。个数少时，插入删除成本高，用链表；个数多时，查询成本高，用红黑树。需要定一个值，比这个值大就转红黑树，比这个值小就转链表，而且要避免频繁的转换。根据泊松分布，在负载因子0.75（HashMap默认）的情况下，单个hash槽内元素个数为8的概率小于百万分之一，将7作为一个分水岭，等于7时不做转换，大于等于8才转红黑树，小于等于6才转链表。
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

- ConcurrentSkipListMap

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

  - ConcurrentHashMap 的 Key 和 Value 都不能为 null，而 HashMap 却可以，你知道这么设计的原因是什么吗？TreeMap、Hashtable 等 Map 的 Key 和 Value 是否支持null 呢？

    - hashmap计算hash值的时候return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);会进行判断，所以key最多只能一个为null。hashtable计算hash值的时候int hash = key.hashCode();直接取的hashcode，key为null报错

    - Doug给出的回答是：在ConcurrentMaps (ConcurrentHashMaps, ConcurrentSkipListMaps)这些考虑并发安全的容器中不允许null值的出现的主要原因是他可能会在并发的情况下带来难以容忍的二义性

    - ```java
      // 当在并发情况下，有两个线程分别在操作map容器，此时线程1在运行以上代码，当线程1运行到代码1与代码2中间时，刚好有另外一个线程2执行了map.remove(key)操作，此时继续运行代码2时，依然会返回null值。而此时的null实际上是map中真实的不存在该key值，应该throw new KeyNotPresentException()的。所以为了保证线程安全，这两个Map容器是不允许key和value为null。
      
      // 而HashMap是非线程安全的，不存在以上我们所说的并发情况  
      if (map.containsKey(key)) {//代码1
         return map.get(key);//代码2
      } else {
         throw new KeyNotPresentException(); 
      }
      ```

      

  

## 并发工具类库

- **没有意识到线程重用导致用户信息错乱的 Bug**

  - ThreadLocal 适用于变量在线程间隔离，而在方法或类间共享的场景
  - 程序运行在 Tomcat 中，执行程序的线程是 Tomcat 的工作线程，而 Tomcat 的工作线程是基于线程池的。顾名思义，线程池会重用固定的几个线程，一旦线程重用，那么很可能首次从 ThreadLocal 获取的值是之前其他用户的请求遗留的值。这时，ThreadLocal 中的用户信息就是其他用户的信息。
  - 这个例子告诉我们，在写业务代码时，首先要理解代码会跑在什么线程上
    - 在 Tomcat 这种 Web 服务器下跑的业务代码，本来就运行在一个多线程环境（否则接口也不可能支持这么高的并发），并不能认为没有显式开启多线程就不会有线程安全问题。
    - 理解了这个知识点后，我们修正这段代码的方案是，在代码的 finally 代码块中，显式清除ThreadLocal 中的数据（remove()）。这样一来，新的请求过来即使使用了之前的线程也不会获取到错误的用户信息了。

- 使用了线程安全的并发工具，并不代表解决了所有线程安全问题

  - “线程安全”这四个字特别容易让人误解，因为 ConcurrentHashMap 只能保证提供的原子性读写操作是线程安全的。

  - 我们需要注意 ConcurrentHashMap 对外提供的方法或能力的限制

    - 诸如 size、isEmpty 和 containsValue 等聚合方法，在并发情况下可能会反映ConcurrentHashMap 的中间状态。因此在并发情况下，这些方法的返回值只能用作参考，而不能用于流程控制。显然，利用 size 方法计算差异值，是一个流程控制。

    - 诸如 putAll 这样的聚合方法也不能确保原子性，在 putAll 的过程中去获取数据可能会获取到部分数据。

    - ```java
      // 代码的修改方案很简单，整段逻辑加锁即可     
      synchronized (concurrenthashMap){
                  int gap=concurrenthashMap.size();
                  concurrenthashMap.putAll(getData(gap));
              }
      ```

  - 有一个含 900 个元素的 Map，现在再补充 100 个元素进去，这个补充操作由 10 个线程并发进行。开发人员误以为使用了 ConcurrentHashMap 就不会有线程安全问题，于是不加思索地写出了下面的代码：在每一个线程的代码逻辑中先通过 size 方法拿到当前元素数量，计算 ConcurrentHashMap 目前还需要补充多少元素，并在日志中输出了这个值，然后通过 putAll 方法把缺少的元素添加进去。

    - ConcurrentHashMap 这个篮子本身，可以确保多个工人在装东西进去时，不会相互影响干扰，但无法确保工人 A 看到还需要装 100 个桔子但是还未装的时候，工人 B 就看不到篮子中的桔子数量。更值得注意的是，你往这个篮子装 100 个桔子的操作不是原子性的，在别人看来可能会有一个瞬间篮子里有 964 个桔子，还需要补 36 个桔子。

    - ```java
      // IntStream boxed（）返回一个由该流元素组成的Stream，每个元素装箱为一个Integer。
      // IntStream rangeClosed（int startInclusive，int endInclusive）以增量步长1返回一个从startInclusive（包括）到endInclusive（包括）的IntStream
      
      // Function是一个接口，Java 8允许在接口中加入具体方法。接口中的具体方法有两种，default方法和static方法，identity()就是Function接口的一个静态方法。Function.identity()返回一个输出跟输入一样的Lambda表达式对象，等价于形如t -> t形式的Lambda表达式。
      // int[] arrayProblem = list.stream().mapToInt(Function.identity()).toArray();
      // 运行的时候就会错误，因为mapToInt要求的参数是ToIntFunction类型，但是ToIntFunction类型和Function没有关系
      
      private ConcurrentHashMap<String, Long> getData(int count){
              return LongStream.rangeClosed(1,count)
                      .boxed()
                      .collect(Collectors.toConcurrentMap(i -> UUID.randomUUID().toString(), Function.identity(),
              (o1,o2)->o1,ConcurrentHashMap::new));
          }
      ```

- **没有充分了解并发工具的特性，从而无法发挥其威力**

  - AtomicLong 的缺陷

    - AtomicLong 的 Add() 是依赖自旋不断的 CAS 去累加一个 Long 值。如果在竞争激烈的情况下，CAS 操作不断的失败，就会有大量的线程不断的自旋尝试 CAS 会造成 CPU 的极大的消耗

  - LongAdder 解决方案

    - AtomicLong 只对一个 Long 值进行 CAS 操作。而 LongAdder 是针对 Cell 数组的某个 Cell 进行 CAS 操作 ，把线程的名字的 hash 值，作为 Cell 数组的下标，然后对 Cell[i] 的 long 进行 CAS 操作。简单粗暴的分散了高并发下的竞争压力。
    - 其实一个 Cell 的本质就是一个 volatile 修饰的 long 值，且这个值能够进行 cas 操作
    - 其实我们可以发现，LongAdder 使用了一个 cell 列表去承接并发的 cas，以提升性能，但是 LongAdder 在统计的时候如果有并发更新，可能导致统计的数据有误差。

  - ```java
    // 使用 ConcurrentHashMap 的原子性方法 computeIfAbsent 来做复合逻辑操作，判断Key 是否存在 Value，如果不存在则把 Lambda 表达式运行后的结果放入 Map 作为Value，也就是新创建一个 LongAdder 对象，最后返回 Value。
    // 由于 computeIfAbsent 方法返回的 Value 是 LongAdder，是一个线程安全的累加器，因此可以直接调用其 increment 方法进行累加。
    ConcurrentHashMap<String, LongAdder> freqs=new ConcurrentHashMap<>();
    ForkJoinPool forkJoinPool =new ForkJoinPool(10);
    forkJoinPool.execute(()-> IntStream.rangeClosed(1,1000000).parallel().forEach(i->{
                String key="item"+ ThreadLocalRandom.current().nextInt(1000);
                //利用computeIfAbsent()方法来实例化LongAdder，然后利用LongAdder来进行线程安全计数
                freqs.computeIfAbsent(key, k -> new LongAdder()).increment();
            }));
            forkJoinPool.shutdown();
            forkJoinPool.awaitTermination(1, TimeUnit.HOURS);
            return freqs.entrySet().stream()
                    .collect(Collectors.toMap(
                            e->e.getKey(),
                            e->e.getValue().longValue())
                    );
    ```

- **没有认清并发工具的使用场景，因而导致性能问题**

  - 在 Java 中，CopyOnWriteArrayList 虽然是一个线程安全的 ArrayList，但因为其实现方式是，每次修改数据时都会复制一份数据出来，所以有明显的适用场景，即读多写少或者说希望无锁读的场景。

- **ThreadLocalRandom是否可以把它的实例设置到静态变量中，在多线程情况下重用呢？**

  - ThreadLocalRandom中存放一个单例的instance，调用current()方法返回这个instance，每个线程首次调用current()方法时，会在各个线程中初始化seed和probe。
  - nextX(）方法会调用nextSeed()，在其中使用各个线程中的种子，计算下一个种子并保存（UNSAFE.getLong(t, SEED) + GAMMA）。
  - 所以，如果使用静态变量，直接调用nextX()方法就跳过了各个线程初始化的步骤，只会在每次调用nextSeed()时来更新种子。

- **computeIfAbsent 和 putIfAbsent 方法的区别**

  - 当Key存在的时候，如果Value获取比较昂贵的话，putIfAbsent就白白浪费时间在获取这个昂贵的Value上（这个点特别注意）
  - Key不存在的时候，putIfAbsent返回null，小心空指针，而computeIfAbsent返回计算后的值
  - 当Key不存在的时候，putIfAbsent允许put null进去，而computeIfAbsent不能（当然了，此条针对HashMap，ConcurrentHashMap不允许put null value进去）

- **ConcurrentHashMap 只能保证提供的原子性读写操作是线程安全的**

  - 线程安全是指多线程访问的操作ConcurrentHashMap，并不会出现状态不一致，数据错乱，异常等问题。
  - 第一，ConcurrentHashMap提供的那些针对单一Key读写的API可以认为是线程安全的，但是诸如putAll这种涉及到多个Key的操作，并发读取可能无法确保读取到完整的数据。
  - 第二，ConcurrentHashMap只能确保提供的API是线程安全的，但是使用者组合使用多个API，ConcurrentHashMap无法从内部确保使用过程中的状态一致。

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