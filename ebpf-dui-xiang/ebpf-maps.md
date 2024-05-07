---
description: 我负责存储
---

# ebpf maps

## 前言

如下的介绍没有深入的研究每一个细节，大概介绍一下map主要是用来干什么的，即这个“技能”的作用是什么，具体的使用细节在开发前最好到[官方技能教学](https://docs.kernel.org/)搜索相应的类型进行查看。

## map类型

内核中定义的所有bpf类型都在[这里](https://github.com/torvalds/linux/blob/v5.10/include/uapi/linux/bpf.h)，可以把map简单理解为（key, value）的集合，我们把它们分成七类来介绍。

### hash map

先看一下最基础的BPF\_MAP\_TYPE\_HASH map的定义方法

```c
// samples/bpf/sockex2_kern.c
struct {
    __uint(type, BPF_MAP_TYPE_HASH);  // BPF map 类型
    __type(key, __be32);              // 目的 IP 地址
    __type(value, struct pair);       // 包数和字节数
    __uint(max_entries, 1024);        // 最大 entry 数量
} hash_map SEC(".maps");
```

* **hash map的底层数据结构都是链表**
* **key/value可删除，可理解为删除链表中的一个节点**
* **查找流程：先对key进行hash然后根据hash值查找**
* [**内核实现**](https://github.com/torvalds/linux/blob/v5.8/kernel/bpf/hashtab.c)

再来看下还有其他什么类型

| 类型                             | 介绍                                                                                                        | 内核代码举例                                                                                                                 |
| ------------------------------ | --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `BPF_MAP_TYPE_HASH`            | 基础hash类型                                                                                                  | <h4 id="id-1-jiang-nei-he-tai-shu-ju-chuan-di-dao-yong-hu-tai-samplesbpfsockex2"><code>samples/bpf/sockex2</code></h4> |
| `BPF_MAP_TYPE_PERCPU_HASH`     | 每个cpu有自己的map，避免并发问题                                                                                       | <h4 id="id-1-samplesbpfmap_perf_test_kernc"><code>samples/bpf/map_perf_test_kern.c</code></h4>                         |
| `BPF_MAP_TYPE_LRU_HASH`        | 前两个map都是固定长度的，一旦entry值达到最大就无法再插入了。但LRU会在满员的情况下自动将**最久未被使用（least recently used）**的 entry 从 map 中移除，从而完成插入。 | <h4 id="id-1-samplesbpfmap_perf_test_kernc-1"><code>samples/bpf/map_perf_test_kern.c</code></h4>                       |
| `BPF_MAP_TYPE_LRU_PERCPU_HASH` | 基于LRU的per cpu                                                                                             |                                                                                                                        |
| `BPF_MAP_TYPE_HASH_OF_MAPS`    | 当map太大，遍历查询链表的时候耗时太久，干脆给它们分类存放吧，利用二级查询来优化掉遍历的损耗，但是千万小心二级查询自己也有损耗的呦。                                       | <h4 id="id-1-samplesbpftest_map_in_map_kernc"><code>samples/bpf/test_map_in_map_kern.c</code></h4>                     |

注意点：因为底层数据结构是链表，在查询的时候是通过hlist\_nulls\_for\_each\_entry\_rcu来查询的，所以我们在开发过程中要留心链表这种数据结构存在的通病，比如Map大小对性能的影响，当Map太大时会不会导致查询效率很低；链表删除插入查询的并发问题等。

另外一个问题，有一种hash map还不够吗，为什么要这么多种类型的呢？其实按照笔者的理解，是为了满足map在不同场景下的特殊需求以优化Maps的性能或者解决某个“硬伤”问题，用下图表示一下，就很好理解了。



<img src="../.gitbook/assets/file.excalidraw.svg" alt="" class="gitbook-drawing">



### array map

先看一下最基础的`BPF_MAP_TYPE_ARRAY` map的定义方法

```c
// samples/bpf/sockex1_kern.c
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __type(key, u32);                  // L4 协议类型（长度是 uint8），例如 IPPROTO_TCP，范围是 0~255
    __type(value, long);               // 累计包长（skb->len）
    __uint(max_entries, 256);
} my_map SEC(".maps");
```

* **hash map的底层数据结构是连续的内存空间**
* **key/value不可删除**（但用空值覆盖掉 value ，可实现删除效果，可理解为平常使用的数组结构）
* **查找流程：直接通过index进行索引**
* [**内核实现**](https://github.com/torvalds/linux/blob/v5.8/kernel/bpf/arraymap.c)

再来看下还有什么其他类型

| 类型                                             | 介绍                                                                         | 内核代码举例                                                                                                                            |
| ---------------------------------------------- | -------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| BPF\_MAP\_TYPE\_ARRAY                          | 简单理解为数组结构                                                                  | <h4 id="id-1-gen-ju-xie-yi-lei-xing-protoaskey-tong-ji-liu-liang-samplesbpfsockex1"><code>samples/bpf/sockex1</code></h4>         |
| BPF\_MAP\_TYPE\_PERCPU\_ARRAY                  | per cpu的                                                                   |                                                                                                                                   |
| BPF\_MAP\_TYPE\_PROG\_ARRAY                    | 程序数组，尾调用 `bpf_tail_call()` 时会用到。                                           | <h4 id="id-1-gen-ju-xie-yi-lei-xing-wei-tiao-yong-dao-xia-yi-ceng-parsersamplesbpfsockex3"><code>samples/bpf/sockex3</code></h4>  |
| BPF\_MAP\_TYPE\_PERF\_EVENT\_ARRAY             | 是一种per-cpu的环形缓冲区，当我们需要将ebpf收集到的数据发送到用户空间记录或者处理时，就可以用该类型的map来完成，后面再做更详细的讲解。 | <h4 id="id-1-bao-cun-perfeventsamplesbpftraceoutputkernc"><code>samples/bpf/trace_output_kern.c</code></h4>                       |
| BPF\_MAP\_TYPE\_CGROUP\_ARRAY（也可以归类到下面cgroup中） | 用来 **检查给定的 skb 是否与 cgroup\_array\[index] 指向的 cgroup 关联**。                  | <h4 id="id-1-pin--update-pinned-cgroup-arraysamplesbpftest_cgrp2_array_pinc"><code>samples/bpf/test_cgrp2_array_pin.c</code></h4> |

数组是通过index直接进行索引的，查询速度较快。这些不同种类的map主要是在value类型上做了文章，可以用下图来表示：

<img src="../.gitbook/assets/file.excalidraw (1).svg" alt="" class="gitbook-drawing">

### cgroup map

* [**内核实现**](https://github.com/torvalds/linux/blob/v5.8/kernel/bpf/cgroup.c)

再来看下还有什么其他类型

| 类型                                      | 介绍                                                                                                                 | 内核代码举例                                                                                                                            |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| BPF\_MAP\_TYPE\_CGROUP\_ARRAY           | 存储cgroup的fd                                                                                                        | <h4 id="id-1-pin--update-pinned-cgroup-arraysamplesbpftest_cgrp2_array_pinc"><code>samples/bpf/test_cgrp2_array_pin.c</code></h4> |
| BPF\_MAP\_TYPE\_CGROUP\_STORAGE         | <h4 id="chang-jing-yi-cgroup-nei-suo-you-bpf-cheng-xu-de-gong-xiang-cun-chu">cgroup 内所有 BPF 程序的共享存储</h4>           | <h4 id="id-1-samplesbpfhbm_kernhhost-bandwidth-manager"><code>samples/bpf/hbm_kern.h</code></h4>                                  |
| BPF\_MAP\_TYPE\_PERCPU\_CGROUP\_STORAGE | <h4 id="chang-jing-yi-cgroup-nei-suo-you-bpf-cheng-xu-de-gong-xiang-cun-chu">per cpu级别的cgroup 内所有 BPF 程序的共享存储</h4> | tools/testing/selftests/bpf/progs/netcnt\_prog.c                                                                                  |

数组是通过index直接进行索引的，查询速度较快。

### tracing map

内核程序能通过 bpf\_get\_stackid() helper 存储 stack 信息。 将 stack 信息关联到一个 id，而这个 id 是**对当前栈的 指令指针地址（instruction pointer address）进行 32-bit hash** 得到的。

| 类型                                 | 介绍                                                                                           | 内核代码举例                                                                                           |
| ---------------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| BPF\_MAP\_TYPE\_STACK              | 是一个栈数据结构，LIFO                                                                                | tools/testing/selftests/bpf/progs/test\_map\_ops.c                                               |
| BPF\_MAP\_TYPE\_STACK\_TRACE       | 这种类型的 map 存储运行进程的堆栈跟踪，配合bpf\_get\_stackid使用。                                                 | <h4 id="id-1-samplesbpfhbm_kernhhost-bandwidth-manager"><code>samples/bpf/hbm_kern.h</code></h4> |
| BPF\_MAP\_TYPE\_PERF\_EVENT\_ARRAY | **per-CPU 环形缓冲区**（circular buffers），能实现高效的 **“内核-用户空间”数据交互**                                 |                                                                                                  |
| BPF\_MAP\_TYPE\_RINGBUF            | **ringbuf 是一个“多生产者、单消费者”**（multi-producer, single-consumer，MPSC） 队列，可**安全地在多个 CPU 之间共享和操作**。 |                                                                                                  |

这里我们重点来讲解一下perfbuf和ringbuf。

BPF\_MAP\_TYPE\_PERF\_EVENT\_ARRAY即perfbuf是内核4.3中引入的，其主要功能如下

优点：&#x20;

1. 可变长数据（variable-length data records）；
2. 通过 memory-mapped region 来高效地从 userspace 读数据，避免内存复制或系统调用；
3. 支持 epoll notifications 和 busy-loop 两种获取数据方式。
4. per-CPU 环形缓冲区（circular buffers），能实现高效的 “内核-用户空间”数据交互

**缺点：**

1. **内存使用效率低下**（inefficient use of memory），
   1. 越大的 per-CPU buffer 越能避免丢数据，但也意味着大部分时间里，大部分内存都是浪费的；
   2. 尽量小的 per-CPU buffer 能提高内存使用效率，但在数据量陡增（毛刺）时将导致丢数据。
   3. 内存使用随着cpu数量的增加而增加
   4. 需要先初始化事件结构体，先将事件内容复制到这个结构体，然后将它复制到perbuf，总共复制了两次
2. **事件顺序无法保证**（event re-ordering）
   1. percpu的数据先生产的不一定先被消费

而ringbuf是5.8引入的，针对缺点做了优化，

1. 通过分配一个所有 CPU 共享的大缓冲区来提高内存使用效率
   1. “大缓冲区”意味着能更好地容忍数据量毛刺
   2. “共享”则意味着内存使用效率更高
   3. 不随cpu数量的增加而增加
   4. 首先申请为数据预留空间（reserve the space），预留成功后，应用就可以直接将准备发送的数据放到 ringbuf 了，从而节省了 perfbuf 中的第一次复制；如果预留不成功，也不用再继续执行了。
2. 事件顺序保证
   1. ringbuf采用所有CPU共享的内存， 保证 如果事件 A 发生在事件 B 之前，那 A 一定会先于 B 被提交，也会在 B 之前被消费。内部使用了一个**非常轻量级的 spin-lock。**

注意：Per-CPU buffer 特性的 **perfbuf 在理论上能支持更高的数据吞吐**， 但这只有在**每秒百万级事件**（millions of events per second）的场景下才会显现。

ringbuf 内部使用了一个**非常轻量级的 spin-lock**，这意味着如果 NMI context 中有竞争，data reservation 可能会失败。 因此，在 NMI context 中，如果 CPU 竞争非常严重，可能会 **导致丢数据，虽然此时 ringbuf 仍然有可用空间**。所以**需要注意、最好先试验一下的场景**：BPF 程序必须在 NMI (non-maskable interrupt) context 中执行时，例如处理 `cpu-cycles` 等 perf events 时。

### socket map

| 类型                                   | 介绍                                                         | 内核代码举例                                                        |
| ------------------------------------ | ---------------------------------------------------------- | ------------------------------------------------------------- |
| BPF\_MAP\_TYPE\_SOCKHASH             | 是一个hashmap，value存储的是socket描述符，可用于socket重定向                 |                                                               |
| BPF\_MAP\_TYPE\_SOCKMAP              | 是一个array map，value存储的是socket描述符，可用于socket重定向               |                                                               |
| BPF\_MAP\_TYPE\_REUSEPORT\_SOCKARRAY | 配合 `BPF_PROG_TYPE_SK_REUSEPORT` 类型的 BPF 程序使用，加速 socket 查找。 | <h4 id="id-1-samplesbpfhbm_kernhhost-bandwidth-manager"></h4> |
| BPF\_MAP\_TYPE\_SK\_STORAGE          | 为每个socket提供Socket-local storage，socket级别而不是map级别。          |                                                               |

### xdp map

如下四个map类型都是用作转发数据包。

|                              |                                                                                     |   |
| ---------------------------- | ----------------------------------------------------------------------------------- | - |
| BPF\_MAP\_TYPE\_DEVMAP       | 是一个array map，用来查询net device的，主要用于XDP程序中在 `bpf_redirect()` 时触发，把数据包转发到另个net device上。 |   |
| BPF\_MAP\_TYPE\_DEVMAP\_HASH | 同上，但是是一个hashmap                                                                     |   |
| BPF\_MAP\_TYPE\_CPUMAP       | 功能类似上两个，只不过是把数据包转到另一个CPU上                                                           |   |
| BPF\_MAP\_TYPE\_XSKMAP       | 可将原始 XDP 帧重定向到 AF\_XDP 套接字（XSK），这是内核中的一种新型地址系列，允许将帧从驱动程序重定向到用户空间，而无需穿越整个网络堆栈。       |   |

### 其他maps

| 类型                          | 介绍                                           | 内核代码举例                                                                                             |
| --------------------------- | -------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| BPF\_MAP\_TYPE\_QUEUE       | FIFO storage                                 |                                                                                                    |
| BPF\_MAP\_TYPE\_STRUCT\_OPS |                                              |                                                                                                    |
| BPF\_MAP\_TYPE\_LPM\_TRIE   | 高效的 longest-prefix matching，经常被用作Ip地址存储、路由查找 | <h4 id="id-2-samplesbpfxdp_router_ipv4_kernc"><code>samples/bpf/xdp_router_ipv4_kern.c</code></h4> |

## map的底层实现总结

这么多种类的map到底是怎么实现的呢？下面就让我们来一探究竟。

### 用户态maps和内核态maps是如何联系起来的

常见的map声明方式如下

```c
// Some codecode
struct bpf_map_def SEC("maps") my_bpf_map = {
  .type       = BPF_MAP_TYPE_HASH, 
  .key_size   = sizeof(int),
  .value_size   = sizeof(int),
  .max_entries = 100,
  .map_flags   = BPF_F_NO_PREALLOC,
};
```

这其实是C语言中很常见的一种结构体声明方式，似乎看起来没有什么技术含量，但是事实真的是这样吗？

我们先来提出几个问题

* 通过 `struct bpf_map_def` 定义的变量究竟是在用户空间创建还是内核中直接创建的？
* 我们都知道用户态通过操作表示maps的fd来操作Maps，那在内核中也是如此吗？
* 用户态的maps又是如何和内核中的Maps关联起来的呢？

这里请大家看这篇[揭秘 BPF map 前生今世](https://www.ebpf.top/post/map\_internal/)，已经讲解的很详细了，笔者就不再赘述了。

### 不同的maps是怎么实现的

通过上面的介绍我们可以看到maps种类繁多，那这么多种maps每个都有自己的底层实现吗？这么多种类实现起来是不是很麻烦？下面就让我们以hashmap为例来一探究竟。

最先来看的肯定是hashmap的结构体了

````c
// Some codecode
```c

struct bpf_htab {
	struct bpf_map map;
	struct bpf_mem_alloc ma;
	struct bpf_mem_alloc pcpu_ma;
	struct bucket *buckets;
	void *elems;
	union {
		struct pcpu_freelist freelist;
		struct bpf_lru lru;
	};
	struct htab_elem *__percpu *extra_elems;
	/* number of elements in non-preallocated hashtable are kept
	 * in either pcount or count
	 */
	struct percpu_counter pcount;
	atomic_t count;
	bool use_percpu_counter;
	u32 n_buckets;	/* number of hash buckets */
	u32 elem_size;	/* size of each element in bytes */
	u32 hashrnd;
	struct lock_class_key lockdep_key;
	int __percpu *map_locked[HASHTAB_MAP_LOCK_COUNT];
};
```
````

等一下，为什么bpf\_htab里面还有一个struct bpf\_map map？其他的map里也有吗？那我们再来看看array map吧

````c
// Some code
```cpp
struct bpf_array {
	struct bpf_map map;
	u32 elem_size;
	u32 index_mask;
	struct bpf_array_aux *aux;
	union {
		DECLARE_FLEX_ARRAY(char, value) __aligned(8);
		DECLARE_FLEX_ARRAY(void *, ptrs) __aligned(8);
		DECLARE_FLEX_ARRAY(void __percpu *, pptrs) __aligned(8);
	};
};
```
````

看来确实是的，也就是说这些maps主要都是基于这个struct bpf\_map map来实现的，只是其他的成员不太一样而已。下面我们来看看bpf\_map中都有什么东西：

````c
// Some code
```cpp

struct bpf_map {
	const struct bpf_map_ops *ops;
	struct bpf_map *inner_map_meta;
#ifdef CONFIG_SECURITY
	void *security;
#endif
	enum bpf_map_type map_type;
	u32 key_size;
	u32 value_size;
	u32 max_entries;
	u64 map_extra; /* any per-map-type extra fields */
	u32 map_flags;
	u32 id;
	struct btf_record *record;
	int numa_node;
	u32 btf_key_type_id;
	u32 btf_value_type_id;
	u32 btf_vmlinux_value_type_id;
	struct btf *btf;
#ifdef CONFIG_MEMCG_KMEM
	struct obj_cgroup *objcg;
#endif
	char name[BPF_OBJ_NAME_LEN];
	struct mutex freeze_mutex;
	atomic64_t refcnt;
	atomic64_t usercnt;
	/* rcu is used before freeing and work is only used during freeing */
	union {
		struct work_struct work;
		struct rcu_head rcu;
	};
	atomic64_t writecnt;
	/* 'Ownership' of program-containing map is claimed by the first program
	 * that is going to use this map or by the first program which FD is
	 * stored in the map to make sure that all callers and callees have the
	 * same prog type, JITed flag and xdp_has_frags flag.
	 */
	struct {
		spinlock_t lock;
		enum bpf_prog_type type;
		bool jited;
		bool xdp_has_frags;
	} owner;
	bool bypass_spec_v1;
	bool frozen; /* write-once; write-protected by freeze_mutex */
	bool free_after_mult_rcu_gp;
	bool free_after_rcu_gp;
	atomic64_t sleepable_refcnt;
	s64 __percpu *elem_count;
};

```
````

这里重点来看一下const struct bpf\_map\_ops \*ops;可以看到不同的maps对用户呈现出的操作接口都一样是因为自己做了“函数重载”。

````c
// Some codecode
```c
const struct bpf_map_ops htab_map_ops = {
	.map_meta_equal = bpf_map_meta_equal,
	.map_alloc_check = htab_map_alloc_check,
	.map_alloc = htab_map_alloc,
	.map_free = htab_map_free,
	.map_get_next_key = htab_map_get_next_key,
	.map_release_uref = htab_map_free_timers,
	.map_lookup_elem = htab_map_lookup_elem,
	.map_lookup_and_delete_elem = htab_map_lookup_and_delete_elem,
	.map_update_elem = htab_map_update_elem,
	.map_delete_elem = htab_map_delete_elem,
	.map_gen_lookup = htab_map_gen_lookup,
	.map_seq_show_elem = htab_map_seq_show_elem,
	.map_set_for_each_callback_args = map_set_for_each_callback_args,
	.map_for_each_callback = bpf_for_each_hash_elem,
	.map_mem_usage = htab_map_mem_usage,
	BATCH_OPS(htab),
	.map_btf_id = &htab_map_btf_ids[0],
	.iter_seq_info = &iter_seq_info,
};
```
````

其他东西笔者就不一一介绍了，感兴趣的自己可以去琢磨琢磨源码，推荐一篇文章[BPF Map 内核实现](https://arthurchiao.art/blog/bpf-advanced-notes-3-zh/)。









