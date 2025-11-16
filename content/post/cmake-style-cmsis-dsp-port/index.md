+++
date = '2025-07-20T01:12:48+08:00'
draft = false
title = '手把手教你移植 CMSIS-DSP 到 STM32CubeMX 生成的 CMake 项目'
author = "molqzone"
tags = ["STM32", "嵌入式", "CMake"]
+++

在使用 STM32 系列单片机进行信号处理的过程中，我们往往会选择 ARM 提供的 CMSIS-DSP 库。CMSIS-DSP库涵盖了嵌入式信号处理的大部分常用算法函数，同时针对 Cortex-M 核心做了手工汇编优化，还提供了统一的接口。目前使用 CMSIS-DSP 库有以下几种方案：

1. Keil 项目 + 封装好的 CMSIS-DSP 库 CMSIS-Pack
2. STM32CubeMX CMake 项目 + Software Components
3. STM32CubeMX CMake 项目 + CMSIS-DSP 源码

其中后两种方案更现代化，可以适配 STM32 for Visual Studio Code、CLion 等现代开发环境。而现在（2025 年 7 月）通过 Software Components 安装的 DSP Library 版本还停留在 v1.4.0，这是 2013 年 1 月发布的 CMSIS-DSP 版本，距今已超过十年。这十年里 ARM 为 CMSIS-DSP 追加了更多Cortex-M 架构的支持，新增了**窗函数**等功能模块。而第三种方案直接从 GitHub 上抓取源码，版本最新，功能最全面。但是需要一些 CMake 配置，没有第二种方案直接快捷。而这篇文章的使命就是带你将 GitHub 上的 CMSIS-DSP 源码加入到 STM32CubeMX 生成的 CMake 项目中。（下图是第二种方法通过 Software Components 配置 CMSIS-DSP）

![cmsis-dap-in-stm32cubemx](cmsis-dap-in-stm32cubemx.png)

## 第一步：从 GitHub 下载源码并解压

我们有许多种下载源码的办法，其中每一种都需要科学上网：

- 安装 Git 后在命令行输入（无需解压）：
```shell
git clone https://github.com/ARM-software/CMSIS-DSP.git
```

- 在 Release 页的 Assets 里找到 Source code (zip) 下载
- 在 Code 窗口选择 Download ZIP

## 第二步：将源码加入到自己的项目中

直接将解压下来的文件复制到项目文件夹中。我删除目录的版本号后复制到了项目文件夹下的Drivers文件夹下，下面是我现在的项目结构：

```
Project/
├── .gitattributes          # Git属性文件，用于定义特定路径的属性
├── .gitignore              # Git忽略文件，指定哪些文件或目录不被版本控制
├── .mxproject              # STM32CubeMX项目文件，存储IDE特定的配置
├── *.ioc       # STM32CubeMX配置文件，核心文件，定义了MCU的引脚、时钟和中间件配置
├── build/                  # 编译输出目录，存放编译生成的目标文件和可执行文件（如.elf, .bin）
├── cmake/                  # 可能包含自定义的CMake脚本或模块
├── CMakeLists.txt          # 主CMake构建脚本，定义了整个项目的构建规则
├── CMakePresets.json       # CMake预设文件，用于配置常用的构建选项
├── Core/                   # 核心代码目录
│   ├── Inc/                # 存放核心代码的头文件 (.h)，例如 main.h, stm32h7xx_it.h
│   └── Src/                # 存放核心代码的源文件 (.c)，例如 main.c, stm32h7xx_it.c
├── Drivers/                # 驱动程序目录
│   ├── CMSIS/              # ARM Cortex微控制器软件接口标准，包含MCU核心访问文件
│   ├── CMSIS-DSP/          # CMSIS提供的数字信号处理库
│   └── STM32*xx_HAL_Driver/ # ST官方提供的硬件抽象层(HAL)驱动库，用于简化对MCU外设的操作
├── LICENSE                 # 项目的开源许可证文件
├── Middlewares/            # 中间件目录，例如FreeRTOS, FatFS, USB等
├── README.md               # 项目说明文件，使用Markdown格式
├── startup_*xx.s   # 启动文件(汇编)，MCU上电后最先执行的代码，用于初始化堆栈和中断向量表
├── *_FLASH.ld    # 链接器脚本(Linker Script)，告诉链接器如何组织代码和数据在内存中的布局
└── USB_DEVICE/             # USB设备库相关文件
```

## 第三步：将 CMSIS-DSP 目录加入到项目 CMakeLists 中

> [!WARNING] 
STM32CubeMX v6.15.0 前，项目 CMakeLists 中非 User defined 注释下的内容会在生成时被覆盖。v6.15.0 版本更新后，用户可以自由更改 CMakeLists.txt，STM32CubeMX 只会在项目第一次生成时构建 CMakeLists.txt 模板。本教程的操作依赖自由更改 CMakeLists.txt，故请确认自己的 STM32CubeMX 是否为 v6.15.0 及更高版本！

CMSIS-DSP 根目录下有一个引用了整个库的 CMakeLists.txt，我们需要将其加入到我们的项目构建中：

```cmake
# Add CMSIS-DSP sources
add_subdirectory(Drivers/CMSIS-DSP)
```

同时我们需要在 target_link_libraries 中加入 CMSISDSP 库，保证项目链接到 CMSIS-DSP 库

```cmake
# Add linked libraries
target_link_libraries(${CMAKE_PROJECT_NAME}
    stm32cubemx
​
    # Add user defined libraries
        CMSISDSP
)
```

理论上到这里就已经结束了，但是我们此时编译会发现报错：找不到cmsis_compiler.h。这是因为 CMSIS-DSP 依赖 CMSIS-Core，而 CMSIS-Core 是 STM32CubeMX 生成时携带的，就在Drivers/CMSIS里。（CMSIS-Core 的路径可以通过在项目中找哪里有cmsis_compiler.h来找到）如果我们在项目 CMakeLists.txt 加入：

```cmake
# Add include paths
target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
    # Add user defined include paths
        /Drivers/CMSIS/Include
)
```

会发现依然报错，因为 CMSIS-DSP 库中的内容并不受项目的 CMakeLists.txt 管理，而是由库中的 CMakeLists.txt 管理。山重水复疑无路，回头看 CMSIS-DSP 的 README，才发现答案人家已经给出了——

## 第四步：在 CMakePresets.json 加入 CMSISCORE 定义

在 README 中写到：

> CMSIS-DSP is dependent on the CMSIS Core includes. So, you should define CMSISCORE on the cmake command line. The path used by CMSIS-DSP will be ${CMSISCORE}/Include.

因此我们需要给 CMake 传入 CMSISCORE 的定义。传入的方法就是在CMakePresets.json中找到"default"项，在其中的"cacheVariables"加入 CMSIS-Core 的路径：

```json
"configurePresets": [
    {
        "name": "default",
        "hidden": true,
        "generator": "Ninja",
        "binaryDir": "${sourceDir}/build/${presetName}",
        "toolchainFile": "${sourceDir}/cmake/gcc-arm-none-eabi.cmake",
     "cacheVariables": {
         "CMSISCORE": "${sourceDir}/Drivers/CMSIS"
     }
 }
]
 ```

此时清除缓存重新编译，会看到 CMSIS-DSP 库已经正常引入到了我们的项目中。大功告成！

