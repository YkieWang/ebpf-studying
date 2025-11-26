# ebpf verify报错定位手段

错误信息：

Error: Failed to load BPF program, prog\_name = reuseport\_prog, fd = -1, kernel log: back-edge from insn 200 to 201

processed 0 insns (limit 1000000) max\_states\_per\_insn 0 total\_states 0 peak\_states 0 mark\_read 0

报错信息

Error: Failed to load BPF program, prog\_name = reuseport\_prog, fd = -1, kernel log: jump out of range from insn 32 to 647 processed 0 insns (limit 1000000) max\_states\_per\_insn 0 total\_states 0 peak\_states 0 mark\_read 0

这个错误提示“jump out of range from insn 32 to 647”表示你的 BPF 程序在指令 32 尝试跳转到指令 647，但这个跳转超出了 BPF 允许的范围。BPF 对跳转的距离有限制，通常在 128 个指令之内。

如何从指令标号定位到具体的代码

首先在bpf程序的编译过程添加调试信息在编译中添加以下选项

```
-O0 -g
```

例如

clang -O0 -target bpf -g -c test.c -o test.o

然后可以使用如下方法定位指令号对应的具体代码，注意一行代码通常对应多个指令

要找到 BPF 程序中指令 32 和 647 对应的具体代码，可以按照以下步骤进行：

#### 1. 使用 `llvm-objdump`

如果你有编译后的 BPF 目标文件，可以使用 `llvm-objdump` 来反汇编并查看指令：

```bash
llvm-objdump -d -S bpf/test.o
```

* `-S` 选项会显示反汇编的同时显示源代码，方便你直接看到对应的行号。

#### 2. 查看反汇编输出

在反汇编的输出中，指令通常会以如下形式列出：

llvm-objdump -d -S test.o

```
0000000000000000 <function_name>:
       0:	...           ; 代码行
       1:	...           ; 代码行
       ...
       32:	...           ; 代码行
       ...
       647:	...          ; 代码行
```

你可以找到对应的行号，特别是指令 32 和 647。

#### 3. 使用 `addr2line`

如果反汇编输出没有足够的信息，你可以使用 `addr2line` 来查找特定地址对应的源代码行。确保你在编译时使用了 `-g` 选项生成调试信息。

例如，使用以下命令：

```bash
addr2line -e bpf/kes_reuseport_helper.o 0000000000000030
addr2line -e bpf/kes_reuseport_helper.o 0000000000000280
```

请注意，这里的地址要根据你反汇编输出中的实际地址进行调整。

#### 4. 使用 GDB

你也可以使用 `gdb` 来加载目标文件并查找指令对应的行：

```bash
gdb bpf/kes_reuseport_helper.o
(gdb) info line *0x0000000000000030
(gdb) info line *0x0000000000000280
```

#### 5. 逐步查找

如果程序比较简单，你也可以手动从头开始查找代码行，直到找到对应的指令。
