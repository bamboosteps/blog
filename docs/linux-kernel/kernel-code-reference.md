---
sidebar_position: 10
title: 常用内核写法速查
description: LED / I2C / SPI 各篇驱动代码里反复出现的内核写法,按内存操作、错误处理、驱动注册、并发控制、数据传递分类速查。
---

## 内存与结构体操作

### container_of {/* #container-of */}

驱动回调(`direction_output`、`read`)拿到的只是结构体里某个成员的指针,不是整个结构体。以 GPIO provider 驱动为例:

```c
struct simple_gpio {
    struct gpio_chip chip;   /* 偏移量 0,第一个成员 */
    void __iomem *base;
    spinlock_t lock;
};
```

`container_of` 从成员指针反推整个结构体地址,定义在 `include/linux/kernel.h`:

```c
#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))
```

`offsetof(type, member)` 是 `member` 相对 `type` 起始地址的字节偏移量,编译期即可算出:

```c
#define offsetof(TYPE, MEMBER) ((size_t)&((TYPE *)0)->MEMBER)
```

`sg = container_of(gc, struct simple_gpio, chip)` 用 `gc` 的地址减去 `chip` 在 `simple_gpio` 里的偏移量,得到 `simple_gpio` 结构体的起始地址:

```text
┌─────────────────────────┐  ◄── offset 0
│ struct gpio_chip chip   │  
├─────────────────────────┤  ◄── gc  
│ void __iomem *base      │
├─────────────────────────┤  ◄── offset N
│ spinlock_t lock         │
└─────────────────────────┘  ◄── sg(= container_of(gc, ..., chip))
```

`chip` 是第一个成员,偏移量为 0,`gc` 与 `sg` 数值相等;若目标成员排在后面(例如 I2C/SPI 驱动里的 `miscdev` 是第二个成员),偏移量非 0,减法才真正生效。

### devm_* 系列函数 {/* #devm */}

`devm_` 前缀绑定设备生命周期:分配的内存、映射的寄存器、请求到的 GPIO,在设备移除或 `probe()` 失败时由内核自动释放,无需手写 `kfree()`/`iounmap()`/`gpiod_put()`。

| 函数 | 作用 |
|---|---|
| `devm_kzalloc(dev, size, GFP_KERNEL)` | 分配并清零内存,替代 `kzalloc()` + 手动 `kfree()` |
| `devm_ioremap_resource(dev, res)` | 把 `platform_get_resource()` 取到的物理地址映射成可 `readl()`/`writel()` 的虚拟地址 |
| `devm_gpiochip_add_data(dev, chip, data)` | `gpiochip_add_data()` 的资源管理版本,注册一个 GPIO 控制器 |
| `devm_gpiod_get(dev, con_id, flags)` | 一步完成请求 GPIO、设置方向、设置初始电平 |

## 错误处理

### IS_ERR / PTR_ERR {/* #is-err-ptr-err */}

内核统一的错误编码方式:部分函数失败时不返回 `NULL`,而是返回数值等于某个负错误码(如 `-ENOMEM`)的指针。`IS_ERR(ptr)` 判断指针是否为这种错误编码,`PTR_ERR(ptr)` 取出错误码,通常直接 `return PTR_ERR(ptr)` 向上传递。`devm_ioremap_resource()`、`devm_gpiod_get()` 等函数均用此方式报错。

## 驱动注册与匹配

### probe() 触发机制 {/* #probe */}

`probe()` 由内核的设备-驱动匹配机制触发,不是被直接调用:注册驱动、总线核心比对设备与驱动的匹配表、匹配上则调用 `.probe`。三条总线的差异:

| 总线 | 设备对象 | 注册触发 | 匹配顺序 | `.probe` 签名 |
|---|---|---|---|---|
| platform | `platform_device`(设备树节点转换而来) | `module_platform_driver()` → `platform_driver_register()` | 仅比较 `of_match_table` 的 `compatible` | `.probe(pdev)` |
| I2C | `i2c_client`(含从设备地址) | `module_i2c_driver()` → `i2c_add_driver()` | 先 `of_match_table`,匹配不上再用 `id_table` | `.probe(client, id)` |
| SPI | `spi_device`(含片选号) | `module_spi_driver()` → `spi_register_driver()` | 先 `of_match_table`,匹配不上再用 `id_table` | `.probe(spi)` |

