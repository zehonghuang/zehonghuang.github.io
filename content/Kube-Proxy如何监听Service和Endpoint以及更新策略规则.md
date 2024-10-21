+++
title = '【深度解析】Kube Proxy如何监听Service和Endpoint以及更新策略规则'
date = 2022-08-13T22:36:08+08:00
draft = false
categories = [
    "Kubernetes",
    "深度解析",
]
weight = 3
+++

本文会探讨Kubernetes另一个核心网络组件Kube-Proxy，它承担着Service及其后端Pod对宿主机配置的影响。

## 一、 先聊一下三个核心API

这三个API在不同版本下的Kube-Proxy发挥着主要作用，尤其是后两者。先上几个三个资源的常用配置一步步展开。

### 1. Service

`Service`作用核心其实就是提供一个ClusterIP和域名的配置容器以及EP/EPS组织，并没有一个单独的controller进行管理，而是被endpointslice-controller所控制。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
  labels:
    app: my-app
  annotations:
    description: "This is a demo service"
spec:
  selector:
    app: my-app
    tier: backend
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      name: http
  ## 粘性会话，一般用于长连接并且开启ClusterIP的场景
  sessionAffinity: None
  ## Cluster和Local，默认前者，后者用于本地流量请求
  ## 例如说，在Node1的PodA请求ServiceB，只能路由到Node1的ServiceB的Pod地址，如果本地没有则无法请求，报文会被iptables drop掉
  externalTrafficPolicy: Cluster
  ## 控制NotReady的Pod是否要包含进Service，为true的话有可能导致流量损失
  publishNotReadyAddresses: false
  ipFamilyPolicy: SingleStack
  ipFamilies:
    - IPv4
  healthCheckNodePort: 30009
```


### 2. Endpoints、EndpointSlice

`EndpointSlice`是在k8s 1.19版本开始默认支持的，相较于`Endpoints`能支持更大规模部署，受限于etcd的value值大小，ep即一个Service只能部署6000个节点，
而eps理论上可以无上限，并且eps支持网络拓扑，它可以根据集群节点的资源信息，按需部署Pod数量。具体文档在这[K8s文档-EndpointSlice](https://kubernetes.io/zh-cn/docs/concepts/services-networking/endpoint-slices/)

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  annotations:
    ## 每一次Service或Pod改动都会update这个字段
    endpoints.kubernetes.io/last-change-trigger-time: "2022-08-19T03:32:20Z"
  generateName: python-server-headless-
  generation: 27
  labels:
    endpointslice.kubernetes.io/managed-by: endpointslice-controller.k8s.io
    kubernetes.io/service-name: python-server-headless
    service.kubernetes.io/headless: ""
  name: python-server-headless-5xvm5
  namespace: net-test
  ownerReferences:
    - apiVersion: v1
      blockOwnerDeletion: true
      controller: true
      kind: Service
      name: python-server-headless
      uid: fb7d24b3-c7cb-4b1a-8bc2-7caf2cb32101
  resourceVersion: "6538663"
  uid: d7475629-ac4f-46bc-9aeb-479417162ec3
addressType: IPv4
endpoints:
- addresses:
  - 10.100.58.221
  conditions:
    ready: true
    serving: true
    terminating: false
  nodeName: k8s-node02
  ## 控制器基本上靠着这些信息对eps进行增删改查
  targetRef:
    kind: Pod
    name: python-server-65b886d59c-nnktx
    namespace: net-test
    uid: b3cf8828-5311-44cc-b9b1-6e8165f793f4
ports:
  - name: ""
    port: 80
    protocol: TCP
```

## 二、Kube-Proxy的源码分析

首先我们还是需要先认识一下Kube-Proxy的整体架构：

- cmd/kube-proxy/app/server.go 这是程序主入口，对宿主机的参数和启动参数进行注入
	- cmd/kube-proxy/app/server_linux.go 对应不同编译平台的代码，这里会继续设置部分协议栈的一些配置，如nf_conntrack、iptables等
		- 这里就会应用到pkg/proxy中的各种 Proxy 创建器，根据不同设置有对应的代理
          - pkg/proxy/iptables/proxier.go 
          - pkg/proxy/ipvs/proxier.go
		- pkg/util/async/bounded_frequency_runner.go 这是控制更新iptables/ipvs的核心组件，可以自定义更新的频率窗口
        - 两个资源的同步器，被绑定在infomer的回调函数上，用来存储监听到的缓存
          - pkg/proxy/endpointschangetracker.go
          - pkg/proxy/servicechangetracker.go

下面是kube-proxy的代码主流程，没有太复杂的内容

![kube-proxy.png](/images/kube-proxy.png)
<!--more-->

### 1. Kube-Proxy的相关配置

config还是比较有价值的，能了解到iptables、conntrack的核心配置有哪些，常用几个选项除了Mode就是SyncPeriod相关的

```go
package config

type KubeProxyConfiguration struct {
	/**
	 *  1. 设置masqueradeAll，即所有发送到Service的请求都做SNAT
	 *  2. oom_score_adj，默认值是-999，也就是在系统内存非常低的情况下，被OOMKiller的优先级最低
	 *  3. 几个比较重要的conntrack相关的系统参数
	 *      net.netfilter.nf_conntrack_tcp_timeout_established
	 *      net.netfilter.nf_conntrack_tcp_timeout_close_wait
	 *      net.netfilter.nf_conntrack_tcp_be_liberal
	 */
	Linux KubeProxyLinuxConfiguration
    // 开启一些实验性功能
	FeatureGates map[string]bool
	// client-go配置
	ClientConnection componentbaseconfig.ClientConnectionConfiguration

	HostnameOverride string
	BindAddress string

	// mode specifies which proxy mode to use.
	Mode ProxyMode
	/**
	 *  1. LocalhostNodePorts, false不允许回环地址访问NodePort，仅在iptables&ipv4生效
	    2. 
	 */
	IPTables KubeProxyIPTablesConfiguration
	// ipvs contains ipvs-related configuration options.
	IPVS KubeProxyIPVSConfiguration
	// nftables contains nftables-related configuration options.
	NFTables KubeProxyNFTablesConfiguration

	// 检测本地流量的模式，默认通过ClusterCIDR
	DetectLocalMode LocalMode
	// 网桥名称、ClusterCIDR、设备前缀等设置
	DetectLocal DetectLocalConfiguration

	// 指定哪些CIDR访问能使用NodePort暴露端口，比如数据库或特定的服务
	NodePortAddresses []string
	// 下面两个参数都应用在BoundedFrequencyRunner定时器中，Kube-Proxy更新的最高频率受限于MinSyncPeriod和内部burstRuns两个参数
	// 这是Proxy规则同步的最大间隔
	SyncPeriod metav1.Duration
    // 最小间隔
	MinSyncPeriod metav1.Duration
	ConfigSyncPeriod metav1.Duration
}
```

