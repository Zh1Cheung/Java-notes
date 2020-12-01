# sync.Mutex与sync.RWMutex

- 竞态条件、临界区与同步工具
  - 一旦数据被多个线程共享，那么就很可能会产生争用和冲突的情况。这种情况也被称为**竞态条件（race condition）**，这往往会破坏共享数据的一致性。
  - **同步的用途有两个，一个是避免多个线程在同一时刻操作同一个数据块，另一个是协调多个线程，以避免它们在同一时刻执行同一个代码块。**
  - 只要一个代码片段需要实现对共享资源的串行化访问，就可以被视为一个临界区（critical section）
  - **施加保护的重要手段之一，就是使用实现了某种同步机制的工具，也称为同步工具。**
  - **在 Go 语言中，可供我们选择的同步工具并不少。其中，最重要且最常用的同步工具当属互斥量（mutual exclusion，简称 mutex）。**`sync`包中的`Mutex`就是与其对应的类型，该类型的值可以被称为互斥量或者互斥锁。
- 我们使用互斥锁时有哪些注意事项
  - 不要重复锁定互斥锁；
    - **一旦产生死锁，程序必然崩溃**
  - 不要忘记解锁互斥锁，必要时使用`defer`语句；
    - 如果一个流程在锁定了某个互斥锁之后分叉了，或者有被中断的可能，那么就应该使用`defer`语句来对它进行解锁，而且这样的`defer`语句应该紧跟在锁定操作之后。这是最保险的一种做法。
  - 不要对尚未锁定或者已解锁的互斥锁解锁；
  - 不要在多个函数之间直接传递互斥锁。
    - Go 语言中的互斥锁是开箱即用的。换句话说，一旦我们声明了一个`sync.Mutex`类型的变量，就可以直接使用它了。
    - 不过要注意，该类型是一个**结构体类**型，属于值类型中的一种。把它传给一个函数、将它从函数中返回、把它赋给其他变量、让它进入某个通道都会导致它的**副本**的产生。
    - 并且，**原值和它的副本，以及多个副本之间都是完全独立的，它们都是不同的互斥锁。**
    - 如果你把一个互斥锁作为参数值传给了一个函数，那么在这个函数中对传入的锁的所有操作，都不会对存在于该函数之外的那个原锁产生任何的影响。



# 条件变量sync.Cond

- 条件变量与互斥锁

  - 条件变量是基于互斥锁的，**它必须有互斥锁的支撑才能发挥作用**。
  - 条件变量并不是被用来保护临界区和共享资源的，它是用于协调想要访问共享资源的那些线程的。**当共享资源的状态发生变化时，它可以被用来通知被互斥锁阻塞的线程。**
  - **条件变量的初始化离不开互斥锁，并且它的方法有的也是基于互斥锁的。**
    - 条件变量提供的方法有三个：等待通知（wait）、单发通知（signal）和广播通知（broadcast）。
    - 我们在利用条件变量等待通知的时候，需要在它基于的那个互斥锁保护下进行。而在进行单发通知或广播通知的时候，却是恰恰相反的，也就是说，需要在对应的互斥锁解锁之后再做这两种操作。

- 先来创建如下几个变量

  - ```go
    var mailbox uint8
    var lock sync.RWMutex
    // 利用sync.NewCond函数创建它的指针值。这个函数需要一个sync.Locker类型的参数值。
    sendCond := sync.NewCond(&lock)
    recvCond := sync.NewCond(lock.RLocker())
    
    ```

  - `sync.Locker`其实是一个接口，在它的声明中只包含了两个方法定义，即：`Lock()`和`Unlock()`。`sync.Mutex`类型和`sync.RWMutex`类型都拥有`Lock`方法和`Unlock`方法，只不过它们都是**指针方法**。**因此，这两个类型的指针类型才是`sync.Locker`接口的实现类型。**

  - 为了初始化`recvCond`这个条件变量，我们需要的是`lock`变量中的读锁，并且还需要是`sync.Locker`类型的。

    - 可是，`lock`变量中用于对读锁进行锁定和解锁的方法却是`RLock`和`RUnlock`，它们与`sync.Locker`接口中定义的方法并不匹配。
    - 好在`sync.RWMutex`类型的`RLocker`方法可以实现这一需求。我们只要在调用`sync.NewCond`函数时，**传入调用表达式`lock.RLocker()`的结果值**，就可以使该函数返回符合要求的条件变量了。
    - 为什么说通过`lock.RLocker()`得来的值就是`lock`变量中的读锁呢？**实际上，这个值所拥有的`Lock`方法和`Unlock`方法，在其内部会分别调用`lock`变量的`RLock`方法和`RUnlock`方法。也就是说，前两个方法仅仅是后两个方法的代理而已。**

- `*sync.Cond`类型的值可以被传递吗？那`sync.Cond`类型的值呢？

  - 指针可以传递，值不可以，传递值会拷贝一份，导致出现两份条件变量，彼此之间没有联系

  - ```go
    // 指针类型可以被传递，因为底层数据是同一份。
    
    // 传值的话
    type Cond struct {
      noCopy noCopy
      // L is held while observing or changing the condition
      L Locker
      notify notifyList
      checker copyChecker
    }
    /*
    
    Locker 是接口，传递时是引用类型
    notifyList 是结构体，会被浅拷贝
    checker 是数值类型，也会被拷贝
    
    如果sync.Cond在传递过程中，结构体和数值类型各层都有一份备份数据，容易造成不一致情况，所以极容易出问题。
    
    */
    ```

- 条件变量的`Wait`方法做了什么

  - 把调用它的 goroutine（也就是当前的 goroutine）加入到当前条件变量的通知队列中。
  - **解锁当前的条件变量基于的那个互斥锁**。
    - 因为条件变量的`Wait`方法在阻塞当前的 goroutine 之前，会解锁它基于的互斥锁，所以在调用该`Wait`方法之前，我们必须先锁定那个互斥锁，否则在调用这个`Wait`方法时，就会引发一个不可恢复的 panic。
  - 让当前的 goroutine 处于等待状态，等到通知到来时再决定是否唤醒它。此时，这个 goroutine 就会阻塞在调用这个`Wait`方法的那行代码上。
  - 如果通知到来并且决定唤醒这个 goroutine，那么就在唤醒它之后重新锁定当前条件变量基于的互斥锁。自此之后，当前的 goroutine 就会继续执行后面的代码了

- 为什么要用`for`语句来包裹调用其`Wait`方法的表达式

  - 这主要是为了保险起见。如果一个 goroutine 因收到通知而被唤醒，但却发现共享资源的状态，依然不符合它的要求，那么就应该再次调用条件变量的`Wait`方法，并继续等待下次通知的到来。

- 条件变量的`Signal`方法和`Broadcast`方法有哪些异同

  - 前者的通知只会唤醒一个因此而等待的 goroutine，而后者的通知却会唤醒所有为此等待的 goroutine。
  - 请注意，条件变量的通知具有即时性。也就是说，如果发送通知的时候没有 goroutine 为此等待，那么该通知就会被直接丢弃。在这之后才开始等待的 goroutine 只可能被后面的通知唤醒。

- `sync.Cond`类型中的公开字段`L`是做什么用的？我们可以在使用条件变量的过程中改变这个字段的值吗？

  - ```go
    // 这是个Locker，那么也是通过它来控制共享资源的并发访问。
    
    func (c *Cond) Wait() {
      c.checker.check()
      t := runtime_notifyListAdd(&c.notify)
      c.L.Unlock()
      runtime_notifyListWait(&c.notify, t)
      c.L.Lock()
    }
    func (c *Cond) Signal() {
      c.checker.check()
      runtime_notifyListNotifyOne(&c.notify)
    }
    func (c *Cond) Broadcast() {
      c.checker.check()
      runtime_notifyListNotifyAll(&c.notify)
    }
    
    // 很显然在wait时使用了以及外部逻辑使用了。随意更改这个值有可能引发panic
    ```

    



# 原子操作

- 原子性执行与原子操作

  - 对于一个 Go 程序来说，Go 语言运行时系统中的调度器会恰当地安排其中所有的 goroutine 的运行。不过，**在同一时刻，只可能有少数的 goroutine 真正地处于运行状态，并且这个数量只会与 M 的数量一致**，而不会随着 G 的增多而增长。
  - **所以，为了公平起见，调度器总是会频繁地换上或换下这些 goroutine**。
    - **换上**的意思是，让一个 goroutine 由非运行状态转为运行状态，并促使其中的代码在某个 CPU 核心上执行。
    - **换下**的意思正好相反，即：使一个 goroutine 中的代码中断执行，并让它由运行状态转为非运行状态。
    - **这个中断的时机有很多，任何两条语句执行的间隙，甚至在某条语句执行的过程中都是可以的。**
  - 互斥锁虽然可以保证临界区中代码的串行执行，但却不能保证这些代码执行的原子性（atomicity）。
    - **在众多的同步工具中，真正能够保证原子性执行的只有[原子操作](https://baike.baidu.com/item/原子操作/1880992?fr=aladdin)（atomic operation）。**原子操作在进行的过程中是不允许中断的。在底层，这会由 CPU 提供芯片级别的支持，所以绝对有效。即使在拥有多 CPU 核心，或者多 CPU 的计算机系统中，原子操作的保证也是不可撼动的。
  - **更具体地说，正是因为原子操作不能被中断，所以它需要足够简单，并且要求快速。**

- `sync/atomic`包中提供了几种原子操作？可操作的数据类型又有哪些？

  - `sync/atomic`包中的函数可以做的原子操作有：加法（add）、比较并交换（compare and swap，简称 CAS）、加载（load）、存储（store）和交换（swap）。
  - 这些函数针对的数据类型并不多。但是，对这些类型中的每一个，`sync/atomic`包都会有一套函数给予支持。这些数据类型有：`int32`、`int64`、`uint32`、`uint64`、`uintptr`，以及`unsafe`包中的`Pointer`。不过，**针对`unsafe.Pointer`类型，该包并未提供进行原子加法操作的函数。**
  - 此外，`sync/atomic`包还提供了一个名为`Value`的类型，它可以被用来存储任意类型的值。

- 我们都知道，传入这些原子操作函数的第一个参数值对应的都应该是那个被操作的值。比如，**`atomic.AddInt32`函数的第一个参数**，对应的一定是那个要被增大的整数。可是，**这个参数的类型为什么不是`int32`而是`*int32`呢？**

  - 因为**原子操作函数需要的是被操作值的指针，而不是这个值本身**；被传入函数的参数值都会被复制，像这种基本类型的值一旦被传入函数，就已经与函数外的那个值毫无关系了。
  - 只要原子操作函数拿到了被操作值的指针，就可以定位到存储该值的内存地址。只有这样，它们才能够通过底层的指令，准确地操作这个内存地址上的数据。

- 用于原子加法操作的函数可以做原子减法吗？比如，`atomic.AddInt32`函数可以用于减小那个被操作的整数值吗？

  - **`atomic.AddInt32`函数的第二个参数代表差量，它的类型是`int32`，是有符号的。**如果我们想做原子减法，那么把这个差量设置为负整数就可以了。
  - 对于`atomic.AddInt64`函数来说也是类似的。不过，**要想用`atomic.AddUint32`和`atomic.AddUint64`函数做原子减法，就不能这么直接了，因为它们的第二个参数的类型分别是`uint32`和`uint64`，都是无符号的**，不过，这也是可以做到的，就是稍微麻烦一些。
    - 例如，如果想对`uint32`类型的被操作值`18`做原子减法，比如说差量是`-3`，那么我们可以先把这个差量转换为有符号的`int32`类型的值，然后再把该值的类型转换为`uint32`，用表达式来描述就是`uint32(int32(-3))`。
    - 不过要注意，直接这样写会使 Go 语言的编译器报错，它会告诉你：“常量`-3`不在`uint32`类型可表示的范围内”，换句话说，这样做会让表达式的结果值溢出。
    - 不过，如果我们**先把`int32(-3)`的结果值赋给变量`delta`**，再把`delta`的值转换为`uint32`类型的值，就可以**绕过编译器的检查**并得到正确的结果了。
    - 最后，我们把这个结果作为`atomic.AddUint32`函数的第二个参数值，就可以达到对`uint32`类型的值做原子减法的目的了。
  - 一种更加直接的方式。我们可以依据下面这个表达式来给定`atomic.AddUint32`函数的第二个参数值：^uint32(-N-1))
  - 其中的`N`代表由负整数表示的差量。也就是说，我们先要把差量的绝对值减去`1`，然后再把得到的这个无类型的整数常量，转换为`uint32`类型的值，最后，在这个值之上做按位异或操作，就可以获得最终的参数值了。

