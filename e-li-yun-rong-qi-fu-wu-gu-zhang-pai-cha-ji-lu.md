# 阿里云容器服务故障排查记录

### 1. 镜像仓库被设置为公有，导致镜像泄露风险：    <a id="1"></a>

   **错误现象：**  
  公有镜像仓库可能会被云上其它用户拉取，导致泄露镜像安全风险；部分运维或者开发同学，因为没有设置准确的 secret 到 Deployment，为了解决无法拉取镜像问题，直接开放镜像仓库为公有。  
   **解决方法：**  
   镜像仓库的命名空间一定要设置为私有，准确设置绑定云效中docker 镜像账号，通过云效发布应用；  
   严格设定容器镜像仓库的维护权限；

### 2. 镜像拉取失败： <a id="2"></a>

   **错误现象：**

```text
## 查看 pod 部署日志   
kubectl logs {pod}     
## 错误信息
Failed to pull image "registry-vpc.{region_id}.aliyuncs.com/{app_name}-daily/{app_name}:20190823150817": 
rpc error: code = Unknown desc = Error response from daemon: 
pull access denied for registry-vpc.{region_id}.aliyuncs.com/{app_name}-daily/{app_name}, repository does not exist or may require 'docker login'
```

  错误原因：   

* 当前 tag 的镜像不存在、镜像地址错误、镜像网络不通，没法访问；            **解决方法：**

   只需修改正确地址或者打通网络即可；   

* Deployment 或者 Statefulset 的imagePullSecrets 没有设置或者设置错误    **解决方法：**

  控制台或者使用命令建立保密字典，然后使用 imagePullSecrets 引入，或者自己建立 Secret:       

```text
## deplyment yaml 设置： 
imagePullSecrets:            
    - name: acr-credential-be5ac8be6a88c42ac1d831b85135a585            
```

### 3. SLB被容器服务清除，导致故障，需要重建和安全配置： <a id="3"></a>

   **错误现象:**  
与容器服务关联配置的负载均衡\(SLB\)被清除；  
   **错误原因:**  
   因为有状态副本或者 Deployment集部署删除，存在级联删除 Service 情况，开发和运维人员使用重建方式修改自己配置的时候，导致 service 级联相应 SLB 被删除，导致故障，需要紧急重建 SLB 并多方增加访问控制等配置。  
   Service 配置任意修改或者删除，比如将 SLB 模式修改为 NodePort 或者 Cluster 模式，导致 SLB 负载均衡配置被清除。  
   **解决与防止方法：**  
   kubernetes 使用 NodePort，再通过手动建立负载均衡\(SLB\)与 NodePort 关联，解耦 Service 与 SLB 级联关系。  
   使用 Ingress 暴露服务，Service 使用虚拟集群 IP，与 Ingress 关联。

使用此种方式需要注意 SLB 到后端服务的负载均衡，具体参考[负载均衡](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2Fwww.atatech.org%2Farticles%2F148053) 中负载均衡请求部分。

### 4. ECS 添加到集群失败： <a id="4"></a>

   **错误现象:**  
  集群增加已有节点或者扩容失败。  
错误日志例如下：

```text
2019-07-31 19:44:59cf7c629dbf1dc4088a5a6b316fa5e561a | Wait k8s node i-9dpfd2n6ijvdd5tb642r join cluster timeout  
2019-07-31 19:44:59cf7c629dbf1dc4088a5a6b316fa5e561a | Failed to check instance i-9dpfd2n6ijvdd5tb642r healthy : Wait for cn-north-2-gov-1.i-9dpfd2n6ijvdd5tb642r join to cluster cf7c629dbf1dc4088a5a6b316fa5e561a timeout  
2019-07-31 19:44:59cf7c629dbf1dc4088a5a6b316fa5e561a | Failed to init instance i-9dpfd2n6ijvdd5tb642r, err Wait for cn-north-2-gov-1.i-9dpfd2n6ijvdd5tb642r join to cluster cf7c629dbf1dc4088a5a6b316fa5e561a timeout
2019-07-31 19:44:59cf7c629dbf1dc4088a5a6b316fa5e561a | Failed to attach node i-9dpfd2n6ijvdd5tb642r, err Wait for cn-north-2-gov-1.i-9dpfd2n6ijvdd5tb642r join to cluster cf7c629dbf1dc4088a5a6b316fa5e561a timeout  
```

   **错误原因:**

* 单个集群内节点数量配额达到阈值，导致 ECS 几点没法加入；
* 虚拟网络 VPC中路由表的路由条目达到阈值,导致新增节点没法添加路由条目；
* kubernetes apiserver 的 SLB 负载均衡设置有访问控制，导致添加的 ECS 没法访问 ApiServer;
* 添加的 ECS 节点自身安全组限制或者底层网络故障，导致没法访问 apiserver;

   **解决方法:**

* 联系阿里云同学增加集群或者路由表阈值；
* 配置 SLB 访问控制，增加白名单；
* 配置安全组，增加白名单，或者重建 ECS，释放故障 ECS;

### 5. 集群中，个别 POD 网络访问不通： <a id="5"></a>

   **错误现象:**  
   个别应用产生一定比例的访问超时错误报告，经过监控系统 sunfire 配置发现特定的A 应用 pod 与另外一个应用B pod 网络不通；  
网络测试：

* A pod 访问不通 B pod;
* B pod 能访问通 A pod;
* A pod 宿主机 ECS 能访问通 B pod宿主机 ECS；
* B pod 宿主机 ECS 能访问通 A pod宿主机 ECS；
* A pod 访问通 B pod宿主机 ECS；
* B pod 访问通 A pod宿主机 ECS； 抓包并与阿里云同学网络排查发现， 云上 VPC 的 NC 网络控制模块没有正确下发路由信息，导致网络故障。

   **解决方法:**

联系阿里云 vpc 同学，排查 vpc 中 NC 路由下发问题。

### 6. 部分 ECS 网络故障，Master 访问Node 的 kube-proxy 端口访问不通：  <a id="6"></a>

   **错误现象:**  
新添加一批 ECS 节点，个别 ECS 总是添加失败，报告超时，排除 SLB 访问控制等原因；  
监控 kubelet-TelnetStatus.Value 报警；

```text
【阿里云监控】应用分组-k8s-cbf861623f10144c488813375a8a0d489-worker-1个实例发生报警, 触发规则：kubelet-TelnetStatus.Value   
14:16 可用性监控[kubelet dingtalk-a-prod-node-X06/172.16.6.9] ，状态码（631>400 ），持续时间1天3分钟
```

   **错误原因:**  
经过观察和多次测试，失败的 ECS 网络很不稳定，经常网络不通；  
该故障排查错层较长，一直没怀疑机器问题；  
ECS 宿主机基础设施有问题，排除释放此宿主机上的 ECS。  
   **解决方法:**  
新建 ECS, 释放故障 ECS，重新加入 kubernetes 集群。

