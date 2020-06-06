## Pod概念

- Pod，是 Kubernetes 项目中最小的 API 对象。如果换一个更专业的说法，我们可以这样描述：Pod，是 Kubernetes 项目的原子调度单位。
  - Pod 这个概念，提供的是一种编排思想，而不是具体的技术方案
  - 容器的“单进程模型”，并不是指容器里只能运行“一个”进程，而是指容器没有管理多个进程的能力
    - 因为容器里 PID=1 的进程就是应用本身，其他的进程都是这个PID=1 进程的子进程
    - 到了 Kubernetes 项目里，这样的问题就迎刃而解了：Pod 是 Kubernetes 里的原子调度单位。这就意味着，Kubernetes 项目的调度器，是统一按照 Pod 而非容器的资源需求进行计算的
    - Pod，实际上是在扮演传统基础设施里“虚拟机”的角色；而容器，则是这个虚拟机里运行的用户程序。
- Pod 的实现原理
  - 关于 Pod 最重要的一个事实是：它只是一个逻辑概念
    - 也就是说，Kubernetes 真正处理的，还是宿主机操作系统上 Linux 容器的 Namespace 和Cgroups，而并不存在一个所谓的 Pod 的边界或者隔离环境。
  - Pod 里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个Volume。
  - 在 Kubernetes 项目里，Pod 的实现需要使用一个中间容器，这个容器叫作 Infra 容器。在这个 Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起
- Pod 在 Kubernetes 项目里还有更重要的意义，那就是：容器设计模式。
  - 容器设计模式里最常用的一种模式，它的名字叫：sidecar。sidecar 指的就是我们可以在一个 Pod 中，启动一个辅助容器，来完成一些独立于主进程（主容器）之外的工作。
  - Pod 的另一个重要特性是，它的所有容器都共享同一个 Network Namespace。这就使得很多与 Pod 网络相关的配置和管理，也都可以交给 sidecar 完成，而完全无须干涉用户容器。这里最典型的例子莫过于 Istio 这个微服务治理项目了。







## Pod使用

- 到底哪些属性属于 Pod 对象，而又有哪些属性属于 Container 呢？

  - 凡是调度、网络、存储，以及安全相关的属性，基本上是 Pod 级别的。这些属性的共同特征是，它们描述的是“机器”这个整体，而不是里面运行的“程序”

-  Pod 中几个重要字段的含义和用法

  - NodeSelector：是一个供用户将 Pod 与 Node 进行绑定的字段
  - HostAliases：定义了 Pod 的 hosts 文件（比如 /etc/hosts）里的内容
  -  shareProcessNamespace=true，意味着这个 Pod 里的容器要共享 PID Namespace。
  - 凡是 Pod 中的容器要共享宿主机的 Namespace，也一定是 Pod 级别的定义
    - spec: hostNetwork: true hostIPC: true hostPID: true
    - 在这个 Pod 中，我定义了共享宿主机的 Network、IPC 和 PID Namespace。这就意味着，这个Pod 里的所有容器，会直接使用宿主机的网络、直接与宿主机进行 IPC 通信、看到宿主机里正在运行的所有进程。

- Pod 里最重要的字段当属“Containers”了

  -  Image（镜像）、Command（启动命令）、workingDir（容器的工作目录）、Ports（容器要开发的端口），以及 volumeMounts（容器要挂载的 Volume）都是构成 Kubernetes 项目中 Container 的主要字段。
  - ImagePullPolicy 字段。它定义了镜像拉取的策略。
    - ImagePullPolicy 的值默认是 Always，即每次创建 Pod 都重新拉取一次镜像
    - 而如果它的值被定义为 Never 或者 IfNotPresent，则意味着 Pod 永远不会主动拉取这个镜像，或者只在宿主机上不存在这个镜像时才拉取。
  -  Lifecycle 字段
    - 在容器状态发生变化时触发一系列“钩子”
    - postStart 。它指的是，在容器启动后，立刻执行一个指定的操作
    - preStop 发生的时机，则是容器被杀死之前（比如，收到了 SIGKILL 信号）。而需要明确的是，preStop 操作的执行，是同步的。所以，它会阻塞当前的容器杀死流程，直到这个 Hoo定义操作完成之后，才允许容器被杀死，这跟 postStart 不一样。

