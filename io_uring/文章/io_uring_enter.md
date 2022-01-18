io_uring_enter - initiate and/or complete asynchronous I/O

## SYNOPSIS

```
#include <linux/io_uring.h>
```

- int `io_uring_enter`(unsigned int *fd*, unsigned int *to_submit*, unsigned int *min_complete*, unsigned int *flags*, sigset_t **sig*) 

  

## DESCRIPTION

**io_uring_enter**() is used to initiate and complete I/O using the shared submission and completion queues setup by a call to **io_uring_setup**(2). A single call can both submit new I/O and wait for completions of I/O initiated by this call or previous calls to **io_uring_enter**().

*fd* is the file descriptor returned by **io_uring_setup**(2). *to_submit* specifies the number of I/Os to submit from the submission queue. If the **IORING_ENTER_GETEVENTS** bit is set in *flags*, then the system call will attempt to wait for *min_complete* event completions before returning. If the io_uring instance was configured for polling, by specifying **IORING_SETUP_IOPOLL** in the call to **io_uring_setup**(2), then min_complete has a slightly different meaning. Passing a value of 0 instructs the kernel to return any events which are already complete, without blocking. If *min_complete* is a non-zero value, the kernel will still return immediately if any completion events are available. If no event completions are available, then the call will poll either until one or more completions become available, or until the process has exceeded its scheduler time slice.

Note that, for interrupt driven I/O (where **IORING_SETUP_IOPOLL** was not specified in the call to **io_uring_setup**(2)), an application may check the completion queue for event completions without entering the kernel at all.

When the system call returns that a certain amount of SQEs have been consumed and submitted, it’s safe to reuse SQE entries in the ring. This is true even if the actual IO submission had to be punted to async context, which means that the SQE may in fact not have been submitted yet. If the kernel requires later use of a particular SQE entry, it will have made a private copy of it.

*sig* is a pointer to a signal mask (see **sigprocmask**(2)); if *sig* is not NULL, **io_uring_enter**() first replaces the current signal mask by the one pointed to by *sig*, then waits for events to become available in the completion queue, and then restores the original signal mask. The following **io_uring_enter**() call:

```
ret = io_uring_enter(fd, 0, 1, IORING_ENTER_GETEVENTS, &sig);
```

is equivalent to *atomically* executing the following calls:

```
pthread_sigmask(SIG_SETMASK, &sig, &orig);
ret = io_uring_enter(fd, 0, 1, IORING_ENTER_GETEVENTS, NULL);
pthread_sigmask(SIG_SETMASK, &orig, NULL);
```

See the description of **pselect**(2) for an explanation of why the *sig* parameter is necessary.

Submission queue entries are represented using the following data structure:

```
/*
 * IO submission data structure (Submission Queue Entry)
 */
struct io_uring_sqe {
    __u8    opcode;         /* type of operation for this sqe */
    __u8    flags;          /* IOSQE_ flags */
    __u16   ioprio;         /* ioprio for the request */
    __s32   fd;             /* file descriptor to do IO on */
    union {
        __u64   off;            /* offset into file */
        __u64   addr2;
    };
    __u64   addr;           /* pointer to buffer or iovecs */
    __u32   len;            /* buffer size or number of iovecs */
    union {
        __kernel_rwf_t  rw_flags;
        __u32    fsync_flags;
        __u16    poll_events;
        __u32    sync_range_flags;
        __u32    msg_flags;
        __u32    timeout_flags;
        __u32    accept_flags;
        __u32    cancel_flags;
    };
    __u64    user_data;     /* data to be passed back at completion time */
    union {
     struct {
         /* index into fixed buffers, if used */
            __u16    buf_index;
         /* personality to use, if used */
         __u16    personality;
     };
        __u64    __pad2[3];
    };
};
```

The *opcode* describes the operation to be performed. It can be one of:

- **IORING_OP_NOP**

  Do not perform any I/O. This is useful for testing the performance of the io_uring implementation itself.

**IORING_OP_READV**

- **IORING_OP_WRITEV**

  Vectored read and write operations, similar to **preadv2**(2) and **pwritev2**(2).

**IORING_OP_READ_FIXED**

- **IORING_OP_WRITE_FIXED**

  Read from or write to pre-mapped buffers. See **io_uring_register**(2) for details on how to setup a context for fixed reads and writes.

