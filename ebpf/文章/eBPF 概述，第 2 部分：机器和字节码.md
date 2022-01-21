在我们的[第一篇文章](https://kubesphere.com.cn/blogs/ebpf-guide/)中，我们介绍了 eBPF VM、它刻意的设计限制以及如何从用户空间进程与其交互。如果您还没有阅读它，您可能需要在继续阅读本篇文章之前阅读上一篇文章，因为如果没有适当了解，直接从机器和字节码细节开始学习可能会很困难。如有疑问，请参阅第一部分开头的流程图。

本系列的第二部分更深入地研究了第一部分中研究的 eBPF VM 和程序。掌握这种底层知识不是强制性的，但对于本系列的其余部分来说是非常有用的基础，我们将在其中检查建立在这些机制之上的更高级别的工具。

## 虚拟机

eBPF 是一个 RISC 寄存器机，共有 [11 个 64 位寄存器](https://github.com/torvalds/linux/blob/v4.20/include/uapi/linux/bpf.h#L45)，一个程序计数器和一个 512 字节固定大小的堆栈。九个寄存器是通用读写的，一个是只读堆栈指针，程序计数器是隐式的，即我们只能跳转到计数器的某个偏移量。VM 寄存器始终为 64 位宽（即使在 32 位 ARM 处理器内核中运行！）并且如果最高有效的 32 位为零，则支持 32 位子寄存器寻址 - 这将在第四部分在嵌入式设备上交叉编译和运行 eBPF 程序非常有用。

这些寄存器是：

|           |                                                            |
| :-------- | :--------------------------------------------------------- |
| r0：      | 存储函数调用和当前程序退出代码的返回值                     |
| r1 - r5： | 作为函数调用的参数，在程序开始时 r1 包含 “上下文” 参数指针 |
| r6 - r9： | 这些在内核函数调用之间被保留                               |
| r10：     | 每个 eBPF 程序512字节堆栈的只读指针                        |

在加载时提供的 eBPF [程序类型](https://github.com/torvalds/linux/blob/v4.20/include/uapi/linux/bpf.h#L136)准确地决定了哪些内核函数子集可以调用，以及在程序启动时通过 r1 提供的 “上下文” 参数。r0 中存储的程序退出值的含义也是由程序类型决定的。

每个函数调用在寄存器 r1 - r5 中最多可以有 5 个参数；这适用于 eBPF 到 eBPF 和内核函数的调用。寄存器 r1 - r5 只能存储数字或指向堆栈的指针（作为参数传递给函数），从不直接指向任意内存的指针。所有内存访问都必须先将数据加载到 eBPF 堆栈中，然后才能在 eBPF 程序中使用它。此限制有助于 eBPF 验证器，它简化了内存模型以实现更轻松的正确性检查。

BPF 可访问的内核“辅助”函数由内核核心（不可通过模块扩展）通过类似于定义系统调用的 API 定义，使用 [BPF_CALL_*](https://github.com/torvalds/linux/blob/v4.20/include/linux/filter.h#L441) 宏。[bpf.h](https://github.com/torvalds/linux/blob/v4.20/include/uapi/linux/bpf.h#L420) 试图为所有 BPF 可访问的内核“辅助”函数提供参考。例如 [bpf_trace_printk](https://github.com/torvalds/linux/blob/v4.20/kernel/trace/bpf_trace.c#L163) 的定义使用 BPF_CALL_5 和 5 对类型/参数名称。定义[参数数据类型](https://github.com/torvalds/linux/blob/v4.20/kernel/trace/bpf_trace.c#L276)很重要，因为在每个 eBPF 程序加载时，eBPF 验证器确保寄存器数据类型与被调用方参数类型匹配。

eBPF 指令也是固定大小的 64 位编码，大约 100 条指令（目前...）分为 [8 类](https://github.com/torvalds/linux/blob/v4.20/include/uapi/linux/bpf_common.h#L5)。VM 支持来自通用内存（ map 、堆栈、“上下文”如数据包缓冲区等）的 1 - 8 字节加载/存储、向前/向后（非）条件跳转、算术/逻辑运算和函数调用。如需深入了解操作码格式，请参阅 Cilium 项目[指令集文档](https://cilium.readthedocs.io/en/latest/bpf/#instruction-set)。IOVisor 项目还维护了一个有用的[指令规范](https://github.com/iovisor/bpf-docs/blob/master/eBPF.md)。

在本系列第一部分研究的示例中，我们使用了一些有用的[内核宏](https://github.com/torvalds/linux/blob/v4.20/samples/bpf/bpf_insn.h)来使用以下[结构](https://github.com/torvalds/linux/blob/v4.20/include/uapi/linux/bpf.h#L64)创建 eBPF 字节码指令数组（所有指令都以这种方式编码）：

```
struct bpf_insn { 
	__u8 代码；/* opcode */ 
	__u8 dst_reg:4; /* dest register */ 
	__u8 src_reg:4; /* source register */ 
	__s16 off; /* signed offset */ 
	__s32 imm; /* signed immediate constant */ 
}; 

msb                                                        lsb
+------------------------+----------------+----+----+--------+
|immediate               |offset          |src |dst |opcode  |
+------------------------+----------------+----+----+--------+
```

让我们看一下 [BPF_JMP_IMM](https://github.com/torvalds/linux/blob/v4.20/samples/bpf/bpf_insn.h#L167) 指令，它针对立即值对条件跳转进行编码。下面的宏注释对指令逻辑应该是不言自明的。操作码编码指令类 BPF_JMP 、操作（通过 BPF_OP 位域传递以确保正确性）和表示它是对立即数/常量值 BPF_K 的操作的标志进行编码。

```
#define BPF_OP(code) ((code) & 0xf0) 
#define BPF_K 0x00 

/* 针对立即数的条件跳转，if (dst_reg 'op' imm32) goto pc + off16 */ 

#define BPF_JMP_IMM(OP, DST, IMM, OFF) \ 
	((struct bpf_insn) { \ 
		.code = BPF_JMP | BPF_OP(OP) | BPF_K, \ 
		.dst_reg = DST, \ 
		.src_reg = 0 
		, \ 
		.off = OFF, \ .imm = IMM })
```

如果我们计算值或反汇编包含 BPF_JMP_IMM ( BPF_JEQ , BPF_REG_0 , 0 , 2 ) 的 eBPF 字节码二进制文件，我们会发现它是 0x020015 格式。这个特定的字节码非常频繁地用于测试存储在 r0 中的函数调用的返回值；如果 r0 == 0，它会跳过接下来的 2 条指令。

## 重温我们的字节码

现在我们已经掌握了必要的知识来完全理解本系列第一部分中使用的字节码 eBPF 示例，我们将逐步解释它。请记住，[sock_example.c](https://github.com/torvalds/linux/blob/v4.20/samples/bpf/sock_example.c) 是一个简单的用户态程序，它使用 eBPF 来计算在环回接口上接收到的 TCP、UDP 和 ICMP 协议数据包的数量。

在高层次上，代码所做的是从接收到的数据包中读取协议编号，然后将其推送到 eBPF 堆栈上，用作 map_lookup_elem 调用的索引，该调用获取相应协议的数据包计数。map_lookup_elem 函数采用 r0 中的索引（或键）指针和 r1 中的 map 文件描述符。如果查找调用成功，r0 将包含一个指向存储在协议索引处的 map 值的指针。然后我们原子地增加 map 值并退出。

```
BPF_MOV64_REG(BPF_REG_6, BPF_REG_1),
```

当 eBPF 程序启动时，上下文（在这种情况下是数据包缓冲区）由 r1 中的地址指向。r1 将在函数调用期间用作参数，因此我们也将其存储在 r6 中作为备份。

```
BPF_LD_ABS(BPF_B, ETH_HLEN + offsetof(struct iphdr, protocol) /* R0 = ip->proto */),
```

该指令将一个字节（ BPF_B ）从上下文缓冲区（在本例中为网络数据包缓冲区）中的偏移量加载到 r0 中，因此我们提供要加载到 r0 的 [iphdr 结构](https://github.com/torvalds/linux/blob/v4.20/include/uapi/linux/ip.h#L86)中的协议字节的偏移量。

```
BPF_STX_MEM(BPF_W, BPF_REG_10, BPF_REG_0, -4), /* *(u32 *)(fp - 4) = r0 */
```

将包含先前读取的协议的字 ( BPF_W ) 压入堆栈（由 r10 指向，以偏移量 -4 字节开头）。

```
BPF_MOV64_REG(BPF_REG_2, BPF_REG_10), 
BPF_ALU64_IMM(BPF_ADD, BPF_REG_2, -4), /* r2 = fp - 4 */
```

将堆栈地址指针移至 r2 并减去 4，因此现在 r2 指向协议值，用作下一次 map 键查找的参数。

```
BPF_LD_MAP_FD(BPF_REG_1, map_fd),
```

将本地进程内的文件描述符引用包含协议包计数的 map 到 r1 寄存器。

```
BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0, BPF_FUNC_map_lookup_elem),
```

使用 r2 指向的堆栈中的协议值作为键执行 map 查找调用。结果存储在 r0 中：指向由键索引的值的指针地址。

```
BPF_JMP_IMM(BPF_JEQ, BPF_REG_0, 0, 2),
```

还记得 0x020015 格式吗？这与第一部分的字节码相同。如果 map 查找没有成功，则 r0 == 0 所以我们跳过接下来的两条指令。

```
BPF_MOV64_IMM(BPF_REG_1, 1), /* r1 = 1 */ 
BPF_RAW_INSN(BPF_STX | BPF_XADD | BPF_DW, BPF_REG_0, BPF_REG_1, 0, 0), /* xadd r0 += r1 */
```

增加 r0 指向的地址处的 map 值。

```
BPF_MOV64_IMM(BPF_REG_0, 0), /* r0 = 0 */ 
BPF_EXIT_INSN(),
```

将 eBPF retcode 设置为 0 并退出。

尽管这个 sock_example 逻辑非常简单（它只是在 map 中增加的一些数字），但在原始字节码中实现或理解是困难的。更复杂的任务在像这样的汇编程序中完成时变得极其困难。展望未来，我们将开始使用更高级的语言和工具，以更少的工作开启更强大的 eBPF 用例。

## 总结

在这一部分中，我们仔细观察了 eBPF 虚拟机的寄存器和指令集，了解了 eBPF 可访问的内核函数是如何从字节码中调用的，以及它们是如何被核心内核通过类似 syscall 的特殊目的 API 定义的。我们也完全理解了第一部分例子中使用的字节码。还有一些未探索的领域，如创建多个 eBPF 程序函数或链式 eBPF 程序以绕过 Linux 发行版的 4096 条指令限制。也许我们会在以后的文章中探讨这些。

现在，主要的问题是编写原始字节码很困难的，这非常像编写汇编代码，而且编写效率低下。在第三部分中，我们将开始研究使用高级语言编译成 eBPF 字节码，到此为止我们已经了解了虚拟机工作的底层基础知识。
