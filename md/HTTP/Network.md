

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

-  Servlet 线程不再是一直处于阻塞状态以等待业务逻辑的处理，而是启动异步线程之后可以立即返回

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





# Network



## DNS

- 层次树状结构的联机分布式数据库系统
  - 产生于应用层上的域名系统 NDS就可以用来把互联网上的主机名转换成 IP 地址
  - 把待解析的域名放在 DNS 的请求报中，以 UDP 用户数据报方式发送给本地域名服务器。本地域名服务器在查找域名后，把对应的 IP 地址放在回答报文中返回。获得 IP 地址的后主机即可进行通信
- 域名解析过程
  - 域名解析过程
    - 本地域名服务器向根域名服务器的查询方式通常采取迭代查询
  - 递归查询
    - 主机向本地域名服务器的查询一般都采用递归查询





## CDN

- 内容分发网络，解决的是如何将数据快速可靠从源站传递到用户
- 数据从服务器端交付到用户端，至少有4个地方可能会造成网络拥堵
  - 网站服务器接入互联网的链路
  - 用户接入互联网的链路
  - ISP互联，即因特网服务提供商之间的互联
  - 长距离传输时延问题
- 基本过程
  - 用户在浏览器中输入要访问的域名。 
  2. 浏览器向DNS服务器请求对域名进行解析。由于CDN对域名解析进行了调整，DNS服务器会最终将域名的解析权交给CDN专用DNS服务器。 
  3. CDN的DNS服务器将CDN的负载均衡设备IP地址返回给用户。 
  4. 用户向CDN的负载均衡设备发起内容URL访问请求。 
  - CDN负载均衡设备会为用户选择一台合适的缓存服务器提供服务。 
    5. 选择的依据包括：根据用户IP地址，判断哪一台服务器距离用户最近；根据用户所请求的URL中携带的内容名称，判断哪一台服务器上有用户所需内容；查询各个服务器的负载情况，判断哪一台服务器的负载较小。 
  5. 用户向缓存服务器发出请求。
  7. 缓存服务器响应用户请求，将用户所需内容传送到用户。
7. CDN的工作原理：通过权威DNS服务器来实现最优节点的选择，通过缓存来减少源站的压力





## HTTP

- HTTP幂等性
  - 幂等性是数学中的一个概念，表达的是N次变换与1次变换的结果相同
  - HTTP POST和PUT，二者均可用于创建资源，更为本质的差别是在幂等性方面
    - POST所对应的URI并非创建的资源本身，而是资源的接收者
      - 两次相同的POST请求会在服务器端创建两份资源，它们具有不同的URI；所以，POST方法不具备幂等性
    - PUT所对应的URI是要创建或更新的资源本身
      - 对同一URI进行多次PUT的副作用和一次PUT是相同的；因此，PUT方法具有幂等性
- HTTP协议的瓶颈及其优化技巧都是基于TCP协议本身的特性
  - 三次握手有1.5个RTT（round-trip time）的延迟
  - 不同策略的http长链接方案
    - HTTP的长连接和短连接本质上是TCP长连接和短连接
  - TCP在建立连接的初期有慢启动
- http和socket长连接和短连接区别
  - Socket是应用层与TCP/IP协议族通信的中间软件抽象层，它是一组接口
  - 门面模式，它把复杂的TCP/IP协议族隐藏在Socket接口后面，对用户来说，一组简单的接口就是全部，让Socket去组织数据，以符合指定的协议
- HTTP2.0
  - HTTP1.x有以下几个主要缺点：
    - 一次只允许在一个TCP连接上发起一个请求
    - HTTP/1.1使用的流水线技术也只能部分处理请求并发，仍然会存在队列头阻塞问题，因此客户端在需要发起多次请求时，通常会采用建立多连接来减少延迟。
    - 单向请求，只能由客户端发起。
    - 请求报文与响应报文首部信息冗余量大。
    - 数据未压缩，导致数据的传输量大。
  - 多路复用 (Multiplexing)
    - 所谓多路复用，即在一个TCP连接中存在多个流
    - HTTP/2 通信都在一个连接上完成，这个连接可以承载任意数量的双向数据流。在过去， HTTP 性能优化的关键并不在于高带宽，而是低延迟
  - 二进制分帧
    - 很容易的去实现多流并行而不用依赖建立多个 TCP 连接，HTTP/2 把 HTTP 协议通信的基本单位缩小为一个一个的帧
    - HTTP2.0中，有两个概念非常重要：帧（frame）和流（stream）。
      帧是最小的数据单位，每个帧会标识出该帧属于哪个流，流是多个帧组成的数据流。
  - 首部压缩（Header Compression）
  - 服务端推送（Server Push）