- **IORING_OP_FSYNC**

  File sync. See also **fsync**(2). Note that, while I/O is initiated in the order in which it appears in the submission queue, completions are unordered. For example, an application which places a write I/O followed by an fsync in the submission queue cannot expect the fsync to apply to the write. The two operations execute in parallel, so the fsync may complete before the write is issued to the storage. The same is also true for previously issued writes that have not completed prior to the fsync.

- **IORING_OP_POLL_ADD**

  Poll the *fd* specified in the submission queue entry for the events specified in the *poll_events* field. Unlike poll or epoll without **EPOLLONESHOT**, this interface always works in one shot mode. That is, once the poll operation is completed, it will have to be resubmitted.

- **IORING_OP_POLL_REMOVE**

  Remove an existing poll request. If found, the *res* field of the *struct io_uring_cqe* will contain 0. If not found, *res* will contain **-ENOENT.**

- **IORING_OP_EPOLL_CTL**

  Add, remove or modify entries in the interest list of **epoll**(7). See **epoll_ctl**(2) for details of the system call. *fd* holds the file descriptor that represents the epoll instance, *addr* holds the file descriptor to add, remove or modify, *len* holds the operation (EPOLL_CTL_ADD, EPOLL_CTL_DEL, EPOLL_CTL_MOD) to perform and, *off* holds a pointer to the *epoll_events* structure. Available since 5.6.

- **IORING_OP_SYNC_FILE_RANGE**

  Issue the equivalent of a **sync_file_range** (2) on the file descriptor. The *fd* field is the file descriptor to sync, the *off* field holds the offset in bytes, the *len* field holds the length in bytes, and the *flags* field holds the flags for the command. See also **sync_file_range**(2). for the general description of the related system call. Available since 5.2.

- **IORING_OP_SENDMSG**

  Issue the equivalent of a **sendmsg(2)** system call. *fd* must be set to the socket file descriptor, *addr* must contain a pointer to the msghdr structure, and *flags* holds the flags associated with the system call. See also **sendmsg**(2). for the general description of the related system call. Available since 5.3.

- **IORING_OP_RECVMSG**

  Works just like IORING_OP_SENDMSG, except for **recvmsg(2)** instead. See the description of IORING_OP_SENDMSG. Available since 5.3.

- **IORING_OP_SEND**

  Issue the equivalent of a **send(2)** system call. *fd* must be set to the socket file descriptor, *addr* must contain a pointer to the buffer, *len* denotes the length of the buffer to send, and *flags* holds the flags associated with the system call. See also **send(2).** for the general description of the related system call. Available since 5.6.

- **IORING_OP_RECV**

  Works just like IORING_OP_SEND, except for **recv(2)** instead. See the description of IORING_OP_SEND. Available since 5.6.

- **IORING_OP_TIMEOUT**

  This command will register a timeout operation. The *addr* field must contain a pointer to a struct timespec64 structure, *len* must contain 1 to signify one timespec64 structure, *timeout_flags* may contain IORING_TIMEOUT_ABS for an absolute timeout value, or 0 for a relative timeout. *off* may contain a completion event count. If not set, this defaults to 1. A timeout will trigger a wakeup event on the completion ring for anyone waiting for events. A timeout condition is met when either the specified timeout expires, or the specified number of events have completed. Either condition will trigger the event. io_uring timeouts use the **CLOCK_MONOTONIC** clock source. The request will complete with *-ETIME* if the timeout got completed through expiration of the timer, or *0* if the timeout got completed through requests completing on their own. If the timeout was cancelled before it expired, the request will complete with *-ECANCELED.* Available since 5.4.

- **IORING_OP_TIMEOUT_REMOVE**

  Attempt to remove an existing timeout operation. *addr* must contain the *user_data* field of the previously issued timeout operation. If the specified timeout request is found and cancelled successfully, this request will terminate with a result value of *0* If the timeout request was found but expiration was already in progress, this request will terminate with a result value of *-EBUSY* If the timeout request wasn’t found, the request will terminate with a result value of *-ENOENT* Available since 5.5.

- **IORING_OP_ACCEPT**

  Issue the equivalent of an **accept4(2)** system call. *fd* must be set to the socket file descriptor, *addr* must contain the pointer to the sockaddr structure, and *addr2* must contain a pointer to the socklen_t addrlen field. See also **accept4(2)** for the general description of the related system call. Available since 5.5.

