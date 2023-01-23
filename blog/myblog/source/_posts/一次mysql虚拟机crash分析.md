---
title: crash工具使用
date: 2023-01-14 11:13:19
tags:
---

crash是redhat的工程师开发的，主要用来离线分析linux内核转存文件(即 coredump，保存了进程某一时刻的运行状态，它在进程发生问题时产生)，它整合了gdb工具，功能非常强大。可以查看堆栈，dmesg日志，内核数据结构，反汇编等等。crash支持多种工具生成的转存文件格式，如kdump，LKCD，netdump和diskdump，而且还可以分析虚拟机Xen和Kvm上生成的内核转存文件。同时crash还可以调试运行时系统，直接运行crash即可，ubuntu下内核映象存放在/proc/kcore。

# 安装

## centos

```bash
yum install crash
```

## 加载coredump

crash在加载内核转存文件是会输出系统基本信息，如出问题的进程（crond - 21230），系统内存大小（32 GB），系统架构（x86_64）等等，可以看到这个dump是crond触发的一个panic系统崩溃。


```bash
crash /lib/debug/lib/modules/3.10.0-327.36.3.el7.x86_64/vmlinux vmcore
```

![系统基本信息](/pictures/crash-1.png)

## 查看堆栈

一般可以先查看堆栈（bt），看看系统死在什么地方，进而确定调查方向。可以看到 #5 位置打印出很多寄存器的地址，exception RIP表示出问题时候执行的指令。

netlink_compare发生kernel panic（当内核无法正确加载，无法正常启动或崩溃时）并输出以下回溯。

![堆栈图](/pictures/crash-2.png)


查询资料后，定位到这是 `kernel Linux 3.10.0-327.36.3.el7.x86_64` 的bug，[详细描述参见](https://www.dell.com/support/kbdoc/zh-cn/000171350/kernel-panic-in-rhel-7-2-kernel-3-10-0-327-el7-x86-64) 或 https://bugs.centos.org/view.php?id=12012 ，该bug在 7.3 kernel (3.10.0-514.el7)后修复。

## 其他

### kmem

```
kmem -i           查看内存的使用统计信息
kmem -g           查看page flags的定义
kmem -g 0x201     将指定的数字翻译为page的flags
kmem -p <page *>  查看指定page的信息
```

### 基本信息含义

```
KERNEL:    内核崩溃时运行的 Kernel ELF 文件.
DUMPFILE:  内核转储文件.
CPUS:      机器 CPU 的数量.
DATE:      系统崩溃的时间.
UPTIME:    系统启动到系统奔溃的时间.
LOAD AVERAGE:
TASKS:     系统崩溃时内存中的任务数.
NODENAME:  崩溃系统的主机名.
RELEASE:   崩溃内核的版本号.
VERSION:   崩溃内核的版本号.
MACHINE:   CPU 架构和主频信息.
MEMORY:    崩溃主机的物理内存.
PANIC:     崩溃类型.
PID:       导致内核崩溃的进程号.
COMMAND:   导致内核崩溃的命令.
TASK:
CPU:
STATE:
```

### bt
    
```
bt 打印内核奔溃 CPU 上运行 TASK 的堆栈
bt -a 打印内核奔溃时所有 CPU 上 TASK 的堆栈
bt -g
bt -p 只打印发生内核 PANIC CPU 上 TASK 的堆栈
bt -r 打印内核奔溃 CPU 上 TASK 的堆栈内容
bt -t 打印内核崩溃 CPU 上 TASK 的函数调用栈
bt -T 打印内核崩溃 CPU 上 TASK 堆栈中所有的调用函数
bt -l 打印内核崩溃 CPU 上 TASK 堆栈栈帧函数的文件及行号
bt -e 打印内核崩溃 CPU 上 TASK 堆栈中可能异常的帧
bt -E 打印所有 CPU 的中断堆栈
bt -f 打印内核崩溃 CPU 上 TASK 堆栈栈帧的内容
bt -F[F] 打印内核崩溃 CPU 上 TASK 栈帧中 SLAB CACHE 对象信息
bt -o 以老式堆栈方式打印内核崩溃 CPU 上 TASK 的堆栈
bt -O 将 CRASH 堆栈打印模式设置为老式堆栈模式
bt [-R symbol] 显示包含 symbol 的堆栈信息
bt [-I ip]
bt [-S sp] 从 SP 处开始打印内核崩溃 CPU 上 TASK 的堆栈
bt pid 按 PID 打印 TASK 的堆栈
bt taskp 按 TASK 的 task_struct 十六进制值来打印堆栈
bt -c cpu 打印指定 CPU 上 TASK 的堆栈
```

### 进程状态

```
TASK_RUNNING : 进程处于可运行状态，但并不意味着进程已经实际上已分配到 CPU ，它可能会一直等到调度器选中它。该状态只是确保进程一旦被 CPU 选中时立马可以运行，而无需等待外部事件。
TASK_INTERRUPTIBLE : 这是针对等待某事件或其他资源而睡眠的进程设置的。在内核发送信号给该进程时表明等待的事件已经发生或资源已经可用，进程状态变为 TASK_RUNNING，此时只要被调度器选中就立即可恢复运行。
TASK_UNINTERRUPTIBLE : 处于此状态，不能由外部信号唤醒，只能由内核亲自唤醒。
TASK_STOPPED : 表示进程特意停止运行。比如在调试程序时，进程被调试器暂停下来。
TASK_TRACED : 本来不属于进程状态，用于从停止的进程中，将当前被调试的那些进程与常规进程区分开来。
```

下面常量既可以用于 struct task_struct 的进程状态字段，也可以用于 exit_state 字段(该字段明确的用于退出进程)：

```
EXIT_ZOMBIE : 僵尸状态。
EXIT_DEAD : 处于该状态， 表示 wait 系统调用已经发出，而进程完全从系统移除之前的状态。只有多个线程对同一个进程发出 wait 调用时，该状态才有意义(为了防止其他执行线程在同一个进程也执行wait()类系统调用，而把进程的状态由僵死状态(EXIT_ZOMBIE)改为撤销状态(EXIT_DEAD)。
```

### 其他

```
ps           查看系统崩溃时在运行的所有进程
set 3016     查看进程 3016 的状态
bt -f        打印函数栈数据
dmesg        查看系统崩溃时dmesg的信息
```