- 假设我已经保证了对一个变量的写操作都是原子操作，比如：加或减、存储、交换等等，那我对它进行读操作的时候，还有必要使用原子操作吗？

  - 有必要，一旦你决定了要对一个共享资源进行保护，那就要做到完全的保护
  - 在很多应用场景下，互斥锁往往是更加适合的。
  - 不过，一旦我们确定了在某个场景下可以使用原子操作函数，比如：只涉及并发地读写单一的整数类型值，或者多个互不相关的整数类型值，那就不要再考虑互斥锁了。

- 怎样用好`sync/atomic.Value`

  - 此类型的值相当于一个容器，可以被用来“原子地”存储和加载任意的值。（它只有两个指针方法：`Store`和`Load`）
  - 我们只要用它来存储值了，就相当于开始真正使用了。**`atomic.Value`类型属于结构体类型，而结构体类型属于值类型。**
  - **所以，复制该类型的值会产生一个完全分离的新值**。这个新值相当于被复制的那个值的一个快照。之后，不论后者存储的值怎样改变，都不会影响到前者，反之亦然。
  - 有两条强制性的使用规则。
    - **第一条规则，不能用原子值存储`nil`。**
      - 如果有一个接口类型的变量，它的动态值是`nil`，但动态类型却不是`nil`，那么它的值就不等于`nil`。我在前面讲接口的时候和你说明过这个问题。正因为如此，这样一个变量的值是可以被存入原子值的。
    - **第二条规则，我们向原子值存储的第一个值，决定了它今后能且只能存储哪一个类型的值。**
      - 你可能会想：我先存储一个接口类型的值，然后再存储这个接口的某个实现类型的值，这样是不是可以呢？
      - 很可惜，这样是不可以的，同样会引发一个 panic。因为**原子值内部是依据被存储值的实际类型来做判断的。**所以，即使是实现了同一个接口的不同类型，它们的值也不能被先后存储到同一个原子值中。

- **具体的使用建议**

  - **不要把内部使用的原子值暴露给外界**。比如，声明一个全局的原子变量并不是一个正确的做法。这个变量的访问权限最起码也应该是包级私有的。
  - 如果不得不让包外，或模块外的代码使用你的原子值，那么可以**声明一个包级私有的原子变量，然后再通过一个或多个公开的函数，让外界间接地使用到它**。注意，这种情况下不要把原子值传递到外界，不论是传递原子值本身还是它的指针值。
  - **如果通过某个函数可以向内部的原子值存储值的话，那么就应该在这个函数中先判断被存储值类型的合法性。**若不合法，则应该直接返回对应的错误值，从而避免 panic 的发生。
  - 如果可能的话，**我们可以把原子值封装到一个数据类型中，比如一个结构体类型**。这样，我们既可以通过该类型的方法更加安全地存储值，又可以在该类型中包含可存储值的合法类型信息。

- 再特别强调一点：尽量不要向原子值中存储引用类型的值。因为这很容易造成安全漏洞。

  - ```go
    // 注意，切片类型属于引用类型。所以，我在外面改动这个切片值，就等于修改了box6中存储的那个值。这相当于绕过了原子值而进行了非并发安全的操作。
    var box6 atomic.Value
    v6 := []int{1, 2, 3}
    box6.Store(v6)
    v6[1] = 4 // 注意，此处的操作不是并发安全的！
    
    
    // 那么，应该怎样修补这个漏洞呢？可以这样做
    
    // 我先为切片值v6创建了一个完全的副本。这个副本涉及的数据已经与原值毫不相干了。然后，我再把这个副本存入box6。如此一来，无论我再对v6的值做怎样的修改，都不会破坏box6提供的安全保护。
    store := func(v []int) {
     replica := make([]int, len(v))
     copy(replica, v)
     box6.Store(replica)
    }
    store(v6)
    v6[2] = 5 // 此处的操作是安全的。
    
    
    ```

- 如果要对原子值和互斥锁进行二选一，你认为最重要的三个决策条件应该是什么？
  - 是否一定要操作引用类型的值；
  - 是否一定要操作nil；
  - 是否需要处理一个接口的不同类型。
- 什么时候使用atomic.value
  
  - 其实需要保护非引用类型的值的时候都挺适用的。比如保护全局配置、同时保护一坨全局计数、保护 bit array，等等





# sync.WaitGroup和sync.Once

- 声明一个通道，使它的容量与我们手动启用的 goroutine 的数量相同，之后再利用这个通道，让主 goroutine 等待其他 goroutine 的运行结束。

  - `sync`包的`WaitGroup`类型。它比通道更加适合实现这种一对多的 goroutine 协作流程。与我们前面讨论的几个同步工具一样，它一旦被真正使用就不能被复制了。

  - `WaitGroup`类型拥有**三个指针方法：`Add`、`Done`和`Wait`。**你可以想象该类型中有一个计数器，它的默认值是`0`。我们可以通过调用该类型值的`Add`方法来增加，或者减少这个计数器的值。

  - 一般情况下，我会用这个方法来记录需要等待的 goroutine 的数量。相对应的，这个**类型的`Done`方法，用于对其所属值中计数器的值进行减一操作**。我们可以在需要等待的 goroutine 中，通过`defer`语句调用它。          

  - ```go
    func coordinateWithWaitGroup() {
     var wg sync.WaitGroup
     wg.Add(2)
     num := int32(0)
     fmt.Printf("The number: %d [with sync.WaitGroup]\n", num)
     max := int32(10)
     go addNum(&num, 3, max, wg.Done)
     go addNum(&num, 4, max, wg.Done)
     wg.Wait()
    }
    
    func addNum(numP *int32, id, max int32, deferFunc func()) {
    	defer func() {
    		deferFunc()
    	}()
    	for i := 0; ; i++ {
    		currNum := atomic.LoadInt32(numP)
    		if currNum >= max {
    			break
    		}
    		newNum := currNum + 2
    		time.Sleep(time.Millisecond * 200)
    		if atomic.CompareAndSwapInt32(numP, currNum, newNum) {
    			fmt.Printf("The number: %d [%d-%d]\n", newNum, id, i)
    		} else {
    			fmt.Printf("The CAS operation failed. [%d-%d]\n", id, i)
    		}
    	}
    ```

- **`sync.WaitGroup`类型值中计数器的值可以小于`0`吗**

  - **之所以说`WaitGroup`值中计数器的值不能小于`0`，是因为这样会引发一个 panic。** 不适当地调用这类值的`Done`方法和`Add`方法都会如此。别忘了，我们在调用`Add`方法的时候是可以传入一个负数的。
  - 实际上，导致`WaitGroup`值的方法抛出 panic 的原因不只这一种。
    - 如果我们对它的`Add`方法的首次调用，与对它的`Wait`方法的调用是同时发起的，比如，在同时启用的两个 goroutine 中，分别调用这两个方法，**那么就有可能会让这里的`Add`方法抛出一个 panic。**
    - 所以，虽然`WaitGroup`值本身并不需要初始化，但是尽早地增加其计数器的值，还是非常有必要的。
  - **只要计数器的值始于`0`又归为`0`，就可以被视为一个计数周期。**在一个此类值的生命周期中，它可以经历任意多个计数周期。但是，**只有在它走完当前的计数周期之后，才能够开始下一个计数周期。**
    - 因此，也可以说，**如果一个此类值的`Wait`方法在它的某个计数周期中被调用，那么就会立即阻塞当前的 goroutine，直至这个计数周期完成**。在这种情况下，该值的下一个计数周期，必须要等到这个`Wait`方法执行结束之后，才能够开始。
    - 如果在一个此类值的`Wait`方法被执行期间，跨越了两个计数周期，**那么就会引发一个 panic。**
    - 例如，在当前的 goroutine 因调用此类值的`Wait`方法，而被阻塞的时候，另一个 goroutine 调用了该值的`Done`方法，并使其计数器的值变为了`0`。
    - 这会唤醒当前的 goroutine，并使它试图继续执行`Wait`方法中其余的代码。但在这时，又有一个 goroutine 调用了它的`Add`方法，并让其计数器的值又从`0`变为了某个正整数。**此时，这里的`Wait`方法就会立即抛出一个 panic。**
    - 如果在一个此类值的`Wait`方法被执行期间，跨越了两个计数周期，**那么就会引发一个 panic。**
  - 纵观上述会引发 panic 的后两种情况，我们可以总结出这样一条关于`WaitGroup`值的使用禁忌，即：**不要把增加其计数器值的操作和调用其`Wait`方法的代码，放在不同的 goroutine 中执行。换句话说，要杜绝对同一个`WaitGroup`值的两种操作的并发执行。**

- `sync.Once`类型值的`Do`方法是怎么保证只执行参数函数一次的

  - 与`sync.WaitGroup`类型一样，`sync.Once`类型（以下简称`Once`类型）**也属于结构体类型，同样也是开箱即用和并发安全的。**
  - **`Once`类型的`Do`方法只接受一个参数，这个参数的类型必须是`func()`**，即：无参数声明和结果声明的函数。
  - **该方法的功能并不是对每一种参数函数都只执行一次，而是只执行“首次被调用时传入的”那个函数**，并且之后不会再执行任何参数函数。
  - 所以，如果你有多个只需要执行一次的函数，那么就应该为它们中的每一个都分配一个`sync.Once`类型的值（以下简称`Once`值）。
  
- `Once`类型中还有一个名叫`done`的`uint32`类型的字段。它的作用是记录其所属值的`Do`方法被调用的次数。不过，该字段的值只可能是`0`或者`1`。一旦`Do`方法的首次调用完成，它的值就会从`0`变为`1`。

  - 既然`done`字段的值不是`0`就是`1`，那为什么还要使用需要四个字节的`uint32`类型呢？
    - 原因很简单，因为**对它的操作必须是“原子”的**。`Do`方法在一开始就会**通过调用`atomic.LoadUint32`函数来获取该字段的值**，并且一旦发现该值为`1`，就会直接返回。这也初步保证了“`Do`方法，只会执行首次被调用时传入的函数”。
    - 不过，单凭这样一个判断的保证是不够的。因为，**如果有两个 goroutine 都调用了同一个新的`Once`值的`Do`方法，并且几乎同时执行到了其中的这个条件判断代码，那么它们就都会因判断结果为`false`，而继续执行`Do`方法中剩余的代码。**
    - **在这个条件判断之后**，`Do`方法会立即锁定其所属值中的那个`sync.Mutex`类型的字段`m`。**然后，它会在临界区中再次检查`done`字段的值，并且仅在条件满足时，才会去调用参数函数，以及用原子操作把`done`的值变为`1`。**

