# Cgroup - Linux的IO资源隔离

## Linux的IO隔离

跟内存管理那部分复杂度类似，IO的资源隔离要讲清楚也是比较麻烦的。这部分内容都是这样，配置起来简单，但是要理解清楚确没那么简单。这次是跟Linux内核的IO实现有关系。对于IO的速度限制，实现思路跟CPU和内存都不一样。CPU是针对进程占用时间的比例限制，内存是空间限制，而当我们讨论IO资源隔离的时候，实际上有两个资源需要考虑，一个是空间，另一个是速度。对于空间来说，这个很简单，大不了分区就是了。现实手段中，分区、LVM、磁盘配额、目录配额等等，不同的分区管理方式，不同的文件系统都给出了很多不同的解决方案。所以，空间的限制实际上不是cgroup要解决的问题，那就是说，我们在这里要解决的问题是：如何进行IO数据传输的速度限制。

限速这件事情，现实中有很多模型、算法去解决这个问题。比如，如果我们想控制高速公路上的汽车单位时间通过率，就让收费站每隔固定时间周期只允许通过固定个数的车就好了。这是一种非常有效的控制手段－－漏斗算法。现实中这种算法可能在特定情况下会造成资源浪费以及用户的体验不好，于是又演化出令牌桶算法。这里我们不去详细分析这些算法，但是我们要知道，对io的限速基本是一个漏斗算法的限速效果。无论如何，这种限速都要有个“收费站”这样的设施来执行限速，那么对于Linux的IO体系来说，这个”收费站”建在哪里呢？于是我们就必须先来了解一下：

### Linux的IO体系

Linux的IO体系是个层级还算清晰的结构，它基本上分成了如图示这样几层：

**Linux的IO体系层次结构**

我们可以通过追踪一个read()系统调用来一窥这些层次的结构，当read()系统调用发生，内核首先会通过汇编指令引发一个软中断，然后根据中断传入的参数查询系统调用影射表，找到read()对应的内核调用方法名，并去执行相关调用，这个系统调用名一般情况下就是sys_read()。从此，便开始了调用在内核中处理的过程的第一步：

1. VFS层：虚拟文件系统层。由于内核要跟多种文件系统打交道，而每一种文件系统所实现的数据结构和相关方法都可能不尽相同，所以，内核抽象了这一层，专门用来适配各种文件系统，并对外提供统一操作接口。
2. 文件系统层：不同的文件系统实现自己的操作过程，提供自己特有的特征，具体不多说了，大家愿意的话自己去看代码即可。
3. 页缓存层：我们的老朋友了，如果不了解缓存是什么的，可以先来看看Linux内存资源管理部分。
4. 通用块层：由于绝大多数情况的io操作是跟块设备打交道，所以Linux在此提供了一个类似vfs层的块设备操作抽象层。下层对接各种不同属性的块设备，对上提供统一的Block IO请求标准。
5. IO调度层：因为绝大多数的块设备都是类似磁盘这样的设备，所以有必要根据这类设备的特点以及应用的不同特点来设置一些不同的调度算法和队列。以便在不同的应用环境下有针对性的提高磁盘的读写效率，这里就是大名鼎鼎的Linux电梯所起作用的地方。针对机械硬盘的各种调度方法就是在这实现的。
6. 块设备驱动层：驱动层对外提供相对比较高级的设备操作接口，往往是C语言的，而下层对接设备本身的操作方法和规范。
7. 块设备层：这层就是具体的物理设备了，定义了各种真对设备操作方法和规范。

根据这几层的特点，如果你是设计者，你会在哪里实现真对块设备的限速策略呢？6、7都是相关具体设备的，如果在这个层次提供，那就不是内核全局的功能，而是某些设备自己的feture。文件系统层也可以实现，但是如果要全局实现也是不可能的，需要每种文件系统中都实现一遍，成本太高。所以，可以实现限速的地方比较合适的是VFS、缓存层、通用块层和IO调度层。而VFS和page cache这样的机制并不是面向块设备设计的，都是做其他事情用的，虽然也在io体系中，但是并不适合用来做block io的限速。所以这几层中，最适合并且成本最低就可以实现的地方就是IO调度层和通用块层。IO调度层本身已经有队列了，我们只要在队列里面实现一个限速机制即可，但是在IO调度层实现的限速会因为不同调度算法的侧重点不一样而有很多局限性，从通用块层实现的限速，原则上就可以对几乎所有的块设备进行带宽和iops的限制。截止目前（4.3.3内核），IO限速主要实现在这两层中。

