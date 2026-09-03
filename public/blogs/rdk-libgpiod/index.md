# GpioMgr —— RDK X5 40pin GPIO 管理模块（libgpiod 方案）

> **适用人群**：RDK X5 开发者、嵌入式 Linux GPIO 调试者  
> **阅读时长**：约 8 分钟  
> **关键词**：GPIO、RDK X5、libgpiod、C++、wiringPi 替代

---

## 0. 版权与许可证

**Copyright © 2026 wangmr. All rights reserved.**

本文内容及示例代码采用 **MIT License** 开源，可自由使用、修改、分发及商用，但需保留以上版权声明及作者署名。

> **第三方依赖声明**：本模块依赖 `libgpiod`，其许可证为 **LGPL-2.1**，使用时请遵守其条款。

---

## 1. 你遇到过这些问题吗？

最近在 RDK X5 上调试 SPI 通讯，想用 `wiringPi` 控制 GPIO，折腾了半天才发现 RDK X5 并没有官方适配的 wiringPi 版本。

为了避免以后再踩同样的坑，我基于官方推荐的 `libgpiod v1.6` 封装了一套 **GpioMgr** 管理模块，专门用于 RDK X5 的 40pin GPIO 控制。

**这套方案能帮你解决：**
- ✅ 引脚编号混乱 —— 直接用物理 40pin 号（1~40）
- ✅ 多芯片管理复杂 —— 自动打开所有需要的 gpiochip
- ✅ pinmux 冲突 —— 明确标注哪些脚被外设占用
- ✅ 上下拉不生效 —— 接口预留，实测驱动不支持
- ✅ 代码移植性差 —— 换板卡只需改一张映射表

---

## 2. 先跑起来看看效果

### 安装依赖（Ubuntu/Debian）

```bash
sudo apt update
sudo apt install libgpiod-dev gpiod
```

### 一个完整的示例

```cpp
#include "GpioMgr.h"
#include <iostream>
#include <unistd.h>

int main() {
    GpioMgr gpio;

    // 1. 初始化
    if (!gpio.Init()) {
        std::cerr << "Init failed\n";
        return -1;
    }

    // 2. 配置 pin 7 为输出，初始低电平
    if (!gpio.SetOutput(7, 0)) {
        std::cerr << "SetOutput pin 7 failed\n";
        return -1;
    }

    // 3. 配置 pin 11 为输入（默认无偏置）
    if (!gpio.SetInput(11)) {
        std::cerr << "SetInput pin 11 failed\n";
        return -1;
    }

    // 4. 循环读写
    for (int i = 0; i < 10; ++i) {
        gpio.Write(7, i % 2);
        int val = gpio.Read(11);
        std::cout << "Pin 11 reads: " << val << std::endl;
        sleep(1);
    }

    // 5. 释放（析构时自动 DeInit，也可手动调用）
    gpio.DeInit();
    return 0;
}
```

编译运行：

```bash
g++ -std=c++11 -o gpio_test main.cpp -lgpiod
sudo ./gpio_test
```

**就是这么简单！** 下面我们深入看看它是怎么设计的。

---

## 3. 设计思路：一张表搞定所有引脚映射

RDK X5 的 40pin 分布在多个 gpiochip 上（gpiochip3、gpiochip4、gpiochip5），每个芯片的 line offset 都不一样。

GpioMgr 的核心思想是：**把引脚映射关系集中在一张表里**，更换板卡或修改引脚分配时只需更新这张表，其余代码无需改动。

```cpp
struct GpioPinDef
{
    int pin;                // 40pin 物理引脚号（1~40）
    const char* chip;       // /dev/ 下的 gpiochip 名
    int offset;             // 芯片内 line 偏移
    const char* name;       // 引脚功能名（复用功能）
};
```

> **全局编号 = 芯片 base + 偏移**，与板卡 `gpio_pin_data.py` 的 RDK_X5_PIN 表一致。

---

## 4. 核心 API 设计

### 4.1 初始化

```cpp
bool Init();
```
自动打开映射表中涉及的所有 gpiochip（去重），最多支持 4 个芯片。使用任何引脚前必须先调用。

### 4.2 配置引脚方向

```cpp
// 配置为输入，可选上/下拉偏置（当前驱动不生效）
bool SetInput(int pin, Pull pull = PULL_NONE);

// 配置为输出，可带初始电平（默认低）
bool SetOutput(int pin, int value = 0);
```

