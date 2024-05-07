---
description: 用好一项技能的前提是充分了解它
---

# ebpf概览

## ebpf架构

ebpf是事件驱动型程序，下图是编译、加载、校验的架构。

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

1. 使用高级语言如c编写代码
2. 经clang编译成bpf字节码，bpf字节码和maps相互配合，maps是bpf程序所用的内存（可简单理解为堆，后面再做详细探讨）。
3. bpf字节码经过verify校验安全性
4. 经过jit进行即时编译，提高运行速度

其中所涉及到的maps/verifier/JIT在后面的文章中会逐一介绍

## ebpf应用场景一览

<figure><img src=".gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

verify

加载 eBPF 程序的进程拥有所需的能力（权限）。除非启用了非特权 eBPF，否则只有特权进程才能加载 eBPF 程序。 程序不会崩溃或以其他方式损害系统。 程序始终运行到完成（即程序不会永远处于循环状态，阻碍进一步处理）。

jit即时编译器

即时编译（JIT）步骤将程序的通用字节码翻译成机器专用指令集，以优化程序的执行速度。这样，eBPF 程序就能像本地编译的内核代码或作为内核模块加载的代码一样高效运行。

maps

eBPF 程序的一个重要方面是共享收集的信息和存储状态的能力。为此，eBPF 程序可以利用 eBPF map的概念（很像堆的概念），在各种数据结构中存储和检索数据。eBPF maps既可以从 eBPF 程序中访问，也可以通过系统调用从用户空间的应用程序中访问。

helper calls

eBPF 程序不能调用任意内核函数。如果允许这样做，就会将 eBPF 程序绑定到特定的内核版本，使程序的兼容性变得复杂。相反，eBPF 程序可以调用辅助函数，这是内核提供的众所周知的稳定 API。

tail call & fuction calls

普通程序会出现线程之间的交互调用等，对应的，eBPF 程序可通过尾部和函数调用的概念进行组合。函数调用允许在 eBPF 程序中定义和调用函数。尾部调用可以调用和执行另一个 eBPF 程序，并替换执行上下文，类似于普通进程的 execve() 系统调用。

<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

ebpf 安全性

权力越大，责任也越大。

ebpf运行在内核中，怎么保证它不会危及到内核的正常运作？ebpf从权限和校验两个方面给出了答案。

所需的权限 除非启用了非特权 eBPF，否则所有打算将 eBPF 程序载入 Linux 内核的进程都必须在特权模式（root）下运行，或者需要 CAP\_BPF 功能。这意味着不受信任的程序无法加载 eBPF 程序。如果启用了非特权 eBPF，那么非特权进程就可以加载某些 eBPF 程序，但其功能会受到限制，而且对内核的访问也会受到限制。

检验：每个程序被load之前都要经过verify模块的校验，它扮演着一个十分严格的reviewer的角色，来杜绝代码中一些人工都很难发现的问题。

[hardening](https://en.wikipedia.org/wiki/Hardening\_\(computing\))

在成功完成验证后，eBPF 程序会根据程序是从有权限进程还是无权限进程加载的情况运行一个加固程序。该步骤包括

程序执行保护：存放 eBPF 程序的内核内存受到保护并被设置为只读。如果出于任何原因，无论是内核漏洞还是恶意操作，试图修改 eBPF 程序，内核都会崩溃，而不会允许继续执行被破坏/操纵的程序。&#x20;

针对 Spectre 的缓解措施：在猜测情况下，CPU 可能会错误预测分支，并留下可观察到的副作用，这些副作用可通过侧信道提取。举几个例子：

eBPF 程序会屏蔽内存访问，以便将瞬时指令下的访问重定向到受控区域；

验证器也会跟踪只有在投机执行下才能访问的程序路径；

JIT 编译器会在尾部调用无法转换为直接调用的情况下发出 Retpolines。&#x20;

常量盲化：代码中的所有常量都被屏蔽，以防止 JIT 喷射攻击。这样可以防止攻击者将可执行代码作为常量注入，如果内核存在其他漏洞，攻击者就可以跳转到 eBPF 程序的内存部分执行代码。



<figure><img src=".gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>
