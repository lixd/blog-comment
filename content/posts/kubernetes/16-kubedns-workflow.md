---
title: "Kubernetes教程(十六)---从 Service DNS 记录到 IP 地址，KubeDNS 工作原理"
description: "k8s 中的 kubedns 是如何工作的，如何将 service DNS 记录解析为 IP 地址的"
date: 2023-02-25
draft: false
categories: ["Kubernetes"]
tags: ["Kubernetes"]
---

本文主要记录了 k8s 中的 kubedns 是如何工作的，如何将 service DNS 记录解析为 IP 地址的。包括 CoreDNS 如何解析请求的，具体的数据来源以及处理逻辑，同时 Pod 又是怎么知道要把 DNS 解析请求发送给 CoreDNS 的等等问题。

<!--more-->

## 1. CoreDNS 与 KubeDNS

CoreDNS 是一个 Go 语言实现的 DNS Server，通过 **链式插件(chains plugins)** 实现的，因此用户可以很轻松的以添加插件的方式在实现自定义逻辑。同时 CoreDNS 也是一个 CNCF 的毕业项目，稳定性、可用性方面不用担心。

**什么需要 KubeDNS 组件**

Kubernetes Service 通过虚拟 IP 地址或者节点端口为用户应用提供访问入口，然而这些 IP 地址和端口是动态分配的，实际项目中无法把一个可变的入口发布出去供用户访问。为了解决这个问题，Kubernetes 提供了内置的域名服务，用户定义的服务会自动获取域名，通过域名解析，可以对外向用户提供一个固定的服务访问地址。

最初 k8s 使用自定义组件 KubDNS 来实现，不过从 K8S 1.11 开始，K8S 已经使用 CoreDNS，替换 KubeDNS 来充当其 DNS 解析的重任。

为了搞清楚 k8s 中的 KubeDNS  组件工作原理，我们需要清楚下面两个问题：

* 1）**CoreDNS 是如何解析请求的**
  * CoreDNS 怎么知道 service 域名和 clusterIP 或者 externalIP 的对应关系的，数据是从哪儿来的呢
* 2）**Pod 如何知道要把请求发给 CoreDNS**
  * Pod 里的 DNS 配置是怎么样的，又是什么时候生成的等等



首先我们需要一个 k8s 集群，单节点即可。

