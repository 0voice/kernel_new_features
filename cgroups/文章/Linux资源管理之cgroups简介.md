### 引子

cgroups 是Linux内核提供的一种可以限制单个进程或者多个进程所使用资源的机制，可以对 cpu，内存等资源实现精细化的控制，目前越来越火的轻量级容器 Docker 就使用了 cgroups 提供的资源限制能力来完成cpu，内存等部分的资源控制。

另外，开发者也可以使用 cgroups 提供的精细化控制能力，限制某一个或者某一组进程的资源使用。比如在一个既部署了前端 web 服务，也部署了后端计算模块的八核服务器上，可以使用 cgroups 限制 web server 仅可以使用其中的六个核，把剩下的两个核留给后端计算模块。

本文从以下四个方面描述一下 cgroups 的原理及用法：

1. cgroups 的概念及原理
2. cgroups 文件系统概念及原理
3. cgroups 使用方法介绍
4. cgroups 实践中的例子

### 概念及原理

#### cgroups子系统

cgroups 的全称是control groups，cgroups为每种可以控制的资源定义了一个子系统。典型的子系统介绍如下：

1. cpu 子系统，主要限制进程的 cpu 使用率。
2. cpuacct 子系统，可以统计 cgroups 中的进程的 cpu 使用报告。
3. cpuset 子系统，可以为 cgroups 中的进程分配单独的 cpu 节点或者内存节点。
4. memory 子系统，可以限制进程的 memory 使用量。
5. blkio 子系统，可以限制进程的块设备 io。
6. devices 子系统，可以控制进程能够访问某些设备。
7. net_cls 子系统，可以标记 cgroups 中进程的网络数据包，然后可以使用 tc 模块（traffic control）对数据包进行控制。
8. freezer 子系统，可以挂起或者恢复 cgroups 中的进程。
9. ns 子系统，可以使不同 cgroups 下面的进程使用不同的 namespace。

这里面每一个子系统都需要与内核的其他模块配合来完成资源的控制，比如对 cpu 资源的限制是通过进程调度模块根据 cpu 子系统的配置来完成的；对内存资源的限制则是内存模块根据 memory 子系统的配置来完成的，而对网络数据包的控制则需要 Traffic Control 子系统来配合完成。本文不会讨论内核是如何使用每一个子系统来实现资源的限制，而是重点放在内核是如何把 cgroups 对资源进行限制的配置有效的组织起来的，和内核如何把cgroups 配置和进程进行关联的，以及内核是如何通过 cgroups 文件系统把cgroups的功能暴露给用户态的。

#### cgroups 层级结构（Hierarchy）

内核使用 cgroup 结构体来表示一个 control group 对某一个或者某几个 cgroups 子系统的资源限制。cgroup 结构体可以组织成一颗树的形式，每一棵cgroup 结构体组成的树称之为一个 cgroups 层级结构。cgroups层级结构可以 attach 一个或者几个 cgroups 子系统，当前层级结构可以对其 attach 的 cgroups 子系统进行资源的限制。每一个 cgroups 子系统只能被 attach 到一个 cpu 层级结构中。

![cgroups层级结构示意图](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2015/b3270d03.png)

比如上图表示两个cgroups层级结构，每一个层级结构中是一颗树形结构，树的每一个节点是一个 cgroup 结构体（比如cpu_cgrp, memory_cgrp)。第一个 cgroups 层级结构 attach 了 cpu 子系统和 cpuacct 子系统， 当前 cgroups 层级结构中的 cgroup 结构体就可以对 cpu 的资源进行限制，并且对进程的 cpu 使用情况进行统计。 第二个 cgroups 层级结构 attach 了 memory 子系统，当前 cgroups 层级结构中的 cgroup 结构体就可以对 memory 的资源进行限制。

在每一个 cgroups 层级结构中，每一个节点（cgroup 结构体）可以设置对资源不同的限制权重。比如上图中 cgrp1 组中的进程可以使用60%的 cpu 时间片，而 cgrp2 组中的进程可以使用20%的 cpu 时间片。

\####cgroups与进程

上面的小节提到了内核使用 cgroups 子系统对系统的资源进行限制，也提到了 cgroups 子系统需要 attach 到 cgroups 层级结构中来对进程进行资源控制。本小节重点关注一下内核是如何把进程与 cgroups 层级结构联系起来的。

