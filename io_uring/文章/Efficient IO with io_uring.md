### 0x00 引言

io uring是Linux上面新的异步IO机制。在io uring之前，Linux下面的异步IO机制是aio，不过aio存在不少的问题。io uring一出现就受到了比较大的关注，这也可能多亏了aio衬托地好吧¯_(ツ)_/¯。io uring的核心是三个系统调用，直接使用这三个系统调用的话会比较麻烦。所以作者还额外添加了一个库liburing，方便使用。

### 0x01 liburing

liburing是为了简化io uring的使用的一个库，估计后面使用io uring的话，使用到这个库的机会还是很大的。liburing中，一个核心的结构是struct io_uring，

```c
struct io_uring {
	struct io_uring_sq sq;
	struct io_uring_cq cq;
	unsigned flags;
	int ring_fd;
};
// 其中的两个struct结构如下
struct io_uring_sq {
	unsigned *khead;
	unsigned *ktail;
	unsigned *kring_mask;
	unsigned *kring_entries;
	unsigned *kflags;
	unsigned *kdropped;
	unsigned *array;
	struct io_uring_sqe *sqes;

	unsigned sqe_head;
	unsigned sqe_tail;

	size_t ring_sz;
	void *ring_ptr;
};

struct io_uring_cq {
	unsigned *khead;
	unsigned *ktail;
	unsigned *kring_mask;
	unsigned *kring_entries;
	unsigned *koverflow;
	struct io_uring_cqe *cqes;

	size_t ring_sz;
	void *ring_ptr;
};
```

可以看出后面两个结构的前面四个字段和后面两个字段是相同的。初始化struct io_uring的时候，使用下面的第一个函数。这个函数的作用就如名字表现的一些，是io uring相关队列的初始化。实际上在这个函数的实现中，会调用多个mmap来初始化一些内存。在后面会分析。在初始化完成之后，为了提交IO请求，需要获取里面queue的一个项。使用下面的第二个函数。在获取到了空闲的项之后，使用下面的第三、四个函数初始化读、写请求。这两个函数除了第一个参数是struct io_uring_sqe之外，和preadv、pwritev是差不多的。在准备完成之后，使用下面的第五个函数提交请求。在提交了IO请求的时候，可以通过下面的第六、七个函数获取请求完成的情况，这两个函数的区别是一个非阻塞和一个阻塞的区别。默认情况下吗，完成的IO请求还会存在内部的队列中需要通过io_uring_cqe_seen表标记完成操作。使用完成之后要通过io_uring_queue_exit来完成资源清理的工作。在http://git.kernel.dk/cgit/liburing/tree/examples/io_uring-test.c 这里有一个简单的例子。

```c
extern int io_uring_queue_init(unsigned entries, struct io_uring *ring, unsigned flags);
extern struct io_uring_sqe *io_uring_get_sqe(struct io_uring *ring);
void io_uring_prep_readv(struct io_uring_sqe *sqe, int fd,
				       const struct iovec *iovecs, unsigned nr_vecs, off_t offset)；
void io_uring_prep_writev(struct io_uring_sqe *sqe, int fd,
					const struct iovec *iovecs, unsigned nr_vecs, off_t offset);
extern int io_uring_submit(struct io_uring *ring);
extern int io_uring_peek_cqe(struct io_uring *ring, struct io_uring_cqe **cqe_ptr);
extern int io_uring_wait_cqe(struct io_uring *ring, struct io_uring_cqe **cqe_ptr);
/*
 * Must be called after io_uring_{peek,wait}_cqe() after the cqe has
 * been processed by the application.
 */
static inline void io_uring_cqe_seen(struct io_uring *ring, struct io_uring_cqe *cqe);
extern void io_uring_queue_exit(struct io_uring *ring);
```

### 0x02 系统调用

io uring相关的三个syscall以及几个习惯的struct结构如下。前面的io_uring_queue_init中的操作主要就是io_uring_setup，然后再初始化struct io_uring_sq sq 和 struct io_uring_cq cq的内存，另外还会分配一个struct io_uring_sqe的数组。这里mmap的时候，使用了IORING_OFF_SQ_RING、IORING_OFF_SQES和IORING_OFF_CQ_RING这个预先定义的offset，用于告诉相同要分配的是io uring相关的一些内存。这些内存在使用io_uring_setup的时候内核会分配好，但是在用户空间要使用这些内存的话，需要将这些内存映射到用户空间，这里就利用mmap，并添加了几个offset用于实现io uring要求的映射。这些offset值定义了保存到这个三个结构保存到位置。

