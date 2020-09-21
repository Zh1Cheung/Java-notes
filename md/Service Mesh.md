# 架构

- 1.5版本
  - 重建控制平面
    - 整合为istiod
    - 废弃Mixer



# 核心功能

## 流量控制

- 主要功能
  - 路由、流量转移
  - 流量进出
  - 网络弹性能力
    - 超时、重试、熔断
  - 测试相关
    - 故障注入、流量镜像（把生产环境的流量镜像一份然后发送到你的镜像服务中）
- 核心资源（CRD）
  - 虚拟服务（Virtual Service）
    - 将流量路由到给定目标地址（本质上来讲就是一组路由规则）
    - 包含一组路有规则
    - 通常和目标规划成对出现
    - 丰富的路有匹配规则
  - 目标规则（Destination Rule）
    - 定义虚拟服务路由目标地址的真实地址，即子集
    - 设置负载均衡的方式
      - 随机
      - 权重
      - 最少请求数
  - 网关(Gateway)
    - 虚拟服务和目标规则主要是用来管理网格内部的流量
    - 网关主要是用来管理网格以外的流量
      - 管理进出网格的流量
        - 管理入流量的网关叫Ingress
        - 管理出流量的网关叫Egress
      - 处在网格边界
  - 服务入口（Service Entry）
    - 前三个是面向流量的，这个是面向服务。
    - 把外部服务注册到网格中，这样你就可以像管理网格内到服务一样去管理外部的服务。
    - 功能
      - 为外部目标转发请求
      - 添加超时重试等策略
      - 扩展网格
    - 例子
      - 把不同的集群集中起来共同用一个网格进行管理
  - SideCar
    - 对流量进行一个全局的控制
      - 调整Envoy代理接管的端口和协议（默认Envoy是接管所有发送到服务的请求的）
      - 限制Envoy代理可访问的服务



## 可观察性

- 从开发者的角度探究系统的状态
- 组成：指标、日志、追踪
- 指标
  - 以聚合的方式监控和理解系统行为
  - Istio中的指标分类
    - 代理级别
      - 收集SideCar代理上的一些数据
    - 服务级别
      - 收集服务本身的一些信息
        - 延迟、流量、错误、饱和
        - 默认指标导出到Prometheus
    - 控制平面
      - 监控Istio本身的运行情况
- 日志
  - 通过应用产生的事件来了解系统
  - 包括了完整的元数据信息（目标、源）
  - 生成位置可选（本地、远端，如ELK）
  - 日志内容
    - 应用日志
    - Envoy日志
- 分布式追踪
  - 通过追踪请求，了解服务的调用关系



## 安全架构

- 绝大部分功能都是由Istiod这个控制平面来实现的

  - CA主要是用来负责证书的管理，认证策略和授权策略都会被存储在对应的模块中，由API server将这些策略变成配置，下发到对应的数据平面中。当一个终端用户调用网格内的服务的时候，他可以使用经典的JWT方式进行认证。而网格内服务与服务之间，可以使用双向的TLS进行认证。

- 认证

  - 认证方式

    - 对等认证

      - 用于服务间身份认证
      - 设置mTLS
        - 先使用兼容模式，调试完成之后再使用严格模式

    - 身份 认证

      - 用于终端用户身份认证
      - JWT
        - 可以把新旧JWT同x时配置到策略中，当请求全部迁移到新的JWT里面，再把旧的删除就可以了。

    - 对于不同的认证策略需要建立不同的CRD

      - 下面例子中的认证策略要求：与带有 `app:reviews` 标签的工作负载的传输层认证，必须使用双向 TLS

      - ```yaml
        apiVersion: "security.istio.io/v1beta1"
        kind: "PeerAuthentication"
        metadata:
          name: "example-peer-policy"
          namespace: "foo"
        spec:
          selector:
            matchLabels:
              app: reviews
          mtls:
            mode: STRICT
        
        ```

