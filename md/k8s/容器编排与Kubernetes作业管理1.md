## DaemonSet

- 金丝雀发布（Canary Deploy）或者灰度发布

  - 这个字段，正是 StatefulSet 的 spec.updateStrategy.rollingUpdate 的 partition 字段。
  - 比如，现在我将前面这个 StatefulSet 的 partition 字段设置为 2：
  - 这样，我就指定了当 Pod 模板发生变化的时候，比如 MySQL 镜像更新到 5.7.23，那么只有Pod序号大于或者等于 2 的 Pod 会被更新到这个版本

- DaemonSet 的主要作用，是让你在 Kubernetes 集群里，运行一个 DaemonPod。 所以，这个 Pod 有如下三个特征

  - 运行在 Kubernetes 集群里的每一个节点（Node）上；
    - 网络插件、存储插件、监控和日志插件
  - 每个节点上只有一个这样的 Pod 实例；
  - 当有新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉。
  - 更重要的是，跟其他编排对象不一样，DaemonSet 开始运行的时机，很多时候比整个Kubernetes 集群出现的时机都要早。

- DaemonSet Controller，首先从 Etcd 里获取所有的 Node 列表，然后遍历所有的 Node。这时，它就可以很容易地去检查，当前这个 Node 上是不是有一个携带了 name=fluentdelasticsearch 标签的 Pod 在运行。而检查的结果，可能有这三种情况：

  - 没有这种 Pod，那么就意味着要在这个 Node 上创建这样一个 Pod；

    - ```yaml
      # 这个 Pod，将来只允许运行在“metadata.name”是“aaa”的节点上。    
          spec:
            affinity:
              nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                  - matchExpressions:
                    - key: metadata.name
                      operator: In
                      values:
                      - aaa
      ```

      

  - 有这种 Pod，但是数量大于 1，那就说明要把多余的 Pod 从这个 Node 上删除掉；

  - 正好只有一个这种 Pod，那说明这个节点是正常的。

- 在正常情况下，被标记了 unschedulable“污点”的 Node，是不会有任何 Pod 被调度上去的（effect: NoSchedule）。可是，DaemonSet 自动地给被管理的 Pod 加上了这个特殊的Toleration，就使得这些 Pod 可以忽略这个限制，继而保证每个节点上都会被调度一个 Pod。

  - 而通过这样一个 Toleration，调度器在调度这个 Pod 的时候，就会忽略当前节点上的“污点”，从而成功地将网络插件的 Agent 组件调度到这台机器上启动起来。

- DaemonSet 其实是一个非常简单的控制器。在它的控制循环中，只需要遍历所有节点，然后根据节点上是否有被管理 Pod 的情况，来决定是否要创建或者删除一个 Pod。

  - 只不过，在创建每个 Pod 的时候，DaemonSet 会自动给这个 Pod 加上一个 nodeAffinity，从而保证这个 Pod 只会在指定节点上启动。同时，它还会自动给这个 Pod 加上一个Toleration，从而忽略节点的 unschedulable“污点”。
  - 也可以在 Pod 模板里加上更多种类的 Toleration，从而利用 DaemonSet 实现自己的目的。
  - 相比于 Deployment，DaemonSet 只管理 Pod 对象，然后通过 nodeAffinity 和 Toleration这两个调度器的小功能，保证了每个节点上有且只有一个 Pod





## Job与CronJob

-  Deployment、StatefulSet，以及 DaemonSet 这三个编排概念。
  - 实际上，它们主要编排的对象，都是“在线业务”，即：Long Running Task（长作业）。比如，我在前面举例时常用的 Nginx、Tomcat，以及 MySQL 等等。这些应用一旦运行起来，除非出错或者停止，它的容器进程会一直保持在 Running 状态。
  - 有一类作业显然不满足这样的条件，这就是“离线业务”，或者叫作 Batch Job（计算业务），计算完成后就直接退出了
  - 描述离线业务的 API 对象，它的名字就是：Job

- TODO





## 声明式API与Kubernetes编程范式

- 到底什么才是“声明式 API”呢？答案是，kubectl apply 命令

  - 实际上，你可以简单地理解为，kubectl replace 的执行过程，是使用新的 YAML 文件中的 API对象，替换原有的 API 对象；而 kubectl apply，则是执行了一个对原有 API 对象的 PATCH 操作。
  - 更进一步地，这意味着 kube-apiserver 在响应命令式请求（比如，kubectl replace）的时候，一次只能处理一个写请求，否则会有产生冲突的可能。而对于声明式请求（比如，kubectl apply），一次能处理多个写操作，并且具备 Merge 能力。

