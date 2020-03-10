# ceph内核模块编译及调试

虽然librbd和librados可以满足大部分ceph的使用需求，但是在实际使用中（特别是ceph与kubernetes结合），仍需krbd模块。当然，rbd-nbd也是一种解决方案，在这不多说。显然，krbd依赖linux的内核版本，而普遍地，生产环境下的系统很少会升级到最新的内核（毕竟稳定性还是第一位）。这就会造成krbd的代码跟不上社区版本，有些bug即使修复了也难以应用起来。

因此，本文基于centos7的发行版，简单介绍如何使用ceph进行修改模块的编译和问题调式定位。同时，如果使用过centos7.0的朋友们应该知道，在kernel-3.10.0-123版本，rbd模块默认是不安装的，即只能通过模块编译加载的方式来使用ceph的rbd，否则执行rbd map操作是会失败的。




## linux模块编译
linux的模块编译在网上很多地方都能够找到，这里不描述详细细节，只说一下需要注意的事情。

1. 如果不打算升级内核，那么模块编译时的kernel源码一定要与当前发行版本的内核一致。比如，centos7.0对应内核版本为kernel-3.10.0-123，如果不确定，可以使用`uname -r`来查询到（针对rhel系的系统）。一般地，内核源码可以在rpmsearch中找到。千万不要将发行版的linux内核和torvalds/linux搞混了。
2. 手动编译的内核模块一般是没有签名的（module signing），一般情况是不影响使用的。但如果有特殊需求的环境，比如需要验证签名才能加载内核模块的系统安全设定，则需要手动为模块增加签名。详细的可以参考红帽的官方说明：[signing secure module for secure boot](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/sect-signing-kernel-modules-for-secure-boot)
3. 当内核源码包缺少symvers文件，手动编译的modules在加载时报`no symbol version for module_layout`错误.此时，需要做如下操作: 1. `locate symvers`, 2. 将symvers文件拷贝到对应-C目录下


那么，我们假设已经从网上或者其它途径上拿到了linux某个发行版本的内核源码（这里使用的是centos7.1 即 kernel-3.10.0-229.el7.src.rpm或者kernel.tar.bz包)。如果是src.rpm包，则还需要`yum install xxx.src.rpm`进行安装，成功后在/root目录下会多出一个rpmbuild目录，该源码包位置在/root/rpmbuild/SOURCES/linux-3.10.0-514.el7.tar.xz。

如果要编译整个内核，可以这样：
1. 通过解压tar包，获取源码`tar xvf kernel.tar.xz`
2. `make mrproper` # 清除上次残留项
3. `make menuconfig` # 需要安装ncurses 和 ncurses-devel库，选择内核需要的模块
4. `make & make modules_install`
5. 在对应的模块目录下找到特定的ko文件

静静等待几个小时编译即可完成。幸运的是，内核模块是可以单独编译的。通过以下步骤，可以编译某个模块，而不用等待所有代码编译完成，非常节省时间：
1. `make oldconfig`
2. `make prepare`
3. `make scripts`
4. `make <MODULE>=m -C /usr/src/kernels/{linux-version}/  M=/{src-dir}/{modules_path}/ modules` -C \$(KDIR) 指明跳转到内核源码目录下读取那里的Makefile，M=\$(PWD) 表明然后返回到当前目录继续读入、执行当前的Makefile
5. 然后再{module_path}路径中找到对应的ko文件即可



## ceph模块编译
ceph的内核模块主要有三个，包含rbd-nbd的话就有四个，其路径及模块名分别是：
1. cephfs内核模块ceph.ko：CONFIG_CEPH_FS=m，模块路径：./fs/ceph/
2. rbd内核模块rbd.ko：CONFIG_BLK_DEV_RBD=m，模块路径：./drivers/block
3. 通用模块libceph.ko：CONFIG_CEPH_LIB=m，模块路径：./net/ceph/ ./include/linux/ceph/ ./include/linux/crush/
4. nbd模块nbd.ko: CONFIG_BLK_DEV_NBD=m，模块路径：./drivers/block

