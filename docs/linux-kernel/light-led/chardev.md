---
sidebar_position: 3
title: 字符设备接口点亮 LED
description: gpiolib 为每个注册成功的 gpio_chip 自动提供 /dev/gpiochipN 字符设备接口,本文实现一个最小 GPIO 控制器驱动并通过 ioctl 验证,基于 Linux 4.9。
---

字符设备接口(`/dev/gpiochipN`)和 sysfs 一样,是 gpiolib 为每一个注册成功的 `gpio_chip` 自动提供的功能,由 `CONFIG_GPIO_CDEV` 控制(Linux 4.8 引入)。chardev 与 sysfs 只是两种不同入口(`ioctl()` 与文件读写),内核内部最终都走到同一个 `gpio_chip` 回调函数。

:::tip[前置条件]
编译前在 `menuconfig` 中开启 `CONFIG_GPIOLIB`、`CONFIG_OF_GPIO`、`CONFIG_GPIO_CDEV`,并准备 Linux 4.9 源码树和支持设备树的开发板。
:::

## 架构与调用机制

用户态通过 `ioctl()` 操作 `/dev/gpiochipN` 到寄存器电平变化的路径:

```text
用户态 ioctl(GPIO_GET_LINEHANDLE_IOCTL / GPIOHANDLE_SET_LINE_VALUES_IOCTL)
        │
        ▼
linehandle_create() / linehandle_ioctl()   drivers/gpio/gpiolib.c
        │
        ▼
gpiod_direction_output() / gpiod_set_array_value_complex()
        │
        ▼
chip->direction_output() / chip->set()     控制器驱动实现的回调
        │
        ▼
写寄存器,引脚电平变化
```

| ioctl 命令 | 处理函数 | 内部调用 |
|---|---|---|
| `GPIO_GET_LINEHANDLE_IOCTL` | `linehandle_create()` | 对请求的每个偏移调用 `gpiod_request()`,再按 `flags` 调用 `gpiod_direction_output()` 或 `gpiod_direction_input()` |
| `GPIOHANDLE_SET_LINE_VALUES_IOCTL` | `linehandle_ioctl()` | `gpiod_set_array_value_complex()` |
| `GPIOHANDLE_GET_LINE_VALUES_IOCTL` | `linehandle_ioctl()` | `gpiod_get_value_cansleep()` |

这些 `gpiod_*` 函数和 sysfs 用的是同一套,最终同样落到 `gpio_chip` 自己实现的回调上:

```c
// drivers/gpio/gpiolib.c(节选)
err = chip->direction_output(chip, offset, val);
err = chip->direction_input(chip, offset);
chip->set(chip, offset, value);      /* 经 gpiod_set_array_value_complex 间接调用 */
```

## 配置设备树节点

目标是一个假想的内存映射 GPIO 控制器:占用 8 字节寄存器空间,映射在物理地址 `0x10000000`,偏移 `0x00` 为数据寄存器、`0x04` 为方向寄存器(对应位为 1 表示输出),共管理 8 根引脚。

```dts
simple_gpio: gpio@10000000 {
    compatible = "example,simple-gpio";
    reg = <0x10000000 0x8>;
    gpio-controller;
    #gpio-cells = <2>;
};
```

| 属性 | 值 | 说明 |
|---|---|---|
| `compatible` | `"example,simple-gpio"` | 匹配驱动 `of_match_table` |
| `reg` | `<0x10000000 0x8>` | 寄存器物理地址与长度 |
| `gpio-controller` | 空属性 | 标记该节点是一个 GPIO 控制器 |
| `#gpio-cells` | `<2>` | 消费者引用时需提供的参数个数:引脚偏移 + 极性标志 |

## 驱动程序实现

`struct gpio_chip` 必须填写的字段(定义于 `include/linux/gpio/driver.h`):

| 字段 | 类型 | 作用 |
|---|---|---|
| `label` | `const char *` | 控制器名称,显示在 debugfs 中 |
| `parent` | `struct device *` | 对应的物理设备 |
| `owner` | `struct module *` | 一般填 `THIS_MODULE` |
| `base` | `int` | 起始全局编号,`-1` 表示由内核自动分配 |
| `ngpio` | `u16` | 该控制器管理的引脚数量 |
| `get_direction` | 函数指针 | 查询引脚方向 |
| `direction_input` | 函数指针 | 设置引脚为输入 |
| `direction_output` | 函数指针 | 设置引脚为输出并给定初始电平 |
| `get` | 函数指针 | 读取引脚电平 |
| `set` | 函数指针 | 设置引脚电平 |
| `of_node` | `struct device_node *` | 对应的设备树节点 |

