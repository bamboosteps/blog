---
sidebar_position: 2
title: sysfs 接口
description: 编写一个最小的 GPIO 控制器驱动(provider),通过 sysfs 接口验证驱动、点亮 LED,基于 Linux 4.9。
---

## 核心概念

sysfs 接口(`/sys/class/gpio`)不是一个独立的驱动,而是 gpiolib 为每一个注册成功的 `gpio_chip` 自动提供的功能,由 `CONFIG_GPIO_SYSFS` 控制:只要驱动调用 `gpiochip_add_data()` 完成注册,sysfs 节点就会自动出现,不需要控制器驱动的作者额外写代码。写 `value` 文件 → `value_store()` → `gpiod_set_value_cansleep()` → `chip->set()`,sysfs 只是一层文件系统包装,真正改变引脚电平的仍然是驱动自己实现的 `set` 回调。

:::tip 前置条件
编译前在 `menuconfig` 中开启 `CONFIG_GPIOLIB`、`CONFIG_OF_GPIO`、`CONFIG_GPIO_SYSFS`,并准备 Linux 4.9 源码树和支持设备树的开发板。
:::

## 编写驱动

以下驱动针对一个假想的内存映射 GPIO 控制器:占用 8 字节寄存器空间,映射在物理地址 `0x10000000`,偏移 `0x00` 为数据寄存器、`0x04` 为方向寄存器(对应位为 1 表示输出),共管理 8 根引脚。

### 分析调用链

以 Linux 4.9 的实现(`drivers/gpio/gpiolib-sysfs.c`)为例,几个关键文件对应的处理函数是:

| sysfs 文件 | 处理函数 | 内部调用 |
|---|---|---|
| `export`(写) | `export_store()` | `gpio_to_desc()` 校验编号,再 `gpiod_request()` 占用该引脚 |
| `gpioN/direction`(写) | `direction_store()` | `gpiod_direction_output_raw()` 或 `gpiod_direction_input()` |
| `gpioN/value`(读/写) | `value_show()` / `value_store()` | `gpiod_get_value_cansleep()` / `gpiod_set_value_cansleep()` |

这几个 `gpiod_*` 函数定义在 `drivers/gpio/gpiolib.c` 中,最终都会落到 `gpio_chip` 自己实现的回调上:

```c
// drivers/gpio/gpiolib.c(节选,函数名与参数为实际内核源码)
err = chip->direction_input(chip, offset);
err = chip->direction_output(chip, offset, value);
chip->set(chip, offset, value);
value = chip->get(chip, offset);
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
        gpiolib 自动创建 /sys/class/gpio/gpioN 节点
                    │
        用户态 echo 写 direction / value 文件
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

驱动加载成功后,用 `cat /sys/kernel/debug/gpio` 确认分配到的全局编号(以下假设是 `17`,对应设备树中第 3 号引脚外接的 LED)。sysfs 本质上就是普通文件,命令行的 `echo`/`cat` 和代码里的 `open()`/`write()` 操作的是同一组文件,只是调用方式不同。

### 运行命令行工具

<details>
<summary>Show code</summary>

```bash
# 导出编号为 17 的 GPIO,导出后会出现 /sys/class/gpio/gpio17 目录
echo 17 > /sys/class/gpio/export

# 设置方向为输出
echo out > /sys/class/gpio/gpio17/direction

# 输出高电平,点亮 LED
echo 1 > /sys/class/gpio/gpio17/value

# 输出低电平,熄灭 LED
echo 0 > /sys/class/gpio/gpio17/value

# 使用完毕后取消导出
echo 17 > /sys/class/gpio/unexport
```

</details>

### 编写用户态程序

`echo x > file` 在 shell 里等价于 `open(file, O_WRONLY)` 后 `write()` 写入字符串 `x`,用 C 代码实现同样的流程:

<details>
<summary>Show code</summary>

```c
// led_sysfs.c
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

static int write_sysfs(const char *path, const char *value)
{
    int fd = open(path, O_WRONLY);
    if (fd < 0) {
        perror(path);
        return -1;
    }

    ssize_t len = strlen(value);
    if (write(fd, value, len) != len) {
        perror("write");
        close(fd);
        return -1;
    }

    close(fd);
    return 0;
}

int main(void)
{
    const char *gpio = "17";
    char path[64];

    /* 导出 GPIO,导出后才会出现 gpio17/ 目录 */
    write_sysfs("/sys/class/gpio/export", gpio);

    /* 设置方向为输出 */
    snprintf(path, sizeof(path), "/sys/class/gpio/gpio%s/direction", gpio);
    write_sysfs(path, "out");

    /* 点亮 LED */
    snprintf(path, sizeof(path), "/sys/class/gpio/gpio%s/value", gpio);
    write_sysfs(path, "1");
    sleep(1);

    /* 熄灭 LED */
    write_sysfs(path, "0");

    /* 取消导出 */
    write_sysfs("/sys/class/gpio/unexport", gpio);

    return 0;
}
```

</details>

:::caution export/unexport 不能省略
`export`/`unexport` 写入的是 `/sys/class/gpio` 顶层的两个文件,不是每个 GPIO 独立一份。省略这两次调用会导致后续对 `gpio17/direction`、`gpio17/value` 的 `open()` 因为目录不存在而返回 `-ENOENT`。
:::

:::caution sysfs 接口已被标记为 deprecated
原因包括:编号由内核动态分配、不同板卡间不保证一致;`export`/`unexport` 缺乏原子性;无法在一次系统调用中操作多个引脚。4.9 内核仍保留该接口,但不建议在新项目中使用。
:::