```c
int io_uring_setup(unsigned entries, struct io_uring_params *p);
int io_uring_enter(unsigned fd, unsigned to_submit, unsigned min_complete, unsigned flags, sigset_t *sig);
int io_uring_register(int fd, unsigned int opcode, const void *arg, unsigned int nr_args);

#define IORING_OFF_SQ_RING		0ULL
#define IORING_OFF_CQ_RING		0x8000000ULL
#define IORING_OFF_SQES			0x10000000ULL

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

/*
 * Filled with the offset for mmap(2)
 */
struct io_sqring_offsets {
	__u32 head;
	__u32 tail;
	__u32 ring_mask;
	__u32 ring_entries;
	__u32 flags;
	__u32 dropped;
	__u32 array;
	__u32 resv1;
	__u64 resv2;
};
struct io_cqring_offsets {
	__u32 head;
	__u32 tail;
	__u32 ring_mask;
	__u32 ring_entries;
	__u32 overflow;
	__u32 cqes;
	__u64 resv[2];
};

/*
 * IO submission data structure (Submission Queue Entry)
 */
struct io_uring_sqe {
	__u8	opcode;		/* type of operation for this sqe */
	__u8	flags;		/* IOSQE_ flags */
	__u16	ioprio;		/* ioprio for the request */
	__s32	fd;		/* file descriptor to do IO on */
	__u64	off;		/* offset into file */
	__u64	addr;		/* pointer to buffer or iovecs */
	__u32	len;		/* buffer size or number of iovecs */
	union {
		__kernel_rwf_t	rw_flags;
		__u32		fsync_flags;
		__u16		poll_events;
		__u32		sync_range_flags;
		__u32		msg_flags;
	};
	__u64	user_data;	/* data to be passed back at completion time */
	union {
		__u16	buf_index;	/* index into fixed buffers, if used */
		__u64	__pad2[3];
	};
};
/*
 * IO completion data structure (Completion Queue Entry)
 */
struct io_uring_cqe {
	__u64	user_data;	/* sqe->data submission passed back */
	__s32	res;		/* result code for this event */
	__u32	flags;
};

#define IORING_ENTER_GETEVENTS	(1U << 0)
```

在初始化完成之后，用户空间就可以使用这些队列来添加IO请求。为了提交这些请求到内核执行，这里就要使用`int io_uring_enter(unsigned fd, unsigned to_submit, unsigned min_complete, unsigned flags, sigset_t *sig) `syscall。这里有两个相关的flags，如果设置了IORING_ENTER_GETEVENTS的flags，这个调用就会一直等到至少有min_complete个请求完成才会返回。如果在io_uring_setup的时候设置了IORING_SETUP_SQPOLL的flag，则系统会使用额外的一个线程来执行轮询的操作，这个线程可以运行在指定的CPU核心上面，io_uring_params中的 sq_thread_cpu 和sq_thread_idle连个参数可以对轮询的CPU和时间进行一些设置。用于内核和运用直接使用了同一块内存来处理，这里实际上可以看作一种特定的共享内存，所以在设置SQ提及新的IO请求内核里面可以直接看到，不需要经过系统调用。另外的IORING_SQ_NEED_WAKEUP可以在一些时候唤醒休眠中的轮询线程。如果在io_uring_setup的时候设置了IORING_SETUP_SQPOLL的flag，则执行完成的线程可以直接通过访问相关的队列就可以获取到，不需要经过系统调用。另外io uring还包含了其它的一些高级特性。

### 0x03 内部实现

