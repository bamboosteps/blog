---
sidebar_position: 1
title: I2C 访问方案对比
description: Linux I2C 子系统分 adapter(总线控制器)与 client(从设备)两层,本文对比 i2c-dev 与 i2c_driver 两种从设备访问方案,基于 Linux 4.9。
---

I2C 是一种双线(SCL/SDA)总线协议,总线上有一个主设备(master)和若干从设备(slave),每个从设备有一个 7 位或 10 位地址。

## 架构与调用机制

```text
              adapter(总线控制器,SoC 提供,已由 BSP 实现)
                          │
        ┌─────────────────┼─────────────────┐
        ▼                                   ▼
  /dev/i2c-N + ioctl()              client 驱动 probe()
  (i2c-dev,不写内核代码)              (i2c_driver,内核态开发)
        │                                   │
        ▼                                   ▼
  i2c_master_send/recv()          i2c_smbus_read/write_byte_data()
        └───────────────┬───────────────────┘
                          ▼
              adapter->algo->master_xfer()
                          │
                          ▼
                    总线时序收发
```

| 方案 | 是否编写内核驱动 | 接口 | 特点 |
|---|---|---|---|
| i2c-dev | 否 | `/dev/i2c-N` + `ioctl()`/`read()`/`write()` | 用户态直接收发字节,常用于调试或简单场景 |
| i2c_driver 客户端驱动 | 是 | 内核态 `probe()`/`remove()` + `i2c_smbus_*` | 把设备接入内核设备模型,寄存器操作封装在驱动内,只给用户态暴露一个简单接口 |

## 相关文档

- [i2c-dev 接口](./i2c-dev.md) — 不编写内核驱动,直接从用户态操作总线上的从设备
- [i2c_driver 驱动](./i2c-driver.md) — 编写一个客户端驱动,读取从设备的一个寄存器,并通过字符设备暴露给用户态