根据IO调度层和通用块层的特点，这两层分别实现了两种不同策略的IO控制策略，也是目前blkio子系统提供的两种控制策略，一个是权重比例方式的控制，另一个是针对IO带宽和IOPS的控制。

### IO调度层

我们需要先来认识一下IO调度层。这一层要解决的核心问题是，如何提高块设备IO的整体性能？这一层也主要是针对用途最广泛的机械硬盘结构而设计的。众所周知，机械硬盘的存储介质是磁介质，并且是盘状，用磁头在盘片上移动进行数据的寻址，这类似播放一张唱片。这种结构的特点是，顺序的数据读写效率比较理想，但是如果一旦对盘片有随机读写，那么大量的时间都会浪费在磁头的移动上，这时候就会导致每次IO的响应时间很长，极大的降低IO的响应速度。磁头在盘片上寻道的操作，类似电梯调度，如果在寻道的过程中，能把路过的相关磁道的数据请求都“顺便”处理掉，那么就可以在比较小影响响应速度的前提下，提高整体IO的吞吐量。所以，一个好的IO调度算法的需求就此产生。在最开始的阶段，Linux就把这个算法命名为Linux电梯算法。目前在内核中默认开启了三种算法，其实严格算应该是两种，因为第一种叫做noop，就是空操作调度算法，也就是没有任何调度操作，并不对io请求进行排序，仅仅做适当的io合并的一个fifo队列。

目前内核中默认的调度算法应该是cfq，叫做完全公平队列调度。这个调度算法人如其名，它试图给所有进程提供一个完全公平的IO操作环境。它为每个进程创建一个同步IO调度队列，并默认以时间片和请求数限定的方式分配IO资源，以此保证每个进程的IO资源占用是公平的，cfq还实现了针对进程级别的优先级调度，这里我们不去细节解释。我们在此只需要知道，既然时间片分好了，优先级实现了，那么cfq肯定是实现进程级别的权重比例分配的最好方案。内核就是这么做的，cgroup blkio的权重比例限制就是基于cfq调度器实现的。如果你要使用权重比例分配，请先确定对应的块设备的IO调度算法是cfq。

查看和修改的方法是：

```
[zorro@zorrozou-pc0 ~]$ cat /sys/block/sda/queue/scheduler 
noop deadline [cfq] 
[zorro@zorrozou-pc0 ~]$ echo cfq > /sys/block/sda/queue/scheduler
```

cfq是通用服务器比较好的IO调度算法选择，对桌面用户也是比较好的选择。但是对于很多IO压力较大的场景就并不是很适应，尤其是IO压力集中在某些进程上的场景。因为这种场景我们需要更多的满足某个或者某几个进程的IO响应速度，而不是让所有的进程公平的使用IO，比如数据库应用。

deadline调度（最终期限调度）就是更适应这样的场景的解决方案。deadline实现了四个队列，其中两个分别处理正常read和write，按扇区号排序，进行正常io的合并处理以提高吞吐量.因为IO请求可能会集中在某些磁盘位置，这样会导致新来的请求一直被合并，于是可能会有其他磁盘位置的io请求被饿死。于是实现了另外两个处理超时read和write的队列，按请求创建时间排序，如果有超时的请求出现，就放进这两个队列，调度算法保证超时（达到最终期限时间）的队列中的请求会优先被处理，防止请求被饿死。由于deadline的特点，无疑在这里无法区分进程，也就不能实现针对进程的io资源控制。

其实不久前，内核还是默认标配四种算法，还有一种叫做as的算法（Anticipatory scheduler），预测调度算法。一个高大上的名字，搞得我一度认为Linux内核都会算命了。结果发现，无非是在基于deadline算法做io调度的之前等一小会时间，如果这段时间内有可以合并的io请求到来，就可以合并处理，提高deadline调度的在顺序读写情况下的数据吞吐量。其实这根本不是啥预测，我觉得不如叫撞大运调度算法。估计结果是不仅没有提高吞吐量，还降低了响应速度，所以内核干脆把它从默认配置里删除了。毕竟Linux的宗旨是实用。

