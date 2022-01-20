# 用 cgruops 管理进程内存占用

cgroups 中有个 memory 子系统，用于限制和报告进程的内存使用情况。

其中，很明显有两组对应的文件，一组带 memsw ，另一组不带

```
memory.failcnt
memory.limit_in_bytes
memory.max_usage_in_bytes
memory.usage_in_bytes

memory.memsw.failcnt
memory.memsw.limit_in_bytes
memory.memsw.max_usage_in_bytes
memory.memsw.usage_in_bytes
```

带 memsw 的表示虚拟内存，即物理内存加交换区。不带 memsw 的那组仅包括物理内存。其中，limit_in_bytes 是用来限制内存使用的，其他的则是统计报告。

```
# echo 10485760 >/sys/fs/cgroup/memory/foo/memory.limit_in_bytes
```

即可限制该组中的进程使用的物理内存总量不超过 10MB。对 memory.memsw.limit_in_bytes 来说，则是限制虚拟内存使用。memory.memsw.limit_in_bytes 必须大于或等于 memory.limit_in_byte。这些值还可以用更方便的 100M，20G 这样的形式来设置。要解除限制，就把这个值设为 -1 即可。

这种方式限制进程内存占用会有个风险。当进程试图占用的内存超过限制，访问内存时发生缺页，又没有足够的非活动内存页可以换出时会触发 oom ，导致进程直接被杀，从而造成可用性问题。即使关闭控制组的 oom killer，进程在内存不足的时候，虽然不会被杀，但是会长时间进入 D （等待系统调用的不可中断休眠）状态，无法继续执行，导致仍然无法服务。因此，我认为，用 memory.limit_in_bytes 或 memory.memsw.limit_in_bytes 限制进程内存占用仅应当作为一个保险，避免在进程异常时耗尽系统资源。如，预期一组进程最多只会消耗 1G 内存，那么可以设置为 1.4G 。这样在发生内存泄露等异常情况时，可以避免造成更严重问题。

在 memory 子系统中，还有一个 memory.soft_limit_in_bytes 。和 memory.limit_in_bytes 的差异是，这个限制并不会阻止进程使用超过限额的内存，只是在系统内存不足时，会优先回收超过限额的进程占用的内存，使之向限定值靠拢。

前面说控制组的 oom killer 是可以关闭的，就是通过 memory.oom_control 来实现的。`cat memory.oom_control` 可以看到当前设置以及目前是否触发了 oom 。`echo 1 >memory.oom_control` 就可以禁用 oom killer。

usage_in_bytes、max_usage_in_bytes、failcnt 则分别对应 当前使用量，最高使用量和发生的缺页次数。

memory 子系统中还有一个很重要的设置是 memory.use_hierarchy 这是个布尔开关，默认为 0。此时不同层次间的资源限制和使用值都是独立的。当设为 1 时，子控制组进程的内存占用也会计入父控制组，并上溯到所有 memory.use_hierarchy = 1 的祖先控制组。这样一来，所有子孙控制组的进程的资源占用都无法超过父控制组设置的资源限制。同时，在整个树中的进程的内存占用达到这个限制时，内存回收也会影响到所有子孙控制组的进程。这个值只有在还没有子控制组时才能设置。之后在其中新建的子控制组默认的 memory.use_hierarchy 也会继承父控制组的设置。

memory.swappiness 则是控制内核使用交换区的倾向的。值的范围是 0 – 100。值越小，越倾向使用物理内存。设为 0 时，只有在物理内存不足时才会使用交换区。默认值是系统全局设置： /proc/sys/vm/swappiness 。

memory.stat 就是内存使用情况报告了。包括当前资源总量、使用量、换页次数、活动页数量等等。

> 原文链接：http://xiezhenye.com/2013/10/%e7%94%a8-cgruops-%e7%ae%a1%e7%90%86%e8%bf%9b%e7%a8%8b%e5%86%85%e5%ad%98%e5%8d%a0%e7%94%a8.html
