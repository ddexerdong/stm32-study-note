# 10 DWT 与单总线传感器

> 覆盖课程：[28-1] [28-2] [29]
>
> 目标：理解 DWT 周期计数器的调试价值，掌握微秒级计时的框架，并用“严格时序 + GPIO 输入输出切换”的思路理解 DHT11 温湿度传感器。

---

## 本节目标

完成本节后，应能做到：

- 说明 DWT / CYCCNT 为什么适合观察程序运行时间。
- 写出 DWT 初始化、读取计数值、换算时间的最小框架。
- 理解 DHT11 这类单总线传感器为什么对微秒时序敏感。
- 搭建 DHT11 读取的最小状态框架，而不是死记整段驱动。
- 遇到读数失败时，按硬件、时序、GPIO 模式、校验顺序排查。

---

## 核心概念

### 知识地图

| 名词 | 工程含义 | 常见位置 |
|------|----------|----------|
| DWT | Data Watchpoint and Trace，Cortex-M 调试组件，可用于周期计数 | `[28-1] [28-2]` |
| CYCCNT | CPU 周期计数器，记录运行了多少个核心时钟周期 | DWT 寄存器 |
| 微秒延时 | 比 `HAL_Delay()` 更细的延时尺度 | 单总线、脉冲测量 |
| 单总线 | 数据线只有一根，主机和从机分时拉高/拉低 | `[29]` |
| DHT11 | 温湿度传感器，使用单数据线按时序输出数据 | `[29]` |
| 起始信号 | MCU 拉低数据线通知传感器开始一次通信 | DHT11 |
| 响应信号 | 传感器拉低/拉高回应 MCU | DHT11 |
| 位时序 | 数据 0/1 由高电平持续时间区分 | DHT11 |
| 校验和 | 用于判断一帧数据是否可信 | DHT11 |
| GPIO 模式切换 | 同一根线在输出和输入之间切换 | `Core/Src/gpio.c` 或 BSP |

### 常用 HAL API

| API / 宏 | 作用 | 常见用法 |
|----------|------|----------|
| `HAL_GPIO_WritePin()` | 主机拉高/拉低数据线 | 起始信号、释放总线 |
| `HAL_GPIO_ReadPin()` | 读取 DHT11 数据线电平 | 响应和数据位 |
| `HAL_GPIO_Init()` | 运行时切换 GPIO 输入/输出模式 | 单总线收发 |
| `HAL_GetTick()` | 毫秒级超时保护 | 防止等待卡死 |
| DWT 寄存器封装 | 启动 CYCCNT、读取周期数 | 微秒延时和性能测量 |

---

## 本质理解

DWT 这章解决的是“我怎么知道一段代码到底跑了多久”。普通 `HAL_Delay()` 是毫秒级阻塞延时，适合慢速观察；DWT 的 CYCCNT 是按 CPU 周期计数，适合做更细粒度的运行时间测量。

DHT11 这章解决的是“没有标准 I2C/SPI 外设时，如何用普通 GPIO 按时序和外部芯片说话”。它的关键不是复杂算法，而是：

```text
按规定时间拉低/释放数据线
-> 切换为输入
-> 等待传感器响应
-> 按每一位的高电平宽度判断 0 或 1
-> 校验数据
```

PDF 中没有 DWT 和 DHT11 专题章节，所以本章只写通用框架、调试方法和需要核对的时序点，不把具体时序参数写成确定事实。

> 视频待核对：DWT 初始化封装、Debug 练习步骤、DHT11 数据线引脚、上拉方式和精确时序参数。

---

## 工作流程 / 原理

### DWT 计时思路

```text
使能调试跟踪单元
-> 清零 CYCCNT
-> 使能 CYCCNT
-> 记录 start
-> 执行待测代码
-> 记录 end
-> cycles = end - start
-> time_us = cycles / (SystemCoreClock / 1000000)
```

如果 `SystemCoreClock = 72 MHz`，那么 72 个周期约等于 1 us。实际主频以当前工程时钟配置为准。

### DHT11 单总线流程

```text
MCU 配置数据线为输出
MCU 发起始信号
MCU 释放总线并切换为输入
DHT11 响应
DHT11 连续输出 40 位数据
MCU 按位读取并组装 5 字节
校验和正确 -> 使用温湿度数据
校验失败 -> 丢弃本次数据
```

DHT11 的具体低电平/高电平持续时间、采样点和超时阈值必须看视频或模块手册确认。

以实际开发板原理图和 CubeMX 配置为准。

### 为什么要加超时

单总线读取常见写法会等待某个电平变化。如果传感器没接好、数据线被拉死、GPIO 模式错，程序可能永远卡在等待循环里。

所以每个等待边沿的地方都应该有超时：

```text
等待数据线变低
-> 超时则返回错误

等待数据线变高
-> 超时则返回错误

等待本位高电平结束
-> 超时则返回错误
```

---

## CubeMX / MDK 配置

### DWT Debug 练习

