---
title: RocksDB的Compaction/Flush分析
tag: [RocksDB, Compaction, Flush, KV store, LSM-tree]
index_img: /img/rocksdb.png
date: 2024-12-17 20:33:00
updated: 2024-12-27 21:50:00
category: 代码分析
---

# RocksDB关键流程

## Flush流程

### 后台Flush作业

- 调度Flush作业流程与compaction类似: `DBImpl::BGWorkFlush`-->`DBImpl::BackgroundCallFlush`-->`DBImpl::BackgroundFlush`-->`DBImpl::FlushMemTablesToOutputFiles`-->`DBImpl::FlushMemTableToOutputFile`
- `DBImpl::BGWorkFlush(void* arg)`: 开启后台Flush作业
  - 转发到`DBImpl::BackgroundCallFlush(Env::Priority thread_pri)`
    - 调用`DBImpl::BackgroundFlush()`
      - 调用`DBImpl::FlushMemTablesToOutputFiles()`
        - 调用`DBImpl::FlushMemTableToOutputFile()`
  - 检查状态
  - 调用`DBImpl::MaybeScheduleFlushOrCompaction()`

### Flush作业主控流程

- `DBImpl::FlushMemTableToOutputFile`: 将Memtable刷盘, 核心流程
  - 创建`FlushJob flush_job`
  - 调用成员方法`FlushJob::PickMemTable()`, 选择需要刷盘的memtable
    - 获取该列族的imm_(imutable memtable)列表, 调用`MemTableList::PickMemtablesToFlush(...)`
      - 遍历`current_->memlist_`, 将未flush的(!flush_in_grogress_)的memtable加入到`mems_`中, `mems_`中按时间升序排列
      - 不管`mems_`中有多少个memtable, 均有第一个`memtable`的`edit_`来记录该次flush的元数据
  - 调用成员方法`FlushJob::Run(...)`, 启动刷盘
    - 首先会检查是否需要`MemPurge()`, 即Memtable垃圾回收
    > 将Immutable Memtable中过期内容清除, 将有效数据保存到新的memtable中, 旨在减少对SSD的写操作, 并降低写放大
    - **执行`FlushJob::WriteLevel0Table()`, 执行刷盘**
      - `memtables`保存所有mm的迭代器
      - 设定`TableBuilderOptions tboptions`
      - 调用`BuildTable()`, 生成L0层SSTable
      - 调用`AddFile()`将生成文件加入当前`edit_`, 相当于注册到LSM树的L0层
      - 调用`FlushJob::GetFlushJobInfo()`获得作业详细信息(包括file_path, table_properties), 并设置mems_队列的首个memtable
    - 如果需要写最新版本MANIFEST, 会调用`MemtableList::TryInstallMemtableFlushResults(...)`
  - 安装新版本`DBImpl::InstallSuperVersionAndScheduleWork`
  - 通知完成Flush, 在`SstFileManagerImpl`注册生成的SSTable对象

### BuildTable工作流程

- 调用`NewWritableFile`申请可写文件`file`, 并设定IO优先级和LifetimeHint
- 设置`WritableFileWriter`, 并调用`NewTableBuilder`, 与`CompactionJob::OpenCompactionOutputFile()`类似
- 设置`MergeHelper`和用于KV迭代的`CompactionIterator`
- 迭代`c_iter`, 调用`TableBuilder`的`Add(key, value)`将KV对加入l0层的`SSTable`中
- 更新待刷新第一个Memtable的`FileMetaData`(保存整个FlushJob元数据)和入参`TableProperties`(生成L0层SSTable元数据)
- 构建`OutputValidator`和`InternalIterator`检查新生成SSTable的有效性

## Compactoion流程

### 调度Flush/Compaction作业

- `DBImpl::InstallSuperVersionAndScheduleWork`: 所有列族的状态通过该函数改变, 该方法分析列族的新状态, 并决定是否需要flush/compaction
  1. 调用`cfs->InstallSuperVersion(...)`
  2. 调用`SchedulePendingCompaction(cfd)`:
  3. 调用`MaybeScheduleFlushOrCompaction`
