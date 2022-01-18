io_uring_register - register files or user buffers for asynchronous I/O

## SYNOPSIS

```
#include <linux/io_uring.h>
```

- int `io_uring_register`(unsigned int *fd*, unsigned int *opcode*, void **arg*, unsigned int *nr_args*) 

  

## DESCRIPTION

The **io_uring_register**() system call registers user buffers or files for use in an **io_uring**(7) instance referenced by *fd*. Registering files or user buffers allows the kernel to take long term references to internal data structures or create long term mappings of application memory, greatly reducing per-I/O overhead.

*fd* is the file descriptor returned by a call to **io_uring_setup**(2). *opcode* can be one of:

- **IORING_REGISTER_BUFFERS**

  *arg* points to a *struct iovec* array of *nr_args* entries. The buffers associated with the iovecs will be locked in memory and charged against the user’s **RLIMIT_MEMLOCK** resource limit. See **getrlimit**(2) for more information. Additionally, there is a size limit of 1GiB per buffer. Currently, the buffers must be anonymous, non-file-backed memory, such as that returned by **malloc**(3) or **mmap**(2) with the **MAP_ANONYMOUS** flag set. It is expected that this limitation will be lifted in the future. Huge pages are supported as well. Note that the entire huge page will be pinned in the kernel, even if only a portion of it is used.

After a successful call, the supplied buffers are mapped into the kernel and eligible for I/O. To make use of them, the application must specify the **IORING_OP_READ_FIXED** or **IORING_OP_WRITE_FIXED** opcodes in the submission queue entry (see the *struct io_uring_sqe* definition in **io_uring_enter**(2)), and set the *buf_index* field to the desired buffer index. The memory range described by the submission queue entry’s *addr* and *len* fields must fall within the indexed buffer.

It is perfectly valid to setup a large buffer and then only use part of it for an I/O, as long as the range is within the originally mapped region.

An application can increase or decrease the size or number of registered buffers by first unregistering the existing buffers, and then issuing a new call to **io_uring_register**() with the new buffers.

Note that registering buffers will wait for the ring to idle. If the application currently has requests in-flight, the registration will wait for those to finish before proceeding.

An application need not unregister buffers explicitly before shutting down the io_uring instance. Available since 5.1.

- **IORING_UNREGISTER_BUFFERS**

  This operation takes no argument, and *arg* must be passed as NULL. All previously registered buffers associated with the io_uring instance will be released. Available since 5.1.

- **IORING_REGISTER_FILES**

  Register files for I/O. *arg* contains a pointer to an array of *nr_args* file descriptors (signed 32 bit integers).

To make use of the registered files, the **IOSQE_FIXED_FILE** flag must be set in the *flags* member of the *struct io_uring_sqe*, and the *fd* member is set to the index of the file in the file descriptor array.

The file set may be sparse, meaning that the **fd** field in the array may be set to **-1.** See **IORING_REGISTER_FILES_UPDATE** for how to update files in place.

Note that registering files will wait for the ring to idle. If the application currently has requests in-flight, the registration will wait for those to finish before proceeding. See **IORING_REGISTER_FILES_UPDATE** for how to update an existing set without that limitation.

Files are automatically unregistered when the io_uring instance is torn down. An application need only unregister if it wishes to register a new set of fds. Available since 5.1.

- **IORING_REGISTER_FILES_UPDATE**

  This operation replaces existing files in the registered file set with new ones, either turning a sparse entry (one where fd is equal to -1) into a real one, removing an existing entry (new one is set to -1), or replacing an existing entry with a new existing entry. *arg* must contain a pointer to a struct io_uring_files_update, which contains an offset on which to start the update, and an array of file descriptors to use for the update. *nr_args* must contain the number of descriptors in the passed in array. Available since 5.5.

- **IORING_UNREGISTER_FILES**

  This operation requires no argument, and *arg* must be passed as NULL. All previously registered files associated with the io_uring instance will be unregistered. Available since 5.1.