根据以上几种io调度算法的简单分析，我们也能对各种调度算法的使用场景有一些大致的思路了。从原理上看，cfq是一种比较通用的调度算法，是一种以进程为出发点考虑的调度算法，保证大家尽量公平。deadline是一种以提高机械硬盘吞吐量为思考出发点的调度算法，只有当有io请求达到最终期限的时候才进行调度，非常适合业务比较单一并且IO压力比较重的业务，比如数据库。而noop呢？其实如果我们把我们的思考对象拓展到固态硬盘，那么你就会发现，无论cfq还是deadline，都是针对机械硬盘的结构进行的队列算法调整，而这种调整对于固态硬盘来说，完全没有意义。对于固态硬盘来说，IO调度算法越复杂，效率就越低，因为额外要处理的逻辑越多。所以，固态硬盘这种场景下，使用noop是最好的，deadline次之，而cfq由于复杂度的原因，无疑效率最低。但是，如果你想对你的固态硬盘做基于权重比例的IO限速的话，那就没啥办法了，毕竟这时候，效率并不是你的需求，要不你限速干嘛？

### 通用块设备层

这层的作用我这里就不再复述了，本节其实主要想解释的是，既然这里实现了对blkio的带宽和iops的速度限制，那么有没有什么需要注意的地方？这自然是有的。首先我们还是要先来搞清楚IO中的几个概念。

**一般IO**：

一个正常的文件io，需要经过vfs -> buffer\page cache -> 文件系统 -> 通用块设备层 -> IO调度层 -> 块设备驱动 -> 硬件设备这所有几个层次。其实这就是一般IO。当然，不同的状态可能会有变化，比如一个进程正好open并read一个已经存在于page cache中的数据。这样的事情我们排出在外不分析。那么什么是比较特别的io呢？

**Direct IO**：

中文也可以叫直接IO操作，其特点是，VFS之后跳过buffer\page cache层，直接从文件系统层进行操作。那么就意味着，无论读还是写，都不会进行cache。我们基本上可以理解这样的io看起来效率要低很多，直接看到的速度就是设备的速度，并且缺少了cache层对数据的缓存之后，文件系统和数据块的操作效率直接暴露给了应用程序，块的大小会直接影响io速度。

**Sync IO** & **write-through**:

中文叫做同步IO操作，如果是写操作的话也叫write-through，这个操作往往容易跟上面的DIO搞混，因为看起来他们速度上差不多，但是是有本质区别的。这种方式写的数据要等待存储写入返回才能成功返回，所以跟DIO效率差不多，但是，写的数据仍然是要在cache中写入的，这样其他一般IO的程度仍然可以使用cache机制加速IO操作。所以，这里的sync的意思就是，在执行write操作的时候，让cache和存储上的数据一致。那么他跟一般IO其实一样，数据是要经过cache层的。

**write-back**:

既然明白了write-thuough，那么write-back就好理解了，无非就是将目前在cache中还没写回存储的脏数据写回到存储。这个名词一般指的是一个独立的过程，这个过程不是随着应用的写而发生，这往往是内核自己找个时间来单独操作的。说白了就是，应用写文件，感觉自己很快写完了，其实内核都把数据放倒cache里了，然后内核自己找时间再去写回到存储上。实际上write-back只是在一般IO的情况下，保证数据一致性的一种机制而已。

有人将IO过程中，以是否使用缓冲（缓存）的区别，将IO分成了缓存IO（Buffered IO）和直接IO（Direct io）。其实就是名词上的不同而已。这里面的buffer的含义跟内存中buffer cache有概念上的不同。实际上这里Buffered IO的含义，相当于内存中的buffer cache+page cache，就是IO经过缓存的意思。到这我们思考一个问题，如果cgroup针对IO的资源限制实现在了通用块设备层，那么将会对哪些IO操作有影响呢？其实原则上说都有影响，因为绝大多数数据都是要经过通用块设备层写入存储的，但是对于应用程序来说感受可能不一样。在一般IO的情况下，应用程序很可能很快的就写完了数据（在数据量小于缓存空间的情况下），然后去做其他事情了。这时应用程序感受不到自己被限速了，而内核在处理write-back的阶段，由于没有相关page cache中的inode是属于那个cgroup的信息记录，所以所有的page cache的回写只能放到cgroup的root组中进行限制，而不能在其他cgroup中进行限制，因为root组的cgroup一般是不做限制的，所以就相当于目前的cgroup的blkio对buffered IO是没有限速支持的。这个功能将在使用了unified-hierarchy体系的cgroup v2中的部分文件系统（ext系列）已经得到得到支持，目前这个功能还在开发中，据说将在4.5版本的内核中正式发布。