Kernel中io uring表示一个io uring的实例的结构是io uring ctx。其中两个主要的部分是sq ring和cq ring。分别代表了提交的请求的ring和已经完成的请求返回结构的ring。io_sq_ring和io_cq_ring的前面是一些控制字段和一些其它的数据，尾部是一个分配的时候变长的一块内存。io_sq_ring和io_cq_ring不同的一个地方就是io_cq_ring后面保存的直接是一个struct io_uring_cqe 的数组，前面的struct io_uring保存了其中被使用的entry的访问。而io_sq_ring保存的是一个offset index的数组，里面的保存的信息才最终定位到对应的struct io_uring_sqe，这些struct io_uring_sqe被保存到另外的一块内存中。从直觉上来说，sq和cq这里的结构会是相同的。但是这里用了不同的方式，sq加了一层的间接，从网上能找到一些信息，但不是很理解。个人猜测是不同的请求完成的时候不同，而这样进行一段时间之后，sqe里面可用的entry就是不连续的，通过一个index数据间接的方式有可用当作是连续的使用。这两个数据在前面使用mmap分配内存的时候，对应到了不同的offset，即前面IORING_OFF_SQ_RING、IORING_OFF_CQ_RING和IORING_OFF_SQES的预定于的值。

```c
struct io_sq_ring {
	/*
	 * Head and tail offsets into the ring; the offsets need to be
	 * masked to get valid indices.
	 *
	 * The kernel controls head and the application controls tail.
	 */
	struct io_uring		r;
	/*
	 * Bitmask to apply to head and tail offsets (constant, equals
	 * ring_entries - 1)
	 */
	u32			ring_mask;
	/* Ring size (constant, power of 2) */
	u32			ring_entries;
	/*
	 * Number of invalid entries dropped by the kernel due to
	 * invalid index stored in array
	 *
	 * Written by the kernel, shouldn't be modified by the
	 * application (i.e. get number of "new events" by comparing to
	 * cached value).
	 *
	 * After a new SQ head value was read by the application this
	 * counter includes all submissions that were dropped reaching
	 * the new SQ head (and possibly more).
	 */
	u32			dropped;
	/*
	 * Runtime flags
	 *
	 * Written by the kernel, shouldn't be modified by the
	 * application.
	 *
	 * The application needs a full memory barrier before checking
	 * for IORING_SQ_NEED_WAKEUP after updating the sq tail.
	 */
	u32			flags;
	/*
	 * Ring buffer of indices into array of io_uring_sqe, which is
	 * mmapped by the application using the IORING_OFF_SQES offset.
	 *
	 * This indirection could e.g. be used to assign fixed
	 * io_uring_sqe entries to operations and only submit them to
	 * the queue when needed.
	 *
	 * The kernel modifies neither the indices array nor the entries
	 * array.
	 */
	u32			array[];
};

/*
 * This data is shared with the application through the mmap at offset
 * IORING_OFF_CQ_RING.
 *
 * The offsets to the member fields are published through struct
 * io_cqring_offsets when calling io_uring_setup.
 */
struct io_cq_ring {
	/*
	 * Head and tail offsets into the ring; the offsets need to be
	 * masked to get valid indices.
	 *
	 * The application controls head and the kernel tail.
	 */
	struct io_uring		r;
	/*
	 * Bitmask to apply to head and tail offsets (constant, equals
	 * ring_entries - 1)
	 */
	u32			ring_mask;
	/* Ring size (constant, power of 2) */
	u32			ring_entries;
	/*
	 * Number of completion events lost because the queue was full;
	 * this should be avoided by the application by making sure
	 * there are not more requests pending thatn there is space in
	 * the completion queue.
	 *
	 * Written by the kernel, shouldn't be modified by the
	 * application (i.e. get number of "new events" by comparing to
	 * cached value).
	 *
	 * As completion events come in out of order this counter is not
	 * ordered with any other data.
	 */
	u32			overflow;
	/*
	 * Ring buffer of completion events.
	 *
	 * The kernel writes completion events fresh every time they are
	 * produced, so the application is allowed to modify pending
	 * entries.
	 */
	struct io_uring_cqe	cqes[];
};


struct io_ring_ctx {
	struct {
		struct percpu_ref	refs;
	} ____cacheline_aligned_in_smp;

	struct {
		unsigned int		flags;
		bool			compat;
		bool			account_mem;

		/* SQ ring */
		struct io_sq_ring	*sq_ring;
		unsigned		cached_sq_head;
		unsigned		sq_entries;
		unsigned		sq_mask;
		unsigned		sq_thread_idle;
		struct io_uring_sqe	*sq_sqes;

		struct list_head	defer_list;
	} ____cacheline_aligned_in_smp;

	/* IO offload */
	struct workqueue_struct	*sqo_wq;
	struct task_struct	*sqo_thread;	/* if using sq thread polling */
	struct mm_struct	*sqo_mm;
	wait_queue_head_t	sqo_wait;
	struct completion	sqo_thread_started;

	struct {
		/* CQ ring */
		struct io_cq_ring	*cq_ring;
		unsigned		cached_cq_tail;
		unsigned		cq_entries;
		unsigned		cq_mask;
		struct wait_queue_head	cq_wait;
		struct fasync_struct	*cq_fasync;
		struct eventfd_ctx	*cq_ev_fd;
	} ____cacheline_aligned_in_smp;

	/*
	 * If used, fixed file set. Writers must ensure that ->refs is dead,
	 * readers must ensure that ->refs is alive as long as the file* is
	 * used. Only updated through io_uring_register(2).
	 */
	struct file		**user_files;
	unsigned		nr_user_files;

	/* if used, fixed mapped user buffers */
	unsigned		nr_user_bufs;
	struct io_mapped_ubuf	*user_bufs;

	struct user_struct	*user;

	struct completion	ctx_done;

	struct {
		struct mutex		uring_lock;
		wait_queue_head_t	wait;
	} ____cacheline_aligned_in_smp;

	struct {
		spinlock_t		completion_lock;
		bool			poll_multi_file;
		/*
		 * ->poll_list is protected by the ctx->uring_lock for
		 * io_uring instances that don't use IORING_SETUP_SQPOLL.
		 * For SQPOLL, only the single threaded io_sq_thread() will
		 * manipulate the list, hence no extra locking is needed there.
		 */
		struct list_head	poll_list;
		struct list_head	cancel_list;
	} ____cacheline_aligned_in_smp;

	struct async_list	pending_async[2];

#if defined(CONFIG_UNIX)
	struct socket		*ring_sock;
#endif
};
```

