# NetWork Policies 网络策略



Kubernetes通过namespace支持了多租户，不同租户间可以通过ResourceQuota来保证资源的隔离，也无法查看其他namespace的Pod信息等。但默认kubernetes并不限制访问Pod的网络请求，对于一些敏感的应用来说可能不够用。

#### NetworkPolicy 资源 <a id="networkpolicy-&#x8D44;&#x6E90;"></a>

kubernetes可以通过在namespace下设置 Network Policies 来将Policy选中的Pod隔离。

Example

```text
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

* PodSelector：选中需要隔离的Pod。
* policyTypes： 策略类型。NetworkPolicy将流量分为 ingress 和 egress，即入方向和出方向。如果没有指定则表示不闲置。
* ingress：入方向。白名单。需要指定from、ports，即来源、目的端口号。from有三种类型，ipBlock/namespaceSelector/podSelector。
* egress：出方向。白名单。类似ingress，egress需要指定to、ports，即目的、目的端口号。

#### 默认NetworkPolicy <a id="&#x9ED8;&#x8BA4;networkpolicy"></a>

默认，如果namespace下没有配置任何策略，则流量全通。可以在namespace下配置默认策略。

**默认禁止所有入方向流量**

```text
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

**默认允许所有入方向流量**

```text
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
spec:
  podSelector: {}
  ingress:
  - {}
```

**默认禁止所有出方向流量**

```text
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

**默认允许所有出方向流量**

```text
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
spec:
  podSelector: {}
  egress:
  - {}
  policyTypes:
  - Egress
```

**默认禁止所有出入方向流量**

```text
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

怎么记住这些策略呢？很简单，默认ns下没有任何策略，即没有指定policyTypes；一旦指定了某个policyType则表示要禁止；由于是白名单，因此如果没有指定ingress/egress匹配规则，那就0匹配，即任意流量都无法通过白名单；如果指定select为{}，则表示匹配所有，即任意流量都可以通过。

Ref：

* [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