- `Do`方法在功能方面的两个特点

  - **第一个特点**，由于`Do`方法只会在参数函数执行结束之后把`done`字段的值变为`1`，因此，**如果参数函数的执行需要很长时间或者根本就不会结束（比如执行一些守护任务），那么就有可能会导致相关 goroutine 的同时阻塞。**
  - **第二个特点**，`Do`方法在参数函数执行结束后，对`done`字段的赋值用的是原子操作，并且，**这一操作是被挂在`defer`语句中的**。因此，不论参数函数的执行会以怎样的方式结束，`done`字段的值都会变为`1`。
    - 也就是说，即使这个参数函数没有执行成功（比如引发了一个 panic），我们也无法使用同一个`Once`值重新执行它了。所以，如果你需要为参数函数的执行设定重试机制，那么就要考虑`Once`值的适时替换问题。

- 在使用`WaitGroup`值实现一对多的 goroutine 协作流程时，怎样才能让分发子任务的 goroutine 获得各个子任务的具体执行结果？

  - ```go
    func getAllGoroutineResult(){
      wg := sync.WaitGroup{}
      wg.Add(3)
    
      once := sync.Once{}
      var aAndb int
      var aStrAndb string
      var gflag int32
    
      addNum := func(a,b int, ret *int) {
        defer wg.Done()
        time.Sleep(time.Millisecond * 2000)
        *ret = a+b
        atomic.AddInt32(&gflag,1)
      }
    
      addStr := func(a,b string, ret *string) {
        defer wg.Done()
        time.Sleep(time.Millisecond * 1000)
        *ret = a+b
        atomic.AddInt32(&gflag,1)
      }
    
      // waitRet需要等待 addNum和addStr执行完成后的结果
      waitRet := func(ret *int, strRet *string) {
        defer wg.Done()
        once.Do(func() {
        for atomic.LoadInt32(&gflag) != 2 {
          fmt.Println("Wait: addNum & addStr")
          time.Sleep(time.Millisecond * 200)
        }
      })
        fmt.Println(fmt.Sprintf("AddNum's Ret is: %d\n", *ret))
        fmt.Println(fmt.Sprintf("AddStr's Ret is: %s\n", *strRet))
      }
    
      // waitRet goroutine等待AddNum和AddStr结束
      go waitRet(&aAndb, &aStrAndb)
      go addNum(10, 20, &aAndb)
      go addStr("aaa", "bbb", &aStrAndb)
    
      wg.Wait()
    }
    
    ```

    



# Context

- **在使用**WaitGroup**值的时候，我们最好用“先统一**Add**，再并发**Done**，最后**Wait**”的标 准模式来构建协作流程。**

  - 如果在调用该值的Wait方法的同时，为了增大其计数器的值，而并发地调用该值的Add方 法，那么就很可能会引发 panic。
  - 这就带来了一个问题，如果我们不能在一开始就确定执行子任务的 goroutine 的数量，那么使用WaitGroup值来协调它们和分发子任务的 goroutine，就是有一定风险的。一个解决方案是:**分批地启用执行子任务的 goroutine**。

- **怎样使用**context**包中的程序实体，实现一对多的 goroutine 协作流程?**

  - ```go
    // 在这个函数体中，我先后调用了context.Background函数和context.WithCancel函 数，并得到了一个可撤销的context.Context类型的值(由变量cxt代表)，以及一个 context.CancelFunc类型的撤销函数(由变量cancelFunc代表)。
    
    func coordinateWithContext() {
    	total := 12
    	var num int32
    	fmt.Printf("The number: %d [with context.Context]\n", num) 
      cxt,cancelFunc := context.WithCancel(context.Background()) 
      for i := 1; i <= total; i++ {
        // 请注意我给予addNum函数的最后一个参数值。它是一个匿名函数，其中只包含了一条if语 句。这条if语句会“原子地”加载num变量的值，并判断它是否等于total变量的值。
    		go addNum(&num, i, func() {
        // 如果两个值相等，那么就调用cancelFunc函数。其含义是，如果所有的addNum函数都执行完毕，那么就立即通知分发子任务的 goroutine。
    		if atomic.LoadInt32(&num) == int32(total) {
       	 cancelFunc()
       		}
    		}) 
      }
      // 这里分发子任务的 goroutine，即为执行coordinateWithContext函数的 goroutine。 它在执行完for语句后，会立即调用cxt变量的Done函数，并试图针对该函数返回的通 道，进行接收操作。
      // 由于一旦cancelFunc函数被调用，针对该通道的接收操作就会马上结束，所以，这样做 就可以实现“等待所有的addNum函数都执行完毕”的功能。
    
    	<-cxt.Done()
    	fmt.Println("End.") 
    }
    ```

- Context类型之所以受到了标准库中众多代码包的积极支持，主要是因为**它是一种非常通用的同步工具。它的值不但可以被任意地扩散，而且还可以被用来传递额外的信息和信号。**

  - 更具体地说，Context类型**可以提供一类代表上下文的值**。此类值是**并发安全**的，也就是说它可以被传播给多个 goroutine。
  - 由于Context类型实际上是一个接口类型，而context包中实现该接口的**所有私有类型**， 都是基于某个数据类型的指针类型，所以，如此传播并不会影响该类型值的功能和安全。

- Context类型的值(以下简称Context值)是可以**繁衍**的，这意味着我们可以通过一个 Context值产生出任意个子值。这些子值可以携带其父值的属性和数据，也可以响应我们 通过其父值传达的信号。

  - 正因为如此，**所有的Context值共同构成了一颗代表了上下文全貌的树形结构**。这棵树的**树根(或者称上下文根节点)是一个已经在context包中预定义好的Context值，它是全局唯一的。通过调用context.Background函数，我们就可以获取到它。**
    - 这个上下文根节点仅仅是一个最基本的支点，它不提供任何额外的功能。也 就是说，它既不可以被撤销(cancel)，也不能携带任何数据。
  - 除此之外，context包中还包含了四个用于繁衍Context值的函数，即:WithCancel、 WithDeadline、WithTimeout和WithValue。
    - **这些函数的第一个参数的类型都是context.Context，而名称都为parent**。顾名思义， **这个位置上的参数对应的都是它们将会产生的Context值的父值。**
    - WithCancel函数用于产生一个可撤销的parent的子值。在coordinateWithContext 函数中，我通过调用该函数，获得了一个衍生自上下文根节点的Context值，和一个用于 触发撤销信号的函数。
    - 而WithDeadline函数和WithTimeout函数则都可以被用来产生一个会定时撤销的 parent的子值。至于WithValue函数，我们可以通过调用它，产生一个会携带额外数据 的parent的子值。

- **“可撤销的“在context包中代表着什么?撤销”一个Context值又意味着什么?**

  - 这需要从Context类型的声明讲起。这个接口中有两个方法与“撤销”息息相关。**Done方 法会返回一个元素类型为struct{}的接收通道**。不过，这个接收通道的用途并不是传递元素值，而是**让调用方去感知“撤销”当前Context值的那个信号。**
    - 一旦当前的Context值被撤销，这里的接收通道就会被立即关闭。我们都知道，对于一个 未包含任何元素值的通道来说，它的关闭会使任何针对它的接收操作立即结束。
    - **除了让Context值的使用方感知到撤销信号，让它们得到“撤销”的具体原因，有时也是 很有必要的。后者即是Context类型的Err方法的作用。**该方法的结果是error类型的， 并且其值只可能等于context.Canceled变量的值，或者 context.DeadlineExceeded变量的值。前者用于表示手动撤销，而后者则代表:由于我们给定的过期时间已到，而导致的撤销。
  - 你可能已经感觉到了，对于Context值来说，“撤销”这个词如果当名词讲，指的其实就 是被用来表达“撤销”状态的信号;如果当动词讲，指的就是对撤销信号的传达;而**“可撤 销的”指的则是具有传达这种撤销信号的能力。**
  - **当我们通过调用context.WithCancel函数产生一个可撤销的Context 值时，还会获得一个用于触发撤销信号的函数。**
    - **通过调用这个函数，我们就可以触发针对这个Context值的撤销信号。**一旦触发，撤销信号就会立即被传达给这个Context值，并由它的Done方法的结果值(一个接收通道)表达出来。
    - **撤销函数只负责触发信号，而对应的可撤销的Context值也只负责传达信号，它们都不会去管后边具体的“撤销”操作。**实际上，我们的代码可以在感知到撤销信号之后，进行任意的操作，Context值对此并没有任何的约束。

- **撤销信号是如何在上下文树中传播的?**

  - context包中包含了四个用于繁衍Context值的函数。其中的 WithCancel、WithDeadline和WithTimeout都是被用来**基于给定的Context值产生 可撤销的子值的。**
  - context包的WithCancel函数在被调用后会产生两个结果值。第一个结果值就是那个可 撤销的Context值，而第二个结果值则是用于触发撤销信号的函数。
    - 在撤销函数被调用之后，对应的Context值会先关闭它内部的接收通道，也就是它的Done 方法会返回的那个通道。
    - 然后，它会向它的所有子值(或者说子节点)传达撤销信号。这些子值会如法炮制，把撤销 信号继续传播下去。最后，这个Context值会断开它与其父值之间的关联。
  - 我们通过调用context包的WithDeadline函数或者WithTimeout函数生成的Context 值也是可撤销的。它们不但可以被手动撤销，还会依据在生成时被给定的过期时间，自动地 进行定时撤销。这里定时撤销的功能是借助它们内部的计时器来实现的。
  - 最后要注意，通过调用context.WithValue函数得到的Context值是不可撤销的。撤销信号在被传播时，若遇到它们则会直接跨过，并试图将信号直接传给它们的子值。

- **怎样通过**Context**值携带数据?怎样从中获取数据?**

  - WithValue函数在产生新的Context值(以下简称含数据的Context值)的时候需要三个 参数，即:父值、键和值。与“字典对于键的约束”类似，这里键的类型必须是可判等的。
  - **Context类型的Value方法就是被用来获取数据的。**在我们调用含数据的Context值的 Value方法时，它会先判断给定的键，是否与当前值中存储的键相等，如果相等就把该值中 存储的值直接返回，否则就到其父值中继续查找。
  - 注意，**除了含数据的Context值以外，其他几种Context值都是无法携带数据的。**因此， Context值的Value方法在沿路查找的时候，会直接跨过那几种值。
  - **如果我们调用的Value方法的所属值本身就是不含数据的，那么实际调用的就将会是其父辈 或祖辈的Value方法。**这是由于这几种Context值的实际类型，都属于结构体类型，并且 它们都是通过“将其父值嵌入到自身”，来表达父子关系的。
  - **最后，提醒一下，Context接口并没有提供改变数据的方法。**因此，在通常情况下，我们 只能通过在上下文树中添加含数据的Context值来存储新的数据，或者通过撤销此种值的 父值丢弃掉相应的数据。如果你存储在这里的数据可以从外部改变，那么必须自行保证安 全。





# 临时对象池sync.Pool

