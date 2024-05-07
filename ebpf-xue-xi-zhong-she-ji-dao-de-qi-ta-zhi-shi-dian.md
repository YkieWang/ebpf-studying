# ebpf学习中涉及到的其他知识点

## 匿名句柄fd

一般情况下，一个句柄fd对应一个file结构体，一个file结构体关联一个inode，`file/dentry/inode`  这三驾马车是一定要配齐的，就算是匿名的（无 path，无效 dentry），对于 file 结构体来说，一定要绑定 inode 和 dentry ，**哪怕是伪造的、不完整的 inode**。

如果我们并不需要一个真实的inode，就可以使用匿名fd。匿名 fd 背后的是一个叫做 anon\_inodefs 的内核文件系统（ 位于 `fs/anon_inodes.c`），这个文件系统极其简单，整个文件系统只有一个 inode ，这个 inode 是文件系统初始化的时候创建好的。之后，所有需要一个匿名 inode 的句柄都直接跟这个 inode 关联即可。。其中的priv字段用来存储prog，op字段用来存储bpf\_prog\_fops。后续在attach过程中会使用到priv来读取prog。

我们经常在 `/proc/${pid}/fd/` 下面能看到 `anon_inode :` 前缀的句柄，如下：

```jsx
root@ubuntu:~/temp# ll /proc/5398/fd

lr-x------ 1 root root 64 Aug 24 09:39 11 -> anon_inode:inotify
lrwx------ 1 root root 64 Aug 24 09:39 4 -> anon_inode:[eventpoll]
lrwx------ 1 root root 64 Aug 24 09:39 5 -> anon_inode:[signalfd]
lrwx------ 1 root root 64 Aug 24 09:39 7 -> anon_inode:[timerfd]
lrwx------ 1 root root 64 Aug 24 09:39 9 -> anon_inode:[eventpoll]
```

如果是正常的文件句柄，一般显式的是一个路径：

```jsx
root@ubuntu:~/temp# ll /proc/5398/fd

lr-x------ 1 root root 64 Aug 24 09:39 10 -> /proc/5398/mountinfo
lr-x------ 1 root root 64 Aug 24 09:39 12 -> /proc/swaps
```

当然 path 只是一个浅层次的感官，因为对于 socket 句柄来说也不算有 path ，所以这个匿名其实**匿的是 inode** 。

## ELF文件

### ELF文件分类

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1161a44f-d8be-469b-ac13-a0fd5ed893a3/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f7e6b3bd-69d4-4461-b9ce-c1e8f488725d/Untitled.png)

### 目标文件长什么样

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5909140a-e797-4120-87bd-eac55e3d975a/Untitled.png)

fileheader：整个文件的文件属性。

* 文件是否可执行（如果是可执行文件会记录程序的入口地址）
* 是静态链接还是动态链接
* 目标硬件
* 目标操作系统
* 一个段表
  * 是一个描述文件中各个段的数组，表述了各个段的属性和在文件中的偏移。

.text段：C语言编译后的机器码

.data：已初始化的全局变量和静态局部变量

.bss：未初始化的全局变量和静态局部变量。

| DATA（data segment）     | BSS（Block Started by Symbol）                      |
| ---------------------- | ------------------------------------------------- |
| 已经初始化的全局变量和静态局部变量和它们的值 | 未初始化的全局变量和静态局部变量                                  |
| 存储在目标文件当中，占用实际的空间      | 只记录数据大小总和（即数据的开始地址和结束地址），链接器得到这个大小的内存块，紧跟在data段后面 |
|                        | 系统上电前，操作系统把.bss段初始化为0                             |
|                        | 节省空间，不必在可执行文件中存储很多全为0的数据                          |

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9d49708f-195d-4e48-8f3f-90e8d44e9083/Untitled.png)

### 自定义段

编译时为变量指定段

```jsx
__attribute__((section("name")))
RealView Compilation Tools for µVision Compiler Reference Guide Version 4.0 
 
Home > Compiler-specific Features > Variable attributes > __attribute__((section("name"))) 
 
4.5.6. __attribute__((section("name")))
Normally, the ARM compiler places the objects it generates in sections like data and bss. However, you might require additional data sections or you might want a variable to appear in a special section, for example, to map to special hardware. The section attribute specifies that a variable must be placed in a particular data section. If you use the section attribute, read-only variables are placed in RO data sections, read-write variables are placed in RW data sections unless you use the zero_init attribute. In this case, the variable is placed in a ZI section.
 
Note
This variable attribute is a GNU compiler extension supported by the ARM compiler.
 
Example
/* in RO section */
const int descriptor[3] __attribute__ ((section ("descr"))) = { 1,2,3 };
/* in RW section */
long long rw[10] __attribute__ ((section ("RW")));
/* in ZI section *
long long altstack[10] __attribute__ ((section ("STACK"), zero_init));/
```

编译时为函数指定段

```jsx
__attribute__((section("name")))
RealView Compilation Tools for µVision Compiler Reference Guide Version 4.0 
 
Home > Compiler-specific Features > Function attributes > __attribute__((section("name"))) 
 
4.3.13. __attribute__((section("name")))
The section function attribute enables you to place code in different sections of the image.
 
Note
This function attribute is a GNU compiler extension that is supported by the ARM compiler.
 
Example
In the following example, Function_Attributes_section_0 is placed into the RO section new_section rather than .text.
 
void Function_Attributes_section_0 (void)
    __attribute__ ((section ("new_section")));
void Function_Attributes_section_0 (void)
{
    static int aStatic =0;
    aStatic++;
}
In the following example, section function attribute overrides the #pragma arm section setting.
 
#pragma arm section code="foo"
  int f2()
  {
      return 1;
  }                                  // into the 'foo' area
  __attribute__ ((section ("bar"))) int f3()
  {
      return 1;
  }                                  // into the 'bar' area
  int f4()
  {
      return 1;
  }                                  // into the 'foo' area
#pragma arm section
```

RCU

read copy update，随意读取，写的时候先拷贝一份，在拷贝的数据上进行修改，然后一性替换。

适用于频繁读取，少量写。

grace period指的是，多个读的时候，等待一段时间到当前的读操作全部结束，这段时间就是grace period。
