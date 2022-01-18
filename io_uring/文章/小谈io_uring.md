## 什么是io_uring？

在 Linux 底下操作 IO 有以下方式:

- `read` 系列
- `pread`
- `preadv`

但他们都是synchronous 的，所以POSIX 有实做`aio_read`，但其乏善可陈且效能欠佳。

事实上Linux 也有自己的native async IO interface，但是包含以下缺点:

1. async IO 只支援`O_DIRECT`(or un-buffered) accesses -> file descriptor 的设定
2. IO submission 伴随104 bytes 的data copy (for IO that's supposedly zero copy)，此外一次IO 必须呼叫两个system call (submit + wait-for-completion)
3. 有很多可能会导致async 最后变成blocking (如: submission 会block 等待meta data; request slots 如果都被占满， submission 亦会block 住)

Jens Axboe 一开始先尝试改写原有的native aio，但是失败收场，因此他决定提出一个新的interface，包含以下目标(越后面越重要):

1. 易于理解和直观的使用
2. Extendable，除了 block oriented IO，networking 和 non-block storage 也要能用
3. 效率
4. 可扩展性

在设计io_uring 时，为了避免过多data copy，Jens Axboe 选择透过shared memory 来完成application 和kernel 间的沟通。其中不可避免的是同步问题，使用single producer and single consumer ring buffer 来替代shared lock 解决shared data 的同步问题。而这沟通的管道又可分为submission queue (SQ) 和completion queue (CQ)。

以CQ 来说，kernel 就是producer，user application 就是consumer。SQ 则是相反。

## 链接请求

CQEs can arrive in any order as they become available。(举例: 先读在HDD 上的A.txt，再读SSD 上的B.txt，若限制完成顺序的话，将会影响到效能)。事实上，也可以强制ordering (see [example](https://kernel.dk/io_uring.pdf) )

liburing 预设会非循序的执行submit queue 上的operation，但是有些特别的情况，我们需要这些operation 被循序的执行，如：`write`+ `close`。所以我们可以透过添加 `IOSQE_IO_LINK` 来达到效果。详细用法可参考[linking request](https://unixism.net/loti/tutorial/link_liburing.html)

## 提交队列轮询

liburing 可以透过设定flag:`IORING_SETUP_SQPOLL`切换成poll 模式，这个模式可以避免使用者一直呼叫`io_uring_enter`(system call)。此模式下，kernel thread 会一直去检查submission queue 上是否有工作要做。详细用法可参考[Submission Queue Polling](https://unixism.net/loti/tutorial/sq_poll.html#sq-poll)

值得注意的是kernel thread 的数量需要被控制，否则大量的CPU cycle 会被k-thread 占据。为了避免这个机制，liburing 的kthread 在一定的时间内没有收到工作要做，kthread 就会sleep，所以下一次要做submission queue 上的工作就需要走原本的方式:`io_uring_enter()`

使用 liburing 时，您永远不会直接调用 io_uring_enter() 系统调用。这通常由 liburing 的 io_uring_submit() 函数处理。它会自动确定您是否使用轮询模式，并处理您的程序何时需要调用 io_uring_enter() 而无需您费心。

## 内存排序

如果要直接使用liburing 就不用管这个议题，但是如果是要操作raw interface，那这个就很重要。提供两种操作:

1. `read_barrier()`：确保在进行后续内存读取之前可以看到之前的写入
2. `write_barrier()`：在先前的写入之后订购此写入

> 内核将在读取 SQ 环尾之前包含一个 read_barrier()，以确保来自应用程序的尾部写入可见。从 CQ 环的角度来看，由于消费者/生产者的角色是相反的，应用程序只需要在读取 CQ 环尾之前发出 read_barrier() 以确保它看到内核所做的任何写入。

## 解放图书馆

- 不再需要样板代码来设置 io_uring 实例
- 为基本用例提供简化的 API。

### 高级用例和功能

#### 固定文件和缓冲区

#### 轮询 IO

I/O 依靠硬件中断来发出完成事件的信号。当 IO 被轮询时，应用程序将反复询问硬件驱动程序提交的 IO 请求的状态。

[注][真实轮询示例](https://unixism.net/loti/tutorial/sq_poll.html)
[注] 提交队列轮询仅与固定文件（非固定缓冲区）结合使用

#### 内核端轮询

会有kernel thread 主动侦测SQ 上是否有东西，这样可以避免呼叫syscall: `io_uring_enter`

## 原始程式码

### `io_uring_setup`

基本的设定。我们关注的是setup 时需要设定哪些关于 `IORING_SETUP_SQPOLL` 的操作，预期找到kthread 的建立，kthread 的工作内容等等。从 `io_sq_offload_create` 可知offload 和kthread 有关。

往里面看可以找到[create_io_thread](https://github.com/torvalds/linux/blob/65090f30ab791810a3dc840317e57df05018559c/kernel/fork.c#L2444)，透过 `copy_process` 达到fork，搭配 `wake_up_new_task` 启动process。该process 要做的事为[io_sq_thread](https://github.com/torvalds/linux/blob/master/fs/io_uring.c#L6882)，

### `io_uring_enter`

[io_uring_enter](https://github.com/torvalds/linux/blob/master/fs/io_uring.c#L9332)在prepare 完write/read 之类的operation 后会被呼叫，这里我们只关注在poll 模式下的行为:

```
if (ctx->flags & IORING_SETUP_SQPOLL) {
    io_cqring_overflow_flush(ctx, false);

    ret = -EOWNERDEAD;
    if (unlikely(ctx->sq_data->thread == NULL))
        goto out;
    if (flags & IORING_ENTER_SQ_WAKEUP)
        wake_up(&ctx->sq_data->wait);
    if (flags & IORING_ENTER_SQ_WAIT) {
        ret = io_sqpoll_wait_sq(ctx);
        if (ret)
            goto out;
    }
    submitted = to_submit;
} else if ...
```

1. 若 `kthread` 闲置太久，为了避免霸占CPU，所以会主动sleep，所以若看到flag:`IORING_ENTER_SQ_WAKEUP`设起，就必须要唤醒kthread。
2. [PATCH：为 SQPOLL SQ 环等待提供 IORING_ENTER_SQ_WAIT](https://www.spinics.net/lists/io-uring/msg04097.html)

## 安装 liburing

1. 下载source [code](https://github.com/axboe/liburing/releases)
2. 。/配置
3. 须藤使安装
4. 编译示例： gcc -Wall -O2 -D_GNU_SOURCE -o io_uring-test io_uring-test.c -luring

## 自由流动

io_uring_queue_init -> alloc iov -> io_uring_get_sqe -> io_uring_prep_readv -> io_uring_sqe_set_data -> io_uring_submit

io_uring_wait_cqe -> io_uring_cqe_get_data -> io_uring_cqe_seen -> io_uring_queue_exit

## liburing/io_uring API

### i_uring

1. io_uring_setup（u32 个条目，结构 io_uring_params *p）

- 描述：建立一个提交队列（SQ）和完成队列（CQ）至少有条目条目，并返回一个文件描述符，该文件描述符可用于对io_uring实例执行后续操作。

- 关系：通过 liburing 函数包装： `io_uring_queue_init`

- 标志：成员 

  ```
  struct io_uring_params
  ```

  - IORING_SETUP_IOPOLL：
    - 忙于等待 I/O 完成
    - 提供更低的延迟，但可能比中断驱动的 I/O 消耗更多的 CPU 资源
    - 仅可用于使用 O_DIRECT 标志打开的文件描述符
    - 在 io_uring 实例上混合和匹配轮询和非轮询 I/O 是非法的
  - IORING_SETUP_SQPOLL：
    - 设置后创建内核线程进行提交队列轮询
    - `IORING_SQ_NEED_WAKEUP`如果内核线程空闲超过`sq_thread_idle`毫秒，将设置标志
    - `io_uring_register`在 linux 5.11 之前，应用程序必须通过操作`IORING_REGISTER_FILES`码注册一组用于 IO 的文件
  - IORING_SETUP_SQ_AFF：
    - poll线程将绑定到cpu设置的`sq_thread_cpu`字段`struct io_uring_params`

- 如果未指定标志，则为中断驱动的 I/O 设置 io_uring 实例

1. io_uring_register(unsigned int fd, unsigned int opcode, void *arg, unsigned int nr_args)

- 操作码：
  - IORING_REGISTER_BUFFERS：
    - uffers 被映射到内核并有资格进行 I/O
    - 使用它们，应用程序必须在提交队列条目中指定`IORING_OP_READ_FIXED`或`IORING_OP_WRITE_FIXED`操作码，并将该`buf_index`字段设置为所需的缓冲区索引
  - IORING_REGISTER_FILES：
    - 描述：I/O 的寄存器文件。包含指向文件描述符`arg`数组的指针`nr_args`
    - 使用它们，`IOSQE_FIXED_FILE`flag 必须设置在 的 flags 成员中`struct io_uring_sqe`，并且该`fd`成员设置为文件描述符数组中文件的索引
  - IORING_REGISTER_EVENTFD：
    - 可以用于`eventfd()`获取 io_uring 实例上的完成事件的通知。

1. io_uring_enter(unsigned int fd, unsigned int to_submit, unsigned int min_complete, unsigned int flags, sigset_t *sig)

- 说明：单次调用既可以提交新的 I/O，也可以等待本次调用或之前调用的 I/O 完成 `io_uring_enter()`
- 标志：
  - IORING_ENTER_GETEVENTS：
    - `min_complete`在返回之前等待指定数量的事件
    - 可以与 to_submit 一起设置为在单个系统调用中提交和完成事件
  - IORING_ENTER_SQ_WAKEUP
  - IORING_ENTER_SQ_WAIT
- 操作码：
  - IORING_OP_READV
  - IORING_OP_WRITEV
- 一些细节：
  - 如果 io_uring 实例被配置为轮询，通过`IORING_SETUP_IOPOLL`在对 的调用中指定`io_uring_setup()`，则`min_complete`含义略有不同。传递值 0 指示内核返回任何已经完成的事件，而不会阻塞。如果`min_complete`是一个非零值，如果有任何完成事件可用，内核仍然会立即返回。如果没有可用的事件完成，则调用将轮询直到一个或多个完成变得可用，或者直到进程超过其调度程序时间片。

### 解放

## io_uring 与 epoll

### io_uring 比 epoll 慢？

- [解放 #189](https://github.com/axboe/liburing/issues/189)
- [解放 #215](https://github.com/axboe/liburing/issues/215)
- [网络 #10622](https://github.com/netty/netty/issues/10622)

## 参考

- [约灵之主](https://unixism.net/loti/what_is_io_uring.html)
- [使用 io_uring 实现高效 IO - 官方 pdf](https://kernel.dk/io_uring.pdf)
- [i_uring](https://unixism.net/2020/04/io-uring-by-example-part-1-introduction/)
- [解放例子](https://unixism.net/loti/tutorial/index.html)
- [解放网络服务器](https://github.com/shuveb/loti-examples/blob/master/webserver_liburing.c)

> 原文链接：https://hackmd.io/@sysprog/iouring#What-is-io_uring

