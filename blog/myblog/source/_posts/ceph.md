---
title: ceph
date: 2022-12-26 20:36:11
tags:
---

# Ceph

## OSD

通常情况下一个OSD对应一块硬盘，每个OSD都拥有一个自己的Daemon。这个Daemon负责完成OSD的所有逻辑功能，包括Monitor和其他OSD（应该说是和其他OSD的Daemon）通信以更新系统状态，与他们OSD共同完成数据存储和维护，与Client通信以完成各种数据对象操作。

RADOS集群从Ceph客户端（三种类型存储客户端 + librados自定义）接收数据并存储为对象。每个对象都是文件系统的一个文件，他们被存储在OSD的存储设备上，由OSD守护进程处理存储设备上的读写操作。

OSD把所有数据存储为对象。对象包含一个标识符、二进制数据和由 `名称/值` 对组成的元数据。不同类型Ceph客户端元数据语义不同。

ino 待操作 File 的元数据，即文件唯一id
ono File 切割后某 Object 的序号
OID （ino，ono）

一个 File 被映射成多个 Object 后，需要把每个 Object 独立的映射到一个 PG 上。

Object 的 size 应当被合理设置，通常为2或4MB
PG也应该足够多，推荐为 OSD 的数百倍。

PG -> OSD 的映射是多对多的，一个 PG 由 CRUSH 算法得到一组 OSD（共n个，n可配置）。  
这n个OSD共同负责存储和维护一个 PG 中所有的 Object。

Cpeh 客户端在读写数据之前必须连个某个 ceph-mon ，获取最新的集群运行图副本。

Mon 不会主动轮询各个OSD的当前状态，OSD会主动上报。MON 收到信息后，会更新 Cluster Map 信息并加以扩散。

PG 与 OSD 之间的映射关系通常是不变的，除非扩容、设备损坏、修改分配策略， CRUSH 会重新分配映射关系。

这些的映射关系，使得一份文件被拆分并散列分步在不同OSD上，不仅有多份文件副本，还极大优化磁盘I/O性能。


## Monitor

Monitor是一个独立部署的daemon进程。通过组成Monitor集群来保证自己的高可用。Monitor集群通过Paxos算法实现了自己数据的一致性。它提供了整个存储系统的节点信息等全局的配置信息。

## 数据平衡

当在集群中新添加一个OSD存储设备时，整个集群会发生数据的迁移，使得数据分布达到均衡。Ceph数据迁移的基本单位是PG，即数据迁移是将PG中的所有对象作为一个整体来迁移。

迁移触发的流程为：当新加入一个OSD时，会改变系统的CRUSH Map，从而引起对象寻址过程中的第二步，PG到OSD列表的映射发生了变化，从而引发数据的迁移。

## Peering

当OSD启动，或者某个OSD失效时，该OSD上的主PG会发起一个Peering的过程。Ceph的Peering过程是指一个PG内的所有副本通过PG日志来达成数据一致的过程。当Peering完成之后，该PG就可以对外提供读写服务了。此时PG的某些对象可能处于数据不一致的状态，其被标记出来，需要恢复。在写操作的过程中，遇到处于不一致的数据对象需要恢复的话，则需要等待，系统优先恢复该对象后，才能继续完成写操作。

## Recovery和Backfill

Ceph的Recovery过程是根据在Peering的过程中产生的、依据PG日志推算出的不一致对象列表来修复其他副本上的数据。
Recovery过程的依据是根据PG日志来推测出不一致的对象加以修复。当某个OSD长时间失效后重新加入集群，它已经无法根据PG日志来修复，就需要执行Backfill(回填)过程。Backfill过程是通过逐一对比两个PG的对象列表来修复。当新加入一个OSD产生了数据迁移，也需要通过Backfill过程来完成。

## Scrub

Scrub机制用于系统检查数据的一致性。它通过在后台定期(默认每天一次)扫描，比较一个PG内的对象分别在其他OSD上的各个副本的元数据和数据来检查是否一致。根据扫描的内容分为两种，第一种是只比较对象各个副本的元数据，它代价比较小，扫描比较高效，对系统影响比较小。另一种扫描称为deep scrub，它需要进一步比较副本的数据内容检查数据是否一致。

