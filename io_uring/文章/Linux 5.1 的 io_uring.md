虽然在 Facebook 上分享了一些技术文章的心得，被网友建议说可以在博客上，但似乎之前想在博客上一些比较长且整理过的东西，那么如果放的心得让更多的人看到，并且有机会交流也是的事情，接下来应该会很快将之前的笔记誊不错过来。

https://www.facebook.com/kkcliu/posts/10157179358206129

# io_uring

前阵子在 epoll 的时候刚刚看到了一个讨论串写出来的 io_uring，其实原本没听过 io_uring 是什么，后来才发现新版的 Linux kernel.1 会知道，这个主要目的是加入这个文章很好的写出原本 Linux 原生 AIO 的问题，其实一般来说 AIO 的效果应该会比 epoll 还好，简单一点的比较可以看 stackoverflow 的，[https: //stackoverflow.com/questions/5844955/whats-事件驱动和异步epoll-and-a之间的差异](https://stackoverflow.com/questions/5844955/whats-the-difference-between-event-driven-and-asynchronous-between-epoll-and-a)

- epoll 是一个阻塞操作（epoll_wait()）——你阻塞线程直到某个事件发生，然后你将事件分派到代码中的不同过程/函数/分支。
- 在 AIO 中，您将回调函数（完成例程）的地址传递给系统，系统会在发生某些事情时调用您的函数。

简单来说 epoll 是等待事件发生，才执行，epoll_wait 是一个阻塞的操作，而 AIO 是把的回调函数，AIO 交给系统执行，真正做到了异步，Mysql 的 innodb 也是使用原生 linux蛮推荐多看一下Linux下的问题，所以这里太流行了，可以这样cloudflare [https:](https://blog.cloudflare.com/io_submit-the-epoll-alternative-youve-never-heard-about/) //blog..cloudflare.com/io_submit-the-the-cloudflare.com/io_submit-the-the-cloudflare.com/io_submit-the-the-cloudflare.com/io_submit-the-the-cloudflare.com/io_submit-the-the-cloudflare.com [-youve-never-hear-about/](https://blog.cloudflare.com/io_submit-the-epoll-alternative-youve-never-heard-about/)，有介绍如何评价使用 AIO，也提到了 Lin Alternative 的一些问题，你的地方好像提到了我们对 AIO 的：

> AIO 是一种可怕的临时设计，其主要借口是“其他天赋较低的人做出了该设计，我们正在实施它以实现兼容性，因为数据库人员 - 他们很少有任何品味 - 实际使用它”。但是AIO总是真的很丑。

除了是又看到 Facebook 分享的幻灯片：[https](https://www.slideshare.net/ennael/kernel-recipes-2019-faster-io-through-iouring) : //www.slideshare.net/ennael/kernel-recipes-2019-faster-io-through-iouring 和 Hackernews [https://news.ycombinator.com/item?id =19843464](https://news.ycombinator.com/item?id=19843464)的介绍，最重要的是性能真的好上多，从这里[https://github.com/frevib/io_uring-echo-server/blob/io-uring-feat-fast-poll/benchmarks /benchmarks.md](https://github.com/frevib/io_uring-echo-server/blob/io-uring-feat-fast-poll/benchmarks/benchmarks.md)，可以找到 epoll vs io_uring 的 benchmark ，可以快速到 io_uring 的功效到 40% 以上。

![img](https://kkc.github.io/2020/08/19/io-uring/benchmark.png)

然后也看到了很多不同的项目，比如libuv、rust、ceph、rocksdb，正在讨论数据库和云相关的集成，这对相关的产业会有很大的影响，省的成本光用想的就很重要很神奇，虽然要等到大家升到5.1不容易，但是会期待这个发展了。

后记：首席副总裁 Champ 的问题是因为只有 DIRECT_IO，对于很多程序来说，都是为了解决 Linux AIO 的问题。

# 参考

- https://kernel.dk/io_uring-whatsnew.pdf
- https://github.com/agnivade/frodo
- https://github.com/hodgesds/iouring-go
- https://lwn.net/Articles/776703/
- https://www.slideshare.net/ennael/kernel-recipes-2019-faster-io-through-iouring

> 原文链接：https://kkc.github.io/2020/08/19/io-uring/

