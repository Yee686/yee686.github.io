---
title: NVMeVirt模拟ZNS SSD
tag: [RocksDB, Compaction, Flush, KV store, LSM-tree, ZNS SSD, NVMe]
index_img: https://raw.githubusercontent.com/Yee686/Picbed/main/2024-05-31-16-26-43-NVMeVirt.png
date: 2024-06-10 20:50:00
updated: 2024-10-16 20:50:00
category: 工具与框架
---

# NVMeVirt模拟ZNS

## 搭建步骤

服务器上ZenFS的依赖已经完备，直接开装

### 1 ZenFS和RocksDB安装

使用最新版本的RocksDB和ZenFS时，安装报错([如issue#288所示](https://github.com/westerndigitalcorporation/zenfs/issues/288))，原因时新版本的RocksDB更换了API而ZenFS未适配，使用RocksDB 8.11.3可以解决，直接按照README即可

``` shell
 git clone https://github.com/facebook/rocksdb.git
 cd rocksdb
 git clone https://github.com/westerndigitalcorporation/zenfs plugin/zenfs
 cd plugin/zenfs/util
 make
```

### 2 NVMeVirt搭建

#### 2.1 内存预留

``` shell
# 从16G开始预留32内存空间给NVMeVirt使用
GRUB_CMDLINE_LINUX="memmap=32G\\\$16G"
```

``` shell
sudo update-grub
sudo reboot
sudo cat /proc/iomem | grep Reserved 
```

![2024-05-31-16-18-47-NVMeVirt](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-05-31-16-18-47-NVMeVirt.png)

`400000000- bffffffff`预留了16GB的内存空间
`400000000-147fffffff`

#### 2.2 修改配置

选择zns模式，使用西数ZN540的配置

``` shell
# Select one of the targets to build
# CONFIG_NVMEVIRT_NVM := y
#CONFIG_NVMEVIRT_SSD := y
CONFIG_NVMEVIRT_ZNS := y
#CONFIG_NVMEVIRT_KV := y
```

修改zone_size，需要注意在ZenFS挂载时最少需要32个zone

``` c++
#elif (BASE_SSD == WD_ZN540)
...
#define ZONE_SIZE MB(256ULL)
...
```

![2024-05-31-16-21-36-NVMeVirt](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-05-31-16-21-36-NVMeVirt.png)

#### 2.3 加入内核

我选择只使用16GB的空间，其实地址为16GB，注意在设定memmap_size时增加1M，不然会报段错误(需要reboot才能重新插入内核模块，很麻烦)，dmesg显示zns_ftl的101行报错，原因是容量(Total Capacity)不能整除分区大小(Zone Size)，加1M即可

将2.2中编译好的内核模块加入内核(后续要重新更改修改分区时，先rmmod内核模块，然后修改配置文件重新编译)

##### 模拟16G ZNS

``` shell
sudo insmod /home/yzy/nvmevirt/nvmev.ko memmap_start=16G memmap_size=16385M cpus=6,7
```

##### 模拟32G ZNS

``` shell
sudo insmod /home/yzy/nvmevirt/nvmev.ko memmap_start=16G memmap_size=32769M cpus=6,7
```

报错

``` shell
[2278744.083448] NVMeVirt: Version 1.10 for >> WD ZN540 ZNS SSD <<
[2278744.083459] NVMeVirt: [mem 0x400000000-0xc000fffff] is usable, not reseved region
```

查看预留区域

``` shell
yzy@host1:~/DZR-rocksdb/rocksdb-7.10.2$ sudo cat /proc/iomem | grep Reserved
00000000-00000fff : Reserved
0003e000-0003efff : Reserved
000a0000-000fffff : Reserved
5f105000-5f105fff : Reserved
66520000-6841ffff : Reserved
68dc0000-6c1fefff : Reserved
6e580000-6ef2bfff : Reserved
6f800000-8fffffff : Reserved
fd000000-fe7fffff : Reserved
fed20000-fed44fff : Reserved
ff000000-ffffffff : Reserved
400000000-bffffffff : Reserved
1ffc00000000-1fffffffffff : Reserved
```

可以通过`sudo nvme zns zns-reports nvme4n1`查看详细信息

![2024-05-31-16-26-05-NVMeVirt](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-05-31-16-26-05-NVMeVirt.png)
![2024-05-31-16-26-43-NVMeVirt](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-05-31-16-26-43-NVMeVirt.png)

#### 2.4 ZenFS挂载

磁盘调度器设为deadline(貌似同一编号如nvme4n1只需设置一次, 更换另一配置的模拟zns不需该操作)

``` shell
sudo echo deadline > /sys/class/block/nvme4n1/queue/scheduler
```

挂载ZenFS，指定log和lock文件

``` shell
 sudo ./plugin/zenfs/util/zenfs mkfs --zbd=nvme4n1 --aux_path=/home/yzy/zone_256M_log
```

db_bench测试

``` shell
sudo ./db_bench --fs_uri=zenfs://dev:nvme4n1 --benchmarks=fillrandom --use_direct_io_for_flush_and_compaction
```

![2024-05-31-16-34-05-NVMeVirt](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-05-31-16-34-05-NVMeVirt.png)

![2024-05-31-16-33-46-NVMeVirt](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-05-31-16-33-46-NVMeVirt.png)

#### 2.3 参考

- [lzq的notion笔记](https://ziqi-rocks-on-zns.notion.site/1-NVMEvirt-NVMEvirt-RockDB-Zenfs-7292a6396ed84fc29010a8d0ed768d9b)
- [ZenFS+rocksdb+nvmevirt搭建ZNS SSD模拟环境](https://blog.csdn.net/yumWant2debug/article/details/131355223)

### 3 ZNS实验

实验环境：NVMeVirt+RocksDB+ZenFS
实验目的：分区大小与compection和GC的关系量化

#### 3.1 实验记录

- key-size:16 B
- value-size:100B

- 测试前使用`sudo nvme zns reset-zone nvme4n1 -a`将zns模拟盘上所有的分区重置
- fillseq

``` shell
sudo ./db_bench --benchmarks=fillseq,stats --num=100000000 --value_size=100 --compression_type=none --histogram --fs_uri=zenfs://dev:nvme4n1 --target_file_size_base=510027366 --use_direct_io_for_flush_and_compaction --max_bytes_for_level_multiplier=4 --max_background_jobs=8 --use_direct_reads --write_buffer_size=2147483648 --statistics > /home/yzy/rocksdb811/rocksdb-8.11.3/plugin/zenfs/zns_test/results/16GB_<zone size>_fillseq_flush_10G.out
```

- update

``` shell
sudo ./db_bench --benchmarks=updaterandom,stats --num=10000000 --value_size=100 --compression_type=none --histogram --fs_uri=zenfs://dev:nvme2n1 --target_file_size_base=510027366 --use_direct_io_for_flush_and_compaction --max_bytes_for_level_multiplier=4 --max_background_jobs=8 --use_direct_reads --write_buffer_size=2147483648 --statistics --use_existing_db=true --use_existing_keys=true > /home/yzy/rocksdb811/rocksdb-8.11.3/plugin/zenfs/zns_test/results/16GB_<zone size>_update_flush_100M.out
```

sudo ./db_bench --benchmarks=overwrite,stats --num=10000000 --value_size=100 --compression_type=none --histogram --fs_uri=zenfs://dev:nvme2n1 --target_file_size_base=510027366 --use_direct_io_for_flush_and_compaction --max_bytes_for_level_multiplier=4 --max_background_jobs=8 --use_direct_reads --write_buffer_size=2147483648 --statistics -- use_existing_db=true --use_existing_keys=true > /home/yzy/rocksdb811/rocksdb-8.11.3/plugin/zenfs/zns_test/results/16GB_64MB_overwrite_flush_100M.out


#### 按论文提供的配置进行实验

``` shell
./db_bench --fs_uri=zenfs://dev:nvme4n1 –benchmarks=fillrandom,stats –perf_level=3 -use_direct_io_for_flush_and_compaction=true -use_direct_reads=true -cache_size=268435456 -key_size=48 -value_size=43 -num=200000000 -db=./50M_random_insert_kv -disable_auto_compactions=true
```

``` shell
./db_bench --fs_uri=zenfs://dev:nvme4n1 –benchmarks=mixgraph,stats -use_direct_io_for_flush_and_compaction=true -use_direct_reads=true -cache_size=268435456 -keyrange_num=1 -value_k=0.2615 -value_sigma=25.45 -iter_k=2.517 -iter_sigma=14.236 -mix_get_ratio=0.83 -mix_put_ratio=0.14 -mix_seek_ratio=0.03 -sine_mix_rate_interval_milliseconds=5000 -sine_a=1000 -sine_b=0.000073 -sine_d=4500 –perf_level=2 -reads=420000000 -num=50000000 -key_size=48 -db=./50M_random_insert_kv -use_existing_db=true -disable_auto_compactions=true
```

- Zenfs
- fillrandom   :       2.685 micros/op 372401 ops/sec 268.527 seconds 100000000 operations;   52.6 MB/s

### NVMeBenchmarks

- `sudo python3 grid_bench.py -d nvme4n1 -m zns-a -l lbaf0 -f write -s /home/yzy/NVMeBenchmarks/submodules/spdk-22.09 --overwrite 1`