- **IORING_REGISTER_EVENTFD**

  It’s possible to use eventfd(2) to get notified of completion events on an io_uring instance. If this is desired, an eventfd file descriptor can be registered through this operation. *arg* must contain a pointer to the eventfd file descriptor, and *nr_args* must be 1. Available since 5.2.

- **IORING_REGISTER_EVENTFD_ASYNC**

  This works just like **IORING_REGISTER_EVENTFD** , except notifications are only posted for events that complete in an async manner. This means that events that complete inline while being submitted do not trigger a notification event. The arguments supplied are the same as for **IORING_REGISTER_EVENTFD.** Available since 5.6.

- **IORING_UNREGISTER_EVENTFD**

  Unregister an eventfd file descriptor to stop notifications. Since only one eventfd descriptor is currently supported, this operation takes no argument, and *arg* must be passed as NULL and *nr_args* must be zero. Available since 5.2.

- **IORING_REGISTER_PROBE**

  This operation returns a structure, io_uring_probe, which contains information about the opcodes supported by io_uring on the running kernel. *arg* must contain a pointer to a struct io_uring_probe, and *nr_args* must contain the size of the ops array in that probe struct. The ops array is of the type io_uring_probe_op, which holds the value of the opcode and a flags field. If the flags field has **IO_URING_OP_SUPPORTED** set, then this opcode is supported on the running kernel. Available since 5.6.

- **IORING_REGISTER_PERSONALITY**

  This operation registers credentials of the running application with io_uring, and returns an id associated with these credentials. Applications wishing to share a ring between separate users/processes can pass in this credential id in the sqe **personality** field. If set, that particular sqe will be issued with these credentials. Must be invoked with *arg* set to NULL and *nr_args* set to zero. Available since 5.6.

- **IORING_UNREGISTER_PERSONALITY**

  This operation unregisters a previously registered personality with io_uring. *nr_args* must be set to the id in question, and *arg* must be set to NULL. Available since 5.6.

## RETURN VALUE

On success, **io_uring_register**() returns 0. On error, -1 is returned, and *errno* is set accordingly.

## ERRORS

- **EBADF**

  One or more fds in the *fd* array are invalid.

- **EBUSY**

  **IORING_REGISTER_BUFFERS** or **IORING_REGISTER_FILES** was specified, but there were already buffers or files registered.

- **EFAULT**

  buffer is outside of the process’ accessible address space, or *iov_len* is greater than 1GiB.

- **EINVAL**

  **IORING_REGISTER_BUFFERS** or **IORING_REGISTER_FILES** was specified, but *nr_args* is 0.

- **EINVAL**

  **IORING_REGISTER_BUFFERS** was specified, but *nr_args* exceeds **UIO_MAXIOV**

- **EINVAL**

  **IORING_UNREGISTER_BUFFERS** or **IORING_UNREGISTER_FILES** was specified, and *nr_args* is non-zero or *arg* is non-NULL.

- **EMFILE**

  **IORING_REGISTER_FILES** was specified and *nr_args* exceeds the maximum allowed number of files in a fixed file set.

- **EMFILE**

  **IORING_REGISTER_FILES** was specified and adding *nr_args* file references would exceed the maximum allowed number of files the user is allowed to have according to the **RLIMIT_NOFILE** resource limit and the caller does not have **CAP_SYS_RESOURCE** capability. Note that this is a per user limit, not per process.

- **ENOMEM**

  Insufficient kernel resources are available, or the caller had a non-zero **RLIMIT_MEMLOCK** soft resource limit, but tried to lock more memory than the limit permitted. This limit is not enforced if the process is privileged (**CAP_IPC_LOCK**).

- **ENXIO**

  **IORING_UNREGISTER_BUFFERS** or **IORING_UNREGISTER_FILES** was specified, but there were no buffers or files registered.

- **ENXIO**

  Attempt to register files or buffers on an io_uring instance that is already undergoing file or buffer registration, or is being torn down.

- **EOPNOTSUPP**

  User buffers point to file-backed memory.

> 原文链接：https://unixism.net/loti/ref-iouring/io_uring_register.html

