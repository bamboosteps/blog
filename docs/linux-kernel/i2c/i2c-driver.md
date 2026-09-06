---
sidebar_position: 3
title: i2c_driver 驱动封装寄存器
description: 编写一个 i2c_driver 客户端驱动,用 i2c_smbus_read_byte_data 封装单字节寄存器读取,通过字符设备暴露给用户态,基于 Linux 4.9。
---

`i2c_driver` 是内核态的客户端驱动框架:`struct i2c_driver` 填写 `probe()`/`remove()` 和匹配表,用 `module_i2c_driver()` 注册;`probe()` 拿到的 `struct i2c_client` 已带有地址、所属 adapter 等信息。以下驱动针对地址 `0x50`、寄存器 `0x00` 为 8 位只读状态寄存器的从设备,把"读这个寄存器"封装成一个字符设备。

## 架构与调用机制

```text
用户态 read(fd, buf, 1)
        │
        ▼
VFS → file_operations.read = demo_sensor_read()
        │
        ▼
i2c_smbus_read_byte_data(client, REG_STATUS)
        │
        ▼
adapter->algo->master_xfer()   总线控制器实际收发
        │
        ▼
copy_to_user()   拷贝结果回用户态缓冲区
```

| 内核对象 | 定位 |
|---|---|
| `struct i2c_client` | `probe()` 的参数,已含从设备地址、所属 adapter,不需要驱动指定地址 |
| `i2c_smbus_read_byte_data(client, reg)` | 单字节寄存器读,省去手动拼 `i2c_msg` |
| `i2c_smbus_write_byte_data(client, reg, val)` | 单字节寄存器写 |

## 配置设备树节点

```dts
&i2c1 {
    status = "okay";

    demo_sensor: sensor@50 {
        compatible = "example,demo-sensor";
        reg = <0x50>;
    };
};
```

| 属性 | 值 | 说明 |
|---|---|---|
| `compatible` | `"example,demo-sensor"` | 匹配驱动 `of_match_table` |
| `reg` | `<0x50>` | 从设备 7 位地址,`i2c_client` 自动携带,驱动代码里不需要硬编码 |

## 驱动程序实现

```text
demo_sensor_probe(client, id)
    │
    ├─ devm_kzalloc()                → 分配私有结构体 sensor
    ├─ sensor->client = client
    ├─ 填充 miscdev(minor / name / fops)
    ├─ i2c_set_clientdata(client, sensor)
    └─ misc_register(&sensor->miscdev)  → 创建 /dev/demo-sensor
```

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

:::tip[`id_table` 与 `of_match_table` 都要写]
两条匹配路径相互独立:走设备树的板卡靠 `of_match_table` 匹配,走传统 `i2c_board_info` 静态注册的板卡靠 `id_table` 匹配,4.9 内核建议两个都写。
:::

## 验证与控制

驱动加载并 probe 成功后,`/dev/demo-sensor` 节点自动出现,用户态不需要知道背后是哪个 I2C 地址、哪个寄存器:

```bash
cat /dev/demo-sensor | xxd
```

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
