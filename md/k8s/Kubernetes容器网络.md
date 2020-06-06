## 容器网络

-  Linux 容器能看见的“网络栈”，实际上是被隔离在它自己的 Network Namespace 当中的。
  - 所谓“网络栈”，就包括了：网卡（Network Interface）、回环设备（LoopbacDevice）、路由表（Routing Table）和 iptables 规则。对于一个进程来说，这些要素，其实就构成了它发起和响应网络请求的基本环境。
  - 器要想跟外界进行通信，它发出的 IP 包就必须从它的 NetworNamespace 里出来，来到宿主机上。
    - 解决这个问题的方法就是：为容器创建一个一端在容器里充当默认网卡、另一端在宿主机上的Veth Pair 设备。
- 这个被隔离的容器进程，该如何跟其他 Network Namespace 里的容器进程进行交互呢？
  - 如果你想要实现多台主机之间的通信，那就需要用网线，把它们连接在一台交换机上。在 Linux 中，能够起到虚拟交换机作用的网络设备，是网桥（Bridge）
  - 为了实现上述目的，Docker 项目会默认在宿主机上创建一个名叫 docker0 的网桥，凡是连接在 docker0 网桥上的容器，就可以通过它来进行通信。
- Veth Pair的虚拟设备
  - 如何把这些容器“连接”到 docker0 网桥上
  - 被创建出来后，总是以两张虚拟网卡（Veth Peer）的形式成对出现的。
  - 从其中一个“网卡”发出的数据包，可以直接出现在与它对应的另一张“网卡”上，哪怕这两个“网卡”在不同的 Network Namespace 里。
  - 因为这个容器里有一张叫作 eth0 的网卡，它正是一个 Veth Pair 设备在容器里的这一端，这个 Veth Pair 设备的另一端，则在宿主机上。所以同一宿主机上的两个容器默认就是相互连通
  - 一旦一张虚拟网卡被“插”在网桥上，它就会变成该网桥的“从设备”。从设备会被“剥夺”调用网络协议栈处理数据包的资格，从而“降级”成为网桥上的一个端口
- 如果我们通过软件的方式，创建一个整个集群“公用”的网桥，然后把集群里的所有容器都连接到这个网桥上，不就可以相互通信了吗？
  - 构建这种容器网络的核心在于：我们需要在已有的宿主机网络上，再通过软件构建一个覆盖在已有宿主机网络之上的、可以把所有容器连通在一起的虚拟网络。所以，这种技术就被称为：Overlay Network（覆盖网络）。
  - 这个 Overlay Network 本身，可以由每台宿主机上的一个“特殊网桥”共同组成







## 容器跨主机网络

- 要理解容器“跨主通信”的原理，就一定要先从 Flannel 这个项目说起。
  - 事实上，Flannel 项目本身只是一个框架，真正为我们提供容器网络功能的，是 Flannel 的后端实现。
  - 我们在进行系统级编程的时候，有一个非常重要的优化原则，就是要减少用户态到内核态的切换次数，并且把核心的处理逻辑都放在内核态进行。这也是为什么，Flannel 后来支持的VXLAN 模式，逐渐成为了主流的容器网络方案的原因。
  - VXLAN，即 Virtual Extensible LAN（虚拟可扩展局域网），是 Linux 内核本身就支持的一种网络虚似化技术。所以说，VXLAN 可以完全在内核态实现上述封装和解封装的工作，从而通过与前面相似的“隧道”机制，构建出覆盖网络（Overlay Network）。
- 设计思想
  - 为了能够在二层网络上打通“隧道”，VXLAN 会在宿主机上设置一个特殊的网络设备作为“隧道”的两端。这个设备就叫作 VTEP(虚拟隧道端点)
    - 这些 VTEP 设备之间，就需要想办法组成一个虚拟的二层网络，即：通过二层数据帧进行通信。
    - “源 VTEP 设备”收到“原始 IP 包”后，就要想办法把“原始 IP 包”加上一个目的 MAC 地址，封装成一个二层数据帧，然后发送给“目的 VTEP 设备”
      - Linux 内核还需要再把“内部数据帧”进一步封装成为宿主机网络里的一个普通的数据帧，好让它“载着”“内部数据帧”，通过宿主机的 eth0 网卡进行传输。
      - 然后，Linux 内核会把这个数据帧封装进一个 UDP 包里发出去。
    - 接下来的流程，就是一个正常的、宿主机网络上的封包工作。
  - VXLAN 模式组建的覆盖网络，其实就是一个由不同宿主机上的 VTEP 设备，也就是 flannel.1 设备组成的虚拟二层网络。对于 VTEP 设备来说，它发出的“内部数据帧”就仿佛是一直在这个虚拟的二层网络上流动。这，也正是覆盖网络的含义。







