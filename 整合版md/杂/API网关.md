







## 网关

**通俗一点的讲：网关就是要去别的网络的时候，把报文首先发送到的那台设备。稍微专业一点的术语，网关就是当前主机的默认路由。**

**网关一般就是一台路由器，有点像“一个小区中的一个邮局”，小区里面的住户互相是知道怎么走，但是要向外地投递东西就不知道了，怎么办？把地址写好送到本小区的邮局就好了。**

那么，怎么区分是“本小区”和“外地小区”的呢？根据IP地址 + 掩码。如果是在一个范围内的，就是本小区（局域网内部），如果掩不住的，就是外地的。

例如，你的机器的IP地址是：192.168.0.2/24，网关是192.168.0.1

如果机器访问的IP地址范围是：192.168.0.1~192.168.0.254的，说明是一个小区的邻居，你的机器就直接发送了（和网关没任何关系）。如果你访问的IP地址不是这个范围的，则就投递到192.168.0.1上，让这台设备来转发。





# API网关

## 概述

API网关（API Gateway），提供API托管服务，覆盖设计、开发、测试、发布、售卖、运维监测、安全管控、下线等API各个生命周期阶段，帮助您快速建设以API为核心的系统架构。

- 构建中台。通过API网关强大的适配和集成能力，可以将各种业务系统的API实现统一管理和统一调用：

  - 支持异构网络环境：无论您的业务系统部署在阿里云、本地数据中心、或其他云，API网关均可以统一管理；
  - 支持对接多种系统实现方式：您的业务系统基于ECS、基于函数计算、基于微服务等方式实现；
  - 支持数据的服务能力输出：您在使用Dataworks（一站式大数据智能云研发平台）、Dataphin（智能数据构建与管理）、DMS（数据库管理）等阿里云大数据、数据库方面的产品时，可以快速将数据API通过API网关发布，实现数据的服务化输出；
  - 支持AI能力输出：您可以将PAI（机器学习平台）上的完成的模型、算法，以API形态通过API网关发布，供各个业务系统进行调用；
  - 支持数据大屏：您可以直接在DataV（数据可视化）上直接调用API网关发布的API，实现数据的可视化展示。

- 构建各种技术架构。

  - Serverless架构。函数计算和API网关的联姻让众多开发者看到可以大展身手的代码空间，快速构建低成本、高可用、实时弹性伸缩的后端服务。业务触角延展到移动端、WEB端、物联网、云市场等多个数据来源，扩展业务无限可能，扩展商业无限边界，增加了产品组合的灵活度和柔韧度。
  - 微服务架构。API网关作为成熟的云产品，放在Kubernetes集群前面作为应用集群的接入服务使用，将大大提高Kubernetes集群的服务能力，可以作为标准的大型互联网应用的标准架构。
  - 前后台分离架构。在大规模的前后台分离架构中，需要由API网关承载起API认证、调度、路由职责，并且通过集群化的部署模式来解决高性能和大规模使用的问题。

- 构建生态。企业在经营过程中，会需要上下游合作伙伴的诸多业务协作。通过开放API，可以大大提高各个环节的协作效率，促成更加健康的商业生态。

- 跨界创新。API即能力、API即数据，通过API可以进行创新。一方面可以将您的算法、能力、数据以API商品的形态发布到 阿里云API市场，探索新的商业模式；另一方面也可以通过API网关，将其他厂商的API服务整合到自己的业务中，专注专业，借力发展。

  

## 优势

- 解放生产力
  - 托管式PaaS产品，围绕API生命周期的各个阶段提供生产力工具，大大提高API管理、运维效率，降低日常维护成本。
- 高性能高可靠
  - 分布式部署，承载大规模访问，低延迟处理。
- 安全
  - 提供全链路API安全保障方案。API网关提供了API认证、参数清洗、全链路签名、CA证书、访问控制、流量控制、内网访问等多种安全机制，并且可以与WAF、DDoS高防IP、IDaaS等结合使用，形成全链路API安全保障方案。
- 丰富的集成能力
  - 与腾讯云的计算产品、大数据产品、AI产品、数据可视化产品等深度集成，快速构建您的业务。
- 多种计费方式
  - 丰富多样的计费方式，满足各种使用场景。共享实例按调用量计费，适用于研发测试场景；专享实例按性能容量和使用时长计费，提供更高的SLA，适用于正式生产环境。

## 功能

