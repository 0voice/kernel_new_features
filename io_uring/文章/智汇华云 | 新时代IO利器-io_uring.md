>  io_uring是kernel 5.1中引入的一套新的syscall接口，用于支持异步IO。随着客户在高性能计算中的求解问题规模的越来越大，对计算能力和存储IO的需求不断增长，并成为计算和存储技术发展最直接的动力。本文将对io_uring的原理和功能进行分析，让大家了解io_uring的性能以及其应用场景、发展趋势。

io_uring是kernel 5.1中引入的一套新的syscall接口，用于支持异步IO。随着客户在高性能计算中的求解问题规模的越来越大，对计算能力和存储IO的需求不断增长，并成为计算和存储技术发展最直接的动力。本文将对io_uring的原理和功能进行分析，让大家了解io_uring的性能以及其应用场景、发展趋势。

**概述**

高性能计算是继理论科学和实验科学之后科学探索的第三范式，被广泛应用在高能物理研究、核武器设计、航天航空飞行器设计、国民经济的预测和决策等领域，对国民经济发展和国防建设具有重要的价值。它作为世界高技术领域的战略制高点，已经成为科技进步的重要标志之一，同时也是一个国家科技综合实力的集中体现。

作为信创云计算专家，华云数据已经在政府金融、国防军工、教育医疗、能源电力、交通运输等十几个行业打造了行业标杆案例，客户总量超过30万。在高性能计算方面也部下重局。在华云数据的研发和客户需求调研中发现，随着客户在高性能计算中的求解问题规模的越来越大，对计算能力和存储IO的需求不断增长，并成为计算和存储技术发展最直接的动力。

io_uring是Linux Kernel在v5.1版本引入的一套新异步编程框架，同时支持Buffer IO和Direct IO。io_uring 在设计之初就充分考虑框架自身的高性能和通用性，不仅仅面向传统基于块设备的FS/Block IO领域，对网络异步编程也提供支持，以通用的系统调用提供比肩spdk/dpdk 的性能。

 **IO演变过程**

**1、基于fd的阻塞式 I/O：read()/write()**

