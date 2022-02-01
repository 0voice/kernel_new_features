本文旨在作为 Linux 的最新 IO 接口 —— `io_uring` 的介绍，并将其与现有技术进行比较。我们将探讨其存在的原因，它的内部工作原理以及其用户可见的接口。

本文不会详细介绍具体的命令和函数，因为那只是重复相关 manpage 中的信息。相反，本文将尽力阐述对 `io_uring` 的介绍以及它是如何工作的，目标是希望让读者对这一切如何联系在一起有更深的理解。

也就是说，这篇文章和 manpage 之间会有一些重叠。要是什么细节也不提，那也基本不可能讲明白 `io_uring`。

## 1.0 介绍

Linux 上有很多基于文件 IO 的方法。最古老的和最基本的是 `read(2)` 和 `write(2)` 系统调用。后来 `pread(2)` 和 `pwrite(2)` 通过允许传入偏移量的方式增强了它们。之后引入的 `preadv(2)` 和 `pwritev2(2)` 是前者的矢量版本。因为这些系统调用依然不够丰富，Linux 还拥有 `pread2(2)` 和 `pwrite2(2)` 的系统调用，它们进一步扩展了 API 以允许使用修饰符标志。

除去这些系统调用的各种差异，它们的共同特征是同步的接口。这意味着当数据准备就绪（或写入完毕）时，系统调用才结束。

对于某些次优的情况，需要一个异步的接口。POSIX 拥有 `aio_read(3)` 和 `aio_write(3)` 来满足这个需求，但是他们的实现乏善可陈，并且性能很差。

Linux有一个原生的异步 IO 接口，简称为 aio。不幸的是，这个东西（aio）有以下的局限性：

1. **最大的限制条件显然是这东西只能支持 `O_DIRECT`（或称无缓冲）的异步 IO 访存。** 由于 `O_DIRECT` 本身的限制（缓存绕过，以及大小/对齐约束条件），原生的 aio 接口对大多数场景都不适用。对普通的（有缓冲的）IO，接口的行为是同步的。

2. **即使你满足了 IO 能变成异步的所有条件，有些时候它其实也不是异步的。** 有很多情况能让 IO 提交最终变成阻塞的 —— 如果需要元数据来执行 IO，提交最后就会被阻塞。

   对于存储设备，同一时间只有固定数目的请求槽位可用。要是这些槽位都被占满了，提交就会阻塞等待，直到有一个能用的槽位。依赖于异步提交的程序依然被迫阻塞于这些不确定的因素。

3. **垃圾 API。** 每个 IO 提交最终都需要拷贝 64+8 字节，每次 IO 完成都要拷贝 32 字节。这就得拷贝 104 字节的内存，然而 IO 本应是零额外拷贝的。取决于你的 IO 大小，这一点肯定很明显。

   公开的完成事件（completion event）环缓冲区（ring buffer）大多数情况下导致 IO 完成更慢，并且非常非常难（几乎不可能）从应用层正确地使用。

   IO 总是需要至少两次系统调用（提交 + 等待完成），而在 Intel 修复 spectre/meltdown 漏洞后会严重拖慢速度。

多年以来，人们一直在为解决上述的第一条限制（译注：不支持缓冲的 IO）而做着各种努力（作者本人在 2010 年的时候也试过），但没人成功过。

从效率的方面来说，支持低于 10 微秒时延、超高 IOPS 的硬件设备的到来，使得这接口开始显老了。对于这些类型的设备来说，缓慢和非确定性的提交延迟是很致命的问题，就像单核榨不出多少性能一样。

除此之外，因为上述的限制，可以说原生 Linux aio 的用例并不多。它已经被归入小众应用的一个角落，带来了随之而来的所有问题（长期未发现的 bug 等）。

此外，“普通”的应用用不着 aio，这说明 Linux 仍然缺乏一个能够提供他们所希望的功能的接口。应用或库完全没有理由继续需要创建私有的 IO 线程池来获得像样的异步 IO，尤其是当内核可以更高效地完成这些工作的时候。

## 2.0 改善现状

最初的工作集中在改善 aio 接口上。并且在放弃之前，这项工作进展的相当缓慢.

选择此初始方向有多种原因:

1. **如果你扩展和改进现有接口，比提供新接口要好的多。** 采用新的接口需要花费时间，并且要审核和批准新接口可能是一项漫长而艰巨的任务。
2. **一般而言，这项工作更加轻松。** 作为一个开发者，你总是希望以最少的投入完成最大的产出。扩展现有接口在已有的测试基础架构方面会带来许多优势。

现有的 aio 接口由三个主要的系统调用组成：用于设置 aio 上下文的系统调用（`io_setup(2)`）、用于提交 IO 的系统调用（`io_submit(2)`）和用于获取和等待 IO 完成的系统调用（`io_getevents(2)`）。由于需要对多个这些系统调用行为进行更改，因此我们需要添加新的系统调用以传递此信息。

这样就创建了指向同一代码的多个入口点，以及在其他位置的快捷方式。这样的结果在代码复杂性和可维护性方面不是很好，并且最终只能解决 AIO 一个比较突出的问题而已。最重要的是，这实际上使另外的问题变得更糟了，因为修改后的API更难被理解和使用。

尽管放弃一系列的工作，然后另起炉灶很艰难，但是很显然，我们需要一些全新的东西。一个可以满足我们所有需求的东西。我们需要它具有良好的性能和可扩展性，同时还要使它易于使用，并具有现有接口所缺乏的功能。

## 3.0 新接口的设计目标

尽管从头开始不是个很容易做出的决定，这确实让我们有了充分发挥艺术自由、创造新东西的空间。

按重要性由高到低的顺序，主要设计目标是：

