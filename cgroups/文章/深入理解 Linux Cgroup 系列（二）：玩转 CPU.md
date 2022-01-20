# 深入理解 Linux Cgroup 系列（二）：玩转 CPU

上篇文章主要介绍了 cgroup 的一些基本概念，包括其在 `CentOS` 系统中的默认设置和控制工具，并以 CPU 为例阐述 cgroup 如何对资源进行控制。这篇文章将会通过具体的示例来演示如何通过 cgroup 来限制 `CPU` 的使用以及不同的 cgroup 设置对性能的影响。

## 1. 查看当前 cgroup 信息

有两种方法来查看系统的当前 cgroup 信息。第一种方法是通过 `systemd-cgls` 命令来查看，它会返回系统的整体 cgroup 层级，cgroup 树的最高层由 `slice` 构成，如下所示：

```shell
$ systemd-cgls --no-page
├─1 /usr/lib/systemd/systemd --switched-root --system --deserialize 22
├─user.slice
│ ├─user-1000.slice
│ │ └─session-11.scope
│ │   ├─9507 sshd: tom [priv]
│ │   ├─9509 sshd: tom@pts/3
│ │   └─9510 -bash
│ └─user-0.slice
│   └─session-1.scope
│     ├─ 6239 sshd: root@pts/0
│     ├─ 6241 -zsh
│     └─11537 systemd-cgls --no-page
└─system.slice
  ├─rsyslog.service
  │ └─5831 /usr/sbin/rsyslogd -n
  ├─sshd.service
  │ └─5828 /usr/sbin/sshd -D
  ├─tuned.service
  │ └─5827 /usr/bin/python2 -Es /usr/sbin/tuned -l -P
  ├─crond.service
  │ └─5546 /usr/sbin/crond -n
```

可以看到系统 cgroup 层级的最高层由 `user.slice` 和 `system.slice` 组成。因为系统中没有运行虚拟机和容器，所以没有 `machine.slice`，所以当 CPU 繁忙时，`user.slice` 和 `system.slice` 会各获得 `50%` 的 CPU 使用时间。

user.slice 下面有两个子 slice：`user-1000.slice` 和 `user-0.slice`，每个子 slice 都用 User ID (`UID`) 来命名，因此我们很容易识别出哪个 slice 属于哪个用户。例如：从上面的输出信息中可以看出 `user-1000.slice` 属于用户 tom，`user-0.slice` 属于用户 root。

`systemd-cgls` 命令提供的只是 cgroup 层级的静态信息快照，要想查看 cgroup 层级的动态信息，可以通过 `systemd-cgtop` 命令查看：

```shell
$ systemd-cgtop
Path                                                                                                                                       Tasks   %CPU   Memory  Input/s Output/s

/                                                                                                                                            161    1.2   161.0M        -        -
/system.slice                                                                                                                                  -    0.1        -        -        -
/system.slice/vmtoolsd.service                                                                                                                 1    0.1        -        -        -
/system.slice/tuned.service                                                                                                                    1    0.0        -        -        -
/system.slice/rsyslog.service                                                                                                                  1    0.0        -        -        -
/system.slice/auditd.service                                                                                                                   1      -        -        -        -
/system.slice/chronyd.service                                                                                                                  1      -        -        -        -
/system.slice/crond.service                                                                                                                    1      -        -        -        -
/system.slice/dbus.service                                                                                                                     1      -        -        -        -
/system.slice/gssproxy.service                                                                                                                 1      -        -        -        -
/system.slice/lvm2-lvmetad.service                                                                                                             1      -        -        -        -
/system.slice/network.service                                                                                                                  1      -        -        -        -
/system.slice/polkit.service                                                                                                                   1      -        -        -        -
/system.slice/rpcbind.service                                                                                                                  1      -        -        -        -
/system.slice/sshd.service                                                                                                                     1      -        -        -        -
/system.slice/system-getty.slice/getty@tty1.service                                                                                            1      -        -        -        -
/system.slice/systemd-journald.service                                                                                                         1      -        -        -        -
/system.slice/systemd-logind.service                                                                                                           1      -        -        -        -
/system.slice/systemd-udevd.service                                                                                                            1      -        -        -        -
/system.slice/vgauthd.service                                                                                                                  1      -        -        -        -
/user.slice                                                                                                                                    3      -        -        -        -
/user.slice/user-0.slice/session-1.scope                                                                                                       3      -        -        -        -
/user.slice/user-1000.slice                                                                                                                    3      -        -        -        -
/user.slice/user-1000.slice/session-11.scope                                                                                                   3      -        -        -        -
/user.slice/user-1001.slice/session-8.scope                                                                                                    3      -        -        -        -
```