## Kubernetes网络模型与CNI网络插件

- 以 Flannel 项目为例，容器跨主机网络的两种实现方法：UDP 和 VXLAN

  - 这些例子有一个共性，那就是用户的容器都连接在 docker0 网桥上
    - 网络插件则在宿主机上创建了一个特殊的设备
    - 然后，网络插件真正要做的事情，则是通过某种方法，把不同宿主机上的特殊设备连通，从而达到容器跨主机通信的目的。
    - docker0 与这个设备之间，通过 IP 转发（路由表）进行协作。
  - Kubernetes 是通过一个叫作 CNI 的接口，维护了一个单独的网桥来代替 docker0。这个网桥的名字就叫作：CNI 网桥，它在宿主机上的设备名称默认是：cni0。

-  CNI 网桥

  - CNI 网桥只是接管所有 CNI 插件负责的、即 Kubernetes 创建的容器（Pod）。
  - Kubernetes 之所以要设置这样一个与 docker0 网桥功能几乎一样的 CNI 网桥，主要原因包括两个方面：
    - 一方面，Kubernetes 项目并没有使用 Docker 的网络模型（CNM）
    - 另一方面，这还与 Kubernetes 如何配置 Pod，也就是 Infra 容器的 Network Namespace密切相关。
  - CNI 的设计思想，就是：Kubernetes 在启动 Infra 容器之后，就可以直接调用 CNI 网络插件，为这个 Infra 容器的 Network Namespace，配置符合预期的网络栈。

-  CNI 插件的工作原理

  - 在 Kubernetes 中，处理容器网络相关的逻辑并不会在 kubelet 主干代码里执行，而是会在具体的 CRI（Container Runtime Interface，容器运行时接口）实现里完成。对于 Docker 项目来说，它的 CRI 实现叫作 dockershim

  - CNI bridge 插件

    - 功能：“将容器加入到CNI 网络里”

    - CNI bridge 插件会在宿主机上检查 CNI 网桥是否存在。如果没有的话，那就创建它

    - CNI bridge 插件会通过 Infra 容器的 Network Namespace 文件，进入到这个Network Namespace 里面，然后创建一对 Veth Pair 设备。

    - 紧接着，它会把这个 Veth Pair 的其中一端，“移动”到宿主机上。

    - 接下来，CNI bridge 插件会调用 CNI ipam 插件，为容器分配一个可用的 IP 地址。然后，CNI bridge 插件就会把这个 IP 地址添加在容器的 eth0 网卡上，同时为容器设置默认路由

      - > 在容器里
        >
        > $ ip addr add 10.244.0.2/24 dev eth0
        > $ ip route add default via 10.244.0.1 dev eth0

    - 最后，CNI bridge 插件会为 CNI 网桥添加 IP 地址。

      - >  在宿主机上
        >
        > $ ip addr add 10.244.0.1/24 dev cni0







## Service、DNS与服务发现

- Kubernetes 之所以需要 Service，一方面是因为 Pod 的 IP 不是固定的，另一方面则是因为一组 Pod 实例之间总会有负载均衡的需求。

  - > 这个 Service 的 80 端口，代理的是 Pod 的9376 端口
    > 	port: 80
    >  	 targetPort: 9376

- Kubernetes 里的 Service 究竟是如何工作的呢？

  - 实际上，Service 是由 kube-proxy 组件，加上 iptables 来共同实现的。

    - >  Service一旦它被提交给Kubernetes， kube-proxy 就可以通过 Service 的 Informer 感知到这样一个 Service 对象的添加，而作为对这个事件的响应，它就会在宿主机上创建这样一条 iptables 规则
      >
      > 
      >
      > 凡是目的地址是 10.0.1.175、目的端口是 80 的 IP包，都应该跳转到另外一条名叫 KUBE-SVC-NWV5X2332I4OT4T3

    - 实际上，它是一组规则的集合，这一组规则（ DNAT 规则），实际上是一组随机模式（–mode random）的 iptables 链。

      - 这条链指向的最终目的地，其实就是这个 Service 代理的Pod
      - DNAT 规则的作用，就是在路由之前，将流入 IP 包的目的地址和端口，改成–to-destination 所指定的新的目的地址和端口。可以看到，这个目的地址和端口，正是被代理 Pod 的 IP 地址和端口。

  - 所以这一组规则，就是 Service 实现负载均衡的位置。

