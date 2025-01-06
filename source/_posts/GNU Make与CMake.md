---
title: GNU Make与CMake
tag: [c/c++, CMake, Make]
index_img: /img/toolchain.png
date: 2024-09-26 20:00:00
updated: 2025-01-06 16:00:00
category: 工具与框架
---

# CMake和Make

## Make与Makefile

### 基本结构与规则

```makefile
target ... : prerequisites ...
    recipe
    ...
    ...
```

- target: 可以是目标文件、可执行文件或者标签
- prerequisites：生成对应target对应的文件或者其他target
- recipe：生成target的命令，可以是任意的shell命令
- 变量：通过`var = value1 value2 ...`声明，在使用时通过`$(var)`引用
- 自动推理：make在处理`.o`文件时会自动推理出对应的`.c`文件，并加入依赖关系中
- 每个Makefile都应该有一个清空目标文件和可执行文件的规则,`.PHONY`声明一个伪目标，不会生成对应的文件，`-`则表示强制执行，忽略错误

  ``` makefile
  .PHONY : clean
  clean :
      -rm edit $(objects)
  ```

- 注释使用`#`标识，命令必须以`tab`开头
- Makefile可以使用`include`命令包含其他文件。`include <filenames>...`，不能以`tab`开头
  - 有`-I`或者`--include-dir`时，首先搜索该目录下的文件
  - 然后搜索`<prefix>/include`多为`usr/local/bin`、`usr/gnu/include`、`usr/local/include`、`usr/include`目录下的文件
- `$(CXX)` 自带变量, 表示编译器, 默认为系统默认编译器

### 编写规则

- 第一条规则一般为最终目标
- 支持的通配符有`*`、`?`、`~`,如果变量中使用了通配符，而在命令总需要展开，使用`obj := (wildcard *.c)`声明变量
- `$(patsubst pattern,replacement,text)`，将text中的pattern替换为replacement
- `VPATH`变量指定搜索路径，以冒号依次分隔多个路径，`VPATH = src:../headers`
- `path`关键字可以灵活指定搜索路径，pattern需要包含%以匹配任意字符
  - `vpath <pattern> <dir>`，将符合pattern文件指定dir路径
  - `vpath <pattern>`清除pattern的搜索路径
  - `vpath`清除所有搜索路径
- `.PHONY: label`声明一个伪目标，不会生成对应的文件, clean使用
- `$@`是自动化变量，多用用户多目标的规则中
- `$(subst from,to,text)`,在text中查找所有出现的from字符串，并将其替换为to字符串。
- `$(filter pattern, text)`,函数会遍历text中的每个元素，并将符合pattern模式的元素返回作为结果
- 以`@`开头的命令不会将命令打印到终端
- 如果要让上一条命令的结果应用在下一条命令时，应该使用分号分隔这两条命令
- `:=`比`=`更加安全高效
- `override`对命令行参数值进行修改覆盖
- `make -e`设定环境变量并覆盖makefile中的定义
- 条件判断
  - `ifeq` `ifneq` `ifdef`
  - `else` `endif`
- [常用函数](https://seisman.github.io/how-to-write-makefile/functions.html)
  - `$(strip <string>)` 去除空格
  - `$(word <n>,<text>)` 取单词
  - `$(dir <names...>)`和`$(notdir <names...>)`取目录名和文件名
  - `$(call <expression>,<parm1>,<parm2>,...,<parmn>)`创建新函数

### make的指令与使用

- `-B`或者`--always-make`，重写编译
- `-C`或者`--directory`，指定目录
- `-f`或者`--file`，指定makefile文件
- `-I`或者`--include-dir`，指定搜索路径，可以有多个
- `-n` `--just-print` `--dry-run` `--recon`，打印命令，但不执行

## Cmake

- CMakeLists.txt示例

``` CMake
cmake_minimum_required(VERSION 3.10)

# 项目名称
project(zbd_demo)

# 设置编译器标准和编译选项
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED True)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -g")

# 查找 libzbd 库
find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBZBD REQUIRED libzbd)

# 定义子文件夹
add_subdirectory(util)

# 定义可执行文件
add_executable(zbd_demo main.cc)

# 链接 libzbd 库
target_link_libraries(zbd_demo PRIVATE ${LIBZBD_LIBRARIES} zbdutil)

# 添加头文件路径
target_include_directories(zbd_demo PRIVATE ${LIBZBD_INCLUDE_DIRS})
```

### 执行

- `cmake -B path`: `-B`参数指定生成Makefile及其他文件的路径
- `cmake -G`参数指定生成器，如`make`、`ninja`、`Visual Studio`等
- `cmake --build path`: 执行构建命令, 好处是不管使用的是什么生成器, 统一执行

### 语法

- `add_executable(target source1 source2 ...)`：添加构建可执行文件, 直接添加依赖的CC文件, 不用单独处理`.h`与`.o`
- `add_library(target [STATIC] source1 source2...)`：添加构建库文件, `STATIC`表示编译为静态链接库
- `target_include_directories(target PRIVATE include_dir1 include_dir2...)`: 添加头文件路径, 当为PUBLIC时外部可以看到该路径, 当为PRIVATE时仅在该target中可见
- `target_link_libraries(target library1 library2...)`：添加链接库, 对于自己常用的函数可以编译为链接库, 然后再生成可执行文件时直接链接该库
- `message(STATUS info)`用于输出调试信息或错误信息, 除了STATUS还有DEBUG, WARNING等
- `find_package XXXX REQUIRED`查找第三方库, `REQUIRED`表示找不到时会报错, XXXX是包的CMAKE命名, 为了简化记忆建议使用`Pkg-config`管理
  - `find_package(PkgConfig REQUIRED)`
  - `pkg_check_modules(NAME REQUIRED package)`
- `set(XXX value CACHE TYPE "message")`定义cache变量
  - `TYPE`常用的有`STRING`和`BOOL`, `BOOL`用`OFF`和`ON`表示
  - 变量可以通过`-D`参数设置, 如`cmake -DXXX=value`
- `target_compile_definitions(target PRIVATE DEF1 DEF2...)`：添加编译定义, 当为PUBLIC时外部可以看到该定义, 当为PRIVATE时仅在该target中可见, 定义的宏可以通过`${XXX}`引用, 如果是`STRING`类型需要加上双引号
  - `target_compile_definitions(libanswer PRIVATE VTYPE="${VTYPE}")`
- `add_library(libxxxx INTERFACE)`: Header-only的库可以添加为`INTERFACE`库, 后续为该库通过`target_XXX`添加属性时都需要使用`INTERFACE`
  
  ``` cmake
  add_library(libanswer INTERFACE)
  target_include_directories(libanswer INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/include)
  ```

- 编译设置
  - `set(CMAKE_CXX_STANDARD 14)`: 设置C++标准, 全局设置
  - `target_compile_features(target INTERFACE cxx_std_17)`: 为target设置编译特性, 更加细粒度

## 参考

- [全网最细的CMake教程](https://zhuanlan.zhihu.com/p/534439206)
- [跟我一起写Makefile](https://seisman.github.io/how-to-write-makefile/overview.html)
- [上交IPADS一小时CMake培训](https://www.bilibili.com/video/BV14h41187FZ/?vd_source=a4ad9d5a0b5f697bd7bd62602e8dc7f9)
