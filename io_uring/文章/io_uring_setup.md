io_uring_setup - setup a context for performing asynchronous I/O

## SYNOPSIS

```
#include <linux/io_uring.h>
```

- int `io_uring_setup`(u32 *entries*, *struct* io_uring_params **p*) 

  

## DESCRIPTION

The io_uring_setup() system call sets up a submission queue (SQ) and completion queue (CQ) with at least *entries* entries, and returns a file descriptor which can be used to perform subsequent operations on the io_uring instance. The submission and completion queues are shared between userspace and the kernel, which eliminates the need to copy data when initiating and completing I/O.

*params* is used by the application to pass options to the kernel, and by the kernel to convey information about the ring buffers.

```
struct io_uring_params {
    __u32 sq_entries;
    __u32 cq_entries;
    __u32 flags;
    __u32 sq_thread_cpu;
    __u32 sq_thread_idle;
    __u32 features;
    __u32 resv[4];
    struct io_sqring_offsets sq_off;
    struct io_cqring_offsets cq_off;
};
```

The *flags*, *sq_thread_cpu*, and *sq_thread_idle* fields are used to configure the io_uring instance. *flags* is a bit mask of 0 or more of the following values ORed together:

- **IORING_SETUP_IOPOLL**

  Perform busy-waiting for an I/O completion, as opposed to getting notifications via an asynchronous IRQ (Interrupt Request). The file system (if any) and block device must support polling in order for this to work. Busy-waiting provides lower latency, but may consume more CPU resources than interrupt driven I/O. Currently, this feature is usable only on a file descriptor opened using the **O_DIRECT** flag. When a read or write is submitted to a polled context, the application must poll for completions on the CQ ring by calling **io_uring_enter**(2). It is illegal to mix and match polled and non-polled I/O on an io_uring instance.

- **IORING_SETUP_SQPOLL**

  When this flag is specified, a kernel thread is created to perform submission queue polling. An io_uring instance configured in this way enables an application to issue I/O without ever context switching into the kernel. By using the submission queue to fill in new submission queue entries and watching for completions on the completion queue, the application can submit and reap I/Os without doing a single system call.

If the kernel thread is idle for more than *sq_thread_idle* milliseconds, it will set the **IORING_SQ_NEED_WAKEUP** bit in the *flags* field of the *struct io_sq_ring*. When this happens, the application must call **io_uring_enter**(2) to wake the kernel thread. If I/O is kept busy, the kernel thread will never sleep. An application making use of this feature will need to guard the **io_uring_enter**(2) call with the following code sequence:

```
/*
 * Ensure that the wakeup flag is read after the tail pointer has been
 * written.
 */
smp_mb();
if (*sq_ring->flags & IORING_SQ_NEED_WAKEUP)
    io_uring_enter(fd, 0, 0, IORING_ENTER_SQ_WAKEUP);
```

where *sq_ring* is a submission queue ring setup using the *struct io_sqring_offsets* described below.

> To successfully use this feature, the application must register a set of files to be used for IO through **io_uring_register**(2) using the **IORING_REGISTER_FILES** opcode. Failure to do so will result in submitted IO being errored with **EBADF.**

- **IORING_SETUP_SQ_AFF**

  If this flag is specified, then the poll thread will be bound to the cpu set in the *sq_thread_cpu* field of the *struct io_uring_params*. This flag is only meaningful when **IORING_SETUP_SQPOLL** is specified.

- **IORING_SETUP_CQSIZE**

  Create the completion queue with *struct io_uring_params.cq_entries* entries. The value must be greater than *entries*, and may be rounded up to the next power-of-two.

If no flags are specified, the io_uring instance is setup for interrupt driven I/O. I/O may be submitted using **io_uring_enter**(2) and can be reaped by polling the completion queue.

The *resv* array must be initialized to zero.

*features* is filled in by the kernel, which specifies various features supported by current kernel version.

- **IORING_FEAT_SINGLE_MMAP**

  If this flag is set, the two SQ and CQ rings can be mapped with a single *mmap(2)* call. The SQEs must still be allocated separately. This brings the necessary *mmap(2)* calls down from three to two.

- **IORING_FEAT_NODROP**

  If this flag is set, io_uring supports never dropping completion events. If a completion event occurs and the CQ ring is full, the kernel stores the event internally until such a time that the CQ ring has room for more entries. If this overflow condition is entered, attempting to submit more IO with fail with the **-EBUSY** error value, if it can’t flush the overflown events to the CQ ring. If this happens, the application must reap events from the CQ ring and attempt the submit again.

- **IORING_FEAT_SUBMIT_STABLE**

  If this flag is set, applications can be certain that any data for async offload has been consumed when the kernel has consumed the SQE.

- **IORING_FEAT_RW_CUR_POS**

  If this flag is set, applications can specify *offset* == -1 with **IORING_OP_{READV,WRITEV}** , **IORING_OP_{READ,WRITE}_FIXED** , and **IORING_OP_{READ,WRITE}** to mean current file position, which behaves like *preadv2(2)* and *pwritev2(2)* with *offset* == -1. It’ll use (and update) the current file position. This obviously comes with the caveat that if the application has multiple reads or writes in flight, then the end result will not be as expected. This is similar to threads sharing a file descriptor and doing IO using the current file position.