```text
insmod xxx.ko
    │
    ▼
module_xxx_driver() → xxx_register_driver()
    │
    ▼
总线核心扫描未绑定驱动的设备
    │
    ▼
比较设备的 compatible/名字 与驱动的匹配表
    │
    ▼
匹配成功 → 调用 .probe(device)
```

`compatible`/名字与设备树一致后,`probe()` 的调用时机和参数完全由内核处理,驱动作者不需要干预。

### MODULE_DEVICE_TABLE {/* #module-device-table */}

把设备匹配表(`of_device_id`/`i2c_device_id`/`spi_device_id` 数组)写进 `.ko` 文件的元信息,使 `udev`/`mdev` 能在启动时根据 `compatible`(或总线枚举到的设备名)自动加载对应模块,无需手动 `insmod`。

### 存取私有数据 {/* #set-drvdata */}

`probe()` 与 `remove()` 是两次独立调用,`probe()` 分配的结构体需要一个地方存指针供 `remove()` 取回。三条总线各提供一对存取函数:

| 总线 | 存 | 取 |
|---|---|---|
| platform | `platform_set_drvdata(pdev, priv)` | `platform_get_drvdata(pdev)` |
| I2C | `i2c_set_clientdata(client, priv)` | `i2c_get_clientdata(client)` |
| SPI | `spi_set_drvdata(spi, priv)` | `spi_get_drvdata(spi)` |

`probe()` 里调用"存",对应总线触发 `remove()` 时调用"取"。

## 并发控制

### spin_lock_irqsave / spin_unlock_irqrestore {/* #spin-lock */}

自旋锁保护"读-改-写寄存器"操作不被并发打断。`irqsave`/`irqrestore` 后缀表示加锁前保存当前中断使能状态、解锁后恢复,使这段代码即便在中断关闭的场景下执行也不会死锁或永久关闭中断。

:::caution[并发覆盖风险]
两个 CPU 同时调用 `simple_gpio_set()` 操作不同的位,若不加锁,后写入的一次可能覆盖前一次的结果。
:::

## 用户态数据传递

### struct file_operations 与 .read 回调 {/* #file-operations */}

内核用这个结构体把用户态的系统调用和驱动里的具体函数关联起来:

```text
用户态 read(fd, buf, len)
        │
        ▼
   VFS 层查该设备节点的 file_operations
        │
        ▼
   调用填在 .read 里的驱动函数
```

:::tip[`*ppos` 的 EOF 处理]
`*ppos` 是文件当前读写位置;读到末尾后再次 `read()` 应返回 `0`(EOF),避免用户重复读到同一份数据。
:::

### copy_to_user / copy_from_user {/* #copy-to-user */}

内核态与用户态地址空间不同,驱动不能直接解引用用户态传入的指针:

```text
│内核缓冲区 kernel_buf         用户缓冲区 user_ptr│
│── copy_to_user(user_ptr, kernel_buf, len)   ──►│
│◄── copy_from_user(kernel_buf, user_ptr, len) ──│
```

:::danger[直接解引用用户指针的风险]
内核态代码直接读写用户态指针可能导致内核崩溃或绕过权限检查;必须通过 `copy_to_user`/`copy_from_user` 传递数据。
:::

### MISC_DYNAMIC_MINOR / misc_register {/* #misc-device */}

misc 设备是内核提供的字符设备封装:无需申请设备号、创建 `class`、维护 `cdev`,只需填好 `struct miscdevice`(名字、`file_operations`)并调用 `misc_register()`。`MISC_DYNAMIC_MINOR` 让内核自动分配次设备号,注册成功后 `/dev/xxx` 节点自动出现;卸载时对称调用 `misc_deregister()`。
