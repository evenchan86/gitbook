# ssh端口转发：远程和本地



#### 1 需求描述 <a id="1-&#x9700;&#x6C42;&#x63CF;&#x8FF0;"></a>

**1.1 机器状况**

| 机器号 | IP | 用户名 | 备注 |
| :--- | :--- | :--- | :--- |
| A | 192.168.0.A | usr\_a | 目标服务器，在局域网中，可以访问A |
| B | B.B.B.B | usr\_b | 代理服务器，在外网中，无法访问A |
| C | - | - | 可以直接访问B，无法直接访问A |

**1.2 目标**

从C机器使用 SSH 访问 A

#### 2 解决方案 <a id="2-&#x89E3;&#x51B3;&#x65B9;&#x6848;"></a>

在A机器上配置到B机器上的远程端口转发；在B机器上做本地端口转发。

上述方案，使用了ssh 2个技术：远程端口转发和本地端口转发。

**2.1 远程端口转发**

远程端口转发：ssh client跟服务client不在一个网络，其命令为`ssh -R <local port>:<remote host>:<remote port> <SSH hostname>`

远程端口转发实例：如下图，LDAP client要访问防火墙后的LDAP server，但但防火墙设置只允许ssh服务通过，或者LDAP server只允许本地访问，因此LDAP client无法直接访问到LDAP server。此时可以通过SSH本地端口转发，即在SSH client建立SSH连接后，由与LDAP server同一网络内的ssh client，将报文转发到同处于防火墙内部的LDAP server。

