# 🔰 深挖 Linux 内核的新功能特性，以 io_uring, cgroup, ebpf, llvm, kvm, ceph, fuse 为代表，包含开源项目，代码案例，文章，视频，架构脑图等

## 🔥 [io_uring](https://en.wikipedia.org/wiki/Io_uring) 

<div  align=center>
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/149773115-12090153-72dc-4d48-ab2a-fbb39a0d4503.png"/>
  
#### —— 2019 年 Linux 5.1 内核首次引入的高性能 异步 I/O 框架，能显著加速 I/O 密集型应用的性能。

</div>

### 文档
- 官方文档: [Efficient I/O with io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring.pdf)
- 其他文档：
  - [Improved Storage Performance Using the New Linux Kernel I.O Interface](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E6%A1%A3/Improved%20Storage%20Performance%20Using%20the%20New%20Linux%20Kernel%20I.O%20Interface.pdf)
  - [I/O-uring speed the RocksDB & TiKV](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E6%A1%A3/IO-uring%20speed%20the%20RocksDB%20%26%20TiKV.pdf)
  - [The Evolution of File Descriptor Monitoring in Linux](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E6%A1%A3/The%20Evolution%20of%20File%20Descriptor%20Monitoring%20in%20Linux.pdf)
  - [io_uring-BPF](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E6%A1%A3/io_uring-BPF.pdf)
  - [Enabling Financial-Grade Secure Infrastructure with Confidential Computing](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E6%A1%A3/Enabling%20Financial-Grade%20Secure%20Infrastructure%20with%20Confidential%20Computing.pdf)
  - [Boosting Compaction in B-Tree Based Key-Value Store by Exploiting Parallel Reads in Flash SSDs](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E6%A1%A3/Boosting%20Compaction%20in%20B-Tree%20Based%20Key-Value%20Store%20by%20Exploiting%20Parallel%20Reads%20in%20Flash%20SSDs.pdf)
  - [Programming Emerging Storage Interfaces](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E6%A1%A3/Programming%20Emerging%20Storage%20Interfaces.pdf)
  - [I/O is faster than the OS](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E6%A1%A3/O%20is%20faster%20than%20the%20OS.pdf)
  - [StefanMetzmacher_sambaxp2021_multichannel_io-uring-rev0-presentation](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E6%A1%A3/StefanMetzmacher_sambaxp2021_multichannel_io-uring-rev0-presentation.pdf)
  - [I/O Stack](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E6%A1%A3/O%20Stack.pdf)
  - [io_uring-徐浩-阿里云](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E6%A1%A3/io_uring-%E5%BE%90%E6%B5%A9-%E9%98%BF%E9%87%8C%E4%BA%91.pdf)