`probe()` 阶段执行顺序:

```text
simple_gpio_probe(pdev)
    │
    ├─ devm_kzalloc()                          → 分配私有结构体 sg
    ├─ platform_get_resource() + devm_ioremap_resource() → 映射寄存器到 sg->base
    ├─ spin_lock_init()
    ├─ 填充 sg->chip 各字段(direction_input/output、get、set ...)
    ├─ platform_set_drvdata(pdev, sg)
    └─ devm_gpiochip_add_data(dev, &sg->chip, sg)  → 注册进 gpiolib,自动创建 /dev/gpiochipN
```

<details>
<summary>Show code</summary>

```c
// drivers/gpio/gpio-simple-demo.c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/io.h>
#include <linux/gpio/driver.h>
#include <linux/spinlock.h>
#include <linux/slab.h>

#define REG_DATA   0x00
#define REG_DIR    0x04

struct simple_gpio {
    struct gpio_chip chip;
    void __iomem *base;
    spinlock_t lock;
};

static inline struct simple_gpio *to_simple_gpio(struct gpio_chip *gc)
{
    return container_of(gc, struct simple_gpio, chip);
}

static int simple_gpio_get_direction(struct gpio_chip *gc, unsigned offset)
{
    struct simple_gpio *sg = to_simple_gpio(gc);
    u32 dir = readl(sg->base + REG_DIR);

    return !(dir & BIT(offset));
}

static int simple_gpio_direction_input(struct gpio_chip *gc, unsigned offset)
{
    struct simple_gpio *sg = to_simple_gpio(gc);
    unsigned long flags;
    u32 val;

    spin_lock_irqsave(&sg->lock, flags);
    val = readl(sg->base + REG_DIR);
    val &= ~BIT(offset);
    writel(val, sg->base + REG_DIR);
    spin_unlock_irqrestore(&sg->lock, flags);

    return 0;
}

static int simple_gpio_direction_output(struct gpio_chip *gc, unsigned offset, int value)
{
    struct simple_gpio *sg = to_simple_gpio(gc);
    unsigned long flags;
    u32 val;

    spin_lock_irqsave(&sg->lock, flags);

    /* 先设置电平,再切换方向,避免切换瞬间输出上一次遗留的电平 */
    val = readl(sg->base + REG_DATA);
    if (value)
        val |= BIT(offset);
    else
        val &= ~BIT(offset);
    writel(val, sg->base + REG_DATA);

    val = readl(sg->base + REG_DIR);
    val |= BIT(offset);
    writel(val, sg->base + REG_DIR);

    spin_unlock_irqrestore(&sg->lock, flags);

    return 0;
}

static int simple_gpio_get(struct gpio_chip *gc, unsigned offset)
{
    struct simple_gpio *sg = to_simple_gpio(gc);

    return !!(readl(sg->base + REG_DATA) & BIT(offset));
}

static void simple_gpio_set(struct gpio_chip *gc, unsigned offset, int value)
{
    struct simple_gpio *sg = to_simple_gpio(gc);
    unsigned long flags;
    u32 val;

    spin_lock_irqsave(&sg->lock, flags);
    val = readl(sg->base + REG_DATA);
    if (value)
        val |= BIT(offset);
    else
        val &= ~BIT(offset);
    writel(val, sg->base + REG_DATA);
    spin_unlock_irqrestore(&sg->lock, flags);
}

static const struct of_device_id simple_gpio_of_match[] = {
    { .compatible = "example,simple-gpio" },
    { }
};
MODULE_DEVICE_TABLE(of, simple_gpio_of_match);

static int simple_gpio_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct simple_gpio *sg;
    struct resource *res;

    sg = devm_kzalloc(dev, sizeof(*sg), GFP_KERNEL);
    if (!sg)
        return -ENOMEM;

    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    sg->base = devm_ioremap_resource(dev, res);
    if (IS_ERR(sg->base))
        return PTR_ERR(sg->base);

    spin_lock_init(&sg->lock);

    sg->chip.label = "simple-gpio";
    sg->chip.parent = dev;
    sg->chip.owner = THIS_MODULE;
    sg->chip.base = -1;
    sg->chip.ngpio = 8;
    sg->chip.get_direction = simple_gpio_get_direction;
    sg->chip.direction_input = simple_gpio_direction_input;
    sg->chip.direction_output = simple_gpio_direction_output;
    sg->chip.get = simple_gpio_get;
    sg->chip.set = simple_gpio_set;
    sg->chip.of_node = dev->of_node;
    sg->chip.can_sleep = false;

    platform_set_drvdata(pdev, sg);

    return devm_gpiochip_add_data(dev, &sg->chip, sg);
}

static struct platform_driver simple_gpio_driver = {
    .driver = {
        .name = "simple-gpio",
        .of_match_table = simple_gpio_of_match,
    },
    .probe = simple_gpio_probe,
};
module_platform_driver(simple_gpio_driver);

MODULE_LICENSE("GPL v2");
MODULE_DESCRIPTION("Minimal example GPIO controller driver");
```

