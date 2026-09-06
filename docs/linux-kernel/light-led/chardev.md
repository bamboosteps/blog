---
sidebar_position: 3
title: chardev 驱动
description: 编写一个最小的 GPIO 控制器驱动(provider),通过字符设备(chardev)接口验证驱动、点亮 LED,基于 Linux 4.9。
---

## 核心概念

字符设备接口(`/dev/gpiochipN`)和 sysfs 一样,是 gpiolib 为每一个注册成功的 `gpio_chip` 自动提供的功能,由 `CONFIG_GPIO_CDEV` 控制(Linux 4.8 引入),不需要控制器驱动的作者额外写代码。chardev 和 sysfs 只是两种不同的入口(一个是 `ioctl()`,一个是文件读写),内核内部最终都走到同一个 `gpio_chip` 的回调函数。

:::tip 前置条件
编译前在 `menuconfig` 中开启 `CONFIG_GPIOLIB`、`CONFIG_OF_GPIO`、`CONFIG_GPIO_CDEV`,并准备 Linux 4.9 源码树和支持设备树的开发板。
:::

## 编写驱动

以下驱动针对一个假想的内存映射 GPIO 控制器:占用 8 字节寄存器空间,映射在物理地址 `0x10000000`,偏移 `0x00` 为数据寄存器、`0x04` 为方向寄存器(对应位为 1 表示输出),共管理 8 根引脚。

### 分析调用链

用户态通过 `ioctl()` 操作 `/dev/gpiochipN` 节点,内核侧的处理逻辑在 `drivers/gpio/gpiolib.c` 中:

| ioctl 命令 | 处理函数 | 内部调用 |
|---|---|---|
| `GPIO_GET_LINEHANDLE_IOCTL` | `linehandle_create()` | 对请求的每个偏移调用 `gpiod_request()`,再按 `flags` 调用 `gpiod_direction_output()` 或 `gpiod_direction_input()` |
| `GPIOHANDLE_SET_LINE_VALUES_IOCTL` | `linehandle_ioctl()` | `gpiod_set_array_value_complex()` |
| `GPIOHANDLE_GET_LINE_VALUES_IOCTL` | `linehandle_ioctl()` | `gpiod_get_value_cansleep()` |

这些 `gpiod_*` 函数和 sysfs 用的是同一套,最终同样会落到 `gpio_chip` 自己实现的回调上:

```c
// drivers/gpio/gpiolib.c(节选,函数名与参数为实际内核源码)
err = chip->direction_output(chip, offset, val);
err = chip->direction_input(chip, offset);
chip->set(chip, offset, value);      /* 经 gpiod_set_array_value_complex 间接调用 */
```

### 填写 gpio_chip 字段

| 字段 | 类型 | 作用 |
|---|---|---|
| `label` | `const char *` | 控制器名称,显示在 debugfs 中 |
| `parent` | `struct device *` | 对应的物理设备 |
| `owner` | `struct module *` | 一般填 `THIS_MODULE` |
| `base` | `int` | 起始全局编号,填 `-1` 表示由内核自动分配 |
| `ngpio` | `u16` | 该控制器管理的引脚数量 |
| `get_direction` | 函数指针 | 查询引脚方向 |
| `direction_input` | 函数指针 | 设置引脚为输入 |
| `direction_output` | 函数指针 | 设置引脚为输出并给定初始电平 |
| `get` | 函数指针 | 读取引脚电平 |
| `set` | 函数指针 | 设置引脚电平 |
| `of_node` | `struct device_node *` | 对应的设备树节点 |

### 编写设备树节点

```dts
simple_gpio: gpio@10000000 {
    compatible = "example,simple-gpio";
    reg = <0x10000000 0x8>;
    gpio-controller;
    #gpio-cells = <2>;
};
```

`gpio-controller` 是一个空属性,标记该节点是一个 GPIO 控制器。`#gpio-cells` 声明消费者引用该控制器时需要提供几个参数,惯例为 `2`:引脚偏移 + 极性标志。

### 实现驱动代码

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

### 梳理执行流程

