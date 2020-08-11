# [Docker常用命令](https://www.cnblogs.com/DeepInThought/p/10896790.html)



# 1、Docker容器信息

```shell
##查看docker容器版本
docker version
##查看docker容器信息
docker info
##查看docker容器帮助
docker --help
```

# 2、镜像操作

提示：对于镜像的操作可使用镜像名、镜像长ID和短ID。

## 2.1、镜像查看

```shell
##列出本地images
docker images
##含中间映像层
docker images -a
```

![img](https://img2018.cnblogs.com/blog/1659331/201905/1659331-20190521104721523-485290950.png)

```shell
##只显示镜像ID
docker images -q
##含中间映像层
docker images -qa   
```

![img](https://img2018.cnblogs.com/blog/1659331/201905/1659331-20190521104927909-600452122.png)

```shell
##显示镜像摘要信息(DIGEST列)
docker images --digests
##显示镜像完整信息
docker images --no-trunc
```

![img](https://img2018.cnblogs.com/blog/1659331/201905/1659331-20190521105114405-1780655005.png)

```shell
##显示指定镜像的历史创建；参数：-H 镜像大小和日期，默认为true；--no-trunc  显示完整的提交记录；-q  仅列出提交记录ID
docker history -H redis
```

## 2.2、镜像搜索

```shell
##搜索仓库MySQL镜像
docker search mysql
## --filter=stars=600：只显示 starts>=600 的镜像
docker search --filter=stars=600 mysql
## --no-trunc 显示镜像完整 DESCRIPTION 描述
docker search --no-trunc mysql
## --automated ：只列出 AUTOMATED=OK 的镜像
docker search  --automated mysql
```

![img](https://img2018.cnblogs.com/blog/1659331/201905/1659331-20190521110514156-691788920.png)

## 2.3、镜像下载

```shell
##下载Redis官方最新镜像，相当于：docker pull redis:latest
docker pull redis
##下载仓库所有Redis镜像
docker pull -a redis
##下载私人仓库镜像
docker pull bitnami/redis
```

![img](https://img2018.cnblogs.com/blog/1659331/201905/1659331-20190521112716615-10141164.png)

## 2.4、镜像删除

```shell
##单个镜像删除，相当于：docker rmi redis:latest
docker rmi redis
##强制删除(针对基于镜像有运行的容器进程)
docker rmi -f redis
##多个镜像删除，不同镜像间以空格间隔
docker rmi -f redis tomcat nginx
##删除本地全部镜像
docker rmi -f $(docker images -q)
```

## 2.5、镜像构建

```shell
##（1）编写dockerfile
cd /docker/dockerfile
vim mycentos
##（2）构建docker镜像
docker build -f /docker/dockerfile/mycentos -t mycentos:1.1
```

# 3、容器操作

提示：对于容器的操作可使用CONTAINER ID 或 NAMES。

## 3.1、容器启动

```shell
##新建并启动容器，参数：-i  以交互模式运行容器；-t  为容器重新分配一个伪输入终端；--name  为容器指定一个名称
docker run -i -t --name mycentos
##后台启动容器，参数：-d  已守护方式启动容器
docker run -d mycentos
```

注意：此时使用"docker ps -a"会发现容器已经退出。这是docker的机制：要使Docker容器后台运行，就必须有一个前台进程。解决方案：将你要运行的程序以前台进程的形式运行。

```shell
##启动一个或多个已经被停止的容器
docker start redis
##重启容器
docker restart redis
```

## 3.2、容器进程

```shell
##top支持 ps 命令参数，格式：docker top [OPTIONS] CONTAINER [ps OPTIONS]
##列出redis容器中运行进程
docker top redis
##查看所有运行容器的进程信息
for i in  `docker ps |grep Up|awk '{print $1}'`;do echo \ &&docker top $i; done
```

## 3.3、容器日志

```shell
##查看redis容器日志，默认参数
docker logs rabbitmq
##查看redis容器日志，参数：-f  跟踪日志输出；-t   显示时间戳；--tail  仅列出最新N条容器日志；
docker logs -f -t --tail=20 redis
##查看容器redis从2019年05月21日后的最新10条日志。
docker logs --since="2019-05-21" --tail=10 redis
```

## 3.4、容器的进入与退出

```shell
##使用run方式在创建时进入
docker run -it centos /bin/bash
##关闭容器并退出
exit
##仅退出容器，不关闭
快捷键：Ctrl + P + Q
##直接进入centos 容器启动命令的终端，不会启动新进程，多个attach连接共享容器屏幕，参数：--sig-proxy=false  确保CTRL-D或CTRL-C不会关闭容器
docker attach --sig-proxy=false centos 
##在 centos 容器中打开新的交互模式终端，可以启动新进程，参数：-i  即使没有附加也保持STDIN 打开；-t  分配一个伪终端
docker exec -i -t  centos /bin/bash
##以交互模式在容器中执行命令，结果返回到当前终端屏幕
docker exec -i -t centos ls -l /tmp
##以分离模式在容器中执行命令，程序后台运行，结果不会反馈到当前终端
docker exec -d centos  touch cache.txt 
```

## 3.5、查看容器

```shell
##查看正在运行的容器
docker ps
##查看正在运行的容器的ID
docker ps -q
##查看正在运行+历史运行过的容器
docker ps -a
##显示运行容器总文件大小
docker ps -s
```

![img](https://img2018.cnblogs.com/blog/1659331/201905/1659331-20190521132255698-500560462.png)
![img](https://img2018.cnblogs.com/blog/1659331/201905/1659331-20190521133039811-1994116017.png)

```shell
##显示最近创建容器
docker ps -l
##显示最近创建的3个容器
docker ps -n 3
##不截断输出
docker ps --no-trunc 
```

![img](https://img2018.cnblogs.com/blog/1659331/201905/1659331-20190521132741451-294716433.png)

```shell
##获取镜像redis的元信息
docker inspect redis
##获取正在运行的容器redis的 IP
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' redis
```

## 3.6、容器的停止与删除

```shell
##停止一个运行中的容器
docker stop redis
##杀掉一个运行中的容器
docker kill redis
##删除一个已停止的容器
docker rm redis
##删除一个运行中的容器
docker rm -f redis
##删除多个容器
docker rm -f $(docker ps -a -q)
docker ps -a -q | xargs docker rm
## -l 移除容器间的网络连接，连接名为 db
docker rm -l db 
## -v 删除容器，并删除容器挂载的数据卷
docker rm -v redis
```

## 3.7、生成镜像

```shell
##基于当前redis容器创建一个新的镜像；参数：-a 提交的镜像作者；-c 使用Dockerfile指令来创建镜像；-m :提交时的说明文字；-p :在commit时，将容器暂停
docker commit -a="DeepInThought" -m="my redis" [redis容器ID]  myredis:v1.1
```

## 3.8、容器与主机间的数据拷贝

```shell
##将rabbitmq容器中的文件copy至本地路径
docker cp rabbitmq:/[container_path] [local_path]
##将主机文件copy至rabbitmq容器
docker cp [local_path] rabbitmq:/[container_path]/
##将主机文件copy至rabbitmq容器，目录重命名为[container_path]（注意与非重命名copy的区别）
docker cp [local_path] rabbitmq:/[container_path]
```





# k8s

```shell
#查看所有namespace的pods运行情况
kubectl get pods --all-namespaces
#查看具体pods，记得后边跟namespace名字哦
kubectl get pods  kubernetes-dashboard-76479d66bb-nj8wr --namespace=kube-system
# 查看pods具体信息
kubectl get pods -o wide kubernetes-dashboard-76479d66bb-nj8wr --namespace=kube-system
# 查看集群健康状态
kubectl get cs
# 获取所有deployment
kubectl get deployment --all-namespaces
# 列出该 namespace 中的所有 pod 包括未初始化的
kubectl get pods --include-uninitialized
# 查看deployment()
kubectl get deployment nginx-app
# 查看rc和servers
kubectl get rc,services
# 查看pods结构信息（重点，通过这个看日志分析错误）
# 对控制器和服务，node同样有效
kubectl describe pods xxxxpodsname --namespace=xxxnamespace
# 其他控制器类似吧，就是kubectl get 控制器 控制器具体名称
# 查看pod日志
kubectl logs $POD_NAME
# 查看pod变量
kubectl exec my-nginx-5j8ok -- printenv | grep SERVICE
# 集群
kubectl get cs           # 集群健康情况
kubectl cluster-info     # 集群核心组件运行情况
kubectl get namespaces    # 表空间名
kubectl version           # 版本
kubectl api-versions      # API
kubectl get events       # 查看事件
kubectl get nodes      //获取全部节点
kubectl delete node k8s2  //删除节点
kubectl rollout status deploy nginx-test
# 创建
kubectl create -f ./nginx.yaml           # 创建资源
kubectl create -f .                            # 创建当前目录下的所有yaml资源
kubectl create -f ./nginx1.yaml -f ./mysql2.yaml     # 使用多个文件创建资源
kubectl create -f ./dir                        # 使用目录下的所有清单文件来创建资源
kubectl create -f https://git.io/vPieo         # 使用 url 来创建资源
kubectl run -i --tty busybox --image=busybox    ----创建带有终端的pod
kubectl run nginx --image=nginx                # 启动一个 nginx 实例
kubectl run mybusybox --image=busybox --replicas=5    ----启动多个pod
kubectl explain pods,svc                       # 获取 pod 和 svc 的文档
# 更新
kubectl rolling-update python-v1 -f python-v2.json           # 滚动更新 pod frontend-v1
kubectl rolling-update python-v1 python-v2 --image=image:v2  # 更新资源名称并更新镜像
kubectl rolling-update python --image=image:v2                 # 更新 frontend pod 中的镜像
kubectl rolling-update python-v1 python-v2 --rollback        # 退出已存在的进行中的滚动更新
cat pod.json | kubectl replace -f -                              # 基于 stdin 输入的 JSON 替换 pod
强制替换，删除后重新创建资源。会导致服务中断。
kubectl replace --force -f ./pod.json
为 nginx RC 创建服务，启用本地 80 端口连接到容器上的 8000 端口
kubectl expose rc nginx --port=80 --target-port=8000

更新单容器 pod 的镜像版本（tag）到 v4
kubectl get pod nginx-pod -o yaml | sed 's/\(image: myimage\):.*$/\1:v4/' | kubectl replace -f -
kubectl label pods nginx-pod new-label=awesome                      # 添加标签
kubectl annotate pods nginx-pod icon-url=http://goo.gl/XXBTWq       # 添加注解
kubectl autoscale deployment foo --min=2 --max=10                # 自动扩展 deployment “foo”
# 编辑资源
kubectl edit svc/docker-registry                      # 编辑名为 docker-registry 的 service
KUBE_EDITOR="nano" kubectl edit svc/docker-registry   # 使用其它编辑器
# 动态伸缩pod
kubectl scale --replicas=3 rs/foo                                 # 将foo副本集变成3个
kubectl scale --replicas=3 -f foo.yaml                            # 缩放“foo”中指定的资源。
kubectl scale --current-replicas=2 --replicas=3 deployment/mysql  # 将deployment/mysql从2个变成3个
kubectl scale --replicas=5 rc/foo rc/bar rc/baz                   # 变更多个控制器的数量
kubectl rollout status deploy deployment/mysql                         # 查看变更进度
# 删除
kubectl delete -f ./pod.json                                              # 删除 pod.json 文件中定义的类型和名称的 pod
kubectl delete pod,service baz foo                                        # 删除名为“baz”的 pod 和名为“foo”的 service
kubectl delete pods,services -l name=myLabel                              # 删除具有 name=myLabel 标签的 pod 和 serivce
kubectl delete pods,services -l name=myLabel --include-uninitialized      # 删除具有 name=myLabel 标签的 pod 和 service，包括尚未初始化的
kubectl -n my-ns delete po,svc --all # 删除 my-ns namespace下的所有 pod 和 serivce，包括尚未初始化的
kubectl delete pods prometheus-7fcfcb9f89-qkkf7 --grace-period=0 --force 强制删除

# 交互
kubectl logs nginx-pod                                 # dump 输出 pod 的日志（stdout）
kubectl logs nginx-pod -c my-container                 # dump 输出 pod 中容器的日志（stdout，pod 中有多个容器的情况下使用）
kubectl logs -f nginx-pod                              # 流式输出 pod 的日志（stdout）
kubectl logs -f nginx-pod -c my-container              # 流式输出 pod 中容器的日志（stdout，pod 中有多个容器的情况下使用）
kubectl run -i --tty busybox --image=busybox -- sh  # 交互式 shell 的方式运行 pod
kubectl attach nginx-pod -i                            # 连接到运行中的容器
kubectl port-forward nginx-pod 5000:6000               # 转发 pod 中的 6000 端口到本地的 5000 端口
kubectl exec nginx-pod -- ls /                         # 在已存在的容器中执行命令（只有一个容器的情况下）
kubectl exec nginx-pod -c my-container -- ls /         # 在已存在的容器中执行命令（pod 中有多个容器的情况下）
kubectl top pod POD_NAME --containers               # 显示指定 pod和容器的指标度量
# 调度配置
$ kubectl cordon k8s-node                                                # 标记 my-node 不可调度
$ kubectl drain k8s-node                                                 # 清空 my-node 以待维护
$ kubectl uncordon k8s-node                                              # 标记 my-node 可调度
$ kubectl top node k8s-node                                              # 显示 my-node 的指标度量
$ kubectl cluster-info dump                                             # 将当前集群状态输出到 stdout
$ kubectl cluster-info dump --output-directory=/path/to/cluster-state   # 将当前集群状态输出到 /path/to/cluster-state
#如果该键和影响的污点（taint）已存在，则使用指定的值替换
$ kubectl taint nodes foo dedicated=special-user:NoSchedule
```

