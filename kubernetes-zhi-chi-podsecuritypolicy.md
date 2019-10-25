# kubernetes支持PodSecurityPolicy



#### PodSecurityPolicy是什么 <a id="podsecuritypolicy&#x662F;&#x4EC0;&#x4E48;"></a>

PodSecurityPolicy是一种用来控制Pod安全相关配置的全局资源。

在开启RBAC的kubernetes集群上，如果允许用户使用kubectl，那么必须开启PodSecurityPolicy，否则用户可能会使用一些特权资源（例如privileged，hostNetwork，hostPath等等），影响node机器的稳定性。

开启PSP后，用户只能使用管理员允许的资源。PSP支持以下方面\(具体请见[官网](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#policy-reference)\)：

| Control Aspect | Field Names |
| :--- | :--- |
| Running of privileged containers | [`privileged`](https://ieevee.com/tech/2019/02/18/psp.html#privileged) |
| Usage of host namespaces | [`hostPID`, `hostIPC`](https://ieevee.com/tech/2019/02/18/psp.html#host-namespaces) |
| Usage of host networking and ports | [`hostNetwork`, `hostPorts`](https://ieevee.com/tech/2019/02/18/psp.html#host-namespaces) |
| Usage of volume types | [`volumes`](https://ieevee.com/tech/2019/02/18/psp.html#volumes-and-file-systems) |
| Usage of the host filesystem | [`allowedHostPaths`](https://ieevee.com/tech/2019/02/18/psp.html#volumes-and-file-systems) |
| White list of Flexvolume drivers | [`allowedFlexVolumes`](https://ieevee.com/tech/2019/02/18/psp.html#flexvolume-drivers) |
| Allocating an FSGroup that owns the pod’s volumes | [`fsGroup`](https://ieevee.com/tech/2019/02/18/psp.html#volumes-and-file-systems) |
| Requiring the use of a read only root file system | [`readOnlyRootFilesystem`](https://ieevee.com/tech/2019/02/18/psp.html#volumes-and-file-systems) |
| The user and group IDs of the container | [`runAsUser`, `runAsGroup`, `supplementalGroups`](https://ieevee.com/tech/2019/02/18/psp.html#users-and-groups) |
| Restricting escalation to root privileges | [`allowPrivilegeEscalation`, `defaultAllowPrivilegeEscalation`](https://ieevee.com/tech/2019/02/18/psp.html#privilege-escalation) |
| Linux capabilities | [`defaultAddCapabilities`, `requiredDropCapabilities`, `allowedCapabilities`](https://ieevee.com/tech/2019/02/18/psp.html#capabilities) |
| The SELinux context of the container | [`seLinux`](https://ieevee.com/tech/2019/02/18/psp.html#selinux) |
| The Allowed Proc Mount types for the container | [`allowedProcMountTypes`](https://ieevee.com/tech/2019/02/18/psp.html#allowedprocmounttypes) |
| The AppArmor profile used by containers | [annotations](https://ieevee.com/tech/2019/02/18/psp.html#apparmor) |
| The seccomp profile used by containers | [annotations](https://ieevee.com/tech/2019/02/18/psp.html#seccomp) |
| The sysctl profile used by containers | [annotations](https://ieevee.com/tech/2019/02/18/psp.html#sysctl) |

#### 开启PodSecurityPolicy <a id="&#x5F00;&#x542F;podsecuritypolicy"></a>

开启很简单，配置apiserver增加admission plugin PodSecurityPolicy即可。

```text
--enable-admission-plugins=NodeRestriction,PodSecurityPolicy
```

修改后apiserver会重启。由于这里没有配置任何Policy，所以重启后PSP会阻止所有Pod的创建。

由于PSP API `policy/v1beta1/podsecuritypolicy`与admission controller是独立的，对于现存的集群，建议先配置psp以及授权，再开启PS。

#### 授权策略 <a id="&#x6388;&#x6743;&#x7B56;&#x7565;"></a>

Policy本身并不会产生实际作用，需要将其与用户或者serviceaccount绑定才可以完成授权。

绑定方法：创建Role，该Role可以use 该PodSecurityPolicy，然后将该role与用户或者serviceaccount绑定。

#### 示例 <a id="&#x793A;&#x4F8B;"></a>

下面举个例子。

**1. 创建Policy**

```text
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: example
spec:
  privileged: false  # Don't allow privileged pods!
  # The rest fills in some required fields.
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
```

**2. 创建serviceaccount**

```text
kubectl create namespace hellobaby
kubectl create sa fake-user
alias kubectl-admin='kubectl -n hellobaby'
alias kubectl-user='kubectl --as=system:serviceaccount:hellobaby:fake-user -n hellobaby'
```

**3. 创建Pod**

```text
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
```

此时使用kubectl-user创建Pod会提示`Error from server (Forbidden): error when creating "STDIN": pods "nginx" is forbidden: unable to validate against any pod security policy: []`，因为sa fake-user并没有psp example的权限，可以通过auth检查下。

```text
kubectl-user auth can-i use podsecuritypolicy/example
```

所以下面需要创建role、rolebinding。

**4. 创建带psp example use权限的role，并与sa hellobaby绑定**

```text
kubectl-admin create role psp:unprivileged --verb=use \
    --resource=podsecuritypolicy \
    --resource-name=example

kubectl-admin create rolebinding fake-user:psp:unprivileged \
    --role=psp:unprivileged \
    --serviceaccount=hellobaby:fake-user
rolebinding "fake-user:psp:unprivileged" created
```

现在再去创建上述Pod nginx，就可以创建了，而且可以创建成功，因为这个Pod没有请求任何特殊权限，是个乖孩子。

加上特权字段，再创建呢？

```text
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    securityContext:
      privileged: true
```

kubectl-user会返回`Error from server (Forbidden): error when creating "STDIN": pods "privileged" is forbidden: unable to validate against any pod security policy: [spec.containers[0].securityContext.privileged: Invalid value: true: Privileged containers are not allowed]`

提示很清楚：`Privileged containers are not allowed`。

PodSecurityPolicy 生效了。

**5. 非sa直接创建的Pod**

大部分情况我们并不会直接创建Pod，而是会创建Deployment。试试看。

```text
$ kubectl-user run nginx --image=nginx
deployment "nginx" created

$ kubectl-user get pods
No resources found.

$ kubectl-user get events | head -n 2
LASTSEEN   FIRSTSEEN   COUNT     NAME              KIND         SUBOBJECT                TYPE      REASON                  SOURCE                                  MESSAGE
1m         2m          15        nginx-7774d79b5   ReplicaSet                            Warning   FailedCreate            replicaset-controller                   Error creating: pods "nginx-7774d79b5-" is forbidden: no providers available to validate pod request
```

提示没有providers用来检查Pod请求，可是我们已经给sa hellobaby:fake-user授权了呀。

原因是，这个Pod是replicaset controller创建的，默认kubectl run的形式创建的应用指定的`spec.template.spec.serviceAccount`和`spec.template.spec.serviceAccountName`都是default，而上面我们RBAC绑定的SA是`hellobaby:fake-user`，并不是`hellobaby:default`。

怎么解决呢？官网给的方法是，再给`hellobaby:default`绑定 role `psp:unprivileged`。的确这样可以解决问题，但这个解决方法并不合理。

我们知道，一个namespace下是可以创建多个serviceaccount的，如果我需要给同一namespace下不同是sa授予不同的psp，那么应该如何给sa default授权呢？显然就无法绑定了。

事实上，我认为官网的这个例子并不合理，而且`kubectl run`命令也要被取消了。

更好的做法是创建一个deployment，并且设置其`spec.template.spec.serviceAccount`和`spec.template.spec.serviceAccountName`为实际的sa。

下面是一个实际的例子。

```text
metadata:
  labels:
    run: nginx-test
  name: nginx-test
  namespace: hellobaby
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx-test
  template:
    metadata:
      labels:
        run: nginx-test
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx-test
        #securityContext:
        #  privileged: true
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      serviceAccount: fake-user
      serviceAccountName: fake-user
```

此时replicaset controller会使用 sa `hellobaby:fake-user`创建pod，由于前面已经为该sa绑定了psp example，所以可以成功创建Pod。

Ref:

* [Pod Security Policies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)

