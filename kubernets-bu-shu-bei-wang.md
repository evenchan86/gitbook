# kubernets部署备忘

## [![Zlatan Eevee](https://ieevee.com/assets/logo.png)](https://ieevee.com/)

记录下GFW内k8s的部署流程，备忘.

1、各节点上配置hostname，配置resole.conf

```text
echo "titan1" > /etc/hostname
sysctl kernel.hostname="titan1"
echo "nameserver x.x.x.x" >> /etc/resolv.conf
```

2、各节点上加k8s的repo

```text
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

3、各节点上装基础包

```text
yum install -y docker kubelet kubectl kubernetes-cni kubeadm
```

4、各节点上配置docker mirror

修改 `/usr/lib/systemd/system/docker.service`，加上 `--registry-mirror=https://ocez8l09.mirror.aliyuncs.com`:

```text
ExecStart=/usr/bin/docker-current daemon  --registry-mirror=https://ocez8l09.mirror.aliyuncs.com\
          --exec-opt native.cgroupdriver=systemd \
          $OPTIONS \
```

并重新加载配置，并重启docker服务

```text
systemctl daemon-reload
systemctl restart  docker.service
```

5、各节点上拉取k8s的包并tag为gcr.io

```text
#!/bin/bash
images=(kube-proxy-amd64:v1.5.1 kube-discovery-amd64:1.0 kubedns-amd64:1.9 kube-scheduler-amd64:v1.5.1 kube-controller-manager-amd64:v1.5.1 kube-apiserver-amd64:v1.5.1 etcd-amd64:3.0.14-kubeadm kube-dnsmasq-amd64:1.4 exechealthz-amd64:1.2 pause-amd64:3.0 kubernetes-dashboard-amd64:v1.5.0 dnsmasq-metrics-amd64:1.0)
for imageName in ${images[@]} ; do
  docker pull ist0ne/$imageName
  docker tag ist0ne/$imageName gcr.io/google_containers/$imageName
  docker rmi ist0ne/$imageName
done
```

6、 在master上：

```text
kubeadm init --pod-network-cidr 10.244.0.0/16 --use-kubernetes-version v1.5.1
```

注意flannel网络方案必须要设置–pod-network-cidr 10.244.0.0/16。

最终kubectl get pods –all-namespaces 可以看到除了kube-dns外其他的都RUNNING状态。kube-dns要等到下面flannel部署ok了以后才能RUNNING。

7、部署flannel

所有节点上：

```text
docker pull docker.io/fenghan/flannel:v0.7.0-amd64
docker tag docker.io/fenghan/flannel:v0.7.0-amd64 quay.io/coreos/flannel:v0.7.0-amd64
docker rmi docker.io/fenghan/flannel:v0.7.0-amd64
```

master上：

```text
kubectl create -f kube-flannel.yml
```

此时只有master上有flannel，kubectl get pods –all-namespaces -o wide可以看到kube-flannel和kube-dns都RUNNING。

8、各节点上配置防火墙，准备接入minio节点

```text
iptables -I INPUT -p tcp -m tcp --dport 8472 -j ACCEPT
iptables -I INPUT -p tcp -m tcp --dport 6443 -j ACCEPT
iptables -I INPUT -p tcp -m tcp --dport 9898 -j ACCEPT
iptables -I INPUT -p tcp -m tcp --dport 10250 -j ACCEPT
```

其中8472是flannel使用，9898和6443是minio访问master使用。centos必须配置，否则iptables -L -vn\|more会看到INPUT的reject-with icmp-host-prohibited计数一直在增加。 10250是kubectl exec使用的，不加会报“Error from server: error dialing backend: dial tcp 192.168.128.164:10250: getsockopt: no route to host”。

9、minio节点加入k8s集群

```text
kubeadm join --token=ce91a6.91890123c3be69b1 192.168.128.158
```

10、最终状态

```text
[root@titan1 k8s]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                    READY     STATUS    RESTARTS   AGE
default       kube-flannel-ds-0zmt9                   2/2       Running   2          3d
default       kube-flannel-ds-90gk5                   2/2       Running   2          3d
default       kube-flannel-ds-cw5z4                   2/2       Running   0          3d
kube-system   dummy-2088944543-n4t7k                  1/1       Running   0          3d
kube-system   etcd-titan1                             1/1       Running   1          3d
kube-system   kube-apiserver-titan1                   1/1       Running   0          3d
kube-system   kube-controller-manager-titan1          1/1       Running   0          3d
kube-system   kube-discovery-1769846148-tnfhv         1/1       Running   0          3d
kube-system   kube-dns-2924299975-8b8t7               4/4       Running   462        3d
kube-system   kube-proxy-86pbd                        1/1       Running   0          3d
kube-system   kube-proxy-tqqkv                        1/1       Running   1          3d
kube-system   kube-proxy-vsxmr                        1/1       Running   1          3d
kube-system   kube-scheduler-titan1                   1/1       Running   0          3d
kube-system   kubernetes-dashboard-3109525988-z637x   1/1       Running   15         3d
```

kube-flannel在default命名空间里。下次部署我要改成kube-system。

11、部署dashboard

```text
docker pull fenghan/kubernetes-dashboard-amd64:v1.5.1
docker tag docker.io/fenghan/kubernetes-dashboard-amd64:v1.5.1 gcr.io/google_containers/kubernetes-dashboard-amd64:v1.5.1
docker rmi docker.io/fenghan/kubernetes-dashboard-amd64:v1.5.1
```

wget https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml

因为已经本地已经有镜像了，所以将 imagePullPolicy: Always 改为 imagePullPolicy: IfNotPresent

kubectl create -f kubernetes-dashboard.yaml



