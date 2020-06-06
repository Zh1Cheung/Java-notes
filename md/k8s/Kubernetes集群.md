## kubeadm

- kubeadm这个项目的目的，就是要让用户能够通过这样两条指令完成一个 Kubernetes 集群的部署

  - > 创建一个 Master 节点 
    >
    > $ kubeadm init 
    >
    > 将一个 Node 节点加入到当前集群中 
    >
    > kubeadm join <Master 节点的 IP 和端口 >

- kubeadm 的工作原理

  - 除了跟容器运行时打交道外，kubelet 在配置容器网络、管理容器数据卷时，都需要直接操作宿主机。
  - 把 kubelet 直接运行在宿主机上，然后使用容器部署其他的 Kubernetes 组件。

- kubeadm init 的工作流程

  - 当你执行 kubeadm init 指令后，kubeadm 首先要做的，是一系列的检查工作
  - 接下来，kubeadm 会为 Master 组件生成 Pod 配置文件。
    - Kubernetes 有三个 Master 组件 kube-apiserver、kube-controller-manager、kube-scheduler，而它们都会被使用 Pod 的方式部署起来。
    - 这时，Kubernetes 集群尚不存在，难道 kubeadm 会直接执行 docker run 来启动这些容器吗？
    - 当然不是。在 Kubernetes 中，有一种特殊的容器启动方法叫做“Static Pod”。它允许你把要部署的 Pod 的YAML 文件放在一个指定的目录里。这样，当这台机器上的 kubelet 启动时，它会自动检查这个目录，加载所有的 Pod YAML 文件，然后在这台机器上启动它们。
    - 在 kubeadm 中，Master 组件的 YAML 文件会被生成在 /etc/kubernetes/manifests 路径下
  - 在这一步完成后，kubeadm 还会再生成一个 Etcd 的 Pod YAML 文件，用来通过同样的 StaticPod 的方式启动 Etcd。
  - 而一旦这些 YAML 文件出现在被 kubelet 监视的 /etc/kubernetes/manifests 目录下，kubelet 就会自动创建这些 YAML 文件中定义的 Pod，即 Master 组件的容器。
  - 然后kubeadm 就会为集群生成一个 bootstrap token。在后面，只要持有这个 token，任何一个安装了 kubelet 和 kubadm 的节点，都可以通过 kubeadm join 加入到这个集群当中。
  - kubeadm init 的最后一步，就是安装默认插件。

- 配置 kubeadm 的部署参数

  - > 我强烈推荐你在使用 kubeadm init 部署 Master 节点时，使用下面这条指令：
    > $ kubeadm init --config kubeadm.yaml

  - 这时，你就可以给 kubeadm 提供一个 YAML 文件（比如，kubeadm.yaml）

    - 通过制定这样一个部署参数配置文件，你就可以很方便地在这个文件里填写各种自定义的部署参数了。
    - 然后，kubeadm 就会使用上面这些信息替换 /etc/kubernetes/manifests/kube-apiserver.yaml里的字段里的参数了。

-  kubeadm 目前最欠缺的是，一键部署一个高可用的 Kubernetes 集群，即：Etcd、Master 组件都应该是多节点集群，而不是现在这样的单点。这，当然也正是 kubeadm 接下来发展的主要方向。







## 搭建一个完整的Kubernetes集群

- 部署 Kubernetes 的 Master 节点

  - > 使用 kubectl get 命令来查看当前唯一一个节点的状态：
    >
    > $ kubectl get nodes
    >
    > 
    >
    > 在调试 Kubernetes 集群时，最重要的手段就是用 kubectl describe 来查看这个节点（Node）对象的详细信息、状态和事件（Event）
    >
    > $ kubectl describe node master
    >
    > 
    >
    > 还可以通过 kubectl 检查这个节点上各个系统 Pod 的状态，其中，kube-system 是Kubernetes 项目预留的系统 Pod 的工作空间（Namepsace，注意它并不是 Linux Namespace，它只是 Kubernetes 划分不同工作空间的单位）
    >
    > $ kubectl get pods -n kube-system

