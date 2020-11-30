





# servlet



## 简介

- 处理请求和发送响应的过程是由一种叫做Servlet的程序来完成的，并且Servlet是为了解决实现动态页面而衍生的东西



## tomcat和servlet的关系

- Tomcat 是Web应用服务器,是一个Servlet/JSP容器，处理客户请求,把请求传送给Servlet,并将Servlet的响应传送回给客户
  - Tomcat将http请求文本接收并解析，然后封装成HttpServletRequest类型的request对象
  - Tomcat同时会要响应的信息封装为HttpServletResponse类型的response对象
- Servlet是一种运行在支持Java语言的服务器上的组件. Servlet最常见的用途是扩展Java Web服务器功能





## servlet重要对象

- 对象
  - ServletConfig对象
  - ServletContext对象
    - 功能：tomcat为每个web项目都创建一个ServletContext实例，tomcat在启动时创建，服务器关闭时销毁
  - request对象
  - response对象





## 过滤器监听器

- Servlet、过滤器、监听器实例化对象的优先级和销毁的优先级
  - 创建（初始化）:  （ServletContext） 监听器–>过滤器–>Servlet. 
  - 调用时候：过滤器–>监听器–>Servlet. 
  - 销亡：Servlet–>过滤器–>监听器.





## Servlet3.0

- Servlet 线程不再是一直处于阻塞状态以等待业务逻辑的处理，而是启动异步线程之后可以立即返回

  - Servlet 3.0 在 <servlet>和 <filter> 标签中增加了 <async-supported> 子标签
  - @WebServlet 和 @WebFilter 进行 Servlet 或 Filter 配置的情况，这两个注解都提供了 asyncSupported 属性

- ```java
  //1.获得异步上下文对象
    		AsyncContext ac = request.startAsync();
    		//2.启动一个耗时的子线程
    		ThreadTask tt = new ThreadTask(ac);
    		//3.可设置异步超时对象，需在启动异步上下文对象前设置
    		/*
    		 * 设置超时后，在超时时间内子线程没有结束，主线程则会停止等待，继续往下执行
    		 */
    		ac.setTimeout(3000);
    		//4.开启异步上下文对象
    		ac.start(tt);
    
  //进行异步的一些处理
          HttpServletRequest requst = (HttpServletRequest) ac.getRequest();
          HttpSession session = requst.getSession();
  
  //通知主线程已经处理完成
  			/* 
  			 * 除了使用 ac.complete() 方法通知主线程已经处理外
  			 * 还可以使用 ac.dispatch() 方法重定向到一个页面
  			 */
          ac.dispatch("/show.jsp");
  
  ```

- servlet 3.0 还为异步处理提供了一个监听器，使用 AsyncListener 接口表示

  - 异步线程开始时，调用 AsyncListener 的 onStartAsync(AsyncEvent event) 方法；
  - 异步线程出错时，调用 AsyncListener 的 onError(AsyncEvent event) 方法；
  - 异步线程执行超时，则调用 AsyncListener 的 onTimeout(AsyncEvent event) 方法；
  - 异步执行完毕时，调用 AsyncListener 的 onComplete(AsyncEvent event) 方法。
  - 如果要注册一个 AsyncListener，只需将准备好的 AsyncListener 对象传递给 AsyncContext 对象的 addListener() 方法即可







## GET/POST

- 区别
  - GET请求在URL中传送的参数是有长度限制的，而POST没有。
    - 对于一个字节流的解析，必须分配buffer来保存所有要存储的数据。而URL这种东西必须当作一个整体看待，无法一块一块处理，于是就处理一个URL请求时必须分配一整块足够大的内存
  - 最直观的区别就是GET把参数包含在URL中，POST通过request body传递参数
  - GET和POST本质上没有区别
    - HTTP是基于TCP/IP的关于数据如何在万维网中如何通信的协议
    - HTTP只是个行为准则，而TCP才是GET和POST怎么实现的基本
    - GET和POST的底层也是TCP/IP，GET/POST都是TCP链接
    - GET产生一个TCP数据包；POST产生两个TCP数据包
  - 从攻击的角度，无论是GET还是POST都不够安全
    - HTTP本身是明文协议。每个HTTP请求和返回的每个byte都会在网络上传播，不管是url，header还是body
- ElasticSearch的_search接口使用GET，却用body来表达查询，因为查询很复杂







# Session&Cookie



## 概念

- HTTP协议是无状态的协议。一旦数据交换完毕，客户端与服务器端的连接就会关闭，再次交换数据需要建立新的连接。这就意味着服务器无法从连接上跟踪会话
  - 会话，指用户登录网站后的一系列动作
  - 常用的会话跟踪技术 是Cookie与Session
  - Cookie通过在客户端记录信息确定用户身份，Session通过在服务器端记录信息确定用户身份





## Cookie

- cookie的内容主要包括：名字，值，过期时间，路径和域。路径与域一起构成cookie的作用范围
  - 如果我们想让 www.china.com能够访问bbs.china.com设置的cookies，该怎么办? 我们可以把domain属性设置成“china.com”，并把path属性设置成“/”
- Cookie具有不可跨域名性



## Session

- 用户与服务器建立连接的同时，服务器会自动为其分配一个SessionId
- HttpSession是Servlet三大域对象之一（request、session、application（ServletContext））
- 禁用cookie
  - URL重写，就是把sessionId直接附加在URL路径的后面
    - 使用Response.encodeURL则可以直接得到路径+jsessionid的全部url路径，不需要自己手动拼接字符串了。然后将这个url返回给客户端，用户通过一个链接点击(通过refresh来刷新，再次访问本页面效果如下图)
  - 表单隐藏字段。就是服务器会自动修改表单，添加一个隐藏字段，以便在表单提交时能够把session id传递回服务器





