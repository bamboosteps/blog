---
sidebar_position: 1
title: SPI 访问方案对比
description: Linux SPI 子系统分 master controller(总线控制器)与 client(从设备)两层,本文对比 spidev 与 spi_driver 两种从设备访问方案,基于 Linux 4.9。
---

SPI 是一种四线(SCLK/MOSI/MISO/CS)全双工总线协议,总线上有一个主设备(master)和若干从设备,通过片选(CS)区分。

## 架构与调用机制

```text
        master controller(总线控制器,SoC 提供,已由 BSP 实现)
                          │
        ┌─────────────────┼─────────────────┐
        ▼                                   ▼
  /dev/spidevB.C + ioctl()          client 驱动 probe()
  (spidev,不写内核代码)               (spi_driver,内核态开发)
        │                                   │
        ▼                                   ▼
  SPI_IOC_MESSAGE(N)                spi_write_then_read()
        └───────────────┬───────────────────┘
                          ▼
                    spi_sync()
                          │
                          ▼
                    总线时序收发
```

| 方案 | 是否编写内核驱动 | 接口 | 特点 |
|---|---|---|---|
| spidev | 否 | `/dev/spidevB.C` + `ioctl()` | 用户态直接发起全双工传输,常用于调试或简单场景;需要在设备树里显式声明 `compatible = "spidev"` 才会创建设备节点 |
| spi_driver 客户端驱动 | 是 | 内核态 `probe()`/`remove()` + `spi_sync()`/`spi_write_then_read()` | 把设备接入内核设备模型,协议细节封装在驱动内,只给用户态暴露一个简单接口 |

## 相关文档

- [spidev 接口](./spidev.md) — 不编写内核驱动,直接从用户态操作总线上的从设备
- [spi_driver 驱动](./spi-driver.md) — 编写一个客户端驱动,读取从设备的一个寄存器,并通过字符设备暴露给用户态
