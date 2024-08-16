+++
title = 'Kube Proxy如何监听Service和Endpoint以及更新策略规则'
date = 2022-08-13T22:36:08+08:00
draft = false
tags = [
    "云原生",
    "Kubernetes",
    "网络通讯",
    "Kube-Proxy"
]
categories = [
    "Kubernetes"
]
+++

本文会探讨Kubernetes另一个核心网络组件Kube-Proxy，它承担着Service及其后端Pod对宿主机配置的影响。

## 一、 先聊一下三个核心API

这三个API在不同版本下的Kube-Proxy发挥着主要作用，尤其是后两者。

### 1. Service

### 2. Endpoint、EndpointSlice

`EndpointSlice`是在k8s 1.9版本开始支持的

## 二、Kube-Proxy的源码分析

<!--more-->

### 1. Kube-Proxy的相关配置

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
	 *  1. LocalhostNodePorts, 不允许本地访问NodePort端口。仅在iptables&ipv4生效
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

	// nodePortAddresses is a list of CIDR ranges that contain valid node IPs, or
	// alternatively, the single string 'primary'. If set to a list of CIDRs,
	// connections to NodePort services will only be accepted on node IPs in one of
	// the indicated ranges. If set to 'primary', NodePort services will only be
	// accepted on the node's primary IPv4 and/or IPv6 address according to the Node
	// object. If unset, NodePort connections will be accepted on all local IPs.
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



### 2. Informer的监听函数和Provider.SyncLoop()


```go
package config
// NodeHandler is an abstract interface of objects which receive
// notifications about node object changes.
type NodeHandler interface {
	// OnNodeAdd is called whenever creation of new node object
	// is observed.
	OnNodeAdd(node *v1.Node)
	// OnNodeUpdate is called whenever modification of an existing
	// node object is observed.
	OnNodeUpdate(oldNode, node *v1.Node)
	// OnNodeDelete is called whenever deletion of an existing node
	// object is observed.
	OnNodeDelete(node *v1.Node)
	// OnNodeSynced is called once all the initial event handlers were
	// called and the state is fully propagated to local cache.
	OnNodeSynced()
}


type ServiceHandler interface {
	// OnServiceAdd is called whenever creation of new service object
	// is observed.
	OnServiceAdd(service *v1.Service)
	// OnServiceUpdate is called whenever modification of an existing
	// service object is observed.
	OnServiceUpdate(oldService, service *v1.Service)
	// OnServiceDelete is called whenever deletion of an existing service
	// object is observed.
	OnServiceDelete(service *v1.Service)
	// OnServiceSynced is called once all the initial event handlers were
	// called and the state is fully propagated to local cache.
	OnServiceSynced()
}

// EndpointSliceHandler is an abstract interface of objects which receive
// notifications about endpoint slice object changes.
type EndpointSliceHandler interface {
	// OnEndpointSliceAdd is called whenever creation of new endpoint slice
	// object is observed.
	OnEndpointSliceAdd(endpointSlice *discoveryv1.EndpointSlice)
	// OnEndpointSliceUpdate is called whenever modification of an existing
	// endpoint slice object is observed.
	OnEndpointSliceUpdate(oldEndpointSlice, newEndpointSlice *discoveryv1.EndpointSlice)
	// OnEndpointSliceDelete is called whenever deletion of an existing
	// endpoint slice object is observed.
	OnEndpointSliceDelete(endpointSlice *discoveryv1.EndpointSlice)
	// OnEndpointSlicesSynced is called once all the initial event handlers were
	// called and the state is fully propagated to local cache.
	OnEndpointSlicesSynced()
}

type ServiceCIDRHandler interface {
    // OnServiceCIDRsChanged is called whenever a change is observed
    // in any of the ServiceCIDRs, and provides complete list of service cidrs.
    OnServiceCIDRsChanged(cidrs []string)
}

// Provider is the interface provided by proxier implementations.
type Provider interface {
	config.EndpointSliceHandler
	config.ServiceHandler
	config.NodeHandler
	config.ServiceCIDRHandler

	// Sync immediately synchronizes the Provider's current state to proxy rules.
	Sync()
	// SyncLoop runs periodic work.
	// This is expected to run as a goroutine or as the main loop of the app.
	// It does not return.
	SyncLoop()
}
```