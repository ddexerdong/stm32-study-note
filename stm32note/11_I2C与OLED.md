# 11 I2C 与 OLED

> 覆盖课程：[30] [31-1] [31-2] [31-3]
>
> 目标：理解 I2C 两线通信的工程本质，掌握 CubeMX 中 I2C 主机配置思路，并用 OLED 初始化、清屏、字符显示和位图显示建立“向显示控制器写命令/写数据”的框架。

---

## 本节目标

完成本节后，应能做到：

- 说清楚 I2C 为什么只用 SCL/SDA 两根线就能挂多个设备。
- 区分 7 位设备地址、读写位、寄存器地址和数据。
- 在 CubeMX 中配置 STM32F103 的 I2C 主机模式。
- 用 HAL I2C API 搭建 OLED 写命令、写数据、清屏、显示字符、显示位图的框架。
- 按“供电 -> 共地 -> 上拉 -> 地址 -> CubeMX -> HAL 返回值 -> 波形/串口 -> 业务逻辑”的顺序排查 I2C/OLED 问题。

---

## 核心概念

### 知识地图

| 名词 | 工程含义 | 常见位置 |
|------|----------|----------|
| I2C | 两线同步串行总线，适合低速板级外设通信 | `[30]` |
| SCL | Serial Clock，主机输出时钟 | I2C 引脚 |
| SDA | Serial Data，双向数据线 | I2C 引脚 |
| 开漏输出 | 设备只主动拉低，高电平依赖上拉电阻 | I2C GPIO |
| 上拉电阻 | 让 SCL/SDA 空闲时回到高电平 | I2C 总线 |
| 起始信号 | SCL 高电平时 SDA 从高到低 | I2C 时序 |
| 停止信号 | SCL 高电平时 SDA 从低到高 | I2C 时序 |
| ACK / NACK | 接收方应答/不应答 | I2C 时序 |
| 7 位地址 | I2C 从设备地址，不含最低位读写位 | OLED / 传感器 |
| 控制字节 | OLED 常用来区分后面是命令还是显示数据 | OLED 驱动 |
| 显存 / Buffer | 在 MCU 内存中先组织显示内容，再写入 OLED | `BSP/oled.c` |
| 页地址 | OLED 常见按页写显示 RAM 的寻址方式 | OLED 控制器 |

### 常用 HAL API

| API / 宏 | 作用 | 常见用法 |
|----------|------|----------|
| `HAL_I2C_Init()` | 初始化 I2C 外设 | 通常由 CubeMX 生成 |
| `HAL_I2C_Master_Transmit()` | 主机发送一段数据 | OLED 写命令/数据 |
| `HAL_I2C_Mem_Write()` | 向从设备某个寄存器/内存地址写数据 | EEPROM/传感器寄存器 |
| `HAL_I2C_Mem_Read()` | 从从设备某个寄存器/内存地址读数据 | 传感器读取 |
| `HAL_I2C_IsDeviceReady()` | 探测某个 I2C 地址是否应答 | 地址扫描/设备检测 |
| `HAL_I2C_GetState()` | 查询 I2C 状态 | 调试和错误恢复 |

---

## 本质理解

I2C 的本质是：**一个主机按时钟节奏，在两根线上选择某个从设备，再读写它内部的数据。**

```text
主机 STM32
  SCL --------------+---- OLED
  SDA --------------+---- 传感器
                    +---- EEPROM
```

每个从设备靠地址区分。STM32 先发送地址，只有地址匹配的设备回应 ACK，后面的命令或数据才由它处理。

OLED 显示的本质不是“printf 到屏幕”，而是：

```text
准备命令或显示数据
-> 通过 I2C 发给 OLED 控制器
-> OLED 控制器把数据写入内部显示 RAM
-> 屏幕按显示 RAM 刷新像素
```

PDF 的 I2C 章节能支撑 I2C 物理层、协议层、HAL I2C 初始化和内存读写的通用思路；PDF 只把 OLED 作为显示器类型介绍，没有给出课程小 OLED 模块的控制器、地址、分辨率和字库细节。

> 视频待核对：OLED 控制器型号、I2C 地址、分辨率、页寻址方式、初始化命令、字库来源和显示函数接口。

以实际开发板原理图和 CubeMX 配置为准。

---

## 工作流程 / 原理

### I2C 基本写流程

```text
START
-> 发送从设备地址 + 写方向
-> 等 ACK
-> 发送寄存器地址 / 控制字节
-> 等 ACK
-> 发送数据
-> 等 ACK
-> STOP
```

OLED 写命令和写数据通常只是在“控制字节”和后续内容上不同：

```text
写命令：地址 -> 控制字节(命令) -> 命令值
写数据：地址 -> 控制字节(数据) -> 显示数据
```

控制字节的具体数值不能凭空写死，要按 OLED 控制器手册或课程代码确认。

### I2C 基本读流程

I2C 读寄存器常见是复合格式：

```text
START
-> 发送从设备地址 + 写方向
-> 发送寄存器地址
-> RESTART
-> 发送从设备地址 + 读方向
-> 接收数据
-> NACK
-> STOP
```

