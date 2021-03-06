# ð° æ·±æ Linux åæ ¸çæ°åè½ç¹æ§ï¼ä»¥ io_uring, cgroup, ebpf, llvm ä¸ºä»£è¡¨ï¼åå«å¼æºé¡¹ç®ï¼ä»£ç æ¡ä¾ï¼æç« ï¼è§é¢ï¼æ¶æèå¾ç­

æææ°æ®æ¥æºäºäºèç½ãæè°åä¹äºäºèç½ï¼ç¨ä¹äºäºèç½ã

å¦ææ¶åçæä¾µç¯ï¼è¯·é®ä»¶è³ wchao_isvip@163.com ï¼æä»¬å°ç¬¬ä¸æ¶é´å¤çã

å¦ææ¨å¯¹æä»¬çé¡¹ç®è¡¨ç¤ºèµåä¸æ¯æï¼æ¬¢è¿æ¨ lssuesæä»¬ï¼æèé®ä»¶ wchao_isvip@163.com æä»¬ï¼æ´å æ¬¢è¿æ¨ pull requests å å¥æä»¬ã

æè°¢æ¨çæ¯æï¼


## ð¥ [io_uring](https://en.wikipedia.org/wiki/Io_uring) 

<div  align=center>
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/149773115-12090153-72dc-4d48-ab2a-fbb39a0d4503.png"/>
  
#### ââ 2019 å¹´ Linux 5.1 åæ ¸é¦æ¬¡å¼å¥çé«æ§è½ å¼æ­¥ I/O æ¡æ¶ï¼è½æ¾èå é I/O å¯éååºç¨çæ§è½ã

</div>

### ææ¡£
- å®æ¹ææ¡£: [Efficient I/O with io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring.pdf)
- å¶ä»ææ¡£ï¼
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
  - [io_uring-å¾æµ©-é¿éäº](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E6%A1%A3/io_uring-%E5%BE%90%E6%B5%A9-%E9%98%BF%E9%87%8C%E4%BA%91.pdf)

