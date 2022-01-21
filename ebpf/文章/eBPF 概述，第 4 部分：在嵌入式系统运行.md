## 1. 前言

在本系列的[第 1 部分](https://kubesphere.com.cn/blogs/ebpf-guide/)和[第 2 部分](https://kubesphere.com.cn/blogs/ebpf-machine-bytecode/)，我们介绍了 eBPF 虚拟机内部工作原理，在[第 3 部分](https://kubesphere.com.cn/blogs/ebpf-software-stack/)我们研究了基于底层虚拟机机制之上开发和使用 eBPF 程序的主流方式。

在这一部分中，我们将从另外一个视角来分析项目，尝试解决嵌入式 Linux 系统所面临的一些独特的问题：如需要非常小的自定义操作系统镜像，不能容纳完整的 BCC LLVM 工具链/python 安装，或试图避免同时维护主机的交叉编译（本地）工具链和交叉编译的目标编译器工具链，以及其相关的构建逻辑，即使在使用像 OpenEmbedded/Yocto 这样的高级构建系统时也很重要。

## 2. 关于可移植性

在第 3 部分研究的运行 eBPF/BCC 程序的主流方式中，可移植性并不是像在嵌入式设备上面临的问题那么大：eBPF 程序是在被加载的同一台机器上编译的，使用已经运行的内核，而且头文件很容易通过发行包管理器获得。嵌入式系统通常运行不同的 Linux 发行版和不同的处理器架构，与开发人员的计算机相比，有时具有重度修改或上游分歧的内核，在构建配置上也有很大的差异，或还可能使用了只有二进制的模块。

eBPF 虚拟机的字节码是通用的（并未与特定机器相关），所以一旦编译好 eBPF 字节码，将其从 x86_64 移动到 ARM 设备上并不会引起太多问题。当字节码探测内核函数和数据结构时，问题就开始了，这些函数和数据结构可能与目标设备的内核不同或者会不存在，所以至少目标设备的内核头文件必须存在于构建 eBPF 程序字节码的主机上。新的功能或 eBPF 指令也可能被添加到以后的内核中，这可以使 eBPF 字节码向前兼容，但不能在内核版本之间向后兼容（参见[内核版本与 eBPF 功能](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md)）。建议将 eBPF 程序附加到稳定的内核 ABI 上，如跟踪点（tracepoint），这可以缓解常见的可移植性。

最近一个重要的工作已经开始，通过在 LLVM 生成的 eBPF 对象代码中嵌入数据类型信息，通过增加 BTF（BTF 类型格式）数据，以增加 eBPF 程序的可移植性（CO-RE 一次编译，到处运行）。更多信息见这里的[补丁](https://lwn.net/Articles/750695/)和[文章](https://lwn.net/Articles/773198/)。这很重要，因为 BTF 涉及到 eBPF 软件技术栈的所有部分（内核虚拟机和验证器、clang/LLVM 编译器、BCC 等），但这种方式可带来很大的便利，允许重复使用现有的 BCC 工具，而不需要特别的 eBPF 交叉编译和在嵌入式设备上安装 LLVM 或运行 BPFd。截至目前，CO-RE BTF 工作仍处于早期开发阶段，还需要付出相当多的工作才能可用【译者注：当前在高版本内核已经可以使用或者编译内核时启用了 BTF 编译选项】。也许我们会在其完全可用后再发表一篇博文。

## 3. BPFd

[BPFd](https://lwn.net/Articles/744522/)（项目地址 https://github.com/joelagnel/bpfd）更像是一个为 Android 设备开发的概念验证，后被放弃，转而通过 [adeb](https://github.com/joelagnel/adeb) 包运行一个完整的设备上的 BCC 工具链【译者注：BCC 在 adeb 的编译文档参见[这里](https://github.com/joelagnel/adeb/blob/master/BCC.md)】。如果一个设备足够强大，可以运行 Android 和 Java，那么它也可能可以安装 BCC/LLVM/python。尽管这个实现有些不完整（通信是通过 Android USB 调试桥或作为一个本地进程完成的，而不是通过一个通用的传输层），但这个设计很有趣，有足够时间和资源的人可以把它拿起来合并，继续搁置的 [PR 工作](https://github.com/iovisor/bcc/pull/1675)。

简而言之，BPFd 是一个运行在嵌入式设备上的守护程序，作为本地内核/libbpf 的一个远程过程调用（RPC）接口。Python 在主机上运行，调用 BCC 来编译/部署 eBPF 字节码，并通过 BPFd 创建/读取 map。BPFd 的主要优点是，所有的 BCC 基础设施和脚本都可以工作，而不需要在目标设备上安装 BCC、LLVM 或 python，BPFd 二进制文件只有 100kb 左右的大小，并依赖 libc。

![img](https://pek3b.qingstor.com/kubesphere-community/images/eBPF-Part4-Diagram1.jpeg)

## 4. Ply

[ply](https://wkz.github.io/ply/) 项目实现了一种与 BPFtrace 非常相似的高级领域特定语言（受到 AWK 和 C 的启发），其明确的目的是将运行时的依赖性降到最低。它只依赖于一个现代的 libc（不一定是 GNU 的 libc）和 shell（与 sh 兼容）。Ply 本身实现了一个 eBPF 编译器，需要根据目标设备的内核头文件进行构建，然后作为一个单一的二进制库和 shell 包装器部署到目标设备上。

![img](https://pek3b.qingstor.com/kubesphere-community/images/eBPF-Part4-Diagram2.jpeg)

为了更好解释 ply，我们把第 3 部分中的 BPFtrace 例子和与 ply 实现进行对比：

- BPFtrace：要运行该例子，你需要数百 MB 的 LLVM/clang、libelf 和其他依赖项：

  `bpftrace -e 'tracepoint:raw_syscalls:sys_enter {@[pid, comm] = count();}'`

- ply：你只需要一个 ~50kb 的二进制文件，它产生的结果是相同的，语法几乎相同：

  `ply 'tracepoint:raw_syscalls/sys_enter {@[pid, comm] = count();}'`

Ply 仍在大量开发中（最近的 v2.0 版本是完全重写的）【译者注：当前最新版本为 2.1.1，最近一次代码提交是 8 个月前，活跃度一般】，除了一些示例之外，该语言还不不稳定或缺乏文档，它不如完整的 BCC 强大，也没有 BPFtrace 丰富的功能特性，但它对于通过 ssh 或串行控制台快速调试远程嵌入式设备仍然非常有用。

## 5. Gobpf

[Gobpf](https://github.com/iovisor/gobpf) 及其合并的子项目（goebpf, gobpf-elf-loader），是 IOVisor 项目的一部分，为 BCC 提供 Golang 语言绑定。eBPF 的内核逻辑仍然用 "限制性 C" 编写，并由 LLVM 编译，只有标准的 python/lua 用户空间脚本被 Go 取代。这个项目对嵌入式设备的意义在于它的 eBPF [elf 加载模块](https://github.com/iovisor/gobpf/tree/master/elf)，其可以被交叉编译并在嵌入式设备上独立运行，以加载 eBPF 程序至内核并与与之交互。

![img](https://pek3b.qingstor.com/kubesphere-community/images/eBPF-Part4-Diagram3.jpeg)

值得注意的是，go 加载器可以被写成通用的（我们很快就会看到），因此它可以加载和运行任何 eBPF 字节码，并在本地重新用于多个不同的跟踪会话。

使用 gobpf 很痛苦的，主要是因为缺乏文档。目前最好的 "文档" 是 [tcptracer 源码](https://github.com/weaveworks/tcptracer-bpf)，它相当复杂（他们使用 kprobes 而不依赖于特定的内核版本！），但从它可以学到很多。Gobpf 本身也是一项正在进行的工作：虽然 elf 加载器相当完整，并支持加载带有套接字、(k|u)probes、tracepoints、perf 事件等加载的 eBPF ELF 对象，但 bcc go 绑定模块还不容易支持所有这些功能。例如，尽管你可以写一个 socket_ilter ebpf 程序，将其编译并加载到内核中，但你仍然不能像 BCC 的 python 那样从 go 用户空间轻松地与 eBPF 进行交互，BCC 的 API 更加成熟和用户友好。无论如何，gobpf 仍然比其他具有类似目标的项目处于更好的状态。

让我们研究一个简单的例子来说明 gobpf 如何工作的。首先，我们将在本地 x86_64 机器上运行它，然后交叉编译并在 32 位 ARMv7 板上运行它，比如流行的 Beaglebone 或 Raspberry Pi。我们的文件目录结构如下：

```
$ find . -type f
./src/open-example.go
./src/open-example.c
./Makefile
```

**open-example.go**：这是建立在 gobpf/elf 之上的 eBPF ELF 加载器。它把编译好的 "限制性 C" ELF 对象作为参数，加载到内核并运行，直到加载器进程被杀死，这时内核会自动卸载 eBPF 逻辑【译者注：通常情况是这样的，也有场景加载器退出，ebpf 程序继续运行的】。我们有意保持加载器的简单性和通用性（它加载在对象文件中发现的任何探针），因此加载器可以被重复使用。更复杂的逻辑可以通过使用 [gobpf 绑定](https://github.com/iovisor/gobpf/blob/master/bcc/module.go) 模块添加到这里。

```
package main

import (
    "fmt"
    "os"
    "os/signal"
    "github.com/iovisor/gobpf/elf"
)

func main() {mod := elf.NewModule(os.Args[1])

    err := mod.Load(nil);
    if err != nil {fmt.Fprintf(os.Stderr, "Error loading'%s'ebpf object: %v\n", os.Args[1], err)os.Exit(1)
    }

    err = mod.EnableKprobes(0)
    if err != nil {fmt.Fprintf(os.Stderr, "Error loading kprobes: %v\n", err)
        os.Exit(1)
    }

    sig := make(chan os.Signal, 1)
    signal.Notify(sig, os.Interrupt, os.Kill)
    // ...
}
```

**open-example.c**：这是上述加载器加载至内核的 "限制性 C" 源代码。它挂载在 do_sys_open 函数，并根据 [ftrace format](https://raw.githubusercontent.com/torvalds/linux/v4.20/Documentation/trace/ftrace.rst) 将进程命令、PID、CPU、打开文件名和时间戳打印到跟踪环形缓冲区，（详见 "输出格式" 一节）。打开的文件名作为 [do_sys_open call](https://github.com/torvalds/linux/blob/v4.20/fs/open.c#L1048) 的第二个参数传递，可以从代表函数入口的 CPU 寄存器的上下文结构中访问。

```
#include <uapi/linux/bpf.h>
#include <uapi/linux/ptrace.h>
#include <bpf/bpf_helpers.h>

SEC("kprobe/do_sys_open")
int kprobe__do_sys_open(struct pt_regs *ctx)
{char file_name[256];

    bpf_probe_read(file_name, sizeof(file_name), PT_REGS_PARM2(ctx));

    char fmt[] = "file %s\n";
    bpf_trace_printk(fmt, sizeof(fmt), &file_name);

    return 0;
}

char _license[] SEC("license") = "GPL";
__u32 _version SEC("version") = 0xFFFFFFFE;
```

在上面的代码中，我们定义了特定的 "SEC" 区域，这样 gobpf 加载器就可获取到哪里查找或加载内容的信息。在我们的例子中，区域为 kprobe、license 和 version。特殊的 0xFFFFFFFE 值告诉加载器，这个 eBPF 程序与任何内核版本都是兼容的，因为打开系统调用而破坏用户空间的机会接近于 0。

**Makefile**：这是上述两个文件的构建逻辑。注意我们是如何在 include 路径中加入 "arch/x86/..." 的；在 ARM 上它将是 "arch/arm/..."。

```
SHELL=/bin/bash -o pipefail
LINUX_SRC_ROOT="/home/adi/workspace/linux"
FILENAME="open-example"

ebpf-build: clean go-build
	clang \
	-D__KERNEL__ -fno-stack-protector -Wno-int-conversion \
	-O2 -emit-llvm -c "src/${FILENAME}.c" \
	-I ${LINUX_SRC_ROOT}/include \
	-I ${LINUX_SRC_ROOT}/tools/testing/selftests \
	-I ${LINUX_SRC_ROOT}/arch/x86/include \
	-o - | llc -march=bpf -filetype=obj -o "${FILENAME}.o"

go-build:
	go build -o ${FILENAME} src/${FILENAME}.go

clean:
	rm -f ${FILENAME}*
```

运行上述 makefile 在当前目录下产生两个新文件：

- open-example：这是编译后的 src/*.go 加载器。它只依赖于 libc 并且可以被复用来加载多个 eBPF ELF 文件运行多个跟踪。
- open-example.o：这是编译后的 eBPF 字节码，将在内核中加载。

“open-example" 和 "open-example.o" ELF 二进制文件可以进一步合并成一个；加载器可以包括 eBPF 二进制文件作为资产，也可以像 [tcptracer](https://github.com/weaveworks/tcptracer-bpf/blob/master/pkg/tracer/tcptracer-ebpf.go#L80) 那样在其源代码中直接存储为字节数。然而，这超出了本文的范围。

运行例子显示以下输出（见 [ftrace 文档](https://kubesphere.io/zh/blogs/ebpf-working-with-embedded-systems/(https://raw.githubusercontent.com/torvalds/linux/v4.20/Documentation/trace/ftrace.rst)) 中的 "输出格式" 部分）。

```
# (./open-example open-example.o &) && cat /sys/kernel/debug/tracing/trace_pipe
electron-17494 [007] ...3 163158.937350: 0: file /proc/self/maps
systemd-1      [005] ...3 163160.120796: 0: file /proc/29261/cgroup
emacs-596      [006] ...3 163163.501746: 0: file /home/adi/
(...)
```

沿用我们在本系列的第 3 部分中定义的术语，我们的 eBPF 程序有以下部分组成：

- **后端**：是 open-example.o ELF 对象。它将数据写入内核跟踪环形缓冲区。
- **加载器**：这是编译过的 open-example 二进制文件，包含 gobpf/elf 加载器模块。只要它运行，数据就会被添加到跟踪缓冲区中。
- **前端**：这就是 `cat /sys/kernel/debug/tracing/trace_pipe`。非常 UNIX 风格。
- **数据结构**：内核跟踪环形缓冲区。

现在将我们的例子交叉编译为 32 位 ARMv7。 基于你的 ARM 设备运行的内核版本：

- 内核版本>=5.2：只需改变 makefile，就可以交叉编译与上述相同的源代码。
- 内核版本<5.2：除了使用新的 makefile 外，还需要将 PT_REGS_PARM* 宏从 [这个 patch](https://lore.kernel.org/bpf/20190304205019.15071-1-adrian.ratiu@collabora.com/) 复制到 "受限制 C" 代码。

新的 makefile 告诉 LLVM/Clang，eBPF 字节码以 ARMv7 设备为目标，使用 32 位 eBPF 虚拟机子寄存器地址模式，以便虚拟机可以正确访问本地处理器提供的 32 位寻址内存（还记得第 2 部分中介绍的所有 eBPF 虚拟机寄存器默认为 64 位宽），设置适当的包含路径，然后指示 Go 编译器使用正确的交叉编译设置。在运行这个 makefile 之前，需要一个预先存在的交叉编译器工具链，它被指向 CC 变量。

```
SHELL=/bin/bash -o pipefail
LINUX_SRC_ROOT="/home/adi/workspace/linux"
FILENAME="open-example"

ebpf-build: clean go-build
	clang \
		--target=armv7a-linux-gnueabihf \
		-D__KERNEL__ -fno-stack-protector -Wno-int-conversion \
		-O2 -emit-llvm -c "src/${FILENAME}.c" \
		-I ${LINUX_SRC_ROOT}/include \
		-I ${LINUX_SRC_ROOT}/tools/testing/selftests \
		-I ${LINUX_SRC_ROOT}/arch/arm/include \
		-o - | llc -march=bpf -filetype=obj -o "${FILENAME}.o"

go-build:
	GOOS=linux GOARCH=arm CGO_ENABLED=1 CC=arm-linux-gnueabihf-gcc \
	go build -o ${FILENAME} src/${FILENAME}.go

clean:
	rm -f ${FILENAME}*
```

运行新的 makefile，并验证产生的二进制文件已经被正确地交叉编译：

```
[adi@iwork]$ file open-example*
open-example:   ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter (...), stripped
open-example.o: ELF 64-bit LSB relocatable, *unknown arch 0xf7* version 1 (SYSV), not stripped
```

然后将加载器和字节码复制到设备上，与在 x86_64 主机上使用上述相同的命令来运行。记住，只要修改和重新编译 C eBPF 代码，加载器就可以重复使用，用于运行不同的跟踪。

```
[root@ionelpi adi]# (./open-example open-example.o &) && cat /sys/kernel/debug/tracing/trace_pipe
ls-380     [001] d..2   203.410986: 0: file /etc/ld-musl-armhf.path
ls-380     [001] d..2   203.411064: 0: file /usr/lib/libcap.so.2
ls-380     [001] d..2   203.411922: 0: file /
zcat-397   [002] d..2   432.676010: 0: file /etc/ld-musl-armhf.path
zcat-397   [002] d..2   432.676237: 0: file /usr/lib/libtinfo.so.5
zcat-397   [002] d..2   432.679431: 0: file /usr/bin/zcat
gzip-397   [002] d..2   432.693428: 0: file /proc/
gzip-397   [002] d..2   432.693633: 0: file config.gz
```

由于加载器和字节码加起来只有 2M 大小，这是一个在嵌入式设备上运行 eBPF 的相当好的方法，而不需要完全安装 BCC/LLVM。

## 6. 总结

在本系列的第 4 部分，我们研究了可以用于在小型嵌入式设备上运行 eBPF 程序的相关项目。不幸的是，当前使用这些项目还是比较很困难的：它们有的被遗弃或缺乏人力，在早期开发时一切都在变化，或缺乏基本的文档，需要用户深入到源代码中并自己想办法解决。正如我们所看到的，gobpf 项目作为 BCC/python 的替代品是最有活力的，而 ply 也是一个有前途的 BPFtrace 替代品，其占用空间最小。随着更多的工作投入到这些项目中以降低使用者的门槛，eBPF 的强大功能可以用于资源受限的嵌入式设备，而无需移植/安装整个 BCC/LLVM/python/Hover 技术栈。