- **IORING_OP_ASYNC_CANCEL**

  Attempt to cancel an already issued request. *addr* must contain the *user_data* field of the request that should be cancelled. The cancellation request will complete with one of the following results codes. If found, the *res* field of the cqe will contain 0. If not found, *res* will contain -ENOENT. If found and attempted cancelled, the *res* field will contain -EALREADY. In this case, the request may or may not terminate. In general, requests that are interruptible (like socket IO) will get cancelled, while disk IO requests cannot be cancelled if already started. Available since 5.5.

- **IORING_OP_LINK_TIMEOUT**

  This request must be linked with another request through *IOSQE_IO_LINK* which is described below. Unlike *IORING_OP_TIMEOUT,* *IORING_OP_LINK_TIMEOUT* acts on the linked request, not the completion queue. The format of the command is otherwise like *IORING_OP_TIMEOUT,* except there’s no completion event count as it’s tied to a specific request. If used, the timeout specified in the command will cancel the linked command, unless the linked command completes before the timeout. The timeout will complete with *-ETIME* if the timer expired and the linked request was attempted cancelled, or *-ECANCELED* if the timer got cancelled because of completion of the linked request. Like **IORING_OP_TIMEOUT** the clock source used is **CLOCK_MONOTONIC** Available since 5.5.

- **IORING_OP_CONNECT**

  Issue the equivalent of a **connect(2)** system call. *fd* must be set to the socket file descriptor, *addr* must contain the pointer to the sockaddr structure, and *off* must contain the socklen_t addrlen field. See also **connect(2)** for the general description of the related system call. Available since 5.5.

- **IORING_OP_FALLOCATE**

  Issue the equivalent of a **fallocate(2)** system call. *fd* must be set to the file descriptor, *off* must contain the offset on which to operate, and *len* must contain the length. See also **fallocate(2)** for the general description of the related system call. Available since 5.6.

- **IORING_OP_FADVISE**

  Issue the equivalent of a **posix_fadvise(2)** system call. *fd* must be set to the file descriptor, *off* must contain the offset on which to operate, *len* must contain the length, and *fadvise_advice* must contain the advice associated with the operation. See also **posix_fadvise(2)** for the general description of the related system call. Available since 5.6.

- **IORING_OP_MADVISE**

  Issue the equivalent of a **madvise(2)** system call. *addr* must contain the address to operate on, *len* must contain the length on which to operate, and *fadvise_advice* must contain the advice associated with the operation. See also **madvise(2)** for the general description of the related system call. Available since 5.6.

- **IORING_OP_OPENAT**

  Issue the equivalent of a **openat(2)** system call. *fd* is the *dirfd* argument, *addr* must contain a pointer to the **pathname* argument, *open_flags* should contain any flags passed in, and *mode* is access mode of the file. See also **openat(2)** for the general description of the related system call. Available since 5.6.

- **IORING_OP_OPENAT2**

  Issue the equivalent of a **openat2(2)** system call. *fd* is the *dirfd* argument, *addr* must contain a pointer to the **pathname* argument, *len* should contain the size of the open_how structure, and *off* should be set to the address of the open_how structure. See also **openat2(2)** for the general description of the related system call. Available since 5.6.

- **IORING_OP_CLOSE**

  Issue the equivalent of a **close(2)** system call. *fd* is the file descriptor to be closed. See also **close(2)** for the general description of the related system call. Available since 5.6.

- **IORING_OP_STATX**

  Issue the equivalent of a **statx(2)** system call. *fd* is the *dirfd* argument, *addr* must contain a pointer to the **pathname* string, *statx_flags* is the *flags* argument, *len* should be the *mask* argument, and *off* must contain a pointer to the *statxbuf* to be filled in. See also **statx(2)** for the general description of the related system call. Available since 5.6.

**IORING_OP_READ**

- **IORING_OP_WRITE**

  Issue the equivalent of a **read(2)** or **write(2)** system call. *fd* is the file descriptor to be operated on, *addr* contains the buffer in question, and *len* contains the length of the IO operation. These are non-vectored versions of the **IORING_OP_READV** and **IORING_OP_WRITEV** opcodes. See also **read(2)** and **write(2)** for the general description of the related system call. Available since 5.6.