- Pod 对象在 Kubernetes 中的生命周期

  - Pod 生命周期的变化，主要体现在 Pod API 对象的Status 部分。这是它除了 Metadata 和 Spec之外的第三个重要字段。其中pod.status.phase，就是 Pod 的当前状态
    - Pending 这个 Pod 里有些容器因为某种原因而不能被顺利创建。比如，调度不成功。
    - Running Pod 已经调度成功
    - Succeeded Pod 里的所有容器都正常运行完毕，并且已经退出了
    - Failed Pod 里至少有一个容器以不正常的状态（非 0 的返回码）退出
    - Unknown 意味着 Pod 的状态不能持续地被 kubelet 汇报给 kubeapiserver，这很有可能是主从节点（Master 和 Kubelet）间的通信出现了问题。
  - 更进一步地，Pod 对象的 Status 字段，还可以再细分出一组 Conditions。
    - PodScheduled、Ready、Initialized，以及 Unschedulable
    - Ready 这个细分状态非常值得我们关注：它意味着 Pod 不仅已经正常启动（Running 状态），而且已经可以对外提供服务了。这两者之间（Running 和 Ready）是有区别的，你不妨仔细思考一下。

- Projected Volume，“投射数据卷”。

  - 是为容器提供预先定义好的数据。所以，从容器的角度来看，这些 Volume 里的信息就是仿佛是被 Kubernetes“投射”（Project）进入容器当中的
  - Secret；
  2. ConfigMap；
  - Downward API；
    3. 让 Pod 里的容器能够直接获取到这个 Pod API 对象本身的信息
    3. 需要注意的是，Downward API 能够获取到的信息，一定是 Pod 里的容器进程启动之前就能够确定下来的信息。而如果你想要获取 Pod 容器运行后才会出现的信息，比如，容器进程的PID，那就肯定不能使用 Downward API 了，而应该考虑在 Pod 里定义一个 sidecar 容器。
    3. 其实，Secret、ConfigMap，以及 Downward API 这三种 Projected Volume 定义的信息，大多还可以通过环境变量的方式出现在容器里。但是，通过环境变量获取这些信息的方式，不具备自动更新的能力。所以，一般情况下，我都建议你使用 Volume 文件的方式获取这些信息。
  - ServiceAccountToken
    4. Service Account 对象的作用，就是 Kubernetes 系统内置的一种“服务账户”，它是 Kubernetes进行权限分配的对象
    4.  Service Account 的授权信息和文件，实际上保存在它所绑定的一个特殊的 Secret 对象里的。这个特殊的 Secret 对象，就叫作ServiceAccountToken
    - 另外，为了方便使用，Kubernetes 已经为你提供了一个的默认“服务账户”（default ServiceAccount）。并且，任何一个运行在 Kubernetes 里的 Pod，都可以直接使用这个默认的 ServiceAccount，而无需显示地声明挂载它。
      4. 所以说，Kubernetes 其实在每个 Pod 创建的时候，自动在它的 spec.volumes 部分添加上了默认ServiceAccountToken 的定义，然后自动给每个容器加上了对应的 volumeMounts 字段

- 容器健康检查和恢复机制

  - 健康检查“探针”（Probe）
    4. kubelet 就会根据这个 Probe 的返回值决定这个容器的状态
  -  livenessProbe（健康检查）
    4. 在容器启动 5 s 后开始执行（initialDelaySeconds: 5）
    4. 每 5 s 执行一次（periodSeconds: 5）。

