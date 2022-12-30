---
title: vmware串口打印dmesg
date: 2022-10-28
tags:
    - linux
    - kernel
---


## 1. 背景

对于linux内核开发人员来说，系统崩溃是一座绕不开的大山，每一次的系统崩溃，都是对我们的一次沉重打击，因为我们代码有bug，这时linux内核会给我提供一些信息，那就是会把一些崩溃信息，如寄存器信息、调用栈信息等打印到`dmesg`中，我们通过`dmesg`可以查看这些信息，定位代码哪里出了问题，不过当系统崩溃时我们无法在执行操作，去查看`dmesg`信息了，这时就需要通过串口将信息发送出来，下面我将介绍一种方法，可以在系统崩溃时通过窗口将信息发送出来。

## 2. 配置信息

首先在虚拟机的设置中添加串口（虚拟机需要关机），在`Use output file`中填写输出文件的路径
![vmware_machine_setting](/images/vmware_machine_setting.png)

之后开机，进行系统，打开文件`/etc/default/grub`，找到`GRUB_CMDLINE_LINUX_DEFAULT`选型行，在后面添加上`console=tty1 console=ttyS1`,

```conf
...
GRUB_CMDLINE_LINUX_DEFAULT="splash quiet"
...
```

```conf
...
GRUB_CMDLINE_LINUX_DEFAULT="splash quiet console=tty1 console=ttyS1"
...
```

这里正常情况下一般是`ttyS1`，具体是多少，需要读者实际测试一下，测试方法也很简单，

在宿主机中执行`tail -f file.txt`，这里的`file.txt`就是前面在配置虚拟机时填写的文件路径，之后在虚拟机中执行`echo "test" > /dev/ttyS1`，
如果宿主机中`file.txt`文件有输出，表示参数没有问题，如果没有输出，则执行`echo "test" > /dev/ttyS2`，依次下去，直到有输出为止，这时的 `ttySx`就是咱们想要的参数了。

在修改完`/etc/default/grub`文件后，还需要执行`update-grub`，更新`grub`的命令行参数，之后重启系统。

重启系统时，我们就可以在宿主机中发现`file.txt`有输出了，
如果文件没有输出，可以查看一下咱们的修改是否生效了，`cat /proc/cmdline`，在其输出中是否有咱们添加的信息，如果没有相应的信息，可能是读者忘记执行`update-grub`了。
即使有信息输出，这时的输出也是不完整的，咱们在系统开启后调整输出级别，执行`echo "8 4 1 7" > /proc/sys/kernel/printk`，至于为什么这么执行，后面我再详细介绍。
到这里咱们就可以正常调试了，如果系统发生崩溃，`file.txt`都会记录。

## 3. 验证配置

前面的配置后配置后，开机进入系统，执行如下三个命令

```bash
echo "8 4 1 7" > /proc/sys/kernel/printk
echo 1 > /proc/sys/kernel/sysrq
echo c > /proc/sysrq-trigger
```

在最后一个命令执行完，你的系统应该就崩溃了，并且在`file.txt`会看到类似如下输出：

```txt
[  106.683226][ 0]  sysrq: Trigger a crash
[  106.685320][ 0]  Kernel panic - not syncing: sysrq triggered crash
[  106.688088][ 0]  CPU: 0 PID: 2134 Comm: bash Not tainted 5.4.18-22-generic #8b6-KYLINOS
[  106.691843][ 0]  Hardware name: VMware, Inc. VMware Virtual Platform/440BX Desktop Reference Platform, BIOS 6.00 11/12/2020
[  106.696559][ 0]  Call Trace:
[  106.697856][ 0]   dump_stack+0x6d/0x9a
[  106.699482][ 0]   panic+0x101/0x2e3
[  106.701013][ 0]   sysrq_handle_crash+0x15/0x20
[  106.703054][ 0]   __handle_sysrq.cold+0xbf/0x11c
[  106.705166][ 0]   write_sysrq_trigger+0x2b/0x40
[  106.708102][ 0]   proc_reg_write+0x43/0x70
[  106.709964][ 0]   __vfs_write+0x1b/0x40
[  106.711824][ 0]   vfs_write+0xb9/0x1a0
[  106.713444][ 0]   ksys_write+0x67/0xe0
[  106.714882][ 0]   __x64_sys_write+0x1a/0x20
[  106.716438][ 0]   do_syscall_64+0x57/0x190
[  106.718295][ 0]   entry_SYSCALL_64_after_hwframe+0x44/0xa9
[  106.720052][ 0]  RIP: 0033:0x7f8e040831e7
[  106.721377][ 0]  Code: 64 89 02 48 c7 c0 ff ff ff ff eb bb 0f 1f 80 00 00 00 00 f3 0f 1e fa 64 8b 04 25 18 00 00 00 85 c0 75 10 b8 01 00 00 00 0f 05 <48> 3d 00 f0 ff ff 77 51 c3 48 83 ec 28 48 89 54 24 18 48 89 74 24
[  106.727815][ 0]  RSP: 002b:00007fffef987148 EFLAGS: 00000246 ORIG_RAX: 0000000000000001
[  106.730479][ 0]  RAX: ffffffffffffffda RBX: 0000000000000002 RCX: 00007f8e040831e7
[  106.732844][ 0]  RDX: 0000000000000002 RSI: 0000559a7a3ff0f0 RDI: 0000000000000001
[  106.735253][ 0]  RBP: 0000559a7a3ff0f0 R08: 000000000000000a R09: 0000000000000001
[  106.737676][ 0]  R10: 0000559a79224017 R11: 0000000000000246 R12: 0000000000000002
[  106.740172][ 0]  R13: 00007f8e0415e6a0 R14: 00007f8e0415f4a0 R15: 00007f8e0415e8a0
[  106.747387][ 0]  Kernel Offset: 0x1a00000 from 0xffffffff81000000 (relocation range: 0xffffffff80000000-0xffffffffbfffffff)
[  106.751256][ 0]  ---[ end Kernel panic - not syncing: sysrq triggered crash ]---
```

到这里结束。