### 4.3 读写操作

```cpp
int  Read(int pin);   // 失败返回 -1
bool Write(int pin, int value);
```

### 4.4 资源释放

```cpp
void Release(int pin);   // 释放单个引脚
void DeInit();           // 释放全部（析构时自动调用）
```

---

## 5. ⚠️ 最容易踩的坑：pinmux 冲突

**这是 90% 的问题根源。**

RDK X5 的默认系统镜像中，很多 40pin 脚被 pinmux 复用为外设（I2C、UART、SPI、PWM）。在映射表中，我用 `*` 标记了这些默认被占用的引脚：

```cpp
inline constexpr GpioPinDef RDK_40PIN_MAP[] = {
    // pin  chip         offset  功能名                       备注
    {  3, "gpiochip4",  11, "I2C5_SDA/UART3_TXD" },          // *
    {  5, "gpiochip4",  10, "I2C5_SCL/UART3_RXD" },          // *
    {  7, "gpiochip3",   9, "GPCLK0/I2S1_MCLK" },            // 可用
    {  8, "gpiochip4",   4, "UART1_TXD" },                   // *
    { 10, "gpiochip4",   5, "UART1_RXD" },                   // *
    { 11, "gpiochip4",   1, "GPIO17/UART7_TXD" },            // 可用
    // ... 完整表见源码
};
```

**关键问题**：这些引脚的 GPIO 请求**不会报 EBUSY 错误**，但 pinmux 仍归外设所有，物理引脚**不响应 GPIO 输出**。也就是说，`SetOutput` 返回成功，但引脚电平不变。

### 解决方案

在设备树中禁用对应外设节点，将 `status = "disabled"`，然后重启系统：

```bash
# 查看当前引脚状态，确认是否被占用
gpioinfo | grep -E "gpiochip[3-5]"
```

需要禁用的外设节点示例：
- `spi1` —— 影响 pin 19, 21, 23, 24, 26
- `uart1` —— 影响 pin 8, 10
- `i2c5` —— 影响 pin 3, 5
- `i2c0` —— 影响 pin 27, 28
- `pwm` —— 影响 pin 32, 33

---

## 6. 完整头文件源码（GpioMgr.h）

以下是完整的头文件，可直接复制到项目中使用：

