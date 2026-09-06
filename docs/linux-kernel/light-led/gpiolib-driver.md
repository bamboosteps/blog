---
sidebar_position: 4
title: gpiolib 框架
description: 编写一个最小的 GPIO 控制器驱动(provider)和消费者驱动(consumer),通过 gpiolib 完成同一个点亮 LED 的目标,基于 Linux 4.9。
---

## 核心概念

gpiolib 分两种角色:**Provider(提供者)** 实现 `struct gpio_chip` 并调用 `gpiochip_add_data()` 完成注册,声明"这里有 N 根 GPIO,读写方法是这些函数",典型例子是 SoC 内置 GPIO 控制器驱动、I2C/SPI 接口的 GPIO 扩展芯片驱动;**Consumer(使用者)** 调用 `gpiod_get()` 等函数,根据设备树属性名拿到一个 `struct gpio_desc *` 描述符,再用 `gpiod_set_value()`/`gpiod_get_value()` 等统一接口操作,不关心这个 GPIO 具体来自哪个控制器。Consumer 侧接口定义在 `include/linux/gpio/consumer.h`,provider 侧接口定义在 `include/linux/gpio/driver.h`。

Linux 3.13 引入了 `gpiod_*` 描述符式 consumer API,取代早期基于全局整数编号的 `gpio_request()`/`gpio_set_value()` 系列接口(定义于 `include/linux/gpio.h`);到 4.9 内核时描述符式 API 已是编写新代码的推荐方式,本节只使用描述符式 API。

:::tip 前置条件
阅读本节需要具备 `platform_driver` 框架与设备树基础语法的背景知识。编译前需在 `menuconfig` 中开启 `CONFIG_GPIOLIB`、`CONFIG_OF_GPIO`、`CONFIG_GPIO_CDEV`(供[字符设备接口](./chardev.md)一节使用)、`CONFIG_DEBUG_FS`(供调试使用)。
:::

## 填写 gpio_chip 字段

以下字段是实现一个最小 provider 驱动必须填写的部分(定义于 `include/linux/gpio/driver.h`):

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
| `can_sleep` | `bool` | 读写操作是否可能睡眠(挂在 I2C/SPI 总线后的扩展芯片为 `true`) |
| `of_node` | `struct device_node *` | 对应的设备树节点 |

Consumer 侧拿到的 `struct gpio_desc *` 是一个不透明指针,由 gpiolib 内部维护,消费者不应也无法直接访问其内部字段。

## 编写设备树节点

目标 GPIO 控制器是一个内存映射寄存器控制器,占用 8 字节寄存器空间,映射在物理地址 `0x10000000`:

```dts
simple_gpio: gpio@10000000 {
    compatible = "example,simple-gpio";
    reg = <0x10000000 0x8>;
    gpio-controller;
    #gpio-cells = <2>;
};
```

`gpio-controller` 是一个空属性,标记该节点是一个 GPIO 控制器。`#gpio-cells` 声明消费者引用该控制器时需要提供几个参数,惯例为 `2`:第一个是控制器内部的引脚偏移,第二个是极性标志(`GPIO_ACTIVE_HIGH`/`GPIO_ACTIVE_LOW`,定义于 `include/dt-bindings/gpio/gpio.h`)。

LED 消费者节点引用第 3 号引脚:

```dts
led0: led {
    compatible = "example,simple-led";
    led-gpios = <&simple_gpio 3 GPIO_ACTIVE_HIGH>;
};
```

属性名必须是 `<con_id>-gpios` 的形式,`gpiod_get(dev, "led", ...)` 会自动在设备树中查找 `led-gpios` 属性,两者的命名需要严格对应。

## 编写 Provider 驱动

编写以下驱动代码,对应上一节的假想控制器:寄存器偏移 `0x00` 为数据寄存器、`0x04` 为方向寄存器(对应位为 1 表示输出)。

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
        gpiolib 内部记录这个 gpio_chip,后续 consumer(下一节)
        通过 gpiod_* 调用时,最终会落到 sg->chip 的回调上