在创建了 cgroups 层级结构中的节点（cgroup 结构体）之后，可以把进程加入到某一个节点的控制任务列表中，一个节点的控制列表中的所有进程都会受到当前节点的资源限制。同时某一个进程也可以被加入到不同的 cgroups 层级结构的节点中，因为不同的 cgroups 层级结构可以负责不同的系统资源。所以说进程和 cgroup 结构体是一个多对多的关系。

![cgroups层级结构示意图](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2015/3982f44c.png)

上面这个图从整体结构上描述了进程与 cgroups 之间的关系。最下面的`P`代表一个进程。每一个进程的描述符中有一个指针指向了一个辅助数据结构`css_set`（cgroups subsystem set）。 指向某一个`css_set`的进程会被加入到当前`css_set`的进程链表中。一个进程只能隶属于一个`css_set`，一个`css_set`可以包含多个进程，隶属于同一`css_set`的进程受到同一个`css_set`所关联的资源限制。

上图中的”M×N Linkage”说明的是`css_set`通过辅助数据结构可以与 cgroups 节点进行多对多的关联。但是 cgroups 的实现不允许`css_set`同时关联同一个cgroups层级结构下多个节点。 这是因为 cgroups 对同一种资源不允许有多个限制配置。

一个`css_set`关联多个 cgroups 层级结构的节点时，表明需要对当前`css_set`下的进程进行多种资源的控制。而一个 cgroups 节点关联多个`css_set`时，表明多个`css_set`下的进程列表受到同一份资源的相同限制。

### cgroups文件系统

Linux 使用了多种数据结构在内核中实现了 cgroups 的配置，关联了进程和 cgroups 节点，那么 Linux 又是如何让用户态的进程使用到 cgroups 的功能呢？ Linux内核有一个很强大的模块叫 VFS (Virtual File System)。 VFS 能够把具体文件系统的细节隐藏起来，给用户态进程提供一个统一的文件系统 API 接口。 cgroups 也是通过 VFS 把功能暴露给用户态的，cgroups 与 VFS 之间的衔接部分称之为 cgroups 文件系统。下面先介绍一下 VFS 的基础知识，然后再介绍下 cgroups 文件系统的实现。

#### VFS

VFS 是一个内核抽象层，能够隐藏具体文件系统的实现细节，从而给用户态进程提供一套统一的 API 接口。VFS 使用了一种通用文件系统的设计，具体的文件系统只要实现了 VFS 的设计接口，就能够注册到 VFS 中，从而使内核可以读写这种文件系统。 这很像面向对象设计中的抽象类与子类之间的关系，抽象类负责对外接口的设计，子类负责具体的实现。其实，VFS本身就是用 c 语言实现的一套面向对象的接口。

##### 通用文件模型

VFS 通用文件模型中包含以下四种元数据结构：

1. 超级块对象(superblock object)，用于存放已经注册的文件系统的信息。比如ext2，ext3等这些基础的磁盘文件系统，还有用于读写socket的socket文件系统，以及当前的用于读写cgroups配置信息的 cgroups 文件系统等。
2. 索引节点对象(inode object)，用于存放具体文件的信息。对于一般的磁盘文件系统而言，inode 节点中一般会存放文件在硬盘中的存储块等信息；对于socket文件系统，inode会存放socket的相关属性，而对于cgroups这样的特殊文件系统，inode会存放与 cgroup 节点相关的属性信息。这里面比较重要的一个部分是一个叫做 inode_operations 的结构体，这个结构体定义了在具体文件系统中创建文件，删除文件等的具体实现。
3. 文件对象(file object)，一个文件对象表示进程内打开的一个文件，文件对象是存放在进程的文件描述符表里面的。同样这个文件中比较重要的部分是一个叫 file_operations 的结构体，这个结构体描述了具体的文件系统的读写实现。当进程在某一个文件描述符上调用读写操作时，实际调用的是 file_operations 中定义的方法。 对于普通的磁盘文件系统，file_operations 中定义的就是普通的块设备读写操作；对于socket文件系统，file_operations 中定义的就是 socket 对应的 send/recv 等操作；而对于cgroups这样的特殊文件系统，file_operations 中定义的就是操作 cgroup 结构体等具体的实现。
4. 目录项对象(dentry object)，在每个文件系统中，内核在查找某一个路径中的文件时，会为内核路径上的每一个分量都生成一个目录项对象，通过目录项对象能够找到对应的 inode 对象，目录项对象一般会被缓存，从而提高内核查找速度。