1. 使用已经能下载和 Debug 的 STM32F103 工程。
2. 确认系统时钟配置已稳定，`SystemCoreClock` 与实际主频一致。
3. 不需要在 CubeMX 中额外打开某个外设。
4. 在 MDK 中进入 Debug。
5. 在 DWT 初始化前后、待测代码前后设置断点。
6. 把 `start_cycles`、`end_cycles`、`cost_us` 加入 Watch 窗口。
7. 单步或运行到断点，观察周期数变化。

### DHT11 最小配置

1. 选择一个普通 GPIO 作为 DHT11 数据线。
2. 如果模块没有板载上拉，数据线需要上拉电阻；具体以模块原理图为准。
3. CubeMX 中先把该引脚配置为普通 GPIO 输出或输入均可，最终驱动会在运行时切换模式。
4. 不需要 NVIC 和 DMA。
5. 生成工程。
6. 建议把 DHT11 驱动放在 `BSP/dht11.c` 和 `BSP/dht11.h`。
7. 用户代码在 `main.c` 中只调用初始化、读取、错误处理接口。

DHT11 引脚、上拉方式、采样时序：

以实际开发板原理图和 CubeMX 配置为准。

---

## 最小代码

以下代码是框架，不保证直接匹配课程模块。具体时序参数必须按视频和模块手册修正。

### `BSP/dwt_delay.h`

```c
#ifndef DWT_DELAY_H
#define DWT_DELAY_H

#include "main.h"

void DWT_Delay_Init(void);
uint32_t DWT_GetCycles(void);
void DWT_Delay_Us(uint32_t us);

#endif
```

### `BSP/dwt_delay.c`

```c
#include "dwt_delay.h"

#define DEMCR_TRCENA    (1UL << 24)
#define DWT_CTRL_CYCCNT (1UL << 0)

void DWT_Delay_Init(void)
{
    CoreDebug->DEMCR |= DEMCR_TRCENA;
    DWT->CYCCNT = 0;
    DWT->CTRL |= DWT_CTRL_CYCCNT;
}

uint32_t DWT_GetCycles(void)
{
    return DWT->CYCCNT;
}

void DWT_Delay_Us(uint32_t us)
{
    uint32_t start = DWT->CYCCNT;
    uint32_t ticks = us * (SystemCoreClock / 1000000UL);

    while ((DWT->CYCCNT - start) < ticks)
    {
    }
}
```

> 视频待核对：课程中是否使用同样的 DWT 寄存器封装，是否需要处理不同编译器或 Debug 设置差异。

### `BSP/dht11.h`

```c
#ifndef DHT11_H
#define DHT11_H

#include "main.h"

typedef struct
{
    uint8_t humidity_int;
    uint8_t humidity_dec;
    uint8_t temperature_int;
    uint8_t temperature_dec;
} DHT11_Data_t;

uint8_t DHT11_Read(DHT11_Data_t *data);

#endif
```

### `BSP/dht11.c`

这里只写结构框架，关键等待和时序阈值需要按课程视频补齐。

```c
#include "dht11.h"
#include "dwt_delay.h"

#define DHT11_PORT GPIOB
#define DHT11_PIN  GPIO_PIN_0

static void DHT11_Pin_Output(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    GPIO_InitStruct.Pin = DHT11_PIN;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_OD;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(DHT11_PORT, &GPIO_InitStruct);
}

static void DHT11_Pin_Input(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    GPIO_InitStruct.Pin = DHT11_PIN;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(DHT11_PORT, &GPIO_InitStruct);
}

static uint8_t DHT11_WaitLevel(GPIO_PinState level, uint32_t timeout_us)
{
    uint32_t start = DWT_GetCycles();
    uint32_t timeout_ticks = timeout_us * (SystemCoreClock / 1000000UL);

    while (HAL_GPIO_ReadPin(DHT11_PORT, DHT11_PIN) != level)
    {
        if ((DWT_GetCycles() - start) > timeout_ticks)
        {
            return 0;
        }
    }

    return 1;
}

uint8_t DHT11_Read(DHT11_Data_t *data)
{
    uint8_t raw[5] = {0};
    uint8_t checksum = 0;

    if (data == 0)
    {
        return 0;
    }

    DHT11_Pin_Output();

    /* 起始信号时长需要按视频或模块手册核对。 */
    HAL_GPIO_WritePin(DHT11_PORT, DHT11_PIN, GPIO_PIN_RESET);
    HAL_Delay(20);
    HAL_GPIO_WritePin(DHT11_PORT, DHT11_PIN, GPIO_PIN_SET);
    DWT_Delay_Us(30);

    DHT11_Pin_Input();

    if (!DHT11_WaitLevel(GPIO_PIN_RESET, 100))
    {
        return 0;
    }

    if (!DHT11_WaitLevel(GPIO_PIN_SET, 100))
    {
        return 0;
    }

    if (!DHT11_WaitLevel(GPIO_PIN_RESET, 100))
    {
        return 0;
    }

    /*
     * 后续 40 位读取需要按课程时序补齐：
     * 1. 等待每一位开始。
     * 2. 测量高电平持续时间。
     * 3. 根据阈值判断 0/1。
     * 4. raw[4] 与 raw[0]~raw[3] 的和做校验。
     *
     * 这里是框架代码，位读取未补齐前不能返回成功。
     */
    (void)raw;
    (void)checksum;

    return 0;
}
```

