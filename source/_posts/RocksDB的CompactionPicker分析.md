---
title: RocksDB的Compaction Picker分析
tag: [RocksDB, Compaction, KV store, LSM-tree]
index_img: /img/rocksdb.png
date: 2025-4-25 17:00:00
updated: 2025-4-25 17:00:00
category: 代码分析
---

# RocksDB Compaction Picker分析

相关头文件: `compaction_picker.h`, `compaction_picker_level.h`, `version_set.h`

## 抽象类`CompactionPicker`

- 触发条件: 当`NeedCompaction()`返回`ture`时调用`PickCompation()`找到哪里需要合并, 并往哪一层合并
- `class CompactionPicker`
  - **主要成员方法**
    - `virtual Compaction* PickCompaction(...) = 0`: 对指定列族的新Compaction选择level和input
    - `virtual Compaction* CompactRange(...)`: 对指定列族的指定level的指定Range生成compaction对象
      - 检查input(L0除外)避免一次压缩过多数据, 看每个输入层SSTable和输出层重叠的SSTable总和是否超过`mutable_cf_options.max_compaction_bytes`, 如超过则保留当前文件并停止处理
    - `bool ExpandInputsToCleanCut(...)`: 扩张$L_i$层输入Key的范围直至不与上一个文件和下一个文件不与输入文件存在键重合(clean cut), 循环扩张直至输入数目不变, 通过`class VersionStorageInfo`的`GetOverlappingInputs`方法更新输入文件

      ``` c++
        do {
        old_size = inputs->size();
        GetRange(*inputs, &smallest, &largest);
        inputs->clear();
        vstorage->GetOverlappingInputs(level, &smallest, &largest, &inputs->files,
                                       hint_index, &hint_index, true,
                                       next_smallest);
      } while (inputs->size() > old_size);
      ```

    - `void GetRange(const CompactionInputFiles& inputs, InternalKey* smallest, InternalKey* largest)`: 获取输入文件的的Range, 如果是L0则对比SSTable的最大键和最小键; 否则获取第一个文件的最小键和最后一个文件的最大键(leveled compaction策略)
    - `Compaction* CompactFiles(...)`: 针对输入的一组文件返回压缩对象, 主要在手动压缩场景下使用
    - `bool FilesRangeOverlapWithCompaction`: 检查输入文件的key range与正在执行的合并压缩是否存在重叠, 存在返回true
      - 需要检查output_level的下一层penultimate_level, 防止与penultimate_level正在进行的合并压缩由于key range overlap导致竞争, 影响性能
      - 后去range后转发到`RangeOverlapWithCompaction`
    - `bool CompactionPicker::FilesRangeOverlapWithCompaction`: 检查正在执行的合并压缩是否与指定key range重叠
    - `bool FindIntraL0Compaction`: 找一个合适的文件范围进行内部压缩, 以减少 L0 文件数量, 满足参数max_compaction_bytes, max_compact_bytes_per_del_file和min_files_to_compact的限制
  - **主要数据成员**
    - `const ImmutableOptions& ioptions_`: 不可变操作
    - `const InternalKeyComparator* const icmp_`: 内部键比较器
    - `std::set<Compaction*> level0_compactions_in_progress_`: L0层正在进行的合并
    - `std::unordered_set<Compaction*> compactions_in_progress_`: 正在进行的所有合并

## 派生类`LevelCompactionPicker`

