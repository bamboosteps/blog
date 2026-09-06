---
sidebar_position: 3
title: spi_driver 驱动
description: 编写一个 spi_driver 客户端驱动,读取从设备的 ID,并通过字符设备暴露给用户态,基于 Linux 4.9。
---

## 核心概念

针对[上一节](./spidev.md)同一颗假想从设备(发送命令字节 `0x9F` 后读取 3 字节 ID),把这个过程封装成一个字符设备,用户态只需要 `read()` 就能拿到结果。`spi_driver` 是内核态的客户端驱动框架,核心是三个概念:`struct spi_driver` 填写 `probe(struct spi_device *)`/`remove(struct spi_device *)` 和匹配表(`id_table` 或设备树 `of_match_table`),用 `spi_register_driver()` / `module_spi_driver()` 注册;`struct spi_device` 代表总线上具体的这一颗从设备,`probe()` 的参数就是它,内部已经包含了片选号、所属控制器等信息;`spi_write_then_read(spi, txbuf, n_tx, rxbuf, n_rx)` 先发送 `n_tx` 字节,片选保持有效,紧接着再读回 `n_rx` 字节,是"发命令再读响应"这类场景最常用的封装,比自己拼 `spi_message`/`spi_transfer` 更简单;需要完全控制传输细节(比如同时收发的全双工场景)时才需要用更底层的 `spi_sync()`。

## 编写驱动

### 编写设备树节点

```dts
&spi0 {
    status = "okay";

    demo_flash: flash@0 {
        compatible = "example,demo-flash";
        reg = <0>;               /* 片选号 0 */
        spi-max-frequency = <10000000>;
    };
};
```

### 实现驱动代码

<details>
<summary>Show code</summary>

```c
// drivers/misc/spi-demo-flash.c
#include <linux/module.h>
#include <linux/spi/spi.h>
#include <linux/of.h>
#include <linux/miscdevice.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

#define CMD_READ_ID 0x9F
#define ID_LEN      3

struct demo_flash {
    struct spi_device *spi;
    struct miscdevice miscdev;
};

static inline struct demo_flash *to_demo_flash(struct file *file)
{
    return container_of(file->private_data, struct demo_flash, miscdev);
}

static ssize_t demo_flash_read(struct file *file, char __user *buf,
                                size_t count, loff_t *ppos)
{
    struct demo_flash *flash = to_demo_flash(file);
    u8 cmd = CMD_READ_ID;
    u8 id[ID_LEN];
    int ret;

    if (*ppos > 0)
        return 0;               /* 已经读过一次,返回 EOF */
    if (count < ID_LEN)
        return -EINVAL;

    ret = spi_write_then_read(flash->spi, &cmd, 1, id, ID_LEN);
    if (ret < 0)
        return ret;

    if (copy_to_user(buf, id, ID_LEN))
        return -EFAULT;

    *ppos += ID_LEN;
    return ID_LEN;
}

static const struct file_operations demo_flash_fops = {
    .owner = THIS_MODULE,
    .read = demo_flash_read,
};

static const struct of_device_id demo_flash_of_match[] = {
    { .compatible = "example,demo-flash" },
    { }
};
MODULE_DEVICE_TABLE(of, demo_flash_of_match);

static const struct spi_device_id demo_flash_id[] = {
    { "demo-flash", 0 },
    { }
};
MODULE_DEVICE_TABLE(spi, demo_flash_id);

static int demo_flash_probe(struct spi_device *spi)
{
    struct demo_flash *flash;

    flash = devm_kzalloc(&spi->dev, sizeof(*flash), GFP_KERNEL);
    if (!flash)
        return -ENOMEM;

    spi->mode = SPI_MODE_0;
    spi->bits_per_word = 8;
    if (spi_setup(spi))
        return -EINVAL;

    flash->spi = spi;
    flash->miscdev.minor = MISC_DYNAMIC_MINOR;
    flash->miscdev.name = "demo-flash";
    flash->miscdev.fops = &demo_flash_fops;

    spi_set_drvdata(spi, flash);

    return misc_register(&flash->miscdev);
}

static int demo_flash_remove(struct spi_device *spi)
{
    struct demo_flash *flash = spi_get_drvdata(spi);

    misc_deregister(&flash->miscdev);
    return 0;
}

static struct spi_driver demo_flash_driver = {
    .driver = {
        .name = "demo-flash",
        .of_match_table = demo_flash_of_match,
    },
    .probe = demo_flash_probe,
    .remove = demo_flash_remove,
    .id_table = demo_flash_id,
};
module_spi_driver(demo_flash_driver);

MODULE_LICENSE("GPL v2");
MODULE_DESCRIPTION("Minimal example SPI client driver");
```

</details>

### 梳理执行流程

驱动加载时的注册流程:

```
insmod spi-demo-flash.ko
        │
        ▼
module_spi_driver() → spi_register_driver()
        │
        ▼
SPI 核心按 compatible/id_table 匹配设备树里的 flash@0
        │
        ▼
demo_flash_probe(spi) 被调用
        │
        ├─ devm_kzalloc()                   分配私有结构体 flash
        ├─ spi_setup(spi)                   应用 mode / bits_per_word
        ├─ 填充 miscdev(minor / name / fops)
        ├─ spi_set_drvdata(spi, flash)
        └─ misc_register(&flash->miscdev) → 创建 /dev/demo-flash
```

用户态读取时的调用流程:

```
用户态 read(fd, buf, 3)
        │
        ▼
VFS 根据 file_operations 调用 demo_flash_read()
        │
        ▼
spi_write_then_read(flash->spi, &cmd, 1, id, 3)
        │
        ▼
spi_sync()   控制器驱动实际收发
        │
        ▼
copy_to_user()   把结果拷贝回用户态缓冲区
```

### 对照相关写法

流程图里几个关键函数/宏的原理统一放在了[《常用内核写法速查》](../kernel-code-reference.md)里,这里只列出对应链接:

- [`container_of`](../kernel-code-reference.md#container-of)(这里反推的是 `miscdev`,和 [i2c_driver 驱动](../i2c/i2c-driver.md)里的用法是同一个手法)
- [`probe()` 触发机制(SPI 总线)](../kernel-code-reference.md#probe)
- [`struct file_operations` / `.read`](../kernel-code-reference.md#file-operations)
- [`copy_to_user`](../kernel-code-reference.md#copy-to-user)
- [`spi_set_drvdata` / `spi_get_drvdata`](../kernel-code-reference.md#set-drvdata)
- [`MISC_DYNAMIC_MINOR` / `misc_register`](../kernel-code-reference.md#misc-device)

## 读取芯片 ID

驱动加载并 probe 成功后,`/dev/demo-flash` 节点会自动出现,用户态不需要知道命令字节、也不需要处理全双工的哑字节:

```bash
cat /dev/demo-flash | xxd
```

或者用 C 代码:

<details>
<summary>Show code</summary>

```c
// read_flash_id.c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    int fd = open("/dev/demo-flash", O_RDONLY);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    unsigned char id[3];
    if (read(fd, id, sizeof(id)) != sizeof(id)) {
        perror("read");
        close(fd);
        return 1;
    }

    printf("id = %02x %02x %02x\n", id[0], id[1], id[2]);
    close(fd);
    return 0;
}
```

</details>

对比[spidev 接口](./spidev.md):那里的应用需要自己发命令字节、跳过哑字节读偏移;这里的应用只需要 `open()` + `read()` 一个语义明确的设备节点,协议细节被封装在驱动里。