</details>

:::tip[寄存器写顺序]
`direction_output` 中先写数据寄存器、再写方向寄存器,避免切换瞬间输出上一次遗留的电平。`chip.base = -1` 表示编号由内核自动分配,避免与系统里其他 GPIO 控制器的编号段冲突。
:::

## 验证与控制

```bash
make -C /path/to/linux-4.9 M=$(pwd) modules
insmod gpio-simple-demo.ko
dmesg | tail
```

驱动加载成功后,用 `gpiodetect`/`gpioinfo` 或 `cat /sys/kernel/debug/gpio` 确认对应的 `gpiochipN` 和引脚偏移(以下假设控制器是 `gpiochip0`,LED 接在偏移 `3`)。`libgpiod` 是一套独立于内核发布的用户态工具与库,需单独编译安装:

```bash
gpiodetect              # 列出系统中的 gpiochip 设备
gpioinfo gpiochip0       # 查看该 chip 下每个引脚的编号、方向、占用情况
gpioget gpiochip0 3      # 读取 3 号引脚的电平
gpioset gpiochip0 3=1    # 将 3 号引脚设置为高电平
```

不依赖 `libgpiod`,也可以直接调用底层 ioctl 完成同样的操作。以下代码基于 Linux 4.9 的 `include/uapi/linux/gpio.h`,请求 3 号引脚为输出并写入电平:

<details>
<summary>Show code</summary>

```c
// led_chardev.c
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/gpio.h>

int main(void)
{
    struct gpiohandle_request req;
    struct gpiohandle_data data;
    int chip_fd, ret;

    chip_fd = open("/dev/gpiochip0", O_RDONLY);
    if (chip_fd < 0) {
        perror("open");
        return 1;
    }

    memset(&req, 0, sizeof(req));
    req.lineoffsets[0] = 3;
    req.flags = GPIOHANDLE_REQUEST_OUTPUT;
    req.default_values[0] = 0;
    req.lines = 1;
    strncpy(req.consumer_label, "led-demo", sizeof(req.consumer_label) - 1);

    ret = ioctl(chip_fd, GPIO_GET_LINEHANDLE_IOCTL, &req);
    if (ret < 0) {
        perror("GPIO_GET_LINEHANDLE_IOCTL");
        close(chip_fd);
        return 1;
    }
    close(chip_fd); /* 已经拿到 req.fd,可以关闭 chip 本身的 fd */

    /* 点亮 LED */
    data.values[0] = 1;
    ioctl(req.fd, GPIOHANDLE_SET_LINE_VALUES_IOCTL, &data);

    sleep(1);

    /* 熄灭 LED */
    data.values[0] = 0;
    ioctl(req.fd, GPIOHANDLE_SET_LINE_VALUES_IOCTL, &data);

    close(req.fd);
    return 0;
}
```

</details>

`GPIO_GET_LINEHANDLE_IOCTL` 触发 `linehandle_create()`,内部调用 `gpiod_request()` + `gpiod_direction_output()`,对应驱动里的 `simple_gpio_direction_output()`;`GPIOHANDLE_SET_LINE_VALUES_IOCTL` 触发 `linehandle_ioctl()` 里的 `gpiod_set_array_value_complex()`,最终调用到 `simple_gpio_set()`。

:::caution[拿到行句柄后及时关闭 chip fd]
`ioctl(GPIO_GET_LINEHANDLE_IOCTL)` 成功后,实际用于读写电平的是返回的 `req.fd`,不再是打开 `/dev/gpiochipN` 得到的 `chip_fd`。可以立即关闭 `chip_fd`,但 `req.fd` 要留到操作结束后再关闭。
:::