## osd的Flags

### noin
通常和noout一起用防止OSD up/down跳来跳去    

### noout  
MON在过了300秒(mon_osd_down_out_interval)后自动将down掉的OSD标记为out，一旦out数据就会开始迁移，建议在处理故障期间设置该标记，避免数据迁移。(故障处理的第一要则设置osd noout 防止数据迁移。ceph osd set noout ,ceph osd unset noout)

### noup
通常和nodwon一起用解决OSD up/down跳来跳去

### nodown
网络问题可能会影响到Ceph进程之间的心跳，有时候OSD进程还在,却被其他OSD一起举报标记为down,导致不必要的损耗，如果确定OSD进程始终正常可以设置nodown标记防止OSD被误标记为down.

### full
如果集群快要满了，你可以预先将其设置为FULL，注意这个设置会停止写操作。(有没有效需要实际测试)

### pause
这个标记会停止一切客户端的读写，但是集群依旧保持正常运行。

### nobackfill

### norebalance
这个标记通常和上面的nobackfill和下面的norecover一起设置，在操作集群(挂掉OSD或者整个节点)时，如果不希望操作过程中数据发生恢复迁移等，可以设置这个标志，记得操作完后unset掉。

### norecover
也是在操作磁盘时防止数据发生恢复。

### noscrub
ceph集群不做osd清理

### nodeep-scrub
有时候在集群恢复时，scrub操作会影响到恢复的性能，和上面的noscrub一起设置来停止scrub。一般不建议打开。

### notieragent
停止tier引擎查找冷数据并下刷到后端存储。

### cpeh osd set {option}
设置所有osd标志

### ceph osd unset {option}
接触所有osd标志

### 使用下面的命令去修复pg和osd

```
ceph osd repair ：修复一个特定的osd
ceph pg repair 修复一个特定的pg，可能会影响用户的数据，请谨慎使用。
ceph pg scrub：在指定的pg上做处理
ceph deep-scrub:在指定的pg上做深度清理。
ceph osd set pause 搬移机房时可以暂时停止读写，等待客户端读写完毕了，就可以关闭集群
```

## PG的States

### Creating
当创建一个池的时候，Ceph会创建一些PG(通俗点说就是在OSD上建目录)，处于创建中的PG就被标记为creating，当创建完之后，那些处于Acting集合(ceph pg map 1.0 osdmap e9395 pg 1.0 (1.0) -> up [27,4,10] acting [27,4,10]，对于pg它的三副本会分布在osd.27,osd.4,osd.10上，那么这三个OSD上的pg1.0就会发生沟通，确保状态一致)的PG就会进行peer(osd互联)，当peering完成后，也就是这个PG的三副本状态一致后，这个PG就会变成active+clean状态，也就意味着客户端可以进行写入操作了。

### Peering
peer过程实际上就是让三个保存同一个PG副本的OSD对保存在各自OSD上的对象状态和元数据进行协商的过程，但是呢peer完成并不意味着每个副本都保存着最新的数据。直到OSD的副本都完成写操作，Ceph才会通知客户端写操作完成。这确保了Acting集合中至少有一个副本，自最后一次成功的peer后。剩下的不好翻译因为没怎么理解。（对象和元数据的状态达成一致的过程。）

### Active
当PG完成了Peer之后，就会成为active状态，这个状态意味着主从OSD的该PG都可以提供读写了。

### Clean
这个状态的意思就是主从OSD已经成功peer并且没有滞后的副本。PG的正常副本数满足集群副本数。

### Degraded
当客户端向一个主OSD写入一个对象时，主OSD负责向从OSD写剩下的副本， 在主OSD写完后,在从OSD向主OSD发送ack之前，这个PG均会处于降级状态。而PG处于active+degraded状态是因为一个OSD处于active状态但是这个OSD上的PG并没有保存所有的对象。当一个OSDdown了，Ceph会将这个OSD上的PG都标记为降级。当这个挂掉的OSD重新上线之后，OSD们必须重新peer。然后，客户端还是可以向一个active+degraded的PG写入的。当OSDdown掉五分钟后，集群会自动将这个OSD标为out,然后将缺少的PGremap到其他OSD上进行恢复以保证副本充足，这个五分钟的配置项是mon osd down out interval，默认值为300s。
PG如果丢了对象，Ceph也会将其标记为降级。你可以继续访问没丢的对象，但是不能读写已经丢失的对象了。假设有9个OSD，三副本，然后osd.8挂了，在osd.8上的PG都会被标记为降级，如果osd.8不再加回到集群那么集群就会自动恢复出那个OSD上的数据，在这个场景中，PG是降级的然后恢复完后就会变成active状态。

