# linux cgroups 概述

从 2.6.24 版本开始，linux 内核提供了一个叫做 cgroups（控制组）的特性。cgroups 就是 control groups 的缩写，用来对一组进程所占用的资源做限制、统计、隔离。也是目前轻量级虚拟化技术 lxc （linux container）的基础之一。每一组进程就是一个控制组，也就是一个 cgroup。cgroups 分为几个子系统，每个子系统代表一种设施或者说是资源控制器，用来调度某一类资源的使用，如 cpu 时钟、内存、块设备 等。在实现上，cgroups 并没有增加新的系统调用，而是表现为一个 cgroup 文件系统，可以把一个或多个子系统挂载到某个目录。如

```
mount -t cgroup -o cpu cpu /sys/fs/cgroup/cpu
```

就将 cpu 子系统挂载在了 /sys/fs/cgroup/cpu 。也可以在一个目录上挂载多个子系统，甚至全部挂载到一个目录也是可以的，不过我觉得，把每个子系统都挂载在不同目录会有更好的灵活性。用 `mount|awk '$5=="cgroup" {print $0}'` 可以看到当前挂载的控制组。用 `cat /proc/cgroups` 可以看到当前所有控制组的状态。下面这个脚本，可以把全部子系统各种挂载到各自的目录上去。

```
#!/bin/bash

cgroot="${1:-/sys/fs/cgroup}"
subsys="${2:-blkio cpu cpuacct cpuset devices freezer memory net_cls net_prio ns perf_event}"

mount -t tmpfs cgroup_root "${cgroot}"
for ss in $subsys; do
  mkdir -p "$cgroot/$ss"
  mount -t cgroup -o "$ss" "$ss" "$cgroot/$ss"
done
```

看看那些目录里都有些啥，比如 ls 一下 /sys/fs/cgroup/cpu。

```
cgroup.event_control  cpu.cfs_period_us  cpu.rt_period_us   cpu.shares  notify_on_release  tasks
cgroup.procs          cpu.cfs_quota_us   cpu.rt_runtime_us  cpu.stat    release_agent
```

其中 “cpu.” 开头的就是这个子系统里特有的东西。其他的那些是每个子系统所对应目录里都有的。这些文件就是用来读取资源使用信息和进行资源限制的。要创建一个控制组，就在需要的子系统里创建一个目录即可。如 `mkdir /sys/fs/cgroup/cpu/foo` 就创建了一个 /foo 的控制组。在新建的目录里就会出现同样一套文件。在这个目录里，也一样可以继续通过创建目录来创建 cgroup。也就是说，cgroup 是可以和目录结构一样有层次的。对与每个子系统挂载点点目录，就相当于根目录。每一条不同的路径就代表了一个不同的 cgroup。在不同的子系统里，路径相同就代表了同一个控制组。如，在 cpu、memory 中都有 foo/bar 目录，就可以用 那 /foo/bar 来操作 cpu、memory 两个子系统。对于同一个子系统，每个进程都属于且只属于一个 cgroup，默认是在根 cgroup。层次结构方便了控制组的组织和管理，对于某些配置项来说，层次结构还和资源分配有关。另外，也可以修改某个目录的 owner ，让非 root 用户也能操作某些特定的安全组。

cgroups 的设置和信息读取是通过对那些文件的读写来进行的。例如

```
# echo 2048 >/sys/fs/cgroup/cpu/foo/cpu.shares
```

就把 /foo 这个控制组的 cpu.shares 参数设为了 2048。

前面说，有些文件是每个目录里共有的。那些就是通用的设置。其中，tasks 和 cgroups.procs 是用来管理控制组中的进程的。要把一个进程加入到某个控制组，把 pid 写入到相应目录的 tasks 文件即可。如

```
# echo 5678 >/sys/fs/cgroup/cpu/foo/tasks
```