而在Sync IO和Direct IO的情况下，由于应用程序写的数据是不经过缓存层的，所以能直接感受到速度被限制，一定要等到整个数据按限制好的速度写完或者读完，才能返回。这就是当前cgroup的blkio限制所能起作用的环境限制。了解了这个之后，我们就可以来看：

## blkio配置方法

### 权重比例分配

我们这次直接使用命令行的方式对cgroup进行操作。在我们的系统上，我们现在想创建两个cgroup组，一个叫test1，一个叫test2。我们想让这两个组的进程在对/dev/sdb，设备号为8:16的这个磁盘进行读写的时候按权重比例进行io资源的分配。具体配置方法如下：

首先确认系统上已经mount了相关的cgroup目录：

```
[root@zorrozou-pc0 ~]# ls /sys/fs/cgroup/blkio/
blkio.io_merged           blkio.io_service_bytes_recursive  blkio.io_wait_time           blkio.sectors            blkio.throttle.read_iops_device   blkio.weight         tasks
blkio.io_merged_recursive  blkio.io_serviced             blkio.io_wait_time_recursive  blkio.sectors_recursive        blkio.throttle.write_bps_device   blkio.weight_device
blkio.io_queued           blkio.io_serviced_recursive         blkio.leaf_weight           blkio.throttle.io_service_bytes  blkio.throttle.write_iops_device  cgroup.clone_children
blkio.io_queued_recursive  blkio.io_service_time         blkio.leaf_weight_device       blkio.throttle.io_serviced        blkio.time                  cgroup.procs
blkio.io_service_bytes       blkio.io_service_time_recursive   blkio.reset_stats           blkio.throttle.read_bps_device   blkio.time_recursive          notify_on_release
```

然后创建两个针对blkio的cgroup

```
[root@zorrozou-pc0 ~]# mkdir /sys/fs/cgroup/blkio/test1
[root@zorrozou-pc0 ~]# mkdir /sys/fs/cgroup/blkio/test2
```

相关目录下会自动产生相关配置项：

```
[root@zorrozou-pc0 ~]# ls /sys/fs/cgroup/blkio/test{1,2}
/sys/fs/cgroup/blkio/test1:
blkio.io_merged           blkio.io_service_bytes_recursive  blkio.io_wait_time           blkio.sectors            blkio.throttle.read_iops_device   blkio.weight         tasks
blkio.io_merged_recursive  blkio.io_serviced             blkio.io_wait_time_recursive  blkio.sectors_recursive        blkio.throttle.write_bps_device   blkio.weight_device
blkio.io_queued           blkio.io_serviced_recursive         blkio.leaf_weight           blkio.throttle.io_service_bytes  blkio.throttle.write_iops_device  cgroup.clone_children
blkio.io_queued_recursive  blkio.io_service_time         blkio.leaf_weight_device       blkio.throttle.io_serviced        blkio.time                  cgroup.procs
blkio.io_service_bytes       blkio.io_service_time_recursive   blkio.reset_stats           blkio.throttle.read_bps_device   blkio.time_recursive          notify_on_release

/sys/fs/cgroup/blkio/test2:
blkio.io_merged           blkio.io_service_bytes_recursive  blkio.io_wait_time           blkio.sectors            blkio.throttle.read_iops_device   blkio.weight         tasks
blkio.io_merged_recursive  blkio.io_serviced             blkio.io_wait_time_recursive  blkio.sectors_recursive        blkio.throttle.write_bps_device   blkio.weight_device
blkio.io_queued           blkio.io_serviced_recursive         blkio.leaf_weight           blkio.throttle.io_service_bytes  blkio.throttle.write_iops_device  cgroup.clone_children
blkio.io_queued_recursive  blkio.io_service_time         blkio.leaf_weight_device       blkio.throttle.io_serviced        blkio.time                  cgroup.procs
blkio.io_service_bytes       blkio.io_service_time_recursive   blkio.reset_stats           blkio.throttle.read_bps_device   blkio.time_recursive          notify_on_release
```