- Istio 项目

  - Istio 项目，实际上就是一个基于 Kubernetes 项目的微服务治理框架。
  - Istio 最根本的组件，是运行在每一个应用 Pod 里的 Envoy 容器。这个 Envoy 项目是 Lyft 公司推出的一个高性能 C++ 网络代理
    - Istio 项目，则把这个代理服务以 sidecar 容器的方式，运行在了每一个被治理的应用 Pod中。我们知道，Pod 里的所有容器都共享同一个 Network Namespace。所以，Envoy 容器就能够通过配置 Pod 里的 iptables 规则，把整个 Pod 的进出流量接管下来。
    - Istio 的控制层（Control Plane）里的 Pilot 组件，就能够通过调用每个 Envoy 容器的API，对这个 Envoy 代理进行配置，从而实现微服务治理。
  - Istio 项目使用的，是 Kubernetes 中的一个非常重要的功能，叫作 Dynamic Admission Control。
    - 当一个 Pod 或者任何一个 API 对象被提交给 APIServer 之后，总有一些“初始化”性质的工作需要在它们被 Kubernetes 项目正式处理之前进行。比如，自动为所有Pod 加上某些标签（Labels）。
    - “初始化”操作的实现，借助的是一个叫作 Admission 的功能
    - 被称为 Admission Controller 的代码，可以选择性地被编译进 APIServer 中，在API 对象创建之后会被立刻调用到。
  - Kubernetes 项目为我们额外提供了一种“热插拔”式的 Admission 机制，它就是Dynamic Admission Control，也叫作：Initializer。
    - Istio 又是如何在用户完全不知情的前提下完成这个操作的呢？Istio 要做的，就是编写一个用来为 Pod“自动注入”Envoy 容器的 Initializer。
      - Istio 会将这个 Envoy 容器本身的定义，以 ConfigMap 的方式保存在 Kubernetes 当中。
      - Initializer 要做的工作，就是把这部分 Envoy 相关的字段，自动添加到用户提交的Pod 的 API 对象里。
      - 用户提交的 Pod 里本来就有 containers 字段和 volumes 字段，所以 Kubernetes 在处理这样的更新请求时，就必须使用类似于 git merge 这样的操作，才能将这两部分内容合并在一起。
    - 在 Initializer 更新用户的 Pod 对象的时候，必须使用 PATCH API 来完成。而这种PATCH API，正是声明式 API 最主要的能力。
  - 接下来，Istio 将一个编写好的 Initializer，作为一个 Pod 部署在 Kubernetes 中
    - Initializer 的代码就可以使用这个 patch 的数据，调用Kubernetes 的 Client，发起一个 PATCH 请求。
    - 这样，一个用户提交的 Pod 对象里，就会被自动加上 Envoy 容器相关的字段。

- 以上，就是关于 Initializer 最基本的工作原理和使用方法了。相信你此时已经明白，Istio 项目的核心，就是由无数个运行在应用 Pod 中的 Envoy 容器组成的服务代理网格。这也正是Service Mesh 的含义。

  - 这个机制得以实现的原理，正是借助了 Kubernetes 能够对 API 对象进行在线更新的能力，这也正是Kubernetes“声明式 API”的独特之处

  - 所谓“声明式”，指的就是我只需要提交一个定义好的 API 对象来“声明”，我所期望的状态是什么样子。

  - “声明式 API”允许有多个 API 写端，以 PATCH 的方式对 API 对象进行修改，而无需关心本地原始 YAML 文件的内容。

  - 最后，也是最重要的，有了上述两个能力，Kubernetes 项目才可以基于对 API 对象的增、删、改、查，在完全无需外界干预的情况下，完成对“实际状态”和“期望状态”的调谐（Reconcile）过程。

    







## 声明式API的API对象

- 一个 API 对象在 Etcd 里的完整资源路径，是由：Group（API 组）、Version（API 版本）和 Resource（API 资源类型）三个部分组成的。

  - 创建一个 CronJob 对象，那么我的 YAML 文件的开始部分会这么写

    - > apiVersion: batch/v2alpha1
      > kind: CronJob
      >
      > “CronJob”就是这个 API 对象的资源类型（Resource），“batch”就是它的组（Group），v2alpha1 就是它的版本（Version）。
      >
      > Kubernetes 就会把这个 YAML 文件里描述的内容，转换成 Kubernetes 里的一个 CronJob 对象。