\#####cgroups文件系统的实现

基于 VFS 实现的文件系统,都必须实现 VFS 通用文件模型定义的这些对象，并实现这些对象中定义的部分函数。cgroup 文件系统也不例外,下面来看一下 cgroups 中这些对象的定义。

首先看一下 cgroups 文件系统类型的结构体：

```c
static struct file_system_type cgroup_fs_type = {
        .name = "cgroup",
        .mount = cgroup_mount,
        .kill_sb = cgroup_kill_sb,
};
```

这里面两个函数分别代表安装和卸载某一个 cgroup 文件系统所需要执行的函数。每次把某一个 cgroups 子系统安装到某一个装载点的时候，cgroup_mount 方法就会被调用，这个方法会生成一个 cgroups_root（cgroups层级结构的根）并封装成超级快对象。

然后看一下 cgroups 超级块对象定义的操作：

```c
static const struct super_operations cgroup_ops = {
        .statfs = simple_statfs,
        .drop_inode = generic_delete_inode,
        .show_options = cgroup_show_options,
        .remount_fs = cgroup_remount,
};
```

这里只有部分函数的实现，这是因为对于特定的文件系统而言，所支持的操作可能仅是 super_operations 中所定义操作的一个子集，比如说对于块设备上的文件对象，肯定是支持类似 fseek 的查找某个位置的操作，但是对于 socket 或者 cgroups 这样特殊的文件系统，就不支持这样的操作。

同样简单看下 cgroups 文件系统对 inode 对象和 file 对象定义的特殊实现函数：

```c
static const struct inode_operations cgroup_dir_inode_operations = {
        .lookup = cgroup_lookup,
        .mkdir = cgroup_mkdir,
        .rmdir = cgroup_rmdir,
        .rename = cgroup_rename,
};

static const struct file_operations cgroup_file_operations = {
        .read = cgroup_file_read,
        .write = cgroup_file_write,
        .llseek = generic_file_llseek,
        .open = cgroup_file_open,
        .release = cgroup_file_release,
};
```

本文并不去研究这些函数的代码实现是什么样的，但是从这些代码可以推断出，cgroups 通过实现 VFS 的通用文件系统模型，把维护 cgroups 层级结构的细节，隐藏在 cgroups 文件系统的这些实现函数中。

从另一个方面说，用户在用户态对 cgroups 文件系统的操作，通过 VFS 转化为对 cgroups 层级结构的维护。通过这样的方式，内核把 cgroups 的功能暴露给了用户态的进程。

### cgroups使用方法

#### cgroups文件系统挂载

Linux中，用户可以使用mount命令挂载 cgroups 文件系统，格式为: `mount -t cgroup -o subsystems name /cgroup/name`，其中 subsystems 表示需要挂载的 cgroups 子系统， /cgroup/name 表示挂载点，如上文所提，这条命令同时在内核中创建了一个cgroups 层级结构。

比如挂载 cpuset, cpu, cpuacct, memory 4个subsystem到/cgroup/cpu_and_mem 目录下，就可以使用 `mount -t cgroup -o remount,cpu,cpuset,memory cpu_and_mem /cgroup/cpu_and_mem`

在centos下面，在使用`yum install libcgroup`安装了cgroups模块之后，在 /etc/cgconfig.conf 文件中会自动生成 cgroups 子系统的挂载点:

```c
mount {
    cpuset  = /cgroup/cpuset;
    cpu = /cgroup/cpu;
    cpuacct = /cgroup/cpuacct;
    memory  = /cgroup/memory;
    devices = /cgroup/devices;
    freezer = /cgroup/freezer;
    net_cls = /cgroup/net_cls;
    blkio   = /cgroup/blkio;
}
```

上面的每一条配置都等价于展开的 mount 命令，例如`mount -t cgroup -o cpuset cpuset /cgroup/cpuset`。这样系统启动之后会自动把这些子系统挂载到相应的挂载点上。

\####子节点和进程

