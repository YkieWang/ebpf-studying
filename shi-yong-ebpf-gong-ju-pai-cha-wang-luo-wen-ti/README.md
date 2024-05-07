# 使用ebpf工具排查网络问题

## 工具集

[bpftrace](https://github.com/bpftrace/bpftrace/blob/master/man/adoc/bpftrace.adoc)

## 内核网络相关函数

[Socket Buffer Functions](https://archive.kernel.org/oldlinux/htmldocs/networking/ch01s02.html)

## 常见网络问题及排查手段

### 丢包/网络不通

丢包或者网络不通，那么数据包一定在内核栈某处被消耗掉了，查询上面文档发现，内核中释放 SKB 相关的函数有两个

kfree\_skb ，它经常在网络异常丢包时调用；

consume\_skb ，它在正常网络连接完成时调用。

使用bpftrace可以跟踪调用栈，看数据包最终是在哪里被丢掉了，其中的curl可以替换成你正在使用的工具

```bash
// Some code
sudo bpftrace -e 'kprobe:kfree_skb /comm=="curl"/ {printf("kstack: %s\n", kstack);}'
```

#### 举例1

我们追踪curl www.baidu.com的内核函数调用栈（你可能注意到我追踪了kfree\_skb\_reason函数而不是kfree\_skb，原因在[这里](wen-ti-ji-jie-jue-si-lu-ji-lu.md)），就可以看到如下信息，然后就可以追踪数据包路径，从下到上是内核函数调用顺序。

```bash
// Some codecode
my-ebpf$ sudo bpftrace -e 'kprobe:kfree_skb_reason /comm=="curl"/ {printf("kstack: %s\n", kstack);}'
[sudo] password for parallels: 
Attaching 1 probe...
kstack: 
        kfree_skb_reason+0
        __sys_connect_file+136
        __sys_connect+188
        __arm64_sys_connect+40
        invoke_syscall+120
        el0_svc_common.constprop.0+84
        do_el0_svc+48
        el0_svc+72
        el0t_64_sync_handler+164
        el0t_64_sync+420

```