- Kubernetes 是如何对 Resource、Group 和 Version 进行解析，从而在 Kubernetes 项目里找到 CronJob 对象的定义呢？

  - 首先，Kubernetes 会匹配 API 对象的组。
    - 对于 Kubernetes 里的核心 API 对象，比如：Pod、Node 等，是不需要Group 的（即：它们 Group 是“”）。所以，对于这些 API 对象来说，Kubernetes 会直接在api 这个层级进行下一步的匹配过程。
    - 而对于 CronJob 等非核心 API 对象来说，Kubernetes 就必须在 /apis 这个层级里查找它对应的 Group，进而根据“batch”这个 Group 的名字，找到 /apis/batch。
  - 然后，Kubernetes 会进一步匹配到 API 对象的版本号。
    - 在 Kubernetes 中，同一种 API 对象可以有多个版本，这正是 Kubernetes 进行 API 版本化管理的重要手段。
  - 最后，Kubernetes 会匹配 API 对象的资源类型。
  - 这时候，APIServer 就可以继续创建这个 CronJob 对象了
    - APIServer 会进行一个 Convert 工作，即：把用户提交的 YAML 文件，转换成一个叫作 Super Version 的对象，它正是该 API 资源类型所有版本的字段全集。
    - 最后，APIServer 会把验证过的 API 对象转换成用户最初提交的版本，进行序列化操作，并调用 Etcd 的 API 把它保存起来。

- 在 Kubernetes v1.7 之后，这个工作就变得轻松得多了。这，当然得益于一个全新的API 插件机制：CRD

  - 它指的就是，允许用户在Kubernetes 中添加一个跟 Pod、Node 类似的、新的 API 资源类型，即：自定义 API 资源。
  - 具体的“自定义 API 资源”实例，也叫 CR（CustomResource）。而为了能够让 Kubernetes 认识这个 CR，你就需要让 Kubernetes 明白这个 CR的宏观定义是什么，也就是 CRD
    - 先需编写一个 CRD 的 YAML 文件，它的名字叫作 network.yaml，
    - 在这个 CRD 中，我指定了“group: samplecrd.k8s.io”“version:v1”这样的 API 信息，也指定了这个 CR 的资源类型叫作 Network，复数（plural）是networks。
    - 还声明了它的 scope 是 Namespaced，即：我们定义的这个 Network 是一个属于Namespace 的对象，类似于 Pod。
  - 接下来，我还需要让 Kubernetes“认识”这种 YAML 文件里描述的“网络”部分，比如“cidr”（网段），“gateway”（网关）这些字段的含义
    - 就需要稍微做些代码工作了
    - goalng
  - 这样，Network 对象的定义工作就全部完成了。可以看到，它其实定义了两部分内容：
    - 第一部分是，自定义资源类型的 API 描述，包括：组（Group）、版本（Version）、资源类型（Resource）等。这相当于告诉了计算机：兔子是哺乳动物。
    - 第二部分是，自定义资源类型的对象描述，包括：Spec、Status 等。这相当于告诉了计算机：兔子有长耳朵和三瓣嘴。
  - 接下来，我就要使用 Kubernetes 提供的代码生成工具，为上面定义的 Network 资源类型自动生成 clientset、informer 和 lister。这个代码生成工具名叫k8s.io/code-generator
  - 有了这些内容，现在你就可以在 Kubernetes 集群里创建一个 Network 类型的 API 对象了。

- 创建出这样一个自定义 API 对象，我们只是完成了 Kubernetes 声明式 API 的一半工作。

  - 接下来的另一半工作是：为这个 API 对象编写一个自定义控制器（Custom Controller）。这样， Kubernetes 才能根据 Network API 对象的“增、删、改”操作，在真实环境中做出相应的响应。比如，“创建、删除、修改”真正的 Neutron 网络。
  - 而这，正是 Network 这个 API 对象所关注的“业务逻辑”。







## 基于角色的权限控制：RBAC

- Kubernetes 中所有的 API 对象，都保存在 Etcd 里。可是，对这些 API 对象的操作，却一定都是通过访问 kube-apiserver 实现的。其中一个非常重要的原因，就是你需要APIServer 来帮助你做授权工作。

  - 负责完成授权（Authorization）工作的机制，就是 RBAC：基于角色的访问控制
    -  Role：角色，它其实是一组规则，定义了一组对 Kubernetes API 对象的操作权限。
    -  Subject：被作用者
    -  RoleBinding：定义了“被作用者”和“角色”的绑定关系

