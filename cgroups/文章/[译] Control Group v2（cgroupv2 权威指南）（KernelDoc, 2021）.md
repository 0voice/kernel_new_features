# [译] Control Group v2（cgroupv2 权威指南）（KernelDoc, 2021）

Published at 2021-09-10 | Last Update 2021-09-10

### 译者序

本文翻译自 2021 年 Linux `5.10` 内核文档： [Control Group v2](https://www.kernel.org/doc/html/v5.10/admin-guide/cgroup-v2.html)， 它是描述 cgroupv2 **用户空间侧**的设计、接口和规范的**权威文档**。

原文非常全面详细，本文只翻译了目前感兴趣的部分，其他部分保留原文。 另外，由于技术规范的描述比较抽象，因此翻译时加了一些系统测试输出、内核代码片段和 链接，便于更好理解。

**由于译者水平有限，本文不免存在遗漏或错误之处。如有疑问，请查阅原文。**

本文（指[英文原文](https://www.kernel.org/doc/html/v5.10/admin-guide/cgroup-v2.html)） 是描述 cgroup v2 设计、接口和规范的**权威文档**。 未来所有改动/变化都需反应到本文档中。v1 的文档见 [cgroup-v1](https://www.kernel.org/doc/html/v5.10/admin-guide/cgroup-v1/index.html)。

本文描述 cgroup v2 所有**用户空间可见的部分**，包括 cgroup core 和各 controller。

# 1 引言

## 1.1 术语

“cgroup” 是 “control group” 的缩写，并且**首字母永远不大写**（never capitalized）。

- 单数形式（cgroup）指这个特性，或用于 “cgroup controllers” 等术语中的修饰词。
- 复数形式（cgroups）显式地指多个 cgroup。

## 1.2 cgroup 是什么？

cgroup 是一种**以 hierarchical（树形层级）方式组织进程的机制**（a mechanism to organize processes hierarchically），以及在层级中**以受控和 可配置的方式**（controlled and configurable manner）**分发系统资源** （distribute system resources）。

### 1.2.1 cgroup 组成部分

cgroup 主要由两部分组成：

1. **核心（core）**：主要负责**层级化地组织进程**；
2. **控制器（controllers）**：大部分控制器负责 cgroup 层级中 **特定类型的系统资源的分配**，少部分 utility 控制器用于其他目的。

### 1.2.2 进程/线程与 cgroup 关系

所有 cgroup 组成一个**树形结构**（tree structure），

- 系统中的**每个进程都属于且只属于**某一个 cgroup；
- 一个**进程的所有线程**属于同一个 cgroup；
- 创建子进程时，继承其父进程的 cgroup；
- 一个进程可以被**迁移**到其他 cgroup；
- 迁移一个进程时，**子进程（后代进程）不会自动**跟着一起迁移；

### 1.2.3 控制器

- 遵循特定的结构规范（structural constraints），可以选择性地 **针对一个 cgroup 启用或禁用某些控制器**；
- **控制器的所有行为都是 hierarchical 的**。
  - 如果一个 cgroup 启用了某个控制器，那这个 cgroup 的 sub-hierarchy 中所有进程都会受控制。
  - 如果在更接近 root 的节点上设置了资源限制（restrictions set closer to the root），那在下面的 sub-hierarchy 是无法覆盖的。

# 2 基础操作

## 2.1 挂载（mounting）

与 v1 不同，cgroup **v2 只有单个层级树**（single hierarchy）。 用如下命令挂载 v2 hierarchy：

```
# mount -t <fstype> <device> <dir>
$ mount -t cgroup2 none $MOUNT_POINT
```

**cgroupv2 文件系统** 的 magic number 是 `0x63677270` (“cgrp”)。

### 2.1.1 控制器与 v1/v2 绑定关系

- 所有**支持 v2 且未绑定到 v1 的控制器，会被自动绑定到 v2** hierarchy，出现在 root 层级中。
- **v2 中未在使用的控制器**（not in active use），可以绑定到其他 hierarchies。

这说明我们能以完全后向兼容的方式，**混用 v2 和 v1 hierarchy**。

下面通过实际例子理解以上是什么意思。

### 2.1.2 示例：ubuntu 20.04 同时挂载 cgroupv1/cgroupv2（译注）

查看 ubuntu 20.04 (5.11 内核）cgroup 相关的挂载点：

```
$ mount | grep cgroup
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755,inode64)
cgroup2 on /sys/fs/cgroup/unified type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,name=systemd)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/rdma type cgroup (rw,nosuid,nodev,noexec,relatime,rdma)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
```

可以看到，系统**同时挂载了 cgroup 和 cgroup2**：

1. cgroup v2 是单一层级树，因此只有一个挂载点（第二行）**`/sys/fs/cgroup/unified`**，这就是上一小节所说的 **root 层级**。
2. cgroup v1 根据控制器类型（`cpuset/cpu,cpuacct/hugetlb/...`），挂载到不同位置。

接下来看**哪些控制绑定到了 cgroup v2**：

```
$ ls -ahlp /sys/fs/cgroup/unified/
total 0
-r--r--r--   1 root root   0 cgroup.controllers
-rw-r--r--   1 root root   0 cgroup.max.depth
-rw-r--r--   1 root root   0 cgroup.max.descendants
-rw-r--r--   1 root root   0 cgroup.procs
-r--r--r--   1 root root   0 cgroup.stat
-rw-r--r--   1 root root   0 cgroup.subtree_control
-rw-r--r--   1 root root   0 cgroup.threads
-rw-r--r--   1 root root   0 cpu.pressure
-r--r--r--   1 root root   0 cpu.stat
drwxr-xr-x   2 root root   0 init.scope/
-rw-r--r--   1 root root   0 io.pressure
-rw-r--r--   1 root root   0 memory.pressure
drwxr-xr-x 121 root root   0 system.slice/
drwxr-xr-x   3 root root   0 user.slice/
```

只有 cpu/io/memory 等**少量控制器**（大部分还在 cgroup v1 中，系统默认使用 v1）。

最后看几个控制器文件的内容，加深一点直观印象，后面章节会详细解释这些分别表示什么意思：

```
$ cd /sys/fs/cgroup/unified
$ cat cpu.pressure
some avg10=0.00 avg60=0.00 avg300=0.00 total=2501067303

$ cat cpu.stat
usage_usec 44110960000
user_usec 29991256000
system_usec 14119704000

$ cat io.pressure
some avg10=0.00 avg60=0.00 avg300=0.00 total=299044042
full avg10=0.00 avg60=0.00 avg300=0.00 total=271257559

$ cat memory.pressure
some avg10=0.00 avg60=0.00 avg300=0.00 total=298215
full avg10=0.00 avg60=0.00 avg300=0.00 total=229843
```

### 2.1.3 控制器在 v1 和 v2 之间切换

1. 控制器在当前 hierarchy 中已经**不再被引用**（no longer referenced）， 才能移动到其他 hierarchy。
2. 由于 **per-cgroup 控制器状态**是异步销毁的，从 v1 umount 之后 可能会有 linger reference，因此可能不会立即出现在 v2 hierarchy 中。
3. 类似地，一个控制器只有被完全禁用之后，才能被移出 v2 hierarchy，且可能 过一段时间才能在 v1 hierarchy 中可用；
4. 此外，由于控制器间的依赖，其他控制器也可能需要被禁用。

在 v2 和 v1 之间动态移动控制器对开发和手动配置很有用，但 **强烈建议不要在生产环境这么做**。建议在系统启动、控制器开始使用之后， 就不要再修改 hierarchy 和控制器的关联关系了。

另外，迁移到 v2 时，**系统管理软件可能仍然会自动 mount v1 cgroup 文件系统**， 因此需要在**系统启动过程中**劫持所有的控制器，因为启动之后就晚了。 为方便测试，内核提供了 **`cgroup_no_v1=allows`** 配置， 可完全禁用 v1 控制器（强制使用 v2）。

### 2.1.4 cgroupv2 mount 选项

前面 mount 命令没指定任何特殊参数。目前支持如下 mount 选项：

- `nsdelegate`：将 cgroup namespaces **（cgroupns）作为 delegation 边界**。

  系统层选项，只能在 init namespace 通过 mount/unmount 来修改这个配置。在 non-init namespace 中，这个选项会被忽略。详见下面的 [Delegation 小结](https://arthurchiao.art/blog/cgroupv2-zh/#delegation)。

- `memory_localevents`：只为当前 cgroup populate `memory.events`，**不统计任何 subtree**。

  这是 legacy 行为，如果没配置这个参数，默认行为会统计所有的 subtree。

  系统层选项，只能在 init namespace 通过 mount/unmount 来修改这个配置。在 non-init namespace 中，这个选项会被忽略。详见下面的 [Delegation 小结](https://arthurchiao.art/blog/cgroupv2-zh/#delegation)。

- `memory_recursiveprot`

  Recursively apply memory.min and memory.low protection to entire subtrees, without requiring explicit downward propagation into leaf cgroups. This allows protecting entire subtrees from one another, while retaining free competition within those subtrees. This should have been the default behavior but is a mount-option to avoid regressing setups relying on the original semantics (e.g. specifying bogusly high ‘bypass’ protection values at higher tree levels).

## 2.2 组织（organizing）进程和线程

### 2.2.1 进程：创建/删除/移动/查看 cgroup

**初始状态下，只有 root cgroup**，所有进程都属于这个 cgroup。

1. **创建 sub-cgroup**：只需创建一个子目录，

   ```
    $ mkdir $CGROUP_NAME
   ```

   - 一个 cgroup 可以有多个子 cgroup，形成一个树形结构；
   - 每个 cgroup 都有一个**可读写的接口文件 `cgroup.procs`**：
     - 读该文件会列出这个 cgroup 内的所有 PID，每行一个；
     - PID 并未排序；
     - 同一 PID 可能出现多次：进程先移出再移入该 cgroup，或读文件期间 PID 被重用了，都可能发生这种情况。

2. **将进程移动到指定 cgroup**：将 PID 写到相应 cgroup 的 **`cgroup.procs`** 文件即可。

   - 每次 `write(2)` 只能迁移**一个进程**；
   - 如果进程有**多个线程**，那将任意线程的 PID 写到文件，都会将该进程的所有线程迁移到相应 cgroup。
   - 如果进程 fork 出一个子进程，那子进程属于执行 fork 操作时父进程所属的 cgroup。
   - 进程退出（exit）后，**仍然留在退出时它所属的 cgroup**，直到这个进程被收割（reap）；
   - **僵尸进程不会出现在 cgroup.procs 中**，因此**无法对僵尸进程执行 cgroup 迁移操作**。

3. 删除 cgroup/sub-cgroup

   - 如果一个 cgroup 已经没有任何 children 或活进程，那直接 **删除对应的文件夹**就删除该 cgroup 了。
   - 如果一个 cgroup 已经没有 children，虽然其中还有进程但**全是僵尸进程** （zombie processes），那**认为这个 cgroup 是空的**，也可以直接删除。

4. 查看进程的 cgroup 信息：`cat /proc/$PID/cgroup` 会列出该进程的 **cgroup membership**。

   如果系统启用了 v1，这个文件可能会包含多行，**每个 hierarchy 一行**。 **v2 对应的行永远是 `0::$PATH` 格式**：

   ```
    $ cat /proc/$$/cgroup # ubuntu 20.04 上的输出，$$ 是当前 shell 的进程 ID
    12:devices:/user.slice
    11:freezer:/
    10:memory:/user.slice/user-1000.slice/session-1.scope
    9:hugetlb:/
    8:cpuset:/
    7:perf_event:/
    6:rdma:/
    5:pids:/user.slice/user-1000.slice/session-1.scope
    4:cpu,cpuacct:/user.slice
    3:blkio:/user.slice
    2:net_cls,net_prio:/
    1:name=systemd:/user.slice/user-1000.slice/session-1.scope
    0::/user.slice/user-1000.slice/session-1.scope
   ```

   如果一个进程变成**僵尸进程**（zombie），并且与它关联的 **cgroup 随后被删掉了**，那行尾会出现 **`(deleted)`** 字样：

   ```
    $ cat /proc/842/cgroup
    ...
    0::/test-cgroup/test-cgroup-nested (deleted)
   ```

### 2.2.2 线程

- cgroup v2 的**一部分控制器**支持线程粒度的资源控制， 这种控制器称为 **threaded controllers**。
  - 默认情况下，一个进程的所有线程属于同一个 cgroup，
  - 线程模型使我们能将不同线程放到 subtree 的不同位置，而同时还能保持二者在同一 资源域（resource domain）内。
- 不支持线程模式的控制器称为 **domain controllers**。

将一个 cgroup 标记为 threaded，那它将作为 threaded cgroup 将加入 parent 的资源域 。而 parent 可能也是一个 threaded cgroup，它所属的资源域在 hierarchy 层级中的更 上面。一个 threaded subtree 的 root，即第一个不是 threaded 的祖先，称为 threaded domain 或 threaded root，作为整个 subtree 的资源域。

Inside a threaded subtree, threads of a process can be put in different cgroups and are not subject to the no internal process constraint - threaded controllers can be enabled on non-leaf cgroups whether they have threads in them or not.

As the threaded domain cgroup hosts all the domain resource consumptions of the subtree, it is considered to have internal resource consumptions whether there are processes in it or not and can’t have populated child cgroups which aren’t threaded. Because the root cgroup is not subject to no internal process constraint, it can serve both as a threaded domain and a parent to domain cgroups.

The current operation mode or type of the cgroup is shown in the “cgroup.type” file which indicates whether the cgroup is a normal domain, a domain which is serving as the domain of a threaded subtree, or a threaded cgroup.

#### 将 cgroup 改成 theaded 模式（单向/不可逆操作）

cgroup 创建之后都是 domain cgroup，可以通过下面的命令将其**改成 threaded 模式**：

```
$ echo threaded > cgroup.type
```

但注意：**这个操作是单向的**，一旦设置成 threaded 模式之后，就无法再切回 domain 模式了。

开启 thread 模型必须先满足如下条件：

1. As the cgroup will join the parent’s resource domain. The parent must either be a valid (threaded) domain or a threaded cgroup.
2. When the parent is an unthreaded domain, it must not have any domain controllers enabled or populated domain children. The root is exempt from this requirement.

Topology-wise, a cgroup can be in an invalid state. Please consider the following topology:

```
  A (threaded domain) - B (threaded) - C (domain, just created)
```

C is created as a domain but isn’t connected to a parent which can host child domains. C can’t be used until it is turned into a threaded cgroup. “cgroup.type” file will report “domain (invalid)” in these cases. Operations which fail due to invalid topology use EOPNOTSUPP as the errno.

A domain cgroup is turned into a threaded domain when one of its child cgroup becomes threaded or threaded controllers are enabled in the “cgroup.subtree_control” file while there are processes in the cgroup. A threaded domain reverts to a normal domain when the conditions clear.

When read, “cgroup.threads” contains the list of the thread IDs of all threads in the cgroup. Except that the operations are per-thread instead of per-process, “cgroup.threads” has the same format and behaves the same way as “cgroup.procs”. While “cgroup.threads” can be written to in any cgroup, as it can only move threads inside the same threaded domain, its operations are confined inside each threaded subtree.

The threaded domain cgroup serves as the resource domain for the whole subtree, and, while the threads can be scattered across the subtree, all the processes are considered to be in the threaded domain cgroup. “cgroup.procs” in a threaded domain cgroup contains the PIDs of all processes in the subtree and is not readable in the subtree proper. However, “cgroup.procs” can be written to from anywhere in the subtree to migrate all threads of the matching process to the cgroup.

Only threaded controllers can be enabled in a threaded subtree. When a threaded controller is enabled inside a threaded subtree, it only accounts for and controls resource consumptions associated with the threads in the cgroup and its descendants. All consumptions which aren’t tied to a specific thread belong to the threaded domain cgroup.

Because a threaded subtree is exempt from no internal process constraint, a threaded controller must be able to handle competition between threads in a non-leaf cgroup and its child cgroups. Each threaded controller defines how such competitions are handled.

## 2.3 [Un]populated Notification（进程退出通知）

每个 non-root cgroup 都有一个 **`cgroup.events` 文件**， 其中包含了 **`populated`** 字段，描述这个 cgroup 的 sub-hierarchy 中**是否存在活进程**（live processes）。

- 如果值是 0，表示 cgroup 及其 sub-cgroup 中没有活进程；
- 如果值是 1：那这个值变为 0 时，会触发 poll 和 [id]notify 事件。

这可以用来，例如，在一个 sub-hierarchy 内的**所有进程退出之后触发执行清理操作**。

The populated 状态更新和通知是递归的。以下图为例，括号中的数字表示该 cgroup 中的进程数量：

```
  A(4) - B(0) - C(1)
              \ D(0)
```

- A、B 和 C 的 `populated` 字段都应该是 `1`，而 D 的是 `0`。
- 当 C 中唯一的进程退出之后，B 和 C 的 `populated` 字段将变成 `0`，将 **在这两个 cgroup 内触发一次 cgroup.events 文件的文件修改事件**。

## 2.3 管理控制器（controlling controllers）

### 2.3.1 启用和禁用

每个 cgroup 都有一个 **`cgroup.controllers`** 文件， 其中列出了这个 cgroup 可用的所有控制器：

```
$ cat cgroup.controllers
cpu io memory
```

**默认没有启用任何控制**。启用或禁用是通过写 **`cgroup.subtree_control`** 文件完成的：

```
$ echo "+cpu +memory -io" > cgroup.subtree_control
```

只有**出现在 cgroup.controllers 中**的控制器**才能被启用**。

- 如果像上面的命令一样，一次指定多个操作，那它们要么全部功能，要么全部失败；
- 如果对同一个控制器指定了多个操作，最后一个是有效的。

启用 cgroup 的某个控制器，意味着控制它在子节点之间分配目标资源（target resource）的行为。 考虑下面的 sub-hierarchy，括号中是已经启用的控制器：

```
  A(cpu,memory) - B(memory) - C()
                            \ D()
```

- A 启用了 `cpu` 和 `memory`，因此会控制它的 child（即 B）的 CPU 和 memory 使用；
- B 只启用了 `memory`，因此 C 和 D 的 memory 使用量会受 B 控制，但 **CPU 可以随意竞争**（compete freely）。

控制器限制 children 的资源使用方式，是**创建或写入 children cgroup 的接口文件**。 还是以上面的拓扑为例：

- 在 B 上启用 `cpu` 将会在 C 和 D 的 cgroup 目录中创建 `cpu.` 开头的接口文件；
- 同理，禁用 `memory` 时会删除对应的 `memory.` 开头的文件。

这也意味着**cgroup 目录中所有不以 `cgroup.`开头的** 控制器接口文件 —— 在管理上 **都属于 parent cgroup 而非当前 cgroup 自己**。

### 2.3.2 自顶向下启用（top-down constraint）

资源是自顶向下（top-down）分配的，只有当一个 cgroup 从 parent 获得了某种资源，它 才可以继续向下分发。这意味着

- 只有父节点启用了某个控制器，子节点才能启用；
- 对应到实现上，**所有非根节点**（non-root）的 `cgroup.subtree_control` 文件中， 只能包含它的父节点的 `cgroup.subtree_control` 中有的控制器；
- 另一方面，只要有子节点还在使用某个控制器，父节点就无法禁用之。

### 2.3.3 将资源分给 children 时，parent cgroup 内不能有进程（no internal process）

只有当一个 non-root cgroup 中**没有任何进程时**，才能将其 domain resource 分配给它的 children。换句话说，只有那些没有任何进程的 domain cgroup， 才能**将它们的 domain controllers 写到其 children 的 `cgroup.subtree_control` 文件中**。

这种方式保证了在给定的 domain controller 范围内，**所有进程都位于叶子节点上**， 因而**避免了 child cgroup 内的进程与 parent 内的进程竞争**的情况，便于 domain controller 扫描 hierarchy。

但 **root cgroup 不受此限制**。

- 对大部分类型的控制器来说，root 中包含了一些**没有与任何 cgroup 相关联的进程和匿名资源占用** （anonymous resource consumption），需要特殊对待。
- root cgroup 的资源占用是如何管理的，**因控制器而异**（更多信息可参考 Controllers 小结）。

注意，在 parent 的 `cgroup.subtree_control` 启用控制器之前，这些限制不会生效。 这非常重要，因为它决定了创建 populated cgroup children 的方式。 **要控制一个 cgroup 的资源分配**，这个 cgroup 需要：

1. 创建 children cgroup，
2. 将自己所有的进程转移到 children cgroup 中，
3. 在它自己的 `cgroup.subtree_control` 中启用控制器。



## 2.4 Delegation（委派）

### 2.4.1 Model of Delegation

cgroup 能以两种方式 delegate。

1. 通过授予该目录以及目录中的 `cgroup.procs`、`cgroup.threads`、`cgroup.subtree_control` 文件的写权限， 将 cgroup delegate 给一个 less privileged 用户；
2. 如果配置了 `nsdelegate` 挂载选项，会在创建 cgroup 时自动 delegate。

对于一个给定的目录，由于其中的 resource control 接口文件控制着 parent 的资源的分配， 因此 delegatee 不应该被授予写权限。

1. For the first method, this is achieved by not granting access to these files.
2. 对第二种方式，内核会拒绝除了在该 namespace 内对 `cgroup.procs`、`cgroup.subtree_control` 之外的对其他文件的写操作。

The end results are equivalent for both delegation types. Once delegated, the user can build sub-hierarchy under the directory, organize processes inside it as it sees fit and further distribute the resources it received from the parent. The limits and other settings of all resource controllers are hierarchical and regardless of what happens in the delegated sub-hierarchy, nothing can escape the resource restrictions imposed by the parent.

目前，cgroup 并未对 delegated sub-hierarchy 的 cgroup 数量或嵌套深度施加限制；但未来可能会施加显式限制。

### 2.4.2 Delegation Containment

A delegated sub-hierarchy is contained in the sense that processes can’t be moved into or out of the sub-hierarchy by the delegatee.

For delegations to a less privileged user, this is achieved by requiring the following conditions for a process with a non-root euid to migrate a target process into a cgroup by writing its PID to the “cgroup.procs” file.

- The writer must have write access to the “cgroup.procs” file.
- The writer must have write access to the “cgroup.procs” file of the common ancestor of the source and destination cgroups.

The above two constraints ensure that while a delegatee may migrate processes around freely in the delegated sub-hierarchy it can’t pull in from or push out to outside the sub-hierarchy.

For an example, let’s assume cgroups C0 and C1 have been delegated to user U0 who created C00, C01 under C0 and C10 under C1 as follows and all processes under C0 and C1 belong to U0::

```
~~~~~~~~~~~~~ - C0 - C00
~ cgroup    ~      \ C01
~ hierarchy ~
~~~~~~~~~~~~~ - C1 - C10
```

Let’s also say U0 wants to write the PID of a process which is currently in C10 into “C00/cgroup.procs”. U0 has write access to the file; however, the common ancestor of the source cgroup C10 and the destination cgroup C00 is above the points of delegation and U0 would not have write access to its “cgroup.procs” files and thus the write will be denied with -EACCES.

For delegations to namespaces, containment is achieved by requiring that both the source and destination cgroups are reachable from the namespace of the process which is attempting the migration. If either is not reachable, the migration is rejected with -ENOENT.

## 2.5 指导原则

### 2.5.1 避免频繁在 cgroup 之间迁移进程（Organize once and control）

原则：创建进程前，先想好应该放在哪个 cgroup；进程启动后，通过 controller 接口文件进行控制。

在 cgroup 之间迁移进程是一个**开销相对较高**的操作，而且 **有状态资源（例如 memory）**是**不会随着进程一起迁移**的。 这种行为是有意设计的，因为 there often exist inherent trade-offs between migration and various hot paths in terms of synchronization cost.

因此，**不建议为了达到某种资源限制目的而频繁地在 cgroup 之间迁移进程**。 一个进程启动时，就应该根据系统的逻辑和资源结构分配到合适的 cgroup。 动态调整资源分配可以通过修改接口文件来调整 controller 配置。

### 2.5.2 避免文件名冲突（Avoid Name Collisions）

cgroup 自己的接口文件和它的 children cgroup 的接口文件**位于同一目录中**， 因此创建 children cgroup 时有可能与 cgroup 自己的接口文件冲突。

- 所有 **cgroup 核心接口文件**都是以 `cgroup.` 开头，并且不会以常用的 job/service/slice/unit/workload 等作为开头或结尾。
- 每个控制器的接口文件都以 `<controller name>.` 开头，其中 `<controller> name` 由小写字母和下划线组成，但不会以 `_` 开头。

因此为避免冲突，可以用 `_` 作为前缀。

**cgroup 没有任何文件名冲突检测机制**，因此避免文件冲突是用户自己的责任。

# 3 资源分配模型（Resource distribution models）

根据资源类型（resource type）与使用场景的不同，cgroup 控制器实现了机制不同的资源 分发方式。本节介绍主要的几种机制及其行为。

## 3.1 Weights（资源量权重）

这种模型的一个**例子**是 `cpu.weight`，负责在 active children 之间 按比例分配 CPU cycle 资源。

这种模型中，parent 会**根据所有 active children 的权重来计算它们各自的占比** （ratio）。

- 由于只有那些能使用这些资源的 children 会参与到资源分配，因此这种模型 **能实现资源的充分利用**（work-conserving）。
- 这种分配模型**本质上是动态的**（the dynamic nature）, 因此常用于**无状态资源**。
- 权重值范围是 **`[1, 10000]`**，默认 `100`。这使得能以 足够细的粒度增大或缩小权重（以 100 为中心，`100/100 = 1`，`100*100 = 10000`）。

## 3.2 Limits（资源量上限，可超分）

这种模型的一个**例子**是 `io.max`，负责在 IO device 上限制 cgroup 的最大 BPS 或 IOPS。

- 这种模型给 child 配置的资源使用量上限（limit）。
- **资源是可以超分的**（over-committed），即所有 children 的份额加起来可以大于 parent 的总可用量。
- Limits 值范围是 **`[0, max]`**，默认 `max`，也就是没做限制。
- 由于 limits 是可以超分的，因此所有配置组合都是合法的。

## 3.3 Protections（资源量保护，可超分）

这种模型的一个**例子**是 `memory.low`，实现了 **best-effort 内存保护**。

- 在这种模型中，只要一个 cgroup 的所有祖先都处于各自的 protected level 以下，那 这个 cgroup 拿到的资源量就能达到配置值（有保障）。这里的保障可以是
  - hard guarantees
  - best effort soft boundaries
- Protection 可以超分，在这种情况下，only upto the amount available to the parent is protected among children.
- Protection 值范围是 `[0, max]`，默认是 `0`，也就是没有特别限制。
- 由于 protections 是可以超分的，因此所有配置组合都是合法的。

## 3.4 Allocations（独占资源量，不可超分）

这种模型的一个**例子**是 `cpu.rt.max`，它 hard-allocates realtime slices。

- 这种模型中，cgroup 会**排他性地分配**（exclusively allocated）资源量。
- Allocation **不可超分**，即所有 children 的 allocations 之和不能超过 parent 的可用资源量。
- Allocation 值范围是 `[0, max]`，默认是 `0`，也就是不会排他性地分配资源。
- 由于 allocation 不可超分，因此某些配置可能不合法，会被拒绝；如果强制迁移进程，可能会因配置不合法（资源达到上限）而失败。

# 4 接口文件（Interface Files）

## 4.1 文件格式

所有接口文件都应该属于以下某种类型：

1. **换行符分隔的值**（每次 write 操作只允许写入一个值）

   ```
    VAL0\n
    VAL1\n
    ...
   ```

2. 空格分隔的值（只读场景，或一次可写入多个值的场景）

   ```
    VAL0 VAL1 ...\n
   ```

3. **扁平 key 类型** （flat keyed，每行一个 key value 对）

   ```
    KEY0 VAL0\n
    KEY1 VAL1\n
    ...
   ```

4. 嵌套 key 类型（nested keyed，每行一个 Key value 对，其中 value 中又包含 subkey/subvalue）

   ```
    KEY0 SUB_KEY0=VAL00 SUB_KEY1=VAL01...
    KEY1 SUB_KEY0=VAL10 SUB_KEY1=VAL11...
    ...
   ```

对于可写文件（writable file），通常来说写的格式应与读的格式保持一致； 但对于大部分常用场景，控制器可能会允许省略后面的字段（later fields），或实现了受 限的快捷方式（restricted shortcuts）。

对于 flat 和 nested key 文件来说，每次只能写一个 key （及对于的 values）。 对于 nested keyed files，sub key pairs 的顺序可以随意，也不必每次都指定所有 pairs。

## 4.2 一些惯例（conventions）

1. 每个特性的配置应该放到单独文件。

2. **root cgroup 不受资源控制的限制**，因此不应该有资源控制**接口文件**（`cgroup.*`）。

3. **默认的时间单位是微秒 `us`**（microseconds）。如果改 用其他时间单位，必须显式加上一个单位后缀。

4. 表示各部分占比时，应该用十进制百分比表示，且小数点后保留至少两位，例如 `13.40`。

5. 如果一个控制器实现了 weight 模型，那接口文件应命名为 `weight`，值范围 `[1, 10000]`，默认 100。

6. 如果一个控制器实现了绝对 resource guarantee and/or limit，则接口文件应命名为 `min` 和 `max`。如果实现了 best effort resource guarantee and/or limit，应命名为 `low` 和 `high`。对于这四种控制文件，`"max"` 是一个**特殊的合法值**（special token）， 表示**读和写无上限**（upward infinity）。

7. 如果一个配置项的默认值可配置，且有 keyed specific overrides，那默认 default entry 的 key 应该是 `"default"`，并出现在这个文件的第一行。

   更新/覆盖默认值：将 `default $VAL` 或 `$VAL` 写入文件。单纯写入 `default` 恢复默认配置。

   例如，下面的配置项以 `major:minor` 设备号为 key，整数为 value：

   ```
    # cat cgroup1.example-interface-file
    default 150
    8:0 300
   ```

   可用如下方式**更新默认值**：

   ```
    # 方式一
    $ echo 125 > cgroup-example-interface-file
    # 方式二
    $ echo "default 125" > cgroup-example-interface-file
   ```

   用自定义值覆盖默认值：

   ```
    $ echo "8:16 170" > cgroup-example-interface-file
   ```

   清除配置：

   ```
    $ echo "8:0 default" > cgroup-example-interface-file
    $ cat cgroup-example-interface-file
    default 125
    8:16 170
   ```

8. 对于不是太频繁的 events，应该创建一个接口文件 `"events"`，读取这个文件能 list event key value pairs。当发生任何 notifiable event 时，这个文件上都应该生成一个 file modified event。

## 4.3 核心接口文件（core interface files）

所有的 cgroup 核心文件都以 `cgroup.` 开头。

1. **`cgroup.type`**

   可读写文件，只能**位于 non-root cgroup 中**。类型可以是：

   1. “domain”：正常的 domain cgroup。
   2. “domain threaded”：threaded domain cgroup，作为 threaded subtree 的 root。
   3. “domain invalid”：该 cgroup 当前处于 invalid 状态。这种状态下无法被 populate，或者启用控制器。可能能变成一个 threaded cgroup。
   4. “threaded” : 表示当前 cgroup 是某个 threaded subtree 的一个 member。

   可以将一个 cgroup 设置成 threaded group，只需将字符串 `"threaded"` 写入这个文件。

2. **`cgroup.procs`**

   可读写文件，每行一个 PID，可用于所有 cgroups。

   读时，返回这个 cgroup 内的所有进程 ID，每行一个。PID 列表没有排序，同一个 PID 可能会出现多次 —— 如果该进程先移除再移入该 cgroup，或 PID 循环利用了， 都可以回出现这种情况。

   要将一个进程移动到该 cgroup，只需将 PID 写入这个文件。写入时必须满足：

   1. 必须有对改 cgroup 的 cgroup.procs 文件写权限。
   2. 必须对 source and destination cgroups 的**共同祖先**的 cgroup.procs 文件有写权限。

   When delegating a sub-hierarchy, write access to this file should be granted along with the containing directory.

   In a threaded cgroup, reading this file fails with EOPNOTSUPP as all the processes belong to the thread root. Writing is supported and moves every thread of the process to the cgroup.

3. **`cgroup.threads`**

   A read-write new-line separated values file which exists on all cgroups.

   When read, it lists the TIDs of all threads which belong to the cgroup one-per-line. The TIDs are not ordered and the same TID may show up more than once if the thread got moved to another cgroup and then back or the TID got recycled while reading.

   A TID can be written to migrate the thread associated with the TID to the cgroup. The writer should match all of the following conditions.

   - It must have write access to the “cgroup.threads” file.
   - The cgroup that the thread is currently in must be in the same resource domain as the destination cgroup.
   - It must have write access to the “cgroup.procs” file of the common ancestor of the source and destination cgroups.

   When delegating a sub-hierarchy, write access to this file should be granted along with the containing directory.

4. **`cgroup.controllers`**

   **只读**（read-only）文件，内容是空格隔开的值，可用于所有 cgroups。

   读取这个文件，得到的是该 cgroup 的所有可用控制器，空格隔开。控制器列表未排序。

5. **`cgroup.subtree_control`**

   可读写，空格隔开的值，可用于所有控制器，初始时是空的。

   读取时，返回这个 cgroup 已经启用的控制器，对其 children 做资源控制。

   可通过 `+<controller>` 和 `-<controller>` 来启用或禁用控制器。如果一个控制器在文件中出现多次，最后一次有效。 如果一次操作中指定了启用或禁用多个动作，那要么全部成功，要么全部失败。

6. **`cgroup.events`**

   只读，flat-keyed file，只可用于 non-root cgroups。

   定义了下面两个配置项：

   - populated：1 if the cgroup or its descendants contains any live processes; otherwise, 0.
   - frozen：1 if the cgroup is frozen; otherwise, 0.

   除非有特别设置，否则修改本文件会触发一次 file modified event.

7. **`cgroup.max.descendants`**

   可读写 single value files，默认值 `"max"`。

   允许的最大 descent cgroups 数量。如果实际的 descendants 数量等于或大于该值，在 hierarchy 中再创建新 cgroup 时会失败。

8. **`cgroup.max.depth`**

   可读写 single value files，默认值 `"max"`。

   当前 cgroup 内允许的最大 descent depth。如果实际的 depth 数量等于或大于该值，再创建新 child cgroup 时会失败。

9. **`cgroup.stat`**

   只读 flat-keyed file，定义了下列 entries:

   - nr_descendants：可见的 descendant cgroups 总数。

   - nr_dying_descendants

     Total number of dying descendant cgroups. A cgroup becomes dying after being deleted by a user. The cgroup will remain in dying state for some time undefined time (which can depend on system load) before being completely destroyed.

     A process can’t enter a dying cgroup under any circumstances, a dying cgroup can’t revive.

     A dying cgroup can consume system resources not exceeding limits, which were active at the moment of cgroup deletion.

10. **`cgroup.freeze`**

    可读写 single value file，只能用于 non-root cgroups。 Allowed values are “0” and “1”. The default is “0”.

    Writing “1” to the file causes freezing of the cgroup and all descendant cgroups. This means that all belonging processes will be stopped and will not run until the cgroup will be explicitly unfrozen. Freezing of the cgroup may take some time; when this action is completed, the “frozen” value in the cgroup.events control file will be updated to “1” and the corresponding notification will be issued.

    A cgroup can be frozen either by its own settings, or by settings of any ancestor cgroups. If any of ancestor cgroups is frozen, the cgroup will remain frozen.

    Processes in the frozen cgroup can be killed by a fatal signal. They also can enter and leave a frozen cgroup: either by an explicit move by a user, or if freezing of the cgroup races with fork(). If a process is moved to a frozen cgroup, it stops. If a process is moved out of a frozen cgroup, it becomes running.

    Frozen status of a cgroup doesn’t affect any cgroup tree operations: it’s possible to delete a frozen (and empty) cgroup, as well as create new sub-cgroups.

# 5 Controllers（控制器）

## 5.1 CPU

The “cpu” controllers 控制着 CPU cycle 的分配。这个控制器实现了

- **常规调度**策略：weight and absolute bandwidth limit 模型
- **实时调度**策略：absolute bandwidth allocation 模型

在所有以上模型中，cycles distribution 只定义在 temporal base 上，it does not account for the frequency at which tasks are executed. The (optional) utilization clamping support allows to hint the schedutil cpufreq governor about the minimum desired frequency which should always be provided by a CPU, as well as the maximum desired frequency, which should not be exceeded by a CPU.

警告：cgroupv2 还**不支持对实时进程的控制**，并且只有当所有实时进程**都位于 root cgroup 时**， cpu 控制器才能启用。需要注意：一些系统管理软件可能已经在系统启动期间，将实时进程放到了 non-root cgroup 中， 因此在启用 CPU 控制器之前，需要将这些进程移动到 root cgroup。

### CPU Interface Files

所有时间单位都是 microseconds。

1. **`cpu.stat`**

   A read-only flat-keyed file. This file exists whether the controller is enabled or not.

   It always reports the following three stats:

   - usage_usec
   - user_usec
   - system_usec

   and the following three when the controller is enabled:

   - nr_periods
   - nr_throttled
   - throttled_usec

2. **`cpu.weight`**

   A read-write single value file which exists on non-root cgroups. The default is “100”.

   The weight in the range [1, 10000].

3. **`cpu.weight.nice`**

   A read-write single value file which exists on non-root cgroups. The default is “0”.

   The nice value is in the range [-20, 19].

   This interface file is an alternative interface for “cpu.weight” and allows reading and setting weight using the same values used by nice(2). Because the range is smaller and granularity is coarser for the nice values, the read value is the closest approximation of the current weight.

4. **`cpu.max`**

   A read-write two value file which exists on non-root cgroups. The default is “max 100000”.

   The maximum bandwidth limit. It’s in the following format::

   $MAX $PERIOD

   which indicates that the group may consume upto $MAX in each $PERIOD duration. “max” for $MAX indicates no limit. If only one number is written, $MAX is updated.

5. **`cpu.pressure`**

   A read-only nested-key file which exists on non-root cgroups.

   Shows pressure stall information for CPU. See `Documentation/accounting/psi.rst <psi>` for details.

6. **`cpu.uclamp.min`**

   A read-write single value file which exists on non-root cgroups. The default is “0”, i.e. no utilization boosting.

   The requested minimum utilization (protection) as a percentage rational number, e.g. 12.34 for 12.34%.

   This interface allows reading and setting minimum utilization clamp values similar to the sched_setattr(2). This minimum utilization value is used to clamp the task specific minimum utilization clamp.

   The requested minimum utilization (protection) is always capped by the current value for the maximum utilization (limit), i.e. `cpu.uclamp.max`.

7. **`cpu.uclamp.max`**

   A read-write single value file which exists on non-root cgroups. The default is “max”. i.e. no utilization capping

   The requested maximum utilization (limit) as a percentage rational number, e.g. 98.76 for 98.76%.

   This interface allows reading and setting maximum utilization clamp values similar to the sched_setattr(2). This maximum utilization value is used to clamp the task specific maximum utilization clamp.

## 5.2 Memory

The “memory” controller regulates distribution of memory. 内存是**有状态的**，实现了 limit 和 protection 两种模型。 Due to the intertwining between memory usage and reclaim pressure and the stateful nature of memory, the distribution model is relatively complex.

While not completely water-tight, 给定 cgroup 的所有主要 memory usages 都会跟踪，因此总内存占用可以控制在一个合理的范围内。目前**下列类型的内存**使用会被跟踪：

1. **Userland memory** - page cache and anonymous memory.
2. **Kernel data structures** such as dentries and inodes.
3. **TCP socket buffers**.

The above list may expand in the future for better coverage.

### Memory Interface Files

All memory amounts are in bytes. If a value which is not aligned to PAGE_SIZE is written, the value may be rounded up to the closest PAGE_SIZE multiple when read back.

1. memory.current

   A read-only single value file which exists on non-root cgroups.

   The total amount of memory currently being used by the cgroup and its descendants.

2. memory.min

   A read-write single value file which exists on non-root cgroups. The default is “0”.

   Hard memory protection. If the memory usage of a cgroup is within its effective min boundary, the cgroup’s memory won’t be reclaimed under any conditions. If there is no unprotected reclaimable memory available, OOM killer is invoked. Above the effective min boundary (or effective low boundary if it is higher), pages are reclaimed proportionally to the overage, reducing reclaim pressure for smaller overages.

   Effective min boundary is limited by memory.min values of all ancestor cgroups. If there is memory.min overcommitment (child cgroup or cgroups are requiring more protected memory than parent will allow), then each child cgroup will get the part of parent’s protection proportional to its actual memory usage below memory.min.

   Putting more memory than generally available under this protection is discouraged and may lead to constant OOMs.

   If a memory cgroup is not populated with processes, its memory.min is ignored.

3. memory.low

   A read-write single value file which exists on non-root cgroups. The default is “0”.

   Best-effort memory protection. If the memory usage of a cgroup is within its effective low boundary, the cgroup’s memory won’t be reclaimed unless there is no reclaimable memory available in unprotected cgroups. Above the effective low boundary (or effective min boundary if it is higher), pages are reclaimed proportionally to the overage, reducing reclaim pressure for smaller overages.

   Effective low boundary is limited by memory.low values of all ancestor cgroups. If there is memory.low overcommitment (child cgroup or cgroups are requiring more protected memory than parent will allow), then each child cgroup will get the part of parent’s protection proportional to its actual memory usage below memory.low.

   Putting more memory than generally available under this protection is discouraged.

4. memory.high

   A read-write single value file which exists on non-root cgroups. The default is “max”.

   Memory usage throttle limit. This is the main mechanism to control memory usage of a cgroup. If a cgroup’s usage goes over the high boundary, the processes of the cgroup are throttled and put under heavy reclaim pressure.

   Going over the high limit never invokes the OOM killer and under extreme conditions the limit may be breached.

5. memory.max

   A read-write single value file which exists on non-root cgroups. The default is “max”.

   Memory usage hard limit. This is the final protection mechanism. If a cgroup’s memory usage reaches this limit and can’t be reduced, the OOM killer is invoked in the cgroup. Under certain circumstances, the usage may go over the limit temporarily.

   In default configuration regular 0-order allocations always succeed unless OOM killer chooses current task as a victim.

   Some kinds of allocations don’t invoke the OOM killer. Caller could retry them differently, return into userspace as -ENOMEM or silently ignore in cases like disk readahead.

   This is the ultimate protection mechanism. As long as the high limit is used and monitored properly, this limit’s utility is limited to providing the final safety net.

6. memory.oom.group

   A read-write single value file which exists on non-root cgroups. The default value is “0”.

   Determines whether the cgroup should be treated as an indivisible workload by the OOM killer. If set, all tasks belonging to the cgroup or to its descendants (if the memory cgroup is not a leaf cgroup) are killed together or not at all. This can be used to avoid partial kills to guarantee workload integrity.

   Tasks with the OOM protection (oom_score_adj set to -1000) are treated as an exception and are never killed.

   If the OOM killer is invoked in a cgroup, it’s not going to kill any tasks outside of this cgroup, regardless memory.oom.group values of ancestor cgroups.

7. memory.events

   A read-only flat-keyed file which exists on non-root cgroups. The following entries are defined. Unless specified otherwise, a value change in this file generates a file modified event.

   Note that all fields in this file are hierarchical and the file modified event can be generated due to an event down the hierarchy. For for the local events at the cgroup level see memory.events.local.

   ```
      low
        The number of times the cgroup is reclaimed due to
        high memory pressure even though its usage is under
        the low boundary.  This usually indicates that the low
        boundary is over-committed.
   
      high
        The number of times processes of the cgroup are
        throttled and routed to perform direct memory reclaim
        because the high memory boundary was exceeded.  For a
        cgroup whose memory usage is capped by the high limit
        rather than global memory pressure, this event's
        occurrences are expected.
   
      max
        The number of times the cgroup's memory usage was
        about to go over the max boundary.  If direct reclaim
        fails to bring it down, the cgroup goes to OOM state.
   
      oom
        The number of time the cgroup's memory usage was
        reached the limit and allocation was about to fail.
   ```

   ```
   This event is not raised if the OOM killer is not considered as an
   option, e.g. for failed high-order allocations or if caller asked to not
   retry attempts.
   ```

   ```
      oom_kill
        The number of processes belonging to this cgroup
        killed by any kind of OOM killer.
   ```

8. memory.events.local

   Similar to memory.events but the fields in the file are local to the cgroup i.e. not hierarchical. The file modified event generated on this file reflects only the local events.

9. memory.stat

   A read-only flat-keyed file which exists on non-root cgroups.

   This breaks down the cgroup’s memory footprint into different types of memory, type-specific details, and other information on the state and past events of the memory management system.

   All memory amounts are in bytes.

   The entries are ordered to be human readable, and new entries can show up in the middle. Don’t rely on items remaining in a fixed position; use the keys to look up specific values!

   If the entry has no per-node counter(or not show in the mempry.numa_stat). We use ‘npn’(non-per-node) as the tag to indicate that it will not show in the mempry.numa_stat.

   ```
      anon
        Amount of memory used in anonymous mappings such as
        brk(), sbrk(), and mmap(MAP_ANONYMOUS)
   
      file
        Amount of memory used to cache filesystem data,
        including tmpfs and shared memory.
   
      kernel_stack
        Amount of memory allocated to kernel stacks.
   
      percpu(npn)
        Amount of memory used for storing per-cpu kernel
        data structures.
   
      sock(npn)
        Amount of memory used in network transmission buffers
   
      shmem
        Amount of cached filesystem data that is swap-backed,
        such as tmpfs, shm segments, shared anonymous mmap()s
   
      file_mapped
        Amount of cached filesystem data mapped with mmap()
   
      file_dirty
        Amount of cached filesystem data that was modified but
        not yet written back to disk
   
      file_writeback
        Amount of cached filesystem data that was modified and
        is currently being written back to disk
   
      anon_thp
        Amount of memory used in anonymous mappings backed by
        transparent hugepages
   
      inactive_anon, active_anon, inactive_file, active_file, unevictable
        Amount of memory, swap-backed and filesystem-backed,
        on the internal memory management lists used by the
        page reclaim algorithm.
   
        As these represent internal list state (eg. shmem pages are on anon
        memory management lists), inactive_foo + active_foo may not be equal to
        the value for the foo counter, since the foo counter is type-based, not
        list-based.
   
      slab_reclaimable
        Part of "slab" that might be reclaimed, such as
        dentries and inodes.
   
      slab_unreclaimable
        Part of "slab" that cannot be reclaimed on memory
        pressure.
   
      slab(npn)
        Amount of memory used for storing in-kernel data
        structures.
   
      workingset_refault_anon
        Number of refaults of previously evicted anonymous pages.
   
      workingset_refault_file
        Number of refaults of previously evicted file pages.
   
      workingset_activate_anon
        Number of refaulted anonymous pages that were immediately
        activated.
   
      workingset_activate_file
        Number of refaulted file pages that were immediately activated.
   
      workingset_restore_anon
        Number of restored anonymous pages which have been detected as
        an active workingset before they got reclaimed.
   
      workingset_restore_file
        Number of restored file pages which have been detected as an
        active workingset before they got reclaimed.
   
      workingset_nodereclaim
        Number of times a shadow node has been reclaimed
   
      pgfault(npn)
        Total number of page faults incurred
   
      pgmajfault(npn)
        Number of major page faults incurred
   
      pgrefill(npn)
        Amount of scanned pages (in an active LRU list)
   
      pgscan(npn)
        Amount of scanned pages (in an inactive LRU list)
   
      pgsteal(npn)
        Amount of reclaimed pages
   
      pgactivate(npn)
        Amount of pages moved to the active LRU list
   
      pgdeactivate(npn)
        Amount of pages moved to the inactive LRU list
   
      pglazyfree(npn)
        Amount of pages postponed to be freed under memory pressure
   
      pglazyfreed(npn)
        Amount of reclaimed lazyfree pages
   
      thp_fault_alloc(npn)
        Number of transparent hugepages which were allocated to satisfy
        a page fault. This counter is not present when CONFIG_TRANSPARENT_HUGEPAGE
                is not set.
   
      thp_collapse_alloc(npn)
        Number of transparent hugepages which were allocated to allow
        collapsing an existing range of pages. This counter is not
        present when CONFIG_TRANSPARENT_HUGEPAGE is not set.
   ```

10. memory.numa_stat

    A read-only nested-keyed file which exists on non-root cgroups.

    This breaks down the cgroup’s memory footprint into different types of memory, type-specific details, and other information per node on the state of the memory management system.

    This is useful for providing visibility into the NUMA locality information within an memcg since the pages are allowed to be allocated from any physical node. One of the use case is evaluating application performance by combining this information with the application’s CPU allocation.

    All memory amounts are in bytes.

    The output format of memory.numa_stat is::

    ```
       type N0=<bytes in node 0> N1=<bytes in node 1> ...
    ```

    The entries are ordered to be human readable, and new entries can show up in the middle. Don’t rely on items remaining in a fixed position; use the keys to look up specific values!

    The entries can refer to the memory.stat.

11. memory.swap.current

    A read-only single value file which exists on non-root cgroups.

    The total amount of swap currently being used by the cgroup and its descendants.

12. memory.swap.high

    A read-write single value file which exists on non-root cgroups. The default is “max”.

    Swap usage throttle limit. If a cgroup’s swap usage exceeds this limit, all its further allocations will be throttled to allow userspace to implement custom out-of-memory procedures.

    This limit marks a point of no return for the cgroup. It is NOT designed to manage the amount of swapping a workload does during regular operation. Compare to memory.swap.max, which prohibits swapping past a set amount, but lets the cgroup continue unimpeded as long as other memory can be reclaimed.

    Healthy workloads are not expected to reach this limit.

13. memory.swap.max

    A read-write single value file which exists on non-root cgroups. The default is “max”.

    Swap usage hard limit. If a cgroup’s swap usage reaches this limit, anonymous memory of the cgroup will not be swapped out.

14. memory.swap.events

    A read-only flat-keyed file which exists on non-root cgroups. The following entries are defined. Unless specified otherwise, a value change in this file generates a file modified event.

    ```
       high
         The number of times the cgroup's swap usage was over
         the high threshold.
    
       max
         The number of times the cgroup's swap usage was about
         to go over the max boundary and swap allocation
         failed.
    
       fail
         The number of times swap allocation failed either
         because of running out of swap system-wide or max
         limit.
    ```

    When reduced under the current usage, the existing swap entries are reclaimed gradually and the swap usage may stay higher than the limit for an extended period of time. This reduces the impact on the workload and memory management.

15. memory.pressure

    A read-only nested-key file which exists on non-root cgroups.

    Shows pressure stall information for memory. See :ref:`Documentation/accounting/psi.rst <psi>` for details.

### 使用建议

`memory.high` 是**控制内存使用量的主要机制**。重要策略：

1. high limit 超分（high limits 总和大于可用内存）
2. 让全局内存压力（global memory pressure）根据使用量分配内存

由于超过 high limit 之后**只会 throttle 该 cgroup 而不会触发 OOM killer**， 因此 management agent 有足够的机会来监控这种情况及采取合适措施， 例如增加内存配额，或者干掉该 workload。

**判断一个 cgroup 内存是否够**并不是一件简单的事情，因为内存使用量 并不能反映出增加内存之后，workload 性能是否能有改善。例如，从网络接收数据然后写 入本地文件的 workload，能充分利用所有可用内存；但另一方面，即使只给它很小一部分 内存，这种 workload 的性能也同样是高效的。 **内存压力（memory pressure）的测量** —— 即由于内存不足导致 workload 受了多少影响 —— 对判断一个 workload 是否需要更多内存来说至关重要；但不 幸的是，**内核还未实现内存压力监控机制**。

### Memory Ownership

A memory area is charged to the cgroup which instantiated it and stays charged to the cgroup until the area is released. Migrating a process to a different cgroup doesn’t move the memory usages that it instantiated while in the previous cgroup to the new cgroup.

A memory area may be used by processes belonging to different cgroups. To which cgroup the area will be charged is in-deterministic; however, over time, the memory area is likely to end up in a cgroup which has enough memory allowance to avoid high reclaim pressure.

If a cgroup sweeps a considerable amount of memory which is expected to be accessed repeatedly by other cgroups, it may make sense to use POSIX_FADV_DONTNEED to relinquish the ownership of memory areas belonging to the affected files to ensure correct memory ownership.

## 5.3 IO

The “io” controller regulates the distribution of IO resources. This controller implements both weight based and absolute bandwidth or IOPS limit distribution; however, weight based distribution is available only if cfq-iosched is in use and neither scheme is available for blk-mq devices.

### IO Interface Files

1. io.stat A read-only nested-keyed file.

   Lines are keyed by $MAJ:$MIN device numbers and not ordered. The following nested keys are defined.

   ```
      ======    =====================
      rbytes    Bytes read
      wbytes    Bytes written
      rios        Number of read IOs
      wios        Number of write IOs
      dbytes    Bytes discarded
      dios        Number of discard IOs
      ======    =====================
   ```

   An example read output follows::

   ```
      8:16 rbytes=1459200 wbytes=314773504 rios=192 wios=353 dbytes=0 dios=0
      8:0 rbytes=90430464 wbytes=299008000 rios=8950 wios=1252 dbytes=50331648 dios=3021
   ```

2. io.cost.qos

   A read-write nested-keyed file with exists only on the root cgroup.

   This file configures the Quality of Service of the IO cost model based controller (CONFIG_BLK_CGROUP_IOCOST) which currently implements “io.weight” proportional control. Lines are keyed by $MAJ:$MIN device numbers and not ordered. The line for a given device is populated on the first write for the device on “io.cost.qos” or “io.cost.model”. The following nested keys are defined.

   ```
    ======    =====================================
    enable    Weight-based control enable
    ctrl      "auto" or "user"
    rpct      Read latency percentile    [0, 100]
    rlat      Read latency threshold
    wpct      Write latency percentile   [0, 100]
    wlat      Write latency threshold
    min       Minimum scaling percentage [1, 10000]
    max       Maximum scaling percentage [1, 10000]
    ======    =====================================
   ```

   The controller is disabled by default and can be enabled by setting “enable” to 1. “rpct” and “wpct” parameters default to zero and the controller uses internal device saturation state to adjust the overall IO rate between “min” and “max”.

   When a better control quality is needed, latency QoS parameters can be configured. For example::

   ```
      8:16 enable=1 ctrl=auto rpct=95.00 rlat=75000 wpct=95.00 wlat=150000 min=50.00 max=150.0
   ```

   shows that on sdb, the controller is enabled, will consider the device saturated if the 95th percentile of read completion latencies is above 75ms or write 150ms, and adjust the overall IO issue rate between 50% and 150% accordingly.

   The lower the saturation point, the better the latency QoS at the cost of aggregate bandwidth. The narrower the allowed adjustment range between “min” and “max”, the more conformant to the cost model the IO behavior. Note that the IO issue base rate may be far off from 100% and setting “min” and “max” blindly can lead to a significant loss of device capacity or control quality. “min” and “max” are useful for regulating devices which show wide temporary behavior changes - e.g. a ssd which accepts writes at the line speed for a while and then completely stalls for multiple seconds.

   When “ctrl” is “auto”, the parameters are controlled by the kernel and may change automatically. Setting “ctrl” to “user” or setting any of the percentile and latency parameters puts it into “user” mode and disables the automatic changes. The automatic mode can be restored by setting “ctrl” to “auto”.

3. io.cost.model

   A read-write nested-keyed file with exists only on the root cgroup.

   This file configures the cost model of the IO cost model based controller (CONFIG_BLK_CGROUP_IOCOST) which currently implements “io.weight” proportional control. Lines are keyed by $MAJ:$MIN device numbers and not ordered. The line for a given device is populated on the first write for the device on “io.cost.qos” or “io.cost.model”. The following nested keys are defined.

   ```
      =====        ================================
      ctrl        "auto" or "user"
      model        The cost model in use - "linear"
      =====        ================================
   ```

   When “ctrl” is “auto”, the kernel may change all parameters dynamically. When “ctrl” is set to “user” or any other parameters are written to, “ctrl” become “user” and the automatic changes are disabled.

   When “model” is “linear”, the following model parameters are defined.

   ```
      =============    ========================================
      [r|w]bps    The maximum sequential IO throughput
      [r|w]seqiops    The maximum 4k sequential IOs per second
      [r|w]randiops    The maximum 4k random IOs per second
      =============    ========================================
   ```

   From the above, the builtin linear model determines the base costs of a sequential and random IO and the cost coefficient for the IO size. While simple, this model can cover most common device classes acceptably.

   The IO cost model isn’t expected to be accurate in absolute sense and is scaled to the device behavior dynamically.

   If needed, tools/cgroup/iocost_coef_gen.py can be used to generate device-specific coefficients.

4. io.weight

   A read-write flat-keyed file which exists on non-root cgroups. The default is “default 100”.

   The first line is the default weight applied to devices without specific override. The rest are overrides keyed by $MAJ:$MIN device numbers and not ordered. The weights are in the range [1, 10000] and specifies the relative amount IO time the cgroup can use in relation to its siblings.

   The default weight can be updated by writing either “default $WEIGHT” or simply “$WEIGHT”. Overrides can be set by writing “$MAJ:$MIN $WEIGHT” and unset by writing “$MAJ:$MIN default”.

   An example read output follows::

   ```
      default 100
      8:16 200
      8:0 50
   ```

5. io.max A read-write nested-keyed file which exists on non-root cgroups.

   BPS and IOPS based IO limit. Lines are keyed by $MAJ:$MIN device numbers and not ordered. The following nested keys are defined.

   ```
      =====        ==================================
      rbps        Max read bytes per second
      wbps        Max write bytes per second
      riops        Max read IO operations per second
      wiops        Max write IO operations per second
      =====        ==================================
   ```

   When writing, any number of nested key-value pairs can be specified in any order. “max” can be specified as the value to remove a specific limit. If the same key is specified multiple times, the outcome is undefined.

   BPS and IOPS are measured in each IO direction and IOs are delayed if limit is reached. Temporary bursts are allowed.

   ```
    Setting read limit at 2M BPS and write at 120 IOPS for 8:16::
   
      echo "8:16 rbps=2097152 wiops=120" > io.max
   
    Reading returns the following::
   
      8:16 rbps=2097152 wbps=max riops=max wiops=120
   
    Write IOPS limit can be removed by writing the following::
   
      echo "8:16 wiops=max" > io.max
   
    Reading now returns the following::
   
      8:16 rbps=2097152 wbps=max riops=max wiops=max
   ```

6. io.pressure A read-only nested-key file which exists on non-root cgroups.

   Shows pressure stall information for IO. See :ref:`Documentation/accounting/psi.rst <psi>` for details.

### Writeback

Page cache is dirtied through buffered writes and shared mmaps and written asynchronously to the backing filesystem by the writeback mechanism. Writeback sits between the memory and IO domains and regulates the proportion of dirty memory by balancing dirtying and write IOs.

The io controller, in conjunction with the memory controller, implements control of page cache writeback IOs. The memory controller defines the memory domain that dirty memory ratio is calculated and maintained for and the io controller defines the io domain which writes out dirty pages for the memory domain. Both system-wide and per-cgroup dirty memory states are examined and the more restrictive of the two is enforced.

cgroup writeback requires explicit support from the underlying filesystem. Currently, cgroup writeback is implemented on ext2, ext4, btrfs, f2fs, and xfs. On other filesystems, all writeback IOs are attributed to the root cgroup.

There are inherent differences in memory and writeback management which affects how cgroup ownership is tracked. Memory is tracked per page while writeback per inode. For the purpose of writeback, an inode is assigned to a cgroup and all IO requests to write dirty pages from the inode are attributed to that cgroup.

As cgroup ownership for memory is tracked per page, there can be pages which are associated with different cgroups than the one the inode is associated with. These are called foreign pages. The writeback constantly keeps track of foreign pages and, if a particular foreign cgroup becomes the majority over a certain period of time, switches the ownership of the inode to that cgroup.

While this model is enough for most use cases where a given inode is mostly dirtied by a single cgroup even when the main writing cgroup changes over time, use cases where multiple cgroups write to a single inode simultaneously are not supported well. In such circumstances, a significant portion of IOs are likely to be attributed incorrectly. As memory controller assigns page ownership on the first use and doesn’t update it until the page is released, even if writeback strictly follows page ownership, multiple cgroups dirtying overlapping areas wouldn’t work as expected. It’s recommended to avoid such usage patterns.

The sysctl knobs which affect writeback behavior are applied to cgroup writeback as follows.

```
  vm.dirty_background_ratio, vm.dirty_ratio
    These ratios apply the same to cgroup writeback with the
    amount of available memory capped by limits imposed by the
    memory controller and system-wide clean memory.

  vm.dirty_background_bytes, vm.dirty_bytes
    For cgroup writeback, this is calculated into ratio against
    total available memory and applied the same way as
    vm.dirty[_background]_ratio.
```

### IO Latency

This is a cgroup v2 controller for IO workload protection. You provide a group with a latency target, and if the average latency exceeds that target the controller will throttle any peers that have a lower latency target than the protected workload.

The limits are only applied at the peer level in the hierarchy. This means that in the diagram below, only groups A, B, and C will influence each other, and groups D and F will influence each other. Group G will influence nobody::

```
            [root]
      /       |       \
      A       B        C
     /  \     |
    D    F    G
```

So the ideal way to configure this is to set io.latency in groups A, B, and C. Generally you do not want to set a value lower than the latency your device supports. Experiment to find the value that works best for your workload. Start at higher than the expected latency for your device and watch the avg_lat value in io.stat for your workload group to get an idea of the latency you see during normal operation. Use the avg_lat value as a basis for your real setting, setting at 10-15% higher than the value in io.stat.

### How IO Latency Throttling Works

io.latency is work conserving; so as long as everybody is meeting their latency target the controller doesn’t do anything. Once a group starts missing its target it begins throttling any peer group that has a higher target than itself. This throttling takes 2 forms:

- Queue depth throttling. This is the number of outstanding IO’s a group is allowed to have. We will clamp down relatively quickly, starting at no limit and going all the way down to 1 IO at a time.
- Artificial delay induction. There are certain types of IO that cannot be throttled without possibly adversely affecting higher priority groups. This includes swapping and metadata IO. These types of IO are allowed to occur normally, however they are “charged” to the originating group. If the originating group is being throttled you will see the use_delay and delay fields in io.stat increase. The delay value is how many microseconds that are being added to any process that runs in this group. Because this number can grow quite large if there is a lot of swapping or metadata IO occurring we limit the individual delay events to 1 second at a time.

Once the victimized group starts meeting its latency target again it will start unthrottling any peer groups that were throttled previously. If the victimized group simply stops doing IO the global counter will unthrottle appropriately.

### IO Latency Interface Files

1. io.latency

   This takes a similar format as the other controllers.

   ```
        "MAJOR:MINOR target=<target time in microseconds"
   ```

2. io.stat

   If the controller is enabled you will see extra stats in io.stat in addition to the normal ones.

   ```
      depth
        This is the current queue depth for the group.
   
      avg_lat
        This is an exponential moving average with a decay rate of 1/exp
        bound by the sampling interval.  The decay rate interval can be
        calculated by multiplying the win value in io.stat by the
        corresponding number of samples based on the win value.
   
      win
        The sampling window size in milliseconds.  This is the minimum
        duration of time between evaluation events.  Windows only elapse
        with IO activity.  Idle periods extend the most recent window.
   ```

## 5.4 PID

PID 控制器用于在**进程数量超过设置的 limit 之后，禁止通过 fork() 或 clone() 创建新进程**。

- 依靠其他控制器是无法避免 cgroup 中的进程暴增问题的，例如，fork 炸弹能在触发内存 限制之前耗尽 PID 空间，因此引入了 PID 控制器。
- 注意，这里所说的 PID 指的是内核在使用的 TID 和进程 ID。

### 5.4.1 PID 接口文件：`pids.current/pids.max`

1. **`pids.max`**

   A read-write single value file which exists on non-root cgroups. The default is “max”.

   Hard limit of number of processes.

2. **`pids.current`**

   A read-only single value file which exists on all cgroups.

   cgroup 及其 descendants 中的**当前进程数**。

### 5.4.2 绕开 cgroup PID 限制，实现 `pids.current > pids.max`

上面提到，PID 控制器是**限制通过 `fork/clone` 来创建新进程**（超过限制之后返回 `-EAGAIN`）。 因此**只要不用这两个系统调用**，我们还是能将 cgroup 内的进程数量搞成 `current > max` 的。例如：

1. 设置 `pids.max` 小于 `pids.current`（即先有足够多的进程，再降低 `max` 配置），或者
2. 将足够多的进程从其他 cgroup 移动到当前 cgroup（迁移现有进程不需要 fork/clone）。

## 5.5 Cpuset

The “cpuset” controller provides a mechanism for constraining the CPU and memory node placement of tasks to only the resources specified in the cpuset interface files in a task’s current cgroup. This is especially valuable on large NUMA systems where placing jobs on properly sized subsets of the systems with careful processor and memory placement to reduce cross-node memory access and contention can improve overall system performance.

The “cpuset” controller is hierarchical. That means the controller cannot use CPUs or memory nodes not allowed in its parent.

### Cpuset Interface Files

1. cpuset.cpus A read-write multiple values file which exists on non-root cpuset-enabled cgroups.

   It lists the requested CPUs to be used by tasks within this cgroup. The actual list of CPUs to be granted, however, is subjected to constraints imposed by its parent and can differ from the requested CPUs.

   The CPU numbers are comma-separated numbers or ranges. For example::

   ```
    $ cat cpuset.cpus
    0-4,6,8-10
   ```

   An empty value indicates that the cgroup is using the same setting as the nearest cgroup ancestor with a non-empty “cpuset.cpus” or all the available CPUs if none is found.

   The value of “cpuset.cpus” stays constant until the next update and won’t be affected by any CPU hotplug events.

2. cpuset.cpus.effective

   A read-only multiple values file which exists on all cpuset-enabled cgroups.

   It lists the onlined CPUs that are actually granted to this cgroup by its parent. These CPUs are allowed to be used by tasks within the current cgroup.

   If “cpuset.cpus” is empty, the “cpuset.cpus.effective” file shows all the CPUs from the parent cgroup that can be available to be used by this cgroup. Otherwise, it should be a subset of “cpuset.cpus” unless none of the CPUs listed in “cpuset.cpus” can be granted. In this case, it will be treated just like an empty “cpuset.cpus”.

   Its value will be affected by CPU hotplug events.

3. cpuset.mems

   A read-write multiple values file which exists on non-root cpuset-enabled cgroups.

   It lists the requested memory nodes to be used by tasks within this cgroup. The actual list of memory nodes granted, however, is subjected to constraints imposed by its parent and can differ from the requested memory nodes.

   The memory node numbers are comma-separated numbers or ranges. For example::

   ```
    $ cat cpuset.mems
    0-1,3
   ```

   An empty value indicates that the cgroup is using the same setting as the nearest cgroup ancestor with a non-empty “cpuset.mems” or all the available memory nodes if none is found.

   The value of “cpuset.mems” stays constant until the next update and won’t be affected by any memory nodes hotplug events.

4. cpuset.mems.effective

   A read-only multiple values file which exists on all cpuset-enabled cgroups.

   It lists the onlined memory nodes that are actually granted to this cgroup by its parent. These memory nodes are allowed to be used by tasks within the current cgroup.

   If “cpuset.mems” is empty, it shows all the memory nodes from the parent cgroup that will be available to be used by this cgroup. Otherwise, it should be a subset of “cpuset.mems” unless none of the memory nodes listed in “cpuset.mems” can be granted. In this case, it will be treated just like an empty “cpuset.mems”.

   Its value will be affected by memory nodes hotplug events.

5. cpuset.cpus.partition

   A read-write single value file which exists on non-root cpuset-enabled cgroups. This flag is owned by the parent cgroup and is not delegatable.

   It accepts only the following input values when written to.

   ```
        "root"   - a partition root
        "member" - a non-root member of a partition
   ```

   When set to be a partition root, the current cgroup is the root of a new partition or scheduling domain that comprises itself and all its descendants except those that are separate partition roots themselves and their descendants. The root cgroup is always a partition root.

   There are constraints on where a partition root can be set. It can only be set in a cgroup if all the following conditions are true.

   ```
    1) The "cpuset.cpus" is not empty and the list of CPUs are
       exclusive, i.e. they are not shared by any of its siblings.
    2) The parent cgroup is a partition root.
    3) The "cpuset.cpus" is also a proper subset of the parent's
       "cpuset.cpus.effective".
    4) There is no child cgroups with cpuset enabled.  This is for
       eliminating corner cases that have to be handled if such a
       condition is allowed.
   ```

   Setting it to partition root will take the CPUs away from the effective CPUs of the parent cgroup. Once it is set, this file cannot be reverted back to “member” if there are any child cgroups with cpuset enabled.

   A parent partition cannot distribute all its CPUs to its child partitions. There must be at least one cpu left in the parent partition.

   Once becoming a partition root, changes to “cpuset.cpus” is generally allowed as long as the first condition above is true, the change will not take away all the CPUs from the parent partition and the new “cpuset.cpus” value is a superset of its children’s “cpuset.cpus” values.

   Sometimes, external factors like changes to ancestors’ “cpuset.cpus” or cpu hotplug can cause the state of the partition root to change. On read, the “cpuset.sched.partition” file can show the following values.

   ```
    "member"       Non-root member of a partition
    "root"         Partition root
    "root invalid" Invalid partition root
   ```

   It is a partition root if the first 2 partition root conditions above are true and at least one CPU from “cpuset.cpus” is granted by the parent cgroup.

   A partition root can become invalid if none of CPUs requested in “cpuset.cpus” can be granted by the parent cgroup or the parent cgroup is no longer a partition root itself. In this case, it is not a real partition even though the restriction of the first partition root condition above will still apply. The cpu affinity of all the tasks in the cgroup will then be associated with CPUs in the nearest ancestor partition.

   An invalid partition root can be transitioned back to a real partition root if at least one of the requested CPUs can now be granted by its parent. In this case, the cpu affinity of all the tasks in the formerly invalid partition will be associated to the CPUs of the newly formed partition. Changing the partition state of an invalid partition root to “member” is always allowed even if child cpusets are present.

## 5.6 Device controller

Device controller 管理对设备文件（device files）的访问，包括创建新文件（使用 `mknod`）和访问已有文件。

### 5.6.1 控制方式：基于 cgroup BPF 而非接口文件

cgroupv2 设备控制器**没有接口文件，而是实现在 cgroup BPF 之上**。 要控制对设备文件的访问时，用户需要：

1. 编写的 **`BPF_PROG_TYPE_CGROUP_DEVICE`** 类型的 BPF 程序；
2. 将 BPF 程序 attach 到指定的 cgroup，其中指定 **attach 类型为 `BPF_CGROUP_DEVICE`**。

在访问设备文件时，会**触发相应 BPF 程序的执行**，后者的返回值决定了 是否允许访问。

### 5.6.2 cgroup BPF 程序上下文和返回值

这种 BPF 程序接受一个 **`struct bpf_cgroup_dev_ctx \*`** 指针，

```
// https://github.com/torvalds/linux/blob/v5.10/include/uapi/linux/bpf.h#L4833

struct bpf_cgroup_dev_ctx {
    __u32 access_type; /* encoded as (BPF_DEVCG_ACC_* << 16) | BPF_DEVCG_DEV_* */
    __u32 major;
    __u32 minor;
};
```

字段含义：

- `access_type`：访问操作的类型，例如 **`mknod/read/write`**；
- `major` 和 `minor`：主次设备号；

BPF 程序返回值：

1. `0`：访问失败（`-EPERM`）
2. 其他值：访问成功。

### 5.6.3 cgroup BPF 程序示例

内核测试用例：

1. [tools/testing/selftests/bpf/progs/dev_cgroup.c](https://github.com/torvalds/linux/blob/v5.10/tools/testing/selftests/bpf/progs/dev_cgroup.c)
2. [tools/testing/selftests/bpf/test_dev_cgroup.c](https://github.com/torvalds/linux/blob/v5.10/tools/testing/selftests/bpf/test_dev_cgroup.c)

## 5.7 RDMA

The “rdma” controller regulates the distribution and accounting of RDMA resources.

### RDMA Interface Files

1. rdma.max

   A readwrite nested-keyed file that exists for all the cgroups except root that describes current configured resource limit for a RDMA/IB device.

   Lines are keyed by device name and are not ordered. Each line contains space separated resource name and its configured limit that can be distributed.

   The following nested keys are defined.

   ```
      ==========    =============================
      hca_handle    Maximum number of HCA Handles
      hca_object     Maximum number of HCA Objects
      ==========    =============================
   ```

   An example for mlx4 and ocrdma device follows::

   ```
      mlx4_0 hca_handle=2 hca_object=2000
      ocrdma1 hca_handle=3 hca_object=max
   ```

2. rdma.current

   A read-only file that describes current resource usage. It exists for all the cgroup except root.

   An example for mlx4 and ocrdma device follows::

   ```
      mlx4_0 hca_handle=1 hca_object=20
      ocrdma1 hca_handle=1 hca_object=23
   ```

## 5.8 HugeTLB

The HugeTLB controller allows to limit the HugeTLB usage per control group and enforces the controller limit during page fault.

### HugeTLB Interface Files

1. hugetlb..current

   Show current usage for “hugepagesize” hugetlb. It exists for all the cgroup except root.

2. hugetlb..max

   Set/show the hard limit of “hugepagesize” hugetlb usage. The default value is “max”. It exists for all the cgroup except root.

3. hugetlb..events

   A read-only flat-keyed file which exists on non-root cgroups.

   ```
      max
        The number of allocation failure due to HugeTLB limit
   ```

4. hugetlb..events.local

   Similar to hugetlb..events but the fields in the file are local to the cgroup i.e. not hierarchical. The file modified event generated on this file reflects only the local events.

## 5.9 Misc

### perf_event

perf_event controller, if not mounted on a legacy hierarchy, is automatically enabled on the v2 hierarchy so that perf events can always be filtered by cgroup v2 path. The controller can still be moved to a legacy hierarchy after v2 hierarchy is populated.

## 5.10 规范外（non-normative）的一些信息

本节内容**不属于 stable kernel API**，随时可能变化。

### CPU controller root cgroup 处理行为

When distributing CPU cycles in the root cgroup each thread in this cgroup is treated as if it was hosted in a separate child cgroup of the root cgroup. This child cgroup weight is dependent on its thread nice level.

For details of this mapping see sched_prio_to_weight array in kernel/sched/core.c file (values from this array should be scaled appropriately so the neutral - nice 0 - value is 100 instead of 1024).

### IO controller root cgroup 处理行为

Root cgroup processes are hosted in an implicit leaf child node. When distributing IO resources this implicit child node is taken into account as if it was a normal child cgroup of the root cgroup with a weight value of 200.

# 6 cgroup 命名空间（cgroupns）

容器环境中用 cgroup 和其他一些 namespace 来隔离进程，但 `/proc/$PID/cgroup` 文件 **可能会泄露潜在的系统层信息**。例如：

```
$ cat /proc/self/cgroup
0::/batchjobs/container_id1 # <-- cgroup 的绝对路径，属于系统层信息，不希望暴露给隔离的进程
```

因此**引入了 cgroup namespace**，以下简写为 **cgroupns** （类似于 network namespace 简写为 netns）。

## 6.1 基础

### 6.1.1 功能：对 `/proc/PID/cgroup` 和 cgroup mount 进行虚拟化

cgroupns 对**`/proc/$PID/cgroup` 文件和 cgroup 挂载视角**进行了虚拟化。

- **如果没有 cgroupns**，`cat /proc/$PID/cgroup` 看到的是**进程所属 cgroup 的绝对路径**；
- 有了 cgroupns 之后，看到的将是 cgroupns root 范围内的路径。

下面具体来看。

### 6.1.2 新建 cgroup namespace

可以用 **`clone(2)/unshare(2)` 指定 `CLONE_NEWCGROUP` flag** 来创建一个新的 cgroupns。

- 创建该 cgroupns 时，`unshare/clone` 所在的 cgroup namespace 称为 **cgroupns root**；
- 该 cgroupns 内的进程查看 `/proc/$PID/cgroup` 时，只能看到其 cgroupns root 范围内的 cgroup 文件路径。

也就是说，cgroupns 限制了 cgroup 文件路径的可见性。例如，没有创建 cgroup namespace 时的视图：

```
$ ls -l /proc/self/ns/cgroup
lrwxrwxrwx 1 root root 0 2014-07-15 10:37 /proc/self/ns/cgroup -> cgroup:[4026531835]

$ cat /proc/self/cgroup
0::/batchjobs/container_id1  # <-- 绝对路径
```

用 `unshare` 创建一个新 cgroupns 之后的视图：

```
$ ls -l /proc/self/ns/cgroup
lrwxrwxrwx 1 root root 0 2014-07-15 10:35 /proc/self/ns/cgroup -> cgroup:[4026532183]

$ cat /proc/self/cgroup
0::/                         # <-- cgroupns root 限制范围内的路径
```

### 6.1.3 多线程进程：线程 unshare 后的行为

对于多线程的进程，任何一个线程通过 unshare 创建新 cgroupns 时，整个进程（所有线程） 都会进入到新的 cgroupns。这对 v2 hierarchy 是很自然的事情，但对 v1 来说，可能是 不期望的行为。

### 6.1.4 cgroupns 生命周期

只要以下条件之一满足，cgroupns 就会活着：

1. 该 **cgroup 中还有活着的进程**，
2. 挂载的文件系统中，还有对象 **pin 在这个 cgroupns 上**。

当最后一个还在使用 cgroupns 的进程退出或文件系统 unmount 之后，这个 cgroupns 就销毁了。 但 cgroupns root 和真正的 cgroups 还是继续存在的。

## 6.2 进一步解释 cgroupns root 和视图

前面提到，cgroupns root 是指 `unshare(2)` 创建 cgroupns 时所在的 cgroup。 例如，如果 `/batchjobs/container_id1` cgroup 中的一个进程调用 `unshare`，那 `/batchjobs/container_id1` 就成了 cgroupns root。For the init_cgroup_ns, this is the real root (‘/’) cgroup.

即便创建这个 cgroupns 的进程后面移动到其他 cgroup，这个 **cgroupns root 也是不会变的**：

```
$ ~/unshare -c # 在当前 cgroup 中通过 unshare 命令创建一个 cgroupns

# 以下命令都在刚创建的 cgroupns 中执行的
$ cat /proc/self/cgroup
0::/

$ mkdir sub_cgrp_1                  # 创建一个 sub-cgroup
$ echo 0 > sub_cgrp_1/cgroup.procs  # 将当前 shell 进程迁移到新创建的 cgroup sub_cgrp_1
$ cat /proc/self/cgroup             # 查看当前 shell 进程的 cgroup 信息
0::/sub_cgrp_1                      # 可以看到是相对路径
```

每个进程获得了它自己的 namespace-specific `/proc/$PID/cgroup`。

运行在 cgroupns 中的进程，在 `/proc/self/cgroup` 中只能看到它们的 root cgroup 内的 cgroup 路径。 例如，还是在刚才 `unshare` 创建的 cgroupns 中：

```
# 接着上面的窗口，现在还是在创建出的 cgroupns 中执行命令
$ sleep 100000 &                      # 创建一个进程，放在后台执行
[1] 7353

$ echo 7353 > sub_cgrp_1/cgroup.procs # 将进程迁移到前面创建的 sub-cgroup 中
$ cat /proc/7353/cgroup               # 查看这个进程的 cgroup 信息，会看到是相对路径
0::/sub_cgrp_1
```

在默认 cgroupns 中，真实 cgroup path 还是可见的：

```
$ cat /proc/7353/cgroup
0::/batchjobs/container_id1/sub_cgrp_1 # 绝对路径，说明没有在新建的 cgroupns 中
```

在 sibling（兄弟）cgroupns (a namespace rooted at a different cgroup) 中，会显示 **相对 cgroup path**（相对于它自己的cgroupns root）。例如，如果 PID 7353 的 cgroupns root 是 `/batchjobs/container_id2`，那它将看到：

```
$ cat /proc/7353/cgroup
0::/../container_id2/sub_cgrp_1
```

注意：相对路径永远以 `/` 开头，以提醒用户这是相对于调用者的 cgroupns root 的路径。

## 6.3 在 cgroupns 之间迁移进程

cgroupns 内的进程，可以移出或移入 ns root，只要有**对外部 cgroup 的访问权限** （proper access to external cgroups）。例如，在 cgroupns root 是 `/batchjobs/container_id1` 的某 cgroupns 内，假设能访问全局 hierarchy，那可以通 过如下命令迁移进程：

```
$ cat /proc/7353/cgroup
0::/sub_cgrp_1

$ echo 7353 > batchjobs/container_id2/cgroup.procs
$ cat /proc/7353/cgroup
0::/../container_id2
```

注意，这种迁移方式**并不推荐**。cgroupns 内的进程只应当被暴露到它 自己的 cgroupns hierarchy 内。

还可以使用 `setns(2)` 将进程移动到其他 cgroupns，前提条件：

1. 有 CAP_SYS_ADMIN against its current user namespace
2. 有 CAP_SYS_ADMIN against the target cgroupns’s userns

当 attach 到其他 cgroupns 时，不会有隐式的 cgroup changes。 It is expected that the someone moves the attaching process under the target cgroupns root.

## 6.4 与其他 cgroupns 交互

**Namespace 相关的 cgroup hierarchy** 可以在 non-init cgroupns 内以如下方式挂载：

```
# mount -t <fstype> <device> <dir>
$ mount -t cgroup2 none $MOUNT_POINT
```

这会**挂载默认的 unified cgroup hierarchy**，并**将 cgroupns root 作为 filesystem root**。 这个操作需要 CAP_SYS_ADMIN 权限。

`/proc/self/cgroup` 的虚拟化，以及通过 namespace-private cgroupfs mount 来限制 进程能看到的 cgroup hierarchy，提供了容器的隔离的 cgroup 视角。

# 7 内核编程相关信息

一些与 cgroup 交互相关的内核编程信息。

## 文件系统对 writeback 的支持

A filesystem can support cgroup writeback by updating `address_space_operations->writepage[s]()` to annotate bio’s using the following two functions.

```
  wbc_init_bio(@wbc, @bio)
    Should be called for each bio carrying writeback data and
    associates the bio with the inode's owner cgroup and the
    corresponding request queue.  This must be called after
    a queue (device) has been associated with the bio and
    before submission.

  wbc_account_cgroup_owner(@wbc, @page, @bytes)
    Should be called for each data segment being written out.
    While this function doesn't care exactly when it's called
    during the writeback session, it's the easiest and most
    natural to call it as data segments are added to a bio.
```

With writeback bio’s annotated, cgroup support can be enabled per super_block by setting SB_I_CGROUPWB in ->s_iflags. This allows for selective disabling of cgroup writeback support which is helpful when certain filesystem features, e.g. journaled data mode, are incompatible.

wbc_init_bio() binds the specified bio to its cgroup. Depending on the configuration, the bio may be executed at a lower priority and if the writeback session is holding shared resources, e.g. a journal entry, may lead to priority inversion. There is no one easy solution for the problem. Filesystems can try to work around specific problem cases by skipping wbc_init_bio() and using bio_associate_blkg() directly.

# 8 `v1 core` 已弃用特性

1. Multiple hierarchies including named ones are not supported.
2. All v1 mount options are not supported.
3. The “tasks” file is removed and “cgroup.procs” is not sorted.
4. “cgroup.clone_children” is removed.
5. /proc/cgroups is meaningless for v2. Use “cgroup.controllers” file at the root instead.

# 9 `v1` 存在的问题及 `v2` 的设计考虑（rationales）

## 9.1 `v1` 多 hierarchy 带来的问题

v1 允许任意数量的 hierarchy，每个 hierarchy 可以启用任意数量的 controller。 这种方式看上去高度灵活，但**在实际中并不是很有用**。例如，

1. utility 类型的 controller（例如 freezer）本可用于多个 hierarchy，而 由于 v1 中每个 controller 只有一个实例，utility controller 的作用就大打折扣； 而 hierarchy 一旦被 populated 之后，控制器就不能移动到其他 hierarchy 的事实， 更是加剧了这个问题。
2. 另一个问题是，关联到某个 hierarchy 的所有控制器，只能拥有相同的 hierarchy 视图。 **无法在 controller 粒度改变这种视图**。

在实际中，这些问题严重制约着每个 hierarchy 能启用哪些控制器，导致的结果就是： **大部分 hierarchy 都启用了所有控制器**。而实际上只有联系非常紧密的 控制器 —— 例如 `cpu` 和 `cpuacct` —— 放到同一个 hierarchy 中才有意义。 因此最终的结果就是：

1. 用户空间最后**管理着多个非常类似的 hierarchy**，
2. 在执行 hierarchy 管理操作时，**每个 hierarchy 上都重复着相同的操作**。

此外，支持多个 hierarchy 代价也非常高。它使得 cgroup core 的实现更加复杂，更重要的是， 限制了 cgroup 如何使用以及每个控制器能做什么。

1. 由于未限制 hierarchy 数量，因此一个线程的 cgroup membership 无法用有限长度来描述。

   cgroup 文件可能包含任意数量（行数）的 entry，长度是没有限制的，使得管理非常棘手， 最终不得**不加一些特殊的控制器**，而这些控制器的唯一目的就是识 别 membership，这反过来又加剧了最初的问题：hierarchy 数量不断增加。

2. 由于 controller 无法对其他 controller 所在的 hierarchy 拓扑做出预测，每个 controller 只能假设所有控制器都 attach 到了完全正交的 hierarchies。 这使得无法 —— 或至少非常困难 —— **实现控制器之间的协作**。

   在大部分场景下，将控制器放到多个完全正交的 hierarchy 都是没必要的。大家更 希望的是不同控制器能有不同层级的控制粒度。换句话说，从某个具体的 controller 角 度来看时，hierarchy 能够自底向上（from leaf towards root）collapse。例如 ，某个配置能不关心内存是否已经超过限制，而只关心 CPU cycle 的分配是否符合设置。

## 9.2 线程粒度（thread granularity）

cgroup v1 allowed threads of a process to belong to different cgroups. This didn’t make sense for some controllers and those controllers ended up implementing different ways to ignore such situations but much more importantly it blurred the line between API exposed to individual applications and system management interface.

Generally, in-process knowledge is available only to the process itself; thus, unlike service-level organization of processes, categorizing threads of a process requires active participation from the application which owns the target process.

cgroup v1 had an ambiguously defined delegation model which got abused in combination with thread granularity. cgroups were delegated to individual applications so that they can create and manage their own sub-hierarchies and control resource distributions along them. This effectively raised cgroup to the status of a syscall-like API exposed to lay programs.

First of all, cgroup has a fundamentally inadequate interface to be exposed this way. For a process to access its own knobs, it has to extract the path on the target hierarchy from /proc/self/cgroup, construct the path by appending the name of the knob to the path, open and then read and/or write to it. This is not only extremely clunky and unusual but also inherently racy. There is no conventional way to define transaction across the required steps and nothing can guarantee that the process would actually be operating on its own sub-hierarchy.

cgroup controllers implemented a number of knobs which would never be accepted as public APIs because they were just adding control knobs to system-management pseudo filesystem. cgroup ended up with interface knobs which were not properly abstracted or refined and directly revealed kernel internal details. These knobs got exposed to individual applications through the ill-defined delegation mechanism effectively abusing cgroup as a shortcut to implementing public APIs without going through the required scrutiny.

This was painful for both userland and kernel. Userland ended up with misbehaving and poorly abstracted interfaces and kernel exposing and locked into constructs inadvertently.

## 9.3 内部节点（inner nodes）与线程之间竞争

cgroup v1 允许线程在任意 cgroup，这导致了一个很有趣的问题： threads belonging to a parent cgroup and its children cgroups competed for resources. This was nasty as two different types of entities competed and there was no obvious way to settle it. Different controllers did different things.

The cpu controller considered threads and cgroups as equivalents and mapped nice levels to cgroup weights. This worked for some cases but fell flat when children wanted to be allocated specific ratios of CPU cycles and the number of internal threads fluctuated - the ratios constantly changed as the number of competing entities fluctuated. There also were other issues. The mapping from nice level to weight wasn’t obvious or universal, and there were various other knobs which simply weren’t available for threads.

The io controller implicitly created a hidden leaf node for each cgroup to host the threads. The hidden leaf had its own copies of all the knobs with `leaf_` prefixed. While this allowed equivalent control over internal threads, it was with serious drawbacks. It always added an extra layer of nesting which wouldn’t be necessary otherwise, made the interface messy and significantly complicated the implementation.

The memory controller didn’t have a way to control what happened between internal tasks and child cgroups and the behavior was not clearly defined. There were attempts to add ad-hoc behaviors and knobs to tailor the behavior to specific workloads which would have led to problems extremely difficult to resolve in the long term.

Multiple controllers struggled with internal tasks and came up with different ways to deal with it; unfortunately, all the approaches were severely flawed and, furthermore, the widely different behaviors made cgroup as a whole highly inconsistent.

This clearly is a problem which needs to be addressed from cgroup core in a uniform way.

## 9.4 其他 cgroup 接口相关的问题

v1 的设计并**没有前瞻性**，因此后面引入了**大量的怪异特性和不一致性**。

### 9.4.1 核心接口

cgroup core 中的问题，例如：

- 如何通知一个 empty cgroup。v1 的实现非常粗暴： 对于每个事件都 fork 执行一个用户空间 helper binary。
- event delivery 也是不可递归或 delegatable 的。这也使内核中的事件 delivery 过滤机制让 cgroup 接口变得更复杂。

### 9.4.2 控制器接口

控制器接口也有问题。

- 一个极端的例子：控制器完全不管 hierarchical organization，认为所有 cgroup 都直接位于 root cgroup 下面。
- 一些控制器暴露了大量的、不一致的实现细节给用户空间。

### 9.4.3 控制器行为

控制器行为也有不一致。

创建一个新 cgroup 之后，某些控制器默认不会施加限制，而另一些控制器则会直接禁用 资源，需要用户显式配置来解除禁用。 Configuration knobs for the same type of control used widely differing naming schemes and formats. Statistics and information knobs were named arbitrarily and used different formats and units even in the same controller.

v2 建立了**通用约定**，并更新了控制器设计，以使得它们只需**暴露最少且一致的接口**。

## 9.5 一些 controller 相关的问题及解决方式

### Memory

The original lower boundary, the soft limit, is defined as a limit that is per default unset. As a result, the set of cgroups that global reclaim prefers is opt-in, rather than opt-out. The costs for optimizing these mostly negative lookups are so high that the implementation, despite its enormous size, does not even provide the basic desirable behavior. First off, the soft limit has no hierarchical meaning. All configured groups are organized in a global rbtree and treated like equal peers, regardless where they are located in the hierarchy. This makes subtree delegation impossible. Second, the soft limit reclaim pass is so aggressive that it not just introduces high allocation latencies into the system, but also impacts system performance due to overreclaim, to the point where the feature becomes self-defeating.

The memory.low boundary on the other hand is a top-down allocated reserve. A cgroup enjoys reclaim protection when it’s within its effective low, which makes delegation of subtrees possible. It also enjoys having reclaim pressure proportional to its overage when above its effective low.

The original high boundary, the hard limit, is defined as a strict limit that can not budge, even if the OOM killer has to be called. But this generally goes against the goal of making the most out of the available memory. The memory consumption of workloads varies during runtime, and that requires users to overcommit. But doing that with a strict upper limit requires either a fairly accurate prediction of the working set size or adding slack to the limit. Since working set size estimation is hard and error prone, and getting it wrong results in OOM kills, most users tend to err on the side of a looser limit and end up wasting precious resources.

The memory.high boundary on the other hand can be set much more conservatively. When hit, it throttles allocations by forcing them into direct reclaim to work off the excess, but it never invokes the OOM killer. As a result, a high boundary that is chosen too aggressively will not terminate the processes, but instead it will lead to gradual performance degradation. The user can monitor this and make corrections until the minimal memory footprint that still gives acceptable performance is found.

In extreme cases, with many concurrent allocations and a complete breakdown of reclaim progress within the group, the high boundary can be exceeded. But even then it’s mostly better to satisfy the allocation from the slack available in other groups or the rest of the system than killing the group. Otherwise, memory.max is there to limit this type of spillover and ultimately contain buggy or even malicious applications.

Setting the original memory.limit_in_bytes below the current usage was subject to a race condition, where concurrent charges could cause the limit setting to fail. memory.max on the other hand will first set the limit to prevent new charges, and then reclaim and OOM kill until the new limit is met - or the task writing to memory.max is killed.

The combined memory+swap accounting and limiting is replaced by real control over swap space.

The main argument for a combined memory+swap facility in the original cgroup design was that global or parental pressure would always be able to swap all anonymous memory of a child group, regardless of the child’s own (possibly untrusted) configuration. However, untrusted groups can sabotage swapping by other means - such as referencing its anonymous memory in a tight loop - and an admin can not assume full swappability when overcommitting untrusted jobs.

For trusted jobs, on the other hand, a combined counter is not an intuitive userspace interface, and it flies in the face of the idea that cgroup controllers should account and limit specific physical resources. Swap space is a resource like all others in the system, and that’s why unified hierarchy allows distributing it separately.

> 原文链接：https://arthurchiao.art/blog/cgroupv2-zh/

