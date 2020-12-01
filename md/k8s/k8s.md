## Docker

- Docker 实际上只是一个使用 Cgroups 和 Namespace 实现的“沙盒”而已
  - 所谓 Docker 镜像，其实就是一个压缩包。
  - 大多数 Docker 镜像是直接由一个完整操作系统的所有文件和目录构成的，所以这个压缩包里的内容跟你本地开发和测试环境用的操作系统是完全一样的。
  - 这就是 Docker 镜像最厉害的地方：只要有这个压缩包在手，你就可以使用某种技术创建一个“沙盒”，在“沙盒”中解压这个压缩包，然后就可以运行你的程序了。
  - 容器时代，“编排”显然就是对 Docker 容器的一系列定义、配置和创建动作的管理

# 一、关于Dockerfile

　　在Docker中创建镜像最常用的方式，就是使用Dockerfile。Dockerfile是一个Docker镜像的描述文件，我们可以理解成火箭发射的A、B、C、D…的步骤。Dockerfile其内部**包含了一条条的指令**，**每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建**。

![img](https://img2018.cnblogs.com/blog/381412/201908/381412-20190811220705871-2130672519.png)

　　一个Dockerfile的示例如下所示：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
#基于centos镜像
FROM centos

#维护人的信息
MAINTAINER The CentOS Project <303323496@qq.com>

#安装httpd软件包
RUN yum -y update
RUN yum -y install httpd

#开启80端口
EXPOSE 80

#复制网站首页文件至镜像中web站点下
ADD index.html /var/www/html/index.html

#复制该脚本至镜像中，并修改其权限
ADD run.sh /run.sh
RUN chmod 775 /run.sh

#当启动容器时执行的脚本文件
CMD ["/run.sh"]
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　由上可知，Dockerfile结构大致分为四个部分：

　　（1）基础镜像信息

　　（2）维护者信息

　　（3）镜像操作指令

　　（4）容器启动时执行指令。

　　Dockerfile每行支持一条指令，每条指令可带多个参数，支持使用以#号开头的注释。下面会对上面使用到的一些常用指令做一些介绍。

# 二、Dockerfile常用指令

首先，来一张通俗易懂的**全景图**：

![img](https://img2018.cnblogs.com/blog/450977/201905/450977-20190512115951746-136143052.png)

2.1 FROM

　　指明构建的新镜像是来自于哪个基础镜像，例如：

```
FROM centos:6
```

2.2 MAINTAINER

　　指明镜像维护着及其联系方式（一般是邮箱地址），例如：

```
MAINTAINER Edison Zhou <edisonchou@hotmail.com>
```

　　不过，MAINTAINER并不推荐使用，更推荐使用LABEL来指定镜像作者，例如：

```
LABEL maintainer="edisonzhou.cn"
```

2.3 RUN

　　构建镜像时运行的Shell命令，例如：

```
RUN ["yum", "install", "httpd"]
RUN yum install httpd
```

　　又如，我们在使用微软官方ASP.NET Core Runtime镜像时往往会加上以下RUN命令，弥补无法在默认镜像下使用Drawing相关接口的缺憾：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

FROM microsoft/dotnet:2.2.1-aspnetcore-runtime
RUN apt-get update
RUN apt-get install -y libgdiplus
RUN apt-get install -y libc6-dev
RUN ln -s /usr/lib/libgdiplus.so /lib/x86_64-linux-gnu/libgdiplus.so

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

2.4 CMD

　　启动容器时执行的Shell命令，例如：

```
CMD ["-C", "/start.sh"] 
CMD ["/usr/sbin/sshd", "-D"] 
CMD /usr/sbin/sshd -D
```

2.5 EXPOSE

　　声明容器运行的服务端口，例如：

```
EXPOSE 80 443
```

2.6 ENV

　　设置环境内环境变量，例如：

```
ENV MYSQL_ROOT_PASSWORD 123456
ENV JAVA_HOME /usr/local/jdk1.8.0_45
```

2.7 ADD

　　拷贝文件或目录到镜像中，例如：

```
ADD <src>...<dest>
ADD html.tar.gz /var/www/html
ADD https://xxx.com/html.tar.gz /var/www/html

```

　　***PS：***如果是URL或压缩包，会自动下载或自动解压。

2.8 COPY

　　拷贝文件或目录到镜像中，用法同ADD，只是不支持自动下载和解压，例如：

```
COPY ./start.sh /start.sh

```

2.9 ENTRYPOINT

　　启动容器时执行的Shell命令，同CMD类似，只是由ENTRYPOINT启动的程序**不会被docker run命令行指定的参数所覆盖**，而且，**这些命令行参数会被当作参数传递给ENTRYPOINT指定指定的程序**，例如：

```
ENTRYPOINT ["/bin/bash", "-C", "/start.sh"]
ENTRYPOINT /bin/bash -C '/start.sh'

```

　　***PS：***Dockerfile文件中也可以存在多个ENTRYPOINT指令，但仅有最后一个会生效。

2.10 VOLUME

　　指定容器挂载点到宿主机自动生成的目录或其他容器，例如：

```
VOLUME ["/var/lib/mysql"]

```

　　***PS：***一般不会在Dockerfile中用到，更常见的还是在docker run的时候指定-v数据卷。

2.11 USER

　　为RUN、CMD和ENTRYPOINT执行Shell命令指定运行用户，例如：

```
USER <user>[:<usergroup>]
USER <UID>[:<UID>]
USER edisonzhou

```

2.12 WORKDIR

　　为RUN、CMD、ENTRYPOINT以及COPY和AND设置工作目录，例如：

```
WORKDIR /data

```

2.13 HEALTHCHECK

　　告诉Docker如何测试容器以检查它是否仍在工作，即健康检查，例如：

```
HEALTHCHECK --interval=5m --timeout=3s --retries=3 \
    CMD curl -f http:/localhost/ || exit 1

```

　　其中，一些选项的说明：

- --interval=DURATION (default: 30s)：每隔多长时间探测一次，默认30秒
- -- timeout= DURATION (default: 30s)：服务响应超时时长，默认30秒
- --start-period= DURATION (default: 0s)：服务启动多久后开始探测，默认0秒
- --retries=N (default: 3)：认为检测失败几次为宕机，默认3次

　　一些返回值的说明：

- 0：容器成功是健康的，随时可以使用
- 1：不健康的容器无法正常工作
- 2：保留不使用此退出代码

2.14 ARG

　　在构建镜像时，指定一些参数，例如：

```
FROM centos:6
ARG user # ARG user=root
USER $user

```

　　这时，我们在docker build时可以带上自定义参数user了，如下所示：

```
docker build --build-arg user=edisonzhou Dockerfile .

```

# 三、综合Dockerfile案例

　　下面是一个Java Web应用的镜像Dockerfile，综合使用到了上述介绍中最常用的几个命令：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
FROM centos:7
MAINTANIER www.edisonchou.com

ADD jdk-8u45-linux-x64.tar.gz /usr/local
ENV JAVA_HOME /usr/local/jdk1.8.0_45

ADD apache-tomcat-8.0.46.tar.gz /usr/local
COPY server.xml /usr/local/apache-tomcat-8.0.46/conf

RUN rm -f /usr/local/*.tar.gz

WORKDIR /usr/local/apache-tomcat-8.0.46
EXPOSE 8080
ENTRYPOINT ["./bin/catalina.sh", "run"]

```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

　　有了Dockerfile，就可以创建镜像了：

```
docker build -t tomcat:v1 .

```

　　最后，可以通过以下命令创建容器：

```
docker run -itd --name=tomcate -p 8080:8080 \
    -v /app/webapps/:/usr/local/apache-tomcat-8.0.46/webapps/ \
    tomcat:v1

```

四、小结

　　本文介绍了Dockerfile的背景和组成，以及最常用的一些Dockerfile命令，最后介绍了一个综合使用了Dockefile指令的一个案例来说明Dockerfile的应用。





## Docker底层

- 对于进程来说，它的静态表现就是程序，平常都安安静静地待在磁盘上；而一旦运行起来，它就变成了计算机里的数据和状态的总和，这就是它的动态表现。
- 而容器技术的核心功能，就是通过约束和修改进程的动态表现，从而为其创造出一个“边界”。
  - Cgroups 技术是用来制造约束的主要手段，而Namespace 技术则是用来修改进程视图的主要方法。
  - 本来，每当我们在宿主机上运行了一个 /bin/sh 程序，操作系统都会给它分配一个进程编号，比如 PID=100。这个编号是进程的唯一标识，就像员工的工牌一样。
  - 而现在，我们要通过 Docker 把这个 /bin/sh 程序运行在一个容器当中。这时候，Docker 就会在这个第 100 号员工入职时给他施一个“障眼法”，让他永远看不到前面的其他 99 个员工，更看不到比尔 · 盖茨。这样，他就会错误地以为自己就是公司里的第 1 号员工。
  - 这种机制，其实就是对被隔离应用的进程空间做了手脚，使得这些进程只能看到重新计算过的进程编号，比如 PID=1。可实际上，他们在宿主机的操作系统里，还是原来的第 100 号进程。
  - 除了我们刚刚用到的 PID Namespace，Linux 操作系统还提供了 Mount、UTS、IPC、Network 和 User 这些 Namespace，用来对各种不同的进程上下文进行“障眼法”操作。
- 所以，Docker 容器这个听起来玄而又玄的概念，实际上是在创建容器进程时，指定了这个进程所需要启用的一组 Namespace 参数。这样，容器就只能“看”到当前 Namespace 所限定的资源、文件、设备、状态，或者配置。而对于宿主机以及其他不相关的程序，它就完全看不到了。
- 虚拟机和容器的对比图
  - Hypervisor
    - 通过硬件虚拟化功能，模拟出了运行一个操作系统需要的各种硬件，比如 CPU、内存、I/O 设备等等
    - 然后，它在这些虚拟的硬件上安装了一个新的操作系统，即 Guest OS。
  - Docker Engine
    - 把虚拟机的概念套在了容器上，这样的说法，并不严谨。
    - 在理解了 Namespace 的工作方式之后，你就会明白，跟真实存在的虚拟机不同，在使用Docker 的时候，并没有一个真正的“Docker 容器”运行在宿主机里面。Docker 项目帮助用户启动的，还是原来的应用进程，只不过在创建这些进程时，Docker 为它们加上了各种各样的Namespace 参数。
    - 这时，这些进程就会觉得自己是各自 PID Namespace 里的第 1 号进程，只能看到各自 Mount Namespace 里挂载的目录和文件，只能访问到各自 Network Namespace 里的网络设备，就仿佛运行在一个个“容器”里面，与世隔绝。



## 隔离与限制（Namespace、cgroup）

- 真正对隔离环境负责的是宿主机操作系统本身：
  - 用户运行在容器里的应用进程，跟宿主机上的其他进程一样，都由宿主机操作系统统一管理，只不过这些被隔离的进程拥有额外设置过的 Namespace 参数。而 Docker 项目在这里扮演的角色，更多的是旁路式的辅助和管理工作。
  - “敏捷”和“高性能”是容器相较于虚拟机最大的优势，也是它能够在 PaaS 这种更细粒度的资源管理平台上大行其道的重要原因。
- 基于 Linux Namespace 的隔离机制相比于虚拟化技术也有很多不足之处，其中最主要的问题就是：隔离得不彻底
  - 既然容器只是运行在宿主机上的一种特殊的进程，那么多个容器之间使用的就还是同一个宿主机的操作系统内核。
  - 在 Linux 内核中，有很多资源和对象是不能被 Namespace 化的，最典型的例子就是：时间。
- 容器的“限制”问题
  - 还是以 PID Namespace 为例，虽然第 100 号进程表面上被隔离了起来，但是它所能够使用到的资源（比如 CPU、内存），却是可以随时被宿主机上的其他进程（或者其他容器）占用的，这些情况，显然都不是一个“沙盒”应该表现出来的合理行为。
  - 而Linux Cgroups 就是 Linux 内核中用来为进程设置资源限制的一个重要功能。
    - 它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。
    - 在 Linux 中，Cgroups 给用户暴露出来的操作接口是文件系统，即它以文件和目录的方式组织在操作系统的 /sys/fs/cgroup 路径下。
    - 在 /sys/fs/cgroup 下面有很多诸如 cpuset、cpu、 memory 这样的子目录，也叫子系统。这些都是我这台机器当前可以被 Cgroups 进行限制的资源种类。而在子系统对应的资源种类下，你就可以看到该类资源具体可以被限制的方法
    - Linux Cgroups 的设计还是比较易用的，简单粗暴地理解，它就是一个子系统目录加上一组资源限制文件的组合。
      - 而对于 Docker 等 Linux 容器项目来说，它们只需要在每个子系统下面，为每个容器创建一个控制组（即创建一个新目录）
      - 然后在启动容器进程之后，把这个进程的 PID 填写到对应控制组的 tasks 文件中就可以了。
- 容器本身的设计，就是希望容器和应用能够同生命周期，这个概念对后续的容器编排非常重要。否则，一旦出现类似于“容器是正常运行的，但是里面的应用早已经挂了”的情况，编排系统处理起来就非常麻烦了
- 另外，跟 Namespace 的情况类似，Cgroups 对资源的限制能力也有很多不完善的地方，被提及最多的自然是 /proc 文件系统的问题。
  - Linux 下的 /proc 目录存储的是记录当前内核运行状态的一系列特殊文件，用户可以通过访问这些文件，查看系统以及当前正在运行的进程的信息
  - 你如果在容器里执行 top 指令，就会发现，它显示的信息居然是宿主机的 CPU 和内存数据，而不是当前容器的数据。
  - 造成这个问题的原因就是，/proc 文件系统并不知道用户通过 Cgroups 给这个容器做了什么样的资源限制，即：/proc 文件系统不了解 Cgroups 限制的存在。





## 容器镜像（挂载rootfs）

- 容器里的进程看到的文件系统又是什么样子的呢
  - Mount Namespace 修改的，是容器进程对文件系统“挂载点”的认知。但是，这也就意味着，只有在“挂载”这个操作发生之后，进程的视图才会被改变。而在此之前，新创建的容器会直接继承宿主机的各个挂载点。
  - Mount Namespace ：它对容器进程视图的改变，一定是伴随着挂载操作（mount）才能生效。
  - 作为一个普通用户，我们希望的是一个更友好的情况：每当创建一个新容器时，我希望容器进程看到的文件系统就是一个独立的隔离环境，而不是继承自宿主机的文件系统。怎么才能做到这一点呢？
    - 在 Linux 操作系统里，有一个名为 chroot 的命令可以帮助你在 shell 中方便地完成这个工作。顾名思义，它的作用就是帮你“change root file system”，即改变进程的根目录到你指定的位置。
    - Mount Namespace 正是基于对 chroot 的不断改良才被发明出来的，它也是 Linux操作系统里的第一个 Namespace。
- 这个挂载在容器根目录上、用来为容器进程提供隔离后执行环境的文件系统，就是所谓的“容器镜像”。它还有一个更为专业的名字，叫作：rootfs（根文件系统）。
  - rootfs 只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。在 Linux 操作系统中，这两部分是分开存放的，操作系统只有在开机启动时才会加载指定版本的内核镜像。
  - 这也是容器相比于虚拟机的主要缺陷之一：毕竟后者不仅有模拟出来的硬件机器充当沙盒，而且每个沙盒里还运行着一个完整的 Guest OS 给应用随便折腾。
  - 不过，正是由于 rootfs 的存在，容器才有了一个被反复宣传至今的重要特性：一致性。
    - 由于 rootfs 里打包的不只是应用，而是整个操作系统的文件和目录，也就意味着，应用以及它运行所需要的所有依赖，都被封装在了一起。
    - 这就赋予了容器所谓的一致性：无论在本地、云端，还是在一台任何地方的机器上，用户只需要解压打包好的容器镜像，那么这个应用运行所需要的完整的执行环境就被重现出来了。
- 难道我每开发一个应用，或者升级一下现有的应用，都要重复制作一次 rootfs 吗？
  - Docker 在镜像的设计中，引入了层（layer）的概念。也就是说，用户制作镜像的每一步操作，都会生成一个层，也就是一个增量 rootfs。
  - 这个想法不是凭空臆造出来的，而是用到了一种叫作联合文件系统（Union File System）的能力。最主要的功能是将多个不同位置的目录联合挂载（union mount）到同一个目录下
- Docker 镜像使用的rootfs，往往由多个“层”组成：
  - 第一部分，只读层。
    - ro+wh，即 readonly+whiteout
    - 可以看到，这些层，都以增量的方式分别包含了 Ubuntu/centos 操作系统的一部分。
  - 第二部分，Init 层。
    - 它是一个以“-init”结尾的层，夹在只读层和读写层之间。Init 层是 Docker 项目单独生成的一个内部层，专门用来存放 /etc/hosts、/etc/resolv.conf 等信息。
    - 需要这样一层的原因是，这些文件本来属于只读的 Ubuntu 镜像的一部分，但是用户往往需要在启动容器时写入一些指定的值比如 hostname，所以就需要在可读写层对它们进行修改。
    - 可是，这些修改往往只对当前的容器有效，我们并不希望执行 docker commit 时，把这些信息连同可读写层一起提交掉。
    - 所以，Docker 做法是，在修改了这些文件之后，以一个单独的层挂载了出来。而用户执行docker commit 只会提交可读写层，所以是不包含这些内容的。
  - 第三部分，可读写层。
    - 专门用来存放你修改 rootfs 后产生的增量，无论是增、删、改，都发生在这里。
    - 如果我现在要做的，是删除只读层里的一个文件呢？
      - 为了实现这样的删除操作，AuFS 会在可读写层创建一个 whiteout 文件，把只读层里的文件“遮挡”起来。
  - 最终，这些层都被联合挂载到 /var/lib/docker/aufs/mnt 目录下，表现为一个完整的Ubuntu 操作系统供容器使用。







## 重新认识Docker容器（docker底层）

- Dockerfile使用一些标准的原语，描述我们所要构建的 Docker 镜像。并且这些原语，都是按顺序处理的。

- 容器技术使用了 rootfs 机制和 Mount Namespace，构建出了一个同宿主机完全隔离开的文件系统环境。

  - Volume 机制，允许你将宿主机上指定的目录或者文件，挂载到容器里面进行读取和修改操作。

  - > $ docker run -v /test ... 
    > $ docker run -v /home:/test ...
    > 在第一种情况下，由于你并没有显示声明宿主机目录，那么 Docker 就会默认在宿主机上创建一个临时目录 /var/lib/docker/volumes/[VOLUME_ID]/_data，然后把它挂载到容器的test 目录上。而在第二种情况下，Docker 就直接把宿主机的 /home 目录挂载到容器的 /test目录上。

- Docker 又是如何做到把一个宿主机上的目录或者文件，挂载到容器里面去呢

  - 当容器进程被创建之后，尽管开启了 Mount Namespace，但是在它执行 chroot（或者 pivot_root）之前，容器进程一直可以看到宿主机上的整个文件系统。
  - 宿主机上的文件系统，也自然包括了我们要使用的容器镜像
  - 这个镜像的各个层,在容器进程启动后，它们会被联合挂载在var/lib/docker/aufs/mnt/ 目录中,这样容器所需的 rootfs 就准备好了。
  - 所以，我们只需要在 rootfs 准备好之后，在执行 chroot 之前，把 Volume 指定的宿主机目录（比如 /home 目录），挂载到指定的容器目录（比如 /test 目录）在宿主机上对应的目录（即var/lib/docker/aufs/mnt/[可读写层 ID]/test）上，这个 Volume 的挂载工作就完成了。
  - 由于执行这个挂载操作时，“容器进程”已经创建了，也就意味着此时 MountNamespace 已经开启了。所以，这个挂载事件只在这个容器里可见。你在宿主机上，是看不见容器内部的这个挂载点的。这就保证了容器的隔离性不会被 Volume 打破
    - 这里提到的 " 容器进程 "，是 Docker 创建的一个容器初始化进程(dockerinit)，而不是应用进程 (ENTRYPOINT + CMD)。
    - dockerinit 会负责完成根目录的准备、挂载设备和目录、配置 hostname 等一系列需要在容器内进行的初始化操作
    - 最后，它通过 execv() 系统调用，让应用进程取代自己，成为容器里的 PID=1 的进程。

- 而这里要使用到的挂载技术，就是 Linux 的**绑定挂载（Bind Mount）机制**

  - 你在该挂载点上进行的任何操作，只是发生在被挂载的目录或者文件上，而原挂载点的内容则会被隐藏起来且不受影响。
  - 绑定挂载实际上是一个 inode 替换的过程。在Linux 操作系统中，inode 可以理解为存放文件内容的“对象”，而 dentry，也叫目录项，就是访问这个 inode 所使用的“指针”。
  - mount --bind /home /test，会将 /home 挂载到 /test 上。其实相当于将test 的 dentry，重定向到了 /home 的 inode。
    - 当我们修改 /test 目录时，实际修改的是home 目录的 inode。这也就是为何，一旦执行 umount 命令，/test 目录原先的	内容就会恢复：因为修改真正发生在的，是 /home 目录里。
    - 可以成功地将一个宿主机上的目录或文件，不动声色地挂载到容器中。
    - 容器的镜像操作，比如 docker commit，都是发生在宿主机空间的。而由于 Mount Namespace 的隔离作用，宿主机并不知道这个绑定挂载的存在。所以，在宿主机看来，容器中可读写层的 /test 目录（/var/lib/docker/aufs/mnt/[可读写层ID]/test），始终是空的。
    - 由于 Docker 一开始还是要创建 /test 这个目录作为挂载点，所以执行了 docker commit 之后，你会发现新产生的镜像里，会多出来一个空的 /test 目录。







## Kubernetes的本质（架构）

- 一个正在运行的 Linux 容器，其实可以被“一分为二”地看待：
  - 一组联合挂载在 /var/lib/docker/aufs/mnt 上的 rootfs，这一部分我们称为“容器镜像”（Container Image），是容器的静态视图；
  - 一个由 Namespace+Cgroups 构成的隔离环境，这一部分我们称为“容器运行时”（Container Runtime），是容器的动态视图。
- 在整个“开发 - 测试 - 发布”的流程中，真正承载着容器信息进行传递的，是容器镜像，而不是容器运行时。
- **Kubernetes 项目要解决的问题是什么**？
  - 对于大多数用户来说，他们希望 Kubernetes 项目带来的体验是确定的：现在我有了应用的容器镜像，请帮我在一个给定的集群上把这个应用运行起来。
  - 更进一步地说，我还希望 Kubernetes 能给我提供路由网关、水平扩展、监控、备份、灾难恢复等一系列运维能力。
- **Kubernetes 项目的架构**
  - Master 和 Node两种节点组成，而这两种角色分别对应着控制节点和计算节点。
  - Master 节点由三个紧密协作的独立组件组合而成
    - 负责 API 服务的 kube-apiserver
    - 负责调度的 kube-scheduler
    - 以及负责容器编排的 kube-controller-manager。
    - 整个集群的持久化数据，则由 kube-apiserver 处理后保存在 Ectd 中。
  - 计算节点上最核心的部分，则是一个叫作 kubelet 的组件。
    - kubelet 主要负责同容器运行时（比如 Docker 项目）打交道。而这个交互所依赖的，是一个称作 CRI（Container Runtime Interface）的远程调用接口，这个接口定义了容器运行时的各项核心操作，比如：启动一个容器需要的所有参数。
      - 这也是为何，Kubernetes 项目并不关心你部署的是什么容器运行时、使用的什么技术实现，只要你的这个容器运行时能够运行标准的容器镜像，它就可以通过实现 CRI 接入到 Kubernetes 项目当中。
    - 具体的容器运行时，比如 Docker 项目，则一般通过 OCI 这个容器运行时规范同底层的 Linux 操作系统进行交互，即：把 CRI 请求翻译成对 Linux 操作系统的调用
    - 而kubelet 的另一个重要功能，则是调用网络插件和存储插件为容器配置网络和持久化存储。（CNI（Container Networking Interface）、CSI（Container Storage Interface））
    - kubelet 完全就是为了实现 Kubernetes 项目对容器的管理能力而重新实现的一个组件
  - 如何编排、管理、调度用户提交的作业？
    - 运行在大规模集群中的各种任务之间，实际上存在着各种各样的关系。这些关系的处理，才是作业编排和管理系统最困难的地方。
    - Kubernetes 项目对容器间的“访问”进行了分类，首先总结出了一类非常常见的“紧密交互”的关系，
      - Pod 是 Kubernetes 项目中最基础的一个对象
      - 在 Kubernetes 项目中，这些容器则会被划分为一个“Pod”，Pod 里的容器共享同一个 Network Namespace、同一组数据卷，从而达到高效率交换信息的目的。
- 围绕着容器和 Pod 不断向真实的技术场景扩展，我们就能够摸索出一幅以Kubernetes 项目核心功能的“全景图”。
  - 首先遇到了容器间“紧密协作”关系的难题，于是就扩展到了 Pod；
  - 有了 Pod 之后，我们希望能一次启动多个应用的实例，这样就需要Deployment 这个 Pod 的多实例管理器；
  - 而有了这样一组相同的 Pod 后，我们又需要通过一个固定的 IP 地址和端口以负载均衡的方式访问它，于是就有了 Service。
  - 如果现在两个不同 Pod 之间不仅有“访问关系”，还要求在发起时加上授权信息，Kubernetes 项目提供了一种叫作 Secret 的对象，它其实是一个保存在 Etcd 里的键值对数据
  - 除了应用与应用之间的关系外，应用运行的形态是影响“如何容器化这个应用”的第二个重要因素。
    - Kubernetes 定义了新的、基于 Pod 改进后的对象。比如 Job，用来描述一次性运行的Pod（比如，大数据任务）；
    - 再比如 DaemonSet，用来描述每个宿主机上必须且只能运行一个副本的守护进程服务；
    - 又比如 CronJob，则用于描述定时任务等等。
  - 相比之下，在 Kubernetes 项目中，我们所推崇的使用方法是：
    - 首先，通过一个“编排对象”，比如 Pod、Job、CronJob 等，来描述你试图管理的应用；
    - 然后，再为它定义一些“服务对象”，比如 Service、Secret、Horizontal Pod Autoscaler（自动水平扩展器）等。这些对象，会负责具体的平台级功能。
    - 这种使用方法，就是所谓的“声明式 API”。这种 API 对应的“编排对象”和“服务对象”，都是Kubernetes 项目中的 API 对象（API Object）。
    - 这就是 Kubernetes 最核心的设计理念
    - Kubernetes 项目所擅长的，是按照用户的意愿和整个系统的规则，完全自动化地处理好容器之间的各种关系。这种功能，就是我们经常听到的一个概念：编排。



## 版本号

Kubernetes版本表示为xyz，其中x是主要版本，y是次要版本，z是补丁版本，遵循[语义版本控制术语](https://link.zhihu.com/?target=http%3A//semver.org/)。有关更多信息，请参阅[Kubernetes发布版本](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/community/blob/master/contributors/design-proposals/release/versioning.md%23kubernetes-release-versioning)。

- k8s 发行版与 github 分支的关系
  简单来讲，kubernetes项目存在3类分支(branch)，分别为`master`，`release-X.Y`,`release-X.Y.Z`。 master分支上的代码是最新的，每隔2周会生成一个发布版本(release)，由新到旧以此为`master`-->`alpha`-->`beta`-->`Final release`，这当中存在一些cherry picking的规则，也就是说从一个分支上挑选一些必要pull request应用到另一个分支上。 我们可以认为`X.Y.0`为稳定的版本，这个版本号意味着一个`Final release`。一个`X.Y.0`版本会在`X.(Y-1).0`版本的3到4个月后出现。 `X.Y.Z`为经过cherrypick后解决了必须的安全性漏洞、以及影响大量用户的无法解决的问题的补丁版本。 总体而言，我们一般关心`X.Y.0`(稳定版本)，和`X.Y.Z`(补丁版本)的特性。
- 例子
  `v1.14.0` : `1`为主要版本 : `14`为次要版本 : `0`为补丁版本



## kubeadm

- **kubeadm这个项目的目的，就是要让用户能够通过这样两条指令完成一个 Kubernetes 集群的部署**

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

- kubeadm 目前最欠缺的是，一键部署一个高可用的 Kubernetes 集群，即：Etcd、Master 组件都应该是多节点集群，而不是现在这样的单点。这，当然也正是 kubeadm 接下来发展的主要方向。

- 在 Bare-metal 环境下使用 kubeadm 工具部署了一个完整的Kubernetes 集群：这个集群有一个 Master 节点和多个 Worker 节点；使用 Weave 作为容器网络插件；使用 Rook 作为容器持久化存储插件；使用 Dashboard 插件提供了可视化的 Web 界面。





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

- **通过 Taint/Toleration 调整 Master 执行 Pod 的策略**

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

  

  







## 容器化应用（代码）

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







## Pod使用（字段用法）

- 到底哪些属性属于 Pod 对象，而又有哪些属性属于 Container 呢？

  - 凡是调度、网络、存储，以及安全相关的属性，基本上是 Pod 级别的。这些属性的共同特征是，它们描述的是“机器”这个整体，而不是里面运行的“程序”

- Pod 中几个重要字段的含义和用法

  - NodeSelector：是一个供用户将 Pod 与 Node 进行绑定的字段
  - HostAliases：定义了 Pod 的 hosts 文件（比如 /etc/hosts）里的内容
  - shareProcessNamespace=true，意味着这个 Pod 里的容器要共享 PID Namespace。
  - 凡是 Pod 中的容器要共享宿主机的 Namespace，也一定是 Pod 级别的定义
    - spec: hostNetwork: true hostIPC: true hostPID: true
    - 在这个 Pod 中，我定义了共享宿主机的 Network、IPC 和 PID Namespace。这就意味着，这个Pod 里的所有容器，会直接使用宿主机的网络、直接与宿主机进行 IPC 通信、看到宿主机里正在运行的所有进程。

- Pod 里最重要的字段当属“Containers”了

  - Image（镜像）、Command（启动命令）、workingDir（容器的工作目录）、Ports（容器要开发的端口），以及 volumeMounts（容器要挂载的 Volume）都是构成 Kubernetes 项目中 Container 的主要字段。
  - ImagePullPolicy 字段。它定义了镜像拉取的策略。
    - ImagePullPolicy 的值默认是 Always，即每次创建 Pod 都重新拉取一次镜像
    - 而如果它的值被定义为 Never 或者 IfNotPresent，则意味着 Pod 永远不会主动拉取这个镜像，或者只在宿主机上不存在这个镜像时才拉取。
  - Lifecycle 字段
    - 在容器状态发生变化时触发一系列“钩子”
    - postStart 。它指的是，在容器启动后，立刻执行一个指定的操作
    - preStop 发生的时机，则是容器被杀死之前（比如，收到了 SIGKILL 信号）。而需要明确的是，preStop 操作的执行，是同步的。所以，它会阻塞当前的容器杀死流程，直到这个 Hoo定义操作完成之后，才允许容器被杀死，这跟 postStart 不一样。

- **Pod 对象在 Kubernetes 中的生命周期**

  - Pod 生命周期的变化，主要体现在 Pod API 对象的Status 部分。这是它除了 Metadata 和 Spec之外的第三个重要字段。其中pod.status.phase，就是 Pod 的当前状态
    - Pending 这个 Pod 里有些容器因为某种原因而不能被顺利创建。比如，调度不成功。
    - Running Pod 已经调度成功
    - Succeeded Pod 里的所有容器都正常运行完毕，并且已经退出了
    - Failed Pod 里至少有一个容器以不正常的状态（非 0 的返回码）退出
    - Unknown 意味着 Pod 的状态不能持续地被 kubelet 汇报给 kubeapiserver，这很有可能是主从节点（Master 和 Kubelet）间的通信出现了问题。
  - 更进一步地，Pod 对象的 Status 字段，还可以再细分出一组 Conditions。
    - PodScheduled、Ready、Initialized，以及 Unschedulable
    - Ready 这个细分状态非常值得我们关注：它意味着 Pod 不仅已经正常启动（Running 状态），而且已经可以对外提供服务了。这两者之间（Running 和 Ready）是有区别的，你不妨仔细思考一下。

- **Projected Volume，“投射数据卷”。**

  - 是为容器提供预先定义好的数据。所以，从容器的角度来看，这些 Volume 里的信息就是仿佛是被 Kubernetes“投射”（Project）进入容器当中的
  - Secret；

  2. ConfigMap；

  - Downward API；

    3. 让 Pod 里的容器能够直接获取到这个 Pod API 对象本身的信息
    4. 需要注意的是，Downward API 能够获取到的信息，一定是 Pod 里的容器进程启动之前就能够确定下来的信息。而如果你想要获取 Pod 容器运行后才会出现的信息，比如，容器进程的PID，那就肯定不能使用 Downward API 了，而应该考虑在 Pod 里定义一个 sidecar 容器。
    5. 其实，Secret、ConfigMap，以及 Downward API 这三种 Projected Volume 定义的信息，大多还可以通过环境变量的方式出现在容器里。但是，通过环境变量获取这些信息的方式，不具备自动更新的能力。所以，一般情况下，我都建议你使用 Volume 文件的方式获取这些信息。

  - ServiceAccountToken

    4. Service Account 对象的作用，就是 Kubernetes 系统内置的一种“服务账户”，它是 Kubernetes进行权限分配的对象
    5. Service Account 的授权信息和文件，实际上保存在它所绑定的一个特殊的 Secret 对象里的。这个特殊的 Secret 对象，就叫作ServiceAccountToken

    - 另外，为了方便使用，Kubernetes 已经为你提供了一个的默认“服务账户”（default ServiceAccount）。并且，任何一个运行在 Kubernetes 里的 Pod，都可以直接使用这个默认的 ServiceAccount，而无需显示地声明挂载它。
      4. 所以说，Kubernetes 其实在每个 Pod 创建的时候，自动在它的 spec.volumes 部分添加上了默认ServiceAccountToken 的定义，然后自动给每个容器加上了对应的 volumeMounts 字段

- **容器健康检查和恢复机制**

  - 健康检查“探针”（Probe）
    4. kubelet 就会根据这个 Probe 的返回值决定这个容器的状态
  - livenessProbe（健康检查）
    4. 在容器启动 5 s 后开始执行（initialDelaySeconds: 5）
    5. 每 5 s 执行一次（periodSeconds: 5）。

- **Pod 恢复机制，也叫 restartPolicy**

  - 但一定要强调的是，Pod 的恢复过程，永远都是发生在当前节点上

  - 而如果你想让 Pod 出现在其他的可用节点上，就必须使用 Deployment 这样的“控制器”来管理Pod，哪怕你只需要一个 Pod 副本。

  - 而作为用户，你还可以通过设置 restartPolicy，改变 Pod 的恢复策略。除了 Always，它还有OnFailure 和 Never 两种情况：

    - > Always：在任何情况下，只要容器不在运行状态，就自动重启容器；
      > OnFailure: 只在容器 异常时才自动重启容器；
      > Never: 从来不重启容器。

    - 比如，一个 Pod，它只计算 1+1=2，计算完成输出结果后退出，变成 Succeeded 状态。这时，你如果再用 restartPolicy=Always 强制重启这个 Pod 的容器，就没有任何意义了。







## “控制器”模型（kube-controller-manager ）

- ```
  apiVersion: apps/v1 
  kind: Deployment 
  metadata: 
  	name: nginx-deployment 
  spec: 
  	selector: 
  		matchLabels: 
  			app: nginx 
  	replicas: 2 
  	template: 
  		metadata: 
  			labels: 
  				app: nginx 
  		spec: 
  			containers: 
  			- name: nginx 
  				image: nginx:1.7.9 
  				ports: 
  				- containerPort: 80
  ```
  
  
  
- **kube-controller-manager 的组件**
  
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





## Deployment 作业副本与水平扩展

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







## StatefulSet拓扑状态（Headless Service）

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

- **Headless Service**

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
    apiVersion: v1 
    kind: Service 
    metadata: 
    	name: nginx 
    	labels: 
    		app: nginx
    spec:
     ports:
      - port: 80c
        name: web
        clusterIP: None
        selector:
         app: nginx
    ```

- StatefulSet 又是如何使用这个 DNS 记录来维持 Pod 的拓扑状态的呢？

  - ```
    apiVersion: apps/v1 
    kind: StatefulSet 
    metadata: 
    	name: web 
    spec: 
    	serviceName: "nginx" 
    	replicas: 2 
    	selector: 
    		matchLabels: 
    			app: nginx 
    	template: 
    		metadata: 
    			labels:
    				app: nginx 
    		spec: 
    			containers: 
    			- name: nginx 
    			image: nginx:1.9.1 
    			ports: 
    			- containerPort: 80 
    				name: web
    ```

    

  - serviceName=nginx 字段告诉 StatefulSet 控制器，在执行控制循环（Control Loop）的时候，请使用 nginx 这个 Headless Service 来保证 Pod 的“可解析身份”。

  - StatefulSet 给它所管理的所有 Pod 的名字，进行了编号，编号规则是：-

- StatefulSet 这个控制器的主要作用之一，就是使用 Pod 模板创建 Pod 的时候，对它们进行编号，并且按照编号顺序逐一完成创建工作。

  - 而当 StatefulSet 的“控制循环”发现 Pod 的“实际状态”与“期望状态”不一致，需要新建或者删除Pod 进行“调谐”的时候，它会严格按照这些 Pod 编号的顺序，逐一完成这些操作
  - 任何pod的变化都会触发一次statefulset的滚动更新。
  - 在部署“有状态应用”的时候，应用的每个实例拥有唯一并且稳定的“网络标识”，是一个非常重要的假设。







## StatefulSet存储状态（PV、PVC）

- Kubernetes 项目引入了一组叫作 Persistent VolumeClaim（PVC）和 Persistent Volume（PV）的 API 对象，大大降低了用户声明和使用持久化Volume 的门槛。
- PVC
  - 第一步：定义一个 PVC，声明想要的 Volume 的属性：
    
    - storage: 1Gi，表示我想要的 Volume 大小至少是 1 GB；accessModes:ReadWriteOnce，表示这个 Volume 的挂载方式是可读写，并且只能被挂载在一个节点上而非被多个节点共享。
    
    - ```
      kind: PersistentVolumeClaim 
      apiVersion: v1 
      metadata: 
      	name: pv-claim 
      spec: 
      	accessModes: 
      	- ReadWriteOnce 
      	resources: 
      		requests: 
      			storage: 1Gi
      ```
    
  - 第二步：在应用的 Pod 中，声明使用这个 PVC：
    - 在这个 Pod 的 Volumes 定义中，我们只需要声明它的类型是persistentVolumeClaim，然后指定 PVC 的名字，而完全不必关心 Volume 本身的定义。
    
    - 只要我们创建这个 PVC 对象，Kubernetes 就会自动为它绑定一个符合条件的Volume。可是，这些符合条件的 Volume 来自于由运维人员维护的 PV（Persistent Volume）对象。
    
    - ```
      apiVersion: v1 
      kind: Pod 
      metadata: 
      	name: pv-pod 
      spec:
      	containers: 
      		- name: pv-container 
      			image: nginx 
      			ports: 
      				- containerPort: 80 
      					name: "http-server" 
      			volumeMounts: - mountPath: "/usr/share/nginx/html" 
      										name: pv-storage 
      volumes: 
      	- name: pv-storage 
      		persistentVolumeClaim: 
      			claimName: pv-claim
      ```
    
  
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







## DaemonSet（金丝雀发布、灰度发布）

- **金丝雀发布（Canary Deploy）或者灰度发布（希望只升级 StatefulSet 中的任意个节点进行测试,）**

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



## Operator工作原理

- 在 Kubernetes 生态中，还有一个相对更加灵活和编程友好的管理“有状态应用”的解决方案，它就是：Operator。

- 以 Etcd Operator 为例，来为你讲解一下 Operator 的工作原理和编写方法

  - 第一步，将这个 Operator 的代码 Clone 到本地：
  - 第二步，将这个 Etcd Operator 部署在 Kubernetes 集群里

- 在部署 Etcd Operator 的 Pod 之前，你需要先执行这样一个脚本

  - $ example/rbac/create_role.sh

  - 这个脚本的作用，就是为 Etcd Operator 创建 RBAC 规则。这是因

    为，Etcd Operator 需要访问 Kubernetes 的 APIServer 来创建对象。

  - 更具体地说，上述脚本为 Etcd Operator 定义了如下所示的权限：

    1. 对 Pod、Service、PVC、Deployment、Secret 等 API 对象，有所有权限；

    2. 对 CRD 对象，有所有权限；

    3. 对属于 etcd.database.coreos.com 这个 API Group 的 CR（Custom Resource）对象，有所有权限。

- Etcd Operator 本身，其实就是一个 Deployment

  - 一旦 Etcd Operator 的 Pod 进入了 Running 状态，你就会发现，有一个 CRD 被自动创建了出来
  - 这个 CRD 名叫etcdclusters.etcd.database.coreos.com 。你可以通过 kubectldescribe 命令看到它的细节
  - 这个 CRD 相当于告诉了 Kubernetes：接下来，如果有 API 组（Group）是etcd.database.coreos.com、API 资源类型（Kind）是“EtcdCluster”的 YAML 文件被提交上来，你可一定要认识啊

- 通过上述两步操作，你实际上是在 Kubernetes 里添加了一个名叫 EtcdCluster 的自定义资源类型。而 Etcd Operator 本身，就是这个自定义资源类型对应的自定义控制器

- 当 Etcd Operator 部署好之后，接下来在这个 Kubernetes 里创建一个 Etcd 集群的工作就非常简单了。你只需要编写一个 EtcdCluster 的 YAML 文件，然后把它提交给 Kubernetes 即可

  - 这个 example-etcd-cluster.yaml 文件里描述的，是一个 3 个节点的 Etcd 集群。我们可以看到它被提交给 Kubernetes 之后，就会有三个 Etcd 的 Pod 运行起来

- 从这个 example-etcd-cluster.yaml 文件开始说起

  - 这个文件里定义的，正是 EtcdCluster 这个 CRD 的一个具体实例，也就是一个Custom Resource（CR）。而它的内容非常简单
  - EtcdCluster 的 spec 字段非常简单。其中，size=3 指定了它所描述的 Etcd 集群的节点个数。而 version=“3.2.13”，则指定了 Etcd 的版本，仅此而已。
  - **而真正把这样一个 Etcd 集群创建出来的逻辑，就是 Etcd Operator 要实现的主要工作了。**

- Operator 的工作原理，实际上是利用了 Kubernetes 的自定义 API 资源（CRD），来描述我们想要部署的“有状态应用”；然后在自定义控制器里，根据自定义 API 对象的变化，来完成具体的部署和运维工作。

- Etcd Operator 部署 Etcd 集群，采用的是静态集群（Static）的方式。

  - 静态集群的好处是，它不必依赖于一个额外的服务发现机制来组建集群，非常适合本地容器化部署。而它的难点，则在于你必须在部署的时候，就规划好这个集群的拓扑结构，并且能够知道这些节点固定的 IP 地址。

  - ```
    $ etcd --name infra0 --initial-advertise-peer-urls http://10.0.1.10:2380 \
     --listen-peer-urls http://10.0.1.10:2380 \
    ...
     --initial-cluster-token etcd-cluster-1 \
     --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0
     --initial-cluster-state new
     
     10.0.1.10 11 12 启三个
    ```

  - 启动了三个 Etcd 进程，组成了一个三节点的 Etcd 集群

  - 这些节点启动参数里的–initial-cluster 参数，非常值得你关注。它的含义，正是当前节点启动时集群的拓扑结构。说得更详细一点，就是当前这个节点启动时，需要跟哪些节点通信来组成集群。

    - 可以看到，–initial-cluster 参数是由“< 节点名字 >=< 节点地址 >”格式组成的一个数组。而上面这个配置的意思就是，当 infra2 节点启动之后，这个 Etcd 集群里就会有 infra0、infra1和 infra2 三个节点。
    - 同时，这些 Etcd 节点，需要通过 2380 端口进行通信以便组成集群，这也正是上述配置中–listen-peer-urls 字段的含义。
    - 此外，一个 Etcd 集群还需要用–initial-cluster-token 字段，来声明一个该集群独一无二的Token 名字。

  - 而我们要编写的 Etcd Operator，就是要把上述过程自动化。这其实等同于：用代码来生成每个Etcd 节点 Pod 的启动命令，然后把它们启动起来

- 当然，在编写自定义控制器之前，我们首先需要完成 EtcdCluster 这个 CRD 的定义，它对应的types.go 文件的主要内容

  - 只关注 Size（即：Etcd 集群的大小）字段即可。
  - Size 字段的存在，就意味着将来如果我们想要调整集群大小的话，应该直接修改 YAML 文件里size 的值，并执行 kubectl apply -f。
  - 这样，Operator 就会帮我们完成 Etcd 节点的增删操作。这种“scale”能力，也是 EtcdOperator 自动化运维 Etcd 集群需要实现的主要功能
  - 而为了能够支持这个功能，我们就不再像前面那样在–initial-cluster 参数里把拓扑结构固定死。

- 所以，Etcd Operator 的实现，虽然选择的也是静态集群，但这个集群具体的组建过程，是逐个节点动态添加的方式

  - 首先，Etcd Operator 会创建一个“种子节点”；
  - 然后，Etcd Operator 会不断创建新的 Etcd 节点，然后将它们逐一加入到这个集群当中，直到集群的节点数等于 size。
  - 而这两种节点的不同之处，就在于一个名叫–initial-cluster-state 的启动参数：
    - 当这个参数值设为 new 时，就代表了该节点是种子节点。而我们前面提到过，种子节点还必须通过–initial-cluster-token 声明一个独一无二的 Token。
    - 而如果这个参数值设为 existing，那就是说明这个节点是一个普通节点，Etcd Operator 需要把它加入到已有集群里。

- 那么接下来的问题就是，每个 Etcd 节点的–initial-cluster 字段的值又是怎么生成的呢？

  - 由于这个方案要求种子节点先启动，所以对于种子节点 infra0 来说，它启动后的集群只有它自己，即：–initial-cluster=infra0=http://10.0.1.10:2380。
  - 而对于接下来要加入的节点，比如 infra1 来说，它启动后的集群就有两个节点了，所以它的–initial-cluster 参数的值应该是：infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380。
  - 其他节点，都以此类推。

- 现在，你就应该能在脑海中构思出上述三节点 Etcd 集群的部署过程了

  - 首先，只要用户提交 YAML 文件时声明创建一个 EtcdCluster 对象（一个 Etcd 集群），那么Etcd Operator 都应该先创建一个单节点的种子集群（Seed Member），并启动这个种子节点。

  - 接下来，对于其他每一个节点，Operator 只需要执行如下两个操作即可，以 infra1 为例

    - 第一步：通过 Etcd 命令行添加一个新成员：

      - $ etcdctl member add infra1 http://10.0.1.11:2380

    - 第二步：为这个成员节点生成对应的启动参数，并启动它：

      - ```
        $ etcd
         --data-dir=/var/etcd/data
         --name=infra1
         --initial-advertise-peer-urls=http://10.0.1.11:2380
         --listen-peer-urls=http://0.0.0.0:2380
         --listen-client-urls=http://0.0.0.0:2379
         --advertise-client-urls=http://10.0.1.11:2379
         --initial-cluster=infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380
         --initial-cluster-state=existing
        ```

    - 可以看到，对于这个 infra1 成员节点来说，它的 initial-cluster-state 是 existing，也就是要加入已有集群。而它的 initial-cluster 的值，则变成了 infra0 和 infra1 两个节点的 IP 地址。

    - 所以，以此类推，不断地将 infra2 等后续成员添加到集群中，直到整个集群的节点数目等于用户指定的 size 之后，部署就完成了。

- **Etcd Operator 的工作原理**

  - 跟所有的自定义控制器一样，Etcd Operator 的启动流程也是围绕着 Informer 展开的
  - Etcd Operator 启动要做的第一件事（ c.initResource），是创建 EtcdCluster 对象所需要的 CRD，即：前面提到的etcdclusters.etcd.database.coreos.com。这样Kubernetes 就能够“认识”EtcdCluster 这个自定义 API 资源了。
  - 而接下来，Etcd Operator 会定义一个 EtcdCluster 对象的 Informer。
  - 具体来讲，我们在控制循环里执行的业务逻辑，往往是比较耗时间的。比如，创建一个真实的Etcd 集群。而 Informer 的 WATCH 机制对 API 对象变化的响应，则非常迅速。所以，控制器里的业务逻辑就很可能会拖慢 Informer 的执行周期，甚至可能 Block 它。而要协调这样两个快、慢任务的一个典型解决方法，就是引入一个工作队列。
  - 由于 Etcd Operator 里没有工作队列，那么在它的 EventHandler 部分，就不会有什么入队操作，而直接就是每种事件对应的具体的业务逻辑了。
  - 不过，Etcd Operator 在业务逻辑的实现方式上，与常规的自定义控制器略有不同。
    - Etcd Operator 的特殊之处在于，它为每一个 EtcdCluster 对象，都启动了一个控制循环，“并发”地响应这些对象的变化。显然，这种做法不仅可以简化 Etcd Operator 的代码实现，还有助于提高它的响应速度。

- 以文章一开始的 example-etcd-cluster 的 YAML 文件为例。

  - 当这个 YAML 文件第一次被提交到 Kubernetes 之后，Etcd Operator 的 Informer，就会立刻“感知”到一个新的 EtcdCluster 对象被创建了出来。所以，EventHandler 里的“添加”事件会被触发。
  - 而这个 Handler 要做的操作也很简单，即：在 Etcd Operator 内部创建一个对应的 Cluster 对象（cluster.New）
    - 这个 Cluster 对象，就是一个 Etcd 集群在 Operator 内部的描述，所以它与真实的 Etcd 集群的生命周期是一致的。
  - 而一个 Cluster 对象需要具体负责的，其实有两个工作。
    - 其中，第一个工作只在该 Cluster 对象第一次被创建的时候才会执行。这个工作，就是我们前面提到过的 Bootstrap，即：创建一个单节点的种子集群。
      - 由于种子集群只有一个节点，所以这一步直接就会生成一个 Etcd 的 Pod 对象。这个 Pod 里有一个 InitContainer，负责检查 Pod 的 DNS 记录是否正常。如果检查通过，用户容器也就是Etcd 容器就会启动起来。
    - Cluster 对象的第二个工作，则是启动该集群所对应的控制循环。
      - 首先，控制循环要获取到所有正在运行的、属于这个 Cluster 的 Pod 数量，也就是该 Etcd 集群的“实际状态”。
      - 而这个 Etcd 集群的“期望状态”，正是用户在 EtcdCluster 对象里定义的 size。所以接下来，控制循环会对比这两个状态的差异。

- 第一个问题是，在 StatefulSet 里，它为 Pod 创建的名字是带编号的，这样就把整个集群的拓扑状态固定了下来（比如：一个三节点的集群一定是由名叫 web-0、web-1 和 web-2 的三个Pod 组成）。可是，在 Etcd Operator 里，为什么我们使用随机名字就可以了呢？

  - 这是因为，Etcd Operator 在每次添加 Etcd 节点的时候，都会先执行 etcdctl member add<Pod 名字 >；每次删除节点的时候，则会执行 etcdctl member remove <Pod 名字 >。这些操作，其实就会更新 Etcd 内部维护的拓扑信息，所以 Etcd Operator 无需在集群外部通过编号来固定这个拓扑关系。

- 第二个问题是，为什么我没有在 EtcdCluster 对象里声明 Persistent Volume？难道，我们不担心节点宕机之后 Etcd 的数据会丢失吗？

  - 我们知道，Etcd 是一个基于 Raft 协议实现的高可用 Key-Value 存储。根据 Raft 协议的设计原则，当 Etcd 集群里只有半数以下（在我们的例子里，小于等于一个）的节点失效时，当前集群依然可以正常工作。此时，Etcd Operator 只需要通过控制循环创建出新的 Pod，然后将它们加入到现有
  - 但是，当这个 Etcd 集群里有半数以上（在我们的例子里，大于等于两个）的节点失效的时候，这个集群就会丧失数据写入的能力，从而进入“不可用”状态。此时，即使 Etcd Operator 创建出新的 Pod 出来，Etcd 集群本身也无法自动恢复起来。
  - 这个时候，我们就必须使用 Etcd 本身的备份数据来对集群进行恢复操作。在有了 Operator 机制之后，上述 Etcd 的备份操作，是由一个单独的 Etcd Backup Operator负责完成的。







## Job与CronJob

- Deployment、StatefulSet，以及 DaemonSet 这三个编排概念。
  - 实际上，它们主要编排的对象，都是“在线业务”，即：Long Running Task（长作业）。比如，我在前面举例时常用的 Nginx、Tomcat，以及 MySQL 等等。这些应用一旦运行起来，除非出错或者停止，它的容器进程会一直保持在 Running 状态。
  - 有一类作业显然不满足这样的条件，这就是“离线业务”，或者叫作 Batch Job（计算业务），计算完成后就直接退出了
  - 描述离线业务的 API 对象，它的名字就是：Job
- TODO



## Pod 弹性伸缩详解与使用

- Kubernetes HPA(Horizontal Pod Autoscaling)Pod水平自动伸缩，通过此功能，只需简单的配置，集群便可以利用监控指标（cpu使用率等）自动的扩容或者缩容服务中Pod数量，当业务需求增加时，系统将为您无缝地自动增加适量容器 ，提高系统稳定性。

- ## 1. HPA概览

  ![img](https://blog-10039692.file.myqcloud.com/1499234415756_6524_1499234415663.png)

  HPA在kubernetes中被设计为一个controller，可以简单的使用kubectl autoscale命令来创建。HPA Controller默认30秒轮询一次，查询指定的resource中（Deployment,RC）的资源使用率，并且与创建时设定的值和指标做对比，从而实现自动伸缩的功能。

  - 当你创建了HPA后，HPA会从Heapster或者用户自定义的RESTClient获取定义的资源中每一个pod利用率或原始值（取决于指定的目标类型）的平均值，然后和HPA中定义的指标进行对比，同时计算出需要伸缩的具体值并进行操作。
  - **当Pod没有设置request时，HPA不会工作。**
  - 目前，HPA可以从两种取到获取数据:
    - Heapster(稳定版本，仅支持CPU使用率，在使用腾讯云[容器服务](https://cloud.tencent.com/product/tke?from=10680)时，需要手动安装)。
    - 自定义的监控(alpha版本，不推荐用于生产环境) 。
  - 当需要从自定义的监控中获取数据时，只能设置绝对值，无法设置使用率。
  - 现在只支持Replication Controller, Deployment or Replica Set的扩缩容。

  ## 2. 自动伸缩算法

  - HPA Controller会通过调整副本数量使得CPU使用率尽量向期望值靠近，而且不是完全相等．另外，官方考虑到自动扩展的决策可能需要一段时间才会生效：例如当pod所需要的CPU负荷过大，从而在创建一个新pod的过程中，系统的CPU使用量可能会同样在有一个攀升的过程。所以，在每一次作出决策后的一段时间内，将不再进行扩展决策。对于扩容而言，这个时间段为3分钟，缩容为5分钟。
  - HPA Controller中有一个tolerance（容忍力）的概念，它允许一定范围内的使用量的不稳定，现在默认为0.1，这也是出于维护系统稳定性的考虑。例如，设定HPA调度策略为cpu使用率高于50%触发扩容，那么只有当使用率大于55%或者小于45%才会触发伸缩活动，HPA会尽力把Pod的使用率控制在这个范围之间。
  - 具体的每次扩容或者缩容的多少Pod的算法为： `        Ceil(前采集到的使用率 / 用户自定义的使用率) * Pod数量) `
  - 每次最大扩容pod数量不会超过当前副本数量的2倍

  ## 3. HPA YAML文件详解

  下面是一个标准的基于heapster的HPA YAML文件，同时也补充了关键字段的含义

  ```javascript
  apiVersion: autoscaling/v1
  kind: HorizontalPodAutoscaler
  metadata:
    creationTimestamp: 2017-06-29T08:04:08Z
    name: nginxtest
    namespace: default
    resourceVersion: "951016361"
    selfLink: /apis/autoscaling/v1/namespaces/default/horizontalpodautoscalers/nginxtest
    uid: 86febb63-5ca1-11e7-aaef-5254004e79a3
  spec:
    maxReplicas: 5 //资源最大副本数
    minReplicas: 1 //资源最小副本数
    scaleTargetRef:
      apiVersion: extensions/v1beta1
      kind: Deployment //需要伸缩的资源类型
      name: nginxtest  //需要伸缩的资源名称
    targetCPUUtilizationPercentage: 50 //触发伸缩的cpu使用率
  status:
    currentCPUUtilizationPercentage: 48 //当前资源下pod的cpu使用率
    currentReplicas: 1 //当前的副本数
    desiredReplicas: 2 //期望的副本数
    lastScaleTime: 2017-07-03T06:32:19Z
  ```

  ## 4. 如何使用

  - 在上文的介绍中我们知道，HPA Controller有两种途径获取监控数据：Heapster和自定义监控，由于自定义监控一直处于alpha阶段，所以本文这次主要介绍在腾讯云容器服务中使用基于Heapster的HPA方法。
  - 腾讯云容器服务没有默认安装Heapster，所以如果需要使用HPA需要手动安装。
  - 此方法中需要使用kubectl命令操作集群，集群apiservice地址，账号和证书相关信息暂时可以提工单申请，相关功能的产品化方案已经在设计中。

  ## 4.1 创建Heapster

  创建Heapster ServiceAccount

  ```javascript
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: heapster
    namespace: kube-system
    labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
  ```

  创建Heapster deployment

  ```javascript
  apiVersion: extensions/v1beta1
  kind: Deployment
  metadata:
    name: heapster-v1.4.0-beta.0
    namespace: kube-system
    labels:
      k8s-app: heapster
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
      version: v1.4.0-beta.0
  spec:
    replicas: 1
    selector:
      matchLabels:
        k8s-app: heapster
        version: v1.4.0-beta.0
    template:
      metadata:
        labels:
          k8s-app: heapster
          version: v1.4.0-beta.0
        annotations:
          scheduler.alpha.kubernetes.io/critical-pod: ''
      spec:
        containers:
          - image: ccr.ccs.tencentyun.com/library/heapster-amd64:v1.4.0-beta.0
            name: heapster
            livenessProbe:
              httpGet:
                path: /healthz
                port: 8082
                scheme: HTTP
              initialDelaySeconds: 180
              timeoutSeconds: 5
            command:
              - /heapster
              - --source=kubernetes.summary_api:''
          - image: ccr.ccs.tencentyun.com/library/addon-resizer:1.7
            name: heapster-nanny
            resources:
              limits:
                cpu: 50m
                memory: 90Mi
              requests:
                cpu: 50m
                memory: 90Mi
            command:
              - /pod_nanny
              - --cpu=80m
              - --extra-cpu=0.5m
              - --memory=140Mi
              - --extra-memory=4Mi
              - --threshold=5
              - --deployment=heapster-v1.4.0-beta.0
              - --container=heapster
              - --poll-period=300000
              - --estimator=exponential
        serviceAccountName: heapster
  ```

  创建Heapster Service

  ```javascript
  kind: Service
  apiVersion: v1
  metadata: 
    name: heapster
    namespace: kube-system
    labels: 
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
      kubernetes.io/name: "Heapster"
  spec: 
    ports: 
      - port: 80
        targetPort: 8082
    selector: 
      k8s-app: heapster
  ```

  保存上述的文件，并使用 `kubectl create -f FileName.yaml`创建，当创建完成后，可以使用`kubectl get` 查看

  ```javascript
  $ kubectl get deployment heapster-v1.4.0-beta.0 -n=kube-system
  NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
  heapster-v1.4.0-beta.0   1         1         1            1           1m
  $ kubectl get svc heapster -n=kube-system
  NAME       CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
  heapster   172.16.255.119   <none>        80/TCP    4d
  ```

  ## 4.2 创建服务

  创建一个用于测试的服务，可以选择从控制台创建，实例数量设置为1

  ![img](https://blog-10039692.file.myqcloud.com/1499234432350_5294_1499234432376.png)

  ## 4.3 创建HPA

  现在，我们要创建一个HPA，可以使用如下命令

  ```javascript
  $ kubectl autoscale deployment nginxtest --cpu-percent=10 --min=1 --max=10
  deployment "nginxtest" autoscaled
  $ kubectl get hpa                                                         
  NAME        REFERENCE              TARGET    CURRENT   MINPODS   MAXPODS   AGE
  nginxtest   Deployment/nginxtest   10%       0%        1         10        13s
  ```

  此命令创建了一个关联资源nginxtest的HPA，最小的pod副本数为1，最大为10。HPA会根据设定的cpu使用率（10%）动态的增加或者减少pod数量，此地方用于测试，所以设定的伸缩阈值会比较小。

  ## 4.4 测试

  ### 4.4.1 增大负载

  我们来创建一个busybox，并且循环访问上面创建的服务。

  ```javascript
  $ kubectl run -i --tty load-generator --image=busybox /bin/sh
  If you don't see a command prompt, try pressing enter.
  / # while true; do wget -q -O- http://172.16.255.60:4000; done
  ```

  下图可以看到，HPA已经开始工作。

  ```javascript
  $ kubectl get hpa
  NAME        REFERENCE              TARGET    CURRENT   MINPODS   MAXPODS   AGE
  nginxtest   Deployment/nginxtest   10%       29%        1         10        27m
  ```

  同时我们查看相关资源nginxtest的副本数量，副本数量已经从原来的1变成了3。

  ```javascript
  $ kubectl get deployment nginxtest
  NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
  nginxtest   3         3         3            3           4d
  ```

  同时再次查看HPA，由于副本数量的增加，使用率也保持在了10%左右。

  ```javascript
  $ kubectl get hpa
  NAME        REFERENCE              TARGET    CURRENT   MINPODS   MAXPODS   AGE
  nginxtest   Deployment/nginxtest   10%       9%        1         10        35m
  ```

  ### 4.4.2 减小负载

  我们关掉刚才的busbox并等待一段时间。

  ```javascript
  $ kubectl get hpa     
  NAME        REFERENCE              TARGET    CURRENT   MINPODS   MAXPODS   AGE
  nginxtest   Deployment/nginxtest   10%       0%        1         10        48m
  $ kubectl get deployment nginxtest
  NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
  nginxtest   1         1         1            1           4d
  ```

  可以看到副本数量已经由3变为1。

  ## 5. 总结

  本文主要介绍了HPA的相关原理和使用方法，此功能可以能对服务的容器数量做自动伸缩，对于服务的稳定性是一个很好的提升。但是当前稳定版本中只有cpu使用率这一个指标，是一个很大的弊端。我们会继续关注社区HPA自定义监控指标的特性，待功能稳定后，会持续输出相关文档。









## 声明式API与Kubernetes编程范式（kubectl apply 命令）

- 到底什么才是“声明式 API”呢？答案是，kubectl apply 命令
  - 实际上，你可以简单地理解为，kubectl replace 的执行过程，是使用新的 YAML 文件中的 API对象，替换原有的 API 对象；而 kubectl apply，则是执行了一个对原有 API 对象的 PATCH 操作。
  - 更进一步地，这意味着 kube-apiserver 在响应命令式请求（比如，kubectl replace）的时候，一次只能处理一个写请求，否则会有产生冲突的可能。而对于声明式请求（比如，kubectl apply），一次能处理多个写操作，并且具备 Merge 能力。



## **Istio 项目**

- Istio 项目，实际上就是一个基于 Kubernetes 项目的微服务治理框架。
- Istio 最根本的组件，是运行在每一个应用 Pod 里的 Envoy 容器。这个 Envoy 项目是 Lyft 公司推出的一个高性能 C++ 网络代理
  - Istio 项目，则把这个代理服务以 sidecar 容器的方式，运行在了每一个被治理的应用 Pod中。我们知道，Pod 里的所有容器都共享同一个 Network Namespace。所以，Envoy 容器就能够通过配置 Pod 里的 iptables 规则，把整个 Pod 的进出流量接管下来。
  - Istio 的控制层（Control Plane）里的 Pilot 组件，就能够通过调用每个 Envoy 容器的API，对这个 Envoy 代理进行配置，从而实现微服务治理。
- **Istio 项目使用的，是 Kubernetes 中的一个非常重要的功能，叫作 Dynamic Admission Control。**
  
  - 当一个 Pod 或者任何一个 API 对象被提交给 APIServer 之后，总有一些“初始化”性质的工作需要在它们被 Kubernetes 项目正式处理之前进行。比如，自动为所有Pod 加上某些标签（Labels）。
  - “初始化”操作的实现，借助的是一个叫作 Admission 的功能
  - 被称为 Admission Controller 的代码，可以选择性地被编译进 APIServer 中，在API 对象创建之后会被立刻调用到。
- **Kubernetes 项目为我们额外提供了一种“热插拔”式的 Admission 机制，它就是Dynamic Admission Control，也叫作：Initializer**。
  
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

    







## API对象内部结构（API 插件机制CRD）

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

- 在 Kubernetes v1.7 之后，这个工作就变得轻松得多了。这，当然得益于一个全新的API 插件机制：**CRD**

  - 它指的就是，允许用户在Kubernetes 中添加一个跟 Pod、Node 类似的、新的 API 资源类型，即：**自定义 API 资源**。
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



## 编写自定义控制器

- 为 Network 这个自定义API 对象编写一个自定义控制器
- 总得来说，编写自定义控制器代码的过程包括：编写 main 函数、编写自定义控制器的定义，以及编写控制器里的业务逻辑三个部分。
  - 这个 main 函数主要通过三步完成了初始化并启动一个自定义控制器的工作。
  - 第一步：main 函数根据我提供的 Master 配置（APIServer 的地址端口和 kubeconfig 的路径），创建一个 Kubernetes 的 client（kubeClient）和 Network 对象的client（networkClient）。
  - 但是，如果我没有提供 Master 配置呢？
  - 这时，main 函数会直接使用一种名叫InClusterConfig的方式来创建这个 client。这个方式，会假设你的自定义控制器是以 Pod 的方式运行在 Kubernetes 集群里的。
  - 第二步：main 函数为 Network 对象创建一个叫作 InformerFactory（即：networkInformerFactory）的工厂，并使用它生成一个 Network 对象的 Informer，传递给控制器。
  - 第三步：main 函数启动上述的 Informer，然后执行 controller.Run，启动自定义控制器。
- 自定义控制器的工作原理。
  - 这个控制器要做的第一件事，是从 Kubernetes 的 APIServer 里获取它所关心的对象，也就是我定义的 Network 对象。
  - 这个操作，依靠的是一个叫作 Informer（可以翻译为：通知器）的代码库完成的。Informer 与API 对象是一一对应的，所以我传递给自定义控制器的，正是一个 Network 对象的Informer（Network Informer）
  - 我在创建这个 Informer 工厂的时候，需要给它传递一个networkClient。
    - 事实上，Network Informer 正是使用这个 networkClient，跟 APIServer 建立了连接。不过，真正负责维护这个连接的，则是 Informer 所使用的 Reflector 包。
    - 更具体地说，Reflector 使用的是一种叫作ListAndWatch的方法，来“获取”并“监听”这些Network 对象实例的变化。
    - 在 ListAndWatch 机制下，一旦 APIServer 端有新的 Network 实例被创建、删除或者更新，Reflector 都会收到“事件通知”。这时，该事件及它对应的 API 对象这个组合，就被称为增量（Delta），它会被放进一个 Delta FIFO Queue（即：增量先进先出队列）中。
    - 而另一方面，Informe 会不断地从这个 Delta FIFO Queue 里读取（Pop）增量。每拿到一个增量，Informer 就会判断这个增量里的事件类型，然后创建或者更新本地对象的缓存。这个缓存，在 Kubernetes 里一般被叫作 Store。
    - 比如，如果事件类型是 Added（添加对象），那么 Informer 就会通过一个叫作 Indexer 的库把这个增量里的 API 对象保存在本地缓存中，并为它创建索引。相反地，如果增量的事件类型是 Deleted（删除对象），那么 Informer 就会从本地缓存中删除这个对象。
  - 这个同步本地缓存的工作，是 Informer 的第一个职责，也是它最重要的职责。
  - 而Informer 的第二个职责，则是根据这些事件的类型，触发事先注册好的ResourceEventHandler。这些 Handler，需要在创建控制器的时候注册给它对应的Informer。
  - 编写这个控制器的定义的主要内容
    - 我前面在 main 函数里创建了两个 client（kubeclientset 和 networkclientset），然后在这段代码里，使用这两个 client 和前面创建的 Informer，初始化了自定义控制器。
    - 然后，我为 networkInformer 注册了三个 Handler（AddFunc、UpdateFunc 和DeleteFunc），分别对应 API 对象的“添加”“更新”和“删除”事件。而具体的处理操作，都是将该事件对应的 API 对象加入到工作队列中。
  - 需要注意的是，实际入队的并不是 API 对象本身，而是它们的 Key，即：该 API 对象的/。
  - 而我们后面即将编写的控制循环，则会不断地从这个工作队列里拿到这些 Key，然后开始执行真正的控制逻辑。
  - 综合上面的讲述，你现在应该就能明白，**所谓 Informer，其实就是一个带有本地缓存和索引机制的、可以注册 EventHandler 的 client。它是自定义控制器跟 APIServer 进行数据同步的重要组件**。
    - 更具体地说，Informer 通过一种叫作 ListAndWatch 的方法，把 APIServer 中的 API 对象缓存在了本地，并负责更新和维护这个缓存。
    - 其中，ListAndWatch 方法的含义是：首先，通过 APIServer 的 LIST API“获取”所有最新版本的 API 对象；然后，再通过 WATCH API 来“监听”所有这些 API 对象的变化。
    - 而通过监听到的事件变化，Informer 就可以实时地更新本地缓存，并且调用这些事件对应的EventHandler 了。
    - 此外，在这个过程中，每经过 resyncPeriod 指定的时间，Informer 维护的本地缓存，都会使用最近一次 LIST 返回的结果强制更新一次，从而保证缓存的有效性。在 Kubernetes 中，这个缓存强制更新的操作就叫作：resync。
    - 需要注意的是，这个定时 resync 操作，也会触发 Informer 注册的“更新”事件。但此时，这个“更新”事件对应的 Network 对象实际上并没有发生变化，即：新、旧两个 Network 对象的 ResourceVersion 是一样的。在这种情况下，Informer 就不需要对这个更新事件再做进一步的处理了。
- 控制循环（Control Loop）部分
  - 可以看到，启动控制循环的逻辑非常简单：
    - 首先，等待 Informer 完成一次本地缓存的数据同步操作；
    - 然后，直接通过 goroutine 启动一个（或者并发启动多个）“无限循环”的任务。
  - 而这个“无限循环”任务的每一个循环周期，执行的正是我们真正关心的业务逻辑。
- 编写这个自定义控制器的业务逻辑
  - 在这个执行周期里（processNextWorkItem），我们首先从工作队列里出队（workqueue.Get）了一个成员，也就是一个 Key（Network 对象的：namespace/name）。
  - 然后，在 syncHandler 方法中，我使用这个 Key，尝试从 Informer 维护的缓存中拿到了它所对应的 Network 对象。
  - 可以看到，在这里，我使用了 networksLister 来尝试获取这个 Key 对应的 Network 对象。这个操作，其实就是在访问本地缓存的索引。实际上，在 Kubernetes 的源码中，你会经常看到控制器从各种 Lister 里获取对象，比如：podLister、nodeLister 等等，它们使用的都是Informer 和缓存机制。
  - 而如果控制循环从缓存中拿不到这个对象（即：networkLister 返回了 IsNotFound 错误），那就意味着这个 Network 对象的 Key 是通过前面的“删除”事件添加进工作队列的。所以，尽管队列里有这个 Key，但是对应的 Network 对象已经被删除了。
  - 这时候，我就需要调用 Neutron 的 API，把这个 Key 对应的 Neutron 网络从真实的集群里删除掉。
  - 而如果能够获取到对应的 Network 对象，我就可以执行控制器模式里的对比“期望状态”和“实际状态”的逻辑了。
  - 其中，自定义控制器“千辛万苦”拿到的这个 Network 对象，正是 APIServer 里保存的“期望状态”，即：用户通过 YAML 文件提交到 APIServer 里的信息。当然，在我们的例子里，它已经被 Informer 缓存在了本地。
- “实际状态”又从哪里来呢？
  - 当然是来自于实际的集群了。所以，我们的控制循环需要通过 Neutron API 来查询实际的网络情况。
  - 比如，我可以先通过 Neutron 来查询这个 Network 对象对应的真实网络是否存在。
    - 如果不存在，这就是一个典型的“期望状态”与“实际状态”不一致的情形。这时，我就需要使用这个 Network 对象里的信息（比如：CIDR 和 Gateway），调用 Neutron API 来创建真实的网络。
    - 如果存在，那么，我就要读取这个真实网络的信息，判断它是否跟 Network 对象里的信息一致，从而决定我是否要通过 Neutron 来更新这个已经存在的真实网络。
  - 这样，我就通过对比“期望状态”和“实际状态”的差异，完成了一次调协（Reconcile）的过程。
- 把这个项目运行起来，查看一下它的工作情况
  - 你可以自己编译这个项目，也可以直接使用我编译好的二进制文件（samplecrd-controller）。编译并启动这个项目的具体流程如下所示
  - 接下来，我就可以进行 Network 对象的增删改查操作了。
  - 首先，创建一个 Network 对象
    - 可以看到，我们上面创建 example-network 的操作，触发了 EventHandler 的“添加”事件，从而被放进了工作队列。
    - 紧接着，控制循环就从队列里拿到了这个对象，并且打印出了正在“处理”这个 Network 对象的日志。
    - 可以看到，这个 Network 的 ResourceVersion，也就是 API 对象的版本号，是 479015，而它的 Spec 字段的内容，跟我提交的 YAML 文件一摸一样，比如，它的 CIDR 网段是：192.168.0.0/16。
  - 我来修改一下这个 YAML 文件的内容
    - 可以看到，这一次，Informer 注册的“更新”事件被触发，更新后的 Network 对象的 Key 被添加到了工作队列之中。
    - 所以，接下来控制循环从工作队列里拿到的 Network 对象，与前一个对象是不同的：它的ResourceVersion 的值变成了 479062；而 Spec 里的字段，则变成了 192.168.1.0/16 网段
- 实际上，这套流程不仅可以用在自定义 API 资源上，也完全可以用在 Kubernetes 原生的默认API 对象上。
- Kubernetes API 编程范式的核心思想
  - 所谓的 Informer，就是一个自带缓存和索引机制，可以触发 Handler 的客户端库。这个本地缓存在 Kubernetes 中一般被称为 Store，索引一般被称为 Index。
  - Informer 使用了 Reflector 包，它是一个可以通过 ListAndWatch 机制获取并监视 API 对象变化的客户端封装。
  - Reflector 和 Informer 之间，用到了一个“增量先进先出队列”进行协同。而 Informer 与你要编写的控制循环之间，则使用了一个工作队列来进行协同。
  - 在实际应用中，除了控制循环之外的所有代码，实际上都是 Kubernetes 为你自动生成的，即：pkg/client/{informers, listers, clientset}里的内容。
  - 而这些自动生成的代码，就为我们提供了一个可靠而高效地获取 API 对象“期望状态”的编程库。
  - 所以，接下来，作为开发者，你就只需要关注如何拿到“实际状态”，然后如何拿它去跟“期望状态”做对比，从而决定接下来要做的业务逻辑即可。









## 基于角色的权限控制：RBAC

- Kubernetes 中所有的 API 对象，都保存在 Etcd 里。可是，对这些 API 对象的操作，却一定都是通过访问 kube-apiserver 实现的。其中一个非常重要的原因，就是你需要APIServer 来帮助你做授权工作。

  - 负责完成授权（Authorization）工作的机制，就是 RBAC：基于角色的访问控制
    - Role：角色，它其实是一组规则，定义了一组对 Kubernetes API 对象的操作权限。
    - Subject：被作用者
    - RoleBinding：定义了“被作用者”和“角色”的绑定关系

- Role

  - Role 本身就是一个 Kubernetes 的 API 对象

  - rules 字段，就是它所定义的权限规则

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

- RoleBinding

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

- **ServiceAccount 分配权限的过程**

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
    - Kubernetes 需要做的工作，就是使用这些存储服务，来为容器准备一个持久化的宿主机目录，以供将来进行绑定挂载时使用
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





## k8s常见命令

### kubernetes用到的一些命令

**kubectl管理工具以及命令**



![18885539-d5d52769d1469bfe.png](https://i.loli.net/2020/02/13/Rd5qDIjtnwe2ucs.png)

### 基础命令：create，delete，get，run，expose，set，explain，edit。

**create命令**：根据文件或者输入来创建资源

```css
# 创建Deployment和Service资源
kubectl create -f javak8s-deployment.yaml
kubectl create -f javak8s-service.yaml
```

**delete命令：**删除资源

```cpp
# 根据yaml文件删除对应的资源，但是yaml文件并不会被删除，这样更加高效
kubectl delete -f javak8s-deployment.yaml 
kubectl delete -f javak8s-service.yaml
# 也可以通过具体的资源名称来进行删除，使用这个删除资源，需要同时删除pod和service资源才行
kubectl delete 具体的资源名称
```

**get命令：**获得资源信息

```csharp
# 查看所有的资源信息
kubectl get all
# 查看pod列表
kubectl get pod
# 显示pod节点的标签信息
kubectl get pod --show-labels
# 根据指定标签匹配到具体的pod
kubectl get pods -l app=example
# 查看node节点列表
kubectl get node 
# 显示node节点的标签信息
kubectl get node --show-labels
# 查看pod详细信息，也就是可以查看pod具体运行在哪个节点上（ip地址信息）
kubectl get pod -o wide
# 查看服务的详细信息，显示了服务名称，类型，集群ip，端口，时间等信息
kubectl get svc
[root@master ~]# kubectl get svc
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
go-service      NodePort    10.10.10.247   <none>        8089:33702/TCP   29m
java-service    NodePort    10.10.10.248   <none>        8082:32823/TCP   5h17m
kubernetes      ClusterIP   10.10.10.1     <none>        443/TCP          5d16h
nginx-service   NodePort    10.10.10.146   <none>        88:34823/TCP     2d19h
# 查看命名空间
kubectl get ns
# 查看所有pod所属的命名空间
kubectl get pod --all-namespaces
# 查看所有pod所属的命名空间并且查看都在哪些节点上运行
kubectl get pod --all-namespaces  -o wide
# 查看目前所有的replica set，显示了所有的pod的副本数，以及他们的可用数量以及状态等信息
kubectl get rs
[root@master ~]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
go-deployment-58c76f7d5c      1         1         1       32m
java-deployment-76889f56c5    1         1         1       5h21m
nginx-deployment-58d6d6ccb8   3         3         3       2d19h
# 查看目前所有的deployment
kubectl get deployment
[root@master ~]# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
go-deployment      1/1     1            1           34m
java-deployment    1/1     1            1           5h23m
nginx-deployment   3/3     3            3           2d19h
# 查看已经部署了的所有应用，可以看到容器，以及容器所用的镜像，标签等信息
 kubectl get deploy -o wide
[root@master bin]# kubectl get deploy -o wide     
NAME    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES       SELECTOR
nginx   3/3     3            3           16m   nginx        nginx:1.10   app=example
```

**run命令：**在集群中创建并运行一个或多个容器镜像。

```objectivec
# 基本语法
run NAME --image=image [--env="key=value"] [--port=port] [--replicas=replicas] [--dry-run=bool] [--overrides=inline-json] [--command] -- [COMMAND] [args...]
# 示例，运行一个名称为nginx，副本数为3，标签为app=example，镜像为nginx:1.10，端口为80的容器实例
kubectl run nginx --replicas=3 --labels="app=example" --image=nginx:1.10 --port=80
其他用法参见：http://docs.kubernetes.org.cn/468.html
```

**expose命令：**创建一个service服务，并且暴露端口让外部可以访问

```bash
# 创建一个nginx服务并且暴露端口让外界可以访问
kubectl expose deployment nginx --port=88 --type=NodePort --target-port=80 --name=nginx-service
关于expose的详细用法参见：http://docs.kubernetes.org.cn/475.html
```

**set命令：** 配置应用的一些特定资源，也可以修改应用已有的资源

```cpp
# 使用kubectl set --help查看，它的子命令，env，image，resources，selector，serviceaccount，subject。
set命令详情参见：http://docs.kubernetes.org.cn/669.html
语法：
resources (-f FILENAME | TYPE NAME) ([--limits=LIMITS & --requests=REQUESTS]
```

**kubectl set resources 命令**

这个命令用于设置资源的一些范围限制。

资源对象中的[Pod](https://links.jianshu.com/go?to=http%3A%2F%2Fdocs.kubernetes.org.cn%2F312.html)可以指定计算资源需求（CPU-单位m、内存-单位Mi），即使用的最小资源请求（Requests），限制（Limits）的最大资源需求，Pod将保证使用在设置的资源数量范围。

对于每个Pod资源，如果指定了Limits（限制）值，并省略了Requests（请求），则Requests默认为Limits的值。

**可用资源对象包括(支持大小写)：replicationcontroller、deployment、daemonset、job、replicaset。**

例如：

```bash
# 将deployment的nginx容器cpu限制为“200m”，将内存设置为“512Mi”
kubectl set resources deployment nginx -c=nginx --limits=cpu=200m,memory=512Mi
# 为nginx中的所有容器设置 Requests和Limits
kubectl set resources deployment nginx --limits=cpu=200m,memory=512Mi --requests=cpu=100m,memory=256Mi
# 删除nginx中容器的计算资源值
kubectl set resources deployment nginx --limits=cpu=0,memory=0 --requests=cpu=0,memory=0
```

**kubectl set selector命令**

设置资源的selector（选择器）。如果在调用"set selector"命令之前已经存在选择器，则新创建的选择器将覆盖原来的选择器。

selector必须以字母或数字开头，最多包含63个字符，可使用：字母、数字、连字符" - " 、点"."和下划线" _ "。如果指定了--resource-version，则更新将使用此资源版本，否则将使用现有的资源版本。

**注意：目前selector命令只能用于Service对象。**

```bash
# 语法
selector (-f FILENAME | TYPE NAME) EXPRESSIONS [--resource-version=version]
```

**kubectl set image命令**

 用于更新现有资源的容器镜像。

可用资源对象包括：pod (po)、replicationcontroller (rc)、deployment (deploy)、daemonset (ds)、job、replicaset (rs)。

```bash
# 语法
image (-f FILENAME | TYPE NAME) CONTAINER_NAME_1=CONTAINER_IMAGE_1 ... CONTAINER_NAME_N=CONTAINER_IMAGE_N
# 将deployment中的nginx容器镜像设置为“nginx：1.9.1”。
kubectl set image deployment/nginx busybox=busybox nginx=nginx:1.9.1
# 所有deployment和rc的nginx容器镜像更新为“nginx：1.9.1”
kubectl set image deployments,rc nginx=nginx:1.9.1 --all
# 将daemonset abc的所有容器镜像更新为“nginx：1.9.1”
kubectl set image daemonset abc *=nginx:1.9.1
# 从本地文件中更新nginx容器镜像
kubectl set image -f path/to/file.yaml nginx=nginx:1.9.1 --local -o yaml
```

------

**explain命令**：用于显示资源文档信息

```undefined
kubectl explain rs
```

**edit命令**:用于编辑资源信息

```bash
# 编辑Deployment nginx的一些信息
kubectl edit deployment nginx
# 编辑service类型的nginx的一些信息
kubectl edit service/nginx
```

------

### 设置命令：label，annotate，completion

**label命令**:用于更新（增加、修改或删除）资源上的 label（标签）

- label 必须以字母或数字开头，可以使用字母、数字、连字符、点和下划线，最长63个字符。
- 如果--overwrite 为 true，则可以覆盖已有的 label，否则尝试覆盖 label 将会报错。
- 如果指定了--resource-version，则更新将使用此资源版本，否则将使用现有的资源版本。

```bash
# 语法
label [--overwrite] (-f FILENAME | TYPE NAME) KEY_1=VAL_1 ... KEY_N=VAL_N [--resource-version=version]
# 给名为foo的Pod添加label unhealthy=true
kubectl label pods foo unhealthy=true
# 给名为foo的Pod修改label 为 'status' / value 'unhealthy'，且覆盖现有的value
kubectl label --overwrite pods foo status=unhealthy
# 给 namespace 中的所有 pod 添加 label
kubectl label  pods --all status=unhealthy
# 仅当resource-version=1时才更新 名为foo的Pod上的label
kubectl label pods foo status=unhealthy --resource-version=1
# 删除名为“bar”的label 。（使用“ - ”减号相连）
kubectl label pods foo bar-
```

**annotate命令**：更新一个或多个资源的Annotations信息。也就是注解信息，可以方便的查看做了哪些操作。

- Annotations由key/value组成。
- Annotations的目的是存储辅助数据，特别是通过工具和系统扩展操作的数据，更多介绍在[这里](https://links.jianshu.com/go?to=http%3A%2F%2Fdocs.kubernetes.org.cn%2F255.html)。
- 如果--overwrite为true，现有的annotations可以被覆盖，否则试图覆盖annotations将会报错。
- 如果设置了--resource-version，则更新将使用此resource version，否则将使用原有的resource version。

```bash
# 语法
annotate [--overwrite] (-f FILENAME | TYPE NAME) KEY_1=VAL_1 ... KEY_N=VAL_N [--resource-version=version]
# 更新Pod“foo”，设置annotation “description”的value “my frontend”，如果同一个annotation多次设置，则只使用最后设置的value值
kubectl annotate pods foo description='my frontend'
# 根据“pod.json”中的type和name更新pod的annotation
kubectl annotate -f pod.json description='my frontend'
# 更新Pod"foo"，设置annotation“description”的value“my frontend running nginx”，覆盖现有的值
kubectl annotate --overwrite pods foo description='my frontend running nginx'
# 更新 namespace中的所有pod
kubectl annotate pods --all description='my frontend running nginx'
# 只有当resource-version为1时，才更新pod ' foo '
kubectl annotate pods foo description='my frontend running nginx' --resource-version=1
# 通过删除名为“description”的annotations来更新pod ' foo '。#不需要- overwrite flag。
kubectl annotate pods foo description-
```

**completion命令**：用于设置kubectl命令自动补全

```bash
$ source <(kubectl completion bash) # setup autocomplete in bash, bash-completion package should be installed first.
$ source <(kubectl completion zsh)  # setup autocomplete in zsh
```

------

### kubectl 部署命令：rollout，rolling-update，scale，autoscale

**rollout命令**:用于对资源进行管理

可用资源包括：deployments，daemonsets。

子命令：

-  [*history*](https://links.jianshu.com/go?to=http%3A%2F%2Fdocs.kubernetes.org.cn%2F645.html)（查看历史版本）
-  [*pause*](https://links.jianshu.com/go?to=http%3A%2F%2Fdocs.kubernetes.org.cn%2F647.html)（暂停资源）
-  [*resume*](https://links.jianshu.com/go?to=http%3A%2F%2Fdocs.kubernetes.org.cn%2F650.html)（恢复暂停资源）
-  [*status*](https://links.jianshu.com/go?to=http%3A%2F%2Fdocs.kubernetes.org.cn%2F652.html)（查看资源状态）
-  [*undo*](https://links.jianshu.com/go?to=http%3A%2F%2Fdocs.kubernetes.org.cn%2F654.html)（回滚版本）

```bash
# 语法
kubectl rollout SUBCOMMAND
# 回滚到之前的deployment
kubectl rollout undo deployment/abc
# 查看daemonet的状态
kubectl rollout status daemonset/foo
```

**rolling-update命令:**执行指定ReplicationController的滚动更新。

该命令创建了一个新的RC， 然后一次更新一个pod方式逐步使用新的PodTemplate，最终实现Pod滚动更新，new-controller.json需要与之前RC在相同的namespace下。

```bash
# 语法
rolling-update OLD_CONTROLLER_NAME ([NEW_CONTROLLER_NAME] --image=NEW_CONTAINER_IMAGE | -f NEW_CONTROLLER_SPEC)
# 使用frontend-v2.json中的新RC数据更新frontend-v1的pod
kubectl rolling-update frontend-v1 -f frontend-v2.json
# 使用JSON数据更新frontend-v1的pod
cat frontend-v2.json | kubectl rolling-update frontend-v1 -f -
# 其他的一些滚动更新
kubectl rolling-update frontend-v1 frontend-v2 --image=image:v2

kubectl rolling-update frontend --image=image:v2

kubectl rolling-update frontend-v1 frontend-v2 --rollback
```

**scale命令**：扩容或缩容 Deployment、ReplicaSet、Replication Controller或 Job 中Pod数量

scale也可以指定多个前提条件，如：当前副本数量或 --resource-version ，进行伸缩比例设置前，系统会先验证前提条件是否成立。这个就是弹性伸缩策略

```bash
# 语法
kubectl scale [--resource-version=version] [--current-replicas=count] --replicas=COUNT (-f FILENAME | TYPE NAME)
# 将名为foo中的pod副本数设置为3。
kubectl scale --replicas=3 rs/foo
kubectl scale deploy/nginx --replicas=30
# 将由“foo.yaml”配置文件中指定的资源对象和名称标识的Pod资源副本设为3
kubectl scale --replicas=3 -f foo.yaml
# 如果当前副本数为2，则将其扩展至3。
kubectl scale --current-replicas=2 --replicas=3 deployment/mysql
# 设置多个RC中Pod副本数量
kubectl scale --replicas=5 rc/foo rc/bar rc/baz
```

**autoscale命令：** 这个比scale更加强大，也是弹性伸缩策略 ，它是根据流量的多少来自动进行扩展或者缩容

指定Deployment、ReplicaSet或ReplicationController，并创建已经定义好资源的自动伸缩器。使用自动伸缩器可以根据需要自动增加或减少系统中部署的pod数量。

```swift
# 语法
kubectl autoscale (-f FILENAME | TYPE NAME | TYPE/NAME) [--min=MINPODS] --max=MAXPODS [--cpu-percent=CPU] [flags]
# 使用 Deployment “foo”设定，使用默认的自动伸缩策略，指定目标CPU使用率，使其Pod数量在2到10之间
kubectl autoscale deployment foo --min=2 --max=10
# 使用RC“foo”设定，使其Pod的数量介于1和5之间，CPU使用率维持在80％
kubectl autoscale rc foo --max=5 --cpu-percent=80
```

------

### 集群管理命令：certificate，cluster-info，top，cordon，uncordon，drain，taint

**certificate命令**：用于证书资源管理，授权等

```bash
[root@master ~]# kubectl certificate --help
Modify certificate resources.

Available Commands:
  approve     Approve a certificate signing request
  deny        Deny a certificate signing request

Usage:
  kubectl certificate SUBCOMMAND [options]

Use "kubectl <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).

# 例如，当有node节点要向master请求，那么是需要master节点授权的
kubectl  certificate approve node-csr-81F5uBehyEyLWco5qavBsxc1GzFcZk3aFM3XW5rT3mw node-csr-Ed0kbFhc_q7qx14H3QpqLIUs0uKo036O2SnFpIheM18
```

**cluster-info命令：**显示集群信息

```csharp
kubectl cluster-info

[root@master ~]# kubectl cluster-info
Kubernetes master is running at http://localhost:8080
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

**top命令：**用于查看资源的cpu，内存磁盘等资源的使用率

```undefined
kubectl top pod --all-namespaces
它需要heapster运行才行
```

**cordon命令**：用于标记某个节点不可调度

**uncordon命令：**用于标签节点可以调度

**drain命令：** 用于在维护期间排除节点。

**taint命令**：参见：[https://blog.frognew.com/2018/05/taint-and-toleration.html](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.frognew.com%2F2018%2F05%2Ftaint-and-toleration.html)

------

### 集群故障排查和调试命令：describe，logs，exec，attach，port-foward，proxy，cp，auth

**describe命令**：显示特定资源的详细信息

```bash
# 语法
kubectl describe TYPE NAME_PREFIX
（首先检查是否有精确匹配TYPE和NAME_PREFIX的资源，如果没有，将会输出所有名称以NAME_PREFIX开头的资源详细信息）
支持的资源包括但不限于（大小写不限）：pods (po)、services (svc)、 replicationcontrollers (rc)、nodes (no)、events (ev)、componentstatuses (cs)、 limitranges (limits)、persistentvolumes (pv)、persistentvolumeclaims (pvc)、 resourcequotas (quota)和secrets。
#查看my-nginx pod的详细状态
kubectl describe po my-nginx
```

**logs命令：**用于在一个pod中打印一个容器的日志，如果pod中只有一个容器，可以省略容器名

```ruby
# 语法
kubectl logs [-f] [-p] POD [-c CONTAINER]

# 返回仅包含一个容器的pod nginx的日志快照
$ kubectl logs nginx
# 返回pod ruby中已经停止的容器web-1的日志快照
$ kubectl logs -p -c ruby web-1
# 持续输出pod ruby中的容器web-1的日志
$ kubectl logs -f -c ruby web-1
# 仅输出pod nginx中最近的20条日志
$ kubectl logs --tail=20 nginx
# 输出pod nginx中最近一小时内产生的所有日志
$ kubectl logs --since=1h nginx
# 参数选项
  -c, --container="": 容器名。
  -f, --follow[=false]: 指定是否持续输出日志（实时日志）。
      --interactive[=true]: 如果为true，当需要时提示用户进行输入。默认为true。
      --limit-bytes=0: 输出日志的最大字节数。默认无限制。
  -p, --previous[=false]: 如果为true，输出pod中曾经运行过，但目前已终止的容器的日志。
      --since=0: 仅返回相对时间范围，如5s、2m或3h，之内的日志。默认返回所有日志。只能同时使用since和since-time中的一种。
      --since-time="": 仅返回指定时间（RFC3339格式）之后的日志。默认返回所有日志。只能同时使用since和since-time中的一种。
      --tail=-1: 要显示的最新的日志条数。默认为-1，显示所有的日志。
      --timestamps[=false]: 在日志中包含时间戳。
```

**exec命令**：进入容器进行交互，在容器中执行命令

```bash
# 语法
kubectl exec POD [-c CONTAINER] -- COMMAND [args...]
#命令选项
  -c, --container="": 容器名。如果未指定，使用pod中的一个容器。
  -p, --pod="": Pod名。
  -i, --stdin[=false]: 将控制台输入发送到容器。
  -t, --tty[=false]: 将标准输入控制台作为容器的控制台输入。
# 进入nginx容器，执行一些命令操作
kubectl exec -it nginx-deployment-58d6d6ccb8-lc5fp bash
```

**attach命令**：连接到一个正在运行的容器。

```bash
#语法
kubectl attach POD -c CONTAINER
# 参数选项
-c, --container="": 容器名。如果省略，则默认选择第一个 pod
  -i, --stdin[=false]: 将控制台输入发送到容器。
  -t, --tty[=false]: 将标准输入控制台作为容器的控制台输入。

# 获取正在运行中的pod 123456-7890的输出，默认连接到第一个容器
kubectl attach 123456-7890
# 获取pod 123456-7890中ruby-container的输出
kubectl attach 123456-7890 -c ruby-container
# 切换到终端模式，将控制台输入发送到pod 123456-7890的ruby-container的“bash”命令，并将其输出到控制台/
# 错误控制台的信息发送回客户端。
kubectl attach 123456-7890 -c ruby-container -i -t
```

**cp命令**：拷贝文件或者目录到pod容器中

用于pod和外部的文件交换,类似于docker 的cp，就是将容器中的内容和外部的内容进行交换。

------

### 其他命令：api-servions，config，help，plugin，version

**api-servions命令**：打印受支持的api版本信息

```csharp
# kubectl api-versions
[root@master ~]# kubectl api-versions
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1beta1
apps/v1
apps/v1beta1
apps/v1beta2
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
batch/v1
batch/v1beta1
certificates.k8s.io/v1beta1
events.k8s.io/v1beta1
extensions/v1beta1
networking.k8s.io/v1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
```

**help命令**：用于查看命令帮助

```bash
# 显示全部的命令帮助提示
kubectl --help
# 具体的子命令帮助，例如
kubectl create --help
```

**config**:用于修改kubeconfig配置文件（用于访问api，例如配置认证信息）

**version命令：**打印客户端和服务端版本信息

```css
[root@master ~]# kubectl version
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.3", GitCommit:"2d3c76f9091b6bec110a5e63777c332469e0cba2", GitTreeState:"clean", BuildDate:"2019-08-19T11:13:54Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.0", GitCommit:"925c127ec6b946659ad0fd596fa959be43f0cc05", GitTreeState:"clean", BuildDate:"2017-12-15T20:55:30Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}
```

**plugin命令：**运行一个命令行插件

------

### 高级命令：apply，patch，replace，convert

**apply命令：** 通过文件名或者标准输入对资源应用配置

通过文件名或控制台输入，对资源进行配置。 如果资源不存在，将会新建一个。可以使用 JSON 或者 YAML 格式。

```php
# 语法
kubectl apply -f FILENAME

# 将pod.json中的配置应用到pod
kubectl apply -f ./pod.json
# 将控制台输入的JSON配置应用到Pod
cat pod.json | kubectl apply -f -

选项
-f, --filename=[]: 包含配置信息的文件名，目录名或者URL。
      --include-extended-apis[=true]: If true, include definitions of new APIs via calls to the API server. [default true]
  -o, --output="": 输出模式。"-o name"为快捷输出(资源/name).
      --record[=false]: 在资源注释中记录当前 kubectl 命令。
  -R, --recursive[=false]: Process the directory used in -f, --filename recursively. Useful when you want to manage related manifests organized within the same directory.
      --schema-cache-dir="~/.kube/schema": 非空则将API schema缓存为指定文件，默认缓存到'$HOME/.kube/schema'
      --validate[=true]: 如果为true，在发送到服务端前先使用schema来验证输入。
```

**patch命令：** 使用补丁修改，更新资源的字段，也就是修改资源的部分内容

```rust
# 语法
kubectl patch (-f FILENAME | TYPE NAME) -p PATCH

# Partially update a node using strategic merge patch
kubectl patch node k8s-node-1 -p '{"spec":{"unschedulable":true}}'
# Update a container's image; spec.containers[*].name is required because it's a merge key
kubectl patch pod valid-pod -p '{"spec":{"containers":[{"name":"kubernetes-serve-hostname","image":"new image"}]}}'
```

**replace命令：** 通过文件或者标准输入替换原有资源

```bash
# 语法
kubectl replace -f FILENAME

# Replace a pod using the data in pod.json.
kubectl replace -f ./pod.json
# Replace a pod based on the JSON passed into stdin.
cat pod.json | kubectl replace -f -
# Update a single-container pod's image version (tag) to v4
kubectl get pod mypod -o yaml | sed 's/\(image: myimage\):.*$/\1:v4/' | kubectl replace -f -
# Force replace, delete and then re-create the resource
kubectl replace --force -f ./pod.json
```

**convert命令：** 不同的版本之间转换配置文件

```bash
# 语法
kubectl convert -f FILENAME

# Convert 'pod.yaml' to latest version and print to stdout.
kubectl convert -f pod.yaml
# Convert the live state of the resource specified by 'pod.yaml' to the latest version
# and print to stdout in json format.
kubectl convert -f pod.yaml --local -o json
# Convert all files under current directory to latest version and create them all.
kubectl convert -f . | kubectl create -f -
```

以上就是k8s的一些基本命令





























































































































