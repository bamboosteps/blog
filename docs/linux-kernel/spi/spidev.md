---
sidebar_position: 2
title: spidev 接口
description: 不编写内核驱动,通过 spidev 字符设备接口从用户态直接操作 SPI 总线上的从设备,基于 Linux 4.9。
---

## 核心概念

`spidev`(`drivers/spi/spidev.c`)是内核自带的通用从设备驱动,但和 i2c-dev 不同,它**不会自动出现**——必须在设备树里把某个 SPI 从设备节点的 `compatible` 显式写成 `"spidev"`,对应的 `/dev/spidevB.C` 才会被创建(`B` 是总线号,`C` 是片选号)。核心操作是 `ioctl(fd, SPI_IOC_MESSAGE(N), transfers)`:传入 N 个 `struct spi_ioc_transfer`,每个描述一次全双工传输(同时指定发送缓冲区 `tx_buf` 和接收缓冲区 `rx_buf`),内核收到后直接组装成一次 `spi_sync()` 提交给控制器驱动;此外还有 `SPI_IOC_WR_MODE`/`SPI_IOC_RD_MODE`、`SPI_IOC_WR_MAX_SPEED_HZ` 等 ioctl 用来配置模式和速率。以下针对一个假想从设备操作:发送命令字节 `0x9F` 后,芯片会在紧接着的 3 个字节里返回一个 ID。

## 声明 spidev 设备树节点

```dts
&spi0 {
    status = "okay";

    demo_flash: flash@0 {
        compatible = "spidev";
        reg = <0>;              /* 片选号 0 */
        spi-max-frequency = <10000000>;
    };
};
```

:::caution
`compatible = "spidev"` 仅用于调试或临时接入,不建议出现在正式产品的设备树里。正式驱动应改用 [spi_driver 驱动](./spi-driver.md)。
:::

## 读取芯片 ID

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

SPI 是全双工的,发送 `tx` 的同时一定会收到等长的 `rx`:发送命令字节 `0x9F` 那一个字节周期里,`rx[0]` 是无意义的数据(芯片还没来得及响应),真正的 ID 在紧随其后的 3 个字节里,即 `rx[1]`、`rx[2]`、`rx[3]`。`tx_buf`/`rx_buf` 在结构体里是 `__u64`,传裸指针时需要强制转换成整数。