### 1.1 BoundedFrequencyRunner

正如名称所说，这是一个有边界频率的任务执行器，会限制runner执行的qps，无法高频更新配置。

```go
func NewBoundedFrequencyRunner(name string, fn func(), minInterval, maxInterval time.Duration, burstRuns int) *BoundedFrequencyRunner {
	// fn: 执行方法
    // minInterval和burstRuns用于控制runner最高qps是多少，超过会合并至下一个interval执行。该值允许为0
	// maxInterval则是最大周期，离上一次更新满足这个区间会自动tick
	timer := &realTimer{timer: time.NewTimer(0)} // will tick immediately
	<-timer.C()                                  // consume the first tick
	return construct(name, fn, minInterval, maxInterval, burstRuns, timer)
}

// Make an instance with dependencies injected.
func construct(name string, fn func(), minInterval, maxInterval time.Duration, burstRuns int, timer timer) *BoundedFrequencyRunner {
	//...
	// bfr := &BoundedFrequencyRunner{ ...
	
	if minInterval == 0 {
		bfr.limiter = nullLimiter{}
	} else {
		// allow burst updates in short succession
		qps := float32(time.Second) / float32(minInterval)
		// flowcontrol是client-go中专门用于流量控制的工具库，提供了几种令牌实现工具
		// 具体路径staging/src/k8s.io/client-go/util/flowcontrol/throttle.go
		bfr.limiter = flowcontrol.NewTokenBucketRateLimiterWithClock(qps, burstRuns, timer)
	}
	return bfr
}
```

### 2. Kube-Proxy核心，Proxier实现类

每个interface都有个XXXSynced()的方法，写过控制器的都知道`cache.WaitForNamedCacheSync()`，其实是为了infomer.start()后第一次拉去apiserver准备的，同步本地缓存后才执行

![Proxier继承图.png](/images/Proxier基础图.png)

### 2.1 Informer回调方法

下面几个接口都在`pkg/proxy/config/config.go`
```go
type NodeHandler interface{}
type ServiceHandler interface{
    // 这个是Service监听回调的核心接口
    OnServiceUpdate(oldService, service *v1.Service)
}
type EndpointSliceHandler interface{}
// 这个是监听Service ClusterIP的分配区间
type ServiceCIDRHandler interface{}
```
这里没什么好说的，只是调用链比较隐晦，都是通过同一个go文件下`RegisterEventHandler`方法注册进来的

### 2.2 Service、EndpointSlice的变化跟踪器

`ServiceChangeTracker`唯一对外的方法就只有Update，功能就是更新成员变量`item`，并且告知调用方是否有Service更新。

```go
// 一方三用：previous空为add，current空为del，两个都有为update
func (sct *ServiceChangeTracker) Update(previous, current *v1.Service) bool {
	if previous == nil && current == nil {
		return false
	}

	svc := current
	if svc == nil {
		svc = previous
	}
	metrics.ServiceChangesTotal.Inc()
	namespacedName := types.NamespacedName{Namespace: svc.Namespace, Name: svc.Name}

	sct.lock.Lock()
	defer sct.lock.Unlock()

	change, exists := sct.items[namespacedName]
	if !exists {
		change = &serviceChange{}
		// 把Service声明不同端口的配置，映射成多个Service
		change.previous = sct.serviceToServiceMap(previous)
		sct.items[namespacedName] = change
	}
	change.current = sct.serviceToServiceMap(current)
	// 相等意味着Service没有变化
	// 不想等，一般来说就是更新了port配置
	if reflect.DeepEqual(change.previous, change.current) {
		delete(sct.items, namespacedName)
	} else {
		klog.V(4).InfoS("Service updated ports", "service", klog.KObj(svc), "portCount", len(change.current))
	}
	metrics.ServiceChangesPending.Set(float64(len(sct.items)))
	return len(sct.items) > 0
}
```

`EndpointsChangeTracker`的数据结构明显复杂多了，但基本上围绕着`trackerByServiceMap`展开

```go
func (ect *EndpointsChangeTracker) EndpointSliceUpdate(endpointSlice *discovery.EndpointSlice, removeSlice bool) bool {
	// ...
	namespacedName, _, err := endpointSliceCacheKeys(endpointSlice)
	// ...
	ect.lock.Lock()
	defer ect.lock.Unlock()

	// 这里的比对现有cache的缓存，是否最新版本
	changeNeeded := ect.endpointSliceCache.updatePending(endpointSlice, removeSlice)

	if changeNeeded {
		metrics.EndpointChangesPending.Inc()
		// lastChangeTriggerTime仅用于metrics:kubeproxy_network_programming_duration_seconds记录消耗时间
		// 从Pod的IP地址被增删改查的时刻，到完成所有rule被同步的时长
		if removeSlice {
			delete(ect.lastChangeTriggerTimes, namespacedName)
		} else if t := getLastChangeTriggerTime(endpointSlice.Annotations); !t.IsZero() && t.After(ect.trackerStartTime) {
			ect.lastChangeTriggerTimes[namespacedName] =
				append(ect.lastChangeTriggerTimes[namespacedName], t)
		}
	}

	return changeNeeded
}

func (cache *EndpointSliceCache) updatePending(endpointSlice *discovery.EndpointSlice, remove bool) bool {
    serviceKey, sliceKey, err := endpointSliceCacheKeys(endpointSlice)
    
    esData := &endpointSliceData{endpointSlice, remove}
    
    cache.lock.Lock()
    defer cache.lock.Unlock()
    
    if _, ok := cache.trackerByServiceMap[serviceKey]; !ok {
        // 这个是新的Service
        cache.trackerByServiceMap[serviceKey] = newEndpointSliceTracker()
    }
    // 这里就是在比对有没有被缓存过，或者真准备被proixer应用，如果都没有则追加到pending中
	// 在同步的时候会做一次全量checkout到applied
    changed := cache.esDataChanged(serviceKey, sliceKey, esData)
    
    if changed {
        cache.trackerByServiceMap[serviceKey].pending[sliceKey] = esData
    }
    
    return changed
}
```

