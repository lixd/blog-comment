---
title: "裸机 Kubernetes 集群负载均衡器: MetalLB 简明教程"
description: "什么是 MetalLB，MetalLB 部署与测试，工作流程，Layer2 模式和 BGP 模式各种工作原理及使用建议"
date: 2023-02-11
draft: false
categories: ["CloudNative"]
tags: ["CloudNative","MetalLB"]
---

本文包括以下内容：什么是 MetalLB，MetalLB 部署及测试，MetalLB  工作流程，Layer2 模式和 BGP 模式各种工作原理及使用建议。

<!--more-->

## 0. 什么是 MetalLB

> Repo: https://github.com/metallb/metallb
>
> 官网: https://metallb.universe.tf/installation

**一句话描述，什么是 MetalLB**：

**MetalLB is a load-balancer implementation for bare metal [Kubernetes](https://kubernetes.io/) clusters, using standard routing protocols.**

MetalLB 是一个用于裸机 Kubernetes 集群的负载均衡器实现，使用标准路由协议。



k8s 并没有为裸机集群实现负载均衡器，因此我们只有在以下 IaaS 平台（GCP, AWS, Azure）上才能使用 LoadBalancer 类型的 service。

因此裸机集群只能使用 NodePort 或者 externalIPs service 来对面暴露服务，然而这两种方式和 LoadBalancer  service 相比都有很大的缺点。

而 MetalLB 的出现就是为了解决这个问题。



## 1. QuickStart

### 限制条件

安装 MetalLB 很简单，不过还是有一些限制：

* 1）需要 Kubernetes v1.13.0 或者更新的版本
* 2）集群中的 CNI 要能兼容 MetalLB，具体兼容性参考这里 [network-addons](https://metallb.universe.tf/installation/network-addons/) 
  * 像常见的 Flannel、Cilium 等都是兼容的，Calico 的话大部分情况都兼容，BGP 模式下需要额外处理
* 3）提供一下 IPv4 地址给 MetalLB 用于分配
  * 一般在内网使用，提供同一网段的地址即可。
* 4）BGP 模式下需要路由器支持 BGP
* 5）L2 模式下需要各个节点间 7946 端口联通

看起来限制比较多，实际上这些都比较容器满足，除了第四条。



### 安装 MetalLB

官方提供了好几种安装方式，yaml、helm、operator 等，这里使用 yaml 方式安装。

如果 kube-proxy 使用的是 ipvs 模式，需要修改 kube-proxy 配置文件，启用严格的 ARP

```Bash
kubectl edit configmap -n kube-system kube-proxy

# 修改点如下，黄色标记
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```

使用 yaml 安装

```Bash
# 原生
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml

# 启用 FRR
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-frr.yaml
```



### 配置

MetalLB 有 Layer2 模式和 BGP 模式，**任选一种模式进行配置即可**。

> 因为 BGP 对路由器有要求，因此建议测试时使用 Layer2 模式。



#### Layer 2 模式配置

**1）创建 IPAdressPool**

> 多个实例`IP地址池`可以共存,并且地址可以由CIDR定义， 按范围分配，并且可以分配IPV4和IPV6地址。

```Bash
cat <<EOF > IPAddressPool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  # 可分配的 IP 地址,可以指定多个，包括 ipv4、ipv6
  - 172.20.175.140-172.20.175.150
EOF

kubectl apply -f IPAddressPool.yaml
```

**2）创建 L2Advertisement，并关联 IPAdressPool**

> L2 模式不要求将 IP 绑定到网络接口 工作节点。它的工作原理是响应本地网络 arp 请求，以将计算机的 MAC 地址提供给客户端。

如果不设置关联到 IPAdressPool，那默认 L2Advertisement 会关联上所有可用的 IPAdressPool

```Bash
cat <<EOF > L2Advertisement.yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool #上一步创建的 ip 地址池，通过名字进行关联
EOF

kubectl apply -f L2Advertisement.yaml
```



#### BGP 模式配置

**1）配置 BGPPeer**

需要为每个需要连接的路由器都创建一个 BGPPeer 实例，这样 MetalLB 才能与 BGP 路由器建立会话。

```Bash
cat <<EOF > BGPPeer.yaml
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: sample
  namespace: metallb-system
spec:
  myASN: 64500 # MetalLB 使用的 AS 号
  peerASN: 64501 # 路由器的 AS 号
  peerAddress: 10.0.0.1 # 路由器地址
EOF

kubectl apply -f BGPPeer.yaml
```

**2）创建 IPAdressPool**

```Bash
cat <<EOF > IPAddressPool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250 # 可分配的 IP 地址
EOF

kubectl apply -f IPAddressPool.yaml
```

**3）创建 L2Advertisement，并关联 IPAdressPool**

如果不设置关联到 IPAdressPool，那默认 L2Advertisement 会关联上所有可用的 IPAdressPool。

也可以使用标签选择器来筛选需要关联的 IPAdressPool 列表。

```Bash
cat <<EOF > L2Advertisement.yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
EOF

kubectl apply -f  L2Advertisement.yaml
```

启动 FRR 作为 BGP 的后端，不需要额外的配置，只是在安装的时候会创建不同的 CR。启动 FRR 模式后，需要用到某些特性要单独进行配置。



### 测试

创建一个 nginx deploy 以及一个 loadbalance 类型的 svc 来测试。

使用以下命令创建 nginx deploy：

```Bash
cat <<EOF > nginx-dp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: docker.io/nginx:latest
        ports:
        - containerPort: 80
EOF

kubectl apply -f nginx-dp.yaml
```

使用以下命令创建 nginx-svc：

```Bash
cat <<EOF > nginx-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx2
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
  - name: nginx-port
    protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
EOF

kubectl apply -f nginx-svc.yaml
```

然后查看 svc，看看是不是真的分配了 ExternalIP

```Bash
[root@iam2 ~]# kubectl get svc nginx
NAME    TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)        AGE
nginx   LoadBalancer   10.103.240.239   192.168.1.241   80:30164/TCP   5s
```

访问对应的 EXTERNAL-IP 

```Bash
curl http://192.168.1.241
```

能够访问，说明 LB 正常工作。



## 2. 工作流程

MetalLB 做的工作可以分为两个部分：

- 1）**地址分配**：当创建 *LoadBalancer* Service 时，MetalLB 会为其分配 IP 地址。这个 IP 地址是从**预先配置的 IP 地址库**获取的。同样，当 Service 删除后，已分配的 IP 地址会重新回到地址库。
- 2）**对外广播**：分配了 IP 地址之后，需要**让集群外的网络知道这个地址的存在**。MetalLB 使用了标准路由协议实现：ARP、NDP 或者 BGP。
  -  在 Layer 2 模式，使用 ARP（ipv4）/NDP（ipv6） 协议；
  - 在 BPG 模式，自然是使用 BGP 协议。

> ***ARP（Address Resolution Protocol）***：是根据IP地址获取物理地址的一个TCP/IP协议。
>
> ***NDP（neighbor Discovery protocol）***：是ICMPv6的子协议是IPV6协议体系中一个重要的基础协议，邻居发现协议替代了IPV4的ARP，ICMP路由器发现。它定义了使用ICMPv6报文实现地址解析，跟踪邻居状态，重复地址检测，路由器发现，以及重定向等功能。

同时 MetalLB 分别使用两个组件来实现了上述两个功能：

- **Controller**：实现地址分配，以 *Deployment* 方式运行，用于监听 Service 的变更，分配/回收 IP 地址。
- **Speaker**：实现地址对外广播，以 *DaemonSet* 方式运行，对外广播 Service 的 IP 地址。



具体的工作流如下：

- **Controller 负责监听 Service 变化并分配或回收 IP**，当 Service 配置为 LoadBalancer 模式时，从 IP 池分配给到相应的 IP 地址并对该 IP 的生命周期进行管理。
  - 创建 Service 时（或者从非 LoadBalancer 类型修改为 LoadBalancer 类型）时从 IP 池选择一个 IP 并分配，
  - 删除 Service （或者从 LoadBalancer 类型修改为非 LoadBalancer 类型）时回收该 IP 到 IP 池
- **Speaker 则会依据选择的协议进行相应的广播或应答，实现 IP 地址的通信响应**。当业务流量通过 TCP/UDP 协议到达指定的 Node 时，由 Node 上面运行的 Kube-Proxy 组件对流量进行处理，并分发到对应服务的 Pod 上面。
  - 如果是 Layer2 模式 Speaker 就会响应 ARP（ipv4）/NDP（ipv6）请求
  - 如果是 BGP 模式 Speaker 则发送 BGP 广播，将路由规则同步给 peer。



## 3. Layer2 模式工作原理

> [METALLB IN LAYER 2 MODE](https://metallb.universe.tf/concepts/layer2/)

### 大致原理

Layer 2 中的 Speaker 工作负载是 DeamonSet 类型，在每个节点上都调度一个 Pod，首先，几个 Pod 会先进行选举，选举出 Leader。

由 Leader Pod 获取所有 LoadBalancer 类型的 Service，并将已分配的 IP 地址绑定到当前主机到网卡上。同时该 Leader 会响应对  ExternalIP 的  ARP（ipv4）/NDP（ipv6） 请求，因此从局域网层面来看，speaker 所在的机器是有多个 IP 地址的，当前其中也包含 ExternalIP。

> 从网络的角度来看，计算机的网络接口分配了多个IP地址,因为对不同 ip 地址的 arp 请求返回的都是这个节点的 mac 地址。

因此与 ExternalIP 相关的所有流量都会流向该节点。在该节点上， kube-proxy 将接收到的流量传播到对应 Service 的后端 Pod。

> **也就是说，所有 LoadBalancer 类型的 Service 的 IP 同一时间都是绑定在同一台节点的网卡上。**

### 局限性

在 Layer2 模式中会存在以下两种局限性：**单节点瓶颈**和**故障转移慢**。

#### 单节点瓶颈

由于 Layer 2 模式会使用单个选举出来的 Leader 来接收 ExternalIP 的所有流量，这就意味着服务的入口带宽被限制为单个节点的带宽，单节点的流量处理能力将成为整个集群的接收外部流量的瓶颈。

> 从这个角度来看，Layer2 模式更像是实现了**故障转移**，而不是负载均衡，因为同时只能在一个节点负责接收数据。

#### 故障转移慢

在故障转移方面，MetalLB 也实现了**自动故障转移**。目前的机制是通过 [memberlist](https://github.com/hashicorp/memberlist) 这个基于 gossip 协议的成员和故障检测的库，其他节点检测到 Leader 故障后自动重新选举出新 Leader，新的 Leader 自动接管所有  ExternalIP  并发送大量 二层数据包来通知客户端(也就是区域网中的其他节点) ExternalIP 的 MAC 地址变化。

> 大部分操作系统都能处理这部分 二层数据包并更新 **neighbor caches**，但是也有极少部分系统不能正确处理这个数据包，从而无法及时更新缓存，还会继续请求旧的 Leader 节点，这种情况下可以让旧 Leader 多存活几分钟，用于处理这部分客户端的请求。

根据官网文档描述**故障转移正常情况下会在几秒内完成**，一般不会超过 10 秒。但是在更新完成前 ExternalIP 会无法访问。

> 即：会出现几秒钟的服务中断
>
> 这个 10s 只是官方说的，可能经常测试或者是一个大概值，和这段 [代码#announcer.go#L51](https://github.com/metallb/metallb/blob/main/internal/layer2/announcer.go#L51) 里的10s 的循环不是一回事





## 4. BGP 模式工作原理

> [METALLB IN BGP MODE](https://metallb.universe.tf/concepts/bgp/)

### 大致原理

在 BGP 模式下每个节点（ 上的 Speaker Pod）都会和路由器建立一个 BGP peer session，并且通过这些 peer session 来告知外部网络 ExternalIPs 的存在。

BGP模式是以集群中的主机与对等体进行共享路由信息，从而实现集群外部的服务能够访问到集群内部的服务。



和 Layer2 模式不同，**BGP模式真正的实现了负载均衡**，不过具体的负载均衡行为和路由器以及配置有关系。一般默认的行为是根据数据包中的某些**关键字段**进行 hash，并更新 hash 值分配给其中某一个连接。

> 原文：but the common behavior is to balance *per-connection*, based on a *packet hash*.

关键字段常见选择为 **三元组（协议、源IP、目的IP）** 或者**五元组（协议、源IP、目的IP、源端口、目的端口）**

也就是说默认情况下一个连接里的所有数据包都会发送到固定的节点，这样能拥有更好的性能。



### 局限性

BGP 模式最大的弊端就是**不能优雅的处理节点下线**。当集群中某个节点下线时，所有客户端对这个节点的连接都会被主动断开。

> 客户端一般会出现一个 Connection reset by peer 错误

同时由于是**对每个数据包基于 hash 值进行负载均衡**，因此**对后端节点数是非常敏感**的，这也是 BGP 的一个优点，**故障转移非常快。**

**正因为** **BGP** **故障转移很快，反而引发了一个 BGP 模式的最大缺点**：由于 BGP 会对每个数据包做负载均衡，在主机发生故障时会快速切换到新的主机上，从而引发节点变动时**同一连接的不同数据包可能会发送到不同主机上导致网络导致的网络重排问题**。

> 比如第一个包转发到节点 A，然后处理第二个包时添加或故障了一个节点，按照新的节点数进行负载均衡计算，可能第二个数据包就被分到节点 B 了，节点 B 很明显是不能正确处理这个数据包的。
>
> 对客户端来说就是这次请求直接失败了。

解决该弊端的方法没有太理想的解决办法，只能尽量采取一些优化手段：

- 1）路由器配置更加稳定的 hash 算法，比如 "resilient ECMP" 或者 "resilient LAG"。使用这样的算法极大地减少了 后端节点更改时受影响的连接。
- 2）尽量少的增删节点
- 3）在流量少时变更节点，以降低影响
- 4）使用 ingress 来对外暴露服务等等
- ....

### 兼容性

因为 BGP 单 session 的限制，如果 CNI 插件为 Calico ，同时 Calico 也是使用的 BGP 模式，就会有冲突从而导致 MetalLB 无法正常工作。

> Calico 和 MetalLB 都会尝试建立 session



## 5. 小结

MetalLB 是一个用于裸机 Kubernetes 集群的负载均衡器实现，使用标准路由协议。

MetalLB 主要分为 Controller 和 Speaker 两个部分，**Controller 负责 IPAM，Speaker 则负责 IP 通告**。这两个组件就是实现 MetalLB 所需要的全部功能了。

**二者实现原理**

MetalLB 包括 Layer2 模式和 BGP 模式，主要区别在于 IP 通告部分的实现不一样：

* Layer2 模式通过响应对  ExternalIP 的  ARP（ipv4）/NDP（ipv6） 请求来告知其他节点某个 IP 在这台机器上
  * arp 请求是在 二层的，这可能就是该模式为什么叫做 Layer2
* BGP 模式则通过于路由器建立 BGP peer session，从而进行数据同步，让路由器知道某个 IP 在这台机器上

**二者优缺点**

Layer2 模式

- 优点：**通用性好**，适用于任何网络环境，不需要特殊的硬件，甚至不需要花哨的路由器。
- 缺点：单节点瓶颈和故障转移慢

**Layer2 模式是一种基础、通用的实现**，能用，而且易于使用，没有任何限制，但是局限性比较大。

BGP 模式：

- 优点：使用 BGP 可以在多节点间负载均衡，没有单节点瓶颈，同时故障转移很快。
- 缺点： 需要支持 BGP 路由协议的软路由或者硬件路由器设备。
  - 其实不能算缺点了，想要功能总得付出代价。



**使用建议**

**BGP** **模式则是比较理想的实现**，除了依赖支持 BGP 的路由之外，其他方面则没有任何限制，并且没有可用性的问题



**一句话总结**：如果说 Layer2 模式为基础实现，那么 BGP 模式则是 LoadBalance 的终极实现，能用 BGP 模式就尽量用 BGP 模式，否则的话就只能用 Layer2 模式了，如果不知道用什么模式的话直接用 Layer2 模式即可。