- 授权

  - 当管理员增加一个授权策略的时候，API server会把策略转换成对应的配置，下发给对应数据平面的SideCar，每个SideCar会运行一个授权引擎，该引擎在运行的时候进行授权请求，如果请求到达代理时，授权引擎根据当前的策略评估请求的上下文，并返回授权的结果。

  - 授权策略

    要配置授权策略，请创建一个 [`AuthorizationPolicy` 自定义资源](https://istio.io/latest/zh/docs/reference/config/security/authorization-policy/)。 一个授权策略包括选择器（selector），动作（action） 和一个规则（rules）列表：

    - `selector` 字段指定策略的目标

    - `action` 字段指定允许还是拒绝请求

    - rules指定何时触发动作

      - `rules` 下的 `from` 字段指定请求的来源
      - `rules` 下的 `to` 字段指定请求的操作
      - `rules` 下的 `when` 字段指定应用规则所需的条件

    - 以下示例显示了一个授权策略，该策略允许两个源（服务帐号 `cluster.local/ns/default/sa/sleep` 和命名空间 `dev`），在使用有效的 JWT 令牌发送请求时，可以访问命名空间 foo 中的带有标签 `app: httpbin` 和 `version: v1` 的工作负载。

    - ```yaml
      apiVersion: security.istio.io/v1beta1
      kind: AuthorizationPolicy
      metadata:
       name: httpbin
       namespace: foo
      spec:
       selector:
         matchLabels:
           app: httpbin
           version: v1
       action: ALLOW
       rules:
       - from:
         - source:
             principals: ["cluster.local/ns/default/sa/sleep"]
         - source:
             namespaces: ["dev"]
         to:
         - operation:
             methods: ["GET"]
         when:
         - key: request.auth.claims[iss]
           values: ["https://accounts.google.com"]
      
      ```





# 实例

## 动态路由

- 已定义的路由

  - 定义了四个虚拟服务，在这种情况下，virtual service 将所有流量路由到每个微服务的 `v1` 版本。

  - ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: details
      ...
    spec:
      hosts:
      - details
      http:
      # 具体的路由匹配规则
      # match：什么样的对象可以被接受
      # route：路由到什么位置 
      - route:
        - destination:
            host: details
            subset: v1
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: productpage
      ...
    spec:
      gateways:
      - bookinfo-gateway
      - mesh
      hosts:
      - productpage
      http:
      - route:
        - destination:
            host: productpage
            subset: v1
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: ratings
      ...
    spec:
      hosts:
      - ratings
      http:
      - route:
        - destination:
            host: ratings
            subset: v1
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: reviews
      ...
    spec:
      hosts:
      - reviews
      http:
      - route:
        - destination:
            host: reviews
            subset: v1
    ---
    ```

- Virtual Service和 Destination Rule的应用场景

  - 按服务版本路由
  - 按比例切分流量
  - 根据匹配规则进行路由
  - 定义各种策略（负载均衡、连接池等）



## 网关

- 特性

  - 一个运行在网格边缘的负载均衡器
  - 按照外部请求，转发给网格内的服务
  - 配置对外的端口、协议与内部服务的映射关系

- 创建网关

  - ss

    - ```yaml
      apiVersion: networking.istio.io/v1alpha3
      kind: Gateway
      metadata:
        name: httpbin-gateway
      spec:
        selector:
          istio: ingressgateway # use Istio default gateway implementation
        # servers：定义一个入口点
        # selector：选择一个现有的网关的pod的标签
        servers:
        - port:
        		# 端口、名称、协议
            number: 80
            name: http
            protocol: HTTP
          hosts:
          - "*"
      ---
      apiVersion: networking.istio.io/v1alpha3
      kind: VirtualService
      metadata:
        name: httpbin
      spec:
        hosts:
        - "*"
        # 在VS里使用
        gateways:
        - httpbin-gateway
        http:
        - match:
          - uri:
              prefix: /headers
          route:
          - destination:
              port:
                number: 8000
              host: httpbin
      ```

- Gateway的应用场景
  - 暴露网格内服务给外界访问
  - 访问安全（mTLS、HTTPS）
  - 统一应用入口，API聚合



## 服务入口

- 特性

  - 添加外部服务到网格内
  - 管理到外部服务的请求
  - 扩展网格

- 下面示例的 mesh-external 服务入口将 `ext-resource` 外部依赖项添加到 Istio 的服务注册中心：

  - ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: ServiceEntry
    metadata:
      name: svc-entry
    spec:
    	# hosts：外部服务的域名
      hosts:
      - ext-svc.example.com
      # ports：具体的访问这个服务的端口以及协议
      ports:
      - number: 443
        name: https
        protocol: HTTPS
      # 定义网格内部还是外部
      location: MESH_EXTERNAL
      # 服务发现的机制
      resolution: DNS
    
    ```

    



## 流量转移

- 特性（应用场景）

  - 蓝绿部署
  - 灰度发布（金丝雀发布）
  - A/B测试

- 将请求按比例路由到对应的服务版本

  - ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: reviews
      ...
    spec:
      hosts:
      - reviews
      http:
      - route:
        - destination:
            host: reviews
            subset: v1
          # weight：权重
          weight: 50
        - destination:
            host: reviews
            subset: v3
          weight: 50
    ```

    



## Ingress

- 概念

  - 服务的访问入口，接受外部请求并转发到后端服务

-  Istio 的 Ingress gateway 和 Kubernetes Ingress 的区别

  - Kubernetes: 针对L7协议（只支持HTTP协议，资源受限），可定义路由规则
  - Istio: 针对 L4-6 协议，只定义接入点，复用 Virtual Service 的 L7 路由定义（把所有的路由规则全部交给VirtualService），此时Virtual Service与其他的服务也是解耦的。

- 为httpbin服务配置Ingress网关

  - ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: httpbin-gateway
    spec:
      selector:
        istio: ingressgateway
      servers:
      - port:
          number: 80
          name: http
          protocol: HTTP
        hosts:
        - "httpbin.example.com"
    
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: httpbin
    spec:
    	# 要跟网关里配置的hosts一致
      hosts:
      - "httpbin.example.com"
      gateways:
      - httpbin-gateway
      http:
      - match:
        - uri:
            prefix: /status
        - uri:
            prefix: /delay
        route:
        - destination:
            port:
              number: 8000
            host: httpbin
    
    ```

    

## Egress

- 访问外部服务的方法
  - 配置 global.outboundTrafficPolicy.mode = ALLOW_ANY
  - 使用服务入口（ServiceEntry）
  - 配置sidecar让流量绕过代理
  - 配置Egress网关
- 概念
  - 定义了网格的出口点，允许你将监控、路由等功能应用于离开网格的流量
- 应用场景
  - 所有出口流量必须流经一组专用节点（安全因素） 
  - 为无法访问公网的内部服务做代理

- 创建一个 Egress 网关，让内部服务通过它访问外部服务

  - ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: istio-egressgateway
    spec:
      selector:
        istio: egressgateway
      servers:
      - port:
          number: 80
          name: http
          protocol: HTTP
        hosts:
        - httpbin.org
    
    ---
    
    apiVersion: networking.istio.io/v1alpha3
    kind: ServiceEntry
    metadata:
      name: httpbin
    spec:
      hosts:
      - httpbin.org
      ports:
      - number: 80
        name: http-port
        protocol: HTTP
      resolution: DNS
    
    
    ---
    
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: vs-for-egressgateway
    spec:
      hosts:
      - httpbin.org
      gateways:
      - istio-egressgateway
      - mesh
      http:
      - match:
      	# 匹配网关
        - gateways:
          - mesh
          port: 80
        route:
        - destination:
            host: istio-egressgateway.istio-system.svc.cluster.local
            subset: httpbin
            port:
              number: 80
          weight: 100
      - match:
        - gateways:
          - istio-egressgateway
          port: 80
        route:
        - destination:
            host: httpbin.org
            port:
              number: 80
          weight: 100
          
          
    ---
    
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: dr-for-egressgateway
    spec:
      host: istio-egressgateway.istio-system.svc.cluster.local
      subsets:
      - name: httpbin
    
    ```

    



## 超时重试

- 超时

  - 控制故障范围，避免故障扩散 

- 重试

  - 解决网络抖动时通信失败的问题

- 添加超时重试策略

  - 给 ratings 服务添加延迟

    - ```yaml
      apiVersion: networking.istio.io/v1alpha3
      kind: VirtualService
      metadata:
        name: ratings
      spec:
        hosts:
        - ratings
        http:
        - fault:
            delay:
              percent: 100
              # 给对 ratings 服务的调用添加 2 秒的延迟
              fixedDelay: 2s
          route:
          - destination:
              host: ratings
              subset: v1
      
      ```

  - 给 reviews 服务添加超时策略

    - ```yaml
      apiVersion: networking.istio.io/v1alpha3
      kind: VirtualService
      metadata:
        name: reviews
      spec:
        hosts:
        - reviews
        http:
        - route:
          - destination:
              host: reviews
              subset: v2
          # 给对 reviews 服务的调用增加一个1秒的请求超时
          timeout: 1s
      
      ```

  - 给 ratings 服务添加重试策略

    - ```yaml
      apiVersion: networking.istio.io/v1alpha3
      kind: VirtualService
      metadata:
        name: ratings
      spec:
        hosts:
        - ratings
        http:
        - fault:
            delay:
              percent: 100
              fixedDelay: 5s
          route:
          - destination:
              host: ratings
              subset: v1
          retries:
            # 尝试次数：2
            attempts: 2
            # 间隔超时：1s
            perTryTimeout: 1s
      
      ```

      

## 熔断

- 一种过载保护的手段 
  - 目的：避免服务的级联失败
  - 关键点：三个状态（close、open、halfopen）、失败计数器（阈值）、超时时钟

- 在服务的 DestinationRule 中添加熔断设置（创建一个目标规则，在调用 `httpbin` 服务时应用熔断设置）

  - ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: DestinationRule
    metadata:
      name: httpbin
    spec:
      host: httpbin
      trafficPolicy:
      	# ConnectionPool可以对上游服务的并发连接数和请求数进行限制，适用于TCP和HTTP。ConnectionPool又称之是限流。
        connectionPool:
        # TCP 和 HTTP 连接池大小为 1 
        # 只容许出错 2 次 
        # 每秒 1 次请求计数
    		# 可以从负载池中移除全部 pod
    		# 发生故障的 pod 移除 3m 之后才能再次加入
          tcp:
          	# maxConnections：到目标主机的HTTP1/TCP最大连接数量，只作用于http1.1，不作用于http2，因为后者只建立一次连接。
            maxConnections: 1
          # http连接池设置用于http1.1/HTTP2/GRPC连接。
    			# http1MaxPendingRequests：http请求pending状态的最大请求数，从应用容器发来的HTTP请求的最大等待转发数，默认是1024。
    			# maxRequestsPerConnection：在一定时间内限制对后端服务发起的最大请求数，如果超过了这个限制，就会开启限流。如果将这一参数设置为 1 则会禁止 keepalive 特性；
          http:
            http1MaxPendingRequests: 1
            maxRequestsPerConnection: 1
        # consecutiveErrors：从连接池开始拒绝连接，已经连接失败的次数。
        # Interval：拒绝访问扫描的时间间隔，即在interval（1s）内连续发生1个consecutiveErrors错误，则触发服务熔断，格式是1h/1m/1s/1ms
        # baseEjectionTime：最短拒绝访问时长。
        # maxEjectionPercent：服务在负载均衡池中被拒绝访问（被移除）的最大百分比
        # 设置的参数如下，该配置表示每秒钟扫描一次上游主机，连续失败1 次返回 5xx 错误码的所有主机会被移出连接池 3 分钟。
        outlierDetection:
          consecutiveErrors: 2
          interval: 1s
          baseEjectionTime: 3m
          maxEjectionPercent: 100
    
    ```

    

## 故障注入

-  ratings 服务添加一个中止（abort）故障

  - ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: ratings
      ...
    spec:
      hosts:
      - ratings
      http:
      - fault:
          abort:
            httpStatus: 500
            percentage:
              value: 100
        match:
        - headers:
            end-user:
              exact: jason
        route:
        - destination:
            host: ratings
            subset: v1
      - route:
        - destination:
            host: ratings
            subset: v1
    ```

  -  ratings 服务添加一个延迟故障

    - ```yaml
      apiVersion: networking.istio.io/v1alpha3
      kind: VirtualService
      metadata:
        name: ratings
        ...
      spec:
        hosts:
        - ratings
        http:
        - fault:
            delay:
              # 7s延迟
              fixedDelay: 7s
              percentage:
                value: 100
          match:
          - headers:
              end-user:
                exact: jason
          route:
          - destination:
              host: ratings
              subset: v1
        - route:
          - destination:
              host: ratings
              subset: v1
      ```

      



## 流量镜像

- 实时复制请求到镜像服务

- 应用场景

  - 线上问题排查（troubleshooting） 
  - 观察生产环境的请求处理能力（压力测试）
  - 复制请求信息用于分析

- 将发送到 v1 版本的流量镜像到 v2 版本

  - ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: httpbin
    spec:
      hosts:
        - httpbin
      http:
      - route:
        - destination:
            host: httpbin
            subset: v1
          weight: 100
        # 镜像目标地址
        mirror:
          host: httpbin
          subset: v2
        # 百分比
        mirror_percentage: 
          value: 100
    
    ```

    

# 可观察性

## Kiali 观测你的微服务应用

- 官方定义：
  - Istio 的可观察性控制台
  - 通过服务拓扑帮助你理解服务网格的结构
  - 提供网格的健康状态视图
  - 具有服务网格配置功能
- 优势
  - 梳理服务的交互关系
  - 了解应用的行为与状态



## Prometheus 收集指标

- Istio 1.5 的遥测指标

  > • 请求数（istio_requests_total） 
  >
  > • 请求时长（istio_request_duration_milliseconds） 
  >
  > • 请求大小（istio_request_bytes） 
  >
  > • 响应大小（istio_response_bytes） 
  >
  > • TCP 发送字节数（istio_tcp_sent_bytes_total） 
  >
  > • TCP 接受字节数（istio_tcp_received_bytes_total） 
  >
  > • TCP 连接打开数（istio_tcp_connections_opened_total） 
  >
  > • TCP 连接关闭数（istio_tcp_connections_closed_total）

  

## Grafana 查看系统

- Istio Dashboard
  - Mesh Dashboard：查看应用（服务）数据
    - 网格数据总览
    - 服务视图
    - 工作负载视图
  - Performance Dashboard：查看 Istio 自身（各组件）数据
    - Istio 系统总览
    - 各组件负载情况



## 获取 Envoy 的日志

- Envoy 流量五元组

  - A->Envoy->B
    - A->Envoy是下游
    - Envoy->B是上游
  - ![img](https://mmbiz.qpic.cn/mmbiz_jpg/uH1hdj5dlcP4ugXdJGo8NgYmBuwRKCC1oicGLDSH9olOvGAoDNWP8VaaymiavGIbv80j4fAFYdavyZbVs6knePNg/640?wx_fmt=jpeg)
  - ![img](https://mmbiz.qpic.cn/mmbiz_jpg/uH1hdj5dlcP4ugXdJGo8NgYmBuwRKCC17uthYuUmWzav0WibMuiaox5eaHbFErXU62zPfxTuyVpwPtdNDzyVic2Ig/640?wx_fmt=jpeg)
  - 这是 envoy 日志里最重要的部分，通过这个五元组我们可以准确的观测流量「从哪里来」和「到哪里去」。
    - UPSTREAM_CLUSTER：目的主机集合
    - DOWNSTREAM_REMOTE_ADDRESS：下游连接中远端地址
    - DOWNSTREAM_LOCAL_ADDRESS：下游连接中，当前 envoy 的本地地址。
    - UPSTREAM_LOCAL_ADDRESS：上游连接中，当前 envoy 的本地地址，此值是「当前pod-ip : 随机端口」
    - UPSTREAM_HOST：上游主机的 host，表示从 envoy 发出的请求的目的端，通常是「ip:port」

- 调试关键字段：RESPONSE_FLAGS

  > • UH：upstream cluster 中没有健康的 host，503
  >
  > • UF：upstream 连接失败，503
  >
  > • UO：upstream overflow（熔断） 
  >
  > • NR：没有路由配置，404
  >
  > • URX：请求被拒绝因为限流或最大连接次数



## Jaeger 对应用进行分布式追踪

- 概念
  - 开源、端到端的分布式追踪系统
  - 针对复杂的分布式系统，对业务链路进行监控和问题排查

- 术语
  - **Span：** 
    - 逻辑单元
    - 有操作名、执行时间
    - 嵌套、有序、因果关系
  - **Trace：** 
    - 数据/执行路径
    - Span 的组合



## 小结

- Metrics
- Logging
- Tracing

# 安全

## 单向TLS

- 公钥基础设施 (PKI)
  - Istio PKI 使用 X.509 证书为每个工作负载都提供强大的身份标识。可以大规模进行自动化密钥和证书轮换，伴随每个 Envoy 代理都运行着一个 `istio-agent` 负责证书和密钥的供应。
- Istio 供应身份是通过 secret discovery service（SDS）来实现的，具体流程如下
  - CA 提供 gRPC 服务以接受[证书签名请求](https://en.wikipedia.org/wiki/Certificate_signing_request)（CSRs）。
  - Envoy 通过 Envoy 秘密发现服务（SDS）API 发送证书和密钥请求。
  - 在收到 SDS 请求后，`istio-agent` 创建私钥和 CSR，然后将 CSR 及其凭据发送到 Istio CA 进行签名。
  - CA 验证 CSR 中携带的凭据并签署 CSR 以生成证书。
  - `Istio-agent` 通过 Envoy SDS API 将私钥和从 Istio CA 收到的证书发送给 Envoy。
  - 上述 CSR 过程会周期性地重复，以处理证书和密钥轮换。
- 安全发现服务（SDS） 
  - 身份和证书管理
    - 实现安全配置自动化
    - 中心化 SDS Server
  - 优点：
    - 无需挂载 secret 卷 
    - 动态更新证书，无需重启
    - 可监视多个证书密钥对

- 配置 TLS 安全网关

  - 生成证书、密钥
  - 创建secret
  - 配置 TLS 网关和路由

- 配置安全网关，为外部提供 HTTPS 访问方式

  - ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: mygateway
    spec:
      selector:
        istio: ingressgateway 
      servers:
      - port:
          number: 443
          name: https
          protocol: HTTPS
        tls:
        	# 单向TLS
          mode: SIMPLE
          # 刚才建立的secret
          credentialName: httpbin-credential
        hosts:
        - httpbin.example.com
    ```

- 路由

  - Service
  - Virtual Service



## 双向TLS

- 认证策略的分类
  - 对等认证（PeerAuthentication）
  - 请求认证（RequestAuthentication) 
- 认证策略范围
  - 网格
  - 命名空间 
  - 特定服务

- 为网格内的服务开启自动 mTLS

  - ```yaml
    # 给default添加命名空间策略
    
    # 兼容模式
    apiVersion: "security.istio.io/v1beta1"
    kind: "PeerAuthentication"
    metadata:
      name: "default"
      namespace: "default"
    spec:
      mtls:
        mode: PERMISSIVE
    
    ---
    
    
    # 严格模式
    apiVersion: "security.istio.io/v1beta1"
    kind: "PeerAuthentication"
    metadata:
      name: "default"
      namespace: "default"
    spec:
      mtls:
      	# 开启双向TLS
        mode: STRICT
    ```

- TLS：客户端根据服务端证书验证其身份
- mTLS：客户端、服务端彼此都验证对方身份
  - 当一个工作负载使用双向 TLS 认证向另一个工作负载发送请求时，该请求的处理方式如下：
    1. Istio 将出站流量从客户端重新路由到客户端的本地 sidecar Envoy。
    2. 客户端 Envoy 与服务器端 Envoy 开始双向 TLS 握手。在握手期间，客户端 Envoy 还做了[安全命名](https://istio.io/latest/zh/docs/concepts/security/#secure-naming)检查，以验证服务器证书中显示的服务帐户是否被授权运行目标服务。
    3. 客户端 Envoy 和服务器端 Envoy 建立了一个双向的 TLS 连接，Istio 将流量从客户端 Envoy 转发到服务器端 Envoy。
    4. 授权后，服务器端 Envoy 通过本地 TCP 连接将流量转发到服务器服务。



## JWT 身份认证与授权

- 实现基于 JWT 的授权访问（配置 JWT 的认证与授权）

  - ```yaml
    #创建请求认证
    apiVersion: "security.istio.io/v1beta1"
    kind: "RequestAuthentication"
    metadata:
      name: "jwt-example"
      namespace: testjwt
    spec:
      selector:
      	# 选择到的服务会需要进行请求认证
        matchLabels:
          app: httpbin
      jwtRules:
      	# JWT的签发人
      - issuer: "testing@secure.istio.io"
      	# 配置JWT公钥的url
        jwksUri: "https://raw.githubusercontent.com/malphi/geektime-servicemesh/master/c3-19/jwks.json"
        
    ----
    
    # 创建授权策略
    apiVersion: security.istio.io/v1beta1
    kind: AuthorizationPolicy
    metadata:
      name: require-jwt
      namespace: testjwt
    spec:
      selector:
        matchLabels:
          app: httpbin
      # 允许or拒绝
      action: ALLOW
      rules:
      # 来源
      - from:
        - source:
        	 # 具体来源的身份列表
           requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
    
    ```

    

## 安全功能总结

- 功能
  - 认证（authentication）：you are who you say you are
  - 授权（authorization）：you can do what you want to do
  - 验证（validation）：input is correct（对外界的输入进行验证）
- Istio 的安全机制
  - 透明的安全层
  - CA：密钥和证书管理
  - API Server：认证、授权策略分发
  - Envoy：服务间安全通信（认证、加密）





# 运维

## 构建和发布应用 - Flux 

- 自动化部署工具（基于 GitOps） 

  - GitOps是一种持续交付的方式。它的核心思想是将应用系统的声明性基础架构和应用程序存放在Git版本库中。
  - 将Git作为交付流水线的核心，每个开发人员都可以提交拉取请求（Pull Request）并使用Git来加速和简化Kubernetes的应用程序部署和运维任务。通过使用像Git这样的简单熟悉工具，开发人员可以更高效地将注意力集中在创建新功能而不是运维相关任务上（例如，应用系统安装、配置、迁移等）。

- ### GitOps流水线

  ![img](https://choerodon.io/blog/img/gitOps/assembly-line.png) 这是一个新图，显示部署上游的所有内容都围绕Git库工作的。在“拉式流水线”中讲过，开发人员将更新的代码推送到Git代码库，CI工具获取更改并最终构建Docker镜像。**GitOps的Config Update检测到有镜像**，从存储库中提取新镜像，然后在Git配置仓库中更新其YAML。然后，**GitOps的Deploy Operator会检测到群集已过期**，并从配置库中提取已更改的清单，并将新镜像部署到群集。

- 特性

  - 自动同步、自动部署
  - 声明式 
  - 基于代码（Pull request），而不是容器



## 自动化灰度发布 - Flagger

- 灰度配置

  - ```yaml
    
    # 创建 canary 分析
    apiVersion: flagger.app/v1beta1
    kind: Canary
    metadata:
      name: httpbin
      namespace: demo
    spec:
      # 指定deployment
      targetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: httpbin
      # 出现失败时回滚的一个间隔时间
      progressDeadlineSeconds: 60
      # HPA reference (optional)
      autoscalerRef:
        apiVersion: autoscaling/v2beta1
        kind: HorizontalPodAutoscaler
        name: httpbin
      # Virtual Service的配置
      service:
        # service port number
        port: 8000
        # container port number or name (optional)
        targetPort: 80
        # 配置gateways
        gateways:
        - public-gateway.istio-system.svc.cluster.local
      # 进行金丝雀发布的一个分析过程
      analysis:
        # 每30s进行一次路由的切换
        interval: 30s
        # 出现错误的次数，超过会进行回滚操作
        threshold: 5
        # 最大权重的转移比例
        maxWeight: 100
        # 每次转移20%的流量
        stepWeight: 20
        metrics:
        # 请求的成功率
        - name: request-success-rate
          # 最少成功率
          thresholdRange:
            min: 99
          interval: 1m
          # 请求的延迟
        - name: latency
          templateRef:
            name: latency
            namespace: istio-system
          # 大于500ms就进行回滚操作
          thresholdRange:
            max: 500
          interval: 30s
        # load-test工具
        webhooks:
          - name: load-test
            url: http://flagger-loadtester.test/
            timeout: 5s
            metadata:
              cmd: "hey -z 1m -q 10 -c 2 http://httpbin-canary.demo:8000/headers"
    
    ```

    

## 弹性能力

- 应对故障的一种方法，让系统具有容错和适应能力
  - 防止故障（Fault）转化为失败（Failure） 
  - 主要包括： 
    - 容错性：重试、幂等
    - 伸缩性：自动水平扩展（autoscaling） 
    - 过载保护：超时、熔断、降级、限流
    - 弹性测试：故障注入
- Istio 的弹性能力
- 超时、重试、熔断、故障注入



## 安全策略

- 微服务也有特殊的安全需求：
  - 为了抵御中间人攻击，需要流量加密。
  - 为了提供灵活的服务访问控制，需要双向 TLS 和细粒度的访问策略。
  - 要确定谁在什么时候做了什么，需要审计工具。

- Istio 安全功能提供强大的身份，强大的策略，透明的 TLS 加密，认证，授权和审计（AAA）工具来保护你的服务和数据。Istio 安全的目标是：
  - 默认安全：应用程序代码和基础设施无需更改
  - 深度防御：与现有安全系统集成以提供多层防御
  - 零信任网络：在不受信任的网络上构建安全解决方案
- Istio 中的安全性涉及多个组件：
  - 用于密钥和证书管理的证书颁发机构（CA）
  - 配置 API 服务器分发给代理：
    - [认证策略](https://istio.io/latest/zh/docs/concepts/security/#authentication-policies)
    - [授权策略](https://istio.io/latest/zh/docs/concepts/security/#authorization-policies)
    - [安全命名信息](https://istio.io/latest/zh/docs/concepts/security/#secure-naming)
  - Sidecar 和边缘代理作为 [Policy Enforcement Points](https://www.jerichosystems.com/technology/glossaryterms/policy_enforcement_point.html)(PEPs) 以保护客户端和服务器之间的通信安全.
  - 一组 Envoy 代理扩展，用于管理遥测和审计

   ![](https://istio.io/latest/zh/docs/concepts/security/arch-sec.svg)





## 收集指标、监控应用

- Prometheus 的服务发现机制
  - **kubernetes_sd_config - role**
    - node：集群节点
    - service：服务，常用于黑盒监控
    - pod：以pod中容器为目标 
    - endpoints：端点 
    - ingress：入口网关
- **relabel_configs** 过滤机制
  - 通过正则表达式进行过滤
- 整合 Grafana
  - 添加数据源 
  - 导入 Dashboard



## 集成 ELK Stack

- ELK Stack 日志架构
  - ElasticSearch
  - Logstash
  - Kibana
  - LibBeats

- 通过 ELK 完成 Envoy 日志的收集和检索

## 分布式追踪工具

