- `DBImpl::SchedulePendingCompaction(ColumnFamilyData* cfd)`:
  1. 如果当前列族未在`compaction_queue_`中, 且`NeedsCompaction()`返回需要, 则通过`AddToCompactionQueue(cfd)`加入等待调度的Compaction队列
  2. `cfd->NeedCompaction()`会检查是否开启了自动compaction, 并转发至`compaction_picker->NeedCompaction(current_-storage_info())`
  3. 完成后调用`AddToCompactionQueue(cfd)`, 增加待调度的Compaction作业数
  
- `class CompactionPicker`: `db/compaction/compaction_picker.h`
  - 用于执行compaction文件选择的抽象类, RocksDB提供了`compaction_picker_level`/`compaction_picker_fifo`/`compaction_picker_universal`
  - 默认使用`LevelCompactionPicker`, 如下
    - 看是否需要Compaction接口`LevelCompactionPicker::NeedsCompaction()`
      - 如果有超时TTL/被标记为Compaction的则返回Ture
      - 自顶向下获取(已被排序)CompactionScore, 有大于1返回Ture, **CompactionScore事先由VersionStorageInfo::ComputeCompactionScore计算**
      - `VersionStorageInfo::ComputeCompactionScore`逻辑
        - 如果为L0: 统计该层文件数/文件大小, 得分=文件数/预先设定的阈值
        - L1开始: 未参与Compaction文件总大小/设定阈值,
        - 将compaction_score从高到低排序, 高分在list前部
        - 更新TTL队列, 周期Compaction队列等, 这些均在NeedCompaction中首先被判断
    - `LevelCompactionPicker::PickCompaction(...)`: 构建`LevelCompactionBuilder`并调用`builder.PickCompaction()`, 返回一个Compaction
    - `LevelCompactionBuilder::PickCompaction()`流程如下
      - `LevelCompactionBuilder::SetupInitialFiles()`: 对所有大于阈值的层调用`PickFileToCompact`,获得`CompactionInputFiles`,并确定`CompactionReason`
        - `struct CompactionInputFiles`, 位于`db/compaction/compaction.h`
          - 维护Compaction输入文件数组`vector<FileMetaData*> files`
          - 维护compaction边界`std::vector<AtomicCompactionUnitBoundary> atomic_compaction_unit_boundaries`, 即`InternalKey`的最大值和最小值
      - `LevelCompactionBuilder::SetupOtherL0FilesIfNeeded()`
      - `LevelCompactionBuilder::SetupOtherInputsIfNeeded()`
      - `LevelCompactionBuilder::GetCompaction()`

- `DBImpl::MaybeScheduleFlushOrCompaction()`:
  1. 获取后台作业数(`DBImpl::GetBGJobLimits()`)
  2. 获取flush_pool是否为空, 即`env_->GetBackgroundThreads(Env::Priority::HIGH) == 0`
  3. 如果flush_pool不空, 且未达到job上限, 在while循环中调度高优先级的`BGWorkFlush`
  4. 如果flush_pool为空, 则调度低优先级的`BGWorkFlush`
  5. 如果bg_compaction_scheduled_ 与bg_bottom_compaction_scheduled_总数小于上限, 调度低优先级的`BGWorkCompaction`
  
  ``` c++
  // function函数指针为待注册的作业, arg为参数, pri为优先级, tag为标签, unschedFunction为移除时的回调函数
  virtual void Schedule(void (*function)(void* arg), void* arg,
                        Priority pri = LOW, void* tag = nullptr,
                        void (*unschedFunction)(void* arg) = nullptr) = 0;
                    
  // 调度Flush作业
  env_->Schedule(&DBImpl::BGWorkFlush, fta, Env::Priority::HIGH, this, &DBImpl::UnscheduleFlushCallback);

  // 调度Compaction作业
  env_->Schedule(&DBImpl::BGWorkCompaction, ca, Env::Priority::LOW, this, &DBImpl::UnscheduleCompactionCallback);
  ```

### 后台Compaction作业

- `DBImpl::BGWorkCompaction(void* arg)`: 开启后台Compaction作业
  - 将`arg`转为`CompactionArg`, 并从中拿到`prepicked_compaction`, 通过`BackgroundCallCompaction`传递
  - 转发到`DBImpl::BackgroundCallCompaction()`
    - 调用`DBImpl::BackgroundCompaction()`
    - 调用`DBImpl::MaybeScheduleFlushOrCompaction()`, 调度其他可能的作业