-  IPVS

  - kube-proxy 通过 iptables 处理 Service 的过程，其实需要在宿主机上设置相当多的 iptables 规则。而且，kube-proxy 还需要在控制循环里不断地刷新这些规则来确保它们始终是正确的
    - 大量 Pod不断地被刷新，会大量占用该宿主机的 CPU 资源，基于iptables 的 Service 实现，都是制约 Kubernetes 项目承载更多量级的 Pod 的主要障碍。
    - IPVS 模式的 Service，就是解决这个问题的一个行之有效的方法。
  - IPVS 模式的工作原理，其实跟 iptables 模式类似。当我们创建了前面的 Service 之后，kube-proxy 首先会在宿主机上创建一个虚拟网卡（叫作：kube-ipvs0），并为它分配 Service VIP作为 IP 地址
    - kube-proxy 就会通过 Linux 的 IPVS 模块，为这个 IP 地址设置三个 IPVS 虚拟主机，并设置这三个虚拟主机之间使用轮询模式 (rr) 来作为负载均衡策略
    - 这三个 IPVS 虚拟主机的 IP 地址和端口，对应的正是三个被代理的 Pod。
    - 这时候，任何发往 10.102.128.4:80 的请求，就都会被 IPVS 模块转发到某一个后端 Pod 上了

- Service 与 DNS 的关系

  - 在 Kubernetes 中，Service 和 Pod 都会被分配对应的 DNS A 记录（从域名解析 IP 的记录）

  - ClusterIP 模式的 Service 为你提供的，就是一个 Pod 的稳定的 IP 地址，即 VIP。并且，这里 Pod 和 Service 的关系是可以通过 Label 确定的。

  - 而 Headless Service 为你提供的，则是一个 Pod 的稳定的 DNS 名字，并且，这个名字是可以

    通过 Pod 名字和 Service 名字拼接出来的。

- Service 机制，以及Kubernetes 里的 DNS 插件，都是在帮助你解决同样一个问题，即：如何找到我的某一个容器？

  - 这个问题在平台级项目中，往往就被称作服务发现，即：当我的一个服务（Pod）的 IP 地址是不固定的且没办法提前获知时，我该如何通过一个固定的方式访问到这个 Pod 呢？







## Service

- 所谓 Service 的访问入口，其实就是每台宿主机上由 kube-proxy 生成的iptables 规则，以及 kube-dns 生成的 DNS 记录。而一旦离开了这个集群，这些信息对用户来说，也就自然没有作用了。

  - 在使用 Kubernetes 的 Service 时，一个必须要面对和解决的问题就是：如何从外部（Kubernetes 集群之外），访问到 Kubernetes 里创建的 Service？

- 最常用的一种方式就是：NodePort

  - ```yaml
    spec:
     type: NodePort
     ports:
     - nodePort: 8080
       targetPort: 80
       protocol: TCP
       name: http
     - nodePort: 443
       protocol: TCP
       name: https
    ```

    

  - 需要注意的是，在 NodePort 方式下，Kubernetes 会在 IP 包离开宿主机发往目的 Pod 时，对这个 IP 包做一次 SNAT 操作

    - 将这个 IP 包的源地址替换成了这台宿主机上的 CNI 网桥地址，或者宿主机本身的 IP 地址（如果 CNI 网桥不存在的话）。

- 从外部访问 Service 的第二种方式，适用于公有云上的 Kubernetes 服务。这时候，你可以指定一个 LoadBalancer 类型的 Service

  - ```yaml
    spec:
     ports:
     - port: 8765
       targetPort: 9376
     selector:
      app: example
     type: LoadBalancer
    ```

- 第三种方式，是 Kubernetes 在 1.7 之后支持的一个新特性，叫作 ExternalName

  - ```yaml
    # 当你通过 Service 的 DNS 名字访问它的时候，比如访问：my-service.default.svc.cluster.local。那么，Kubernetes 为你返回的就是my.database.example.com。
    metadata:
     name: my-service
    spec:
     type: ExternalName
     externalName: my.database.example.com
     
     
     # Kubernetes 的 Service 还允许你为 Service 分配公有 IP 地址
     spec:
      selector:
       app: MyApp
      ports:
      - name: http
        protocol: TCP
        port: 80
        targetPort: 9376
        externalIPs:
         - 80.11.12.10
    ```

