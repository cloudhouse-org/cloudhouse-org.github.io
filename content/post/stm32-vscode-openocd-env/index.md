+++
date = '2025-11-22T12:00:00+08:00'
draft = false
title = '手把手教你配置 STM32CubeIDE for VSCode 下基于 OpenOCD 的调试配置'
author = "molqzone"
tags = ["STM32", "嵌入式", "入门"]
+++

如果你在使用 STM32CubeIDE for VSCode 插件，可能会发现它默认的烧录工具只完美支持正版 STLink。市面上几十块钱的仿制 STLink 经常出现连接不稳定、烧录失败、甚至无法识别的“玄学”问题。

为了彻底解决这个问题，我们推荐使用 DAPLink 调试器配合 OpenOCD。这不仅能解决烧录问题，还能让你拥有一套适用于几乎所有 ARM 芯片（不仅限于 STM32）的通用开发环境。本教程将以 STM32CubeIDE for VSCode 生成的项目为基础，教你如何“外挂” OpenOCD 来接管烧录和调试工作。

> 本教程虽然以 STM32CubeIDE for VSCode 为例，但同样适用于任何基于 OpenOCD 的工具链配置，只需要稍作修改。

> [!warning]
>
> 本教程涉及 Scoop 的安装。安装 Scoop 前需要检查自己的用户文件夹是否含有中文，如果有中文会**导致安装后无法正常使用，且卸载复杂**的后果。可以修改默认用户文件夹或重装系统，建议重装系统。

## 硬件准备

### 选购 DAPLink

市场上 5-20 元的 DAPLink 基本就可以满足日常使用需求。可以买一个淘宝九块九的 **PowerWriter 2 Lite**，性价比高。

>  [!tip]
>
> **Why DAPLink？**
>
> **DAPLink** 是 ARM 官方推出的开源调试协议。与 STLink（仅支持 STM32）不同，DAPLink 几乎可以覆盖所有 ARM 内核的设备，通用性更强。

### 检查驱动 (WinUSB)

大多数 DAPLink 插上就能用。但为了确保 OpenOCD 能通过 `libusb` 访问它，建议检查一下：
1. 插上调试器。
2. 右键此电脑，在右键菜单选择 `管理`。打开设备管理器，看是否有 `CMSIS-DAP` 或 `WinUSB` 设备。

## 安装 OpenOCD

OpenOCD 可以通过 Scoop 安装或者在 GitHub Release 直装（直装后建议添加 OpenOCD 到环境变量）。