cqe是内核写应用读的部分，而sqe是应用写内核读的部分。io uring setup的系统调用就是初始化相关数据结构的操作。如果省略很多的错误检查、权限检查和资源配额检查等等的部分，整体的操作还是比较简单的。liburing中初始化的一个函数如下，调用io_uring_setup函数的时候，传入的io_uring_params参数中，一部分是用户设置传入的，还有很大的一部分部分都是在内核中io uring create操作中设置的，之后设置好的struct io_uring_params会被传到用户空间，用户空间根据这些数据来使用mmap分配内存，初始化一些数据结构，具体可以参看http://git.kernel.dk/cgit/liburing/tree/src/setup.c 这里，

```c
int io_uring_queue_init(unsigned entries, struct io_uring *ring, unsigned flags) {
	struct io_uring_params p;
	int fd, ret;

	memset(&p, 0, sizeof(p));
	p.flags = flags;

	fd = io_uring_setup(entries, &p);
	if (fd < 0)
		return -errno;

	ret = io_uring_queue_mmap(fd, &p, ring);
	if (ret)
		close(fd);

	return ret;
}
```

内核中io_uring_enter的部分的代码如下，这里省略了一些检查的代码。这里整体的思路比较清晰。第一种情况，flags设置了IORING_SETUP_SQPOLL参数，这样的话，内核会使用一个特定的线程去轮询操作。这里使用的轮询结构会最终对应到struct file_operations中的iopoll操作，这个操作作为一个新的接口在最近才添加到这里，Linux native aio的新功能也使用了这个iopoll。这里io uring实际上只有vfs层的改动，其它的都是使用以及存在的东西，而且几个核心的东西和aio使用的相同/类似。这一步没有设置的时候，会走到io_ring_submit，进行提交请求的操作。另外如前面所提到的，在设置了flagsIORING_ENTER_GETEVENTS的情况，会等到min_complete个请求完成才会返回，这里会根据是否使用了IORING_SETUP_IOPOLL走如到两个分支。