## JWT

- 轻量级的认证规范，这个规范允许我们使用JWT在用户和服务器之间传递安全可靠的信息

- 整个 jwt 串会被置于 http 的 Header 或者 url 中

- 在 jwt 中以.分割三个部分

  - 一个JWT实际上就是一个字符串，它由三部分组成，头部、载荷与签名
  - jwt的第三部分是一个签名信息，这个签名信息由三部分组成：	
    - header (base64后的)
    - payload (base64后的)
    - secret	
    - 通过header中声明的加密方式进行加密

- jwt token泄露了怎么办

  - https
  - 返回 jwt 给客户端时设置 httpOnly=true 并且使用 cookie 而不是 LocalStorage 存储 jwt，这样可以防止 XSS 攻击和 CSRF 攻击

- 我们使用JWT的初衷就在于，我们可以不用通过读取状态来得知请求者的一些信息，因为JWT中自带了一些不敏感的信息，比如用户Id，权限列表，而且JWT不可伪造，所以我们直接通过解析JWT就能够知道一些原本需要从数据库/缓存里读取的数据。

  - 如果每一个请求到来的时候都要去读取状态并检测这个JWT是否有效，那么就和使用JWT的初衷相违背了，这是自相矛盾的。如果我都要去读取数据库和缓存了，那我为什么还要用JWT呢？为什么不直接给一个随机的字符串ID（比如SessionId），然后每次请求到来的时候通过这个Id取出当前的请求者信息不就完了？

- jwt 的特性天然不支持续签

  - 因为 payload 是参与签名的，一旦过期时间被修改，整个 jwt 串就变了

- jwt 不仅仅是作为身份认证，还在其 payload 中存储着会话信息，这是 jwt 和 session 的最大区别

- 什么场景该适合使用jwt？

  - 一次性验证，用户注册后需要发一封邮件让其激活账户
  - 单点登录系统
  - restful api 的无状态认证
    - JWT是自我校验的，所以是无状态的。JWT在客户端是不能做任何操作的，只有客户端发送请求时附带token，然后由服务端解析JWT后做自我校验
    - 服务端无需存储jwt令牌，通过特定的算法和密钥校验token，同时取出Payload中携带的用户ID，减少不必要的数据库查询

