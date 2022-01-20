# Cgroup - 从CPU资源隔离说起

## 什么是Cgroup？

> cgroups，其名称源自控制组群（control groups）的简写，是Linux内核的一个功能，用来限制，控制与分离一个进程组群的资源（如CPU、内存、磁盘输入输出等）。

引用官方说法总是那么冰冷的让人不好理解，所以我还是稍微解释一下：

一个正在运行着服务的计算机系统，跟我们高中上课的情景还是很相似的。如果把系统中的每个进程理解为一个同学的话，那么班主任就是操作系统的核心（kernel），负责管理班里的同学。而cgroup，就是班主任控制学生行为的一种手段，所以，它起名叫control groups。

既然是一种控制手段，那么cgroup能控制什么呢？当然是资源啦！对于计算机来说，资源大概可以分成以下几个部分：

- 计算资源
- 内存资源
- io资源
- 网络资源

这就是我们常说的内核四大子系统。当我们学习内核的时候，我们也基本上是围绕这四大子系统进行研究。 我们今天要讨论的，主要是cgroup是如何对系统中的CPU资源进行隔离和分配的。其他资源的控制，我们以后有空再说喽。

## 如何看待CPU资源？

*由于进程和线程在Linux的CPU调度看来没啥区别，所以本文后续都会用**进程**这个名词来代表内核的调度对象，一般来讲也包括线程*

如果要分配资源，我们必须先搞清楚这个资源是如何存在的，或者说是如何组织的。我想CPU大家都不陌生，我们都在系统中用过各种工具查看过CPU的使用率，比如说以下这个命令和它的输出：

```
[zorro@zorrozou-pc0 ~]$ mpstat -P ALL 1 1
Linux 4.2.5-1-ARCH (zorrozou-pc0)     2015年12月22日 _x86_64_     (4 CPU)mt

16时01分08秒  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
16时01分09秒  all    0.25    0.00    0.25    0.00    0.00    0.00    0.00    0.00    0.00   99.50
16时01分09秒    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
16时01分09秒    1    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
16时01分09秒    2    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
16时01分09秒    3    0.00    0.00    1.00    0.00    0.00    0.00    0.00    0.00    0.00   99.00

Average:     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
Average:     all    0.25    0.00    0.25    0.00    0.00    0.00    0.00    0.00    0.00   99.50
Average:       0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       1    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       2    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
Average:       3    0.00    0.00    1.00    0.00    0.00    0.00    0.00    0.00    0.00   99.00
```

显示的内容具体什么意思，希望大家都能了解，我就不在这细解释了。根据显示内容我们知道，这个计算机有4个cpu核心，目前的cpu利用率几乎是0，就是说系统整体比较闲。

从这个例子大概可以看出，我们对cpu资源的评估一般有两个观察角度：

- 核心个数
- 百分比

目前的计算机基本都是多核甚至多cpu系统，一个服务器上存在几个到几十个cpu核心的情况都很常见。所以，从这个角度看，cgroup应该提供一种手段，可以给进程们指定它们可以占用的cpu核心，以此来做到cpu计算资源的隔离。 百分比这个概念我们需要多解释一下：这个百分比究竟是怎么来的呢？难道每个cpu核心的计算能力就像一个带刻度表的水杯一样？一个进程要占用就会占用到它的一定刻度么？

当然不是啦！这个cpu的百分比是按时间比率计算的。基本思路是：一个CPU一般就只有两种状态，要么被占用，要么不被占用。当有多个进程要占用cpu的时候，那么操作系统在一个cpu核心上是进行分时处理的。比如说，我们把一秒钟分成1000份，那么每一份就是1毫秒，假设现在有5个进程都要用cpu，那么我们就让它们5个轮着使用，比如一人一毫秒，那么1秒过后，每个进程只占用了这个CPU的200ms，使用率为20%。整体cpu使用比率为100%。 同理，如果只有一个进程占用，而且它只用了300ms，那么在这一秒的尺度看来，cpu的占用时间是30％。于是显示出来的状态就是占用30%的CPU时间。

