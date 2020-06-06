

## 抽象类和接口

- 两者都是抽象类，都不能实例化。
- 使用抽象类是为了代码的复用，而使用接口的动机是为了实现多态性
- interface是完全抽象的，只能声明pulic的方法，实现类必须要实现不能定义方法体，也不能声明实例变量。abstract class的子类可以有选择地实现，abstract可以声明常量变量。
- abstract class的子类在继承它时，对非抽象方法既可以直接继承，也可以覆盖；而对抽象方法，可以选择实现，也可以通过再次声明其方法为抽象的方式，无需实现，留给其子类来实现，但此类必须也声明为抽象类。





## 动态代理

- 静态代理的实现比较简单：编写一个代理类，实现与目标对象相同的接口，并在内部维护一个目标对象的引用。通过构造器塞入目标对象，在代理对象中调用目标对象的同名方法，并添加前拦截，后拦截等所需的业务功能。

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





## 零拷贝与堆外内存

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

        