- Pod 恢复机制，也叫 restartPolicy

  - 但一定要强调的是，Pod 的恢复过程，永远都是发生在当前节点上

  - 而如果你想让 Pod 出现在其他的可用节点上，就必须使用 Deployment 这样的“控制器”来管理Pod，哪怕你只需要一个 Pod 副本。

  - 而作为用户，你还可以通过设置 restartPolicy，改变 Pod 的恢复策略。除了 Always，它还有OnFailure 和 Never 两种情况：

    - > Always：在任何情况下，只要容器不在运行状态，就自动重启容器；
      > OnFailure: 只在容器 异常时才自动重启容器；
      > Never: 从来不重启容器。

    - 比如，一个 Pod，它只计算 1+1=2，计算完成输出结果后退出，变成 Succeeded 状态。这时，你如果再用 restartPolicy=Always 强制重启这个 Pod 的容器，就没有任何意义了。







## “控制器”模型

- kube-controller-manager 的组件
  - 实际上，这个组件，就是一系列控制器的集合
  - 这些控制器之所以被统一放在 pkg/controller 目录下，就是因为它们都遵循 Kubernetes项目中的一个通用编排模式，即：控制循环（control loop）。
    - 在具体实现中，实际状态往往来自于 Kubernetes 集群本身。
    - 期望状态，一般来自于用户提交的 YAML 文件。
    - Deployment 控制器将两个状态做比较，然后根据比较结果，确定是创建 Pod，还是删除已有的 Pod
      - 被控制对象的定义，则来自于一个“模板”。比如，Deployment 里的 template 字段。
      - 所有被这个 Deployment 管理的 Pod 实例，其实都是根据这个 template 字段的内容创建出来的。
- “控制器模式”（controller pattern）的设计方法，来统一地实现对各种不同的对象或者资源进行的编排操作。
  - 在后面的讲解中，很多不同类型的容器编排功能，比如 StatefulSet、DaemonSet 等等，它们无一例外地都有这样一个甚至多个控制器的存在，并遵循控制循环（control loop）的流程，完成各自的编排逻辑。
  - 跟 Deployment 相似，这些控制循环最后的执行结果，要么就是创建、更新一些 Pod（或者其他的 API 对象、资源），要么就是删除一些已经存在的 Pod（或者其他的 API 对象、资源）。
  - 这个实现思路，正是 Kubernetes 项目进行容器编排的核心原理





## 作业副本与水平扩展

- Deployment 看似简单，但实际上，它实现了 Kubernetes 项目中一个非常重要的功能：Pod的“水平扩展 / 收缩”（horizontal scaling out/in）。这个功能，是从 PaaS 时代开始，一个平台级项目就必须具备的编排能力。
  - 如果你更新了 Deployment 的 Pod 模板（比如，修改了容器的镜像），那么Deployment 就需要遵循一种叫作“滚动更新”（rolling update）的方式，来升级现有的容器。
  - 而这个能力的实现，依赖的是 Kubernetes 项目中的一个非常重要的概念（API 对象）：ReplicaSet。
    - 一个 ReplicaSet 对象，其实就是由副本数目的定义和一个 Pod 模板组成的。不难发现，它的定义其实是 Deployment 的一个子集。
    - 更重要的是，Deployment 控制器实际操纵的，正是这样的 ReplicaSet 对象，而不是 Pod 对象。
- Deployment 同样通过“控制器模式”，来操作 ReplicaSet 的个数和属性，进而实现“水平扩展 / 收缩”和“滚动更新”这两个编排动作。
  - “水平扩展 / 收缩”非常容易实现，Deployment Controller 只需要修改它所控制的ReplicaSet 的 Pod 副本个数就可以了。
  - 将一个集群中正在运行的多个 Pod 版本，交替地逐一升级的过程，就是“滚动更新”。
- 为了进一步保证服务的连续性，Deployment Controller 还会确保，在任何时间窗口内，只有指定比例的 Pod 处于离线状态。同时，它也会确保，在任何时间窗口内，只有指定比例的新Pod 被创建出来。
  - 这个策略，是 Deployment 对象的一个字段，名叫 RollingUpdateStrategy
    - maxSurge 指定的是除了 DESIRED 数量之外，在一次“滚动”中，Deployment 控制器还可以创建多少个新 Pod；
    - 而 maxUnavailable指的是，在一次“滚动”中，Deployment 控制器可以删除多少个旧 Pod。
  - 通过这样的多个 ReplicaSet 对象，Kubernetes 项目就实现了对多个“应用版本”的描述。
