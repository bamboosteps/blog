---
sidebar_position: 1
title: GPIO 控制方案对比
description: gpiolib 是三种 GPIO 用户态接口(sysfs、字符设备、debugfs)的统一底层实现,本文给出三种方案的调用关系与选型对比,基于 Linux 4.9。
---

GPIO(General Purpose Input/Output)是 SoC 上可由软件配置为输入或输出的通用引脚。Linux 内核用统一框架 **gpiolib**(`drivers/gpio/gpiolib.c`)管理所有 GPIO 控制器:任何控制器驱动调用 `gpiochip_add_data()` 完成注册后,内核自动提供 sysfs、字符设备、debugfs 三种接口,控制器驱动作者不需要为此额外编写代码。

## 架构与调用机制

```text
                     ┌─ /sys/class/gpio         sysfs(CONFIG_GPIO_SYSFS)
用户态 ── 接口层 ────┼  /dev/gpiochipN          字符设备(CONFIG_GPIO_CDEV)
                     └─ /sys/kernel/debug/gpio  debugfs(只读)
                                 │
                                 ▼
                     gpiolib(drivers/gpio/gpiolib.c)
                                 │
                                 ▼
                     struct gpio_chip 回调(控制器驱动)
                                 │
                                 ▼
                           硬件寄存器
```

sysfs 与字符设备操作的是同一个已注册的 `gpio_chip`,区别仅在用户态入口。"编写 gpiolib 驱动"是另一件事:实现 `struct gpio_chip` 把新控制器注册进 gpiolib(provider),或编写内核态消费者驱动通过 `gpiod_*` 使用已注册的 GPIO(consumer)。

## 方案映射

| 方案 | 是否编写内核代码 | 涉及接口 | 内核版本要求 | 备注 |
|---|---|---|---|---|
| sysfs | 否 | `/sys/class/gpio/export`、`/gpioN/direction`、`/gpioN/value` | 早期版本即支持 | 已标记为 deprecated |
| 字符设备(chardev) | 否 | `/dev/gpiochipN` + ioctl,或 `libgpiod` | Linux 4.8 起支持 | 官方推荐的用户态访问接口 |
| gpiolib 驱动 | 是 | `struct gpio_chip`(provider)、`gpiod_*`(consumer) | 长期存在,API 随版本演进 | 用于把 GPIO 集成进内核设备模型 |

## 相关文档

- [sysfs 接口](./sysfs.md) — 用遗留的 sysfs 接口控制一个已存在的 GPIO 控制器,不编写内核代码
- [chardev 驱动](./chardev.md) — 用当前推荐的字符设备接口控制同一个已存在的 GPIO 控制器,不编写内核代码
- [gpiolib 框架](./gpiolib-driver.md) — 编写一个最小的 GPIO 控制器驱动(provider)和使用它的消费者驱动(consumer)
