> BPF 是 Linux 内核中一个非常灵活与高效的**类虚拟机**组件， 能够在许多内核 hook 点安全地执行字节码。本文简单整理了 eBPF 的技术原理与应用场景。

## eBPF 起源

BPF 全称为 **B**erkeley **P**acket **F**ilter，顾名思义，BPF 最初用于数据包过滤，被用于`tcpdump`指令，例如运行`tcpdump tcp and dst port 443`这样的过滤规则会复制协议为 tcp 并且目的端口是 443 的数据包到用户态。

BPF 程序运行在内核中，以便于过滤掉不必要的流量，仅检索那些我们需要监视的数据包，从而减少了将不必要的数据包复制到用户空间并随后进行筛选的开销。

BPF 是基于内核中的一个虚拟机来实现的，通过翻译 BPF 规则到字节码运行到内核中的虚拟机当中。

eBPF（extended BPF）由 BPF 发展而来，相比于之前的 BPF，eBPF 有了更丰富的指令，从 2 个 32 位寄存器扩展到了 11 个 64 位寄存器，而之前的版本被称为 cBPF（classic BPF）。在 linux 内核 v3.15 之后，内核开始支持 eBPF，所以有专门的程序会负责将 cBPF 指令翻译成 eBPF 指令来执行。

> 关于 eBPF 的成长史，可以阅读相关文章 [eBPF 年鉴](https://arthurchiao.art/blog/ebpf-and-k8s-zh/#7-ebpf-年鉴) 。

eBPF 大大扩展了功能，使其不仅限于包过滤，使得我们能够在内核中运行任意 eBPF 代码，例如，可以通过将程序附加到`kprobe`事件上，在相应的内核功能启动时会触发 eBPF 程序运行。借用 [Brendan Gregg](http://www.brendangregg.com/blog/index.html) 对 eBPF 技术的描述：

> *eBPF does to Linux what JavaScript does to HTML.*

就像我们可以使用 JavaScript 在网页上开发一些事件触发程序，在发生诸如鼠标单击按钮之类的事件时运行对应的函数，这些微型程序在浏览器的安全虚拟机中运行。使用 eBPF 时，我们可以编写当磁盘 I/O、系统调用等事件发生时运行的微型程序，这些程序在内核上的安全虚拟机中运行。更确切地说，eBPF 更像是运行 JavaScript 的 v8 虚拟机，我们可以将高级编程语言编译成字节码，运行在 eBPF 的沙盒环境中。

![/posts/linux/ebpf/ebpf-overview@2x.png](https://wingsxdu.com/posts/linux/ebpf/ebpf-overview@2x.png)事件触发任务是指实时操作系统中的一个任务只有在与之相关的特定事件发生的条件下才能开始运行。

如今 eBPF 不仅仅特定于网络，还可以用于内核跟踪、安全等场景，eBPF 的通用性也吸引了更大的社区进行创新，开发相关的生态软件。

#### 为什么使用？

由于 eBPF 程序可以由一系列事件触发，因此基于 eBPF 的应用程序可以帮助我们发现系统中正在发生的事情。eBPF 的一个应用场景是程序监控——识别应用程序的异常行为，例如当将文件写入重要的系统目录时，eBPF 代码可以响应文件事件而运行，以检查该程序的行为是否合法。

虽然有一些现成的工具也可以完成类似的任务，例如`ps`命令可以报告当前系统的进程状态。但是，此类监控程序的监测精度在于程序采样频率，执行时间比监视程序的采样间隔短的程序将不会被检测到，增加采样频率又会影响系统的性能。**而 eBPF 程序是事件触发的，而不是基于采样的，并且它在内核中运行，速度非常快**，所以基于 eBPF 的探测工具比传统的基于采样的方法更加准确。

除此之外，使用 eBPF 程序也无需重新编译内核。我们可以用 C 语言的一个子集编写程序，然后用编译器后端将其编译成 eBPF 字节码，最后内核再通过一个位于内核中的 JIT 即时编译器将 eBPF 指令映射成处理器的原生指令（opcode ），以取得在内核中的最佳执行性能。

基于 eBPF 的安全工具的缺点是，我们只能在新进程或文件访问事件启动时检测到它们，但是我们不能阻止它们启动。这些监测结果可以告诉我们进程何时以异常方式运行，并且可以通过 eBPF 来触发防护操作，例如关闭进程或关闭容器。但是，如果此意外行为是恶意的，则可能已经造成了损害。

cilium 的文档 *[BPF and XDP Reference Guide](https://docs.cilium.io/en/v1.9/bpf/)* 中总结了 eBPF 的一些优点：

- 无需在内核/用户空间切换就可以实现内核的可编程，需要在 eBPF 程序之间或内核/用户空间之间共享状态时，可以使用 eBPF Map。
- eBPF 程序能在编译时将不需要的特性禁用掉， 例如，如果容器不需要 IPv4，那编写 eBPF 程序时就可以只处理 IPv6 的情况；
- eBPF 给用户空间提供了一个稳定的 ABI，而且不依赖任何第三方内核模块：eBPF 是 Linux 内核的一个核心组成部分，而 Linux 已经得到了广泛的部署，因此可以保证现有的 eBPF 程序能在新的内核版本上继续运行。这种保证与系统调用是同一级别的，并且 eBPF 程序在不同平台上是可移植的；
- 在网络场景下，eBPF 程序可以在无需重启内核、系统服务或容器的情况下实现原子更新，并且不会导致网络中断，除此之外，更新 eBPF Map 不会导致程序状态的丢失；
- BPF 程序与内核协同工作，复用已有的内核基础设施（例如驱动、netdevice、 隧道、协议栈和 socket）和工具（例如 iproute2），以及内核提供的安全保证。**与 Linux Module 不同，eBPF 程序会被一个位于内核中的校验器（in-kernel verifier）进行校验， 以确保它们不会造成内核崩溃、程序永远会终止等等**。例如，XDP 程序会复用已有的内核驱动，能够直接操作存放在 DMA 缓冲区中的数据帧，而不用像某些模型（例如 DPDK） 那样将这些数据帧甚至整个驱动暴露给用户空间。而且，XDP 程序复用内核协议栈而不是绕过它。eBPF 程序可以看做是内核设施之间的通用`胶水代码`，利用中间层设计巧妙的程序 ，解决特定的问题。

## eBPF 编程

eBPF 被设计为一个通用目的 RISC 指令集，能够直接映射到 `x86_64`、`arm64`，因此所有的 eBPF 寄存器可以一一映射到 CPU 硬件的寄存器，并且通用操作都是 64 位的，这样可以对指针进行算术操作。

虽然指令集中包含前向和后向跳转，但内核中的 eBPF 校验器禁止程序中有循环，以此来保证程序最终会停止。因为 eBPF 程序运行在内核，校验器的工作是保证这些程序在运行时是安全的，不会影响到系统的稳定性。这意味着，从指令集的角度来说循环是可以实现的，但校验器会对其施加限制。

下面介绍一些 eBPF 编程中的概念与约定。

#### 寄存器和调用约定

BPF 由下面三部分组成：

1. 11 个 64 位寄存器，这些寄存器包含 32 位子寄存器；
2. 一个程序计数器 (Program Counter, PC)；
3. 一个 512 字节大小的 eBPF 栈空间。

寄存器的名字从 `r0` 到 `r10`，eBPF 的调用约定如下：

- `r0` 存放被调用的辅助函数的返回值，还用于保存 eBPF 程序的退出值；
- `r1` - `r5` 存放 eBPF 调用内核辅助函数时传递的参数；
- `r6` - `r9` 由被调用方保存，在函数返回之后调用方可以读取；
- `r10` 是唯一的只读寄存器，存放着 eBPF 栈空间的栈帧指针的地址；
- 栈空间用于临时保存`r1` - `r5` 的值，由于寄存器数量有限，如果要在多次辅助函数调用之间重用这些寄存器内的值，那 eBPF 程序需要负责将这些值临时转储到 eBPF 栈上 ，或者保存到被调用方保存的寄存器中。

> 注意：默认的运行模式是 64 位，32 位子寄存器只能通过特殊的 ALU（Arithmetic Logic Unit）访问，向 32 位子寄存器写入时，会用 0 填充到 64 位。

BPF 程序开始执行时，`r1` 寄存器中存放的是程序的上下文，也就是程序的输入参数。eBPF 只能在单个上下文中工作，这个上下文是由程序类型定义的， 例如网络程序可以将网络包的内核表示`skb`作为输入参数。

**在内核版本 5.2 之前 eBPF 程序的最大指令数严格限制在 4096 条以内**，这意味着从设计上就可以保证每个程序都会很快结束，但是随着 eBPF 程序越来越复杂，从 5.2 版本开始放宽到了 100 万条指令。另外，eBPF 中有尾部调用的概念，允许一 个 eBPF 程序调用另一个 eBPF 程序。尾部调用也是有限制的，目前上限是 32 层调用。现在这个功能常用来对程序逻辑进行解耦，解耦成几个不同阶段。

#### 程序类型与辅助函数

每个 eBPF 程序都属于某个特定的程序类型（Program Types），我们可以在 *[linux/bpf.h#L168 at v5.9](https://github.com/torvalds/linux/blob/v5.9/include/uapi/linux/bpf.h#L168)* 文件中查看当前版本支持的程序类型，并且还在不断增加中。可以大致分为网络、跟踪、安全等几大类，eBPF 程序的输入参数也根据程序类型有所不同。

辅助函数（Helper Functions）主要用来协助处理用户空间和内核空间的交互，比如从内核获取 PID、GID、时间、操作内核的对象等。不同的内核版本支持的辅助函数也是不一样的。我们可以从 *[linux/bpf.h#L3399 at v5.9](https://github.com/torvalds/linux/blob/v5.9/include/uapi/linux/bpf.h#L3399)* 查看当前版本支持的辅助函数。

不同类型的 BPF 程序能够使用的辅助函数可能是不同的，例如 eBPF 程序类型为`BPF_PROG_TYPE_SOCKET_FILTER`的只能使用下面几种辅助函数：

| `1 2 3 4 5 6 ` | `BPF_FUNC_skb_load_bytes() BPF_FUNC_skb_load_bytes_relative() BPF_FUNC_get_socket_cookie() BPF_FUNC_get_socket_uid() BPF_FUNC_perf_event_output() Base functions ` |
| -------------- | ------------------------------------------------------------ |
|                |                                                              |

受于篇幅限制，文章只举了几个简单的例子，详细的辅助函数分类与关系可以参考 BCC 的文档 *[bcc/docs/kernel-versions](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md)* 。

#### eBPF 映射

eBPF 映射（eBPF Maps）是**驻留在内核空间中的高效键值仓库**。Map 中的数据可以被任意eBPF 程序访问，如果想在多次 eBPF 程序调用之间保存状态，可以将状态信息放到 Map。Map 还可以从用户空间通过文件描述符访问，可以在任意 eBPF 程序以及用户空间应用之间共享。因此可以通过 Map 来达到 eBPF 程序与 eBPF 程序之间，eBPF 程序与用户态程序之间的数据交互。

共享 Map 的 eBPF 程序不要求是相同的程序类型，例如监控程序可以和网络程序共享 Map。目前单个 eBPF 程序最多可直接访问 64 个不同 Map。

![/posts/linux/ebpf/ebpf-map@2x.png](https://wingsxdu.com/posts/linux/ebpf/ebpf-map@2x.png)

##### 映射类型

Map 为上层程序提供了一层基础数据结构的映射，现在有二十几种数据类型，均由 core kernel 实现，因此我们无法增添或修改数据结构。Map 分为通用 Map 和非通用 Map 两种。下面列出了几个通用 Map 的类型：

| `1 2 3 4 5 6 7 ` | `// 通用 MAP BPF_MAP_TYPE_HASH BPF_MAP_TYPE_ARRAY BPF_MAP_TYPE_LRU_HASH BPF_MAP_TYPE_PERCPU_HASH BPF_MAP_TYPE_PERCPU_ARRAY BPF_MAP_TYPE_LRU_PERCPU_HASH ` |
| ---------------- | ------------------------------------------------------------ |
|                  |                                                              |

通用 Map 提供了哈希表、数组、LRU 等数据结构的映射，除此之外，还提供了对应的单 CPU 映射类型：我们可以给类型映射分配 CPU，每个 CPU 会看到自己独立的映射版本，这样更有利于高性能查找和指标聚合。

eBPF 程序可以通过辅助函数读写 Map，用户态程序也可以通过`bpf()`系统调用读写 Map，下面列出了所有对通用 Map 进行操作的函数和命令：

| ` 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 ` | `// Map 相关的辅助函数 BPF_FUNC_map_lookup_elem()		// 查找元素 BPF_FUNC_map_update_elem()		// 更新或创建元素 BPF_FUNC_map_delete_elem()		// 删除元素 BPF_FUNC_map_peek_elem() BPF_FUNC_map_pop_elem() BPF_FUNC_map_push_elem() // bpf() 系统调用可操作的 API BPF_MAP_LOOKUP_ELEM, BPF_MAP_UPDATE_ELEM, BPF_MAP_DELETE_ELEM, BPF_MAP_GET_NEXT_KEY,			// 用于迭代查询 BPF_MAP_LOOKUP_AND_DELETE_ELEM,	// 查找并删除 BPF_MAP_LOOKUP_BATCH,			// 对应操作的批量处理 BPF_MAP_LOOKUP_AND_DELETE_BATCH, BPF_MAP_UPDATE_BATCH, BPF_MAP_DELETE_BATCH, ` |
| --------------------------------------------- | ------------------------------------------------------------ |
|                                               |                                                              |

数据操作并不复杂，从函数名称可以看出它的作用，用户态`bpf()`系统调用则又封装了一些高级操作，方便进行批量数据处理。

| `1 2 3 4 5 ` | `// 非通用 MAP BPF_MAP_TYPE_PROG_ARRAY BPF_MAP_TYPE_ARRAY_OF_MAPS BPF_MAP_TYPE_HASH_OF_MAPS BPF_MAP_TYPE_CGROUP_ARRAY ` |
| ------------ | ------------------------------------------------------------ |
|              |                                                              |

上面列出了几个非通用 Map，它们只用于特定的场景，例如`BPF_MAP_TYPE_PROG_ARRAY` 用于保存其它的 eBPF 程序的引用，可以与尾部调用配合实现程序间的跳转；`BPF_MAP_TYPE_ARRAY_OF_MAPS` 和 `BPF_MAP_TYPE_HASH_OF_MAPS` 都用于持有其他 Map 的指针，这样整个 Map 就可以在运行时实现原子替换，`BPF_MAP_TYPE_CGROUP_ARRAY`则用于保存对 cgroup 的引用。

##### eBPF 虚拟文件系统

eBPF 映射的基本特征是文件描述符`fd`，这意味着当文件描述符关闭后保存的数据也会消失。为了使这些数据在创建它的程序终止后依然保存，从 Linux 内核 4.4 版本开始引入了 eBPF 虚拟文件系统，默认将数据挂载在`/sys/fs/bpf/`目录下，通过路径来标识这些持久化对象。

我们只能通过系统调用`bpf()`来操作这些对象，`BPF_OBJ_PIN`命令用于将 Map 对象保存到文件系统中，`BPF_OBJ_GET`用于获取已经固定到文件系统中的对象。

##### 并发访问

由于 eBPF 映射可以发生许多程序同时并发访问同一个 Map，这可能会产生竞争条件。为了防止数据竞争，eBPF 引入了**自旋锁**的概念，对访问的映射元素进行锁定。

> 自旋锁这一特性由 Linux 5.1 版本引入，且仅适用于数组、哈希、cgroup 映射。

在 eBPF 程序中，我们可以使用`BPF_FUNC_spin_lock()`和`BPF_FUNC_spin_unlock()`这两个辅助函数对数据加锁解锁，释放锁后其它程序就可以安全地访问该元素。而在用户空间，我们在更新或读取元素时可以使用`BPF_F_LOCK`标志，从而避免数据竞争。

## eBPF 应用场景

在了解 eBPF 编程的基本概念后，我们来看看 eBPF 的几个应用场景。

#### 跟踪

跟踪的目的是提供运行时有用的信息以便将来进行问题的分析。使用 eBPF 进行跟踪的主要优点是几乎可以访问 Linux 内核和应用程序的任何信息，并且 eBPF 对系统性能和延迟造成的开销最小，也不需要为了收集数据而去修改业务程序。 eBPF 提供了探针和追踪点两种跟踪方式

##### 探针

eBPF 提供的探针分为两种：

- **内核探针**：提供对内核中内部组件的动态访问；
- **用户空间探针**：提供对用户空间运行程序的动态访问。

其中内核探针分为两类：kprobes 允许在执行任何内核指令之前插入 eBPF 程序，kretprobes 则是在内核指令有返回值时插入 eBPF 程序。用户空间探针允许在用户空间运行的程序中设置动态标志，它们等同于内核探针，分为 uprobes 和 ureprobes。

在下面的 BCC 示例代码中，我们编写了一段 kprobes 探针程序，该程序通过跟踪`execve()`系统调用来演示 eBPF 的功能。监视`execve()`系统调用对于检测意外的可执行文件很有用。

代码中的 中`do_sys_execve()`函数用于获取内核正在运行的命令名，并打印至控制台；然后我们利用 BCC 提供的 bpf.attach_kprobe() 方法将`do_sys_execve()`函数和`execve()`系统调用绑定起来。运行这段程序将会看到，每当内核执行`execve()`系统调用时都会在控制台打印相应的命令名称。

| ` 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 ` | `from bcc import BPF bpf_source = """ #include <uapi/linux/ptrace.h> int do_sys_execve(struct pt_regs *ctx) {  char comm[16];  bpf_get_current_comm(&comm, sizeof(comm));  bpf_trace_printk("executing program: %s\\n", comm);  return 0; } """ bpf = BPF(text=bpf_source) execve_function = bpf.get_syscall_fnname("execve") bpf.attach_kprobe(event=execve_function, fn_name="do_sys_execve") bpf.trace_print() ` |
| --------------------------------------------- | ------------------------------------------------------------ |
|                                               |                                                              |

需要注意的是，内核探针没有稳定的应用程序二进制接口(ABI)，因此在不同的内核版本中同样的程序代码可能无法工作。

##### 追踪点

追踪点是内核代码的静态标记，由内核人员开发编写，并且保证旧版本上的追踪点在新版本上存在。

在 Linux 上每个跟踪点都对应一个`/sys/kernel/debug/tracing/events`条目。例如，查看网络相关的追踪点：

| ` 1 2 3 4 5 6 7 8 9 10 11 ` | `# sudo ls -la /sys/kernel/debug/tracing/events/net # 篇幅限制删减了部分追踪点 total 0 drwxr-xr-x  2 root root 0 Nov 20 21:00 net_dev_queue drwxr-xr-x  2 root root 0 Nov 20 21:00 net_dev_start_xmi drwxr-xr-x  2 root root 0 Nov 20 21:00 netif_receive_skb drwxr-xr-x  2 root root 0 Nov 20 21:00 netif_receive_skb_entry drwxr-xr-x  2 root root 0 Nov 20 21:00 netif_receive_skb_exit drwxr-xr-x  2 root root 0 Nov 20 21:00 netif_rx drwxr-xr-x  2 root root 0 Nov 20 21:00 netif_rx_entry drwxr-xr-x  2 root root 0 Nov 20 21:00 netif_rx_exit ` |
| --------------------------- | ------------------------------------------------------------ |
|                             |                                                              |

下面的示例中，我们将 eBPF 程序绑定到`net:netif_rx`追踪点上，并在控制台打印出调用程序名称。

| ` 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 ` | `from bcc import BPF bpf_source = """ int trace_netif_rx(void *ctx) {  char comm[16];  bpf_get_current_comm(&comm, sizeof(comm));   bpf_trace_printk("%s is doing netif_rx", comm);  return 0; } """ bpf = BPF(text = bpf_source) bpf.attach_tracepoint(tp = "net:netif_rx", fn_name = "trace_netif_rx") bpf.trace_print() ` |
| --------------------------------------- | ------------------------------------------------------------ |
|                                         |                                                              |

#### 网络

eBPF 程序在网络中主要有两个用途：数据包捕获和过滤，用户空间的程序可以将过滤器加到任何套接字中，提取数据包的相关信息，或者对特定类型的数据包放行、禁止、重定向等。

##### 数据包过滤

数据包过滤是 eBPF 最常用的场景之一，主要用于三种情况：

- 实时流量丢弃，例如只允许 UDP 流量通过，丢弃其它数据包；
- 实时观测特定条件过滤后的数据包；
- 对实时系统中抓取的网络流量进行后续分析。

`tcpdump`是典型的 eBPF 数据包过滤应用，现在我们可以编写一些自定义的网络程序，在 *[bcc/examples/networking](https://github.com/iovisor/bcc/tree/master/examples/networking)* 中可以看到一些使用示例

##### 流量控制分类器

流量控制的几个用例如下：

- 优先处理某些类型的数据包；
- 丢弃特定类型的数据包；
- 带宽分配。

##### XDP

XDP（eXpress Data Path）为 Linux 内核提供了高性能、可编程的网络数据路径。由于网络包在还未进入网络协议栈之前就处理，它给 Linux 网络带来了巨大的性能提升。

![/posts/linux/ebpf/receive-socket.jpg](https://wingsxdu.com/posts/linux/ebpf/receive-socket.jpg)

通过上面的图片我们可以大致看到一个完整的数据包接收过程，可以看到整个处理链路还是比较长的，且需要在内核态与用户态之间做内存拷贝、上下文切换、软硬件中断等。处理大量的数据包会占用大量 CPU 资源，难以满足高并发网络的需求。

XDP 解决了这个问题。它相当于在 Linux 网络栈中加了新的一层。在报文到达 CPU 的最早时刻进行处理，甚至避免了`sk_buff`的分配，减少了内存拷贝上的负荷。在特定网卡硬件与驱动的支持下，甚至可以将 XDP 程序卸载到网卡上执行，进一步减少了 CPU 的使用

XDP 依赖 eBPF 技术，并提供了一套完整的、可编程的报文处理方案，我们可以利用 XDP 实现数据包的转发、重定向与向下传递。相关示例可以在 [bcc/examples/networking/xdp](https://github.com/iovisor/bcc/tree/master/examples/networking/xdp) 中查看。

#### 安全

Seccomp（Secure Computing）在 2.6.12 版本中引入 Linux 内核，将进程可用的系统调用限制为四种：`read`、`write`、`_exit`、`sigreturn`。最初 Seccomp 模式采用白名单方式实现，在这种安全模式下，除了已打开的文件描述符和允许的四种系统调用，如果尝试其他系统调用，内核就会使用 SIGKILL 或 SIGSYS 终止该进程。

尽管 Seccomp 保证了主机的安全，但由于限制太强实际作用并不大。在实际应用中需要更加精细的限制，为了解决此问题，引入了 Seccomp-BPF 。Seccomp-BPF 是 Seccomp 和 cBPF 规则的结合（注意并不是 eBPF），它允许用户使用可配置的策略过滤系统调用，该策略使用 BPF 规则实现，它可以对任意系统调用及其参数进行过滤。

使用示例可以参考 [seccomp](http://wh4lter.icu/2020/04/20/seccomp/) 。

## References

- [A Deep Dive into eBPF: The Technology that Powers Tracee (aquasec.com)](https://blog.aquasec.com/intro-ebpf-tracing-containers)
- [[译\] Cilium：BPF 和 XDP 参考指南（2019）](http://arthurchiao.art/blog/cilium-bpf-xdp-reference-guide-zh/)
- [bcc Reference Guide](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md)
- [Seccomp从0到1](https://www.anquanke.com/post/id/208364)

## 相关文章

- [Linux 虚拟文件系统](https://wingsxdu.com/posts/linux/vfs/)
- [Linux 系统中的虚拟内存](https://wingsxdu.com/posts/linux/memory-management/virtual-memory/)
- [为什么进程 fork 采用写时复制 · Why](https://wingsxdu.com/posts/linux/concurrency-oriented-programming/fork-and-cow/)
- [浅论并发编程中的同步问题 · Analyze](https://wingsxdu.com/posts/linux/concurrency-oriented-programming/synchronous/)
- [浅析进程与线程的设计 · Analyze](https://wingsxdu.com/posts/linux/concurrency-oriented-programming/process-and-thread/)

> https://wingsxdu.com/posts/linux/ebpf/
