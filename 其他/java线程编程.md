

## 有四个线程1、2、3、4，

线程1的功能就是输出1，线程2的功能就是输出2，以此类推......... 现在有四个文件A B C D,初始都为空。现要让四个文件呈如下格式：A：1 2 3 4 1 2..

具体题目如下：

有四个线程1、2、3、4，

线程1的功能就是输出1，线程2的功能就是输出2，

以此类推......... 

现在有四个文件A B C D,

初始都为空。现要让四个文件呈如下格式：

A：1 2 3 4 1 2....

B：2 3 4 1 2 3....

C：3 4 1 2 3 4....

D：4 1 2 3 4 1....

以上就是我看到的一个多线程相关的面试题，看完了 ，就想想怎么实现。

下面就看代码

理论上讲，都是从main方法走起，



```
package com.lxk.threadTest.mianShiTest.googleTest;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * 有四个线程1、2、3、4。
 * <p>
 * 线程1的功能就是输出1，线程2的功能就是输出2，
 * <p>
 * 以此类推......... 现在有四个文件A B C D,
 * 初始都为空。现要让四个文件呈如下格式：
 * A：1 2 3 4 1 2....
 * B：2 3 4 1 2 3....
 * C：3 4 1 2 3 4....
 * D：4 1 2 3 4 1....
 * <p>
 * Created by lxk on 2017/7/14
 */
public class Main {
    public static void main(String[] args) {
        FileWriteUtil util = new FileWriteUtil();
        ExecutorService service = Executors.newCachedThreadPool();
        service.execute(new WriteRunnable(util, 1, '1'));
        service.execute(new WriteRunnable(util, 2, '2'));
        service.execute(new WriteRunnable(util, 3, '3'));
        service.execute(new WriteRunnable(util, 4, '4'));
        service.shutdown();

        //new Thread(new WriteRunnable(util, 1, '1')).start();
        //new Thread(new WriteRunnable(util, 2, '2')).start();
        //new Thread(new WriteRunnable(util, 3, '3')).start();
        //new Thread(new WriteRunnable(util, 4, '4')).start();
    }
}
```



上面关于启动线程，有2中方式，

第一种，也就是未注释的，略显高级点，看类名大概就知道使用的是个线程池的东西。这个实现姿势有很多种。这只是其中的一个。

第二种，也就是下面注释的代码，也不low，是我们常见的启动线程 的方式。



然后就是我们说的那个实现多线程的类的实现啦



```
package com.lxk.threadTest.mianShiTest.googleTest;

/**
 * Created by lxk on 2017/7/14
 */
public class WriteRunnable implements Runnable {
    private final FileWriteUtil util;
    private int threadNum;
    private char value;

    /**
     * @param util      写文件工具类
     * @param threadNum 线程号
     * @param value     写的字符
     */
    public WriteRunnable(FileWriteUtil util, int threadNum, char value) {
        this.util = util;
        this.threadNum = threadNum;
        this.value = value;
    }

    public void run() {
        /*
         * 假设循环6次，一直循环可以使用while(true)或者for(;;)
         */
        for (int i = 0; i < 6; i++) {
            synchronized (util) {
                while (threadNum != util.getCurrentThreadNum()) {
                    try {
                        util.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                util.write(value, threadNum);
                util.notifyAll();
            }
        }
    }
}
```

最后，就是这个写文件的类啦。





```
package com.lxk.threadTest.mianShiTest.googleTest;

import java.io.FileWriter;
import java.io.IOException;

/**
 * 此类，是四个线程共享的，
 * <p>
 * Created by lxk on 2017/7/14
 */
public class FileWriteUtil {
    private int currentThreadNum = 1;
    /**
     * 记录将字符写入文件的次数
     */
    private int count = 0;

    private String currentFileName;

    public void write(char value, int threadNum) {
        getCurrentFileName();
        FileWriter writer = null;
        try {
            //生成文件位置
            writer = new FileWriter("D:/test/test/" + currentFileName + ".txt", true);
            writer.write(value + " ");
            System.out.printf(
                    "ThreadNum=%d is executing. %c is written into file file%s.txt \n",
                    currentThreadNum, value, currentFileName);
            writer.flush();
            //System.out.println(count);//
            count++;
            currentThreadNum = threadNum;
        } catch (IOException e) {
            e.printStackTrace();
        }

        if (null != writer) {
            try {
                writer.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        getNextThreadNum();
    }

    public int getCurrentThreadNum() {
        return currentThreadNum;
    }

    public void setCurrentThreadNum(int currentThreadNum) {
        this.currentThreadNum = currentThreadNum;
    }

    /**
     * 根据写的次数，判断该写哪个文件了？A，B,C,D.
     */
    private void getCurrentFileName() {
        int temp = count % 4;
        switch (temp) {
            case 0:
                currentFileName = "A";
                break;
            case 1:
                currentFileName = "B";
                break;
            case 2:
                currentFileName = "C";
                break;
            case 3:
                currentFileName = "D";
                break;
            default:
                currentFileName = "E";
        }
    }

    private void getNextThreadNum() {
        if (count % 4 == 0) {
            if (currentThreadNum < 3) {
                currentThreadNum += 2;
            } else {
                currentThreadNum = (currentThreadNum + 2) % 4;
            }
        } else {
            if (currentThreadNum == 4) {
                currentThreadNum = 1;
            } else {
                currentThreadNum++;
            }
        }
    }
}
```

最后，就是看下代码实际运行的结果。