```
insmod gpio-simple-demo.ko
        │
        ▼
module_platform_driver() 展开的 module_init()
        │
        ▼
platform_driver_register(&simple_gpio_driver)
        │
        ▼
platform 总线按 compatible = "example,simple-gpio" 匹配设备树节点
        │
        ▼
simple_gpio_probe(pdev) 被调用
        │
        ├─ devm_kzalloc()                            分配私有结构体 sg
        ├─ platform_get_resource() + devm_ioremap_resource()   映射寄存器 → sg->base
        ├─ spin_lock_init()
        ├─ 依次填好 sg->chip 的各个字段(direction_input/output、get、set ...)
        ├─ platform_set_drvdata(pdev, sg)
        └─ devm_gpiochip_add_data(dev, &sg->chip, sg) → 注册进 gpiolib
                    │
                    ▼
        gpiolib 自动创建 /dev/gpiochipN 节点
                    │
        用户态 ioctl(GPIO_GET_LINEHANDLE_IOCTL / GPIOHANDLE_SET_LINE_VALUES_IOCTL)
                    │
                    ▼
        gpiolib 调用 sg->chip 的 direction_output / set / get 回调
```

`sg->chip.base = -1` 表示编号由内核自动分配,避免和系统里其他 GPIO 控制器的编号段冲突;`direction_output` 里先写数据寄存器、再写方向寄存器,避免切换瞬间输出上一次遗留的电平。

### 对照相关写法

流程图里几个关键函数/宏的原理统一放在了[《常用内核写法速查》](../kernel-code-reference.md)里,这里只列出对应链接:

- [`container_of`](../kernel-code-reference.md#container-of)
- [`probe()` 触发机制(platform 总线)](../kernel-code-reference.md#probe)
- [`devm_kzalloc` / `devm_ioremap_resource`](../kernel-code-reference.md#devm)
- [`IS_ERR` / `PTR_ERR`](../kernel-code-reference.md#is-err-ptr-err)
- [`spin_lock_irqsave` / `spin_unlock_irqrestore`](../kernel-code-reference.md#spin-lock)
- [`MODULE_DEVICE_TABLE`](../kernel-code-reference.md#module-device-table)
- [`platform_set_drvdata` / `platform_get_drvdata`](../kernel-code-reference.md#set-drvdata)

## 编译并加载驱动

```makefile
obj-m += gpio-simple-demo.o
```

```bash
make -C /path/to/linux-4.9 M=$(pwd) modules
insmod gpio-simple-demo.ko
dmesg | tail
```

## 点亮 LED

驱动加载成功后,用 `gpiodetect`/`gpioinfo` 或 `cat /sys/kernel/debug/gpio` 确认对应的 `gpiochipN` 和引脚偏移(以下假设控制器是 `gpiochip0`,LED 接在偏移 `3`)。

### 运行命令行工具

`libgpiod` 是一套独立于内核发布的用户态工具与库,提供了对该接口的封装,需要单独编译安装:

```bash
gpiodetect              # 列出系统中的 gpiochip 设备
gpioinfo gpiochip0       # 查看该 chip 下每个引脚的编号、方向、占用情况
gpioget gpiochip0 3      # 读取 3 号引脚的电平
gpioset gpiochip0 3=1    # 将 3 号引脚设置为高电平
```

### 编写用户态程序

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

关键步骤对照"分析调用链"一节:`GPIO_GET_LINEHANDLE_IOCTL` 触发 `linehandle_create()`,内部调用 `gpiod_request()` + `gpiod_direction_output()`,对应到驱动里的 `simple_gpio_direction_output()`;`GPIOHANDLE_SET_LINE_VALUES_IOCTL` 触发 `linehandle_ioctl()` 里的 `gpiod_set_array_value_complex()`,最终调用到 `simple_gpio_set()`。

:::caution 拿到行句柄后及时关闭 chip fd
`ioctl(GPIO_GET_LINEHANDLE_IOCTL)` 成功后,实际用于读写电平的是返回的 `req.fd`,不再是打开 `/dev/gpiochipN` 得到的 `chip_fd`。可以立即关闭 `chip_fd`,但 `req.fd` 要留到操作结束后再关闭。
:::