- 部署 Kubernetes 的 Worker 节点

  - Kubernetes 的 Worker 节点跟 Master 节点几乎是相同的，它们运行着的都是一个 kubelet 组件。唯一的区别在于，在 kubeadm init 的过程中，kubelet 启动后，Master 节点上还会自动运行kube-apiserver、kube-scheduler、kube-controller-manger 这三个系统 Pod。
  - 在所有 Worker 节点上执行“安装 kubeadm 和 Docker”一节的所有步骤。
  - 执行部署 Master 节点时生成的 kubeadm join 指令

- 通过 Taint/Toleration 调整 Master 执行 Pod 的策略

  - 默认情况下 Master 节点是不允许运行用户 Pod 的。而 Kubernetes 做到这一点，依靠的是 Kubernetes 的 Taint/Toleration 机制。

    - 一旦某个节点被加上了一个 Taint，即被“打上了污点”，那么所有 Pod 就都不能在这个节点上运行，因为 Kubernetes 的 Pod 都有“洁癖”。
    - 除非，有个别的 Pod 声明自己能“容忍”这个“污点”，即声明了 Toleration，它才可以在这个节点上运行。

  - $ kubectl taint nodes node1 foo=bar:NoSchedule

    - 该 node1 节点上就会增加一个键值对格式的 Taint，即：foo=bar:NoSchedule
    - 值里面的 NoSchedule，意味着这个 Taint 只会在调度新 Pod 时产生作用，而不会影响已经在 node1上运行的 Pod，哪怕它们没有Toleration。

  - 那么 Pod 又如何声明 Toleration 呢？

    - ```yaml
      spec: 
      	tolerations: 
      	- key: "foo" 
      		operator: "Equal" 
      		value: "bar" 
      		effect: "NoSchedule"
      		
      Equal”，“等于”操作
      ```

- 部署容器存储插件

  - 如果你在某一台机器上启动的一个容器，显然无法看到其他机器上的容器在它们的数据卷里写入的文件。这是容器最典型的特征之一：无状态。
  - 容器的持久化存储，就是用来保存容器存储状态的重要手段：存储插件会在容器里挂载一个基于网络或者其他机制的远程数据卷，使得在容器里创建的文件，实际上是保存在远程存储服务器上，或者以分布式的方式保存在多个节点上，而与当前宿主机没有任何绑定关系

- 其实，在很多时候，大家说的所谓“云原生”，就是“Kubernetes 原生”的意思。

- 在 Bare-metal 环境下使用 kubeadm 工具部署了一个完整的Kubernetes 集群：这个集群有一个 Master 节点和多个 Worker 节点；使用 Weave 作为容器网络插件；使用 Rook 作为容器持久化存储插件；使用 Dashboard 插件提供了可视化的 Web 界面。

  

  







## 容器化应用

- 有了容器镜像之后，你需要按照 Kubernetes 项目的规范和要求，将你的镜像组织为它能够“认识”的方式（这就是使用 Kubernetes 的必备技能：编写配置文件），然后提交上去。