- Deployment 对应用进行版本控制的具体原理
  - 我们只需要执行一条 kubectl rollout undo 命令，就能把整个 Deployment 回滚到上一个版本
  - kubectl rollout history 命令，查看每次 Deployment 变更对应的版本
    - kubectl rollout undo 命令行最后，加上要回滚到的指定版本的版本号，就可以回滚到指定版本了。
  - Kubernetes 项目还提供了一个指令，使得我们对 Deployment 的多次更新操作，最后只生成一个 ReplicaSet。
    - 具体的做法是，在更新 Deployment 前，你要先执行一条 kubectl rollout pause 指令。
    - 而等到我们对 Deployment 修改操作都完成之后，只需要再执行一条 kubectl rollout resume指令，就可以把这个 Deployment“恢复”回来
  - Deployment 对象有一个字段，叫作 spec.revisionHistoryLimit，就是 Kubernetes为 Deployment 保留的“历史版本”个数。所以，如果把它设置为 0，你就再也不能做回滚操作了。
- Deployment 实际上是一个两层控制器。首先，它通过ReplicaSet 的个数来描述应用的版本；然后，它再通过ReplicaSet 的属性（比如 replicas 的值），来保证 Pod 的副本数量。







## StatefulSet拓扑状态

- Deployment认为，一个应用的所有 Pod，是完全一样的。所以，它们互相之间没有顺序，也无所谓运行在哪台宿主机上。

- 在实际的场景中，并不是所有的应用都可以满足这样的要求。

  - 尤其是分布式应用，它的多个实例之间，往往有依赖关系，比如：主从关系、主备关系。
  - 数据存储类应用，它的多个实例，往往都会在本地磁盘上保存一份数据。而这些实例一旦被杀掉，即便重建出来，实例与数据之间的对应关系也已经丢失，从而导致应用失败。
  - 所以，这种实例之间有不对等关系，以及实例对外部数据有依赖关系的应用，就被称为“有状态应用”（Stateful Application）。

- StatefulSet 的设计

  - 拓扑状态
    - 应用的多个实例之间不是完全对等的关系，须按照某些顺序启动
  - 存储状态
    - 应用的多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A 第一次读取到的数据，和隔了十分钟之后再次读取到的数据，应该是同一份，哪怕在此期间 Pod A 被重新创建过。这种情况最典型的例子，就是一个数据库应用的多个存储实例。
  - StatefulSet 的核心功能，就是通过某种方式记录这些状态，然后在 Pod 被重新创建时，能够为新 Pod 恢复这些状态。

- Headless Service

  - Service 是 Kubernetes 项目中用来将一组 Pod 暴露给外界访问的一种机制。
  - 第一种方式，是以 Service 的 VIP（Virtual IP，即：虚拟 IP）方式。
    - 比如：当我访问 10.0.23.1这个 Service 的 IP 地址时，10.0.23.1 其实就是一个VIP，它会把请求转发到该 Service 所代理的某一个 Pod 上
  - 第二种方式，就是以 Service 的 DNS 方式
    - 第一种处理方法，是 Normal Service。
      - 你访问“my-svc.mynamespace.svc.cluster.local”这条 DNS 记录解析到的，正是 my-svc 这个 Service 的 VIP，后面的流程就跟VIP 方式一致了
    - 而第二种处理方法，正是 Headless Service
      - 你访问“my-svc.my-namespace.svc.cluster.local”这条 DNS 记录解析到的，直接就是 my-svc 代理的某一个 Pod 的 IP 地址
      - clusterIP 字段的值是：None，即：这个 Service，没有一个 VIP 作为“头”。这也就是Headless 的含义
      - Headless Service 不需要分配一个 VIP，而是可以直接以 DNS 记录的方式解析出被代理 Pod 的 IP 地址。

