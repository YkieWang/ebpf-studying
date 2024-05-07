# ebpf全局变量和常量

在之前脚手架文件讲解那一篇[例2](ebpf-gong-ju/bpf-object-skeleton-file.md#jiao-shou-jia-wen-jian-ju-li-2)，可以看到全局变量和常量的存储位置分别是bss 和 rodata?

注意，虽然数据是存储在map中的，但是在内核态ebpf程序中是直接访问的，像操作普通全局变量一样，只是在用户态是需要通过map进行访问。另外要注意全局变量的并发问题。



热知识，printk中的字符串也在rodata中

libbpf-bootstrap/examples/c/tc.bpf.c

```bash
// Some codecode
parallels@ubuntu-linux-22-04-02-desktop:/media/psf/code/libbpf-bootstrap/examples/c$ sudo bpftool map list
[sudo] password for parallels: 
1: hash  flags 0x0
	key 9B  value 1B  max_entries 500  memlock 8192B
78: array  name tc_bpf.rodata  flags 0x80
	key 4B  value 36B  max_entries 1  memlock 4096B
	btf_id 183  frozen
parallels@ubuntu-linux-22-04-02-desktop:/media/psf/code/libbpf-bootstrap/examples/c$ sudo bpftool map dump id 78
[{
        "value": {
            ".rodata": [{
                    "tc_ingress.____fmt": "Got IP packet: tot_len: %d, ttl: %d"
                }
            ]
        }
    }
]
parallels@ubuntu-linux-22-04-02-desktop:/media/psf/code/libbpf-bootstrap/examples/c$ 

```
