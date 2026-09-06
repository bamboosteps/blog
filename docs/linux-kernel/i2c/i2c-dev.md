---
sidebar_position: 2
title: i2c-dev 接口读取寄存器
description: i2c-dev 通过 ioctl 提供寻址、能力查询、裸字节收发、I2C_RDWR 复合传输与 I2C_SMBUS 标准传输,本文对比这些操作并从用户态读取从设备寄存器,基于 Linux 4.9。
---

`i2c-dev`(`drivers/i2c/i2c-dev.c`)是内核自带的通用适配器驱动,随 I2C adapter 注册自动出现为 `/dev/i2c-N`,不需要为具体从设备编写代码。以下示例针对地址 `0x50`、寄存器 `0x00` 为 8 位只读状态寄存器的从设备。

## 架构与调用机制

```text
用户态 ioctl(I2C_SLAVE / I2C_SLAVE_FORCE, addr)   指定/强制绑定从设备地址
用户态 ioctl(I2C_FUNCS, &funcs)                   查询适配器支持的传输能力
用户态 write()/read()                             i2c_master_send()/i2c_master_recv()
用户态 ioctl(I2C_RDWR, &data)                     提交多条 i2c_msg,重复起始不释放总线
用户态 ioctl(I2C_SMBUS, &args)                    标准 SMBus 语义传输
        │
        ▼
  i2c-dev(drivers/i2c/i2c-dev.c)
        │
        ▼
  adapter->algo->master_xfer() / ->smbus_xfer()   总线控制器实际收发
```

| 操作 | 内核路径 | 适用场景 |
|---|---|---|
| `ioctl(fd, I2C_SLAVE, addr)` | 设置 `file->private_data` 中的从设备地址 | 后续 `read`/`write`/`I2C_SMBUS` 前必须先调用一次 |
| `ioctl(fd, I2C_SLAVE_FORCE, addr)` | 同上,跳过"地址是否已被其他驱动占用"的检查 | 仅在确认该地址未被其他内核驱动使用时才用,强行绑定有冲突风险 |
| `ioctl(fd, I2C_FUNCS, &funcs)` | 返回 adapter 支持的功能位图(`I2C_FUNC_I2C`、`I2C_FUNC_SMBUS_*` 等) | 使用 `I2C_RDWR`/`I2C_SMBUS` 前检查适配器是否支持,也可用 `i2cdetect -F` 查看 |
| `write()` / `read()` | `i2c_master_send()` / `i2c_master_recv()` | 收发裸字节流,两次调用间总线会释放(产生 STOP) |
| `ioctl(fd, I2C_RDWR, &data)` | 提交 `struct i2c_msg` 数组,重复起始(repeated START) | 需要"先写寄存器地址、再读数据"且中途不能释放总线的芯片 |
| `ioctl(fd, I2C_SMBUS, &args)` | 提交 `struct i2c_smbus_ioctl_data`,adapter 原生支持则走 `smbus_xfer()`,否则由核心转换成 `i2c_msg` 走 `master_xfer()` | 标准 SMBus 语义读写(单字节/字/块),`i2cget`/`i2cset` 底层即调用此 ioctl |

`struct i2c_smbus_ioctl_data`(用户态头文件 `linux/i2c-dev.h`)描述一次 SMBus 传输:

| 字段 | 类型 | 作用 |
|---|---|---|
| `read_write` | `__u8` | `I2C_SMBUS_READ` 或 `I2C_SMBUS_WRITE` |
| `command` | `__u8` | 寄存器地址/命令字节 |
| `size` | `__u32` | 传输类型,如 `I2C_SMBUS_BYTE_DATA`、`I2C_SMBUS_WORD_DATA`、`I2C_SMBUS_BLOCK_DATA` |
| `data` | `union i2c_smbus_data *` | 传入/传出的数据缓冲区 |

## 验证与控制

安装独立于内核发布的 `i2c-tools`:

```bash
i2cdetect -l              # 列出系统里的 I2C adapter
i2cdetect -y 1             # 扫描 adapter 1 上有哪些地址被占用
i2cdetect -F 1             # 查看 adapter 1 支持的传输能力,等价于 I2C_FUNCS
i2cget -y 1 0x50 0x00      # 读取地址 0x50 设备的寄存器 0x00
```

分步 `write()` + `read()` 读取:

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

:::caution[`write()` 与 `read()` 之间总线会释放]
两次调用之间会产生 STOP。对时序要求严格的芯片可能因总线被其他设备占用而读到错误数据,应改用 `I2C_RDWR` 或 `I2C_SMBUS`。
:::

`I2C_RDWR` 在一次 `ioctl` 里提交两条 `i2c_msg`,中间仅插入一次重复起始,不产生 STOP:

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

`I2C_SMBUS` 表达的是"读/写一个寄存器"这一标准语义,不需要手工拼 `i2c_msg`,是 `i2cget` 命令的底层实现:

<details>
<summary>Show code</summary>

```c
// i2c_dev_smbus.c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/i2c-dev.h>
#include <linux/i2c.h>

static __s32 smbus_read_byte_data(int fd, __u8 command)
{
    union i2c_smbus_data data;
    struct i2c_smbus_ioctl_data args = {
        .read_write = I2C_SMBUS_READ,
        .command = command,
        .size = I2C_SMBUS_BYTE_DATA,
        .data = &data,
    };

    if (ioctl(fd, I2C_SMBUS, &args) < 0)
        return -1;

    return data.byte;
}

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

    __s32 value = smbus_read_byte_data(fd, 0x00);
    if (value < 0) {
        perror("I2C_SMBUS");
        close(fd);
        return 1;
    }

    printf("reg 0x00 = 0x%02x\n", value);
    close(fd);
    return 0;
}
```

</details>

:::tip[三种读法的选择顺序]
`write()`+`read()` 时序最弱,仅适合总线上无并发访问的简单场景;`I2C_RDWR` 控制粒度最细,适合手册明确要求重复起始、且传输格式超出标准 SMBus 语义(如批量寄存器)的芯片;`I2C_SMBUS` 语义最贴近"读写一个寄存器",能用则优先用,内核和 i2c-tools 都以它作为标准接口。
:::
