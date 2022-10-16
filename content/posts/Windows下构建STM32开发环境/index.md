---
title: "Windows 下构建 STM32 开发环境"
date: 2022-09-05T20:37:29+08:00
draft: false

summary: Keil，狗都不用
description: 

tags: ["C++", "STM32"]
categories: ["embedded"]

cover: 
    image: 'img/cover.jpg'
---

## 目标

在 Windows 下脱离 Keil MDK 完成 STM32L4xxx 系列 MCU 的开发和调试工作。

网上很多教程在一些配置上解释的都有错误，或者根本没解释为什么要这样做，简单记录一下配置流程。

> **为什么不用 Keil？**
>
> Keil 狗都不用。
>
> **为什么不用 PlatformIO？**
>
> PlatformIO 是以开发板而非芯片为中心的 IDE，这就导致它更适合于 Arduino 这类开源硬件的开发，和实际的嵌入式开发需求不符合。PlatformIO 的开发者似乎也意识到了这个问题，在库里提供了部分 generic 配置文件，但覆盖面太窄了，并没有 STM32L4 的 generic 配置文件，手写`.ini`文件又太蠢了一些。
>
> **为什么不用 stm32-for-vscode 和 Cortex-Debug 插件？**
>
> 第一次使用还是手动写写配置文件比较好，能熟悉一下每个工具。安装完这两个插件后就直接自动完成了所有工作，不利于前期学习。

## 大体流程

1. 使用 STM32CubeMX 生成配置代码及 Makefile

2. 使用 VS Code 编写程序

3. 在 VS Code 中调用 make 完成交叉编译

4. 使用 OpenOCD 完成程序的烧录和调试