### Recovering
Ceph设计之初就考虑到了容错性，比如软硬件的错误。当一个OSD挂了，它所包含的副本内容将会落后于其他副本，当这个OSD起来之后，这个OSD的数据将会更新到当前最新的状态。这段时间，这个OSD上的PG就会被标记为recover。而recover是不容忽视的，因为有时候一个小的硬件故障可能会导致多个OSD发生一连串的问题。比如，如果一个机架或者机柜的路由挂了，会导致一大批OSD数据滞后，每个OSD在故障解决重新上线后都需要进行recover。Ceph提供了一些配置项，用来解决客户端请求和数据恢复的请求优先级问题，这些配置参考上面加粗的字体吧。

### Backfilling
当一个新的OSD加入到集群后，CRUSH会重新规划PG将其他OSD上的部分PG迁移到这个新增的PG上。如果强制要求新OSD接受所有的PG迁入要求会极大的增加该OSD的负载。回填这个OSD允许进程在后端执行。一旦回填完成后，新的OSD将会承接IO请求。在回填过程中，你可能会看到如下状态：

```
backfill_wait: 表明回填动作被挂起，并没有执行。

backfill：表明回填动作正在执行。

backfill_too_full：表明当OSD收到回填请求时，由于OSD已经满了不能再回填PG了。 

imcomplete: 当一个PG不能被回填时，这个PG会被认为是不完整的。同样，Ceph提供了一系列的参数来限制回填动作，包括osd_max_backfills：OSD最大回填PG数。

sd_backfill_full_ratio：当OSD容量达到默认的85%是拒绝回填请求。osd_backfill_retry_interval:字面意思。
Remmapped   当Acting集合里面的PG组合发生变化时，数据从旧的集合迁移到新的集合中。这段时间可能比较久，新集合的主OSD在迁移完之前不能响应请求。所以新主OSD会要求旧主OSD继续服务指导PG迁移完成。一旦数据迁移完成，新主OSD就会生效接受请求。
```

### Stale
Ceph使用心跳来确保主机和进程都在运行，OSD进程如果不能周期性的发送心跳包，那么PG就会变成stuck状态。默认情况下，OSD每半秒钟汇报一次PG，up thru,boot, failure statistics等信息，要比心跳包更会频繁一点。如果主OSD不能汇报给MON或者其他OSD汇报主OSD挂了，Monitor会将主OSD上的PG标记为stale。当启动集群后，直到peer过程完成，PG都会处于stale状态。而当集群运行了一段时间后，如果PG卡在stale状态，说明主OSD上的PG挂了或者不能给MON发送信息。

### Misplaced
有一些回填的场景：PG被临时映射到一个OSD上。而这种情况实际上不应太久，PG可能仍然处于临时位置而不是正确的位置。这种情况下个PG就是misplaced。这是因为正确的副本数存在但是有个别副本保存在错误的位置上。

### Incomplete
当一个PG被标记为incomplete,说明这个PG内容不完整或者peer失败，比如没有一个完整的OSD用来恢复数据了。

### scrubbing
清理中，pg正在做不一致性校验。

### inconsistent
不一致的，pg的副本出现不一致。比如说对象的大小不一样了。


## 卡住的pg状态： 
```
Unclean: 归置组里有些对象的副本数未达到期望次数，它们应该在恢复中； 
Inactive: 归置组不能处理读写请求，因为它们在等着一个持有最新数据的OSD 回到up 状态； 
Stale: 归置组们处于一种未知状态，因为存储它们的OSD 有一阵子没向监视器报告了（由mon osd report timeout 配置） 
为找出卡住的归置组，执行：
ceph pg dump_stuck [unclean|inactive|stale|undersized|degraded]
```