systemd-cgtop 提供的统计数据和控制选项与 `top` 命令类似，但该命令只显示那些开启了资源统计功能的 service 和 slice。比如：如果你想开启 `sshd.service` 的资源统计功能，可以进行如下操作：

```shell
$ systemctl set-property sshd.service CPUAccounting=true MemoryAccounting=true
```

该命令会在 `/etc/systemd/system/sshd.service.d/` 目录下创建相应的配置文件：

```shell
$ ll /etc/systemd/system/sshd.service.d/
总用量 8
4 -rw-r--r-- 1 root root 28 5月  31 02:24 50-CPUAccounting.conf
4 -rw-r--r-- 1 root root 31 5月  31 02:24 50-MemoryAccounting.conf

$ cat /etc/systemd/system/sshd.service.d/50-CPUAccounting.conf
[Service]
CPUAccounting=yes

$ cat /etc/systemd/system/sshd.service.d/50-MemoryAccounting.conf
[Service]
MemoryAccounting=yes
```

配置完成之后，再重启 `sshd` 服务：

```shell
$ systemctl daemon-reload
$ systemctl restart sshd
```

这时再重新运行 systemd-cgtop 命令，就能看到 sshd 的资源使用统计了：



![图片](https://mmbiz.qpic.cn/mmbiz_png/qFG6mghhA4YX5zs1YbLIib3BSW91XHC0hdp3HAR71skLicfOU95vHzM8E3cYw7csNvNhlEluGUjzt2nkw8Bf4ib4A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



> 开启资源使用量统计功能可能会增加系统的负载，因为资源统计也要消耗 CPU 和内存，大多数情况下使用 `top` 命令来查看就足够了。当然了，这是 Linux 系统嘛，一切的控制权都在你自己手里，你想怎么做就怎么做。

## 2. 分配 CPU 相对使用时间

通过上篇文章的学习我们知道了 CPU `shares` 可以用来设置 CPU 的相对使用时间，接下来我们就通过实践来验证一下。

> 下面所做的实验都是在单核 CPU 的系统上进行的，多核与单核的情况完全不同，文末会单独讨论。

测试对象是 1 个 service 和两个普通用户，其中用户 `tom` 的 UID 是 1000，可以通过以下命令查看：

```shell
$ cat /etc/passwd|grep tom
tom:x:1000:1000::/home/tom:/bin/bash
```

创建一个 `foo.service`：

```shell
$ cat /etc/systemd/system/foo.service
[Unit]
Description=The foo service that does nothing useful
After=remote-fs.target nss-lookup.target

[Service]
ExecStart=/usr/bin/sha1sum /dev/zero
ExecStop=/bin/kill -WINCH ${MAINPID}

[Install]
WantedBy=multi-user.target
```

`/dev/zero` 在 linux 系统中是一个特殊的设备文件，当你读它的时候，它会提供无限的空字符，因此 foo.service 会不断地消耗 CPU 资源。现在我们将 foo.service 的 CPU shares 改为 `2048`：

```shell
$ mkdir /etc/systemd/system/foo.service.d
$ cat << EOF > /etc/systemd/system/foo.service.d/50-CPUShares.conf
[Service]
CPUShares=2048
EOF
```

由于系统默认的 CPU shares 值为 `1024`，所以设置成 2048 后，在 CPU 繁忙的情况下，`foo.service` 会尽可能获取 `system.slice` 的所有 CPU 使用时间。

现在通过 `systemctl start foo.service` 启动 foo 服务，并使用 `top` 命令查看 CPU 使用情况：



![图片](https://mmbiz.qpic.cn/mmbiz_png/qFG6mghhA4YX5zs1YbLIib3BSW91XHC0hEdZKhbwuUBF1SIN70c2EDiaDiaAELrXia0MNfrC6HKV2XNxibG4tIjP5Fg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



目前没有其他进程在消耗 CPU，所以 foo.service 可以使用几乎 100% 的 CPU。

现在我们让用户 `tom` 也参与进来，先将 `user-1000.slice` 的 CPU shares 设置为 `256`：

```shell
$ systemctl set-property user-1000.slice CPUShares=256
```

使用用户 `tom` 登录该系统，然后执行命令 `sha1sum /dev/zero`，再次查看 CPU 使用情况：



![图片](https://mmbiz.qpic.cn/mmbiz_png/qFG6mghhA4YX5zs1YbLIib3BSW91XHC0hVw9wYxIdbzkQjkMNlNsrPZ9xoZ1Yu8qYQIhZuHhEMJzzj6LOhh1MCw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



现在是不是感到有点迷惑了？foo.service 的 CPU shares 是 `2048`，而用户 tom 的 CPU shares 只有 `256`，难道用户 `tom` 不是应该只能使用 10% 的 CPU 吗？回忆一下我在上一节提到的，当 CPU 繁忙时，`user.slice` 和 `system.slice` 会各获得 `50%` 的 CPU 使用时间。而这里恰好就是这种场景，同时 `user.slice` 下面只有 sha1sum 进程比较繁忙，所以会获得 50% 的 CPU 使用时间。

最后让用户 `jack` 也参与进来，他的 CPU shares 是默认值 1024。使用用户 `jack` 登录该系统，然后执行命令 `sha1sum /dev/zero`，再次查看 CPU 使用情况：



![图片](https://mmbiz.qpic.cn/mmbiz_png/qFG6mghhA4YX5zs1YbLIib3BSW91XHC0hk4SKNV513vNVcYtlbIFB8hyfAnLaKWofS1vBMLFm8Mx39MOvJqUoUA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



上面我们已经提到，这种场景下 `user.slice` 和 `system.slice` 会各获得 `50%` 的 CPU 使用时间。用户 tom 的 CPU shares 是 `256`，而用户 jack 的 CPU shares 是 `1024`，因此用户 jack 获得的 CPU 使用时间是用户 tom 的 `4` 倍。

## 3. 分配 CPU 绝对使用时间

上篇文章已经提到，如果想严格控制 CPU 资源，设置 CPU 资源的使用上限，即不管 CPU 是否繁忙，对 CPU 资源的使用都不能超过这个上限，可以通过 `CPUQuota` 参数来设置。下面我们将用户 tom 的 CPUQuota 设置为 `5%`：

```shell
$ systemctl set-property user-1000.slice CPUQuota=5%
```

这时你会看到用户 tom 的 sha1sum 进程只能获得 5% 左右的 CPU 使用时间。



![图片](https://mmbiz.qpic.cn/mmbiz_png/qFG6mghhA4YX5zs1YbLIib3BSW91XHC0hQ2NBe7qOQ2mIIFoqvVYfVnr2ic562zL9nVmkbiaUPskSJWzrhs8Onvcw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



如果此时停止 `foo.service`，关闭用户 jack 的 sha1sum 进程，你会看到用户 tom 的 sha1sum 进程仍然只能获得 `5%`左右的 CPU 使用时间。



![图片](https://mmbiz.qpic.cn/mmbiz_png/qFG6mghhA4YX5zs1YbLIib3BSW91XHC0hL0Jus7F3AGAwa6jzNpWOdAsBJ5f8XKND1NkXoKgcVXxWZjWUNOQTAA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



如果某个非核心服务很消耗 CPU 资源，你可以通过这种方法来严格限制它对 CPU 资源的使用，防止对系统中其他重要的服务产生影响。

## 4. 动态设置 cgroup

cgroup 相关的所有操作都是基于内核中的 cgroup virtual filesystem，使用 cgroup 很简单，挂载这个文件系统就可以了。系统默认情况下都是挂载到 `/sys/fs/cgroup` 目录下，当 service 启动时，会将自己的 cgroup 挂载到这个目录下的子目录。以 `foo.service` 为例：

先进入 `system.slice` 的 CPU 子系统：

```shell
$ cd /sys/fs/cgroup/cpu,cpuacct/system.slice
```

查看 foo.service 的 cgroup 目录：

```shell
$ ls foo.*
zsh: no matches found: foo.*
```

因为 foo.service 没有启动，所以没有挂载 cgroup 目录，现在启动 foo.service，再次查看它的 cgroup 目录：

```shell
$ ls foo.serice
cgroup.clone_children  cgroup.procs  cpuacct.usage         cpu.cfs_period_us  cpu.rt_period_us   cpu.shares  notify_on_release
cgroup.event_control   cpuacct.stat  cpuacct.usage_percpu  cpu.cfs_quota_us   cpu.rt_runtime_us  cpu.stat    tasks
```

也可以查看它的 PID 和 CPU shares：

```shell
$ cat foo.service/tasks
20225

$ cat foo.service/cpu.shares
2048
```

> 理论上我们可以在 `/sys/fs/cgroup` 目录中动态改变 cgroup 的配置，但我不建议你在生产环境中这么做。如果你想通过实验来深入理解 cgroup，可以多折腾折腾这个目录。

## 5. 如果是多核 CPU 呢？

上面的所有实验都是在单核 CPU 上进行的，下面我们简单讨论一下多核的场景，以 2 个 CPU 为例。

首先来说一下 CPU shares，shares 只能针对单核 CPU 进行设置，也就是说，无论你的 shares 值有多大，该 cgroup 最多只能获得 100% 的 CPU 使用时间（即 1 核 CPU）。还是用本文第 2 节的例子，将 foo.service 的 CPU shares 设置为 2048，启动 foo.service，这时你会看到 foo.service 仅仅获得了 100% 的 CPU 使用时间，并没有完全使用两个 CPU 核：



![图片](https://mmbiz.qpic.cn/mmbiz_png/qFG6mghhA4YX5zs1YbLIib3BSW91XHC0hGh07XJhTr3bORg524Hsl2V0V8tz533iaVSGgEspBQAujd6eTFIkRMBw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



再使用用户 `tom` 登录系统，执行命令 `sha1sum /dev/zero`，你会发现用户 tom 的 sha1sum 进程和 foo.service 各使用 1 个 CPU 核：



![图片](https://mmbiz.qpic.cn/mmbiz_png/qFG6mghhA4YX5zs1YbLIib3BSW91XHC0hqfpVyNgkllDDSB0Hrk7hfsgYC4ofCS3TDlibMLUbghb2vNKrhgOcVmA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



再来说说 CPUQuota，这个上篇文章结尾已经提过了，如要让一个 cgroup 完全使用两个 CPU 核，可以通过 CPUQuota 参数来设置。例如：

```shell
$ systemctl set-property foo.service CPUQuota=200%
```

至于进程最后能不能完全使用两个 CPU 核，就要看它自身的设计支持不支持了。

## 6. 总结

本文通过具体的示例来观察不同的 cgroup 设置对性能的影响，下面一篇文章将会演示如何通过 cgroup 来限制内存的使用。

> 原文链接：https://mp.weixin.qq.com/s?__biz=MzU1MzY4NzQ1OA==&mid=2247484160&idx=1&sn=d593f4693a07a2a5f958fe2e0f489edd&chksm=fbee43cdcc99cadbd2df86c57d051c9742b2f4864071f6269156ec405662037c1bb9eea4c89d&scene=21#wechat_redirect