- `sync.Pool`类型可以被称为临时对象池，它的值可以被用来**存储临时的对象**。与 Go 语言的很多同步工具一样，`sync.Pool`类型也属于结构体类型，它的值在被真正使用之后，就不应该再被复制了。

  - **这里的“临时对象”的意思是：不需要持久使用的某一类值**。这类值对于程序来说可有可无，但如果有的话会明显更好。它们的创建和销毁可以在任何时候发生，并且完全不会影响到程序的功能。

  - **我们可以把临时对象池当作针对某种数据的缓存来用**。实际上，在我看来，临时对象池最主要的用途就在于此。

  - `sync.Pool`类型只有两个方法——`Put`和`Get`。Put 用于在当前的池中存放临时对象，它接受一个`interface{}`类型的参数；而 Get 则被用于从当前的池中获取临时对象，它会返回一个`interface{}`类型的值。

    - 更具体地说，这个类型的`Get`方法可能会从当前的池中删除掉任何一个值，然后把这个值作为结果返回。如果此时当前的池中没有任何值，那么这个方法就会使用当前池的`New`字段创建一个新值，并直接将其返回。

  - **`sync.Pool`类型的`New`字段代表着创建临时对象的函数。**它的类型是没有参数但有唯一结果的函数类型，即：`func() interface{}`。

  - 这里的`New`字段的实际值需要我们在初始化临时对象池的时候就给定。否则，在我们调用它的`Get`方法的时候就有可能会得到`nil`。所以，`sync.Pool`类型并不是开箱即用的

    - 举个例子。标准库代码包`fmt`就使用到了`sync.Pool`类型。这个包会创建一个用于缓存某类临时对象的`sync.Pool`类型值，并将这个值赋给一个名为`ppFree`的变量。这类临时对象可以识别、格式化和暂存需要打印的内容。

    - ```go
      var ppFree = sync.Pool{
       New: func() interface{} { return new(pp) },
      }
      ```

    - 临时对象池`ppFree`的`New`字段在被调用的时候，总是会返回一个全新的`pp`类型值的指针（即临时对象）。**这就保证了`ppFree`的`Get`方法总能返回一个可以包含需要打印内容的值。**

    - `pp`类型是`fmt`包中的私有类型，它有很多实现了不同功能的方法。不过，这里的重点是，它的每一个值都是独立的、平等的和可重用的。

    - 另外，这些代码在使用完临时对象之后，都会先抹掉其中已缓冲的内容，然后再把它存放到`ppFree`中。这样就为重用这类临时对象做好了准备。

    - 众所周知的`fmt.Println`、`fmt.Printf`等打印函数都是如此使用`ppFree`，以及其中的临时对象的。**因此，在程序同时执行很多的打印函数调用的时候，`ppFree`可以及时地把它缓存的临时对象提供给它们，以加快执行的速度。**

- 为什么说临时对象池中的值会被及时地清理掉？

  - 因为，Go 语言运行时系统中的**垃圾回收器**，所以在每次开始执行之前，都会对所有已创建的临时对象池中的值进行全面地清除。
  - `sync`包在被初始化的时候，会向 Go 语言运行时系统注册一个函数，这个函数的功能就是清除所有已创建的临时对象池中的值。我们可以把它称为**池清理函数**。
  - 另外，**在`sync`包中还有一个包级私有的全局变量**。这个变量代表了当前的程序中使用的所有临时对象池的汇总，**它是元素类型为`*sync.Pool`的切片。我们可以称之为池汇总列表**
  - **更具体地说，池清理函数会遍历池汇总列表。**对于其中的每一个临时对象池，它都会先将池中所有的私有临时对象和共享临时对象列表都**置为`nil`**，然后再把这个池中的所有本地池列表都销毁掉。

- 临时对象池存储值所用的数据结构是怎样的

  - 在临时对象池中，有一个多层的数据结构。正因为有了它的存在，临时对象池才能够非常高效地存储大量的值。
  - 这个数据结构的顶层，我们可以称之为**本地池列表**，不过更确切地说，它是一个数组。**这个列表的长度，总是与 Go 语言调度器中的 P 的数量相同。**
    - **P 存在的一个很重要的原因是为了分散并发程序的执行压力**，而让临时对象池中的本地池列表的长度与 P 的数量相同的主要原因也是分散压力。这里所说的压力包括了**存储和性能**两个方面。
  - 在本地池列表中的每个本地池都包含了**三个字段**（或者说组件），它们是：存储私有临时对象的字段`private`、代表了共享临时对象列表的字段`shared`，以及一个`sync.Mutex`类型的嵌入字段。
  - 一个临时对象池的`Put`方法或`Get`方法会获取到哪一个本地池，完全取决于调用它的代码所在的 goroutine 关联的那个 P。

- 临时对象池是怎样利用内部数据结构来存取值的

  - 临时对象池的`Put`方法总会先试图把新的临时对象，存储到对应的本地池的`private`字段中，以便在后面获取临时对象的时候，可以快速地拿到一个可用的值。
    - 只有当这个`private`字段已经存有某个值时，该方法才会去访问本地池的`shared`字段。
  - 相应的，临时对象池的`Get`方法，总会先试图从对应的本地池的`private`字段处获取一个临时对象。只有当这个`private`字段的值为`nil`时，它才会去访问本地池的`shared`字段。（`private`字段是 P 级私有的）
    - 一个本地池的`shared`字段原则上可以被任何 goroutine 中的代码访问到，不论这个 goroutine 关联的是哪一个 P。这也是我把它叫做共享临时对象列表的原因。
  - **以临时对象池的`Put`方法为例，它一旦发现对应的本地池的`private`字段已存有值，就会去访问这个本地池的`shared`字段。**当然，由于`shared`字段是共享的，所以此时必须受到互斥锁的保护。
    - 相应的，临时对象池的`Get`方法在发现对应本地池的`private`字段未存有值时，也会去访问后者的`shared`字段。
  - **当然了，即使这样也可能无法拿到一个可用的临时对象**，比如，在所有的临时对象池都刚被大清洗的情况下就会是如此。
    - **这时，`Get`方法就会使出最后的手段——调用可创建临时对象的那个函数**

- 怎样保证一个临时对象池中总有比较充足的临时对象

  - 临时对象池初始化时指定new字段对应的函数返回一个新建临时对象；
  - 临时对象使用完毕时调用临时对象池的put方法，把该临时对象put回临时对象池中。





# 并发安全字典sync.Map

- Go 的开发者极力推荐使用 Channel，不过，这两年，大家意识到，Channel 并不是处理并发问题的“银弹”，有时候使用并发原语更简单，而且不容易出错。所以，我给你提供一套选择的方法:

  - 共享资源的并发访问使用传统并发原语；
  - 复杂的任务编排和消息传递使用 Channel；
  - 消息通知机制使用 Channel，除非只想 signal 一个 goroutine，才使用 Cond；
  - 简单等待所有任务的完成用 WaitGroup，也有 Channel 的推崇者用 Channel，都可以；
  - 需要和 Select 语句结合，使用 Channel；
  - 需要和超时配合时，使用 Channel 和 Context。

- Go 语言自带的字典类型`map`并不是并发安全的。

  - 我们常说，能用原子操作就不要用锁，不过这很有局限性，毕竟**原子只能对一些基本的数据类型提供支持。**
  - 无论在何种场景下使用`sync.Map`，我们都需要注意，与原生`map`明显不同，它只是 Go 语言标准库中的一员，而不是语言层面的东西。也正因为这一点，**Go 语言的编译器并不会对它的键和值，进行特殊的类型检查。**
  - 如果你看过`sync.Map`的文档或者实际使用过它，那么就一定会知道，它所有的方法涉及的键和值的类型都是`interface{}`，也就是空接口，这意味着可以包罗万象。所以，**我们必须在程序中自行保证它的键类型和值类型的正确性。**

- 并发安全字典对键的类型有要求吗

  - 有要求。键的实际类型不能是函数类型、字典类型和切片类型。
  - Go 语言的原生字典的键类型不能是函数类型、字典类型和切片类型。由于**并发安全字典内部使用的存储介质正是原生字典，又因为它使用的原生字典键类型也是可以包罗万象的`interface{}`**
  - **由于这些键值的实际类型只有在程序运行期间才能够确定**，所以 Go 语言编译器是无法在编译期对它们进行检查的，不正确的键值实际类型肯定会引发 panic。
  - 总之，我们必须保证键的类型是可比较的（或者说可判等的）。如果你实在拿不准，那么可以先通过调用**`reflect.TypeOf`函数**得到一个键值对应的反射类型值（即：`reflect.Type`类型的值），然后再调用这个值的**`Comparable`方法**，得到确切的判断结果。

- 怎样保证并发安全字典中的键和值的类型正确性

  - 简单地说，可以使用类型断言表达式或者反射操作来保证它们的类型正确性。

- 为了进一步明确并发安全字典中键值的实际类型，这里大致有两种方案可选。

  - **第一种方案是，让并发安全字典只能存储某个特定类型的键**

    - 比如，指定这里的键只能是`int`类型的，或者只能是字符串，又或是某类结构体。一旦完全确定了键的类型，你就可以在进行存、取、删操作的时候，使用类型断言表达式去对键的类型做检查了。

    - 一般情况下，这种检查并不繁琐。而且，**你要是把并发安全字典封装在一个结构体类型里面，那就更加方便了。你这时完全可以让 Go 语言编译器帮助你做类型检查。**

    - ```go
      type IntStrMap struct {
       m sync.Map
      }
       
      func (iMap *IntStrMap) Delete(key int) {
       iMap.m.Delete(key)
      }
       
      func (iMap *IntStrMap) Load(key int) (value string, ok bool) {
       v, ok := iMap.m.Load(key)
       if v != nil {
        value = v.(string)
       }
       return
      }
       
      func (iMap *IntStrMap) LoadOrStore(key int, value string) (actual string, loaded bool) {
       a, loaded := iMap.m.LoadOrStore(key, value)
       actual = a.(string)
       return
      }
       
      func (iMap *IntStrMap) Range(f func(key int, value string) bool) {
       f1 := func(key, value interface{}) bool {
        return f(key.(int), value.(string))
       }
       iMap.m.Range(f1)
      }
       
      func (iMap *IntStrMap) Store(key int, value string) {
       iMap.m.Store(key, value)
      }
      ```

    - 显然，**这些方法在接受键和值的时候，就不用再做类型检查了**。另外，这些方法在从`m`中取出键和值的时候，完全不用担心它们的类型会不正确，**因为它的正确性在当初存入的时候，就已经由 Go 语言编译器保证了。**

  - **在第二种方案中，我们封装的结构体类型的所有方法，都可以与`sync.Map`类型的方法完全一致（包括方法名称和方法签名）。**

    - 不过，在这些方法中，我们就需要添加一些做**类型检查**的代码了。另外，这样并发安全字典的**键类型和值类型，必须在初始化的时候就完全确定**。并且，这种情况下，我们**必须先要保证键的类型是可比较的**。

    - ```go
      type ConcurrentMap struct {
       m         sync.Map
        // reflect.Type，我们可称之为反射类型
        // 这个类型可以代表 Go 语言的任何数据类型。并且，这个类型的值也非常容易获得：通过调用reflect.TypeOf函数并把某个样本值传入即可。
        // 调用表达式reflect.TypeOf(int(123))的结果值，就代表了int类型的反射类型值。
       keyType   reflect.Type
       valueType reflect.Type
      }
      
      func (cMap *ConcurrentMap) Load(key interface{}) (value interface{}, ok bool) {
       if reflect.TypeOf(key) != cMap.keyType {
        return
       }
       return cMap.m.Load(key)
      }
      
      func (cMap *ConcurrentMap) Store(key, value interface{}) {
       if reflect.TypeOf(key) != cMap.keyType {
        panic(fmt.Errorf("wrong key type: %v", reflect.TypeOf(key)))
       }
       if reflect.TypeOf(value) != cMap.valueType {
        panic(fmt.Errorf("wrong value type: %v", reflect.TypeOf(value)))
       }
       cMap.m.Store(key, value)
      }
      
      ```