![flowchart](img/flowchart.svg#center)

> 图中横线上方加粗的内容是脱离 Keil 开发和调试 STM32 程序必需的工具，横线下方为推荐的工具和插件。

## 准备工作

安装必需的工具：

* STM32CubeMX
* VS Code

除了上面两个之外，还需要安装：

* make
* Arm GNU Toolchain
* Git Bash

> Windows 并不提供 make 命令，而 Mingw-w64 包含了了`mingw64-make.exe`程序，因此可以通过安装 Mingw-w64 来安装 make：
>
> 1. 将安装目录`bin`文件夹下的`mingw64-make.exe`复制一份，并重命名为`make.exe`；
>
> 2. 将`bin`文件夹添加到环境变量；
>
> 3. 在命令行执行`make -v`，确认环境变量配置正确。

> 现在已经 2022 年了，网络上有不少互相抄来抄去教程，要求读者安装 GNU Arm Embedded Toolchain，然后甩一个旧版的下载链接。
>
> 但实际上 Arm GNU Toolchain 已经取代了 GNU Arm Embedded Toolchain，后者被 Arm 标记为 discontinued，没有特殊理由当然应优先选择最新的 Toolchain。
>
>  — <cite>Arm GNU Toolchain Official Site[^1]</cite>

[^1]: [Arm GNU Toolchain Downloads](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads).

## 环境配置

打开 STM32CubeMX，这里我们已经有了一个名为 Test 的工程，切换到 Project Manager，将 Toolchain/IDE 修改为 Makefile：

![STM32CubeMX](img/STM32CubeMX.png#center)

修改完成后生成代码，在输出目录（我这里是`E:\Test`）中，可以看到目录下已经有了一个 Makefile 文件。

在**该目录**打开命令行（ powershell 或 cmd 均可），运行 `make` 命令，会得到下面的结果：

```bash
PS E:\Test> make
mkdir build
process_begin: CreateProcess(NULL, mkdir build, ...) failed.
make (e=2): 系统找不到指定的文件。
make: *** [Makefile:179: build] Error 2
```

这里产生了一个报错，提示没有找到 build 目录，为什么会产生这个错误我们在后面会解释，这里先不管它，在`E:\Test`目录下我们手动创建一个文件夹`build`，再次执行命令`make`：

![make](img/make.png#center)

如果最终得到以下输出结果，则说明程序编译没有任何问题，`Mingw-w64`和`gcc-arm-none-eabi`均已被正确安装。

```bash
arm-none-eabi-size build/Test.elf
   text    data     bss     dec     hex filename
   4384      32    1568    5984    1760 build/Test.elf
arm-none-eabi-objcopy -O ihex build/Test.elf build/Test.hex
arm-none-eabi-objcopy -O binary -S build/Test.elf build/Test.bin
```

> 如果尝试执行`make clean`，也会遇到类似的错误：
>
> ```bash
> PS E:\Test> make clean
> rm -fR build
> process_begin: CreateProcess(NULL, rm -fR build, ...) failed.
> make (e=2): 系统找不到指定的文件。
> make: [Makefile:185: clean] Error 2 (ignored)
> ```
>
> 造成这一问题的原因和`mkdir`无法执行的原因相同，将在后面进行解释。

## 在 VS Code 中开发和编译程序

在调通前面的步骤之后，我们实际上已经可以脱离 Keil 环境完成 STM32 程序的编译，接下来还需要修改 VSCode 的配置，才能直接在 VSCode 中编译代码，并享受到 VSCode 提供的代码补全等功能。

首先我们在 VSCode 中打开项目文件夹`E:\Test`，我们将要修改下面三个文件的配置：

* `tasks.json`
* `settings.json`
* `c_cpp_properties.json`

这三个文件夹将被放置在工程目录下 `.vscode` 文件夹中。

### tasks.json

`tasks.json`文件告诉了 VSCode 如何按照上一节的步骤编译我们的 STM32 项目，为了修改该文件，按下快捷键`ctrl+shift+p`，输入`configure task`，点击“配置任务”，并选择“使用模板创建`tasks.json`文件”，模板则选择`Others`，然后将下面的内容填入到新创建的`tasks.json`文件中：

```json
// tasks.json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Build",
            "type": "shell",
            "command": "make",
            "args": [
                "-j16"
            ],
            "problemMatcher": ["$gcc"],
            "group": {
              "kind": "build",
              "isDefault": true
            }
        }
    ]
}
```

复制上面这段内容时记得根据电脑实际配置修改`args`中的`-j16`参数，将`16`替换为CPU实际的线程数，其它内容保持不变，保存。

随后，点击顶部菜单栏“终端”，“运行生成任务”，可以看到底部终端中 VSCode 将自动执行`make -j16`命令（尽管依然会出现错误，“系统找不到指定的文件”）。

### settings.json

前面提到了，手动执行 `make` 时，产生了如下的错误：

```bash
mkdir build
process_begin: CreateProcess(NULL, mkdir build, ...) failed.
make (e=2): 系统找不到指定的文件。
make: *** [Makefile:179: build] Error 2
```

这四行的意思很简单，第一行尝试执行了`mkdir build`命令来创建文件夹`build`，但第二行告诉我们`mkdir build`命令执行失败，如果我们看一下 `Makefile`文件，可以看到，`build`文件夹是编译结果的输出文件夹，如果这个文件夹未能正确创建，自然也就无法进行后续的编译操作。

```Makefile
#######################################
# paths
#######################################
# Build path
BUILD_DIR = build

# 第 178 行
$(BUILD_DIR):
    mkdir $@
```

此时在工程目录手动创建`build`文件夹虽然能够绕过问题，但为什么`mkdir build`会执行失败呢？问题就隐含在错误输出的第二行：

```bash
process_begin: CreateProcess(NULL, mkdir build, ...) failed.
```

这里告诉我们，程序尝试通过`CreateProcess()`函数来创建一个新的进程，然而在 Windows 下，`mkdir`并非一个可执行文件（或者说，Windows 下并没有`mkdir.exe`），而仅仅是`cmd.exe`的一个内置命令，自然也就无法直接通过`CreateProcess()`来调用。

> 类似的，`rm`，`mv`等 Linux 下可以随意调用的命令，在 Windows 下都无法直接通过`CreateProcess()`调用，这也是为什么无法直接运行`make clean`:
>
> ```Makefile
> #######################################
> # clean up
> #######################################
> clean:
>     -rm -fR $(BUILD_DIR)
> ```

这一问题的解决办法也很简单，在安装 Git 的同时我们同时也安装了 Git Bash，而 Git Bash 提供了大量 Unix 命令（如`rm`、`cp`、`mv`），只要在 Git Bash 下执行 `make` 就不会遇到这类问题了。

那么如何修改 VSCode 工作区的默认终端环境为 Git Bash 呢？只需要修改工作区的`setting.json`文件即可达到目的。按下`ctrl+shift+p`，输入`open Workspace Settings (JSON)`，然后向其中填入：

```JSON
// settings.json
{
    "terminal.integrated.defaultProfile.windows": "Git Bash"
}
```

重新启动 VSCode 工作区，点击顶部菜单栏“终端”，“运行生成任务”，可以看到 STM32 工程已被正确编译。

### c_cpp_properties.json

在正确填写`task.json`和`settings.json`后，虽然我们已经可以绕开命令行操作，直接在 VSCode 中编译整个项目，但在 VSCode 中打开任意源文件（如`main.c`），仍会看到大量的红色波浪线，代码补全等功能依然无法使用。这就需要修改`c_cpp_properties.json`，使 VSCode IntelliSense 在正常工作。

```Json
// c_cpp_properties.json
{
    "configurations": [
        {
            "name": "STM32L475xx",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "defines": [
                "USE_HAL_DRIVER",
                "STM32L475xx"
            ],
            "compilerPath": "C:/Program Files/mingw-w64/bin/gcc.exe",
            "cStandard": "c17",
            "cppStandard": "c++17",
            "intelliSenseMode": "gcc-x64",
            "configurationProvider": "ms-vscode.makefile-tools"
        }
    ],
    "version": 4
}
```

这里要重点注意的主要是`defines`和`compilerPath`两个选项（至于`name`随便填就好了），`defines`实际上就是在 Keil 中填写的全局宏定义标识符（Preprocessor Symbols -> Define）：

![Keil Defines](img/Keil.png#center)

而`compilerPath`则是我们之前安装的 Mingw-w64 编译器的路径，按实际情况填写就行。

> 一些文章在配置`c_cpp_properties.json`时，还修改了`includePath`：
>
> ```Json
> "includePath": [
>     "${workspaceFolder}/**",
>     "${workspaceFolder}/Core/Inc",
>     "${workspaceFolder}/Drivers/STM32L4xx_HAL_Driver/Inc",
>     "${workspaceFolder}/Drivers/STM32L4xx_HAL_Driver/Inc/Legacy",
>     "${workspaceFolder}/Drivers/CMSIS/Device/ST/STM32L4xx/Include",
>     "${workspaceFolder}/Drivers/CMSIS/Include"
> ],
> ```
>
> 这些文章把 Keil 中填写的 Include Paths 填入到了`c_cpp_properties.json`里，这种写法纯属画蛇添足，因为`${workspaceFolder}/**`就已经告诉了 VSCode 要递归遍历项目文件夹下的所有文件及其子文件，在后面手动添加的这些内容没有任何意义，正确的做法是要么只写第一行，要么不写第一行。

### 简单总结

* `tasks.json`：用于配置 VSCode 调用外部工具完成编译工作；
* `settings.json`：在该文件中修改工作区内使用的 terminal，以消除 `make` 命令执行时遇到的各种问题；
* `c_cpp_properties.json`：用于配置 VSCode 的智能感知插件，使其能正确解析工作区内所有代码，实现代码补全等功能。
