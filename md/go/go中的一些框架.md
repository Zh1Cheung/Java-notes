# go的包

- `string`
  - 在 Go 语言中，**`string`类型的值是不可变的**。 如果我们想获得一个不一样的字符串，那么就只能基于原字符串进行**裁剪、拼接**等操作，从而生成一个新的字符串。

    - 裁剪操作可以使用切片表达式；
    - 拼接操作可以用操作符`+`实现。
  - **与`string`值相比，`Builder`值的优势其实主要体现在字符串拼接方面**
  - `strings.Reader`类型的值可以**高效地读取字符串**
- `bytes.Buffer`
  - `strings`包主要面向的是 Unicode **字符**和经过 UTF-8 编码的**字符串**，而`bytes`包面对的则主要是**字节和字节切片**。
  - **可以说，`bytes.Buffer`是集读、写功能于一身的数据类型**。当然了，这些也基本上都是作为一个缓冲区应该拥有的功能。
  - 在`bytes.Buffer`中，**`Bytes`方法和`Next`方法**都可能会造成内容的泄露。原因在于，**它们都把基于内容容器的切片直接返回给了方法的调用方。**
- 对比`strings.Builder`和`bytes.Buffer`的`String`方法，并判断哪一个更高效
  - 我们可以直接查看两个String方法的源代码，其中strings.Builder String方法中
    *(*string)(unsafe.Pointer(&b.buf)) 是直接取得buf的地址然后转换成string返回。
    而bytes.Buffer的String方法是 string(b.buf[b.off:])
  - string(b.buf[b.off:])，这个是深拷贝，string()操作新建了一个底层数组，然后将b.buf的内容拷贝到新数组。而strings的方法是浅拷贝，所以性能会更高
- `io`
  - 有的接口有着众多的扩展接口和实现类型，我们可以称之为**核心接口**。**`io`包中的核心接口只有 3 个，它们是：`io.Reader`、`io.Writer`和`io.Closer`。**
- `bufio`
  - `bufio`包中的数据类型主要有：
    1. `Reader`；
    2. `Scanner`；
    3. `Writer`和`ReadWriter`。
- `os`
  - 这些 API 基于（或者说抽象自）操作系统，为我们使用操作系统的功能提供高层次的支持，但是，**它们并不依赖于具体的操作系统**。



# go基础

- 怎样判断一个变量的类型

  - ```go
    // 使用“类型断言”表达式
    // 它包括了用来把container变量的值转换为空接口值的interface{}(container)。以及一个用于判断前者的类型是否为切片类型 []string 的 .([]string)。
    value, ok := interface{}(container).([]string)
    ```

- 通道

  - Don’t communicate by sharing memory; share memory by communicating. （不要通过共享内存来通信，而应该通过通信来共享内存。）
  - **这里要注意的一个细节是，元素值从外界进入通道时会被复制。更具体地说，进入通道的并不是在接收操作符右边的那个元素值，而是它的副本。**
  - 单向通道最主要的用途就是约束其他代码的行为。
    - 比如函数getIntChan会返回一个<-chan int类型的通道，这就意味着得到该通道的程序，只能从通道中接收元素值。这实际上就是对函数调用方的一种约束了。
  - `select`语句与通道怎样联用
    - `select`语句只能与通道联用，它一般由若干个分支组成。每次执行这种语句的时候，一般只有一个分支中的代码会被运行。
    - `select`语句只能对其中的每一个`case`表达式各求值一次。所以，如果我们想连续或定时地操作其中的通道的话，就往往需要通过在`for`语句中嵌入`select`语句的方式实现。但这时要注意，**简单地在`select`语句的分支中使用`break`语句，只能结束当前的`select`语句的执行，而并不会对外层的`for`语句产生作用**。

- 锁

  - **使用 chan 也可以实现互斥锁**。在 chan 的内部实现中，就有一把互斥锁保护着它的所有字段。从外在表现上，chan 的发送和接收之间也存在着 happens-before 的关系，保证元素放进去之后，receiver 才能读取到
  - 要想使用 chan 实现互斥锁，至少有两种方式。**一种方式是先初始化一个 capacity 等于 1 的 Channel，然后再放入一个元素。这个元素就代表锁，谁取得了这个元素，就相当于获取了这把锁。**另一种方式是，先初始化一个 capacity 等于 1 的 Channel，它的“空槽”代表锁，谁能成功地把元素发送到这个 Channel，谁就获取了这把锁。

- 结构体及其方法

  - 字面量`struct{}`代表了什么？又有什么用处？
    - 空结构体不占用内存空间，但是具有结构体的一切属性，如可以拥有方法，可以写入channel。所以当我们需要使用结构体而又不需要具体属性时可以使用它。
  - Go 语言中根本没有继承的概念，它所做的是通过嵌入字段的方式实现了类型之间的**组合**	
    - 这时候，**被嵌入类型也就自然而然地实现了嵌入字段所实现的接口。**再者，组合要比继承**更加简洁和清晰**
  - 值方法和指针方法
    - **值方法的接收者**是该方法所属的那个类型值的一个**副本**。我们在该方法内对该**副本的修改一般都不会体现在原值上**，除非这个类型本身是某个引用类型（比如切片或字典）的别名类型。
    - 而**指针方法的接收者**，是该方法所属的那个基本类型值的指针值的一个副本。我们在这样的方法内**对该副本指向的值进行修改，却一定会体现在原值上**。