之后我们就可以进行限制了。针对cgroup进行权重限制的配置有**blkio.weight**，是单纯针对cgroup进行权重配置的，还有**blkio.weight_device**可以针对设备单独进行限制，我们都来试试。首先我们想设置test1和test2使用任何设备的io权重比例都是1:2：

```
[root@zorrozou-pc0 zorro]# echo 100 > /sys/fs/cgroup/blkio/test1/blkio.weight
[root@zorrozou-pc0 zorro]# echo 200 > /sys/fs/cgroup/blkio/test2/blkio.weight
```

注意权重设置的取值范围为：10-1000。然后我们来写一个测试脚本：

```
#!/bin/bash

testfile1=/home/test1
testfile2=/home/test2

if [ -e $testfile1 ]
then
    rm -rf $testfile1
fi

if [ -e $testfile2 ]
then
    rm -rf $testfile2
fi

sync
echo 3 > /proc/sys/vm/drop_caches

cgexec -g blkio:test1 dd if=/dev/zero of=$testfile1 oflag=direct bs=1M count=1023 &

cgexec -g blkio:test2 dd if=/dev/zero of=$testfile2 oflag=direct bs=1M count=1023 &
```

我们dd的时候使用的是direct标记，在这使用sync和不加任何标记的话都达不到效果。因为权重限制是基于cfq实现，cfq要标记进程，而buffered IO都是内核同步，无法标记进程。使用iotop查看限制效果：

```
[root@zorrozou-pc0 zorro]# iotop -b -n1|grep direct
 1519 be/4 root        0.00 B/s  110.00 M/s  0.00 % 99.99 % dd if=/dev/zero of=/home/test2 oflag=direct bs=1M count=1023
 1518 be/4 root        0.00 B/s   55.00 M/s  0.00 % 99.99 % dd if=/dev/zero of=/home/test1 oflag=direct bs=1M count=1023
```

却是达到了1:2比例限速的效果。此时对于磁盘读取的限制效果也一样，具体测试用例大家可以自己编写。读取的时候要注意，仍然要保证读取的文件不在page cache中，方法就是：echo 3 > /proc/sys/vm/drop_caches。因为在page cache中的数据已经在内存里了，直接修改是直接改内存中的内容，只有write-back的时候才会经过cfq。

我们再来试一下针对设备的权重分配，请注意设备号的填写格式：

```
[root@zorrozou-pc0 zorro]# echo "8:16 400" > /sys/fs/cgroup/blkio/test1/blkio.weight_device 
[root@zorrozou-pc0 zorro]# echo "8:16 200" > /sys/fs/cgroup/blkio/test2/blkio.weight_device 

[root@zorrozou-pc0 zorro]# iotop -b -n1|grep direct
 1800 be/4 root        0.00 B/s  102.24 M/s  0.00 % 99.99 % dd if=/dev/zero of=/home/test1 oflag=direct bs=1M count=1023
 1801 be/4 root        0.00 B/s   51.12 M/s  0.00 % 99.99 % dd if=/dev/zero of=/home/test2 oflag=direct bs=1M count=1023
```

我们会发现，这时权重确实是按照最后一次的设置，test1和test2变成了2:1的比例，而不是1:2了。这里要说明的就是，注意blkio.weight_device的设置会覆盖blkio.weight的设置，因为前者的设置精确到了设备，Linux在这里的策略是，越精确越优先。

### 读写带宽和iops限制

针对读写带宽和iops的限制都是绝对值限制，所以我们不用两个cgroup做对比了。我们就设置test1的写带宽速度为1M/s:

```
[root@zorrozou-pc0 zorro]# echo "8:16  1048576" > /sys/fs/cgroup/blkio/test1/blkio.throttle.write_bps_device

[root@zorrozou-pc0 zorro]# sync
[root@zorrozou-pc0 zorro]# echo 3 > /proc/sys/vm/drop_caches

[root@zorrozou-pc0 zorro]# cgexec -g blkio:test1 dd if=/dev/zero of=/home/test oflag=direct count=1024 bs=1M
^C21+0 records in
21+0 records out
22020096 bytes (22 MB) copied, 21.012 s, 1.0 MB/s
```

此时不用dd命令执行完，稍等一下中断执行就能看到速度确实限制在了1M／s。写的同时，iostat显示为：

```
[zorro@zorrozou-pc0 ~]$ iostat -x 1
Linux 4.3.3-2-ARCH (zorrozou-pc0.tencent.com)     2016年01月15日     _x86_64_    (4 CPU)


avg-cpu:  %user   %nice %system %iowait  %steal   %idle
       0.50    0.00    0.50   25.13    0.00   73.87

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
sdb               0.00     0.00    0.00    1.00     0.00  1024.00  2048.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-0              0.00     0.00    0.00    1.00     0.00  1024.00  2048.00     1.00 1000.00    0.00 1000.00 1000.00 100.00
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-2              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-3              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
        1.25    0.00    0.50   24.81    0.00   73.43

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
sdb               0.00     5.00    0.00    6.00     0.00  1060.00   353.33     0.06    9.33    0.00    9.33   9.50   5.70
dm-0              0.00     0.00    0.00   10.00     0.00  1060.00   212.00     1.08  109.00    0.00  109.00 100.00 100.00
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-2              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-3              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
       1.25    0.00    1.00   24.44    0.00   73.32

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
sdb               0.00     0.00    0.00    1.00     0.00  1024.00  2048.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-0              0.00     0.00    0.00    1.00     0.00  1024.00  2048.00     1.00  993.00    0.00  993.00 1000.00 100.00
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-2              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-3              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
       1.50    0.25    0.75   24.50    0.00   73.00

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
sdb               0.00     0.00    0.00    1.00     0.00  1024.00  2048.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-0              0.00     0.00    0.00    1.00     0.00  1024.00  2048.00     1.00 1000.00    0.00 1000.00 1000.00 100.00
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-2              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-3              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
```

可以看到写的速度确实为1024wkB/s左右。我们再来试试读，先创建一个大文件，此处没有限速：

```
[root@zorrozou-pc0 zorro]# dd if=/dev/zero of=/home/test oflag=direct count=1024 bs=1M
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB) copied, 10.213 s, 105 MB/s
```

然后进行限速设置并确认：

```
[root@zorrozou-pc0 zorro]# sync
[root@zorrozou-pc0 zorro]# echo 3 > /proc/sys/vm/drop_caches 
[root@zorrozou-pc0 zorro]# echo "8:16  1048576" > /sys/fs/cgroup/blkio/test1/blkio.throttle.read_bps_device
[root@zorrozou-pc0 zorro]# cgexec -g blkio:test1 dd if=/home/test of=/dev/null iflag=direct count=1024 bs=1M
^C15+0 records in
14+0 records out
14680064 bytes (15 MB) copied, 15.0032 s, 978 kB/s
```

iostat结果：

```
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
       0.75    0.00    0.75   24.63    0.00   73.88

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
sdb               0.00     0.00    2.00    0.00  1024.00     0.00  1024.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-0              0.00     0.00    2.00    0.00  1024.00     0.00  1024.00     1.65  825.00  825.00    0.00 500.00 100.00
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-2              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-3              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
       0.75    0.00    0.50   24.87    0.00   73.87

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     2.00    0.00    2.00     0.00    16.00    16.00     0.02   10.00    0.00   10.00  10.00   2.00
sdb               0.00     0.00    2.00    0.00  1024.00     0.00  1024.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-0              0.00     0.00    2.00    0.00  1024.00     0.00  1024.00     1.65  825.00  825.00    0.00 500.00 100.00
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-2              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-3              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
```

最后是iops的限制，我就不废话了，直接上命令执行结果：