以rbd内核模块编译为例子，其步骤如下：
1. `make CONFIG_BLK_DEV_RBD=m -C /usr/src/kernels/3.10.0-229.el7.x86_64/  M=/root/linux-3.10.0-229/ modules`
2. 在`/root/linux-3.10.0-229/drivers/block`目录下找到rbd.ko文件
3. 放置到`/usr/lib/modules/3.10.0-229.el7.x86_64/kernel/drivers/block/rbd.ko`位置。
4. `depmod -a` 更新所有模块依赖，也可以执行 `rmmod rbd`,然后`modprobe rbd` 或者 `insmod rbd`

**PS:** cephfs放置到`/usr/lib/modules/3.10.0-229.el7.x86_64/kernel/fs/ceph/ceph.ko`， libceph放置到`/usr/lib/modules/3.10.0-229.el7.x86_64/kernel/net/ceph/libceph.ko`




## ceph内核模块调试
ceph的内核调试其实就是linux的内核日志输出调试吧。其所有的输出一般会打印在kern.log上，使用内置的debugfs和dynamic_debug。dynamic_debug允许用户手动打开/关闭内核输出信息，更详细的可自行参考相关资料（因为我不是内核开发，对其调试工具不太熟悉，就不班门弄斧了）。

如果不打开内核的printk等级，那么我们默认在/var/log/message中只会看到少量的消息，比如`rbd map`时，其消息如下：
```
Mar 26 21:45:56 devp kernel: Key type dns_resolver registered
Mar 26 21:45:56 devp kernel: Key type ceph registered
Mar 26 21:45:56 devp kernel: libceph: loaded (mon/osd proto 15/24)
Mar 26 21:45:56 devp kernel: rbd: loaded (major 252)
Mar 26 21:45:56 devp kernel: libceph: mon2 127.0.0.1:40693 session established
Mar 26 21:45:56 devp kernel: libceph: client24119 fsid e91b954b-97e7-444e-8da2-a18be2d5c74d
Mar 26 21:45:56 devp kernel: rbd: rbd0: capacity 1073741824 features 0x1
```

然而这些消息并不能够准确追踪具体的运行情况，特别是在需要调试或者定位bug的时候。因此，需要将kernel中printk的信日志打印出来。如下步骤所示：

1. 确保系统可以使用dynamic_debug功能：```sudo cat /boot/config-`uname -r` | grep DYNAMIC_DEBUG```，其执行结果为：CONFIG_DYNAMIC_DEBUG=y表明功能可以使用

2. 挂载debugfs, 并打开想要输出日志的模块
```
mount -o rw.remount -t debugfs none /sys/kernel/debug/
echo "module libceph +p" >/sys/kernel/debug/dynamic_debug/control
echo "module rbd +p" >/sys/kernel/debug/dynamic_debug/control
```

3. 通过`echo "7 7 7 7" >  /proc/sys/kernel/printk` 设置内核打印日志级别 0~7， 默认 4 4 1 7。其级别消息如下所示：
```
#define KERN_EMERG 0    /*紧急事件消息，系统崩溃之前提示，表示系统不可用*/
#define KERN_ALERT 1    /*报告消息，表示必须立即采取措施*/
#define KERN_CRIT 2     /*临界条件，通常涉及严重的硬件或软件操作失败*/
#define KERN_ERR 3      /*错误条件，驱动程序常用KERN_ERR来报告硬件的错误*/
#define KERN_WARNING 4  /*警告条件，对可能出现问题的情况进行警告*/
#define KERN_NOTICE 5   /*正常但又重要的条件，用于提醒。常用于与安全相关的消息*/
#define KERN_INFO 6     /*提示信息，如驱动程序启动时，打印硬件信息*/
#define KERN_DEBUG" 7   /*调试级别的消息*/
```
如果没有效果，可以使用 `echo 9 > /proc/sysrq-trigger` 的方式，sysrq-trigger的使用是比较极端的，但有时也挺好用，需要谨慎使用，因为其会直接触发内核系统的修改。其具体参数如下([rhel_sysrq-trigger](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/4/html/Reference_Guide/s3-proc-sys-kernel.html))：
```
• r — Disables raw mode for the keyboard and sets it to XLATE (a limited keyboard mode which does not recognize modifiers such as Alt, Ctrl, or Shift for all keys).
• k — Kills all processes active in a virtual console. Also called Secure Access Key (SAK), it is often used to verify that the login prompt is spawned from init and not a trojan copy designed to capture usernames and passwords.
• b — Reboots the kernel without first unmounting file systems or syncing disks attached to the system.
• c — Crashes the system without first unmounting file systems or syncing disks attached to the system.
• o — Shuts off the system.
• s — Attempts to sync disks attached to the system.
• u — Attempts to unmount and remount all file systems as read-only.
• p — Outputs all flags and registers to the console.
• t — Outputs a list of processes to the console.
• m — Outputs memory statistics to the console.
• 0 through 9 — Sets the log level for the console.
• e — Kills all processes except init using SIGTERM.
• i — Kills all processes except init using SIGKILL.
• l — Kills all processes using SIGKILL (including init). The system is unusable after issuing this System Request Key code.
• h — Displays help text.
```