挂载某一个 cgroups 子系统到挂载点之后，就可以通过在挂载点下面建立文件夹或者使用cgcreate命令的方法创建 cgroups 层级结构中的节点。比如通过命令`cgcreate -t sankuai:sankuai -g cpu:test`就可以在 cpu 子系统下建立一个名为 test 的节点。结果如下所示：

```shell
[root@idx cpu]# ls
cgroup.event_control  cgroup.procs  cpu.cfs_period_us  cpu.cfs_quota_us  cpu.rt_period_us   cpu.rt_runtime_us  cpu.shares  cpu.stat  lxc  notify_on_release  release_agent  tasks  test
```

然后可以通过写入需要的值到 test 下面的不同文件，来配置需要限制的资源。每个子系统下面都可以进行多种不同的配置，需要配置的参数各不相同，详细的参数设置需要参考 cgroups 手册。使用 cgset 命令也可以设置 cgroups 子系统的参数，格式为 `cgset -r parameter=value path_to_cgroup`。

当需要删除某一个 cgroups 节点的时候，可以使用 cgdelete 命令，比如要删除上述的 test 节点，可以使用 `cgdelete -r cpu:test`命令进行删除

把进程加入到 cgroups 子节点也有多种方法，可以直接把 pid 写入到子节点下面的 task 文件中。也可以通过 cgclassify 添加进程，格式为 `cgclassify -g subsystems:path_to_cgroup pidlist`，也可以直接使用 cgexec 在某一个 cgroups 下启动进程，格式为`gexec -g subsystems:path_to_cgroup command arguments`.

### 实践中的例子

相信大多数人都没有读过 Docker 的源代码，但是通过这篇文章，可以估计 Docker 在实现不同的 Container 之间资源隔离和控制的时候，是可以创建比较复杂的 cgroups 节点和配置文件来完成的。然后对于同一个 Container 中的进程，可以把这些进程 PID 添加到同一组 cgroups 子节点中已达到对这些进程进行同样的资源限制。

通过各大互联网公司在网上的技术文章，也可以看到很多公司的云平台都是基于 cgroups 技术搭建的，其实也都是把进程分组，然后把整个进程组添加到同一组 cgroups 节点中，受到同样的资源限制。

笔者所在的广告组，有一部分任务是给合作的广告投放网站生成“商品信息”，广告投放网站使用这些信息，把广告投放在他们各自的网站上。但是有时候会有恶意的爬虫过来爬取商品信息，所以我们生成了另外“一小份”数据供优先级较低的用户下载，这时候基本能够区分开大部分恶意爬虫。对于这样的“一小份”数据，对及时更新的要求不高，生成商品信息又是一个比较费资源的任务，所以我们把这个任务的cpu资源使用率限制在了50%。

首先在 cpu 子系统下面创建了一个 halfapi 的子节点：`cgcreate abc:abc -g cpu:halfapi`。

然后在配置文件中写入配置数据：`echo 50000 > /cgroup/cpu/halfapi/cpu.cfs_quota_us`。`cpu.cfs_quota_us`中的默认值是100000，写入50000表示只能使用50%的 cpu 运行时间。

最后在这个cgroups中启动这个任务：`cgexec -g "cpu:/halfapi" php halfapi.php half >/dev/null 2>&1`

在 cgroups 引入内核之前，想要完成上述的对某一个进程的 cpu 使用率进行限制，只能通过 nice 命令调整进程的优先级，或者 cpulimit 命令限制进程使用进程的 cpu 使用率。但是这些命令的缺点是无法限制一个进程组的资源使用限制，也就无法完成 Docker 或者其他云平台所需要的这一类轻型容器的资源限制要求。

同样，在 cgroups 之前，想要完成对某一个或者某一组进程的物理内存使用率的限制，几乎是不可能完成的。使用 cgroups 提供的功能，可以轻易的限制系统内某一组服务的物理内存占用率。 对于网络包，设备访问或者io资源的控制，cgroups 同样提供了之前所无法完成的精细化控制。

### 结语

本文首先介绍了 cgroups 在内核中的实现方式，然后介绍了 cgroups 如何通过 VFS 把相关的功能暴露给用户，然后简单介绍了 cgroups 的使用方法，最后通过分析了几个 cgroups 在实践中的例子，进一步展示了 cgroups 的强大的精细化控制能力。

> 原文链接：https://tech.meituan.com/2015/03/31/cgroups.html