### 2.3 iptables的syncProxyRules

```go
func (proxier *Proxier) syncProxyRules() {
	proxier.mu.Lock()
	defer proxier.mu.Unlock()

	// The value of proxier.needFullSync may change before the defer funcs run, so
	// we need to keep track of whether it was set at the *start* of the sync.
	tryPartialSync := !proxier.needFullSync

	start := time.Now()
	defer func() {
		// metrics 统计该方法的执行时间
	}()

	serviceUpdateResult := proxier.svcPortMap.Update(proxier.serviceChanges)
	endpointUpdateResult := proxier.endpointsMap.Update(proxier.endpointsChanges)
	
	success := false
	defer func() {
		if !success {
			// 失败重试 -> metrics
		}
	}()

	if !tryPartialSync {
		// Ensure that our jump rules (eg from PREROUTING to KUBE-SERVICES) exist.
		// We can't do this as part of the iptables-restore because we don't want
		// to specify/replace *all* of the rules in PREROUTING, etc.
		//
		// We need to create these rules when kube-proxy first starts, and we need
		// to recreate them if the utiliptables Monitor detects that iptables has
		// been flushed. In both of those cases, the code will force a full sync.
		// In all other cases, it ought to be safe to assume that the rules
		// already exist, so we'll skip this step when doing a partial sync, to
		// save us from having to invoke /sbin/iptables 20 times on each sync
		// (which will be very slow on hosts with lots of iptables rules).
		for _, jump := range append(iptablesJumpChains, iptablesKubeletJumpChains...) {
			if _, err := proxier.iptables.EnsureChain(jump.table, jump.dstChain); err != nil {
				proxier.logger.Error(err, "Failed to ensure chain exists", "table", jump.table, "chain", jump.dstChain)
				return
			}
			args := jump.extraArgs
			if jump.comment != "" {
				args = append(args, "-m", "comment", "--comment", jump.comment)
			}
			args = append(args, "-j", string(jump.dstChain))
			if _, err := proxier.iptables.EnsureRule(utiliptables.Prepend, jump.table, jump.srcChain, args...); err != nil {
				proxier.logger.Error(err, "Failed to ensure chain jumps", "table", jump.table, "srcChain", jump.srcChain, "dstChain", jump.dstChain)
				return
			}
		}

		// ensure the nfacct counters
		if proxier.nfacct != nil {
			for name := range proxier.nfAcctCounters {
				if err := proxier.nfacct.Ensure(name); err != nil {
					proxier.nfAcctCounters[name] = false
					proxier.logger.Error(err, "Failed to create nfacct counter; the corresponding metric will not be updated", "counter", name)
				} else {
					proxier.nfAcctCounters[name] = true
				}
			}
		}
	}

	//
	// Below this point we will not return until we try to write the iptables rules.
	//

	// Reset all buffers used later.
	// This is to avoid memory reallocations and thus improve performance.
	proxier.filterChains.Reset()
	proxier.filterRules.Reset()
	proxier.natChains.Reset()
	proxier.natRules.Reset()

	skippedNatChains := proxyutil.NewDiscardLineBuffer()
	skippedNatRules := proxyutil.NewDiscardLineBuffer()

	// Write chain lines for all the "top-level" chains we'll be filling in
	for _, chainName := range []utiliptables.Chain{kubeServicesChain, kubeExternalServicesChain, kubeForwardChain, kubeNodePortsChain, kubeProxyFirewallChain} {
		proxier.filterChains.Write(utiliptables.MakeChainLine(chainName))
	}
	for _, chainName := range []utiliptables.Chain{kubeServicesChain, kubeNodePortsChain, kubePostroutingChain, kubeMarkMasqChain} {
		proxier.natChains.Write(utiliptables.MakeChainLine(chainName))
	}

	// Install the kubernetes-specific postrouting rules. We use a whole chain for
	// this so that it is easier to flush and change, for example if the mark
	// value should ever change.

	proxier.natRules.Write(
		"-A", string(kubePostroutingChain),
		"-m", "mark", "!", "--mark", fmt.Sprintf("%s/%s", proxier.masqueradeMark, proxier.masqueradeMark),
		"-j", "RETURN",
	)
	// Clear the mark to avoid re-masquerading if the packet re-traverses the network stack.
	proxier.natRules.Write(
		"-A", string(kubePostroutingChain),
		"-j", "MARK", "--xor-mark", proxier.masqueradeMark,
	)
	masqRule := []string{
		"-A", string(kubePostroutingChain),
		"-m", "comment", "--comment", `"kubernetes service traffic requiring SNAT"`,
		"-j", "MASQUERADE",
	}
	if proxier.iptables.HasRandomFully() {
		masqRule = append(masqRule, "--random-fully")
	}
	proxier.natRules.Write(masqRule)

	// Install the kubernetes-specific masquerade mark rule. We use a whole chain for
	// this so that it is easier to flush and change, for example if the mark
	// value should ever change.
	proxier.natRules.Write(
		"-A", string(kubeMarkMasqChain),
		"-j", "MARK", "--or-mark", proxier.masqueradeMark,
	)

	isIPv6 := proxier.iptables.IsIPv6()
	if !isIPv6 && proxier.localhostNodePorts {
		// Kube-proxy's use of `route_localnet` to enable NodePorts on localhost
		// creates a security hole (https://issue.k8s.io/90259) which this
		// iptables rule mitigates.

		// NOTE: kubelet creates an identical copy of this rule. If you want to
		// change this rule in the future, you MUST do so in a way that will
		// interoperate correctly with skewed versions of the rule created by
		// kubelet. (Actually, kubelet uses "--dst"/"--src" rather than "-d"/"-s"
		// but that's just a command-line thing and results in the same rule being
		// created in the kernel.)
		proxier.filterChains.Write(utiliptables.MakeChainLine(kubeletFirewallChain))
		proxier.filterRules.Write(
			"-A", string(kubeletFirewallChain),
			"-m", "comment", "--comment", `"block incoming localnet connections"`,
			"-d", "127.0.0.0/8",
			"!", "-s", "127.0.0.0/8",
			"-m", "conntrack",
			"!", "--ctstate", "RELATED,ESTABLISHED,DNAT",
			"-j", "DROP",
		)
	}

	// Accumulate NAT chains to keep.
	activeNATChains := sets.New[utiliptables.Chain]()

	// To avoid growing this slice, we arbitrarily set its size to 64,
	// there is never more than that many arguments for a single line.
	// Note that even if we go over 64, it will still be correct - it
	// is just for efficiency, not correctness.
	args := make([]string, 64)

	// Compute total number of endpoint chains across all services
	// to get a sense of how big the cluster is.
	totalEndpoints := 0
	for svcName := range proxier.svcPortMap {
		totalEndpoints += len(proxier.endpointsMap[svcName])
	}
	proxier.largeClusterMode = (totalEndpoints > largeClusterEndpointsThreshold)

	// These two variables are used to publish the sync_proxy_rules_no_endpoints_total
	// metric.
	serviceNoLocalEndpointsTotalInternal := 0
	serviceNoLocalEndpointsTotalExternal := 0

	// Build rules for each service-port.
	for svcName, svc := range proxier.svcPortMap {
		svcInfo, ok := svc.(*servicePortInfo)
		if !ok {
			proxier.logger.Error(nil, "Failed to cast serviceInfo", "serviceName", svcName)
			continue
		}
		protocol := strings.ToLower(string(svcInfo.Protocol()))
		svcPortNameString := svcInfo.nameString

		// Figure out the endpoints for Cluster and Local traffic policy.
		// allLocallyReachableEndpoints is the set of all endpoints that can be routed to
		// from this node, given the service's traffic policies. hasEndpoints is true
		// if the service has any usable endpoints on any node, not just this one.
		allEndpoints := proxier.endpointsMap[svcName]
		clusterEndpoints, localEndpoints, allLocallyReachableEndpoints, hasEndpoints := proxy.CategorizeEndpoints(allEndpoints, svcInfo, proxier.nodeLabels)

		// clusterPolicyChain contains the endpoints used with "Cluster" traffic policy
		clusterPolicyChain := svcInfo.clusterPolicyChainName
		usesClusterPolicyChain := len(clusterEndpoints) > 0 && svcInfo.UsesClusterEndpoints()

		// localPolicyChain contains the endpoints used with "Local" traffic policy
		localPolicyChain := svcInfo.localPolicyChainName
		usesLocalPolicyChain := len(localEndpoints) > 0 && svcInfo.UsesLocalEndpoints()

		// internalPolicyChain is the chain containing the endpoints for
		// "internal" (ClusterIP) traffic. internalTrafficChain is the chain that
		// internal traffic is routed to (which is always the same as
		// internalPolicyChain). hasInternalEndpoints is true if we should
		// generate rules pointing to internalTrafficChain, or false if there are
		// no available internal endpoints.
		internalPolicyChain := clusterPolicyChain
		hasInternalEndpoints := hasEndpoints
		if svcInfo.InternalPolicyLocal() {
			internalPolicyChain = localPolicyChain
			if len(localEndpoints) == 0 {
				hasInternalEndpoints = false
			}
		}
		internalTrafficChain := internalPolicyChain

		// Similarly, externalPolicyChain is the chain containing the endpoints
		// for "external" (NodePort, LoadBalancer, and ExternalIP) traffic.
		// externalTrafficChain is the chain that external traffic is routed to
		// (which is always the service's "EXT" chain). hasExternalEndpoints is
		// true if there are endpoints that will be reached by external traffic.
		// (But we may still have to generate externalTrafficChain even if there
		// are no external endpoints, to ensure that the short-circuit rules for
		// local traffic are set up.)
		externalPolicyChain := clusterPolicyChain
		hasExternalEndpoints := hasEndpoints
		if svcInfo.ExternalPolicyLocal() {
			externalPolicyChain = localPolicyChain
			if len(localEndpoints) == 0 {
				hasExternalEndpoints = false
			}
		}
		externalTrafficChain := svcInfo.externalChainName // eventually jumps to externalPolicyChain

		// usesExternalTrafficChain is based on hasEndpoints, not hasExternalEndpoints,
		// because we need the local-traffic-short-circuiting rules even when there
		// are no externally-usable endpoints.
		usesExternalTrafficChain := hasEndpoints && svcInfo.ExternallyAccessible()

		// Traffic to LoadBalancer IPs can go directly to externalTrafficChain
		// unless LoadBalancerSourceRanges is in use in which case we will
		// create a firewall chain.
		loadBalancerTrafficChain := externalTrafficChain
		fwChain := svcInfo.firewallChainName
		usesFWChain := hasEndpoints && len(svcInfo.LoadBalancerVIPs()) > 0 && len(svcInfo.LoadBalancerSourceRanges()) > 0
		if usesFWChain {
			loadBalancerTrafficChain = fwChain
		}

		var internalTrafficFilterTarget, internalTrafficFilterComment string
		var externalTrafficFilterTarget, externalTrafficFilterComment string
		if !hasEndpoints {
			// The service has no endpoints at all; hasInternalEndpoints and
			// hasExternalEndpoints will also be false, and we will not
			// generate any chains in the "nat" table for the service; only
			// rules in the "filter" table rejecting incoming packets for
			// the service's IPs.
			internalTrafficFilterTarget = "REJECT"
			internalTrafficFilterComment = fmt.Sprintf(`"%s has no endpoints"`, svcPortNameString)
			externalTrafficFilterTarget = "REJECT"
			externalTrafficFilterComment = internalTrafficFilterComment
		} else {
			if !hasInternalEndpoints {
				// The internalTrafficPolicy is "Local" but there are no local
				// endpoints. Traffic to the clusterIP will be dropped, but
				// external traffic may still be accepted.
				internalTrafficFilterTarget = "DROP"
				internalTrafficFilterComment = fmt.Sprintf(`"%s has no local endpoints"`, svcPortNameString)
				serviceNoLocalEndpointsTotalInternal++
			}
			if !hasExternalEndpoints {
				// The externalTrafficPolicy is "Local" but there are no
				// local endpoints. Traffic to "external" IPs from outside
				// the cluster will be dropped, but traffic from inside
				// the cluster may still be accepted.
				externalTrafficFilterTarget = "DROP"
				externalTrafficFilterComment = fmt.Sprintf(`"%s has no local endpoints"`, svcPortNameString)
				serviceNoLocalEndpointsTotalExternal++
			}
		}

		filterRules := proxier.filterRules
		natChains := proxier.natChains
		natRules := proxier.natRules

		// Capture the clusterIP.
		if hasInternalEndpoints {
			natRules.Write(
				"-A", string(kubeServicesChain),
				"-m", "comment", "--comment", fmt.Sprintf(`"%s cluster IP"`, svcPortNameString),
				"-m", protocol, "-p", protocol,
				"-d", svcInfo.ClusterIP().String(),
				"--dport", strconv.Itoa(svcInfo.Port()),
				"-j", string(internalTrafficChain))
		} else {
			// No endpoints.
			filterRules.Write(
				"-A", string(kubeServicesChain),
				"-m", "comment", "--comment", internalTrafficFilterComment,
				"-m", protocol, "-p", protocol,
				"-d", svcInfo.ClusterIP().String(),
				"--dport", strconv.Itoa(svcInfo.Port()),
				"-j", internalTrafficFilterTarget,
			)
		}

		// Capture externalIPs.
		for _, externalIP := range svcInfo.ExternalIPs() {
			if hasEndpoints {
				// Send traffic bound for external IPs to the "external
				// destinations" chain.
				natRules.Write(
					"-A", string(kubeServicesChain),
					"-m", "comment", "--comment", fmt.Sprintf(`"%s external IP"`, svcPortNameString),
					"-m", protocol, "-p", protocol,
					"-d", externalIP.String(),
					"--dport", strconv.Itoa(svcInfo.Port()),
					"-j", string(externalTrafficChain))
			}
			if !hasExternalEndpoints {
				// Either no endpoints at all (REJECT) or no endpoints for
				// external traffic (DROP anything that didn't get
				// short-circuited by the EXT chain.)
				filterRules.Write(
					"-A", string(kubeExternalServicesChain),
					"-m", "comment", "--comment", externalTrafficFilterComment,
					"-m", protocol, "-p", protocol,
					"-d", externalIP.String(),
					"--dport", strconv.Itoa(svcInfo.Port()),
					"-j", externalTrafficFilterTarget,
				)
			}
		}

		// Capture load-balancer ingress.
		for _, lbip := range svcInfo.LoadBalancerVIPs() {
			if hasEndpoints {
				natRules.Write(
					"-A", string(kubeServicesChain),
					"-m", "comment", "--comment", fmt.Sprintf(`"%s loadbalancer IP"`, svcPortNameString),
					"-m", protocol, "-p", protocol,
					"-d", lbip.String(),
					"--dport", strconv.Itoa(svcInfo.Port()),
					"-j", string(loadBalancerTrafficChain))

			}
			if usesFWChain {
				filterRules.Write(
					"-A", string(kubeProxyFirewallChain),
					"-m", "comment", "--comment", fmt.Sprintf(`"%s traffic not accepted by %s"`, svcPortNameString, svcInfo.firewallChainName),
					"-m", protocol, "-p", protocol,
					"-d", lbip.String(),
					"--dport", strconv.Itoa(svcInfo.Port()),
					"-j", "DROP")
			}
		}
		if !hasExternalEndpoints {
			// Either no endpoints at all (REJECT) or no endpoints for
			// external traffic (DROP anything that didn't get short-circuited
			// by the EXT chain.)
			for _, lbip := range svcInfo.LoadBalancerVIPs() {
				filterRules.Write(
					"-A", string(kubeExternalServicesChain),
					"-m", "comment", "--comment", externalTrafficFilterComment,
					"-m", protocol, "-p", protocol,
					"-d", lbip.String(),
					"--dport", strconv.Itoa(svcInfo.Port()),
					"-j", externalTrafficFilterTarget,
				)
			}
		}

		// Capture nodeports.
		if svcInfo.NodePort() != 0 {
			if hasEndpoints {
				// Jump to the external destination chain.  For better or for
				// worse, nodeports are not subect to loadBalancerSourceRanges,
				// and we can't change that.
				if proxier.localhostNodePorts && proxier.ipFamily == v1.IPv4Protocol && proxier.nfAcctCounters[metrics.LocalhostNodePortAcceptedNFAcctCounter] {
					natRules.Write(
						"-A", string(kubeNodePortsChain),
						"-m", "comment", "--comment", svcPortNameString,
						"-m", protocol, "-p", protocol,
						"-d", "127.0.0.0/8",
						"--dport", strconv.Itoa(svcInfo.NodePort()),
						"-m", "nfacct", "--nfacct-name", metrics.LocalhostNodePortAcceptedNFAcctCounter,
						"-j", string(externalTrafficChain))
				}
				natRules.Write(
					"-A", string(kubeNodePortsChain),
					"-m", "comment", "--comment", svcPortNameString,
					"-m", protocol, "-p", protocol,
					"--dport", strconv.Itoa(svcInfo.NodePort()),
					"-j", string(externalTrafficChain))
			}
			if !hasExternalEndpoints {
				// Either no endpoints at all (REJECT) or no endpoints for
				// external traffic (DROP anything that didn't get
				// short-circuited by the EXT chain.)
				filterRules.Write(
					"-A", string(kubeExternalServicesChain),
					"-m", "comment", "--comment", externalTrafficFilterComment,
					"-m", "addrtype", "--dst-type", "LOCAL",
					"-m", protocol, "-p", protocol,
					"--dport", strconv.Itoa(svcInfo.NodePort()),
					"-j", externalTrafficFilterTarget,
				)
			}
		}

		// Capture healthCheckNodePorts.
		if svcInfo.HealthCheckNodePort() != 0 {
			// no matter if node has local endpoints, healthCheckNodePorts
			// need to add a rule to accept the incoming connection
			filterRules.Write(
				"-A", string(kubeNodePortsChain),
				"-m", "comment", "--comment", fmt.Sprintf(`"%s health check node port"`, svcPortNameString),
				"-m", "tcp", "-p", "tcp",
				"--dport", strconv.Itoa(svcInfo.HealthCheckNodePort()),
				"-j", "ACCEPT",
			)
		}

		// If the SVC/SVL/EXT/FW/SEP chains have not changed since the last sync
		// then we can omit them from the restore input. However, we have to still
		// figure out how many chains we _would_ have written, to make the metrics
		// come out right, so we just compute them and throw them away.
		if tryPartialSync && !serviceUpdateResult.UpdatedServices.Has(svcName.NamespacedName) && !endpointUpdateResult.UpdatedServices.Has(svcName.NamespacedName) {
			natChains = skippedNatChains
			natRules = skippedNatRules
		}

		// Set up internal traffic handling.
		if hasInternalEndpoints {
			args = append(args[:0],
				"-m", "comment", "--comment", fmt.Sprintf(`"%s cluster IP"`, svcPortNameString),
				"-m", protocol, "-p", protocol,
				"-d", svcInfo.ClusterIP().String(),
				"--dport", strconv.Itoa(svcInfo.Port()),
			)
			if proxier.masqueradeAll {
				natRules.Write(
					"-A", string(internalTrafficChain),
					args,
					"-j", string(kubeMarkMasqChain))
			} else if proxier.localDetector.IsImplemented() {
				// This masquerades off-cluster traffic to a service VIP. The
				// idea is that you can establish a static route for your
				// Service range, routing to any node, and that node will
				// bridge into the Service for you. Since that might bounce
				// off-node, we masquerade here.
				natRules.Write(
					"-A", string(internalTrafficChain),
					args,
					proxier.localDetector.IfNotLocal(),
					"-j", string(kubeMarkMasqChain))
			}
		}

		// Set up external traffic handling (if any "external" destinations are
		// enabled). All captured traffic for all external destinations should
		// jump to externalTrafficChain, which will handle some special cases and
		// then jump to externalPolicyChain.
		if usesExternalTrafficChain {
			natChains.Write(utiliptables.MakeChainLine(externalTrafficChain))
			activeNATChains.Insert(externalTrafficChain)

			if !svcInfo.ExternalPolicyLocal() {
				// If we are using non-local endpoints we need to masquerade,
				// in case we cross nodes.
				natRules.Write(
					"-A", string(externalTrafficChain),
					"-m", "comment", "--comment", fmt.Sprintf(`"masquerade traffic for %s external destinations"`, svcPortNameString),
					"-j", string(kubeMarkMasqChain))
			} else {
				// If we are only using same-node endpoints, we can retain the
				// source IP in most cases.

				if proxier.localDetector.IsImplemented() {
					// Treat all locally-originated pod -> external destination
					// traffic as a special-case.  It is subject to neither
					// form of traffic policy, which simulates going up-and-out
					// to an external load-balancer and coming back in.
					natRules.Write(
						"-A", string(externalTrafficChain),
						"-m", "comment", "--comment", fmt.Sprintf(`"pod traffic for %s external destinations"`, svcPortNameString),
						proxier.localDetector.IfLocal(),
						"-j", string(clusterPolicyChain))
				}

				// Locally originated traffic (not a pod, but the host node)
				// still needs masquerade because the LBIP itself is a local
				// address, so that will be the chosen source IP.
				natRules.Write(
					"-A", string(externalTrafficChain),
					"-m", "comment", "--comment", fmt.Sprintf(`"masquerade LOCAL traffic for %s external destinations"`, svcPortNameString),
					"-m", "addrtype", "--src-type", "LOCAL",
					"-j", string(kubeMarkMasqChain))

				// Redirect all src-type=LOCAL -> external destination to the
				// policy=cluster chain. This allows traffic originating
				// from the host to be redirected to the service correctly.
				natRules.Write(
					"-A", string(externalTrafficChain),
					"-m", "comment", "--comment", fmt.Sprintf(`"route LOCAL traffic for %s external destinations"`, svcPortNameString),
					"-m", "addrtype", "--src-type", "LOCAL",
					"-j", string(clusterPolicyChain))
			}

			// Anything else falls thru to the appropriate policy chain.
			if hasExternalEndpoints {
				natRules.Write(
					"-A", string(externalTrafficChain),
					"-j", string(externalPolicyChain))
			}
		}

		// Set up firewall chain, if needed
		if usesFWChain {
			natChains.Write(utiliptables.MakeChainLine(fwChain))
			activeNATChains.Insert(fwChain)

			// The service firewall rules are created based on the
			// loadBalancerSourceRanges field. This only works for VIP-like
			// loadbalancers that preserve source IPs. For loadbalancers which
			// direct traffic to service NodePort, the firewall rules will not
			// apply.
			args = append(args[:0],
				"-A", string(fwChain),
				"-m", "comment", "--comment", fmt.Sprintf(`"%s loadbalancer IP"`, svcPortNameString),
			)

			// firewall filter based on each source range
			allowFromNode := false
			for _, cidr := range svcInfo.LoadBalancerSourceRanges() {
				natRules.Write(args, "-s", cidr.String(), "-j", string(externalTrafficChain))
				if cidr.Contains(proxier.nodeIP) {
					allowFromNode = true
				}
			}
			// For VIP-like LBs, the VIP is often added as a local
			// address (via an IP route rule).  In that case, a request
			// from a node to the VIP will not hit the loadbalancer but
			// will loop back with the source IP set to the VIP.  We
			// need the following rules to allow requests from this node.
			if allowFromNode {
				for _, lbip := range svcInfo.LoadBalancerVIPs() {
					natRules.Write(
						args,
						"-s", lbip.String(),
						"-j", string(externalTrafficChain))
				}
			}
			// If the packet was able to reach the end of firewall chain,
			// then it did not get DNATed, so it will match the
			// corresponding KUBE-PROXY-FIREWALL rule.
			natRules.Write(
				"-A", string(fwChain),
				"-m", "comment", "--comment", fmt.Sprintf(`"other traffic to %s will be dropped by KUBE-PROXY-FIREWALL"`, svcPortNameString),
			)
		}

		// If Cluster policy is in use, create the chain and create rules jumping
		// from clusterPolicyChain to the clusterEndpoints
		if usesClusterPolicyChain {
			natChains.Write(utiliptables.MakeChainLine(clusterPolicyChain))
			activeNATChains.Insert(clusterPolicyChain)
			proxier.writeServiceToEndpointRules(natRules, svcPortNameString, svcInfo, clusterPolicyChain, clusterEndpoints, args)
		}

		// If Local policy is in use, create the chain and create rules jumping
		// from localPolicyChain to the localEndpoints
		if usesLocalPolicyChain {
			natChains.Write(utiliptables.MakeChainLine(localPolicyChain))
			activeNATChains.Insert(localPolicyChain)
			proxier.writeServiceToEndpointRules(natRules, svcPortNameString, svcInfo, localPolicyChain, localEndpoints, args)
		}

		// Generate the per-endpoint chains.
		for _, ep := range allLocallyReachableEndpoints {
			epInfo, ok := ep.(*endpointInfo)
			if !ok {
				proxier.logger.Error(nil, "Failed to cast endpointInfo", "endpointInfo", ep)
				continue
			}

			endpointChain := epInfo.ChainName

			// Create the endpoint chain
			natChains.Write(utiliptables.MakeChainLine(endpointChain))
			activeNATChains.Insert(endpointChain)

			args = append(args[:0], "-A", string(endpointChain))
			args = proxier.appendServiceCommentLocked(args, svcPortNameString)
			// Handle traffic that loops back to the originator with SNAT.
			natRules.Write(
				args,
				"-s", epInfo.IP(),
				"-j", string(kubeMarkMasqChain))
			// Update client-affinity lists.
			if svcInfo.SessionAffinityType() == v1.ServiceAffinityClientIP {
				args = append(args, "-m", "recent", "--name", string(endpointChain), "--set")
			}
			// DNAT to final destination.
			args = append(args, "-m", protocol, "-p", protocol, "-j", "DNAT", "--to-destination", epInfo.String())
			natRules.Write(args)
		}
	}

	// Delete chains no longer in use. Since "iptables-save" can take several seconds
	// to run on hosts with lots of iptables rules, we don't bother to do this on
	// every sync in large clusters. (Stale chains will not be referenced by any
	// active rules, so they're harmless other than taking up memory.)
	deletedChains := 0
	if !proxier.largeClusterMode || time.Since(proxier.lastIPTablesCleanup) > proxier.syncPeriod {
		proxier.iptablesData.Reset()
		if err := proxier.iptables.SaveInto(utiliptables.TableNAT, proxier.iptablesData); err == nil {
			existingNATChains := utiliptables.GetChainsFromTable(proxier.iptablesData.Bytes())
			for chain := range existingNATChains.Difference(activeNATChains) {
				chainString := string(chain)
				if !isServiceChainName(chainString) {
					// Ignore chains that aren't ours.
					continue
				}
				// We must (as per iptables) write a chain-line
				// for it, which has the nice effect of flushing
				// the chain. Then we can remove the chain.
				proxier.natChains.Write(utiliptables.MakeChainLine(chain))
				proxier.natRules.Write("-X", chainString)
				deletedChains++
			}
			proxier.lastIPTablesCleanup = time.Now()
		} else {
			proxier.logger.Error(err, "Failed to execute iptables-save: stale chains will not be deleted")
		}
	}

	// Finally, tail-call to the nodePorts chain.  This needs to be after all
	// other service portal rules.
	if proxier.nodePortAddresses.MatchAll() {
		destinations := []string{"-m", "addrtype", "--dst-type", "LOCAL"}
		// Block localhost nodePorts if they are not supported. (For IPv6 they never
		// work, and for IPv4 they only work if we previously set `route_localnet`.)
		if isIPv6 {
			destinations = append(destinations, "!", "-d", "::1/128")
		} else if !proxier.localhostNodePorts {
			destinations = append(destinations, "!", "-d", "127.0.0.0/8")
		}

		proxier.natRules.Write(
			"-A", string(kubeServicesChain),
			"-m", "comment", "--comment", `"kubernetes service nodeports; NOTE: this must be the last rule in this chain"`,
			destinations,
			"-j", string(kubeNodePortsChain))
	} else {
		nodeIPs, err := proxier.nodePortAddresses.GetNodeIPs(proxier.networkInterfacer)
		if err != nil {
			proxier.logger.Error(err, "Failed to get node ip address matching nodeport cidrs, services with nodeport may not work as intended", "CIDRs", proxier.nodePortAddresses)
		}
		for _, ip := range nodeIPs {
			if ip.IsLoopback() {
				if isIPv6 {
					proxier.logger.Error(nil, "--nodeport-addresses includes localhost but localhost NodePorts are not supported on IPv6", "address", ip.String())
					continue
				} else if !proxier.localhostNodePorts {
					proxier.logger.Error(nil, "--nodeport-addresses includes localhost but --iptables-localhost-nodeports=false was passed", "address", ip.String())
					continue
				}
			}

			// create nodeport rules for each IP one by one
			proxier.natRules.Write(
				"-A", string(kubeServicesChain),
				"-m", "comment", "--comment", `"kubernetes service nodeports; NOTE: this must be the last rule in this chain"`,
				"-d", ip.String(),
				"-j", string(kubeNodePortsChain))
		}
	}

	// Drop the packets in INVALID state, which would potentially cause
	// unexpected connection reset if nf_conntrack_tcp_be_liberal is not set.
	// Ref: https://github.com/kubernetes/kubernetes/issues/74839
	// Ref: https://github.com/kubernetes/kubernetes/issues/117924
	if !proxier.conntrackTCPLiberal {
		rule := []string{
			"-A", string(kubeForwardChain),
			"-m", "conntrack",
			"--ctstate", "INVALID",
		}
		if proxier.nfAcctCounters[metrics.IPTablesCTStateInvalidDroppedNFAcctCounter] {
			rule = append(rule,
				"-m", "nfacct", "--nfacct-name", metrics.IPTablesCTStateInvalidDroppedNFAcctCounter,
			)
		}
		rule = append(rule,
			"-j", "DROP",
		)
		proxier.filterRules.Write(rule)
	}

	// If the masqueradeMark has been added then we want to forward that same
	// traffic, this allows NodePort traffic to be forwarded even if the default
	// FORWARD policy is not accept.
	proxier.filterRules.Write(
		"-A", string(kubeForwardChain),
		"-m", "comment", "--comment", `"kubernetes forwarding rules"`,
		"-m", "mark", "--mark", fmt.Sprintf("%s/%s", proxier.masqueradeMark, proxier.masqueradeMark),
		"-j", "ACCEPT",
	)

	// The following rule ensures the traffic after the initial packet accepted
	// by the "kubernetes forwarding rules" rule above will be accepted.
	proxier.filterRules.Write(
		"-A", string(kubeForwardChain),
		"-m", "comment", "--comment", `"kubernetes forwarding conntrack rule"`,
		"-m", "conntrack",
		"--ctstate", "RELATED,ESTABLISHED",
		"-j", "ACCEPT",
	)

	metrics.IPTablesRulesTotal.WithLabelValues(string(utiliptables.TableFilter)).Set(float64(proxier.filterRules.Lines()))
	metrics.IPTablesRulesLastSync.WithLabelValues(string(utiliptables.TableFilter)).Set(float64(proxier.filterRules.Lines()))
	metrics.IPTablesRulesTotal.WithLabelValues(string(utiliptables.TableNAT)).Set(float64(proxier.natRules.Lines() + skippedNatRules.Lines() - deletedChains))
	metrics.IPTablesRulesLastSync.WithLabelValues(string(utiliptables.TableNAT)).Set(float64(proxier.natRules.Lines() - deletedChains))

	// Sync rules.
	proxier.iptablesData.Reset()
	proxier.iptablesData.WriteString("*filter\n")
	proxier.iptablesData.Write(proxier.filterChains.Bytes())
	proxier.iptablesData.Write(proxier.filterRules.Bytes())
	proxier.iptablesData.WriteString("COMMIT\n")
	proxier.iptablesData.WriteString("*nat\n")
	proxier.iptablesData.Write(proxier.natChains.Bytes())
	proxier.iptablesData.Write(proxier.natRules.Bytes())
	proxier.iptablesData.WriteString("COMMIT\n")

	proxier.logger.V(2).Info("Reloading service iptables data",
		"numServices", len(proxier.svcPortMap),
		"numEndpoints", totalEndpoints,
		"numFilterChains", proxier.filterChains.Lines(),
		"numFilterRules", proxier.filterRules.Lines(),
		"numNATChains", proxier.natChains.Lines(),
		"numNATRules", proxier.natRules.Lines(),
	)
	proxier.logger.V(9).Info("Restoring iptables", "rules", proxier.iptablesData.Bytes())

	// NOTE: NoFlushTables is used so we don't flush non-kubernetes chains in the table
	err := proxier.iptables.RestoreAll(proxier.iptablesData.Bytes(), utiliptables.NoFlushTables, utiliptables.RestoreCounters)
	if err != nil {
		if pErr, ok := err.(utiliptables.ParseError); ok {
			lines := utiliptables.ExtractLines(proxier.iptablesData.Bytes(), pErr.Line(), 3)
			proxier.logger.Error(pErr, "Failed to execute iptables-restore", "rules", lines)
		} else {
			proxier.logger.Error(err, "Failed to execute iptables-restore")
		}
		metrics.IPTablesRestoreFailuresTotal.Inc()
		return
	}
	success = true
	proxier.needFullSync = false

	for name, lastChangeTriggerTimes := range endpointUpdateResult.LastChangeTriggerTimes {
		for _, lastChangeTriggerTime := range lastChangeTriggerTimes {
			latency := metrics.SinceInSeconds(lastChangeTriggerTime)
			metrics.NetworkProgrammingLatency.Observe(latency)
			proxier.logger.V(4).Info("Network programming", "endpoint", klog.KRef(name.Namespace, name.Name), "elapsed", latency)
		}
	}

	metrics.SyncProxyRulesNoLocalEndpointsTotal.WithLabelValues("internal").Set(float64(serviceNoLocalEndpointsTotalInternal))
	metrics.SyncProxyRulesNoLocalEndpointsTotal.WithLabelValues("external").Set(float64(serviceNoLocalEndpointsTotalExternal))
	if proxier.healthzServer != nil {
		proxier.healthzServer.Updated(proxier.ipFamily)
	}
	metrics.SyncProxyRulesLastTimestamp.SetToCurrentTime()

	// Update service healthchecks.  The endpoints list might include services that are
	// not "OnlyLocal", but the services list will not, and the serviceHealthServer
	// will just drop those endpoints.
	if err := proxier.serviceHealthServer.SyncServices(proxier.svcPortMap.HealthCheckNodePorts()); err != nil {
		proxier.logger.Error(err, "Error syncing healthcheck services")
	}
	if err := proxier.serviceHealthServer.SyncEndpoints(proxier.endpointsMap.LocalReadyEndpoints()); err != nil {
		proxier.logger.Error(err, "Error syncing healthcheck endpoints")
	}

	// Finish housekeeping, clear stale conntrack entries for UDP Services
	conntrack.CleanStaleEntries(proxier.conntrack, proxier.svcPortMap, serviceUpdateResult, endpointUpdateResult)
}
```

### 2.4 ipvs的syncProxyRules

