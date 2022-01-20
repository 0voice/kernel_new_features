# 用 cgroups 管理 cpu 资源

这回说说怎样通过 cgroups 来管理 cpu 资源。先说控制进程的 cpu 使用。在一个机器上运行多个可能消耗大量资源的程序时，我们不希望出现某个程序占据了所有的资源，导致其他程序无法正常运行，或者造成系统假死无法维护。这时候用 cgroups 就可以很好地控制进程的资源占用。这里单说 cpu 资源。

cgroups 里，可以用 cpu.cfs_period_us 和 cpu.cfs_quota_us 来限制该组中的所有进程在单位时间里可以使用的 cpu 时间。这里的 cfs 是完全公平调度器的缩写。cpu.cfs_period_us 就是时间周期，默认为 100000，即百毫秒。cpu.cfs_quota_us 就是在这期间内可使用的 cpu 时间，默认 -1，即无限制。

跑一个耗 cpu 的程序

```
# echo 'while True: pass'|python &
[1] 1532
```

top 一下可以看到，这进程占了 100% 的 cpu

```
  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
 1532 root      20   0  112m 3684 1708 R 99.6  0.7   0:30.42 python
...
```

然后就来对这个进程做一下限制。先把 /foo 这个控制组的限制修改一下，然后把进程加入进去。

```
echo 50000 >/sys/fs/cgroup/cpu/foo/cpu.cfs_quota_us
echo 1532 >/sys/fs/group/cpu/foo/tasks
```

可见，修改设置只需要写入相应文件，将进程加入 cgroup 也只需将 pid 写入到其中的 tasks 文件即可。这里将 cpu.cfs_quota_us 设为 50000，相对于 cpu.cfs_period_us 的 100000 即 50%。再 top 一下看看效果。

```
  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
 1532 root      20   0  112m 3684 1708 R 50.2  0.7   5:00.31 python
...
```

可以看到，进程的 cpu 占用已经被成功地限制到了 50% 。这里，测试的虚拟机只有一个核心。在多核情况下，看到的值会不一样。另外，cfs_quota_us 也是可以大于 cfs_period_us 的，这主要是对于多核情况。有 n 个核时，一个控制组中的进程自然最多就能用到 n 倍的 cpu 时间。

这两个值在 cgroups 层次中是有限制的，下层的资源不能超过上层。具体的说，就是下层的 cpu.cfs_period_us 值不能小于上层的值，cpu.cfs_quota_us 值不能大于上层的值。

另外的一组 cpu.rt_period_us、cpu.rt_runtime_us 对应的是实时进程的限制，平时可能不会有机会用到。

在 cpu 子系统中，cpu.stat 就是用前面那种方法做的资源限制的统计了。nr_periods、nr_throttled 就是总共经过的周期，和其中受限制的周期。throttled_time 就是总共被控制组掐掉的 cpu 使用时间。

还有个 cpu.shares， 它也是用来限制 cpu 使用的。但是与 cpu.cfs_quota_us、cpu.cfs_period_us 有挺大区别。cpu.shares 不是限制进程能使用的绝对的 cpu 时间，而是控制各个组之间的配额。比如

```
/cpu/cpu.shares : 1024
/cpu/foo/cpu.shares : 2048
```

那么当两个组中的进程都满负荷运行时，/foo 中的进程所能占用的 cpu 就是 / 中的进程的两倍。如果再建一个 /foo/bar 的 cpu.shares 也是 1024，且也有满负荷运行的进程，那 /、/foo、/foo/bar 的 cpu 占用比就是 1:2:1 。前面说的是各自都跑满的情况。如果其他控制组中的进程闲着，那某一个组的进程完全可以用满全部 cpu。可见通常情况下，这种方式在保证公平的情况下能更充分利用资源。

此外，还可以限定进程可以使用哪些 cpu 核心。cpuset 子系统就是处理进程可以使用的 cpu 核心和内存节点，以及其他一些相关配置。这部分的很多配置都和 NUMA 有关。其中 cpuset.cpus、cpuset.mems 就是用来限制进程可以使用的 cpu 核心和内存节点的。这两个参数中 cpu 核心、内存节点都用 id 表示，之间用 “,” 分隔。比如 0,1,2 。也可以用 “-” 表示范围，如 0-3 。两者可以结合起来用。如“0-2,6,7”。在添加进程前，cpuset.cpus、cpuset.mems 必须同时设置，而且必须是兼容的，否则会出错。例如

```
# echo 0 >/sys/fs/cgroup/cpuset/foo/cpuset.cpus
# echo 0 >/sys/fs/cgroup/cpuset/foo/cpuset.mems
```

这样， /foo 中的进程只能使用 cpu0 和内存节点0。用

```
# cat /proc/<pid>/status|grep '_allowed_list'
```

可以验证效果。

cgroups 除了用来限制资源使用外，还有资源统计的功能。做云计算的计费就可以用到它。有一个 cpuacct 子系统专门用来做 cpu 资源统计。cpuacct.stat 统计了该控制组中进程用户态和内核态的 cpu 使用量，单位是 USER_HZ，也就是 jiffies、cpu 滴答数。每秒的滴答数可以用 `getconf CLK_TCK` 来获取，通常是 100。将看到的值除以这个值就可以换算成秒。

cpuacct.usage 和 cpuacct.usage_percpu 是该控制组中进程消耗的 cpu 时间，单位是纳秒。后者是分 cpu 统计的。

P.S. 2014-4-22

发现在 SLES 11 sp2、sp3 ，对应内核版本 3.0.13、 3.0.76 中，对 cpu 子系统，将 pid 写入 cgroup.procs 不会实际生效，要写入 tasks 才行。在其他环境中，更高版本或更低版本内核上均未发现。

> 原文链接：http://xiezhenye.com/2013/10/%e7%94%a8-cgroups-%e7%ae%a1%e7%90%86-cpu-%e8%b5%84%e6%ba%90.html