- **IORING_OP_SPLICE**

  Issue the equivalent of a **splice(2)** system call. *splice_fd_in* is the file descriptor to read from, *splice_off_in* is a pointer to an offset to read from, *fd* is the file descriptor to write to, *off* is a pointer to an offset to from which to start writing to. *len* contains the number of bytes to copy. *splice_flags* contains a bit mask for the flag field associated with the system call. Please note that one of the file descriptors must refer to a pipe. See also **splice(2)** for the general description of the related system call. Available since 5.7.

- **IORING_OP_FILES_UPDATE**

  This command is an alternative to using **IORING_REGISTER_FILES_UPDATE** which then works in an async fashion, like the rest of the io_uring commands. The arguments passed in are the same. *addr* must contain a pointer to the array of file descriptors, *len* must contain the length of the array, and *off* must contain the offset at which to operate. Note that the array of file descriptors pointed to in *addr* must remain valid until this operation has completed. Available since 5.6.

The *flags* field is a bit mask. The supported flags are:

- **IOSQE_FIXED_FILE**

  When this flag is specified, *fd* is an index into the files array registered with the io_uring instance (see the **IORING_REGISTER_FILES** section of the **io_uring_register**(2) man page). Available since 5.1.

- **IOSQE_IO_DRAIN**

  When this flag is specified, the SQE will not be started before previously submitted SQEs have completed, and new SQEs will not be started before this one completes. Available since 5.2.

- **IOSQE_IO_LINK**

  When this flag is specified, it forms a link with the next SQE in the submission ring. That next SQE will not be started before this one completes. This, in effect, forms a chain of SQEs, which can be arbitrarily long. The tail of the chain is denoted by the first SQE that does not have this flag set. This flag has no effect on previous SQE submissions, nor does it impact SQEs that are outside of the chain tail. This means that multiple chains can be executing in parallel, or chains and individual SQEs. Only members inside the chain are serialized. A chain of SQEs will be broken, if any request in that chain ends in error. io_uring considers any unexpected result an error. This means that, eg, a short read will also terminate the remainder of the chain. If a chain of SQE links is broken, the remaining unstarted part of the chain will be terminated and completed with **-ECANCELED** as the error code. Available since 5.3.

- **IOSQE_IO_HARDLINK**

  Like IOSQE_IO_LINK, but it doesn’t sever regardless of the completion result. Note that the link will still sever if we fail submitting the parent request, hard links are only resilient in the presence of completion results for requests that did submit correctly. IOSQE_IO_HARDLINK implies IOSQE_IO_LINK. Available since 5.5.

- **IOSQE_ASYNC**

  Normal operation for io_uring is to try and issue an sqe as non-blocking first, and if that fails, execute it in an async manner. To support more efficient overlapped operation of requests that the application knows/assumes will always (or most of the time) block, the application can ask for an sqe to be issued async from the start. Available since 5.6.

*ioprio* specifies the I/O priority. See **ioprio_get**(2) for a description of Linux I/O priorities.

*fd* specifies the file descriptor against which the operation will be performed, with the exception noted above.

If the operation is one of **IORING_OP_READ_FIXED** or **IORING_OP_WRITE_FIXED**, *addr* and *len* must fall within the buffer located at *buf_index* in the fixed buffer array. If the operation is either **IORING_OP_READV** or **IORING_OP_WRITEV**, then *addr* points to an iovec array of *len* entries.

*rw_flags*, specified for read and write operations, contains a bitwise OR of per-I/O flags, as described in the **preadv2**(2) man page.

The *fsync_flags* bit mask may contain either 0, for a normal file integrity sync, or **IORING_FSYNC_DATASYNC** to provide data sync only semantics. See the descriptions of **O_SYNC** and **O_DSYNC** in the **open**(2) manual page for more information.

The bits that may be set in *poll_events* are defined in *<poll.h>*, and documented in **poll**(2).

*user_data* is an application-supplied value that will be copied into the completion queue entry (see below). *buf_index* is an index into an array of fixed buffers, and is only valid if fixed buffers were registered. *personality* is the credentials id to use for this operation. See **io_uring_register(2)** for how to register personalities with io_uring. If set to 0, the current personality of the submitting task is used.