- 并发安全字典如何做到尽量避免使用锁

  - `sync.Map`类型在内部使用了大量的原子操作来存取键和值，并使用了两个原生的`map`作为存储介质。
    - **其中一个原生`map`被存在了`sync.Map`的`read`字段中，该字段是`sync/atomic.Value`类型的。** 这个原生字典可以被看作一个快照，它总会在条件满足时，去重新保存所属的`sync.Map`值中包含的所有键值对。
      - **为了描述方便，我们在后面简称它为只读字典**。不过，**只读字典虽然不会增减其中的键，但却允许变更其中的键所对应的值**。所以，它并不是传统意义上的快照，它的只读特性只是对于其中键的集合而言的。
      - 由`read`字段的类型可知，`sync.Map`在替换只读字典的时候根本用不着锁。另外，**这个只读字典在存储键值对的时候，还在值之上封装了一层。**
      - **它先把值转换为了`unsafe.Pointer`类型的值，然后再把后者封装，并储存在其中的原生字典中。**如此一来，在变更某个键所对应的值的时候，就也可以使用原子操作了。
    - **`sync.Map`中的另一个原生字典由它的`dirty`字段代表。** 它存储键值对的方式与`read`字段中的原生字典一致，它的键类型也是`interface{}`，并且同样是把值先做转换和封装后再进行储存的。我们暂且把它称为脏字典。
    - **注意，脏字典和只读字典如果都存有同一个键值对，那么这里的两个键指的肯定是同一个基本值，对于两个值来说也是如此。**
      - 这两个字典在存储键和值的时候都只会存入它们的某个指针，而不是基本值
      - `sync.Map`在**查找**指定的键所对应的值的时候，总会先去只读字典中寻找，并不需要锁定互斥锁。**只有当确定“只读字典中没有，但脏字典中可能会有这个键”的时候，它才会在锁的保护下去访问脏字典。**
      - 相对应的，`sync.Map`在**存储**键值对的时候，只要只读字典中已存有这个键，并且该键值对未被标记为“已删除”，就会把新值存到里面并直接返回，这种情况下也不需要用到锁。
      - 否则，它才会在锁的保护下把键值对存储到脏字典中。这个时候，该键值对的“已删除”标记会被抹去。
  - **sync.Map 中的 read 与 dirty**
    - **只有当一个键值对应该被删除，但却仍然存在于只读字典中的时候，才会被用标记为“已删除”的方式进行逻辑删除，而不会直接被物理删除。**
      - **这种情况会在重建脏字典以后的一段时间内出现。不过，过不了多久，它们就会被真正删除掉。**在查找和遍历键值对的时候，已被逻辑删除的键值对永远会被无视。
    - **对于删除键值对，`sync.Map`会先去检查只读字典中是否有对应的键**。如果没有，脏字典中可能有，那么它就会在锁的保护下，试图从脏字典中删掉该键值对。
      - 最后，`sync.Map`会把该键值对中指向值的那个指针置为`nil`，这是另一种逻辑删除的方式。
    - 除此之外，还有一个细节需要注意，**只读字典和脏字典之间是会互相转换的。在脏字典中查找键值对次数足够多的时候，`sync.Map`会把脏字典直接作为只读字典**，保存在它的`read`字段中，然后把代表脏字典的`dirty`字段的值置为`nil`
      - 在这之后，一旦再有新的键值对存入，它就会依据只读字典去重建脏字典。这个时候，它会把只读字典中已被逻辑删除的键值对过滤掉。理所当然，这些转换操作肯定都需要在锁的保护下进行。
  - **sync.Map 中 read 与 dirty 的互换**
    - 综上所述，`sync.Map`的只读字典和脏字典中的键值对集合，并不是实时同步的，它们在某些时间段内可能会有不同。
    - 由于只读字典中键的集合不能被改变，所以其中的键值对有时候可能是不全的。相反，脏字典中的键值对集合总是完全的，并且其中不会包含已被逻辑删除的键值对。
    - **因此，可以看出，在读操作有很多但写操作却很少的情况下，并发安全字典的性能往往会更好。**在几个写操作当中，新增键值对的操作对并发安全字典的性能影响是最大的，其次是删除操作，最后才是修改操作。
    - **如果被操作的键值对已经存在于`sync.Map`的只读字典中，并且没有被逻辑删除，那么修改它并不会使用到锁，对其性能的影响就会很小。**





# **Channel**

- Channel 的地位在编程语言中的地位之 高，比较罕见。

- **Channel** **的发展**

  - **CSP** 是 Communicating Sequential Process 的简称，**中文直译为通信顺序进程，或者叫做交换信息的循序进程，是用来描述并发系统中进行交互的一种模式。**
  - **CSP 允许使用进程组件来描述系统，它们独立运行，并且只通过消息传递的方式通信。**
  - **Channel 类型是 Go 语言内置的类型，你无需引入某个包，就能使用它**。
  - Channel 和 Go 的另一个独特的特性 goroutine 一起为并发编程提供了优雅的、便利的、 与传统并发控制不同的方案，并演化出很多并发模式。

- **Channel的应用场景**

  - Don’t communicate by sharing memory; share memory by communicating. （不要通过共享内存来通信，而应该通过通信来共享内存。）
    - **“communicate by sharing memory”是传统的并发编程处理方式**，就是指，共享的数据需要用锁进行保护，goroutine 需要获取到锁，才能并发访问 数据。
    - **“share memory by communicating”则是类似于 CSP 模型的方式**，通过通信的方式， 一个 goroutine 可以把数据的“所有权”交给另外一个 goroutine(虽然 Go 中没有“所 有权”的概念，但是从逻辑上说，你可以把它理解为是所有权的转移)。
  - Channel 的应用场景分为五种类型
    - **数据交流**:当作并发的 buffer 或者 queue，解决生产者 - 消费者问题。多个 goroutine 可以并发当作生产者(Producer)和消费者(Consumer)。
    - **数据传递**:一个 goroutine 将数据交给另一个 goroutine，相当于把数据的拥有权 (引用) 托付出去。
    - **信号通知**:一个 goroutine 可以将信号 (closing、closed、data ready 等) 传递给另一 个或者另一组 goroutine 。
    - **任务编排**:可以让一组 goroutine 按照一定的顺序并发或者串行的执行，这就是编排的 功能。
    - **锁**:利用 Channel 也可以实现互斥锁的机制。

- **Channel** **基本用法**

  - 可以往 Channel 中发送数据，也可以从 Channel 中接收数据，所以，Channel 类型 (为了说起来方便，我们下面都把 Channel 叫做 chan)分为**只能接收**、**只能发送**、**既可 以接收又可以发送**三种类型。

    - ```go
      // 可以发送接收string 
      1 chan string
      // 只能发送struct{} 
      2 chan<- struct{} 
      // 只能从chan接收int
      3 <-chan int
      ```

  - **chan 中的元素是任意的类型，所以也可能是 chan 类型**

  - “<-”有个规则，总是尽量和左边的 chan 结合

    - ```go
      // <-和第一个chan结合
      chan<- (chan int)
      // 第一个<-和最左边的chan结合，第二个<-和左边第二个chan结合
      chan<- (<-chan int) 
      // 第一个<-和最左边的chan结合，第二个<-和左边第二个chan结合
      <-chan (<-chan int) 
      // 因为括号的原因，<-和括号内第一个chan结合
      chan (<-chan int) 
      ```

  - 通过 make，我们可以初始化一个 chan，未初始化的 chan 的零值是 nil。你可以设置它的 容量，我们把这样的 chan 叫做 buffered chan;如果 没有设置，它的容量是 0，我们把这样的 chan 叫做 unbuffered chan。

  - 通道类型的值本身就是并发安全的，这也是 Go 语言自带的、唯一一个可以满足并发安全性的类型。

  - **nil 是 chan 的零值，是一种特殊的 chan，对值是 nil 的 chan 的发送接收调用者总是会阻塞。**

  - Go 内建的函数 close、cap、len 都可以操作 chan 类型:close 会把 chan 关闭掉，cap 返回 chan 的容量，len 返回 chan 中缓存的还未被取走的元素数量。

    - send 和 recv 都可以作为 select 语句的 case clause
    - chan 还可以应用于 for-range 语句中
    
  - **元素值的发送和接收都需要用到操作符`<-`**。我们也可以叫它接送操作符。一个左尖括号紧接着一个减号形象地代表了元素值的传输方向。

