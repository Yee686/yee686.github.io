---
title: NVMe规范与技术细节
tag: [NVMe]
index_img: /img/nvme.png
date: 2024-12-05 14:00:00
updated: 2024-12-06 16:00:00
category: 技术调研
---

# NVMe技术细节

- [NVMe技术细节](#nvme技术细节)
  - [1 NVMe技术概述](#1-nvme技术概述)
  - [2 队列管理 Queue Manage](#2-队列管理-queue-manage)
  - [3 命令仲裁机制 Arbitration](#3-命令仲裁机制-arbitration)
  - [4 寻址模型PRP和SGL解析](#4-寻址模型prp和sgl解析)
  - [mq-deadline调度器原理](#mq-deadline调度器原理)

参考自*编程随笔NVMe专题*

## 1 [NVMe技术概述](https://mp.weixin.qq.com/s?__biz=MzIwNTUxNDgwNg==&mid=2247484348&idx=1&sn=1fd3356c6cd9fee9492dcc7d6eb30345&chksm=972ef2e5a0597bf3190c7e5d7717e25ef3a5d83a1b6d7a20f8c4adb1beafc4372f1d79288dd7&scene=21#wechat_redirect)

- AHCI: Serial ATA Advanced Host Controller Interface, 串行ATA高级主机主控接口
- AHCI, NVMe, SATA, PCIe关系, 设计之初NVMe主要服务于PCIe SSD
  - ![2024-12-04-20-18-33-NVMe技术](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-12-04-20-18-33-NVMe技术.png)
- NVMe的特点与优势
  - ![2024-12-04-20-20-32-NVMe技术](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-12-04-20-20-32-NVMe技术.png)
  - 高速, IO延迟低(PCIe不需要和SATA一样连接到南桥中转, 直达Root Complex)
  - 理论上IOPS与队列深度(Queue Depth)和IO延迟有关, 用数学表达式是 $IOPS=队列深度/IO延迟$, NVMe支持64K个深度为64K的队列, 所以IOPS高

## 2 [队列管理 Queue Manage](https://mp.weixin.qq.com/s?__biz=MzIwNTUxNDgwNg==&mid=2247484355&idx=1&sn=04f0617bf774fa3c6020d90288b679e8&chksm=972ef29aa0597b8ca79b040f3222eef85835a5cd693167aa6f7ffd34a78ae15f696d7b736304&scene=21#wechat_redirect)

- NVMe命令种类
  - ![2024-12-04-20-34-34-NVMe技术](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-12-04-20-34-34-NVMe技术.png)
  - > 当Host要下发Admin command时,需要一个放置Admin command的队列, 这个队列就叫做Admin Submission Queue, 简称Admin SQ.
  - > Device执行完成Admin command时,会生成一个对应的Completion回应, 此时也需要一个放置Completion的队列, 这个队列就叫做Admin Completion Queue, 简称Admin CQ.
  - > 同样, 执行IO Command时, 也会有对应的两个队列, 分别是IO SQ和IO CQ.
  - Admin SQ/CQ放Admin管理命令, 管理SSD, 队列2~4K
  - IO SQ/CQ放IO命令,完成数据传输, 队列2~16K
  - 系统中只有一对Admin SQ/CQ, 可以有最多64K对 IO SQ/CQ, SQ命令条目大小为64K, CQ命令完成状态条目大小16K, IO SQ/CQ可以一对多也可以一对一
  - Admin和IO的SQ/CQ均放在Host端Memory中, SQ由Host来更新，CQ则由NVMe Controller更新
  - Doorbell Register/Pointer Register
  - ![2024-12-04-21-05-14-NVMe技术](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-12-04-21-05-14-NVMe技术.png)
  - 由于SQ/CQ均位于控制器内存, Controller把SQ Head和CQ Tail的信息写入了Completion报文, 让主机看到工作状态

## 3 [命令仲裁机制](https://mp.weixin.qq.com/s?__biz=MzIwNTUxNDgwNg==&mid=2247484375&idx=1&sn=d7854b5dddd0407a24753b9da176e40a&chksm=972ef28ea0597b98c60ddf0e2f62ff80be42a60e5c2a0ae9429e51644bd3b74152e3dd4ec684&scene=21#wechat_redirect) Arbitration

- NVMe Spec没有规定命令放入SQ的执行顺序, Controller可以一次取多个命令批量处理. 一个SQ中命令执行顺序不是固定的, 多个SQ间的顺序也不是固定的, 涉及命令仲裁机制
  - 循环仲裁: 所有Admin SQ和IO SQ优先级一样, 按序从所有SQ中取出一定数目(通过Set Feature定义)命令
  - ![2024-12-05-13-33-47-NVMe技术](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-12-05-13-33-47-NVMe技术.png)
  - 加权循环仲裁: 三个绝对优先级和三个加权优先级
    - `Admin Class`: 最高优先级, 必须最先执行, Admin SQ为该优先级
    - `Urgent Class`: 该类IO SQ必须在Admin SQ之后执行
    - `WRR Class`: 最低绝对优先级, 有High, Midium, Low三个加权优先级
  - ![2024-12-05-13-36-28-NVMe技术](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-12-05-13-36-28-NVMe技术.png)
  - ![2024-12-05-13-44-36-NVMe技术](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-12-05-13-44-36-NVMe技术.png)
  - 先3给weight=3的, 再2个weight=2的, 然后1个weight=1的

## 4 [寻址模型PRP和SGL解析](https://mp.weixin.qq.com/s?__biz=MzIwNTUxNDgwNg==&mid=2247484376&idx=1&sn=61d9b52a7f7b9205b8df86d710a56f87&chksm=972ef281a0597b97bf85f1e6af38031021d8e2829aca808e1c7b989ad88a9aaacd9398ca434c&scene=21#wechat_redirect)

> 当Host下发NVMe Write命令时, Host会先放数据放在Host内存中, 然后通知Controller过来取数据. Controller接到信息后, 会通过PCIe Memory Read TLP读取相应的数据, 接着Host返回的PCIe Completion报文中会携带数据给Controller, 最后再写入NAND中.
> 当Host下发NVMe Read命令时, Controller先从NAND中读出相应数据, 然后通过PCIe Memory Write TLP将数据写入Host内存中.

![2024-12-05-13-56-09-NVMe技术](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-12-05-13-56-09-NVMe技术.png)

- NVMe Controller通过PRP和SGL这两种模型获取数据再host内存中的地址
  - **PRP(Physical Region Page)**, 记录host内存中物理页位置. PRP entry是一个64位的结构, 高[64:n]位是物理页的基地址, 低[n:0]位是页内偏移, n由物理页大小决定
  - NVMe Command中定义了两个PRP Entry, 第二个Entry偏移为0, 若需要传输的数据超过了两个物理页(即2个PRP entry不够), 则第二个Entry指向PRP list, list中偏移为0
  - **SGL(Scatter Gather List)**, 再NVMe over PCIe中可用于IO命令, 不能用于Admin命令. SGL主要用于NVMe over Fabric
  > 一个SGL包含多个SGL Segment. 一个SGL Segment包含多个SGL Discriptor. 其中, SGL Segment要求Qword对齐. 另外, 倒数第二个SGL Segment中的最后一个Discriptor有一个特殊的名字, 叫做SGL Last Segment Discriptor
  ![2024-12-05-14-13-29-NVMe技术](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-12-05-14-13-29-NVMe技术.png)
  - SGL Bit Bucket Descriptor: 为Host服务, 告知Controller哪些数据是不必要写入Host内存
  - Keyed SGL Data Block Descriptor: 代表了数据传输中带有密匙
  > PRP和SGL是描述Host内存物理空间的两种方式, 本质的不同是：PRP必须是物理页对齐的, 而SGL则可以表述任意的物理空间
  ![2024-12-05-14-19-35-NVMe技术](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-12-05-14-19-35-NVMe技术.png)
  ![2024-12-05-14-23-28-NVMe技术](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-12-05-14-23-28-NVMe技术.png)

## mq-deadline调度器原理

- [源码链接](https://github.com/torvalds/linux/blob/master/block/mq-deadline.c)
- 适配block层多队列机制, 将IO分为read和write两类, 每类IO都有一个红黑树和fifo队列, 红黑树用于将IO按照LBA排序方便查找, fifo队列用于顺序保证, 并提供超时保障. 穿透性IO发送至dispatch队列

``` c++
    struct dd_per_prio {
        struct list_head dispatch;
        struct rb_root sort_list[DD_DIR_COUNT];
        struct list_head fifo_list[DD_DIR_COUNT];
        /* Position of the most recently dispatched request. */
        sector_t latest_pos[DD_DIR_COUNT];
        struct io_stats_per_prio stats;
    };   

    struct deadline_data {
        /*
        * run time data
        */

        struct dd_per_prio per_prio[DD_PRIO_COUNT];

        /* Data direction of latest dispatched request. */
        enum dd_data_dir last_dir;
        unsigned int batching;		/* number of sequential requests made */
        unsigned int starved;		/* times reads have starved writes */

        /*
        * settings that change how the i/o scheduler behaves
        */
        int fifo_expire[DD_DIR_COUNT];
        int fifo_batch;
        int writes_starved;
        int front_merges;
        u32 async_depth;
        int prio_aging_expire;

        spinlock_t lock;
    };
```

- 一个块设备对应一个deadline_data, read可以抢占write的分发, 当达到饥饿starved限制时必须处理write.mq-deadline调度器会优先去批量式地分发IO而不去管IO的到期时间, 当批量分发到一定的个数再关心到期时间, 然后去分发即将到期的IO.[mq-deadline调度器原理及源码分析](https://www.cnblogs.com/kanie/p/15252921.html)
- ![2024-12-06-15-23-16-NVMe技术](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-12-06-15-23-16-NVMe技术.png)