- 像这样的一个 YAML 文件，对应到 Kubernetes 中，就是一个 API Object（API 对象）

  - 每一个 API 对象都有一个叫作 Metadata 的字段，这个字段就是 API 对象的“标识”，即元数据，它也是我们从 Kubernetes 里找到这个对象的主要依据。这其中最主要使用到的字段是 Labels。

    - Labels 就是一组 key-value 格式的标签。而像 Deployment 这样的控制器对象，就可以通过这个 Labels 字段从 Kubernetes 中过滤出它所关心的被控制对象。
    - 而这个过滤规则的定义，是在 Deployment 的“spec.selector.matchLabels”字段。我们一般称之为：Label Selector。

  - 一个 Kubernetes 的 API 对象的定义，大多可以分为 Metadata 和 Spec 两个部分。

    - 前者存放的是这个对象的元数据，对所有 API 对象来说，这一部分的字段和格式基本上是一样的；

    - 而后者存放的，则是属于这个对象独有的定义，用来描述它所要表达的功能。

    - ```yaml
      #test-pod 
      apiVersion: v1 #指定api版本，此值必须在kubectl apiversion中   
      kind: Pod #指定创建资源的角色/类型   
      metadata: #资源的元数据/属性   
        name: test-pod #资源的名字，在同一个namespace中必须唯一   
        labels: #设定资源的标签 
          k8s-app: apache   
          version: v1   
          kubernetes.io/cluster-service: "true"   
        annotations:            #自定义注解列表   
          - name: String        #自定义注解名字   
      spec: #specification of the resource content 指定该资源的内容   
        restartPolicy: Always #表明该容器一直运行，默认k8s的策略，在此容器退出后，会立即创建一个相同的容器   
        nodeSelector:     #节点选择，先给主机打标签kubectl label nodes kube-node1 zone=node1   
          zone: node1   
        containers:   
        - name: test-pod #容器的名字   
          image: 10.192.21.18:5000/test/chat:latest #容器使用的镜像地址   
          imagePullPolicy: Never #三个选择Always、Never、IfNotPresent，每次启动时检查和更新（从registery）images的策略， 
                                 # Always，每次都检查 
                                 # Never，每次都不检查（不管本地是否有） 
                                 # IfNotPresent，如果本地有就不检查，如果没有就拉取 
          command: ['sh'] #启动容器的运行命令，将覆盖容器中的Entrypoint,对应Dockefile中的ENTRYPOINT   
          args: ["$(str)"] #启动容器的命令参数，对应Dockerfile中CMD参数   
          env: #指定容器中的环境变量   
          - name: str #变量的名字   
            value: "/etc/run.sh" #变量的值   
          resources: #资源管理 
            requests: #容器运行时，最低资源需求，也就是说最少需要多少资源容器才能正常运行   
              cpu: 0.1 #CPU资源（核数），两种方式，浮点数或者是整数+m，0.1=100m，最少值为0.001核（1m） 
              memory: 32Mi #内存使用量   
            limits: #资源限制   
              cpu: 0.5   
              memory: 1000Mi   
          ports:   
          - containerPort: 80 #容器开发对外的端口 
            name: httpd  #名称 
            protocol: TCP   
          livenessProbe: #pod内容器健康检查的设置 
            httpGet: #通过httpget检查健康，返回200-399之间，则认为容器正常   
              path: / #URI地址   
              port: 80   
              #host: 127.0.0.1 #主机地址   
              scheme: HTTP   
            initialDelaySeconds: 180 #表明第一次检测在容器启动后多长时间后开始   
            timeoutSeconds: 5 #检测的超时时间   
            periodSeconds: 15  #检查间隔时间   
            #也可以用这种方法   
            #exec: 执行命令的方法进行监测，如果其退出码不为0，则认为容器正常   
            #  command:   
            #    - cat   
            #    - /tmp/health   
            #也可以用这种方法   
            #tcpSocket: //通过tcpSocket检查健康    
            #  port: number    
          lifecycle: #生命周期管理   
            postStart: #容器运行之前运行的任务   
              exec:   
                command:   
                  - 'sh'   
                  - 'yum upgrade -y'   
            preStop:#容器关闭之前运行的任务   
              exec:   
                command: ['service httpd stop']   
          volumeMounts:  #挂载持久存储卷 
          - name: volume #挂载设备的名字，与volumes[*].name 需要对应     
            mountPath: /data #挂载到容器的某个路径下   
            readOnly: True   
        volumes: #定义一组挂载设备   
        - name: volume #定义一个挂载设备的名字   
          #meptyDir: {}   
          hostPath:   
            path: /opt #挂载设备类型为hostPath，路径为宿主机下的/opt,这里设备类型支持很多种 
          #nfs
      ```

      

  - emptyDir

    - 它其实就等同于我们之前讲过的 Docker 的隐式 Volume 参数，即：不显式声明宿主机目录的Volume。所以，Kubernetes 也会在宿主机上创建一个临时目录，这个目录将来就会被绑定挂载到容器所声明的 Volume 目录上

  - Kubernetes 也提供了显式的 Volume 定义，它叫做 hostPath

    - > hostPath: 
      > 	path: /var/data
      > 这样，容器 Volume 挂载的宿主机目录，就变成了 /var/data。

- Kubernetes 推荐的使用方式，是用一个 YAML 文件来描述你所要部署的 API 对象。然后，统一使用 kubectl apply 命令完成对这个对象的创建和更新操作。

  

  

  

  

  

  

  









































