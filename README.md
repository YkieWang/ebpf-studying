---
description: 历史不会简单重复，但总是有迹可循
---

# ebpf的起源和发展

## 前身BPF：包过滤工具

**BPF:** The Berkeley Packet Filter, found in FreeBSD. 1993.

1992年，BSD 操作系统带来了革命性的包过滤机制 BSD Packet Filter（简称为 BPF），是用来抓包和过滤的一个工具。比当时最先进的数据包过滤技术还快 20 倍。为什么性能这么好呢？这主要得益于 BPF 的两大设计：

1.内核态引入了一个虚拟机，所有指令都在内核虚拟机中运行

2.用户态使用BPF字节码来定义过滤表达式，然后传递给内核，由内核虚拟机解释执行。

这样数据包的过滤就可以直接在内核中进行，避免了像用户态复制每个数据包。

之后五年也就是1997年，BPF内首次引入Linux 2.1.75。

Linux3.0增加BPF jit即时编译期，替换掉了原来的解释器，使得BPF指令运行效率大大提高。不过到这时候，BPF的作用都只是作为包过滤的工具。

## ebpf：从小工具到内核子系统

2014年，为了研究新的软件定义网络方案，[Alexei Starovoitov ](https://www.linkedin.com/in/alexey1/)为 BPF 带来了第一次革命性的更新，将 BPF 扩展为一个通用的虚拟机，也就是 eBPF。此次更新的主要内容有

1. 将BPF扩展为一个通用的虚拟机ebpf
2. 扩展寄存器数量
3. 引入BPF映射存储
4. 在4.x内核中ebpf就不仅仅能过滤数据包了，把数据包的过滤事件扩展到内核态函数、用户态函数、跟踪点、性能事件以及安全控制等。

eBPF 的诞生是 BPF 技术的一个转折点，使得 BPF 不再仅限于网络栈，而是成为内核的一个顶级子系统。

## ebpf生态的发展时间线

iovisor的bcc、bpftrace等工具，Cilium、Katran、Falco 等一系列基于 eBPF 优化网络和安全的开源项目。Calico，就在最近的版本中引入了 eBPF 数据面网络，大大提升了网络的性能。

### BCC（BPF compiler collection）

2015年，BCC发布，

同年，Linux4.1也开始支持kprobe和cls\_bpf。

### XDP和跟踪点

2016年，Linux4.7—Linux4.10，新增了跟踪点，perf事件，XDP以及cgroups对ebpf的支持，ebpf能够发挥作用的事件源大大增加。

同年，cilium发布。

### bpftool和libbpf

2017年，BPF成为内核独立子模块，并支持kTLS/bpftool/libbpf等。

同年，netflix/facebook/cloudflare开始应用ebpf。

### BTF

2018年，新增BTF，和AF\_XDP类型。

同年，bpftrace和bpffilter项目正式发布。

### 尾调用和热更新

2019年

同年，cilium发布基于ebpf的服务代理，完全替换基于iptables的kube-proxy。

### LSM和TCP拥塞控制

2020年，Google和Facebook为BPF新增LSM和TCP拥塞控制的支持。

主流云厂商开始通过sriov支持XDP。

微软基于ebpf开始为windows监控工具sysmon增加linux的支持。

### ebpf基金会

2021年，微软发布Windows ebpf，并于facebook/google/isovalent/netflix等一起成立ebpf基金会。

## 总结

从ebpf发展的时间线来看，一切都起源于当初那个“在内核中加一个虚拟机”的想法，然后ebpf一路发展从虚拟机发展到子系统。尽管ebpf的本意也并不是为了网络流量数据的处理，但是由于它可以在内核中处理数据的功能，改变了数据包在内核中只能走既定的协议栈路线的现状，提供了一种“挖地洞、抄近道、埋地雷”的手段，所以在网络流量处理中能够大显身手。

回过头来想一下，ebpf其实和dpdk有很多异曲同工之妙，

1. 两者都有“passby”内核思想。dpdk是passby的很彻底，直接接管网卡来自己处理，ebpf是部分passby内核，它简化了数据包在内核中的路径，能抄近道的都抄近道，能提前处理的不多处理一步。
2. 对内存拷贝和锁的优化，这在高性能流量处理平台中是老生常谈了，尽量使用共享内存，减少拷贝次数，每个CPU有自己的内存。
