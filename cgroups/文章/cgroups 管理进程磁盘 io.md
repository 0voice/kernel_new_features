# 用 cgroups 管理进程磁盘 io

linux 的 cgroups 还可以限制和监控进程的磁盘 io。这个功能通过 blkio 子系统实现。

blkio 子系统里东西很多。不过大部分都是只读的状态报告，可写的参数就只有下面这几个：

```
blkio.throttle.read_bps_device
blkio.throttle.read_iops_device
blkio.throttle.write_bps_device
blkio.throttle.write_iops_device
blkio.weight
blkio.weight_device
```

这些都是用来控制进程的磁盘 io 的。很明显地分成两类，其中带“throttle”的，顾名思义就是节流阀，将流量限制在某个值下。而“weight”就是分配 io 的权重。

“throttle”的那四个参数看名字就知道是做什么用的。拿 blkio.throttle.read_bps_device 来限制每秒能读取的字节数。先跑点 io 出来

```
dd if=/dev/sda of=/dev/null &
[1] 2750 
```

用 iotop 看看目前的 io

```
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
 2750 be/4 root       66.76 M/s    0.00 B/s  0.00 % 68.53 % dd if=/dev/sda of=/dev/null
...
```

然后修改一下资源限制，把进程加入控制组

```
echo '8:0   1048576' >/sys/fs/cgroup/blkio/foo/blkio.throttle.read_bps_device
echo 2750 >/sys/fs/cgroup/blkio/foo/tasks
```

这里的 8:0 就是对应块设备的主设备号和副设备号。可以通过 `ls -l` 设备文件名查看。如

```
# ls -l /dev/sda
brw-rw----. 1 root disk 8, 0 Oct 24 11:27 /dev/sda
```

这里的 8, 0 就是对应的设备号。所以，cgroups 可以对不同的设备做不同的限制。然后来看看效果

```
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
 2750 be/4 root      989.17 K/s    0.00 B/s  0.00 % 96.22 % dd if=/dev/sda of=/dev/null
...
```

可见，进程的每秒读取立马就降到了 1MB 左右。要解除限制，写入如 “8:0 0” 到文件中即可

对写入的限制也类似。只是测试的时候要注意，dd 需要加上 oflag=sync，避免文件系统缓存影响。如

```
dd if=/dev/zero of=/data/test.dat oflag=sync
```

还有别犯傻把 of 输出直接写设备文件 :-)

不过需要注意的是，这种方式对小于采样间隔里产生的大量 io 是没用的。比如，就算在 1s 内产生一个每秒写入 100M 的峰值，也不会因此被限制掉。

再看看 blkio.weight 。blkio 的 throttle 和 weight 方式和 cpu 子系统的 quota 和 shares 有点像，都是一种是绝对限制，另一种是相对限制，并且在不繁忙的时候可以充分利用资源，权重值的范围在 10 – 1000 之间。

测试权重方式要麻烦一点。因为不是绝对限制，所以会受到文件系统缓存的影响。如在虚拟机中测试，要关闭虚机如我用的 VirtualBox 在宿主机上的缓存。如要测试读 io 的效果，先生成两个几个 G 的大文件 /tmp/file_1，/tmp/file_2 ，可以用 dd 搞。然后设置两个权重

```
# echo 500 >/sys/fs/cgroup/blkio/foo/blkio.weight
# echo 100 >/sys/fs/cgroup/blkio/bar/blkio.weight
```

测试前清空文件系统缓存，以免干扰测试结果

```
sync
echo 3 >/proc/sys/vm/drop_caches
```

在这两个控制组中用 dd 产生 io 测试效果。

```
# cgexec -g "blkio:foo" dd if=/tmp/file_1 of=/dev/null &
[1] 1838
# cgexec -g "blkio:bar" dd if=/tmp/file_2 of=/dev/null &
[2] 1839
```

还是用 iotop 看看效果

```
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
 1839 be/4 root       48.14 M/s    0.00 B/s  0.00 % 99.21 % dd if=/tmp/file_2 of=/dev/null
 1838 be/4 root      223.59 M/s    0.00 B/s  0.00 % 16.44 % dd if=/tmp/file_1 of=/dev/null
```

两个进程每秒读的字节数虽然会不断变动，但是大致趋势还是维持在 1:5 左右，和设定的 weight 比例一致。blkio.weight_device 是分设备的。写入时，前面再加上设备号即可。

blkio 子系统里还有很多统计项

- blkio.time

  各设备的 io 访问时间，单位毫秒

- blkio.sectors

  换入者或出各设备的扇区数

- blkio.io_serviced

  各设备中执行的各类型 io 操作数，分read、write、sync、async 和 total

- blkio.io_service_bytes

  各类型 io 换入者或出各设备的字节数

- blkio.io_service_time

  各设备中执行的各类型 io 时间，单位微秒

- blkio.io_wait_time

  各设备中各类型 io 在队列中的 等待时间

- blkio.io_merged

  各设备中各类型 io 请求合并的次数

- blkio.io_queued

  各设备中各类型 io 请求当前在队列中的数量

通过这些统计项更好地统计、监控进程的 io 情况
用

```
echo 1 >blkio.reset_stats
```

可以将所有统计项清零。

> 原文链接：http://xiezhenye.com/2013/10/%e7%94%a8-cgroups-%e7%ae%a1%e7%90%86-cpu-%e8%b5%84%e6%ba%90.html