- 当我们为一个接口变量赋值时会发生什么

  - 如果我们使用一个变量给另外一个变量赋值，那么真正赋给后者的，并不是前者持有的那个值，而是该值的一个**副本**

  - ```go
    // 运行后发现输出不仅d的name字段变为了“big dog”，同样pet接口变量也变成了“big dog”。
    // 传递给pet变量的同样是&d的一个指针副本，因为传递的是副本，所以无论是指针还是值，都可以说是浅复制；且由于传递的是指针（虽然是副本），但还是会对指向的底层变量做修改。
    d := Dog{name: "little dog"}
    var pet Pet = &d
    d.SetName("big dog")
    ```

- 指针

  - Go 语言中的哪些值是不可寻址的
    - **不可变的**值不可寻址。常量、基本类型的值字面量、字符串变量的值、函数以及方法的字面量都是如此。其实这样规定也有安全性方面的考虑。
    - 绝大多数被视为**临时结果**的值都是不可寻址的。算术操作的结果值属于临时结果，针对值字面量的表达式结果值也属于临时结果。但有一个例外，对切片字面量的索引结果值虽然也属于临时结果，但却是可寻址的。
    - 若拿到某值的指针可能会破坏程序的一致性，那么就是**不安全的**，该值就不可寻址。由于字典的内部机制，对字典的索引结果值的取址操作都是不安全的。另外，获取由字面量或标识符代表的函数或方法的地址显然也是不安全的。
    - 如果我们把临时结果赋给一个变量，那么它就是可寻址的了。如此一来，取得的指针指向的就是这个变量持有的那个值了。

- panic、recover以及defer

  - Go 语言的内建函数`recover`专用于恢复 panic，或者说平息运行时恐慌。**`recover`函数无需任何参数，并且会返回一个空接口类型的值。**

  - panic 一旦发生，控制权就会讯速地沿着调用栈的反方向传播。所以，**在`panic`函数调用之后的代码，根本就没有执行的机会。**

  - `defer`语句就是被用来延迟执行代码的。**注意，被延迟执行的是`defer`函数，而不是`defer`语句。**延迟到什么时候呢？这要**延迟到该语句所在的函数即将执行结束的那一刻，无论结束执行的原因是什么。**

    - ```go
       
      func main() {
       fmt.Println("Enter function main.")
       defer func(){
        fmt.Println("Enter defer function.")
        if p := recover(); p != nil {
         fmt.Printf("panic: %s\n", p)
        }
        fmt.Println("Exit defer function.")
       }()
       // 引发 panic。
       panic(errors.New("something wrong"))
       fmt.Println("Exit function main.")
      }
      ```

  - defer是用来做函数级别的资源清理工作的，处理panic是兼职。

- 测试的基本规则和流程

  - 我们可以为 Go 程序编写三类测试，即：功能测试（test）、基准测试（benchmark，也称性能测试），以及示例测试（example）。
  - Go 语言对测试函数的名称和签名都有哪些规定

    - **对于功能测试函数来说，其名称必须以`Test`为前缀**，并且参数列表中只应有一个`*testing.T`类型的参数声明。
    - **对于性能测试函数来说，其名称必须以`Benchmark`为前缀**，并且唯一参数的类型必须是`*testing.B`类型的。
    - **对于示例测试函数来说，其名称必须以`Example`为前缀**，但对函数的参数列表没有强制规定。
  - 怎样解释性能测试的测试结果

    - 我在运行`go test`命令的时候加了两个标记。**第一个标记及其值为`-bench=.`，只有有了这个标记，命令才会进行性能测试。**该标记的值`.`表明需要执行任意名称的性能测试函数
    - 第二个标记及其值是`-run=^$`，这个标记用于表明需要执行哪些功能测试函数，这同样也是以函数名称为依据的。**该标记的值`^$`意味着：只执行名称为空的功能测试函数，换句话说，不执行任何功能测试函数。**
    - 你可能已经看出来了，这两个标记的值都是**正则表达式**。实际上，它们只能以正则表达式为值。此外，如果运行`go test`命令的时候不加`-run`标记，那么就会使它执行被测代码包中的所有功能测试函数。
  - **在对 Go 程序执行某种自动化测试的过程中，测试日志会显得特别多，而且好多都是重复的。**

    - **比如，对于功能测试函数来说，我们通常没有必要重复执行它**
    - **正好相反，对于性能测试函数来说，我们常常需要反复地执行**，并以此试图抹平当时的计算资源调度的细微差别对被测程序性能的影响。通过`-cpu`标记，我们还能够模拟被测程序在计算能力不同计算机中的性能表现。





# postman

- 四种常见接口请求

- Collection用例管理及批量执行

  - 场景
    - 用例分类管理，方便后期维护
    - 可以进行批量用例回归测试 。

- 全局变量/环境变量/集合变量

  - 想要使用变量中的值只需俩个步骤，分别是定义变量和获取变量 。

    1. 定义变量（设置变量）
    2. 获取变量（访问变量）

  - 如果在**请求参数中**获取变量，无论是获取全局变量，还是环境变量，还是集合变量，获取的方式都是一样的编写规则：{{变量名}} 。

    - 请求参数指的是：URL，Params , Authorization , Headers , Body

    如果是在编写代码的位置(Tests,Pre-requests Script)获取变量，获取不同类型的变量，编写的代码都不相同，具体如下：

    - 获取环境变量：pm.environment.get(‘变量名’)
    - 获取全局变量：pm.globals.get('变量名')
    - 获取集合变量：pm.pm.collectionVariables.get.get('变量名')