```

`sg->chip.base = -1` 表示编号由内核自动分配,避免和系统里其他 GPIO 控制器的编号段冲突,`devm_gpiochip_add_data()` 是资源管理版本,不需要额外实现 `remove()`。

### 对照相关写法

流程图里几个关键函数/宏的原理统一放在了[《常用内核写法速查》](../kernel-code-reference.md)里,这里只列出对应链接:

- [`container_of`](../kernel-code-reference.md#container-of)
- [`probe()` 触发机制(platform 总线)](../kernel-code-reference.md#probe)
- [`devm_kzalloc` / `devm_ioremap_resource`](../kernel-code-reference.md#devm)
- [`IS_ERR` / `PTR_ERR`](../kernel-code-reference.md#is-err-ptr-err)
- [`spin_lock_irqsave` / `spin_unlock_irqrestore`](../kernel-code-reference.md#spin-lock)
- [`MODULE_DEVICE_TABLE`](../kernel-code-reference.md#module-device-table)
- [`platform_set_drvdata` / `platform_get_drvdata`](../kernel-code-reference.md#set-drvdata)

## 编写 Consumer 驱动

### 实现驱动代码

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

### 梳理执行流程

```
insmod simple-led.ko
        │
        ▼
module_platform_driver() → platform_driver_register()
        │
        ▼
platform 总线按 compatible = "example,simple-led" 匹配设备树节点
        │
        ▼
simple_led_probe(pdev) 被调用
        │
        ├─ devm_kzalloc()                              分配私有结构体 led
        ├─ devm_gpiod_get(dev, "led", GPIOD_OUT_LOW)
        │        │
        │        ├─ 根据设备树 "led-gpios" 找到对应的 gpio_desc
        │        └─ 自动调用一次 gpiod_direction_output()
        │                 │
        │                 ▼
        │        最终调用到 provider 驱动的 chip->direction_output()
        ├─ gpiod_set_value_cansleep(led->gpiod, 1)
        │                 │
        │                 ▼
        │        最终调用到 provider 驱动的 chip->set()   → LED 点亮
        └─ platform_set_drvdata(pdev, led)
```

`simple_led_probe()` 的调用机制和前面 Provider 驱动的 `simple_gpio_probe()` 完全一样,都是 platform 总线按 `compatible` 匹配后自动触发,详见[《常用内核写法速查》](../kernel-code-reference.md#probe)。

`devm_gpiod_get(dev, "led", GPIOD_OUT_LOW)` 一步做了三件事:① 根据设备树属性 `led-gpios` 找到对应的 `gpio_desc`;② 把它设置为输出;③ 初始电平设为低。`GPIOD_OUT_LOW`/`GPIOD_OUT_HIGH`/`GPIOD_IN` 是 `gpiod_get()` 系列函数专门定义的枚举值,免去再手动调一次 `gpiod_direction_output()`。

`GPIO_ACTIVE_HIGH`/`GPIO_ACTIVE_LOW` 标志由 gpiolib 在描述符层面自动处理极性反转:无论硬件上 LED 是高电平点亮还是低电平点亮,消费者代码里 `gpiod_set_value(gpiod, 1)` 始终表示"逻辑点亮",不需要驱动自己判断硬件极性。

### 对照相关写法

- [`container_of`](../kernel-code-reference.md#container-of)(这份驱动本身没直接用到,但 `gpio_desc` 内部原理相同)
- [`devm_kzalloc`](../kernel-code-reference.md#devm)
- [`IS_ERR` / `PTR_ERR`](../kernel-code-reference.md#is-err-ptr-err)
- [`platform_set_drvdata` / `platform_get_drvdata`](../kernel-code-reference.md#set-drvdata)

## 编译并加载驱动

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

## 查看 debugfs 状态

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

:::caution 优先使用 devm_* 系列函数
`devm_gpiochip_add_data()`、`devm_gpiod_get()` 会在设备卸载或 probe 失败时自动释放对应资源。手写的 `remove()` 中若遗漏 `gpiochip_remove()` 或 `gpiod_put()`,会导致资源未释放。
:::

:::caution 设备树属性名与 con_id 必须对应
`gpiod_get(dev, "led", ...)` 只查找 `led-gpios`,命名不一致会导致返回 `-ENOENT`。
:::

:::caution can_sleep 与 _cansleep 函数需要匹配
总线型扩展芯片的读写操作可能睡眠,必须使用带 `_cansleep` 后缀的接口。不能在中断上下文或自旋锁保护区间内调用这类接口。
:::

:::caution gpio_chip.base 优先使用 -1
固定编号在多控制器系统中容易与其他驱动的编号段冲突,交由内核自动分配可以避免这个问题。
:::

:::caution direction_output 应先设置电平、再切换方向
顺序颠倒会在切换瞬间输出上一次遗留的电平,对部分外设(例如某些使能引脚)可能造成影响。
:::