- 用户认证

  - 所谓用户认证（Authentication），就是让用户登录，并且在接下来的一段时间内让用户访问网站时可以使用其账户，而不需要再次登录的机制。

    > 小知识：可别把用户认证和用户授权（Authorization）搞混了。用户授权指的是规定并允许用户使用自己的权限，例如发布帖子、管理站点等。

    首先，服务器应用（下面简称“应用”）让用户通过 Web 表单将自己的用户名和密码发送到服务器的接口。这一过程一般是一个 HTTP POST 请求。建议的方式是通过 SSL 加密的传输（https 协议），从而避免敏感信息被嗅探。

    [![auth1](https://kirito.iocoder.cn/jwtauth1.png)](https://kirito.iocoder.cn/jwtauth1.png)auth1

    接下来，应用和数据库核对用户名和密码。

    [![auth2](https://kirito.iocoder.cn/jwtauth2.png)](https://kirito.iocoder.cn/jwtauth2.png)auth2

    核对用户名和密码成功后，应用将用户的 `id`（图中的 `user_id`）作为 JWT Payload 的一个属性，将其与头部分别进行 Base64 编码拼接后签名，形成一个 JWT。这里的 JWT 就是一个形同 `lll.zzz.xxx` 的字符串。

    [![auth3](https://kirito.iocoder.cn/jwtauth3.png)](https://kirito.iocoder.cn/jwtauth3.png)auth3

    应用将 JWT 字符串作为该请求 Cookie 的一部分返回给用户。注意，在这里必须使用 `HttpOnly` 属性来防止 Cookie 被 JavaScript 读取，从而避免 [跨站脚本攻击（XSS 攻击）](http://www.cnblogs.com/bangerlee/archive/2013/04/06/3002142.html)。

    [![auth4](https://kirito.iocoder.cn/jwtauth4.png)](https://kirito.iocoder.cn/jwtauth4.png)auth4

    在 Cookie 失效或者被删除前，用户每次访问应用，应用都会接受到含有 `jwt` 的 Cookie。从而应用就可以将 JWT 从请求中提取出来。

    [![auth5](https://kirito.iocoder.cn/jwtauth5.png)](https://kirito.iocoder.cn/jwtauth5.png)auth5

    应用通过一系列任务检查 JWT 的有效性。例如，检查签名是否正确；检查 Token 是否过期；检查 Token 的接收方是否是自己（可选）。

    [![auth6](https://kirito.iocoder.cn/jwtauth6.png)](https://kirito.iocoder.cn/jwtauth6.png)auth6

    应用在确认 JWT 有效之后，JWT 进行 Base64 解码（可能在上一步中已经完成），然后在 Payload 中读取用户的 id 值，也就是 `user_id` 属性。这里用户的 `id` 为 1025。

    应用从数据库取到 `id` 为 1025 的用户的信息，加载到内存中，进行 ORM 之类的一系列底层逻辑初始化。

    [![auth7](https://kirito.iocoder.cn/jwtauth7.png)](https://kirito.iocoder.cn/jwtauth7.png)auth7

    应用根据用户请求进行响应。

    [![auth8](https://kirito.iocoder.cn/jwtauth8.png)](https://kirito.iocoder.cn/jwtauth8.png)auth8

    ### 和 Session 方式存储 id 的差异

    Session 方式存储用户 id 的最大弊病在于要占用大量服务器内存，对于较大型应用而言可能还要保存许多的状态。一般而言，大型应用还需要借助一些 KV 数据库和一系列缓存机制来实现 Session 的存储。

    而 JWT 方式将用户状态分散到了客户端中，可以明显减轻服务端的内存压力。除了用户 id 之外，还可以存储其他的和用户相关的信息，例如该用户是否是管理员、用户所在的分桶（见 [《你所应该知道的 A/B 测试基础》一文] 等。

    虽说 JWT 方式让服务器有一些计算压力（例如加密、编码和解码），但是这些压力相比磁盘 I/O 而言或许是半斤八两。具体是否采用，需要在不同场景下用数据说话。

    ### 单点登录

    Session 方式来存储用户 id，一开始用户的 Session 只会存储在一台服务器上。对于有多个子域名的站点，每个子域名至少会对应一台不同的服务器，例如：

    - [www.taobao.com](http://www.taobao.com/)
    - nv.taobao.com
    - nz.taobao.com
    - login.taobao.com

    所以如果要实现在 `login.taobao.com` 登录后，在其他的子域名下依然可以取到 Session，这要求我们在多台服务器上同步 Session。

    使用 JWT 的方式则没有这个问题的存在，因为用户的状态已经被传送到了客户端。因此，我们只需要将含有 JWT 的 Cookie 的 `domain` 设置为顶级域名即可，例如

    ```
    Set-Cookie: jwt=lll.zzz.xxx; HttpOnly; max-age=980000; domain=.taobao.com
    ```

    注意 `domain` 必须设置为一个点加顶级域名，即 `.taobao.com`。这样，taobao.com 和 *.taobao.com 就都可以接受到这个 Cookie，并获取 JWT 了。



## jwt session

- http 无状态，所以为了实现有状态 http，才有了会话（session）的概念。
  - 会话(Session)是一个客户与服务器之间的不中断的请求响应序列。对客户的每个请求，服务器能够识别出请求来自于同一个客户。当一个未知的客户向Web应用程序发送第一个请求时就开始了一个会话。当客户明确结束会话或服务器在一个预定义的时限内不从客户接受任何请求时，会话就结束了。当会话结束后，服务器就忘记了客户以及客户的请求。
- 不管是“session id”，还是所谓“token”(如 jwt)，其实都是会话的一种实现方式。形式上“session id”和“token”都是“字符串”，这个“字符串”可以是任意的编码，本质上都是 credential（会话凭证）。
- 在各种 session 方案中，你会发现实现细节都不一样，flask session, django session, spring session, jwt 等等。但万变不离其宗， credential 的客户端保存方式，credential 的传输方式和会话信息的保存方式，都只是这几个流程的细节有改变而已，本质都是为了实现有状态的 http。

## OAuth2.0是什么

#### OAuth2.0是什么——豆瓣和QQ的故事

OAuth简单说就是一种授权的**协议**，只要授权方和被授权方遵守这个协议去写代码提供服务，那双方就是实现了OAuth模式。

举个例子，你想登录豆瓣去看看电影评论，但你丫的从来没注册过豆瓣账号，又不想新注册一个再使用豆瓣，怎么办呢？不用担心，豆瓣已经为你这种懒人做了准备，用你的qq号可以授权给豆瓣进行登录，请看。

**第一步：在豆瓣官网点击用qq登录**

**![img](https://images2017.cnblogs.com/blog/1096103/201708/1096103-20170824150221230-104373567.png)**

**第二步：跳转到qq登录页面输入用户名密码，然后点授权并登录**

 ![img](https://images2017.cnblogs.com/blog/1096103/201708/1096103-20170824151117089-994331290.png)

**第三步：跳回到豆瓣页面，成功登录**

![img](https://images2017.cnblogs.com/blog/1096103/201708/1096103-20170824151220418-2113396046.png)

 这几秒钟之内发生的事情，在**无知的用户视角**看来，就是在豆瓣官网上输了个qq号和密码就登录成功了。在一些**细心的用户视角**看来，页面经历了从豆瓣到qq，再从qq到豆瓣的两次页面跳转。但作为一群专业的程序员，我们还应该从**上帝视角**来看这个过程。

#### OAuth2.0是什么——上帝视角

　　简单来说，上述例子中的豆瓣就是**客户端**，QQ就是**认证服务器**，OAuth2.0就是客户端和认证服务器之间由于相互**不信任**而产生的一个**授权协议**。呵呵，要是相互信任那QQ直接把自己数据库给豆瓣好了，你直接在豆瓣输入qq账号密码查下数据库验证就登陆呗，还跳来跳去的多麻烦。

　　先上一张图，该图描绘了只几秒钟发生的所有事情用**上帝视角**来看的流程

![img](https://images2017.cnblogs.com/blog/1096103/201708/1096103-20170824142737402-1297004164.png)

 就这这张图，来说一下上述例子中的三个步骤在图中的表现。所用到的请求路径名称都是虚构的，所附带的请求参数忽略了一些非重点的。

如想了解每次的请求和响应的标准齐全的参数，还是去读那篇阮一峰的文章。http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html

**第一步：在豆瓣官网点击用qq登录**

　　当你点击用qq登录的小图标时，实际上是向豆瓣的服务器发起了一个 http://www.douban.com/leadToAuthorize 的请求，豆瓣服务器会响应一个**重定向地址**，指向qq授权登录

　　浏览器接到重定向地址 http://www.qq.com/authorize?callback=www.douban.com/callback ，再次访问。并注意到这次访问带了一个参数是callback，以便qq那边授权成功再次让浏览器发起这个callback请求。不然qq怎么知道你让我授权后要返回那个页面啊，每天让我授权的像豆瓣这样的网站这么多。

　　至于访问这个地址之后，qq那边做出怎样的回应，就是第二步的事情了。总之第一步即对应了图中的这些部分。

![img](https://images2017.cnblogs.com/blog/1096103/201708/1096103-20170824155817777-2073704717.png)

**第二步：跳转到qq登录页面输入用户名密码，然后点授权并登录**

　　上一步中浏览器接到重定向地址并访问 http://www.qq.com/authorize?callback=www.douban.com/callback

　　qq的服务器接受到了豆瓣访问的authorize，在次例中所给出的回应是跳转到qq的登录页面，用户输入账号密码点击授权并登录按钮后，一定还会访问qq服务器中校验用户名密码的方法，若校验成功，该方法会响应浏览器一个重定向地址，并附上一个**code（授权码）**。由于豆瓣只关心像qq发起authorize请求后会返回一个code，并不关心qq是如何校验用户的，并且这个过程每个授权服务器可能会做些个性化的处理，只要最终的结果是返回给浏览器一个重定向并附上code即可，所以这个过程在图中并没有详细展开。现把展开图画给大家。

![img](https://images2017.cnblogs.com/blog/1096103/201708/1096103-20170824161339668-1419889465.png)

**第三步：跳回到豆瓣页面，成功登录**

 这一步背后的过程其实是最繁琐的，但对于用户来说是完全感知不到的。用户在QQ登录页面点击授权登陆后，就直接跳转到豆瓣首页了，但其实经历了很多隐藏的过程。

首先接上一步，QQ服务器在判断登录成功后，使页面重定向到之前豆瓣发来的callback并附上code授权码，即 callback=www.douban.com/callback 

页面接到重定向，发起 http://www.douban.com/callback 请求

豆瓣服务器收到请求后，做了两件再次与QQ沟通的事，即模拟浏览器发起了两次请求。一个是用拿到的code去换token，另一个就是用拿到的token换取用户信息。最后将用户信息储存起来，返回给浏览器其首页的视图。到此OAuth2.0授权结束。

![img](https://images2017.cnblogs.com/blog/1096103/201708/1096103-20170824162606793-113527222.png)



## 微服务设计中的API网关模式

根据 Gartner 对微服务的定义：“微服务是范围狭窄、封装紧密、松散耦合、可独立部署且可独立伸缩的应用程序组件。”

与将模块高度耦合并部署为一个大的应用程序相比，微服务的目标是将应用程序充分分解或者解耦为松散耦合的许多微服务或者模块，这样做对下面几点有很大帮助：

- 每个微服务都可以独立于应用程序中的同级服务进行部署、升级、扩展、维护和重新启动。
- 通过自治的跨职能团队进行敏捷开发和敏捷部署。
- 运用技术时具备灵活性和可扩展性

在微服务架构中，我们根据各自的特定需求部署不同的松耦合服务，其中每个服务都有其更细粒度的 API 模型，用以服务于不同的客户端（Web，移动和第三方 API）。

1客户端到微服务的连接

![img](https://oscimg.oschina.net/oscnet/044c6408-bbc4-4856-92dc-89fb49f70d03.png)

在考虑客户端与每个已部署的微服务 **直接通信** 的问题时，应考虑以下挑战：

1. 如果微服务向客户端公开了细粒度的 API，则客户端应向每个微服务发出请求。在典型的单页中，可能需要进行 **多次服务器往返**，才能满足请求。对于较差的网络条件下运行的设备（例如移动设备），这可能会更糟。
2. 微服务中存在的 **多种通信协议**（例如 gRpc、thrift、REST、AMQP 等）使客户端很难轻松采用所有这些协议。
3. 必须在每个微服务中实现 **通用网关功能**（例如身份验证、授权、日志记录）。
4. 在不中断客户端连接的情况下，**很难在微服务中进行更改**。例如，在合并或划分微服务时，可能需要重新编写客户端部分代码。

2API 网关

为了解决上述挑战，人们引入了一个附加层，该附加层位于客户端和服务器之间，充当从客户端到服务器的反向代理路由请求。与面向对象设计的模式相似，它为封装底层系统架构的 API 提供了一个单一的入口，称为 API 网关。

简而言之，它的行为就像 API 管理员一样，但重要的是不要将 API 管理与 API Gateway 混为一谈。

![img](https://oscimg.oschina.net/oscnet/3810bf35-ea8d-40ac-ad63-9388734530e3.png)

3API 网关的功能

路由

网关封装了底层系统并与客户端分离，为客户端提供了与微服务系统进行通信的单个入口点。

整合

API 网关整合了一些边缘的重复功能，无需让每个微服务都实现它们。它包括如下功能：

- 认证和授权
- 服务发现集成
- 缓存响应结果
- 重试策略、熔断器、QoS
- 限速和节流
- 负载均衡
- log 日志、链路追踪、关联
- Header、query 字符串 以及 claims 转义
- IP 白名单
- IAM
- 集中式日志管理（服务之间的 transaction ID、错误日志等）
- 身份的提供方，验证与授权

4后端服务前端模式（BFF Backend for Frontend）

它是 API 网关模式的一种变体。它提供了基于客户端的多个网关，而不是提供给客户端一个单一的入口点。目的是根据客户端的需求提供量身定制的 API，从而消除了为所有客户端制作通用 API 造成的大量的浪费。

![img](https://oscimg.oschina.net/oscnet/c18ea4ad-e32e-4f0a-8386-e284bb09926e.png)

到底需要多少 BFF

BFF 的基本概念是为每种用户体验开发利基后端。菲尔·卡尔萨多（PhilCalçado） 的指导建议是“**一种体验，一种 BFF**”。如果跨客户端（IOS 客户端、Android 客户端、Web 浏览器等）的要求有很大差异，并且单个代理或 API 的发布时间有严格要求，则 BFF 是一个很好的解决方案。还应注意，更复杂的设计需要复杂的步骤。

GraphQL 与 BFF

GraphQL 是一种 API 的查询语言。PhilCalçado 提出 BFF 和 GraphQL 的想法是相似的，但不是互斥的概念。他补充说，BFF 与你端口的形状无关，而在于赋予客户端对应用程序的自治权，您可以在其中构建与许多 BFF 或 OSFA（one-size-fits-all）的 GraphQL API。

5著名的 API 网关

Netflix API 网关：Zuul

Netflix 的流媒体服务可在 1000 多种不同类型的设备（电视、机顶盒、智能手机、游戏系统、平板电脑等）上使用，在高峰时段可以每秒处理 50,000 个请求，这种需求是 OSFA （one-size-fits-all）的 REST API 难以满足的，因此他们为每个设备量身定制了 API 网关。

Netflix 的 Zuul 2 是所有进入 Netflix 云基础架构的请求的第一步。Zuul 2 大大改进了架构和功能，使我们的网关能够处理、路由和保护 Netflix 的云系统，并帮助为我们的 1.25 亿会员提供最佳体验。



亚马逊 API 网关

AWS 提供了完备的托管服务，用于创建、发布、维护、监视以及保护 REST、HTTP 和 WebSocket，开发人员可以在其中创建用于访问 AWS 或其他 Web 服务的 API，并将数据存储在 AWS 云上面。



Kong API 网关

Kong Gateway 是一个开源的，轻量级的微服务 API 网关，可提供无与伦比的延迟性能优化和可伸缩性。如果您只需要这些基础能力，那么它就是很合适的选项。只需要增加更多节点就可以轻松横向扩展。它以非常低的延迟来支持大量可变的工作负载。

其他 API 网关

- Apigee API Gateway
- MuleSoft
- Tyk.io
- Akana
- SwaggerHub
- Azure API Gateway
- Express API Gateway
- Karken D

选择正确的网关

评估标准里面，一些常见的指标包括简便性、开源还是专有、可伸缩性和灵活性、安全性、后续功能、社区、管理（支持情况、监控和部署）、环境配置（安装、配置、是否支持托管）、定价和文档等。

6API 组合与聚合

API 网关中的一些 API 请求直接映射到单个服务的 API 上，可以通过将请求路由到相应的微服务来提供服务。但是，在需要从多个微服务获得结果的复杂 API 操作的情况下，可以通过 **API 组合 / 聚合**（分散 - 收集机制）来提供服务。在需要同步通信的情况下，如果服务彼此依赖，则必须遵循链式组合模式。组合层必须支持很大一部分的 ESB / 集成功能，例如转换、编排、弹性和稳定性模式。

根容器的部署必须配备特殊的分发器和聚合器功能（或微服务）。分发者负责分解成细粒度的任务，并将这些任务分发给微服务实例。聚合器负责聚合业务工作流从组合微服务中得出的结果。

API 网关和聚合

具备复杂功能的网关会增大测试和部署的难度。强烈建议大家避免在 API 网关中进行聚合和数据转换。领域专属的功能更应该遵循软件开发实践的定义，在应用程序的代码中完成。Netflix API Gateway Zuul 2 从他们在 Zuul 到原始系统的网关中，删除了许多业务逻辑。 

![img](https://oscimg.oschina.net/oscnet/3ad4e5c6-a6c4-454f-a754-f5b041a23f45.png)

Service Mesh 与 API 网关

微服务中的 Service Mesh 是处理进程间通信的可配置网络基础结构层。这和通常称为 Sidecar 代理或 Sidecar 网关的东西很像。它提供了许多功能，例如：

- 负载均衡
- 服务发现
- 健康检查
- 安全性

从表面上看，**API 网关和 Service Mesh 似乎解决了相同的问题，因此好像是多余的。它们确实解决了相同的问题，但是应用在不同的场景**。API 网关被部署为业务解决方案的一部分，被外部的服务发现，处理纵向的流量（面对外部客户端），但是，Service Mesh 是用来处理横向流量（在不同的微服务之间）。

实现 Service Mesh 可避免在您自己的代码中出现一些弹性交互，例如熔断器、服务发现、健康检查以及服务观察。对于少量的微服务，应考虑使用其他替代方法来进行故障管理，因为 Service Mesh 集成可能代价太大了。但对于大量的微服务，它的收益是显著的。

结合这两种技术可能是确保应用程序正常运行时间和弹性伸缩能力的一种有效方法，同时又可以确保您的应用程序易于使用。将两者视为同样的产品是不对的，最好将两者视为在涉及微服务和 API 的部署中相辅相成的工具。

API 网关实现的注意事项：

- 可能产生的单点故障或者瓶颈
- 由于通过 API 网关进行了额外的网络跳转以及复杂性风险，响应时间增长了。





# Web安全





## XSS

- 有很多用户发送了同样类型的内容，而且这些内容都是一个带有诱惑性的问题和一个可以点击的链接。
- 简单来说，XSS 就是利用 Web 漏洞，在用户的浏览器中执行黑客定义的 JavaScript 脚本，这样一种攻击方式。
- 如何进行 XSS 防护？
  - 验证输入 OR 验证输出
  - 编码
  - 检测和过滤
  - CSP
    - CSP（Content Security Policy，内容安
      全策略）来提升 Web 的安全性。所谓 CSP，就是在服务端返回的 HTTP header 里面添加
      一个 Content-Security-Policy 选项，然后定义资源的白名单域名。浏览器就会识别这个字
      段，并限制对非白名单资源的访问。
    - 那我们为什么要限制外域资源的访问呢？这是因为 XSS 通常会受到长度的限制，导致黑客无法提交一段完整的 JavaScript 代码。为了解决这个问题，黑客会采取引用一个外域JavaScript 资源的方式来进行注入。



## SQL注入

- 通常来说，我们会将应用的用户信息存储在数据库中。每次用户登录时，都会执行一个相应的 SQL 语句。这时，黑客会通过构造一些恶意的输入参数，在应用拼接 SQL 语句的时候，去篡改正常的 SQL 语意，从而执行黑客所控制的 SQL 查询功能。这个过程，就相当于黑客“注入”了一段 SQL 代码到应用中。这就是我们常说的 SQL 注入。

- SELECT * FROM Users WHERE Username ="" AND Password ="" or ""=""

- 使用 PreparedStatement

  - 通过合理地使用 PreparedStatement，我们就能够避免 99.99% 的 SQL 注入问题。

  - 当数据库在处理一个 SQL 命令的时候，大致可以分为两个步骤：

    - 将 SQL 语句解析成数据库可使用的指令集。我们在使用 EXPLAIN 关键字分析 SQL 语句，就是干的这个事情；
    - 将变量代入指令集，开始实际执行。之所以在批量处理 SQL 的时候能够提升性能，就是因为这样做避免了重复解析 SQL 的过程。

  - SQL 注入是在解析的过程中生效的，用户的输入会影响 SQL 解析的结果。因此，我们可以通过使用 PreparedStatement，将 SQL 语句的解析和实际执行过程分开，只在执行的过程中代入用户的操作。这样一来，无论黑客提交的参数怎么变化，数据库都不会去执行额外的逻辑，也就避免了 SQL 注入的发生。

  - ```java
    1 String sql = "SELECT * FROM Users WHERE UserId = ?"; 
    2 PreparedStatement statement = connection.prepareStatement(sql); 
    3 statement.setInt(1, userId); 
    4 ResultSet results = statement.executeQuery();
    
    
    // 如果你在使用 PreparedStatement 的时候，还是通过字符串拼接来构造 SQL语句，那仍然是将解析和执行放在了一块，也就不会产生相应的防护效果了。
    
    1 String sql = "SELECT * FROM Users WHERE UserId = " + userId; 
    2 PreparedStatement statement = connection.prepareStatement(sql); 
    3 ResultSet results = statement.executeQuery();
    ```







## CSRF/SSRF

- 在平常使用浏览器访问各种网页的时候，是否遇到过，自己的银行应用突然发起了一笔转账，又或者，你的微博突然发送了一条内容？

- 为了能够准确地代表你的身份，浏览器通常会在 Cookie 中存储一些必要的身份信息。所以，在我们使用一个网页的时候，只需要在首次访问的时候登录就可以了。

  - 黑客正是利用这一点，来编写带有恶意JavaScript 脚本的网页，通过“钓鱼”的方式诱导你访问。然后，黑客会通过这些JavaScript 脚本窃取你保存在网页中的身份信息，通过仿冒你，让你的浏览器发起伪造的请求，最终执行黑客定义的操作。而这一切对于你自己而言都是无感知的。这就是CSRF（Cross-Site Request Forgery，跨站请求伪造）攻击。

- 和 XSS 一样，CSRF 也可以仿冒用户去进行一些功能操作的请求，比如修改密码、转账等等，相当于绕过身份认证，进行未授权的操作。

- 行业内标准的 CSRF 防护方法是CSRFToken

  - CSRF 是通过自动提交表单的形式来发起攻击的。所以，在前面转账的例子中，黑客可以通过抓包分析出 http://bank.com/transfer 这个接口所需要的参数，从而构造对应的 form 表单。因此，我们只需要在这个接口中，加入一个黑客无法猜到的参数，就可以有效防止 CSRF 了。这就是 CSRF Token 的工作原理。
  - 因为 CSRF Token 是每次用户正常访问页面时，服务端随机生成返回给浏览器的。所以，每一次正常的转账接口调用，都会携带不同的 CSRF Token。黑客没有办法进行提前猜测，也就没有办法构造出正确的表单了。

- SSRF：同样的原理，发生在服务端又会发生什么？

  - 我们知道，服务端也有代理请求的功能：用户在浏览器中输入一个 URL（比如某个图片资源），然后服务端会向这个 URL 发起请求，通过访问其他的服务端资源来完成正常的页面展示。
  - 这个时候，只要黑客在输入中提交一个内网 URL，就能让服务端发起一个黑客定义的内网
    请求，从而获取到内网数据。这就是SSRF（Server Side Request Forgery，服务端请求伪造）的原理。而服务端作为内网设备，通常具备很高的权限，所以，这个伪造的请求往往
    因为能绕过大部分的认证和授权机制，而产生很严重的后果。
  - 比方说，当我们在百度中搜索图片时，会涉及图片的跨域加载保护，百度不会直接在页面中加载图片的源地址，而是将地址通过 GET 参数提交到百度服务器，然后百度服务器请求到对应的图片，再返回到页面展示出来。
  - 这个过程中，百度服务器实际上会向另外一个 URL 地址发起请求。利用这个代理发起请求的功能，黑客可以通过提交一个内网的地址，实现对内网任意服务的访问。这就是 SSRF 攻击的实现过程，也就是我们常说的“内网穿透”。

- 因为 SSRF 最终的结果，是接受代理请求的服务端发生数据泄漏。所以，SSRF防护不仅仅涉及接收 URL 的服务端检测，也需要接受代理请求的服务端进行配合。在这种情况下，我们就需要用到请求端限制，它的防护措施主要包括两个方面。

  - 第一，为其他业务提供的服务接口尽量使用 POST，避免 GET 的使用。因为，在 SSRF 中
    （以及大部分的 Web 攻击中），发起一个 POST 请求的难度是远远大于 GET 请求的。
  - 第二，为其他业务提供的服务接口，最好每次都进行验证。通过 SSRF，黑客只能发起请求，并不能获取到服务端存储的验证信息（如认证的 key 和 secret 等）。因此，只要接受代理请求的端对每次请求都进行完整的验证，黑客无法成功通过验证，也就无法完成请求了。

  



# 抓包

- **抓取我们本地电脑与远端服务器通信时候所传递的数据包**
- Network 面板记录页面上每个网络操作的相关信息，包括详细的耗时数据、HTTP 请求与响应标头和 Cookie

结构：由五个窗格组成，如图



![img](https://pic2.zhimg.com/80/v2-db7c344e230b41f719e727b542bdf151_1440w.jpg)



1：**`Controls（控件）`** 使用这些选项可以控制 Network(网络)面板的外观和功能



![img](https://pic3.zhimg.com/80/v2-c95a73ba3432ac83685dc31587e8ae12_1440w.jpg)



2：**`Filters（过滤器）`**

**`使用这些选项可以控制在请求列表中显示哪些资源`**



**`提示：按住Ctrl（Window / Linux），然后点击过滤器可以同时选择多个过滤器。`**

**`此外，筛选框可以实现很多定制化的筛选，比如字符串匹配，关键词筛选等，其中关键词筛选主要有如下几种：`**



- **`domain`**：仅显示来自指定域的资源。您可以使用通配符（*）来包括多个域。例如，*.com显示以.com结尾的所有域名中的资源。 DevTools会在自动完成下拉菜单中自动填充它遇到的所有域。
- **`has-response-header`**：显示包含指定HTTP响应头信息的资源。 DevTools会在自动完成下拉菜单中自动填充它遇到的所有响应头。
- **`is`**：通过is:running找出WebSocket请求。
- **`larger-than(大于)`** ：显示大于指定大小的资源（以字节为单位）。设置值1000等效于设置值1k。
- **`method(方法)`** ：显示通过指定的HTTP方法类型检索的资源。DevTools使用它遇到的所有HTTP方法填充下拉列表。
- **`mime-type（mime类型`**：显示指定MIME类型的资源。 DevTools使用它遇到的所有MIME类型填充下拉列表。
- **`mixed-content（混合内容`**：显示所有混合内容资源（mixed-content:all）或仅显示当前显示的内容（mixed-content:displayed）。
- **`Scheme（协议）`**：显示通过不受保护的HTTP（scheme:http）或受保护的HTTPS（scheme:https）检索的资源。
- **`set-cookie-domain（cookie域）`**：显示具有Set-Cookie头,并且其Domain属性与指定值匹配的资源。DevTools会在自动完成下拉菜单中自动填充它遇到的所有Cookie域。
- **`set-cookie-name（cookie名）`**：显示具有Set-Cookie头,并且名称与指定值匹配的资源。DevTools会在自动完成下拉菜单中自动填充它遇到的所有Cookie名。
- **`set-cookie-value（cookie值）`**：显示具有Set-Cookie头,并且值与指定值匹配的资源。DevTools会在自动完成下拉菜单中自动填充它遇到的所有cookie值。
- **`status-code（状态码）`**：仅显示其HTTP状态代码与指定代码匹配的资源。DevTools会在自动完成下拉菜单中自动填充它遇到的所有状态码。

![img](https://pic3.zhimg.com/80/v2-602e6d178e72a8df38106f95c4538c4a_1440w.jpg)



3：**`Overview（概览）`**

这个图表显示检索资源的时间轴。如果您看到多个垂直堆叠的栏，这意味着这些资源被同时检索。

4：**`Requests Table（请求列表）`**

此列表列出了检索的每个资源。默认情况下，此表按时间顺序排序，也就是最早的资源在顶部。单击资源名称可以获得更多信息。提示：右键单击列表的任何标题栏可以以添加或删除信息列。

**`查看单个资源的详细信息`**

点击资源名称（位于 Requests Table 的 Name 列下）可以查看与该资源有关的更多信息。

可用标签会因您所选择资源类型的不同而不同，但下面四个标签最常见：

- **[Headers](https://link.zhihu.com/?target=https%3A//static.dcxueyuan.com/content/disk/train/other/27973d34-2af3-481d-9a97-8850a177d266.html%23jump1)**：与资源关联的 HTTP 标头。
- **[Preview](https://link.zhihu.com/?target=https%3A//static.dcxueyuan.com/content/disk/train/other/27973d34-2af3-481d-9a97-8850a177d266.html%23jump2)**：JSON、图像和文本资源的预览。
- **[Response](https://link.zhihu.com/?target=https%3A//static.dcxueyuan.com/content/disk/train/other/27973d34-2af3-481d-9a97-8850a177d266.html%23jump3)**：HTTP 响应数据（如果存在）。
- **`Timing`**：资源请求生命周期的精细分解。

**`Headers（查看 HTTP 标头）`** 点击 **`Headers`** 可以显示该资源的标头。 **`Headers`** 标签可以显示资源的请求网址、HTTP 方法以及响应状态代码。 此外，该标签还会列出 HTTP 响应和请求标头、它们的值以及任何查询字符串参数



![img](https://pic3.zhimg.com/80/v2-4f93efd5f2b8a7257d9f2da1b95d6dc2_1440w.jpg)



**`点击每一部分旁边的 view source 或 view parsed 链接，您能够以源格式或者解析格式查看响应标头、请求标头或者查询字符串参数。`**



**`Response（查看 HTTP 响应内容）`**
点击 Response 标签可以查看资源未格式化的 HTTP 响应内容。 Preview 标签可能包含一些有用的信息，也可能不包含，具体取决于您所选择资源的类型。



![img](https://pic1.zhimg.com/80/v2-5be61980a4c589fb7e740a2880ed0a74_1440w.jpg)



**`查看 Cookie`**
点击 Cookies 标签可以查看在资源的 HTTP 请求和响应标头中传输的 Cookie 表。 只有传输 Cookie 时，此标签才可用。
下面是 Cookie 表中每一列的说明：

- Name：Cookie 的名称。
- Value：Cookie 的值。
- Domain：Cookie 所属的域。
- Path：Cookie 来源的网址路径。
- Expires / Max-Age：Cookie 的 expires 或 max-age 属性的值。
- Size：Cookie 的大小（以字节为单位）。
- HTTP：指示 Cookie 应仅由浏览器在 HTTP 请求中设置，而无法通过 JavaScript 访问。
- Secure：如果存在此属性，则指示 Cookie 应仅通过安全连接传输。



![img](https://pic1.zhimg.com/80/v2-95a59627d85bcd163e8fc4718b83dc58_1440w.jpg)

**复制、保存和清除网络信息**

- 右键单击**`Requests Table`**（**`请求列表`**）以复制、保存或删除网络信息。一些选项是上下文相关的，所以如果想在单个资源上操作，需要右键单击该资源行。下面的列表描述了每个选项
- **`Copy Response（复制响应）`**
  将所选资源的HTTP响应复制到系统剪贴板。
- **`Copy as cURL（复制为cURL）`** 将所选资源的网络请求作为cURL命令字符串复制到系统剪贴板。 请参阅将复制请求为cURL命令。
  `curl命令是一个利用URL规则在命令行下工作的文件传输工具。它支持文件的上传和下载，所以是综合传输工具，但按传统，习惯称curl为下载工具。作为一款强力工具，curl支持包括HTTP、HTTPS、ftp等众多协议，还支持POST、cookies、认证、从指定偏移处下载部分文件、用户代理字符串、限速、文件大小、进度条等特征。做网页处理流程和数据检索自动化，curl可以祝一臂之力。l`
- **`Copy All as HAR（全部复制为HAR）`**
  将所有资源复制到系统剪贴板作为HAR数据。 HAR文件包含描述网络“瀑布”的JSON数据结构。一些第三方工具可以使用HAR文件中的数据重建网络瀑布。有关详细信息，请参阅Web性能强大工具：HTTP归档（HAR）。
- **`Save as HAR with Content（另存为带内容的HAR）`**
  将所有网络数据与每个页面资源一起保存到HAR文件中。 二进制资源（包括图像）被编码为Base64编码文本。
- **`Clear Browser Cache（清除浏览器缓存）`**
  清除浏览器高速缓存。提示：您也可以从Network Conditions(网络条件)抽屉式窗格中启用或禁用浏览器缓存。
- **`Clear Browser Cookies（清除浏览器Cookie）`**
  清除浏览器的Cookie。
- **`Open in Sources Panel（在源文件面板中打开）`**
  在Sources(源文件)面板中打开选定的资源。
- **`Open Link in New Tab（在新标签页中打开链接）`**
  在新标签页中打开所选资源。您还可以在Requests Table(请求列表)中双击资源名称。
- **`Copy Link Address（复制链接地址）`**
  将资源URL复制到系统剪贴板。
- **`Save（保存）`**
  保存所选的文本资源。仅显示在文本资源上。
- **`Replay XHR（重新发送XHR）`**
  重新发送所选的XMLHTTPRequest。仅显示在XHR资源上。

**`查看资源发起者和依赖关系`**

按住`Shift`并移动鼠标到资源上可查看它的发起者和依赖关系。这部分是你鼠标悬停的资源的target(目标)引用。 从target(目标)往上查找，第一个颜色编码为`绿色`的资源是target(目标)的发起者。如果存在第二个颜色编码为`绿色`资源，那么这个是发起者的发起者。从target(目标)向下查找，任何颜色编码为`红色`的资源都是target的依赖。



![img](https://pic3.zhimg.com/80/v2-23779e7ca8ce0aae38f75c8b11ea3d6a_1440w.jpg)









































​			