就把 5678 进程加入到了 /foo 控制组。那么 tasks 和 cgroups.procs 有什么区别呢？前面说的对“进程”的管理限制其实不够准确。系统对任务调度的单位是线程。在这里，tasks 中看到的就是线程 id。而 cgroups.procs 中是线程组 id，也就是一般所说的进程 id 。将一个一般的 pid 写入到 tasks 中，只有这个 pid 对应的线程，以及由它产生的其他进程、线程会属于这个控制组，原有的其他线程则不会。而写入 cgroups.procs 会把当前所有的线程都加入进去。如果写入 cgroups.procs 的不是一个线程组 id，而是一个一般的线程 id，那会自动找到所对应的线程组 id 加入进去。进程在加入一个控制组后，控制组所对应的限制会即时生效。想知道一个进程属于哪些控制组，可以通过 `cat /proc/<pid>/cgroup` 查看。

要把进程移出控制组，把 pid 写入到根 cgroup 的 tasks 文件即可。因为每个进程都属于且只属于一个 cgroup，加入到新的 cgroup 后，原有关系也就解除了。要删除一个 cgroup，可以用 rmdir 删除相应目录。不过在删除前，必须先让其中的进程全部退出，对应子系统的资源都已经释放，否则是无法删除的。

前面都是通过文件系统访问方式来操作 cgroups 的。实际上，也有一组命令行工具。

`lssubsys -am` 可以查看各子系统的挂载点，还有一组“cg”开头的命令可以用来管理。其中 cgexec 可以用来直接在某些子系统中的指定控制组运行一个程序。如 `cgexec -g "cpu,blkio:/foo" bash` 。其他的命令和具体的参数可以通过 man 来查看。

下面是个 bash 版的 cgexec，演示了 cgroups 的用法，也可以在不确定是否安装命令行工具的情况下使用。

```
#!/bin/bash

# usage: 
# ./cgexec.sh cpu:g1,memory:g2/g21 sleep 100

blkio_dir="/sys/fs/cgroup/blkio"
memory_dir="/sys/fs/cgroup/memory"
cpuset_dir="/sys/fs/cgroup/cpuset"
perf_event_dir="/sys/fs/cgroup/perf_event"
freezer_dir="/sys/fs/cgroup/freezer"
net_cls_dir="/sys/fs/cgroup/net_cls"
cpuacct_dir="/sys/fs/cgroup/cpuacct"
cpu_dir="/sys/fs/cgroup/cpu"
hugetlb_dir="/sys/fs/cgroup/hugetlb"
devices_dir="/sys/fs/cgroup/devices"

groups="$1"
shift

IFS=',' g_arr=($groups)
for g in ${g_arr[@]}; do
  IFS=':' g_info=($g)
  if [ ${#g_info[@]} -ne 2 ]; then
    echo "bad arg $g" >&2
    continue
  fi
  g_name=${g_info[0]}
  g_path=${g_info[1]}
  if [ "$g_path" == "${g_path#/}" ]; then
    g_path="/$g_path"
  fi
  echo $g_name $g_path
  var="${g_name}_dir"
  d=${!var}
  if [ -z "$d" ]; then
    echo "bad cg name $g_name" >&2
    continue
  fi
  path="${d}${g_path}"
  if [ ! -d "$path" ]; then
    echo "cg not exists" >&2
    continue
  fi
  echo "$$" >"${path}/tasks"
done

exec $*
```

参考资料：
[cgroups docs – kernel.org](https://www.kernel.org/doc/Documentation/cgroups/)

[Resource Management Guide – redhat.com](https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/pref-Resource_Management_Guide-Preface.html)

[How I Used CGroups to Manage System Resources – oracle.com](http://www.oracle.com/technetwork/articles/servers-storage-admin/resource-controllers-linux-1506602.html)

> 原文链接：http://xiezhenye.com/2013/10/linux-cgroups-%e6%a6%82%e8%bf%b0.html
