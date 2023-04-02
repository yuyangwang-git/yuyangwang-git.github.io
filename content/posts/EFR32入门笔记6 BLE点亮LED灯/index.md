---
title: "EFR32 入门笔记 6: BLE 点亮 LED 灯"
date: 2023-04-01T23:25:48+08:00
draft: false

summary: 终于，到了最难的蓝牙通讯部分了
description: 过去没接触过蓝牙开发，硬啃蓝牙协议啃了好久

tags: ["C", "EFR32"]
categories: ["embedded"]

cover:
    image: 'img/cover.jpg'
---

## 背景知识

整个蓝牙协议栈内容太多了，以后如果有时间单独写几篇相关文章梳理一下，本文将直接上手写程序。

## 利用 Bluetooth LE 点亮 LED 灯

最终目标：利用智能手机远程控制（开启及关闭） EFR32BG22 IO 口连接的 LED 灯。

上述 EFR32BG22 将运行在 Standalone mode。

### 预先准备

手机上需要安装蓝牙调试程序，常用的调试软件有：

1. nRF Connect
2. LightBlue
3. EFR Connect

![bluetooth tools](img/bluetooth_tools.jpg#center)

三个软件的功能完全重合。每个都试了一下，感觉在 iOS 上做的最完善的还是 Nordic Semiconductor 的 nRF Connect。

~~这何尝不是一种 NTR？~~

### 创建 Project

首先，基于 `Bluetooth - SoC Empty` 模板创建一个空蓝牙项目，命名为 `bluetooth_led_blink`：

![new project wizard](img/new_project_wizard.png#center)

随后，安装 `Services - I/O: stream USART` 并创建 `instance`；安装 `Application - Utility: Log`，卸载 `Platform - RAIL Utility, PTI`，完成后就可以写应用相关代码了。

> 为什么要卸载 PTI Components？
>
> Packet Trace Interface 是一个用于调试 network activity 的组件，我们暂时用不到这个功能，当然也可以不卸载，但不卸载的话会，编译后在 Simplicity IDE 底部的 Problems 中出现一条报错消息：
>
> ```txt
>
> Errors (1 item): 
> ERROR: DOUT is not defined - DOUT is required when PTI enabled, please select pin for DOUT
>
> ```
>
> ![pti problems](img/pti_problems.png#center)
>
> 虽然不影响最后生成 `.hex` 文件，但是显然让人看着心烦，不如直接删了。

### 修改 GATT

在正式编写程序之前，还需要修改 Generic Attribute Profile (GATT)，

打开 `bluetooth_led_blink.slcp`，切换到 Configuration Tools Tab，随后打开 `Bluetooth GATT Configurator`：

![configuration tools](img/configuration_tools.png#center)

修改 `Device Name` 为 `LED Controller`，后面我们使用手机搜索蓝牙设备时，看到的名称就是这里填写的 `Device Name`。

![Bluetooth GATT Configurator](img/bluetooth_gatt_configurator.png#center)

`Device Information` 里都是一些“无关紧要的”设备信息，包括制造商名称，产品序列号，版本号等，这里就不修改了，不影响点灯。

接下来就是最重要的部分，在 GATT 中增加一个 Service，该 Service 包含一个控制 LED 开关的 Characteristics 字段：

![led status](img/led_status.png#center)

这里 `Characteristic Name` 其实无所谓，重要的是 `ID` （将影响我们的程序中的变量名），`Value Settings`（将决定手机和芯片通信时传输的数据类型） 和 `Properties` （将决定其他设备是否有权限读取和修改该字段，换句话说，手机是否有权限控制灯的打开和关闭）。另外，**记下这里的 UUID**。

完成修改后保存该文件。

### 修改程序

最后就是程序部分了，先来简单过一遍初始文件模板。

```c
// app.c

#include "em_common.h"
#include "app_assert.h"
#include "sl_bluetooth.h"
#include "app.h"

// The advertising set handle allocated from Bluetooth stack.
static uint8_t advertising_set_handle = 0xff;

/**************************************************************************//**
 * Application Init.
 *****************************************************************************/
SL_WEAK void app_init(void)
{
  /////////////////////////////////////////////////////////////////////////////
  // Put your additional application init code here!                         //
  // This is called once during start-up.                                    //
  /////////////////////////////////////////////////////////////////////////////
}

/**************************************************************************//**
 * Application Process Action.
 *****************************************************************************/
SL_WEAK void app_process_action(void)
{
  /////////////////////////////////////////////////////////////////////////////
  // Put your additional application code here!                              //
  // This is called infinitely.                                              //
  // Do not call blocking functions from here!                               //
  /////////////////////////////////////////////////////////////////////////////
}

/**************************************************************************//**
 * Bluetooth stack event handler.
 * This overrides the dummy weak implementation.
 *
 * @param[in] evt Event coming from the Bluetooth stack.
 *****************************************************************************/
void sl_bt_on_event(sl_bt_msg_t *evt)
{
  sl_status_t sc;

  switch (SL_BT_MSG_ID(evt->header)) {
    // -------------------------------
    // This event indicates the device has started and the radio is ready.
    // Do not call any stack command before receiving this boot event!
    case sl_bt_evt_system_boot_id:
      // Create an advertising set.
      sc = sl_bt_advertiser_create_set(&advertising_set_handle);
      app_assert_status(sc);

      // Generate data for advertising
      sc = sl_bt_legacy_advertiser_generate_data(advertising_set_handle,
                                                 sl_bt_advertiser_general_discoverable);
      app_assert_status(sc);

      // Set advertising interval to 100ms.
      sc = sl_bt_advertiser_set_timing(
        advertising_set_handle,
        160, // min. adv. interval (milliseconds * 1.6)
        160, // max. adv. interval (milliseconds * 1.6)
        0,   // adv. duration
        0);  // max. num. adv. events
      app_assert_status(sc);
      // Start advertising and enable connections.
      sc = sl_bt_legacy_advertiser_start(advertising_set_handle,
                                         sl_bt_advertiser_connectable_scannable);
      app_assert_status(sc);
      break;

    // -------------------------------
    // This event indicates that a new connection was opened.
    case sl_bt_evt_connection_opened_id:
      break;

    // -------------------------------
    // This event indicates that a connection was closed.
    case sl_bt_evt_connection_closed_id:
      // Generate data for advertising
      sc = sl_bt_legacy_advertiser_generate_data(advertising_set_handle,
                                                 sl_bt_advertiser_general_discoverable);
      app_assert_status(sc);

      // Restart advertising after client has disconnected.
      sc = sl_bt_legacy_advertiser_start(advertising_set_handle,
                                         sl_bt_advertiser_connectable_scannable);
      app_assert_status(sc);
      break;

    ///////////////////////////////////////////////////////////////////////////
    // Add additional event handlers here as your application requires!      //
    ///////////////////////////////////////////////////////////////////////////

    // -------------------------------
    // Default event handler.
    default:
      break;
  }
}
```

其实文件里就是三个函数：

```c
SL_WEAK void app_init(void)

SL_WEAK void app_process_action(void)

void sl_bt_on_event(sl_bt_msg_t *evt)
```

`app_init()` 和 `app_process_action()` 都是老熟人了，至于它们为什么被 `SL_WEAK` 关键字修饰，这里暂且先不用管，只要把我们应用代码放到这两个函数里就行。重点是最后一个函数 `sl_bt_on_event()`，我们删掉一部分内容，简单看一下这个函数的结构：

```c
/**************************************************************************//**
 * Bluetooth stack event handler.
 * This overrides the dummy weak implementation.
 *
 * @param[in] evt Event coming from the Bluetooth stack.
 *****************************************************************************/
void sl_bt_on_event(sl_bt_msg_t *evt)
{
  sl_status_t sc;

  switch (SL_BT_MSG_ID(evt->header)) {
    // -------------------------------
    // This event indicates the device has started and the radio is ready.
    // Do not call any stack command before receiving this boot event!
    case sl_bt_evt_system_boot_id:
      break;

    // -------------------------------
    // This event indicates that a new connection was opened.
    case sl_bt_evt_connection_opened_id:
      break;

    // -------------------------------
    // This event indicates that a connection was closed.
    case sl_bt_evt_connection_closed_id:
      break;

    ///////////////////////////////////////////////////////////////////////////
    // Add additional event handlers here as your application requires!      //
    ///////////////////////////////////////////////////////////////////////////

    // -------------------------------
    // Default event handler.
    default:
      break;
  }
```

该函数内部就是一个大的 `switch...case...` 循环，具体执行哪些代码分支，将由传入的参数 `evt` 决定。当 EFR32 的蓝牙状态发生改变时（如有新的蓝牙设备连接到 EFR32，或者已连接的设备尝试读取或写入数据），`sl_bt_on_event()` 函数将被自动调用，用户可以在这里增加一个 `switch...case...` 分支，通过传入的 `evt` 参数来判断到底是哪一个 bluetooth event 触发了本次调用，并在对应的 `case` 语句里进行相应的处理。

> 其实就是 UI 编程里非常常用的事件驱动模型嘛。

看到这里自然会产生疑问，我怎么知道有哪些 event 会触发该函数的调用？实际上官方文档里写的非常详细，具体可以查阅文档的 [API Reference](https://docs.silabs.com/bluetooth/5.0/index) 部分。

回到我们的目的上来，在 `app.c` 中进行如下修改：

首先，增加对以下几个头文件的引用：

```c
// 为了控制 GPIO：
#include "em_cmu.h"
#include "em_gpio.h"

// 为了引用我们在 GATT 中创建的所有内容：
#include "gatt_db.h"

// 为了更方便的打印日志，当然也可以用 printf() 代替：
#include "app_log.h"
```

配置 GPIO 初始化：

```c
SL_WEAK void app_init(void)
{
  CMU_ClockEnable(cmuClock_GPIO, true);

  // LED 灯连接在 PB02 上, 高电平点亮
  GPIO_PinModeSet(gpioPortB, 2, gpioModePushPull, 0);
  GPIO_PinOutSet(gpioPortB, 2);
}
```

蓝牙事件处理，这里主要针对三个事件进行处理：

* `sl_bt_evt_connection_opened_id`, 蓝牙连接打开事件
* `sl_bt_evt_connection_closed_id`, 蓝牙连接关闭事件
* `sl_bt_evt_gatt_server_attribute_value_id`, GATT 修改事件，在这里切换 LED 状态

```c
void sl_bt_on_event(sl_bt_msg_t *evt)
{
  sl_status_t sc;

  switch (SL_BT_MSG_ID(evt->header)) {
    // -------------------------------
    // This event indicates the device has started and the radio is ready.
    // Do not call any stack command before receiving this boot event!
    case sl_bt_evt_system_boot_id:
      // Create an advertising set.
      sc = sl_bt_advertiser_create_set(&advertising_set_handle);
      app_assert_status(sc);

      // Generate data for advertising
      sc = sl_bt_legacy_advertiser_generate_data(advertising_set_handle,
                                                 sl_bt_advertiser_general_discoverable);
      app_assert_status(sc);

      // Set advertising interval to 100ms.
      sc = sl_bt_advertiser_set_timing(
        advertising_set_handle,
        160, // min. adv. interval (milliseconds * 1.6)
        160, // max. adv. interval (milliseconds * 1.6)
        0,   // adv. duration
        0);  // max. num. adv. events
      app_assert_status(sc);
      // Start advertising and enable connections.
      sc = sl_bt_legacy_advertiser_start(advertising_set_handle,
                                         sl_bt_advertiser_connectable_scannable);
      app_assert_status(sc);
      break;

    // -------------------------------
    // This event indicates that a new connection was opened.
    case sl_bt_evt_connection_opened_id:
      app_log_info("Connection opened.\n");
      break;

    // -------------------------------
    // This event indicates that a connection was closed.
    case sl_bt_evt_connection_closed_id:
      app_log_info("Connection closed.\n");

      // Generate data for advertising
      sc = sl_bt_legacy_advertiser_generate_data(advertising_set_handle,
                                                 sl_bt_advertiser_general_discoverable);
      app_assert_status(sc);

      // Restart advertising after client has disconnected.
      sc = sl_bt_legacy_advertiser_start(advertising_set_handle,
                                         sl_bt_advertiser_connectable_scannable);
      app_assert_status(sc);
      break;

    // -------------------------------
    // This event indicates that the value of an attribute in the local GATT
    // database was changed by a remote GATT client.
    case sl_bt_evt_gatt_server_attribute_value_id:
      // The value of the gattdb_led_status characteristic was changed.
      if (gattdb_led_status == evt->data.evt_gatt_server_attribute_value.attribute) {
        uint8_t data_recv;
        size_t data_recv_len;

        // Read characteristic value.
        sc = sl_bt_gatt_server_read_attribute_value(gattdb_led_status,
                                                    0,
                                                    sizeof(data_recv),
                                                    &data_recv_len,
                                                    &data_recv);
        (void)data_recv_len;
        app_log_status_error(sc);

        if (sc != SL_STATUS_OK) {
          break;
        }

        // Toggle LED.
        if (data_recv == 0x00) {
          GPIO_PinOutClear(gpioPortB, 2);;
          app_log_info("LED off.\n");
        } else if (data_recv == 0x01) {
          GPIO_PinOutSet(gpioPortB, 2);;
          app_log_info("LED on.\n");
        } else {
          app_log_error("Invalid attribute value: 0x%02x\n", (int)data_recv);
        }
      }
      break;

    ///////////////////////////////////////////////////////////////////////////
    // Add additional event handlers here as your application requires!      //
    ///////////////////////////////////////////////////////////////////////////

    // -------------------------------
    // Default event handler.
    default:
      break;
  }
```

到这里编译并烧录其实就可以了，但是我们注意 build console 中的提示信息，会发现存在两条警告：

```txt
../sl_gatt_service_device_information.c: In function 'sl_gatt_service_device_information_on_event':
../sl_gatt_service_device_information.c:63:2: warning: #warning "Could not set Model Number String." [-Wcpp]
   63 | #warning "Could not set Model Number String."
      |  ^~~~~~~
../sl_gatt_service_device_information.c:77:2: warning: #warning "Could not set Hardware Revision String." [-Wcpp]
   77 | #warning "Could not set Hardware Revision String."
      |  ^~~~~~~
```

为此，打开 `sl_gatt_service_device_information.c` 文件，定位到报错的 line 63 和 77，可以看到：

```c
#if defined(gattdb_model_number_string) && defined(SL_BOARD_NAME)
      sc = sl_bt_gatt_server_write_attribute_value(gattdb_model_number_string,
                                                   0,
                                                   GATTDB_MODEL_NUMBER_STRING_LEN,
                                                   (uint8_t *)SL_BOARD_NAME);
      app_assert_status(sc);
#else
#warning "Could not set Model Number String."
// Check the presence of this characteristic and the ID reference in the GATT
// Configurator. If using a custom board, remove this section and use the
// GATT Configurator to set the value manually.
#endif

      // Hardware Revision String
#if defined(gattdb_hardware_revision_string) && defined(SL_BOARD_REV)
      sc = sl_bt_gatt_server_write_attribute_value(gattdb_hardware_revision_string,
                                                   0,
                                                   GATTDB_HARDWARE_REVISION_STRING_LEN,
                                                   (uint8_t *)SL_BOARD_REV);
      app_assert_status(sc);
#else
#warning "Could not set Hardware Revision String."
// Check the presence of this characteristic and the ID reference in the GATT
// Configurator. If using a custom board, remove this section and use the
// GATT Configurator to set the value manually.
#endif
```

注释里写的很清楚，如果我们用的不是官方开发板，把这两条 `#warning` 直接注释掉就行。

注释后重新编译：

```txt
Build Finished. 0 errors, 0 warnings. (took 8s.351ms)
```

![0 error 0 warning](img/0_error_0_warning.jpg#center)

### 测试

PC 端启动串口调试程序。手机端启动 nRF Connect，并搜索附近的设备：

![nRF Connect Scanner](img/nrf_connect_scanner.jpg#center)

连接后，切换到 Client Tab，根据前面记下的 UUID，在 Attribute Table 中找到对应的 Characteristics，并点击向上的箭头（其实在我们的程序里，只有这个 Characteristics 开启了 Write 权限，所以只有它有上箭头，看不看 UUID 意义不大）：

![led controller](img/led_controller.jpg#center)

尝试改变写入的值，就可以看到 LED 灯随着我们的操作被点亮和关闭。