- 哈希算法
  - 将任意长度的信息转换为较短的固定长度的值，通常其长度要比信息小得多，且算法不可逆。







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





## RestTemplate

- RestTemplate能大幅简化了提交表单数据的难度，并且附带了自动转换JSON数据的功能

  | HTTP method | RestTemplate methods |
  | :---------- | :------------------- |
  | DELETE      | delete               |
  | GET         | getForObject         |
  |             | getForEntity         |
  | HEAD        | headForHeaders       |
  | OPTIONS     | optionsForAllow      |
  | POST        | postForLocation      |
  |             | postForObject        |
  | PUT         | put                  |
  | any         | exchange             |
  |             | execute              |

- 手动指定转换器(HttpMessageConverter)

  - 调用reseful接口传递的数据内容是json格式的字符串，返回的响应也是json格式的字符串
  - restTemplate.postForObject方法的请求参数RequestBean和返回参数ResponseBean都是java类。是RestTemplate通过HttpMessageConverter自动帮我们做了转换的操作
    - StringHttpMessageConverter来处text/plain;
    - MappingJackson2HttpMessageConverter来处理application/json;
    - MappingJackson2XmlHttpMessageConverter来处理application/xml

- RestTemplate直接使用一个HttpClient作为底层实现

- 设置拦截器(ClientHttpRequestInterceptor)

  - ```java
    // 1.实现ClientHttpRequestInterceptor接口
    
    
    RestTemplate restTemplate = new RestTemplate();
    // 2.向restTemplate中添加自定义的拦截器
    restTemplate.getInterceptors().add(new TokenInterceptor());
    ```

- getForObject()其实比getForEntity()多包含了将HTTP转成POJO的功能，但是getForObject没有处理response的能力。因为它拿到手的就是成型的pojo。省略了很多response的信息

- postForEntity

  - ```java
    // httpEntity
    HttpEntity<MultiValueMap<String, String>> request = new HttpEntity<>(map, headers);
    ResponseEntity<String> response = restTemplate.postForEntity( url, request , String.class );
    
    // MultiValueMap是Map的一个子类，它的一个key可以存储多个value
    
    // 为什么用MultiValueMap?因为HttpEntity接受的request类型是它
    
    // 为什么用HttpEntity是因为restTemplate.postForEntity方法虽然表面上接收的request是@Nullable Object request类型，但是你追踪下去会发现这个request是用HttpEntity来解析
    ```

- 使用exchange指定调用方式

  ```java
  HttpEntity<String> entity = new HttpEntity<>(jsonObj.toString(), headers);
  ResponseEntity<JSONObject> exchange = restTemplate.exchange(url,HttpMethod.GET, entity, JSONObject.class);
  ```





## 在浏览器输入 URL 回车之后发生了什么

- 大致流程
  - URL 解析
  - DNS 查询
  - TCP 连接
  - 处理请求
  - 接受响应
  - 渲染页面







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
  -  https
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





## jwt session

- http 无状态，所以为了实现有状态 http，才有了会话（session）的概念。
  - 会话(Session)是一个客户与服务器之间的不中断的请求响应序列。对客户的每个请求，服务器能够识别出请求来自于同一个客户。当一个未知的客户向Web应用程序发送第一个请求时就开始了一个会话。当客户明确结束会话或服务器在一个预定义的时限内不从客户接受任何请求时，会话就结束了。当会话结束后，服务器就忘记了客户以及客户的请求。
- 不管是“session id”，还是所谓“token”(如 jwt)，其实都是会话的一种实现方式。形式上“session id”和“token”都是“字符串”，这个“字符串”可以是任意的编码，本质上都是 credential（会话凭证）。
- 在各种 session 方案中，你会发现实现细节都不一样，flask session, django session, spring session, jwt 等等。但万变不离其宗， credential 的客户端保存方式，credential 的传输方式和会话信息的保存方式，都只是这几个流程的细节有改变而已，本质都是为了实现有状态的 http。







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

  