# 问题及解决思路记录

问题1

使用

```
sudo bpftrace -e 'kprobe:kfree_skb /comm=="curl"/ {printf("kstack: %s\n", kstack);}'
```

提示

```bash
stdin:1:1-17: WARNING: kfree_skb is not traceable (either non-existing, inlined, or marked as "notrace"); attaching to it will likely fail kprobe:kfree_skb /comm=="curl"/ {printf("kstack: %s\n", kstack);}
```

遇到问题不要慌，根据提示来一步一步分析一下。首先看到提示是说我们这个kfree\_skb函数是either non-existing, inlined, or marked as "notrace"，那么先看看是不是存在吧。

使用如下命令来列举一下所有的kprobe点，发现确实没有kfree\_skb（这很正常，因为内核函数是在变化的）

```bash
// Some code
parallels@ubuntu-linux-22-04-02-desktop:/media/psf/code/my-ebpf$ sudo bpftrace -l 'kprobe:*' | grep kfree_skb
kprobe:__dev_kfree_skb_any
kprobe:__dev_kfree_skb_irq
kprobe:__kfree_skb
kprobe:__kfree_skb_defer
kprobe:__traceiter_kfree_skb
kprobe:kfree_skb_list
kprobe:kfree_skb_partial
kprobe:kfree_skb_reason
kprobe:kfree_skbmem
kprobe:net_dm_packet_trace_kfree_skb_hit
kprobe:rtnl_kfree_skbs
kprobe:trace_kfree_skb_hit

```

然后根据提示inlined, or marked as "notrace"，说这个函数可能是内联的，那我们就去看看自己内核版本相应的源码，发现确实是内联了，所以我们要追踪的实际上是kfree\_skb\_reason这个函数，OK问题解决。

````c
// Some code
```cpp
/**
 *	kfree_skb - free an sk_buff with 'NOT_SPECIFIED' reason
 *	@skb: buffer to free
 */
static inline void kfree_skb(struct sk_buff *skb)
{
	kfree_skb_reason(skb, SKB_DROP_REASON_NOT_SPECIFIED);
}
```
````

#### todolist

* [ ] 是否可以用bpftrace追踪bpftrace自己的调用栈？
