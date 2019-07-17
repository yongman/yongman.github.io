---
title: 黑群晖Synology DS918+打造个人网络媒体中心
categories: 
- [生产力]
- [生命不息]
date: 2018-11-05 23:53:38
---

还记得几年前，一个机缘巧合，碰到有同学利用台式机安装了一个黑群晖，当时自己也将废弃在角落08年的联想笔记本成功跑了黑群晖，然后继续被仍在房间的角落。

当时就被群晖的软件生态吸引了，各种app涵盖了文件管理、照片、视频、各种数据同步等场景。

趁着房子装修，现在局域网起码也要千兆网络，目前在使用的小米路由器R2D虽然也是千兆，但是网络延迟抖动比较严重，是时候升级了。

### DIY

和前同事咨询了一番，也在tg交流群和youtube上了解了一番，决定自己DIY硬件，具体的使用的硬件：
- 华擎J4105主板
![](https://raw.githubusercontent.com/yongman/i/img/picgo/lviGPA8.jpg)
- 迎广MS04机箱 自带电源
![](https://raw.githubusercontent.com/yongman/i/img/picgo/kTfBhtX.png)
![](https://raw.githubusercontent.com/yongman/i/img/picgo/lx45AQQ.jpg)
- 三星DDR4 8G内存
![](https://raw.githubusercontent.com/yongman/i/img/picgo/SmAX52G.png)
- PCIex1双网口千兆网卡
![](https://raw.githubusercontent.com/yongman/i/img/picgo/FewkC7q.jpg)
- 西数红盘4T
预算有限，先上一块盘，后续再上
![](https://raw.githubusercontent.com/yongman/i/img/picgo/8GWerbS.jpg)
- 闪迪酷豆8G U盘引导（真的很小，插到主板上，还真不易被发现）

硬件就是这样。NAS系统选择还是群晖，安装的是最新版DSM6.2.1。
![](https://raw.githubusercontent.com/yongman/i/img/picgo/Dfe0xIV.png)

有Virtual Machine Manager和docker支持，可以虚拟机运行其他系统，考虑到后续升级16G内存的可能，先上单条8G。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/oAc5z2W.png)
网络拓扑图

如果让NAS只用作存储有点浪费资源，内存和CPU性能过剩，所以干脆来跑软路由吧。

在VMM中运行的LEDE软路由，破解移动光猫，设置光猫为桥接模式，光猫负责光纤入户桥接，拨号交给LEDE，拨号成功。
![](https://raw.githubusercontent.com/yongman/i/img/picgo/cYUuDj7.png)

移动宽带支持多拨，100M宽带在4拨的情况下，测试能达到300M的带宽。
![](https://raw.githubusercontent.com/yongman/i/img/picgo/4astvzq.png)
![](https://raw.githubusercontent.com/yongman/i/img/picgo/IDNDFfl.png)

LEDE自带软件中心，可以直接配置ss相关客户端，实现智能科学上网，不需要再自己折腾脚本。

### 遇到的问题

1. 无法启用第三个网卡

由于DS918+只有两个千兆网卡，所以黑群晖的DS918+系统中对于网卡数据有一定限制，具体表现是ifconfig能看到所有的物理网卡设备eth0、eth1、eth2，启动后只有两个网卡是enable状态，eth2没有启用。

通过查看/etc/rc.network网络初始化脚本，发现三个网卡其实是都识别的，新插入的网卡会在/etc/sysconfig/network-scripts下生成默认配置文件，每次开机会发现eth2相关的配置总会莫名其妙的被删除，导致无法对eth2网卡初始化。
![](https://raw.githubusercontent.com/yongman/i/img/picgo/DQgCkGL.png)

通过`dmesg`查看，发现奇怪日志：
```
2018-11-05T00:35:41+08:00 DiskStation synochecknetworkcfg: synochecknetworkcfg.c:50 unlink [/etc/sysconfig/network-scripts/ifcfg-eth2], max LAN id is [1]
```
ifcfg-eth2被unlink，最大id是1??? synochecknetworkcfg这个bin文件是c编译的，没有源码也没办法看内部的逻辑，索性想办法把这个check过程去掉。

找到在文件`/etc/init/network.conf`中会调用该命令，注释掉。
![](https://raw.githubusercontent.com/yongman/i/img/picgo/QyT1xiI.png)
重启验证，成功识别。
![](https://raw.githubusercontent.com/yongman/i/img/picgo/nvfBGE8.png)

2. lede多拨稳定性

初次设置多拨成功概率大，重启路由器有一定概率存在部分虚拟wan口拨号失败，未解决。
![](https://raw.githubusercontent.com/yongman/i/img/picgo/6p9bDWo.png)

3. i915显卡驱动加载失败

dmesg -T看到有kernel panic输出：

```
[Wed Dec 12 12:44:30 2018] Call Trace:
[Wed Dec 12 12:44:30 2018]  [<ffffffff8140c409>] ? i2c_transfer+0x79/0x90
[Wed Dec 12 12:44:30 2018]  [<ffffffffa0515da2>] drm_do_probe_ddc_edid+0xc2/0x130 [drm]
[Wed Dec 12 12:44:30 2018]  [<ffffffffa0517360>] drm_get_edid+0x300/0x390 [drm]
[Wed Dec 12 12:44:30 2018]  [<ffffffffa06768bf>] intel_dp_init_connector+0xa7f/0xf00 [i915]
[Wed Dec 12 12:44:30 2018]  [<ffffffffa066b533>] intel_ddi_init+0x383/0x4f0 [i915]
[Wed Dec 12 12:44:30 2018]  [<ffffffffa064952a>] intel_modeset_init+0x15da/0x1a70 [i915]
[Wed Dec 12 12:44:30 2018]  [<ffffffffa0683507>] ? intel_setup_gmbus+0x2e7/0x310 [i915]
[Wed Dec 12 12:44:30 2018]  [<ffffffffa05beccf>] i915_driver_load+0xa0f/0xe00 [i915]
[Wed Dec 12 12:44:30 2018]  [<ffffffffa05c9797>] i915_pci_probe+0x27/0x40 [i915]
[Wed Dec 12 12:44:30 2018]  [<ffffffff812fc85c>] pci_device_probe+0x8c/0x100
[Wed Dec 12 12:44:30 2018]  [<ffffffff813842d1>] driver_probe_device+0x1f1/0x310
[Wed Dec 12 12:44:30 2018]  [<ffffffff81384472>] __driver_attach+0x82/0x90
[Wed Dec 12 12:44:30 2018]  [<ffffffff813843f0>] ? driver_probe_device+0x310/0x310
[Wed Dec 12 12:44:30 2018]  [<ffffffff81382361>] bus_for_each_dev+0x61/0xa0
[Wed Dec 12 12:44:30 2018]  [<ffffffff81383d69>] driver_attach+0x19/0x20
[Wed Dec 12 12:44:30 2018]  [<ffffffff81383993>] bus_add_driver+0x1b3/0x230
[Wed Dec 12 12:44:30 2018]  [<ffffffffa06fa000>] ? 0xffffffffa06fa000
[Wed Dec 12 12:44:30 2018]  [<ffffffff81384c7b>] driver_register+0x5b/0xe0
[Wed Dec 12 12:44:30 2018]  [<ffffffff812fb337>] __pci_register_driver+0x47/0x50
[Wed Dec 12 12:44:30 2018]  [<ffffffffa06fa03e>] i915_init+0x3e/0x45 [i915]
[Wed Dec 12 12:44:30 2018]  [<ffffffff810003b6>] do_one_initcall+0x86/0x1b0
[Wed Dec 12 12:44:30 2018]  [<ffffffff810dfdd8>] do_init_module+0x56/0x1be
[Wed Dec 12 12:44:30 2018]  [<ffffffff810b61ad>] load_module+0x1ded/0x2070
[Wed Dec 12 12:44:30 2018]  [<ffffffff810b3510>] ? __symbol_put+0x50/0x50
[Wed Dec 12 12:44:30 2018]  [<ffffffff810b65b9>] SYSC_finit_module+0x79/0x80
[Wed Dec 12 12:44:30 2018]  [<ffffffff810b65d9>] SyS_finit_module+0x9/0x10
[Wed Dec 12 12:44:30 2018]  [<ffffffff81567444>] entry_SYSCALL_64_fastpath+0x18/0x8c
[Wed Dec 12 12:44:30 2018] Code:  Bad RIP value.
[Wed Dec 12 12:44:30 2018] RIP  [<          (null)>]           (null)
```

表现就是视频解码无法启用硬件加速，找到的解决方案是：https://bbs.archlinux.org/viewtopic.php?id=198852

在内核启动时增加参数i915.enable_execlists=0，验证有效，重启内核无panic输出。

```
[Sun Dec 30 16:20:51 2018] [drm] Finished loading DMC firmware i915/glk_dmc_ver1_04.bin (v1.4)
[Sun Dec 30 16:20:52 2018] [drm] failed to retrieve link info, disabling eDP
[Sun Dec 30 16:20:52 2018] [drm] Initialized i915 1.6.0 20180514 for 0000:00:02.0 on minor 0
```



### 使用感受

`2018-11-05` 系统已经稳定运行一周，网速有改善明显，一个NAS+多个AP完美实现上网，性能堪比专业路由。

`2019-01-20` 使用一段时间LEDE软路由，在多拨的情况下存在bug，在大流量下载的时候会出现wan口拨号失败掉线的情况，只能重启lede恢复，所以增加了ikuai做多拨主路由，lede做旁路路由，避免出现拨号失败导致的断网问题，跑一段时间看看ikuai稳定性，毕竟ikuai是商用的系统，稳定性应该比lede好得多。双软路网络拓扑图：

![](https://raw.githubusercontent.com/yongman/i/img/picgo/n0EKgbH.png)

`2019-04-10` 使用过程中出现了几次ikuai拨号在大流量的情况下断网的情况，表现是群晖物理网卡正常，但是ikuai的虚拟wan口网络不通，断网情况下ikuai依然会重试拨号，但是出现超时。和之前lede拨号表现一致，只能重启虚拟机恢复。由于lan口和wan口对应的物理网卡是自己插入的pcie双网口网卡，怀疑是pcie网卡虚拟驱动等导致的虚拟网卡停止工作。将ikuai的wan口重新桥接群晖的第三个物理网卡，使用板载网卡进行wan口拨号，后续没再出现断网的情况。

`2019-05-11`老问题再次复现，添加了下载任务后，出现了ikuai无法拨号的情况，这就说明不是物理网卡驱动的问题。决定升级dsm 6.2.2，最新版本更新了virtual machine manager，都说稳定优先，没有大问题黑群升级大概率会出点问题的。升级后出现的问题：

1. 依旧无法正常关机。
2. 开机必须插着vga显示器，显示器可以是关闭状态，不插显示器无法启动。
3. i915驱动panic再次出现，添加启动参数方式失效。
4. J4105板载网卡无法使用。

得出了结论就是：等这些问题有了解决方案后，以后无特殊情况以后还是不升级了吧，稳定优先。逃...

`2019-05-25` 先说一下半年以来遇到的软路由断网情况没有复现，改动就是去掉了多拨，采用的普通的单拨形式。

上一周没忍住更新了dsm6.2.2，过了一周终于决定回退6.2.1，回退方式就是格式化`/dev/md0`分区，然后重启安装。具体步骤：

```
1. 创建ubuntu启动盘
2. 将群晖的loader拔出，插入刚做好的启动盘
3. 启动ubuntu live
4. sudo -i
5. mdadm -Asf && vgchange -ay
6. mdadm -AU byteorder /dev/md0 /dev/sda1 /dev/sdb1(具体看自己的raid方式)
7. mkfs.ext4 /dev/md0
8. 拔下u盘，插入重新做好的群晖loader
9. 开始重新安装，这样安装后设置和安装的软件丢失，但是数据不会丢失。
```

安装后发现以前使用的sn和mac无效了，估计被封了，主要是使用ds video的硬件转码加速功能，所以还是需要个合法的sn来开启硬件转码功能。这里使用的是复制ddsm中的序列号到grub中，修改grub重启，硬件转码功能正常。以后坚决不做小白鼠！！！