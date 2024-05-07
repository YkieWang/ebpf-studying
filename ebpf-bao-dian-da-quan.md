# ebpf宝典大全

## 各大公司开源的教程，适合系统学习

[cilium bpf文档](https://docs.cilium.io/en/stable/bpf/)

[ebpf官方教程](https://ebpf.io/what-is-ebpf/)

## 大佬写的教程，适合工作的时候有针对性的看

[ebpf tracing技术](https://www.brendangregg.com/blog/2019-01-01/learn-ebpf-tracing.html)教程，大佬Brendan Gregg写的，这是他的网站 [Brendan Gregg's Blog](https://www.brendangregg.com/blog/index.html)

[ebpf xdp教程](https://github.com/xdp-project/xdp-tutorial)，大佬们写的开源教程

[深入理解cilium的datapath](https://arthurchiao.art/blog/understanding-ebpf-datapath-in-cilium-zh/#step-2xdp-%E7%A8%8B%E5%BA%8F%E5%A4%84%E7%90%86)，介绍**数据包是如何穿过 network datapath（网络数据路径）的，**包括从硬件到 内核，再到用户空间。非常值得一看，不一定要记住，但是在进行实际开发的时候可以来查阅一下，以明确自己的程序到底应该挂载在哪里。

一些[翻译的文章](https://arthurchiao.art/articles-zh/)，不全是讲ebpf的，很高质量

## 内核相关

[内核bpf详细介绍文档](https://www.kernel.org/doc/html/latest/bpf/index.html)，适合学习

[bpf内核相关问题Q\&A](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/bpf/bpf\_design\_QA.rst)，介绍一些对bpf的疑问，和bpf长期的发展方向

[ebpf在内核中的feature和对应的内核版本](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md)，在进行实际开发的时候需要明确想要的功能在内核的哪个版本。

## 动手实验教程

[ebpf lab](https://ebpf.io/labs/)，一个能一步一步带你动手做实验学习ebpf的网站，由[isovalent](https://twitter.com/isovalent)开发。

[探索eppf汇编代码的网站](https://godbolt.org/)，由几个大公司支持的。

## ebpf儿童指南

[https://ebpf.io/books/buzzing-across-space-illustrated-childrens-guide-to-ebpf.pdf](https://ebpf.io/books/buzzing-across-space-illustrated-childrens-guide-to-ebpf.pdf)

300个月大也算儿童吧

## ebpf干货知识点和热点资讯

[https://www.ebpf.top/post/](https://www.ebpf.top/post/)

## ebpf书籍推荐