```c
SYSCALL_DEFINE6(io_uring_enter, unsigned int, fd, u32, to_submit,
		u32, min_complete, u32, flags, const sigset_t __user *, sig,
		size_t, sigsz)
{
	struct io_ring_ctx *ctx;
	long ret = -EBADF;
	int submitted = 0;
	struct fd f;
  //...
	/*
	 * For SQ polling, the thread will do all submissions and completions.
	 * Just return the requested submit count, and wake the thread if
	 * we were asked to.
	 */
	if (ctx->flags & IORING_SETUP_SQPOLL) {
		if (flags & IORING_ENTER_SQ_WAKEUP)
			wake_up(&ctx->sqo_wait);
		submitted = to_submit;
		goto out_ctx;
	}

	ret = 0;
	if (to_submit) {
		to_submit = min(to_submit, ctx->sq_entries);
		mutex_lock(&ctx->uring_lock);
		submitted = io_ring_submit(ctx, to_submit);
		mutex_unlock(&ctx->uring_lock);
	}
  
	if (flags & IORING_ENTER_GETEVENTS) {
		unsigned nr_events = 0;
		min_complete = min(min_complete, ctx->cq_entries);
		if (ctx->flags & IORING_SETUP_IOPOLL) {
			ret = io_iopoll_check(ctx, &nr_events, min_complete);
		} else {
			ret = io_cqring_wait(ctx, min_complete, sig, sigsz);
		}
	}

out_ctx:
	io_ring_drop_ctx_refs(ctx, 1);
out_fput:
	fdput(f);
	return submitted ? submitted : ret;
}
```

以io_iopoll_check为例，正常情况下执行路线是io_iopoll_check -> io_iopoll_getevents -> io_do_iopoll -> (kiocb->ki_filp->f_op->iopoll). 在完成请求的操作之后，会调用下面这个函数提交结果到cqe数组中，这样应用就能看到结果了。这里的io_cqring_fill_event就是获取一个目前可以写入到cqe，写入数据。这里最终调用的会是io_get_cqring，可以见就是返回目前tail的后面的一个。

```c
static void io_iopoll_complete(struct io_ring_ctx *ctx, unsigned int *nr_events,
			       struct list_head *done)
{
	void *reqs[IO_IOPOLL_BATCH];
	struct io_kiocb *req;
	int to_free;

	to_free = 0;
	while (!list_empty(done)) {
		req = list_first_entry(done, struct io_kiocb, list);
		list_del(&req->list);

		io_cqring_fill_event(ctx, req->user_data, req->result);
		(*nr_events)++;

		if (refcount_dec_and_test(&req->refs)) {
			/* If we're not using fixed files, we have to pair the
			 * completion part with the file put. Use regular
			 * completions for those, only batch free for fixed
			 * file and non-linked commands.
			 */
			if ((req->flags & (REQ_F_FIXED_FILE|REQ_F_LINK)) ==
			    REQ_F_FIXED_FILE) {
				reqs[to_free++] = req;
				if (to_free == ARRAY_SIZE(reqs))
					io_free_req_many(ctx, reqs, &to_free);
			} else {
				io_free_req(req);
			}
		}
	}

	io_commit_cqring(ctx);
	io_free_req_many(ctx, reqs, &to_free);
}

static struct io_uring_cqe *io_get_cqring(struct io_ring_ctx *ctx)
{
	struct io_cq_ring *ring = ctx->cq_ring;
	unsigned tail;

	tail = ctx->cached_cq_tail;
	/*
	 * writes to the cq entry need to come after reading head; the
	 * control dependency is enough as we're using WRITE_ONCE to
	 * fill the cq entry
	 */
	if (tail - READ_ONCE(ring->r.head) == ring->ring_entries)
		return NULL;

	ctx->cached_cq_tail++;
	return &ring->cqes[tail & ctx->cq_mask];
}
```

static int io_sq_thread(void *data)为内核轮询线程的逻辑，在设置了对应的flags的时候会启动。

### 0x04 评估

这里的一些信息可以参看[1]。另外网上可以找到不少的关于io uring性能的数据。在一些情况下，io uring可以达到甚至超过spdk的性能。io uring的表现很惊艳。

```
... For peak performance, io_uring helps us get to 1.7M 4k IOPS with polling. aio reaches a performance cliff much lower than that, at 608K. The comparison here isn't quite fair, since aio doesn't support polled IO. If we disable polling, io_uring is able to drive about 1.2M IOPS for the (otherwise) same test case.
```

## 参考

1. Efficient IO with io_uring, http://kernel.dk/io_uring.pdf

> 原文链接：https://nan01ab.github.io/2019/05/io_uring.html
