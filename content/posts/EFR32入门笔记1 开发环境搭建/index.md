---
title: "EFR32 入门笔记 1: 开发环境搭建"
date: 2023-03-09T10:18:38+08:00
draft: false

summary: 来试试组里一直说很难的 EFR32
description: 基于 EFR32BG22C112F352GM32

tags: ["C", "EFR32"]
categories: ["embedded"]

cover: 
    image: 'img/cover.jpg'
---

## 预备知识

### 预备知识 1: EFR32 系列芯片编号规则

![Ordering Code Key](img/OrderingCodeKey.svg#center)

> EFR32 Family:
>
> M: Mighty (EFR32MG)
>
> B: Blue (EFR32BG)
>
> F: Flex (EFR32FG)

根据 Family 选择对应的 API 即可，如 EFR32BG22 应查阅文档 Blue Gecko。

### 预备知识 2：什么是 Gecko SDK？

Gecko SDK（GSDK）是 Silicon Labs 为该司生产的 MCU 所提供的一系列 SDK 的统称。GSDK 中包含了若干具有不同功能的子 SDK，[Github Release](https://github.com/SiliconLabs/gecko_sdk/releases) 里可以看到详细的内容，包括但不限于：`Bluetooth SDK`，`Zigbee EmberZNet SDK` 等等。

## 开发工具链

初学就按照官方建议，使用 [Simplicity Studio](https://www.silabs.com/developers/simplicity-studio) 就好了，[Simplicity Studio 5 User’s Guide](https://docs.silabs.com/simplicity-studio-5-users-guide/latest/ss-5-users-guide-getting-started/install-ss-5-and-software) 里给出了 Step-by-step 的安装说明。

安装完成后，在 Simplicity Studio 中打开 Installation Manager，选择安装 Gecko SDK 即完成了环境配置。

### 创建并编译第一个程序

使用 Simplicity Studio 中的 `Silicon Labs Project Wizard` 创建第一个 Project，由于我采用的是非官方开发板，因此 Target Boards 栏直接留空，输入 MCU 的型号 `EFR32BG22C112F352GM32`，并选择 Gecko SDK 即可。

![NewProjectWizard](img/NewProjectWizard.png#center)

点击下一步，直接选择 `Empty C Project` 模板来创建第一个程序。

> 如果希望使用 VSCode 打开项目，宜勾选 `With project files` 中的 `Copy contents` 而不是默认的 `Link sdk and copy project sources`，否则会提示找不到部分头文件。

创建完成后，先点击 Project Explorer 里的项目名称，再点击编译按钮，如果环境配置没有问题，项目将被成功编译。

> 0 errors, 0 warnings!

### Gecko SDK 的编程模型

前面创建好的项目里除 SDK，还包含了三个文件：`main.c`，`app.c`，`app.h`。

> Empty C Project 并不是真的 Empty

这就涉及到了 Gecko SDK 的 Programming Model，可以参考[官方文档](https://docs.silabs.com/gecko-platform/latest/sdk-programming-model)对这一部分的介绍，这里简单介绍一下：

```C
#include "sl_component_catalog.h"
#include "sl_system_init.h"
#include "app.h"
#if defined(SL_CATALOG_POWER_MANAGER_PRESENT)
#include "sl_power_manager.h"
#endif
#if defined(SL_CATALOG_KERNEL_PRESENT)
#include "sl_system_kernel.h"
#else // SL_CATALOG_KERNEL_PRESENT
#include "sl_system_process_action.h"
#endif // SL_CATALOG_KERNEL_PRESENT

int main(void)
{
  // Initialize Silicon Labs device, system, service(s) and protocol stack(s).
  // Note that if the kernel is present, processing task(s) will be created by
  // this call.
  sl_system_init();

  // Initialize the application. For example, create periodic timer(s) or
  // task(s) if the kernel is present.
  app_init();

#if defined(SL_CATALOG_KERNEL_PRESENT)
  // Start the kernel. Task(s) created in app_init() will start running.
  sl_system_kernel_start();
#else // SL_CATALOG_KERNEL_PRESENT
  while (1) {
    // Do not remove this call: Silicon Labs components process action routine
    // must be called from the super loop.
    sl_system_process_action();

    // Application process.
    app_process_action();

#if defined(SL_CATALOG_POWER_MANAGER_PRESENT)
    // Let the CPU go to sleep if the system allows it.
    sl_power_manager_sleep();
#endif
  }
#endif // SL_CATALOG_KERNEL_PRESENT
}
```

可以看到 `main.c` 里将根据两个宏来实现条件编译：

* SL_CATALOG_POWER_MANAGER_PRESENT
* SL_CATALOG_KERNEL_PRESENT

`SL_CATALOG_POWER_MANAGER_PRESENT` 宏与芯片的内部的低功耗模式密切相关。如图所示，EFR32BG22 具有四种不同的电源模式：`EM0`、`EM1`、`EM2`、`EM3`、`EM4`，利用 Gecko SDK 提供的 Power Manager 可以切换当前的电源模式，这里暂且略过，详细内容可参考[官方文档](https://docs.silabs.com/gecko-platform/latest/service/power_manager/overview)。

![EFR32BlockDiagram](img/EFR32BlockDiagram.svg#center)

`SL_CATALOG_KERNEL_PRESENT` 宏决定了项目是否使用 RTOS。默认未定义该宏，即默认采用裸机编程，实际编译的代码为：

```C
#include "sl_component_catalog.h"
#include "sl_system_init.h"
#include "app.h"

#include "sl_system_process_action.h"

int main(void)
{
  sl_system_init();

  app_init();

  while (1) {
    // Do not remove this call: Silicon Labs components process action routine
    // must be called from the super loop.
    sl_system_process_action();

    // Application process.
    app_process_action();
  }

}
```

这里的 `app_init()` 和 `app_process_action()` 就相当于 Arduino 中的 `setup()` 和 `loop()` 函数，也就是说用户程序直接写在 `app.c` 和 `app.h` 里就行。

如果需要使用 RTOS，就应当在 `sl_component_catalog.h` 中填入`#define SL_CATALOG_KERNEL_PRESENT`：

```c
#ifndef SL_COMPONENT_CATALOG_H
#define SL_COMPONENT_CATALOG_H

// APIs present in project
#define SL_CATALOG_DEVICE_INIT_NVIC_PRESENT
#define SL_CATALOG_EMLIB_CORE_PRESENT
#define SL_CATALOG_EMLIB_CORE_DEBUG_CONFIG_PRESENT
#define SL_CATALOG_KERNEL_PRESENT

#endif // SL_COMPONENT_CATALOG_H
```

此时编译的代码为：

```C
#include "sl_component_catalog.h"
#include "sl_system_init.h"
#include "app.h"

#include "sl_system_kernel.h"

int main(void)
{
  sl_system_init();

  app_init();

  // Start the kernel. Task(s) created in app_init() will start running.
  sl_system_kernel_start();
}
```

## 使用 VSCode 编辑程序

主要是 `c_cpp_properties.json` 文件的修改，待更。

## 烧录程序时遇到的问题

将设备通过 SWD 接口连接到 J-Link （笔者使用的是 J-Link v11），安装 J-Link 驱动后，使用 Simplicity Commander 时提示：

```bash
WARNING: Could not connect to target device
```

于是打开 J-Link Commander，输入 connect，发现还是无法连接：

```bash
- Iterating through AP map to find AHB-AP to use
......
......
......
- ERROR: Failed to connect.
Could not establish a connection to target.
```

接着查阅官网资料，官方论坛建议在命令行中执行：

```bash
# Simplicity Studio 的安装目录
$ cd 'D:\Program Files\SiliconLabs\SimplicityStudio\v5\developer\adapter_packs\commander'
# --device 后指定芯片型号
$ .\commander device recover --device=EFR32BG22C112F352GM32
```

随后得到了：

```bash
Recovering "bricked" device...
Resetting device...
Attempting to connect to DCI...
Success after 1 attempts (6 ms)
Running device erase command...
Success after 1 attempts (74 ms)
Resetting device...
Device recovery successful.
DONE
```

此时就可以使用 Simplicity Commander 烧录程序了。