- API网关支持以下功能
  - API 生命周期管理
    - 覆盖设计、开发、测试、发布、运维监测、安全管控、下线等API各个生命周期阶段，API网关为每个阶段提供生产力工具；
  - 协议处理
    - 支持HTTPS、SSL卸载、支持HTTP2.0、客户端websocket接入、双向通信；
    - 支持通过泛域名调用API；
  - 请求转发
    - 支持参数清洗，包括参数类型、参数值（范围、枚举、正则）校验，无效校验会被 API 网关直接拒绝；
    - 支持参数映射，实现前端、后端请求信息映射；
  - 安全防护支持多种认证方式
    - 支持 HMAC (SHA-1，SHA-256) 算法签名；
    - 支持HTTPS双向认证、全链路CA证书、全链路签名验证；
    - 支持IP访问控制；支持参数访问控制，可针对HTTP请求、应答或系统上下文中获取参数，并使用自定义的条件表达式对参数内容进行访问控制；
    - 请求防重放、请求防篡改、跨域访问，并与WAF、高防IP高效集成，形成全链路API防护体系；
  - 集成能力
    - 支持多种后端服务类型，包括HTTP(s)服务、模拟(Mock)、VPC内资源、函数计算等，可有效对接现有业务系统；
    - 支持数据服务，与Dataworks、Dataphin、DMS等阿里云大数据产品、数据库管理产品进行无缝对接，多种数据源、海量数据都能以API形式提供数据服务；
    - 支持VPC内网访问；
    - 支持混合云管理，对云上资源、云下资源的API进行统一管控；
  - 发布和路由
    - 提供API发布管理，线上版本一键切换；
    - 支持多环境管理，满足日常研发、预发测试、生产环境并行使用需求；
    - 支持灰度发布功能；
    - 支持参数路由，可根据HTTP请求内容创建不同的路由转发规则，提高系统灵活性；
  - API可用性
    - 支持API应答内容缓存，提高访问效率，减轻后端服务压力；
    - 提供默认断路器和自定义降级策略配置，避免极端情况下雪崩效应；
    - 支持精细化的流量控制，既能粗粒度的针对API访问频率、APP的请求频率进行流控设置，也能可针对HTTP请求、系统上下文中获取参数并加以逻辑判断，进行参数流控；
  - 监控告警
    - 快速设置，将API调用日志投递到日志服务(SLS)中，从而实现全量日志的查询分析；
    - 提供仪表盘对API调用情况进行监控，包括调用量、响应时间、错误率等，快速了解当前API情况；
    - 支持设置不同的告警条件，在API调用发生异常时，能够及时通过短信等方式通知管理员；
  - 调试和调用
    - 提供可视化的界面调试工作；
    - 支持API调用跟踪，追踪API执行全过程，快速追查排错；
    - 支持自动生成多种语言SDK，生成API说明文档，降低客户端API调用门槛；
  - 持续集成
    - 提供API网关的全量管控API；
    - 支持swagger导入导出，支持swagger 2.0标准，结合管控API，可有效与您的运维系统、CICD系统对接；
    - 支持Terraform编排；
  - API市场
    - 支持以API商品的方式上架到阿里云API市场，并提供多种计费方式。

## 场景

API 网关为您在各种场景下开放API 提供支撑，具体有：

- 支持建立 API 生态，将 API 开放给合作伙伴、开发者，实现企业核心能力的货币化；
- 支持将 API 适配多端，如：移动、互联网、物联，实现系统前后端分离；
- 支持内部系统整合，模块化、微服务化。

**1. 丰富的 API 生态，互相借力，协同发展**

用户日益膨胀的、碎片化的需求促使企业不断探索新的商业模式，以解决客户的各类场景化问题。API 网关提供了丰富的 API 生态。API 提供者在此提供标准的 API 服务，开发者在此将标准化的 API 服务整合进自己的应用，从而衍生出新的应用，新的服务。API 网关以此促进企业建立商业生态、跨界创新。

- 通过 API 网关将企业的核心能力，开放给合作伙伴，达成深度合作，协同发展；

- 将 API 接入阿里云市场，以 API 的形式开放能力、服务、数据供广大开发者采购使用，产生价值；

- 在 API 市场，采购第三方成熟的能力和服务，避免平铺式开发，专注专业，借力发展。

  [![拥抱 API 经济](http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/pic/29468/cn_zh/1484724886634/%E5%9C%BA%E6%99%AF1%E6%8B%A5%E6%8A%B1API%E7%BB%8F%E6%B5%8E-559-287%EF%BC%882x%EF%BC%89.jpg)](http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/pic/29468/cn_zh/1484724886634/场景1拥抱API经济-559-287（2x）.jpg)

**2. 安全地实现多端统一，一套服务，多端输出**

随着移动、物联网的普及，API 需要支持更多的终端设备，以扩充业务规模，但同时也带来系统复杂性的提升。通过 API 网关可以使 API 适配多端，企业只需要在 API 网关调整 API 定义，无需做额外工作。

- 企业只需维护一个服务体系，即可面向多端输出；只需调整API定义，即可实现对APP、设备、web端等多种终端的支持；

- 避免多个场景多套API，大幅降低管理运维成本。

  [![多端统一](http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/pic/29468/cn_zh/1484724903237/%E5%9C%BA%E6%99%AF3%E5%A4%9A%E7%AB%AF%E7%BB%9F%E4%B8%80.jpg)](http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/pic/29468/cn_zh/1484724903237/场景3多端统一.jpg)

**3. 轻松实现系统集成，规范化、标准化**

- 通过 API 网关对系统间接口进行规范统一，用标准化的接口实现系统集成；
- 快速完成资源整合和管理，消除快速发展造成的冗余和浪费，聚力发展业务。

[![系统集成](http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/pic/29468/cn_zh/1484724916639/%E5%9C%BA%E6%99%AF4%E7%B3%BB%E7%BB%9F%E9%9B%86%E6%88%90.jpg)](http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/pic/29468/cn_zh/1484724916639/场景4系统集成.jpg)

# API网关灰度发布最佳实践

本文主要用于介绍在API后端服务版本迭代过程中，新版本服务正式发布前通过API网关进行灰度发布，A/B Test的通用方法实践。该方法的核心是通过配置后端路由插件来确保可以控制服务升级对用户造成的影响。

## 一、概述