1. **易于使用，难以误用。** 一切用户层可见的接口都应以此为主要目标。接口应该直观易用。
2. **可扩展。** 虽然我的背景主要是与存储相关的，但我希望这套接口不仅仅是用于面向块的 IO。也就是说，很快会添加网络和非块存储接口。造新轮子当然要（或者至少尝试）面向未来嘛。
3. **功能丰富。** Linux aio 只满足了一部分应用的需要。我不想再造一套只覆盖部分功能，或者需要应用重复造轮子（比如 IO 线程池）的接口。
4. **高效。** 尽管存储 IO 大多都是基于块的，因而大小多在 512 或 4096 比特，但在这些大小上的效率还是对某些应用至关重要的。此外，一些请求甚至可能不携带数据有效载荷。新接口得在每次请求的开销上精打细算。
5. **可伸缩。** 尽管效率和低时延很重要，但在峰值端提供最佳性能也很关键。特别是在存储方面，我们一直在努力提供一个可扩展的基础架构。新接口应该允许我们将这种可扩展性公开给应用。

上面的有些目标可能看起来互相矛盾。高效和可扩展的接口往往很难使用，更重要的是，很难正确使用。既要功能丰富，又要效率高，也很难做到。不过，这些都是我们的目标。

## 4.0 进入 `io_uring` 新时代

不论设计目标先后如何，最初的设计是以效率为中心。效率不是可以事后考虑的事情，必须从一开始进行设计，一旦接口固定就无法修改。

无论是提交或完成事件，我都不想有任何内存副本或间接内存访问。之前基于 aio 的设计时，aio 必须使用多个独立副本来处理两边的 IO，效率和可扩展性都明显受到了损害。

由于拷贝是不可取的，所以很明显，内核以及程序必须优雅地共享 IO 自身定义的结构并完成事件。如果你把共享的思路发散的足够远，自然会延伸到把共享数据的协调同样驻留在应用与内核共享的内存中。一旦你实现了这一飞跃，那么两者之间的同步必须以某种方式进行管理也就很清楚了。

一个应用无法在不执行系统调用的情况下与内核共享锁，并且系统调用肯定会减少与内核通信的速率。这与我们的效率目标是相悖的。

一种满足我们需求的数据结构应该是单生产者单消费者环形缓冲区。有了共享的环形缓冲区，我们可以通过巧妙运用内存顺序（memory ordering）和内存屏障（memory barriers）取代在应用和内核之间的共享锁。

异步接口有两个基本操作：提交请求的操作，以及请求完成后的完成事件。

对于提交 IO，应用是生产者，内核是消费者。而对于完成 IO 则恰好相反，内核会生产完成事件，应用则负责消费它们。因此，我们需要一对环形队列（ring）来为内核和应用之间提供一个高效通信通道。

这对环形队列就是新接口 `io_uring` 的核心。它们被合理命名为提交队列 （submission queue，SQ）以及完成队列（completion queue，CQ），并组成了新接口的基础部分。

### 4.1 数据结构

有了通信基础后，是时候看看如何定义用于描述请求和完成事件的数据结构了。

完成（completion）方面清晰明了。它需要携带与操作结果有关的信息，以及以某种方式将完成情况链接到来源的请求上。对于 `io_uring`，选定的布局如下：

```c
struct io_uring_cqe {
    __u64 user_data;
    __s32 res;
    __u32 flags;
};
```

现在我们应该认识了 `io_uring`，`_cqe` 后缀指的是一个完成队列事件（completion queue event），后面就略称为一个 cqe。

cqe 包含一个 `user_data` 字段。该字段是从请求提交开始就携带的，并且可以包含应用识别所述请求所需的任何信息。一种常见的用例是使其成为指向原始请求的指针。内核不会改动这个字段，只是简单地把这个字段从提交（submission）传递给完成事件（completion event）。

`res` 字段是请求的结果。可以把它类比成系统调用的返回值。对于普通的读/写操作，这就像 `read(2)` 或 `write(2)` 的返回值一样。成功就返回传输了多少字节。失败就返回一个负的错误代码。比如发生了 IO 错误，`res` 的值就是 `-EIO`。

最后，结构体的 `flags` 字段携带与本次操作有关的元信息数据。目前这个字段还没用上。

------

请求（request）类型的定义会更加复杂。它不仅需要描述比完成事件更多的信息，还要考虑到 `io_uring` 的一个设计目标，即对请求类型未来的扩展。

目前的定义如下：

```c
struct io_uring_sqe {
   __u8 opcode;
   __u8 flags;
   __u16 ioprio;
   __s32 fd;
   __u64 off;
   __u64 addr;
   __u32 len;
   union {
       __kernel_rwf_t rw_flags;
       __u32 fsync_flags;
       __u16 poll_events;
       __u32 sync_range_flags;
       __u32 msg_flags;   
   };
   __u64 user_data;
   union {
       __u16 buf_index;
       __u64 __pad2[3];
   };
};
```

