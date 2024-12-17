---
title: RocksDB的Compaction分析
tag: [RocksDB, Compaction]
index_img: /img/rocksdb.png
date: 2024-12-17 20:33:00
updated: 2024-12-17 21:00:00
category: 代码分析
---

# RocksDB关键流程

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

### 后台Flush/Compaction作业

- `DBImpl::BGWorkFlush(void* arg)`: 开启后台Flush作业
  - 转发到`DBImpl::BackgroundCallFlush(Env::Priority thread_pri)`
    - 调用`DBImpl::BackgroundFlush()`
      - 调用`DBImpl::FlushMemTablesToOutputFiles()`
        - 调用`DBImpl::FlushMemTableToOutputFile()`
  - 检查状态
  - 调用`DBImpl::MaybeScheduleFlushOrCompaction()`

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

- `CompactionJob::ProcessKeyValueCompaction()`: 调用filter, 遍历输入KV并执行compaction
  - 如果有`CompactionService`(可以理解为使用外部Compaction算法)则调用`CompactionJob::ProcessKeyValueCompactionWithCompactionService()`, 执行成功则返回;否则使用自带的Compaction算法, 即主分支
  - 构建`InternalIterator`, 指针对象`raw_input`和`input`
  - 构建`MergeHelper`
  - 构建`CompactionIterator`, 指针对象`c_iter`
  - `c_iter->SeekToFirst()`, 定位到第一个KV对
  - **定义`CompactionFileOpenFunc`/`CompactionFileCloseFunc`**
    - 转发到`CompactionJob::OpenCompactionOutputFile()`
    - 转发到`CompactionJob::FinishCompactionOutputFile()`
  - 开始迭代合并
    - 调用`sub_compact->AddToOutput(*c_iter, open_file_func, close_file_func)`, 将`c_iter`的KV对加到`Current`输出组中 
      - `SubcompactionState::AddToOutput()`转发到`Current().AddToOutput(...)`, `Current()`返回指向`class CompactionOutputs`对象的指针
    - `c_iter->Next()`迭代到下一KV-pair
  - 状态检查, 再次执行`sub_compact->CloseCompactionFiles()`完成收尾工作

### Compaction文件读写

- `CompactionJob::OpenCompactionOutputFile(...)`
  - 调用`read_write_util`中的`NewWritableFile`, 转发到`fs->NewWritableFile`, 创建一个制定了文件名的新文件对象, 用指针`writable_file`保存
  - 设置该文件的`FileMetaData`, 并调用`outputs.AddOutput`加入当前输出队列中
  - 通过文件系统接口层中`class FSWritableFile`提供的接口, 设置`writable_file`的IO优先级\`lifetime_hint`等信息
  - 调用`CompactionOutputs::AssignFileWriter()`设置写入器, 设置`TableBuilderOptions`, 再通过`CompactionOutputs::NewBuilder()`创建`TableBuilder`对象
- `Status CompactionJob::FinishCompactionOutputFile(...)`

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

## RocksDB-Table-build源码分析

### 公共接口

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

### 实现

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