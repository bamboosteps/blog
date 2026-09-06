---
sidebar_position: 1
title: GPIO 控制方案对比
description: 以点亮一个 LED 为线索,梳理 Linux 内核提供的几种 GPIO 控制方案(sysfs 接口、chardev 驱动、gpiolib 框架)之间的关系和适用场景,基于 Linux 4.9。
---

## 核心概念

GPIO(General Purpose Input/Output)是 SoC 上可由软件配置为输入或输出的通用引脚。Linux 内核用统一框架 **gpiolib**(`drivers/gpio/gpiolib.c`)管理所有 GPIO 控制器。gpiolib 不是"几种控制方案之一",而是所有方案共同的底层实现:任何控制器驱动只要调用 `gpiochip_add_data()` 完成注册,内核就自动提供以下三种接口,控制器驱动作者不需要额外编写代码:

1. **sysfs 接口**(`/sys/class/gpio`,需要 `CONFIG_GPIO_SYSFS`)
2. **字符设备接口**(`/dev/gpiochipN`,需要 `CONFIG_GPIO_CDEV`,Linux 4.8 引入)
3. **debugfs 只读接口**(`/sys/kernel/debug/gpio`,仅用于查看状态,不能控制)

"用 sysfs 点灯"和"用字符设备点灯"操作的都是已经由某个厂商驱动注册好的 GPIO 控制器,全程不需要编写内核代码,区别只在于用户态调用的接口。"编写 gpiolib 驱动"是另一件事:实现 `struct gpio_chip` 把新的 GPIO 控制器注册进 gpiolib,或编写内核态消费者驱动通过 `gpiod_*` 函数使用已注册的 GPIO,属于内核态开发。

## 三种方案对比

| 方案 | 是否编写内核代码 | 涉及接口 | 内核版本要求 | 备注 |
|---|---|---|---|---|
| sysfs | 否 | `/sys/class/gpio/export`、`/gpioN/direction`、`/gpioN/value` | 早期版本即支持 | 内核已将该接口标记为 deprecated |
| 字符设备(chardev) | 否 | `/dev/gpiochipN` + ioctl,或封装库 `libgpiod` | Linux 4.8 起支持 | 官方推荐的用户态访问接口 |
| gpiolib 驱动 | 是 | `struct gpio_chip`(内核态提供者)、`gpiod_*`(内核态消费者) | 长期存在,API 随版本演进 | 用于把 GPIO 集成进内核设备模型,供其他内核驱动调用 |

## 相关文章

- [用 sysfs 点亮 LED](./sysfs.md):用遗留的 sysfs 接口控制一个已存在的 GPIO 控制器,不编写内核代码。
- [用字符设备点亮 LED](./chardev.md):用当前推荐的字符设备接口控制同一个已存在的 GPIO 控制器,不编写内核代码。
- [用 gpiolib 驱动点亮 LED](./gpiolib-driver.md):编写一个最小的 GPIO 控制器驱动(provider)和使用它的消费者驱动(consumer)。