- 很多与 Service 相关的问题，其实都可以通过分析 Service 在宿主机上对应的 iptables 规则（或者 IPVS 配置）得到解决。

  - 比如，当你的 Service 没办法通过 DNS 访问到的时候。你就需要区分到底是 Service 本身的配置问题，还是集群的 DNS 出了问题。一个行之有效的方法，就是检查 Kubernetes 自己的Master 节点的 Service DNS 是否正常：

    - > 在一个 Pod 里执行
      >
      > $ nslookup kubernetes.default
      > 如果上面访问 kubernetes.default 返回的值都有问题，那你就需要检查 kube-dns 的运行状态和日志了。否则的话，你应该去检查自己的 Service 定义是不是有问题

  - 而如果你的 Service 没办法通过 ClusterIP 访问到的时候，你首先应该检查的是这个 Service 是否有 Endpoints

    - > $ kubectl get endpoints hostnames
      > NAME ENDPOINTS
      > hostnames 10.244.0.5:9376,10.244.0.6:9376,10.244.0.7:9376
      > 如果 Endpoints 正常，那么你就需要确认 kube-proxy 是否在正确运行
      > 如果 kube-proxy 一切正常，你就应该仔细查看宿主机上的 iptables 了。

  - 还有一种典型问题，就是 Pod 没办法通过 Service 访问到自己

    - 你只需要确保将 kubelet 的 hairpin-mode 设置为 hairpin-veth 或者promiscuous-bridge 即可。

- 所谓 Service，其实就是 Kubernetes 为 Pod 分配的固定的、基于iptables（或者 IPVS）的访问入口。而这些访问入口代理的 Pod 信息，则来自于 Etcd，由kube-proxy 通过控制循环来维护。

  - Kubernetes 里面的 Service 和 DNS 机制，也都不具备强多租户能力。比如，在多租户情况下，每个租户应该拥有一套独立的 Service 规则
  - 再比如 DNS，在多租户情况下，每个租户应该拥有自己的 kube-dns
  - 多租户是指软件架构支持一个实例服务多个用户（Customer），每一个用户被称之为租户（tenant），软件给予租户可以对系统进行部分定制的能力







## Service与Ingress

- 作为用户，我其实更希望看到 Kubernetes 为我内置一个全局的负载均衡器。然后，通过我访问的 URL，把请求转发给不同的后端 Service。
  - 这种全局的、为了代理不同后端 Service 而设置的负载均衡服务，就是 Kubernetes 里的Ingress 服务。
  - 所谓 Ingress，就是 Service 的“Service”
- 我如何能使用 Kubernetes 的 Ingress 来创建一个统一的负载均衡器，从而实现当用户访问不同的域名时，能够访问到不同的 Deployment 呢？
  - 上述功能，在 Kubernetes 里就需要通过 Ingress 对象来描述
  - IngressRule 的 Key，就叫做：host。是这个 Ingress 的入口
  - IngressRule 规则的定义，则依赖于 path 字段，你可以简单地理解为，这里的每一个path 都对应一个后端 Service。
- 所谓 Ingress 对象，其实就是 Kubernetes 项目对“反向代理”的一种抽象。
  - 实际的使用中，你只需要从社区里选择一个具体的 Ingress Controller，把它部署在Kubernetes 集群里即可。
  - 然后，这个 Ingress Controller 会根据你定义的 Ingress 对象，提供对应的代理能力。
- Ingress Controller
  -  Nginx 官方为你维护的 Ingress Controller 的定义的YAML 文件中定义了一个使用 nginx-ingress-controller 镜像的Pod。
    - 当一个新的 Ingress 对象由用户创建后，nginx-ingress-controller 就会根据 Ingress 对象里定义的内容，生成一份对应的 Nginx 配置文件（/etc/nginx/nginx.conf），并使用这个配置文件启动一个 Nginx 服务。
    - nginx-ingress-controller 还允许你通过 Kubernetes 的 ConfigMap 对象来对上述Nginx 配置文件进行定制
    - 一个 Nginx Ingress Controller 为你提供的服务，其实是一个可以根据 Ingress对象和被代理后端 Service 的变化，来自动进行更新的 Nginx 负载均衡器。
  - 为了让用户能够用到这个 Nginx，我们就需要创建一个 Service 来把 Nginx Ingress Controller 管理的 Nginx 服务暴露出去
- 我们就可以通过访问这个 Ingress 的地址和端口，访问到我们前面部署的应用了
  - 目前，Ingress 只能工作在七层，而 Service 只能工作在四层。所以当你想要在 Kubernetes 里为应用进行 TLS 配置等 HTTP 相关的操作时，都必须通过 Ingress 来进行。
  - Ingress 简单的理解就是你原来需要改 Nginx 配置，然后配置各种域名对应哪个 Service，现在把这个动作抽象出来，变成一个 Ingress 对象，你可以用 yaml 创建，每次不要去改 Nginx 了，直接改 yaml 然后创建/更新就行了；
  - 那么问题来了：”Nginx 该怎么处理？”Ingress Controller 这东西就是解决 “Nginx 的处理方式” 的；Ingress Controoler 通过与 Kubernetes API 交互，动态的去感知集群中 Ingress 规则变化，然后读取他，按照他自己模板生成一段 Nginx 配置，再写到 Nginx Pod 里，最后 reload 一下









