# io_uring 系统性整理

这里有个误解，I/O模型其实是针对整个系统的所有I/O操作的，但是平时很少对文件系统使用异步读写，同步或直接映射的情况比较多。更别提多路复用了，这个机制基本只用在network中。

[lwn Kernel article index](https://lwn.net/Kernel/Index/)

## I/O 模型

- blocking I/O

![blocking](https://www.zi-c.wang/2020/11/15/io-uring-%E7%B3%BB%E7%BB%9F%E6%80%A7%E6%95%B4%E7%90%86/io-blocking.gif)

同步阻塞，直到内核收到数据返回给线程。

- nonblocking I/O

![nonblocking](https://www.zi-c.wang/2020/11/15/io-uring-%E7%B3%BB%E7%BB%9F%E6%80%A7%E6%95%B4%E7%90%86/io-nonblocking.gif)

同步不阻塞，但是如果内核没收到数据会返回一个 `EWOULDBLOCK`

- I/O multiplexing (select and poll)

![multiplex](https://www.zi-c.wang/2020/11/15/io-uring-%E7%B3%BB%E7%BB%9F%E6%80%A7%E6%95%B4%E7%90%86/io-multiplex.gif)

异步阻塞，使用selet（using select requires two system calls instead of one）、poll系统调用循环等待socket可读时，使用recvfrom收取数据。主要优势在于能够在单线程监控多个文件描述符fd。

初次之外还有epoll,使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次

优点有：
没有最大并发连接的限制，能打开的FD的上限远大于1024（1G的内存上能监听约10万个端口）。
效率提升，不是轮询的方式，不会随着FD数目的增加效率下降。只有活跃可用的FD才会调用callback函数；即Epoll最大的优点就在于它只管你“活跃”的连接，而跟连接总数无关，因此在实际的网络环境中，Epoll的效率就会远远高于select和poll。
内存拷贝，利用mmap()文件映射内存加速与内核空间的消息传递；即epoll使用mmap减少复制开销。

这种方法基本等价于 一个进程创建多个线程，每个线程维护一个blocking I/O

- signal driven I/O (SIGIO)

![multiplex](https://www.zi-c.wang/2020/11/15/io-uring-%E7%B3%BB%E7%BB%9F%E6%80%A7%E6%95%B4%E7%90%86/io-sigio.gif)

非阻塞，通过sigaction系统调用安装signal handler，当datagram数据报可读时，向I/O接收进程发送SIGIO信号，可以在signal handler里面读这个数据，然后通知main loop；也可以先通知main loop，让main loop去读这个数据。

- asynchronous I/O (the POSIX aio_functions)

![aio](https://www.zi-c.wang/2020/11/15/io-uring-%E7%B3%BB%E7%BB%9F%E6%80%A7%E6%95%B4%E7%90%86/io-aio.gif)

异步非阻塞，也是调用aio_read之后立刻返回，和SIGIO的区别是直到接收到数据并将数据传输到用户时，才产生完成信号。

### comparison

![comparison](https://www.zi-c.wang/2020/11/15/io-uring-%E7%B3%BB%E7%BB%9F%E6%80%A7%E6%95%B4%E7%90%86/io-comparison.gif)

参考:
[io models](http://www.masterraghu.com/subjects/np/introduction/unix_network_programming_v1.3/ch06lev1sec2.html)
[彻底理解 IO多路复用](https://juejin.im/post/6844904200141438984)
[聊聊IO多路复用之select、poll、epoll详解](https://www.jianshu.com/p/dfd940e7fca2)

## Asynchronous I/O

首先确定这里的AIO是内核态的，由libaio封装的系统调用运行库，而不是glibc用户态AIO，使用多线程模拟的。

linux kernel AIO的主要缺点在于项目泥潭，bug太多，项目设计和领导更换，而且实现比较复杂，直到现在只能比较稳定支持以O_DIRECT（直接映射修改，bypass page cache）方式打开文件，需要自己处理buffer、offset对其这些问题，不能用page cache层以bio的方式读写block数据。

因为使用page buffer层时涉及到block driver里面的队列，相比O_DIRECT多出很多阻塞点，因此实现起来比较令人恼火。因此这个项目根本就没实现起来。

因此io_uring的主要对比对象是多路复用和DPDK、SPDK，是一个事实上的新异步IO API

Linux AIO does suffer from a number of ailments. The subsystem is quite complex and requires explicit code in any I/O target for it to be supported.

实现不了的地方基本上都开一个kernel thread跑，感觉开销更大了。

参考：
[Linux Asynchronous I/O](https://oxnz.github.io/2016/10/13/linux-aio/)

[Fixing asynchronous I/O, again](https://lwn.net/Articles/671649/)
[Linux kernel AIO这个奇葩](https://www.aikaiyuan.com/4556.html)
[2017Toward non-blocking asynchronous I/O](https://lwn.net/Articles/724198/)

## io_uring

参考：

[Kernel Recipes 2019 - Faster IO through io_uring](https://www.youtube.com/watch?v=-5T4Cjw46ys)

[20190115Ringing in a new asynchronous I/O API](https://lwn.net/Articles/776703/)

[20200715Operations restrictions for io_uring](https://lwn.net/Articles/826053/)

[20200320Automatic buffer selection for io_uring](https://lwn.net/Articles/815491/)

[20200124The rapid growth of io_uring](https://lwn.net/Articles/810414/)

[20200511Hussain: Lord of the io_uring](https://lwn.net/Articles/820220/)

[Linux异步IO新时代：io_uring](https://kernel.taobao.org/2019/06/io_uring-a-new-linux-asynchronous-io-API/)

[20200716io_uring: add restrictions to support untrusted applications and guests](https://lwn.net/Articles/826255/)

[20200225io_uring support for automatic buffers](https://lwn.net/Articles/813311/)

[io_uring（1） – 我们为什么会需要 io_uring](https://www.byteisland.com/io_uring（1）-我们为什么会需要-io_uring/)

[linux “io_uring” 提权漏洞(CVE-2019-19241)分析](https://www.anquanke.com/post/id/200486)

[io_uring（2）- 从创建必要的文件描述符 fd 开始](https://www.byteisland.com/io_uring（2）-从创建必要的文件描述符-fd-开始/)

uring这个词没有翻译”something that looks a little less like io_urine”.

这是一个为了高速I/O提出的新的一系列系统调用，简单来说就是新的ring buffer。之前的异步I/O策略是libaio，这个机制饱受诟病，于是Jens Axboe直接提出io_uring，性能远超aio。

从5.7开始超出纯I/O的范畴，io_uring开始为一部接口提供FAST POLL机制，用户无需再像使用select、event poll等多路复用机制来监听文件句柄，只要把读写请求直接丢到io_uring的submission queue中提交 ，当文件句柄不可读写时，内核会主动添加poll handler，当文件句柄可读写时主动调用poll handler再次下发读写请求，从而减少系统调用次数提高性能

这是一个线程粒度的异步I/O机制，分为 submission queue和completion queue，在使用系统调用申请之后，直接返回可以使用mmap映射的file discriptor。

应用程序可以直接使用mmap映射的两个ring buffer直接与内核进行I/O数据传输交换，减少了大量系统调用的开销。

具体流程：

- setup `int io_uring_setup(int entries, struct io_uring_params *params);`

其中entries表示submission and completion queues两个队列的大小

param中设置两个队列和具体的状态

```
struct io_uring_params {
	__u32 sq_entries;
	__u32 cq_entries;
	__u32 flags;
	__u16 resv[10];
	struct io_sqring_offsets sq_off;
	struct io_cqring_offsets cq_off;
};
```

最终实现目的通过file descriptor与内核共享ring buffer

```
subqueue = mmap(0, params.sq_off.array + params.sq_entries*sizeof(__u32),
    PROT_READ|PROT_WRITE|MAP_SHARED|MAP_POPULATE,
             ring_fd, IORING_OFF_SQ_RING);

sqentries = mmap(0, params.sq_entries*sizeof(struct io_uring_sqe),
    PROT_READ|PROT_WRITE|MAP_SHARED|MAP_POPULATE,
		    ring_fd, IORING_OFF_SQES);


cqentries = mmap(0, params.cq_off.cqes + params.cq_entries*sizeof(struct io_uring_cqe),
      PROT_READ|PROT_WRITE|MAP_SHARED|MAP_POPULATE,
		    ring_fd, IORING_OFF_CQ_RING);
```

相关资料：
[Ringing in a new asynchronous I/O API](https://lwn.net/Articles/776703/)
[The rapid growth of io_uring](https://lwn.net/Articles/810414/)

```
#include <liburing.h>
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

int main()
{
    struct io_uring ring;
    io_uring_queue_init(32, &ring, 0);

    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    int fd = open("/home/carter/test.txt", O_WRONLY | O_CREAT);
    struct iovec iov = {
        .iov_base = "Hello world",
        .iov_len = strlen("Hello world"),
    };
    io_uring_prep_writev(sqe, fd, &iov, 1, 0);
    io_uring_submit(&ring);

    struct io_uring_cqe *cqe;

    for (;;) {
        io_uring_peek_cqe(&ring, &cqe);
        if (!cqe) {
            puts("Waiting...");
            // accept 新连接，做其他事
        } else {
            puts("Finished.");
            break;
        }
    }
    io_uring_cqe_seen(&ring, cqe);
    io_uring_queue_exit(&ring);
}
```

## 总结邮件

今天重新思考了一下IO模型，并阅读了io_uring和多路复用相关代码，感觉突然想通了。io_uring取代了AIO而不是取代了usercopy，usercopy部署的安全策略未必适用擅长传输大量数据的io_uring，这个工作可以推后再进行。具体如下。

一、将I/O模型的设计和实现分离

I/O在操作系统中含义包括与I/O设备通信和输入输出数据，I/O模型是针对第一种含义提出的解决方案。

linux在实现I/O的过程中参考了这些模型进行实现，但并没有在同一层次进行实现。例如LKM开发中定义的file_operations实际只包括 read（同步）/read_iter（异步）/mmap/poll 等函数指针，在I/O模型中的阻塞同步和非阻塞同步的情况可以通过使用read/read_iter 附加O_NONBLOCK的方式实现。

select epoll 多路复用和AIO这些I/O模型，则是分别在与之相同或不同的层次对底层函数进行封装。

例如 select 系统调用是在内核层通过vfs_poll遍历相关的file_descriptor，glibc实现的AIO是在用户空间多线程调用这些阻塞/非阻塞的同步/异步系统调用，epoll（更像是一个通知机制）是将select/poll中需要每次都传递的file descriptor都保存在内核中，减少了usercopy；通过event监听callback进行通知，减少了对fd的遍历开销。

二、思考io_uring的设计和实现

io_uring的设计借鉴了以上的优点，在内核空间通过kthread实现对阻塞读写任务的托管，并加入了zero copy特性，开发者可以通过一次系统调用唤醒线程一直向共享ringbuffer中写数据，而不是每次写数据都需要系统调用，这在内核和用户通信范畴内很大程度上减少了系统调用的次数，消除了usercopy的负担。

但无法否认io_uring是对下层file_operations的封装，下层函数又是device driver file_operations的封装（甚至对buffer I/O中间还有一层page cache、一层block layer、一层I/O schedule），因此io_uring在许多情况无法获得SPDK用户空间直通driver的性能优势。

三、对安全问题的思考

我目前理解的安全风险主要来自于usercopy造成的out-of-bound、information/pointer leakage和race情况，尤其是struct结构可能存在的函数/数据指针，但是io_uring消除掉的usercopy主要负责大量I/O数据的传输，而非带有指针的控制数据结构（io_uring中的控制数据也在用copy_*_user传输，如图），因此对安全问题的认识比我预期要复杂一些（主要问题可能是OOB和Iago攻击），需要加深对漏洞形式的理解，但好处是急迫程度下降了。

我只能继续积累漏洞阅读量提升认知水平，思考copy_*_user可以部署的安全机制和策略。

![usercopy](https://www.zi-c.wang/2020/11/15/io-uring-%E7%B3%BB%E7%BB%9F%E6%80%A7%E6%95%B4%E7%90%86/usercopy.PNG)

四、总结

我这阶段应该继续把重点放在kernel extension问题的描述上，对这个问题我已经基本有了一定想法，大致是将威胁模型定位为kernel rootkits，通过修改页表或切换地址空间构建运行时沙箱，使用可信基截获、保护gateway，使用hook方式监控driver相关函数和数据I/O。可以将性能的提升和对DPDK/SPDK使用的UIO和VFIO保护作为贡献点（这可能是这次突发奇想的意外收获），现在面临的问题是不确定相关方案是否有实现、近期driver保护方案相关只有四篇。

> 原文链接：https://www.zi-c.wang/2020/11/15/io-uring-%E7%B3%BB%E7%BB%9F%E6%80%A7%E6%95%B4%E7%90%86/