![img](http://img.voidcn.com/vcimg/static/loading.png)

从打印结果看，看到了是四个线程在跑，而且分别写入到ABCD四个文件去。

![img](http://img.voidcn.com/vcimg/static/loading.png)

注意，图中的红线框，奥，也可以不注意啦。

其实，说得最透彻点，就是这四个线程，轮着执行，

每次，都是有一个线程可以执行，这的执行也就是写文件啦，然后其他的三个都稍息，也是wait()啦。

等这个线程执行完毕之后，也就是写文件完毕之后，唤醒另外三个在wait()的线程，然后，设置一下，下一个可以处于运行态的线程，然后又开始重复了。

不能执行的，都wait()，如此往复。

这里有个需要注意的是。

这四个线程都在共享操作的就是那个写文件工具类。也就是对这个上锁。四个线程用的是一个锁，那就可以保证线程安全啦。

理论，是这个理论，但是，真让你写，可不一定能分分钟就写好。



## 三个线程按顺序输出1,2,3

```java
import java.util.concurrent.locks.ReentrantLock;

/**
 * 标题、简要说明. <br>
 * 类详细说明.
 * <p>
 * Copyright: Copyright (c) 2014年11月6日 上午9:50:41
 * <p>
 * Company: 
 * <p>
 * 
 * @author 
 * @version 1.0.0
 */
public class TestLock {

	private static int state = 0;

	/**
	 * @param args
	 */
	public static void main(String[] args) {

		final ReentrantLock lock = new ReentrantLock();

		// thread1
		Thread t1 = new Thread(new Runnable() {

			@Override
			public void run() {
				while (state <= 30) {
					try {
						// 加锁
						lock.lock();
						if (state % 3 == 0) {
							System.out.print("1");
							state++;
						}
					}
					finally {
						lock.unlock();
					}

				}
			}
		});

		// thread2
		Thread t2 = new Thread(new Runnable() {

			@Override
			public void run() {
				while (state <= 30) {
					try {
						// 加锁
						lock.lock();
						if (state % 3 == 1) {
							System.out.print("2");
							state++;
						}
					}
					finally {
						lock.unlock();
					}
				}
			}
		});

		// thread3
		Thread t3 = new Thread(new Runnable() {

			@Override
			public void run() {
				while (state <= 30) {
					try {
						// 加锁
						lock.lock();
						if (state % 3 == 2) {
							System.out.println("3");
							state++;
						}
					}
					finally {
						lock.unlock();
					}
				}
			}
		});

		t1.start();
		t2.start();
		t3.start();
	}

}
```

## 



## 两个线程，循环输出1~100



```
public class Test {  
    //state==1表示线程1开始打印，state==2表示线程2开始打印  
    private static int state = 1;  
      
    private static int num1 = 1;  
    private static int num2 = 2;  
      
    public static void main(String[] args) {  
        final Test t = new Test();  
        new Thread(new Runnable() {  
            @Override  
            public void run() {  
                while(num1<100){  
                    //两个线程都用t对象作为锁，保证每个交替期间只有一个线程在打印  
                    synchronized (t) {  
                        // 如果state!=1, 说明此时尚未轮到线程1打印, 线程1将调用t的wait()方法, 直到下次被唤醒  
                        if(state!=1){  
                            try {  
                                t.wait();  
                            } catch (InterruptedException e) {  
                                e.printStackTrace();  
                            }  
                        }  
                         // 当state=1时, 轮到线程1打印5次数字  
                        for(int j=0; j<5; j++){  
                            System.out.println(num1);  
                            num1 += 2;  
                        }  
                        // 线程1打印完成后, 将state赋值为2, 表示接下来将轮到线程2打印  
                        state = 2;  
                        // notifyAll()方法唤醒在t上wait的线程2, 同时线程1将退出同步代码块, 释放t锁  
                        t.notifyAll();  
                    }  
                }  
            }  
        }).start();  
          
        new Thread(new Runnable() {  
            @Override  
            public void run() {  
                while(num2<100){  
                    synchronized (t) {  
                        if(state!=2){  
                            try {  
                                t.wait();  
                            } catch (InterruptedException e) {  
                                e.printStackTrace();  
                            }  
                        }  
                        for(int j=0; j<5; j++){  
                            System.out.println(num2);  
                            num2 += 2;  
                        }  
                        state = 1;  
                        t.notifyAll();  
                    }  
                }  
            }  
        }).start();  
    }  
  
}  
```





## 三个线程实现输出ABCABC循环

```
//标记类，用来让三个线程共享，同时也是三个线程中同步代码快的标记对象。
//之前这个标记我设置成Integer，但是发现Integer进行加法运算时会改变对
//象引用（原因是自动装箱），因此出现异常抛出。所以索性自己定义Flag类。
class Flag{
    int i=0;
    public synchronized void  setI()
    {
        i++;
        if(i==3)
            i=0;
    }

}
//输出A的线程
class SafeTestA implements Runnable{
    int num=10;
    Flag flag;
    public SafeTestA(Flag flag)
    {
        this.flag=flag;
    }
    public void run()
    {
        synchronized(flag){
            while(true)
            {
                if(flag.i==0){
                    System.out.println("A");
                    flag.setI();
                    flag.notifyAll();
                }else{
                    try {
                        flag.wait();
                    } catch (InterruptedException e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                    }
                }
            }
        }

    }
    /*public static void main(String[] args) { Runnable sf=new SafeTest(); new Thread(sf).start(); new Thread(sf).start(); new Thread(sf).start(); }*/
}
//输出B的线程：
class SafeTestB implements Runnable{
    Flag flag;
    public SafeTestB(Flag flag)
    {
        this.flag=flag;
    }
    public void run()
    {
        while(true)
        {
            synchronized(flag){
                while(true)
                {
                    if(flag.i==1){
                        System.out.println("B");
                        flag.setI();
                        flag.notifyAll();
                    }else{
                        try {
                            flag.wait();
                        } catch (InterruptedException e) {
                            // TODO Auto-generated catch block
                            e.printStackTrace();
                        }
                    }
                }
            }
        }
    }

}

//输出C的线程
public class SafeTestC implements Runnable{
    Flag flag;
    public SafeTestC(Flag flag)
    {
        this.flag=flag;
    }
    public void run()
    {
        synchronized(flag){
            while(true)
            {
                if(flag.i==2){
                    System.out.println("C");
                    flag.setI();
                    flag.notifyAll();
                }else{
                    try {
                        flag.wait();
                    } catch (InterruptedException e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                    }
                }
            }
        }
    }

    public static void main(String[] args) {
        Flag flag=new Flag();
        new Thread(new SafeTestA(flag)).start();
        new Thread(new SafeTestB(flag)).start();
        new Thread(new SafeTestC(flag)).start();

    }
}
```





## 四个线程，要求顺序打印01010101

四个线程，两个执行i++，两个执行i--，初始变量1，要求顺序打印01010101







## 三个线程有先后顺序

我们提供了一个类：

```
public class Foo {
  public void one() { print("one"); }
  public void two() { print("two"); }
  public void three() { print("three"); }
}
```

三个不同的线程将会共用一个 `Foo` 实例。

- 线程 A 将会调用 `one()` 方法
- 线程 B 将会调用 `two()` 方法
- 线程 C 将会调用 `three()` 方法

请设计修改程序，以确保 `two()` 方法在 `one()` 方法之后被执行，`three()` 方法在 `two()` 方法之后被执行。



我们需要构造 2 道**屏障**，`second` 线程等待 `first` 屏障，`third` 线程等待 `second` 屏障。：first 线程会释放 first 屏障，而 second 线程会释放 second 屏障。

Java 中，我们使用线程等待的方式实现执行屏障，使用释放线程等待的方式实现屏障消除。具体代码如下：
代码：

class Foo {
    
```java
private boolean firstFinished;
private boolean secondFinished;
private Object lock = new Object();

public Foo() {
    
}

public void first(Runnable printFirst) throws InterruptedException {
    
    synchronized (lock) {
        // printFirst.run() outputs "first". Do not change or remove this line.
        printFirst.run();
        firstFinished = true;
        lock.notifyAll(); 
    }
}

public void second(Runnable printSecond) throws InterruptedException {
    
    synchronized (lock) {
        while (!firstFinished) {
            lock.wait();
        }
    
        // printSecond.run() outputs "second". Do not change or remove this line.
        printSecond.run();
        secondFinished = true;
        lock.notifyAll();
    }
}

public void third(Runnable printThird) throws InterruptedException {
    
    synchronized (lock) {
       while (!secondFinished) {
            lock.wait();
        }

        // printThird.run() outputs "third". Do not change or remove this line.
        printThird.run();
    } 
}
}
```
我们使用一个 Ojbect 对象 lock 实现所有执行屏障的锁对象，两个布尔型对象 firstFinished，secondFinished 保存屏障消除的条件。



**volatile的方式**

```cs
class Foo {

    public Foo() {
        
    }
    volatile int count=1;
    public void first(Runnable printFirst) throws InterruptedException {
        printFirst.run();
        count++;
    }

    public void second(Runnable printSecond) throws InterruptedException {
        while (count!=2);
        printSecond.run();
        count++;
    }

    public void third(Runnable printThird) throws InterruptedException {
        while (count!=3);
        printThird.run();
    }
}
```





**countDownLatch**

countDownLatch这个类使一个线程等待其他线程各自执行完毕后再执行。是通过一个计数器来实现的，计数器的初始值是线程的数量。每当一个线程执行完毕后，计数器的值就-1，当计数器的值为0时，表示所有线程都执行完毕，然后在闭锁上等待的线程就可以恢复工作了。

1:首先考虑使用一个线程安全的线程标记，选择了countDownLatch计数器。
2:根据题目描述就是在线程1,2,3不论谁先执行，都要使执行结果为first - second - third
3:分析countDownLatch计数器，他的作用就是初始化一个计数器数值指定，只有在该计数器的值为0时才继续执行。
1、我们先创建两个计数器 c2 是second的，c3是third 初始化都为1.
2、我们可以使用它的.countDown（解释：使当前计数器减一）方法在first执行后将c2的计数器置为0，在c2执行 后通过.countDown方法将c3的计数器置为0。
3、同时在应对如果 second、third先执行，在他们之中.await（解释：如果当前计数器不为0，那么挂起，等待 计数器为0后继续执行）自己的计数器。
4：问题解决。
代码

class Foo {
    private CountDownLatch c2;
    private CountDownLatch c3;
    public Foo() {
         c2 = new CountDownLatch(1);
         c3 = new CountDownLatch(1);
    }
    
```java
public void first(Runnable printFirst) throws InterruptedException {
    
    // printFirst.run() outputs "first". Do not change or remove this line.
    printFirst.run();
    c2.countDown();
}

public void second(Runnable printSecond) throws InterruptedException {
    c2.await();
    // printSecond.run() outputs "second". Do not change or remove this line.
    printSecond.run();
    c3.countDown();
}

public void third(Runnable printThird) throws InterruptedException {
    c3.await();
    // printThird.run() outputs "third". Do not change or remove this line.
    printThird.run();
}
```
}

**利用ReentrantLock和Condition实现利用ReentrantLock和Condition实现**

class Foo {

```java
private ReentrantLock lock = new ReentrantLock();
private Condition condition = lock.newCondition();
private int count = 1;
public Foo() {}

public void first(Runnable printFirst) throws InterruptedException {
    lock.lock();
    // printFirst.run() outputs "first". Do not change or remove this line.
    printFirst.run();
    count = 2;
    //其实这里调用signal是有问题
    //假设这里唤醒第三个线程
    //第三个线程会再次调用await,进入等待池等待唤醒
    //第二个线程此时也在等待池中
    //这样就会导致第一个线程和第二个线程都在等待池中
    //condition.signal(); 
    condition.signalAll();
    lock.unlock();
}

public void second(Runnable printSecond) throws InterruptedException {
    lock.lock();
    //为什么这里可以使用if,而下面必须使用while呢
    //如果线程一运行完,线程三拿到了锁,则需要再次判断count值让其进入等待池
    if(count != 2) {
        condition.await();
    }
    // printSecond.run() outputs "second". Do not change or remove this line.
    printSecond.run();
    count = 3;
    //这里可以调用signal方法,因为只剩下一个线程在等待了
    condition.signal();
    lock.unlock();
}

public void third(Runnable printThird) throws InterruptedException {
    lock.lock();
    while(count != 3) {
        condition.await();
    }
    // printThird.run() outputs "third". Do not change or remove this line.
    printThird.run();
    lock.unlock();
}
```


## 交替打印

我们提供一个类：

class FooBar {
  public void foo() {
    for (int i = 0; i < n; i++) {
      print("foo");
    }
  }

  public void bar() {
    for (int i = 0; i < n; i++) {
      print("bar");
    }
  }
}

两个不同的线程将会共用一个 FooBar 实例。其中一个线程将会调用 foo() 方法，另一个线程将会调用 bar() 方法。

**请设计修改程序，以确保 "foobar" 被输出 n 次。**



**利用ReentrantLock和Condition实现**

```java
class FooBar {
    private ReentrantLock lock = new ReentrantLock();
    private Condition fooCondition = lock.newCondition();
    private Condition barCondition = lock.newCondition();
    private int count = 1;
    private int n;
    
public FooBar(int n) {
    this.n = n;
}

public void foo(Runnable printFoo) throws InterruptedException {        
    for (int i = 0; i < n; i++) {
        lock.lock();
        if(count != 1) {
            fooCondition.await();
        }
        // printFoo.run() outputs "foo". Do not change or remove this line.
    	printFoo.run();
        barCondition.signal();
        count=2;
        lock.unlock();
    }
}

public void bar(Runnable printBar) throws InterruptedException {
    for (int i = 0; i < n; i++) {
        lock.lock();
        if(count != 2) {
            barCondition.await();
        }
        // printBar.run() outputs "bar". Do not change or remove this line.
    	printBar.run();
        fooCondition.signal();
        count=1;
        lock.unlock(); 
    }
}
}
```


**CountDownLatch和一个CyclicBarrier**





```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.CyclicBarrier;
class FooBar {
    private int n;
    private CountDownLatch a;
    private CyclicBarrier barrier;// 使用CyclicBarrier保证任务按组执行
    public FooBar(int n) {
        this.n = n;
        a = new CountDownLatch(1);
        barrier = new CyclicBarrier(2);// 保证每组内有两个任务
    }
public void foo(Runnable printFoo) throws InterruptedException {

    try {
        for (int i = 0; i < n; i++) {
            printFoo.run();
            a.countDown();// printFoo方法完成调用countDown
            barrier.await();// 等待printBar方法执行完成
        }
    } catch(Exception e) {}
}

public void bar(Runnable printBar) throws InterruptedException {

    try {
        for (int i = 0; i < n; i++) {
            a.await();// 等待printFoo方法先执行
            printBar.run();
            a = new CountDownLatch(1); // 保证下一次依旧等待printFoo方法先执行
            barrier.await();// 等待printFoo方法执行完成
        }
    } catch(Exception e) {}
}
}
```


**使用阻塞队列来控制**

```java
class FooBar {
    private int n;

    public FooBar(int n) {
        this.n = n;
    }
    private BlockingDeque deque = new LinkedBlockingDeque(1);

    public void foo(Runnable printFoo) throws InterruptedException {

        for (int i = 0; i < n; i++) {
            deque.put(i);
            // printFoo.run() outputs "foo". Do not change or remove this line.
            printFoo.run();
            deque.put(i);
            deque.put(i);
        }
    }

    public void bar(Runnable printBar) throws InterruptedException {

        for (int i = 0; i < n; i++) {
            deque.take();
            deque.take();
            // printBar.run() outputs "bar". Do not change or remove this line.
            printBar.run();
            deque.take();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        FooBar t = new FooBar(20);
        Thread foo = new Thread(() -> {
            try {
                t.foo(() -> System.out.print("foo"));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        foo.setDaemon(false);
        foo.start();

        Thread bar = new Thread(() -> {
            try {
                t.bar(() -> System.out.print("bar"));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        bar.setDaemon(false);
        bar.start();
    }
}


```





## 打印零与奇偶数

假设有这么一个类：

```
class ZeroEvenOdd {
  public ZeroEvenOdd(int n) { ... }      // 构造函数
  public void zero(printNumber) { ... }  // 仅打印出 0
  public void even(printNumber) { ... }  // 仅打印出 偶数
  public void odd(printNumber) { ... }   // 仅打印出 奇数
}
```

相同的一个 `ZeroEvenOdd` 类实例将会传递给三个不同的线程：

1. 线程 A 将调用 `zero()`，它只输出 0 。
2. 线程 B 将调用 `even()`，它只输出偶数。
3. 线程 C 将调用 `odd()`，它只输出奇数。

每个线程都有一个 `printNumber` 方法来输出一个整数。请修改给出的代码以输出整数序列 `010203040506`... ，其中序列的长度必须为 2*n*。







通过三个信号量来控制
zero方法中的for表示要输出的0个次数，同时用来控制要唤醒偶数还是奇数方法
even方法用来输出偶数同时唤醒zero方法
odd方法用来输出奇数同时唤醒zero方法
代码



```java
import java.util.concurrent.Semaphore;
import java.util.function.IntConsumer;

class ZeroEvenOdd {
    private int n;
    private Semaphore zero = new Semaphore(1);
    private Semaphore even = new Semaphore(0);
    private Semaphore odd = new Semaphore(0);
public ZeroEvenOdd(int n) {
    this.n = n;
}

// printNumber.accept(x) outputs "x", where x is an integer.
public void zero(IntConsumer printNumber) throws InterruptedException {
    for (int i=1;i<=n;i++){
        zero.acquire();
        printNumber.accept(0);
        if(i%2==1){
            odd.release();
        }else{
            even.release();
        }
    }
}

public void even(IntConsumer printNumber) throws InterruptedException {
    for (int i=2;i<=n;i+=2){
        even.acquire();
        printNumber.accept(i);
        zero.release();
    }
}

public void odd(IntConsumer printNumber) throws InterruptedException {
    for (int i=1;i<=n;i+=2){
        odd.acquire();
        printNumber.accept(i);
        zero.release();
    }
}

public static void main(String[] args) {
    ZeroEvenOdd zeroEvenOdd = new ZeroEvenOdd(6);
    new Thread(() -> {
        try {
            zeroEvenOdd.zero(System.out::print);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }).start();
    new Thread(() -> {
        try {
            zeroEvenOdd.even(System.out::print);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }).start();
    new Thread(() -> {
        try {
            zeroEvenOdd.odd(System.out::print);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }).start();
}
```
}



## H20

现在有两种线程，氧 oxygen 和氢 hydrogen，你的目标是组织这两种线程来产生水分子。

存在一个屏障（barrier）使得每个线程必须等候直到一个完整水分子能够被产生出来。

氢和氧线程会被分别给予 releaseHydrogen 和 releaseOxygen 方法来允许它们突破屏障。

这些线程应该三三成组突破屏障并能立即组合产生一个水分子。

你必须保证产生一个水分子所需线程的结合必须发生在下一个水分子产生之前。

换句话说:

    如果一个氧线程到达屏障时没有氢线程到达，它必须等候直到两个氢线程到达。
    如果一个氢线程到达屏障时没有其它线程到达，它必须等候直到一个氧线程和另一个氢线程到达。

书写满足这些限制条件的氢、氧线程同步代码。

 

示例 1:

输入: "HOH"
输出: "HHO"
解释: "HOH" 和 "OHH" 依然都是有效解。

示例 2:

输入: "OOHHHH"
输出: "HHOHHO"
解释: "HOHHHO", "OHHHHO", "HHOHOH", "HOHHOH", "OHHHOH", "HHOOHH", "HOHOHH" 和 "OHHOHH" 依然都是有效解。

 

提示：

    输入字符串的总长将会是 3n, 1 ≤ n ≤ 50；
    输入字符串中的 “H” 总数将会是 2n 。
    输入字符串中的 “O” 总数将会是 n 。



关键在于H和O如何互相等待互相通知，利用信号量可以解决，代码如下。


```JAVA
import java.util.concurrent.*;

class H2O {
    
private Semaphore s1,s2,s3,s4;

public H2O() {
    s1 = new Semaphore(2); // H线程信号量
    s2 = new Semaphore(1); // O线程信号量
    
    s3 = new Semaphore(0); // 反应条件信号量
    s4 = new Semaphore(0); // 反应条件信号量
}

public void hydrogen(Runnable releaseHydrogen) throws InterruptedException {
    s1.acquire(); // 保证只有2个H线程进入执行
    s3.release(); // 释放H原子到达信号
    s4.acquire(); // 等待O原子到达
    releaseHydrogen.run();
    s1.release(); // 相当于唤醒1个H线程
}

public void oxygen(Runnable releaseOxygen) throws InterruptedException {
    s2.acquire(); // 保证只有1个O线程进入执行
    s4.release(2); // 释放O原子到达信号，因为有2个H线程等待所以释放2个
    s3.acquire(2); // 等待H原子到达，2个原因同上
    releaseOxygen.run();
    s2.release(); // 相当于唤醒1个O线程
}
}
```


## 交替打印字符串

编写一个可以从 1 到 n 输出代表这个数字的字符串的程序，但是：

    如果这个数字可以被 3 整除，输出 "fizz"。
    如果这个数字可以被 5 整除，输出 "buzz"。
    如果这个数字可以同时被 3 和 5 整除，输出 "fizzbuzz"。

例如，当 n = 15，输出： 1, 2, fizz, 4, buzz, fizz, 7, 8, fizz, buzz, 11, fizz, 13, 14, fizzbuzz。

假设有这么一个类：

class FizzBuzz {
  public FizzBuzz(int n) { ... }               // constructor
  public void fizz(printFizz) { ... }          // only output "fizz"
  public void buzz(printBuzz) { ... }          // only output "buzz"
  public void fizzbuzz(printFizzBuzz) { ... }  // only output "fizzbuzz"
  public void number(printNumber) { ... }      // only output the numbers
}

请你实现一个有四个线程的多线程版  FizzBuzz， 同一个 FizzBuzz 实例会被如下四个线程使用：

    线程A将调用 fizz() 来判断是否能被 3 整除，如果可以，则输出 fizz。
    线程B将调用 buzz() 来判断是否能被 5 整除，如果可以，则输出 buzz。
    线程C将调用 fizzbuzz() 来判断是否同时能被 3 和 5 整除，如果可以，则输出 fizzbuzz。
    线程D将调用 number() 来实现输出既不能被 3 整除也不能被 5 整除的数字。





    private static CyclicBarrier barrier = new CyclicBarrier(4);
    public FizzBuzz(int n) {
        this.n = n;
    }
    
    // printFizz.run() outputs "fizz".
    public void fizz(Runnable printFizz) throws InterruptedException {
        for (int i = 1; i <= n; i++) {
            if (i % 3 == 0 && i % 5 != 0) {
                printFizz.run();
            }
            try {
                barrier.await();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
    
    // printBuzz.run() outputs "buzz".
    public void buzz(Runnable printBuzz) throws InterruptedException {
        for (int i = 1; i <= n; i++) {
            if (i % 3 != 0 && i % 5 == 0) {
                printBuzz.run();
            }
            try {
                barrier.await();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
    
    // printFizzBuzz.run() outputs "fizzbuzz".
    public void fizzbuzz(Runnable printFizzBuzz) throws InterruptedException {
        for (int i = 1; i <= n; i++) {
            if (i % 3 == 0 && i % 5 == 0) {
                printFizzBuzz.run();
            }
            try {
                barrier.await();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
    
    // printNumber.accept(x) outputs "x", where x is an integer.
    public void number(IntConsumer printNumber) throws InterruptedException {
        for (int i = 1; i <= n; i++) {
            if (i % 3 != 0 && i % 5 != 0) {
                printNumber.accept(i);
            }
            try {
                barrier.await();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
    }


这个思路同样适用于 Leetcode-1116 打印零与奇偶数
思路

首先理解volatile关键字的底层原理，不知道的建议百度。
其次就是理解 给变量 赋 字面量值的操作可认为是原子性的。

volatile

```java
class FizzBuzz {
    private int n;
    private volatile int state = -1;
public FizzBuzz(int n) {
    this.n = n;
}

public void fizz(Runnable printFizz) throws InterruptedException {
    for (int i = 3; i <= n; i += 3) {   //只输出3的倍数(不包含15的倍数)
        if (i % 15 == 0)    //15的倍数不处理，交给fizzbuzz()方法处理
            continue;
        while (state != 3)
            Thread.sleep(1);

        printFizz.run();
        state = -1;    //控制权交还给number()方法
    }
}

public void buzz(Runnable printBuzz) throws InterruptedException {
    for (int i = 5; i <= n; i += 5) {   //只输出5的倍数(不包含15的倍数)
        if (i % 15 == 0)    //15的倍数不处理，交给fizzbuzz()方法处理
            continue;
        while (state != 5)
            Thread.sleep(1);

        printBuzz.run();
        state = -1;    //控制权交还给number()方法
    }
}

public void fizzbuzz(Runnable printFizzBuzz) throws InterruptedException {
    for (int i = 15; i <= n; i += 15) {   //只输出15的倍数
        while (state != 15)
            Thread.sleep(1);

        printFizzBuzz.run();
        state = -1;    //控制权交还给number()方法
    }
}

public void number(IntConsumer printNumber) throws InterruptedException {
    for (int i = 1; i <= n; ++i) {
        while (state != -1)
            Thread.sleep(1);

        if (i % 3 != 0 && i % 5 != 0)
            printNumber.accept(i);
        else {
            if (i % 15 == 0)
                state = 15;    //交给fizzbuzz()方法处理
            else if (i % 5 == 0)
                state = 5;    //交给buzz()方法处理
            else
                state = 3;    //交给fizz()方法处理
        }
    }
}
```
}



reentrantlock+condition

```java
class FizzBuzz {
    private int n;
    private int state = -1;
    private Lock lock = new ReentrantLock();
    private Condition cond = lock.newCondition();
public FizzBuzz(int n) {
    this.n = n;
}

// printFizz.run() outputs "fizz".
public void fizz(Runnable printFizz) throws InterruptedException {
    for (int i = 3; i <= n; i += 3) {   //只输出3的倍数(不包含15的倍数)
        if (i % 15 == 0)    //15的倍数不处理，交给fizzbuzz()方法处理
            continue;
        lock.lock();
        while (state != 3)
            cond.await();

        printFizz.run();
        state = -1;    //控制权交还给number()方法
        cond.signalAll();    //全体起立
        lock.unlock();
    }
}

// printBuzz.run() outputs "buzz".
public void buzz(Runnable printBuzz) throws InterruptedException {
    for (int i = 5; i <= n; i += 5) {   //只输出5的倍数(不包含15的倍数)
        if (i % 15 == 0)    //15的倍数不处理，交给fizzbuzz()方法处理
            continue;
        lock.lock();
        while (state != 5)
            cond.await();

        printBuzz.run();
        state = -1;    //控制权交还给number()方法
        cond.signalAll();    //全体起立
        lock.unlock();
    }
}

// printFizzBuzz.run() outputs "fizzbuzz".
public void fizzbuzz(Runnable printFizzBuzz) throws InterruptedException {
    for (int i = 15; i <= n; i += 15) {   //只输出15的倍数
        lock.lock();
        while (state != 15)
            cond.await();

        printFizzBuzz.run();
        state = -1;    //控制权交还给number()方法
        cond.signalAll();    //全体起立
        lock.unlock();

    }
}

// printNumber.accept(x) outputs "x", where x is an integer.
public void number(IntConsumer printNumber) throws InterruptedException {
    for (int i = 1; i <= n; ++i) {
        lock.lock();
        while (state != -1)
            cond.await();

        if (i % 3 != 0 && i % 5 != 0)
            printNumber.accept(i);
        else {
            if (i % 15 == 0)
                state = 15;    //交给fizzbuzz()方法处理
            else if (i % 5 == 0)
                state = 5;    //交给buzz()方法处理
            else
                state = 3;    //交给fizz()方法处理

            cond.signalAll();    //全体起立
        }
        lock.unlock();
    }
}
```
}



## 哲学家进餐

哲学家从 0 到 4 按 顺时针 编号。请实现函数 void wantsToEat(philosopher, pickLeftFork, pickRightFork, eat, putLeftFork, putRightFork)：

    philosopher 哲学家的编号。
    pickLeftFork 和 pickRightFork 表示拿起左边或右边的叉子。
    eat 表示吃面。
    putLeftFork 和 putRightFork 表示放下左边或右边的叉子。
    由于哲学家不是在吃面就是在想着啥时候吃面，所以思考这个方法没有对应的回调。

给你 5 个线程，每个都代表一个哲学家，请你使用类的同一个对象来模拟这个过程。在最后一次调用结束之前，可能会为同一个哲学家多次调用该函数。



这道题本质上其实是想考察如何避免死锁。
易知：当 5 个哲学家都拿着其左边(或右边)的叉子时，会进入死锁。

PS：死锁的 4 个必要条件：

    互斥条件：一个资源每次只能被一个进程使用，即在一段时间内某 资源仅为一个进程所占有。此时若有其他进程请求该资源，则请求进程只能等待。
    请求与保持条件：进程已经保持了至少一个资源，但又提出了新的资源请求，而该资源 已被其他进程占有，此时请求进程被阻塞，但对自己已获得的资源保持不放。
    不可剥夺条件:进程所获得的资源在未使用完毕之前，不能被其他进程强行夺走，即只能 由获得该资源的进程自己来释放（只能是主动释放)。
    循环等待条件: 若干进程间形成首尾相接循环等待资源的关系。

故最多只允许 4 个哲学家去持有叉子，可保证至少有 1 个哲学家能吃上意大利面（即获得到 2 个叉子）。
因为最差情况下是：4 个哲学家都各自持有1个叉子，此时还 剩余 1 个叉子 可供使用，这 4 个哲学家中必然有1人能获取到这个 剩余的 1 个叉子，从而手持 2 个叉子，可以吃意大利面。
即：4 个人中，1 个人有 2 个叉子，3 个人各持 1 个叉子，共计 5个叉子。

既然最多只允许4个哲学家去持有叉子，那么如果只允许3个哲学家去持有叉子是否可行呢？

当然可行，3个哲学家可以先都各自持有1把叉子，此时还剩余2把叉子；

当这333个哲学家刚好都相邻(比如：编号为图中的0, 1, 2)，可能会造成只有111个哲学家能吃到意面的情况，具体而言即0号哲学家拿到了其左侧的叉子(编号为1)，1号哲学家也拿到了其左侧的叉子(编号为2)，2号哲学家也拿到了其左侧的叉子(编号为3)，此时只有0号哲学家能拿到其右侧的叉子(编号为0)，因此只有0号哲学家能吃到意面。
而其余情况下，3个哲学家中都能有2人吃到意面。
即：3 个人中，2个人各自持有 2 个叉子，1 个人持有 1 个叉子，共计 5 个叉子。

并且仔细想想，叉子的数目是固定的(个数为5)，直觉上来讲3个人去抢5个叉子 比 4个人去抢5个叉子效率高。

用Semaphore去实现上述的限制：Semaphore eatLimit = new Semaphore(4);
一共有5个叉子，视为5个ReentrantLock，并将它们全放入1个数组中。

给叉子编号 0,1,2,3,40, 1, 2, 3, 40,1,2,3,4（对应数组下标）



示例：

输入：n = 1
输出：[[4,2,1],[4,1,1],[0,1,1],[2,2,1],[2,1,1],[2,0,3],[2,1,2],[2,2,2],[4,0,3],[4,1,2],[0,2,1],[4,2,2],[3,2,1],[3,1,1],[0,0,3],[0,1,2],[0,2,2],[1,2,1],[1,1,1],[3,0,3],[3,1,2],[3,2,2],[1,0,3],[1,1,2],[1,2,2]]
解释:
n 表示每个哲学家需要进餐的次数。
输出数组描述了叉子的控制和进餐的调用，它的格式如下：
output[i] = [a, b, c] (3个整数)
- a 哲学家编号。
- b 指定叉子：{1 : 左边, 2 : 右边}.
- c 指定行为：{1 : 拿起, 2 : 放下, 3 : 吃面}。
如 [4,2,1] 表示 4 号哲学家拿起了右边的叉子。



代码具体实现：



```java
class DiningPhilosophers {
    //1个Fork视为1个ReentrantLock，5个叉子即5个ReentrantLock，将其都放入数组中
    private final ReentrantLock[] lockList = {new ReentrantLock(),
            new ReentrantLock(),
            new ReentrantLock(),
            new ReentrantLock(),
            new ReentrantLock()};
//限制 最多只有4个哲学家去持有叉子
private Semaphore eatLimit = new Semaphore(4);

public DiningPhilosophers() {

}

// call the run() method of any runnable to execute its code
public void wantsToEat(int philosopher,
                       Runnable pickLeftFork,
                       Runnable pickRightFork,
                       Runnable eat,
                       Runnable putLeftFork,
                       Runnable putRightFork) throws InterruptedException {

    int leftFork = (philosopher + 1) % 5;    //左边的叉子 的编号
    int rightFork = philosopher;    //右边的叉子 的编号

    eatLimit.acquire();    //限制的人数 -1

    lockList[leftFork].lock();    //拿起左边的叉子
    lockList[rightFork].lock();    //拿起右边的叉子

    pickLeftFork.run();    //拿起左边的叉子 的具体执行
    pickRightFork.run();    //拿起右边的叉子 的具体执行

    eat.run();    //吃意大利面 的具体执行

    putLeftFork.run();    //放下左边的叉子 的具体执行
    putRightFork.run();    //放下右边的叉子 的具体执行

    lockList[leftFork].unlock();    //放下左边的叉子
    lockList[rightFork].unlock();    //放下右边的叉子

    eatLimit.release();//限制的人数 +1
}
```
}



他是用C++实现的，将其转为Java代码如下：

方法 2：
设置 111 个临界区以实现 111 个哲学家 “同时”拿起左右 222 把叉子的效果。
即进入临界区之后，保证成功获取到左右 222 把叉子 并 执行相关代码后，才退出临界区。

评论区看到有题友说方法2就是“只让1个哲学家就餐”的思路，无需将叉子视为ReentrantLock。

下面我也给出了“只允许1个哲学家就餐”的代码。

但是2者之间还是有细微的差别：
方法2是在成功拿起左右叉子之后就退出临界区，而“只让1个哲学家就餐”是在拿起左右叉子 + 吃意面 + 放下左右叉子 一套流程走完之后才退出临界区。

前者的情况可大概分为2种，举具体例子说明(可参照上面给出的图片)：

    1号哲学家拿起左右叉子(1号叉子 + 2号叉子)后就退出临界区，此时4号哲学家成功挤进临界区，他也成功拿起了左右叉子(0号叉子和4号叉子)，然后就退出临界区。
    1号哲学家拿起左右叉子(1号叉子 + 2号叉子)后就退出临界区，此时2号哲学家成功挤进临界区，他需要拿起2号叉子和3号叉子，但2号叉子有一定的概率还被1号哲学家持有(1号哲学家意面还没吃完)，因此2号哲学家进入临界区后还需要等待2号叉子。至于3号叉子，根本没其他人跟2号哲学家争夺，因此可以将该种情况视为“2号哲学家只拿起了1只叉子，在等待另1只叉子”的情况。

总之，第1种情况即先后进入临界区的2位哲学家的左右叉子不存在竞争情况，因此先后进入临界区的2位哲学家进入临界区后都不用等待叉子，直接就餐。此时可视为2个哲学家在同时就餐(当然前1个哲学家有可能已经吃完了，但姑且当作是2个人同时就餐)。

第2种情况即先后进入临界区的2位哲学家的左右叉子存在竞争情况(说明这2位哲学家的编号相邻)，因此后进入临界区的哲学家还需要等待1只叉子，才能就餐。此时可视为只有1个哲学家在就餐。

至于“只允许1个哲学家就餐”的代码，很好理解，每次严格地只让1个哲学家就餐，由于过于严格，以至于都不需要将叉子视为ReentrantLock。

方法2有一定的概率是“并行”，“只允许1个哲学家就餐”是严格的“串行”。

代码如下：



```java
class DiningPhilosophers {
    //1个Fork视为1个ReentrantLock，5个叉子即5个ReentrantLock，将其都放入数组中
    private final ReentrantLock[] lockList = {new ReentrantLock(),
            new ReentrantLock(),
            new ReentrantLock(),
            new ReentrantLock(),
            new ReentrantLock()};
//让 1个哲学家可以 “同时”拿起2个叉子(搞个临界区)
private ReentrantLock pickBothForks = new ReentrantLock();

public DiningPhilosophers() {

}

// call the run() method of any runnable to execute its code
public void wantsToEat(int philosopher,
                       Runnable pickLeftFork,
                       Runnable pickRightFork,
                       Runnable eat,
                       Runnable putLeftFork,
                       Runnable putRightFork) throws InterruptedException {

    int leftFork = (philosopher + 1) % 5;    //左边的叉子 的编号
    int rightFork = philosopher;    //右边的叉子 的编号

    pickBothForks.lock();    //进入临界区

    lockList[leftFork].lock();    //拿起左边的叉子
    lockList[rightFork].lock();    //拿起右边的叉子

    pickLeftFork.run();    //拿起左边的叉子 的具体执行
    pickRightFork.run();    //拿起右边的叉子 的具体执行

    pickBothForks.unlock();    //退出临界区

    eat.run();    //吃意大利面 的具体执行

    putLeftFork.run();    //放下左边的叉子 的具体执行
    putRightFork.run();    //放下右边的叉子 的具体执行

    lockList[leftFork].unlock();    //放下左边的叉子
    lockList[rightFork].unlock();    //放下右边的叉子
}
```
}

方法 3：
前面说过，该题的本质是考察 如何避免死锁。
而当5个哲学家都左手持有其左边的叉子 或 当5个哲学家都右手持有其右边的叉子时，会发生死锁。
故只需设计1个避免发生上述情况发生的策略即可。

即可以让一部分哲学家优先去获取其左边的叉子，再去获取其右边的叉子；再让剩余哲学家优先去获取其右边的叉子，再去获取其左边的叉子。

代码如下：



```java
class DiningPhilosophers {
    //1个Fork视为1个ReentrantLock，5个叉子即5个ReentrantLock，将其都放入数组中
    private final ReentrantLock[] lockList = {new ReentrantLock(),
            new ReentrantLock(),
            new ReentrantLock(),
            new ReentrantLock(),
            new ReentrantLock()};
public DiningPhilosophers() {

}

// call the run() method of any runnable to execute its code
public void wantsToEat(int philosopher,
                       Runnable pickLeftFork,
                       Runnable pickRightFork,
                       Runnable eat,
                       Runnable putLeftFork,
                       Runnable putRightFork) throws InterruptedException {

    int leftFork = (philosopher + 1) % 5;    //左边的叉子 的编号
    int rightFork = philosopher;    //右边的叉子 的编号

    //编号为偶数的哲学家，优先拿起左边的叉子，再拿起右边的叉子
    if (philosopher % 2 == 0) {
        lockList[leftFork].lock();    //拿起左边的叉子
        lockList[rightFork].lock();    //拿起右边的叉子
    }
    //编号为奇数的哲学家，优先拿起右边的叉子，再拿起左边的叉子
    else {
        lockList[rightFork].lock();    //拿起右边的叉子
        lockList[leftFork].lock();    //拿起左边的叉子
    }

    pickLeftFork.run();    //拿起左边的叉子 的具体执行
    pickRightFork.run();    //拿起右边的叉子 的具体执行

    eat.run();    //吃意大利面 的具体执行

    putLeftFork.run();    //放下左边的叉子 的具体执行
    putRightFork.run();    //放下右边的叉子 的具体执行

    lockList[leftFork].unlock();    //放下左边的叉子
    lockList[rightFork].unlock();    //放下右边的叉子
}
```
}

改进:

位运算就可以表示5个叉子的使用状态，只需用1个volatile修饰的int变量即可 + CAS操作即可。
而volatile修饰的int变量 + CAS操作 -> AtomicInteger类



```java
class DiningPhilosophers {
    //初始化为0, 二进制表示则为00000, 说明当前所有叉子都未被使用
    private AtomicInteger fork = new AtomicInteger(0);
    //每个叉子的int值(即二进制的00001, 00010, 00100, 01000, 10000)
    private final int[] forkMask = new int[]{1, 2, 4, 8, 16};
    //限制 最多只有4个哲学家去持有叉子
    private Semaphore eatLimit = new Semaphore(4);
public DiningPhilosophers() {

}

// call the run() method of any runnable to execute its code
public void wantsToEat(int philosopher,
                       Runnable pickLeftFork,
                       Runnable pickRightFork,
                       Runnable eat,
                       Runnable putLeftFork,
                       Runnable putRightFork) throws InterruptedException {

    int leftMask = forkMask[(philosopher + 1) % 5], rightMask = forkMask[philosopher];
    eatLimit.acquire();    //限制的人数 -1

    while (!pickFork(leftMask)) Thread.sleep(1);    //拿起左边的叉子
    while (!pickFork(rightMask)) Thread.sleep(1);   //拿起右边的叉子

    pickLeftFork.run();    //拿起左边的叉子 的具体执行
    pickRightFork.run();    //拿起右边的叉子 的具体执行

    eat.run();    //吃意大利面 的具体执行

    putLeftFork.run();    //放下左边的叉子 的具体执行
    putRightFork.run();    //放下右边的叉子 的具体执行

    while (!putFork(leftMask)) Thread.sleep(1);     //放下左边的叉子
    while (!putFork(rightMask)) Thread.sleep(1);    //放下右边的叉子

    eatLimit.release(); //限制的人数 +1
}

private boolean pickFork(int mask) {
    int expect = fork.get();
    return (expect & mask) > 0 ? false : fork.compareAndSet(expect, expect ^ mask);
}

private boolean putFork(int mask) {
    int expect = fork.get();
    return fork.compareAndSet(expect, expect ^ mask);
}
```
}





