# 🔰 深挖 Linux 内核的新功能特性，以 io_uring, cgroup, ebpf, llvm 为代表，包含开源项目，代码案例，文章，视频，架构脑图等

所有数据来源于互联网。所谓取之于互联网，用之于互联网。

如果涉及版权侵犯，请邮件至 wchao_isvip@163.com ，我们将第一时间处理。

如果您对我们的项目表示赞同与支持，欢迎您 lssues我们，或者邮件 wchao_isvip@163.com 我们，更加欢迎您 pull requests 加入我们。

感谢您的支持！


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
- [axboe/liburing](https://github.com/axboe/liburing): io_uring 库，liburing为设置和拆掉 io_uring 实例，还有一个简化接口不需要（或不想）处理完整内核的应用程序边执行。
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
- [AnSpake/osiris](https://github.com/AnSpake/osiris): 一个简单的服务器/客户端，使用io_uring
- [PanamaUring](https://github.com/dreamlike-ocean/PanamaUring): 基于Java新FFI接口的io_uring绑定且实现了几乎全部的async版本的IO原语（无JNI）

### 文章

- [io_uring 高效 IO](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring%20%E9%AB%98%E6%95%88%20IO.md)
- [ [译] Linux 异步 I_O 框架 io_uring：基本原理、程序示例与性能压测（2020）](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/%5B%E8%AF%91%5D%20Linux%20%E5%BC%82%E6%AD%A5%20I_O%20%E6%A1%86%E6%9E%B6%20io_uring%EF%BC%9A%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86%E3%80%81%E7%A8%8B%E5%BA%8F%E7%A4%BA%E4%BE%8B%E4%B8%8E%E6%80%A7%E8%83%BD%E5%8E%8B%E6%B5%8B%EF%BC%882020%EF%BC%89.md)
- [浅析开源项目之io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/%E6%B5%85%E6%9E%90%E5%BC%80%E6%BA%90%E9%A1%B9%E7%9B%AE%E4%B9%8Bio_uring.md)
- [io_uring 系统性整理](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring%20%E7%B3%BB%E7%BB%9F%E6%80%A7%E6%95%B4%E7%90%86.md)
- [io_uring（1） – 我们为什么会需要 io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring%EF%BC%881%EF%BC%89%20%E2%80%93%20%E6%88%91%E4%BB%AC%E4%B8%BA%E4%BB%80%E4%B9%88%E4%BC%9A%E9%9C%80%E8%A6%81%20io_uring.md)
- [io_uring（2）- 从创建必要的文件描述符 fd 开始](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring%EF%BC%882%EF%BC%89-%20%E4%BB%8E%E5%88%9B%E5%BB%BA%E5%BF%85%E8%A6%81%E7%9A%84%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6%20fd%20%E5%BC%80%E5%A7%8B.md)
- [下一代异步 IO io_uring 技术解密](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/%E4%B8%8B%E4%B8%80%E4%BB%A3%E5%BC%82%E6%AD%A5%20IO%20io_uring%20%E6%8A%80%E6%9C%AF%E8%A7%A3%E5%AF%86.md)
- [小谈io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/%E5%B0%8F%E8%B0%88io_uring.md)
- [智汇华云-新时代IO利器-io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/%E6%99%BA%E6%B1%87%E5%8D%8E%E4%BA%91-%E6%96%B0%E6%97%B6%E4%BB%A3IO%E5%88%A9%E5%99%A8-io_uring.md)
- [Linux 5.1 的 io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/Linux%205.1%20%E7%9A%84%20io_uring.md)
- [What is io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/What%20is%20io_uring)
- [io_uring_setup](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring_setup.md)
- [io_uring_enter](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring_enter.md)
- [io_uring_register](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring_register.md)
- [The Low-level io_uring Interface](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/The%20Low-level%20io_uring%20Interface.md)
- [Submission Queue Polling](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/Submission%20Queue%20Polling.md)
- [Efficient IO with io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/Efficient%20IO%20with%20io_uring.md)

### 视频(提取码：1024)

- [Speeding Up VM’s I_O Sharing Host's io_uring Queues With Guests by Stefano Garzarella【2020】](https://pan.baidu.com/s/1eQC_OQhfBnkd8t6NbBnseQ)
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

## 🔥 [cgroup](https://zh.wikipedia.org/wiki/Cgroups)

<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150078568-4f0de590-793f-41b9-9038-cc8b44894cfb.png"/>
  
#### —— 限制、控制与分离一个进程组的资源（如CPU、内存、磁盘输入输出等）。

</div>

### 文档
- 官方文档:
  - [Control Groups definition, implementation details, examples and API](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/cgroups.txt)
  - [CPU Accounting Controller; account CPU usage for groups of tasks](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/cpuacct.txt)
  - [documents the cpusets feature; assign CPUs and Mem to a set of tasks](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/cpusets.txt)
  - [Device Whitelist Controller; description, interface and security](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/devices.txt)
  - [checkpointing; rationale to not use signals, interface](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/freezer-subsystem.txt)
  - [Memory Resource Controller; implementation details](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/memcg_test.txt)
  - [Memory Resource Controller; design, accounting, interface, testing](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/memory.txt)
  - [Resource Counter API](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/resource_counter.txt)

- 其他文档：
  - [cgroups介绍](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/cgroups%E4%BB%8B%E7%BB%8D.pdf)
  - [CgroupMemcgMaster](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/CgroupMemcgMaster.pdf)
  - [Resource Management](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/Resource%20Management.pdf)
  - [Challenges with the memory resource controller and its performance](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/%20Challenges%20with%20the%20memory%20resource%20controller%20and%20its%20performance.pdf)
  - [Ressource Management in Linux with Control Groups](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/Ressource%20Management%20in%20Linux%20with%20Control%20Groups.pdf)
  - [System Programming for Linux Containers Control Groups (cgroups)](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/System%20Programming%20for%20Linux%20Containers%20Control%20Groups%20(cgroups).pdf)
  - [Managing Resources with cgroups](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/Managing%20Resources%20with%20cgroups.pdf)
  - [5 years of cgroup v2](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/5%20years%20of%20cgroup%20v2.pdf)
  - [Linux’s new unified control group system](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/%20Linux%E2%80%99s%20new%20unified%20control%20group%20system.pdf)
  - [cgroups_intro](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/cgroups_intro.pdf)
  - [red_hat_enterprise_linux-6-resource_management_guide-en-us](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/red_hat_enterprise_linux-6-resource_management_guide-en-us.pdf)
  - [An introduction to Control Groups (cgroups)](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/An%20introduction%20to%20Control%20Groups%20(cgroups).pdf)
  - [Using Linux Control Groups and Systemd to Manage CPU Time and Memory](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/%20Using%20Linux%20Control%20Groups%20and%20Systemd%20to%20Manage%20CPU%20Time%20and%20Memory.pdf)
  - [An introduction to cgroups and cgroupspy](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/An%20introduction%20to%20cgroups%20and%20cgroupspy.pdf)


### 开源项目 
- [containerd/cgroups](https://github.com/containerd/cgroups): 用于创建、管理、检查和销毁cgroup。cgroup上设置的资源格式使用这里找到的OCI运行时规范。
- [mhausenblas/cinf](https://github.com/mhausenblas/cinf): 一个查看命名空间和cgroups的命令行工具
- [flouthoc/vas-quod](https://github.com/flouthoc/vas-quod): 用Rust编写的一个极小的容器运行时
- [poelzi/ulatencyd](https://github.com/poelzi/ulatencyd): 使用cgroups最小化linux系统延迟的守护进程
- [haosdent/jcgroup](https://github.com/haosdent/jcgroup): jcgroup是JVM上的cgroup包装器。您可以使用这个库来限制线程的CPU共享、磁盘I/O速度、网络带宽等。
- [kinvolk/traceloop](https://github.com/kinvolk/traceloop): 使用BPF和可重写的环形缓冲区跟踪cgroup中的系统调用
- [tianon/cgroupfs-mount](https://github.com/tianon/cgroupfs-mount): 挂载cgroupfs (v1)层次结构的简单(过时)脚本，特别是用于Debian打包的结构化脚本
- [francisbouvier/cgroups](https://github.com/francisbouvier/cgroups): 一个库来管理cgroups Linux内核特性
- [bpowers/mstat](https://github.com/bpowers/mstat): 这个工具运行在Linux上，利用cgroups内核API(也被Docker等容器基础设施使用)来记录一组进程随时间的内存使用情况。




### 文章

- [Linux cgroups 概述](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/linux%20cgroups%20%E6%A6%82%E8%BF%B0.md)
- [【译】Control Group v2（cgroupv2 权威指南）（KernelDoc, 2021）](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/%5B%E8%AF%91%5D%20Control%20Group%20v2%EF%BC%88cgroupv2%20%E6%9D%83%E5%A8%81%E6%8C%87%E5%8D%97%EF%BC%89%EF%BC%88KernelDoc%2C%202021%EF%BC%89.md)
- [How I Used CGroups to Manage System Resources](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/How%20I%20Used%20CGroups%20to%20Manage%20System%20Resources.md)
- [Cgroups控制cpu，内存，io示例](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/Cgroups%E6%8E%A7%E5%88%B6cpu%EF%BC%8C%E5%86%85%E5%AD%98%EF%BC%8Cio%E7%A4%BA%E4%BE%8B.md)
- [Linux Control Groups V1 和 V2 原理和区别](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/Linux%20Control%20Groups%20V1%20%E5%92%8C%20V2%20%E5%8E%9F%E7%90%86%E5%92%8C%E5%8C%BA%E5%88%AB.md)
- [Linux资源管理之cgroups简介](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/Linux%E8%B5%84%E6%BA%90%E7%AE%A1%E7%90%86%E4%B9%8Bcgroups%E7%AE%80%E4%BB%8B.md)
- [彻底搞懂容器技术的基石： cgroup](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/%E5%BD%BB%E5%BA%95%E6%90%9E%E6%87%82%E5%AE%B9%E5%99%A8%E6%8A%80%E6%9C%AF%E7%9A%84%E5%9F%BA%E7%9F%B3%EF%BC%9A%20cgroup.md)
- [深入理解 Linux Cgroup 系列（一）：基本概念](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%20Linux%20Cgroup%20%E7%B3%BB%E5%88%97%EF%BC%88%E4%B8%80%EF%BC%89%EF%BC%9A%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5.md)
- [深入理解 Linux Cgroup 系列（二）：玩转 CPU](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%20Linux%20Cgroup%20%E7%B3%BB%E5%88%97%EF%BC%88%E4%BA%8C%EF%BC%89%EF%BC%9A%E7%8E%A9%E8%BD%AC%20CPU.md)
- [深入理解 Linux Cgroup 系列（三）：内存](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%20Linux%20Cgroup%20%E7%B3%BB%E5%88%97%EF%BC%88%E4%B8%89%EF%BC%89%EF%BC%9A%E5%86%85%E5%AD%98.md)
- [Cgroup - 从CPU资源隔离说起](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/Cgroup%20-%20%E4%BB%8ECPU%E8%B5%84%E6%BA%90%E9%9A%94%E7%A6%BB%E8%AF%B4%E8%B5%B7.md)
- [Cgroup - Linux内存资源管理](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/Cgroup%20-%20Linux%E5%86%85%E5%AD%98%E8%B5%84%E6%BA%90%E7%AE%A1%E7%90%86.md)
- [Cgroup - Linux的IO资源隔离](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/Cgroup%20-%20Linux%E7%9A%84IO%E8%B5%84%E6%BA%90%E9%9A%94%E7%A6%BB.md)
- [Cgroup - Linux的网络资源隔离](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/Cgroup%20-%20Linux%E7%9A%84%E7%BD%91%E7%BB%9C%E8%B5%84%E6%BA%90%E9%9A%94%E7%A6%BB.md)
- [用 cgroups 管理 cpu 资源](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/%E7%94%A8%20cgroups%20%E7%AE%A1%E7%90%86%20cpu%20%E8%B5%84%E6%BA%90.md)
- [用 cgruops 管理进程内存占用](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/%E7%94%A8%20cgruops%20%E7%AE%A1%E7%90%86%E8%BF%9B%E7%A8%8B%E5%86%85%E5%AD%98%E5%8D%A0%E7%94%A8.md)
- [用 cgroups 管理进程磁盘 io](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/cgroups%20%E7%AE%A1%E7%90%86%E8%BF%9B%E7%A8%8B%E7%A3%81%E7%9B%98%20io.md)


### 视频

- [Containers_ cgroups, Linux kernel namespaces, ufs, Docker, and intro to Kubernetes pods](https://pan.baidu.com/s/1dfmOxOESpgT9rj4VH1BmRA)---提取码: k4hn
- [Understanding and Working with the Cgroups Interface - Michael Anderson, The PTR Group, LLC](https://pan.baidu.com/s/1wD5MRvHheJv1P8i1iQmosQ)---提取码: 54vs
- [Linux Container Primitives- cgroups, namespaces, and more!](https://pan.baidu.com/s/1LZ9Ff1EuTArxcv6e0c8-2A)---提取码: cjwd
- [Cgroups, namespaces, and beyond](https://pan.baidu.com/s/1IjOURq5X6TEwZn6G5LUhog)---提取码: at6x 
- [Kubernetes On Cgroup v2 - Giuseppe Scrivano, Red Hat](https://pan.baidu.com/s/1apHDcsiCpiZITd_TwfezCg)---提取码: 552y
- [Cgroup Slab Memory Controller and Time Namespace - DevConf.CZ 2021](https://pan.baidu.com/s/1qhVtHJtQjM-7mJVQVDMPwg)---提取码: gayh
- [Modern Linux Servers with cgroups - Brandon Philips, CoreOS](https://pan.baidu.com/s/1okbzLkfA7d0uKJRyj3iyDg)---提取码: afm1
- [LISA21 - 5 Years of Cgroup v2- The Future of Linux Resource Control](https://pan.baidu.com/s/1AGo7vUC0F0uKO5gCd4wVxg)---提取码: ygrv
- [Limit CPU usage on Ubuntu with Systemd cgroups](https://pan.baidu.com/s/17gB4Lv4LyznfMwTxd9Ae_Q)---提取码: ktva
- [What's new in control groups (cgroups) version 2](https://pan.baidu.com/s/1r3V4Htltuy58OUmXGC5aXQ)---提取码: w2tz

## 🔥 [ebpf](https://ebpf.io/)

<div  align=center>
  
<img width="60%" height="60%" src="https://ebpf.io/static/logo-big-9cf8920e80cdc57e6ea60825ebe287ca.png"/>
  
#### —— Linux 内核中顶级子模块
</div>

### 文档
- 官方文档: 
  - Linux 内核：https://www.kernel.org/doc/Documentation/networking/filter.txt and https://www.kernel.org/doc/html/latest/bpf/#
  - 开发QA: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/bpf/bpf_devel_QA.rst
  - eBPF-Helpers：https://github.com/iovisor/bpf-docs/blob/master/bpf_helpers.rst/
- 其他文档：

  - [iovisor/bpf-docs](https://github.com/iovisor/bpf-docs): 列出了 eBPF opcode，项目是 iovisor 总结的系列文档、pre。
  - [Advanced_BPF_Kernel_Features_for_the_Container_Age_FOSDEM](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/Advanced_BPF_Kernel_Features_for_the_Container_Age_FOSDEM.pdf)
  - [BPF to eBPF](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/BPF%20to%20eBPF.pdf)
  - [Calico-eBPF-Dataplane-CNCF-Webinar-Slides](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/Calico-eBPF-Dataplane-CNCF-Webinar-Slides.pdf)
  - [Combining System Visibility and Security Using eBPF](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/Combining%20System%20Visibility%20and%20Security%20Using%20eBPF.pdf)
  - [DPDK+eBPF](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/DPDK%2BeBPF.pdf)
  - [Experience and Lessons Learned](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/Experience%20and%20Lessons%20Learned.pdf)
  - [Fast Packet Processing using eBPF and XDP](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/Fast%20Packet%20Processing%20using%20eBPF%20and%20XDP.pdf)
  - [Kernel Tracing With eBPF](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/Kernel%20Tracing%20With%20eBPF.pdf)
  - [Kernel analysis using eBPF](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/Kernel%20analysis%20using%20eBPF.pdf)
  - [Making the Linux TCP stack more extensible with eBPF](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/Making%20the%20Linux%20TCP%20stack%20more%20extensible%20with%20eBPF.pdf)
  - [Performance Analysis Superpowers with Linux eBPF](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/Performance%20Analysis%20Superpowers%20with%20Linux%20eBPF.pdf)
  - [Performance Implications of Packet Filtering with Linux eBPF](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/Performance%20Implications%20of%20Packet%20Filtering%20with%20Linux%20eBPF.pdf)
  - [The Next Linux Superpower eBPF Primer](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/The%20Next%20Linux%20Superpower%20eBPF%20Primer.pdf)
  - [eBPF - From a Programmer’s Perspective](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/eBPF%20-%20From%20a%20Programmer%E2%80%99s%20Perspective.pdf)
  - [eBPF In-kernel Virtual Machine & Cloud Computin](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/eBPF%20In-kernel%20Virtual%20Machine%20%26%20Cloud%20Computin.pdf)
  - [eBPF for perfomance analysis and networking](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/eBPF%20for%20perfomance%20analysis%20and%20networking.pdf)
  - [eBPF in CPU Scheduler](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/eBPF%20in%20CPU%20Scheduler.pdf)
  - [eBPF-based Content and Computation-aware Communication for Real-time Edge Computing](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/eBPF-based%20Content%20and%20Computation-aware%20Communication%20for%20Real-time%20Edge%20Computing.pdf)

### 开源项目 

- [cilium/cilium](https://github.com/cilium/cilium): 用于提供、保护和观察容器工作负载之间的网络连接——云原生，并由革命性的内核技术eBPF提供支持,https://cilium.io/
- [BPF Compiler Collection (BCC)](https://github.com/iovisor/bcc): BCC -基于bpf的Linux IO分析、联网、监控等工具
- [bpftrace](https://github.com/iovisor/bpftrace): Linux eBPF的高级跟踪语言
- [Falco](https://github.com/falcosecurity/falco): 一种行为活动监视器，旨在检测应用程序中的异常活动。Falco在ebp的帮助下在Linux内核层对系统进行审计。它通过其他输入流(如容器运行时度量和Kubernetes度量)丰富了收集到的数据，并允许持续监视和检测容器、应用程序、主机和网络活动。
- [Katran](https://github.com/facebookincubator/katran): 高性能的四层负载均衡器
- [LLVM Compiler](https://github.com/llvm/llvm-project/): 一个模块化和可重用的编译器和工具链技术的集合。
- [microsoft/ebpf-for-windows](https://github.com/microsoft/ebpf-for-windows): 运行在Windows上的eBPF实现
- [aquasecurity/libbpfgo](https://github.com/aquasecurity/libbpfgo): 一个用于Linux ebbpf项目的Go库。
- [aquasecurity/tracee](https://github.com/aquasecurity/tracee): Linux的运行时安全和取证工具。
- [libbpf/libbpf](https://github.com/libbpf/libbpf): libbpf是一个基于C/ c++的库，作为上游Linux内核的一部分进行维护。它包含一个eBPF加载器，它接管处理LLVM生成的eBPF ELF文件，以便将其加载到内核中。
- [libbpf/libbpf-rs](https://github.com/libbpf/libbpf-rs): Rust生态系统的最小和固执的epf工具
- [foniod/redbpf](https://github.com/foniod/redbpf): Rust库用于构建和运行BPF/eBPF模块
- [aya-rs/aya](https://github.com/aya-rs/aya): 一个用于Rust编程语言的eBPF库，其构建的重点是开发人员的体验和可操作性。
- [cilium/hubble](https://github.com/cilium/hubble): 使用eBPF的Kubernetes网络、服务和安全可观测性
- [kubearmor/KubeArmor](https://github.com/kubearmor/KubeArmor): 一个云本地运行时安全强制系统，它在系统级别限制容器和节点的行为(如进程执行、文件访问和网络操作)。
- [iovisor/kubectl-trace](https://github.com/iovisor/kubectl-trace): 使用kubectl在kubernetes集群上调度bpftrace程序
- [iovisor/ply](https://github.com/iovisor/ply): 一款基于eBPF的Linux动态跟踪软件。


### 文章

- [什么是 eBPF](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/%E4%BB%80%E4%B9%88%E6%98%AF%20eBPF.md)
- [eBPF详解](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%E8%AF%A6%E8%A7%A3.md)
- [BPF 和 eBPF 初探](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/BPF%20%E5%92%8C%20eBPF%20%E5%88%9D%E6%8E%A2.md)
- [Linux 内核监测技术 eBPF](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/Linux%20%E5%86%85%E6%A0%B8%E7%9B%91%E6%B5%8B%E6%8A%80%E6%9C%AF%20eBPF.md)
- [eBPF 如何简化服务网格](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%20%E5%A6%82%E4%BD%95%E7%AE%80%E5%8C%96%E6%9C%8D%E5%8A%A1%E7%BD%91%E6%A0%BC.md)
- [eBPF 用户空间虚拟机实现相关](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%20%E7%94%A8%E6%88%B7%E7%A9%BA%E9%97%B4%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AE%9E%E7%8E%B0%E7%9B%B8%E5%85%B3.md)
- [基于 eBPF 实现容器运行时安全](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/%E5%9F%BA%E4%BA%8E%20eBPF%20%E5%AE%9E%E7%8E%B0%E5%AE%B9%E5%99%A8%E8%BF%90%E8%A1%8C%E6%97%B6%E5%AE%89%E5%85%A8.md)
- [深入理解 Cilium 的 eBPF 收发包路径](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%20Cilium%20%E7%9A%84%20eBPF%20%E6%94%B6%E5%8F%91%E5%8C%85%E8%B7%AF%E5%BE%84.md)
- [eBPF 概述，第 1 部分：介绍](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%20%E6%A6%82%E8%BF%B0%EF%BC%8C%E7%AC%AC%201%20%E9%83%A8%E5%88%86%EF%BC%9A%E4%BB%8B%E7%BB%8D.md)
- [eBPF 概述，第 2 部分：机器和字节码](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%20%E6%A6%82%E8%BF%B0%EF%BC%8C%E7%AC%AC%202%20%E9%83%A8%E5%88%86%EF%BC%9A%E6%9C%BA%E5%99%A8%E5%92%8C%E5%AD%97%E8%8A%82%E7%A0%81.md)
- [eBPF 概述，第 3 部分：软件开发生态](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%20%E6%A6%82%E8%BF%B0%EF%BC%8C%E7%AC%AC%203%20%E9%83%A8%E5%88%86%EF%BC%9A%E8%BD%AF%E4%BB%B6%E5%BC%80%E5%8F%91%E7%94%9F%E6%80%81.md)
- [eBPF 概述，第 4 部分：在嵌入式系统运行](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%20%E6%A6%82%E8%BF%B0%EF%BC%8C%E7%AC%AC%204%20%E9%83%A8%E5%88%86%EF%BC%9A%E5%9C%A8%E5%B5%8C%E5%85%A5%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%BF%90%E8%A1%8C.md)
- [eBPF 概述，第 5 部分：跟踪用户进程](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%20%E6%A6%82%E8%BF%B0%EF%BC%8C%E7%AC%AC%205%20%E9%83%A8%E5%88%86%EF%BC%9A%E8%B7%9F%E8%B8%AA%E7%94%A8%E6%88%B7%E8%BF%9B%E7%A8%8B.md)
- [【译】大规模微服务利器：eBPF + KubernetesKubeCon, 2020](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/%E3%80%90%E8%AF%91%5D%E3%80%91%E5%A4%A7%E8%A7%84%E6%A8%A1%E5%BE%AE%E6%9C%8D%E5%8A%A1%E5%88%A9%E5%99%A8%EF%BC%9AeBPF%20%2B%20Kubernetes%EF%BC%88KubeCon%2C%202020%EF%BC%89.md)

### 视频

- [Netflix talks about Extended BPF - A new software type](https://pan.baidu.com/s/1VD-dsBheyJmDUhiIUJiQOw)---提取码: 83sv
- [containers_ebpf_kernel](https://pan.baidu.com/s/1NFzeWCJHmsXnzmDTHnt9vg)---提取码: hxkt

## 🔥 [llvm](https://llvm.org/)

<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150629940-85fa8f28-dfe9-4024-a97f-c8489471f7e9.png"/>
  
#### —— 模块化、可重用的编译器以及工具链技术的集合
</div>

### 文档
- 官方文档:
  - [LLVM: A Compilation Framework for Lifelong Program Analysis & Transformation](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E6%A1%A3/%20A%20Compilation%20Framework%20for%20Lifelong%20Program%20Analysis%20%26%20Transformation.pdf)
  - [Introduction to the LLVM Compiler System](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E6%A1%A3/Introduction%20to%20the%20LLVM%20Compiler%20System.pdf)
  - [LLVM语言参考手册](https://web.archive.org/web/20120611064155/http://llvm.org/docs/LangRef.html)
  - [LLVM语言参考手册-中文版](https://llvm.liuxfe.com/docs/langref/)
  - [入门LLVM核心库](https://getting-started-with-llvm-core-libraries-zh-cn.readthedocs.io/zh_CN/latest/)
- 用户指南：
  - [使用CMake构建LLVM](https://llvm.org/docs/CMake.html): 使用CMake构建系统的主要入门指南的附录。
  - [在ARM平台上构建LLVM指南](https://llvm.org/docs/HowToBuildOnARM.html): 关于在ARM上构建和测试LLVM/Clang的注意事项。
  - [如何使用配置文件引导优化构建Clang和LLVM](https://llvm.org/docs/HowToBuildWithPGO.html): 使用PGO构建LLVM/Clang的注意事项。
  - [如何为ARM平台交叉编译compiler-rt Builtins](https://llvm.org/docs/HowToCrossCompileBuiltinsOnArm.html): 关于交叉构建和测试ARM的编译器-rt内置函数的注意事项。
  - [如何使用Clang/LLVM交叉编译Clang/LLVM](https://llvm.org/docs/HowToCrossCompileLLVM.html)：关于交叉构建和测试LLVM / Clang的注意事项。
  - [使用Microsoft Visual Studio开始使用LLVM系统](https://llvm.org/docs/GettingStartedVS.html)：Windows上使用Visual Studio的主要入门指南的附录。
  - [LLVM的分析和转换Passes](https://llvm.org/docs/Passes.html):LLVM中实现的优化和分析列表
  - [当前版本的发布说明](https://llvm.org/docs/ReleaseNotes.html):这描述了新功能，已知错误和其他限制。
  - [如何提交LLVM错误报告](https://llvm.org/docs/HowToSubmitABug.html): 有关正确提交有关您在LLVM系统中遇到的任何错误的信息的说明。
  - [sphinx模板快速入门](https://llvm.org/docs/SphinxQuickstartTemplate.html)：使用LLVM测试基础结构的参考手册。
  - [LLVM测试套件基础结构指南](https://llvm.org/docs/TestingGuide.html)：使用LLVM测试基础结构的参考手册。
  - [LLVM测试套件使用指南](https://llvm.org/docs/TestSuiteGuide.html)：描述如何编译和运行测试套件基准测试。
  - [如何构建C，C ++，ObjC和ObjC ++前端](https://clang.llvm.org/get_started.html)：从源代码构建clang前端的说明。
  - [LLVM词典](https://llvm.org/docs/Lexicon.html)：LLVM中使用的首字母缩略词，术语和概念的定义。
  - [如何将构建配置添加到LLVM Buildbot基础结构](https://llvm.org/docs/HowToAddABuilder.html):有关将新构建器添加到LLVM buildbot master的说明。
  - [YAML I/O](https://llvm.org/docs/YamlIO.html)：使用LLVM的YAML I/O库的参考指南。
  - [前端作者的性能提示](https://llvm.org/docs/Frontend/PerformanceTips.html)：前端作者关于如何生成IR的技巧的集合，LLVM能够有效地优化。
  - [Dockerfiles用于构建LLVM的指南](https://llvm.org/docs/Docker.html)：使用随LLVM提供的Dockerfiles的参考。
- 编程文档：
  - [LLVM扩展](https://llvm.org/docs/Extensions.html)：LLVM特定的工具和格式扩展LLVM寻求兼容性。
  - [CommandLine 2.0库手册](https://llvm.org/docs/CommandLine.html)：提供有关使用命令行解析库的信息。
  - [LLVM编码标准](https://llvm.org/docs/CodingStandards.html)：详细介绍了LLVM编码标准，并提供了有关编写高效C ++代码的有用信息。
  - [如何为类层次结构设置LLVM样式的RTTI](https://llvm.org/docs/HowToSetUpLLVMStyleRTTI.html)：如何让`isa<>`，`dyn_cast<>`等可供您的类层次的客户。
  - [扩展LLVM：添加指令，内在函数，类型等](https://llvm.org/docs/ExtendingLLVM.html)：在这里查看如何向LLVM添加指令和内在函数。
  - [libFuzzer - 用于覆盖引导的模糊测试的库](https://llvm.org/docs/LibFuzzer.html)：用于编写进程中引导模糊器的库
  - [模糊LLVM库和工具](https://llvm.org/docs/FuzzingLLVM.html)：有关编写和使用Fuzzers查找LLVM中的错误的信息.
  - [Scudo硬化分配器](https://llvm.org/docs/ScudoHardenedAllocator.html)：一个实现安全加固的malloc（）的库。
- 子系统文档
  - [编写LLVM Passes](https://llvm.org/docs/WritingAnLLVMPass.html)：有关如何编写LLVM转换和分析的信息
  - [编写LLVM后端](https://llvm.org/docs/WritingAnLLVMBackend.html)：有关如何为机器目标编写LLVM后端的信息
  - [LLVM与目标无关的代码生成器](https://llvm.org/docs/CodeGenerator.html)：LLVM代码生成器的设计和实现。如果您正在将LLVM重新定位到新架构，设计新的codegen传递或增强现有组件，则非常有用。
  - [机器IR（MIR）格式参考手册](https://llvm.org/docs/MIRLangRef.html)：MIR序列化格式的参考手册，用于测试LLVM的代码生成过程。
  - [TableGen](https://llvm.org/docs/TableGen/index.html)：描述了TableGen工具，LLVM代码生成器大量使用它。
  - [LLVM别名分析基础结构](https://llvm.org/docs/AliasAnalysis.html)：有关如何编写新别名分析实现或如何使用现有分析的信息。
  - [MemorySSA](https://llvm.org/docs/MemorySSA.html)：有关LLVM中的MemorySSA实用程序的信息，以及如何使用它。
  - [使用LLVM进行垃圾收集](https://llvm.org/docs/GarbageCollection.html)：接口源语言编译器应该用于编译GC程序。
  - [使用LLVM进行源级别调试](https://llvm.org/docs/SourceLevelDebugging.html)：本文档描述了LLVM源代码级调试器背后的设计和理念。
  - [LLVM中的自动矢量化](https://llvm.org/docs/Vectorizers.html)：本文档描述了LLVM中矢量化的当前状态
  - [LLVM中的异常处理](https://llvm.org/docs/ExceptionHandling.html)：本文档描述了LLVM中异常处理的设计和实现
  - [如何添加一个受约束的浮点内在函数](https://llvm.org/docs/AddingConstrainedIntrinsics.html)：在LLVM中添加新的约束数学内在时，提供必要的步骤。
  - [LLVM bugpoint工具：设计和使用](https://llvm.org/docs/Bugpoint.html)：自动错误查找器和测试用例减少器描述和使用信息
  - [LLVM Bitcode文件格式](https://llvm.org/docs/BitCodeFormat.html)：这描述了用于LLVM“bc”文件的文件格式和编码。
  - [支持库](https://llvm.org/docs/SupportLibrary.html)：本文档描述了LLVM支持库（lib/Support）以及如何使LLVM源代码可移植
  - [LLVM链接时间优化：设计和实现](https://llvm.org/docs/LinkTimeOptimization.html)：本文档描述了LLVM模块间优化器与链接器及其设计之间的接口
  - [LLVM黄金插件](https://llvm.org/docs/GoldPlugin.html)：如何在Linux上使用链接时优化来构建程序。
  - [使用GDB调试JIT-ed代码](https://llvm.org/docs/DebuggingJITedCode.html):如何使用GDB调试JITed代码。
  - [MCJIT设计与实施](https://llvm.org/docs/MCJITDesignAndImplementation.html)：描述了MCJIT执行引擎的内部工作原理
  - [LLVM分支权重元数据](https://llvm.org/docs/BranchWeightMetadata.html)：提供有关分支预测信息的信息。
  - [LLVM块频率术语](https://llvm.org/docs/BlockFrequencyTerminology.html):提供有关BlockFrequencyInfo 分析过程中使用的术语的信息
  - [LLVM中的分段堆栈](https://llvm.org/docs/SegmentedStacks.html):本文档描述了分段堆栈以及它们在LLVM中的使用方式
  - [LLVM的可选丰富的反汇编输出](https://llvm.org/docs/MarkedUpDisassembly.html):本文档介绍了可选的丰富反汇编输出语法
  - [如何使用属性](https://llvm.org/docs/HowToUseAttributes.html)：回答有关新属性基础结构的一些问题。
  - [NVPTX后端用户指南](https://llvm.org/docs/NVPTXUsage.html)：本文档描述了使用NVPTX后端编译GPU内核。
  - [AMDGPU后端用户指南](https://llvm.org/docs/AMDGPUUsage.html):本文档描述了使用AMDGPU后端编译GPU内核。
  - [LLVM中的堆栈映射和补丁点](https://llvm.org/docs/StackMaps.html):LLVM支持将指令地址映射到值的位置并允许修补代码。
  - [在big endian模式下使用ARM NEON指令](https://llvm.org/docs/BigEndianNEON.html)：LLVM支持在大端ARM目标上生成NEON指令有点不直观。本文档解释了实施和理由。
  - [LLVM代码覆盖映射格式](https://llvm.org/docs/CoverageMappingFormat.html):LLVM代码覆盖映射格式
  - [LLVM中的垃圾收集安全点](https://llvm.org/docs/Statepoints.html)：这描述了一组垃圾收集支持的实验扩展。
  - [MergeFunctions Pass，它是如何工作的](https://llvm.org/docs/MergeFunctions.html)：描述合并优化的函数。
  - [InAlloca属性的设计和使用](https://llvm.org/docs/InAlloca.html)：inalloca参数属性的描述。
  - [FaultMaps和隐式检查](https://llvm.org/docs/FaultMaps.html)：LLVM支持折叠控制流入错误机器指令。
  - [用clang编译CUDA](https://llvm.org/docs/CompileCudaWithLLVM.html)：LLVM对CUDA的支持。
  - [LLVM中的协同程序](https://llvm.org/docs/Coroutines.html):LLVM中的协同程序.
  - [全局指令选择](https://llvm.org/docs/GlobalISel.html)：这描述了原型指令选择替换GlobalISel
  - [XRay仪表](https://llvm.org/docs/XRay.html)：有关如何在LLVM中使用XRay的高级文档。
  - [使用XRay进行调试](https://llvm.org/docs/XRayExample.html)：如何使用XRay调试应用程序的示例。
  - [Microsoft PDB文件格式](https://llvm.org/docs/PDB/index.html)：Microsoft PDB（程序数据库）文件格式的详细说明。
  - [控制流程验证工具设计文档](https://llvm.org/docs/CFIVerify.html)：控制流完整性验证工具的说明
  - [投机负荷强化](https://llvm.org/docs/SpeculativeLoadHardening.html)：Spectre v1的推测负载强化缓解的描述
  - [堆栈安全分析](https://llvm.org/docs/StackSafetyAnalysis.html)：本文档描述了局部变量的堆栈安全性分析的设计。



### LLVM命令指南

#### 基本命令

| 命令                                                         | 说明                              |
| :----------------------------------------------------------- | :-------------------------------- |
| [llvm-as](https://llvm.liuxfe.com/docs/man/llvm-as.html)     | LLVM汇编器                        |
| [llvm-dis](https://llvm.liuxfe.com/docs/man/llvm-dis.html)   | LLVM反汇编器                      |
| [opt](https://llvm.liuxfe.com/docs/man/opt.html)             | LLVM优化器                        |
| [llc](https://llvm.liuxfe.com/docs/man/llc.html)             | LLVM静态编译器                    |
| [lli](https://llvm.liuxfe.com/docs/man/lli.html)             | LLVM字节码解释器                  |
| [llvm-link](https://llvm.liuxfe.com/docs/man/llvm-link.html) | LLVM字节码连接器                  |
| [llvm-lib](https://llvm.liuxfe.com/docs/man/llvm-lib.html)   | LLVM的与lib.exe兼用的库工具       |
| [llvm-lipo](https://llvm.liuxfe.com/docs/man/llvm-lipo.html) | 用于处理通用二进制文件的LLVM工具  |
| [llvm-config](https://llvm.liuxfe.com/docs/man/llvm-config.html) | 打印LLVM编译选项                  |
| [llvm-cxxmap](https://llvm.liuxfe.com/docs/man/llvm-cxxmap.html) | Mangled name重映射工具            |
| [llvm-diff](https://llvm.liuxfe.com/docs/man/llvm-diff.html) | LLVM 结构”diff”                   |
| [llvm-cov](https://llvm.liuxfe.com/docs/man/llvm-cov.html)   | 发出覆盖信息                      |
| [llvm-profdata](https://llvm.liuxfe.com/docs/man/llvm-profdata.html) | 配置数据工具                      |
| [llvm-stress](https://llvm.liuxfe.com/docs/man/llvm-stress.html) | 生成随机的.ll文件                 |
| [llvm-symbolizer](https://llvm.liuxfe.com/docs/man/llvm-symbolizer.html) | 将地址转换为源代码中的位置        |
| [llvm-dwarfdump](https://llvm.liuxfe.com/docs/man/llvm-dwarfdump.html) | 转储并检验DWARF调试信息           |
| [dsymutil](https://llvm.liuxfe.com/docs/man/dsymutil.html)   | 操作存档文件中的DWARF调试符号文件 |
| [llvm-mca](https://llvm.liuxfe.com/docs/man/llvm-mca.html)   | LLVM机器码分析器                  |
| [llvm-readobj](https://llvm.liuxfe.com/docs/man/llvm-readobj.html) | LLVM目标文件分析器                |

#### GNU bintils替代命令

| 命令                                                         | 说明                               |
| :----------------------------------------------------------- | :--------------------------------- |
| [llvm-addr2line](https://llvm.liuxfe.com/docs/man/llvm-addr2line.html) | addr2line的替代品                  |
| [llvm-ar](https://llvm.liuxfe.com/docs/man/llvm-ar.html)     | LLVM归档器                         |
| [llvm-cxxfilt](https://llvm.liuxfe.com/docs/man/llvm-cxxfilt.html) | LLVM符合名称分析器                 |
| [llvm-nm](https://llvm.liuxfe.com/docs/man/llvm-nm.html)     | 列出LLVM字节码和目标文件中的符号表 |
| [llvm-objcopy](https://llvm.liuxfe.com/docs/man/llvm-objcopy.html) | 目标文件复制和编辑工具             |
| [llvm-objdump](https://llvm.liuxfe.com/docs/man/llvm-objdump.html) | LLVM目标文件转储器                 |
| [llvm-ranlib](https://llvm.liuxfe.com/docs/man/llvm-ranlib.html) | 库存档索引生成工具                 |
| [llvm-readelf](https://llvm.liuxfe.com/docs/man/llvm-readelf.html) | GNU风格的LLVM对象读取器            |
| [llvm-size](https://llvm.liuxfe.com/docs/man/llvm-size.html) | 打印目标文件尺寸信息               |
| [llvm-strings](https://llvm.liuxfe.com/docs/man/llvm-strings.html) | 打印目标文件中的字符串             |
| [llvm-strip](https://llvm.liuxfe.com/docs/man/llvm-strip.html) | 目标文件去除调试信息工具           |

#### 调试工具

| 命令                                                         | 说明                 |
| :----------------------------------------------------------- | :------------------- |
| [bugpoint](https://llvm.liuxfe.com/docs/man/bugpoint.html)   | 自动测试用例缩减工具 |
| [llvm-extract](https://llvm.liuxfe.com/docs/man/llvm-extract.html) | 从LLVM模块中提取函数 |
| [llvm-bcanalyzer](https://llvm.liuxfe.com/docs/man/llvm-bcanalyzer.html) | LLVM字节码分析器     |

#### 开发工具

| 命令                                                         | 说明                        |
| :----------------------------------------------------------- | :-------------------------- |
| [FileCheck](https://llvm.liuxfe.com/docs/man/filechcke.html) | 灵活的模式匹配文件验证程序  |
| [tblgen](https://llvm.liuxfe.com/docs/man/tblgen.html)       | 目标描述到C++代码生成器     |
| [lit](https://llvm.liuxfe.com/docs/man/lit.html)             | LLVM集成测试仪              |
| [llvm-build](https://llvm.liuxfe.com/docs/man/llvm-build.html) | LLVM项目构建实用程序        |
| [llvm-exegesis](https://llvm.liuxfe.com/docs/man/llvm-exegesis.html) | LLVM机器指令基准            |
| [llvm-pdbutil](https://llvm.liuxfe.com/docs/man/llvm-pdbutil.html) | PDB文件取证和诊断           |
| [llvm-locstats](https://llvm.liuxfe.com/docs/man/llvm-locstats.html) | 计算DWARF调试位置的统计信息 |



### 开源项目 

- [emscripten-core/emscripten](https://github.com/emscripten-core/emscripten): Emscripten:一个llvm到webassembly的编译器
- [tinygo-org/tinygo](https://github.com/tinygo-org/tinygo): 微控制器、WebAssembly (WASM/WASI)和命令行工具。基于LLVM。
- [numba/numba](https://github.com/numba/numba): 使用LLVM支持NumPy动态Python编译器
- [avast/retdec](https://github.com/avast/retdec): RetDec是一个基于LLVM的可重定向机器码反编译器。
- [lifting-bits/mcsema](https://github.com/lifting-bits/mcsema): 将x86、amd64、aarch64、sparc32和sparc64程序二进制代码提升到LLVM位码的框架
- [microsoft/DirectXShaderCompiler](https://github.com/microsoft/DirectXShaderCompiler): 这个repo托管了DirectX Shader编译器的源代码，它是基于LLVM/Clang的。
- [andreasfertig/cppinsights](https://github.com/andreasfertig/cppinsights): c++洞察力——用编译器的眼光看你的源代码
- [google/souper](https://github.com/google/souper): LLVM IR的超优化器
- [HikariObfuscator/Hikari](https://github.com/HikariObfuscator/Hikari): LLVM模糊处理
- [dotnet/llilc](https://github.com/dotnet/llilc):这个repo包含LLILC，一个基于LLVM的。net Core编译器。它包括一组跨平台的。net代码生成工具，可以将MSIL字节码编译成LLVM支持的平台。
- [banach-space/llvm-tutor](https://github.com/banach-space/llvm-tutor): 用于教学和学习的树外LLVM通行证的集合
- [numba/llvmlite](https://github.com/numba/llvmlite): 用于编写JIT编译器的轻量级LLVM python绑定
- [yrnkrn/zapcc](https://github.com/yrnkrn/zapcc): zapcc是一个基于clang的缓存c++编译器，旨在执行更快的编译
- [go-llvm/llgo](https://github.com/go-llvm/llgo): 基于llvm的编译器
- [eliben/llvm-clang-samples](https://github.com/eliben/llvm-clang-samples): unmaintenance:使用LLVM和Clang编译库和工具的例子

### 文章

- [LLVM 入门篇](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E7%AB%A0/LLVM%20%E5%85%A5%E9%97%A8%E7%AF%87.md)
- [LLVM-Clang入门](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E7%AB%A0/LLVM-Clang%E5%85%A5%E9%97%A8.md)
- [LLVM编译器框架介绍](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E7%AB%A0/LLVM%E7%BC%96%E8%AF%91%E5%99%A8%E6%A1%86%E6%9E%B6%E4%BB%8B%E7%BB%8D.md)
- [Llvm编译的基本概念和流程](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E7%AB%A0/llvm%E7%BC%96%E8%AF%91%E7%9A%84%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5%E5%92%8C%E6%B5%81%E7%A8%8B.md)
- [后端技术的重用：LLVM不仅仅让你高效](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E7%AB%A0/%E5%90%8E%E7%AB%AF%E6%8A%80%E6%9C%AF%E7%9A%84%E9%87%8D%E7%94%A8%EF%BC%9ALLVM%E4%B8%8D%E4%BB%85%E4%BB%85%E8%AE%A9%E4%BD%A0%E9%AB%98%E6%95%88.md)
- [编译优化｜LLVM代码生成技术详解及在数据库中的应用](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E7%AB%A0/%E7%BC%96%E8%AF%91%E4%BC%98%E5%8C%96%EF%BD%9CLLVM%E4%BB%A3%E7%A0%81%E7%94%9F%E6%88%90%E6%8A%80%E6%9C%AF%E8%AF%A6%E8%A7%A3%E5%8F%8A%E5%9C%A8%E6%95%B0%E6%8D%AE%E5%BA%93%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8.md)
- [编译器及底层名词解释](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E7%AB%A0/%E7%BC%96%E8%AF%91%E5%99%A8%E5%8F%8A%E5%BA%95%E5%B1%82%E5%90%8D%E8%AF%8D%E8%A7%A3%E9%87%8A.md)

### 视频
- [How LLVM & Clang work](https://pan.baidu.com/s/1yTDS9Bn5CiFGhotKjNfcIw)---提取码: 225f
- [2021 LLVM Dev Mtg “Otter- Tracing & Visualizing OpenMP Programs as DAGs Through LLVM's OMPT...”](https://pan.baidu.com/s/1lbO_764_sXgMf5mgwajdcw)---提取码: d2k2
- [2021 LLVM Dev Mtg “Navigating Exotic SIMD Lands with an LLVM Guide”](https://pan.baidu.com/s/1SAbepiyv0X6W7Qxf6qVxuA)---提取码: 5v6s
- [2019 LLVM Developers’ Meeting- E. Christopher & J. Doerfert “Introduction to  LLVM”](https://pan.baidu.com/s/1_g5P0r0Cku30w3m2LiJgWw)---提取码: 8u6e
- [2019 LLVM Developers’ Meeting- S. Haastregt & A. Stulova “An overview of Clang ”](https://pan.baidu.com/s/1VOhM2SOeWnRoy24jaWw7dQ)---提取码: r6ct
- [P. Goldsborough “clang-useful- Building useful tools with LLVM and clang for fun and profit](https://pan.baidu.com/s/1DkpEdZXeBISJMuExVtpZKg)---提取码: xemt
<!--

## 🔥 kvm

<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150633416-9961e8b7-ff81-488b-8cfe-b69ba739c1ff.png"/>
  
#### —— Linux内核中的虚拟化基础设施
</div>


### 文档
- 官方文档:
- 其他文档：

### 开源项目 

### 文章

### 视频(提取码：1024)

## 🔥 ceph

<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150633285-0e03e44f-8755-4b12-9b62-8f8030d44c94.png"/>
  
#### —— 存储的未来
</div>

### 文档
- 官方文档:
- 其他文档：

### 开源项目 

### 文章

### 视频(提取码：1024)

## 🔥 fuse

<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150633338-fde17a17-9cfb-4a32-bc51-d76e02b6904e.png"/>
  
#### —— 用户态文件系统
</div>

### 文档
- 官方文档:
- 其他文档：

### 开源项目 

### 文章

### 视频(提取码：1024)

<br/>
<br/>
<h3 >零领工作</h3> 

---

##### 实时提供，每周发布北京，上海，广州，深圳，杭州，南京，合肥，武汉，长沙，重庆，成都，西安，厦门的c/c++，golang方向的招聘岗位信息。 包含校招，社招，实习岗位， 面经，八股，简历

<img src="https://img.0voice.com/public/0e59910091576beaebe20f303357edf7.jpg" alt="零领工作" style="width:300px;height:300px;">

<br/>
<br/>