OLED 入门显示主要是写；传感器章节会更多用读寄存器。

### OLED 显示框架

```text
OLED_Init()
-> 发送初始化命令序列
-> OLED_Clear()
-> OLED_ShowChar()
-> OLED_ShowString()
-> OLED_ShowBitmap()
```

显示字符的关键是字库：字符编码对应一组点阵字节。显示位图的关键是图片尺寸、取模方向、数据排列和 OLED 显存组织一致。

> 视频待核对：课程使用的 OLED 取模方式、字库数组格式、位图数据排列方向和坐标 API。

---

## CubeMX / MDK 配置

### I2C 主机配置

1. 选择当前 STM32F103 工程或新建工程。
2. 配置系统时钟，沿用前面已验证的时钟方案即可。
3. 在 `Connectivity` 中选择课程使用的 `I2C1` 或 `I2C2`。
4. Mode 选择 `I2C`。
5. 配置 I2C Speed，常见标准模式为 100 kHz，快速模式为 400 kHz；实际以模块和课程配置为准。
6. GPIO 会自动切到 I2C SCL/SDA 复用开漏；确认引脚与硬件接线一致。
7. 如果 OLED 模块没有板载上拉，外部需要上拉电阻。
8. 基础轮询发送不需要 NVIC / DMA。
9. 生成 MDK 工程。
10. 用户代码写在 `USER CODE` 区域；OLED 驱动建议放在 `BSP/oled.c` 和 `BSP/oled.h`。

I2C 引脚、速度、地址、OLED 供电和上拉方式：

以实际开发板原理图和 CubeMX 配置为准。

### MDK 文件加入

1. 新建 `BSP/` 目录。
2. 添加 `BSP/oled.c`、`BSP/oled.h`。
3. 在 MDK 中新建 `BSP` 分组并加入 `.c` 文件。
4. 把 `BSP/` 加入 Include Path。
5. 在 `main.c` 中只调用 `OLED_Init()`、`OLED_Clear()`、`OLED_ShowString()` 等接口。

---

## 最小代码

以下代码是 OLED I2C 驱动的最小示例框架，不声称可直接编译运行。控制器地址、控制字节、初始化命令、坐标和字库都需要按课程视频或模块资料补齐。

### `BSP/oled.h`

```c
#ifndef OLED_H
#define OLED_H

#include "main.h"

void OLED_Init(void);
void OLED_Clear(void);
void OLED_ShowChar(uint8_t x, uint8_t y, char ch);
void OLED_ShowString(uint8_t x, uint8_t y, const char *str);
void OLED_ShowBitmap(uint8_t x, uint8_t y, const uint8_t *bitmap, uint8_t width, uint8_t height);

#endif
```

### `BSP/oled.c`

```c
#include "oled.h"
#include "i2c.h"

#define OLED_I2C_HANDLE        hi2c1
#define OLED_I2C_ADDR          (0x00 << 1)
#define OLED_CONTROL_CMD       0x00
#define OLED_CONTROL_DATA      0x00

static HAL_StatusTypeDef OLED_WriteByte(uint8_t control, uint8_t value)
{
    uint8_t buf[2];

    buf[0] = control;
    buf[1] = value;

    return HAL_I2C_Master_Transmit(&OLED_I2C_HANDLE,
                                   OLED_I2C_ADDR,
                                   buf,
                                   sizeof(buf),
                                   100);
}

static void OLED_WriteCommand(uint8_t cmd)
{
    (void)OLED_WriteByte(OLED_CONTROL_CMD, cmd);
}

static void OLED_WriteData(uint8_t data)
{
    (void)OLED_WriteByte(OLED_CONTROL_DATA, data);
}

void OLED_Init(void)
{
    /*
     * 初始化命令序列与 OLED 控制器强相关。
     * 需要按课程视频或模块资料补齐。
     */
    OLED_WriteCommand(0x00);
}

void OLED_Clear(void)
{
    /*
     * 清屏通常是遍历显示 RAM，把每个字节写 0x00。
     * 页数、列数、寻址命令以实际 OLED 为准。
     */
}

void OLED_ShowChar(uint8_t x, uint8_t y, char ch)
{
    /*
     * 根据 x/y 设置显示位置。
     * 根据 ch 查字库数组。
     * 把字模字节写入 OLED。
     */
    (void)x;
    (void)y;
    (void)ch;
}

void OLED_ShowString(uint8_t x, uint8_t y, const char *str)
{
    while ((str != 0) && (*str != '\0'))
    {
        OLED_ShowChar(x, y, *str);
        x++;
        str++;
    }
}

void OLED_ShowBitmap(uint8_t x, uint8_t y, const uint8_t *bitmap, uint8_t width, uint8_t height)
{
    /*
     * 位图显示需要确认取模方向、宽高单位和 OLED 页组织。
     * bitmap 数据格式未确认前这里只保留接口。
     */
    (void)x;
    (void)y;
    (void)bitmap;
    (void)width;
    (void)height;
}
```

