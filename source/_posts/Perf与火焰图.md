---
title: Perf系统调优
tag: [Perf, FlameGraph, Tuning]
index_img: https://raw.githubusercontent.com/Yee686/Picbed/main/2025-04-22-20-11-10-Perf与火焰图.svg
date: 2025-04-22 20:00:00
category: 工具与框架
---

# Perf与火焰图

- [FlameGraph--将采样信息转为火焰图](https://github.com/brendangregg/FlameGraph)
- [Perf介绍与使用](https://www.cnblogs.com/arnoldlu/p/6241297.html): 通过`perf list`可查看所以支持性能事件
  - 全局性概况：
  > perf list查看当前系统支持的性能事件；
  > perf bench对系统性能进行摸底；
  > perf test对系统进行健全性测试；
  > perf stat对全局性能进行统计；
  - 全局细节：
  > perf top可以实时查看当前系统进程函数占用率情况；
  > perf probe可以自定义动态事件；
  > 特定功能分析：
  > perf kmem针对slab子系统性能分析；
  > perf kvm针对kvm虚拟化分析；
  > perf lock分析锁性能；
  > perf mem分析内存slab性能；
  > perf sched分析内核调度器性能；
  > perf trace记录系统调用轨迹；
  - 最常用功能
  > perf record，可以系统全局，也可以具体到某个进程，更甚具体到某一进程某一事件；可宏观，也可以很微观。
  > pref record记录信息到perf.data；
  > perf report生成报告；
  > perf diff对两个记录进行diff；
  > perf evlist列出记录的性能事件；
  > perf annotate显示perf.data函数代码；
  > perf archive将相关符号打包，方便在其它机器进行分析；
  > perf script将perf.data输出可读性文本；
  - 可视化工具perf timechart
  > perf timechart record记录事件；
  > perf timechart生成output.svg文档；

- 写成脚本后, 老是阻塞, 最后还是手动运行相关命令, 大致流程如下
  - 启动待采样程序命令 `sudo ./db_bench`
  - 获取进程对应PID `ps aux | grep db_bench`, 注意看CPU利用率选择实际进程(340780 74.3)
  
  ``` shell
  yzy@host1:~/liza-rocksdb/rocksdb-7.10.2/plugin/zenfs/util$ ps aux | grep db_bench
  root      340778  0.0  0.0  11508  5828 pts/1    S+   11:48   0:00 sudo ./db_bench 
  root      340779  0.0  0.0  11508   888 pts/9    Ss   11:48   0:00 sudo ./db_bench 
  root      340780 74.3  0.0 471260 200608 pts/9   Sl+  11:48   0:02 ./db_bench 
  yzy       340819  0.0  0.0   6612  2288 pts/2    R+   11:48   0:00 grep --color=auto db_bench
  ```
  
  - 启动Perf采样 `sudo perf record -e cpu-clock -g -p 339940 -o ./perf_data/saza/saza.data`
    - 是否使用sudo结合待采样进程而定
    - `-e cpu-clock`指标为cpu周期
    - `-g`使用函数调用视图
    - `-p`指定进程PID
  - 可通过Report模式查看结果`sudo perf report -i perf.data`
  - 将Perf采样结果转为火焰图, 大致流程是解析为`unfold`, 再进行折叠为`folded`, 最后生成`svg`
  - 执行脚本如下

  ``` python
  #!/usr/bin/env python3
  import subprocess
  from pathlib import Path

  # 配置参数
  FLAMEGRAPH_DIR = "/home/yzy/FlameGraph/"  # FlameGraph工具路径
  INPUT_FILE = "./perf_data/saza/perf.data"    # 输入的perf.data文件路径
  OUTPUT_DIR = "./perf_data/saza/"      # 输出目录

  def check_dependencies():
      """检查必要工具是否存在"""
      required_tools = [
          ("perf", "sudo perf --version"),
          ("FlameGraph", f"test -d {FLAMEGRAPH_DIR}")
      ]
      
      for name, cmd in required_tools:
          try:
              subprocess.run(cmd.split(), check=True, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
          except (subprocess.CalledProcessError, FileNotFoundError):
              print(f"错误: 未找到 {name} 工具")
              if name == "FlameGraph":
                  print("请从 https://github.com/brendangregg/FlameGraph 克隆仓库")
              return False
      return True

  def generate_flamegraph():
      """生成火焰图"""
      # 确保输出目录存在
      Path(OUTPUT_DIR).mkdir(parents=True, exist_ok=True)
      
      print("开始生成火焰图...")
      
      try:
          # 步骤1: 将perf.data转换为文本格式
          print("转换perf.data为文本格式...")
          with open(f"{OUTPUT_DIR}/out.perf", "w") as f:
              subprocess.run(
                  ["sudo", "perf", "script", "-i", INPUT_FILE],
                  stdout=f,
                  check=True
              )
          
          # 步骤2: 折叠堆栈
          print("折叠堆栈信息...")
          subprocess.run(
              [f"{FLAMEGRAPH_DIR}/stackcollapse-perf.pl", f"{OUTPUT_DIR}/out.perf"],
              stdout=open(f"{OUTPUT_DIR}/out.folded", "w"),
              check=True
          )
          
          # 步骤3: 生成SVG火焰图
          print("生成SVG火焰图...")
          subprocess.run(
              [f"{FLAMEGRAPH_DIR}/flamegraph.pl", f"{OUTPUT_DIR}/out.folded"],
              stdout=open(f"{OUTPUT_DIR}/flamegraph.svg", "w"),
              check=True
          )
          
          print(f"火焰图已生成: {OUTPUT_DIR}/flamegraph.svg")
          return True
      
      except subprocess.CalledProcessError as e:
          print(f"生成火焰图失败: {e}")
          return False

  if __name__ == "__main__":
      if not check_dependencies():
          exit(1)
          
      if not Path(INPUT_FILE).exists():
          print(f"错误: 输入文件 {INPUT_FILE} 不存在")
          exit(1)
          
      generate_flamegraph()
  ```
	
	- 得到交互式火焰图如下所示
  
  ![2025-04-22-20-11-10-Perf与火焰图](https://raw.githubusercontent.com/Yee686/Picbed/main/2025-04-22-20-11-10-Perf与火焰图.svg)