- **Channel** **的实现原理**

  - **chan数据结构**
    - qcount:代表 chan 中已经接收但还没被取走的元素的个数。内建函数 len 可以返回这 个字段的值。
    - dataqsiz:队列的大小。chan 使用一个循环队列来存放元素，循环队列很适合这种生产 者 - 消费者的场景(我很好奇为什么这个字段省略 size 中的 e)。
    - buf:存放元素的循环队列的 buffer。
    - elemtype 和 elemsize:chan 中元素的类型和 size。因为 chan 一旦声明，它的元素 类型是固定的，即普通类型或者指针类型，所以元素大小也是固定的。
    - sendx:处理发送数据的指针在 buf 中的位置。一旦接收了新的数据，指针就会加上 elemsize，移向下一个位置。buf 的总大小是 elemsize 的整数倍，而且 buf 是一个循 环列表。
    - recvx:处理接收请求时的指针在 buf 中的位置。一旦取出数据，此指针会移动到下一 个位置。
    - recvq:chan 是多生产者多消费者的模式，如果消费者因为没有数据可读而被阻塞了， 就会被加入到 recvq 队列中。
    - sendq:如果生产者因为 buf 满了而阻塞，会被加入到 sendq 队列中。
    - **一个通道相当于一个先进先出（FIFO）的队列。**也就是说，通道中的各个元素值都是严格地按照发送的顺序排列的，先被发送通道的元素值一定会先被接收。
    
  - **初始化**
    - Go 在编译的时候，会根据容量的大小选择调用 makechan64，还是 makechan。
    - **我们只关注 makechan 就好了，因为 makechan64 只是做了 size 检查，底层还是调用 makechan 实现的**。makechan 的目标就是生成 hchan 对象。
    - makechan 的主要逻辑
      -  它会根据 chan 的容量的大小和元素的类型不同，初始化不同的存储空间
      - 最终，针对不同的容量和元素类型，这段代码分配了不同的对象来初始化 hchan 对象的字 段，返回 hchan 对象。

  - 对通道的发送和接收操作都有哪些基本的特性

    - 对于同一个通道，发送操作之间是互斥的，接收操作之间也是**互斥**的。
      - **这里要注意的一个细节是，元素值从外界进入通道时会被复制。更具体地说，进入通道的并不是在接收操作符右边的那个元素值，而是它的副本。**
    - 发送操作和接收操作中对元素值的处理都是**不可分割**的。
    - 发送操作在完全完成之前会被**阻塞**。接收操作也是如此。

  - 发送操作和接收操作在什么时候可能被长时间的阻塞

    - 先说针对**缓冲通道**的情况。如果通道已满，那么对它的所有发送操作都会被阻塞，直到通道中有元素值被接收走。
      - 相对的，如果通道已空，那么对它的所有接收操作都会被阻塞，直到通道中有新的元素值出现。这时，通道会通知最早等待的那个接收操作所在的 goroutine，并使它再次执行接收操作。
    - 对于**非缓冲通道**，情况要简单一些。无论是发送操作还是接收操作，一开始执行就会被阻塞，直到配对的操作也开始执行，才会继续传递。由此可见，非缓冲通道是在用同步的方式传递数据。也就是说，只有收发双方对接上了，数据才会被传递。
    - 对于值为`nil`的通道，不论它的具体类型是什么，对它的发送操作和接收操作都会永久地处于阻塞状态。它们所属的 goroutine 中的任何代码，都不再会被执行。

  - 发送操作和接收操作在什么时候会引发 panic

    - 对于一个已初始化，但并未关闭的通道来说，收发操作一定不会引发 panic。但是通道一旦关闭，再对它进行发送操作，就会引发 panic。
    - 另外，如果我们试图关闭一个已经关闭了的通道，也会引发 panic。注意，接收操作是可以感知到通道的关闭的，并能够安全退出。
      - 更具体地说，当我们把接收表达式的结果同时赋给两个变量时，第二个变量的类型就是一定`bool`类型。它的值如果为`false`就说明通道已经关闭，并且再没有元素值可取了。
      - 注意，如果通道关闭时，里面还有元素值未被取出，那么接收表达式的第一个结果，仍会是通道中的某一个元素值，而第二个结果值一定会是`true`。
      - 因此，通过接收表达式的第二个结果值，来判断通道是否关闭是可能有延时的。
    - 由于通道的收发操作有上述特性，所以除非有特殊的保障措施，我们千万不要让接收方关闭通道，而应当让发送方做这件事。

  - **send**
    
    - Go 在编译发送数据给 chan 的时候，会把 send 语句转换成 chansend1 函数， chansend1 函数会调用 chansend
    - 最开始，第一部分是进行判断:如果 chan 是 nil 的话，就把调用者 goroutine park(阻 塞休眠)， 调用者就永远被阻塞住了
    - 第二部分的逻辑是当你往一个已经满了的 chan 实例发送数据时，并且想不阻塞当前调 用，那么这里的逻辑是直接返回。chansend1 方法在调用 chansend 的时候设置了阻塞参 数，所以不会执行到第二部分的分支里。
    - 第三部分显示的是，如果 chan 已经被 close 了，再往里面发送数据的话会 panic。
    - 第四部分，如果等待队列中有等待的 receiver，那么这段代码就把它从队列中弹出，然后 直接把数据交给它(通过 memmove(dst, src, t.size))，而不需要放入到 buf 中，速度可 以更快一些。
    - 第五部分说明当前没有 receiver，需要把数据放入到 buf 中，放入之后，就成功返回了。
    - 第六部分是处理 buf 满的情况。如果 buf 满了，发送者的 goroutine 就会加入到发送者的 等待队列中，直到被唤醒。这个时候，数据或者被取走了，或者 chan 被 close 了。
    
  - **recv**
    - 处理从 chan 中接收数据时，Go 会把代码转换成 chanrecv1 函数，如果要返回两个返 回值，会转换成 chanrecv2，chanrecv1 函数和 chanrecv2 会调用 chanrecv。
    - 第一部分是 chan 为 nil 的情况。和 send 一样，从 nil chan 中接收(读取、获取)数据 时，调用者会被永远阻塞。
    - 第二部分是 chan 已经被 close 的情况。如果 chan 已经被 close 了，并且队列中没有缓存 的元素，那么返回 true、false。
    - 第三部分是处理 sendq 队列中有等待者的情况。这个时候，如果 buf 中有数据，优先从 buf 中读取数据，否则直接从等待队列中弹出一个 sender，把它的数据复制给这个 receiver。
    - 第四部分是处理没有等待的 sender 的情况。这个是和 chansend 共用一把大锁，所以不 会有并发的问题。如果 buf 有元素，就取出一个元素给 receiver。
    - 第五部分是处理 buf 中没有元素的情况。如果没有元素，那么当前的 receiver 就会被阻 塞，直到它从 sender 中接收了数据，或者是 chan 被 close，才返回。
    
  - **close**
    - 通过 close 函数，可以把 chan 关闭，编译器会替换成 closechan 方法的调用。
    - 如果 chan 为 nil，close 会 panic;如果 chan 已 经 closed，再次 close 也会 panic。否则的话，如果 chan 不为 nil，chan 也没有 closed，就把等待队列中的 sender(writer)和 receiver(reader)从队列中全部移除并 唤醒。
    
  - chan 的值和状态有多种情况，而不同的操作（send、recv、close）又可能得到不同的结果，这是使用 chan 类型时经常让人困惑的地方。

    - 你一定要特别关注下那些 panic 的情况，另外还要掌握那些会 block 的场景，它们是导致死锁或者 goroutine 泄露的罪魁祸首。

    - 还有一个值得注意的点是，**只要一个 chan 还有未读的数据，即使把它 close 掉，你还是可以继续把这些未读的数据消费完，之后才是读取零值数据。**

    - |         | nil       | empty                | Full                   | Not full&empty         | Closed                         |
      | ------- | --------- | -------------------- | ---------------------- | ---------------------- | ------------------------------ |
      | receive | block     | block                | Read value             | Read value             | 返回未读的元素，读完后返回零值 |
      | send    | block     | write value          | block                  | write value            | **panic**                      |
      | close   | **panic** | closed，没有未读元素 | closed，保留未读的元素 | closed，保留未读的元素 | **panic**                      |

- 单向通道有什么应用价值

  - 概括地说，单向通道最主要的用途就是约束其他代码的行为。

  - ```go
    // 约束函数行为
    func SendInt(ch chan<- int) {
    	ch <- rand.Intn(1000)
    }
    
    ---
    
    // 我们在调用SendInt函数的时候，只需要把一个元素类型匹配的双向通道传给它就行了，没必要用发送通道，因为 Go 语言在这种情况下会自动地把双向通道转换为函数所需的单向通道。
    
    type Notifier interface {
    	SendInt(ch chan<- int)
    }
    
    intChan1 := make(chan int, 3)
    SendInt(intChan1)
    
    ---
    
    // 函数getIntChan会返回一个<-chan int类型的通道，这就意味着得到该通道的程序，只能从通道中接收元素值。这实际上就是对函数调用方的一种约束了。
    func getIntChan() <-chan int {
    	num := 5
    	ch := make(chan int, num)
    	for i := 0; i < num; i++ {
    		ch <- i
    	}
    	close(ch)
    	return ch
    }
    ```

- `select`语句与通道怎样联用，应该注意些什么

  - `select`语句只能与通道联用，它一般由若干个分支组成。每次执行这种语句的时候，一般只有一个分支中的代码会被运行。
  - 由于`select`语句是专为通道而设计的，所以每个`case`表达式中都只能包含操作通道的表达式，比如接收表达式。
  - `select`语句只能对其中的每一个`case`表达式各求值一次。所以，如果我们想连续或定时地操作其中的通道的话，就往往需要通过在`for`语句中嵌入`select`语句的方式实现。但这时要注意，**简单地在`select`语句的分支中使用`break`语句，只能结束当前的`select`语句的执行，而并不会对外层的`for`语句产生作用**。这种错误的用法可能会让这个`for`语句无休止地运行下去。

- 如果在`select`语句中发现某个通道已关闭，那么应该怎样屏蔽掉它所在的分支

  - 如果判断到chan关闭，即取到的第二个值为false。则将该chan赋值为nil。

- 在`select`语句与`for`语句联用时，怎样直接退出外层的`for`语句

  - ```go
    // 可以用 break和标签配合使用，直接break出指定的循环体，或者goto语句直接跳转到指定标签执行
    
    break配合标签：
    
    ch1 := make(chan int, 1)
    time.AfterFunc(time.Second, func() { close(ch1) })
    loop:
    for {
    select {
    case _, ok := <-ch1:
    if !ok {
    break loop
    }
    fmt.Println("ch1")
    }
    }
    fmt.Println("END")
    
    ---
    
    goto配合标签：
    
    ch1 := make(chan int, 1)
    time.AfterFunc(time.Second, func() { close(ch1) })
    for {
    select {
    case _, ok := <-ch1:
    if !ok {
    goto loop
    }
    fmt.Println("ch1")
    }
    }
    loop:
    fmt.Println("END")
    ```

- 使用 Channel 容易犯的错误

  - **使用 Channel 最常见的错误是 panic 和 goroutine 泄漏。**

  - 首先，我们来总结下会 panic 的情况，总共有 3 种：

    - close 为 nil 的 chan；
    - send 已经 close 的 chan；
    - close 已经 close 的 chan。

  - goroutine 泄漏的问题也很常见，下面的代码也是一个实际项目中的例子：

    - ```go
      // 在这个例子中，process 函数会启动一个 goroutine，去处理需要长时间处理的业务，处理完之后，会发送 true 到 chan 中，目的是通知其它等待的 goroutine，可以继续处理了。
      func process(timeout time.Duration) bool {
          ch := make(chan bool)
      
          go func() {
              // 模拟处理耗时的业务
              time.Sleep((timeout + time.Second))
              ch <- true // block
              fmt.Println("exit goroutine")
          }()
          select {
          case result := <-ch:
              return result
          case <-time.After(timeout):
              return false
          }
      }
      ```

    - 我们来看一下第 10 行到第 15 行，主 goroutine 接收到任务处理完成的通知，或者超时后就返回了。这段代码有问题吗？

      - 如果发生超时，process 函数就返回了，这就会导致 unbuffered 的 chan 从来就没有被读取。我们知道，unbuffered chan 必须等 reader 和 writer 都准备好了才能交流，否则就会阻塞。超时导致未读，结果就是子 goroutine 就阻塞在第 7 行永远结束不了，进而导致 goroutine 泄漏。

    - 解决这个 Bug 的办法很简单，就是将 unbuffered chan 改成容量为 1 的 chan，这样第 7 行就不会被阻塞了。

- 透过代码看典型的应用模式

  - 使用反射操作 Channel

    - 如果需要处理三个 chan，你就可以再添加一个 case clause，用它来处理第三个 chan。可是，如果要处理 100 个 chan 呢？一万个 chan 呢？或者是，chan 的数量在编译的时候是不定的，在运行的时候需要处理一个 slice of chan，这个时候，也没有办法在编译前写成字面意义的 select。那该怎么办？

    - 通过 reflect.Select 函数，你可以将一组运行时的 case clause 传入，当作参数执行。Go 的 select 是伪随机的，它可以在执行的 case 中随机选择一个 case，并把选择的这个 case 的索引（chosen）返回，如果没有可用的 case 返回，会返回一个 bool 类型的返回值，这个返回值用来表示是否有 case 成功被选择。如果是 recv case，还会返回接收的元素。

    - ```go
      // 首先，createCases 函数分别为每个 chan 生成了 recv case 和 send case，并返回一个 reflect.SelectCase 数组。
      // 然后，通过一个循环 10 次的 for 循环执行 reflect.Select，这个方法会从 cases 中选择一个 case 执行。第一次肯定是 send case，因为此时 chan 还没有元素，recv 还不可用。等 chan 中有了数据以后，recv case 就可以被选择了。这样，你就可以处理不定数量的 chan 了。
      func main() {
          var ch1 = make(chan int, 10)
          var ch2 = make(chan int, 10)
      
          // 创建SelectCase
          var cases = createCases(ch1, ch2)
      
          // 执行10次select
          for i := 0; i < 10; i++ {
              chosen, recv, ok := reflect.Select(cases)
              if recv.IsValid() { // recv case
                  fmt.Println("recv:", cases[chosen].Dir, recv, ok)
              } else { // send case
                  fmt.Println("send:", cases[chosen].Dir, ok)
              }
          }
      }
      
      func createCases(chs ...chan int) []reflect.SelectCase {
          var cases []reflect.SelectCase
      
      
          // 创建recv case
          for _, ch := range chs {
              cases = append(cases, reflect.SelectCase{
                  Dir:  reflect.SelectRecv,
                  Chan: reflect.ValueOf(ch),
              })
          }
      
          // 创建send case
          for i, ch := range chs {
              v := reflect.ValueOf(i)
              cases = append(cases, reflect.SelectCase{
                  Dir:  reflect.SelectSend,
                  Chan: reflect.ValueOf(ch),
                  Send: v,
              })
          }
      
          return cases
      }
      ```

