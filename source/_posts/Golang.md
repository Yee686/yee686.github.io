---
title: Golang学习笔记
tag: [golang]
index_img: /img/golang.png
date: 2024-11-10 20:00:00
category: 工具与框架
---

# Golang

## 3 Go语言进阶与依赖管理

### 并发

- 线程: 内核态, 线程跑多个协程, 栈空间MB级
- 协程: 用户态, 轻量级线程, 栈空间KB级
- channel 通道: make(chan 元素类型, [缓冲大小(>=2为有缓冲通道)])
- lock 并发安全: sync.Mutex定义互斥锁
- WaitGroup 等待组: sync.WaitGroup定义等待组, `Add(delta uint)`计数器增加delta, `Done()`计数器减1, `Wait()`阻塞直到WaitGroup为0

### 依赖管理

- `GOPATH`管理模式: 项目代码直接依赖src下代码, go get将最新版本的包加到src下
  - `./bin` 项目编译的二进制文件
  - `./pkg` 项目编译的中间产物
  - `./src` 项目源码
  - 缺点: 无法实现pkg的多版本控制
- `Go Vender`管理模式: 在项目目录下增加vender文件夹, 所有依赖包以副本形式存放于`$ProjectRoot/vender`, 首先vender=>GOPATH
  - 当一个项目直接/间接依赖了同一个包的多个版本时, 出现不兼容, 还是由于vender无法控制包的依赖版本
- `Go Module`管理模式: 1.16开始使用, 定义版本规则和项目依赖关系
  - `go.mod`: 管理依赖包版本
  - `go get/go mod`指令来管理依赖包
- 依赖管理三要素
  - 配置文件描述依赖 go.mod
  - 中心仓库管理依赖库 Proxy
  - 本地工具 go get/mod
- 依赖配置
  - ![2024-11-10-20-21-04-字节Go](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-11-10-20-21-04-字节Go.png)
  - version
    - 语义化版本 `${MAJOR}.${MINOR}.${PATCH}`
    - 基于commit的伪版本 `${MAJOR}.${MINOR}.${PATCH}-${timestamp}-${hash-checksum}`
  - indirect 间接依赖
  - incompatible 主版本2+模块需要加/vN后缀, 如果没有go.mod文件且主版本为2+的依赖要使用incompatible
- 依赖分发-Proxy: 缓存托管平台分发版本, 保证依赖的稳定可靠
  - ![2024-11-10-20-30-55-字节Go](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-11-10-20-30-55-字节Go.png)
  - `GOPROXY="URL_ADDRESS1, URL_ADDRESS2,..., direct"`, 从前往后找, direct为源站
- 工具
  - go get 下载并安装指定版本的包和依赖包
    - `@update`默认方法/`@none`删除依赖/`@vN`指定语义版本
  - go mod
    - `init` 初始化创建go.mod
    - `download` 下载模块到本地缓存
    - `tidy` 添加缺少的模块, 并删除不需要的模块

## 3 Go语言工程实践之测试

### 测试

- 单元测试
  - 规则: 以`_test.go`结尾, 函数声明以`func TestXxxx(t *trsting.T)`开头, 初始化逻辑在`TestMain()`中
  - 运行: `go test [flags] [packages]`
  - 覆盖率: 执行代码行的比例, 一般50%以上, 较高80%以上
- Mock测试: 保证测试的稳定不受外部依赖(network/file/DB)的影响
    - `monkey`为例: 为函数或方法打桩,`Patch`/`Unpatch`方法, 运行时替换函数/方法
- 基准测试
  - 规则: 以`_benchmark.go`结尾, 函数声明以`func BenchmarkXxxx(b *testing.B)`开头
  - 运行 `go test -bench=. `

### 实践

- 数据层: 数据model, 外部数据的增删改查
- 逻辑层: 业务Entity, 处理核心业务逻辑输出
- 视图层: 视图View, 处理和外部的交互逻辑

## 5 高质量编程简介及编码规范 && 性能优化 && 性能优化分析

### 高质量编程

- gofmt 自动格式化代码
- 编码规范
  - 包以全小写命名
  - 保持正常代码路径的最小缩进量
  - 用errors.Is批判error是否为特定错误, 将检查整条错误链, 看是否包含; 使用errors.As获取特定错误
  - panic不建议在业务代码中使用; recover在被defer的函数中使用
- 性能优化建议
  - slice预分配内存, make时提供初始容量信息`data := make([]int, size)`, slice本质是指针+长度+容量, 创建新切片时会服用原slice的底层数组; 在大内存slice上做小切片会使得原底层数无法释放, 使用copy代替re-slice
  - ![2024-11-16-15-03-01-字节Go课程学习](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-11-16-15-03-01-字节Go课程学习.png)
  - ![2024-11-16-15-03-17-字节Go课程学习](https://raw.githubusercontent.com/Yee686/Picbed/main/2024-11-16-15-03-17-字节Go课程学习.png)
  - map预分配内存, `mp := make(map[int]int, size)`, 添加元素会触发map的扩容, 预留内存减少内存拷贝和重哈希
  - 字符串拼接使用`strings.Builder`, 性能比`+`好, 因为string类型内存固定, `+`需要重新分配内存, 而`strings.Builder`底层为`[]byte`数组, 不会每次都分配
  
  ``` go
    var builder strings.Builder
    for i := 0; i < 100; i++ {
    	builder.WriteString("hello")
    }
    return builder.String()

    buf := new(bytes.Buffer)
    for i := 0; i < 100; i++ {
    	buf.WriteString("hello")
    }
    return buf.String()
  ```
  - 空结构体节省内存, 因为空结构体不占用内存空间, 如仅需要key时使用`map[int]struct{}`来实现`set`
  - `atomic`包提供原子操作, 比对临界资源加锁(`sync.Mutex`)效率更高, 因为atomic基于底层硬件实现, 锁基于OS实现, 加锁应使用在一段逻辑上而不是某一变量

### 性能调优

- 性能调优原则: 依靠数据, 定位瓶颈, 不要过度/过早优化
- Go能分析工具pprof, 性能分析监听端口为6060`localhost:6060/debug/pprof`
  - 显示上, 支持top命令, 调用图, 火焰图, 定位源码等
  - 采样指标上, 支持CPU, 堆heap, 协程, Mutex锁, 阻塞block等, 通过`go tool pprof -http=:8080 "localhost:6060/debug/pprof/heap"`, 通过8080端口启动可视化
  - top显示中, flat代表当前函数自身耗时, cum担任函数加上调用函数的总耗时