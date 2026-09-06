---
sidebar_position: 2
title: i2c-dev 接口
description: 不编写内核驱动,通过 i2c-dev 字符设备接口从用户态直接操作 I2C 总线上的从设备,基于 Linux 4.9。
---

## 核心概念

`i2c-dev`(`drivers/i2c/i2c-dev.c`)是内核自带的通用适配器驱动,随 I2C adapter 注册自动出现为 `/dev/i2c-N`,不需要为具体从设备编写代码。核心操作有三个:`ioctl(fd, I2C_SLAVE, addr)` 指定接下来通信的从设备地址;`write()`/`read()` 直接对应 `i2c_master_send()`/`i2c_master_recv()`,收发裸字节流;`ioctl(fd, I2C_RDWR, &data)` 一次提交多个 `struct i2c_msg`,中间不释放总线(重复起始,repeated start),用于访问需要"先写寄存器地址、再读数据"的芯片。以下示例针对地址 `0x50`、寄存器 `0x00` 为 8 位只读状态寄存器的从设备。

## 读取从设备寄存器

### 运行命令行工具

安装独立于内核发布的 `i2c-tools`,运行:

```bash
i2cdetect -l              # 列出系统里的 I2C adapter
i2cdetect -y 1             # 扫描 adapter 1 上有哪些地址被占用
i2cget -y 1 0x50 0x00      # 读取地址 0x50 设备的寄存器 0x00
```

### 编写分步读取程序

<details>
<summary>Show code</summary>

```c
// i2c_dev_read.c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/i2c-dev.h>

int main(void)
{
    int fd = open("/dev/i2c-1", O_RDWR);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    if (ioctl(fd, I2C_SLAVE, 0x50) < 0) {
        perror("I2C_SLAVE");
        close(fd);
        return 1;
    }

    unsigned char reg = 0x00;
    if (write(fd, &reg, 1) != 1) {
        perror("write");
        close(fd);
        return 1;
    }

    unsigned char value;
    if (read(fd, &value, 1) != 1) {
        perror("read");
        close(fd);
        return 1;
    }

    printf("reg 0x00 = 0x%02x\n", value);
    close(fd);
    return 0;
}
```

</details>

:::caution
先 `write()` 一次寄存器地址、再 `read()` 一次数据,两次调用之间总线会释放(产生 STOP)。对时序要求严格的芯片可能因总线被其他设备占用而读到错误数据。
:::

### 编写 I2C_RDWR 读取程序

<details>
<summary>Show code</summary>

```c
// i2c_dev_rdwr.c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/i2c-dev.h>
#include <linux/i2c.h>

int main(void)
{
    int fd = open("/dev/i2c-1", O_RDWR);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    unsigned char reg = 0x00;
    unsigned char value = 0;

    struct i2c_msg msgs[2] = {
        {
            .addr = 0x50,
            .flags = 0,           /* 写 */
            .len = 1,
            .buf = &reg,
        },
        {
            .addr = 0x50,
            .flags = I2C_M_RD,    /* 读 */
            .len = 1,
            .buf = &value,
        },
    };

    struct i2c_rdwr_ioctl_data data = {
        .msgs = msgs,
        .nmsgs = 2,
    };

    if (ioctl(fd, I2C_RDWR, &data) < 0) {
        perror("I2C_RDWR");
        close(fd);
        return 1;
    }

    printf("reg 0x00 = 0x%02x\n", value);
    close(fd);
    return 0;
}
```

</details>

两条 `i2c_msg` 在一次 `ioctl(I2C_RDWR)` 里提交,中间不产生 STOP,仅在第二条消息前插入一次重复起始(repeated START),是更贴近实际芯片手册要求的读法。