- 典型的应用场景

  - 消息交流

    - 从 chan 的内部实现看，它是以一个循环队列的方式存放数据，所以，它有时候也会被当成线程安全的队列和 buffer 使用。一个 goroutine 可以安全地往 Channel 中塞数据，另外一个 goroutine 可以安全地从 Channel 中读取数据，goroutine 就可以安全地实现信息交流了。
    - 第一个例子是 worker 池的例子。将用户的请求放在一个 chan Job 中，这个 chan Job 就相当于一个待处理任务队列。除此之外，还有一个 chan chan Job 队列，用来存放可以处理任务的 worker 的缓存队列。
      - dispatcher 会把待处理任务队列中的任务放到一个可用的缓存队列中，worker 会一直处理它的缓存队列。通过使用 Channel，实现了一个 worker 池的任务处理中心，并且解耦了前端 HTTP 请求处理和后端任务处理的逻辑。
      - worker 池的生产者和消费者的消息交流都是通过 Channel 实现的。
    - 第二个例子是 etcd 中的 node 节点的实现，包含大量的 chan 字段，比如 recvc 是消息处理的 chan，待处理的 protobuf 消息都扔到这个 chan 中，node 有一个专门的 run goroutine 处理这些消息。

  - 数据传递

    - 有 4 个 goroutine，编号为 1、2、3、4。每秒钟会有一个 goroutine 打印出它自己的编号，要求你编写程序，让输出的编号总是按照 1、2、3、4、1、2、3、4……这个顺序打印出来。

    - ```go
      // 为了实现顺序的数据传递，我们可以定义一个令牌的变量，谁得到令牌，谁就可以打印一次自己的编号，同时将令牌传递给下一个 goroutine，我们尝试使用 chan 来实现，可以看下下面的代码。
      
      // 首先，我们定义一个令牌类型（Token），接着定义一个创建 worker 的方法，这个方法会从它自己的 chan 中读取令牌。哪个 goroutine 取得了令牌，就可以打印出自己编号，因为需要每秒打印一次数据，所以，我们让它休眠 1 秒后，再把令牌交给它的下家。
      type Token struct{}
      
      func newWorker(id int, ch chan Token, nextCh chan Token) {
          for {
              token := <-ch         // 取得令牌
              fmt.Println((id + 1)) // id从1开始
              time.Sleep(time.Second)
              nextCh <- token
          }
      }
      func main() {
          chs := []chan Token{make(chan Token), make(chan Token), make(chan Token), make(chan Token)}
      
          // 创建4个worker
          for i := 0; i < 4; i++ {
              go newWorker(i, chs[i], chs[(i+1)%4])
          }
      
          //首先把令牌交给第一个worker
          chs[0] <- struct{}{}
        
          select {}
      }
      ```

  - 信号通知

    - chan 类型有这样一个特点：**chan 如果为空，那么，receiver 接收数据的时候就会阻塞等待，直到 chan 被关闭或者有新的数据到来**。利用这个机制，我们可以实现 wait/notify 的设计模式。
    - 传统的并发原语 Cond 也能实现这个功能，但是，Cond 使用起来比较复杂，容易出错，而使用 chan 实现 wait/notify 模式就方便很多了。
    - 除了正常的业务处理时的 wait/notify，我们经常碰到的一个场景，就是程序关闭的时候，我们需要在退出之前做一些清理（doCleanup 方法）的动作。这个时候，我们经常要使用 chan。

  - 锁

    - **使用 chan 也可以实现互斥锁**。在 chan 的内部实现中，就有一把互斥锁保护着它的所有字段。从外在表现上，chan 的发送和接收之间也存在着 happens-before 的关系，保证元素放进去之后，receiver 才能读取到

    - 要想使用 chan 实现互斥锁，至少有两种方式。**一种方式是先初始化一个 capacity 等于 1 的 Channel，然后再放入一个元素。这个元素就代表锁，谁取得了这个元素，就相当于获取了这把锁。**另一种方式是，先初始化一个 capacity 等于 1 的 Channel，它的“空槽”代表锁，谁能成功地把元素发送到这个 Channel，谁就获取了这把锁。

    - ```go
      // 第一种
      // 使用chan实现互斥锁
      type Mutex struct {
          ch chan struct{}
      }
      
      // 使用锁需要初始化
      func NewMutex() *Mutex {
          mu := &Mutex{make(chan struct{}, 1)}
          mu.ch <- struct{}{}
          return mu
      }
      
      // 请求锁，直到获取到
      func (m *Mutex) Lock() {
          <-m.ch
      }
      
      // 解锁
      func (m *Mutex) Unlock() {
          select {
          case m.ch <- struct{}{}:
          default:
              panic("unlock of unlocked mutex")
          }
      }
      
      // 尝试获取锁
      func (m *Mutex) TryLock() bool {
          select {
          case <-m.ch:
              return true
          default:
          }
          return false
      }
      
      // 加入一个超时的设置
      func (m *Mutex) LockTimeout(timeout time.Duration) bool {
          timer := time.NewTimer(timeout)
          select {
          case <-m.ch:
              timer.Stop()
              return true
          case <-timer.C:
          }
          return false
      }
      
      // 锁是否已被持有
      func (m *Mutex) IsLocked() bool {
          return len(m.ch) == 0
      }
      
      
      func main() {
          m := NewMutex()
          ok := m.TryLock()
          fmt.Printf("locked v %v\n", ok)
          ok = m.TryLock()
          fmt.Printf("locked %v\n", ok)
      }
      ```

  - 任务编排

    - Or-Done 模式

      - Or-Done 模式是信号通知模式中更宽泛的一种模式

      - 我们会使用“信号通知”实现某个任务执行完成后的通知机制，在实现时，我们为这个任务定义一个类型为 chan struct{}类型的 done 变量，等任务结束后，我们就可以 close 这个变量，然后，其它 receiver 就会收到这个通知。

      - 这是有一个任务的情况，如果有多个任务，**只要有任意一个任务执行完，我们就想获得这个信号，这就是 Or-Done 模式。**

      - 比如，你发送同一个请求到多个微服务节点，只要任意一个微服务节点返回结果，就算成功，这个时候，就可以参考下面的实现：

      - ```go
        func or(channels ...<-chan interface{}) <-chan interface{} {
            // 特殊情况，只有零个或者1个chan
            switch len(channels) {
            case 0:
                return nil
            case 1:
                return channels[0]
            }
        
            orDone := make(chan interface{})
            go func() {
                defer close(orDone)
        
                switch len(channels) {
                case 2: // 2个也是一种特殊情况
                    select {
                    case <-channels[0]:
                    case <-channels[1]:
                    }
                // 这里的实现使用了一个巧妙的方式，当 chan 的数量大于 2 时，使用递归的方式等待信号。
                default: //超过两个，二分法递归处理
                    m := len(channels) / 2
                    select {
                    case <-or(channels[:m]...):
                    case <-or(channels[m:]...):
                    }
                }
            }()
        
            return orDone
        }
        
        
        
        // 反射的方法，我们也可以实现 Or-Done 模式：
        func or(channels ...<-chan interface{}) <-chan interface{} {
            //特殊情况，只有0个或者1个
            switch len(channels) {
            case 0:
                return nil
            case 1:
                return channels[0]
            }
        
            orDone := make(chan interface{})
            go func() {
                defer close(orDone)
                // 利用反射构建SelectCase
                var cases []reflect.SelectCase
                for _, c := range channels {
                    cases = append(cases, reflect.SelectCase{
                        Dir:  reflect.SelectRecv,
                        Chan: reflect.ValueOf(c),
                    })
                }
        
                // 随机选择一个可用的case
                reflect.Select(cases)
            }()
        
        
            return orDone
        }
        
        func sig(after time.Duration) <-chan interface{} {
            c := make(chan interface{})
            go func() {
                defer close(c)
                time.Sleep(after)
            }()
            return c
        }
        
        
        func main() {
            start := time.Now()
        
            <-or(
                sig(10*time.Second),
                sig(20*time.Second),
                sig(30*time.Second),
                sig(40*time.Second),
                sig(50*time.Second),
                sig(01*time.Minute),
            )
        
            fmt.Printf("done after %v", time.Since(start))
        }
        ```

    - 扇入模式

      - 对于我们这里的 Channel 扇入模式来说，就是指有**多个源 Channel 输入、一个目的 Channel 输出的情况**。扇入比就是源 Channel 数量比 1。

      - 每个源 Channel 的元素都会发送给目标 Channel，相当于目标 Channel 的 receiver 只需要监听目标 Channel，就可以接收所有发送给源 Channel 的数据。

      - ```go
        // 反射的代码比较简短，易于理解，主要就是构造出 SelectCase slice，然后传递给 reflect.Select 语句。
        
        func fanInReflect(chans ...<-chan interface{}) <-chan interface{} {
            out := make(chan interface{})
            go func() {
                defer close(out)
                // 构造SelectCase slice
                var cases []reflect.SelectCase
                for _, c := range chans {
                    cases = append(cases, reflect.SelectCase{
                        Dir:  reflect.SelectRecv,
                        Chan: reflect.ValueOf(c),
                    })
                }
                
                // 循环，从cases中选择一个可用的
                for len(cases) > 0 {
                    i, v, ok := reflect.Select(cases)
                    if !ok { // 此channel已经close
                        cases = append(cases[:i], cases[i+1:]...)
                        continue
                    }
                    out <- v.Interface()
                }
            }()
            return out
        }
        
        // 递归模式也是在 Channel 大于 2 时，采用二分法递归 merge。
        
        func fanInRec(chans ...<-chan interface{}) <-chan interface{} {
            switch len(chans) {
            case 0:
                c := make(chan interface{})
                close(c)
                return c
            case 1:
                return chans[0]
            case 2:
                return mergeTwo(chans[0], chans[1])
            default:
                m := len(chans) / 2
                return mergeTwo(
                    fanInRec(chans[:m]...),
                    fanInRec(chans[m:]...))
            }
        }
        ```

    - 扇出模式

      - ```go
        // 下面是一个扇出模式的实现。从源 Channel 取出一个数据后，依次发送给目标 Channel。在发送给目标 Channel 的时候，可以同步发送，也可以异步发送：
        
        func fanOut(ch <-chan interface{}, out []chan interface{}, async bool) {
            go func() {
                defer func() { //退出时关闭所有的输出chan
                    for i := 0; i < len(out); i++ {
                        close(out[i])
                    }
                }()
        
                for v := range ch { // 从输入chan中读取数据
                    v := v
                    for i := 0; i < len(out); i++ {
                        i := i
                        if async { //异步
                            go func() {
                                out[i] <- v // 放入到输出chan中,异步方式
                            }()
                        } else {
                            out[i] <- v // 放入到输出chan中，同步方式
                        }
                    }
                }
            }()
        }
        ```

    - Stream

      - ```go
        // 把 Channel 当作流式管道使用的方式，也就是把 Channel 看作流（Stream），提供跳过几个元素，或者是只取其中的几个元素等方法。
        func asStream(done <-chan struct{}, values ...interface{}) <-chan interface{} {
            s := make(chan interface{}) //创建一个unbuffered的channel
            go func() { // 启动一个goroutine，往s中塞数据
                defer close(s) // 退出时关闭chan
                for _, v := range values { // 遍历数组
                    select {
                    case <-done:
                        return
                    case s <- v: // 将数组元素塞入到chan中
                    }
                }
            }()
            return s
        }
        
        // 以 takeN 为例来具体解释一下。
        func takeN(done <-chan struct{}, valueStream <-chan interface{}, num int) <-chan interface{} {
            takeStream := make(chan interface{}) // 创建输出流
            go func() {
                defer close(takeStream)
                for i := 0; i < num; i++ { // 只读取前num个元素
                    select {
                    case <-done:
                        return
                     // https://stackoverflow.com/questions/60030756/what-does-it-mean-when-one-channel-uses-two-arrows-to-write-to-another-channel-i
                    case takeStream <- <-valueStream: //从输入流中读取元素
                    }
                }
            }()
            return takeStream
        }
        ```

      - 实现流的方法。

        - takeN：只取流中的前 n 个数据；
        - takeFn：筛选流中的数据，只保留满足条件的数据；
        - takeWhile：只取前面满足条件的数据，一旦不满足条件，就不再取；
        - skipN：跳过流中前几个数据；
        - skipFn：跳过满足条件的数据；
        - skipWhile：跳过前面满足条件的数据，一旦不满足条件，当前这个元素和以后的元素都会输出给 Channel 的 receiver。

    - 和 Map-Reduce

      - ```go
        func mapChan(in <-chan interface{}, fn func(interface{}) interface{}) <-chan interface{} {
            out := make(chan interface{}) //创建一个输出chan
            if in == nil { // 异常检查
                close(out)
                return out
            }
        
            go func() { // 启动一个goroutine,实现map的主要逻辑
                defer close(out)
                for v := range in { // 从输入chan读取数据，执行业务操作，也就是map操作
                    out <- fn(v)
                }
            }()
        
            return out
        }
        
        
        func reduce(in <-chan interface{}, fn func(r, v interface{}) interface{}) interface{} {
            if in == nil { // 异常检查
                return nil
            }
        
            out := <-in // 先读取第一个元素
            for v := range in { // 实现reduce的主要逻辑
                out = fn(out, v)
            }
        
            return out
        }
        
        
        // 生成一个数据流
        func asStream(done <-chan struct{}) <-chan interface{} {
            s := make(chan interface{})
            values := []int{1, 2, 3, 4, 5}
            go func() {
                defer close(s)
                for _, v := range values { // 从数组生成
                    select {
                    case <-done:
                        return
                    case s <- v:
                    }
                }
            }()
            return s
        }
        
        func main() {
            in := asStream(nil)
        
            // map操作: 乘以10
            mapFn := func(v interface{}) interface{} {
                return v.(int) * 10
            }
        
            // reduce操作: 对map的结果进行累加
            reduceFn := func(r, v interface{}) interface{} {
                return r.(int) + v.(int)
            }
        
            sum := reduce(mapChan(in, mapFn), reduceFn) //返回累加结果
            fmt.Println(sum)
        }
        ```











