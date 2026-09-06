---
sidebar_position: 4
title: gpiolib 框架点亮 LED
description: gpiolib 分 provider(实现 gpio_chip 注册控制器)与 consumer(用 gpiod_* 描述符 API 使用 GPIO)两种角色,本文实现一对最小驱动点亮 LED,基于 Linux 4.9。
---

gpiolib 分两种角色:**Provider(提供者)** 实现 `struct gpio_chip` 并调用 `gpiochip_add_data()` 完成注册,声明"这里有 N 根 GPIO,读写方法是这些函数";**Consumer(使用者)** 调用 `gpiod_get()` 等函数,根据设备树属性名拿到一个 `struct gpio_desc *` 描述符,再用 `gpiod_set_value()`/`gpiod_get_value()` 等统一接口操作,不关心该 GPIO 具体来自哪个控制器。

:::tip[前置条件]
阅读本节需要具备 `platform_driver` 框架与设备树基础语法的背景知识。编译前需在 `menuconfig` 中开启 `CONFIG_GPIOLIB`、`CONFIG_OF_GPIO`、`CONFIG_GPIO_CDEV`、`CONFIG_DEBUG_FS`。
:::

## 架构与调用机制

```text
Consumer 驱动                          Provider 驱动
devm_gpiod_get(dev, "led", ...)
        │
        ▼
gpiolib(include/linux/gpio/consumer.h)
根据 "led-gpios" 找到 gpio_desc
        │
        ▼
gpiod_direction_output() / gpiod_set_value_cansleep()
        │
        ▼
chip->direction_output() / chip->set()  ◄── struct gpio_chip 回调
                                                  │
                                                  ▼
                                            硬件寄存器,LED 点亮
```

| 侧 | 接口头文件 | 关键函数 |
|---|---|---|
| Provider | `include/linux/gpio/driver.h` | `gpiochip_add_data()`,实现 `gpio_chip` 各回调 |
| Consumer | `include/linux/gpio/consumer.h` | `gpiod_get()`、`gpiod_set_value()`、`gpiod_get_value()` |

Linux 3.13 引入 `gpiod_*` 描述符式 consumer API,取代早期基于全局整数编号的 `gpio_request()`/`gpio_set_value()` 系列接口(`include/linux/gpio.h`);4.9 内核编写新代码统一使用描述符式 API。

## 配置设备树节点

目标 GPIO 控制器是一个内存映射寄存器控制器,占用 8 字节寄存器空间,映射在物理地址 `0x10000000`:

```dts
simple_gpio: gpio@10000000 {
    compatible = "example,simple-gpio";
    reg = <0x10000000 0x8>;
    gpio-controller;
    #gpio-cells = <2>;
};

led0: led {
    compatible = "example,simple-led";
    led-gpios = <&simple_gpio 3 GPIO_ACTIVE_HIGH>;
};
```

| 属性 | 值 | 说明 |
|---|---|---|
| `gpio-controller` | 空属性 | 标记该节点是一个 GPIO 控制器 |
| `#gpio-cells` | `<2>` | 消费者引用参数个数:引脚偏移 + 极性标志 |
| `led-gpios` | `<&simple_gpio 3 GPIO_ACTIVE_HIGH>` | 引用第 3 号引脚,极性标志见 `include/dt-bindings/gpio/gpio.h` |

属性名必须是 `<con_id>-gpios` 形式:`gpiod_get(dev, "led", ...)` 会在设备树中查找 `led-gpios`,两者命名需严格对应。

## 驱动程序实现

### Provider gpio_chip 字段

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
| `can_sleep` | `bool` | 读写是否可能睡眠(挂在 I2C/SPI 总线后的扩展芯片为 `true`) |
| `of_node` | `struct device_node *` | 对应的设备树节点 |

Consumer 拿到的 `struct gpio_desc *` 是不透明指针,由 gpiolib 内部维护,消费者不应也无法直接访问其内部字段。

```text
simple_gpio_probe(pdev)
    │
    ├─ devm_kzalloc()                          → 分配私有结构体 sg
    ├─ platform_get_resource() + devm_ioremap_resource() → 映射寄存器到 sg->base
    ├─ spin_lock_init()
    ├─ 填充 sg->chip 各字段(direction_input/output、get、set ...)
    ├─ platform_set_drvdata(pdev, sg)
    └─ devm_gpiochip_add_data(dev, &sg->chip, sg)  → 注册进 gpiolib
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
`direction_output` 中先写数据寄存器、再写方向寄存器,避免切换瞬间输出上一次遗留的电平。`devm_gpiochip_add_data()` 是资源管理版本,不需要额外实现 `remove()`。
:::

### Consumer 私有结构体与初始化

```text
simple_led_probe(pdev)
    │
    ├─ devm_kzalloc()                                    → 分配私有结构体 led
    ├─ devm_gpiod_get(dev, "led", GPIOD_OUT_LOW)
    │       ├─ 按 "led-gpios" 找到对应 gpio_desc
    │       └─ 自动调用一次 gpiod_direction_output() → provider 的 chip->direction_output()
    ├─ gpiod_set_value_cansleep(led->gpiod, 1)           → provider 的 chip->set(),LED 点亮
    └─ platform_set_drvdata(pdev, led)
