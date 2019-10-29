# kubernetes安全



Kubernetes 的安全是一个相当广泛的主题，涉及很多高度相关的内容。和探讨大部分安全性相关的问题一样，首先需要考虑威胁模型——谁可能攻击你的系统，以及他们如何做到这一点。这可以帮你确定安全工作的优先级。对于大多数 Kubernetes 应用有三类主要的攻击者：

1. 外部攻击者：当你在内部或云上部署应用时，你可能面临来自集群外的攻击。这类攻击者没有系统权限，所以会专注于公开的服务，会尝试获取访问权限并提升权限。
2. 泄露的容器：Kubernetes 集群通常运行着各种工作负载。攻击者也可能利用集群中运行的容器的漏洞进行攻击，在这种时候，要最大程度降低漏洞攻击影响到整个集群的风险。攻击者可以访问单个容器的资源，因此限制容器权限至关重要。
3. 恶意用户：Kubernetes 是一个多用户系统。攻击者可能拥有某用户的账户，并企图获得更多权限，这种情况比较复杂，要具体分析，需要限制不同用户的访问权限。

围绕云原生基础概念构建的模型可以帮你建立对 Kubernetes 安全的总体认识。下图将安全划分为四个层级，被称为云原生安全的4C模型。管理员将在不同的层次上应对三类攻击者。

