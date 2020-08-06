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

## 2.1 FROM

　　指明构建的新镜像是来自于哪个基础镜像，例如：

```
FROM centos:6
```

## 2.2 MAINTAINER

　　指明镜像维护着及其联系方式（一般是邮箱地址），例如：

```
MAINTAINER Edison Zhou <edisonchou@hotmail.com>
```

　　不过，MAINTAINER并不推荐使用，更推荐使用LABEL来指定镜像作者，例如：

```
LABEL maintainer="edisonzhou.cn"
```

## 2.3 RUN

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

## 2.4 CMD

　　启动容器时执行的Shell命令，例如：

```
CMD ["-C", "/start.sh"] 
CMD ["/usr/sbin/sshd", "-D"] 
CMD /usr/sbin/sshd -D
```

## 2.5 EXPOSE

　　声明容器运行的服务端口，例如：

```
EXPOSE 80 443
```

## 2.6 ENV

　　设置环境内环境变量，例如：

```
ENV MYSQL_ROOT_PASSWORD 123456
ENV JAVA_HOME /usr/local/jdk1.8.0_45
```

## 2.7 ADD

　　拷贝文件或目录到镜像中，例如：

```
ADD <src>...<dest>
ADD html.tar.gz /var/www/html
ADD https://xxx.com/html.tar.gz /var/www/html
```

　　***PS：***如果是URL或压缩包，会自动下载或自动解压。

## 2.8 COPY

　　拷贝文件或目录到镜像中，用法同ADD，只是不支持自动下载和解压，例如：

```
COPY ./start.sh /start.sh
```

## 2.9 ENTRYPOINT

　　启动容器时执行的Shell命令，同CMD类似，只是由ENTRYPOINT启动的程序**不会被docker run命令行指定的参数所覆盖**，而且，**这些命令行参数会被当作参数传递给ENTRYPOINT指定指定的程序**，例如：

```
ENTRYPOINT ["/bin/bash", "-C", "/start.sh"]
ENTRYPOINT /bin/bash -C '/start.sh'
```

　　***PS：***Dockerfile文件中也可以存在多个ENTRYPOINT指令，但仅有最后一个会生效。

## 2.10 VOLUME

　　指定容器挂载点到宿主机自动生成的目录或其他容器，例如：

```
VOLUME ["/var/lib/mysql"]
```

　　***PS：***一般不会在Dockerfile中用到，更常见的还是在docker run的时候指定-v数据卷。

## 2.11 USER

　　为RUN、CMD和ENTRYPOINT执行Shell命令指定运行用户，例如：

```
USER <user>[:<usergroup>]
USER <UID>[:<UID>]
USER edisonzhou
```

## 2.12 WORKDIR

　　为RUN、CMD、ENTRYPOINT以及COPY和AND设置工作目录，例如：

```
WORKDIR /data
```

## 2.13 HEALTHCHECK

　　告诉Docker如何测试容器以检查它是否仍在工作，即健康检查，例如：

```
HEALTHCHECK --interval=5m --timeout=3s --retries=3 \
    CMD curl -f http:/localhost/ || exit 1
```

　　其中，一些选项的说明：

-  --interval=DURATION (default: 30s)：每隔多长时间探测一次，默认30秒
-  -- timeout= DURATION (default: 30s)：服务响应超时时长，默认30秒
-  --start-period= DURATION (default: 0s)：服务启动多久后开始探测，默认0秒
-  --retries=N (default: 3)：认为检测失败几次为宕机，默认3次

　　一些返回值的说明：

-  0：容器成功是健康的，随时可以使用
-  1：不健康的容器无法正常工作
-  2：保留不使用此退出代码

## 2.14 ARG

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

# 四、小结

　　本文介绍了Dockerfile的背景和组成，以及最常用的一些Dockerfile命令，最后介绍了一个综合使用了Dockefile指令的一个案例来说明Dockerfile的应用。