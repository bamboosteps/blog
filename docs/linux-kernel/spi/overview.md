---
sidebar_position: 1
title: SPI 访问方案对比
description: Linux 4.9 内核 SPI 子系统的两层结构,以及 spidev 与 spi_driver 两种开发/使用方案的对比。
---

## 核心概念

SPI 是一种四线(SCLK/MOSI/MISO/CS)全双工总线协议,总线上有一个主设备(master)和若干从设备,通过片选(CS)区分。和 I2C 类似,Linux 内核的 SPI 子系统也分两层:

- **master controller(总线控制器)**:对应 SoC 上的 SPI 控制器硬件,通常已经由芯片厂商的 BSP 提供。
- **client(从设备)**:总线上具体的一颗芯片(Flash、ADC、传感器等),驱动作者通常编写的是这一层。

对应到实际开发,操作总线上的一个 SPI 从设备有两种方案:

| 方案 | 是否编写内核驱动 | 接口 | 特点 |
|---|---|---|---|
| spidev | 否 | `/dev/spidevB.C` + `ioctl()` | 用户态直接发起全双工传输,常用于调试或简单场景;需要在设备树里显式声明 `compatible = "spidev"` 才会创建设备节点 |
| spi_driver 客户端驱动 | 是 | 内核态 `probe()`/`remove()` + `spi_sync()`/`spi_write_then_read()` | 把设备接入内核设备模型,可以把协议细节封装起来,只给用户态暴露一个简单接口 |

## 相关文章

- [spidev 接口](./spidev.md):不编写内核驱动,直接从用户态操作总线上的从设备。
- [spi_driver 驱动](./spi-driver.md):编写一个客户端驱动,读取从设备的一个寄存器,并通过字符设备暴露给用户态。