- **IORING_FEAT_CUR_PERSONALITY**

  If this flag is set, then io_uring guarantees that both sync and async execution of a request assumes the credentials of the task that called *io_uring_enter(2)* to queue the requests. If this flag isn’t set, then requests are issued with the credentials of the task that originally registered the io_uring. If only one task is using a ring, then this flag doesn’t matter as the credentials will always be the same. Note that this is the default behavior, tasks can still register different personalities through *io_uring_register(2)* with **IORING_REGISTER_PERSONALITY** and specify the personality to use in the sqe.

The rest of the fields in the *struct io_uring_params* are filled in by the kernel, and provide the information necessary to memory map the submission queue, completion queue, and the array of submission queue entries. *sq_entries* specifies the number of submission queue entries allocated. *sq_off* describes the offsets of various ring buffer fields:

```
struct io_sqring_offsets {
    __u32 head;
    __u32 tail;
    __u32 ring_mask;
    __u32 ring_entries;
    __u32 flags;
    __u32 dropped;
    __u32 array;
    __u32 resv[3];
};
```

Taken together, *sq_entries* and *sq_off* provide all of the information necessary for accessing the submission queue ring buffer and the submission queue entry array. The submission queue can be mapped with a call like:

```
ptr = mmap(0, sq_off.array + sq_entries * sizeof(__u32),
           PROT_READ|PROT_WRITE, MAP_SHARED|MAP_POPULATE,
           ring_fd, IORING_OFF_SQ_RING);
```

where *sq_off* is the *io_sqring_offsets* structure, and *ring_fd* is the file descriptor returned from **io_uring_setup**(2). The addition of *sq_off.array* to the length of the region accounts for the fact that the ring located at the end of the data structure. As an example, the ring buffer head pointer can be accessed by adding *sq_off.head* to the address returned from **mmap**(2):

```
head = ptr + sq_off.head;
```

The *flags* field is used by the kernel to communicate state information to the application. Currently, it is used to inform the application when a call to **io_uring_enter**(2) is necessary. See the documentation for the **IORING_SETUP_SQPOLL** flag above. The *dropped* member is incremented for each invalid submission queue entry encountered in the ring buffer.

The head and tail track the ring buffer state. The tail is incremented by the application when submitting new I/O, and the head is incremented by the kernel when the I/O has been successfully submitted. Determining the index of the head or tail into the ring is accomplished by applying a mask:

```
index = tail & ring_mask;
```

The array of submission queue entries is mapped with:

```
sqentries = mmap(0, sq_entries * sizeof(struct io_uring_sqe),
                 PROT_READ|PROT_WRITE, MAP_SHARED|MAP_POPULATE,
                 ring_fd, IORING_OFF_SQES);
```

The completion queue is described by *cq_entries* and *cq_off* shown here:

```
struct io_cqring_offsets {
    __u32 head;
    __u32 tail;
    __u32 ring_mask;
    __u32 ring_entries;
    __u32 overflow;
    __u32 cqes;
    __u32 resv[4];
};
```

The completion queue is simpler, since the entries are not separated from the queue itself, and can be mapped with:

```
ptr = mmap(0, cq_off.cqes + cq_entries * sizeof(struct io_uring_cqe),
           PROT_READ|PROT_WRITE, MAP_SHARED|MAP_POPULATE, ring_fd,
           IORING_OFF_CQ_RING);
```

Closing the file descriptor returned by **io_uring_setup**(2) will free all resources associated with the io_uring context.

## RETURN VALUE

**io_uring_setup**(2) returns a new file descriptor on success. The application may then provide the file descriptor in a subsequent **mmap**(2) call to map the submission and completion queues, or to the **io_uring_register**(2) or **io_uring_enter**(2) system calls.

On error, -1 is returned and *errno* is set appropriately.

## ERRORS

- **EFAULT**

  params is outside your accessible address space.

- **EINVAL**

  The resv array contains non-zero data, p.flags contains an unsupported flag, *entries* is out of bounds, **IORING_SETUP_SQ_AFF** was specified, but **IORING_SETUP_SQPOLL** was not, or **IORING_SETUP_CQSIZE** was specified, but *io_uring_params.cq_entries* was invalid.

- **EMFILE**

  The per-process limit on the number of open file descriptors has been reached (see the description of **RLIMIT_NOFILE** in **getrlimit**(2)).

- **ENFILE**

  The system-wide limit on the total number of open files has been reached.

- **ENOMEM**

  Insufficient kernel resources are available.

- **EPERM**

  **IORING_SETUP_SQPOLL** was specified, but the effective user ID of the caller did not have sufficient privileges.

## SEE ALSO

**io_uring_register**(2), **io_uring_enter**(2)

> 原文链接：https://unixism.net/loti/ref-iouring/io_uring_setup.html