- postman断言

  - 一些特点 ，具体如下

    - 断言编写位置：Tests标签
    - 断言所用语言：JavaScript
    - 断言执行顺序：在响应体数据返回后执行 。
    - 断言执行结果查看：Test Results

  - 断言响应体(重点)

    - ```javascript
      
      
      // 断言响应体中包含XXX字符串：Response body:Contains string
      pm.test("Body matches string", function () {
      	//获取响应文本中包含string
      	pm.expect(pm.response.text()).to.include("string")  
      });     
          
      // 断言响应体等于XXX字符串：Response body : is equal to a string
      pm.test("Body is correct", function () {
        // 获取响应体等于response_body_string
      	pm.response.to.have.body("response_body_string");   
      });
      
      // 断言响应体(json)中某个键名对应的值：Response body : JSON value check
      pm.test("Your test name", function () {
        // 获取响应体，以json显示，赋值给jsonData .注意：该响应体必须返会是的json，否则会报错
      	var jsonData = pm.response.json()   
        // 获取jsonData中键名为value的值，然后和100进行比较
      	pm.expect(jsonData.value).to.eql(100)  
      });
      ```

  - 对单个请求断言

    - 以下案例是我们最常见的在单个请求中编写的断言，具体如下 。

    ```json
    {
        "cityid": "101120101",
        "city": "济南",
        "update_time": "2020-04-17 10:50",
        "wea": "晴",
        "wea_img": "qing",
        "tem": "16",
        "tem_day": "20",
        "tem_night": "9",
        "win": "东北风",
        "win_speed": "3级",
        "win_meter": "小于12km/h",
        "air": "113"
    }
    ```

    - 断言响应状态码为200
    - 断言city等于济南
    - 断言update_time包含2020-04-17

    ![postman教程-05-强大的断言功能](https://p3-tt.byteimg.com/origin/pgc-image/4cd86496aac74d488723740413378dbb?from=pc)

  - 为集合断言

    - 编写通用断言的位置是在集合或集合的文件夹中 。具体位置如下图：

    ![postman教程-05-强大的断言功能](https://p6-tt.byteimg.com/origin/pgc-image/8ef3afa7f516417f9cd4f41224d6a0e4?from=pc)

    - 对项目中每个接口返回的响应状态码进行断言，同时对用户管理模块下每个接口的code进行断言。

    ![postman教程-05-强大的断言功能](https://p1-tt.byteimg.com/origin/pgc-image/201dfe6f43574d7598dd10cd9cf392f0?from=pc)

    ![postman教程-05-强大的断言功能](https://p1-tt.byteimg.com/origin/pgc-image/bf8ff99d18b74875912a676124aaca4b?from=pc)

    ![postman教程-05-强大的断言功能](https://p6-tt.byteimg.com/origin/pgc-image/8edd2310d72e43599a272a346b2f34d8?from=pc)

- 接口关联

  - 思路
    - 提取上一个接口的返回数据值
    - 将这个数据值保存到环境变量或全局变量中
    - 在下一个接口获取环境变量或全局变量
  - 步骤
    - 获取上传头像接口返回url的值
    - 将这个值保存成全局变量(环境变量也可以)
    - 在图像预览使用全局变量

  ![postman教程-06-如何解决接口关联](https://p3-tt.byteimg.com/origin/pgc-image/ca6654bf983243b1aec7c9862644f538?from=pc)

  ![postman教程-06-如何解决接口关联](https://p3-tt.byteimg.com/origin/pgc-image/06b60f5544d04a0eb4aa870e2b05d4a3?from=pc)

- 请求前置脚本

  - 案例

    - 请求的登录接口URL，参数t的值要求的规则是每次请求都必须是一个随机数。
    - 接口地址：http://localhost/index.php?m=Home&c=User&a=do_login&t=0.7102045930338428

  - 步骤：

    1. 在前置脚本中编写生成随机数
    2. 将这个值保存成环境变量
    3. 将参数t的值替换成环境变量的值 。

    ![postman教程-07-请求前置脚本](https://p6-tt.byteimg.com/origin/pgc-image/91b7edd83030401b918d10027488ba3d?from=pc)

- 认证

  - 比如登录后返回的token要在每个请求接口的headers中传入 。这时就需要在每个headers中都填写一个认证参数传入 

  - inherit auto from parent——从父级继承身份验证，是每个请求的默认选择 。

  - 实现步骤：

    1. 选中一个集合进行编辑，切换到Pre-Request Script.在这里请求登录接口 ，将返回的token值拿到，然后保存成全局变量 。

    2. 切换到Authorization选项卡，在这里直接获取token 。这里的获取token需要根据具体的项目 。比如我们所测试的项目正好是Bearer token这种形式 。直接在列表中使用这种方式输入{{token}}即可。

    3. 向集合添加请求，无需进行token处理，所有接口都能请求成功 。

       ![postman教程-12-pm对象之sendRequest解析](https://p1-tt.byteimg.com/origin/pgc-image/084bb4b0df4048da859ecac7011cd715?from=pc)

    ![postman教程-08-认证(Authorization)](https://p3-tt.byteimg.com/origin/pgc-image/c316d825343140c3913c87977555f261?from=pc)

    

    ![postman教程-08-认证(Authorization)](https://p1-tt.byteimg.com/origin/pgc-image/cdd3a4fa713248c18e53a630535badb1?from=pc)

    

    ![postman教程-08-认证(Authorization)](https://p6-tt.byteimg.com/origin/pgc-image/124b097b56454d5795cc44022e31c43a?from=pc)

    

  - No Auth

    无需身份认证的可以选择这个 。

  - API Key

    也有很多系统是通过这种认证方式，更多的是系统自定义的认证方式，比如在请求头添加 model: data xxx-xxx-xxx-xxxx

    ![postman教程-08-认证(Authorization)](https://p6-tt.byteimg.com/origin/pgc-image/25d2a9bc7c7f4f16834a65083558ad3d?from=pc)

  - Bearer Token

    通过Bearer Token认证，就相当于在请求头中添加Authorization：Bearer Token 

    ![postman教程-08-认证(Authorization)](https://p6-tt.byteimg.com/origin/pgc-image/e7d3fad7ec93462982a554b78c73a7c1?from=pc)

    

- 导入/导出
  
- 
  
- pm对象

  - sendRequest

    - https://www.toutiao.com/i6821325438598513164/

  - 常用变量方法

    - https://www.toutiao.com/i6821334573809402375/

  - 请求响应断言方法

    - https://www.toutiao.com/i6821337199896691207/

    



# debug

- ## 、基本用法&快捷键

  Debug调试的功能主要对应着图一中4和5两组按钮：

  　　1、首先说第一组按钮，共8个按钮，从左到右依次如下：

  　　　　![img](https://images2017.cnblogs.com/blog/856154/201709/856154-20170905134837851-1615718043.png) [图2.1]

  　　　　> Show Execution Point (Alt + F10)：如果你的光标在其它行或其它页面，点击这个按钮可跳转到当前代码执行的行。

  　　　　> Step Over (F8)：步过，一行一行地往下走，如果这一行上有方法不会进入方法。

  　　　　> Step Into (F7)：步入，如果当前行有方法，可以进入方法内部，一般用于进入自定义方法内，不会进入官方类库的方法

  　　　　> Force Step Into (Alt + Shift + F7)：强制步入，能进入任何方法，查看底层源码的时候可以用这个进入官方类库的方法。

  　　　　> Step Out (Shift + F8)：步出，从步入的方法内退出到方法调用处，此时方法已执行完毕，只是还没有完成赋值。

  　　　　> Drop Frame (默认无)：回退断点，后面章节详细说明。

  在调试的时候，想要重新走一下流程而不用再次发起一个请求？

  　　1、首先认识下这个方法调用栈，如图8.1，首先请求进入DemoController的insertDemo方法，然后调用insert方法，其它的invoke我们且先不管，最上面的方法是当前断点所在的方法。

  　　[图8.1]

  　　![img](https://images2017.cnblogs.com/blog/856154/201709/856154-20170905210917741-1095775464.png) 

  　　2、断点回退

  　　所谓的断点回退，其实就是回退到上一个方法调用的开始处，在IDEA里测试无法一行一行地回退或回到到上一个断点处，而是回到上一个方法。

  　　回退的方式有两种，一种是Drop Frame按钮(图8.2)，按调用的方法逐步回退，包括三方类库的其它方法(取消Show All Frames按钮会显示三方类库的方法，如图8.3)。

  　　第二种方式，在调用栈方法上选择要回退的方法，右键选择Drop Frame(图8.4)，回退到该方法的上一个方法调用处，此时再按F9(Resume Program)，可以看到程序进入到该方法的断点处了。

  　　但有一点需要注意，断点回退只能重新走一下流程，之前的某些参数/数据的状态已经改变了的是无法回退到之前的状态的，如对象、集合、更新了数据库数据等等。

  　　图[8.2]

  　　![img](https://images2017.cnblogs.com/blog/856154/201709/856154-20170905211428554-1617570377.png)

  　　图[8.3]

  　　![img](https://images2017.cnblogs.com/blog/856154/201709/856154-20170905211723304-1223322879.png)

  　　图[8.4]

  　　![img](https://images2017.cnblogs.com/blog/856154/201709/856154-20170905212138101-113776159.png)

   

  　　　　> Run to Cursor (Alt + F9)：运行到光标处，你可以将光标定位到你需要查看的那一行，然后使用这个功能，代码会运行至光标行，而不需要打断点。

  　　　　> Evaluate Expression (Alt + F8)：计算表达式





# 库

## viper

- https://juejin.cn/post/6844904051369312264#heading-10
- viper 是一个配置解决方案，拥有丰富的特性：
  - 支持 JSON/TOML/YAML/HCL/envfile/Java properties 等多种格式的配置文件；
  - 可以设置监听配置文件的修改，修改时自动加载新的配置；
  - 从环境变量、命令行选项和`io.Reader`中读取配置；
  - 从远程配置系统中读取和监听修改，如 etcd/Consul；
  - 代码逻辑中显示设置键值。
- viper 的使用非常简单，它需要很少的设置。
  - 设置文件名（`SetConfigName`）
  - 配置类型（`SetConfigType`）
  - 搜索路径（`AddConfigPath`），然后调用`ReadInConfig`。 viper会自动根据类型来读取配置。使用时调用`viper.Get`方法获取键值。
  - `GetType`系列方法可以返回指定类型的值。 其中，Type 可以为`Bool/Float64/Int/String/Time/Duration/IntSlice/StringSlice`。 但是请注意，**如果指定的键不存在或类型不正确，`GetType`方法返回对应类型的零值**。
  - 如果要判断某个键是否存在，使用`IsSet`方法。 另外，`GetStringMap`和`GetStringMapString`直接以 map 返回某个键下面所有的键值对。
  - 如果某个键通过`viper.Set`设置了值，那么这个值的优先级最高。
  - 只需要调用`viper.WatchConfig`，viper 会自动监听配置修改。如果有修改，重新加载的配置。



## Zap

- [Zap](https://github.com/uber-go/zap)是非常快的、结构化的，分日志级别的Go日志库。

  - 它同时提供了结构化日志记录和printf风格的日志记录
  - 它非常的快

- 通过调用zap.NewProduction()/zap.NewDevelopment()或者zap.Example()创建一个Logger。

  - 上面的每一个函数都将创建一个logger。唯一的区别在于它将记录的信息不同。例如production logger默认记录调用函数信息、日期和时间等。
  - 通过Logger调用Info/Error等。
  - 默认情况下日志都会打印到应用程序的console界面。

- 现在让我们使用Sugared Logger来实现相同的功能。

  - 大部分的实现基本都相同。
  - 惟一的区别是，我们通过调用主logger的. Sugar()方法来获取一个SugaredLogger。
  - 然后使用SugaredLogger以printf格式记录语句

- 定制logger

  - 将日志写入文件而不是终端

    - 我们要做的第一个更改是把日志写入文件，而不是打印到应用程序控制台。

    ```
        func New(core zapcore.Core, options ...Option) *Logger
    ```

    - zapcore.Core需要三个配置——Encoder，WriteSyncer，LogLevel。

      Encoder:编码器(如何写入日志)。我们将使用开箱即用的NewJSONEncoder()，并使用预先设置的ProductionEncoderConfig()。

    ```
       zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig())
    ```

    ​		WriterSyncer ：指定日志将写到哪里去。我们使用zapcore.AddSync()函数并且将打开的文件句柄传进去。

    ```
       file, _ := os.Create("./test.log")
       writeSyncer := zapcore.AddSync(file)
    ```

    ​		Log Level：哪种级别的日志将被写入。

  -  将JSON Encoder更改为普通的Log Encoder
    
    - 现在，我们希望将编码器从JSON Encoder更改为普通Encoder。为此，我们需要将NewJSONEncoder()更改为NewConsoleEncoder()。



## gorm

- 对开发人员友好的 Golang ORM 库

  > 全特性 ORM (几乎包含所有特性)
  > 模型关联 (一对一， 一对多，一对多（反向）， 多对多， 多态关联)
  > 钩子 (Before/After Create/Save/Update/Delete/Find)
  > 预加载
  > 事务
  > 复合主键
  > SQL 构造器
  > 自动迁移
  > 日志
  > 基于 GORM 回调编写可扩展插件
  > 全特性测试覆盖
  > 开发者友好

- CRUD

  - ```go
    type Product struct {
      gorm.Model
      Code string
      Price uint
    }
    ```





## Gin

- Gin 是一个用 Go (Golang) 编写的 HTTP web 框架。

- 特性

  - 快速
  - 支持中间件
  - Crash 处理
  - JSON 验证
  - 路由组
  - 错误管理
  - 内置渲染

- 示例

  - HTML 渲染

  - HTTP2 server 推送

    - ```go
          r.GET("/", func(c *gin.Context) {
              if pusher := c.Writer.Pusher(); pusher != nil {
                  // 使用 pusher.Push() 做服务器推送
                  if err := pusher.Push("/assets/app.js", nil); err != nil {
                      log.Printf("Failed to push: %v", err)
                  }
              }
              c.HTML(200, "https", gin.H{
                  "status": "success",
              })
          })
      
      ```

      

  - JSONP

    - 使用 JSONP 向不同域的服务器请求数据。如果查询参数存在回调，则将回调添加到响应体中。

  - Multipart/Urlencoded 绑定

    - c.ShouldBindWith(&form,)

  - Multipart/Urlencoded 表单

    - message := c.PostForm("message")        
    - nick := c.DefaultPostForm("nick", "anonymous") 可设置默认值

  - SecureJSON

    - 使用 SecureJSON 防止 json 劫持。如果给定的结构是数组值，则默认预置 `"while(1),"` 到响应体。

  - XML/JSON/YAML/ProtoBuf 渲染

  - 文件上传

    - ```go
          router.POST("/upload", func(c *gin.Context) {
              form, _ := c.MultipartForm()
            	// 单文件
            	file, _ := c.FormFile("file")
            	// 多文件
              files := form.File["upload[]"]
      
              for _, file := range files {
                  log.Println(file.Filename)
      
                  // 上传文件至指定目录
                  // c.SaveUploadedFile(file, dst)
              }
      ```

  - 优雅地重启或停止

    - 我们可以使用 [fvbock/endless](https://github.com/fvbock/endless) 来替换默认的 `ListenAndServe`。

    - 如果你使用的是 Go 1.8，可以不需要这些库！考虑使用 http.Server 内置的 [Shutdown()](https://golang.org/pkg/net/http/#Server.Shutdown) 方法优雅地关机。

    - ```go
      func main() {
          router := gin.Default()
          router.GET("/", func(c *gin.Context) {
              time.Sleep(5 * time.Second)
              c.String(http.StatusOK, "Welcome Gin Server")
          })
      
          srv := &http.Server{
              Addr:    ":8080",
              Handler: router,
          }
      
          go func() {
              // 服务连接
              if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
                  log.Fatalf("listen: %s\n", err)
              }
          }()
      
          // 等待中断信号以优雅地关闭服务器（设置 5 秒的超时时间）
          quit := make(chan os.Signal)
          signal.Notify(quit, os.Interrupt)
          <-quit
          log.Println("Shutdown Server ...")
      
          ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
          defer cancel()
          if err := srv.Shutdown(ctx); err != nil {
              log.Fatal("Server Shutdown:", err)
          }
          log.Println("Server exiting")
      }
      ```

  - 自定义中间件

    - ```go
      func Logger() gin.HandlerFunc {
          return func(c *gin.Context) {
              t := time.Now()
      
              // 设置 example 变量
              c.Set("example", "12345")
      
              // 请求前
      
              c.Next()
      
              // 请求后
              latency := time.Since(t)
              log.Print(latency)
      
              // 获取发送的 status
              status := c.Writer.Status()
              log.Println(status)
          }
      }
      ```

  - 自定义验证器

    - 当字段级别的验证没有意义时，也可以在结构级别注册验证。

    - ```go
      
      // Booking 包含绑定和验证的数据。
      type Booking struct {
          CheckIn  time.Time `form:"check_in" binding:"required,bookabledate" time_format:"2006-01-02"`
          CheckOut time.Time `form:"check_out" binding:"required,gtfield=CheckIn" time_format:"2006-01-02"`
      }
      
      func bookableDate(
          v *validator.Validate, topStruct reflect.Value, currentStructOrField reflect.Value,
          field reflect.Value, fieldType reflect.Type, fieldKind reflect.Kind, param string,
      ) bool {
          if date, ok := field.Interface().(time.Time); ok {
              today := time.Now()
              if today.Year() > date.Year() || today.YearDay() > date.YearDay() {
                  return false
              }
          }
          return true
      }
      
      func main() {
          route := gin.Default()
      
          if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
              v.RegisterValidation("bookabledate", bookableDate)
          }
      
          route.GET("/bookable", getBookable)
          route.Run(":8085")
      }
      
      func getBookable(c *gin.Context) {
          var b Booking
          if err := c.ShouldBindWith(&b, binding.Query); err == nil {
              c.JSON(http.StatusOK, gin.H{"message": "Booking dates are valid!"})
          } else {
              c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
          }
      }
      ```

  - 设置和获取 Cookie

    - ```go
      cookie, err := c.Cookie("gin_cookie")
      c.SetCookie("gin_cookie", "test", 3600, "/", "localhost", false, true)
      ```

  - 路由

    - 嵌套路由

      - ```go
        
        func main() {
            r := gin.Default()　　　　//设置路由前缀 调用Group方法
            userGroup := r.Group("/user")
            {
                userGroup.GET("/index", func(c *gin.Context) {
                    c.JSON(http.StatusOK, gin.H{
                        "message": "user/index",
                    })
                })
                shopGroup := userGroup.Group("/shop")
                {
                    shopGroup.GET("/index", func(c *gin.Context) {
                        c.JSON(http.StatusOK, gin.H{
                            "message": "/user/shop/index",
                        })
                    })
                }
            }
            r.Run()
        }
        ```

    - 路由参数

      - ```go
        func main() {
            router := gin.Default()
        
            // 此 handler 将匹配 /user/john 但不会匹配 /user/ 或者 /user
            router.GET("/user/:name", func(c *gin.Context) {
                name := c.Param("name")
                c.String(http.StatusOK, "Hello %s", name)
            })
        
            // 此 handler 将匹配 /user/john/ 和 /user/john/send
            // 如果没有其他路由匹配 /user/john，它将重定向到 /user/john/
            router.GET("/user/:name/*action", func(c *gin.Context) {
                name := c.Param("name")
                action := c.Param("action")
                message := name + " is " + action
                c.String(http.StatusOK, message)
            })
        
            router.Run(":8080")
        }
        ```

    - 路由组

  - 重定向

    - ```go
      r.GET("/test", func(c *gin.Context) {
          c.Request.URL.Path = "/test2"
          r.HandleContext(c)
      })
      r.GET("/test2", func(c *gin.Context) {
          c.JSON(200, gin.H{"hello": "world"})
      })
      
      ---
      
      r.GET("/test", func(c *gin.Context) {
          c.Redirect(http.StatusMovedPermanently, "http://www.google.com/")
      })
      ```

  - 只绑定 url 查询字符串

    - c.ShouldBindQuery(&person)

  - 在中间件中使用 Goroutine

    - 当在中间件或 handler 中启动新的 Goroutine 时，**不能**使用原始的上下文，必须使用只读副本。

    - ```go
          r.GET("/long_async", func(c *gin.Context) {
              // 创建在 goroutine 中使用的副本
              cCp := c.Copy()
              go func() {
                  // 用 time.Sleep() 模拟一个长任务。
                  time.Sleep(5 * time.Second)
                  // 请注意您使用的是复制的上下文 "cCp"，这一点很重要
                  log.Println("Done! in path " + cCp.Request.URL.Path)
              }()
          })
      ```

  - 多模板

    - [multitemplate](https://github.com/gin-contrib/multitemplate)

    - ```go
      func createMyRender() multitemplate.Renderer {
      	r := multitemplate.NewRenderer()
      	r.AddFromFiles("index", "templates/base.html", "templates/index.html")
      	r.AddFromFiles("article", "templates/base.html", "templates/index.html", "templates/article.html")
      	return r
      }
      
      func main() {
      	router := gin.Default()
      	router.HTMLRender = createMyRender()
      	router.GET("/", func(c *gin.Context) {
      		c.HTML(200, "index", gin.H{
      			"title": "Html5 Template Engine",
      		})
      	})
      	router.GET("/article", func(c *gin.Context) {
      		c.HTML(200, "article", gin.H{
      			"title": "Html5 Article Engine",
      		})
      	})
      	router.Run(":8080")
      }
      
      ---
      
      
      func main() {
      	router := gin.Default()
      	router.HTMLRender = loadTemplates("./templates")
      	router.GET("/", func(c *gin.Context) {
      		c.HTML(200, "index.html", gin.H{
      			"title": "Welcome!",
      		})
      	})
      	router.GET("/article", func(c *gin.Context) {
      		c.HTML(200, "article.html", gin.H{
      			"title": "Html5 Article Engine",
      		})
      	})
      
      	router.Run(":8080")
      }
      
      func loadTemplates(templatesDir string) multitemplate.Renderer {
      	r := multitemplate.NewRenderer()
      
      	layouts, err := filepath.Glob(templatesDir + "/layouts/*.html")
      	if err != nil {
      		panic(err.Error())
      	}
      
      	includes, err := filepath.Glob(templatesDir + "/includes/*.html")
      	if err != nil {
      		panic(err.Error())
      	}
      
      	// Generate our templates map from our layouts/ and includes/ directories
      	for _, include := range includes {
      		layoutCopy := make([]string, len(layouts))
      		copy(layoutCopy, layouts)
      		files := append(layoutCopy, include)
      		r.AddFromFiles(filepath.Base(include), files...)
      	}
      	return r
      }
      ```

  - 日志

    - ```go
      f, _ := os.Create("gin.log")
      gin.DefaultWriter = io.MultiWriter(f)
      
      // 如果需要同时将日志写入文件和控制台，请使用以下代码。
      gin.DefaultWriter = io.MultiWriter(f, os.Stdout)
      
      ```

  - 定义路由日志的格式

    - 如果你想要以指定的格式（例如 JSON，key values 或其他格式）记录信息，则可以使用 gin.DebugPrintRouteFunc 指定格式。 在下面的示例中，我们使用标准日志包记录所有路由，但你可以使用其他满足你需求的日志工具。

    - ```go
          gin.DebugPrintRouteFunc = func(httpMethod, absolutePath, handlerName string, nuHandlers int) {
      	log.Printf("endpoint %v %v %v %v\n", httpMethod, absolutePath, handlerName, nuHandlers)
          }
      
      ```

  - 将 request body 绑定到不同的结构体中

    - 一般通过调用  `c.Request.Body` 方法绑定数据，但不能多次调用这个方法。
      - c.ShouldBind 使用了 c.Request.Body，不可重用。
    - 要想多次绑定，可以使用 `c.ShouldBindBodyWit`
      - c.ShouldBindBodyWith(&objA, binding.JSON);

  - 映射查询字符串或表单参数

    - ```go
      POST /post?ids[a]=1234&ids[b]=hello HTTP/1.1
      Content-Type: application/x-www-form-urlencoded
      
      names[first]=thinkerou&names[second]=tianou
      
      ---
      
      func main() {
          router := gin.Default()
      
          router.POST("/post", func(c *gin.Context) {
      
              ids := c.QueryMap("ids")
              names := c.PostFormMap("names")
      
              fmt.Printf("ids: %v; names: %v", ids, names)
          })
          router.Run(":8080")
      }
      
      ```

  - 查询字符串参数

    - ```go
      // 示例 URL： /welcome?firstname=Jane&lastname=Doe
      
      firstname := c.DefaultQuery("firstname", "Guest")
      lastname := c.Query("lastname") 
      // c.Request.URL.Query().Get("lastname") 的一种快捷方式
      
      ```

  - 模型绑定和验证

    - ```go
      // 绑定 JSON ({"user": "manu", "password": "123"})
      c.ShouldBindJSON(&json);
      
      c.ShouldBindXML(&xml);
      
      // 绑定 HTML 表单 (user=manu&password=123)
      c.ShouldBind(&form);
      ```

  - 绑定 Url

    - ```go
      type Person struct {
          ID string `uri:"id" binding:"required,uuid"`
          Name string `uri:"name" binding:"required"`
      }
      
      func main() {
          route := gin.Default()
          route.GET("/:name/:id", func(c *gin.Context) {
              var person Person
              if err := c.ShouldBindUri(&person); err != nil {
                  c.JSON(400, gin.H{"msg": err})
                  return
              }
              c.JSON(200, gin.H{"name": person.Name, "uuid": person.ID})
          })
          route.Run(":8088")
      }
      ```

  - 绑定表单数据至自定义结构体

    - ```go
          var b StructB
          c.Bind(&b)
      ```

  - 自定义 HTTP 配置

    - ```go
      func main() {
          router := gin.Default()
          http.ListenAndServe(":8080", router)
      }
      
      ---
      
      func main() {
          router := gin.Default()
      
          s := &http.Server{
              Addr:           ":8080",
              Handler:        router,
              ReadTimeout:    10 * time.Second,
              WriteTimeout:   10 * time.Second,
              MaxHeaderBytes: 1 << 20,
          }
          s.ListenAndServe()
      }
      
      ```

  - 运行多个服务

    - ```go
      package main
      
      import (
          "log"
          "net/http"
          "time"
      
          "github.com/gin-gonic/gin"
          "golang.org/x/sync/errgroup"
      )
      
      var (
          g errgroup.Group
      )
      
      func router01() http.Handler {
          e := gin.New()
          e.Use(gin.Recovery())
          e.GET("/", func(c *gin.Context) {
              c.JSON(
                  http.StatusOK,
                  gin.H{
                      "code":  http.StatusOK,
                      "error": "Welcome server 01",
                  },
              )
          })
      
          return e
      }
      
      func router02() http.Handler {
          e := gin.New()
          e.Use(gin.Recovery())
          e.GET("/", func(c *gin.Context) {
              c.JSON(
                  http.StatusOK,
                  gin.H{
                      "code":  http.StatusOK,
                      "error": "Welcome server 02",
                  },
              )
          })
      
          return e
      }
      
      func main() {
          server01 := &http.Server{
              Addr:         ":8080",
              Handler:      router01(),
              ReadTimeout:  5 * time.Second,
              WriteTimeout: 10 * time.Second,
          }
      
          server02 := &http.Server{
              Addr:         ":8081",
              Handler:      router02(),
              ReadTimeout:  5 * time.Second,
              WriteTimeout: 10 * time.Second,
          }
      
          g.Go(func() error {
              return server01.ListenAndServe()
          })
      
          g.Go(func() error {
              return server02.ListenAndServe()
          })
      
          if err := g.Wait(); err != nil {
              log.Fatal(err)
          }
      }
      ```




## mongo

| MongoDB术语/概念 |                说明                 | 对比SQL术语/概念 |
| :--------------: | :---------------------------------: | :--------------: |
|     database     |               数据库                |     database     |
|    collection    |                集合                 |      table       |
|     document     |                文档                 |       row        |
|      field       |                字段                 |      column      |
|      index       |                index                |       索引       |
|   primary key    | 主键 MongoDB自动将_id字段设置为主键 |   primary key    |



- https://www.liwenzhou.com/posts/Go/go_mongodb/#autoid-0-3-5



## go-redis

- https://www.liwenzhou.com/posts/Go/go_redis/#autoid-2-1-0





## wrk

- HTTP服务压力测试工具
- https://www.liwenzhou.com/posts/Go/benchmark_tool/





## net/http

- Go语言内置的`net/http`包提供了HTTP客户端和服务端的实现。

- 客户端

  - 关于GET请求的参数需要使用Go语言内置的`net/url`这个标准库来处理。

    - ```go
      	apiUrl := "http://127.0.0.1:9090/get"
      	// URL param
      	data := url.Values{}
      	data.Set("name", "小王子")
      	data.Set("age", "18")
      	u, err := url.ParseRequestURI(apiUrl)
      	if err != nil {
      		fmt.Printf("parse url requestUrl failed, err:%v\n", err)
      	}
      	u.RawQuery = data.Encode() // URL encode
      	fmt.Println(u.String())
      	resp, err := http.Get(u.String())
      
      
      ---
      
      // 对应的Server端HandlerFunc如下：
      func getHandler(w http.ResponseWriter, r *http.Request) {
      	defer r.Body.Close()
      	data := r.URL.Query()
      	fmt.Println(data.Get("name"))
      	fmt.Println(data.Get("age"))
      	answer := `{"status": "ok"}`
      	w.Write([]byte(answer))
      }
      ```

  - post

    - ```go
      
      	url := "http://127.0.0.1:9090/post"
      	// 表单数据
      	//contentType := "application/x-www-form-urlencoded"
      	//data := "name=小王子&age=18"
      	// json
      	contentType := "application/json"
      	data := `{"name":"小王子","age":18}`
      	resp, err := http.Post(url, contentType, strings.NewReader(data))
      
      ---
      
      //对应的Server端HandlerFunc如下：
      func postHandler(w http.ResponseWriter, r *http.Request) {
      	defer r.Body.Close()
      	// 1. 请求类型是application/x-www-form-urlencoded时解析form数据
      	r.ParseForm()
      	fmt.Println(r.PostForm) // 打印form数据
      	fmt.Println(r.PostForm.Get("name"), r.PostForm.Get("age"))
      	// 2. 请求类型是application/json时从r.Body读取数据
      	b, err := ioutil.ReadAll(r.Body)
      	if err != nil {
      		fmt.Printf("read request.Body failed, err:%v\n", err)
      		return
      	}
      	fmt.Println(string(b))
      	answer := `{"status": "ok"}`
      	w.Write([]byte(answer))
      }
      ```

  - 自定义Client

    - ```go
      // 要管理HTTP客户端的头域、重定向策略和其他设置，创建一个Client
      client := &http.Client{
      	CheckRedirect: redirectPolicyFunc,
      }
      resp, err := client.Get("http://example.com")
      // ...
      req, err := http.NewRequest("GET", "http://example.com", nil)
      // ...
      req.Header.Add("If-None-Match", `W/"wyzzy"`)
      resp, err := client.Do(req)
      // ...
      ```

  - 自定义Transport

    - ```go
      // 要管理代理、TLS配置、keep-alive、压缩和其他设置，创建一个Transport：
      // Client和Transport类型都可以安全的被多个goroutine同时使用。出于效率考虑，应该一次建立、尽量重用。
      tr := &http.Transport{
      	TLSClientConfig:    &tls.Config{RootCAs: pool},
      	DisableCompression: true,
      }
      client := &http.Client{Transport: tr}
      resp, err := client.Get("https://example.com")
      
      ```

- 服务端

  - 默认的Server

    - ```go
      // ListenAndServe使用指定的监听地址和处理器启动一个HTTP服务端。处理器参数通常是nil，这表示采用包变量DefaultServeMux作为处理器。
      //Handle和HandleFunc函数可以向DefaultServeMux添加处理器。
      
      http.Handle("/foo", fooHandler)
      http.HandleFunc("/bar", func(w http.ResponseWriter, r *http.Request) {
      	fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
      })
      log.Fatal(http.ListenAndServe(":8080", nil))
      ```

  - 默认的Server示例

    - ```go
      // http server
      
      func sayHello(w http.ResponseWriter, r *http.Request) {
      	fmt.Fprintln(w, "Hello！")
      }
      
      func main() {
      	http.HandleFunc("/", sayHello)
      	err := http.ListenAndServe(":9090", nil)
      	if err != nil {
      		fmt.Printf("http server failed, err:%v\n", err)
      		return
      	}
      }
      ```

  - 自定义Server

    - ```go
      // 要管理服务端的行为，可以创建一个自定义的Server：
      
      s := &http.Server{
      	Addr:           ":8080",
      	Handler:        myHandler,
      	ReadTimeout:    10 * time.Second,
      	WriteTimeout:   10 * time.Second,
      	MaxHeaderBytes: 1 << 20,
      }
      log.Fatal(s.ListenAndServe())
      ```

      

























