+++
date = '2025-11-16T12:00:00+08:00'
draft = false
title = 'Linux 环境下的 CH32 + LibXR 开发环境搭建'
author = "molqzone"
tags = ["CH32", "嵌入式", "CMake", "XRobot"]
+++

笔者最近在研究国产单片机。大家对国产单片机的固有印象可能还停留在一比一复刻 STM32，但是随着国产单片机产业的蓬勃发展，各个国产单片机也在自己的产品中做出了自己的特色，其中沁恒家的 RISC-V 系列单片机我最近比较感兴趣（因为沁恒真的敢送）。

说到沁恒，相信大家对他们家 CH340 这款经典的 USB 转串口芯片并不陌生。而在单片机领域，沁恒同样展现出强大的技术实力，尤其在USB功能方面独树一帜：既有经济实用、集成 USB2.0 接口的 V203 系列，也有搭载高速 480MHz USB PHY 的 V307 系列，为嵌入式开发者提供了丰富的选择。

LibXR 是一个功能强大的跨平台 C++ 开发框架，集成了丰富的外设驱动、数据结构、通信中间件、操作系统封装和数学工具库。它为 CH32 系列单片机提供了一个兼容层，不仅对 CH32 标准库进行了高层次的抽象封装，还修复了原库中的一些已知问题，大大提升了开发效率和代码质量。

本文将详细介绍如何在Linux环境下搭建基于 CH32 单片机和 LibXR 库的完整开发环境，帮助开发者快速上手这一优秀的开发组合。

## 环境准备

### 1. 获取项目模板

LibXR 官方提供了现成的项目模板，支持 CH32V307 和 CH32V203 两种型号，已预配置好构建脚本和调试配置，无需手动设置。

```bash
# 克隆对应芯片的模板项目
git clone https://github.com/xrobot-org/CH32V307_LibXR_Template.git
# 或者选择 CH32V203 模板
# git clone https://github.com/xrobot-org/CH32V203_LibXR_Template.git

# 初始化 LibXR 子模块
git submodule add https://github.com/Jiu-xiao/libxr
```

### 2. 安装调试工具链

CH32 芯片需要专用的调试工具。从[MounRiver 官网](https://www.mounriver.com/download)下载 Linux 版工具链：

1. 访问下载页面，选择 Linux 平台
2. 在"工具链和调试器"栏目下载 **MRS_Toolchain_Linux_x64_Vxxx.tar.xz**
3. 解压到合适的目录（建议 `~/Development/` 目录下）

![MRS 工具链下载](mrs-install.png)

```bash
# 创建开发目录并解压工具链
mkdir -p ~/Development/
cd ~/Development/
tar -xf MRS_Toolchain_Linux_x64_Vxxx.tar.xz  # 根据实际文件名调整
```

**工具链说明：**
- **专用 OpenOCD**：用于 CH32 芯片烧录和调试（开源版本无法支持 CH32）
- **RISC-V Embedded GCC**：沁恒提供的编译器，支持 WCH 扩展指令集，但对 C++ 标准支持不完整（*LibXR 官方不推荐此编译器*）

### 3. 编译推荐编译器

LibXR 推荐使用上游 RISC-V 工具链，具有完整的 C++ 标准支持，可以正常使用 Eigen 等现代 C++ 库。

#### 3.1 获取源码

```bash
git clone https://github.com/riscv-collab/riscv-gnu-toolchain.git
cd riscv-gnu-toolchain
```

#### 3.2 配置编译选项

针对 CH32 芯片特性进行配置：

```bash
./configure --prefix=~/Development/riscv-ch32 \
            --with-arch=rv32imacf_zicsr_zifencei \
            --with-abi=ilp32f
```

**配置说明：**
- `--prefix`：指定安装目录
- `--with-arch`：目标架构，兼容 CH32 指令集（不包含 WCH 私有扩展）
- `--with-abi`：应用程序二进制接口，支持单精度浮点

#### 3.3 开始编译

```bash
make -j$(nproc)  # 使用多核编译加速
```

编译完成后，工具链将安装到 `~/Development/riscv-ch32/` 目录。

## VSCode 开发环境配置

### 1. 安装必要插件

在 VSCode 中安装以下插件：

- **C/C++**：代码智能提示和语法高亮（注意：clangd 对 RISC-V 支持有限）
- **CMake Tools**：CMake 项目管理和构建
- **Cortex Debug**：OpenOCD 调试支持

### 2. 配置编译器

1. 使用快捷键 `Ctrl + Shift + P`
2. 搜索并选择 `CMake: Configure`
3. 在编译器列表中选择刚刚编译的 RISC-V 工具链：`/home/keruth/Development/riscv-ch32/bin/riscv32-unknown-elf-gcc`

### 3. 配置调试器

1. 打开设置界面（`Ctrl + ,`）
2. 切换到 `Workspace` 选项卡
3. 导航至 `Extensions` → `Cortex Debug` → `External GDB Servers`
4. 找到 `Cortex-debug: Openocd Path` 配置项
5. 点击 `Edit in settings.json`，添加以下配置：

```json
{
    "cortex-debug.openocdPath": "/home/keruth/Development/mrs-toolchain/OpenOCD/OpenOCD/bin/openocd"
}
```

## 验证开发环境

### 1. 编译测试

在 VSCode 中：
1. 打开命令面板（`Ctrl + Shift + P`）
2. 选择 `CMake: Build` 或点击状态栏的构建按钮
3. 确认编译过程无错误完成

### 2. 调试测试

1. 使用 WCHLinkE 连接 CH32 开发板到电脑
2. 打开调试界面（`Ctrl + Shift + D`）
3. 选择模板项目预设的调试配置
4. 点击运行按钮，确认能够正常烧录和调试

如果编译和调试都能正常工作，说明开发环境已经搭建完成，可以开始你的 CH32 + LibXR 开发之旅了！