- `DBImpl::BackgroundCompaction`: 
  - 首先会检查各参数的有效性, 是否有足够空间
  - 特列检查: 只需删除输入文件, 只需将输入文件移动到到下一level不涉及Merge/split, 可以转发到`BGWorkBottomCompaction`
  - 构建`CompactionJob`
  - `DBImpl::NotifyOnCompactionBegin()`
  - `compaction_job.Prepare()`即`CompactionJob::Prepare()`: 确定子合并的边界
    - `cfd->CalculateSSTWriteHint(c->output_level())`: 根据输出level计算文件的`Lifetime Hint`
    - 调用`CompactionJob::GenSubcompactionBoundaries`: 对于每个输入文件, 通过扫描文件`index block`估算出128个锚点, 将sst大致划为128个子区间, 再将其合并为n个子合并区间, 并将边界保存再boundaries_中
    - 如果设置了保留时间,则处理`seqno--time`之间的映射, 该映射("class SeqnoToTimeMapping")用于估算key的time

  - `compaction_job.Run()`即`CompactionJob::Run()`: 执行合并操作
    - 初始化线程池, 预留出`子合并数-1`个线程空间用于处理子合并
    - 启动子合并线程`CompactionJob::ProcessKeyValueCompaction`
    - 调用lambda匿名函数`verify_table`验证输出的有效性, 复用线程池
    - 使用`Compaction::SetOutputTableProperties()`设置该Compaction的输出文件属性
  - `compaction_job.Install()`即`CompactionJob::Install()`: 将合并结果加入到当前版本中
    - 调用`InstallCompactionResults()`更新`VersionEdit`
    - 调用`UpdateCompactionJobStats()`更新作业统计信息
  - `InstallSuperVersionAndScheduleWork(...)`: 实际是`InstallSuperVersion`的封装, 还会检查新状态是否需要flush/compaction
  - 调用`SstFileManager`的`OnCompactionCompletion()`, 更新记录
  - `DBImpl::NotifyOnCompactionCompleted()`

### Compaction子合并

- **`CompactionJob::ProcessKeyValueCompaction()`: 调用filter, 遍历输入KV并执行compaction**
  - 如果有`CompactionService`(可以理解为使用外部Compaction算法)则调用`CompactionJob::ProcessKeyValueCompactionWithCompactionService()`, 执行成功则返回;否则使用自带的Compaction算法, 即主分支
  - 构建`InternalIterator`, 指针对象`raw_input`和`input`
  - 构建`MergeHelper`
  - 构建`CompactionIterator`, 指针对象`c_iter`
  - `c_iter->SeekToFirst()`, 定位到第一个KV对
  - **定义`CompactionFileOpenFunc`/`CompactionFileCloseFunc`**
    - 转发到`CompactionJob::OpenCompactionOutputFile()`
    - 转发到`CompactionJob::FinishCompactionOutputFile()`
  - **开始循环迭代**, 每次迭代增加一个KV对文件中
    - 调用`sub_compact->AddToOutput(*c_iter, open_file_func, close_file_func)`, 将`c_iter`的KV对加到`Current`输出组中 
      - `SubcompactionState::AddToOutput()`转发到`Current().AddToOutput(...)`, `Current()`返回指向`class CompactionOutputs`对象的指针
        - 将KV对加入validator
        - 将KV对通过table builder加入SST, `builder_->Add(key, value)`
        - 更新FileMetaData的最大最小键
    - `c_iter->Next()`迭代到下一KV-pair
  - 状态检查, 再次执行`sub_compact->CloseCompactionFiles()`完成收尾工作

### Compaction文件读写

- **`CompactionJob::OpenCompactionOutputFile(...)`**
  - 调用`read_write_util`中的`NewWritableFile`, 转发到`fs->NewWritableFile`, 创建一个制定了文件名的新文件对象, 用指针`writable_file`保存
  - 设置该文件的`FileMetaData`, 并调用`outputs.AddOutput`加入当前输出队列中
  - 通过文件系统接口层中`class FSWritableFile`提供的接口, 设置`writable_file`的IO优先级\`lifetime_hint`等信息
  - 调用`CompactionOutputs::AssignFileWriter()`设置写入器, 设置`TableBuilderOptions`, 再通过`CompactionOutputs::NewBuilder()`创建`TableBuilder`对象
