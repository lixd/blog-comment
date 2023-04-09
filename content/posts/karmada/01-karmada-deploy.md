---
title: "K8s 多集群(一)---Karmada 初体验"
description: "k8s 多集群项目 Karmada 初体验"
date: 2023-04-08
draft: false
categories: ["Karmada"]
tags: ["Karmada"]
---

本文主要记录了 Kubernetes 多集群项目 Karmada 部署及简单使用。

<!--more-->

一句话简介：Karmada 是一个 Kubernetes 多集群领域的开源项目，旨在让用户像使用单集群一样使用多集群。

> github repo：[https://github.com/karmada-io/karmada](https://github.com/karmada-io/karmada)

## 1. 环境准备

Karmada 本身也是运行在 k8s 上的，因此我们需要先准备一个 k8s 环境，同时为了体验多集群效果，最好能多建几个集群。

>  推荐使用 KubeClipper 来创建 k8s 集群，一条命令搞定：[Kubernetes教程(十一)---使用 KubeClipper 通过一条命令快速创建 k8s 集群](https://www.lixueduan.com/posts/kubernetes/11-install-by-kubeclipper/)

本文将使用 3 台 2c4g 机器部署 3 个单 master 集群，其中 1 个用于运行 karmada，另外两个作为业务集群使用。



## 2. Karmada 部署

### 安装 kubectl-karmada 插件

karmada 提供了 kubectl 插件，通过该插件可以很轻松的安装 Karmada，因此我们先安装该插件。

可以使用以下命令一键安装

```Bash
curl -s https://raw.githubusercontent.com/karmada-io/karmada/master/hack/install-cli.sh | sudo bash -s kubectl-karmada
```

或者手动下载

```Bash
wget https://github.com/karmada-io/karmada/releases/download/v1.5.0/kubectl-karmada-linux-amd64.tgz
tar -zxvf kubectl-karmada-linux-amd64.tgz
cp kubectl-karmada /usr/local/bin
kubectl-karmada version
```



### 初始化karmada

正常情况下执行以下命令即可完成 Karmada 初始化

```bash
kubectl karmada init
```

> 不过该命令默认会去 github 下载 crd 压缩包，同时默认会去 k8s.io 拉镜像。

因此在国内使用时，由于网络问题导致还需要指定一些参数才行：

先手动下载 crd，实际上里面就是一些 crd 的 yaml 文件

```Bash
wget https://github.com/karmada-io/karmada/releases/download/v1.5.0/crds.tar.gz
```

然后 init 时指定 crd 地址以及镜像仓库

```Bash
kubectl karmada init \
--kube-image-registry registry.cn-hangzhou.aliyuncs.com/google_containers \
--crds crds.tar.gz
```

安装完成后输出如下：

```SQL
------------------------------------------------------------------------------------------------------
 █████   ████   █████████   ███████████   ██████   ██████   █████████   ██████████     █████████
░░███   ███░   ███░░░░░███ ░░███░░░░░███ ░░██████ ██████   ███░░░░░███ ░░███░░░░███   ███░░░░░███
 ░███  ███    ░███    ░███  ░███    ░███  ░███░█████░███  ░███    ░███  ░███   ░░███ ░███    ░███
 ░███████     ░███████████  ░██████████   ░███░░███ ░███  ░███████████  ░███    ░███ ░███████████
 ░███░░███    ░███░░░░░███  ░███░░░░░███  ░███ ░░░  ░███  ░███░░░░░███  ░███    ░███ ░███░░░░░███
 ░███ ░░███   ░███    ░███  ░███    ░███  ░███      ░███  ░███    ░███  ░███    ███  ░███    ░███
 █████ ░░████ █████   █████ █████   █████ █████     █████ █████   █████ ██████████   █████   █████
░░░░░   ░░░░ ░░░░░   ░░░░░ ░░░░░   ░░░░░ ░░░░░     ░░░░░ ░░░░░   ░░░░░ ░░░░░░░░░░   ░░░░░   ░░░░░
------------------------------------------------------------------------------------------------------
Karmada is installed successfully.

Register Kubernetes cluster to Karmada control plane.

Register cluster with 'Push' mode

Step 1: Use "kubectl karmada join" command to register the cluster to Karmada control plane. --cluster-kubeconfig is kubeconfig of the member cluster.
(In karmada)~# MEMBER_CLUSTER_NAME=$(cat ~/.kube/config  | grep current-context | sed 's/: /\n/g'| sed '1d')
(In karmada)~# kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config  join ${MEMBER_CLUSTER_NAME} --cluster-kubeconfig=$HOME/.kube/config

Step 2: Show members of karmada
(In karmada)~# kubectl --kubeconfig /etc/karmada/karmada-apiserver.config get clusters


Register cluster with 'Pull' mode

Step 1: Use "kubectl karmada register" command to register the cluster to Karmada control plane. "--cluster-name" is set to cluster of current-context by default.
(In member cluster)~# kubectl karmada register 10.0.0.58:32443 --token jcbljd.tfotd1ylodf7jtm6 --discovery-token-ca-cert-hash sha256:3bc5b22efbef8470f010f65b68388266e2b8f3ab62a0151900761cd12760d34f

Step 2: Show members of karmada
(In karmada)~# kubectl --kubeconfig /etc/karmada/karmada-apiserver.config get clusters
```





## 3. 集群管理

Karmada 初始化完成后就可以将业务集群添加到 Karmada 中进行管理了。

注册方式分为 Push 模式和 Pull 模式，二者的注册方式不相同。

- Push 模式下，karmada 控制面会直接访问 member 集群的 kube-apiserver 以获取集群状态和部署资源
- Pull 模式下，会在 member 集群部署 karmada-agent ，包含以下作用
  - 注册集群到 karmada
  - 维护集群状态并报告给 karmada
  - Watch karmada 集群里对应 namespace 下的资源对象，并将其部署到 member 集群里。

### Push 模式

Push 模式完整命令如下:

> 需要注意的是 push 模式下 join 命令是在 karmada 集群中执行

```Bash
# 语法
# kubectl karmada join <member-cluster-name> --kubeconfig=<karmada kubeconfig> --cluster-kubeconfig=<member1 kubeconfig>

# join
kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config join cluster1 --cluster-kubeconfig=$HOME/.kube/config
```

各个参数含义应该一眼就能看出来：

* `--kubeconfig /etc/karmada/karmada-apiserver.config`指定 Karmada kubeconfig，表示操作的是 karmada 的 apiserver，
  * 实际上 Karmada apiserver 也是基于 kube-apiserver 源码修改的，因此相关操作也基本一致。
* ` --cluster-kubeconfig=$HOME/.kube/config` 则是指定待注册到 karmada 的 member 集群的 kubeconfig。

添加好后可以使用 kubectl 命令进行查看

> member 集群在 karmada 中使用 clusters 这个  CR 对象进行存储

```Bash
[root@karmada ~]#  kubectl --kubeconfig /etc/karmada/karmada-apiserver.config get clusters
NAME     VERSION   MODE   READY   AGE
Cluster1 v1.23.6   Push   True   1m24s
```

可以看到 cluster1 已经成功添加进来了。

### Pull 模式

Pull 模式完整命令如下：

> 需要注意的是 pull 模式下的 register 命令需要到 member 集群中执行

pull 模式下的注册命令格式如下：

```bash
kubectl karmada register <karmada-apiserver> --token <token> --discovery-token-ca-cert-hash <hash>
```

类似与 kubeadm join 命令，需要提供 token 以及 ca证书的 hash 值作为校验。

大家仔细观察的话，可以发现在执行 karmada init 之后输出的日志里实际上是有提供该命令的

```bash
kubectl karmada register 10.0.0.58:32443 --token jcbljd.tfotd1ylodf7jtm6 --discovery-token-ca-cert-hash sha256:3bc5b22efbef8470f010f65b68388266e2b8f3ab62a0151900761cd12760d34f
```

当然，我们也可以使用以下命令重新生成 token（有效期 24h）：

```Bash
kubectl karmada token create --print-register-command --kubeconfig /etc/karmada/karmada-apiserver.config
```

输出如下：

```Bash
[root@lixd-tmp-1 ~]# kubectl karmada token create --print-register-command --kubeconfig /etc/karmada/karmada-apiserver.config
kubectl karmada register 10.0.0.58:32443 --token 7pn49b.d9xhhpn9x8haqraq --discovery-token-ca-cert-hash sha256:3bc5b22efbef8470f010f65b68388266e2b8f3ab62a0151900761cd12760d34f
```

在 member 集群执行该命令进行注册：

```Bash
kubectl karmada register 10.0.0.58:32443 --token jcbljd.tfotd1ylodf7jtm6 --discovery-token-ca-cert-hash sha256:3bc5b22efbef8470f010f65b68388266e2b8f3ab62a0151900761cd12760d34f
```

注册的时候会在当前集群启动一个 karmada-agent pod，用于和 karmada apiserver 交互。

```Bash
[root@lixd-tmp-2 ~]# kubectl karmada register 10.0.0.58:32443 --token jcbljd.tfotd1ylodf7jtm6 --discovery-token-ca-cert-hash sha256:3bc5b22efbef8470f010f65b68388266e2b8f3ab62a0151900761cd12760d34f
[preflight] Running pre-flight checks
[prefligt] All pre-flight checks were passed
[karmada-agent-start] Waiting to perform the TLS Bootstrap
[karmada-agent-start] Waiting to construct karmada-agent kubeconfig
[karmada-agent-start] Waiting the necessary secret and RBAC
[karmada-agent-start] Waiting karmada-agent Deployment
W0303 13:35:48.478036   21025 check.go:52] Pod: karmada-agent-5477cc77f7-qhfk4 not ready. status: ContainerCreating
...
I0303 13:36:34.484207   21025 check.go:49] pod: karmada-agent-5477cc77f7-qhfk4 is ready. status: Running

cluster(cluster1) is joined successfully
```

回到 karmada 查看，集群已经注册上了。

```Bash
[root@lixd-tmp-1 ~]# kubectl --kubeconfig /etc/karmada/karmada-apiserver.config get clusters
NAME       VERSION   MODE   READY   AGE
cluster1   v1.23.6   Pull   True    72s
cluster2   v1.23.6   Push   True    3m
```



### 小结

- Push 模式是由 karmada 主动向 member 集群 apiserver 推送数据
- Pull 模式则是在 member 集群部署了 karmada-agent，由该 agent 主动向 karmada 拉取数据。

对比：

- 网络方面：Push 模式需要 karmada 能访问所有 member 集群 apiserver，Pull 模式需要 member 集群能访问 karmada apiserver 即可。
  - 之前的 push 模式注册的集群显示没有 Ready 就是因为无法访问导致的。
- 效率方面：当规模较大时 Push 模式由于需要想所有 member 集群推送数据，可能效率会比较低。



## 4. 应用分发

前面已经分别使用 push 和 pull 模式添加了两个集群到 karmada 了，接下来就简单体验一下 karmada 是如何管理多集群应用的。

在这之前简单给大家讲解一下 Karmada 的运行逻辑：

我们只需要把平时 apply 到 kube-apiserver 的 资源对象 全部 apply 到 karmada-apiserver，然后搭配上 karmada 中的 CRD 对象 PropagationPolicy  即可完成多集群应用管理，Karmada 就会根据 PropagationPolicy 中指定的规则，将这些资源对象分发到各个集群里。

> 比如使用 PropagationPolicy  指定将某个 deploy 分发到 cluster1 和 cluster2

同时 Karmada 还提供了 OverridePolicy CR 对象，用于在不同集群间实现差异化分发。

> 比如 cluster1 为测试集群，那么可以通过 OverridePolicy 指定将分发到 cluster1 的资源对象中的 image 字段进行修改，添加 test 标记。



### 创建 PropagationPolicy

部署之前需要先定义 PropagationPolicy 以告知 karmada 需要把对应对应的资源部署到哪些集群里。

```YAML
# propagationpolicy.yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: example-policy # The default namespace is `default`.
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx # If no namespace is specified, the namespace is inherited from the parent object scope.
  placement:
    clusterAffinity:
      clusterNames:
        - cluster1
        - cluster2
```

这个 PropagationPolicy 表示把 default namespace 下的名叫 nginx 的 deployment 部署到 cluster1 和 cluster2。

将该 PropagationPolicy  apply 到 karmada:

```Bash
# kubectl apply -f propagationpolicy.yaml
kubectl --kubeconfig /etc/karmada/karmada-apiserver.config apply -f propagationpolicy.yaml
```



### 创建 deploy

在 defualt namespace 下创建一个名为 name 的 deploy 进行测试：

```Bash
#kubectl create deployment nginx --image nginx
kubectl --kubeconfig /etc/karmada/karmada-apiserver.config create deployment nginx --image nginx
```

Karmada 就会根据 propagationpolicy 把我们刚创建的 deploy 分发到 cluster1 和 cluster2 上面。

查看 karmada 里的 deploy 状态

```Bash
[root@karmada ~]# kubectl --kubeconfig /etc/karmada/karmada-apiserver.config get deploy
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   0/1     2            0           35s
```

去各个集群查看

```Bash
[root@cluster1 ~]# kubectl get deploy
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   0/1     1            0           43s

[root@cluster2 ~]# kubectl get deploy
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   0/1     1            0           48s
```

等启动完成后再次查看 karmada 中的 deploy 状态

```Bash
[root@karmada ~]# kubectl --kubeconfig /etc/karmada/karmada-apiserver.config get deploy
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   2/1     2            2           3m
```

> cluster1 和 cluster2 都启动了一个，所以 karmada 这里显示的是 ready 2/1



### 创建 OverridePolicy

创建一个 OverridePolicy,把 cluster1 的镜像 tag 替换为 1.20 

```YAML
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: example
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
      labelSelector:
        matchLabels:
          app: nginx
  overrideRules:
    - targetCluster:
        clusterNames:
          - cluster1
      overriders:
        imageOverrider:
          - component: Tag
            operator: replace
            value: '1.20'
kubectl --kubeconfig /etc/karmada/karmada-apiserver.config apply -f op.yaml
```

然后去 cluster 查看,已经替换成功：

```Bash
[root@cluster-2 ~]# kubecyl get deploy -oyaml|grep image
        - image: nginx:1.20
          imagePullPolicy: Always
```



## 5. 小结

本文主要记录了如何部署 Karmada 以及 Karmada 的基本使用，包括集群管理和应用分发功能。

* 部署：karmada 提供了 kubectl 插件 可以一键部署

* 集群管理：有 pull 和 push 两种模式可选

* 应用分发：需要创建 PropagationPolicy 以及 OverridePolicy 对象告知 karmada 如何进行应用分发

简单来说就是以前往 kube-apiserver 发请求创建 deploy、service 等对象，使用 karmada 之后就往 karmada-apiserver 创建这些对象，然后通过分发策略告知 karmada 需要将这些资源分发到哪些集群里，然后 karmada 就会往对应集群里写入这些资源对象了。

>  关于 Karmada 原理可以期待后续文章~