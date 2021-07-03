---
title:  "低内核版本玩转Cilium"
tags:   cilium
key: run-cilium-on-low-kernel
---

Cilium需要较高的内核版本支持，本文介绍如何在4.0+的低内核版本上运行Cilium，并且做到生产级可用。

<!--more-->

# 内核版本依赖
Cilium v1.9依赖的最低内核版本为4.9.17[1],如果实现kube-proxy全部功能替换则需要4.19+的内核版本支持。详细特性依赖见下表格：

| Cilium Feature | Minimum Kernel Version |
| :--- | :---- |
| Minimum Kernel Version | >= 4.10 |
| Restrictions on unique prefix lengths for CIDR policy rules | >= 4.11 |
| Transparent Encryption (stable/beta) in tunneling mode | >= 4.19 |
| Host-Reachable Services | >= 4.19.57, >= 5.1.16, >= 5.2 |
| Kubernetes Without kube-proxy | >= 4.19.57, >= 5.1.16, >= 5.2 |
| Bandwidth Manager (beta) | >= 5.1 |
| Local Redirect Policy (beta) | >= 4.19.57, >= 5.1.16, >= 5.2 |
| Full support for Session Affinity | >= 5.7 |
| BPF-based proxy redirection | >= 5.7 |
| BPF-based host routing | >= 5.10 |
| NodePort[2] | >=4.17.0 |


# 整体方案
对比其他CNI网络插件[3]，Cilium的性能优势是巨大的，主要体现在协议栈的处理流程优化上，东西向流量尽量靠近socket处理，南北向流量靠近驱动层处理，直接跳过网络层的处理流程。
<img src="/images/cilium/run-cilium-on-low-kernel/cilium-bpf-arch.png" alt="drawing" width="800"/>
sock/BPF和xdp/BPF需要4.17以上的内核才能支持，稳定特性支持需要5.0+才行，考虑到现在很多操作系统运行的都是较低的版本，比如4.14，这个时候我们就需要寻求一些替代方案，总结起来就是：基于native-routing打通底层pod网络的通信后，通过tc/BPF实现东西向流量，kube-proxy提供的iptables实现南北向流量管理。

<img src="/images/cilium/run-cilium-on-low-kernel/cilium-arch.png" alt="drawing" width="800"/>

最终，service的具体功能模块实现见下表：

| Service Feature | 实现组件 |
| :--- | :---- |
| ClusterIP | Cilium tc/BPF |
| NodePort | kube-proxy |
| LoadBalancer  | kube-proxy |
| ExternalIPs | kube-proxy |
| HostPort | kube-proxy |
| HostReachable Service | native-routing/iptables |


# kube-proxy替代方案
为了明确kube-proxy只管理南北向流量的职责，将kube-proxy处理东西向流量的功能移除。

<img src="/images/cilium/run-cilium-on-low-kernel/kube-proxy-iptables.png" alt="drawing" width="800"/>

改造1.1中KUBE-SERVICES提供的NAT能力，将KUBE-MARK-MASQ和KUBE-SVC-*的链全部移除

<img src="/images/cilium/run-cilium-on-low-kernel/kube-proxy-adapter.png" alt="drawing" width="800"/>

# Native Routing
Native routing是底层pod之间的网络通信能力，cilium通过native-routing的方式将底层路由实现暴露出去，极大的增强了整个网络的可扩展性。具体实现可以有多种方式，可以基于隧道等overlay，也可以基于underlay bgp等方案实现。Cilium原生提供了vxlan和geneve两种隧道方案，同时支持其他多种第三方路由能力的集成。

# Reference
1. https://docs.cilium.io/en/v1.9/operations/system_requirements/#required-kernel-versions-for-advanced-features
2. https://github.com/cilium/cilium/blob/master/daemon/cmd/kube_proxy_replacement.go#L214
3. https://cilium.io/blog/2021/05/11/cni-benchmark