- **`Status CompactionJob::FinishCompactionOutputFile(...)`**
- `TableBuilder::Add(K, V)`向table中添加KV(以默认的`Block_based_table_builder::Add`为例)
  - 先检查`Key`的`ValueType`
  - 检查是否需要`Flush()`(将缓冲的KV对刷入文件中)
    - 需要则调用`BlockBasedTableBuilder::Flush()`, 主逻辑转发到`BlockBasedTableBuilder::WriteBlock()`
    - 先通过`block->Finish()`关闭当前block
    - 调用`BlockBasedTableBuilder::CompressAndVerifyBlock`尝试压缩block内容
    - 调用`BlockBasedTableBuilder::WriteMaybeCompressedBlock`写入数据块
      - 核心流程`r->file->Append(block_contents)`, 调用`WritableFileWriter::Append(...)`
        - **这里用到了`use_direct_io()`判断是否是直接IO, 我的大部分实验设置为direct**, 如果需要验证数据有效性还是会先写到一个buffer用于计算校验和
        - `FSWritableFile::PrepareWrite`写前准备工作, 通过预分配空间可以减少文件碎片, 或者避免文件系统过度预分配带来的浪费
        - 核心是调用`buf_.Append(src, left)`, `class AlignedBuffer`根据对齐需求来管理一个缓冲区, 主要用于`direct io`场景, `Append`方法使用`memcpy`复制到buffer中
        - `WritableFileWriter::Flush()`: 将内存中数据写入文件系统/设备
          - 有缓冲写`WriteBuffered(...)`, **使用`writable_file_->Append()`写底层文件**
          - 无缓冲写`WriteDirect(...)`, **使用`writable_file_->PositionedAppend()`写底层文件**
          - 调用`writable_file_->Flush(...)`, 这是底层文件系统接口
      - 调用`ComputeBuiltinChecksumWithLastByte`处理`checksum`
      - 调用`InsertBlockInCompressedCache`, 将压缩后的block_content加入缓存
  - `r->data_block.AddWithLastKey`, 调用`BlockBuilder::AddWithLastKey`转发到`BlockBuilder::AddWithLastKeyImpl`, 主要是操作对应`block builder`的`buffer_`(string类型), 通过`append`方法将`key`写入`buffer_`中
  - `r->last_key.assign(key.data(), key.size())`, 更新`last_key`(string类型)值
  - 更新`TableProperties`, 包括`num_entries`/`raw_key_size`/`raw_value_size`

### Flush/Compaction相关数据结构

- `class CompactionOutputs`: 维护子合并产生的文件, 多数方法由compaction_job的open/close合并文件的方法使用
  - 数据成员
    - `std::unique_ptr<TableBuilder> builder_`: SST构建器指针
    - `std::unique_ptr<WritableFileWriter> file_writer_`: 文件写入器指针
    - `std::vector<Output> outputs_`: 当前时刻所有输出
      - `Struct Output`存放`FileMetaData`文件元数据/`OutputValidator`有效性验证器/`TableProperties`表格属性
  - 重要方法
    - `AddToOutput(...)`: 将文件添加到`outputs_`队列
    - `NewBuilder(...)`: 为当前output设置`TableBuilder`构建器
    - `AssignFileWriter(...)`: 为当前output设置`WritableFileWriter`写入器

- `class CompactionState` 维护Compaction状态
  - 数据成员
    - `Compaction* const compaction`: 指向对应Compaction的常量指针
    - `std::vector<SubcompactionState> sub_compact_states`: 维护子合并状态
  - 方法
    - `AggregateCompactionStats`聚合合并状态
    - `Slice SmallestUserKey()`/`Slice LargestUserKey()`获取用户键

- `VersionEdit::AddFile(...)`: 向LSM树某一层添加文件

## 工具方法

- `int sstableKeyCompare(const Comparator* user_cmp, const InternalKey& a, const InternalKey& b)`
  - 先直接调用`user_cmp->CompareWithoutTimestamp(a.user_key(), b.user_key())`, 指比较两个`InternalKey`的`UserKey`部分, 并且不考虑时间戳
  - 如果不相等就直接返回, 如果先等则调用`ExtractInternalKeyFooter()`, 比较两个键的Footer, Footer就是`InternalKey`的最后8字节, 即`SequenceNumber`和`ValueType`的组合
  > `RocksDB`中, `Internal Key`在默认情况下
  > `| User Key (变长) | Sequence Number (7 字节) | Value Type (1 字节) |`
  > 在启用了`User Defined Timestamp`
  > `| User Key (变长) | Timestamp (变长) | Sequence Number (7 字节) | Value Type (1 字节) |`
  > `Timestamp`用于支持基于时间戳的数据管理
  > `Sequence Number`用于支持MVCC, 越大越新, 查询/合并时返回最新版本


