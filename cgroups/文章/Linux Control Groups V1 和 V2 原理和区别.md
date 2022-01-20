# Linux Control Groups V1 和 V2 原理和区别

(备注: 不显示声明就是基于V1版本来讲解的)

1. 什么是 `cgroups`.
2. 为什么我们需要 `cgroups`.
3. `crgoups` 是如何实现的.
4. 如何使用 `Cgroups`
5. `Cgroup` V2 版本有什么不一样
6. 总结

## 什么是 cgroups

### cgroup 基本概念

`cgroups` 机制是用来限制一个进程或者多个进程的资源。

概念:

1. Subsystem(子系统): 每种可以控制的资源都被定义成一个子系统，比如CPU子系统，Memory子系统。
2. Control Group: cgroup 是用来对一个 subsystem(子系统)或者多个子系统的资源进行控制。
3. 层级(Hierarchy): Control group 使用层次结构 (Tree) 对资源做划分。参考下图:

![This is an memory_cgroup.png](https://mikechengwei.github.io/2020/06/03/cgroup%E5%8E%9F%E7%90%86/memory_cgroup.png)

每个层级都会有一个根节点, 子节点是根节点的比重划分。

关系:

1. 一个子系统最多附加到一个层级（Hierarchy） 上。
2. 一个 层级(Hierarchy) 可以附加多个子系统

### 进程和Cgroup的关系

一个进程限制内存和CPU资源，就会绑定到CPU Cgroup和Memory Cgroup的节点上，Cpu cgroup 节点和Memory cgroup节点 属于两个不同的Hierarchy 层级。进程和 cgroup 节点是多对多的关系，因为一个进程涉及多个子系统，每个子系统可能属于不同的层次结构(Hierarchy)。

如图:

![This is an 进程和cgroup的关系.png](https://mikechengwei.github.io/2020/06/03/cgroup%E5%8E%9F%E7%90%86/%E8%BF%9B%E7%A8%8B%E5%92%8Ccgroup%E7%9A%84%E5%85%B3%E7%B3%BB.png)

上图 P 代表进程，因为多个进程可能共享相同的资源，所以会抽象出一个 `CSS_SET`, 每个进程会属于一个CSS_SET 链表中，同一个 `CSS_SET` 下的进程都被其管理。一个 `CSS_SET` 关联多个 Cgroup节点，也就是关联多个子系统的资源控制，那么 `CSS_SET`和 `Cgroup`节点就是多对多的关系。

参考下 `CSS_SET` 结构定义:

```
#ifdef CONFIG_CGROUPS
	/* Control Group info protected by css_set_lock */
	struct css_set __rcu *cgroups; 关联的cgroup 节点
	/* cg_list protected by css_set_lXock and tsk->alloc_lock */
	struct list_head cg_list; // 关联所有的进程
#endif
```

## 为什么我们需要 `cgroups`

我们希望能够细粒度的控制资源，我们可以为一个系统的不同用户提供不同的资源使用量，比如一个学校的校园服务器，老师用户可以使用15%的资源，学生用户可以使用5%的资源。我们可以用 `cgroups` 机制做到。

## crgoups 是如何实现的

### cgroups 数据结构

- 每个进程都会指向一个 `CSS_SET` 数据结构.（上文 进程和cgroups关系已经提供过）

  参考源码:

  ```
  struct task_struct { //进程的数据结构
  ...
  #ifdef CONFIG_CGROUPS
  	/* Control Group info protected by css_set_lock */
  	struct css_set __rcu *cgroups; 关联的cgroup 节点
  	/* cg_list protected by css_set_lXock and tsk->alloc_lock */
  	struct list_head cg_list; // 关联所有的进程
  #endif
  ...
  }
  ```

- 一个 `CSS_SET` 关联多个 `cgroup_subsys_state` 对象，`cgroup_subsys_state` 指向一个 cgroup 子系统。所以进程和 cgroup 是不直接关联的，需要通过 `cgroup_subsys_state` 对象确定属于哪个层级，属于哪个 `Cgroup` 节点。

  参考下 `CSS_SET`源码:

  

- 一个 Cgroup Hierarchy(层次）其实是一个文件系统, 可以挂载在用户空间查看和操作。
- 我们可以查看 绑定任何一个cgroup节点下的所有进程Id(PID).
  - 实现原理: 通过进程的fork和退出，从 `CSS_SET` attach 或者 detach 进程。

### cgroups 文件系统

上面我们了解到进程和Cgroup的关系，那么在用户空间内的进程是如何使用 Cgroup功能的呢？

Cgroup 通过 VFS 文件系统将功能暴露给用户，用户创建一些文件，写入一些参数即可使用，那么用户使用Crgoup功能会创建哪些文件？

文件如下:

- tasks 文件: 列举绑定到某个 cgroup的 所有进程ID（PID）.
- cgroup.procs 文件: 列举 一个Cgroup节点下的所有 线程组Id.
- `notify_on_release` flag 文件: ：填 0 或 1，表示是否在 cgroup 中最后一个 task 退出时通知运行release agent，默认情况下是 0，表示不运行。
- release_agent 文件： 指定 release agent 执行脚本的文件路径（该文件在最顶层 cgroup 目录中存在），在这个脚本通常用于自动化umount无用的 cgroup
- 每个子系统也会创建一些特有的文件。

#### 什么是 VFS 文件系统

VFS 是一个内核抽象层，能够隐藏具体文件系统的实现细节，从而给用户态进程提供一套统一的 API 接口。VFS 使用了一种通用文件系统的设计，具体的文件系统只要实现了 VFS 的设计接口，就能够注册到 VFS 中，从而使内核可以读写这种文件系统。 这很像面向对象设计中的抽象类与子类之间的关系，抽象类负责对外接口的设计，子类负责具体的实现。其实，VFS本身就是用 c 语言实现的一套面向对象的接口。

#### clone_children flag 是干什么的

这个标志只会影响 cpuset 子系统，如果这个标志在 cgroup 中开启，一个新的 cpuset 子系统 cgroup节点 的配置会继承它的父级cgroup节点配置。

## 如何使用 Cgroups

我们创建一个 `Cgroup`,使用到 “cpuset” cgroup子系统，可以按照下面的步骤:

1. mount -t tmpfs cgroup_root /sys/fs/cgroup
2. mkdir /sys/fs/cgroup/cpuset
3. mount -t cgroup -ocpuset cpuset /sys/fs/cgroup/cpuset
4. 通过创建和写入新的配置到 `/sys/fs/cgroup/cpuset` 虚拟文件系统，创建新的 cgroup
5. 启动一个 父进程任务
6. 得到进程PID，写入到 `/sys/fs/cgroup/cpuset` tasks 文件中
7. fork,exec 或者 clone 父进程任务。

举个例子，我们可以创建一个cgroup名字叫 “Charlie”，包含CPU资源 2到3核，memory 节点为1，操作如下:

```
 mount -t tmpfs cgroup_root /sys/fs/cgroup
 mkdir /sys/fs/cgroup/cpuset
 mount -t cgroup cpuset -ocpuset /sys/fs/cgroup/cpuset
 cd /sys/fs/cgroup/cpuset
 mkdir Charlie
 cd Charlie
 /bin/echo 2-3 > cpuset.cpus
 /bin/echo 1 > cpuset.mems
 /bin/echo $$ > tasks
 
## 查看cgroup信息
 sh
 # sh 是进入当前cgroup
 cat /proc/self/cgroup
```

## `Cgroup` V2 版本有什么不一样

不同于 v1 版本， cgroup v2 版本只有一个层级 Hierarchy(层级).

cgroup v2 的层级可以通过下面的命令进行挂载:

```
# mount -t cgroup2 none $MOUNT_POINT
```

cgroup2 文件系统有一个根 Cgroup ，以 `0x63677270`数字来标识，所有支持v2版本的子系统控制器会自动绑定到 v2的唯一层级上并绑定到根 Cgroup.没有使用 cgroup v2版本的进程，也可以绑定到 v1版本的层级上，保证了前后版本的兼容性。

在V2 版本中，因为只有一个层级，所有进程只绑定到cgroup的叶子节点。

如图:

![img](https://mikechengwei.github.io/2020/06/03/cgroup%E5%8E%9F%E7%90%86/V2%E5%B1%82%E7%BA%A7.png)

节点说明:

- 父节点开启的子系统控制器控制到儿子节点，比如 A节点开启了memory controller，那么 C节点cgroup就可以控制进程的memory.
- 叶子节点不能控制开启哪些子系统的controller,因为叶子节点关联进程Id.所以非叶子节点不能控制进程的使用资源。

cgroup v2的cgroup目录下文件说明:

- `cgroup.procs`文件，用来关联 进程Id。这个文件在V1版本使用列举线程组Id的。

- cgroup.controllers文件(只读)和cgroup.subtree_control文件 是用来控制 子 Cgroup 节点可以使用的 子系统控制器。

- tasks文件用来 关联进程信息，只有叶子节点有此文件。

  ### 为什么这么改造？

  v1 版本为了灵活一个进程可能绑定多个层级(Hierarchy)，但是通常是每个层级对应一个子系统，多层级就显得没有必要。所以一个层级包含所有的子系统就比较简单容易管理。

### 线程模式

`Cgroup` v2 版本支持线程模式，将 `threaded` 写入到 cgroup.type 就会开启 Thread模式。当开始线程模式后，一个进程的所有线程属于同一个cgroup,会采用Tree结构进行管理。

## 总结

通过对 Cgroup的学习，大致了解 Linux Crgoup 的数据结构，V2 版本层级结构的优化和 支持线程模式的功能。

> 原文链接：https://mikechengwei.github.io/2020/06/03/cgroup%E5%8E%9F%E7%90%86/