```
[root@zorrozou-pc0 zorro]# echo "8:16  20" > /sys/fs/cgroup/blkio/test1/blkio.throttle.write_iops_device 
[root@zorrozou-pc0 zorro]# rm /home/test 
[root@zorrozou-pc0 zorro]# sync
[root@zorrozou-pc0 zorro]# echo 3 > /proc/sys/vm/drop_caches
[root@zorrozou-pc0 zorro]# cgexec -g blkio:test1 dd if=/dev/zero of=/home/test oflag=direct count=1024 bs=1M
^C121+0 records in
121+0 records out
126877696 bytes (127 MB) copied, 12.0576 s, 10.5 MB/s

[zorro@zorrozou-pc0 ~]$ iostat -x 1
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
       0.50    0.00    0.25   24.81    0.00   74.44

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
sdb               0.00     0.00    0.00   20.00     0.00 10240.00  1024.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-0              0.00     0.00    0.00   20.00     0.00 10240.00  1024.00     2.00  100.00    0.00  100.00  50.00 100.00
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-2              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-3              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
       0.75    0.00    0.25   24.31    0.00   74.69

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
sdb               0.00     0.00    0.00   20.00     0.00 10240.00  1024.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-0              0.00     0.00    0.00   20.00     0.00 10240.00  1024.00     2.00  100.00    0.00  100.00  50.00 100.00
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-2              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-3              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
```

iops的读限制我就不在废话了，大家可以自己做实验测试一下。

## 其他相关文件

### 针对权重比例限制的相关文件

**blkio.leaf_weight[_device]**

其意义等同于blkio.weight[_device]，主要表示当本cgroup中有子cgroup的时候，本cgroup的进程和子cgroup中的进程所分配的资源比例是怎么样的。举个例子说吧，假设有一组cgroups的关系是这样的：

```
          root
       /    |   \
      A     B    leaf
     400   200   200
```

leaf就表示root组下的进程所占io资源的比例。 此时A组中的进程可以占用的比例为：400/（400+200+200） ＊ 100% ＝ 50% B为：200/（400+200+200） ＊ 100% ＝ 25% 而root下的进程为：200/（400+200+200） ＊ 100% ＝ 25%

**blkio.time**

统计相关设备的分配给本组的io处理时间，单位为ms。权重就是依据此时间比例进行分配的。

**blkio.sectors**

统计本cgroup对设备的读写扇区个数。

**blkio.io_service_bytes**

统计本cgroup对设备的读写字节个数。

**blkio.io_serviced**

统计本cgroup对设备的读写操作个数。

**blkio.io_service_time**

统计本cgroup对设备的各种操作时间。时间单位是ns。

**blkio.io_wait_time**

统计本cgroup对设备的各种操作的等待时间。时间单位是ns。

**blkio.io_merged**

统计本cgroup对设备的各种操作的合并处理次数。

**blkio.io_queued**

统计本cgroup对设备的各种操作的当前正在排队的请求个数。

**blkio.\*_recursive**

这一堆文件是相对应的不带_recursive的文件的递归显示版本，所谓递归的意思就是，它会显示出包括本cgroup在内的衍生cgroup的所有信息的总和。

### 针对带宽和iops限制的相关文件

**blkio.throttle.io_serviced**

统计本cgroup对设备的读写操作个数。

**blkio.throttle.io_service_bytes**

统计本cgroup对设备的读写字节个数。

**blkio.reset_stats**

对本文件写入一个int可以对以上所有文件的值置零，重新开始累计。

## 最后

其实一直纠结要不要写这部分IO隔离的文档，因为看上去意义不大。一则因为目前IO隔离似乎工作场景里用的不多，二则因为目前内核中这部分代码还在进行较大变化的调整，还要继续加入其它功能。从内核Linux 3.16版本之后，cgroup调整方向，开始了基于unified hierarchy架构的cgroup v2。IO部分在write-back部分进行了较大调整，加入了对buffered IO的资源限制。我们这次系统环境为ArchLinux，内核版本为Linux 4.3.3，虽然环境中的unified hierarchy的开发版本功能已经部分支持了，但是思考再三还是暂时不加入到此文档中。新架构的cgoup v2预计会跟随Linux 4.5一起推出，到时候我们再做详细分析好了。

> 原文链接：https://zorro.gitbooks.io/poor-zorro-s-linux-book/content/cgroup-linuxde-io-zi-yuan-ge-li.html

