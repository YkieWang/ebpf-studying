---
description: 工欲善其事，必先利其器
---

# ebpf工具

ebpf近些年的繁荣景象得益于这些开发工具的出现，它们大大简化了开发者的开发流程。

## bcc:_BPF Compiler Collection_

[_项目指路_](https://github.com/iovisor/bcc)

bcc的所有工具都是基于ebpf开发的，是一个工具集，它封装了一些编译挂载的操作以及一些库，让用户能够更简单便捷的使用高级语言如c/c++/python开发bpf程序并挂载到内核中。在运行的时候，BCC 会调用 LLVM，把 BPF 源代码编译为字节码，再加载到内核中运行，这也意味着它的体量较大。

1. 用C语言编写嵌入到内核中的代码

<pre class="language-clike"><code class="lang-clike">int hello_world(void *ctx)
{
<strong>    bpf_trace_printk("Hello, World!");
</strong>    return 0;
}
</code></pre>

2. 用Python基于bcc库写用户态文件

```python
#!/usr/bin/env python3
# 1) import bcc library
from bcc import BPF

# 2) load BPF program
b = BPF(src_file="hello.c")
# 3) attach kprobe
b.attach_kprobe(event="do_sys_openat2", fn_name="hello_world")
# 4) read and print /sys/kernel/debug/tracing/trace_pipe
b.trace_print()
```

运行上述代码。eBPF 程序需要以 root 用户来运行，非 root 用户需要加上 sudo 来执行

```python
sudo python3 hello.py
```

## bpftool

linux内核自带的对ebpf程序和map进行检查和操作的工具，比如可以已查看挂载情况，map信息，生成Skeleton文件等。

[项目指路](https://github.com/libbpf/bpftool)

```bash
xxx@raspberrypi:~ $ sudo bpftool prog list | tail -n 4
xlated 64B  not jited  memlock 4096B
21: xdp  name xdp_bridge_prog  tag 610be6df09f4715b  gpl
loaded_at 2023-05-31T13:57:17+0800  uid 0
xlated 704B  not jited  memlock 4096B  map_ids 1
```

## libbpf

首先它是一个基于c语言的library，它包含了bpf loader工具，能够将编译后的bpf文件加载到内核中去。它提供了一套 API，用于加载、验证和将 eBPF 程序附加到内核。它也提供了处理 BPF maps 和 BPF 系统调用的功能。

基于libbpf开发bpf程序的步骤：

* **开发 eBPF 程序，并把源文件命名为 <程序名>.bpf.c；**
* **编译 eBPF 程序为字节码，然后再调用**  [**bpftool gen skeleton  为 eBPF 字节码生成脚手架头文件**](bpf-object-skeleton-file.md)**；**
* **开发用户态程序，引入生成的脚手架头文件后，加载 eBPF 程序并挂载到相应的内核事件中。**

```c
clang -g -O2 -target bpf -D__TARGET_ARCH_x86_64 -I/usr/include/x86_64-linux-gnu -I. -c execsnoop.bpf.c -o execsnoop.bpf.o
bpftool gen skeleton execsnoop.bpf.o > execsnoop.skel.h
```

**其中，clang 的参数  -target bpf  表示要生成 BPF 字节码，-D\_\_TARGET\_ARCH\_x86\_64  表示目标的体系结构是 x86\_64，而  -I  则是引入头文件路径。bpftool把根据字节码生成脚手架头文件execsnoop.skel.h ，这个头文件包含了 BPF 字节码和相关的管理函数。当用户态程序引入这个头文件并编译之后，只需要分发最终用户态程序生成的二进制文件到生产环境即可（如果用户态程序使用了其他的动态库，还需要分发动态库）。**

## **其他**

**当然，我们也可以不使用上述的任何方式，比如说我们可以开发编译一个bpf文件，然后用tc工具加载。**



