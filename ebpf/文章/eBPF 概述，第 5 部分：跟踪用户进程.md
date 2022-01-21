## 1. 前言

在之前的部分中，我们专注于 Linux 内核跟踪，在我们看来，基于 eBPF 的项目是最安全、最广泛可用和最有效的方法（eBPF 在 Linux 中完全是上游支持的，保证稳定的 ABI，在几乎所有的发行版中都默认启用，并可与所有其他跟踪机制集成）。 eBPF 成为内核工作的不二之选。 然而，到目前为止，我们故意避免深入讨论用户空间跟踪，因为它值得特别对待，因此我们在第 5 部分中专门讨论。

首先，我们将讨论为什么使用，然后我们将 eBPF 用户跟踪分为静态和动态两类分别讨论。

## 2. 为什么要在用户空间使用 eBPF？

最重要的用户问题是，既然有这么多其他的调试器/性能分析器/跟踪器，这么多针对特定语言或操作系统的工具为同样的任务而开发，为什么还要使用 eBPF 来跟踪用户空间进程？答案并不简单，根据不同的使用情况，eBPF 可能不是最佳解决方案；在庞大的用户空间生态系统中，并没有一个适合所有情况的调试/跟踪的项目。

eBPF 跟踪具有以下优势：

- 它为内核和用户空间提供了一个统一的跟踪接口，与其他工具（[k,u]probe, (dtrace)tracepoint 等）使用的机制兼容。2015 年的文章[选择 linux 跟踪器](http://www.brendangregg.com/blog/2015-07-08/choosing-a-linux-tracer.html)虽然有些过时，但其提供了很好的见解，说明使用所有不同的工具有多困难，要花多少精力。有一个统一的、强大的、安全的、可广泛使用的框架来满足大多数跟踪的需要，是非常有价值的。一些更高级别的工具，如 Perf/SystemTap/DTrace，正在 eBPF 的基础上重写（成为 eBPF 的前端），所以了解 eBPF 有助于使用它们。
- eBPF 是完全可编程的。Perf/ftrace 和其他工具都需要在事后处理数据，而 eBPF 可直接在内核/应用程序中运行自定义的高级本地编译的 C/Python/Go 检测代码。它可以在多个 eBPF 事件运行之间存储数据，例如以基于函数状态/参数计算每个函数调用统计数据。
- eBPF 可以跟踪系统中的一切，它并不局限于特定的应用程序。例如可以在共享库上设置 uprobes 并跟踪链接和调用它的所有进程。
- 很多调试器需要暂停程序来观察其状态或降低运行时性能，从而难以进行实时分析，尤其是在生产工作负载上。因为 eBPF 附加了 JIT 的本地编译的检测代码，它的性能影响是最小的，不需要长时间暂停执行。

诚然，eBPF 也有一些缺点：

- eBPF 不像其他跟踪器那样可以移植。该虚拟机主要是在 Linux 内核中开发的（有一个正在进行的 BSD 移植），相关的工具是基于 Linux 开发的。
- eBPF 需要一个相当新的内核。例如对于 MIPS 的支持是在 v4.13 中加入的，但绝大多数 MIPS 设备在运行的内核都比 v4.13 老。
- 一般来说，eBPF 不容易像语言或特定应用的用户空间调试器那样提供一样多的洞察力。例如，Emacs 的核心是一个用 C 语言编写的 ELISP 解释器：eBPF 可以通过挂载在 Emacs 运行时的 C 函数调用来跟踪/调试 ELISP 程序，但它不了解更高级别的 ELISP 语言实现，因此使用 Emacs 提供的特殊 ELISP 语言特定跟踪器和调试器变得更加有用。另一个例子是调试在 Web 浏览器引擎中运行的 JavaScript 应用程序。
- 因为 "普通 eBPF" 在 Linux 内核中运行，所以每次 eBPF 检测用户进程时都会发生内核 - 用户上下文切换。这对于调试性能关键的用户空间代码来说可能很昂贵（也许可以使用[用户空间 eBPF 虚拟机](https://github.com/iovisor/ubpf)项目来避免这种切换成本？）。这对于调试性能关键的用户空间代码来说是很昂贵的（也许[用户空间 eBPF VM](https://github.com/iovisor/ubpf) 项目可以用来避免这种切换成本？）。这种上下文切换比正常的调试器（或像 strace 这样的工具）要便宜得多，所以它通常可以忽略不计，但在这种情况下，像 LTTng 这样能够完全运行在用户空间的跟踪器可能更合适。

## 3. 静态跟踪点（USDT 探针）

静态跟踪点（tracepoint），在用户空间也被称为 USDT（用户静态定义的跟踪）探针（应用程序中感兴趣的特定位置），跟踪器可以在此处挂载检查代码执行和数据。它们由开发人员在源代码中明确定义，通常在编译时用 "--enable-trace" 等标志启用。静态跟踪点的优势在于它们不会经常变化：开发人员通常会保持稳定的静态跟踪 ABI，所以跟踪工具在不同的应用程序版本之间工作，这很有用，例如当升级 PostgreSQL 安装并遇到性能降低时。

### 3.1 预定义的跟踪点

[BCC-tools](https://github.com/iovisor/bcc/tree/master/tools) 包含很多有用的且经过测试的工具，可以与特定应用程序或语言运行时定义的跟踪点进行交互。对于我们的示例，我们将跟踪 Python 应用程序。确保你在构建了 python3 时启用了 "--enable-trace" 标识，并在 python 二进制文件或 libpython（取决于你构建方式）上运行 [tplist](https://github.com/iovisor/bcc/blob/master/tools/tplist.py) 以确认跟踪点被启用：

```
$ tplist -l /usr/lib/libpython3.7m.so
b'/usr/lib/libpython3.7m.so' b'python':b'import__find__load__start'
b'/usr/lib/libpython3.7m.so' b'python':b'import__find__load__done'
b'/usr/lib/libpython3.7m.so' b'python':b'gc__start'
b'/usr/lib/libpython3.7m.so' b'python':b'gc__done'
b'/usr/lib/libpython3.7m.so' b'python':b'line'
b'/usr/lib/libpython3.7m.so' b'python':b'function__return'
b'/usr/lib/libpython3.7m.so' b'python':b'function__entry'
```

首先我们使用 BCC 提供的一个很酷的跟踪工具 [uflow](https://github.com/iovisor/bcc/blob/master/tools/lib/uflow_example.txt)，来跟踪 python 的[简单 http 服务器](https://github.com/python/cpython/blob/3.7/Lib/http/server.py)的执行流程。跟踪应该是不言自明的，箭头和缩进表示函数的进入/退出。我们在这个跟踪中看到的是一个工作线程如何在 CPU 3 上退出，而主线程则准备在 CPU 0 上为其他传入的 http 请求提供服务。

```
$ python -m http.server >/dev/null & sudo ./uflow -l python $!
[4] 11727
Tracing method calls in python process 11727... Ctrl-C to quit.
CPU PID    TID    TIME(us) METHOD
3   11740  11757  7.034           /usr/lib/python3.7/_weakrefset.py._remove
3   11740  11757  7.034     /usr/lib/python3.7/threading.py._acquire_restore
0   11740  11740  7.034               /usr/lib/python3.7/threading.py.__exit__
0   11740  11740  7.034             /usr/lib/python3.7/socketserver.py.service_actions
0   11740  11740  7.034     /usr/lib/python3.7/selectors.py.select
0   11740  11740  7.532     /usr/lib/python3.7/socketserver.py.service_actions
0   11740  11740  7.532
```

接下来，我们希望在跟踪点被命中时运行我们的自定义代码，因此我们不完全依赖 BCC 提供的任何工具。 以下示例将自身挂钩到 python 的 function__entry 跟踪点（请参阅 [python 检测](https://docs.python.org/3/howto/instrumentation.html)文档）并在有人下载文件时通知我们：

```
#!/usr/bin/env python
from bcc import BPF, USDT
import sys

bpf = """
#include <uapi/linux/ptrace.h>

static int strncmp(char *s1, char *s2, int size) {for (int i = 0; i < size; ++i)
        if (s1[i] != s2[i])
            return 1;
    return 0;
}

int trace_file_transfers(struct pt_regs *ctx) {
    uint64_t fnameptr;
    char fname[128]={0}, searchname[9]="copyfile";

    bpf_usdt_readarg(2, ctx, &fnameptr);
    bpf_probe_read(&fname, sizeof(fname), (void *)fnameptr);

    if (!strncmp(fname, searchname, sizeof(searchname)))
        bpf_trace_printk("Someone is transferring a file!\\n");
    return 0;
};
"""

u = USDT(pid=int(sys.argv[1]))
u.enable_probe(probe="function__entry", fn_name="trace_file_transfers")
b = BPF(text=bpf, usdt_contexts=[u])
while 1:
    try:
        (_, _, _, _, ts, msg) = b.trace_fields()
    except ValueError:
        continue
    print("%-18.9f %s" % (ts, msg))
```

我们通过再次附加到简单的 http-server 进行测试：

```
$ python -m http.server >/dev/null & sudo ./trace_simplehttp.py $!
[14] 28682
34677.450520000    b'Someone is transferring a file!'
```

上面的例子告诉我们什么时候有人在下载文件，但它不能比这更详细的信息，比如关于谁在下载、下载什么文件等。这是因为 python 只默认启用了几个非常通用的跟踪点（模块加载、函数进入/退出等）。为了获得更多的信息，我们必须在感兴趣的地方定义我们自己的跟踪点，以便我们能够提取相关的数据。

### 3.2 定义我们自己的跟踪点

到目前为止，我们只使用别人定义的跟踪点，但是如果我们的应用程序并没有提供任何跟踪点，或者我们需要添加比现有跟踪点更多的信息，那么我们将不得不添加自己的跟踪点。

添加跟踪点方式有多种，例如 python-core 通过 [pydtrace.h](https://github.com/python/cpython/blob/v3.7.2/Include/pydtrace.h) 和 [pydtrace.d](https://github.com/python/cpython/blob/v3.7.2/Include/pydtrace.d) 使用 systemtap 的开发包 "systemtap-sdt-dev"，但我们将采取另一种方法，使用 [libstapsdt](https://github.com/sthima/libstapsdt)，因为它有一个更简单的 API，更轻巧（只依赖于 libelf），并支持多种语言绑定。为了保持一致性，我们再次把重点放在 python 上，但是跟踪点也可以用其他语言添加，这里有[一个 C 语言示例](https://github.com/sthima/libstapsdt/blob/master/example/demo.c)。

首先，我们给简单的 http 服务器打上补丁，公开跟踪点。代码应该是不言自明的：注意跟踪点的名字 **file_transfer** 及其参数，足够存储两个字符串指针和一个 32 位无符号整数，代表客户端 IP 地址，文件路径和文件大小。

```
diff --git a/usr/lib/python3.7/http/server.py b/usr/lib/python3.7/http/server.py
index ca2dd50..af08e10 100644
--- a/usr/lib/python3.7/http/server.py
+++ b/usr/lib/python3.7/http/server.py
@@ -107,6 +107,13 @@ from functools import partial
 
 from http import HTTPStatus
 
+import stapsdt
+provider = stapsdt.Provider("simplehttp")
+probe = provider.add_probe("file_transfer",
+                           stapsdt.ArgTypes.uint64,
+                           stapsdt.ArgTypes.uint64,
+                           stapsdt.ArgTypes.uint32)+provider.load()
 
 # Default error message template
 DEFAULT_ERROR_MESSAGE = """\
@@ -650,6 +657,8 @@ class SimpleHTTPRequestHandler(BaseHTTPRequestHandler):
         f = self.send_head()
         if f:
             try:
+                path = self.translate_path(self.path)
+                probe.fire(self.address_string(), path, os.path.getsize(path))
                 self.copyfile(f, self.wfile)
             finally:
                 f.close()
```

运行打过补丁的服务器，我们可以使用 tplist 验证我们的 **file_transfer** 跟踪点在运行时是否存在：

```
$ python -m http.server >/dev/null 2>&1 & tplist -p $!
[1] 13297
b'/tmp/simplehttp-Q6SJDX.so' b'simplehttp':b'file_transfer'
b'/usr/lib/libpython3.7m.so.1.0' b'python':b'import__find__load__start'
b'/usr/lib/libpython3.7m.so.1.0' b'python':b'import__find__load__done'
```

我们将对上述示例中的跟踪器示例代码进行以下最重要的修改：

- 它将其逻辑挂接到我们自定义的 **file_transfer** 跟踪点。
- 它使用 [PERF EVENTS](https://perf.wiki.kernel.org/index.php/Tutorial) 来存储可以将任意结构传递到用户空间的数据，而不是我们之前使用的 ftrace 环形缓存区只能传输单个字符串。
- 它**不**使用 **bpf_usdt_readarg** 来获取 USDT 提供的指针，而是直接在处理程序函数签名中声明它们。 这是一个显着的质量改善，可用于所有处理程序。
- 此跟踪器明确使用 **[python2](https://perf.wiki.kernel.org/index.php/Tutorial)**，即使到目前为止我们所有的示例（包括上面的 python http.server 补丁） 使用 **[python3](https://perf.wiki.kernel.org/index.php/Tutorial)**。 希望将来所有 BCC API 和文档都能移植到 python 3。

```
 #!/usr/bin/env python2
 from bcc import BPF, USDT
 import sys

 bpf = """
 #include <uapi/linux/ptrace.h>

 BPF_PERF_OUTPUT(events);

 struct file_transf {char client_ip_str[20];
     char file_path[300];
     u32 file_size;
     u64 timestamp;
 };

 int trace_file_transfers(struct pt_regs *ctx, char *ipstrptr, char *pathptr, u32 file_size) {struct file_transf ft = {0};

     ft.file_size = file_size;
     ft.timestamp = bpf_ktime_get_ns();
     bpf_probe_read(&ft.client_ip_str, sizeof(ft.client_ip_str), (void *)ipstrptr);
     bpf_probe_read(&ft.file_path, sizeof(ft.file_path), (void *)pathptr);

     events.perf_submit(ctx, &ft, sizeof(ft));
     return 0;
 };
 """

 def print_event(cpu, data, size):
     event = b["events"].event(data)
     print("{0}: {1} is downloding file {2} ({3} bytes)".format(event.timestamp, event.client_ip_str, event.file_path, event.file_size))

 u = USDT(pid=int(sys.argv[1]))
 u.enable_probe(probe="file_transfer", fn_name="trace_file_transfers")
 b = BPF(text=bpf, usdt_contexts=[u])
 b["events"].open_perf_buffer(print_event)

 while 1:
     try:
         b.perf_buffer_poll()
     except KeyboardInterrupt:
         exit()
```

跟踪已打过补丁的服务器：

```
$ python -m http.server >/dev/null 2>&1 & sudo ./trace_stapsdt.py $!
[1] 5613
325540469950102: 127.0.0.1 is downloading file /home/adi/ (4096 bytes)
325543319100447: 127.0.0.1 is downloading file /home/adi/.bashrc (827 bytes)
325552448306918: 127.0.0.1 is downloading file /home/adi/workspace/ (4096 bytes)
325563646387008: 127.0.0.1 is downloading file /home/adi/workspace/work.tar (112640 bytes)
(...)
```

上面自定义的 **file_transfer** 跟踪点看起来很简单（直接 python 打印或日志记录调用可能有相同的效果），但它提供的机制非常强大：良好放置的跟踪点保证 ABI 稳定性，提供动态运行的能力安全、本地快速、**可编程**逻辑可以非常有助于快速分析和修复各种问题，而无需重新启动有问题的应用程序（重现问题可能需要很长时间）。

## 4. 动态探针（uprobes）

上面举例说明的静态跟踪点的问题在于，它们需要在源代码中明确定义，并且在修改跟踪点时需要重新构建应用程序。保证现有跟踪点的 ABI 稳定性对维护人员如何重新构建/重写跟踪点数据的代码施加了限制。因此，在某些情况下，完全运行时动态用户空间探测器（uprobes）是首选：它们以特别的方式直接在运行应用程序的内存中进行探测，而无需任何特殊的源代码定义。动态探测器可能会比较容易在应用程序版本之间失效，但即便如此，它们对于实时调试正在运行的实例也很有用。

虽然静态跟踪点对于跟踪用 Python 或 Java 等高级语言编写的应用程序很有用，但 uprobes 对此不太有用，因为它们工作比较底层，并且不了解语言运行时实现（静态跟踪点之所以可以工作，因为开发人员自行承担公开高级应用程序的相关数据）。然而，动态探测器对于调试语言实现/引擎本身或用没有运行时的语言（如 C）编写的应用程序很有用。

可以将 uprobe 添加到优化过（stripped）的二进制文件中，但用户必须手动计算进程内内存偏移位置，uprobe 应通过 objdump 和 /proc//maps 等工具附加到该位置（[参见示例](https://github.com/torvalds/linux/blob/v4.20/Documentation/trace/uprobetracer.rst))，但这种方式比较痛苦且不可移植。 由于大多数发行版都提供调试符号包（或使用调试符号构建的快速方法）并且 BCC 使得使用带有符号名称解析的 uprobes 变得简单，因此绝大多数动态探测使用都是以这种方式进行的。

[gethostlatency](https://github.com/iovisor/bcc/blob/master/tools/gethostlatency.py) BCC 工具通过将 uprobes 附加到 gethostbyname 和 libc 中的相关函数来打印 DNS 请求延迟。 要验证 libc 未优化（stripped）以便可以运行该工具（否则会引发 sybol-not-found 错误）：

```
$ file /usr/lib/libc-2.28.so
/usr/lib/libc-2.28.so: ELF 64-bit LSB shared object, x86-64, version 1 (GNU/Linux), dynamically linked, (...), not stripped
$ nm -na /usr/lib/libc-2.28.so | grep -i -e getaddrinfo
0000000000000000 a getaddrinfo.c
```

[gethostlatency](https://github.com/iovisor/bcc/blob/master/tools/gethostlatency.py) 代码与我们上面检查的跟踪点示例非常相似（并且在某些地方相同，它还使用 BPF_PERF_OUTPUT） ，所以我们不会在这里完整地发布它。 最相关的区别是使用 [BCC uprobe API](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#4-attach_uprobe)：

```
b.attach_uprobe(name="c", sym="getaddrinfo", fn_name="do_entry", pid=args.pid)
b.attach_uretprobe(name="c", sym="getaddrinfo", fn_name="do_return", pid=args.pid)
```

这里需要理解和记住的关键思想是：只要对我们的 BCC eBPF 程序做一些小的改动，我们就可以通过静态和动态探测来跟踪非常不同的应用程序、库甚至是内核。之前我们是静态跟踪 Python 应用程序，现在我们是动态地测量 libc 的主机名解析延时。通过对这些小的（小于 150LOC，很多是模板）例子进行类似的修改，可在运行的系统中跟踪任何内容，这非常安全，没有崩溃的风险或其他工具引起的问题（如调试器应用程序暂停/停顿）。

## 5. 总结

在第 5 部分中，我们研究了如何使用 eBPF 程序来跟踪用户空间应用程序。 使用 eBPF 完成这项任务的最大优势是它提供了一个统一的接口来安全有效地跟踪整个系统：可以在应用程序中重现错误，然后进一步跟踪到库或内核中，通过统一的编程框架/接口提供完整的系统可见性。 然而，eBPF 并不是银弹，尤其是在调试用高级语言编写的应用程序时，特定语言的工具可以更好地提供洞察力，或者对于那些运行旧版本 Linux 内核或需要非 Linux 系统的应用程序。
