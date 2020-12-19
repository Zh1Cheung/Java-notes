# AWS

## 什么是 IAM？

- AWS Identity and Access Management (IAM) 是一种 Web 服务，可以帮助您安全地控制对AWS资源的访问。您可以**使用 IAM 控制对哪个用户进行身份验证 (登录) 和授权 (具有权限) 以使用资源。**

  - **您首次创建 AWS 账户时，最初使用的是一个对账户中所有 AWS 服务和资源有完全访问权限的单点登录身份。**此身份称为 AWS 账户 *根用户*，可使用您创建账户时所用的电子邮件地址和密码登录来获得此身份。强烈建议您不使用 根用户 执行日常任务，即使是管理任务。

- **身份和访问管理 (简称为 IAM 或 IdAM) 是一种说明客户身份及其行为权限的方式。**

- IAM 功能

  - **对您 AWS 账户的共享访问权限**
  - **精细权限**
    - 您可以针对不同资源向不同人员授予不同权限。
  - **在 Amazon EC2 上运行的应用程序针对 AWS 资源的安全访问权限**
    - 您可以使用 IAM 功能安全地为 EC2 实例上运行的应用程序提供凭证。这些凭证为您的应用程序提供权限以访问其他 AWS 资源。
  - **多重验证 (MFA)**
    - 借助 MFA，您或您的用户不仅必须提供使用账户所需的密码或访问密钥，还必须提供来自经过特殊配置的设备的代码。
  - **联合身份**
    - 您可以允许已在其他位置（例如，在您的企业网络中或通过 Internet 身份提供商）获得密码的用户获取对您 AWS 账户的临时访问权限。
  - **实现保证的身份信息**