## RocksDB-ZenFS交互流程分析

- `CompactionJob::OpenCompactionOutputFile(...)`-->调用FS的`FileSystem::NewWritableFile(...)`-->调用ZenFS的`ZenFS::OpenWritableFile(...)`-->通过新建`ZonedWritableFile`对象, 并返回对象指针, `ZonedWritableFile`继承自`public FSWritableFile`
- `ZonedWritableFile`的`Append`方法在非`buffered`模式下, 调用`ZoneFile::Append`-->在对非活动分区写入或写入空间不够时调用`ZoneFile::AllocateNewZone()`分配新分区-->转发到`ZonedBlockDevice::AllocateIOZone(...)`执行分区分配逻辑
- `ZoneFile`维护一个`ZoneExtent`指针队列, 每个`ZoneExtent`对应一个`Zone`, 每个`Zone`维护了写指针`wp_`/`WriteLifeTimeHint`/`capacity`等
- `ZonedWritableFile::Append`与`ZonedWritableFile::PositionedAppend`
  - 如前文所述, `Append`来自BufferedIO模式调用, `PisitionAppend`来自DirectIO模式调用
  - 在ZenFS中, `PisitionAppend`强制保证在`wp_`处写入, 其余与`Append`一致, 转发到`zoneFile_->Append`
  - 上层RocksDB会保证`Append`和`PositionAppend`只选择一种

## RocksDB-Table-build源码分析

## 公共接口

- `table.h`: 与sst相关的抽象类, 两种table
  - `Block-based table`: 默认table类型, 来自LevelDB
  - `Plain table`: RocksDB针对低访问延时设备(纯内存, 低访问延迟介质)优化
  - `class TableFactory : public Customizable`: table工厂函数基类
    - 重要接口
      - `virtual Status NewTableReader(...) = 0`, 读取器有三个地方被调用
        - `TableCache::FindTable()` table cache未命中, 返回一个table
        - `SstFileDumper` 打开table并转储
        - `DBImpl::IngestExternalFile()` 用于添加SST中的内容
      - `virtual TableBuilder* NewTableBuilder(...) = 0`, 构建器/写入器, 有四个地方被调用
        - `DBImpl::WriteLevel0Table()`: 通过`BuildTable()`间接调用, flush操作中Mem=>L0
        - `DBImpl::OpenCompactionOutputFile()`: 写Compaction的输出时
        - `DBImpl::WriteLevel0TableForRecovery`: 通过`BuildTable()`间接调用, 从transaction log中恢复时
        - `Repairer::ConvertLogToTable()`: 通过`BuildTable()`间接调用, 在修复阶段时, 将log转为sst
        - 调用者有责任保持文件打开，并在关闭表构建器后关闭文件

## 实现

- `table\table_builder.h`: 
  - 声明`struct TableReaderOptions`和`struct TableBuilderOptions`
  - `class TableBuilder`: 用于构建Table的接口
    - `virtual void Add(const Slice& key, const Slice& value) = 0` 向构建中的表添加KV对
    - `virtual Status Finish() = 0` 完成构建
    - `virtual uint64_t NumEntries() const = 0`
- `table\block_based\`: block_based table的具体实现
  - `block_based_table_factory.h`
  - `block_based_table_builder.h`
    - `class BlockBasedTableBuilder : public TableBuilder`

## 参考资料

[1] [RocksDB学习笔记#6 Compaction流程(1) —— 触发流程](https://blog.csdn.net/qq_38876396/article/details/143475637)
[2] [RocksDB学习笔记#7 Compaction流程(2) —— 执行流程](https://blog.csdn.net/qq_38876396/article/details/143478199)
[3] [rocksdb介绍之compaction流程](https://blog.csdn.net/xuhaitao23/article/details/125430516)
[4] [RocksDB Wiki -- Compaction](https://github.com/facebook/rocksdb/wiki/Compaction)
[5] [RocksDB Wiki -- Compaction 中文文档](https://github.com/johnzeng/rocksdb-doc-cn/blob/master/doc/Compaction.md)