```cpp
/* @file
---------------------------------------------------------------------------------
Module          : GpioMgr
File            : GpioMgr.h
Description     : 40pin GPIO 管理模块（基于 libgpiod v1.6）
Author          : wangmr
Rev             : 1.0
---------------------------------------------------------------------------------
comment         : 40pin 引脚的偏置（gpiochip + line 偏移）集中在本文件
                 的 RDK_40PIN_MAP 分配表中，更换板卡或修改引脚分配时
                 只需更新该表，其余代码无需改动。
                 Init 打开表中涉及的所有 gpiochip；SetInput/SetOutput
                 配置引脚方向；Read/Write 读写电平；Release/DeInit
                 释放资源。
                 
                 * 标记的引脚：默认系统镜像的 pinmux 已将该脚复用为对应外设。
                 实测这些脚的 GPIO 请求不会报 EBUSY，但 pinmux 仍归外设所有，
                 物理引脚不响应 GPIO 输出——用作 GPIO 前需先在设备树中
                 释放该外设（status=disabled）并重启。
---------------------------------------------------------------------------------
*/
#ifndef __GPIOMGR_H
#define __GPIOMGR_H

//---------------------------- Include files -----------------------------//
#include <errno.h>
#include <gpiod.h>
#include <stdio.h>
#include <string.h>

//---------------------------- 40pin 分配表 ------------------------------//
struct GpioPinDef
{
    int pin;                // 40pin 物理引脚号（1~40）
    const char* chip;       // /dev/ 下的 gpiochip 名
    int offset;             // 芯片内 line 偏移
    const char* name;       // 引脚功能名（复用功能）
};

inline constexpr GpioPinDef RDK_40PIN_MAP[] = {
    // pin  chip         offset  功能名
    {  3, "gpiochip4",  11, "I2C5_SDA/UART3_TXD" },          // *
    {  5, "gpiochip4",  10, "I2C5_SCL/UART3_RXD" },          // *
    {  7, "gpiochip3",   9, "GPCLK0/I2S1_MCLK" },            // 可用
    {  8, "gpiochip4",   4, "UART1_TXD" },                   // *
    { 10, "gpiochip4",   5, "UART1_RXD" },                   // *
    { 11, "gpiochip4",   1, "GPIO17/UART7_TXD" },            // 可用
    { 12, "gpiochip3",  10, "PCM_CLK/I2S1_BCLK" },           // 可用
    { 13, "gpiochip4",   0, "GPIO27/UART7_RXD" },            // 可用
    { 15, "gpiochip4",   9, "GPIO22/UART2_TXD" },            // 可用
    { 16, "gpiochip4",   3, "GPIO23/UART6_TXD" },            // 可用
    { 18, "gpiochip4",  23, "GPIO24/SPI2_MOSI" },            // 可用
    { 19, "gpiochip4",  19, "SPI1_MOSI/JTG_TDO" },           // *
    { 21, "gpiochip4",  18, "SPI1_MISO/JTG_TDI" },           // *
    { 22, "gpiochip4",   8, "GPIO25/UART2_RXD" },            // 可用
    { 23, "gpiochip4",  16, "SPI1_SCLK/JTG_TCK" },           // *
    { 24, "gpiochip4",  15, "SPI1_CSN1/JTG_TMS" },           // *
    { 26, "gpiochip4",  17, "SPI1_CSN0/JTG_TRSTN" },         // *
    { 27, "gpiochip5",   8, "ID_SD/I2C0_SDA" },              // *
    { 28, "gpiochip5",   7, "ID_SC/I2C0_SCL" },              // *
    { 29, "gpiochip4",  20, "GPIO5/SPI2_SCLK" },             // 可用
    { 31, "gpiochip4",  21, "GPIO6/I2C1_SDA" },              // 可用
    { 32, "gpiochip5",   9, "PWM6" },                        // *
    { 33, "gpiochip5",  10, "PWM7" },                        // *
    { 35, "gpiochip3",  11, "PCM_FS/I2S1_LRCK" },            // 可用
    { 36, "gpiochip4",   2, "GPIO16/UART6_RXD" },            // 可用
    { 37, "gpiochip4",  22, "GPIO26/SPI2_MISO" },            // 可用
    { 38, "gpiochip3",  12, "PCM_DIN/I2S1_SDIN" },           // 可用
    { 40, "gpiochip3",  13, "PCM_DOUT/I2S1_SDOUT" },         // 可用
};

//-------------------------------- Define --------------------------------//
#define GPIOMGR_PIN_MAX      40
#define GPIOMGR_CHIP_MAX     4
#define GPIOMGR_CONSUMER     "GpioMgr"

//------------------------------- Class ----------------------------------//
class GpioMgr
{
    public:
    enum Direction { INPUT = 0, OUTPUT = 1 };
    // 注：当前内核 GPIO 驱动为 gpio_stub_drv，实测上/下拉偏置不生效
    enum Pull { PULL_NONE = 0, PULL_UP, PULL_DOWN, PULL_DISABLE };

    public:
    GpioMgr() = default;
    GpioMgr(const GpioMgr&) = delete;
    GpioMgr(GpioMgr&&) = delete;
    GpioMgr& operator=(const GpioMgr&) = delete;
    GpioMgr& operator=(GpioMgr&&) = delete;
    ~GpioMgr() { DeInit(); }

    bool Init();
    bool SetInput(int pin, Pull pull = PULL_NONE);
    bool SetOutput(int pin, int value = 0);
    int  Read(int pin);
    bool Write(int pin, int value);
    void Release(int pin);
    void DeInit();

    private:
    struct ChipSlot {
        const char* name;
        struct gpiod_chip* chip;
    };

    static const GpioPinDef* findPin(int pin);
    struct gpiod_chip* findChip(const char* name);
    static int pullToFlags(Pull pull);
    bool checkLine(int pin);
    bool requestLine(int pin, Direction dir, int flags, int value);

    private:
    bool m_bInited = false;
    int m_iChipCount = 0;
    ChipSlot m_pChips[GPIOMGR_CHIP_MAX] = {};
    struct gpiod_line* m_pLines[GPIOMGR_PIN_MAX + 1] = {};
};

//---------------------------- Implementation ----------------------------//
inline bool GpioMgr::Init()
{
    if (m_bInited) return true;
    for (const auto& def : RDK_40PIN_MAP) {
        if (findChip(def.chip) != nullptr) continue;
        struct gpiod_chip* chip = gpiod_chip_open_by_name(def.chip);
        if (chip == nullptr) {
            printf("[GpioMgr] open %s failed: %s\n", def.chip, strerror(errno));
            DeInit();
            return false;
        }
        if (m_iChipCount >= GPIOMGR_CHIP_MAX) {
            printf("[GpioMgr] too many gpio chips (max %d)\n", GPIOMGR_CHIP_MAX);
            gpiod_chip_close(chip);
            DeInit();
            return false;
        }
        m_pChips[m_iChipCount].name = def.chip;
        m_pChips[m_iChipCount].chip = chip;
        m_iChipCount++;
    }
    m_bInited = true;
    return true;
}

inline bool GpioMgr::SetInput(int pin, Pull pull)
{
    return requestLine(pin, INPUT, pullToFlags(pull), 0);
}

inline bool GpioMgr::SetOutput(int pin, int value)
{
    return requestLine(pin, OUTPUT, 0, value);
}

inline int GpioMgr::Read(int pin)
{
    if (!checkLine(pin)) return -1;
    int value = gpiod_line_get_value(m_pLines[pin]);
    if (value < 0)
        printf("[GpioMgr] read pin %d failed: %s\n", pin, strerror(errno));
    return value;
}

inline bool GpioMgr::Write(int pin, int value)
{
    if (!checkLine(pin)) return false;
    if (gpiod_line_set_value(m_pLines[pin], value) < 0) {
        printf("[GpioMgr] write pin %d failed: %s\n", pin, strerror(errno));
        return false;
    }
    return true;
}

inline void GpioMgr::Release(int pin)
{
    if (pin >= 1 && pin <= GPIOMGR_PIN_MAX && m_pLines[pin] != nullptr) {
        gpiod_line_release(m_pLines[pin]);
        m_pLines[pin] = nullptr;
    }
}

inline void GpioMgr::DeInit()
{
    for (auto& line : m_pLines) {
        if (line != nullptr) {
            gpiod_line_release(line);
            line = nullptr;
        }
    }
    for (int i = 0; i < m_iChipCount; i++) {
        gpiod_chip_close(m_pChips[i].chip);
        m_pChips[i] = {};
    }
    m_iChipCount = 0;
    m_bInited = false;
}

inline const GpioPinDef* GpioMgr::findPin(int pin)
{
    for (const auto& def : RDK_40PIN_MAP) {
        if (def.pin == pin) return &def;
    }
    return nullptr;
}

inline struct gpiod_chip* GpioMgr::findChip(const char* name)
{
    for (int i = 0; i < m_iChipCount; i++) {
        if (strcmp(m_pChips[i].name, name) == 0)
            return m_pChips[i].chip;
    }
    return nullptr;
}

inline int GpioMgr::pullToFlags(Pull pull)
{
    switch (pull) {
        case PULL_UP:      return GPIOD_LINE_REQUEST_FLAG_BIAS_PULL_UP;
        case PULL_DOWN:    return GPIOD_LINE_REQUEST_FLAG_BIAS_PULL_DOWN;
        case PULL_DISABLE: return GPIOD_LINE_REQUEST_FLAG_BIAS_DISABLE;
        default:           return 0;
    }
}

inline bool GpioMgr::checkLine(int pin)
{
    if (!m_bInited) {
        printf("[GpioMgr] not inited, call Init() first\n");
        return false;
    }
    if (pin < 1 || pin > GPIOMGR_PIN_MAX) {
        printf("[GpioMgr] invalid pin %d (1~%d)\n", pin, GPIOMGR_PIN_MAX);
        return false;
    }
    if (m_pLines[pin] == nullptr) {
        printf("[GpioMgr] pin %d not configured, call SetInput/SetOutput first\n", pin);
        return false;
    }
    return true;
}

inline bool GpioMgr::requestLine(int pin, Direction dir, int flags, int value)
{
    if (!m_bInited) {
        printf("[GpioMgr] not inited, call Init() first\n");
        return false;
    }
    const GpioPinDef* def = findPin(pin);
    if (def == nullptr) {
        printf("[GpioMgr] pin %d is not a controllable gpio pin\n", pin);
        return false;
    }
    struct gpiod_chip* chip = findChip(def->chip);
    if (chip == nullptr) {
        printf("[GpioMgr] chip %s not opened\n", def->chip);
        return false;
    }
    Release(pin);

    struct gpiod_line* line = gpiod_chip_get_line(chip, def->offset);
    if (line == nullptr) {
        printf("[GpioMgr] get line %s+%d failed: %s\n", def->chip, def->offset, strerror(errno));
        return false;
    }
    int ret;
    if (dir == OUTPUT)
        ret = gpiod_line_request_output_flags(line, GPIOMGR_CONSUMER, flags, value);
    else
        ret = gpiod_line_request_input_flags(line, GPIOMGR_CONSUMER, flags);
    if (ret < 0) {
        printf("[GpioMgr] request pin %d (%s, %s+%d) failed: %s\n", pin, def->name, def->chip, def->offset, strerror(errno));
        return false;
    }
    m_pLines[pin] = line;
    return true;
}

#endif  // ! __GPIOMGR_H
```