- 基于leveled compaction策略实现CompactionPicker, 实现接口`PickCompaction(...)`和接口`NeedsCompaction(...)`
- `bool LevelCompactionPicker::NeedsCompaction(...)`: 依次检查是否有TTL, 是否有文件被标记为定期压缩, 是否有最底层文件被标记, 是否有文件被标记, 是否有文件被标记为强制BlobGC, 是否有得分超过1的层, 都没有则返回false
- `Compaction* LevelCompactionPicker::PickCompaction(...)`: 执行Compaction选择的逻辑, 实例化对象`LevelCompactionBuilder builder(...)`, 转发`builder.PickCompaction()`
  - `class LevelCompactionBuilder`: 构建leveled compaction的辅助类
    - **主要数据成员**
      - `VersionStorageInfo* vstorage_` 指向当前时刻版本的指针
      - `CompactionInputFiles start_level_inputs_` 起始层`Li`的输入文件
      - `CompactionInputFiles output_level_inputs_` 输出层`Li+1`的输入文件
      - `std::vector<CompactionInputFiles> compaction_inputs_`
    - **主要方法方法**
      - `Compaction* LevelCompactionBuilder::PickCompaction()`: 主工作逻辑, 返回生成的Compaction对象
        - 调用`SetupInitialFiles()`
        - 检查`SetupOtherL0FilesIfNeeded()`
        - 检查`SetupOtherInputsIfNeeded()`
        - 通过`GetCompaction()`获取Compaction并返回
      - `void LevelCompactionBuilder::SetupInitialFiles()`:
        - 从上至下开始检查, 看start_level_score是否大于1, 将满足条件的首个level作为start_level, 并确定output_level(input_level + 1)
        - 调用`PickFileToCompact()`为指定start_level挑选文件作为合并压缩输入, 如果没用返回false
          - 如果没有且start_level为0, 可能是有L0->base_level压缩导致的写暂停, 调用`PickIntraL0Compaction`, 执行L0层内部压缩减少文件数量
        - 至此已挑选出输入文件则可返回, 否则继续检查其他情况
          - 检查是否有文件被标记, 调用`PickFilesMarkedForCompaction`
          - 对最底层文件执行合并压缩以清除删除墓碑, `PickFileToCompact(vstorage_->BottommostFilesMarkedForCompaction(), false)`
          - 检查是否有过期的超出ttl文件, 定期压缩和blob GC的情况
      - **`bool LevelCompactionBuilder::PickFileToCompact()`**
        - 如果L0层有正在进行的合并则返回false
        - 调用`TryPickL0TrivialMove()`尝试直接将L0层文件移动到L1层
        - 通过`vstorage_`的`LevelFiles()`接口拿到该层文件的元数据
        - 通过`vstorage_`的`FilesByCompactionPri()`接口拿到该层的文件压缩合并优先级
        - 通过`NextCompactionIndex()`接口返回的顺序检查文件, 获得`start_level_inputs_`
          - 调用`compaction_picker_`的`ExpandInputsToCleanCut(...)`扩张同层输入成与前后文件无重叠的`clean cut`
          - 调用`compaction_picker_`的`FilesRangeOverlapWithCompaction(...)`检查当前输入文件是否与现有合并重叠, 有则丢弃继续找下一个文件
        - 处理`output_level_inputs`, 将`output_level`上的与`start_level_inputs_`重叠的放入`output_level_inputs`, 并同样扩张成`clean cut`
        - 通过`vstorage_->SetNextCompactionIndex`设定下次合并压缩的起始index

      - `void LevelCompactionBuilder::PickFileToCompact( const autovector<std::pair<int, FileMetaData*>>& level_files, bool compact_to_next_level)`: 在指定level_files选出输入文件, 并扩张
      - `Compaction* LevelCompactionBuilder::GetCompaction()`: 构建Compaction对象, 并向compaction_picker_注册该对象, 插入到
      - 防止加入新合并的SSTable被重复选入, 需要立即重新计算合并压缩得分`vstorage_->ComputeCompactionScore(ioptions_, mutable_cf_options_)`

## 相关类 `VersionStorageInfo`