这就是内核是如何看待和分配计算资源的。当然实际情况要比这复杂的多，但是基本思路就是这样。Linux内核是通过CPU调度器CFS－－完全公平调度器对CPU的时间进行调度的，由于本文的侧重点是cgroup而不是CFS，对这个题目感兴趣的同学可以到[这里](http://www.linuxjournal.com/magazine/completely-fair-scheduler?page=0,1)进一步学习。CFS是内核可以实现真对CPU资源隔离的核心手段，因此，理解清楚CFS对理解清楚CPU资源隔离会有很大的帮助。

## 如何隔离CPU资源？

根据CPU资源的组织形式，我们就可以理解cgroup是如何对CPU资源进行隔离的了。

无非也是两个思路，一个是分配核心进行隔离，另一个是分配CPU使用时间进行隔离。

再介绍如何做隔离之前，我们先来介绍一下我们的实验系统环境：没有特殊情况，我们的实验环境都是一台24核心、128G内存的服务器，上面安装的系统可以认为是Centos7.

### 搭建测试环境

我们将使用cgconfig服务和cgred服务对cgroup进行配置和使用。我们将配置两个group，一个叫zorro，另一个叫jerry。它们分别也是系统上的两个账户，其中zorro用户所运行的进程都默认在zorro group中进行限制，jerry用户所运行的进程都放到jerry group中进行限制。配置文件内容和配置方法如下：

*本文并不对以下配置方法的具体含义做解释，大家只要知道如此配置可以达到相关试验环境要求即可。如果大家对配置的细节感兴趣，可以自行查找相关资料进行学习。*

首先添加两个用户，zorro和jerry：

```
[root@zorrozou-pc ~]# useradd zorro
[root@zorrozou-pc ~]# useradd jerry
```

修改/etc/cgrules.conf，添加两行内容：

```
[root@zorrozou-pc ~]# cat /etc/cgrules.conf 
zorro        cpu,cpuacct    zorro
jerry        cpu,cpuacct    jerry
```

修改/etc/cgconfig.conf，添加以下内容：

```
[root@zorrozou-pc ~]# cat /etc/cgconfig.conf
mount {
    cpuset    = /cgroup/cpuset;
    cpu    = /cgroup/cpu;
    cpuacct    = /cgroup/cpuacct;
    memory    = /cgroup/memory;
    devices    = /cgroup/devices;
    freezer    = /cgroup/freezer;
    net_cls    = /cgroup/net_cls;
    blkio    = /cgroup/blkio;
}

group zorro {
    cpuset {
        cpuset.cpus = "1,2";
    }
}

group jerry {
    cpuset {
        cpuset.cpus = "3,4";
    }
}
```

重启cgconfig服务和cgred服务：

```
[root@zorrozou-pc ~]# service cgconfig restart
[root@zorrozou-pc ~]# service cgred restart
```

根据上面的配置，我们给zorro组合jerry组分别配置了cpuset的隔离设置，那么在cgroup的相关目录下应该出现相关组的配置文件： *本文中所出现的**组**的含义，如无特殊说明都是对应cgroup的控制组，而非用户组身份。* 我们可以通过检查相关目录内容来检查一下环境是否配置完成：

```
[root@zorrozou-pc ~]# ls /cgroup/cpuset/{zorro,jerry}
/cgroup/cpuset/jerry:
cgroup.clone_children  cpuset.cpu_exclusive  cpuset.mem_exclusive   cpuset.memory_pressure     cpuset.mems                      cpuset.stat
cgroup.event_control   cpuset.cpuinfo        cpuset.mem_hardwall    cpuset.memory_spread_page  cpuset.sched_load_balance        notify_on_release
cgroup.procs           cpuset.cpus           cpuset.memory_migrate  cpuset.memory_spread_slab  cpuset.sched_relax_domain_level  tasks

/cgroup/cpuset/zorro:
cgroup.clone_children  cpuset.cpu_exclusive  cpuset.mem_exclusive   cpuset.memory_pressure     cpuset.mems                      cpuset.stat
cgroup.event_control   cpuset.cpuinfo        cpuset.mem_hardwall    cpuset.memory_spread_page  cpuset.sched_load_balance        notify_on_release
cgroup.procs           cpuset.cpus           cpuset.memory_migrate  cpuset.memory_spread_slab  cpuset.sched_relax_domain_level  tasks
```

至此，我们的实验环境已经搭建完成。

### 测试用例设计

无论是针对CPU核心的隔离还是针对CPU时间的隔离，我们都需要一个可以消耗大量的CPU运算资源的程序来进行测试，考虑到我们是一个多CPU核心的环境，所以我们的测试用例一定也是一个可以并发使用多个CPU核心的计算型测试用例。针对这个需求，我们首先设计了一个使用多线程并发进行筛质数的简单程序。这个程序可以打印出从100010001到100020000数字范围内的质数有哪些。并发48个工作线程从一个共享的count整型变量中取数进行计算。程序源代码如下：

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define NUM 48
#define START 100010001
#define END 100020000

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
static int count = 0;

void *prime(void *p)
{
    int n, i, flag;

    while (1) {
            if (pthread_mutex_lock(&mutex) != 0) {
                    perror("pthread_mutex_lock()");
                    pthread_exit(NULL);
            }
            while (count == 0) {
                    if (pthread_cond_wait(&cond, &mutex) != 0) {
                            perror("pthread_cond_wait()");
                            pthread_exit(NULL);
                    }
            }
            if (count == -1) {
                    if (pthread_mutex_unlock(&mutex) != 0) {
                            perror("pthread_mutex_unlock()");
                            pthread_exit(NULL);
                    }
                    break;
            }
            n = count;
            count = 0;
            if (pthread_cond_broadcast(&cond) != 0) {
                    perror("pthread_cond_broadcast()");
                    pthread_exit(NULL);
            }
            if (pthread_mutex_unlock(&mutex) != 0) {
                    perror("pthread_mutex_unlock()");
                    pthread_exit(NULL);
            }
            flag = 1;
            for (i=2;i<n/2;i++) {
                    if (n%i == 0) {
                            flag = 0;
                            break;
                    }
            }
            if (flag == 1) {
                    printf("%d is a prime form %d!\n", n, pthread_self());
            }
    }
    pthread_exit(NULL);
}


int main(void)
{
    pthread_t tid[NUM];
    int ret, i;

    for (i=0;i<NUM;i++) {
            ret = pthread_create(&tid[i], NULL, prime, NULL);
            if (ret != 0) {
                    perror("pthread_create()");
                    exit(1);
            }
    }

    for (i=START;i<END;i+=2) {
            if (pthread_mutex_lock(&mutex) != 0) {
                    perror("pthread_mutex_lock()");
                    pthread_exit(NULL);
            }
            while (count != 0) {
                    if (pthread_cond_wait(&cond, &mutex) != 0) {
                            perror("pthread_cond_wait()");
                            pthread_exit(NULL);
                    }
            }
            count = i;
            if (pthread_cond_broadcast(&cond) != 0) {
                    perror("pthread_cond_broadcast()");
                    pthread_exit(NULL);
            }
            if (pthread_mutex_unlock(&mutex) != 0) {
                    perror("pthread_mutex_unlock()");
                    pthread_exit(NULL);
            }
    }

    if (pthread_mutex_lock(&mutex) != 0) {
            perror("pthread_mutex_lock()");
            pthread_exit(NULL);
    }
    while (count != 0) {
            if (pthread_cond_wait(&cond, &mutex) != 0) {
                    perror("pthread_cond_wait()");
                    pthread_exit(NULL);
            }
    }
    count = -1;
    if (pthread_cond_broadcast(&cond) != 0) {
            perror("pthread_cond_broadcast()");
            pthread_exit(NULL);
    }
    if (pthread_mutex_unlock(&mutex) != 0) {
            perror("pthread_mutex_unlock()");
            pthread_exit(NULL);
    }

    for (i=0;i<NUM;i++) {
            ret = pthread_join(tid[i], NULL);
            if (ret != 0) {

                    perror("pthread_join()");
                    exit(1);
            }
    }

    exit(0);
}
```

我们先来看一下这个程序在不做限制的情况下的执行效果和执行时间：

```
[root@zorrozou-pc ~/test]# time ./prime_thread
......

100019603 is a prime form 2068363008!
100019471 is a prime form 1866938112!
100019681 is a prime form 1934079744!
100019597 is a prime form 1875330816!
100019701 is a prime form 2059970304!
100019657 is a prime form 1799796480!
100019761 is a prime form 1808189184!
100019587 is a prime form 1824974592!
100019659 is a prime form 2076755712!
100019837 is a prime form 1959257856!
100019923 is a prime form 2034792192!
100019921 is a prime form 1908901632!
100019729 is a prime form 1850152704!
100019863 is a prime form -2109106432!
100019911 is a prime form -2125891840!
100019749 is a prime form 2101933824!
100019879 is a prime form 2026399488!
100019947 is a prime form 1942472448!
100019693 is a prime form 1917294336!
100019683 is a prime form 2051577600!
100019873 is a prime form 2110326528!
100019929 is a prime form -2134284544!
100019977 is a prime form 1892116224!

real    0m8.945s
user    3m32.095s
sys    0m0.235s



[root@zorrozou-pc ~]# mpstat -P ALL 1
11:21:51     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
11:21:52     all   99.92    0.00    0.08    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52       0  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52       1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52       2  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52       3  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52       4   99.00    0.00    1.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52       5  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52       6  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52       7  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52       8  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52       9  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52      10  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52      11   99.01    0.00    0.99    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52      12  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52      13  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52      14  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52      15  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52      16   99.01    0.00    0.00    0.00    0.99    0.00    0.00    0.00    0.00    0.00
11:21:52      17  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52      18  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52      19  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52      20  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52      21  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52      22  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
11:21:52      23  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
```

经过多次测试，程序执行时间基本稳定：

```
[root@zorrozou-pc ~/test]# time ./prime_thread &> /dev/null

real    0m8.953s
user    3m31.950s
sys    0m0.227s
[root@zorrozou-pc ~/test]# time ./prime_thread &> /dev/null

real    0m8.932s
user    3m31.984s
sys    0m0.231s
[root@zorrozou-pc ~/test]# time ./prime_thread &> /dev/null

real    0m8.954s
user    3m31.794s
sys    0m0.224s
```

所有相关环境都准备就绪，后续我们将在此程序的基础上进行各种隔离的测试。

### 针对CPU核心进行资源隔离

针对CPU核心进行隔离，其实就是把要运行的进程绑定到指定的核心上运行，通过让不同的进程占用不同的核心，以达到运算资源隔离的目的。其实对于Linux来说，这种手段并不新鲜，也并不是在引入cgroup之后实现的，早在内核使用O1调度算法的时候，就已经支持通过taskset命令来绑定进程的cpu核心了。

好的，废话少说，我们来看看这在cgroup中是怎么配置的。

其实通过刚才的/etc/cgconfig.conf配置文件的内容，我们已经配置好了针对不同的组占用核心的设置，来回顾一下：

```
group zorro {
    cpuset {
        cpuset.cpus = "1,2";
    }
}
```

这段配置内容就是说，将zorro组中的进程都放在编号为1，2的cpu核心上运行。这里要说明的是，cpu核心的编号一般是从0号开始的。24个核心的服务器编号范围是从0-23.我们可以通过查看/proc/cpuinfo的内容来确定相关物理cpu的个数和核心的个数。我们截取一段来看一下：

```
[root@zorrozou-pc ~/test]# cat /proc/cpuinfo
processor    : 23
vendor_id    : GenuineIntel
cpu family    : 6
model        : 63
model name    : Intel(R) Xeon(R) CPU E5-2620 v3 @ 2.40GHz
stepping    : 2
microcode    : 0x2b
cpu MHz        : 2599.968
cache size    : 15360 KB
physical id    : 1
siblings    : 12
core id        : 5
cpu cores    : 6
apicid        : 27
initial apicid    : 27
fpu        : yes
fpu_exception    : yes
cpuid level    : 15
wp        : yes
flags        : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm ida arat epb xsaveopt pln pts dtherm tpr_shadow vnmi flexpriority ept vpid fsgsbase tsc_adjust bmi1 avx2 smep bmi2 erms invpcid
bogomips    : 4796.38
clflush size    : 64
cache_alignment    : 64
address sizes    : 46 bits physical, 48 bits virtual
power management:
```

其中*processor : 23*就是核心编号，说明我们当前显示的是这个服务器上的第24个核心，*physical id : 1*表示的是这个核心所在的物理cpu是哪个。这个编号也是从0开始，表示这个核心在第二个物理cpu上。那就意味着，我这个服务器是一个双物理cpu的服务器，那就可能意味着我们的系统时NUMA架构。另外还有一个要注意的是*core id : 5*这个子段，这里面隐含着一个可能的含义：你的服务器是否开启了超线程。众所周知，开启了超线程的服务器，在系统看来，一个核心会编程两个核心来看待。那么我们再确定一下是否开了超线程，可以grep一下：

```
[root@zorrozou-pc ~/test]# cat /proc/cpuinfo |grep -e "core id" -e "physical id"
physical id    : 0
core id        : 0
physical id    : 0
core id        : 1
physical id    : 0
core id        : 2
physical id    : 0
core id        : 3
physical id    : 0
core id        : 4
physical id    : 0
core id        : 5
physical id    : 1
core id        : 0
physical id    : 1
core id        : 1
physical id    : 1
core id        : 2
physical id    : 1
core id        : 3
physical id    : 1
core id        : 4
physical id    : 1
core id        : 5
physical id    : 0
core id        : 0
physical id    : 0
core id        : 1
physical id    : 0
core id        : 2
physical id    : 0
core id        : 3
physical id    : 0
core id        : 4
physical id    : 0
core id        : 5
physical id    : 1
core id        : 0
physical id    : 1
core id        : 1
physical id    : 1
core id        : 2
physical id    : 1
core id        : 3
physical id    : 1
core id        : 4
physical id    : 1
core id        : 5
```

这个内容显示出我的服务器是开启了超线程的，因为有同一个*physical id : 1*的*core id : 5*可能出现两次，那么就说明这个物理cpu上的5号核心在系统看来出现了2个，那么肯定意味着开了超线程。

> 我在此要强调超线程这个事情，因为在一个开启了超线程的服务器上运行我们当前的测试用例是很可能得不到预想的结果的。因为从原理上看，超线程技术虽然使cpu核心变多了，但是在本测试中并不能反映出相应的性能提高。我们后续会通过cpuset的资源隔离先来说明一下这个问题，然后在后续的测试中，我们将采用一些手段规避这个问题。

我们先通过一个cpuset的配置来反映一下超线程对本测试的影响，顺便学习一下cgroup的cpuset配置方法。

1. 不绑定核心测试：

将/etc/cgconfig.conf文件中zorro组相关配置修改为以下状态，之后重启cgconfig服务：

```
group zorro {
    cpuset {
        cpuset.cpus = "0-23";
        cpuset.mems = "0-1";
    }
}

[root@zorrozou-pc ~]# service cgconfig restart
```

切换用户身份到zorro，并察看zorro组的配置：

```
[root@zorrozou-pc ~]# su - zorro
[zorro@zorrozou-pc ~]$ cat /cgroup/cpuset/zorro/cpuset.cpus 
0-23
```

zorro用户对应的进程已经绑定在0-23核心上执行，我们看一下执行结果：

```
[zorro@zorrozou-pc ~/test]$ time ./prime_thread_zorro &> /dev/null

real    0m8.956s
user    3m31.990s
sys    0m0.246s
[zorro@zorrozou-pc ~/test]$ time ./prime_thread_zorro &> /dev/null

real    0m8.944s
user    3m31.956s
sys    0m0.247s
```

执行速度跟刚才一样，这相当于没绑定的情况。下面，我们对zorro组的进程绑定一半的cpu核心进行测试，先测试绑定0-11号核心，将*cpuset.cpus = "0-23"*改为*cpuset.cpus = "0-11"*。

> 请注意每次修改完/etc/cgconfig.conf文件内容都应该重启cgconfig服务，并重新登陆zorro账户。过程不再复述。

将核心绑定到0-11之后的测试结果如下：

```
[zorro@zorrozou-pc ~/test]$ time ./prime_thread_zorro &> /dev/null

real    0m9.457s
user    1m52.773s
sys    0m0.155s
[zorro@zorrozou-pc ~/test]$ time ./prime_thread_zorro &> /dev/null

real    0m9.460s
user    1m52.589s
sys    0m0.153s

14:52:02     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
14:52:03     all   49.92    0.00    0.08    0.00    0.08    0.00    0.00    0.00    0.00   49.92
14:52:03       0  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
14:52:03       1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
14:52:03       2  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
14:52:03       3  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
14:52:03       4  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
14:52:03       5  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
14:52:03       6   99.01    0.00    0.99    0.00    0.00    0.00    0.00    0.00    0.00    0.00
14:52:03       7  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
14:52:03       8  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
14:52:03       9  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
14:52:03      10  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
14:52:03      11  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
14:52:03      12    0.00    0.00    0.00    0.00    2.00    0.00    0.00    0.00    0.00   98.00
14:52:03      13    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
14:52:03      14    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
14:52:03      15    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
14:52:03      16    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
14:52:03      17    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
14:52:03      18    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
14:52:03      19    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
14:52:03      20    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
14:52:03      21    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
14:52:03      22    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
14:52:03      23    0.00    0.00    0.99    0.00    0.00    0.00    0.00    0.00    0.00   99.01
```

此时会发现一个现象，执行的总体时间变化不大，大概慢了0.5秒，但是user时间下降了将近一半。

我们再降核心绑定成0-5,12-17测试一下，就是*cpuset.cpus = "0-5,12-17"*，测试结果如下：

```
[zorro@zorrozou-pc ~/test]$ time ./prime_thread_zorro &> /dev/null

real    0m17.821s
user    3m32.425s
sys    0m0.223s
[zorro@zorrozou-pc ~/test]$ time ./prime_thread_zorro &> /dev/null    
real    0m17.839s
user    3m32.375s
sys    0m0.223s

15:03:03     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
15:03:04     all   49.94    0.00    0.04    0.00    0.04    0.00    0.00    0.00    0.00   49.98
15:03:04       0  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
15:03:04       1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
15:03:04       2  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
15:03:04       3  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
15:03:04       4  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
15:03:04       5  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
15:03:04       6    0.00    0.00    0.99    0.00    0.00    0.00    0.00    0.00    0.00   99.01
15:03:04       7    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
15:03:04       8    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
15:03:04       9    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
15:03:04      10    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
15:03:04      11    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
15:03:04      12  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
15:03:04      13  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
15:03:04      14  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
15:03:04      15   99.01    0.00    0.99    0.00    0.00    0.00    0.00    0.00    0.00    0.00
15:03:04      16   99.01    0.00    0.99    0.00    0.00    0.00    0.00    0.00    0.00    0.00
15:03:04      17  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
15:03:04      18    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
15:03:04      19    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
15:03:04      20    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
15:03:04      21    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
15:03:04      22    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
15:03:04      23    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
```

这次测试的结果就比较符合我们的常识，看上去cpu核心少了一半，于是执行时间增加了几乎一倍。那么是什么原因导致我们绑定到0-11核心的时候看上去性能没有下降呢？

在此我们不去过多讨论超线程的技术细节，简单来说：0-5核心是属于物理cpu0的6个实际核心，6-11是属于物理cpu1的6个实际核心，当我们使用这12个核心的时候，运算覆盖了两个物理cpu的所有真实核心。而12-17核心是对应0-5核心超线程出来的6个核心，18-23则是对应6-11核心超线程出来的6个。我们的测试应用并不能充分利用超线程之后的运算资源，所以，从我们的测试用例角度看来，只要选择了合适核心，12核跟24核的效果几本差别不大。了解了超线程的这个问题，我们后续的测试过程就要注意对比的环境。从本轮测试看来，我们应该用绑定0-5，12-17的测试结果来参考绑定一半cpu核心的效果，而不是绑定到“0-11”上的结果。从测试结果看，减少一半核心之后，确实让运算时间增加了一倍。

> 出个两个思考题吧：
>
> 1. 我们发现第二轮绑定0-11核心测试的user时间和绑定0-23的测试时间减少一倍，而real时间几乎没变，这是为什么？
> 2. 我们发现第三轮绑定0-5，12-17核心测试的user时间和绑定0-23的测试时间几乎一样，而real时间增加了一倍，这是为什么？

至此，如何使用cgroup的cpuset对cpu核心进行资源分配的方法大家应该学会了，这里需要强调一点：

配置中*cpuset.mems = "0-1"*这段配置非常重要，它相当于打开cpuset功能的开关，本身的意义是用来配置cpu使用的内存节点的，不配置这个字段的结果将是cpuset.cpus设置无效。字段具体含义，请大家自行补脑。

### 针对CPU时间进行资源隔离

再回顾一下系统对cpu资源的使用方式－－分时使用。分时使用要有一个基本本的时间调度单元，这个单元的意思是说，在这样一段时间范围内，我们将多少比例分配给某个进程组。我们刚才举的例子是说1秒钟，但是实际情况是1秒钟这个时间周期对计算机来说有点长。Linux内核将这个时间周期定义放在cgroup相关目录下的一个文件里，这个文件在我们服务器上是：

```
[root@zorrozou-pc ~]# cat /cgroup/cpu/zorro/cpu.cfs_period_us 
100000
```

这个数字的单位是微秒，就是说，我们的cpu时间周期是100ms。还有一点需要注意的是，这个时间是针对单核来说的。

那么针对cgroup的限制放在哪里呢？

```
[root@zorrozou-pc ~]# cat /cgroup/cpu/zorro/cpu.cfs_quota_us  
-1
```

就是这个cpu.cfs_quota_us文件。这里的cfs就是完全公平调度器，我们的资源隔离就是靠cfs来实现的。-1表示目前无限制。

限制方法很简单，就是设置cpu.cfs_quota_us这个文件的值，调度器会根据这个值的大小决定进程组在一个时间周期内（即100ms）使用cpu时间的比率。比如这个值我们设置成50000，那么就是时间周期的50%，于是这个进程组只能在一个cpu上占用50%的cpu时间。理解了这个概念，我们就可以思考一下，如果想让我们的进程在24核的服务器上不绑定核心的情况下占用所有核心的50%的cpu时间，该如何设置？计算公式为：

> （50% *100000* cpu核心数）

在此设置为1200000，我们来试一下。修改cgconfig.conf内容，然后重启cgconfig：

```
group zorro {
    cpu {
            cpu.cfs_quota_us = "1200000";
    }
}

[root@zorrozou-pc ~]# service cgconfig restart
```

测试结果如下：

```
[zorro@zorrozou-pc ~/test]$ time ./prime_thread_zorro &> /dev/null

real    0m17.322s
user    3m27.116s
sys    0m0.266s
[zorro@zorrozou-pc ~/test]$ time ./prime_thread_zorro &> /dev/null

real    0m17.347s
user    3m27.208s
sys    0m0.260s

16:15:12     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
16:15:13     all   49.92    0.00    0.08    0.00    0.04    0.00    0.00    0.00    0.00   49.96
16:15:13       0   51.49    0.00    0.00    0.00    0.99    0.00    0.00    0.00    0.00   47.52
16:15:13       1   51.49    0.00    0.99    0.00    0.00    0.00    0.00    0.00    0.00   47.52
16:15:13       2   54.46    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   45.54
16:15:13       3   51.52    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   48.48
16:15:13       4   48.51    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   51.49
16:15:13       5   48.04    0.00    0.98    0.00    0.00    0.00    0.00    0.00    0.00   50.98
16:15:13       6   49.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   51.00
16:15:13       7   49.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   51.00
16:15:13       8   49.49    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.51
16:15:13       9   49.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   51.00
16:15:13      10   48.00    0.00    1.00    0.00    0.00    0.00    0.00    0.00    0.00   51.00
16:15:13      11   49.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   51.00
16:15:13      12   49.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   51.00
16:15:13      13   49.49    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.51
16:15:13      14   49.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   51.00
16:15:13      15   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
16:15:13      16   50.51    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   49.49
16:15:13      17   49.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   51.00
16:15:13      18   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
16:15:13      19   50.50    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   49.50
16:15:13      20   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
16:15:13      21   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
16:15:13      22   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
16:15:13      23   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
```

我们可以看到，基本跟绑定一半的cpu核心数的效果一样，从这个简单的对比来看，使用cpu核心数绑定的方法和使用cpu分配时间的方法，在隔离上效果几乎是相同的。但是考虑到超线程的影响，我们使用cpu时间比率的方式很可能根cpuset的方式有些差别，为了看到这个差别，我们将针对cpuset和cpuquota进行一个对比测试，测试结果如下表：

| cpu比率（核心数） | cpuset realtime | cpuquota realtime |
| ----------------- | --------------- | ----------------- |
| 8.3%(2)           | 1m46.557s       | 1m36.786s         |
| 16.7%(4)          | 0m53.271s       | 0m51.067s         |
| 25%(6)            | 0m35.528s       | 0m34.539s         |
| 33.3%(8)          | 0m26.643s       | 0m25.923s         |
| 50%(12)           | 0m17.839s       | 0m17.347s         |
| 66.7%(16)         | 0m13.384s       | 0m13.015s         |
| 100%(24)          | 0m8.972s        | 0m8.932s          |

> 思考题时间又到了：请解释这个表格测试得到的数字的差异。

我们现在已经学会了如何使用cpuset和cpuquota两种方式对cpu资源进行分配，但是这两种分配的缺点也是显而易见的－－就是分配完之后，进程都最多只能占用相关比例的cpu资源。即使服务器上还有空闲资源，这两种方式都无法将资源“**借来使用**”。

那么有没有一种方法，既可以保证在系统忙的情况下让cgroup进程组只占用相关比例的资源，而在系统闲的情况下，又可以借用别人的资源，以达到资源利用率最大话的程度呢？当然有！那就是－－

### 权重CPU资源隔离

这里的权重其实是shares。我把它叫做权重是因为这个值可以理解为对资源占用的权重。这种资源隔离方式事实上也是对cpu时间的进行分配。区别是作用在cfs调度器的权重值上。从用户的角度看，无非就是给每个cgroup配置一个share值，cpu在进行时间分配的时候，按照share的大小比率来确定cpu时间的百分比。它对比cpuquota的优势是，当进程不在cfs可执行调度队列中的时候，这个权重是不起作用的。就是说，一旦其他cgroup的进程释放cpu的时候，正在占用cpu的进程可以全占所有计算资源。而当有多个cgroup进程都要占用cpu的时候，大家按比例分配。

我们照例通过实验来说明这个情况，配置方法也很简单，修改cgconfig.conf，添加字段，并重启服务：

```
group zorro {
    cpu {
            cpu.shares = 1000;
    }
}

[root@zorrozou-pc ~]# service cgconfig restart
```

配置完之后，我们就给zorro组配置了一个shares值为1000，但是实际上如果系统中只有这一个组的话，cpu看起来对他是没有限制的。现在的执行效果是这样：

```
[root@zorrozou-pc ~]# mpstat -P ALL 1

17:17:29     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
17:17:30     all   99.88    0.00    0.12    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30       0  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30       1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30       2  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30       3   99.01    0.00    0.99    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30       4   99.00    0.00    1.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30       5  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30       6  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30       7  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30       8  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30       9  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30      10  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30      11  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30      12  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30      13  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30      14  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30      15  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30      16  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30      17   99.00    0.00    1.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30      18  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30      19  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30      20  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30      21  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30      22  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:17:30      23  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00

[zorro@zorrozou-pc ~/test]$ time ./prime_thread_zorro &> /dev/null

real    0m8.937s
user    3m32.190s
sys    0m0.225s
```

如显示，cpu我们是独占的。那么什么时候有隔离效果呢？是系统中有别的cgroup也要占用cpu的时候，就能看出效果了。比如此时我们再添加一个jerry，shares值也配置为1000，并且让jerry组一直有占用cpu的进程在运行。

```
group jerry {
    cpu {
            cpu.shares = "1000";
    }
}


top - 17:24:26 up 1 day, 5 min,  2 users,  load average: 41.34, 16.17, 8.17
Tasks: 350 total,   2 running, 348 sleeping,   0 stopped,   0 zombie
Cpu0  :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu1  : 99.7%us,  0.3%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu2  :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu3  : 99.7%us,  0.3%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu4  : 99.7%us,  0.3%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu5  :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu6  : 99.7%us,  0.3%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu7  :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu8  :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu9  : 99.7%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.3%hi,  0.0%si,  0.0%st
Cpu10 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu11 : 99.7%us,  0.3%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu12 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu13 : 99.7%us,  0.3%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu14 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu15 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu16 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu17 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu18 : 99.3%us,  0.7%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu19 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu20 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu21 : 99.7%us,  0.3%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu22 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu23 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Mem:  131904480k total,  4938020k used, 126966460k free,   136140k buffers
Swap:  2088956k total,        0k used,  2088956k free,  3700480k cached

  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND                                                                                                                                               
13945 jerry     20   0  390m  872  392 S 2397.2  0.0  48:42.54 jerry                                               
```

我们以jerry用户身份执行了一个进程一直100%占用cpu，从上面的显示可以看到，这个进程占用了2400%的cpu，是因为每个cpu核心算100%，24个核心就是2400%。此时我们再以zorro身份执行筛质数的程序，并察看这个程序占用cpu的百分比：

```
top - 19:44:11 up 1 day,  2:25,  3 users,  load average: 60.91, 50.92, 48.85
Tasks: 336 total,   3 running, 333 sleeping,   0 stopped,   0 zombie
Cpu0  : 99.7%us,  0.3%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu1  :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu2  : 99.7%us,  0.3%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu3  :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu4  :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu5  :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu6  :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu7  :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu8  :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu9  :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu10 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu11 : 99.7%us,  0.3%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu12 : 99.7%us,  0.3%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu13 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu14 : 99.7%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.3%hi,  0.0%si,  0.0%st
Cpu15 : 99.7%us,  0.3%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu16 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu17 : 99.7%us,  0.3%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu18 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu19 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu20 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu21 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu22 : 99.7%us,  0.3%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu23 :100.0%us,  0.0%sy,  0.0%ni,  0.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Mem:  131904480k total,  1471772k used, 130432708k free,   144216k buffers
Swap:  2088956k total,        0k used,  2088956k free,   322404k cached

  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND                                                                                                                                               
13945 jerry     20   0  390m  872  392 S 1200.3  0.0   3383:04 jerry                                                                                                                                               
 9311 zorro     20   0  390m  872  392 R 1197.0  0.0   0:51.56 prime_thread_zo                        
```

通过top我们可以看到，以zorro用户身份执行的进程和jerry进程平分了cpu，每人50%。zorro筛质数执行的时间为：

```
[zorro@zorrozou-pc ~/test]$ time ./prime_thread_zorro &> /dev/null

real    0m15.152s
user    2m58.637s
sys    0m0.220s
[zorro@zorrozou-pc ~/test]$ time ./prime_thread_zorro &> /dev/null

real    0m15.465s
user    3m0.706s
sys    0m0.221s
```

根据这个时间看起来，基本与通过cpuquota方式分配50%的cpu时间以及通过cpuset方式分配12个核心的情况相当，而且效率还稍微高一些。当然我要说明的是，这里几乎两秒左右的效率的提高并不具备很大的参考性，它与jerry进程执行的运算是有很大相关性的。此时jerry进程执行的是一个多线程的while死循环，占满所有cpu跑。当我们把jerry进程执行的内容同样变成筛质数的时候，zorro用户的进程执行效率的参考值就比较标准了：

```
[zorro@zorrozou-pc ~/test]$ time ./prime_thread_zorro &> /dev/null

real    0m17.521s
user    3m32.684s
sys    0m0.254s
[zorro@zorrozou-pc ~/test]$ time ./prime_thread_zorro &> /dev/null

real    0m17.597s
user    3m32.682s
sys    0m0.253s
```

如程序执行显示，执行效率基本与cpuset和cpuquota相当。

> 这又引发了另一个问题请大家思考：为什么jerry用户执行的运算的逻辑不同会影响zorro用户的运算效率？

我们可以将刚才cpuset和cpuquota的对比列表加入cpushare一列来一起对比了，为了方便参考，我们都以cpuset为基准进行比较：

| shares zorro/shares jerry（核心数） | cpuset realtime | **cpushare realtime** | cpuquota realtime |
| ----------------------------------- | --------------- | --------------------- | ----------------- |
| 2000/22000(2)                       | 1m46.557s       | **1m41.691s**         | 1m36.786s         |
| 4000/20000(4)                       | 0m53.271s       | **0m51.801s**         | 0m51.067s         |
| 6000/18000(6)                       | 0m35.528s       | **0m35.152s**         | 0m34.539s         |
| 8000/16000(8)                       | 0m26.643s       | **0m26.372s**         | 0m25.923s         |
| 12000/12000(12)                     | 0m17.839s       | **0m17.694s**         | 0m17.347s         |
| 16000/8000(16)                      | 0m13.384s       | **0m13.388s**         | 0m13.015s         |
| 24000/0(24)                         | 0m8.972s        | **0m8.943s**          | 0m8.932s          |

请注意一个问题，由于cpushares无法像cpuquota或者cpuset那样只执行zorro用户的进程，所以在进行cpushares测试的时候，必须让jerry用户同时执行相同的筛质数程序，才能使两个用户分别分到相应比例的cpu时间。这样可能造成本轮测试结果的不准确。通过对比看到，当比率分别都配置了相当于两个核心的计算能力的情况下，本轮测试是cpuquota方式消耗了1m36.786s稍快一些。为了保证相对公平的环境作为参照，我们将重新对这轮测试进行数据采集，这次在cpuset和cpuquota的压测时，都用jerry用户执行一个干扰程序作为参照，重新分析数据。当然，cpushares的测试数据就不必重新测试了：

| shares zorro/shares jerry（核心数） | cpuset realtime | cpushare realtime | cpuquota realtime |
| ----------------------------------- | --------------- | ----------------- | ----------------- |
| 2000/22000(2)                       | 1m46.758s       | 1m41.691s         | 1m42.341s         |
| 4000/20000(4)                       | 0m53.340s       | 0m51.801s         | 0m51.512s         |
| 6000/18000(6)                       | 0m35.525s       | 0m35.152s         | 0m34.392s         |
| 8000/16000(8)                       | 0m26.738s       | 0m26.372s         | 0m25.772s         |
| 12000/12000(12)                     | 0m17.793s       | 0m17.694s         | 0m17.256s         |
| 16000/8000(16)                      | 0m13.366s       | 0m13.388s         | 0m13.155s         |
| 24000/0(24)                         | 0m8.930s        | 0m8.943s          | 0m8.939s          |

至此，cgroup中针对cpu的三种资源隔离都介绍完了，分析我们的测试数据可以得出一些结论：

1. 三种cpu资源隔离的效果基本相同，在资源分配比率相同的情况下，它们都提供了差不多相同的计算能力。
2. cpuset隔离方式是以分配核心的方式进行资源隔离，可以提供的资源分配最小粒度是核心，不能提供更细粒度的资源隔离，但是隔离之后运算的相互影响最低。需要注意的是在服务器开启了超线程的情况下，要小心选择分配的核心，否则不同cgroup间的性能差距会比较大。
3. cpuquota给我们提供了一种比cpuset可以更细粒度的分配资源的方式，并且保证了cgroup使用cpu比率的上限，相当于对cpu资源的硬限制。
4. cpushares给我们提供了一种可以按权重比率弹性分配cpu时间资源的手段：当cpu空闲的时候，某一个要占用cpu的cgroup可以完全占用剩余cpu时间，充分利用资源。而当其他cgroup需要占用的时候，每个cgroup都能保证其最低占用时间比率，达到资源隔离的效果。

大家可以根据这三种不同隔离手段特点，针对自己的环境来选择不同的方式进行cpu资源的隔离。当然，这些手段也可以混合使用，以达到更好的QOS效果。

但是可是but，这就完了么？ 显然并没有。。。。。。

以上测试只针对了一种计算场景，这种场景在如此的简单的情况下，影响测试结果的条件已经很复杂了。如果是其他情况呢？我们线上真正跑业务的环境会这么单纯么？显然不会。我们不可能针对所有场景得出结论，想要找到适用于自己场景的隔离方式，还是需要在自己的环境中进行充分测试。在此只能介绍方法，以及针对一个场景的参考数据，仅此而已。单就这一个测试来说，它仍然不够全面，无法体现出内核cpu资源隔离的真正面目。众所周知，cpu使用主要分两个部分，user和sys。上面这个测试，由于测试用例的选择，只关注了user的使用。那么如果我们的sys占用较多会变成什么样呢？

## CPU资源隔离在sys较高的情况下是什么表现？

### 内核资源不冲突的情况

首先我们简单说一下什么叫sys较高。先看mpstat命令的输出：

```
[root@zorrozou-pc ~]# mpstat 1
Linux 3.10.90-1-linux (zorrozou-pc)         12/24/15     _x86_64_    (24 CPU)

16:08:52     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
16:08:53     all    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
16:08:54     all    0.00    0.00    0.04    0.00    0.04    0.00    0.00    0.00    0.00   99.92
16:08:55     all    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
16:08:56     all    0.04    0.00    0.04    0.00    0.00    0.00    0.00    0.00    0.00   99.92
16:08:57     all    0.04    0.00    0.04    0.00    0.00    0.00    0.00    0.00    0.00   99.92
16:08:58     all    0.00    0.00    0.04    0.00    0.00    0.00    0.00    0.00    0.00   99.96

Average:     all    0.01    0.00    0.03    0.00    0.01    0.00    0.00    0.00    0.00   99.95
```

这里面我们看到cpu的使用比率分了很多栏目，我们一般评估进程占用CPU的时候，最重要的是％user和％sys。％sys一般是指，进程陷入内核执行时所占用的时间，这些时间是内核在工作。常见的情况时，进程执行过程中之行了某个系统调用，而陷入内核态执行所产生的cpu占用。

所以在这一部分，我们需要重新提供一个测试用例，让sys部分的cpu占用变高。基于筛质数进行改造即可，我们这次让每个筛质数的线程，在做运算之前都用非阻塞方式open()打开一个文件，每次拿到一个数运算的时候，循环中都用系统调用read()读一下文件。以此来增加sys占用时间的比率。先来改程序：

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <strings.h>
#include <unistd.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <fcntl.h>

#define NUM 48
#define START 1010001
#define END 1020000

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
static int count = 0;

void *prime(void *p)
{
    int n, i, flag;
    int num, fd, ret;
    char name[BUFSIZ];
    char buf[BUFSIZ];

    bzero(name, BUFSIZ);

    num = (int *)p;
    sprintf(name, "/tmp/tmpfilezorro%d", num);

    fd = open(name, O_RDWR|O_CREAT|O_TRUNC|O_NONBLOCK , 0644);
    if (fd < 0) {
        perror("open()");
        exit(1);
    }

    while (1) {
        if (pthread_mutex_lock(&mutex) != 0) {
            perror("pthread_mutex_lock()");
            pthread_exit(NULL);
        }
        while (count == 0) {
            if (pthread_cond_wait(&cond, &mutex) != 0) {
                perror("pthread_cond_wait()");
                pthread_exit(NULL);
            }
        }
        if (count == -1) {
            if (pthread_mutex_unlock(&mutex) != 0) {
                perror("pthread_mutex_unlock()");
                pthread_exit(NULL);
            }
            break;
        }
        n = count;
        count = 0;
        if (pthread_cond_broadcast(&cond) != 0) {
            perror("pthread_cond_broadcast()");
            pthread_exit(NULL);
        }
        if (pthread_mutex_unlock(&mutex) != 0) {
            perror("pthread_mutex_unlock()");
            pthread_exit(NULL);
        }
        flag = 1;
        for (i=2;i<n/2;i++) {
            ret = read(fd, buf, BUFSIZ);
            if (ret < 0) {
                perror("read()");
            }
            if (n%i == 0) {
                flag = 0;
                break;
            }
        }
        if (flag == 1) {
            printf("%d is a prime form %d!\n", n, pthread_self());
        }
    }

    close(fd);
    pthread_exit(NULL);
}

int main(void)
{
    pthread_t tid[NUM];
    int ret, i, num;

    for (i=0;i<NUM;i++) {
        ret = pthread_create(&tid[i], NULL, prime, (void *)i);
        if (ret != 0) {
            perror("pthread_create()");
            exit(1);
        } 
    }

    for (i=START;i<END;i+=2) {
        if (pthread_mutex_lock(&mutex) != 0) {
            perror("pthread_mutex_lock()");
            pthread_exit(NULL);
        }
        while (count != 0) {
            if (pthread_cond_wait(&cond, &mutex) != 0) {
                perror("pthread_cond_wait()");
                pthread_exit(NULL);
            }
        }
        count = i;
        if (pthread_cond_broadcast(&cond) != 0) {
            perror("pthread_cond_broadcast()");
            pthread_exit(NULL);
        }
        if (pthread_mutex_unlock(&mutex) != 0) {
            perror("pthread_mutex_unlock()");
            pthread_exit(NULL);
        }
    }
    if (pthread_mutex_lock(&mutex) != 0) {
        perror("pthread_mutex_lock()");
        pthread_exit(NULL);
    }
    while (count != 0) {
        if (pthread_cond_wait(&cond, &mutex) != 0) {
            perror("pthread_cond_wait()");
            pthread_exit(NULL);
        }
    }
    count = -1;
    if (pthread_cond_broadcast(&cond) != 0) {
        perror("pthread_cond_broadcast()");
        pthread_exit(NULL);
    }
    if (pthread_mutex_unlock(&mutex) != 0) {
        perror("pthread_mutex_unlock()");
        pthread_exit(NULL);
    }

    for (i=0;i<NUM;i++) {
        ret = pthread_join(tid[i], NULL);
        if (ret != 0) {
            perror("pthread_join()");
            exit(1);
        } 
    }

    exit(0);
}
```

我们将筛质数的范围缩小了两个数量级，并且每个线程都打开一个文件，每次计算的循环中都read一遍。此时这个进程执行的时候的cpu使用状态是这样的：

```
17:20:46     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
17:20:47     all   53.04    0.00   46.96    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47       0   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47       1   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47       2   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47       3   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47       4   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47       5   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47       6   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47       7   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47       8   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47       9   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47      10   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47      11   53.47    0.00   46.53    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47      12   52.00    0.00   48.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47      13   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47      14   53.47    0.00   46.53    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47      15   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47      16   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47      17   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47      18   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47      19   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47      20   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47      21   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47      22   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
17:20:47      23   53.00    0.00   47.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00

[zorro@zorrozou-pc ~/test]$ time ./prime_sys &> /dev/null

real    0m12.227s
user    2m34.869s
sys    2m17.239s
```

测试用例已经基本符合我们的测试条件，可以达到近50%的sys占用，下面开始进行对比测试。测试方法根上一轮一样，仍然用jerry账户运行一个相同的程序在另一个cgroup不断的循环，然后分别看在不同资源分配比率下的zorro用户筛质数程序运行的时间。以下是测试结果：

| shares zorro/shares jerry（核心数） | cpuset realtime | cpushare realtime | cpuquota realtime |
| ----------------------------------- | --------------- | ----------------- | ----------------- |
| 2000/22000(2)                       | 2m27.666s       | 2m27.599s         | 2m27.918s         |
| 4000/20000(4)                       | 1m12.621s       | 1m14.345s         | 1m13.581s         |
| 6000/18000(6)                       | 0m48.612s       | 0m49.474s         | 0m48.730s         |
| 8000/16000(8)                       | 0m36.412s       | 0m37.269s         | 0m36.784s         |
| 12000/12000(12)                     | 0m24.611s       | 0m24.624s         | 0m24.628s         |
| 16000/8000(16)                      | 0m18.401s       | 0m18.688s         | 0m18.480s         |
| 24000/0(24)                         | 0m12.188s       | 0m12.487s         | 0m12.147s         |

| shares zorro/shares jerry（核心数） | cpuset systime | cpushare systime | cpuquota systime |
| ----------------------------------- | -------------- | ---------------- | ---------------- |
| 2000/22000(2)                       | 2m20.115s      | 2m21.024s        | 2m21.854s        |
| 4000/20000(4)                       | 2m16.450s      | 2m21.103s        | 2m20.352s        |
| 6000/18000(6)                       | 2m18.273s      | 2m20.455s        | 2m20.039s        |
| 8000/16000(8)                       | 2m18.054s      | 2m20.611s        | 2m19.891s        |
| 12000/12000(12)                     | 2m20.358s      | 2m18.331s        | 2m20.363s        |
| 16000/8000(16)                      | 2m17.724s      | 2m18.958s        | 2m18.637s        |
| 24000/0(24)                         | 2m16.723s      | 2m17.707s        | 2m16.176s        |

这次我们多了一个表格专门记录systime时间占用。根据数据结果我们会发现，在这次测试循环中，三种隔离方式都呈现出随着资源的增加进程是执行的总时间线性下降，并且隔离效果区别不大。由于调用read的次数一样，systime的使用基本都稳定在一个固定的时间范围内。这说明，在sys占用较高的情况下，各种cpu资源隔离手段都表现出比较理想的效果。

### 内核资源冲突的情况

但是现实的生产环境往往并不是这么理想的，有没有可能在某种情况下，各种CPU资源隔离的手段并不会表现出这么理想的效果呢？有没有可能不同的隔离方式会导致进程的执行会有影响呢？其实这是很可能发生的。我们上一轮测试中，每个cgroup中的线程打开的文件都不是同一个文件，内核在处理这种场景的时候，并不需要使用内核中的一些互斥资源(比如自旋锁或者屏障)进行竞争条件的处理。如果环境变成大家read的是同一个文件，那么情况就可能有很大不同了。下面我们来测试一下每个zorro组中的所有线程都open同一个文件并且read时的执行效果，我们照例把测试用例代码贴出来：

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <strings.h>
#include <unistd.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <fcntl.h>

#define NUM 48
#define START 1010001
#define END 1020000

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
static int count = 0;
#define PATH "/etc/passwd"

void *prime(void *p)
{
    int n, i, flag;
    int num, fd, ret;
    char name[BUFSIZ];
    char buf[BUFSIZ];

    fd = open(PATH, O_RDONLY|O_NONBLOCK);
    if (fd < 0) {
        perror("open()");
        exit(1);
    }

    while (1) {
        if (pthread_mutex_lock(&mutex) != 0) {
            perror("pthread_mutex_lock()");
            pthread_exit(NULL);
        }
        while (count == 0) {
            if (pthread_cond_wait(&cond, &mutex) != 0) {
                perror("pthread_cond_wait()");
                pthread_exit(NULL);
            }
        }
        if (count == -1) {
            if (pthread_mutex_unlock(&mutex) != 0) {
                perror("pthread_mutex_unlock()");
                pthread_exit(NULL);
            }
            break;
        }
        n = count;
        count = 0;
        if (pthread_cond_broadcast(&cond) != 0) {
            perror("pthread_cond_broadcast()");
            pthread_exit(NULL);
        }
        if (pthread_mutex_unlock(&mutex) != 0) {
            perror("pthread_mutex_unlock()");
            pthread_exit(NULL);
        }
        flag = 1;
        for (i=2;i<n/2;i++) {
            ret = read(fd, buf, BUFSIZ);
            if (ret < 0) {
                perror("read()");
            }
            if (n%i == 0) {
                flag = 0;
                break;
            }
        }
        if (flag == 1) {
            printf("%d is a prime form %d!\n", n, pthread_self());
        }
    }

    close(fd);
    pthread_exit(NULL);
}

int main(void)
{
    pthread_t tid[NUM];
    int ret, i, num;

    for (i=0;i<NUM;i++) {
        ret = pthread_create(&tid[i], NULL, prime, (void *)i);
        if (ret != 0) {
            perror("pthread_create()");
            exit(1);
        } 
    }

    for (i=START;i<END;i+=2) {
        if (pthread_mutex_lock(&mutex) != 0) {
            perror("pthread_mutex_lock()");
            pthread_exit(NULL);
        }
        while (count != 0) {
            if (pthread_cond_wait(&cond, &mutex) != 0) {
                perror("pthread_cond_wait()");
                pthread_exit(NULL);
            }
        }
        count = i;
        if (pthread_cond_broadcast(&cond) != 0) {
            perror("pthread_cond_broadcast()");
            pthread_exit(NULL);
        }
        if (pthread_mutex_unlock(&mutex) != 0) {
            perror("pthread_mutex_unlock()");
            pthread_exit(NULL);
        }
    }
    if (pthread_mutex_lock(&mutex) != 0) {
        perror("pthread_mutex_lock()");
        pthread_exit(NULL);
    }
    while (count != 0) {
        if (pthread_cond_wait(&cond, &mutex) != 0) {
            perror("pthread_cond_wait()");
            pthread_exit(NULL);
        }
    }
    count = -1;
    if (pthread_cond_broadcast(&cond) != 0) {
        perror("pthread_cond_broadcast()");
        pthread_exit(NULL);
    }
    if (pthread_mutex_unlock(&mutex) != 0) {
        perror("pthread_mutex_unlock()");
        pthread_exit(NULL);
    }

    for (i=0;i<NUM;i++) {
        ret = pthread_join(tid[i], NULL);
        if (ret != 0) {
            perror("pthread_join()");
            exit(1);
        } 
    }

    exit(0);
}
```

此时jerry组中的所有线程仍然是每个线程一个文件，与上一轮测试一样。测试结果如下：

| shares zorro/shares jerry（核心数） | cpuset realtime | cpushare realtime | cpuquota realtime |
| ----------------------------------- | --------------- | ----------------- | ----------------- |
| 2000/22000(2)                       | 2m27.402s       | 2m41.015s         | 4m37.149s         |
| 4000/20000(4)                       | 1m18.178s       | 1m25.214s         | 2m42.455s         |
| 6000/18000(6)                       | 0m52.592s       | 1m2.691s          | 1m48.492s         |
| 8000/16000(8)                       | 0m43.598s       | 0m57.000s         | 1m21.044s         |
| 12000/12000(12)                     | 0m52.182s       | 0m59.613s         | 0m58.004s         |
| 16000/8000(16)                      | 0m50.712s       | 0m54.371s         | 0m56.911s         |
| 24000/0(24)                         | 0m50.599s       | 0m50.550s         | 0m50.496s         |

| shares zorro/shares jerry（核心数） | cpuset systime | cpushare systime | cpuquota systime |
| ----------------------------------- | -------------- | ---------------- | ---------------- |
| 2000/22000(2)                       | 2m19.829s      | 2m47.706s        | 6m39.800s        |
| 4000/20000(4)                       | 2m41.928s      | 3m6.575s         | 8m14.087s        |
| 6000/18000(6)                       | 2m45.671s      | 3m38.722s        | 8m13.668s        |
| 8000/16000(8)                       | 3m14.434s      | 4m54.451s        | 8m12.904s        |
| 12000/12000(12)                     | 7m39.542s      | 9m7.751s         | 8m57.332s        |
| 16000/8000(16)                      | 10m47.425s     | 11m41.443s       | 12m21.056s       |
| 24000/0(24)                         | 17m17.661s     | 17m7.311s        | 17m14.788s       |

观察这轮测试的结果我们会发现，当线程同时read同一个文件时，时间的消耗并不在呈现线性下降的趋势了，而且，随着分配的资源越来越多，sys占用时间也越来越高，这种现象如何解释呢？本质上来讲，使用cgroup进行资源隔离时，内核资源仍然是共享的。如果业务使用内核资源如果没有产生冲突，那么隔离效果应该会比较理想，但是业务一旦使用了会导致内核资源冲突的逻辑时，那么业务的执行效率就会下降，此时可能所有进程在内核中处理的时候都可能会在竞争的资源上忙等（如果使用了spinlock）。自然的，如果多个cgroup的进程之间也正好使用了可能会导致内核触发竞争条件的资源时，自然也会发生所谓的cgroup之间的相互影响。可能的现象就是，当某一个业务A的cgroup正在运行着，突然B业务的cgroup有请求要处理，会导致A业务的响应速度和处理能力下降。而这种相互干扰，正是资源隔离手段想要尽量避免的。我们认为，如果出现了上述效果，那么资源隔离手段就是打了折扣的。

根据我们的实验结果可以推论，在内核资源有竞争条件的情况下，cpuset的资源隔离方式表现出了相对其他方式的优势，cpushare方式的性能折损尚可接受，而cpuquota表现出了最差的性能，或者说在cpuquota的隔离条件下，cgroup之间进程相互影响的可能性最大。

那么在内核资源存在竞争的时候，cgroup的cpu资源隔离会有相互干扰。结论就是这样了么？这个推断靠谱么？我们再来做一轮实验，这次只对比cpuset和cpuquota。这次我们不用jerry来运行干扰程序测试隔离性，我们让zorro只在单纯的隔离状态下，再有内核资源竞争的条件下进行运算效率测试，就是说这个环境没有多个cgroup可能造成的相互影响。先来看数据：

| cpu比率（核心数） | cpuset realtime | cpuquota realtime |
| ----------------- | --------------- | ----------------- |
| 8.3%(2)           | 2m26.815s       | 9m4.490s          |
| 16.7%(4)          | 1m17.894s       | 4m49.167s         |
| 25%(6)            | 0m52.356s       | 3m13.144s         |
| 33.3%(8)          | 0m42.946s       | 2m23.010s         |
| 50%(12)           | 0m52.014s       | 1m33.571s         |
| 66.7%(16)         | 0m50.903s       | 1m10.553s         |
| 100%(24)          | 0m50.331s       | 0m50.304s         |

| cpu比率（核心数） | cpuset systime | cpuquota systime |
| ----------------- | -------------- | ---------------- |
| 8.3%(2)           | 2m18.713s      | 15m27.738s       |
| 16.7%(4)          | 2m41.172s      | 16m30.741s       |
| 25%(6)            | 2m44.618s      | 16m30.964s       |
| 33.3%(8)          | 3m12.587s      | 16m18.366s       |
| 50%(12)           | 7m36.929s      | 15m55.407s       |
| 66.7%(16)         | 10m49.327s     | 16m1.463s        |
| 100%(24)          | 17m9.482s      | 17m9.533s        |

不知道看完这组数据之后，大家会不会困惑？cpuset的测试结果根上一轮基本一样，这可以理解。但是为什么cpuquota这轮测试反倒比刚才有jerry用户进程占用cpu进行干扰的时候的性能更差了？

如果了解了内核在这种资源竞争条件的原理的话，这个现象并不难解释。可以这样想，如果某一个资源存在竞争的话，那么是不是同时竞争的人越多，那么对于每个人来说，单次得到资源的可能性更低？比如说，老师给学生发苹果，每次只发一个，但是同时有10个人一起抢，每个人每次抢到苹果的几率是10%，如果20个人一起抢，那么每次每人强到苹果的几率就只有5％了。在内核竞争条件下，也是一样的道理，资源只有一个，当抢的进程少的时候，每个进程抢到资源的概率大，于是浪费在忙等上的时间就少。本轮测试的cpuset就可以说明这个现象，可以观察到，cpuset systime随着分配的核心数的增多而上升，就是同时跑的进程越多，sys消耗在忙等资源上的时间就越大。而cpuquota systime消耗从头到尾都基本变化不大，意味着再以quota方式分配cpu的时候，所有核心都是用得上的，所以一直都有24个进程在抢资源，大家消耗在忙等上的时间是一样的。 为什么有jerry进程同时占用cpu的情况下，cpuquota反倒效率要快些呢？这个其实也好理解。在jerry进程执行的时候，这个cgroup的相关线程打开的是不同的文件，所以从内核竞争上没有冲突。另外，jerry消耗了部分cpu，导致内核会在zorro的进程和jerry的进程之间发生调度，这意味着，同一时刻核心数只有24个，可能有18个在给jerry的线程使用，另外6个在给zorro的进程使用，这导致zorro同时争抢资源的进程个数不能始终保持24个，所以内核资源冲突反倒减小了。`这导致，使用cpuquota的情况下，有其他cgroup执行的时候，还可能会使某些业务的执行效率提升，而不是下降。`这种相互影响实在太让人意外了！但这确实是事实！

那么什么情况下会导致cgroup之间的相互影响使性能下降呢？也好理解，当多个cgroup的应用之间使用了相同的内核资源的时候。请大家思考一个问题：现实情况是同一种业务使用冲突资源的可能性更大还是不同业务使用冲突资源的可能性更大呢？从概率上说应该是同一种业务。从这个角度出发来看，如果我们有两台多核服务器，有两个跟我们测试逻辑类似的业务A、B，让你选择一种部署方案，你是选择让A、B两个业务分别独占一个服务器？还是让A、B业务使用资源隔离分别在两个服务器上占用50%的资源？通过这轮分析我想答案很明确了：

1. 从容灾的角度说，让某一个业务使用多台服务器肯定会增加容灾能力。
2. 从资源利用率的角度说，如果让一个业务放在一个服务器上，那么他在某些资源冲突的情况下并不能发挥会最大效率。然而如果使用group分布在两个不同的服务器上，无论你用cpuset，还是cpushare，又或是cpuquota，它的cpu性能表现都应该强于在一个独立的服务器上部署。况且cgroup的cpu隔离是在cfs中实现的，这种隔离几乎是不会浪费额外的计算能力的，就是说，做隔离相比不做隔离，系统本身的性能损耗都可以忽略不计。

那么，究竟还有什么会妨碍我们使用cgoup的cpu资源隔离呢？

> 原文链接：https://zorro.gitbooks.io/poor-zorro-s-linux-book/content/cgroup_linux_cpu_control_group.html
