---
sidebar_position: 3
title: i2c_driver 驱动
description: 编写一个 i2c_driver 客户端驱动,读取从设备的一个寄存器,并通过字符设备暴露给用户态,基于 Linux 4.9。
---

## 核心概念

`i2c_driver` 是内核态的客户端驱动框架。`struct i2c_driver` 填写 `probe()`/`remove()` 和匹配表(`id_table` 或设备树 `of_match_table`),用 `i2c_add_driver()` / `module_i2c_driver()` 注册;`probe()` 的参数是 `struct i2c_client`,已经带有地址、所属 adapter 等信息,不需要驱动自己再指定地址;单字节寄存器读写最常用 `i2c_smbus_read_byte_data(client, reg)` / `i2c_smbus_write_byte_data(client, reg, val)`,省去手动拼 `i2c_msg` 的过程,底层仍通过 adapter 的 `master_xfer` 完成收发。以下驱动针对地址 `0x50`、寄存器 `0x00` 为 8 位只读状态寄存器的从设备,把"读这个寄存器"封装成一个字符设备。

## 编写驱动

### 编写设备树节点

```dts
&i2c1 {
    status = "okay";

    demo_sensor: sensor@50 {
        compatible = "example,demo-sensor";
        reg = <0x50>;
    };
};
```

`reg` 是从设备的 7 位地址,`i2c_client` 会自动携带这个地址,不需要在驱动代码里硬编码。

### 实现驱动代码

<details>
<summary>Show code</summary>

```c
// drivers/misc/i2c-demo-sensor.c
#include <linux/module.h>
#include <linux/i2c.h>
#include <linux/of.h>
#include <linux/miscdevice.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

#define REG_STATUS 0x00

struct demo_sensor {
    struct i2c_client *client;
    struct miscdevice miscdev;
};

static inline struct demo_sensor *to_demo_sensor(struct file *file)
{
    return container_of(file->private_data, struct demo_sensor, miscdev);
}

static ssize_t demo_sensor_read(struct file *file, char __user *buf,
                                 size_t count, loff_t *ppos)
{
    struct demo_sensor *sensor = to_demo_sensor(file);
    s32 value;
    u8 byte;

    if (*ppos > 0)
        return 0;              /* 已经读过一次,返回 EOF */
    if (count < 1)
        return -EINVAL;

    value = i2c_smbus_read_byte_data(sensor->client, REG_STATUS);
    if (value < 0)
        return value;

    byte = value;
    if (copy_to_user(buf, &byte, 1))
        return -EFAULT;

    *ppos += 1;
    return 1;
}

static const struct file_operations demo_sensor_fops = {
    .owner = THIS_MODULE,
    .read = demo_sensor_read,
};

static const struct of_device_id demo_sensor_of_match[] = {
    { .compatible = "example,demo-sensor" },
    { }
};
MODULE_DEVICE_TABLE(of, demo_sensor_of_match);

static const struct i2c_device_id demo_sensor_id[] = {
    { "demo-sensor", 0 },
    { }
};
MODULE_DEVICE_TABLE(i2c, demo_sensor_id);

static int demo_sensor_probe(struct i2c_client *client,
                              const struct i2c_device_id *id)
{
    struct demo_sensor *sensor;

    sensor = devm_kzalloc(&client->dev, sizeof(*sensor), GFP_KERNEL);
    if (!sensor)
        return -ENOMEM;

    sensor->client = client;
    sensor->miscdev.minor = MISC_DYNAMIC_MINOR;
    sensor->miscdev.name = "demo-sensor";
    sensor->miscdev.fops = &demo_sensor_fops;

    i2c_set_clientdata(client, sensor);

    return misc_register(&sensor->miscdev);
}

static int demo_sensor_remove(struct i2c_client *client)
{
    struct demo_sensor *sensor = i2c_get_clientdata(client);

    misc_deregister(&sensor->miscdev);
    return 0;
}

static struct i2c_driver demo_sensor_driver = {
    .driver = {
        .name = "demo-sensor",
        .of_match_table = demo_sensor_of_match,
    },
    .probe = demo_sensor_probe,
    .remove = demo_sensor_remove,
    .id_table = demo_sensor_id,
};
module_i2c_driver(demo_sensor_driver);

MODULE_LICENSE("GPL v2");
MODULE_DESCRIPTION("Minimal example I2C client driver");
```

</details>

### 梳理执行流程

驱动加载时的注册流程:

```
insmod i2c-demo-sensor.ko
        │
        ▼
module_i2c_driver() → i2c_add_driver()
        │
        ▼
I2C 核心按 compatible/id_table 匹配设备树里的 sensor@50
        │
        ▼
demo_sensor_probe(client, id) 被调用
        │
        ├─ devm_kzalloc()                     分配私有结构体 sensor
        ├─ sensor->client = client
        ├─ 填充 miscdev(minor / name / fops)
        ├─ i2c_set_clientdata(client, sensor)
        └─ misc_register(&sensor->miscdev) → 创建 /dev/demo-sensor
```

用户态读取时的调用流程:

```
用户态 read(fd, buf, 1)
        │
        ▼
VFS 根据 file_operations 调用 demo_sensor_read()
        │
        ▼
i2c_smbus_read_byte_data(sensor->client, REG_STATUS)
        │
        ▼
adapter->algo->master_xfer()   实际的 I2C 总线收发
        │
        ▼
copy_to_user()   把结果拷贝回用户态缓冲区
```

### 对照相关写法

流程图里几个关键函数/宏的原理统一放在了[《常用内核写法速查》](../kernel-code-reference.md)里,这里只列出对应链接:

- [`container_of`](../kernel-code-reference.md#container-of)(这里反推的是 `miscdev`,它是 `demo_sensor` 的第二个成员,偏移量不是 0)
- [`probe()` 触发机制(I2C 总线)](../kernel-code-reference.md#probe)
- [`struct file_operations` / `.read`](../kernel-code-reference.md#file-operations)
- [`copy_to_user`](../kernel-code-reference.md#copy-to-user)
- [`i2c_set_clientdata` / `i2c_get_clientdata`](../kernel-code-reference.md#set-drvdata)
- [`MISC_DYNAMIC_MINOR` / `misc_register`](../kernel-code-reference.md#misc-device)

`i2c_device_id` 和 `of_match_table` 是两条独立的匹配路径,4.9 内核建议两个都写:走设备树的板卡靠前者匹配,走传统 `i2c_board_info` 静态注册的板卡靠后者匹配。

## 读取传感器状态

驱动加载并 probe 成功后,`/dev/demo-sensor` 节点自动出现,用户态不需要知道背后是哪个 I2C 地址、哪个寄存器:

```bash
cat /dev/demo-sensor | xxd
```

或者用 C 代码:

<details>
<summary>Show code</summary>

```c
// read_sensor.c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    int fd = open("/dev/demo-sensor", O_RDONLY);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    unsigned char value;
    if (read(fd, &value, 1) != 1) {
        perror("read");
        close(fd);
        return 1;
    }

    printf("status = 0x%02x\n", value);
    close(fd);
    return 0;
}
```

</details>

对比[i2c-dev 接口](./i2c-dev.md):那里的应用需要知道总线号、从设备地址、寄存器编号;这里的应用只需要 `open()` + `read()` 一个语义明确的设备节点,寄存器细节被封装在驱动里,这正是编写客户端驱动的意义所在。