---

## 7. 从 wiringPi 迁移对照表

| wiringPi 函数 | GpioMgr 对应方法 | 说明 |
|--------------|------------------|------|
| `wiringPiSetup()` | `gpio.Init()` | 初始化 |
| `pinMode(pin, INPUT)` | `gpio.SetInput(pin)` | 输入模式 |
| `pinMode(pin, OUTPUT)` | `gpio.SetOutput(pin, val)` | 输出模式，可设初值 |
| `digitalRead(pin)` | `gpio.Read(pin)` | 读电平（返回 0/1 或 -1） |
| `digitalWrite(pin, val)` | `gpio.Write(pin, val)` | 写电平 |
| `pullUpDnControl(pin, mode)` | ❌ 不支持（驱动无效） | 使用外部电阻 |

> **物理编号差异**：wiringPi 通常使用 BCM 编号或自定义编号，而 GpioMgr 直接使用 40pin 物理编号（1~40），迁移时请对照映射表重新定义引脚号。

---

## 8. 扩展与定制

- **增加新引脚**：在 `RDK_40PIN_MAP` 数组中添加 `{pin, chip, offset, name}` 项即可。
- **更换板卡**：若新板卡的 gpiochip 名或偏移不同，更新映射表即可，业务逻辑不变。
- **更多功能**（如中断、批量读写）：可基于 `libgpiod` 的 `gpiod_line_event_wait` 等函数自行扩展。

