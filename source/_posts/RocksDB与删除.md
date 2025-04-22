---
title: 频删场景下键值存储的性能优化
tag: [RocksDB, Compaction, Flush, KV store, LSM-tree, ZNS SSD]
index_img: https://raw.githubusercontent.com/Yee686/Picbed/main/2025-03-31-20-25-15-RocksDB-Delete.png
date: 2025-03-31 20:50:00
updated: 2025-03-31 20:50:00
category: 文献笔记
math: true
---

# RocksDB Delete Heavy Optimization

## RocksDB及其上层应用

### RocksDB -- Single Delete

- Single Delete会当删除墓碑与对应值在合并操作对齐时同时删除, 减少中间层删除墓碑残留
- 然而该机制仅适用于删除键只被写入一次且无后续更新的特定场景, 需要用户明确键的写入序列及其生命周期, 并通过编码保证安全, 如果键被多次写入后调用Single Delete会发生内部错误, 所以该优化解决的问题十分有限。

![范围删除标记在SSTable中的组织方式](https://raw.githubusercontent.com/Yee686/Picbed/main/2025-03-31-20-08-52-RocksDB-Delete.png)

![逻辑范围删除墓碑(左)及其对应物理键值对表示(右)](https://raw.githubusercontent.com/Yee686/Picbed/main/2025-03-31-20-09-00-RocksDB-Delete.png)

![2025-03-31-20-25-15-RocksDB-Delete](https://raw.githubusercontent.com/Yee686/Picbed/main/2025-03-31-20-25-15-RocksDB-Delete.png)

### RocksDB -- Delete Range

- 前台写时, 范围删除会被写入专门的Range Delete Memtable, 在刷盘时再与对应的文件一起刷入单个SSTable, 原因是为了防止墓碑与数据交错
- SSTable中将范围删除墓碑写入专门的Range Tombstone Block
- 读数据路径
  - V1版本
    - 搜索路径上的Memtable、Immutable Memtable和SSTablez中所有的范围删除标记聚合到名为skyline的无序向量中, 再通过扫描确定待读的键是否已经被删除。
    - 虽然构建skyline的开销较高, 但可以快速确定某个键是否被覆盖
  - V2版本
    - 不再通过聚合删除墓碑生成全局skyline, 而直接在每个SSTable和Memtable中分段保存, 保证分段无删除墓碑的重叠并且有序, 使得可用二分查找代替顺序查找
    - 每个打开一个SSTable时本身的范围删除墓碑会被分段并缓存：对于点查询会在每个分段中二分查找, 找到之后便不在更底层搜索, 相对于skyline方式减少了对删除无关层的读操作；对于范围扫描, 将所有分段的迭代器保存在列表中隐式创建skyline

![MyRocks索引设计](https://raw.githubusercontent.com/Yee686/Picbed/main/2025-03-31-20-31-31-RocksDB-Delete.png)

![MyRocks优化后性能评估](https://raw.githubusercontent.com/Yee686/Picbed/main/2025-03-31-20-31-37-RocksDB-Delete.png)

### MyRocks -- 针对Single Delete进行设计

- MyRocks为了支持Single Delete引入了二级键避免对同一RocksDB Key的多次写操作, 当二级键没发生变化则不更新, 当二级键更新时线通过Single Delete删除旧二级键, 再Put写入新二级键。
- MyRocks引入了删除触发合并DTC, MyRocks扩展了RocksDB的合并机制以跟踪相邻删除墓碑, 当刷盘或合并过程中检测到某个键范围内删除墓碑的目睹较高, 会立即触发一次合并操作, 以减少因扫描过多删除墓碑导致范围扫描性能波动。

## X-Engine -- 基于规则加速删除墓碑清理

- X-Engine将合并的优先级定义为: 用于删除的合并、Level0层合并、小合并(合并除最大级别外的两个相邻层)、大合并(合并最大级别和上一级别)、自我大合并(最大层内进行合并以减少已删除记录)
- 当一个级别的空间大小或键范围超过预定义阈值时触发合并操作, 并按预定义的优先级执行合并

![FADE机制示意图](https://raw.githubusercontent.com/Yee686/Picbed/main/2025-03-31-20-40-17-RocksDB-Delete.png)

![KiWi三层键值对组织](https://raw.githubusercontent.com/Yee686/Picbed/main/2025-03-31-20-40-25-RocksDB-Delete.png)

## Lethe -- 删除感知的LSM-Tree引擎

- Lethe用极少的元数据、新的数据布局形式KiWi和新的删除感知合并策略FADE实现了更高的吞吐量（1.17-1.43倍）、更低的空间放大（2.1-9.8倍）和适中的写放大（4%-25%）
- FADE机制
  - 保证在指定时间内失效
  - 为了实现指定时间内删除墓碑失效, 每个包含删除墓碑的文件都记录了失效时间TTL, 需要确保在指定时间内到达LSM-Tree的底部, 结合每一层参与合并的概率和开销, 大致可认为$TTL_i=TTL_{i-1}*T, TTL_0+ TTL_1…+TTL_i=D$, 其中T是相邻层间的乘数因子。
  - 除了TTL外, 还需记录每个文件中最旧删除墓碑的时间戳和预估失效记录数, 失效记录数可通过RocksDB的删除元数据(num_delete)和delete range预估得到, 元数据的获取较轻量。
- KiWi机制
  - KiWi将原本RocksDB中`File--Page`的组织形式改为了`File--Delete tile--Page`三级结构, Delete tile内部按sort key有序排列, Page间按non-sort key排序, Page内部也按sort key排序。
  - 首先, 由于Delete tile内的Page间按照delete key排序, 在执行范围删除时可以直接丢弃满足条件的Page, 无需参与读写;
  - 而部分满足条件的Page则直接将内部有效数据读出生成新的Page, 旧Page由后续合并回收空间, 实现对删除Page的移动更新。
  - 其次, 由于Delete tile内部按non-sort key排序, 所以支持对非排序键使用Delete Range。
  - 但是由于二级结构变成了三级结构, 并且按Delete tile内部按非排序键排列, KiWi的读性能和范围扫描性能会影响。
- 合并策略优化
  - 合并触发模式
    - RocksDB默认触发规则即选择超过层预设目标空间比例最高的层
    - FADE新增基于TTL的触发模式, 当包含删除墓碑的文件TTL到后则调度合并, 即使所在层并未达到目标大小
  - 文件选取规则
    - 重叠驱动的文件选择策略SO即RocksDB的默认规则, 在选择时尽量选择重叠文件少的以减少读写放大
    - 预估失效键驱动SD的文件选择, 优先选择预估失效记录数最多的文件以减少空间放大
    - TTL驱动DD的文件选择, 优先选择TTL到期的文件以满足用户要求
  - 总体规则: 当两个文件在SD/DD模式优先级相同时则选择具有最旧删除墓碑的SSTable, 如果SO模式下优先级相同则选择预估失效数最多的SSTable。

## Zone-Aware Persistent Deletion for Key-Value Store Engine (2024 NVMSA 西交)

- 用于KV存存储的分区感知的删除持久化
![2025-03-24-19-13-46-RocksDB-Delete](https://raw.githubusercontent.com/Yee686/Picbed/main/2025-03-24-19-13-46-RocksDB-Delete.png)
- 动机
  - Lethe-Fade的写放大很高, 在频删场景下写放大是接近RocksDB的3倍
  - Lethe-Fade没有考虑ZNS SSD特性, 忽略了垃圾回收时的垃圾迁移开销

![2025-03-24-19-26-57-RocksDB-Delete](https://raw.githubusercontent.com/Yee686/Picbed/main/2025-03-24-19-26-57-RocksDB-Delete.png)

![2025-03-24-19-29-02-RocksDB-Delete](https://raw.githubusercontent.com/Yee686/Picbed/main/2025-03-24-19-29-02-RocksDB-Delete.png)

- ZAP-Deletion方案设计
  - 墓碑感知的分区分配
    - 使用删除键的生命周期来指导数据布局
    - 记录第i级SSTable的平均生命周期$Lifetime_{Li}$
    - L0和L1层的Tombstone SSTable被视为Hot
    - 如果带有删除墓碑的SSTable的生命周期短于$Lifetime_{Li}$, 放到同级别的SSTable的分区里面, Hot Tombstone SSTable
    - 如果带有删除墓碑的SSTable的生命周期长于$Lifetime_{Li}$, 放到同级别的SSTable的分区里面, Cold Tombstone SSTable
  - 墓碑感知的SSTable选择
    - Lethe在选文件时选即将过期墓碑的SSTable, 额外的写入放大
    - 优先压缩删除墓碑到期的SSTable, 对于其他SSTable分层考虑
    - L0, L1采用选文件时选重叠率最小的
    - L2 ~ Ln采用基于优先级的方法来做
    - $w = -num_{overlap} + inc_{overlap} * \gamma$
      - $-num_{overlap}$为重叠数量, $inc_{overlap}$为上次合并以来新增的重叠文件, $\gamma$为0.8
      - 综合考虑当前重叠情况和重叠的增加
- 性能分析
  - 合并操作的数据写开销基本于RocksDB持平
  - ![2025-03-31-20-03-11-RocksDB-Delete](https://raw.githubusercontent.com/Yee686/Picbed/main/2025-03-31-20-03-11-RocksDB-Delete.png)
  - ZNS SSD上的GC写开销大幅降低
  - ![2025-03-31-20-04-33-RocksDB-Delete](https://raw.githubusercontent.com/Yee686/Picbed/main/2025-03-31-20-04-33-RocksDB-Delete.png)

![面向OCSSD的删除感知垃圾回收机制中版本信息的记录](https://raw.githubusercontent.com/Yee686/Picbed/main/2025-04-01-16-24-00-RocksDB-Delete.png)

## Enabling a deleted-key-value-aware garbage collection strategy for LSM-tree on OCSSD (2024 APCCAS 台湾清华)

- 动机: OCSSD的异地更新机制在处理删除操作时, 给数据擦写带来了挑战, 很长时间内并未被删除, 文章提出了删除键值感知的垃圾回收策略(针对闪存页的垃圾回收), 确保指定删除数据在数据恢复阶段被完全擦除
- 总体设计: 在Wisckey之上来做的设计, Key放在SSTable, Value放在Value Log中; 将包含删除键值的数据标记为敏感数据, 垃圾回收时邮箱选择敏感数据
  - 挑战1: 跟踪不同版本数据的全部位置
  - 挑战2: 如何降低垃圾回收开销
- 具体设计:
  - 版本跟踪: 引入`Version Table`, 记录键信息\冷热\版本号和键对应值在ValueLog内的地址\值得大小; 减少对LSM树依赖(?这读数据流程会变成什么样)
  - 冷热分离: 两个独立Memtable分别装冷热数据, 利用布隆过滤器区分冷热
  - 删除感知的垃圾回收:
    - 传统做法: 选择失效闪存页面最多的闪存块来做垃圾回收
    - 感知做法: 通过VersionTable定位到删除KV所在表, 找到LBA和对应长度, 如何传递给闪存定位到物理地址, 垃圾回收时优先选择这些块

## 参考文献

- [RocksDB-删除机制研究--鲁凯](https://emperorlu.github.io/Delete-in-Rocksdb/)
- Lethe: A Tunable Delete-Aware LSM Engine 删除感知的LSM引擎
- Zone-Aware Persistent Deletion for Key-Value Store Engine 与ZNS SSD结合
- Constructing and Analyzing the LSM Compaction Design Space
- X-Engine: An Optimized Storage Engine for Large-scale E-commerce Transaction Processing
- MyRocks: LSM-tree database storage engine serving Facebook’s social graph
- Enabling a deleted-key-value-aware garbage collection strategy for LSM-tree on OCSSD