![remote port](https://www.ibm.com/developerworks/cn/linux/l-cn-sshforward/image003.jpg)

将上述实例的LDAP server替换为ssh server，也就能通过远程端口转发，实现从B机器到A机器的ssh访问：当在B机器上访问`127.0.0.1:<local port>`时，通过远程端口转发，实际访问的是A机器的服务`<remote host>:<remote port>`。

但是由于远程端口转发只能在B机器上配置本地监听\(`127.0.0.1:<local port>`\)，C机器无法向B机器的local port发起连接，因此还需要在B机器上做一次本地端口转发，从而实现C机器通过访问B.B.B.B:loca port2来实现访问A机器上的ssh服务。

**2.2 本地端口转发**

本地端口转发：ssh client跟实际client在一个网络，其命令为`ssh -L [local addr]<local port>:<remote host>:<remote port> <SSH hostname>`

本地端口转发实例：如下图，LDAP client想要访问防火墙后的LDAP server，但防火墙设置只允许ssh服务通过，或者LDAP server只允许本地访问，因此LDAP client无法直接访问到LDAP server。此时可以通过SSH本地端口转发，即从防火墙外部网络建立到防火墙内SSH server的SSH连接，由防火墙内部SSH server本地转发到同处于防火墙内部的LDAP server。

由于ssh报文总是加密的，因此通过本地端口转发，可以保证不会被抓包窃取到信息。

![local](https://www.ibm.com/developerworks/cn/linux/l-cn-sshforward/image002.jpg)

在本方案中，我们想解决的跟上述实例是同一个问题：服务提供者只允许本地访问，因此需要做SSH 本地端口转发，从而在B机器上建立监听一个新的端口\(port 2\)，该监听服务可以接受任意客户端的请求，并将请求转发给\(在远程端口转发时配置的\)本地监听端口\(port 2\)。

**2.3 多主机转发**

通过远程端口转发和本地端口转发时，能够实现开始时提出的从C机器直接访问A机器的需求：在A机器上配置远程端口转发，从而允许从B机器SSH连接到A机器；在B机器上配置本地端口转发，从而允许从C机器SSH连接到B机器；最终实现从C机器直接SSH连接到A机器。

_此处应该有图_

#### 3 实施配置 <a id="3-&#x5B9E;&#x65BD;&#x914D;&#x7F6E;"></a>

**3.1 环境需求**

每台机器上都需要 SSH 客户端。A、B 两台机器上需要 SSH 服务器端，通常是 openssh-server。在 Ubuntu 上安装过程为

```text
sudo apt-get install openssl-server
```

**3.2 实施步骤**

\(1\) B机器在公网上开启ssh监听；A机器在内网上开启监听。

```text
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      872/sshd     (B机器) 
tcp        0      0 0.0.0.0:22                  0.0.0.0:*                   LISTEN      2233 (A机器)
```

\(2\) 建立A机器到B机器的远程端口转发【A 机器上操作】。A机器将以ssh client身份建立到B机器的连接。需要注意由于A机器在内网，连接到B机器时会做一次SNAT替换源地址和源端口号。

命令：

```text
ssh -fCNR <port_b1>:localhost:22 usr_b@B.B.B.B
```

我选取的 port\_b1 为 8087，该步骤建立的 tcp 连接：

```text
tcp        0      0 [A.A.A.A]:47102   [B.B.B.B]:22            ESTABLISHED 4851/ssh   (A机器)
tcp        0      0 [B.B.B.B]:22    [NAT(A.A.A.A)]:[NAT PORT] ESTABLISHED 819/sshd: root   (B机器)
tcp        0      0 127.0.0.1:8087          0.0.0.0:*         LISTEN      819/sshd: root (B机器)
```

`<port_b1>` 为B机器上端口，用来与A机器上的22端口绑定。可以看到，该步骤会建立 AB 之间的ssh连接，会在B机器上建立针对`<port_b1>`的监听。之后，在B机器上所有到127.0.0.1:8087的报文，会被ssh封装以后，通过AB之间的ssh连接发送给A机器\(注意到后2个连接的pid都是819了吗？想一想为什么\)。

\(3\) 建立B机器上的本地端口转发。做这一步是因为第二步配置后的8087端口只支持本地访问【B 机器上操作】

```text
ssh -fCNL "*:<port_b2>:localhost:<port_b1>' localhost
```

`<port_b2>` 为本地转发端口\(我设置了8086\)，用以和外网通信，并将数据转发到 `<port_b1>` \(即上面选定的8087\)，实现可以从其他机器访问。

其中的\*表示接受来自任意机器的访问。

该步骤会建立对 `<port_b2>` 即8086监听的TCP连接。

```text
tcp        0      0 0.0.0.0:8086            0.0.0.0:*               LISTEN      7885/ssh    (B机器)
```

\(4\) 现在在C机器上可以通过B机器 ssh 到A机器

```text
ssh -p <port_b2> usra@B.B.B.B
```

此时在B机器上会看到C机器发起的连接

```text
tcp        0      0 [B.B.B.B]:8086      [C.C.C.C]:7742       ESTABLISHED 7885/ssh            
```

B 机器的8086监听的ssh进程\(pid 7877\)收到C的请求后，转给sshd:root\(pid 819\)

```text
tcp        0      0 127.0.0.1:51728         127.0.0.1:8087          ESTABLISHED 7877/sshd: root   
tcp        0      0 127.0.0.1:8087          127.0.0.1:51728         ESTABLISHED 819/sshd: root
```

B 机器的sshd:root\(pid 819\)将报文通过ssh隧道发给A机器 client，报文通过8087端口号标记；A机器 ssh client\(pid 4851\)收到后，将报文解包后，向真正的server\(ssh, pid 2233\)建立连接，并将报文发给该A 机器的ssh server

```text
tcp        0      0 0.0.0.0:22                  0.0.0.0:*                   LISTEN      2233
tcp        0      0 ::1:42286                   ::1:22                      ESTABLISHED 4851/ssh   
tcp        0      0 ::1:22                      ::1:42286                   ESTABLISHED 2233
```

#### 4 扩展阅读 <a id="4-&#x6269;&#x5C55;&#x9605;&#x8BFB;"></a>

**4.1 非ssh服务**

上述方案的目的是将内部的ssh服务开放出来，如果是非ssh服务，比如8080端口的web服务呢？实际也是类似的，只要第二步将`localhost:22`改为`localhost:8080`即可，这样C机器在web访问 B.B.B.B:\[PORT B\]时，实际访问的是A机器上配置的web服务。

A 机器并不需要理解C机器client发起的是http请求还是ssh请求，它看到的只是TCP层面，只要跟后端服务建立连接后，将C的报文往 后端服务丢就可以了。

**4.2 SSH动态端口转发**

有时内部网络提供的服务有多种，端口很多，如果一一配置会比较繁琐，此时可以使用SSH动态端口转发。

```text
ssh -D <local port> <SSH Server>
```

![dynamic](https://www.ibm.com/developerworks/cn/linux/l-cn-sshforward/image005.jpg)

这里SSH创建是在内部SSH server上启动了一个socks代理服务。由于前面已经配置了从B机器到A机器的SSH链路，因此只要B机器上配置动态端口转发，并且在C机器上配置socks代理为B机器，就能实现在C机器上访问所有A机器\(及其所在网络\)的服务。

想想动态端口转发还能用来做什么？嘿嘿。

**4.3 SSH参数解释**

* -f 后台运行
* -C 允许压缩数据
* -N 不执行任何命令
* -R 将端口绑定到远程服务器，反向代理
* -L 将端口绑定到本地客户端，正向代理

ref:

* [实战SSH端口转发](https://www.ibm.com/developerworks/cn/linux/l-cn-sshforward/index.html)
* [冰雪殿: 从外网 SSH 进局域网，反向代理+正向代理解决方案](https://segmentfault.com/a/1190000002718360)