推荐使用 [Scoop](https://scoop.sh/) 安装 OpenOCD，方便后续的维护

1. 在开始菜单搜索 `PowerShell`，启动 PoweShell
2. 安装 Scoop

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression
```

3. 安装 OpenOCD

```powershell
scoop install openocd
```

4. 安装完成后，检查 OpenOCD 是否正确安装

```powershell
openocd --version
```

若显示版本号说明安装成功。如：

```
Open On-Chip Debugger 0.12.0 (2023-01-14-23:37)
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
```



> [!tip]
>
> 你的电脑无法直接对芯片说：“把这个程序写进你的 Flash 里”，所以我们需要引入一个程序用来把用户的调试操作转换成芯片能听懂的指令，这个程序在这儿教程中就是 **OpenOCD**。它一边听懂电脑发出的命令（GDB协议），一边控制 USB 调试器发出高低电平（JTAG/SWD协议），像“牵线木偶”一样控制芯片。 它的核心价值就是用一套软件，桥接不同调试器（DAPLink/STLink）与不同芯片，让烧录和调试变得通用。

## 配置烧录功能

VSCode 本身只是一个编辑器，并不懂烧录，我们需要写一个“任务说明书”告诉它该怎么做。

### 创建烧录任务

1. 在VSCode中打开你的项目
2. 按 `Ctrl + Shift + P`，输入 `Tasks: Configure Task`
3. 选择 `Create tasks.json file from template` → `Others`
4. **完全替换**文件内容如下（注意看注释修改芯片型号）:

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Flash Target",
            "type": "shell",
            "command": "openocd",
            "args": [
                "-f", "interface/cmsis-dap.cfg", // 使用 DAPLink
                "-f", "target/stm32f1x.cfg", // 根据你的芯片修改！例如 stm32f4x.cfg, stm32h7x.cfg
                // 下面这行代码会自动获取 STM32CubeIDE 插件编译出的 elf 文件路径,如果环境不是基于 STM32CubeIDE for VSCode 则需要将环境变量替换为具体可执行文件位置，具体可以问 AI
                "-c", "program ${command:st-stm32-ide-debug-launch.get-projects-binary-from-context1} verify reset exit" 
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "presentation": {
                "echo": true,
                "reveal": "always"
            }
        }
    ]
}
```



### 开始烧录

配置完成后，按 `Ctrl + Shift + P`，输入 "Run Task"，选择 "Flash Target" 即可开始烧录。



### 添加一键烧录按钮

不想每次按快捷键？装个插件做个按钮。

`Ctrl + Shift + X`，打开扩展页，搜索 `Task Buttons` 并安装

![VSCode Task Buttons 扩展安装页面](vscode-task-buttons-extension.png)

`Ctrl + ,` 打开设置页，在 `Extensions` -> `VsCodeTaskButtons` 找到 `Tasks`，选择 `Edit in settings.json`

![VSCode Task Buttons 设置页面配置](vscode-task-buttons-settings.png)

在 `settings.json` 追加：

```json
    "VsCodeTaskButtons.tasks": [
        {
            "label": "$(play) Flash Target",
            "task": "Flash Target",
            "tooltip": "Flash the ordered MCU"
        }
    ],
```

> [!tip]
>
> **task.json 本质上是一张“自动化任务说明书”**
>
> VSCode 只懂得编辑文字。你想让它帮你做点“额外的事”，比如烧录程序，但它自己完全不知道“烧录”是什么。
>
> 但是 VSCode 会自己去读 .vscode 文件夹下的 **task.json**。比如我们配置的 task.json 上面详细写着：
>
> 1. **任务名称**：“烧录程序”
> 2. **具体步骤**：“请调用电脑里的 openocd 这个工具，并把这些参数（比如调试器类型、芯片型号、程序文件位置）一字不差地传给它。”
>
> 所以，task.json 解决的核心问题是──**在 VSCode 这个图形界面里架起一座桥梁，连接友好的图形界面和底层强大的命令行工具**。
> 
> 如果感兴趣，你可以借助 AI 去了解我们配置的 task.json 上的每一个参数究竟告诉 VSCode 怎么正确执行这个任务。

## 配置调试功能

### 安装 Everything

可以从微软官方商店搜索，或者使用 WinGet 安装。

> **使用 WinGet 安装 Everything**
> 
> 按刚刚的步骤打开 PowerShell，安装 Everything
> ```powershell
> winget install voidtools.Everything
> ```

### 确认 arm-none-eabi-gdb 位置

我们需要找到调试器（GDB）的位置：

1. 使用 Everything 搜索 `arm-none-eabi-gdb.exe`，找到路径包含 `gnu-tools-for-stm32` 的那个结果
2. 通常路径类似：`AppData/Local/stm32cube/bundles/gnu-tools-for-stm32/13.3.1+st.9/bin/arm-none-eabi-gdb.exe`
3. 在目标文件右键，在右键菜单中选择 `复制为路径`，将路径保存好

![Everything 搜索 arm-none-eabi-gdb.exe 结果](everything-search-gdb.png)

> [!tip]
>
> **什么是 GDB？**
>
> GDB 是连接你的源代码和芯片实际运行的"桥梁调试器"，它负责和程序对话。它解决了在嵌入式开发中"代码如何真正在芯片上执行"的黑盒问题。
>
> 当程序在芯片上运行时，GDB 允许你"暂停时间"——查看变量当前值、分析函数调用关系、跟踪程序执行流程。没有 GDB，调试就像蒙着眼睛找错误；有了 GDB，你可以精确观察程序每一步的行为。
>
> 简单说：**OpenOCD 让电脑能"接触"到芯片，而 GDB 让开发者能"理解"芯片上正在发生什么。**

### 安装 Cortex Debug 扩展

`Ctrl + Shift + X`，打开扩展页，搜索 `Cortex Debug` 并安装

![VSCode Cortex Debug 扩展安装](vscode-cortex-debug-extension.png)

`Ctrl + Shift + D`，打开运行或调试页，点击 `create a launch.json file`，这会在你的项目根目录下的 `.vscode` 文件夹新建一个 `launch.json`。你也可以自己手动创建。

![VSCode 创建 launch.json 文件界面](vscode-create-launch-json.png)

点击 `create a launch.json file` 后，在上方选项选择第一项，然后下拉框选择 `Cortex Debug: OpenOCD`。或者你也可以打开 `launch.json` 文件，点击右下角的 `Add Configuration...`，同样选择 `Cortex Debug: OpenOCD`。

![VSCode 添加 Cortex Debug 配置选项](vscode-add-cortex-debug-config.png)

补全配置，添加 GDB 路径：

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "cwd": "${workspaceRoot}",
            "executable": "${command:st-stm32-ide-debug-launch.get-projects-binary-from-context1}",
            "name": "Debug with OpenOCD",
            "request": "launch",
            "type": "cortex-debug",
            "servertype": "openocd",
            "configFiles": [
            	"interface/cmsis-dap.cfg",
            	"target/stm32f1x.cfg"
            ],
            "searchDir": [],
            "runToEntryPoint": "main",
            "showDevDebugOutput": "none",
            "gdbPath": ".../AppData/Local/stm32cube/bundles/gnu-tools-for-stm32/13.3.1+st.9/bin/arm-none-eabi-gdb" // 填入刚刚找到的路径
        }

    ]
}
```

保存配置后，运行与调试页应该会出现 `Debug with OpenOCD` 选项，点击运行三角形即可开始调试。

![VSCode 调试界面显示 Debug with OpenOCD 选项](vscode-debug-interface.png)

> [!tip]
>
> **什么是 launch.json？**
>
> 如果说 `task.json`是"烧录说明书"，那么 `launch.json`就是"调试说明书"。它告诉 VSCode 如何启动调试会话：设置断点、查看变量、单步执行等。

## 常见问题

### 报错 `target not found`

- 检查连线（SWDIO, SWCLK, GND, 3.3V）是否接好。
- 检查 target/stm32f1x.cfg 是否选对了系列（F1, F4, H7 配置文件不同）。

### GDB 报错 `No such file or directory`

- 检查 `launch.json` 里的 `gdbPath` 路径是否写错，尤其是斜杠的方向。

## 参考教程

### 视频教程

- [[STM32 + VS Code] 用 DAPLink + OpenOCD 调试 - XRobot 官方教程 0.2 节](https://www.bilibili.com/video/BV1bnnZz2ESg)
- [【Windows】VSCode开发STM32，但是使用cmake+clangd+ninja+arm-gcc，全套开源工具链，编译烧录调试无压力。](https://www.bilibili.com/video/BV1X4XAYiEkV)