灰度发布是指在API的新、旧版本间平滑过渡的一种发布方式。API网关客户在应用迭代过程中会不断有新版本API发布，在新版本正式发布前，可以使用灰度流量控制先进行小规模验证，将升级带来的影响限定在指定的用户范围内可以最大程度上保障线上业务的稳定运行，通过收集使用体验的数据，对应用新版本的功能、性能、稳定性等指标进行评判，然后再全量升级。

常见的灰度发布和A/B Test场景中，需要根据最终用户的信息进行区分，借助API网关的路由插件功能，能够从API HTTP Request请求中识别出和最终用户相关的参数，如userId，并进行逻辑条件判断，从而将请求分流到不同的后端服务中。

## 二、典型场景

API网关后端路由插件支持的灰度分流条件

- 使用API网关的APP进行灰度：获得授权的APP事实上代表身份不同的API调用者，这种情况下当有两个以上授权APP的时候，可以根据调用者APP将请求转发到不同后端。假设，某个API授权给了多个应用，在API后端服务有升级的情况下，为了控制升级对线上可能存在的未知影响，我们可以参照本文3.1 给出的后端路由插件配置，将其中两个应用的请求路由到新版后端服务。
- 基于JWT的灰度：参考文档[JWT认证插件](https://help.aliyun.com/document_detail/103228.html)， 当API绑定JWT插件时，API网关可以从JWT中解析没有加密过的用户身份相关的claim，比如用户userId， 支持按照从 Token 中解析得到的参数路由到不同后端，从而实现灰度发布。比如用户可以参考本文 3.2 节给出的插件配置，通过限制userId字段的尾数，实现20%的用户请求到新版本服务。
- 基于HTTP Reuqest 内的用户参数进行灰度，当Query | Form | Header 中存在和用户身份直接相关的参数时，我们就可以直接利用该请求参数转发请求到不同后端。比如针对一部分使用老版本客户端的用户返回特定的提示信息，提示用户升级客户端，可以参考3.3节的说明进行配置。
- 针对特定访问IP段进行灰度：可以利用不同的请求源IP地址段进行后端路由，从而实现灰度发布。比如有的时候我们为了更高效的测试，需要在内测环境和生产环境使用完全相同的前端请求代码，可以参考3.4节的说明实现不同环境访问不同后端。

## 三、配置过程

本节是针对上述四种场景给出对应的后端路由插件配置，需要应用灰度发布的API绑定相应插件后，当请求符合条件时，请求会被转发到插件配置的后端， 否则将被正常访问API后端服务。当有多个灰度场景时，可以参考文档[后端路由](https://help.aliyun.com/document_detail/117198.html)在一个后端路由插件中配置多条路由。在插件条件表达式中使用的参数要按照文档[参数与条件表达式的使用](https://help.aliyun.com/document_detail/153405.html)的描述在插件中定义。

3.1 根据调用者APP进行灰度发布。如果用户需要指定授权给某些应用的的API调用请求进入灰度环境，可以利用系统参数 CaAppId 来配置路由条件，配置如下

```yaml
---
routes:
-  name: appGrey
   condition: "$CaAppId = 4534463 or $CaAppId = 86734572"
   backend:
      type: HTTP
      address: "http://example-test.alicloudapi.com:8080"
      path: "/web/xxx"
      method: GET
      timeout: 7000    
```

3.2 根据JWT中的自定义claim进行灰度发布。假设JWT中有名为userId的自定义claim，并且userId的尾数均匀分布。则按照如下的3个步骤进行配置，可以对20%的用户进行灰度。

- 用户需要先在API上绑定配置如下的JWT插件

```yaml
---
parameter: X-Token         # 从指定的参数中获取JWT, 对应API的参数
parameterLocation: header  # API为映射模式时可选, API为透传模式下必填, 用于指定JWT的读取位置, 仅支持`query`,`header`
claimParameters:           # claims参数转换, 网关会将jwt claims映射为后端参数
- claimName: userId        # claim名称,支持公共和私有
   parameterName: userId    # 映射后参数名称
   location: query          # 映射后的参数位置, 支持`query,header,path,formData`
#
# `Json Web Key`的`Public Key`
jwk:
   kty: RSA
   e: AQAB
   kid: showJwt
   alg: RS256
   n: gNHI8tm3lnsdCi09SrBPs9-Oau7Z1SFhIEOT2h5AJ49FSJA0XEyU4OadtV70BLIEy94dzcUK8f0e477AVoUO0RZdcXjztFtpJnA1Ktrzn9zAmKcXb2IuKXrBKkQStcKqoSbBlR84mDElp_gxfNqpmoLy0q08rkmjh1utd8E_S4QMDDaFtQ68ggJcDY-oX5FSiVidKNrKagEzQKpk5SgJFE8wpJOkW-YKouqLsL5lFyqnkgn7J3MvDqEBKqgiCY-zXYaxnkLNfkrAt7jTe4b4a2PiKD0-bHIZwzd2NVhuLGwx4pB1tFL51E-KeewZhTsoUbQ3v_ZerZ2_630WOH7IWQ
```

- 把灰度条件要用到的字段 userId 作为自定义 claim 加到JWT的payload部分，参考文档[API网关 OpenID Connect 使用指南](https://help.aliyun.com/document_detail/48019.html), 发起API请求的时候在请求header部分增加一个 X-Token 的头，值为使用私钥JWK对JWT进行签名生成的ID Token。
- 再绑定一个如下配置的后端路由插件，可以把userId尾数为“0”和 “1”的用户请求转发到指定的后端地址，从而实现对20%的用户进行灰度发布

```yaml
---
parameters:
    userId: "Token:userId"
routes:
-  name: userIdGrey
    condition: "$userId like '%0' or $userId like '%1'"
    backend:
       type: HTTP
       address: "http://example-test.alicloudapi.com:8080"
       path: "/web/xxx"
       method: GET
       timeout: 7000
```

3.3 根据客户端版本进行灰度发布。例如，需要灰度的API请求示例如下：

```shell
http://example-cn-hangzhou.alicloudapi.com/testJwtShow?clientVersion=1.9
```

需要对clientVersion小于2.0.5的版本进行灰度，配置如下：

```yaml
---
parameters: 
   clientVersion: "Query:clientVersion"
routes:
-  name: oldVersionClient
    condition: "$clientVersion < '2.0.5'"
    backend:
       type: "MOCK"
       statusCode: 400   	
```

3.4 根据请求源IP进行灰度发布。当一些服务内测时，生产环境和内测环境的有访问不同后端的需求，可以根据请求源IP分发到不同版本后端。如下配置，可以利用API网关系统参数CaClientIp 将源ip地址段为 106.11.31.0/24 的请求转发到指定的后端。

```yaml
routes:
-  name: InternalTesting
    condition: "$CaClientIp in_cidr  '106.11.31.0/24'"
    backend:
       type: HTTP
       address: "http://example-test.alicloudapi.com:8080"
       path: "/web/xxx"
       method: GET
       timeout: 7000	
```



# 可以在 Kubernetes 中通过 API 网关暴露微服务吗？



> **长话短说**：可以。看一下[ Kong ](https://konghq.com/blog/kong-kubernetes-ingress-controller/)、[ Ambassador ](https://www.getambassador.io/)、[ Gloo ](https://gloo.solo.io/)等 Ingress 控制器就知道了。

在 Kubernetes 中，Ingress 是一个组件，它将集群外部的流量路由到服务和集群内的 Pod。

简单来说，**Ingress 是作为反向代理或负载均衡器**：所有外部流量都先路由到 Ingress，然后再路由到其他组件。
![可以在Kubernetes中通过API网关暴露微服务吗？](https://uploader.shimo.im/f/gtlYq7scpQ8JqfiS.jpg!thumbnail)
虽然最流行的 Ingress 组件是[ ingress-nginx 项目](https://github.com/kubernetes/ingress-nginx)，但是，在选择和使用 Ingress 时还有其他几个选项。

你可以从以下 Ingress 控制器中选择：

- 处理 HTTP 流量，如[ Contour ](https://github.com/heptio/contour)和[ Treafik Ingress](https://docs.traefik.io/user-guide/kubernetes/)
- 支持 UDP 和 TCP 流量，如[ Citrix Ingress](https://github.com/citrix/citrix-k8s-ingress-controller)
- 支持 Websockets，如[ HAProxy Ingress](https://github.com/jcmoraisjr/haproxy-ingress)

还有其他混合 Ingress 控制器可以与现有的云提供商集成，比如 Zalando 的[ Skipper Ingress ](https://opensource.zalando.com/skipper/)。

谈到 Kubernetes 中的 API 网关，你有三个常见的选项。

## 选项 1——API 网关的王者：Kong

如果你正在构建 API，那么你可能会对[ Kong Ingress ](https://konghq.com/blog/kong-kubernetes-ingress-controller/)提供的功能感兴趣。

**Kong 是基于 Nginx 构建的一个 API 网关**

Kong 侧重于 API 管理，并提供诸如身份验证、速率限制、重试、断路器等功能。

有趣的是，Kong 被包装成一个 Kubernetes Ingress。因此，你可以在集群中使用它作为用户和后端之间的网关。你可以使用标准的 Ingress 对象向外部流量暴露你的 API：

复制代码

```
#ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            backend:
              serviceName: api-service
              servicePort: 80
```

不仅如此。作为安装过程的一部分，Kong 的控制器会注册自定义资源定义（CRD）。其中一个自定义扩展与 Kong 的插件相关。如果你希望以 IP 地址限制对你的 Ingress 的请求，则可以用以下方法定义该限制：

复制代码

```
#limit-by-ip.yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rl-by-ip
config:
  hour: 100
  limit_by: ip
  second: 10
plugin: rate-limiting
```

你可以在 ingress.yaml 文件中通过注解引用该限制：

复制代码

```
#ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    plugins.konghq.com: rl-by-ip
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            backend:
              serviceName: api-service
              servicePort: 80
```

你可以通过官方文档进一步了解[ Kong 自定义资源定义（CRD）](https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/custom-resources.md)。

但 Kong 不是唯一的选项。

## 选项 2——现代化 API 网关 Ambassador

[Ambassador ](https://www.getambassador.io/)是另外一个 Kubernetes Ingress，基于 Envoy 构建，提供了一个健壮的 API 网关。

Ambassador Ingress 是一个现代化的 Kubernetes Ingress 控制器，它提供了健壮的协议支持、速率限制、身份验证 API 和可观测性集成。

Ambassador 与 Kong 的主要区别在于 Ambassador 是为 Kubernetes 设计的，并与之很好地集成。

> Kong 是 2015 年开源的，当时 Kubernetes Ingress 控制器还没有那么先进。

但即使 Ambassador 在设计时考虑了 Kubernetes，它也没有利用人们熟悉的 Kubernetes Ingress。相反，服务是通过注解向外部暴露的：

复制代码

```
#service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    service: api-service
  name: api-service
  annotations:
    getambassador.io/config: |
      ---
      apiVersion: ambassador/v0
      kind: Mapping
      name: example_mapping
      prefix: /
      service: example.com:80
      host_rewrite: example.com
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
  selector:
    service: api-backend
```

这种新方法非常方便，因为你可以在一个地方为部署和 Pod 定义所有路由。**但是，在注解中使用 YAML 作为自由文本可能会导致错误和混淆。**在普通的 YAML 中都很难正确地设置格式，更不用说在更大的 YAML 中作为字符串了。如果希望对 API 应用速率限制，以下就是 Ambassador 中的做法。你有一个 RateLimiting 对象，它定义了需求：

复制代码

```
#rate-limit.yaml
apiVersion: getambassador.io/v1beta1
kind: RateLimit
metadata:
 name: basic-rate-limit
spec:
 domain: ambassador
 limits:
  - pattern: [{x_limited_user: "false"}, {generic_key: "qotm"}]
    rate: 5
    unit: minute
  - pattern: [{x_limited_user: "true"}, {generic_key: "qotm"}]
    rate: 5
    unit: minute
```

你可以在服务中像下面这样引用速率限制：

复制代码

```
#service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  annotations:
    getambassador.io/config: |
      ---
      apiVersion: ambassador/v1
      kind: RateLimitService
      name: basic-rate-limit
      service: "api-service:5000"
spec:
  type: ClusterIP
  selector:
    app: api-service
  ports:
    - port: 5000
      targetPort: http-api
```

关于速率限制，Ambassador 有一个很好的的教程，所以，如果你对使用该特性感兴趣，可以查看[ Ambassador 的官方文档](https://www.getambassador.io/user-guide/rate-limiting-tutorial/)。你可以使用[自定义的路由过滤器](https://www.getambassador.io/docs/guides/filter-dev-guide)来扩展Ambassador，但它没有提供像Kong 那样丰富的插件机制。

## 选项3——把东西粘到一起的Gloo

Ambassador 并不是唯一基于 Envoy 且可以用作 API 网关的 Ingress。 [Gloo 是一个 Kubernetes Ingress ](https://gloo.solo.io/)，也是一个 API 网关，能够提供速率限制、断路器、重试、缓存、外部身份验证和授权、转换、服务网格集成和安全性。

**Glue 的卖点是它能够自动发现应用程序的 API 端点，并自动理解实参和形参。**

我理解，这很难让人相信（他们的文档在这个意义上没有提供什么帮助），所以这里有一个例子。 假设你有一个用于地址簿的 REST API。该应用暴露了以下端点：

- GET /users/{id}：获取用户资料；
- GET /users：获取所有用户；
- POST /users/find：查找特定的用户。

如果你的 API 是使用标准工具（如 OpenAPI）开发的，那么 Gloo 将自动使用 OpenAPI 定义来检查你的 API 并存储这三个端点。如果在发现阶段之后列出 Gloo 提供的所有端点，你将看到：

复制代码

```
#gloo
upstreamSpec:
  kube:
    selector:
      app: addressbook
    serviceName: addressbook
    serviceNamespace: default
    servicePort: 8080
    serviceSpec:
      rest:
        swaggerInfo:
          url: http://addressbook.default.svc.cluster.local:8080/swagger.json
        transformations:
          findUserById:
            body:
              text: '{"id": {{ default(id, "") }}}'
            headers:
              :method:
                text: POST
              :path:
                text: /users/find
              content-type:
                text: application/json
          getUser:
            body: {}
            headers:
              :method:
                text: GET
              :path:
                text: /user/{{ default(id, "") }}
              content-length:
                text: '0'
              content-type: {}
              transfer-encoding: {}
          getUsers:
            body: {}
            headers:
              :method:
                text: GET
              :path:
                text: /users
              content-length:
                text: '0'
              content-type: {}
              transfer-encoding: {}
```

一旦 Gloo 有了一个端点列表，你就可以使用该列表在传入请求到达后端之前对其应用转换。例如，你可能希望在请求到达应用程序之前从传入的请求收集所有报头，并将它们添加到 JSON 的有效负载。

或者，你可以暴露 JSON API，并让 Gloo 应用转换，从而在消息到达遗留组件之前将其呈现为 SOAP。 能够发现 API 并应用转换，使得 Gloo 特别适合具有多种技术的环境——或者当你正处于从旧遗留系统迁移到新堆栈的过程中。

**Gloo 可以发现其他类型的端点，比如 AWS Lambdas**。 当你希望混合和搭配 Kubernetes 与无服务器时，它可以作为一个完美的搭档。

## 回顾

下面是对这三个 Ingress 控制器的简单回顾：

|          | Kong                                                         | Ambassador                           | Gloo                         |
| :------- | :----------------------------------------------------------- | :----------------------------------- | :--------------------------- |
| 协议     | http、https、grpc、tcp、udp                                  | http、https、grpc、tcp、udp、tcp+ssl | http、https、grpc            |
| 基础     | Nginx                                                        | Envoy                                | Envoy                        |
| 弹性     | 主动和被动健康检查、断路器、速率限制、重试                   | 速率限制                             | 速率限制、断路器、重试、缓存 |
| 状态     | 在 PostgreSQL 或 Cassandra 中配置，状态保存在 CRD 中         | Service                              | CRD                          |
| 路由定义 | 作为 Ingress 清单                                            | 在 Service 中                        | 作为 Ingress 清单            |
| 身份验证 | Basic Auth、HMAC、JWT、Key、LDAP、OAuth 2.0、PASETO、OIDC（仅限企业版） | Basic Auth、OIDC                     | Basic Auth、OIDC             |
| 可扩展性 | 插件                                                         | 无，外部集成                         | 插件                         |
| 仪表板   | Kong 企业仪表板或开源社区项目                                | Dashboard                            | 企业版仪表板                 |

*我应该用哪一个？*

- **如果你想要一个经过实战检验的 API 网关，Kong 仍然是你最好的选择。**它可能不是最耀眼的，但其文档是最好的，而且网上有很多资源。它的生产里程也比任何其他网关都要长。
- **如果你需要一个灵活的 API 网关**，它可以很好地与新旧基础设施配合使用，那么你应该看看**Gloo**。自动发现 API 和转换请求的能力非常有吸引力。
- **如果你希望在你的服务中设置所有网络的简单性，你应该考虑 Ambassador**。它有很好的入门教程和文档。注意，YAML 缩进是一个空字符串。











# API网关为K8s容器应用集群提供接入能力



## 1. Kubernetes 集群介绍

Kubernetes（k8s）作为自动化容器操作的开源平台已经名声大噪，目前已经成为成为容器玩家主流选择。Kubernetes在容器技术的基础上，增加调度和节点集群间扩展能力，可以非常轻松地让你快速建立一个企业级容器应用集群，这个集群主要拥有以下能力：

- 自动化容器的部署和复制
- 随时扩展或收缩容器规模
- 将容器组织成组，并且提供容器间的负载均衡
- 很容易地升级应用程序容器的新版本
- 提供容器弹性，如果容器失效就替换它

下面是一个典型的Kubernetes架构图：

[![img](http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/pic/71623/cn_zh/1525425558172/d56441427680948fb56a00af57bda690.png)](http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/pic/71623/cn_zh/1525425558172/d56441427680948fb56a00af57bda690.png)

## 2. API网关作为Kubernetes集群的接入层架构

我们可以看到Kubernetes集群是有足够理由作为应用服务的首选，但是Kubernetes集群没有足够的接入能力，特别在大型应用中，它是不能够直接对用户提供服务的，否则会有非常大的安全风险。而API网关作为成熟的云产品，已经集成了非常丰富的接入能力，把API网关放在Kubernetes集群前面作为应用集群的接入服务使用，将大大提高Kubernetes集群的服务能力，可以作为标准的大型互联网应用的标准架构。下面是使用阿里云架构图：

[![img](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/3706367951/p96242.png)](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/3706367951/p96242.png)



**是否启用Ingress Control？**



从架构图中我们可以看到，API网关作为Kubernetes集群的桥头堡，负责处理所有客户端的接入及安全工作，API网关和Kubernetes集群中Ingress Control的内网SLB或者服务的内网SLB进行通信。具体什么时候用Ingress Control呢？如果Kubernetes集群内只有一个服务，网关直接和此服务关联的内网SLB进行通信最高效。如果Kubernetes集群内有多个服务，如果使用服务的内网SLB对网关提供服务，将会生成很多SLB，资源管理起来会比较麻烦，此时我们可以使用Ingress Control做Kubernetes中服务发现与七层代理工作，API网关的所有请求发送到Ingress Control关联的内网SLB上，由Ingress Control将请求分发到Kubernetes集群内的容器内，这样我们也只需要一个内网SLB就能将Kubernetes集群内的所有服务暴露出去了。

## 3. API网关接入能力

读者要问了，接入了API网关具体能为整个架构带来哪些好处呢？下面我们列一下这种架构中，API网关具体能给整个应用带来什么价值。

1.API网关允许客户端和API网关使用多种协议进行通信，其中包括：

```
* HTTP  
* HTTP2  
* WebScoket
			
```

2.API网关使用多种方法保证和客户端之间的通信安全：

```
* 允许定义API和APP之间的授权关系，只有授权的APP允许调用；
* 全链路通信都使用签名验证机制，包括客户端和API网关之间的通信和API网关和后端服务之间的通信，保证请求在整个链路上不会被篡改；
* 支持用户使用自己的SSL证书进行HTTPS通信；
* 支持OPENID CONNECT；
			
```

3.API网关具备iOS/Android/Java三种SDK的自动生成能力，并且具备API调用文档自动生成能力；

4.API网关支持入参混排能力，请求中的参数可以映射到后端请求中的任何位置；

5.API网关提供参数清洗能力，用户定义API的时候可以指定参数的类型，正则等规则，API网关会帮用户确认传输给后端服务的请求是符合规则的数据；

6.API网关支持流量控制能力，支持的维度为用户/APP/API；

7.API网关提供双向通信的能力；具体双向通信相关的使用规则请参考：https://help.aliyun.com/document_detail/66031.html

8.API网关提供基于请求数/错误数/应答超时时间/流量监控报警能力，所有的报警信息会使用短信或者邮件在一分钟内发出；

9.API网关已经和阿里云的SLS产品打通，用户可以将所有请求日志自动上传到用户自己的SLS中，后继好对访问日志进行统计分析；

10.API网关支持Mock模式，在联调中这个能力非常方便；

11.API网关支持用户配置调用方的IP白名单和黑名单； 12.API网关结合阿里云的云市场，为Provider提供向API使用者收费的能力。

阿里云的API网关是一个上线数年的成熟云产品，在稳定性和性能方面，经过了时间和阿里云的工程师的不断打磨，有高性能需求的用户尽管放马过来。

## 4. 在阿里云快速配置Kubernetes集群和API网关

阿里云支持快速创建Kubernetes集群，同一个Region内的Kubernetes集群和API网关的集成也非常简单，下面我们来一步一步地在阿里云中配置出本文第二节中架构设计。第二节中的架构设计中，API网关和Kubernetes有两种结合的模式，一种是API网关将请求发送到Ingress Control前的SLB，由Ingress Control将请求路由到Kubernetes对应的节点中，第二种是API网关直接将请求发送到Kubernetes中服务前的SLB，由SLB直接将请求转发到Kubernetes中服务对应的节点中。第一种模式只需要Ingress Control前有一个SLB就可以了，由Ingress Contro做服务发现与路由，第二种模式每个服务前都需要申请一个SLB，适合并发量大的场景。

## 4.1 Ingress Control模式的配置方式

## 4.1.1 创建带有Ingress Control组件的Kubernetes集群

首先我们来通过控制台创建一个具备Ingress Control组件的Kubernetes集群。

1.进入Kubernetes集群管理控制台界面：https://cs.console.aliyun.com/#/k8s/cluster/list

2.点击左上角“创建Kubernetes集群”按钮，进入

[![img](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/3706367951/p96396.png)](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/3706367951/p96396.png)

3.在创建Kubernetes集群页面选择不同规格的，具体创建选项和常规创建参数一致，在组件配置子页面，需要注意勾选创建Ingress组件：

[![img](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/3706367951/p96275.png)](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/3706367951/p96275.png)

4.创建成功后可以在Kubernetes列表页面看到刚才创建的集群。

## 4.1.2 在Kubernetes集群内创建一个多容器的服务

现在集群有了，我们需要在集群内创建一个服务，这个服务由2个容器组成，每个容器都由最新的Tomcat镜像生成。容器的端口是8080，Ingress Control提供服务的端口是80。

1.进入Kubernetes集群管理控制台界面：https://cs.console.aliyun.com/#/k8s/cluster/list

2.进入Kubernetes集群的控制台页面后，点击左边菜单栏的“应用”菜单下的“无状态”按钮，进入应用列表页面后，点击右上角的“使用模板创建”按钮进入创建页面：

[![img](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96278.png)](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96278.png)

3.进入创建页面后，点击使用文本创建按钮，输入下面的资源编排文本点击上传按钮进行创建：

```
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: tomcat-demo
  labels:
    app: tomcat
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      containers:
      - name: tomcat
        image: tomcat:latest
        ports:
        - containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: tomcat
  sessionAffinity: None
  type: NodePort
			
```

对这段编排模板创建文本做下解释：使用最新的Tomcat镜像创建两个容器的意思，容器的服务端口是8080，并且创建一个命名为tomcat-service的服务，对外服务的端口为80，映射到容器8080的端口；



好了，目前为止，我们已经创建了一个Kubernetes集群，并且在这个集群下面创建了两个容器，每个容器上面跑着一个最新的Tomcat。这两个容器组成一个无状态应用，并且组成了一个命名为tomcat-service的服务。我们可以进入无状态应用详情页面看到整个应用的运行情况。目前我们的API网关没有办法访问到这个服务，需要我们在Ingress Control上建立一条到这个服务的路由才能把全链路打通

## 4.1.3 在Ingress Control上给服务加上路由

现在我们有应用了，还需要在Ingress Control上为这个应用建立一个服务，然后在服务上建立一条路由，这样API网关将请求发送给Ingress Control时，Ingress Control会根据配置的路由信息将请求代理到对应的应用节点上。

1.在容器服务控制台上点击“路由与负载均衡”菜单下的“服务”按钮，点击右上角的“创建”按钮；

2.在创建路由页面填写服务对应的域名，所监听的端口等：

[![img](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96324.png)](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96324.png)

## 4.1.4 在API网关对Ingress Control的内网SLB授权

API网关如果要访问Ingress Control的内网SLB，需要增加VPC网络的授权，首先我们需要准备VPC的标识和内网SLB的实例ID，我们可以在SLB控制台获取。

1.在Kubernetes管理控制台的“服务于负载均衡”菜单下点击“服务”菜单，进入服务管理页面后，选择命名空间为"所有命名空间"后可以在列表中看到Ingress Control的内网SLB地址：

[![img](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96341.png)](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96341.png)



2.登录SLB控制台（https://slb.console.aliyun.com/），找到刚才创建Kubernetes服务时自动创建的VPC，点击进去查看详情，我们可以在这个页面看到VPC的标识：

[![img](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96332.png)](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96332.png)好了，目前我们已经获取到了VPC的ID和SLB的实例ID,下面我们来创建API网关的授权：

3.进入API网关授权页面：https://apigateway.console.aliyun.com/#/cn-beijing/vpcAccess/list ，点击右上角的创建授权按钮，弹出创建VPC授权的小页面，将刚才查询到的VPC标识和SLB实例ID填入到对应的内容中：

[![img](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96333.png)](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96333.png)

点击确认按钮后，授权关系就创建好了，需要记住刚才填写的授权名称。

## 4.1.5 创建API

在API网关控制台创建API的时候，后端服务这块，有两点需要注意的：

1.我们需要填写刚才创建的VPC授权名称；

2.在创建API页面的常量参数添加一个名字为host，位置header的参数，参数值设置为4.3节中设置的路由中填写的域名。

[![img](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96380.png)](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96380.png)

API定义的其他字段的描述请参阅文档：https://help.aliyun.com/document_detail/153344.html

## 4.1.6 调用测试

API建立好了以后，我们把API发布到线上就可以使用API网关的测试工具进行测试，看看能否将请求发送到刚才创建的Kubernetes集群中去。 我们进入刚才创建好的API的详情页面，点击左侧菜单中的调试API，进入调试页面。在调试页面填写好相应的参数，点击“发起请求”按钮。

[![img](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96378.png)](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96378.png)

我们可以看到，请求发送到了Kubernetes的集群中的容器中，并且收到了容器中tomcate的404的应答（因为没有配置对应的页面）。

## 4.2 Kubernetes服务内网SLB结合模式的配置方式

通过Kubernetes服务的SLB结合API网关与Kubernetes的模式的配置方式与前一节中通过Ingress Control结合模式的配置方式有两点不同：1.申请Kubernetes中的容器服务时，需要指定生成内网SLB，2.找到这个SLB的VPC ID与SLB ID，使用这两个ID到API网关去进行授权即可。上一节中描述的创建Kubernetes集群的步骤，本节不再冗余描述。本节主要描述创建携带内网SLB的Kubernetes服务，并且找到这个内网SLB的IP地址。在找到SLB的IP地址后，具体授权方法和4.1.4中描述的一致，本节也不再重复描述。

## 4.2.1 生成携带内网SLB的Kubernetes服务

在4.1.1操作之后，我们有了一个Kubernetes集群，现在我们通过资源编排文本在这个集群中创建带有内网SLB的服务。我们注意下，本段资源编排代码和4.1.2中的资源编排代码不同的是，我们把最后一行的Type变成了LoadBalancer，并且指定了这个SLBLoadBalancer为内网：

```
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: tomcat-demo
  labels:
    app: tomcat
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      containers:
      - name: tomcat
        image: tomcat:latest
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
  annotations:
    service.beta.kubernetes.io/alicloud-loadbalancer-address-type: intranet
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: tomcat
  type: LoadBalancer
			
```

对这段编排模板创建文本做下解释：使用最新的Tomcat镜像创建两个容器的意思，容器的服务端口是8080，并且创建一个命名为tomcat-service的服务，这个服务前有一个内网SLB，对外服务的端口为80，映射到容器8080的端口。

## 4.2.2 在Kubernetes控制台找到服务的内网SLB地址

现在我们成功创建了一个携带内网SLB的服务，我们可以在Kubernetes控制台的“路由与负载均衡”菜单的“服务”子页面找到这个内网SLB的内网IP：[![img](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96549.png)](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/4706367951/p96549.png)

找到内网SLB之后，就可以像4.1.4中一样，去SLB的控制台找到这个SLB的VPC ID和SLB ID，并且使用这个SLB的VPC ID和SLB ID到API网关去授权了。

## 5 总结

让我们总结一下本文的内容，在前三节中，我们描述了Kubernetes集群和API网关的各项能力，并且画出了结合他俩作为后端应用服务生产的架构图。我们结合API网关+Kubernetes集群的架构，让系统具备了动态伸缩，动态路由，支持多协议接入，SDK自动生成，双向通信等各项能力，成为可用性更高，更灵活，更可靠的一套架构。

## 5.1 配置总结

在最后一节，我们描述了在阿里云的公有云如何创建一整套API网关加Kubernetes的流程，关键点在于Kubernetes集群通过Ingress Control组件服务发现，在API网关对Ingress Control组件的内网SLB进行授权后，将所有请求发送到SLB上。下面我们再总结一下通过Ingress结合API网关与Kubernetes的步骤：

1. 创建一个带有Ingress Control组件的Kubernetes的集群；
2. 使用资源编排命令，在Kubernetes集群中创建两个运行这最新版本的Tomcat的容器，并且基于这两个容器创建对应的服务；
3. 在控制台上给Ingress Control加上一条服务路由；
4. 到SLB控制台找到Ingress Control内网SLB的实例ID，在API网关创建一个VPC授权；
5. 在API网关创建API，后端服务使用刚才创建的VPC授权，并且设定一个名为host的参数，参数值使用Ingress Control服务路由设置的域名。API的请求将发送到Kubernetes集群的Ingress Control的SLB上，由Ingress Control将请求路由到容器内部的Tomcat服务上。

在高并发场景或者只有一个服务的场景，我们可以跳过Ingress Control，直接在服务前面架设一个内网SLB，并且将这个内网SLB授权给API网关，供API网关进行访问。

## 5.2 Ingress Control和SLB两种方案对比

\1. 使用SLB+Ingress Control。Ingress Control可以做Kubernetes集群的服务发现和七层代理工作，如果Kubernetes集群中有多个服务，可以统一使用Ingress Control进行路由，同时只需要一个内网SLB就可以对外暴露多个服务。便于运维管理和Kubernetes集群的服务扩充。推荐使用此种方式。

\2. 直接使用SLB。如果集群中某个服务的业务压力很大，可以考虑为此服务单独建立一个SLB，API网关直接连接此SLB，从而达到更高的通信效率。此种方式的弊端也比较明显，如果Kubernetes集群内有多个服务，需要为每个服务配置一个SLB，因此给运维管理带来较大工作量。