### `Core/Src/main.c`

在 `USER CODE BEGIN Includes` 中：

```c
#include "dwt_delay.h"
#include "dht11.h"
```

在 `USER CODE BEGIN 2` 中：

```c
DWT_Delay_Init();
```

在 `while (1)` 的 `USER CODE` 区域中：

```c
DHT11_Data_t dht = {0};

if (DHT11_Read(&dht))
{
    /* 这里可以通过串口或 OLED 显示温湿度。 */
}
else
{
    /* 读取失败时先不要卡死，保留重试机会。 */
}

HAL_Delay(1000);
```

---

## 调试方法

按“硬件 -> CubeMX -> DWT -> 时序 -> 校验 -> 业务逻辑”的顺序排查。

### DWT 调试

1. 确认 `DWT_Delay_Init()` 被调用。
2. 在 Watch 中观察 `DWT->CYCCNT` 是否递增。
3. 确认 `SystemCoreClock` 与实际主频一致。
4. 用示波器或逻辑分析仪测一个 GPIO 翻转间隔，反推微秒延时是否可信。

### DHT11 调试

1. 确认 DHT11 供电和 GND。
2. 确认数据线有上拉，空闲时应为高电平。
3. 确认 GPIO 运行时能在输出和输入之间切换。
4. 用逻辑分析仪看起始信号和传感器响应。
5. 每个等待电平的循环都要有超时。
6. 校验失败时先打印原始 5 字节，不要直接改换算公式。

### 常见坑

| 现象 | 常见原因 | 处理 |
|------|----------|------|
| `CYCCNT` 不变 | DWT 未使能或 Debug 支持不完整 | 检查初始化和调试设置 |
| 微秒延时不准 | `SystemCoreClock` 不对 | 核对时钟配置 |
| DHT11 无响应 | 未供电、未共地、数据线无上拉、起始时序错 | 先查硬件和波形 |
| 程序卡死 | 等待电平没有超时 | 给每个等待循环加超时 |
| 读数偶尔错误 | 时序阈值不稳、线太长、干扰 | 用逻辑分析仪确认高低电平宽度 |
| 校验失败 | 位判断阈值错误或漏位 | 打印原始字节并检查 40 位读取流程 |

---

## 工程意义

DWT 让我能从“感觉这段代码很快/很慢”变成“用周期数证明它跑了多久”。DHT11 则让我意识到：不是所有模块都有硬件外设帮忙，很多简单传感器靠的就是普通 GPIO 和严格时序。

后面遇到 WS2813E 这类更严格的单线时序模块时，这一章的思路也能复用：

```text
先解决微秒/纳秒级时间基准
-> 再解决 GPIO 输出波形
-> 再看协议数据格式
-> 最后处理业务显示或上传
```

---

## 易混淆点

| 容易混淆 | 正确理解 |
|----------|----------|
| DWT 和 SysTick | DWT 可按 CPU 周期计数，SysTick 常用于 HAL 毫秒时基 |
| `HAL_Delay()` 和微秒延时 | `HAL_Delay()` 是毫秒级，DHT11 位时序通常需要微秒级 |
| 单总线和 UART | 单总线没有 UART 外设帧格式，靠 GPIO 时序识别数据 |
| GPIO 输出和输入 | 同一根数据线需要主机输出起始信号，再切回输入读取传感器 |
| 读取失败和公式错误 | 先确认时序和校验，校验不通过时换算公式没有意义 |
| 确定参数和待核对参数 | PDF 没支撑、视频没确认的时序和接线不能写成确定事实 |

---

## 我的理解

我现在把 DWT 理解成一个贴在 CPU 旁边的“秒表”，它数的不是毫秒，而是 CPU 跑过的周期数。只要主频知道，就能换算成时间。

我把 DHT11 理解成一个很挑时间的模块。STM32 和它通信时，不是发标准串口帧，也不是走 I2C 地址，而是靠一根线上的高低电平持续时间表达数据。

以后调这种单总线传感器，我会先按这个顺序来：

```text
供电和共地
-> 数据线空闲高电平
-> MCU 起始波形
-> 传感器响应波形
-> 40 位原始数据
-> 校验
-> 温湿度显示
```

---

## 复习检查清单

- [ ] 我能说明 DWT / CYCCNT 的作用。
- [ ] 我能用周期数换算程序运行时间。
- [ ] 我知道 DHT11 为什么需要微秒级时序。
- [ ] 我知道单总线读写时 GPIO 需要切换输入/输出。
- [ ] 我知道所有等待电平的循环都必须有超时。
- [ ] 我知道 DHT11 的精确时序参数要看视频或模块手册。
- [ ] 我能先看原始字节和校验，再判断温湿度换算是否可信。
- [ ] 我知道本章接线和参数以实际开发板原理图和 CubeMX 配置为准。