### 开源项目 
- [axboe/liburing](https://github.com/axboe/liburing): 这是 io_uring 库，liburing。liburing 为设置和拆掉 io_uring 实例，还有一个简化的接口不需要（或不想）处理完整内核的应用程序边执行。
- [shuveb/io_uring-by-example](https://github.com/shuveb/io_uring-by-example): 一个io_uring 示例的库
- [bytedance/monoio](https://github.com/bytedance/monoio): 基于io-uring的Rust异步运行时
- [spacejam/rio](https://github.com/spacejam/rio): Rust io_uring库，构建在libc上，线程和异步友好，抗误用
- [Iceber/iouring-go](https://github.com/Iceber/iouring-go): 提供易于使用的异步IO接口io_uring
- [frevib/io_uring-echo-server](https://github.com/frevib/io_uring-echo-server): io_uring echo server
- [hodgesds/iouring-go](https://github.com/hodgesds/iouring-go): Io_uring支持go
- [dshulyak/uring](https://github.com/dshulyak/uring): 用于io_uring框架的Golang库(无CGO)
- [quininer/ritsu](https://github.com/quininer/ritsu): 一个实验性的基于io-uring的异步运行时。
- [shuveb/loti-examples](https://github.com/shuveb/loti-examples): 源代码示例程序，从主的io_uring指南
- [xuanyi-fu/xynet](https://github.com/xuanyi-fu/xynet): 基于io_uring和c++ 20协程的网络库
- [KuiBaDB/kbio](https://github.com/KuiBaDB/kbio): 一个基于io_uring的异步IO框架
- [shuveb/loti](https://github.com/shuveb/loti): io_uring教程，例子和参考
- [MarkReedZ/mrloop](https://github.com/MarkReedZ/mrloop): C语言使用io_uring的事件循环
- [tchaloupka/during](https://github.com/tchaloupka/during): dlang io_uring包装
- [omegacoleman/arkio](https://github.com/omegacoleman/arkio): 基于异步IO的内核IO库
- [ciconia/awesome-io_uring](https://github.com/ciconia/awesome-io_uring): 一个很棒的io_uring资源、库和工具的分类集合。
- [ddeka0/AsyncIO](https://github.com/ddeka0/AsyncIO): 一个用于异步套接字服务器的CPP包装器，使用linux最新的io_uring API
- [uroni/fuseuring](https://github.com/uroni/fuseuring): 使用io_uring实现一个用户空间Linux fuse服务器
- [yunwei37/co-uring-WebServer](https://github.com/yunwei37/co-uring-WebServer): 一个使用io_uring和cpp20协同程序的c++高性能Web服务器
- [romange/helio](https://github.com/romange/helio): 一个基于io_uring Linux接口的现代后端开发框架
- [3541/short-circuit](https://github.com/3541/short-circuit): Linux高性能web服务器，基于io_uring构建。
- [anolis-os-archive/perf-test-for-io_uring](https://github.com/anolis-os-archive/perf-test-for-io_uring): 一个用于io_uring性能测试的框架。
- [BlazeWasHere/Cnidus](https://github.com/BlazeWasHere/Cnidus): 基于io_uring的C语言web框架。
- [AnSpake/osiris](https://github.com/AnSpake/osiris): 一个简单的服务器/客户端，使用 io_uring 来玩它

### 文章

- [io_uring 高效 IO](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring%20%E9%AB%98%E6%95%88%20IO.md)
- [ [译] Linux 异步 I_O 框架 io_uring：基本原理、程序示例与性能压测（2020）](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/%5B%E8%AF%91%5D%20Linux%20%E5%BC%82%E6%AD%A5%20I_O%20%E6%A1%86%E6%9E%B6%20io_uring%EF%BC%9A%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86%E3%80%81%E7%A8%8B%E5%BA%8F%E7%A4%BA%E4%BE%8B%E4%B8%8E%E6%80%A7%E8%83%BD%E5%8E%8B%E6%B5%8B%EF%BC%882020%EF%BC%89.md)
- [浅析开源项目之io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/%E6%B5%85%E6%9E%90%E5%BC%80%E6%BA%90%E9%A1%B9%E7%9B%AE%E4%B9%8Bio_uring.md)
- [io_uring 系统性整理](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring%20%E7%B3%BB%E7%BB%9F%E6%80%A7%E6%95%B4%E7%90%86.md)
- [io_uring（1） – 我们为什么会需要 io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring%EF%BC%881%EF%BC%89%20%E2%80%93%20%E6%88%91%E4%BB%AC%E4%B8%BA%E4%BB%80%E4%B9%88%E4%BC%9A%E9%9C%80%E8%A6%81%20io_uring.md)
- [io_uring（2）- 从创建必要的文件描述符 fd 开始](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring%EF%BC%882%EF%BC%89-%20%E4%BB%8E%E5%88%9B%E5%BB%BA%E5%BF%85%E8%A6%81%E7%9A%84%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6%20fd%20%E5%BC%80%E5%A7%8B.md)
- [下一代异步 IO io_uring 技术解密](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/%E4%B8%8B%E4%B8%80%E4%BB%A3%E5%BC%82%E6%AD%A5%20IO%20io_uring%20%E6%8A%80%E6%9C%AF%E8%A7%A3%E5%AF%86.md)
- [小谈io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/%E5%B0%8F%E8%B0%88io_uring.md)
- [智汇华云 | 新时代IO利器-io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/%E6%99%BA%E6%B1%87%E5%8D%8E%E4%BA%91%20%7C%20%E6%96%B0%E6%97%B6%E4%BB%A3IO%E5%88%A9%E5%99%A8-io_uring.md)
- [Linux 5.1 的 io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/Linux%205.1%20%E7%9A%84%20io_uring.md)
- [What is io_uring?](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/What%20is%20io_uring%3F)
- [io_uring_setup](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring_setup.md)
- [io_uring_enter](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring_enter.md)
- [io_uring_register](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring_register.md)
- [The Low-level io_uring Interface](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/The%20Low-level%20io_uring%20Interface.md)
- [Submission Queue Polling](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/Submission%20Queue%20Polling.md)
- [Efficient IO with io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/Efficient%20IO%20with%20io_uring.md)

### 视频(验证码：1024)

- [【2020】Speeding Up VM’s I_O Sharing Host's io_uring Queues With Guests by Stefano Garzarella](https://pan.baidu.com/s/1eQC_OQhfBnkd8t6NbBnseQ)
- [Asynchronous I_O and coroutines for smooth data streaming - Björn Fahller - NDC TechTown 2021](https://pan.baidu.com/s/1l5ZEOIwRKwWbnhZPnsj4hQ)
- [Guilherme Bernal - Reaching 200k req_s on a single core with io_uring - Crystal 1.0 Conference](https://pan.baidu.com/s/1EzFLmdpq9hEGhTsxhSF5NA)
- [Improved Storage Performance Using the New Linux Kernel I O Interface (SDC 2019)](https://pan.baidu.com/s/19vzNrSVAbjXP_XC5eNxj8g)
- [io_uring- BPF controlled I_O - Pavel Begunkov](https://pan.baidu.com/s/1g5KLbY9nQ2FIQkN7a3MGDw)
- [io_uring in QEMU- high-performance disk I_O for Linux](https://pan.baidu.com/s/1VFOdf6H6rRp3o2EHPmjLXA)
- [Kernel Recipes 2019 - Faster IO through io_uring](https://pan.baidu.com/s/1z7sFE2oFDcS6DAbod4UyOQ)
- [SDC2021- Samba Multi-Channel_io_uring Status Update](https://pan.baidu.com/s/1-YlabCqs03LS7nJxaOqPKQ)
- [Speeding Up VM’s I_O Sharing Host's io_uring Queues With Guests - Stefano Garzarella, Red Hat](https://pan.baidu.com/s/1QW3zvykzFwYKsMZUZK7orA)
- [USENIX ATC '19 - Asynchronous I_O Stack_ A Low-latency Kernel I_O Stack for Ultra-Low Latency SSDs](https://pan.baidu.com/s/1sWdfkSU9yjoY53A4wvkcfQ)
- [来自阿里云的 Linux 内核 io_uring 介绍与实践](https://pan.baidu.com/s/1FykA5evNh3O3JK4Cu9fs0Q)

## 🔥 cgroup

## 🔥 ebpf

## 🔥 llvm

## 🔥 kvm

## 🔥 ceph

## 🔥 fuse