> 译注：该结构已经更新，详细信息可以查看 [Submission Queue Entry](https://unixism.net/loti/ref-liburing/sqe.html)

和完成事件类似，提交端的数据结构叫做提交队列项（Submission Queue Entry），或者简称 sqe。

里面存着一个 `opcode` 字段，描述本次请求的操作码，表示当前的请求的操作。比如，有一个操作码叫 `IORING_OP_READV`，也就是向量读取（vectored read）。

`flags` 字段包含跨命令类型通用的修饰符标志。我们将在之后的进阶使用场景部分提及。

`ioprio` 是本次请求的优先级别。对一般的读写操作，这个字段遵循了 `ioprio_set(2)` 系统调用的定义。

`fd` 字段是与本次请求相关联的文件描述符（file descriptor），`off` 字段是这个操作执行时基于 `fd` 的偏移量。

如果操作码描述了一个传输数据的操作，那么 `addr` 字段就包含在应该在哪个地址进行这个操作。例如，如果操作是一个向量读/向量写之类的操作，那么这个 `addr` 字段里就是一个指向 `iovec` 结构体数组的指针，就和 `preadv(2)` 里用的一样。对于非向量化的 IO 传输，`addr` 就必须直接包含目标地址。

这就引入了另一个字段 `len`，对非向量传输来说这是要传输的字节量，而对向量传输来说就是 addr 指向的向量个数（ `iovec` 结构体数组的长度）。

接下来(`union`)是用于特定操作码的 flag 的一个集合。例如，对于上面提到的向量读（`IORING_OP_READV`），这些 flag 就和 `preadv2(2)` 系统调用中要求的保持一致。

`user_data` 在操作码中是通用的，并且内核不会去碰这个东西。当这个请求的完成事件被发布时，它被拷贝到完成队列事件结构体（cqe）中。

`buf_index` 之后在进阶用法中会提到的。

最后，在这个结构的末尾，有一些填充（padding）。这样做是为了确保 sqe 在内存中以 64 字节的大小对齐，同时也是为了将来可能需要包含更多数据以描述请求的情况。这里有几个能想到的用例 —— 一个是键/值存储命令集，另一个是端到端数据保护，其中程序为其想要写入的数据传递一个预先计算的校验和（checksum）。

### 4.2 通讯信道

描述过这些数据结构之后，我们来看看这个环（ring）工作的细节。

尽管说我们有一个提交端（submission side）和一个完成端（completion side），听起来可能挺对称，但是这两个的索引方式却是完全不同的。

就像上一节一样，让我们从简单的开始，先来讲讲完成环（completion ring）。

完成请求项（cqe）被组织成了一个数组，其对应的内存对应用和内核来说都是可见、可改的。然而，由于完成请求项（cqe）是内核生成的，实际上只有内核在修改 cqe 条目。

通讯是通过一个环缓冲区管理的。当一个新事件被内核提交到完成请求环（CQ 环）中时，它会更新与其相连的尾节点。当程序从中消费一项时，它会更新头节点。因此，只要头尾节点不同，应用就知道还有一个或更多事件可以用来消费。

环计数器（ring counter）本身是自由流动的 32 位整数，当环中事件的数目超过环的容量时，依靠自然进位（译注：二进制进位）处理。这种方法的一个优点是，我们可以利用环的完整容量，而不必额外管理一个“环满了”的 flag（这将让环的管理变得复杂）。因此，环的尺寸必须是 2 的整数次方。

要查找事件的索引，应用必须用环的大小掩码（size mark）对当前尾索引（current tail index）进行掩码（masking）操作。大概看起来像是这样：

```c
unsigned head;

head = cqring->head;
read_barrier();
if (head != cqring->tail) {
    struct io_uring_cqe *cqe;
    unsigned index;
    
    index = head & * (cqring->mask);
    cqe = &cqring->cqes[index];
    /* process completed cqe here */
    ...
    /* we've now consumed this entry */
    head++;
}

cqring->head = head;
write_barrier();
```

`ring->cqes[]` 是 `io_uring_cqe` 结构的共用数组。下一小节中我们将深入讨论这个共享内存（以及 `io_uring` 实例本身）如何设置和管理，以及神奇的读写屏障操作究竟有什么用。

对提交请求来说，角色是正好相反的。应用负责更新环位，而内核负责消费条目（并更新环头）。一个很重要的区别是，尽管完成队列环（CQ ring）直接索引 cqe 的共享数组，提交方与这些元素之间还有一个中间数组（indirection array，译注：中间数组包含多个 sqe）。因此，提交端的环形缓冲区是这个数组的索引，而该数组又包含 sqes 的索引。

这在一开始可能看起来很奇怪和令人困惑，但这是有原因的。有些应用可能会在内部数据结构中嵌入请求单元，这允许它们灵活地在一个操作中提交多个 sqe，使得上述应用更容易迁移到 `io_uring` 的接口。

添加一个 sqe 供内核使用基本上是从内核获取 cqe 的相反操作。一个典型的例子是这样的:

```c
struct io_uring_sqe *sqe;
unsigned tail, index;

tail = sqring->tail;
index = tail & (*sqring->ring_mask);
sqe = &sqring->sqes[index];

/* this call fills in the sqe entries for this IO */
init_io(sqe);

/* fill the sqe index into the SQ ring array */
sqring->array[index] = index;
tail++;

write_barrier();
sqring->tail = tail;
write_barrier();
```

和 CQ 环一样，读写屏障将在后面解释。上面是一个简化的例子，它假设 SQ 环当前是空的，或者至少它还有一个及以上条目的空间。

一旦内核使用了一个 sqe，应用就可以自由地重用这个 sqe 条目。即使在内核尚未完全完成该 sqe 的情况下也是如此。如果内核在条目被消费之后确实需要再次访问它，那么它将获得一个稳定的副本。为什么会发生这种情况并不一定重要，但是它对应用有一个重要的副作用。

通常情况下，应用会请求一个给定大小的环，并且假设这个大小可能直接对应于应用在内核中可以有多少个挂起的请求。但是，由于 sqe 生存期（lifetime）仅仅是实际提交的 sqe 生存期，因此应用可能会开出比 SQ 环大小指示的更高的挂起请求计数。应用必须注意不要这样做，否则可能会有 CQ 环溢出的风险。

默认情况下，CQ 环的大小是 SQ 环的两倍。这使得应用在管理这方面具有一定的灵活性，但并不能完全消除这样做的必要性。如果应用确实违反了这个限制，它将被追踪为 CQ 环中的一个溢出条件。稍后会有更多细节。

> 译注：现在内核提供了CQ环溢出后保证事件不丢失的能力（io_uring CQ ring backpressure），详见：https://lore.kernel.org/io-uring/20191106235307.32196-1-axboe@kernel.dk/

完成事件可以以任何顺序到达，请求提交和相应的完成之间没有顺序。SQ 环和 CQ 环相互独立运行。然而，完成事件将始终对应于给定的提交请求。因此，完成事件总是关联于特定的提交请求（之后）。

## 5.0 `io_uring` 接口

就像 aio 一样，`io_uring` 也有许多与之相关的系统调用来定义它的操作。第一个是用来设置 `io_uring` 实例的系统调用:

```c
int io_uring_setup(unsigned entries, struct io_uring_params *params);
```

应用必须为这个 io_uring 实例提供所需的条目数量，以及与之相关的一组参数。`entries`表示将与这个 io_uring 实例相关的 seq 的数量，它必须是2的幂数，范围是 [1, 4096]。`params`结构会被内核读取和写入，它被定义为：

```c
struct io_uring_params {
    __u32 sq_entries;
    __u32 cq_entries;
    __u32 flags;
    __u32 sq_thread_cpu;
    __u32 sq_thread_idle;
    __u32 resv[5];
    struct io_sqring_offsets sq_off;
    struct io_cqring_offsets cq_off;
};
```

`sq_entries`会被内核填写，让应用知道这个环支持 sqe 条目的数量。同样地，对于 cqe 条目，`cq_entries`告诉应用 CQ 环有多大。除了通过 io_uring 设置基本通信所必要的`sq_off`和`cq_off`字段，这个结构的其余部分将被推迟到高级用例部分再讨论。

在成功调用`io_uring_setup(2)`后，内核将返回一个指向 io_uring 实例的文件描述符，这时`sq_off`和`cq_off`便会派上用场。鉴于 sqe 和 cqe 结构由内核和应用所共享，应用需要一种对该内存访问的方法。这里使用`mmap(2)`将其映射到应用的内存空间，应用使用`sq_off`来计算各个环成员的偏移量。

`io_sqring_offsets`结构的定义如下：

```c
struct io_sqring_offsets {
    __u32 head;          /* offset of ring head */
    __u32 tail;          /* offset of ring tail */
    __u32 ring_mask;     /* ring mask value */
    __u32 ring_entries;  /* entries in ring */
    __u32 flags;         /* ring flags */
    __u32 dropped;       /* number of sqes not submitted */
    __u32 array;         /* sqe index array /
    __u32 resv1;
    __u64 resv2;
};
```

为了访问 sqe 共享内存，应用必须使用 io_uring 文件描述符和 SQ 环内存偏移量来调用`mmap(2)`。io_uring 的 API 定义了以下 mmap 偏移量，供应用使用：

```c
#define IORING_OFF_SQ_RING      0ULL
#define IORING_OFF_CQ_RING      0x8000000ULL
#define IORING_OFF_SQES         0x10000000ULL
```

`IORING_OFF_SQ_RING`用于将 SQ 环映射到应用内存空间，`IORING_OFF_CQ_RING`同理用于 CQ 环， `IORING_OFF_SQES`用于映射 sqe （中间）数组。cqe “数组”是 CQ 环自身的一部分，而 SQ 环记录的是 sqe 数组的索引值，所以 sqe 数组必须被独立映射。

应用将定义自己的结构用于保存这些偏移量。一个可能的例子如下：

```c
struct app_sq_ring {
    unsigned *head;
    unsigned *tail;
    unsigned *ring_mask;
    unsigned *ring_entries;
    unsigned *flags;
    unsigned *dropped;
    unsigned *array;
};
```

一个典型的设置案例：

```c
struct app_sq_ring app_setup_sq_ring(int ring_fd, struct io_uring_param *p){
    struct app_sq_ring sqing;
    void *ptr;
 
    ptr = mmap(NULL, p->sq_off.array + p->sq_entries * sizeof(__u32), PROT_READ | PROT_WRITE, MAP_SHARED | MAP_POPULATE, ring_fd, IORING_OFF_SQ_RING);
 
    sqring->head = ptr + p->sq_off.head;
    sqring->tail = ptr + p->sq_off.tail;
    sqring->ring_mask = ptr + p->sq_off.ring_mask;
    sqring->ring_entries = ptr + p->sq_off.ring_entries;
    sqring->flags = ptr + p->sq_off.flags;
    sqring->dropped = ptr + p->sq_off.dropped;
    sqring->array = ptr + p->sq_off.array;
 
    return sqring
}
```

CQ 环的映射与之类似，`cq_off`和`IORING_OFF_CQ_RING`用于映射 CQ 环。最后，使用`IORING_OFF_SQES`映射 sqe 数组。

由于这些是可以在应用间复用的样板代码， liburing 库接口提供了一组函数，用于帮助简单完成设置和内存映射。关于这方面的细节，请参见io_uring库部分。完成以上操作后，应用就可以通过 io_uring 实例进行通信了。

应用生产了新的请求需要处理时，还需要一种方式用于通知内核。这通过另一个系统调用完成：

```c
int io_uring_enter(unsigned int fd, unsigned int to_submit, unsigned int min_complete, unsigned int flags, sigset_t sig);
```

`fd`指`io_uring_setup(2)`返回的环文件描述符。`to_submit`告诉内核准备被消费和提交的 sqe 的数量。`min_complete`要求内核等待该数量的请求的完成。

这种调用兼具提交和等待请求完成，意味着应用可以通过一次该系统调用同时完成这两种需求。

`flags`包含修改调用行为的标识符，最重要的一个是：

```c
#define IORING_ENTER_GETEVENTS  (1U << 0)
```

如果`flags`中设置了`IORING_ENTER_GETEVENTS`，那么内核将会主动等待`min_complete`个完成事件。看起来`min_complete`和`IORING_ENTER_GETEVENTS`在功能上有重复，但是在有些情况下，这种区别是很重要的，这将在后面介绍。目前而言，如果希望等待完成，必须同时设置`IORING_ENTER_GETEVENTS`。

以上基本涵盖了 io_uring 的基本 API。`io_uring_setup(2)`将根据给定大小创建 io_uring 实例。实例设置完成后，应用可以填充 sqe 并且使用`io_uring_enter(2)`来提交它们。等待完成可以在同一个（`io_uring_enter`）调用中设置，或者稍后单独调用。

除非应用想要等待请求完成，它可以只检查 CQ 环尾是否有可用事件。内核将直接修改 CQ 环尾，因此应用可以直接消费（环中的）完成事件，而不必一定使用带`IORING_ENTER_GETEVENTS`的集合的`io_uring_enter(2)`。

可以通过`io_uring_enter` man page 来查看命令类型和使用方法。

### 5.1 sqe 排序

通常 sqe 被独立使用，也就是说一个 sqe 的执行不会影响环中后续 sqe 的执行和顺序。这使操作具有充分的灵活性，并且使它们能够以最大效率和性能并行地执行和完成。

可能需要排序的用例是保证数据完整性的写入。一个常见例子是一系列写操作，然后执行 fsync/fdatasync。只要我们可以允许以任何顺序完成写入，我们只需要关心在所有写入完成后执行数据同步。应用通常把它转换为一个写-等待操作，然后在所有的写被底层存储确认后发起同步。

io_uring 支持在所有先前的请求完成后再开始消费（drain）提交方队列。这允许应用排列上述的同步操作，并且知道在所有之前的命令完成前不会开始（新的消费）。需要在 sqe 的`flags`字段设置`IOSQE_IO_DRAIN`以完成上述操作。

需要注意，这将使整个提交队列停滞。取决于 io_uring 在具体应用中的使用，这可能导致比预期更大的流水线停顿（pipeline bubbles）。如果这类操作经常发生，应用可以使用一个独立的 io_uring 实例来负责完整性写入，以便更好地同时执行其他不相关命令。

### 5.2 链式 sqe

虽然`IOSQE_IO_DRAIN`包括一个完整的流水线屏障，但 io_uring 也支持更精细的 sqe 序列控制。链式 sqe 描述了更大的提交环内的一系列 sqe 间的依赖性，其中每个 sqe 的执行都依赖于前一个 sqe 的成功完成。链式 sqe 可以实现一系列有序的写操作，或者类似复制的操作，比如共享两个 sqe 的缓冲区，读取一个文件后写入到另一个文件。

应用通过在 sqe 的`flags`字段中设置`IOSQE_IO_LINK`来使用链式 sqe。设置后，sqe 不会在上一个 sqe 未成功完成前启动。如果前一个 sqe 没有完全完成，那么链将会断开，链式 sqe 将会被取消，并返回错误码`-ECANCELED`。在这里，完全完成（fully complete）是指请求完成完全成功（the fully successful completion of the request）。任何错误或者潜在的读写问题都会中断链，请求必须完全完成（the request must complete to its full extent）。

只要在`flags`字段中设置了`IOSQE_IO_LINK`，sqe 链就会一直存在。因此该链定义为从第一个设置了`IOSQE_IO_LINK`的 sqe 开始，到后续的第一个未设置该标识符的 sqe 结束。该链支持任意长度。

链的执行独立于提交环中的其他 sqe。链是一个独立的执行单位，多条链可以并行执行和完成。（执行单位）包括不属于任何链的 sqe。

### 5.3 超时命令

尽管大部分 io_uring 支持的命令都致力于数据，无论是 read/write 这类直接操作，还是 fsync 这类间接命令，但是超时命令稍有出入。`IORING_OP_TIMEOUT`帮助实现在完成环上的等待，而非数据相关的工作。超时命令支持两种不同的触发方式，它们可以同时在一个命令中使用。

一种触发方式是经典超时，调用者传递一个具有非零秒/纳秒值的`timespec`结构。为了保证32位与64位应用和内核空间之间的兼容性，必须使用以下格式：

```c
struct __kernel_timespec {
    int64_t tv_sec;
    long long tv_nsec;
};
```

在某些时候，用户空间也应该有一个`timespec64`结构来匹配这个描述。在此之前，必须使用上述（timespec）结构。如果需要定时超时，sqe 的 addr 字段必须指向这种结构类型，指定的时间过后，将会执行超时命令。

第二种触发方式是完成计数。如果采用这种方式，需要在 sqe 的 `offset` 字段填写完成计数值（completion count value）。经过指定数量的完成后，（超时）队列中的超时命令将会被执行。

一条超时命令可以同时指定两种超时方式。如果超时命令包含两种触发条件，只要有一个满足触发的条件就会生成超时完成事件。当一个超时完成事件被发布时，所有完成事件的等待者都会被唤醒，无论它们需要的完成量是否满足。

## 6.0 内存排序

通过 io_uring 实例进行兼顾安全和高效的通信的一个重要方面就是正确使用内存排序原语（memory ordering primitives）。本文的范围不包含对各种内存排序架构的详细介绍。如果你乐于使用 liburing 库提供的简化的 io_uring API，那么你可以略过此章节，并且跳转到 liburing 库章节阅读。如果你对使用原始接口有兴趣，那么理解本章节就很重要。

简单起见，我们将其简化为两个简单的内存排序操作。为了保证文章简短，解释会被简化。

`read_barrier()`：确保执行后续内存读前，之前的写操作是可见的。
`write_barrier()`：确保写操作有序。

根据讨论的架构不同，两者中的一个或两个可能是无操作的（no-ops）。使用 io_uring 时，这一点无关紧要。重要的是在某些架构中我们需要它们，因此应用开发者应当了解如何实现它们。

write_barrier() 需要确保写操作的顺序。比方说，一个应用想要填写一个 sqe，并通知内核有可供消费的 sqe。这可以分为两个操作阶段——首先，填写不同的 sqe 成员，并将 sqe 索引存放在 SQ 环数组中；然后，更新 SQ 环尾，并且通知内核可以消费新的条目。

在没有任何顺序要求的情况下，处理器以它认为最理想的任何顺序重新排列这些写操作是合法行为。让我们来看看下面的例子，每个数字表示一个内存操作：

```c
    1: sqe→opcode = IORING_OP_READV;
    2: sqe→fd = fd;
    3: sqe→off = 0;
    4: sqe→addr = &iovec;
    5: sqe→len = 1;
    6: sqe→user_data = some_value;
    7: sqring→tail = sqring→tail + 1;
```

（上述例子）无法确保使 sqe 向内核可见的写操作7，会在写入顺序的最后被执行。至关重要的是，所有在写操作7之前的写入操作都要在写操作7（完成前）可见，不然内核可能会看到一个写了一半的 sqe。从应用的角度来看，在通知内核有新的 sqe 之前，需要写屏障来确保写操作的正确顺序。由于实际上 sqe 能以任意顺序写入，只要它们能够在环尾写入（完成）前可见就行，我们可以通过在写操作6之后，写操作7之前使用一个排序原语来解决。因此写入顺序如下所示：

```c
    1: sqe→opcode = IORING_OP_READV;
    2: sqe→fd = fd;
    3: sqe→off = 0;
    4: sqe→addr = &iovec;
    5: sqe→len = 1;
    6: sqe→user_data = some_value;
    write_barrier(); /* ensure previous writes are seen before tail write */
    7: sqring→tail = sqring→tail + 1;
    write_barrier(); /* ensure tail write is seen */
```

内核在读取 SQ 环尾前，会使用 read_barrier() 来确保应用的环尾写操作可见。在 CQ 环这边，因为消费者/生产者角色的转换，应用只需要在读取 CQ 环尾前使用 read_barrier() 来确保内核的任意写操作是可见的。

尽管内存排序被简化为两种具体类型，架构实现还是会根据代码运行的机器不同而不同。即使应用直接使用 io_uring 接口（而非 liburing 帮助函数），它仍然需要架构特定的屏障类型。liburing 库提供这些屏障（函数），建议在应用中使用。

有了内存屏障的基本解释和 liburing 库提供管理它们的帮助函数，不妨回过头看看先前使用了 read_barrier() 和 write_barrier() 的例子，希望能为你解惑。

## 7.0 liburing 库

排除了 io_uring 内部细节，你现在可以放心地学习一种更简单的方式来完成上述大部分操作。liburing 库有两个目标：

 - 免于使用模板化代码设置 io_uring 实例
 - 简化基础使用场景需要的 API

后者确保应用不必担心内存屏障（的设置），也不必自己管理任何环缓冲区。这使 API 更加易于理解和使用，并且不需要去了解内部工作细节。如果我们只是关注基于 liburing 提供的示例，那么本文将会短得多，但是多了解一些内部工作原理，将对提高应用性能多有助益。

此外，liburing 目前专注于减少（使用）模板化代码并且为标准使用场景提供基础帮助函数。liburing 暂时还不能提供一些更高级的特性。然而，这并不意味着你不能混用这两者（译注：应指在标准使用场景下使用 liburing helper，在使用高级特性场景下手动设置）。它们在底层使用相同的结构。即使应用使用了原始接口，仍然推荐（更换）使用 liburing 提供的设置帮助。

### 7.1 liburing io_uring 设置

让我们从一个例子开始。liburing 提供了以下基本的帮助函数，它的功能和手动调用`io_uring_setup(2)`并随后对三个必要的区域（CQ 环、SQ 环和 sqe ）进行`mmap(2)`操作相同：

```c
    struct io_uring ring;
    io_uring_queue_init(ENTRIES, &ring, 0);
```

io_uring 结构保存了 SQ 环和 CQ 环的信息，并且`io_uring_queue_init(3)`的调用处理了所有的设置逻辑。在这里例子中，我们给`flags`参数传递了 0 值。

一旦一个应用使用完一个 io_uring 实例，它可以简单地调用：

```c
    io_uring_queue_exit(&ring);
```

来拆卸这个实例。和应用分配的其他资源相似，一旦应用退出，它们就会被内核自动回收。对于应用坑已经创建的任何 io_uring 实例都是这样。

### 7.2 liburing 提交和完成

一个非常基本的使用场景是，提交一个请求，然后等待它完成。如下是使用 liburing 帮助函数：

```c
    struct io_uring_sqe sqe;
    struct io_uring_cqe cqe;

    /* get an sqe and fill in a READV operation */
    sqe = io_uring_get_sqe(&ring);
    io_uring_prep_readv(sqe, fd, &iovec, 1, offset);

    /* tell the kernel we have an sqe ready for consumption */
    io_uring_submit(&ring);

    /* wait for the sqe to complete */
    io_uring_wait_cqe(&ring, &cqe);

    /* read and process cqe event */
    app_handle_cqe(cqe);
    io_uring_cqe_seen(&ring, cqe);
```

这些不言自明。最后的`io_uring_wait_cqe(3)`调用将会返回我们提交的相应的 sqe 的完成事件，前提是没有其他 sqe 正在运行。如果有，那么完成事件可能是对应其他 sqe 的。

如果应用只想查看完成（结果）而非等待完成事件，可以调用`io_uring_peek_cqe(3)`。在两种情况下（译注：指 io_uring_wait_cqe(3) 和 io_uring_peek_cqe(3)），应用都需要调用`io_uring_cqe_seen(3)`来处理完成事件，否则即使重复调用`io_uring_wait_cqe(3)`或`io_uring_peek_cqe(3)`，都只会返回（与之前）相同的完成事件。这种分隔（处理）是必要的，使内核能够避免在应用处理完之前覆盖掉现有的完成事件。`io_uring_cqe_seen(3)`会增加 CQ 环头，使内核可以在（被消费的完成事件）同一位置填写新的事件。

有很多帮助函数能够填写 sqe，例如`io_uring_prep_readv(3)`。我建议应用尽可能地利用 liburing 提供的帮助函数。

liburing 库仍处于起步阶段，并且正在不断开发，以扩展支持的功能和可用的帮助函数。

## 8.0 高级用例和特性

上述例子和使用场景适用于各种类型的 IO，有基于`O_DIRECT`文件的 IO、缓冲式 IO、套接字 IO等。（使用 io_uring 时）无需特别关注，即可确保它们的正确操作和异步性。不过，io_uring 的确提供了一些可供应用选择的（额外）功能。以下小节将描述其中的大多数功能。

### 8.1 固定文件和固定缓冲区

每当一个文件描述符被填入 sqe 并提交给内核时，内核必须要检索到一个对该文件的引用。一旦 IO 完成，该文件引用就会被删除。由于文件引用的原子性，在高 IOPS 的工作负载下，这可能会导致明显的减速。为了缓解这个问题，io_uring 为 io_uring 实例提供了一种预注册文件集的方法。这由一个新的系统调用完成：

```c
int io_uring_register(unsigned int fd, unsigned int opcode, void *arg, unsigned int nr_args);
```

`fd`是 io_uring 实例环的文件描述符。`opcode`代表将要注册的类型，对于注册文件集而言，需要使用`IORING_REGISTER_FILES`。`arg`必须指向应用准备打开的文件描述符的数组，并且`nr_args`必须包含该数组的长度。

一旦`io_uring_register(2)`成功注册了文件集，应用就可以将文件集数组的索引填入 sqe->fd 字段（从而取代实际的文件描述符），并且将 sqe->flags 字段设为`IOSQE_FIXED_FILE`来标记 sqe->fd 指向的是文件集索引，从而能够使用相关文件。即使已经注册了文件集，应用仍可自由地继续使用未注册的文件，这通过将 sqe->fd 改为未注册的 fd，并且不在 sqe->flags 中设置`IOSQE_FIXED_FILE`来实现。

当 io_uring 实例被移除后，注册的文件集会被自动释放，或者在`io_uring_register(2)`调用中的`opcode`字段设置`IORING_UNREGISTER_FILES`来实现（释放注册文件集）。

还可以注册一个固定 IO 缓冲区（fixed IO buffer）集合。使用`O_DIRECT`时，内核必须将应用内存页映射到内核中，用以向其执行 IO，并且在 IO 结束后取消对这些页的映射。这些操作的开销很大。如果应用复用 IO 缓冲区，就总共只需要进行一次映射和取消映射，不用每次 IO 操作都进行。

为了注册一个固定 IO 缓冲区集，需要在`io_uring_register(2)`调用中的`opcode`字段设置`IORING_REGISTER_BUFFERS`。`args`必须包含填写好每个 iovec 地址和长度的 iovec 结构体数组。`nr_args`表示 iovec 数组的长度。

在成功注册了固定 IO 缓冲区集的基础上，应用可以使用`IORING_OP_READ_FIXED`和`IORING_OP_WRITE_FIXED`来从这些缓冲区执行 IO。当使用这些固定操作码（fixed op-codes）时，sqe->addr 必须包含这些缓冲区中至少一个的地址，并且 sqe->len 必须为请求的长度（以字节为单位）。应用可能会注册比给定的 IO 操作大的缓冲区，一个固定的读/写只是一个固定缓冲区的一部分是完全合法的。

### 8.2 轮询 IO

对追求低延迟的应用，io_uring 提供了对文件的轮询 IO的支持。本文所提到的轮询，是指执行 IO 时不依赖于硬件中断来发出完成事件信号。使用轮询 IO 后，应用将反复向硬件驱动询问已提交的 IO 请求的状态。这和使用非轮询 IO 时，应用一般会休眠等待硬件中断来唤醒是不同的。对极低延迟的设备而言，轮询能够显著提高性能。对极高 IOPS 的应用而言也一样，高中断率会使非轮询的负载开销大得多。在延迟或总体 IOPS 率方面，（采用）轮询是否有意义的界限，取决于应用、IO 设备和机器的性能。

要使用 IO 轮询，需要在`io_uring_setup(2)`调用中的 io_uring_params->flags 字段设置`IORING_SETUP_IOPOLL`，或者使用 liburing 库的`io_uring_queue_init(3)`帮助函数。使用轮询后，因为不再有自动触发的异步的硬件端的完成事件，应用不再能够通过 CQ 环尾来检查可用的完成。应用必须通过调用`io_uring_enter(2)`，并在该调用中设置`IORING_ENTER_GETEVENTS`和`min_complete`，来主动查询并获取这些事件。`min_complete`代表希望获取的完成事件数量，设置`IORING_ENTER_GETEVENTS`和`min_complete` = 0 是合法的，在轮询 IO 中，这要求内核只检查（一下）驱动端（是否有）完成事件，而不必循环进行。

在使用`IORING_SETUP_IOPOLL`注册的（轮询的） io_uring 实例上，只有在轮询时有意义的操作码才能够被使用。这些操作码包括任意读/写命令：`IORING_OP_READV`、`IORING_OP_WRITEV`、`IORING_OP_READ_FIXED`和`IORING_OP_WRITE_FIXED`。在已注册为轮询的 io_uring 实例上，使用非轮询的操作码时不合法的。这么做会使`io_uring_enter(2)`返回`-EINVAL`错误码。背后的原因是，内核不知道`io_uring_enter(2)`调用使用`IORING_ENTER_GETEVENTS`设置时能否安全地休眠以等待事件（唤醒），或者它需要主动轮询事件。

### 8.3 内核侧轮询

虽然 io_uring 已经高效地通过更少的系统调用来发布和完成更多的请求，某些场景下我们可以通过进一步减少系统调用的数量来提升 IO 执行的效率。内核侧轮询就是这样的一种功能。启用该功能后，应用不再需要通过`io_uring_enter(2)`来提交 IO。当应用填写了新的 sqe 并更新了 SQ 环，内核侧将会自动发现新的（一个或多个）sqe 并且提交他们。这一切通过一个针对该 io_uring 实例的内核线程完成。

为了使用这个功能，需要在`io_uring_params->flags`字段设置`IORING_SETUP_SQPOLL`，通过调用`io_uring_setup(2)`或者传递给`io_uring_queue_init(3)`来注册 io_uring 实例。此外，如果应用希望特定 CPU 运行该线程，可以通过在`io_uring_params->flags`字段设置`IORING_SETUP_SQ_AFF`，并且在`io_uring_params->sq_thread_cpu`设置想使用的 CPU。注意，使用`IORING_SETUP_SQPOLL`设置 io_uring 实例是一种特权操作，如果用户没有足够权限，`io_uring_setup(2)`或`io_uring_queue_init(3)`会返回`-EPERM`错误码。

当 io_uring 实例不活跃时，为了避免浪费太多 CPU（性能），内核侧的线程会在空闲一段时间后自动休眠。发生这种情况时，线程将会在 SQ 环的`flags`字段设置`IORING_SQ_NEED_WAKEUP`。设置该值后，应用将无法以来内核自动查找新 sqe，它必须设置`IORING_ENTER_SQ_WAKEUP`并调用`io_uring_enter(2)`（用于重新唤醒线程）。引用侧的逻辑如下所示：

```c
    /* fills in new sqe entries */
    add_more_io();
    /*
    * need to call io_uring_enter() to make the kernel notice the new IO
    * if polled and the thread is now sleeping.
    */
    if ((*sqring→flags) & IORING_SQ_NEED_WAKEUP)
        io_uring_enter(ring_fd, to_submit, to_wait, IORING_ENTER_SQ_WAKEUP);
```

只要应用一直保持执行 IO，就不会用到`IORING_SQ_NEED_WAKEUP`，我们不需要执行任意系统调用就能够实现高效执行 IO。然而，重要的是在线程休眠时，应用程序能够保持与上述类似的逻辑（唤醒线程）。可以通过`io_uring_params->sq_thread_idle`字段来设置线程的具体闲置时间，这个值以毫秒为单位。如果没有设置该值，内核默认在将线程置为休眠状态前等待 1 秒的闲置时间。

对于“一般的”中断驱动的 IO，应用可以通过直接查看 CQ 环来找到完成事件。如果使用`IORING_SETUP_IOPOLL`注册 io_uring 实例，那么内核将会负责获取完成事件。对于两种情况（中断和轮询的选择），除非应用希望等待 IO 发生，否则它可以简单地查看 CQ 环来查找事件。

## 9.0 性能

最后，io_uring 达到了既定的设计目标。我们通过两个不同的环，获得了一个内核和应用间十分高效的传递机制。虽然在应用中正确使用原始接口需要谨慎一些，但主要的复杂之处在于需要使用显式的内存排序原语。这些可以被归结为发布和处理事件时在提交和完成方面的一些具体细节，通常在不同的应用中遵循着相同的模式。随着 liburing 的不断成熟，我希望它提供的 API 能够满足大多数应用的需求。

虽然本文的目的不在于全面细节地介绍 io_uring 的性能和可扩展性，但本节将会简要谈谈在这一领域的优势。更多的细节可以看看[[1]](https://lore.kernel.org/linux-block/20190116175003.17880-1-axboe@kernel.dk/)。注意，由于在阻塞方面的进一步优化，这些结果可能会有些过时。例如，在我的测试环境中，使用 io_uring 的每核心峰值大约为 1700K 4k IOPS，而非 1620K。注意，这些数值是不绝对的，它们大多用来衡量相应的优化。我们将不断使用 io_uring 挖掘更低的延迟和更高的峰值性能，现在应用和内核间的通信机制不再是（制约延迟和峰值性能的）瓶颈。

### 9.1 原始性能

有很多方式能够查看接口的原始性能，大多数测试会涉及内核的其他部分。上面的数字就是一个例子，我们通过随机从块设备或文件中读取来测量性能。峰值性能方面，io_uring 使用轮询获得了 1.7M 4k IOPS。aio 获得了低得多的 608K。这里的比较有点不公平，因为 aio 不支持轮询 IO。如果我们禁止使用轮询，io_uring 能够在（另一个）相同的测试用例下达到 1.2M IOPS。这方面 aio 的缺陷很明显，在相同工作负载下，io_uring 的 IOPS 是 aio 的两倍。

io_uring 还支持 no-op 命令，该命令主要用于检查接口的原始吞吐量。根据所使用的系统，观察到的消息数从 12M 每秒（我的笔记本）到 20M 每秒（用于其他引用结果的测试环境）不等。实际结果根据具体的测试案例有很大的不同，主要受必须执行的系统调用数量的约束。原始接口（性能）和内存相关，由于提交和完成的信息很小，并且在内存中线性排列，因此实现的消息速率可以非常高。

### 9.2 缓冲异步性能

我之前提到过，内核空间缓存的 aio 的实现会比用户空间中的更加高效。一个主要原因是数据缓存与否。当进行有缓冲区的 IO 时，应用通常严重依赖于内核的页缓存（kernels page cache）以获得更好的性能。使用用户空间的应用无法知道它下一个需要的数据是否已经被缓存。应用可以查询这个信息，但是这需要更多的系统调用，并且这个信息未必十分可靠——现在被缓存的数据未必在几毫秒后依然被缓存。因此，一个使用 IO 线程池的应用通常必须用异步的上下文处理请求，这导致至少两次的上下文切换。如果请求的数据已经在页缓存中，将导致性能的急剧下降。

io_uring 处理这种情况就像处理其他可能阻塞应用的资源一样。更重要的是，对于不会造成阻塞的操作，数据将以内联方式提供。这使得 io_uring 对于已经在页缓存中的（数据的） IO 来说，和常规的同步接口一样高效。一旦 IO 提交的调用返回，同时CQ 环中就会出现一个等待被应用消费的完成事件，同时数据就会已经被复制。

## 10.0 延伸阅读

鉴于（io_uring）是一个全新的接口，我们现在还没有大规模使用。截至本文写作时，带有该接口的内核正处于 -rc 阶段。即使有一个相当完整的接口描述，学习研究 io_uring 对于充分理解如何有效使用它也是很有帮助的。

一个例子是 fio [[2]](git://git.kernel.dk/fio)附带的 io_uring 引擎。除了注册文件集，它能够使用上述提到的所有高级功能。

另一个例子是也由 fio 附带的 t/io_uring.c 样本基准应用。它对文件或设备做简单的随机读取，通过可配置的设置来探索高级用例的完整功能集。

liburing 库[[3]](git://git.kernel.dk/liburing)有一套完整的系统调用接口的手册，值得一读。它还附带了一些测试程序，既有针对开发中发现的问题的单元测试，也有技术演示。

LWN也写了一篇关于 io_uring 早期阶段的精彩文章[[4]](https://lwn.net/Articles/776703/)。请注意，io_uring 在该文章写完后有了一些新变化，因此我建议在两篇文章有出入之处以本文为准。

## 11.0 参考文献

[1] https://lore.kernel.org/linux-block/20190116175003.17880-1-axboe@kernel.dk/

[2] git://git.kernel.dk/fio

[3] git://git.kernel.dk/liburing

[4] https://lwn.net/Articles/776703/

版本：0.4, 2019-10-15



> 原文链接：https://kernel.dk/io_uring.pdf