-  Role

  - Role 本身就是一个 Kubernetes 的 API 对象

  -  rules 字段，就是它所定义的权限规则

    - ```yaml
      # 允许“被作用者”，对 mynamespace 下面的 Pod 对象，进行 GET、WATCH 和LIST 操作。
      kind: Role
      apiVersion: rbac.authorization.k8s.io/v1
      metadata:
       namespace: mynamespace
       name: example-role
      rules:
       - apiGroups: [""]
         resources: ["pods"]
         verbs: ["get", "watch", "list"]
       
       # Role 对象的 rules 字段也可以进一步细化，你可以只针对某一个具体的对象进行权限设置
      rules:
       - apiGroups: [""]
         resources: ["configmaps"]
         resourceNames: ["my-config"]
         verbs: ["get"]
       
      ```

-  RoleBinding

  - RoleBinding 本身也是一个 Kubernetes 的 API 对象

  - ```yaml
    # 这个 RoleBinding 对象里定义了一个 subjects 字段，即“被作用者”，类型是User，即 Kubernetes 里的用户。这个用户的名字是 example-user
    
    # 这个 User 到底是从哪里来的呢？在大多数私有的使用环境中，我们只要使用 Kubernetes 提供的内置“用户”，就足够了。
    
    #  roleRef 字段。正是通过这个字段，RoleBinding 对象就可以直接通过名字，来引用我们前面定义的 Role 对象（example-role），从而定义了“被作用者（Subject）”和“角色（Role）”之间的绑定关系。
    
    # 需要再次提醒的是，Role 和 RoleBinding 对象都是 Namespaced 对象（NamespacedObject），它们对权限的限制规则仅在它们自己的 Namespace 内有效，roleRef 也只能引用当前 Namespace 里的 Role 对象。
    
    # 对于非 Namespaced（Non-namespaced）对象（比如：Node），或者，某一个Role 想要作用于所有的 Namespace 的时候，我们又该如何去做授权呢？
    #使用 ClusterRole 和 ClusterRoleBinding 这两个组合。这两个 API 对象的用法跟 Role 和 RoleBinding 完全一样。只不过，它们的定义里，没有了 Namespace 字段
    
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
     name: example-rolebinding
     namespace: mynamespace
    subjects:
     - kind: User
       name: example-user
       apiGroup: rbac.authorization.k8s.io
    roleRef:
     kind: Role
     name: example-role
     apiGroup: rbac.authorization.k8s.io
    ```

-  ServiceAccount 分配权限的过程

  - 定义一个 ServiceAccount，一个最简单的 ServiceAccount 对象只需要 Name 和 Namespace 这两个最基本的字段。
  - 编写 RoleBinding 的 YAML 文件，来为这个 ServiceAccount 分配权限
  - 用 kubectl 命令创建这三个对象（ServiceAccount 、RoleBinding 、Role）
  - 这时候，用户的 Pod，就可以声明使用这个 ServiceAccount 了
  - 如果一个Pod 没有声明 serviceAccountName，Kubernetes 会自动在它的 Namespace 下创建一个名叫 default 的默认 ServiceAccount，然后分配给这个 Pod。
    - 但在这种情况下，这个默认 ServiceAccount 并没有关联任何 Role。
      此时它有访问APIServer 的绝大多数权限。当然，这个访问所需要的 Token，还是默认 
    - ServiceAccount 对应的 Secret 对象为它提供的，
      Kubernetes 会自动为默认 ServiceAccount 创建并绑定一个特殊的 Secret
    - 在生产环境中，我强烈建议你为所有 Namespace 下的默认 ServiceAccount，绑定一个只读权限的 Role

- 所谓角色（Role），其实就是一组权限规则列表。而我们分配这些权限的方式，就是通过创建 RoleBinding 对象，将被作用者（subject）和权限列表进行绑定。

  - 与之对应的 ClusterRole 和ClusterRoleBinding，则是 Kubernetes 集群级别的 Role和 RoleBinding，它们的作用范围不受 Namespace 限制。
  - 尽管权限的被作用者可以有很多种（比如，User、Group 等），但在我们平常的使用中，最普遍的用法还是 ServiceAccount。所以，Role + RoleBinding + ServiceAccount 的权限分配方式是你要重点掌握的内容

  



