![image](https://user-images.githubusercontent.com/87457873/149942207-3af8a5c4-0c46-43be-8247-5176d414dabc.png)

作为大家最熟悉的读写方式，Linux 内核提供了基于文件描述符的系统调用，这些描述符指向的可能是存储文件（storage file），也可能是network sockets：

ssize_t read(int fd, void *buf, size_t count);

ssize_t write(int fd, const void *buf, size_t count);

二者称为阻塞式系统调用（blocking system calls），因为程序调用这些函数时会进入sleep状态，然后被调度出去（让出处理器），直到 I/O 操作完成：

- 如果数据在文件中，并且文件内容已经缓存在page cache中，调用会立即返回；
- 如果数据在另一台机器上，就需要通过网络（例如 TCP）获取，会阻塞一段时间；
- 如果数据在硬盘上，也会阻塞一段时间。

缺点：

随着存储设备越来越快，程序越来越复杂， 阻塞式（blocking）已经这种最简单的方式已经不适用了。

**2、非阻塞式 I/O：select()/poll()/epoll()**

阻塞式之后，出现了一些新的、非阻塞的系统调用，例如select()、poll()以及更新的epoll()。应用程序在调用这些函数读写时不会阻塞，而是立即返回，返回的是一个已经ready的文件描述符列表。

![image](https://user-images.githubusercontent.com/87457873/149942253-f714b31b-f8c8-46b3-8c73-cf307fbb8d60.png)

缺点：只支持network sockets和pipes——epoll()甚至连storage files都不支持。

**3、线程池方式**

对于storage I/O，经典的解决思路是thread pool：主线程将 I/O分发给worker线程，后者代替主线程进行阻塞式读写，主线程不会阻塞。

![image](https://user-images.githubusercontent.com/87457873/149942269-bec5c706-707e-4667-a5d7-239784765786.png)

这种方式的问题是线程上下文切换开销可能非常大，通过性能压测会看到。

**4、Direct** **I/O（数据库软件）：绕过 page cache**

随后出现了更加灵活和强大的方式：数据库软件（database software）有时并不想使用操作系统的page cache，而是希望打开一个文件后，直接从设备读写这个文件（direct access to the device）。这种方式称为直接访问（direct access）或直接 I/O（direct I/O）。

需要指定O_DIRECT flag；

- 需要应用自己管理自己的缓存 —— 这正是数据库软件所希望的；
- 是zero-copy I/O，因为应用的缓冲数据直接发送到设备，或者直接从设备读取。

**5、异步IO**

随着存储设备越来越快，主线程和worker线性之间的上下文切换开销占比越来越高。现在市场上的一些设备。换个方式描述，更能让我们感受到这种开销：上下文每切换一次，我们就少一次dispatch I/O的机会。因此Linux 2.6 内核引入了异步I/O（asynchronous I/O）接口。 



![image](https://user-images.githubusercontent.com/87457873/149942297-b3711c0b-39ce-4512-9b8e-fc5221d64ab7.png)

Linux 原生AIO 处理流程：

- 当应用程序调用io_submit系统调用发起一个异步IO 操作后，会向内核的 IO 任务队列中添加一个IO 任务，并且返回成功。
- 内核会在后台处理 IO 任务队列中的IO 任务，然后把处理结果存储在 IO 任务中。
- 应用程序可以调用io_getevents系统调用来获取异步IO 的处理结果，如果 IO 操作还没完成，那么返回失败信息，否则会返回IO 处理结果。

从上面的流程可以看出，Linux 的异步 IO 操作主要由两个步骤组成：

- 调用 io_submit 函数发起一个异步 IO 操作。
- 调用 io_getevents 函数获取异步 IO 的结果。

AIO的缺陷

- 仅支持direct IO。在采用AIO的时候，只能使用O_DIRECT，不能借助文件系统缓存来缓存当前的IO请求，还存在size对齐（直接操作磁盘，所有写入内存块数量必须是文件系统块大小的倍数，而且要与内存页大小对齐。）等限制，直接影响了aio在很多场景的使用。
- 仍然可能被阻塞。语义不完备。即使应用层主观上，希望系统层采用异步IO，但是客观上，有时候还是可能会被阻塞。io_getevents(2)调用read_events读取AIO的完成events，read_events中的wait_event_interruptible_hrtimeout等待aio_read_events，如果条件不成立（events未完成）则调用__wait_event_hrtimeout进入睡眠（当然，支持用户态设置最大等待时间）。
- 拷贝开销大。每个IO提交需要拷贝64+8字节，每个IO完成需要拷贝32字节，总共104字节的拷贝。这个拷贝开销是否可以承受，和单次IO大小有关：如果需要发送的IO本身就很大，相较之下，这点消耗可以忽略，而在大量小IO的场景下，这样的拷贝影响比较大。
- API不友好。每一个IO至少需要两次系统调用才能完成（submit和wait-for-completion)，需要非常小心地使用完成事件以避免丢事件。
- 系统调用开销大。也正是因为上一条，io_submit/io_getevents造成了较大的系统调用开销，在存在spectre/meltdown（CPU熔断幽灵漏洞，CVE-2017-5754）的机器上，若如果要避免漏洞问题，系统调用性能则会大幅下降。在存储场景下，高频系统调用的性能影响较大。

软件工程中，在现有接口基础上改进，相比新开发一套接口，往往有着更多的优势，然而在过去的数年间，针对上述限制的很多改进努力都未尽如人意。终于，全新的异步IO引擎io_uring就在这样的环境下诞生了。



**io_uring原理**

io_uring 来自资深内核开发者 Jens Axboe 的想法，他在 Linux I/O stack 领域颇有研究。从最早的patch aio: support for IO polling 可以看出，这项工作始于一个很简单的观察：随着设备越来越快，中断驱动（interrupt-driven）模式效率已经低于轮询模式（polling for completions）。

- io_uring 的基本逻辑与 linux-aio 是类似的：提供两个接口，一个将 I/O 请求提交到内核，一个从内核接收完成事件。
- 但随着开发深入，它逐渐变成了一个完全不同的接口：设计者开始从源头思考 如何支持完全异步的操作。

**1、原理及核心数据结构：**

每个 io_uring 实例都有两个环形队列（ring），在内核和应用程序之间共享：

- 提交队列：submission queue (SQ)
- 完成队列：completion queue (CQ)

![image](https://user-images.githubusercontent.com/87457873/149942323-e05da76e-264d-486d-b824-0a29f7d992b3.png)

这两个队列：

- 都是单生产者、单消费者，size 是 2 的幂次；
- 提供无锁接口（lock-less access interface），内部使用 内存屏障做同步（coordinated with memory barriers）。

使用方式：

- 请求

- 应用创建 SQ entries (SQE)，更新 SQ tail；
- 内核消费 SQE，更新 SQ head。

- 完成

- 内核为完成的一个或多个请求创建 CQ entries (CQE)，更新 CQ tail；
- 应用消费 CQE，更新 CQ head。
- 完成事件（completion events）可能以任意顺序到达，到总是与特定的 SQE 相关联的。
- 消费 CQE 过程无需切换到内核态。

**2、 三种工作模式**

io_uring 实例可工作在三种模式：

**(1)中断驱动模式（interrupt driven）**

默认模式。可通过 io_uring_enter() 提交 I/O 请求，然后直接检查 CQ 状态判断是否完成。

**(2)轮询模式（polled）**

Busy-waiting for an I/O completion，而不是通过异步 IRQ（Interrupt Request）接收通知。

这种模式需要文件系统（如果有）和块设备（block device）支持轮询功能。相比中断驱动方式，这种方式延迟更低，但可能会消耗更多 CPU 资源。

目前，只有指定了 O_DIRECT flag 打开的文件描述符，才能使用这种模式。当一个读 或写请求提交给轮询上下文（polled context）之后，应用（application）必须调用 io_uring_enter() 来轮询 CQ 队列，判断请求是否已经完成。

对一个io_uring 实例来说，不支持混合使用轮询和非轮询模式。

**(3)内核轮询模式（kernel polled）**

这种模式中，会创建一个内核线程（kernel thread）来执行SQ 的轮询工作。

使用这种模式的 io_uring 实例，应用无需切到到内核态就能触发（issue）I/O 操作。通过SQ来提交SQE，以及监控CQ的完成状态，应用无需任何系统调用，就能提交和收割 I/O（submit and reap I/Os）。

如果内核线程的空闲时间超过了用户的配置值，它会通知应用，然后进入idle状态。这种情况下，应用必须调用io_uring_enter()来唤醒内核线程。如果 I/O 一直很繁忙，内核线性是不会 sleep 的。

**3、重要特性**

- IORING_SETUP_SQPOLL：创建一个内核线程进行sqe的处理（IO 提交），几乎完全消除用户态内核态上下文切换，消除spectre/ meltdown 缓解场景下对系统调用的性能影响，此特性需要额外消耗一个cpu核。
- ORING_SETUP_IOPOLL：配合blk-mq多队列映射机制，内核IO 协议栈开始真正完整支持IOpolling。
- IORING_REGISTER_FILES/ IORING_REGISTER_FILES_UPDATE/ IORING_UNREGISTER_FILES：减少fget/ fput原子操作带来的开销
- IORING_REGISTER_BUFFERS/ IORING_UNREGISTER_BUFFERS：通过提 前向内核 注册 buffer 减少 get_user_pages / unpin_user_pages 开销，应用适配存在一定难度。
- IORING_FEAT_FAST_POLL：网络编程新利器，向 epoll 等传统基于事件驱动的网络编程模型发起挑战，为用户态提供真正的异步编程 API 。
- ASYNC BUFFERED READS：更好的支持异步 buffered reads ，但当前对于异步 buffered write 的支持还不够完。



**功能对比**

**1、Synchronous I/O、 Libaio和IO_uring对比**

![image](https://user-images.githubusercontent.com/87457873/149942361-1c19bf0f-818f-4cec-9556-d42a0f3a92c3.png)

**2、io_uring和spdk的对比**

![image](https://user-images.githubusercontent.com/87457873/149942381-3a9f6d36-9991-476a-941e-8750b1e3666a.png)



**性能表现**



![image](https://user-images.githubusercontent.com/87457873/149942400-b45feae5-dfb8-4b7f-94f0-ee9ce3604b3d.png)

非polling模式，io_uring相比libaio提升不是很明显；在polling模式下，io_uring能与spdk接近，甚至在queue depth较高时性能更好，完爆libaio。

![image](https://user-images.githubusercontent.com/87457873/149942424-17bb8dea-5bef-4b06-9d31-5a8d2f559779.png)

在queue depth较低时有约7%的差距，但在queue depth较高时基本接近。

![image](https://user-images.githubusercontent.com/87457873/149942440-4924df96-279e-4858-861f-be93aabc00f3.png)

以上数据来自来自Jens Axboe的测试：https://lore.kernel.org/linux-block/20190116175003.17880-1-axboe@kernel.dk/

结论：

io_uring在非polling模式下，相比libaio，性能提升不是非常显著。

io_uring在polling模式下，性能提升显著，与spdk接近，在队列深度较高时性能更好。

**io_uring在kvm-qemu下的性能表现**

**1、直接启用io_uring的场景**

下面测试结果在一台华云超融合一体机测试，qemu 5.0 已经支持了Asynchronous I/O的io_uring，只需要将虚拟机将驱动改成io_uring。

原理如下：



![image](https://user-images.githubusercontent.com/87457873/149942467-a7b1e4f6-e6e6-430a-b2f3-5ea67e18005c.png)



- guestOS的kernel和qemu通信通过 virtqueue
- io_uring通过SQ和CQ进行IO的管理

宿主机环境信息：

![image](https://user-images.githubusercontent.com/87457873/149942487-2a429103-f3b6-4f94-b273-0c50319b8399.png)

虚拟机环境信息

![image](https://user-images.githubusercontent.com/87457873/149942499-6d91b523-daf2-4abd-852c-e1cb5017a7e2.png)

测试参数

```
[global]
ioengine=libaio
time_based
direct=1
thread
group_reporting
randrepeat=0
norandommap
numjobs=4
ramp_time=10
runtime=600
filename=/dev/sdb

[randwrite-4k-io32]
bs=4k
iodepth=32
rw=randwrite
stonewall

[randread-4k-io32]
bs=4k
iodepth=32
rw=randread
stonewall
```

测试数据如下：

![image](https://user-images.githubusercontent.com/87457873/149942516-0ce842ca-85e8-4ee1-8b4f-6a0fe9adb8f8.png)

总结：

1）、libaio和io_uring性能比较，io_uring性能提高不明显，没有跨越式的提升。

2）、目前io_uring 的virtio block开发还在持续进行，poll模式等高级功能没有release。很值得期待。具体参见：https://wiki.qemu.org/Features/IOUring

**2、io passthrough场景**

原理：

![image](https://user-images.githubusercontent.com/87457873/149942524-a776e41b-6545-4107-b18a-fcec4b53f9de.png)

直接透传QEMU Block层

- io_uring的SQ和CQ都在GuestOS的kernel中
- qemu的virtio-blk修改了“fast path”
- 启用了polling模式

测试环境版本信息：

![image](https://user-images.githubusercontent.com/87457873/149942562-a724ac3f-f0b6-4484-b99e-921524186101.png)

创建虚拟机

aio=io_uring -device virtio-blk-pci,drive=hd1,bootindex=2,io-uring-pt=on

测试结果：

![image](https://user-images.githubusercontent.com/87457873/149942598-6875fb40-8bd6-4143-a71a-d8f04f99b43e.png)

测试数据来自：

https://static.sched.com/hosted_files/kvmforum2020/9c/KVMForum_2020_io_uring_passthrough_Stefano_Garzarella.pdf

**总结：**

1）、virtio-blk的性能只有裸设备性能的一半左右。

2）、io_uring passthrough的性能和裸设备持平。

3）、缺点需要在guestOS的kernel中打patch。



**社区支持情况**

**1、内核支持情况**

![image](https://user-images.githubusercontent.com/87457873/149942648-62ae7e45-f25a-484c-96e2-aef574cdd63b.png)

**2、io_uring 社区开发现状**

当前 io_uring 社区开发主要聚焦在以下几个方面。

- **io_uring 框架自身的迭代优化**
  前面提到，io_uring 作为一种新型高性能异步编程框架，当前仍处于快速发展中。因此随着特性越来越丰富，以及各种稳定性问题的修复等等，代码也变得越来越臃肿。因此 io_uring 框架也需要同步”进化“，不断进行代码重构优化。
  通常异步编程一般是将工作交由线程池来做，但对于 io_uring 来说，这只是最坏的 slow path，例如异步读优化，就是想尝试尽量在当前上下文中处理。另外，在 sqpoll 模式下，io_uring 接管用户提交的系统调用，一个系统调用的执行与特定的进程上下文相关，因此 io_uring 需要维护系统调用进程的内存上下文，文件系统上下文等大量信息。同时 io_uring 作为一种框架，框架本身的开销应该尽可能的小，才能与用户态高性能框架 SPDK 对标，因此需要持续优化。
- **特性增强**

- io_uring buffer registration 特性增强
  io_uring 的 buffer registration 特性可以用来减少读写操作时频繁的 get_user_pages()/unpin_user_pages() 调用，get_user_pages() 主要用来实现用户态虚拟地址到内核页的转换，会消耗一定的 cpu 资源。来自 Oracle 的同学这组patchset 让 buffer registration 特性支持更新和共享操作，更加方便用户态编程，目前已发到第三版。我们遇到一些业务也明确提出过需要 buffer registration 支持更新操作。
- fs/userfaultfd: support iouring and polling
- 来自 VMware 的同学这组 patchset 使得 userfaultfd 支持 io_uring，且支持 polling 模式。在RDMA，持久内存等高速场景，系统调用用户态内核态上下文切换的开销在 userfualtfd 整体开销的占比会非常突出，支持 io_uring 后，可以显著减少用户态内核态上下文切换，带来明显的性能提升。
- add io_uring with IOPOLL support in scsi layer
- io_uring 出现后，Linux 内核才真正完整地支持 iopoll，iopoll 相比于中断模式能带来明显的性能提升。此前只有nvme driver 对 iopoll 提供比较好的支持，现在 scsi 层也开始准备支持 iopoll。
- no-copy bvec
- 在用 io_uring 进行 IO 栈性能分析时，来自 Facebook 的 Pavel Begunkov 发现在 direct IO 场景下是不需要拷贝bvec 结构的，他提交的这组 patchset 可以进一步提高内核 direct IO 的性能。
- io_uring and Optane2
- block 社区已经开始重视提高 IO 栈性能，一种思路是利用高性能设备，借助 io_uring 压测 IO 栈，从而发现软件性能瓶颈。比如 Intel提供最新的 Gen2 Optane SSD 给 block / io_uring 的维护者 Jens Axboe 来做内核 IO 栈性能分析。
- fs: Support for LOOKUP_NONBLOCK / RESOLVE_NONBLOCK
- Jens Axboe 优化 vfs 的文件查询逻辑，从而可以使 io_uring 的 IORING_OP_OPENAT 操作效率更高。

- **IO 栈协同优化**
  借助 io_uirng 这一高性能异步编程框架，能比较容易地发现其他内核子系统的软件性能瓶颈，从而对其进行优化，以提高内核 IO 栈的整体性能。我们之前也基于这个思路发现了 block 层的几处优化，同时提出了 device mapper polling 打通的 RFC。



**总结**

1、华云数据产品技术中心始终贴近用户，解决用户的实际问题，同时时刻关注业界的先进技术。得益于io_uring的优异的表现，华云数据在此方面投入大量资源。同时我们在RDMA、SPDK、vdpa、DPU、io_uring、eBPF等热门技术也有相关的研究及产品化的丰富经验，同时对社区不断的共享。

2、io_uring 是完全为性能而生的新一代 native async IO 模型，比 libaio 高级不少。通过全新的设计，共享内存，IO 过程不需要系统调用，由内核完成 IO的提交， 以及Polled IO机制，实现了高IOPS。相比kernel bypass，这种原生内核的方式显得友好一些。一定程度上，io_uring代表了Linux kernel block层的未来，至少从中我们可以看出一些block层进化的方向，而且我们看到io_uring也在快速发展，相信未来io_uring会更加的高效、易用，并拥有更加丰富的功能和更强的扩展能力。这让我们充满期待，NVMe的存储时代需要一个属于自己的高速IO引擎。

3、在kvm方面，qemu支持io_uring，在操作和运维方便比spdk的vhost简单很多。虽然支持的还不是很完善，但社区仍然在积极推进。希望这个早日release。

4、国内阿里在io_uring走的比较靠前，阿里有个专门的内核组在做相关的开发。OpenAnolis已经将io_uring纳入，并且在io_uring方面做了很多探索。通过社区的一些文章可以看到，阿里在数据库、web、echo_sever等相关的领域都已经应用了io_uring。

> 原文链接：https://www.huayun.com/news/1521.html