如果没有 k8s 环境的话可以参数这篇文章：[Kubernetes教程(十一)—使用 KubeClipper 通过一条命令快速创建 k8s 集群](https://www.lixueduan.com/posts/kubernetes/11-install-by-kubeclipper/)，快速搭建一个。



## 2. CoreDNS 是如何解析请求的

### Corefile

首先要从 CoreDNS 的配置文件，Corefile 说起。

集群搭建好后，里面会启动两个叫做 coredns 的 pod

```Bash
[root@test ~]# kubectl -n kube-system get po
NAME                                       READY   STATUS    RESTARTS        AGE
calico-kube-controllers-6c557bcb96-kxlp8   1/1     Running   0               5d19h
calico-node-rd28x                          1/1     Running   0               5d19h
coredns-84798fc4bb-8rrzz                   1/1     Running   0               4d22h
coredns-84798fc4bb-sbtdr                   1/1     Running   0               4d22h
etcd-test                                  1/1     Running   0               5d19h
kc-kubectl-7f854bbd9b-mfdl7                1/1     Running   0               5d19h
kube-apiserver-test                        1/1     Running   0               5d19h
kube-controller-manager-test               1/1     Running   9 (2d12h ago)   5d19h
kube-proxy-2zcch                           1/1     Running   0               5d19h
kube-scheduler-test                        1/1     Running   9 (2d12h ago)   5d19h
```

可以看到在 kube-system namespace 下有两个 CoreDNS pod，而配置文件则来源于 Configmap，

```Bash
[root@test ~]# kubectl -n kube-system get cm
NAME                                 DATA   AGE
calico-config                        4      5d19h
coredns                              1      5d19h
extension-apiserver-authentication   6      5d19h
kube-proxy                           2      5d19h
kube-root-ca.crt                     1      5d19h
kubeadm-config                       1      5d19h
kubelet-config-1.23                  1      5d19h
```

具体内容如下

```Bash
[root@test ~]# kubectl -n kube-system get cm  coredns -oyaml
apiVersion: v1
data:
  Corefile: |-
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 10
        loop
        reload 10s
        loadbalance
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2023-02-27T06:47:05Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "187919"
  uid: 76f7a78f-70ce-4074-9660-aa317cc7c1c1
```

其中 data.Corefile 字段就是 CoreDNS 的配置文件，

```Corefile
.:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 10
        loop
        reload 10s
        loadbalance
    }
```

具体内容大致解释一下：

- `.:53` 表示在 53 端口监听服务,`.` 表示任意的域名都由该服务来处理
  - 可以启动多个服务，因此需要用域名来区分，比如`a.com:53`、`b.com:5300` 就分别在不同端口启动了两个服务，每个服务只会处理各自域名下的 DNS 请求

后续大括号里的就是启动的插件，前面说过 CoreDNS 是链式插件组成的，而具体启动哪些插件则是由配置文件决定的。

比如 errors 则表示开启 errors 插件，用于记录错误，health 则开启健康检测插件，其他插件同理。

**最重要的是 kubernetes 这个，开启了一个叫做 kubernetes 的插件。** 

这个就是 k8s 官方维护的一个 CoreDNS 插件，就是依靠该插件 CoreDNS 才实现了 KubeDNS 需要的功能，因此自然需要分析一下这个插件做了什么。

>  该插件源码存放在 coredns 仓库里，具体见[plugin/kubernetes]( https://github.com/coredns/coredns/tree/master/plugin/kubernetes)



### Kubernetes 插件做了什么

CoreDNS 正是通过 kubernetes 插件实现了解析 k8s 集群内域名的功能。那么我们看下这个插件做了些什么事情。

> 剧透一下：该插件主要调用 k8s api 获取 service、endpoint 等信息，然后组装成 dns 格式数据用以处理客户端 dns 解析请求。

#### 插件初始化

插件的初始化方法如下：

```Go
// plugin/kubernetes/setup.go#33
func setup(c *caddy.Controller) error {
   // Do not call klog.InitFlags(nil) here.  It will cause reload to panic.
   klog.SetOutput(os.Stdout)
   // 检查了 corefile 中 kubernetes 配置的定义，并配置了一些缺省值
   k, err := kubernetesParse(c)
   if err != nil {
      return plugin.Error(pluginName, err)
   }
    // 启动了对 pod, service, endpoint 三种资源增、删、改的 watch，并注册了一些回调
    // 注意：pod 是否启动 watch 是根据配置文件中 pod 的值来决定的，如果值不是 verified 就不会启动 pod 的 watch
    // 这里的 watch 方法观测到变化后，仅仅只改变 dns.modified 这个值，它会将该值设置为当前时间戳
   onStart, onShut, err := k.InitKubeCache(context.Background())
   if err != nil {
      return plugin.Error(pluginName, err)
   }
   if onStart != nil {
      c.OnStartup(onStart)
   }
   if onShut != nil {
      c.OnShutdown(onShut)
   }
   // 添加插件到插件链里
   dnsserver.GetConfig(c).AddPlugin(func(next plugin.Handler) plugin.Handler {
      k.Next = next
      return k
   })

   // get locally bound addresses
   c.OnStartup(func() error {
      k.localIPs = boundIPs(c)
      return nil
   })

   return nil
}
```

*以下这三个方法就是 watch 资源时的回调*

```Go
//plugin/kubernetes/controller.go#563
func (dns *dnsControl) Add(obj interface{})               { dns.updateModified() }
func (dns *dnsControl) Delete(obj interface{})            { dns.updateModified() }
func (dns *dnsControl) Update(oldObj, newObj interface{}) { dns.detectChanges(oldObj, newObj) }

// updateModified set dns.modified to the current time.
func (dns *dnsControl) updateModified() {
   unix := time.Now().Unix()
   atomic.StoreInt64(&dns.modified, unix)
}

// updateExtModified set dns.extModified to the current time.
func (dns *dnsControl) updateExtModifed() {
   unix := time.Now().Unix()
   atomic.StoreInt64(&dns.extModified, unix)
}

// detectChanges detects changes in objects, and updates the modified timestamp
func (dns *dnsControl) detectChanges(oldObj, newObj interface{}) {
   // If both objects have the same resource version, they are identical.
   if newObj != nil && oldObj != nil && (oldObj.(meta.Object).GetResourceVersion() == newObj.(meta.Object).GetResourceVersion()) {
      return
   }
   obj := newObj
   if obj == nil {
      obj = oldObj
   }
   switch ob := obj.(type) {
   case *object.Service:
      imod, emod := serviceModified(oldObj, newObj)
      if imod {
         dns.updateModified()
      }
      if emod {
         dns.updateExtModifed()
      }
   case *object.Pod:
      dns.updateModified()
   case *object.Endpoints:
      if !endpointsEquivalent(oldObj.(*object.Endpoints), newObj.(*object.Endpoints)) {
         dns.updateModified()
      }
   default:
      log.Warningf("Updates for %T not supported.", ob)
   }
}
```

可以看到，不管是 add、update 还是 delete，最终都是调用 updateModified 或者 updateExtModifed 方法来修改对应字段，并没有把新的 svc 或者 endpoint 这些记录下来。

按照正常情况不是应该 watch 变化，然后在内存里维护一份数据，用于在外部 DNS 请求时查询吗。

实际上这里也存了，不过是**复用了** **`client-go`** **中的** **`informer`** **机制**，informer 会在内存中存储 watch 的对象，因此 service、endpoint、pod 这些数据内存里都是有的，插件这边就不用额外存一份了。

> 准确的说是 client-go 中的 Indexer 组件，它已经将所有 watch 的对象在本地内存中存储了一份。

#### 如何处理解析请求

插件里使用 Services 方法来处理所有接收到的 DNS 请求：

```Go
// plugin/kubernetes/kubernetes.go#95
func (k *Kubernetes) Services(ctx context.Context, state request.Request, exact bool, opt plugin.Options) (svcs []msg.Service, err error) {
   // We're looking again at types, which we've already done in ServeDNS, but there are some types k8s just can't answer.
   switch state.QType() {
   // 如果是插件 DNS 服务版本的话就返回编译时的版本号
   case dns.TypeTXT:
      // 1 label + zone, label must be "dns-version".
      t, _ := dnsutil.TrimZone(state.Name(), state.Zone)

      segs := dns.SplitDomainName(t)
      if len(segs) != 1 {
         return nil, nil
      }
      if segs[0] != "dns-version" {
         return nil, nil
      }
      svc := msg.Service{Text: DNSSchemaVersion, TTL: 28800, Key: msg.Path(state.QName(), coredns)}
      return []msg.Service{svc}, nil
   // 如果查询的是 NS 类型记录，就返回 CoreDNS service 自身在集群里的记录
   case dns.TypeNS:
      // We can only get here if the qname equals the zone, see ServeDNS in handler.go.
      nss := k.nsAddrs(false, state.Zone)
      var svcs []msg.Service
      for _, ns := range nss {
         if ns.Header().Rrtype == dns.TypeA {
            svcs = append(svcs, msg.Service{Host: ns.(*dns.A).A.String(), Key: msg.Path(ns.Header().Name, coredns), TTL: k.ttl})
            continue
         }
         if ns.Header().Rrtype == dns.TypeAAAA {
            svcs = append(svcs, msg.Service{Host: ns.(*dns.AAAA).AAAA.String(), Key: msg.Path(ns.Header().Name, coredns), TTL: k.ttl})
         }
      }
      return svcs, nil
   }

   if isDefaultNS(state.Name(), state.Zone) {
      nss := k.nsAddrs(false, state.Zone)
      var svcs []msg.Service
      for _, ns := range nss {
         if ns.Header().Rrtype == dns.TypeA && state.QType() == dns.TypeA {
            svcs = append(svcs, msg.Service{Host: ns.(*dns.A).A.String(), Key: msg.Path(state.QName(), coredns), TTL: k.ttl})
            continue
         }
         if ns.Header().Rrtype == dns.TypeAAAA && state.QType() == dns.TypeAAAA {
            svcs = append(svcs, msg.Service{Host: ns.(*dns.AAAA).AAAA.String(), Key: msg.Path(state.QName(), coredns), TTL: k.ttl})
         }
      }
      return svcs, nil
   }
   // 到这里才正式开始解析 DNS 请求
   s, e := k.Records(ctx, state, false)

   // SRV for external services is not yet implemented, so remove those records.

   if state.QType() != dns.TypeSRV {
      return s, e
   }

   internal := []msg.Service{}
   for _, svc := range s {
      if t, _ := svc.HostType(); t != dns.TypeCNAME {
         internal = append(internal, svc)
      }
   }

   return internal, e
}
```

Record 方法如下：

```Go
// plugin/kubernetes/kubernetes.go#370
func (k *Kubernetes) Records(ctx context.Context, state request.Request, exact bool) ([]msg.Service, error) {
   // 将请求数据解析为service、namespace 之类的格式
   r, e := parseRequest(state.Name(), state.Zone)
   if e != nil {
      return nil, e
   }
   // 数据异常直接返回
   if r.podOrSvc == "" {
      return nil, nil
   }

   if dnsutil.IsReverse(state.Name()) > 0 {
      return nil, errNoItems
   }

   if !k.namespaceExposed(r.namespace) {
      return nil, errNsNotExposed
   }
   // 查询 pod 
   if r.podOrSvc == Pod {
      pods, err := k.findPods(r, state.Zone)
      return pods, err
   }
   // 查询 svc
   services, err := k.findServices(r, state.Zone)
   return services, err
}
```



可以看到，record方法中根据请求是查询 pod 还是查询 service 分别调用了不同的方法。

pod 的查询方法如下：

```Go
// plugin/kubernetes/kubernetes.go#412
func (k *Kubernetes) findPods(r recordRequest, zone string) (pods []msg.Service, err error) {
   if k.podMode == podModeDisabled {
      return nil, errNoItems
   }

   namespace := r.namespace
   if !k.namespaceExposed(namespace) {
      return nil, errNoItems
   }

   podname := r.service

   // handle empty pod name
   if podname == "" {
      if k.namespaceExposed(namespace) {
         // NODATA
         return nil, nil
      }
      // NXDOMAIN
      return nil, errNoItems
   }

   zonePath := msg.Path(zone, coredns)
   ip := ""
   if strings.Count(podname, "-") == 3 && !strings.Contains(podname, "--") {
      ip = strings.ReplaceAll(podname, "-", ".")
   } else {
      ip = strings.ReplaceAll(podname, "-", ":")
   }

   if k.podMode == podModeInsecure {
      if !k.namespaceExposed(namespace) { // namespace does not exist
         return nil, errNoItems
      }

      // If ip does not parse as an IP address, we return an error, otherwise we assume a CNAME and will try to resolve it in backend_lookup.go
      if net.ParseIP(ip) == nil {
         return nil, errNoItems
      }

      return []msg.Service{{Key: strings.Join([]string{zonePath, Pod, namespace, podname}, "/"), Host: ip, TTL: k.ttl}}, err
   }

   // PodModeVerified
   err = errNoItems

   for _, p := range k.APIConn.PodIndex(ip) {
      // check for matching ip and namespace
      if ip == p.PodIP && match(namespace, p.Namespace) {
         s := msg.Service{Key: strings.Join([]string{zonePath, Pod, namespace, podname}, "/"), Host: ip, TTL: k.ttl}
         pods = append(pods, s)

         err = nil
      }
   }
   return pods, err
}
```

service 查询方法如下

```go

// findServices returns the services matching r from the cache.
func (k *Kubernetes) findServices(r recordRequest, zone string) (services []msg.Service, err error) {
   if !k.namespaceExposed(r.namespace) {
      return nil, errNoItems
   }

   // handle empty service name
   if r.service == "" {
      if k.namespaceExposed(r.namespace) {
         // NODATA
         return nil, nil
      }
      // NXDOMAIN
      return nil, errNoItems
   }

   err = errNoItems

   var (
      endpointsListFunc func() []*object.Endpoints
      endpointsList     []*object.Endpoints
      serviceList       []*object.Service
   )

   idx := object.ServiceKey(r.service, r.namespace)
   serviceList = k.APIConn.SvcIndex(idx)
   endpointsListFunc = func() []*object.Endpoints { return k.APIConn.EpIndex(idx) }

   zonePath := msg.Path(zone, coredns)
   for _, svc := range serviceList {
      if !(match(r.namespace, svc.Namespace) && match(r.service, svc.Name)) {
         continue
      }

      // If "ignore empty_service" option is set and no endpoints exist, return NXDOMAIN unless
      // it's a headless or externalName service (covered below).
      if k.opts.ignoreEmptyService && svc.Type != api.ServiceTypeExternalName && !svc.Headless() { // serve NXDOMAIN if no endpoint is able to answer
         podsCount := 0
         for _, ep := range endpointsListFunc() {
            for _, eps := range ep.Subsets {
               podsCount += len(eps.Addresses)
            }
         }

         if podsCount == 0 {
            continue
         }
      }

      // External service
      if svc.Type == api.ServiceTypeExternalName {
         s := msg.Service{Key: strings.Join([]string{zonePath, Svc, svc.Namespace, svc.Name}, "/"), Host: svc.ExternalName, TTL: k.ttl}
         if t, _ := s.HostType(); t == dns.TypeCNAME {
            s.Key = strings.Join([]string{zonePath, Svc, svc.Namespace, svc.Name}, "/")
            services = append(services, s)

            err = nil
         }
         continue
      }

      // Endpoint query or headless service
      if svc.Headless() || r.endpoint != "" {
         if endpointsList == nil {
            endpointsList = endpointsListFunc()
         }

         for _, ep := range endpointsList {
            if object.EndpointsKey(svc.Name, svc.Namespace) != ep.Index {
               continue
            }

            for _, eps := range ep.Subsets {
               for _, addr := range eps.Addresses {

                  // See comments in parse.go parseRequest about the endpoint handling.
                  if r.endpoint != "" {
                     if !match(r.endpoint, endpointHostname(addr, k.endpointNameMode)) {
                        continue
                     }
                  }

                  for _, p := range eps.Ports {
                     if !(matchPortAndProtocol(r.port, p.Name, r.protocol, p.Protocol)) {
                        continue
                     }
                     s := msg.Service{Host: addr.IP, Port: int(p.Port), TTL: k.ttl}
                     s.Key = strings.Join([]string{zonePath, Svc, svc.Namespace, svc.Name, endpointHostname(addr, k.endpointNameMode)}, "/")

                     err = nil

                     services = append(services, s)
                  }
               }
            }
         }
         continue
      }

      // ClusterIP service
      for _, p := range svc.Ports {
         if !(matchPortAndProtocol(r.port, p.Name, r.protocol, string(p.Protocol))) {
            continue
         }

         err = nil

         for _, ip := range svc.ClusterIPs {
            s := msg.Service{Host: ip, Port: int(p.Port), TTL: k.ttl}
            s.Key = strings.Join([]string{zonePath, Svc, svc.Namespace, svc.Name}, "/")
            services = append(services, s)
         }
      }
   }
   return services, err
}
```



可以看到 pod 和 service 逻辑其实差不多，最终都是去  `client-go` 的 `Indexer` 缓存里查数据并返回，比如 service 是这样从 Indexer 里拿数据的：

```Go
func (dns *dnsControl) SvcIndex(idx string) (svcs []*object.Service) {
   os, err := dns.svcLister.ByIndex(svcNameNamespaceIndex, idx)
   if err != nil {
      return nil
   }
   for _, o := range os {
      s, ok := o.(*object.Service)
      if !ok {
         continue
      }
      svcs = append(svcs, s)
   }
   return svcs
}
```

拿到数据后就组装成 DNS resp 格式的数据返回即可。

客户端请求数据里面包含了 serviceName、namespace 等信息，Kubernetes 插件就根据这些信息去 Indexer 中查询拿到对应的 endpoint 组装后返回即可。

> 这也解释了为什么 service 的 dns 记录必须要按照 `<podName>.<serviceName>.<namesapce>.svc.cluster.local ` 这个规范来，因为 coredns 这边就是这样解析的。
>
> 只有按照该规范请求过来，Kubernetes 插件才能从中提取出 podName、serviceName、namespace 等信息，然后才能拿到具体的数据返回。



### 小结

Kubernetes 插件主要在插件启动时启动了 servide、endpoint、pod（根据配置文件可选）的 Controller，然后**复用了 client-go 中的 informer 机制以缓存数据**。

处理 DNS 解析请求时，首先根据请求数据解析出 service、namespace、pod 等信息，然后**从 client-go 中的 Indexer 本地缓存里拿数据组装后响应给客户端**。



## 3. Pod 如何知道要把请求发给 CoreDNS

### Pod DNSConfig

首先我们看一下 pod 里的 dns 配置。

先随便启动一个 pod

```Bash
kubectl create deployment nginx --image nginx
```

等到 pod 启动后，用 exec 命令看一下 pod 里的 dnsConfig：

```Bash
[root@test ~]# k get po
NAME                     READY   STATUS    RESTARTS   AGE
nginx-85b98978db-mttqh   1/1     Running   0          77s
[root@test ~]# kubectl exec -it nginx-85b98978db-mttqh -- cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

可以看到 pod 的 namaserver 配置的是 10.96.0.10,，而这个 IP 正好就是 CoreDNS Service 的 ClusterIP

```Bash
[root@test ~]# kubectl -n kube-system get svc kube-dns
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                                    AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   5d20h
```

也就是说根据这个配置，pod 中的 dns 请求会发给 CoreDNS。

那么问题来了，这个配置是哪儿来的呢？

### 配置来源

#### dnsPolicy



实际上 Pod 都可以指定一个叫做 [dnsPolicy](https://kubernetes.io/zh-cn/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy) 的字段，用于配置 Pod 里的 DNS 策略。

DNS 策略可以逐个 Pod 来设定。目前 Kubernetes 支持以下特定 Pod 的 DNS 策略。 这些策略可以在 Pod 规约中的 `dnsPolicy` 字段设置：

- "`Default`": Pod 从运行所在的节点继承名称解析配置。
- "`ClusterFirst`": 与配置的集群域后缀不匹配的任何 DNS 查询（例如 "www.kubernetes.io"） 都会由 DNS 服务器转发到上游名称服务器。
- "`ClusterFirstWithHostNet`": 对于以 hostNetwork 方式运行的 Pod，应将其 DNS 策略显式设置为 "`ClusterFirstWithHostNet`"。否则，以 hostNetwork 方式和 `"ClusterFirst"` 策略运行的 Pod 将会做出回退至 `"Default"` 策略的行为。
- "`None`": 此设置允许 Pod 忽略 Kubernetes 环境中的 DNS 设置。Pod 会使用其 `dnsConfig` 字段所提供的 DNS 设置。 

#### dnsConfig

pod 的 DNS 配置可让用户对 Pod 的 DNS 设置进行更多控制。

`dnsConfig` 字段是可选的，它可以与任何 `dnsPolicy` 设置一起使用。 但是，当 Pod 的 `dnsPolicy` 设置为 "`None`" 时，必须指定 `dnsConfig` 字段。

用户可以在 `dnsConfig` 字段中指定以下属性：

- `nameservers`：将用作于 Pod 的 DNS 服务器的 IP 地址列表。 最多可以指定 3 个 IP 地址。当 Pod 的 `dnsPolicy` 设置为 "`None`" 时， 列表必须至少包含一个 IP 地址，否则此属性是可选的。 所列出的服务器将合并到从指定的 DNS 策略生成的基本名称服务器，并删除重复的地址。
- `searches`：用于在 Pod 中查找主机名的 DNS 搜索域的列表。此属性是可选的。 指定此属性时，所提供的列表将合并到根据所选 DNS 策略生成的基本搜索域名中。 重复的域名将被删除。Kubernetes 最多允许 6 个搜索域。
- `options`：可选的对象列表，其中每个对象可能具有 `name` 属性（必需）和 `value` 属性（可选）。 此属性中的内容将合并到从指定的 DNS 策略生成的选项。 重复的条目将被删除。

Demo 如下：

```YAML
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 192.0.2.1 # 这是一个示例
    searches:
      - ns1.svc.cluster-domain.example
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

**也就是说只要我们将 dnsPolicy 配置为 None，然后就通过 dnsConfig 来自定义 pod 里的 /etc/resolv.conf 文件内容了。**

当然了，一般这个我们都是用的默认值，不会去配置的。也就是说刚才我们在 nginx pod 里看到的数据是默认值，那么这些默认值是哪儿来的呢？

### Kubelet

实际上是 kubelet 帮我们配置的。

查看一下 kubelet 的配置文件：

```Bash
[root@test ~]# cat /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
cgroupDriver: systemd
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
kind: KubeletConfiguration
logging:
  flushFrequency: 0
  options:
    json:
      infoBufferSize: "0"
  verbosity: 0
memorySwap: {}
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
rotateCertificates: true
runtimeRequestTimeout: 0s
shutdownGracePeriod: 0s
shutdownGracePeriodCriticalPods: 0s
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 0s
syncFrequency: 3s
volumeStatsAggPeriod: 1m0s
```

里面正好有一个叫做 clusterDNS 的字段，值刚好也是 10.96.0.10。

到这里基本可以证实我们的猜想了，pod 里的这个值是由 kubelet 配置的，不过还是翻下源码确认一下。

相关代码如下：

```Go
// pkg/kubelet/network/dns/dns.go#430
// SetupDNSinContainerizedMounter replaces the nameserver in containerized-mounter's rootfs/etc/resolv.conf with kubelet.ClusterDNS
func (c *Configurer) SetupDNSinContainerizedMounter(mounterPath string) {
   resolvePath := filepath.Join(strings.TrimSuffix(mounterPath, "/mounter"), "rootfs", "etc", "resolv.conf")
   dnsString := ""
   for _, dns := range c.clusterDNS {
      dnsString = dnsString + fmt.Sprintf("nameserver %s\n", dns)
   }
   if c.ResolverConfig != "" {
      f, err := os.Open(c.ResolverConfig)
      if err != nil {
         klog.ErrorS(err, "Could not open resolverConf file")
      } else {
         defer f.Close()
         _, hostSearch, _, err := parseResolvConf(f)
         if err != nil {
            klog.ErrorS(err, "Error for parsing the resolv.conf file")
         } else {
            dnsString = dnsString + "search"
            for _, search := range hostSearch {
               dnsString = dnsString + fmt.Sprintf(" %s", search)
            }
            dnsString = dnsString + "\n"
         }
      }
   }
   if err := ioutil.WriteFile(resolvePath, []byte(dnsString), 0600); err != nil {
      klog.ErrorS(err, "Could not write dns nameserver in the file", "path", resolvePath)
   }
}
```

内容也很简单，就是在启动 pod，mount rootfs 的时候，根据 kubelet 的 config 内容生成一个 /etc/resolv.conf 并替换进去。

至此，pod 中的 DNS 请求为什么会发给 CoreDNS 就很清晰了。



### 为什么是 10.96.0.10

不过可能大家还有一个疑问，怎么 coredns 的 clusterIP 每次都是 10.96.0.10 呢。实际上这只是 kubeadm 里的一个默认值罢了。

> 因为现在大家都是使用 kubeadm 来搭的集群，所以创建出来发现 coredns 的 clusterIP 每次都是 10.96.0.10 。

```Go
// cmd/kubeadm/app/componentconfigs/kubelet.go#129
func (kc *kubeletConfig) Default(cfg *kubeadmapi.ClusterConfiguration, _ *kubeadmapi.APIEndpoint, nodeRegOpts *kubeadmapi.NodeRegistrationOptions) {
   const kind = "KubeletConfiguration"

   if kc.config.FeatureGates == nil {
      kc.config.FeatureGates = map[string]bool{}
   }

   if kc.config.StaticPodPath == "" {
      kc.config.StaticPodPath = kubeadmapiv1.DefaultManifestsDir
   } else if kc.config.StaticPodPath != kubeadmapiv1.DefaultManifestsDir {
      warnDefaultComponentConfigValue(kind, "staticPodPath", kubeadmapiv1.DefaultManifestsDir, kc.config.StaticPodPath)
   }

   clusterDNS := ""
   dnsIP, err := constants.GetDNSIP(cfg.Networking.ServiceSubnet)
   if err != nil {
      clusterDNS = kubeadmapiv1.DefaultClusterDNSIP
   } else {
      clusterDNS = dnsIP.String()
   }

   if kc.config.ClusterDNS == nil {
      kc.config.ClusterDNS = []string{clusterDNS}
   } else if len(kc.config.ClusterDNS) != 1 || kc.config.ClusterDNS[0] != clusterDNS {
      warnDefaultComponentConfigValue(kind, "clusterDNS", []string{clusterDNS}, kc.config.ClusterDNS)
   }
   
   if kc.config.ClusterDomain == "" {
   kc.config.ClusterDomain = cfg.Networking.DNSDomain
   } else if cfg.Networking.DNSDomain != "" && kc.config.ClusterDomain != cfg.Networking.DNSDomain {
   warnDefaultComponentConfigValue(kind, "clusterDomain", cfg.Networking.DNSDomain, kc.config.ClusterDomain)
}
   // ....省略 
 }
```

如果我们指定了 service 子网时，kubeadm 会根据子网拿到第一个 IP 作为 coreDNS 的 clusterIP，默认子网就是 10.96.0.10/12 ，所以我们拿到的 IP 也是 10.96.0.10。如果解析失败那就直接用 10.96.0.10：

```Go
DefaultServicesSubnet = "10.96.0.0/12"
DefaultClusterDNSIP = "10.96.0.10"
```



## 4. 小结

相信到这里的时候，文章开头的两个问题大家也有了答案：

* **1）CoreDNS 是如何解析请求的**
  *  CoreDNS 收到的请求由 kubernetes 插件处理
  * 数据来源：该插件借助 client-go 中的 Informer 机制，将所有 service、endpoint、pod 缓存到本地的 Indexer 缓存里
  * 处理逻辑：DNS 请求时客户端按照`<podName>.<serviceName>.<namesapce>.svc.cluster.local`规范发起请求，Kubernetes插件则按照该规范解析出 podName、serviceName、namespace 等信息，然后 从 Indexer 本地缓存里查询到具体数据并返回结果给客户端

* **2）Pod 中的 DNS 配置，Pod 怎么知道该向 CoreDNS 发送解析请求**

  * pod 里的 /etc/resolv.conf 中 nameserver 指向 10.96.0.10（coreDNS 默认 clusterIP），因此 pod 中的 DNS 请求会发送给 CoreDNS

  * pod 中的 /etc/resolv.conf 由 kubelet 启动 pod 时根据 clusterDNS 字段生成

  * Kubelet 配置中的 clusterDNS 字段来源于 kubelet 配置文件 /var/lib/kubelet/config.yaml

  * /var/lib/kubelet/config.yaml 配置文件由 kubeadm 生成
  * kubeadm 中的 clusterDNS 地址则是从 service 子网提取出来的，如果用户在 kubeadm init 时没有指定则使用默认值 `10.96.0.10/12`,此时提取到的 clusterDNS 就是 10.96.0.10。
    * 如果是用别的方式搭建的集群，这里可能会有所不同，不过最终都是根据 service 子网提取一个 ip 写入 kubelet 配置文件
    * 至于为什么访问 10.96.0.10 就访问到 CoreDNS 这个就是 service  的内容了,具体可以参考这篇文章 [Kubernetes教程(四)---Service核心原理](https://www.lixueduan.com/posts/kubernetes/04-service-core)



整个流程大致如下：

![](../../../img/kubernetes/dns/k8s-pod-dns-workflow.png)



