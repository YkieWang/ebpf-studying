# ebpf tailcall vs bpf2bpf

## 什么是tailcall/fuction call？

推荐这篇文章[从 BPF to BPF Calls 到 Tail Calls](https://www.ebpf.top/post/bpf2pbpf\_tail\_call/)，另外这里做一些总结：

|       | tailcall                                                                                                                                                                                                           | func2func                                                                                                                                                     |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 内存使用  | <p>小，栈复用，当然也无法返回，毕竟调用者的栈都没有了."</p><pre><code>The kernel stack is precious, so this helper reuses the current
stack frame and jumps into another BPF program without adding
extra call frame.
</code></pre><p>"</p> | 大（相对）                                                                                                                                                         |
| 镜像大小  | 大，全部是内联                                                                                                                                                                                                            | 小（相对）                                                                                                                                                         |
| 代码复用性 | 低                                                                                                                                                                                                                  | 高（相对）                                                                                                                                                         |
| 数量限制  | 33个                                                                                                                                                                                                                | 256个                                                                                                                                                          |
| 上下文传递 | 一般用per cpu map                                                                                                                                                                                                     | 函数参数传递                                                                                                                                                        |
| 限制    | 只能相同类型的程序进行尾调用                                                                                                                                                                                                     | <p></p><pre><code>Only allow bpf-to-bpf calls for root only and for non-hw-offloaded programs.
These restrictions can be relaxed in the future.
</code></pre> |

## 性能损耗

tailcall的性能损耗推荐这篇文章[The Cost of BPF Tail Calls](https://pchaigno.github.io/ebpf/2021/03/22/cost-bpf-tail-calls.html)

## 为什么要从tailcall到bpf2bpf

tailcall ，fuction call出现的顺序是先有tailcall然后再有function call，然后再混合使用。主要是因为tailcall在开发中“子函数”只能使用inline，这对代码复用不友好，并且编译出来的文件会更大，而fuction call的目的就是想让ebpf程序能跟普通程序一样进行调用，后续的发展预期是能编译成不同的.o文件提高代码的复用性。

| Tail calls | 4.2 | [`04fd61ab36ec`](https://github.com/torvalds/linux/commit/04fd61ab36ec065e194ab5e74ae34a5240d992bb) |
| ---------- | --- | --------------------------------------------------------------------------------------------------- |

| bpf2bpf function calls | 4.16 | [`cc8b0b92a169`](https://github.com/torvalds/linux/commit/cc8b0b92a1699bc32f7fec71daa2bfc90de43a4d) |
| ---------------------- | ---- | --------------------------------------------------------------------------------------------------- |

| Mixing bpf2bpf function calls and tailcalls (x86\_64)   | 5.10 | [`e411901c0b77`](https://github.com/torvalds/linux/commit/e411901c0b775a3ae7f3e2505f8d2d90ac696178) |
| ------------------------------------------------------- | ---- | --------------------------------------------------------------------------------------------------- |
| Mixing bpf2bpf function calls and tailcalls (arm64)     | 6.0  | [`d4609a5d8c70`](https://github.com/torvalds/linux/commit/d4609a5d8c70d21b4a3f801cf896a3c16c613fe1) |
| Mixing bpf2bpf function calls and tailcalls (s390)      | 6.3  | [`dd691e847d28`](https://github.com/torvalds/linux/commit/dd691e847d28ac5f8b8e3005be44fd0e46722809) |
| Mixing bpf2bpf function calls and tailcalls (loongarch) | 6.4  |                                                                                                     |