4. 配置rsyslog，在/etc/rsyslog.conf 中设置 `kern.* /var/log/kern.log`， 然后重启rsyslog: `systemctl restart rsyslog`

更加详细的dynamic-debug的信息，可以参考：[dynami-debug-howto文档](https://www.kernel.org/doc/html/v4.14/admin-guide/dynamic-debug-howto.html)。

至此，我们可以从/var/log/kern.log中获取到libceph和rbd模块的所有输出，如下所示：
```
Mar 26 22:01:59 devp kernel: libceph:  init
Mar 26 22:01:59 devp kernel: libceph:  msgpool osd_op init
Mar 26 22:01:59 devp kernel: libceph:  ceph_msg_new ffff88007dd080f0 front 4096
Mar 26 22:01:59 devp kernel: libceph:  msgpool_alloc (null) ffff88007dd080f0
Mar 26 22:01:59 devp kernel: libceph:  ceph_msg_new ffff88007dd0b660 front 4096
Mar 26 22:01:59 devp kernel: libceph:  msgpool_alloc (null) ffff88007dd0b660
Mar 26 22:01:59 devp kernel: libceph:  ceph_msg_new ffff88007dd0b570 front 4096
Mar 26 22:01:59 devp kernel: libceph:  msgpool_alloc (null) ffff88007dd0b570
Mar 26 22:01:59 devp kernel: libceph:  ceph_msg_new ffff88007dd0b480 front 4096
Mar 26 22:01:59 devp kernel: libceph:  msgpool_alloc (null) ffff88007dd0b480
Mar 26 22:01:59 devp kernel: libceph:  ceph_msg_new ffff88007dd09d10 front 4096
Mar 26 22:01:59 devp kernel: libceph:  msgpool_alloc (null) ffff88007dd09d10
Mar 26 22:01:59 devp kernel: libceph:  ceph_msg_new ffff88007dd0b390 front 4096
Mar 26 22:01:59 devp kernel: libceph:  msgpool_alloc (null) ffff88007dd0b390
Mar 26 22:01:59 devp kernel: libceph:  ceph_msg_new ffff88007dd08e10 front 4096
Mar 26 22:01:59 devp kernel: libceph:  msgpool_alloc (null) ffff88007dd08e10
```



## 参考文档链接

[https://ceph.com/geen-categorie/ceph-collect-kernel-rbd-logs/](https://ceph.com/geen-categorie/ceph-collect-kernel-rbd-logs/)
[https://www.kernel.org/doc/html/v4.14/admin-guide/dynamic-debug-howto.html](https://www.kernel.org/doc/html/v4.14/admin-guide/dynamic-debug-howto.html)
[http://www.selinuxplus.com/?p=540](http://www.selinuxplus.com/?p=540)
[https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/sect-signing-kernel-modules-for-secure-boot](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/sect-signing-kernel-modules-for-secure-boot)