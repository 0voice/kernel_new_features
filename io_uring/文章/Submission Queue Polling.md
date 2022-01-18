Reducing the number of system calls is a major aim for `io_uring`. To this end, `io_uring` lets you submit I/O requests without you having to make a single system call. This is done via a special submission queue polling feature that `io_uring` supports. In this mode, right after your program sets up polling mode, `io_uring` starts a special kernel thread that polls the shared submission queue for entries your program might add. That way, you just have to submit entries into the shared queue and the kernel thread should see it and pick up the submission queue entry without your program having to make the [`io_uring_enter()`](https://unixism.net/loti/ref-iouring/io_uring_enter.html#c.io_uring_enter) system call, which is usually taken care of by `liburing`. This is a benefit of having a queue that is shared between user space and the kernel.

How to use this mode? The idea is simple. You tell `io_uring` that you want use this mode by setting the `IORING_SETUP_SQPOLL` flag in the `io_uring_params` structure’s `flags` member. If the kernel thread that starts along with your process does not see any submission for a period of time, it will quit and your program will need to call the [`io_uring_enter()`](https://unixism.net/loti/ref-iouring/io_uring_enter.html#c.io_uring_enter) system call once more to wake it up. This period of time is configurable via :c:struct`io_uring_params` structure’s `sq_thread_idle` member. If you keep the submissions coming however, the kernel poller thread should never sleep.

Note

> When using `liburing`, you never directly call the [`io_uring_enter()`](https://unixism.net/loti/ref-iouring/io_uring_enter.html#c.io_uring_enter) system call. That is usually taken care of by `liburing`’s [`io_uring_submit()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_submit) function. It automatically determines if you are using polling mode or not and deals with when your program needs to call [`io_uring_enter()`](https://unixism.net/loti/ref-iouring/io_uring_enter.html#c.io_uring_enter) without you having to bother about it.

Note

> The kernel’s poller thread can take up a lot of CPU. You need to be careful about using this feature. Setting a very large `sq_thread_idle` value will cause the kernel thread to continue to consume CPU while there are no submissions happening from your program. It is a good idea to use this feature if you truly expect to handle large amounts of I/O. And even when you do so, it might be a good idea to set the poller thread’s idle value to a few seconds at most.
If you need to use this feature however, you will also need to use it in conjunction with [`io_uring_register_files()`](https://unixism.net/loti/ref-liburing/advanced_usage.html#c.io_uring_register_files). Using this, you tell the kernel about an array of file descriptors beforehand. This is just a regular array of file descriptors you open before initiating I/O. During submission, rather than passing a file descriptor as you normally would to calls like [`io_uring_prep_read()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_prep_read) or [`io_uring_prep_write()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_prep_write), you need to set the `IOSQE_FIXED_FILE` flag in the `flags` field of the SQE and pass the index of the file descriptor from the array of file descriptors you setup before.

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <fcntl.h>
#include <liburing.h>
#include <string.h>

#define BUF_SIZE    512
#define FILE_NAME1  "/tmp/io_uring_sq_test.txt"
#define STR1        "What is this life if, full of care,\n"
#define STR2        "We have no time to stand and stare."

void print_sq_poll_kernel_thread_status() {

    if (system("ps --ppid 2 | grep io_uring-sq" ) == 0)
        printf("Kernel thread io_uring-sq found running...\n");
    else
        printf("Kernel thread io_uring-sq is not running.\n");
}

int start_sq_polling_ops(struct io_uring *ring) {
    int fds[2];
    char buff1[BUF_SIZE];
    char buff2[BUF_SIZE];
    char buff3[BUF_SIZE];
    char buff4[BUF_SIZE];
    struct io_uring_sqe *sqe;
    struct io_uring_cqe *cqe;
    int str1_sz = strlen(STR1);
    int str2_sz = strlen(STR2);

    fds[0] = open(FILE_NAME1, O_RDWR | O_TRUNC | O_CREAT, 0644);
    if (fds[0] < 0 ) {
        perror("open");
        return 1;
    }

    memset(buff1, 0, BUF_SIZE);
    memset(buff2, 0, BUF_SIZE);
    memset(buff3, 0, BUF_SIZE);
    memset(buff4, 0, BUF_SIZE);
    strncpy(buff1, STR1, str1_sz);
    strncpy(buff2, STR2, str2_sz);

    int ret = io_uring_register_files(ring, fds, 1);
    if(ret) {
        fprintf(stderr, "Error registering buffers: %s", strerror(-ret));
        return 1;
    }

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }
    io_uring_prep_write(sqe, 0, buff1, str1_sz, 0);
    sqe->flags |= IOSQE_FIXED_FILE;

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }
    io_uring_prep_write(sqe, 0, buff2, str2_sz, str1_sz);
    sqe->flags |= IOSQE_FIXED_FILE;

    io_uring_submit(ring);

    for(int i = 0; i < 2; i ++) {
        int ret = io_uring_wait_cqe(ring, &cqe);
        if (ret < 0) {
            fprintf(stderr, "Error waiting for completion: %s\n",
                    strerror(-ret));
            return 1;
        }
        /* Now that we have the CQE, let's process the data */
        if (cqe->res < 0) {
            fprintf(stderr, "Error in async operation: %s\n", strerror(-cqe->res));
        }
        printf("Result of the operation: %d\n", cqe->res);
        io_uring_cqe_seen(ring, cqe);
    }

    print_sq_poll_kernel_thread_status();

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }
    io_uring_prep_read(sqe, 0, buff3, str1_sz, 0);
    sqe->flags |= IOSQE_FIXED_FILE;

    sqe = io_uring_get_sqe(ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE.\n");
        return 1;
    }
    io_uring_prep_read(sqe, 0, buff4, str2_sz, str1_sz);
    sqe->flags |= IOSQE_FIXED_FILE;

    io_uring_submit(ring);

    for(int i = 0; i < 2; i ++) {
        int ret = io_uring_wait_cqe(ring, &cqe);
        if (ret < 0) {
            fprintf(stderr, "Error waiting for completion: %s\n",
                    strerror(-ret));
            return 1;
        }
        /* Now that we have the CQE, let's process the data */
        if (cqe->res < 0) {
            fprintf(stderr, "Error in async operation: %s\n", strerror(-cqe->res));
        }
        printf("Result of the operation: %d\n", cqe->res);
        io_uring_cqe_seen(ring, cqe);
    }
    printf("Contents read from file:\n");
    printf("%s%s", buff3, buff4);
}

int main() {
    struct io_uring ring;
    struct io_uring_params params;

    if (geteuid()) {
        fprintf(stderr, "You need root privileges to run this program.\n");
        return 1;
    }

    print_sq_poll_kernel_thread_status();

    memset(&params, 0, sizeof(params));
    params.flags |= IORING_SETUP_SQPOLL;
    params.sq_thread_idle = 2000;

    int ret = io_uring_queue_init_params(8, &ring, &params);
    if (ret) {
        fprintf(stderr, "Unable to setup io_uring: %s\n", strerror(-ret));
        return 1;
    }
    start_sq_polling_ops(&ring);
    io_uring_queue_exit(&ring);
    return 0;
}
```

## How it works

This example program is much like the [Fixed buffers](https://unixism.net/loti/tutorial/fixed_buffers.html#fixed-buffers) example we saw before. Whereas we used specialized functions like [`io_uring_prep_read_fixed()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_prep_read_fixed) and [`io_uring_prep_write_fixed()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_prep_write_fixed) to deal with fixed buffers, we use regular functions like [`io_uring_prep_read()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_prep_read), [`io_uring_prep_readv()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_prep_readv), [`io_uring_prep_write()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_prep_write) or [`io_uring_prep_writev()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_prep_writev). In the SQE that is used the describe the submission however, you set the `IOSQE_FIXED_FILE` flag while using the index of the file descriptors in the file descriptor array rather than the file descriptor itself in calls like [`io_uring_prep_readv()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_prep_readv) and [`io_uring_prep_writev()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_prep_writev).

When the program starts, before we setup the `io_uring` instance, we print the running status of the kernel thread that does the submission queue polling. The name of this thread is `io_uring-sq`. The function `print_sq_poll_kernel_thread_status()` prints this status. Of course, if there is any other process using submission queue polling, you will see that this kernel thread is indeed running. The parent for all kernel threads is the `kthreadd` kernel thread which is started right after `init`, which famously has a process ID of 1. As a result, `kthreadd` has a PID of 2 and we can exploit this fact to filter only kernel threads as a simple optimization.

To initialize `io_uring`, we use the [`io_uring_queue_init_params()`](https://unixism.net/loti/ref-liburing/setup_teardown.html#c.io_uring_queue_init_params) rather than the usual [`io_uring_queue_init()`](https://unixism.net/loti/ref-liburing/setup_teardown.html#c.io_uring_queue_init) since this takes a pointer to a `io_uring_params` structure as an argument. It is in that argument that we specify `IORING_SETUP_SQPOLL` as part of the `flags` field and set `sq_thread_idle` to 2000, which is the idle time for the submission queue poller kernel thread. If there are no submissions for these many milliseconds, the thread will exit and an [`io_uring_enter()`](https://unixism.net/loti/ref-iouring/io_uring_enter.html#c.io_uring_enter) system call will need to be made internally by `liburing` to get the kernel thread going again.

Since submission queue polling only works in combination with fixed files, we first register the lone file descriptor we want to deal with. If you are dealing with more files, this is where you open and register them with the [`io_uring_register_files()`](https://unixism.net/loti/ref-liburing/advanced_usage.html#c.io_uring_register_files) function. For each submission, you need to set the `IOSQE_FIXED_FILE` flag with the `io_sqe_set_flags()` helper function and provide the index of the open file from the array of registered files rather than the actual file descriptor itself to functions like [`io_uring_prep_read()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_prep_read) or [`io_uring_prep_write()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_prep_write).

In this example, we have 4 buffers. The first 2 are used by 2 write operations to write a line each into a file. Later, we use the 3rd and 4th buffers with 2 more read operations to read the 2 written lines and print them. After the write operations, we print the status of the `io_uring-sq` kernel thread, which we should now find running.

```
➜  sudo ./sq_poll
[sudo] password for shuveb:
Kernel thread io_uring-sq is not running.
Result of the operation: 36
Result of the operation: 35
   1750 ?        00:00:00 io_uring-sq
Kernel thread io_uring-sq found running...
Result of the operation: 36
Result of the operation: 35
Contents read from file:
What is this life if, full of care,
We have no time to stand and stare.%                                                                    ➜
```

## Verifying polling by the kernel

You do call [`io_uring_submit()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_submit), though. We saw in previous examples that this caused an [`io_uring_enter()`](https://unixism.net/loti/ref-iouring/io_uring_enter.html#c.io_uring_enter) system call to be issued. Not in this case where you’ve set up the `IORING_SETUP_SQPOLL` flag, though. `liburing` completely hides this from you while keeping a constant interface to your programs. But, can we verify this? Yes, we can via the `bpftrace` program that uses eBPF to let us peek into the system. Here, we will use tracepoints in the kernel that `io_uring` has setup to prove that when we set `IORING_SETUP_SQPOLL` and submit I/O requests, in spite of us calling the [`io_uring_submit()`](https://unixism.net/loti/ref-liburing/submission.html#c.io_uring_submit) function, our program never makes the [`io_uring_enter()`](https://unixism.net/loti/ref-iouring/io_uring_enter.html#c.io_uring_enter) system call. Like discussed previously, for high throughput programs, the idea is to avoid system calls as much as we can.

In the program below, we attach to `io_uring`’s `io_uring_submit_sqe` tracepoint. This tracepoint is triggered whenever an SQE is submitted to the kernel. Each time this tracepoint is triggered, we use `bpftrace` to print the name of the command and its PID. First, let’s run the `bpftrace` command on one terminal while running the [Fixed buffers](https://unixism.net/loti/tutorial/fixed_buffers.html#fixed-buffers) example in another. Here is a sample output from my machine. You can see that `fixed_buffers` is the one submitting the SQE.

```
➜  sudo bpftrace -e 'tracepoint:io_uring:io_uring_submit_sqe {printf("%s(%d)\n", comm, pid);}'
Attaching 1 probe...
fixed_buffers(30336)
fixed_buffers(30336)
fixed_buffers(30336)
fixed_buffers(30336)
```

Let’s repeat the previous exercise, but now by running the current example. You can see that the SQE submission happens via the `io_uring_sq` kernel thread. We thus avoid system calls.

```
➜  sudo bpftrace -e 'tracepoint:io_uring:io_uring_submit_sqe {printf("%s(%d)\n", comm, pid);}'
io_uring-sq(30429)
io_uring-sq(30429)
io_uring-sq(30429)
io_uring-sq(30429)
```

## Source code

Source code for this and other examples is [available on Github](https://github.com/shuveb/loti-examples).

> 原文链接：https://unixism.net/loti/tutorial/sq_poll.html#sq-poll