---

## 9. 常见问题

**Q：`gpio.Init()` 失败，提示“open gpiochip3 failed: No such file or directory”？**  
A：检查 `/dev/` 下是否存在对应的 gpiochip 节点。RDK X5 默认可能只注册部分芯片，若缺少可查看内核配置或设备树。

**Q：设置输出后，引脚电平不变化？**  
A：首先确认该引脚未被外设复用（查看 `gpioinfo` 输出中的 "used" 状态）。若被外设占用，需修改设备树并重启。详见第 5 章。

**Q：`Read` 总是返回 0 或 1 但物理电压不对？**  
A：因驱动偏置失效，悬空脚读值不确定，建议外接固定电平测试。

**Q：可以同时操作多个引脚吗？**  
A：可以，每个引脚独立请求，内部会分别持有 `gpiod_line` 句柄。

---

## 10. 总结

这套 GpioMgr 已经在我的多个 RDK X5 项目中稳定运行。它最大的价值在于：

1. **把复杂的 gpiochip/line 映射关系集中在一张表里**，换板卡只需改表
2. **明确标注了 pinmux 冲突的引脚**，避免走弯路
3. **API 简洁直观**，上手成本极低

如果你也遇到 pinmux 冲突或上下拉失效的问题，欢迎留言讨论！如果你有更好的改进建议，也欢迎交流。

---

## 11. 参考资料

- [libgpiod 官方文档](https://git.kernel.org/pub/scm/libs/libgpiod/libgpiod.git/tree/README)
- RDK X5 硬件手册（GPIO 分配表）
- Horizon 平台 GPIO 驱动说明（`gpio_stub_drv`）

---

**维护者**：wangmr  
**版本**：1.0  
**最后更新**：2026-09-03

---

> 📝 **版权声明**：本文及示例代码采用 MIT License 开源，可自由使用与分发，但需保留作者署名。