- 所谓的 Headless Service，其实仍是一个标准 Service 的 YAML 文件

  - 而它所代理的 Pod，依然是采用 Label Selector 机制选择出来的

  - ```yaml
    spec:
     ports:
      - port: 80c
        name: web
        clusterIP: None
        selector:
         app: nginx
    ```

- StatefulSet 又是如何使用这个 DNS 记录来维持 Pod 的拓扑状态的呢？

  - serviceName=nginx 字段告诉 StatefulSet 控制器，在执行控制循环（Control Loop）的时候，请使用 nginx 这个 Headless Service 来保证 Pod 的“可解析身份”。
  - StatefulSet 给它所管理的所有 Pod 的名字，进行了编号，编号规则是：-

- StatefulSet 这个控制器的主要作用之一，就是使用 Pod 模板创建 Pod 的时候，对它们进行编号，并且按照编号顺序逐一完成创建工作。

  - 而当 StatefulSet 的“控制循环”发现 Pod 的“实际状态”与“期望状态”不一致，需要新建或者删除Pod 进行“调谐”的时候，它会严格按照这些 Pod 编号的顺序，逐一完成这些操作
  - 任何pod的变化都会触发一次statefulset的滚动更新。
  - 在部署“有状态应用”的时候，应用的每个实例拥有唯一并且稳定的“网络标识”，是一个非常重要的假设。







## StatefulSet存储状态

- Kubernetes 项目引入了一组叫作 Persistent VolumeClaim（PVC）和 Persistent Volume（PV）的 API 对象，大大降低了用户声明和使用持久化Volume 的门槛。
- PVC
  - 第一步：定义一个 PVC，声明想要的 Volume 的属性：
    - storage: 1Gi，表示我想要的 Volume 大小至少是 1 GB；accessModes:ReadWriteOnce，表示这个 Volume 的挂载方式是可读写，并且只能被挂载在一个节点上而非被多个节点共享。
  - 第二步：在应用的 Pod 中，声明使用这个 PVC：
    - 在这个 Pod 的 Volumes 定义中，我们只需要声明它的类型是persistentVolumeClaim，然后指定 PVC 的名字，而完全不必关心 Volume 本身的定义。
    - 只要我们创建这个 PVC 对象，Kubernetes 就会自动为它绑定一个符合条件的Volume。可是，这些符合条件的 Volume 来自于由运维人员维护的 PV（Persistent Volume）对象。
- 而 PVC、PV 的设计，也使得 StatefulSet 对存储状态的管理成为了可能
  - 为这个 StatefulSet 额外添加了一个 volumeClaimTemplates 字段。
    - 也就是说，凡是被这个 StatefulSet 管理的 Pod，都会声明一个对应的 PVC；而这个 PVC 的定义，就来自于volumeClaimTemplates 这个模板字段。更重要的是，这个 PVC 的名字，会被分配一个与这个Pod 完全一致的编号。
    - 这个自动创建的 PVC，与 PV 绑定成功后，就会进入 Bound 状态，这就意味着这个 Pod 可以挂载并使用这个 PV 了。
  - 当你把一个 Pod，比如 web-0，删除之后。StatefulSet 控制器发现，一个名叫 web-0 的 Pod 消失了。所以，控制器就会重新创建一个新的、名字还是叫作 web-0 的 Pod 来，“纠正”这个不一致的情况。
    - 这样，新的 Pod 就可以挂载到旧 Pod 对应的那个 Volume，并且获取到保存在 Volume 里的数据。
  - StatefulSet 还为每一个 Pod 分配并创建一个同样编号的 PVC
- StatefulSet 的设计思想：StatefulSet 其实就是一种特殊的Deployment，而其独特之处在于，它的每个 Pod 都被编号了
  - 而且，这个编号会体现在 Pod的名字和 hostname 等标识信息上，这不仅代表了 Pod 的创建顺序，也是 Pod 的重要网络标识（即：在整个集群里唯一的、可被的访问身份）
  - 有了这个编号后，StatefulSet 就使用 Kubernetes 里的两个标准功能：Headless Service 和PV/PVC，实现了对 Pod 的拓扑状态和存储状态的维护。

























