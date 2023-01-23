---
title: disk
date: 2023-01-09 13:27:49
tags:
---

# 磁盘io分析

工具 iostat

## CPU 参数说明

```
%user：CPU处在用户模式下的时间百分比

%nice：CPU处在带NICE值的用户模式下的时间百分比。

%system：CPU处在系统模式下的时间百分比。

%iowait：CPU等待输入输出完成时间的百分比。

%steal：管理程序维护另一个虚拟处理器时，虚拟CPU的无意识等待时间百分比。

%idle：CPU空闲时间百分比。
```

```
如果%iowait的值过高，表示硬盘存在I/O瓶颈

如果%idle值高，表示CPU较空闲

如果%idle值高但系统响应慢时，可能是CPU等待分配内存，应加大内存容量。

如果%idle值持续低于cpu核数，表明CPU处理能力相对较低，系统中最需要解决的资源是CPU。
```

## Device 参数说明

```
tps：该设备每秒的传输次数

kB_read/s：每秒从设备（drive expressed）读取的数据量；

kB_wrtn/s：每秒向设备（drive expressed）写入的数据量；

kB_read：  读取的总数据量；

kB_wrtn：写入的总数量数据量；
```

# 带宽测试

工具： iperf3