```

<details>
<summary>Show code</summary>

```c
// drivers/misc/simple-led.c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/gpio/consumer.h>
#include <linux/of.h>
#include <linux/slab.h>

struct simple_led {
    struct gpio_desc *gpiod;
};

static const struct of_device_id simple_led_of_match[] = {
    { .compatible = "example,simple-led" },
    { }
};
MODULE_DEVICE_TABLE(of, simple_led_of_match);

static int simple_led_probe(struct platform_device *pdev)
{
    struct simple_led *led;

    led = devm_kzalloc(&pdev->dev, sizeof(*led), GFP_KERNEL);
    if (!led)
        return -ENOMEM;

    /* 对应设备树属性 "led-gpios",初始状态设为输出低电平 */
    led->gpiod = devm_gpiod_get(&pdev->dev, "led", GPIOD_OUT_LOW);
    if (IS_ERR(led->gpiod))
        return PTR_ERR(led->gpiod);

    gpiod_set_value_cansleep(led->gpiod, 1);

    platform_set_drvdata(pdev, led);
    return 0;
}

static int simple_led_remove(struct platform_device *pdev)
{
    struct simple_led *led = platform_get_drvdata(pdev);

    gpiod_set_value_cansleep(led->gpiod, 0);
    return 0;
}

static struct platform_driver simple_led_driver = {
    .driver = {
        .name = "simple-led",
        .of_match_table = simple_led_of_match,
    },
    .probe = simple_led_probe,
    .remove = simple_led_remove,
};
module_platform_driver(simple_led_driver);

MODULE_LICENSE("GPL v2");
MODULE_DESCRIPTION("Minimal example GPIO consumer driver");
```

</details>

:::tip[`GPIOD_OUT_LOW` 与极性标志]
`devm_gpiod_get(dev, "led", GPIOD_OUT_LOW)` 一步做了三件事:根据 `led-gpios` 找到 `gpio_desc`、设置为输出、初始电平设为低。`GPIO_ACTIVE_HIGH`/`GPIO_ACTIVE_LOW` 由 gpiolib 在描述符层面自动处理极性反转:无论硬件上 LED 是高电平还是低电平点亮,`gpiod_set_value(gpiod, 1)` 始终表示"逻辑点亮"。
:::

## 验证与控制

```makefile
obj-m += gpio-simple-demo.o
obj-m += simple-led.o
```

```bash
make -C /path/to/linux-4.9 M=$(pwd) modules
insmod gpio-simple-demo.ko
insmod simple-led.ko
dmesg | tail
```

```bash
mount -t debugfs none /sys/kernel/debug
cat /sys/kernel/debug/gpio
```

输出示例:

```
gpiochip0: GPIOs 0-7, simple-gpio:
 gpio-3   (led0               ) out hi
```

该输出显示 3 号引脚已被名为 `led0` 的消费者请求,当前方向为输出、电平为高,是确认"引脚是否被正确请求、方向和电平是否符合预期"最直接的手段。

:::caution[优先使用 `devm_*` 系列函数]
`devm_gpiochip_add_data()`、`devm_gpiod_get()` 会在设备卸载或 probe 失败时自动释放对应资源。手写的 `remove()` 中若遗漏 `gpiochip_remove()` 或 `gpiod_put()`,会导致资源未释放。
:::

:::caution[设备树属性名与 `con_id` 必须对应]
`gpiod_get(dev, "led", ...)` 只查找 `led-gpios`,命名不一致会导致返回 `-ENOENT`。
:::

:::caution[`can_sleep` 与 `_cansleep` 函数需要匹配]
总线型扩展芯片的读写操作可能睡眠,必须使用带 `_cansleep` 后缀的接口,不能在中断上下文或自旋锁保护区间内调用。
:::

:::caution[`gpio_chip.base` 优先使用 `-1`]
固定编号在多控制器系统中容易与其他驱动的编号段冲突,交由内核自动分配可以避免这个问题。
:::