- IAM工作原理

  - IAM 提供了控制您的账户的身份验证和授权所需的基础设施。
  - 条款
    - 资源
      - **存储在 IAM 中的用户、组、角色、策略和身份提供商对象。**与其他 AWS 服务一样，您可以在 IAM 中添加、编辑和删除资源。
    - 身份
      - **用于标识和分组的 IAM 资源对象**。您可以将策略附加到 IAM 身份。其中包括用户、组和角色。
    - 实体
      - AWS 用于进行身份验证的 IAM 资源对象。这些对象包括 IAM 用户、联合用户和代入的 IAM 角色。
    - 委托人
      - 使用 AWS 账户根用户、IAM 用户或 IAM 角色登录并**向 AWS 发出请求的人员或应用程序**。
  - 请求
    - 在委托人尝试使用 AWS 管理控制台、AWS API 或 AWS CLI 时，该委托人将向 AWS 发送*请求*。请求包含以下信息：
      - **操作** – 委托人希望执行的操作。这可以是 AWS 管理控制台中的操作或者 AWS CLI 或 AWS API 中的操作。
      - **资源** – 对其执行操作的 AWS 资源对象。
      - **委托人** – 已使用实体（用户或角色）发送请求的人员或应用程序。有关委托人的信息包括与委托人用于登录的实体关联的策略。
      - **环境数据** – 有关 IP 地址、用户代理、SSL 启用状态或当天时间的信息。
      - **资源数据** – 与请求的资源相关的数据。这可能包括 DynamoDB 表名称或 Amazon EC2 实例上的标签等信息。
    - AWS 将请求信息收集到*请求上下文*中，后者用于评估和授权请求。
  - 身份验证
    - 要从 API 或 AWS CLI 中进行身份验证，您必须提供访问密钥和私有密钥。您还可能需要提供额外的安全信息。例如，AWS 建议您使用多重身份验证 (MFA) 来提高账户的安全性。
  - 授权
    - **在授权期间，AWS 使用请求上下文中的值来检查应用于请求的策略**。然后，它使用策略来确定是允许还是拒绝请求。大多数策略作为 [JSON 文档](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/access_policies.html#access_policies-json)存储在 AWS 中，并指定委托人实体的权限。
  - 操作
    - **在对您的请求进行身份验证和授权后，AWS 将批准请求中的操作。**操作是由服务定义的，包括可以对资源执行的操作，例如，查看、创建、编辑和删除该资源。
  - 资源
    - 在 AWS 批准请求中的操作后，可以对您的账户中的相关资源执行这些操作。资源是位于服务中的对象。示例包括 Amazon EC2 实例、IAM 用户和 Amazon S3 存储桶。

- AWS中的用户

  - 仅限首次访问：您的根用户凭证
    - 创建 AWS 账户时，会创建一个用于登录 AWS 的 AWS 账户根用户身份。您可以使用此根用户身份（即，创建账户时提供的电子邮件地址和密码）登录 AWS 管理控制台。
  - IAM 用户
    - **IAM 用户不是单独的账户；它们是您账户中的用户。每个用户都可以有自己的密码以用于访问 AWS 管理控制台。**您还可以为每个用户创建单独的访问密钥，以便用户可以发出编程请求以使用账户中的资源。比如用户 Li、Mateo、DevApp1、DevApp2、TestApp1 和 TestApp2 已添加到单个 AWS 账户。每个用户都有自己的凭证。
    - 请注意，某些用户实际上是应用程序（例如 DevApp1）。**IAM 用户不必表示实际人员；您可以创建 IAM 用户以便为在公司网络中运行并需要 AWS 访问权限的应用程序生成访问密钥。**
  - 联合现有用户
    - 如果您的组织中的用户已通过某种方法进行身份验证 (例如，通过登录到您的公司网络)，则不必为他们创建单独的 IAM 用户。相反，您可以在 AWS 中对这些用户身份进行*联合身份验证*。
    - **您的用户已在公司目录中拥有身份。**
      - 如果您的公司目录与安全断言标记语言 2.0 (SAML 2.0) 兼容，则可以配置公司目录以便为用户提供对 AWS 管理控制台的单一登录 (SSO) 访问。
    - **您的用户已有 Internet 身份。**
      - 如果您创建的移动应用程序或基于 Web 的应用程序可以允许用户通过 Internet 身份提供商 (如 Login with Amazon、Facebook、Google 或任何与 OpenID Connect (OIDC) 兼容的身份提供商) 标识自己，则应用程序可以使用联合访问 AWS。

- IAM中的权限和策略

  - **您在 AWS 中通过创建策略并将其附加到 IAM 身份（用户、用户组或角色）或 AWS 资源来管理访问权限（授权）。**策略是 AWS 中的对象；在与身份或资源相关联时，策略定义它们的权限。在委托人使用 IAM 实体（如用户或角色）发出请求时，AWS 将评估这些策略。

  - 策略和账户

    - 如果您管理 AWS 中的**单个账户，则使用策略定义该账户中的权限**。如果您**管理跨多个账户的权限**，则管理用户的权限会比较困难。您**可以将 IAM 角色、基于资源的策略或访问控制列表 (ACL) 用于跨账户权限。**但是，如果您**拥有多个账户**，那我们建议您改用该 **AWS Organizations 服务**来帮助您管理这些权限。

  - 策略和用户

    - IAM 用户是服务中的身份。当您创建 IAM 用户时，他们无法访问您账户中的任何内容，**直到您向他们授予权限。向用户授予权限的方法是创建基于身份的策略**，这是附加到用户或用户所属组的策略。

    - ```json
      // 示例演示一个 JSON 策略，该策略允许用户对 us-east-2 区域内的 123456789012 账户中的 Books 表执行所有 Amazon DynamoDB 操作 (dynamodb:*)。
      {
        "Version": "2012-10-17",
        "Statement": {
          "Effect": "Allow",
          "Action": "dynamodb:*",
          "Resource": "arn:aws:dynamodb:us-east-2:123456789012:table/Books"
        }
      }
      ```

    - IAM 控制台中提供了策略摘要表，这些表总结了策略中对每个服务允许或拒绝的访问级别、资源和条件。**策略在三个表中概括：[策略摘要](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/access_policies_understand-policy-summary.html)、[服务摘要](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/access_policies_understand-service-summary.html)和[操作摘要](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/access_policies_understand-action-summary.html)。**策略摘要*表包含服务列表。选择其中的服务可查看*服务摘要。该摘要表包含所选服务的操作和关联权限的列表。您可以选择该表中的操作以查看操作摘要。该表包含所选操作的资源和条件列表。

  - 策略和组

    - **可以将 IAM 用户组织为 *IAM 组*，然后将策略附加到组。**这种情况下，各用户仍有自己的凭证，但是组中的所有用户都具有附加到组的权限。**使用组可更轻松地管理权限**，并遵循我们的 [IAM 中的安全最佳实践](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/best-practices.html)。

  - 联合身份用户和角色

    - 联合身份用户无法通过与 IAM 用户相同的方式在您的 AWS 账户中获得永久身份。**要向联合身份用户分配权限，可以创建称为*角色* 的实体，并为角色定义权限。当联合用户登录 AWS 时，该用户会与角色关联，**被授予角色中定义的权限。

  - 基于身份和基于资源的策略

    - 基于身份的策略是附加到 IAM 身份（如 IAM 用户、组或角色）的权限策略。基于资源的策略是附加到资源（如 Amazon S3 存储桶或 IAM 角色信任策略）的权限策略。

- 开始设置 IAM

  - **利用 IAM，您可以在您的 AWS 账户的伞形结构下创建多个 IAM 用户，或通过与企业目录的联合身份实现临时访问。在某些情况下，您也可以实现跨 AWS 账户访问资源。**
  - 但是，如果不使用 IAM，则只有两种方法：一种是您必须创建多个 AWS 账户（即，每个账户都有自己针对 AWS 产品的计费和订阅），另一种是您的员工必须共享一个 AWS 账户的安全凭证。此外，如果不使用 IAM，您无法控制特定用户或系统可以完成的任务以及他们可能使用哪些 AWS 资源。

- 使用 IAM 为用户授予对您的 AWS 资源的访问权限

  - 以下是您可以使用 IAM 控制您的 AWS 资源访问权限的几种方式：

    | 访问权限类型                                                 | 我为什么使用这种方式？                                       |
    | :----------------------------------------------------------- | :----------------------------------------------------------- |
    | AWS 账户中**用户的访问权限**                                 | 您想要在您的 AWS 账户的伞形结构下添加用户，并希望使用 IAM 创建用户并管理他们的权限。 |
    | 通过您的认证系统和 AWS 之间的联合身份验证实现非 AWS 用户访问 | 您的身份和认证系统中有非 AWS 用户，他们需要访问您的 AWS 资源。 |
    | **AWS 账户间的跨账户访问权限**                               | 您想要与其他 AWS 账户下的用户共享对特定 AWS 资源的访问权限。 |





## 开始使用 IAM

- 在下图所示的简单示例中，AWS 账户有三个组。一个组由一系列具有相似责任的用户组成。在此示例中，一个组为管理员组（名为 *Admins*）。另外还有一个 *Developers* 组和一个 *Test* 组。每个组均包含多个用户
  - ![       AWS 账户、组和用户布局示例     ](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/images/Account_Group_Example.diagram.png)
  - 在随后的流程中，您需要执行下列任务：
    - 创建管理员组并向该组提供访问您 AWS 账户的所有资源的权限。
    - 为您自己创建一个用户并将该用户添加到管理员组。
    - 为您的用户创建密码，以便可以登录 AWS 管理控制台。



# 词汇

- **什么是访问控制？**

  - 通常，访问控制属于物理访问控制或信息访问控制的范畴。
    - **物理访问控制是一组用于控制谁有权进入某一物理位置的策略。**
      - 实践中的物理访问控制示例包括：
        - 吧台保镖
        - 地铁转门
        - 机场报关员
        - 酒店客房钥匙卡扫描器
      - 在所有这些示例中，人员或设备遵循一组策略来决定谁有权进入受限的物理位置。例如，酒店钥匙卡扫描器仅允许持有酒店钥匙的授权客人进入房间。
  - 什么是信息访问控制？
    - 信息访问控制限制对数据以及用于操作该数据的软件的访问。信息访问控制最常在计算机和网络安全中使用。一些示例包括：
      - 用户使用密码登录笔记本电脑
      - 用户通过指纹扫描解锁智能手机
      - Gmail 用户登录电子邮件帐户
      - 远程员工使用 VPN 访问雇主的内部网络
    - 所有这些情形中都使用软件对希望访问数字信息的用户进行身份验证和授权。**身份验证和授权都是信息访问控制的组成部分。**
  - 身份验证和授权有什么区别？
    - 简而言之，身份验证是一种确认某人**是否是其声称的身份**的安全实践，而授权则与授予各个用户的访问权限级别有关。
      - 例如，假设一个旅行者正在办理酒店入住。**他们在前台登记时，被要求出示护照以证明其姓名确实与预订单上相符。这是身份验证的一个例子。**
      - **一旦酒店员工验证了客人的身份，客人便收到具有有限特权的钥匙卡。**这就是授权的一个示例。客人的钥匙卡授予他们权限，让他们能够进入其房间、客人电梯和游泳池。这张钥匙卡不能打开其他客人的房间，也不能呼叫服务电梯，因为客人无权进入这些位置。

- **什么是身份提供商 (IdP)？**

  - **身份提供商（IdP 或 IDP）是一种存储和管理用户数字身份的服务。I**dP 可以比作一种来宾名单，但适用的是数字和云托管应用程序，而不是活动。IdP 可以通过用户名密码组合和其他因素来检查用户身份，也可简单地提供用户身份列表供其他服务提供商（如 SSO）检查。**IdP 最常在[云计算](https://www.cloudflare.com/zh-cn/learning/cloud/what-is-the-cloud)中用于管理用户身份。**

  - 为什么需要 IdP？

    - 数字身份必须在某个地方进行跟踪，尤其是对于云计算，其中用户身份决定了某个人能不能访问敏感数据。**云服务需要确切知道在哪里以及如何检索和验证用户身份。**

  - IdP 如何与 SSO 服务搭配？

    - [SSO 或单点登录](https://www.cloudflare.com/zh-cn/learning/access-management/what-is-sso)服务提供一个统一的场所，供用户一次登录所有云服务。除了为用户带来更多便利之外，实施 SSO 通常还可提升用户登录的安全性。

    - 在大多数情况下，SSO 和 IdP 是分开的。**SSO 服务使用 IdP 来检查用户身份，但实际上并不存储用户身份。**SSO 提供商更接近于中转站，而非一站式店铺；**它可以比作一家安保机构，被雇用来维护公司安全，但实际上并不从属于公司。**

    - SSO 和 IdP 在理论上可以成为一体。但这种此设置更容易受到[中间人](https://www.cloudflare.com/zh-cn/learning/security/threats/man-in-the-middle-attack)攻击，**攻击者通过伪造 [SAML ](https://www.cloudflare.com/zh-cn/learning/access-management/what-is-saml)断言**来获得对应用程序的访问权。**因此，IdP 和 SSO 通常是分开的**

      > *SAML 断言是从 SSO 服务发送到任何云应用程序的专用消息，用于确认用户身份验证，从而允许用户访问和使用该应用程序。*

      > **这一切在实践中会如何？**假设爱丽丝在雇主的办公室中使用她的工作笔记本电脑。爱丽丝需要登录公司的实时聊天应用程序，以便与同事更好地进行沟通。她在浏览器中打开一个标签页，然后加载聊天应用程序。假设她的公司使用 SSO 服务，那么幕后会发生以下步骤：
      >
      > - 聊天应用程序向 SSO 询问爱丽丝的身份验证情况。
      > - SSO 发现爱丽丝尚未登录。
      > - SSO 提示爱丽丝登录。
      >
      > 此时，爱丽丝的浏览器将她重定向到 SSO 登录页面。页面上提供一些栏位，供爱丽丝输入用户名和密码。由于公司要求[两步验证](https://www.cloudflare.com/zh-cn/learning/access-management/what-is-two-factor-authentication)，因此爱丽丝还必须输入一个简短的代码，SSO 会自动将这个代码发送到她的智能手机上。完成此操作后，她单击“登录”。现在会发生以下情况：
      >
      > - SSO 向爱丽丝公司使用的 **IdP** 发送 SAML 请求。
      > - IdP 向 SSO 发送 SAML 响应，以确认爱丽丝的身份。
      > - SSO 将 SAML 断言发送到爱丽丝最初想要使用的聊天应用程序。

- **什么是 SAML？**

  - 安全断言标记语言（SAML）是一种标准方法，**用于告知外部应用程序和服务某一用户正是其声称的身份。**SAML 提供一种方法来对用户进行一次身份验证，然后**将该身份验证传递给多个应用程序**，从而**使[单点登录（SSO）](https://www.cloudflare.com/zh-cn/learning/access-management/what-is-sso)技术成为可能。**

    - SAML 身份验证可以比作身份证，即一种简略展示一个人是谁的标准化方式。譬如，不必进行一系列的 DNA 测试来确认一个人的身份，只需看一眼这个人的身份证便可。

  - 什么是单点登录（SSO）？

    - 单点登录（SSO）是一种可以同时针对多个应用程序和服务验证用户身份的方式。使用 SSO 时，用户可以在一个登录屏幕中登录，然后就能使用多个应用程序。用户无需针对他们使用的每一项服务确认自己的身份。
    - **为实现这个目标，SSO 系统必须与每个外部应用程序通信，告知它们用户已经登录，而这是 SAML 发挥作用的地方。**

  - SAML 如何工作？

    - 典型的 SSO 身份验证过程涉及以下三方：
      - 当事人（有时也称为“主体”）
      - 身份提供者
        - 身份提供者（IdP）是一种云软件服务，通常通过登录过程来存储和确认用户身份。本质上，IdP 的职责是指出“我认识这个人，这是此人被允许做的事情”。
      - 服务提供商
        - 这是用户希望使用的云托管应用程序或服务。
    - 典型的流程如下：当事人向服务提供商发出请求。服务提供商接着向身份提供者请求身份验证。身份提供者将 **SAML**断言发送给服务提供商，然后服务提供商可以将响应发送给当事人。
    - 1. 用户想要访问[http://www.abc.com](https://link.zhihu.com/?target=http%3A//www.abc.com)的资源 
      2. SP发送了一个重定向消息给浏览器。重定向消息的HTTP报文头中包含了Idp登录服务的URI地址，同时还有一个被称为SAMLRequest的认证请求变量。浏览器处理该重定向请求：发起一个GET请求到Idp登录地址并将SAMLRequest作为请求参数。
      3. Idp的单点登录服务要求用户提供身份凭证。
      4. 用户输入自己的身份凭证。 
      5. Idp的单点登录服务发送一个HTML表单给浏览器。HTML表单中包含了SAML响应，该响应中是一个SAML断言。SAML规范要求SAML断言必须被数字签名。
      6. 浏览器收到SAML响应之后，会发起一个请求到SP的AssertionConsumerService，请求中包含了接收到的SAML响应。 
      7. SP的Assertion Consumer验证SAML响应中数字签名的合法性。如果验证通过，他会发起一个到目标资源的重定向请求，请求的cookie中包含了当前用户的session信息。
      8. 浏览器的请求抵达SP，SP的AccessCheck模块检查当前请求的cookie信息并返回目标资源。

  - 什么是 SAML 断言？

    - SAML 断言是告知服务提供商用户已经登录的消息。SAML 断言包含服务提供商确认用户身份需要的所有信息，包括断言的来源、签发时间，以及有效断言应满足的条件。

  - SAML 身份验证是否与用户授权相同？

    - **SAML 是一种用于用户身份验证而不是用户授权的技术，这是关键区别。**用户授权是身份和访问管理的另外一个领域。

      > 可以这样来思考身份验证和授权之间的区别：假设爱丽丝要参加音乐节。在入口处，她出示门票和另外一种身份证明，以证实她有权持有该门票。之后，她被允许参加音乐节。她通过了身份验证。
      >
      > 然而，仅仅因为爱丽丝位于音乐节场地内，并不意味着她可以随意走动、为所欲为。她可以观看音乐节表演，但不能登台表演，也不能到后台与演员互动，因为没有获得相应的授权。如果她购买了后台通行证，或者除了观众之外还担任表演者，那么她将获得更大的授权。

    - [访问管理](https://www.cloudflare.com/zh-cn/learning/access-management/what-is-identity-and-access-management)（IAM）技术处理用户授权。访问管理平台使用若干不同的授权标准（其中一种是 OAuth），而不是 SAML。

- **什么是 OAuth？| SAML 与 OAuth**

  - **OAuth 是用于授权用户的技术标准，它有助于使 SSO 成为可能**。它是一种协议，用于在不共享实际用户凭据（例如用户名和密码）的前提下将授权从一项服务传递到另一项服务。
  - OAuth 授权令牌如何工作？
    - 假设爱丽丝想要访问公司的云文件存储应用程序。她已经登录了公司的 SSO，但是当天还没访问过文件存储应用程序。**当她打开文件存储应用程序时，该应用程序没有直接允许她进入，而是向 SSO 请求爱丽丝的授权。**
    - **作为响应，SSO 将 OAuth 授权令牌发送到应用程序。**令牌包含有关爱丽丝在应用程序中应具备的特权的信息。令牌还会有一个时间限制：过了一定时间后令牌会失效，届时爱丽丝必须再次登录她的 SSO。
    - OAuth 令牌通常使用 [HTTPS](https://www.cloudflare.com/zh-cn/learning/ssl/what-is-https) 发送，这意味着它们已经过[加密](https://www.cloudflare.com/zh-cn/learning/ssl/what-is-encryption)。它们在 [OSI 模型](https://www.cloudflare.com/zh-cn/learning/ddos/glossary/open-systems-interconnection-model-osi)的[第 7 层](https://www.cloudflare.com/zh-cn/learning/ddos/what-is-layer-7)上发送。
  - OAuth 有什么用途？
    - 对企业而言，OAuth 的更常见用例是与[身份和访问管理（IAM）](https://www.cloudflare.com/zh-cn/learning/access-management/what-is-identity-and-access-management)系统相结合。用户可以通过 OAuth 获得使用应用程序的授权。例如，**员工可以使用用户名和密码登录其公司的 SSO 系统。此 SSO 系统使他们能够访问完成工作所需的所有应用程序，SSO 系统将 OAuth 授权令牌传递给这些应用程序来实现此目的。**

- **什么是基于角色的访问控制（RBAC）？**

  - 基于角色的访问控制（RBAC）是一种用于**控制用户在公司 IT 系统中能够执行的操作的方法。RBAC 通过为每个用户分配一个或多个“角色”并为每个角色赋予不同权限来实现此目的。**RBAC 可应用于单个软件应用程序，也可应用于多个应用程序。

    > **想像一间居住了几个人的房屋。每个居民都会得到一把可打开前门的钥匙**：他们收到的钥匙不会有完全不同的设计，而且这些钥匙都可以打开前门。如果他们需要进入这一物业的其他部分，如后院的仓库，他们可能会收到第二把钥匙。没有居民会获得仓库的唯一钥匙，或可同时打开仓库和前门的特殊钥匙。
    >
    > **在 RBAC 中，角色是静态的，就像上例中房屋的钥匙一样。**不论由谁拥有，它们都不会变化，而且需要更多访问权限的任何人都会被分配一个附加角色（或第二把钥匙），而不是获得自定义的权限。

- **什么是ACL 权限管理 ？**

  - ACL 是 Access Control List 的缩写，称为**访问控制列表，包含了对一个对象或一条记录可进行何种操作的权限定义。**

  - ACL 机制是**将每个操作授权给特定的 User 用户或者 Role 角色，只允许这些用户或角色对一个对象执行这些操作。**例如如下 ACL 定义：

    - ```json
      /*
      	所有人可读，但不能写（* 代表所有人）。
      	角色为 admin（包含子角色）的用户可读可写。
      	ID 为 58113fbda0bb9f0061ddc869 的用户可读可写。
      */
      {
        "*":{
          "read":true,
          "write":false
        },
        "role:admin":{
          "read":true,
          "write":true
        },
        "58113fbda0bb9f0061ddc869":{
          "read":true,
          "write":true
        }
      }
      ```










# SAML

- 当Stu单击Salesforce图标时，其公司的身份提供者生成了SAML断言（声明其身份的消息），他的浏览器导航到Salesforce，最后Salesforce验证了SAML断言并授予他访问权限。

- IdP配置

  - SP提供了SAML断言的规范（应包含什么以及应如何格式化），并在IdP上进行设置。
  - 必须在每个SP的IdP处设置以下值，并且通常有很多值。
    - **EntityID** -SP的全局唯一名称。格式各不相同，但越来越常见的是将此值格式化为URL。
    - **断言消费者服务（ACS）** -发送SAML断言的URL位置。
    - **ACS验证程序**-一种安全措施，采用正则表达式（regex）形式，可确保将SAML声明发送到正确的ACS。这仅在SP发起的登录（其中SAML请求包含ACS位置）期间起作用，因此此ACS验证程序将确保SAML请求提供的ACS位置是合法的。
    - **属性**-**属性**的数量和格式可以有很大的不同。通常至少有一个属性，nameID，通常是尝试登录的用户的用户名。
    - **RelayState-**不需要。SAML的深层链接。这告诉SP成功登录用户后将用户带到何处。
    - **SAML签名算法**-SHA-1或SHA-256。不太常见的SHA-384或SHA-512。该算法与下面提到的X.509证书结合使用。

- SP配置

  - 与上述部分相反，本部分涉及由IdP提供并在SP处设置的信息。
  - **X.509证书**-IdP提供的证书，**用于验证IdP在SAML声明的元数据中传递的公钥**。**它允许SP验证SAML断言*实际上*来自其信任的IdP。通常对SAML断言进行签名，但是也可以对SAML请求进行签名。通常，它是从IdP下载或复制的，并通过将其粘贴到SP中进行配置。**
  - **发行者URL** -IdP的唯一标识符。格式化为包含有关IdP信息的URL，以便SP可以验证其接收到的SAML断言是从正确的IdP发出的。
  - **SAML SSO端点/服务提供商登录URL-**一个IdP端点，当SP使用SAML请求将其重定向到此处时，该端点将启动身份验证。
  - **SAML SLO（单一注销）端点**-一个IdP端点，当由SP重定向到此位置时，通常会在用户单击“注销”后关闭该用户的IdP会话。

- SAML与Oauth和Web服务联合有何不同？

  - 企业最常使用的**SAML**，允许其用户访问其付费服务。SAML向服务提供者断言用户是谁；这是身份验证。
  - 消费者应用和服务最常用的**OAuth**，因此用户无需注册新的用户名和密码。“使用Google登录”和“使用Facebook登录”是现实世界中OAuth的示例。OAuth委托第三方访问一个人的Google或Facebook帐户。

- 流程

  - **方案 1：SP 启动的请求式 Web SSO（最终用户在 SP 上启动）**

  ![SP 启动的请求式 Web SSO（最终用户在 SP 上启动）](https://www.ibm.com/support/knowledgecenter/zh/SSEQTP_liberty/com.ibm.websphere.wlp.doc/images/saml_sp_sso.gif)

  1. 最终用户访问 SP。
  2. SP 将用户重定向至 IdP。
  3. 最终用户向 IdP 认证。
  4. IdP 将 SAML 响应和断言发送至 SP。
  5. SP 验证 SAML 响应并对用户请求授权。

  - **方案 2：IdP 启动的非请求式 Web SSO（最终用户在 IdP 上启动）**

  ![IdP 启动的非请求式 Web SSO（最终用户在 IdP 上启动）](https://www.ibm.com/support/knowledgecenter/zh/SSEQTP_liberty/com.ibm.websphere.wlp.doc/images/saml_idp_sso.gif)

  1. 用户代理访问 SAML IdP。
  2. IdP 认证用户并发出 SAML 断言。
  3. IdP 将用户重定向至 SP 并产生 SAMLResponse。
  4. SP 验证 SAML 响应并对用户请求授权。

  - **方案 3：OpenID Connect 提供者和 SAML 服务提供者**

  ![OpenID Connect 提供者和 SAML 服务提供者](https://www.ibm.com/support/knowledgecenter/zh/SSEQTP_liberty/com.ibm.websphere.wlp.doc/images/saml_oidc.gif)

  1. 最终用户访问 OpenID Connect 依赖方 (RP)。
  2. RP 将最终用户重定向至 OpenID Connect 提供者 (OP)。
  3. OP（又称为 SAML SP）将最终用户重定向至 SAML IdP。
  4. 最终用户向 SAML IdP 认证。
  5. IdP 将最终用户重定向至 SP 并产生 SAMLResponse。
  6. OP/SP 验证 SAML，并将授权代码发送至 RP。
  7. RP 针对 `id_token` 和 `access_token` 交换代码。
  8. RP 验证 `id_token` 并向最终用户授权。

- saml流程

  - ![img](https://pic4.zhimg.com/80/v2-1f366cde28c28a9c0d868fbbfb45d907_1440w.jpg)

- openid流程
  - ![img](https://pic1.zhimg.com/80/v2-99c0a91c71bc8e14df1f6afc151510d0_1440w.jpg)
  - 因为提供OpenID的IDP服务并不确定在哪，所以在打开IDP的登陆页面之前，用户需要在SP提供的一个符合OpenID标准的表单上，输入下自己账号所在的IDP地址信息，以便让SP找到相应的IDP服务。
  - 而且值得注意的是，**OpenID只提供了认证，而并没有授权。也就是说OpenID的IDP只确认了确实有这个账号，而这个账号能不能访问该网站的内容，OpenID的IDP并不关心。**
  
- oauth2流程
  
  - ![img](https://pic3.zhimg.com/80/v2-ab87935bbc902eff098de0f0aea640a2_1440w.jpg)
  
- `<saml:Assertion>`元素包含以下子元素：

  - ```xml
    <saml:Assertion
       xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
       xmlns:xs="http://www.w3.org/2001/XMLSchema"
       ID="_d71a3a8e9fcc45c9e9d248ef7049393fc8f04e5f75"
       Version="2.0"
       IssueInstant="2004-12-05T09:22:05Z">
       <saml:Issuer>https://idp.example.org/SAML2</saml:Issuer>
       <ds:Signature
         xmlns:ds="http://www.w3.org/2000/09/xmldsig#">...</ds:Signature>
       <saml:Subject>
         <saml:NameID
           Format="urn:oasis:names:tc:SAML:2.0:nameid-format:transient">
           3f7b3dcf-1674-4ecd-92c8-1544f346baf8
         </saml:NameID>
         <saml:SubjectConfirmation
           Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
           <saml:SubjectConfirmationData
             InResponseTo="aaf23196-1773-2113-474a-fe114412ab72"
             Recipient="https://sp.example.com/SAML2/SSO/POST"
             NotOnOrAfter="2004-12-05T09:27:05Z"/>
         </saml:SubjectConfirmation>
       </saml:Subject>
       <saml:Conditions
         NotBefore="2004-12-05T09:17:05Z"
         NotOnOrAfter="2004-12-05T09:27:05Z">
         <saml:AudienceRestriction>
           <saml:Audience>https://sp.example.com/SAML2</saml:Audience>
         </saml:AudienceRestriction>
       </saml:Conditions>
       <saml:AuthnStatement
         AuthnInstant="2004-12-05T09:22:00Z"
         SessionIndex="b07b804c-7c29-ea16-7300-4f3d6f7928ac">
         <saml:AuthnContext>
           <saml:AuthnContextClassRef>
             urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
           </saml:AuthnContextClassRef>
         </saml:AuthnContext>
       </saml:AuthnStatement>
       <saml:AttributeStatement>
         <saml:Attribute
           xmlns:x500="urn:oasis:names:tc:SAML:2.0:profiles:attribute:X500"
           x500:Encoding="LDAP"
           NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"
           Name="urn:oid:1.3.6.1.4.1.5923.1.1.1.1"
           FriendlyName="eduPersonAffiliation">
           <saml:AttributeValue
             xsi:type="xs:string">member</saml:AttributeValue>
           <saml:AttributeValue
             xsi:type="xs:string">staff</saml:AttributeValue>
         </saml:Attribute>
       </saml:AttributeStatement>
     </saml:Assertion>
    ```

  - 一个`<saml:Issuer>`元件，它包含身份提供者的唯一标识符

  - 一个`<ds:Signature>`元件，其中包含一个完整性保留的数字签名（未示出）在所述`<saml:Assertion>`元件

  - `<saml:Subject>`标识已验证主体的元素（但是在这种情况下，出于隐私原因，主体的身份隐藏在不透明的临时标识符后面）

  - 一个`<saml:Conditions>`元素，该元素给出断言被视为*有效的条件*

  - 一个`<saml:AuthnStatement>`元素，它描述身份提供者处的身份验证行为

  - 一个`<saml:AttributeStatement>`元素，该元素断言与已验证主体相关联的多值属性

- 换句话说，该断言编码以下信息：

  > 断言（“ b07b804c-7c29-ea16-7300-4f3d6f7928ac”）由身份提供者（https://idp.example.org/SAML2）在时间“ 2004-12-05T09：22：05Z”发布，涉及主题（3f7b3dcf -1674-4ecd-92c8-1544f346baf8）专门用于服务提供商（https://sp.example.com/SAML2）。



## SAML Metadata

- 为了安全地进行互操作，合作伙伴以任何形式和可能的方式共享元数据。无论如何，至少必须共享以下元数据：
  - 实体编号
  - 加密密钥
  - 协议端点（绑定和位置）
- 每个SAML系统实体都有一个entity ID，一个用于软件配置的全局唯一标识符，依赖方数据库和客户端Cookie。在线上，每条SAML协议消息都包含发行者的entity ID。
  - 出于身份验证的目的，SAML消息可由发布者进行数字签名。为了验证消息上的签名，消息接收者使用已知属于发行者的公共密钥。类似地，为了加密消息，发行者必须知道属于最终接收者的公共加密密钥。**在签名和加密这两种情况下，必须事先共享受信任的公用密钥。**
  - **重定向的URL中还要有该消息的签名以保证其不备篡改，验证签名的公钥和算法，都是IDP和SP提前协商好的。**
- metadata使得SAML能够正常运转，metadata有下列用途：
  - SP准备传输<samlp:AuthnRequest>给IdP时，SP如何知道IdP是真实的，不是evil IdP（企图钓鱼），为此，**SP在发布authentication request前获取metadata中的可信IdP列表；**
  - 在上一个场景中，SP如何知道向user发送authentication request？ **SP从metadata中查看IdP预先分配的endpoint location（用于authentication的，IdP除了authentication还有其他功能）**
  - IdP从SP处接收到<samlp:AuthnRequest>后，IdP如何知道SP是真实的，不是evil SP（企图获取用户信息）？**IdP在发布authentication response前获取metadata中的可信SP列表。**
  - 在前一个场景中，IdP如何加密SAML assertion，使得仅仅可信的SP能够解密assertion？**IdP使用metadata中的加密证书来加密assertion。**
  - 紧接着上一场景，IdP如何知道给用户发送authentication response的地址？**IdP从metadata中查看可信SP的预先定义的endpoint location。**
  - SP如何知道authentication response是来自于一个可信的IdP？ **SP使用metadata中IdP的公钥验证assertion的签名。**
  - SP如何知道从可信IdP收到的artifact的resolve地址？**SP从metadata查看IdP的artifact resolution service的预先分配的endpoint location。**
- **metadata保证IdP和SP之间的安全transaction。**





## SAML Request

- 此示例包含一个AuthnRequest。服务提供商在SP-SSO发起的流程中将AuthnRequest发送到身份提供商。

- 有两个示例：

  - 具有签名（HTTP重定向绑定）的AuthnRequest。

    - ```xml
      <samlp:AuthnRequest xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol" xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion" ID="ONELOGIN_809707f0030a5d00620c9d9df97f627afe9dcc24" Version="2.0" ProviderName="SP test" IssueInstant="2014-07-16T23:52:45Z" Destination="http://idp.example.com/SSOService.php" ProtocolBinding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST" AssertionConsumerServiceURL="http://sp.example.com/demo1/index.php?acs">
        <saml:Issuer>http://sp.example.com/demo1/metadata.php</saml:Issuer>
        <samlp:NameIDPolicy Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress" AllowCreate="true"/>
        <samlp:RequestedAuthnContext Comparison="exact">
          <saml:AuthnContextClassRef>urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport</saml:AuthnContextClassRef>
        </samlp:RequestedAuthnContext>
      </samlp:AuthnRequest>
      
      
      
      ---
      签名（HTTP重定向绑定
      bM441nuRIzAjKeMM8RhegMFjZ4L4xPBHhAfHYqgnYDQnSxC++Qn5IocWuzuBGz7JQmT9C57nxjxgbFIatiqUCQN17aYrLn/mWE09C5mJMYlcV68ibEkbR/JKUQ+2u/N+mSD4/C/QvFvuB6BcJaXaz0h7NwGhHROUte6MoGJKMPE=
      
      SigAlg=http://www.w3.org/2000/09/xmldsig#rsa-sha1 , RelayState=http://sp.example.com/relaystate
      
      
      ```

  - 内嵌签名的AuthNRequest（HTTP-POST绑定）。

    - ```xml
      <samlp:AuthnRequest xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol" xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion" ID="pfx41d8ef22-e612-8c50-9960-1b16f15741b3" Version="2.0" ProviderName="SP test" IssueInstant="2014-07-16T23:52:45Z" Destination="http://idp.example.com/SSOService.php" ProtocolBinding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST" AssertionConsumerServiceURL="http://sp.example.com/demo1/index.php?acs">
        <saml:Issuer>http://sp.example.com/demo1/metadata.php</saml:Issuer>
        <ds:Signature xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
          <ds:SignedInfo>
            <ds:CanonicalizationMethod Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
            <ds:SignatureMethod Algorithm="http://www.w3.org/2000/09/xmldsig#rsa-sha1"/>
            <ds:Reference URI="#pfx41d8ef22-e612-8c50-9960-1b16f15741b3">
              <ds:Transforms>
                <ds:Transform Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature"/>
                <ds:Transform Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
              </ds:Transforms>
              <ds:DigestMethod Algorithm="http://www.w3.org/2000/09/xmldsig#sha1"/>
              <ds:DigestValue>yJN6cXUwQxTmMEsPesBP2NkqYFI=</ds:DigestValue>
            </ds:Reference>
          </ds:SignedInfo>
          <ds:SignatureValue>g5eM9yPnKsmmE/Kh2qS7nfK8HoF6yHrAdNQxh70kh8pRI4KaNbYNOL9sF8F57Yd+jO6iNga8nnbwhbATKGXIZOJJSugXGAMRyZsj/rqngwTJk5KmujbqouR1SLFsbo7Iuwze933EgefBbAE4JRI7V2aD9YgmB3socPqAi2Qf97E=</ds:SignatureValue>
          <ds:KeyInfo>
            <ds:X509Data>
              <ds:X509Certificate>MIICajCCAdOgAwIBAgIBADANBgkqhkiG9w0BAQQFADBSMQswCQYDVQQGEwJ1czETMBEGA1UECAwKQ2FsaWZvcm5pYTEVMBMGA1UECgwMT25lbG9naW4gSW5jMRcwFQYDVQQDDA5zcC5leGFtcGxlLmNvbTAeFw0xNDA3MTcwMDI5MjdaFw0xNTA3MTcwMDI5MjdaMFIxCzAJBgNVBAYTAnVzMRMwEQYDVQQIDApDYWxpZm9ybmlhMRUwEwYDVQQKDAxPbmVsb2dpbiBJbmMxFzAVBgNVBAMMDnNwLmV4YW1wbGUuY29tMIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC7vU/6R/OBA6BKsZH4L2bIQ2cqBO7/aMfPjUPJPSn59d/f0aRqSC58YYrPuQODydUABiCknOn9yV0fEYm4bNvfjroTEd8bDlqo5oAXAUAI8XHPppJNz7pxbhZW0u35q45PJzGM9nCv9bglDQYJLby1ZUdHsSiDIpMbGgf/ZrxqawIDAQABo1AwTjAdBgNVHQ4EFgQU3s2NEpYx7wH6bq7xJFKa46jBDf4wHwYDVR0jBBgwFoAU3s2NEpYx7wH6bq7xJFKa46jBDf4wDAYDVR0TBAUwAwEB/zANBgkqhkiG9w0BAQQFAAOBgQCPsNO2FG+zmk5miXEswAs30E14rBJpe/64FBpM1rPzOleexvMgZlr0/smF3P5TWb7H8Fy5kEiByxMjaQmml/nQx6qgVVzdhaTANpIE1ywEzVJlhdvw4hmRuEKYqTaFMLez0sRL79LUeDxPWw7Mj9FkpRYT+kAGiFomHop1nErV6Q==</ds:X509Certificate>
            </ds:X509Data>
          </ds:KeyInfo>
        </ds:Signature>
        <samlp:NameIDPolicy Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress" AllowCreate="true"/>
        <samlp:RequestedAuthnContext Comparison="exact">
          <saml:AuthnContextClassRef>urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport</saml:AuthnContextClassRef>
        </samlp:RequestedAuthnContext>
      </samlp:AuthnRequest>
      ```

      





## SAML Response

- ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <saml2p:Response
      xmlns:saml2p="urn:oasis:names:tc:SAML:2.0:protocol" Destination="https://signin.aliyun.com/saml/SSO" ID="_256e1b401c2ac9d2b3ac259d813cd8f9" InResponseTo="a5a808ahjaii493f4436359j4d478f2" IssueInstant="2019-10-05T02:55:49.397Z" Version="2.0">
      <saml2:Issuer
          xmlns:saml2="urn:oasis:names:tc:SAML:2.0:assertion">https://accounts.google.com/o/saml2?idpid=C01k0l1si
      </saml2:Issuer>
      <saml2p:Status>
          <saml2p:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success"/>
      </saml2p:Status>
      <saml2:Assertion
          xmlns:saml2="urn:oasis:names:tc:SAML:2.0:assertion" ID="_a06a19baf0b991c23b41e4c57b808d34" IssueInstant="2019-10-05T02:55:49.397Z" Version="2.0">
          <saml2:Issuer>https://accounts.google.com/o/saml2?idpid=C01k0l1si</saml2:Issuer>
          <ds:Signature
              xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
              <ds:SignedInfo>
                  <ds:CanonicalizationMethod Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
                  <ds:SignatureMethod Algorithm="http://www.w3.org/2001/04/xmldsig-more#rsa-sha256"/>
                  <ds:Reference URI="#_a06a19baf0b991c23b41e4c57b808d34">
                      <ds:Transforms>
                          <ds:Transform Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature"/>
                          <ds:Transform Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
                      </ds:Transforms>
                      <ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256"/>
                      <ds:DigestValue>n6vzikmbE+G8rjj9K7GqNFydq3KXaP6ttZbjU1HUcrA=</ds:DigestValue>
                  </ds:Reference>
              </ds:SignedInfo>
              <ds:SignatureValue>R0z281cp...W8hyg==</ds:SignatureValue>
              <ds:KeyInfo>
                  <ds:X509Data>
                      <ds:X509SubjectName>ST=California,C=US,OU=Google For Work,CN=Google,L=Mountain View,O=Google Inc.</ds:X509SubjectName>
                      <ds:X509Certificate>MIIDdDCCAlygAw...4VL4OJkRnq72oPN+Z</ds:X509Certificate>
                  </ds:X509Data>
              </ds:KeyInfo>
          </ds:Signature>
          <saml2:Subject>
              <saml2:NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified">admin@testForBlog.onaliyun.com</saml2:NameID>
              <saml2:SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
                  <saml2:SubjectConfirmationData InResponseTo="a5a808ahjaii493f4436359j4d478f2" NotOnOrAfter="2019-10-05T03:00:49.397Z" Recipient="https://signin.aliyun.com/saml/SSO"/>
              </saml2:SubjectConfirmation>
          </saml2:Subject>
          <saml2:Conditions NotBefore="2019-10-05T02:50:49.397Z" NotOnOrAfter="2019-10-05T03:00:49.397Z">
              <saml2:AudienceRestriction>
                  <saml2:Audience>https://signin.aliyun.com/1797870240813407/saml/SSO</saml2:Audience>
              </saml2:AudienceRestriction>
          </saml2:Conditions>
          <saml2:AuthnStatement AuthnInstant="2019-10-05T02:29:07.000Z" SessionIndex="_a06a19baf0b991c23b41e4c57b808d34">
              <saml2:AuthnContext>
                  <saml2:AuthnContextClassRef>urn:oasis:names:tc:SAML:2.0:ac:classes:unspecified</saml2:AuthnContextClassRef>
              </saml2:AuthnContext>
          </saml2:AuthnStatement>
      </saml2:Assertion>
  </saml2p:Response>
  ```

- SAMLResponse的内容按照SAML规范解码如下：

  > Destination：代表当前SAMLResponse要发送到的地址
  > ID：代表当前SAMLResponse的唯一标识。
  > InResponseTo：代表当前SAMLResponse用于回应拿给SAMLRequest，对应的是SAMLRequest的ID。
  > IssueInstant：代表当前断言的发布时间
  > Issuer：代表当前断言的发布者，实际上就是IDP的entityID
  > Status：代表认证状态，Success代表认证通过

- **由于SP中声明了WantAssertionsSigned=true，说明SP需要接收被签名的SAMLResponse。**因此SAMLResponse中有签名相关的内容，使用ds:Signature表示。

  > CanonicalizationMethod：表示签名过程中使用的规范化方法
  > SignatureMethod：代表使用的签名方法。这里为RSA-SHA256
  > DigestMethod：代表签名过程中使用的摘要方法
  > DigestValue：代表使用摘要方法获得的值
  > SignatureValue：代表最终获得的签名内容
  > X509Data：代表签名时使用的私钥对应的公钥证书信息，该值和IDP中的证书信息是保持一致的

- 签名部分保证了SAMLResponse消息的机密性以及消息来源的合法性，Subject则用来声明被认证者的身份信息。

  > NameID：代表用户的身份，可以是邮箱、姓名等
  > SubjectConfirmation：代表当前身份的生效条件。用InResponseTo表明该身份是用于响应哪个SAMLRequest的，NotOnOrAfter代表生效的时间限制。
  > Recipient：通常是当前SAMLResponse的消费地址，对应SP元文件中的断言消费地址AssertionConsumerService。

- Conditions部分用来描述当前SAMLResponse生效的限制条件。

  > NotBefore：代表生效时间的起点
  > NotOnOrAfter：代表生效时间的终点
  > Audience：代表当前SAMLResponse的受众，通常来说，Audience等于目标SP的entityID

- AuthnStatement用于描述认证发生时的情况

  > AuthnInstant：代表认证发生的时间
  > AuthnContext：代表认证时的上下文环境，比如当前认证是使用Password认证或者联合认证等等。在某些场景下，SP可能会限制IDP使用的认证手段，此时该字段就会派上用场。

- **SAML Response (IdP -> SP)**

  - 本示例包含多个SAML响应。身份提供商将SAML响应发送到服务提供商，并且如果用户在身份验证过程中成功，则该SAML响应包含具有NameID /用户属性的声明。

  - 有8个示例

    > 具有未签名断言的未签名SAML响应
    > 具有签名断言的未签名SAML响应
    > 具有未签名断言的已签名SAML响应
    > 已签名的SAML响应和已声明的断言
    > 带有加密断言的未签名SAML响应
    > 带有加密的签名断言的未签名的SAML响应
    > 带加密断言的签名SAML响应
    > 带有加密的签名断言的签名的SAML响应

- idp发送samlresponse与sp验证的流程

  > 综合来说，sp和idp会互换metatdata，其中包含了公钥证书。
  >
  >   在签名和加密这两种情况下，必须事先共享受信任的公钥。
  >
  >   sp和idp互相持有对方的公钥
  >
  > metadata保证IdP和SP之间的安全transaction
  >
  > 
  >
  > 当idp发送response的时候：
  >
  >   idp用idp的私钥签名， sp用idp的公钥验证
  >
  >   idp用约定的密钥加密， sp用约定的密钥解密
  >
  > 
  >
  > SignatureMethod：代表使用的签名方法。比如RSA-SHA256
  >
  > DigestMethod：代表签名过程中使用的摘要方法
  >
  > X509Data：代表签名时使用的私钥对应的公钥信息，该值和IDP中的证书信息是保持一致的
  >
  > 
  >
  > encrypt : xml+x.509.cert+encrytpt key(RSA)+encrypt data(AES加密)+AES密钥
  >
  > sign : xml+x.509.cert+idp私钥
  >
  > 
  >
  > decrypt : xml+AES密钥
  >
  > validate : xml+x.509.cert（idp公钥）+sp私钥（可选）+其他
  >
  > 
  >
  > \---
  >
  > 
  >
  > idp——saml response：
  >
  > （（（data密钥加密+密钥）+摘要）私钥签名）
  >
  > 
  >
  > sp——validate and decrypt
  >
  > 公钥验证：（data密钥加密+密钥）
  >
  > 密钥解密： data





## SAML Example

- 每个服务提供商必须建立一个自签名的X.509密钥对。您可以使用以下内容生成自己的内容：

  - ```
    openssl req -x509 -newkey rsa:2048 -keyout myservice.key -out myservice.cert -days 365 -nodes -subj "/CN=myservice.example.com"
    ```

- 测试程序

  - ```go
    package main
    
    import (
    	"crypto/rsa"
    	"crypto/tls"
    	"crypto/x509"
    	"fmt"
    	"net/http"
    	"net/url"
    
    	"github.com/crewjam/saml/samlsp"
    )
    
    func hello(w http.ResponseWriter, r *http.Request) {
    	fmt.Fprintf(w, "Hello, %s!", samlsp.AttributeFromContext(r.Context(), "cn"))
    }
    
    func main() {
    	keyPair, err := tls.LoadX509KeyPair("myservice.cert", "myservice.key")
    	if err != nil {
    		panic(err) // TODO handle error
    	}
    	keyPair.Leaf, err = x509.ParseCertificate(keyPair.Certificate[0])
    	if err != nil {
    		panic(err) // TODO handle error
    	}
    
    	idpMetadataURL, err := url.Parse("https://samltest.id/saml/idp")
    	if err != nil {
    		panic(err) // TODO handle error
    	}
    
    	rootURL, err := url.Parse("http://localhost:8000")
    	if err != nil {
    		panic(err) // TODO handle error
    	}
    
    	samlSP, _ := samlsp.New(samlsp.Options{
    		URL:            *rootURL,
    		Key:            keyPair.PrivateKey.(*rsa.PrivateKey),
    		Certificate:    keyPair.Leaf,
    		IDPMetadataURL: idpMetadataURL,
    	})
    	app := http.HandlerFunc(hello)
    	http.Handle("/hello", samlSP.RequireAccount(app))
    	http.Handle("/saml/", samlSP)
    	http.ListenAndServe(":8000", nil)
    }
    
    ```

- 身份提供者中注册我们的服务提供者，以建立从服务提供者到IDP的信任。

  - 导航到https://samltest.id/upload.php并上传您获取的文件。

- [crewjam](https://github.com/crewjam)/**[saml](https://github.com/crewjam/saml)**

  - 该程序包可以生成签名的SAML断言，并且可以验证签名和加密的SAML断言。它不支持签名或加密的请求。





# OIDC Example

- ```go
  provider, err := oidc.NewProvider(ctx, "https://accounts.google.com")
  if err != nil {
      // handle error
  }
  
  // Configure an OpenID Connect aware OAuth2 client.
  oauth2Config := oauth2.Config{
      ClientID:     clientID,
      ClientSecret: clientSecret,
      RedirectURL:  redirectURL,
  
      // Discovery returns the OAuth2 endpoints.
      Endpoint: provider.Endpoint(),
  
      // "openid" is a required scope for OpenID Connect flows.
      Scopes: []string{oidc.ScopeOpenID, "profile", "email"},
  }
  
  ```

- 在响应上，提供程序可用于验证ID令牌。

  - ```go
    var verifier = provider.Verifier(&oidc.Config{ClientID: clientID})
    
    func handleOAuth2Callback(w http.ResponseWriter, r *http.Request) {
        // Verify state and errors.
    
        oauth2Token, err := oauth2Config.Exchange(ctx, r.URL.Query().Get("code"))
        if err != nil {
            // handle error
        }
    
        // Extract the ID Token from OAuth2 token.
        rawIDToken, ok := oauth2Token.Extra("id_token").(string)
        if !ok {
            // handle missing token
        }
    
        // Parse and verify ID Token payload.
        idToken, err := verifier.Verify(ctx, rawIDToken)
        if err != nil {
            // handle error
        }
    
        // Extract custom claims
        var claims struct {
            Email    string `json:"email"`
            Verified bool   `json:"email_verified"`
        }
        if err := idToken.Claims(&claims); err != nil {
            // handle error
        }
    }
    ```

- These are example uses of the oidc package. Each requires a Google account and the client ID and secret of a registered OAuth2 application. To create one:

  1. Visit your [Google Developer Console](https://console.developers.google.com/apis/dashboard).
  2. Click "Credentials" on the left column.
  3. Click the "Create credentials" button followed by "OAuth client ID".
  4. Select "Web application" and add "http://127.0.0.1:5556/auth/google/callback" as an authorized redirect URI.
  5. Click create and add the printed client ID and secret to your environment using the following variables:

  ```
  GOOGLE_OAUTH2_CLIENT_ID
  GOOGLE_OAUTH2_CLIENT_SECRET
  ```

  Finally run the examples using the Go tool and navigate to [http://127.0.0.1:5556](http://127.0.0.1:5556/).

  ```
  go run ./example/idtoken/app.go
  ```

> ## Redirect to OpenID Connect Server
>
> request
>
> [https://samples.auth0.com/authorize?](https://openidconnect.net/#)
>
> client_id=[kbyuFDidLLm280LIwVFiazOqjO3ty8KH](https://openidconnect.net/#)
> &redirect_uri= https://openidconnect.net/callback
> &scope=[openid profile email phone address](https://openidconnect.net/#)
> &response_type=code
> &state=23c18ea08147b5466cb45138a003104431ec2368
>
> 
>
> ## Exchange Code from Token
>
> Your Code is
>
> a_WAqWqk13s63Hko
>
> Now, we need to turn that access code into an access token, by having our server make a request to your token endpoint
>
> Request
>
> POST https://samples.auth0.com/oauth/token
> grant_type=authorization_code
> &client_id=[kbyuFDidLLm280LIwVFiazOqjO3ty8KH](https://openidconnect.net/#)
> &client_secret=[60Op4HFM0I8ajz0WdiStAbziZ-VFQttXuxixHHs2R7r7-CW8GR79l-mmLqMhc-Sa](https://openidconnect.net/#)
> &redirect_uri=https://openidconnect.net/callback
> &code=a_WAqWqk13s63Hko
>
> 
>
> ## Verify User Token
>
> Now, we need to verify that the ID Token sent was from the correct place by validating the JWT's signature
>
> Your “id_token” is
>
> eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjbGllbnRJRCI6ImtieXVGRGlkTExtMjgwTEl3VkZpYXpPcWpPM3R5OEtIIiwiY3JlYXRlZF9hdCI6IjIwMjAtMTItMTFUMTY6Mzk6MDEuMDY3WiIsImVtYWlsIjoiemgxY2hldW5nbHFAZ21haWwuY29tIiwiZW1haWxfdmVyaWZpZWQiOnRydWUsImZhbWlseV9uYW1lIjoiQ2hldW5nIiwiZ2l2ZW5fbmFtZSI6IlppaGVuZyIsImlkZW50aXRpZXMiOlt7InByb3ZpZGVyIjoiZ29vZ2xlLW9hdXRoMiIsInVzZXJfaWQiOiIxMDU0MjAxODQxODU1NDMwNTkwNjAiLCJjb25uZWN0aW9uIjoiZ29vZ2xlLW9hdXRoMiIsImlzU29jaWFsIjp0cnVlfV0sImxvY2FsZSI6InpoLUNOIiwibmFtZSI6IlppaGVuZyBDaGV1bmciLCJuaWNrbmFtZSI6InpoMWNoZXVuZ2xxIiwicGljdHVyZSI6Imh0dHBzOi8vbGgzLmdvb2dsZXVzZXJjb250ZW50LmNvbS9hLS9BT2gxNEdqelQ5a2FGMWl0bUNWZzNUek53S0N4ZkcxdnhrVjdpWi1VZ2dQMD1zOTYtYyIsInVwZGF0ZWRfYXQiOiIyMDIwLTEyLTExVDE2OjM5OjM4LjA3MloiLCJ1c2VyX2lkIjoiZ29vZ2xlLW9hdXRoMnwxMDU0MjAxODQxODU1NDMwNTkwNjAiLCJwZXJzaXN0ZW50Ijp7fSwidXNlcl9tZXRhZGF0YSI6e30sImFwcF9tZXRhZGF0YSI6e30sImlzcyI6Imh0dHBzOi8vc2FtcGxlcy5hdXRoMC5jb20vIiwic3ViIjoiZ29vZ2xlLW9hdXRoMnwxMDU0MjAxODQxODU1NDMwNTkwNjAiLCJhdWQiOiJrYnl1RkRpZExMbTI4MExJd1ZGaWF6T3FqTzN0eThLSCIsImlhdCI6MTYwNzcwNDgwMCwiZXhwIjoxNjA3NzQwODAwfQ.qw_yTIgDyY5lki-Yh8460CEhkykTDxu0_4yquXNmjcU
>
> This token is cryptographically signed with the **HS256** algorithim. We'll use the client secret to validate it.
>
> 
>
> ## The token is valid!
>
> Decoded Token Payload

















