![](https://oscimg.oschina.net/oscnet/901d33c3f32481901464d0b7182197931c6.jpg)

> 4C 指的是云（Cloud）、集群（Cluster），容器（Containers）和代码（Code）

正如你在图中所看见的，4C 模型中每部分的安全性都是相互包含的。只依靠增强代码层次安全来预防云、集群和容器中安全漏洞是几乎不可能的。适当提高其他层的安全能力，就已经为你的代码提供强大的基础安全保障。下面将详细介绍这四部分内容。

## Cloud <a id="h1_1"></a>

大多数情况下，云为 Kubernetes 集群提供可信的计算资源。如果云的基础设置是不可靠的（或以易受到攻击的方式配置），那就没有办法保证在这个基础上构建的 Kubernetes 集群的安全性。每一个云服务提供商都向他们的客户提供大量如何在其环境安全运行负载的建议。下面提供常用云服务厂商的安全文档链接，并且提供了构建安全 Kubernetes 集群的建议。

### 云服务提供商安全文档列表 <a id="h2_2"></a>

| 云服务厂商 | 链接 |
| :--- | :--- |
| 阿里云 | [https://www.alibabacloud.com/trust-center](https://www.alibabacloud.com/trust-center) |
| AWS | [https://aws.amazon.com/security/](https://aws.amazon.com/security/) |
| Google Cloud Platform | [https://cloud.google.com/security/](https://cloud.google.com/security/) |
| Microsoft Azure | [https://docs.microsoft.com/en-us/azure/security/azure-security](https://docs.microsoft.com/en-us/azure/security/azure-security) |

如果你运行在自己的硬件上或者其他云服务提供商，请查阅文档获取最佳安全实践。

### 通用的安全建议 <a id="h2_3"></a>

* 理想情况下，不开放 Kubernetes Master 节点公网访问，并且限制能够访问集群 Master 节点的 IP 地址。
* 通过网络安全组等配置工作节点只接受 Master 节点上指定端口的连接，并且接受 Kubernetes 中服务类型为 NodePort 和 LoadBalancer 的连接。如果可能，这些节点也不应该暴露在公网中。
* Kubernetes 访问云服务提供商的 API，每一个云服务提供商都赋予 Kubernetes 的 Master 和 Node 不同的权限。该访问权限遵循其管理资源所需资源的最小权限原则。
* 访问 etcd， 只有在 master 节点中通过 TLS 可以访问 etcd。
* 在所有可能情况下，对停用状态的的驱动设备加密。ectd 拥有整个集群的状态（包括密钥信息），因此对其磁盘要在其停用的时候加密。

## Cluster <a id="h1_4"></a>

Kubernetes 是一个复杂的系统，下图展示了一个集群不同的组成部分，为了保证集群整体的安全，需要仔细保护每一个组件。

![](https://oscimg.oschina.net/oscnet/e5a7906b9797ea4d8756f27c47050b97508.jpg)

将需要注意保护的集群组件划分成两个部分：

1. 保护组成集群的可配置组件
2. 保护运行在集群中的应用

### 集群的组件 <a id="h2_5"></a>

1. 控制对 Kubernetes API 的访问

   使用 TLS 进行安全通讯。Kubernetes 期望默认情况下使用 TLS 对集群中所有 API 通信进行加密，大多数安装方式都会创建必要的证书并分发给集群组件。

   API 认证。在安装集群时，为API服务器选择与通用访问模式匹配的身份验证机制。例如，小型单用户集群可能希望使用简单的证书或静态Bearer令牌方法。较大的群集可能希望集成现有的OIDC或LDAP服务器，以将用户细分为多个组。更多有信息请参考[认证](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)。

   API 授权。一旦通过身份验证，每个API调用也都有望通过授权检查。Kubernetes 附带了一个集成的基于角色的访问控制（RBAC）组件，该组件将传入的用户或组与捆绑到角色中的一组权限进行匹配。这些权限将动词（get，create，delete）与资源（pods，service，nodes）结合在一起，并且可以是命名空间或集群作用域。提供了一组现成的角色，这些角色根据客户端可能要执行的操作提供合理的默认责任分离。更多有关信息请参考[授权](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)

2. 控制对 Kubelet 的访问。

   Kubelet 在端口10250和10255上提供小型 REST API。端口10250是读/写端口，而10255是具有API端点子集的只读端口。这些端点授予对节点和容器的强大控制权。默认情况下，Kubelets允许未经身份验证对此API进行访问。生产集群应启用 Kubelet 身份验证和授权，可以通过设置 `--read-only-port=0` 来禁用只读端口，但是10250 端口用于系统指标收集和其他重要功能，所以开发者必须仔细控制对此端口的访问。更多有关信息请参考[Kubelet身份验证/授权](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-authentication-authorization/)

3. 保护集群组件不受损坏
   * 限制对 etcd 的访问。对API的etcd后端的写入访问权等同于在整个集群上获得root权限，而读取访问权也可以获取集群信息。管理员应始终使用从API服务器到etcd服务器的强凭据，例如通过TLS客户端证书的相互身份验证，通常建议 etcd服务器隔离在仅API服务器可以访问的防火墙后面。
   * 限制对Alpha或Beta功能的访问。Alpha和Beta Kubernetes功能正在积极开发中，并且可能会存在限制或错误，从而导致安全漏洞。始终评估Alpha或Beta功能可能提供的价值，以防对您的安全状况造成潜在风险。如有疑问，请禁用不使用的功能。
   * 启用审核日志记录。审核记录器是一项Beta版功能，可记录API的行为，在出现问题后提供分析依据。
   * 经常轮换基础设施凭证。密钥和凭证生存周期越短，就越难被攻击者利用。
   * 在启用第三方集成之前，请先进行审查。
   * 停用时加密 ectd。etcd数据库将包含可通过Kubernetes API访问的所有信息，可能使攻击者深入了解群集的状态。
   * 接收有关安全更新和报告漏洞的警报。

### 集群中的应用 <a id="h2_6"></a>

根据应用程序受到的攻击，关注安全的特定方面。例如，运行中的服务 A 对于其他应用非常重要，而服务 B 容易受到资源耗尽攻击，如果不设置服务 B 的资源限制，就会因为服务 B 耗尽资源导致服务 A 不可用。所以需要注意下面几个常见安全措施：

* 定义资源限制

  默认情况下，Kubernetes 集群中对所有资源没有对 CPU 、内存和磁盘的使用限制。可以通过创建资源配额策略，并附加到 namespace 中来限制资源的使用。

  下面的例子将限制命名空间中Pod 的数量为4个，CPU使用在1和2之间，内存使用在1GB 和 2GB 之间。

  ```text
  # compute-resources.yaml
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: compute-resources
  spec:
    hard:
      pods: "4"
      requests.cpu: "1"
      requests.memory: 1Gi
      limits.cpu: "2"
      limits.memory: 2Gi
      requests.nvidia.com/gpu: 4
  ```

  分配资源配额到命名空间：

  ```text
  kubectl create -f ./compute-resources.yaml --namespace=myspace
  ```

  查看 namespace 资源使用情况：

  ```text
  [root@localhost ~]# kubectl describe quota compute-resources --namespace=myspace
  Name:                    compute-resources
  Namespace:               myspace
  Resource                 Used  Hard
  --------                 ----  ----
  limits.cpu               0     2
  limits.memory            0     2Gi
  pods                     0     4
  requests.cpu             0     1
  requests.memory          0     1Gi
  requests.nvidia.com/gpu  0     4
  ```

  也可以为 Pod 添加资源限制，设置内存请求为 256MiB，内存限制为512MiB

  ```text
  apiVersion: v1
  kind: Pod
  metadata:
    name: default-mem-demo
  spec:
    containers:
    - name: default-mem-demo-ctr
      image: nginx
      resources:
        limits:
          memory: 512Mi
        requests:
          memory: 256Mi
  ```

* Pod 安全策略

  作为 Pod 的组成部分，容器通常配置有非常实用的安全默认值，这些默认值适用于大多数情况。但是，有时 Pod 可能需要其他权限才能执行其预期任务，在这种情况下，我们需要增强 Pod 的默认权限。安全上下文定义了 Pod 或容器的特权和访问控制，安全上下文设置有：

  1. 委托访问：基于用户ID 和用户组ID 权限去访问资源对象（如文件）。
  2. SELinux：为对象分配安全标签
  3. 以特权或非特权用户运行
  4. Linux 功能：为进程提供一些特权，而不去赋予它root用户的所有特权
  5. AppArmor：使用程序配置文件来限制单个程序的功能。
  6. Seccomp：过滤进程的系统调用
  7. 允许提升权限（AllowPrivilegeEscalation）：子进程能否获得父进程更多的权限。这个布尔值直接控制是否在容器进程上设置了 no\_new\_privs 标志。在以下情况下 AllowPrivilegeEscalation 始终为 true——一是，以特权OR运行；二是，具有 `CAP_SYS_ADMIN`。

  在 `securityContext` 字段下配置安全策略，在 Pod 中指定的安全策略将被应用于所有容器，容器中指定的安全策略应用于单个容器，当它们重复时，容器级会覆盖 Pod 级的的安全策略。

  ```text
  apiVersion: v1
  kind: Pod
  metadata:
    name: security-context-demo
  spec:
    securityContext:
      runAsUser: 1000
      runAsGroup: 3000
      fsGroup: 2000
    volumes:
    - name: sec-ctx-vol
      emptyDir: {}
    containers:
    - name: sec-ctx-demo
      image: busybox
      command: [ "sh", "-c", "sleep 1h" ]
      volumeMounts:
      - name: sec-ctx-vol
        mountPath: /data/demo
      securityContext:
        runAsUser: 2000
        allowPrivilegeEscalation: false
  ```

  更多有关信息请参考[Pod 安全策略](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)

* 网络规则

  Kubernetes允许来自集群中任何容器的所有网络流量发送到集群中的任何其他容器并由其接收。当尝试隔离工作负载时，这种开放式方法没有帮助，因此需要应用网络策略来帮助开发者实现所需的隔离。Kubernetes NetworkPolicy API使开发者能够将入口和出口规则应用于选定的Pod，用于第3层和第4层流量，并依赖于实现容器网络接口（CNI）的兼容网络插件的部署。

  网络策略是 namespace 作用域，并根据匹配标签（例如，tier：backend）的选择应用于Pod。当NetworkPolicy对象的Pod选择器与Pod匹配时，根据策略中定义的入口和出口规则来管理进出Pod的流量。 所有来自或发往该Pod的流量都会被拒绝，除非有允许它的规则。

  要在Kubernetes集群中正确隔离网络和传输层的应用程序，网络策略应以“拒绝所有”的默认前提开始。然后，应将每个应用程序组件及其所需源和目标的规则逐个列入白名单，并进行测试以确保流量模式按预期工作。

* 密钥管理

  Kubernetes 使用 Secrets 保护应用程序需要访问敏感信息——密码、X.509证书、SSH密钥或OAuth令牌等。它通过卷装入，对它的访问严格限于那些需要使用它的实体（用户和Pod），且当存储在磁盘上时（静止状态）它是不可访问或只读的。

需要考虑的全部安全问题如下：

| 需要关注的安全领域 | 建议 |
| :--- | :--- |
| RBAC 授权 | [https://kubernetes.io/docs/reference/access-authn-authz/rbac/](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) |
| 认证方式 | [https://kubernetes.io/docs/reference/access-authn-authz/controlling-access/](https://kubernetes.io/docs/reference/access-authn-authz/controlling-access/) |
| 密钥管理 | [https://kubernetes.io/docs/concepts/configuration/secret/](https://kubernetes.io/docs/concepts/configuration/secret/) [https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) |
| Pod 安全规则 | [https://kubernetes.io/docs/concepts/policy/pod-security-policy/](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) |
| 服务质量 | [https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/) |
| 网络规则 | [https://kubernetes.io/docs/concepts/services-networking/network-policies/](https://kubernetes.io/docs/concepts/services-networking/network-policies/) |
| ingress 的 TLS | [https://kubernetes.io/docs/concepts/services-networking/ingress/\#tls](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls) |

## Contanier <a id="h1_7"></a>

为了在 Kunernetes 中运行软件，必须将它打包成容器。需要考虑下面的注意事项，来保证容器符合 Kubernetes 安全配置。

* 最小化镜像

  理想情况下，使用应用程序二进制文件和二进制文件所依赖的的任何相关项来创建镜像。事实上，没有什么阻止开发者将 `scratch` 作为基础镜像，并将静态链接的二进制文件复制到镜像中。它没有其他依赖，镜像中包含单个文件，以及一些描述容器如何运行的元数据。最小化镜像加快镜像的分发速度，并且显著减少容器内部的受攻击面。

* 基础镜像

  但是这并不总是可行的，因此需要慎重选择基础镜像。最佳的方法是使用操作系统供应商支持的镜像，或者是 docker hub 上官方支持的镜像。不要盲目使用之前未经过审查的不受信任来源的镜像，尤其是在生产环境中。

* 镜像扫描

  镜像扫描对于保障容器应用的安全至关重要，确保您的镜像定期通过信誉良好的工具进行扫描，可以使用 CoreOS 的 Clair 之类的工具扫描容器中的已知漏洞，它是容器镜像的静态漏洞分析器。

* 镜像的签名和执行

  使用 CNCF 项目的的 TUF 和 Notary 对容器进行签名，在执行前验证签名，确保镜像的正确没有被篡改。

* 禁止特权用户

  在构建镜像时，创建最低操作系统权限的用户完成容器的运行。

## Code <a id="h1_8"></a>

最后是代码级别，这是程序员能掌控的被攻击面，但不在 Kubernetes 的安全保护范围，提供如下建议：

* 仅通过 TLS 访问

  如果您的代码需要通过TCP进行通信，则理想情况下，它将提前与客户端执行TLS握手。除少数情况外，默认行为应是对传输中的所有内容进行加密。

* 限制通信范围

  无论什么情况下，程序只应该公开必要的端口。

* 第三方依赖安全性

  确保第三方依赖是安全的。

* 静态代码分析

  大多数语言都提供了静态代码分析，可以分析代码中是否存在潜在的不安全编码实践。

* 动态探测攻击

  自动化工具可以针对您的服务尝试会破坏服务的攻击。这些包括SQL注入，CSRF和XSS。最受欢迎的动态分析工具之一是[OWASP Zed Attack](https://www.owasp.org/index.php/OWASP_Zed_Attack_Proxy_Project)代理。

