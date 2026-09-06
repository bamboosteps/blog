---
sidebar_position: 2
title: spidev 接口读取芯片 ID
description: spidev 是内核自带的通用从设备驱动,需在设备树中显式声明 compatible 为 spidev 才会创建节点,本文用它从用户态发起全双工传输读取芯片 ID,基于 Linux 4.9。
---

`spidev`(`drivers/spi/spidev.c`)是内核自带的通用从设备驱动,和 `i2c-dev` 不同,它**不会自动出现**——必须在设备树里把某个 SPI 从设备节点的 `compatible` 显式写成 `"spidev"`,对应的 `/dev/spidevB.C` 才会被创建(`B` 是总线号,`C` 是片选号)。以下针对一个假想从设备操作:发送命令字节 `0x9F` 后,芯片会在紧接着的 3 个字节里返回一个 ID。

## 架构与调用机制

```text
用户态 ioctl(SPI_IOC_MESSAGE(N), transfers)
        │
        ▼
spidev(drivers/spi/spidev.c)
组装 N 个 spi_ioc_transfer → spi_message
        │
        ▼
spi_sync()   控制器驱动实际收发
```

| ioctl 命令 | 作用 |
|---|---|
| `SPI_IOC_MESSAGE(N)` | 提交 N 个 `struct spi_ioc_transfer`,每个描述一次全双工传输(同时指定 `tx_buf`/`rx_buf`),内核组装成一次 `spi_sync()` |
| `SPI_IOC_WR_MODE` / `SPI_IOC_RD_MODE` | 配置/读取 SPI 模式 |
| `SPI_IOC_WR_MAX_SPEED_HZ` | 配置传输速率 |

## 配置设备树节点

```dts
&spi0 {
    status = "okay";

    demo_flash: flash@0 {
        compatible = "spidev";
        reg = <0>;
        spi-max-frequency = <10000000>;
    };
};
```

| 属性 | 值 | 说明 |
|---|---|---|
| `compatible` | `"spidev"` | 触发内核创建 `/dev/spidevB.C` 节点 |
| `reg` | `<0>` | 片选号 |
| `spi-max-frequency` | `<10000000>` | 总线最大工作频率(Hz) |

:::caution[`compatible = "spidev"` 仅用于调试]
不建议出现在正式产品的设备树里。正式驱动应改用 [spi_driver 驱动](./spi-driver.md)。
:::

## 验证与控制

<details>
<summary>Show code</summary>

```c
// spidev_read_id.c
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/spi/spidev.h>

int main(void)
{
    int fd = open("/dev/spidev0.0", O_RDWR);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    __u8 mode = SPI_MODE_0;
    __u8 bits = 8;
    __u32 speed = 1000000;

    ioctl(fd, SPI_IOC_WR_MODE, &mode);
    ioctl(fd, SPI_IOC_WR_BITS_PER_WORD, &bits);
    ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);

    __u8 tx[4] = { 0x9F, 0x00, 0x00, 0x00 };  /* 命令字节 + 3 个哑字节 */
    __u8 rx[4] = { 0 };

    struct spi_ioc_transfer xfer = {
        .tx_buf = (unsigned long)tx,
        .rx_buf = (unsigned long)rx,
        .len = sizeof(tx),
        .speed_hz = speed,
        .bits_per_word = bits,
    };

    if (ioctl(fd, SPI_IOC_MESSAGE(1), &xfer) < 0) {
        perror("SPI_IOC_MESSAGE");
        close(fd);
        return 1;
    }

    printf("id = %02x %02x %02x\n", rx[1], rx[2], rx[3]);
    close(fd);
    return 0;
}
```

</details>

:::tip[全双工的哑字节偏移]
SPI 是全双工的,发送 `tx` 的同时一定会收到等长的 `rx`:发送命令字节 `0x9F` 那一个字节周期里,`rx[0]` 是无意义的数据(芯片还没来得及响应),真正的 ID 在紧随其后的 3 个字节里,即 `rx[1]`、`rx[2]`、`rx[3]`。`tx_buf`/`rx_buf` 在结构体里是 `__u64`,传裸指针时需要强制转换成整数。
:::