### å¼æºé¡¹ç® 
- [axboe/liburing](https://github.com/axboe/liburing): io_uring åºï¼liburingä¸ºè®¾ç½®åææ io_uring å®ä¾ï¼è¿æä¸ä¸ªç®åæ¥å£ä¸éè¦ï¼æä¸æ³ï¼å¤çå®æ´åæ ¸çåºç¨ç¨åºè¾¹æ§è¡ã
- [shuveb/io_uring-by-example](https://github.com/shuveb/io_uring-by-example): ä¸ä¸ªio_uring ç¤ºä¾çåº
- [bytedance/monoio](https://github.com/bytedance/monoio): åºäºio-uringçRustå¼æ­¥è¿è¡æ¶
- [spacejam/rio](https://github.com/spacejam/rio): Rust io_uringåºï¼æå»ºå¨libcä¸ï¼çº¿ç¨åå¼æ­¥åå¥½ï¼æè¯¯ç¨
- [Iceber/iouring-go](https://github.com/Iceber/iouring-go): æä¾æäºä½¿ç¨çå¼æ­¥IOæ¥å£io_uring
- [frevib/io_uring-echo-server](https://github.com/frevib/io_uring-echo-server): io_uring echo server
- [hodgesds/iouring-go](https://github.com/hodgesds/iouring-go): Io_uringæ¯ægo
- [dshulyak/uring](https://github.com/dshulyak/uring): ç¨äºio_uringæ¡æ¶çGolangåº(æ CGO)
- [quininer/ritsu](https://github.com/quininer/ritsu): ä¸ä¸ªå®éªæ§çåºäºio-uringçå¼æ­¥è¿è¡æ¶ã
- [shuveb/loti-examples](https://github.com/shuveb/loti-examples): æºä»£ç ç¤ºä¾ç¨åºï¼ä»ä¸»çio_uringæå
- [xuanyi-fu/xynet](https://github.com/xuanyi-fu/xynet): åºäºio_uringåc++ 20åç¨çç½ç»åº
- [KuiBaDB/kbio](https://github.com/KuiBaDB/kbio): ä¸ä¸ªåºäºio_uringçå¼æ­¥IOæ¡æ¶
- [shuveb/loti](https://github.com/shuveb/loti): io_uringæç¨ï¼ä¾å­ååè
- [MarkReedZ/mrloop](https://github.com/MarkReedZ/mrloop): Cè¯­è¨ä½¿ç¨io_uringçäºä»¶å¾ªç¯
- [tchaloupka/during](https://github.com/tchaloupka/during): dlang io_uringåè£
- [omegacoleman/arkio](https://github.com/omegacoleman/arkio): åºäºå¼æ­¥IOçåæ ¸IOåº
- [ciconia/awesome-io_uring](https://github.com/ciconia/awesome-io_uring): ä¸ä¸ªå¾æ£çio_uringèµæºãåºåå·¥å·çåç±»éåã
- [ddeka0/AsyncIO](https://github.com/ddeka0/AsyncIO): ä¸ä¸ªç¨äºå¼æ­¥å¥æ¥å­æå¡å¨çCPPåè£å¨ï¼ä½¿ç¨linuxææ°çio_uring API
- [uroni/fuseuring](https://github.com/uroni/fuseuring): ä½¿ç¨io_uringå®ç°ä¸ä¸ªç¨æ·ç©ºé´Linux fuseæå¡å¨
- [yunwei37/co-uring-WebServer](https://github.com/yunwei37/co-uring-WebServer): ä¸ä¸ªä½¿ç¨io_uringåcpp20ååç¨åºçc++é«æ§è½Webæå¡å¨
- [romange/helio](https://github.com/romange/helio): ä¸ä¸ªåºäºio_uring Linuxæ¥å£çç°ä»£åç«¯å¼åæ¡æ¶
- [3541/short-circuit](https://github.com/3541/short-circuit): Linuxé«æ§è½webæå¡å¨ï¼åºäºio_uringæå»ºã
- [anolis-os-archive/perf-test-for-io_uring](https://github.com/anolis-os-archive/perf-test-for-io_uring): ä¸ä¸ªç¨äºio_uringæ§è½æµè¯çæ¡æ¶ã
- [BlazeWasHere/Cnidus](https://github.com/BlazeWasHere/Cnidus): åºäºio_uringçCè¯­è¨webæ¡æ¶ã
- [AnSpake/osiris](https://github.com/AnSpake/osiris): ä¸ä¸ªç®åçæå¡å¨/å®¢æ·ç«¯ï¼ä½¿ç¨io_uring

### æç« 

- [io_uring é«æ IO](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring%20%E9%AB%98%E6%95%88%20IO.md)
- [ [è¯] Linux å¼æ­¥ I_O æ¡æ¶ io_uringï¼åºæ¬åçãç¨åºç¤ºä¾ä¸æ§è½åæµï¼2020ï¼](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/%5B%E8%AF%91%5D%20Linux%20%E5%BC%82%E6%AD%A5%20I_O%20%E6%A1%86%E6%9E%B6%20io_uring%EF%BC%9A%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86%E3%80%81%E7%A8%8B%E5%BA%8F%E7%A4%BA%E4%BE%8B%E4%B8%8E%E6%80%A7%E8%83%BD%E5%8E%8B%E6%B5%8B%EF%BC%882020%EF%BC%89.md)
- [æµæå¼æºé¡¹ç®ä¹io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/%E6%B5%85%E6%9E%90%E5%BC%80%E6%BA%90%E9%A1%B9%E7%9B%AE%E4%B9%8Bio_uring.md)
- [io_uring ç³»ç»æ§æ´ç](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring%20%E7%B3%BB%E7%BB%9F%E6%80%A7%E6%95%B4%E7%90%86.md)
- [io_uringï¼1ï¼ â æä»¬ä¸ºä»ä¹ä¼éè¦ io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring%EF%BC%881%EF%BC%89%20%E2%80%93%20%E6%88%91%E4%BB%AC%E4%B8%BA%E4%BB%80%E4%B9%88%E4%BC%9A%E9%9C%80%E8%A6%81%20io_uring.md)
- [io_uringï¼2ï¼- ä»åå»ºå¿è¦çæä»¶æè¿°ç¬¦ fd å¼å§](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring%EF%BC%882%EF%BC%89-%20%E4%BB%8E%E5%88%9B%E5%BB%BA%E5%BF%85%E8%A6%81%E7%9A%84%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6%20fd%20%E5%BC%80%E5%A7%8B.md)
- [ä¸ä¸ä»£å¼æ­¥ IO io_uring ææ¯è§£å¯](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/%E4%B8%8B%E4%B8%80%E4%BB%A3%E5%BC%82%E6%AD%A5%20IO%20io_uring%20%E6%8A%80%E6%9C%AF%E8%A7%A3%E5%AF%86.md)
- [å°è°io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/%E5%B0%8F%E8%B0%88io_uring.md)
- [æºæ±åäº-æ°æ¶ä»£IOå©å¨-io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/%E6%99%BA%E6%B1%87%E5%8D%8E%E4%BA%91-%E6%96%B0%E6%97%B6%E4%BB%A3IO%E5%88%A9%E5%99%A8-io_uring.md)
- [Linux 5.1 ç io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/Linux%205.1%20%E7%9A%84%20io_uring.md)
- [What is io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/What%20is%20io_uring)
- [io_uring_setup](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring_setup.md)
- [io_uring_enter](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring_enter.md)
- [io_uring_register](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/io_uring_register.md)
- [The Low-level io_uring Interface](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/The%20Low-level%20io_uring%20Interface.md)
- [Submission Queue Polling](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/Submission%20Queue%20Polling.md)
- [Efficient IO with io_uring](https://github.com/0voice/kernel_new_features/blob/main/io_uring/%E6%96%87%E7%AB%A0/Efficient%20IO%20with%20io_uring.md)

### è§é¢(æåç ï¼1024)

- [Speeding Up VMâs I_O Sharing Host's io_uring Queues With Guests by Stefano Garzarellaã2020ã](https://pan.baidu.com/s/1eQC_OQhfBnkd8t6NbBnseQ)
- [Asynchronous I_O and coroutines for smooth data streaming - BjÃ¶rn Fahller - NDC TechTown 2021](https://pan.baidu.com/s/1l5ZEOIwRKwWbnhZPnsj4hQ)
- [Guilherme Bernal - Reaching 200k req_s on a single core with io_uring - Crystal 1.0 Conference](https://pan.baidu.com/s/1EzFLmdpq9hEGhTsxhSF5NA)
- [Improved Storage Performance Using the New Linux Kernel I O Interface (SDC 2019)](https://pan.baidu.com/s/19vzNrSVAbjXP_XC5eNxj8g)
- [io_uring- BPF controlled I_O - Pavel Begunkov](https://pan.baidu.com/s/1g5KLbY9nQ2FIQkN7a3MGDw)
- [io_uring in QEMU- high-performance disk I_O for Linux](https://pan.baidu.com/s/1VFOdf6H6rRp3o2EHPmjLXA)
- [Kernel Recipes 2019 - Faster IO through io_uring](https://pan.baidu.com/s/1z7sFE2oFDcS6DAbod4UyOQ)
- [SDC2021- Samba Multi-Channel_io_uring Status Update](https://pan.baidu.com/s/1-YlabCqs03LS7nJxaOqPKQ)
- [Speeding Up VMâs I_O Sharing Host's io_uring Queues With Guests - Stefano Garzarella, Red Hat](https://pan.baidu.com/s/1QW3zvykzFwYKsMZUZK7orA)
- [USENIX ATC '19 - Asynchronous I_O Stack_ A Low-latency Kernel I_O Stack for Ultra-Low Latency SSDs](https://pan.baidu.com/s/1sWdfkSU9yjoY53A4wvkcfQ)
- [æ¥èªé¿éäºç Linux åæ ¸ io_uring ä»ç»ä¸å®è·µ](https://pan.baidu.com/s/1FykA5evNh3O3JK4Cu9fs0Q)

## ð¥ [cgroup](https://zh.wikipedia.org/wiki/Cgroups)

<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150078568-4f0de590-793f-41b9-9038-cc8b44894cfb.png"/>
  
#### ââ éå¶ãæ§å¶ä¸åç¦»ä¸ä¸ªè¿ç¨ç»çèµæºï¼å¦CPUãåå­ãç£çè¾å¥è¾åºç­ï¼ã

</div>

### ææ¡£
- å®æ¹ææ¡£:
  - [Control Groups definition, implementation details, examples and API](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/cgroups.txt)
  - [CPU Accounting Controller; account CPU usage for groups of tasks](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/cpuacct.txt)
  - [documents the cpusets feature; assign CPUs and Mem to a set of tasks](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/cpusets.txt)
  - [Device Whitelist Controller; description, interface and security](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/devices.txt)
  - [checkpointing; rationale to not use signals, interface](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/freezer-subsystem.txt)
  - [Memory Resource Controller; implementation details](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/memcg_test.txt)
  - [Memory Resource Controller; design, accounting, interface, testing](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/memory.txt)
  - [Resource Counter API](https://web.archive.org/web/20120618145303/http://www.kernel.org/doc/Documentation/cgroups/resource_counter.txt)

- å¶ä»ææ¡£ï¼
  - [cgroupsä»ç»](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/cgroups%E4%BB%8B%E7%BB%8D.pdf)
  - [CgroupMemcgMaster](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/CgroupMemcgMaster.pdf)
  - [Resource Management](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/Resource%20Management.pdf)
  - [Challenges with the memory resource controller and its performance](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/%20Challenges%20with%20the%20memory%20resource%20controller%20and%20its%20performance.pdf)
  - [Ressource Management in Linux with Control Groups](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/Ressource%20Management%20in%20Linux%20with%20Control%20Groups.pdf)
  - [System Programming for Linux Containers Control Groups (cgroups)](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/System%20Programming%20for%20Linux%20Containers%20Control%20Groups%20(cgroups).pdf)
  - [Managing Resources with cgroups](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/Managing%20Resources%20with%20cgroups.pdf)
  - [5 years of cgroup v2](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/5%20years%20of%20cgroup%20v2.pdf)
  - [Linuxâs new unified control group system](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/%20Linux%E2%80%99s%20new%20unified%20control%20group%20system.pdf)
  - [cgroups_intro](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/cgroups_intro.pdf)
  - [red_hat_enterprise_linux-6-resource_management_guide-en-us](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/red_hat_enterprise_linux-6-resource_management_guide-en-us.pdf)
  - [An introduction to Control Groups (cgroups)](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/An%20introduction%20to%20Control%20Groups%20(cgroups).pdf)
  - [Using Linux Control Groups and Systemd to Manage CPU Time and Memory](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/%20Using%20Linux%20Control%20Groups%20and%20Systemd%20to%20Manage%20CPU%20Time%20and%20Memory.pdf)
  - [An introduction to cgroups and cgroupspy](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E6%A1%A3/An%20introduction%20to%20cgroups%20and%20cgroupspy.pdf)


### å¼æºé¡¹ç® 
- [containerd/cgroups](https://github.com/containerd/cgroups): ç¨äºåå»ºãç®¡çãæ£æ¥åéæ¯cgroupãcgroupä¸è®¾ç½®çèµæºæ ¼å¼ä½¿ç¨è¿éæ¾å°çOCIè¿è¡æ¶è§èã
- [mhausenblas/cinf](https://github.com/mhausenblas/cinf): ä¸ä¸ªæ¥çå½åç©ºé´åcgroupsçå½ä»¤è¡å·¥å·
- [flouthoc/vas-quod](https://github.com/flouthoc/vas-quod): ç¨Rustç¼åçä¸ä¸ªæå°çå®¹å¨è¿è¡æ¶
- [poelzi/ulatencyd](https://github.com/poelzi/ulatencyd): ä½¿ç¨cgroupsæå°ålinuxç³»ç»å»¶è¿çå®æ¤è¿ç¨
- [haosdent/jcgroup](https://github.com/haosdent/jcgroup): jcgroupæ¯JVMä¸çcgroupåè£å¨ãæ¨å¯ä»¥ä½¿ç¨è¿ä¸ªåºæ¥éå¶çº¿ç¨çCPUå±äº«ãç£çI/Oéåº¦ãç½ç»å¸¦å®½ç­ã
- [kinvolk/traceloop](https://github.com/kinvolk/traceloop): ä½¿ç¨BPFåå¯éåçç¯å½¢ç¼å²åºè·è¸ªcgroupä¸­çç³»ç»è°ç¨
- [tianon/cgroupfs-mount](https://github.com/tianon/cgroupfs-mount): æè½½cgroupfs (v1)å±æ¬¡ç»æçç®å(è¿æ¶)èæ¬ï¼ç¹å«æ¯ç¨äºDebianæåçç»æåèæ¬
- [francisbouvier/cgroups](https://github.com/francisbouvier/cgroups): ä¸ä¸ªåºæ¥ç®¡çcgroups Linuxåæ ¸ç¹æ§
- [bpowers/mstat](https://github.com/bpowers/mstat): è¿ä¸ªå·¥å·è¿è¡å¨Linuxä¸ï¼å©ç¨cgroupsåæ ¸API(ä¹è¢«Dockerç­å®¹å¨åºç¡è®¾æ½ä½¿ç¨)æ¥è®°å½ä¸ç»è¿ç¨éæ¶é´çåå­ä½¿ç¨æåµã




### æç« 

- [Linux cgroups æ¦è¿°](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/linux%20cgroups%20%E6%A6%82%E8%BF%B0.md)
- [ãè¯ãControl Group v2ï¼cgroupv2 æå¨æåï¼ï¼KernelDoc, 2021ï¼](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/%5B%E8%AF%91%5D%20Control%20Group%20v2%EF%BC%88cgroupv2%20%E6%9D%83%E5%A8%81%E6%8C%87%E5%8D%97%EF%BC%89%EF%BC%88KernelDoc%2C%202021%EF%BC%89.md)
- [How I Used CGroups to Manage System Resources](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/How%20I%20Used%20CGroups%20to%20Manage%20System%20Resources.md)
- [Cgroupsæ§å¶cpuï¼åå­ï¼ioç¤ºä¾](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/Cgroups%E6%8E%A7%E5%88%B6cpu%EF%BC%8C%E5%86%85%E5%AD%98%EF%BC%8Cio%E7%A4%BA%E4%BE%8B.md)
- [Linux Control Groups V1 å V2 åçååºå«](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/Linux%20Control%20Groups%20V1%20%E5%92%8C%20V2%20%E5%8E%9F%E7%90%86%E5%92%8C%E5%8C%BA%E5%88%AB.md)
- [Linuxèµæºç®¡çä¹cgroupsç®ä»](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/Linux%E8%B5%84%E6%BA%90%E7%AE%A1%E7%90%86%E4%B9%8Bcgroups%E7%AE%80%E4%BB%8B.md)
- [å½»åºææå®¹å¨ææ¯çåºç³ï¼ cgroup](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/%E5%BD%BB%E5%BA%95%E6%90%9E%E6%87%82%E5%AE%B9%E5%99%A8%E6%8A%80%E6%9C%AF%E7%9A%84%E5%9F%BA%E7%9F%B3%EF%BC%9A%20cgroup.md)
- [æ·±å¥çè§£ Linux Cgroup ç³»åï¼ä¸ï¼ï¼åºæ¬æ¦å¿µ](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%20Linux%20Cgroup%20%E7%B3%BB%E5%88%97%EF%BC%88%E4%B8%80%EF%BC%89%EF%BC%9A%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5.md)
- [æ·±å¥çè§£ Linux Cgroup ç³»åï¼äºï¼ï¼ç©è½¬ CPU](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%20Linux%20Cgroup%20%E7%B3%BB%E5%88%97%EF%BC%88%E4%BA%8C%EF%BC%89%EF%BC%9A%E7%8E%A9%E8%BD%AC%20CPU.md)
- [æ·±å¥çè§£ Linux Cgroup ç³»åï¼ä¸ï¼ï¼åå­](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%20Linux%20Cgroup%20%E7%B3%BB%E5%88%97%EF%BC%88%E4%B8%89%EF%BC%89%EF%BC%9A%E5%86%85%E5%AD%98.md)
- [Cgroup - ä»CPUèµæºéç¦»è¯´èµ·](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/Cgroup%20-%20%E4%BB%8ECPU%E8%B5%84%E6%BA%90%E9%9A%94%E7%A6%BB%E8%AF%B4%E8%B5%B7.md)
- [Cgroup - Linuxåå­èµæºç®¡ç](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/Cgroup%20-%20Linux%E5%86%85%E5%AD%98%E8%B5%84%E6%BA%90%E7%AE%A1%E7%90%86.md)
- [Cgroup - LinuxçIOèµæºéç¦»](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/Cgroup%20-%20Linux%E7%9A%84IO%E8%B5%84%E6%BA%90%E9%9A%94%E7%A6%BB.md)
- [Cgroup - Linuxçç½ç»èµæºéç¦»](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/Cgroup%20-%20Linux%E7%9A%84%E7%BD%91%E7%BB%9C%E8%B5%84%E6%BA%90%E9%9A%94%E7%A6%BB.md)
- [ç¨ cgroups ç®¡ç cpu èµæº](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/%E7%94%A8%20cgroups%20%E7%AE%A1%E7%90%86%20cpu%20%E8%B5%84%E6%BA%90.md)
- [ç¨ cgruops ç®¡çè¿ç¨åå­å ç¨](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/%E7%94%A8%20cgruops%20%E7%AE%A1%E7%90%86%E8%BF%9B%E7%A8%8B%E5%86%85%E5%AD%98%E5%8D%A0%E7%94%A8.md)
- [ç¨ cgroups ç®¡çè¿ç¨ç£ç io](https://github.com/0voice/kernel_new_features/blob/main/cgroups/%E6%96%87%E7%AB%A0/cgroups%20%E7%AE%A1%E7%90%86%E8%BF%9B%E7%A8%8B%E7%A3%81%E7%9B%98%20io.md)


### è§é¢

- [Containers_ cgroups, Linux kernel namespaces, ufs, Docker, and intro to Kubernetes pods](https://pan.baidu.com/s/1dfmOxOESpgT9rj4VH1BmRA)---æåç : k4hn
- [Understanding and Working with the Cgroups Interface - Michael Anderson, The PTR Group, LLC](https://pan.baidu.com/s/1wD5MRvHheJv1P8i1iQmosQ)---æåç : 54vs
- [Linux Container Primitives- cgroups, namespaces, and more!](https://pan.baidu.com/s/1LZ9Ff1EuTArxcv6e0c8-2A)---æåç : cjwd
- [Cgroups, namespaces, and beyond](https://pan.baidu.com/s/1IjOURq5X6TEwZn6G5LUhog)---æåç : at6x 
- [Kubernetes On Cgroup v2 - Giuseppe Scrivano, Red Hat](https://pan.baidu.com/s/1apHDcsiCpiZITd_TwfezCg)---æåç : 552y
- [Cgroup Slab Memory Controller and Time Namespace - DevConf.CZ 2021](https://pan.baidu.com/s/1qhVtHJtQjM-7mJVQVDMPwg)---æåç : gayh
- [Modern Linux Servers with cgroups - Brandon Philips, CoreOS](https://pan.baidu.com/s/1okbzLkfA7d0uKJRyj3iyDg)---æåç : afm1
- [LISA21 - 5 Years of Cgroup v2- The Future of Linux Resource Control](https://pan.baidu.com/s/1AGo7vUC0F0uKO5gCd4wVxg)---æåç : ygrv
- [Limit CPU usage on Ubuntu with Systemd cgroups](https://pan.baidu.com/s/17gB4Lv4LyznfMwTxd9Ae_Q)---æåç : ktva
- [What's new in control groups (cgroups) version 2](https://pan.baidu.com/s/1r3V4Htltuy58OUmXGC5aXQ)---æåç : w2tz

## ð¥ [ebpf](https://ebpf.io/)

<div  align=center>
  
<img width="60%" height="60%" src="https://ebpf.io/static/logo-big-9cf8920e80cdc57e6ea60825ebe287ca.png"/>
  
#### ââ Linux åæ ¸ä¸­é¡¶çº§å­æ¨¡å
</div>

### ææ¡£
- å®æ¹ææ¡£: 
  - Linux åæ ¸ï¼https://www.kernel.org/doc/Documentation/networking/filter.txt and https://www.kernel.org/doc/html/latest/bpf/#
  - å¼åQA: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/bpf/bpf_devel_QA.rst
  - eBPF-Helpersï¼https://github.com/iovisor/bpf-docs/blob/master/bpf_helpers.rst/
- å¶ä»ææ¡£ï¼

  - [iovisor/bpf-docs](https://github.com/iovisor/bpf-docs): ååºäº eBPF opcodeï¼é¡¹ç®æ¯ iovisor æ»ç»çç³»åææ¡£ãpreã
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
  - [eBPF - From a Programmerâs Perspective](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/eBPF%20-%20From%20a%20Programmer%E2%80%99s%20Perspective.pdf)
  - [eBPF In-kernel Virtual Machine & Cloud Computin](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/eBPF%20In-kernel%20Virtual%20Machine%20%26%20Cloud%20Computin.pdf)
  - [eBPF for perfomance analysis and networking](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/eBPF%20for%20perfomance%20analysis%20and%20networking.pdf)
  - [eBPF in CPU Scheduler](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/eBPF%20in%20CPU%20Scheduler.pdf)
  - [eBPF-based Content and Computation-aware Communication for Real-time Edge Computing](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E6%A1%A3/eBPF-based%20Content%20and%20Computation-aware%20Communication%20for%20Real-time%20Edge%20Computing.pdf)

### å¼æºé¡¹ç® 

- [cilium/cilium](https://github.com/cilium/cilium): ç¨äºæä¾ãä¿æ¤åè§å¯å®¹å¨å·¥ä½è´è½½ä¹é´çç½ç»è¿æ¥ââäºåçï¼å¹¶ç±é©å½æ§çåæ ¸ææ¯eBPFæä¾æ¯æ,https://cilium.io/
- [BPF Compiler Collection (BCC)](https://github.com/iovisor/bcc): BCC -åºäºbpfçLinux IOåæãèç½ãçæ§ç­å·¥å·
- [bpftrace](https://github.com/iovisor/bpftrace): Linux eBPFçé«çº§è·è¸ªè¯­è¨
- [Falco](https://github.com/falcosecurity/falco): ä¸ç§è¡ä¸ºæ´»å¨çè§å¨ï¼æ¨å¨æ£æµåºç¨ç¨åºä¸­çå¼å¸¸æ´»å¨ãFalcoå¨ebpçå¸®å©ä¸å¨Linuxåæ ¸å±å¯¹ç³»ç»è¿è¡å®¡è®¡ãå®éè¿å¶ä»è¾å¥æµ(å¦å®¹å¨è¿è¡æ¶åº¦éåKubernetesåº¦é)ä¸°å¯äºæ¶éå°çæ°æ®ï¼å¹¶åè®¸æç»­çè§åæ£æµå®¹å¨ãåºç¨ç¨åºãä¸»æºåç½ç»æ´»å¨ã
- [Katran](https://github.com/facebookincubator/katran): é«æ§è½çåå±è´è½½åè¡¡å¨
- [LLVM Compiler](https://github.com/llvm/llvm-project/): ä¸ä¸ªæ¨¡åååå¯éç¨çç¼è¯å¨åå·¥å·é¾ææ¯çéåã
- [microsoft/ebpf-for-windows](https://github.com/microsoft/ebpf-for-windows): è¿è¡å¨Windowsä¸çeBPFå®ç°
- [aquasecurity/libbpfgo](https://github.com/aquasecurity/libbpfgo): ä¸ä¸ªç¨äºLinux ebbpfé¡¹ç®çGoåºã
- [aquasecurity/tracee](https://github.com/aquasecurity/tracee): Linuxçè¿è¡æ¶å®å¨ååè¯å·¥å·ã
- [libbpf/libbpf](https://github.com/libbpf/libbpf): libbpfæ¯ä¸ä¸ªåºäºC/ c++çåºï¼ä½ä¸ºä¸æ¸¸Linuxåæ ¸çä¸é¨åè¿è¡ç»´æ¤ãå®åå«ä¸ä¸ªeBPFå è½½å¨ï¼å®æ¥ç®¡å¤çLLVMçæçeBPF ELFæä»¶ï¼ä»¥ä¾¿å°å¶å è½½å°åæ ¸ä¸­ã
- [libbpf/libbpf-rs](https://github.com/libbpf/libbpf-rs): Rustçæç³»ç»çæå°ååºæ§çepfå·¥å·
- [foniod/redbpf](https://github.com/foniod/redbpf): Ruståºç¨äºæå»ºåè¿è¡BPF/eBPFæ¨¡å
- [aya-rs/aya](https://github.com/aya-rs/aya): ä¸ä¸ªç¨äºRustç¼ç¨è¯­è¨çeBPFåºï¼å¶æå»ºçéç¹æ¯å¼åäººåçä½éªåå¯æä½æ§ã
- [cilium/hubble](https://github.com/cilium/hubble): ä½¿ç¨eBPFçKubernetesç½ç»ãæå¡åå®å¨å¯è§æµæ§
- [kubearmor/KubeArmor](https://github.com/kubearmor/KubeArmor): ä¸ä¸ªäºæ¬å°è¿è¡æ¶å®å¨å¼ºå¶ç³»ç»ï¼å®å¨ç³»ç»çº§å«éå¶å®¹å¨åèç¹çè¡ä¸º(å¦è¿ç¨æ§è¡ãæä»¶è®¿é®åç½ç»æä½)ã
- [iovisor/kubectl-trace](https://github.com/iovisor/kubectl-trace): ä½¿ç¨kubectlå¨kuberneteséç¾¤ä¸è°åº¦bpftraceç¨åº
- [iovisor/ply](https://github.com/iovisor/ply): ä¸æ¬¾åºäºeBPFçLinuxå¨æè·è¸ªè½¯ä»¶ã


### æç« 

- [ä»ä¹æ¯ eBPF](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/%E4%BB%80%E4%B9%88%E6%98%AF%20eBPF.md)
- [eBPFè¯¦è§£](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%E8%AF%A6%E8%A7%A3.md)
- [BPF å eBPF åæ¢](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/BPF%20%E5%92%8C%20eBPF%20%E5%88%9D%E6%8E%A2.md)
- [Linux åæ ¸çæµææ¯ eBPF](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/Linux%20%E5%86%85%E6%A0%B8%E7%9B%91%E6%B5%8B%E6%8A%80%E6%9C%AF%20eBPF.md)
- [eBPF å¦ä½ç®åæå¡ç½æ ¼](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%20%E5%A6%82%E4%BD%95%E7%AE%80%E5%8C%96%E6%9C%8D%E5%8A%A1%E7%BD%91%E6%A0%BC.md)
- [eBPF ç¨æ·ç©ºé´èææºå®ç°ç¸å³](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%20%E7%94%A8%E6%88%B7%E7%A9%BA%E9%97%B4%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%AE%9E%E7%8E%B0%E7%9B%B8%E5%85%B3.md)
- [åºäº eBPF å®ç°å®¹å¨è¿è¡æ¶å®å¨](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/%E5%9F%BA%E4%BA%8E%20eBPF%20%E5%AE%9E%E7%8E%B0%E5%AE%B9%E5%99%A8%E8%BF%90%E8%A1%8C%E6%97%B6%E5%AE%89%E5%85%A8.md)
- [æ·±å¥çè§£ Cilium ç eBPF æ¶ååè·¯å¾](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%20Cilium%20%E7%9A%84%20eBPF%20%E6%94%B6%E5%8F%91%E5%8C%85%E8%B7%AF%E5%BE%84.md)
- [eBPF æ¦è¿°ï¼ç¬¬ 1 é¨åï¼ä»ç»](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%20%E6%A6%82%E8%BF%B0%EF%BC%8C%E7%AC%AC%201%20%E9%83%A8%E5%88%86%EF%BC%9A%E4%BB%8B%E7%BB%8D.md)
- [eBPF æ¦è¿°ï¼ç¬¬ 2 é¨åï¼æºå¨åå­èç ](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%20%E6%A6%82%E8%BF%B0%EF%BC%8C%E7%AC%AC%202%20%E9%83%A8%E5%88%86%EF%BC%9A%E6%9C%BA%E5%99%A8%E5%92%8C%E5%AD%97%E8%8A%82%E7%A0%81.md)
- [eBPF æ¦è¿°ï¼ç¬¬ 3 é¨åï¼è½¯ä»¶å¼åçæ](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%20%E6%A6%82%E8%BF%B0%EF%BC%8C%E7%AC%AC%203%20%E9%83%A8%E5%88%86%EF%BC%9A%E8%BD%AF%E4%BB%B6%E5%BC%80%E5%8F%91%E7%94%9F%E6%80%81.md)
- [eBPF æ¦è¿°ï¼ç¬¬ 4 é¨åï¼å¨åµå¥å¼ç³»ç»è¿è¡](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%20%E6%A6%82%E8%BF%B0%EF%BC%8C%E7%AC%AC%204%20%E9%83%A8%E5%88%86%EF%BC%9A%E5%9C%A8%E5%B5%8C%E5%85%A5%E5%BC%8F%E7%B3%BB%E7%BB%9F%E8%BF%90%E8%A1%8C.md)
- [eBPF æ¦è¿°ï¼ç¬¬ 5 é¨åï¼è·è¸ªç¨æ·è¿ç¨](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/eBPF%20%E6%A6%82%E8%BF%B0%EF%BC%8C%E7%AC%AC%205%20%E9%83%A8%E5%88%86%EF%BC%9A%E8%B7%9F%E8%B8%AA%E7%94%A8%E6%88%B7%E8%BF%9B%E7%A8%8B.md)
- [ãè¯ãå¤§è§æ¨¡å¾®æå¡å©å¨ï¼eBPF + KubernetesKubeCon, 2020](https://github.com/0voice/kernel_new_features/blob/main/ebpf/%E6%96%87%E7%AB%A0/%E3%80%90%E8%AF%91%5D%E3%80%91%E5%A4%A7%E8%A7%84%E6%A8%A1%E5%BE%AE%E6%9C%8D%E5%8A%A1%E5%88%A9%E5%99%A8%EF%BC%9AeBPF%20%2B%20Kubernetes%EF%BC%88KubeCon%2C%202020%EF%BC%89.md)

### è§é¢

- [Netflix talks about Extended BPF - A new software type](https://pan.baidu.com/s/1VD-dsBheyJmDUhiIUJiQOw)---æåç : 83sv
- [containers_ebpf_kernel](https://pan.baidu.com/s/1NFzeWCJHmsXnzmDTHnt9vg)---æåç : hxkt

## ð¥ [llvm](https://llvm.org/)

<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150629940-85fa8f28-dfe9-4024-a97f-c8489471f7e9.png"/>
  
#### ââ æ¨¡ååãå¯éç¨çç¼è¯å¨ä»¥åå·¥å·é¾ææ¯çéå
</div>

### ææ¡£
- å®æ¹ææ¡£:
  - [LLVM: A Compilation Framework for Lifelong Program Analysis & Transformation](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E6%A1%A3/%20A%20Compilation%20Framework%20for%20Lifelong%20Program%20Analysis%20%26%20Transformation.pdf)
  - [Introduction to the LLVM Compiler System](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E6%A1%A3/Introduction%20to%20the%20LLVM%20Compiler%20System.pdf)
  - [LLVMè¯­è¨åèæå](https://web.archive.org/web/20120611064155/http://llvm.org/docs/LangRef.html)
  - [LLVMè¯­è¨åèæå-ä¸­æç](https://llvm.liuxfe.com/docs/langref/)
  - [å¥é¨LLVMæ ¸å¿åº](https://getting-started-with-llvm-core-libraries-zh-cn.readthedocs.io/zh_CN/latest/)
- ç¨æ·æåï¼
  - [ä½¿ç¨CMakeæå»ºLLVM](https://llvm.org/docs/CMake.html): ä½¿ç¨CMakeæå»ºç³»ç»çä¸»è¦å¥é¨æåçéå½ã
  - [å¨ARMå¹³å°ä¸æå»ºLLVMæå](https://llvm.org/docs/HowToBuildOnARM.html): å³äºå¨ARMä¸æå»ºåæµè¯LLVM/Clangçæ³¨æäºé¡¹ã
  - [å¦ä½ä½¿ç¨éç½®æä»¶å¼å¯¼ä¼åæå»ºClangåLLVM](https://llvm.org/docs/HowToBuildWithPGO.html): ä½¿ç¨PGOæå»ºLLVM/Clangçæ³¨æäºé¡¹ã
  - [å¦ä½ä¸ºARMå¹³å°äº¤åç¼è¯compiler-rt Builtins](https://llvm.org/docs/HowToCrossCompileBuiltinsOnArm.html): å³äºäº¤åæå»ºåæµè¯ARMçç¼è¯å¨-rtåç½®å½æ°çæ³¨æäºé¡¹ã
  - [å¦ä½ä½¿ç¨Clang/LLVMäº¤åç¼è¯Clang/LLVM](https://llvm.org/docs/HowToCrossCompileLLVM.html)ï¼å³äºäº¤åæå»ºåæµè¯LLVM / Clangçæ³¨æäºé¡¹ã
  - [ä½¿ç¨Microsoft Visual Studioå¼å§ä½¿ç¨LLVMç³»ç»](https://llvm.org/docs/GettingStartedVS.html)ï¼Windowsä¸ä½¿ç¨Visual Studioçä¸»è¦å¥é¨æåçéå½ã
  - [LLVMçåæåè½¬æ¢Passes](https://llvm.org/docs/Passes.html):LLVMä¸­å®ç°çä¼åååæåè¡¨
  - [å½åçæ¬çåå¸è¯´æ](https://llvm.org/docs/ReleaseNotes.html):è¿æè¿°äºæ°åè½ï¼å·²ç¥éè¯¯åå¶ä»éå¶ã
  - [å¦ä½æäº¤LLVMéè¯¯æ¥å](https://llvm.org/docs/HowToSubmitABug.html): æå³æ­£ç¡®æäº¤æå³æ¨å¨LLVMç³»ç»ä¸­éå°çä»»ä½éè¯¯çä¿¡æ¯çè¯´æã
  - [sphinxæ¨¡æ¿å¿«éå¥é¨](https://llvm.org/docs/SphinxQuickstartTemplate.html)ï¼ä½¿ç¨LLVMæµè¯åºç¡ç»æçåèæåã
  - [LLVMæµè¯å¥ä»¶åºç¡ç»ææå](https://llvm.org/docs/TestingGuide.html)ï¼ä½¿ç¨LLVMæµè¯åºç¡ç»æçåèæåã
  - [LLVMæµè¯å¥ä»¶ä½¿ç¨æå](https://llvm.org/docs/TestSuiteGuide.html)ï¼æè¿°å¦ä½ç¼è¯åè¿è¡æµè¯å¥ä»¶åºåæµè¯ã
  - [å¦ä½æå»ºCï¼C ++ï¼ObjCåObjC ++åç«¯](https://clang.llvm.org/get_started.html)ï¼ä»æºä»£ç æå»ºclangåç«¯çè¯´æã
  - [LLVMè¯å¸](https://llvm.org/docs/Lexicon.html)ï¼LLVMä¸­ä½¿ç¨çé¦å­æ¯ç¼©ç¥è¯ï¼æ¯è¯­åæ¦å¿µçå®ä¹ã
  - [å¦ä½å°æå»ºéç½®æ·»å å°LLVM Buildbotåºç¡ç»æ](https://llvm.org/docs/HowToAddABuilder.html):æå³å°æ°æå»ºå¨æ·»å å°LLVM buildbot masterçè¯´æã
  - [YAML I/O](https://llvm.org/docs/YamlIO.html)ï¼ä½¿ç¨LLVMçYAML I/Oåºçåèæåã
  - [åç«¯ä½èçæ§è½æç¤º](https://llvm.org/docs/Frontend/PerformanceTips.html)ï¼åç«¯ä½èå³äºå¦ä½çæIRçæå·§çéåï¼LLVMè½å¤ææå°ä¼åã
  - [Dockerfilesç¨äºæå»ºLLVMçæå](https://llvm.org/docs/Docker.html)ï¼ä½¿ç¨éLLVMæä¾çDockerfilesçåèã
- ç¼ç¨ææ¡£ï¼
  - [LLVMæ©å±](https://llvm.org/docs/Extensions.html)ï¼LLVMç¹å®çå·¥å·åæ ¼å¼æ©å±LLVMå¯»æ±å¼å®¹æ§ã
  - [CommandLine 2.0åºæå](https://llvm.org/docs/CommandLine.html)ï¼æä¾æå³ä½¿ç¨å½ä»¤è¡è§£æåºçä¿¡æ¯ã
  - [LLVMç¼ç æ å](https://llvm.org/docs/CodingStandards.html)ï¼è¯¦ç»ä»ç»äºLLVMç¼ç æ åï¼å¹¶æä¾äºæå³ç¼åé«æC ++ä»£ç çæç¨ä¿¡æ¯ã
  - [å¦ä½ä¸ºç±»å±æ¬¡ç»æè®¾ç½®LLVMæ ·å¼çRTTI](https://llvm.org/docs/HowToSetUpLLVMStyleRTTI.html)ï¼å¦ä½è®©`isa<>`ï¼`dyn_cast<>`ç­å¯ä¾æ¨çç±»å±æ¬¡çå®¢æ·ã
  - [æ©å±LLVMï¼æ·»å æä»¤ï¼åå¨å½æ°ï¼ç±»åç­](https://llvm.org/docs/ExtendingLLVM.html)ï¼å¨è¿éæ¥çå¦ä½åLLVMæ·»å æä»¤ååå¨å½æ°ã
  - [libFuzzer - ç¨äºè¦çå¼å¯¼çæ¨¡ç³æµè¯çåº](https://llvm.org/docs/LibFuzzer.html)ï¼ç¨äºç¼åè¿ç¨ä¸­å¼å¯¼æ¨¡ç³å¨çåº
  - [æ¨¡ç³LLVMåºåå·¥å·](https://llvm.org/docs/FuzzingLLVM.html)ï¼æå³ç¼ååä½¿ç¨Fuzzersæ¥æ¾LLVMä¸­çéè¯¯çä¿¡æ¯.
  - [Scudoç¡¬ååéå¨](https://llvm.org/docs/ScudoHardenedAllocator.html)ï¼ä¸ä¸ªå®ç°å®å¨å åºçmallocï¼ï¼çåºã
- å­ç³»ç»ææ¡£
  - [ç¼åLLVM Passes](https://llvm.org/docs/WritingAnLLVMPass.html)ï¼æå³å¦ä½ç¼åLLVMè½¬æ¢ååæçä¿¡æ¯
  - [ç¼åLLVMåç«¯](https://llvm.org/docs/WritingAnLLVMBackend.html)ï¼æå³å¦ä½ä¸ºæºå¨ç®æ ç¼åLLVMåç«¯çä¿¡æ¯
  - [LLVMä¸ç®æ æ å³çä»£ç çæå¨](https://llvm.org/docs/CodeGenerator.html)ï¼LLVMä»£ç çæå¨çè®¾è®¡åå®ç°ãå¦ææ¨æ­£å¨å°LLVMéæ°å®ä½å°æ°æ¶æï¼è®¾è®¡æ°çcodegenä¼ éæå¢å¼ºç°æç»ä»¶ï¼åéå¸¸æç¨ã
  - [æºå¨IRï¼MIRï¼æ ¼å¼åèæå](https://llvm.org/docs/MIRLangRef.html)ï¼MIRåºååæ ¼å¼çåèæåï¼ç¨äºæµè¯LLVMçä»£ç çæè¿ç¨ã
  - [TableGen](https://llvm.org/docs/TableGen/index.html)ï¼æè¿°äºTableGenå·¥å·ï¼LLVMä»£ç çæå¨å¤§éä½¿ç¨å®ã
  - [LLVMå«ååæåºç¡ç»æ](https://llvm.org/docs/AliasAnalysis.html)ï¼æå³å¦ä½ç¼åæ°å«ååæå®ç°æå¦ä½ä½¿ç¨ç°æåæçä¿¡æ¯ã
  - [MemorySSA](https://llvm.org/docs/MemorySSA.html)ï¼æå³LLVMä¸­çMemorySSAå®ç¨ç¨åºçä¿¡æ¯ï¼ä»¥åå¦ä½ä½¿ç¨å®ã
  - [ä½¿ç¨LLVMè¿è¡åå¾æ¶é](https://llvm.org/docs/GarbageCollection.html)ï¼æ¥å£æºè¯­è¨ç¼è¯å¨åºè¯¥ç¨äºç¼è¯GCç¨åºã
  - [ä½¿ç¨LLVMè¿è¡æºçº§å«è°è¯](https://llvm.org/docs/SourceLevelDebugging.html)ï¼æ¬ææ¡£æè¿°äºLLVMæºä»£ç çº§è°è¯å¨èåçè®¾è®¡åçå¿µã
  - [LLVMä¸­çèªå¨ç¢éå](https://llvm.org/docs/Vectorizers.html)ï¼æ¬ææ¡£æè¿°äºLLVMä¸­ç¢éåçå½åç¶æ
  - [LLVMä¸­çå¼å¸¸å¤ç](https://llvm.org/docs/ExceptionHandling.html)ï¼æ¬ææ¡£æè¿°äºLLVMä¸­å¼å¸¸å¤ççè®¾è®¡åå®ç°
  - [å¦ä½æ·»å ä¸ä¸ªåçº¦æçæµ®ç¹åå¨å½æ°](https://llvm.org/docs/AddingConstrainedIntrinsics.html)ï¼å¨LLVMä¸­æ·»å æ°ççº¦ææ°å­¦åå¨æ¶ï¼æä¾å¿è¦çæ­¥éª¤ã
  - [LLVM bugpointå·¥å·ï¼è®¾è®¡åä½¿ç¨](https://llvm.org/docs/Bugpoint.html)ï¼èªå¨éè¯¯æ¥æ¾å¨åæµè¯ç¨ä¾åå°å¨æè¿°åä½¿ç¨ä¿¡æ¯
  - [LLVM Bitcodeæä»¶æ ¼å¼](https://llvm.org/docs/BitCodeFormat.html)ï¼è¿æè¿°äºç¨äºLLVMâbcâæä»¶çæä»¶æ ¼å¼åç¼ç ã
  - [æ¯æåº](https://llvm.org/docs/SupportLibrary.html)ï¼æ¬ææ¡£æè¿°äºLLVMæ¯æåºï¼lib/Supportï¼ä»¥åå¦ä½ä½¿LLVMæºä»£ç å¯ç§»æ¤
  - [LLVMé¾æ¥æ¶é´ä¼åï¼è®¾è®¡åå®ç°](https://llvm.org/docs/LinkTimeOptimization.html)ï¼æ¬ææ¡£æè¿°äºLLVMæ¨¡åé´ä¼åå¨ä¸é¾æ¥å¨åå¶è®¾è®¡ä¹é´çæ¥å£
  - [LLVMé»éæä»¶](https://llvm.org/docs/GoldPlugin.html)ï¼å¦ä½å¨Linuxä¸ä½¿ç¨é¾æ¥æ¶ä¼åæ¥æå»ºç¨åºã
  - [ä½¿ç¨GDBè°è¯JIT-edä»£ç ](https://llvm.org/docs/DebuggingJITedCode.html):å¦ä½ä½¿ç¨GDBè°è¯JITedä»£ç ã
  - [MCJITè®¾è®¡ä¸å®æ½](https://llvm.org/docs/MCJITDesignAndImplementation.html)ï¼æè¿°äºMCJITæ§è¡å¼æçåé¨å·¥ä½åç
  - [LLVMåæ¯æéåæ°æ®](https://llvm.org/docs/BranchWeightMetadata.html)ï¼æä¾æå³åæ¯é¢æµä¿¡æ¯çä¿¡æ¯ã
  - [LLVMåé¢çæ¯è¯­](https://llvm.org/docs/BlockFrequencyTerminology.html):æä¾æå³BlockFrequencyInfo åæè¿ç¨ä¸­ä½¿ç¨çæ¯è¯­çä¿¡æ¯
  - [LLVMä¸­çåæ®µå æ ](https://llvm.org/docs/SegmentedStacks.html):æ¬ææ¡£æè¿°äºåæ®µå æ ä»¥åå®ä»¬å¨LLVMä¸­çä½¿ç¨æ¹å¼
  - [LLVMçå¯éä¸°å¯çåæ±ç¼è¾åº](https://llvm.org/docs/MarkedUpDisassembly.html):æ¬ææ¡£ä»ç»äºå¯éçä¸°å¯åæ±ç¼è¾åºè¯­æ³
  - [å¦ä½ä½¿ç¨å±æ§](https://llvm.org/docs/HowToUseAttributes.html)ï¼åç­æå³æ°å±æ§åºç¡ç»æçä¸äºé®é¢ã
  - [NVPTXåç«¯ç¨æ·æå](https://llvm.org/docs/NVPTXUsage.html)ï¼æ¬ææ¡£æè¿°äºä½¿ç¨NVPTXåç«¯ç¼è¯GPUåæ ¸ã
  - [AMDGPUåç«¯ç¨æ·æå](https://llvm.org/docs/AMDGPUUsage.html):æ¬ææ¡£æè¿°äºä½¿ç¨AMDGPUåç«¯ç¼è¯GPUåæ ¸ã
  - [LLVMä¸­çå æ æ å°åè¡¥ä¸ç¹](https://llvm.org/docs/StackMaps.html):LLVMæ¯æå°æä»¤å°åæ å°å°å¼çä½ç½®å¹¶åè®¸ä¿®è¡¥ä»£ç ã
  - [å¨big endianæ¨¡å¼ä¸ä½¿ç¨ARM NEONæä»¤](https://llvm.org/docs/BigEndianNEON.html)ï¼LLVMæ¯æå¨å¤§ç«¯ARMç®æ ä¸çæNEONæä»¤æç¹ä¸ç´è§ãæ¬ææ¡£è§£éäºå®æ½åçç±ã
  - [LLVMä»£ç è¦çæ å°æ ¼å¼](https://llvm.org/docs/CoverageMappingFormat.html):LLVMä»£ç è¦çæ å°æ ¼å¼
  - [LLVMä¸­çåå¾æ¶éå®å¨ç¹](https://llvm.org/docs/Statepoints.html)ï¼è¿æè¿°äºä¸ç»åå¾æ¶éæ¯æçå®éªæ©å±ã
  - [MergeFunctions Passï¼å®æ¯å¦ä½å·¥ä½ç](https://llvm.org/docs/MergeFunctions.html)ï¼æè¿°åå¹¶ä¼åçå½æ°ã
  - [InAllocaå±æ§çè®¾è®¡åä½¿ç¨](https://llvm.org/docs/InAlloca.html)ï¼inallocaåæ°å±æ§çæè¿°ã
  - [FaultMapsåéå¼æ£æ¥](https://llvm.org/docs/FaultMaps.html)ï¼LLVMæ¯ææå æ§å¶æµå¥éè¯¯æºå¨æä»¤ã
  - [ç¨clangç¼è¯CUDA](https://llvm.org/docs/CompileCudaWithLLVM.html)ï¼LLVMå¯¹CUDAçæ¯æã
  - [LLVMä¸­çååç¨åº](https://llvm.org/docs/Coroutines.html):LLVMä¸­çååç¨åº.
  - [å¨å±æä»¤éæ©](https://llvm.org/docs/GlobalISel.html)ï¼è¿æè¿°äºååæä»¤éæ©æ¿æ¢GlobalISel
  - [XRayä»ªè¡¨](https://llvm.org/docs/XRay.html)ï¼æå³å¦ä½å¨LLVMä¸­ä½¿ç¨XRayçé«çº§ææ¡£ã
  - [ä½¿ç¨XRayè¿è¡è°è¯](https://llvm.org/docs/XRayExample.html)ï¼å¦ä½ä½¿ç¨XRayè°è¯åºç¨ç¨åºçç¤ºä¾ã
  - [Microsoft PDBæä»¶æ ¼å¼](https://llvm.org/docs/PDB/index.html)ï¼Microsoft PDBï¼ç¨åºæ°æ®åºï¼æä»¶æ ¼å¼çè¯¦ç»è¯´æã
  - [æ§å¶æµç¨éªè¯å·¥å·è®¾è®¡ææ¡£](https://llvm.org/docs/CFIVerify.html)ï¼æ§å¶æµå®æ´æ§éªè¯å·¥å·çè¯´æ
  - [ææºè´è·å¼ºå](https://llvm.org/docs/SpeculativeLoadHardening.html)ï¼Spectre v1çæ¨æµè´è½½å¼ºåç¼è§£çæè¿°
  - [å æ å®å¨åæ](https://llvm.org/docs/StackSafetyAnalysis.html)ï¼æ¬ææ¡£æè¿°äºå±é¨åéçå æ å®å¨æ§åæçè®¾è®¡ã



### LLVMå½ä»¤æå

#### åºæ¬å½ä»¤

| å½ä»¤                                                         | è¯´æ                              |
| :----------------------------------------------------------- | :-------------------------------- |
| [llvm-as](https://llvm.liuxfe.com/docs/man/llvm-as.html)     | LLVMæ±ç¼å¨                        |
| [llvm-dis](https://llvm.liuxfe.com/docs/man/llvm-dis.html)   | LLVMåæ±ç¼å¨                      |
| [opt](https://llvm.liuxfe.com/docs/man/opt.html)             | LLVMä¼åå¨                        |
| [llc](https://llvm.liuxfe.com/docs/man/llc.html)             | LLVMéæç¼è¯å¨                    |
| [lli](https://llvm.liuxfe.com/docs/man/lli.html)             | LLVMå­èç è§£éå¨                  |
| [llvm-link](https://llvm.liuxfe.com/docs/man/llvm-link.html) | LLVMå­èç è¿æ¥å¨                  |
| [llvm-lib](https://llvm.liuxfe.com/docs/man/llvm-lib.html)   | LLVMçä¸lib.exeå¼ç¨çåºå·¥å·       |
| [llvm-lipo](https://llvm.liuxfe.com/docs/man/llvm-lipo.html) | ç¨äºå¤çéç¨äºè¿å¶æä»¶çLLVMå·¥å·  |
| [llvm-config](https://llvm.liuxfe.com/docs/man/llvm-config.html) | æå°LLVMç¼è¯éé¡¹                  |
| [llvm-cxxmap](https://llvm.liuxfe.com/docs/man/llvm-cxxmap.html) | Mangled nameéæ å°å·¥å·            |
| [llvm-diff](https://llvm.liuxfe.com/docs/man/llvm-diff.html) | LLVM ç»æâdiffâ                   |
| [llvm-cov](https://llvm.liuxfe.com/docs/man/llvm-cov.html)   | ååºè¦çä¿¡æ¯                      |
| [llvm-profdata](https://llvm.liuxfe.com/docs/man/llvm-profdata.html) | éç½®æ°æ®å·¥å·                      |
| [llvm-stress](https://llvm.liuxfe.com/docs/man/llvm-stress.html) | çæéæºç.llæä»¶                 |
| [llvm-symbolizer](https://llvm.liuxfe.com/docs/man/llvm-symbolizer.html) | å°å°åè½¬æ¢ä¸ºæºä»£ç ä¸­çä½ç½®        |
| [llvm-dwarfdump](https://llvm.liuxfe.com/docs/man/llvm-dwarfdump.html) | è½¬å¨å¹¶æ£éªDWARFè°è¯ä¿¡æ¯           |
| [dsymutil](https://llvm.liuxfe.com/docs/man/dsymutil.html)   | æä½å­æ¡£æä»¶ä¸­çDWARFè°è¯ç¬¦å·æä»¶ |
| [llvm-mca](https://llvm.liuxfe.com/docs/man/llvm-mca.html)   | LLVMæºå¨ç åæå¨                  |
| [llvm-readobj](https://llvm.liuxfe.com/docs/man/llvm-readobj.html) | LLVMç®æ æä»¶åæå¨                |

#### GNU bintilsæ¿ä»£å½ä»¤

| å½ä»¤                                                         | è¯´æ                               |
| :----------------------------------------------------------- | :--------------------------------- |
| [llvm-addr2line](https://llvm.liuxfe.com/docs/man/llvm-addr2line.html) | addr2lineçæ¿ä»£å                  |
| [llvm-ar](https://llvm.liuxfe.com/docs/man/llvm-ar.html)     | LLVMå½æ¡£å¨                         |
| [llvm-cxxfilt](https://llvm.liuxfe.com/docs/man/llvm-cxxfilt.html) | LLVMç¬¦ååç§°åæå¨                 |
| [llvm-nm](https://llvm.liuxfe.com/docs/man/llvm-nm.html)     | ååºLLVMå­èç åç®æ æä»¶ä¸­çç¬¦å·è¡¨ |
| [llvm-objcopy](https://llvm.liuxfe.com/docs/man/llvm-objcopy.html) | ç®æ æä»¶å¤å¶åç¼è¾å·¥å·             |
| [llvm-objdump](https://llvm.liuxfe.com/docs/man/llvm-objdump.html) | LLVMç®æ æä»¶è½¬å¨å¨                 |
| [llvm-ranlib](https://llvm.liuxfe.com/docs/man/llvm-ranlib.html) | åºå­æ¡£ç´¢å¼çæå·¥å·                 |
| [llvm-readelf](https://llvm.liuxfe.com/docs/man/llvm-readelf.html) | GNUé£æ ¼çLLVMå¯¹è±¡è¯»åå¨            |
| [llvm-size](https://llvm.liuxfe.com/docs/man/llvm-size.html) | æå°ç®æ æä»¶å°ºå¯¸ä¿¡æ¯               |
| [llvm-strings](https://llvm.liuxfe.com/docs/man/llvm-strings.html) | æå°ç®æ æä»¶ä¸­çå­ç¬¦ä¸²             |
| [llvm-strip](https://llvm.liuxfe.com/docs/man/llvm-strip.html) | ç®æ æä»¶å»é¤è°è¯ä¿¡æ¯å·¥å·           |

#### è°è¯å·¥å·

| å½ä»¤                                                         | è¯´æ                 |
| :----------------------------------------------------------- | :------------------- |
| [bugpoint](https://llvm.liuxfe.com/docs/man/bugpoint.html)   | èªå¨æµè¯ç¨ä¾ç¼©åå·¥å· |
| [llvm-extract](https://llvm.liuxfe.com/docs/man/llvm-extract.html) | ä»LLVMæ¨¡åä¸­æåå½æ° |
| [llvm-bcanalyzer](https://llvm.liuxfe.com/docs/man/llvm-bcanalyzer.html) | LLVMå­èç åæå¨     |

#### å¼åå·¥å·

| å½ä»¤                                                         | è¯´æ                        |
| :----------------------------------------------------------- | :-------------------------- |
| [FileCheck](https://llvm.liuxfe.com/docs/man/filechcke.html) | çµæ´»çæ¨¡å¼å¹éæä»¶éªè¯ç¨åº  |
| [tblgen](https://llvm.liuxfe.com/docs/man/tblgen.html)       | ç®æ æè¿°å°C++ä»£ç çæå¨     |
| [lit](https://llvm.liuxfe.com/docs/man/lit.html)             | LLVMéææµè¯ä»ª              |
| [llvm-build](https://llvm.liuxfe.com/docs/man/llvm-build.html) | LLVMé¡¹ç®æå»ºå®ç¨ç¨åº        |
| [llvm-exegesis](https://llvm.liuxfe.com/docs/man/llvm-exegesis.html) | LLVMæºå¨æä»¤åºå            |
| [llvm-pdbutil](https://llvm.liuxfe.com/docs/man/llvm-pdbutil.html) | PDBæä»¶åè¯åè¯æ­           |
| [llvm-locstats](https://llvm.liuxfe.com/docs/man/llvm-locstats.html) | è®¡ç®DWARFè°è¯ä½ç½®çç»è®¡ä¿¡æ¯ |



### å¼æºé¡¹ç® 

- [emscripten-core/emscripten](https://github.com/emscripten-core/emscripten): Emscripten:ä¸ä¸ªllvmå°webassemblyçç¼è¯å¨
- [tinygo-org/tinygo](https://github.com/tinygo-org/tinygo): å¾®æ§å¶å¨ãWebAssembly (WASM/WASI)åå½ä»¤è¡å·¥å·ãåºäºLLVMã
- [numba/numba](https://github.com/numba/numba): ä½¿ç¨LLVMæ¯æNumPyå¨æPythonç¼è¯å¨
- [avast/retdec](https://github.com/avast/retdec): RetDecæ¯ä¸ä¸ªåºäºLLVMçå¯éå®åæºå¨ç åç¼è¯å¨ã
- [lifting-bits/mcsema](https://github.com/lifting-bits/mcsema): å°x86ãamd64ãaarch64ãsparc32åsparc64ç¨åºäºè¿å¶ä»£ç æåå°LLVMä½ç çæ¡æ¶
- [microsoft/DirectXShaderCompiler](https://github.com/microsoft/DirectXShaderCompiler): è¿ä¸ªrepoæç®¡äºDirectX Shaderç¼è¯å¨çæºä»£ç ï¼å®æ¯åºäºLLVM/Clangçã
- [andreasfertig/cppinsights](https://github.com/andreasfertig/cppinsights): c++æ´å¯åââç¨ç¼è¯å¨çç¼åçä½ çæºä»£ç 
- [google/souper](https://github.com/google/souper): LLVM IRçè¶ä¼åå¨
- [HikariObfuscator/Hikari](https://github.com/HikariObfuscator/Hikari): LLVMæ¨¡ç³å¤ç
- [dotnet/llilc](https://github.com/dotnet/llilc):è¿ä¸ªrepoåå«LLILCï¼ä¸ä¸ªåºäºLLVMçãnet Coreç¼è¯å¨ãå®åæ¬ä¸ç»è·¨å¹³å°çãnetä»£ç çæå·¥å·ï¼å¯ä»¥å°MSILå­èç ç¼è¯æLLVMæ¯æçå¹³å°ã
- [banach-space/llvm-tutor](https://github.com/banach-space/llvm-tutor): ç¨äºæå­¦åå­¦ä¹ çæ å¤LLVMéè¡è¯çéå
- [numba/llvmlite](https://github.com/numba/llvmlite): ç¨äºç¼åJITç¼è¯å¨çè½»éçº§LLVM pythonç»å®
- [yrnkrn/zapcc](https://github.com/yrnkrn/zapcc): zapccæ¯ä¸ä¸ªåºäºclangçç¼å­c++ç¼è¯å¨ï¼æ¨å¨æ§è¡æ´å¿«çç¼è¯
- [go-llvm/llgo](https://github.com/go-llvm/llgo): åºäºllvmçç¼è¯å¨
- [eliben/llvm-clang-samples](https://github.com/eliben/llvm-clang-samples): unmaintenance:ä½¿ç¨LLVMåClangç¼è¯åºåå·¥å·çä¾å­

### æç« 

- [LLVM å¥é¨ç¯](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E7%AB%A0/LLVM%20%E5%85%A5%E9%97%A8%E7%AF%87.md)
- [LLVM-Clangå¥é¨](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E7%AB%A0/LLVM-Clang%E5%85%A5%E9%97%A8.md)
- [LLVMç¼è¯å¨æ¡æ¶ä»ç»](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E7%AB%A0/LLVM%E7%BC%96%E8%AF%91%E5%99%A8%E6%A1%86%E6%9E%B6%E4%BB%8B%E7%BB%8D.md)
- [Llvmç¼è¯çåºæ¬æ¦å¿µåæµç¨](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E7%AB%A0/llvm%E7%BC%96%E8%AF%91%E7%9A%84%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5%E5%92%8C%E6%B5%81%E7%A8%8B.md)
- [åç«¯ææ¯çéç¨ï¼LLVMä¸ä»ä»è®©ä½ é«æ](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E7%AB%A0/%E5%90%8E%E7%AB%AF%E6%8A%80%E6%9C%AF%E7%9A%84%E9%87%8D%E7%94%A8%EF%BC%9ALLVM%E4%B8%8D%E4%BB%85%E4%BB%85%E8%AE%A9%E4%BD%A0%E9%AB%98%E6%95%88.md)
- [ç¼è¯ä¼åï½LLVMä»£ç çæææ¯è¯¦è§£åå¨æ°æ®åºä¸­çåºç¨](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E7%AB%A0/%E7%BC%96%E8%AF%91%E4%BC%98%E5%8C%96%EF%BD%9CLLVM%E4%BB%A3%E7%A0%81%E7%94%9F%E6%88%90%E6%8A%80%E6%9C%AF%E8%AF%A6%E8%A7%A3%E5%8F%8A%E5%9C%A8%E6%95%B0%E6%8D%AE%E5%BA%93%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8.md)
- [ç¼è¯å¨ååºå±åè¯è§£é](https://github.com/0voice/kernel_new_features/blob/main/llvm/%E6%96%87%E7%AB%A0/%E7%BC%96%E8%AF%91%E5%99%A8%E5%8F%8A%E5%BA%95%E5%B1%82%E5%90%8D%E8%AF%8D%E8%A7%A3%E9%87%8A.md)

### è§é¢
- [How LLVM & Clang work](https://pan.baidu.com/s/1yTDS9Bn5CiFGhotKjNfcIw)---æåç : 225f
- [2021 LLVM Dev Mtg âOtter- Tracing & Visualizing OpenMP Programs as DAGs Through LLVM's OMPT...â](https://pan.baidu.com/s/1lbO_764_sXgMf5mgwajdcw)---æåç : d2k2
- [2021 LLVM Dev Mtg âNavigating Exotic SIMD Lands with an LLVM Guideâ](https://pan.baidu.com/s/1SAbepiyv0X6W7Qxf6qVxuA)---æåç : 5v6s
- [2019 LLVM Developersâ Meeting- E. Christopher & J. Doerfert âIntroduction to  LLVMâ](https://pan.baidu.com/s/1_g5P0r0Cku30w3m2LiJgWw)---æåç : 8u6e
- [2019 LLVM Developersâ Meeting- S. Haastregt & A. Stulova âAn overview of Clang â](https://pan.baidu.com/s/1VOhM2SOeWnRoy24jaWw7dQ)---æåç : r6ct
- [P. Goldsborough âclang-useful- Building useful tools with LLVM and clang for fun and profit](https://pan.baidu.com/s/1DkpEdZXeBISJMuExVtpZKg)---æåç : xemt
<!--

## ð¥ kvm

<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150633416-9961e8b7-ff81-488b-8cfe-b69ba739c1ff.png"/>
  
#### ââ Linuxåæ ¸ä¸­çèæååºç¡è®¾æ½
</div>


### ææ¡£
- å®æ¹ææ¡£:
- å¶ä»ææ¡£ï¼

### å¼æºé¡¹ç® 

### æç« 

### è§é¢(æåç ï¼1024)

## ð¥ ceph

<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150633285-0e03e44f-8755-4b12-9b62-8f8030d44c94.png"/>
  
#### ââ å­å¨çæªæ¥
</div>

### ææ¡£
- å®æ¹ææ¡£:
- å¶ä»ææ¡£ï¼

### å¼æºé¡¹ç® 

### æç« 

### è§é¢(æåç ï¼1024)

## ð¥ fuse

<div  align=center>
  
<img width="60%" height="60%" src="https://user-images.githubusercontent.com/87457873/150633338-fde17a17-9cfb-4a32-bc51-d76e02b6904e.png"/>
  
#### ââ ç¨æ·ææä»¶ç³»ç»
</div>

### ææ¡£
- å®æ¹ææ¡£:
- å¶ä»ææ¡£ï¼

### å¼æºé¡¹ç® 

### æç« 

### è§é¢(æåç ï¼1024)

-->


## ð¥ èç³»ä¸æ 

#### [Linuxåæ ¸æºç /åå­è°ä¼/æä»¶ç³»ç»/è¿ç¨ç®¡ç/è®¾å¤é©±å¨/ç½ç»åè®®æ ](https://ke.qq.com/course/4032547?flowToken=1041395)

#### å³æ³¨å¾®ä¿¡å¬ä¼å·ãåå°æå¡æ¶æå¸ãââãèç³»æä»¬ãï¼å å°å§å§ï¼è·åãæ¯å¤©åè´¹ææ¯ç´æ­é¾æ¥ãä»¥åæ´å¤Linux åæ ¸æºç å­¦ä¹ èµæï¼

<img width="65%" height="65%" src="https://user-images.githubusercontent.com/87457873/130796999-03af3f54-3719-47b4-8e41-2e762ab1c68b.png"/>