#  **内存模型**

- **它描述的是并发环境中多goroutine 读相同变量的时候，变量的可见性条件。**具体点说，就是指，在什么条件下，goroutine 在读取一个变量的值的时候，能够看到其它 goroutine 对这个变量进行的写的结果。

  - 由于 CPU 指令重排和多级 Cache 的存在，保证多核访问同一个变量这件事儿变得非常复 杂。
  - 除了 Go，Java、C++、C、C#、Rust 等编程语言也有内存模型。为什么这些编程语言都 要定义内存模型呢?在我看来，主要是两个目的。
    - 向广大的程序员提供一种保证，以便他们在做设计和开发程序时，面对同一个数据同时 被多个 goroutine 访问的情况，可以做一些串行化访问的控制，比如使用 Channel 或 者 sync 包和 sync/atomic 包中的并发原语。
    - 允许编译器和硬件对程序做一些优化。这一点其实主要是为编译器开发者提供的保证， 这样可以方便他们对 Go 的编译器做优化。

- **重排和可见性的问题**

  - **由于指令重排，代码并不一定会按照你写的顺序执行**
    - 举个例子，当两个 goroutine 同时对一个数据进行读写时，假设 goroutine g1 对这个变 量进行写操作 w，goroutine g2 同时对这个变量进行读操作 r，那么，如果 g2 在执行读 操作 r 的时候，已经看到了 g1 写操作 w 的结果，那么，也不意味着 g2 能看到在 w 之前的其它的写操作。这是一个反直观的结果，不过的确可能会存在。
  - **重排以及多核 CPU 并发执行导致程序的运行和代码的书写顺序不一样的情况。**

- **happens-before**

  - Go 内存模型中很重要的一个概念:happens-before，这是**用来描述两个时间的顺序关系的**。**如果某些操作能提供 happens-before 关系，那么，我们就可 以 100% 保证它们之间的顺序。**
  - **在一个 goroutine 内部，程序的执行顺序和它们的代码指定的顺序是一样的，即使编译器或者 CPU 重排了读写顺序，从行为上来看，也和代码指定的顺序一样。**
    - 但是，对于另一个 goroutine 来说，重排却会产生非常大的影响。**因为 Go 只保证 goroutine 内部重排对读写的顺序没有影响**
    - Go 内存模型通过 happens-before 定义两个事件(读、写 action)的顺序:如果事件 e1 happens before 事件 e2，那么，我们就可以说事件 e2 在事件 e1 之后发生(happens after)。如果 e1 不是 happens before e2， 同时也不 happens after e2，那么，我们 就可以说事件 e1 和 e2 是同时发生的。
    - **在 goroutine 内部对一个局部变量 v 的读，一定能观察到最近一次对这个局部变量 v 的 写。如果要保证多个 goroutine 之间对一个共享变量的读写顺序，在 Go 语言中，可以使 用并发原语为读写操作建立 happens-before 关系，这样就可以保证顺序了。**

- 三个 Go 语言中和内存模型有关的小知识

  - 在 Go 语言中，**对变量进行零值的初始化就是一个写操作。**
  - 如果对**超过机器 word**(64bit、32bit 或者其它)大小的值进行读写，那么，就可以看 作是**对拆成 word 大小的几个读写无序进行。**
  - Go 并不提供直接的 CPU 屏障(CPU fence)来提示编译器或者 CPU 保证顺序性，而是**使用不同架构的内存屏障指令来实现统一的并发原语。**

- **Go** **语言中保证的** **happens-before** **关系**

  - **init函数**
    - 应用程序的初始化是在单一的 goroutine 执行的。如果包 p 导入了包 q，那么，q 的 init函数的执行一定 happens before p 的任何初始化代码。
    - 这里有一个特殊情况需要你记住:**main 函数一定在导入的包的 init 函数之后执行**。
    - 这个例子是一个 **main** 程序，它依赖包 p1，包 p1 依赖包 p2，包 p2 依赖 p3。
      - P3变量初始化-P3的init函数-P2变量初始化-P2的init函数-P1变量初始化-P1的init函数-main的init函数-main函数

- **goroutine**

  - 首先，我们需要明确一个规则:**启动 goroutine 的 go 语句的执行，一定 happens before 此 goroutine 内的代码执行。**
    - 根据这个规则，我们就可以知道，如果 go 语句传入的参数是一个函数执行的结果，那 么，这个函数一定先于 goroutine 内部的代码被执行。
    - 刚刚说的是启动 goroutine 的情况，**goroutine 退出的时候，是没有任何 happens- before 保证的。**所以，如果你想观察某个 goroutine 的执行效果，你需要使用同步机制建 立 happens-before 关系，比如 Mutex 或者 Channel

- **Channel**

  - 通用的 Channel happens-before 关系保证有 4 条规则

    - **第 1 条规则是**，往 Channel 中的发送操作，happens before 从该 Channel 接收相应数 据的动作完成之前

    - **第 2 条规则是**，close 一个 Channel 的调用，肯定 happens before 从关闭的 Channel 中读取出一个零值。

    - **第 3 条规则是**，对于 unbuffered 的 Channel，也就是容量是 0 的 Channel，从此 Channel 中读取数据的调用一定 happens before 往此 Channel 发送数据的调用完成。

      - ```go
        var ch = make(chan int)
        var s string
        func f(){
        	s = "hello"
        	<-ch
        }
        func main(){
        	go f()
        	ch <- struct{}{}
        	print(s)
        }
        ```

      - 如果第 11 行发送语句执行成功(完毕)，那么根据这个规则，第 6 行(接收)的调用肯 定发生了(执行完成不完成不重要，重要的是这一句“肯定执行了”)，那么 s 也肯定初 始化了，所以一定会打印出“hello world”。

    - **第 4 条规则是**，如果 Channel 的容量是 m(m>0)，那么，第 n 个 receive 一定 happens before 第 n+m 个 send 的完成。

- **Mutex/RWMutex**

  - 第 n 次的 m.Unlock 一定 happens before 第 n+1 m.Lock 方法的返回;
  - 对于读写锁 l 的 l.RLock 方法调用，如果存在一个 **n**，这次的 l.RLock 调用 happens after 第 n 次的 l.Unlock，那么，和这个 RLock 相对应的 l.RUnlock 一定 happens before 第 n+1 次 l.Lock。意思是，**读写锁的 Lock 必须等待既有的读锁释放后才能获 取到。**

- **WaitGroup**

  - 对于一个 WaitGroup 实例 wg，在某个时刻 t0 时，它的计数值已经不是零了，假如 t0 时 刻之后调用了一系列的 wg.Add(n) 或者 wg.Done()，并且只有最后一次调用 wg 的计数值 变为了 0，那么，可以保证这些 wg.Add 或者 wg.Done() 一定 happens before t0 时刻 之后调用的 wg.Wait 方法的返回。
  - 这个保证的通俗说法，就是 **Wait 方法等到计数值归零之后才返回**。

- **Once**

  - 它提供的保证是:**对于 once.Do(f) 调用，f 函数的那个单次调用一定 happens before 任何 once.Do(f) 调用的 返回**。换句话说，就是函数 f 一定会在 Do 方法返回之前执行。

- **atomic**

  - 对于 Go 1.15 的官方实现来说，可以保证使用 atomic 的 Load/Store 的变量之间的顺序 性。

  - 在下面的例子中，打印出的 a 的结果总是 1，但是官方并没有做任何文档上的说明和保 证。

    - ```go
      func main(){
      	var a,b int32 = 0,0
      	
      	go func(){
      		atomic.StoreInt32(&a,1)
      		atomic.StoreInt32(&b,1)
      	}()
      	
      	for atomic.LoadInt32(&b) == 0{
      		runtime.Gosched()
      	}
      	fmt.Println(atomic.LoadInt32(&a))
      	
      }
      ```

  




# 总结

- 在**数据类型**方面有：
  - 基于底层数组的切片；
  - 用来传递数据的通道；
  - 作为一等类型的函数；
  - 可实现面向对象的结构体；
  - 能无侵入实现的接口等。
- 在**语法**方面有：
  - 异步编程神器`go`语句；
  - 函数的最后关卡`defer`语句；
  - 可做类型判断的`switch`语句；
  - 多通道操作利器`select`语句；
  - 非常有特色的异常处理函数`panic`和`recover`。
- 除了这些，我们还一起讨论了**测试 Go 程序**的主要方式。这涉及了 Go 语言自带的程序测试套件，相关的概念和工具包括：
  - 独立的测试源码文件；
  - 三种功用不同的测试函数；
  - 专用的`testing`代码包；
  - 功能强大的`go test`命令。
- 另外，就在前不久，我还为你深入讲解了 Go 语言提供的那些**同步工具**。它们也是 Go 语言并发编程工具箱中不可或缺的一部分。这包括了：
  - 经典的互斥锁；
  - 读写锁；
  - 条件变量；
  - 原子操作。
- 以及**Go 语言特有的一些数据类型**，即：
  - 单次执行小助手`sync.Once`；
  - 临时对象池`sync.Pool`；
  - 帮助我们实现多 goroutine 协作流程的`sync.WaitGroup`、`context.Context`；
  - 一种高效的并发安全字典`sync.Map`。



