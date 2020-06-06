## PV、PVC、StorageClass

- 容器化一个应用比较麻烦的地方，莫过于对其“状态”的管理。而最常见的“状态”，又莫过于存储状态了。

- PV、PVC这套持久化存储体系

  - PV 描述的，是持久化存储数据卷，这个 API 对象主要定义的是一个持久化存储在宿主机上的目录，比如一个 NFS 的挂载目录。
  - PVC 描述的，则是 Pod 所希望使用的持久化存储的属性。比如，Volume 存储的大小、可读写权限等等。
  - 用户创建的 PVC 要真正被容器使用起来，就必须先和某个符合条件的 PV 进行绑定，检查的条件，包括两部分：
    - 第一个条件，当然是 PV 和 PVC 的 spec 字段。比如，PV 的存储（storage）大小，就必须满足 PVC 的要求。
    - 而第二个条件，则是 PV 和 PVC 的 storageClassName 字段必须一样。

- 还有一个比较棘手的情况

  - 你在创建 Pod 的时候，系统里并没有合适的 PV 跟它定义的 PVC 绑定，也就是说此时容器想要使用的 Volume 不存在。这时候，Pod 的启动就会报错。
  - 在 Kubernetes 中，实际上存在着一个专门处理持久化存储的控制器，叫作 VolumeController。这个 Volume Controller 维护着多个控制循环，其中有一个循环，扮演的就是撮合 PV 和 PVC 的“红娘”的角色。它的名字叫作 PersistentVolumeController。
    - 不断地查看当前每一个 PVC，是不是已经处于 Bound（已绑定）状态。如果不是，那它就会遍历所有的、可用的 PV，并尝试将其与这个“单身”的 PVC进行绑定
    - 而所谓将一个 PV 与 PVC 进行“绑定”，其实就是将这个 PV 对象的名字，填在了 PVC 对象的spec.volumeName 字段上。所以，接下来 Kubernetes 只要获取到这个 PVC 对象，就一定能够找到它所绑定的 PV。
  - 大多数情况下，持久化 Volume 的实现，往往依赖于一个远程存储服务，比如：远程文件存储（比如，NFS、GlusterFS）、远程块存储（比如，公有云提供的远程磁盘）等等。
    -  Kubernetes 需要做的工作，就是使用这些存储服务，来为容器准备一个持久化的宿主机目录，以供将来进行绑定挂载时使用
    - 所谓“持久化”，指的是容器在这个目录里写入的文件，都会保存在远程存储中，从而使得这个目录具备了“持久性”。

- StorageClass

  - PV 这个对象的创建，是由运维人员完成的。但是，在大规模的生产环境里，这其实是一个非常麻烦的工作。
  - Kubernetes 为我们提供了一套可以自动创建 PV 的机制，即：Dynamic Provisioning。
    - Dynamic Provisioning 机制工作的核心，在于一个名叫 StorageClass 的 API 对象。
    - StorageClass 对象的作用，其实就是创建 PV 的模板。
    - StorageClass 对象会定义如下两个部分内容：
      - 第一，PV 的属性。比如，存储类型、Volume 的大小等等。
      - 第二，创建这种 PV 需要用到的存储插件。比如，Ceph 等等。

- StorageClass 到底是干什么用的

  - PVC 描述的，是 Pod 想要使用的持久化存储的属性
  - PV 描述的，则是一个具体的 Volume 的属性
  - StorageClass 的作用，则是充当 PV 的模板。并且，只有同属于一个 StorageClass 的PV 和 PVC，才可以绑定在一起。
  - StorageClass 的另一个重要作用，是指定 PV 的 Provisioner（存储插件）。这时候，如果你的存储插件支持 Dynamic Provisioning 的话，Kubernetes 就可以自动为你创建 PV 了。

- 以后凡是提到“Volume”，指的就是一个远程存储服务挂载在宿主机上的持久化目录；而“PV”，指的是这个 Volume 在Kubernetes 里的 API 对象。

  

