> 视频待核对：`OLED_I2C_ADDR`、`OLED_CONTROL_CMD`、`OLED_CONTROL_DATA`、初始化命令、清屏页数、字库数组和位图取模格式。

### `Core/Src/main.c`

在 `USER CODE BEGIN Includes` 中：

```c
#include "oled.h"
```

在 `USER CODE BEGIN 2` 中：

```c
OLED_Init();
OLED_Clear();
```

在 `while (1)` 的 `USER CODE` 区域中：

```c
OLED_ShowString(0, 0, "DHT11:");
HAL_Delay(1000);
```

---

## 调试方法

按指定顺序排查：

### 供电

1. OLED 模块供电电压是否符合模块要求。
2. OLED 是否需要 3.3 V 或 5 V 供电，不能凭接口丝印猜。
3. 上电后模块是否有异常发热。

### 共地

1. STM32、OLED 模块、外部电源必须共地。
2. 如果用面包板，先用万用表确认 GND 轨连通。

### 上拉电阻

1. I2C 的 SCL/SDA 需要上拉。
2. 有些 OLED 模块自带上拉，有些没有。
3. 空闲时 SCL/SDA 应为高电平。

### I2C 地址

1. 先用地址扫描框架确认哪个地址有 ACK。
2. HAL 里常用左移后的 8 位地址形式，注意和模块资料中的 7 位地址区分。
3. 不确定地址时不要写死，先扫描。

### CubeMX 配置

1. I2C 外设是否启用。
2. SCL/SDA 引脚是否和接线一致。
3. GPIO 是否为复用开漏。
4. I2C 速度是否在模块可接受范围内。

### HAL API 返回值

1. `HAL_I2C_Master_Transmit()` 是否返回 `HAL_OK`。
2. 失败时记录返回值，不要只看屏幕是否亮。
3. 必要时查询 `HAL_I2C_GetState()`。

### 逻辑分析仪 / 串口打印

1. 用串口打印当前发送的地址、命令、返回值。
2. 用逻辑分析仪观察是否有 START、地址、ACK、STOP。
3. 如果只有地址无 ACK，优先查地址、供电、上拉和接线。

### 业务逻辑

1. 先验证地址 ACK。
2. 再验证初始化命令。
3. 再做清屏。
4. 最后做字符和位图显示。
5. 位图不对时重点查取模方向和坐标，而不是先改 I2C。

---

## 工程意义

I2C + OLED 是嵌入式项目里很实用的“人机反馈出口”。有了 OLED，后面 DHT11、MAX30102、VL53L0X、MPU6050 这些模块不一定都靠串口看数据，也可以直接在板子上显示状态。

更重要的是，OLED 让我第一次完整接触到：

```text
协议通信
-> 外设初始化
-> 显示缓冲
-> 字库
-> 位图数据
```

这比单纯点灯更接近真实小项目。

---

## 易混淆点

| 容易混淆 | 正确理解 |
|----------|----------|
| 7 位地址和 HAL 地址 | 模块资料常写 7 位地址，HAL 发送时通常用左移后的地址 |
| SCL/SDA 和 TX/RX | I2C 不是串口，SCL/SDA 不需要交叉连接 |
| 开漏和推挽 | I2C 总线通常使用开漏加上拉，多个设备才能共享总线 |
| OLED 地址和控制字节 | 地址用来选设备，控制字节用来区分命令/数据 |
| 字符显示和 `printf` | OLED 显示字符本质是查字库写像素点 |
| 清屏和初始化 | 初始化配置控制器，清屏只是清显示 RAM |
| 位图显示和图片文件 | MCU 写的是取模后的字节数组，不是直接写 PNG/JPG |

---

## 我的理解

我现在把 I2C 理解成一条两线公共小路。STM32 先喊地址，被喊中的设备回应，然后 STM32 再告诉它要写命令、写数据或读寄存器。

OLED 对我来说不是一个“会自动显示字符串的屏幕”，而是一个需要我按它的规则写显存的外设。字符、汉字、图标，本质上都要变成字节点阵。

以后 OLED 不显示，我会先按这个顺序查：

```text
供电
-> 共地
-> 上拉
-> 地址 ACK
-> CubeMX I2C
-> HAL 返回值
-> 初始化命令
-> 坐标/字库/位图
```

---

## 复习检查清单

- [ ] 我能说明 I2C 为什么需要上拉电阻。
- [ ] 我能区分 7 位地址和 HAL 常用地址写法。
- [ ] 我知道 SCL/SDA 不像串口 TX/RX 那样交叉连接。
- [ ] 我能在 CubeMX 中配置 I2C 主机模式。
- [ ] 我能写出 OLED 写命令/写数据的框架。
- [ ] 我知道 OLED 控制器型号、地址、分辨率、字库和位图格式必须核对。
- [ ] 我能用 HAL 返回值和逻辑分析仪判断 I2C 是否真的通信。
- [ ] 我知道所有不确定参数以实际开发板原理图和 CubeMX 配置为准。