Once the submission queue entry is initialized, I/O is submitted by placing the index of the submission queue entry into the tail of the submission queue. After one or more indexes are added to the queue, and the queue tail is advanced, the **io_uring_enter**(2) system call can be invoked to initiate the I/O.

Completions use the following data structure:

```
/*
 * IO completion data structure (Completion Queue Entry)
 */
struct io_uring_cqe {
    __u64    user_data; /* sqe->data submission passed back */
    __s32    res;       /* result code for this event */
    __u32    flags;
};
```

*user_data* is copied from the field of the same name in the submission queue entry. The primary use case is to store data that the application will need to access upon completion of this particular I/O. The *flags* is reserved for future use. *res* is the operation-specific result.

For read and write opcodes, the return values match those documented in the **preadv2**(2) and **pwritev2**(2) man pages. Return codes for the io_uring-specific opcodes are documented in the description of the opcodes above.

## RETURN VALUE

**io_uring_enter**() returns the number of I/Os successfully consumed. This can be zero if *to_submit* was zero or if the submission queue was empty. The errors below that refer to an error in a submission queue entry will be returned though a completion queue entry, rather than through the system call itself.

Errors that occur not on behalf of a submission queue entry are returned via the system call directly. On such an error, -1 is returned and *errno* is set appropriately.

## ERRORS

- **EAGAIN**

  The kernel was unable to allocate memory for the request, or otherwise ran out of resources to handle it. The application should wait for some completions and try again.

- **EBUSY**

  The application is attempting to overcommit the number of requests it can have pending. The application should wait for some completions and try again. May occur if the application tries to queue more requests than we have room for in the CQ ring.

- **EBADF**

  The *fd* field in the submission queue entry is invalid, or the **IOSQE_FIXED_FILE** flag was set in the submission queue entry, but no files were registered with the io_uring instance.

- **EFAULT**

  buffer is outside of the process’ accessible address space

- **EFAULT**

  **IORING_OP_READ_FIXED** or **IORING_OP_WRITE_FIXED** was specified in the *opcode* field of the submission queue entry, but either buffers were not registered for this io_uring instance, or the address range described by *addr* and *len* does not fit within the buffer registered at *buf_index*.

- **EINVAL**

  The *index* member of the submission queue entry is invalid.

- **EINVAL**

  The *flags* field or *opcode* in a submission queue entry is invalid.

- **EINVAL**

  **IORING_OP_NOP** was specified in the submission queue entry, but the io_uring context was setup for polling (**IORING_SETUP_IOPOLL** was specified in the call to io_uring_setup).

- **EINVAL**

  **IORING_OP_READV** or **IORING_OP_WRITEV** was specified in the submission queue entry, but the io_uring instance has fixed buffers registered.

- **EINVAL**

  **IORING_OP_READ_FIXED** or **IORING_OP_WRITE_FIXED** was specified in the submission queue entry, and the *buf_index* is invalid.

- **EINVAL**

  **IORING_OP_READV**, **IORING_OP_WRITEV**, **IORING_OP_READ_FIXED**, **IORING_OP_WRITE_FIXED** or **IORING_OP_FSYNC** was specified in the submission queue entry, but the io_uring instance was configured for IOPOLLing, or any of *addr*, *ioprio*, *off*, *len*, or *buf_index* was set in the submission queue entry.

- **EINVAL**

  **IORING_OP_POLL_ADD** or **IORING_OP_POLL_REMOVE** was specified in the *opcode* field of the submission queue entry, but the io_uring instance was configured for busy-wait polling (**IORING_SETUP_IOPOLL**), or any of *ioprio*, *off*, *len*, or *buf_index* was non-zero in the submission queue entry.

- **EINVAL**

  **IORING_OP_POLL_ADD** was specified in the *opcode* field of the submission queue entry, and the *addr* field was non-zero.

- **ENXIO**

  The io_uring instance is in the process of being torn down.

- **EOPNOTSUPP**

  *fd* does not refer to an io_uring instance.

- **EOPNOTSUPP**

  *opcode* is valid, but not supported by this kernel.

- **EINTR**

  The operation was interrupted by a delivery of a signal before it could complete; see **signal(7).** Can happen while waiting for events with **IORING_ENTER_GETEVENTS.**

> 原文链接：https://unixism.net/loti/ref-iouring/io_uring_enter.html