- `class VersionStorageInfo`是管理LSM-Tree版本存储元数据的和核心类, 负责跟踪当前数据库版本Version中所有状态
- 与合并相关的关键方法
  - **选层: 层级优先级计算** `void ComputeCompactionScore(const ImmutableOptions& mmutable_options, const MutableCFOptions& mutable_cf_options)`
    - 对于得分大于1的乘以放大系数10以提高优先级, 从上至下逐层计算得分
      - 如果为L0层
        - 统计该层总文件/有序集合数(num_sort_runs)
        - 如果是Universal模式: 把L0得分作为整个DB是否合并压缩的考虑条件, 并从L1开始的每层不空且收文件未参与, 则`num_sort_runs++`
        - `score = num_sorted_runs / level0_file_num_compaction_trigger`
        - 如果是Leveled模式且num_level > 1: 还需考虑L0层总大小以避免L0或L1层积压
          - 如果开启了动态适应, 当总大小大于`max_bytes_for_level_base`时, 直接赋1.01; 当大于`base_level`的大小限制时, 用`max(base_level_size, level_max_bytes_[base_level_])`做分母, 防止L0长期过大无法使得L1参与压缩
          - 否则直接取max(基于文件数得分, 基于大小得分)
      - 如果为L1-Ln层, 统计level总大小`level_total_bytes`和当前未参与合并压缩的总大小`level_bytes_no_compacting`
        - 非动态适应或`level_bytes_no_compacting`小于限制时, `score = level_bytes_no_compacting / MaxBytesForLevel(level)`
        - 反之当`level_bytes_no_compacting`超出限制时, `score = (level_bytes_no_compacting) / (MaxBytesForLevel(level) + total_downcompact_bytes) * kScoreScale`, 其中`total_downcompact_bytes`是即将通过合并到本层的预计大小, 当有大量数据合入时降低该层的优先级, 可以降低预期的写放大
    - 将得分结果将`compaction_level_`和`compaction_score_`从高到低排序, 分别代表第i个压缩层号及其对应得分
    - 调用`ComputeFilesMarkedForCompaction()`, 将最底层外其他层需要合并压缩的文件添加到`files_marked_for_compaction_`
    - 调用`EstimateCompactionBytesNeeded`预估合并压缩数据量

  - **选文件: 每层文件优先级计算, 只针对Leveled Compaction** `void UpdateFilesByCompactionPri(const ImmutableOptions& ioptions, const MutableCFOptions& options)`
    - 逐层计算文件参与压缩合并的优先级, 根据`ioptions.compaction_pri`执行对应的逻辑, 默认是使用`kByCompensatedSize`, 建议使用`kMinOverlappingRatio`

    ``` c++
      // In Level-based compaction, it Determines which file from a level to be
      // picked to merge to the next level. We suggest people try
      // kMinOverlappingRatio first when you tune your database.
      enum CompactionPri : char {
        // Slightly prioritize larger files by size compensated by #deletes
        kByCompensatedSize = 0x0,
        // First compact files whose data's latest update time is oldest.
        // Try this if you only update some hot keys in small ranges.
        kOldestLargestSeqFirst = 0x1,
        // First compact files whose range hasn't been compacted to the next level
        // for the longest. If your updates are random across the key space,
        // write amplification is slightly better with this option.
        kOldestSmallestSeqFirst = 0x2,
        // First compact files whose ratio between overlapping size in next level
        // and its size is the smallest. It in many cases can optimize write
        // amplification.
        kMinOverlappingRatio = 0x3,
        // Keeps a cursor(s) of the successor of the file (key range) was/were
        // compacted before, and always picks the next files (key range) in that
        // level. The file picking process will cycle through all the files in a
        // round-robin manner.
        kRoundRobin = 0x4,
    };
    ```

    - 如果是`kByCompensatedSize`, 通过`std::partial_sort`按`FileMetaData`的`compensated_file_size`字段排序
    - 如果是`kOldestLargestSeqFirst`或者`kOldestSmallestSeqFirst`, 则直接对该层按`FileMetaData`中`FileDescriptor`的`SequenceNumber`排序
    - 如果是`kMinOverlappingRatio`, 转发到`SortFileByOverlappingRatio()`
      - 统计每个文件的`overlapping_bytes`, `score = overlapping_bytes * 1024U /file->compensated_file_size / ttl_boost_score`
      - `ttl_boost_score`来自`FileTtlBooster`, ​​基于TTL动态调整文件压缩优先级​​的核心逻辑. 其核心思想是:​让接近TTL过期时间的文件获得更高的压缩优先级, 但同时避免过度干扰正常的压缩策略​​.
      - 通过`std::partial_sort`按score排序, 如果得分相等选最小键更小的
