# 07 TIM 触发 ADC 采集

> 覆盖课程：[24]
>
> 例程：TIM3 通过 TRGO 硬件触发 ADC1 采集 STM32 内部温度传感器，串口输出芯片温度。

---

## 概念

### 知识地图

| 名词 | 工程含义 | 常见位置 |
|------|----------|----------|
| Update Event | 定时器计数溢出时产生的更新事件 | TIM |
| TRGO | Trigger Output，定时器对外输出的内部触发信号 | TIM |
| External Trigger | ADC 不由软件启动，而由外设触发启动 | ADC |
| External Trigger Edge | 外部触发信号的有效边沿 | ADC |
| 内部温度传感器 | STM32 芯片内部测温元件，连接 ADC1_IN16 | ADC |
| V25 | 温度传感器在 25 ℃ 时的典型输出电压 | 数据手册 |
| Avg_Slope | 温度传感器平均斜率，单位 mV/℃ | 数据手册 |
| `HAL_MAX_DELAY` | HAL 最大等待时间，常表示不设超时 | HAL |

### 本节关键 API

| API | 作用 |
|-----|------|
| `HAL_ADCEx_Calibration_Start(&hadc1)` | ADC 校准 |
| `HAL_ADC_Start(&hadc1)` | 启动 ADC，使其等待外部触发 |
| `HAL_TIM_Base_Start(&htim3)` | 启动 TIM3，让它产生 TRGO |
| `HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY)` | 等待一次 ADC 转换完成 |
| `HAL_ADC_GetValue(&hadc1)` | 读取 ADC 转换结果 |
| `__io_putchar()` | `printf` 串口重定向 |

---

## 本质理解

之前的 ADC 采样是软件触发：

```text
CPU 调 HAL_ADC_Start()
-> ADC 开始转换
```

本节使用硬件触发：

```text
TIM3 定时溢出
-> 产生 Update Event
-> 通过 TRGO 发给 ADC1
-> ADC1 开始转换
```

本质区别：**采样时刻由定时器硬件决定，而不是由 `while(1)` 循环速度决定。**

---

## 工作流程 / 原理

### 三个角色

```text
TIM3
  负责精确定时，到点发 TRGO

ADC1
  等待 TRGO，收到后采集一次

内部温度传感器
  连接到 ADC1_IN16，提供随芯片温度变化的电压
```

芯片内部连接关系：

```text
TIM3 Update Event
-> TIM3 TRGO
-> ADC1 External Trigger
-> ADC1_IN16 转换
-> EOC 标志
-> CPU 读取结果
```

### TRGO 是什么

定时器计数到 `ARR` 后溢出：

```text
CNT: 0 -> 1 -> 2 -> ... -> ARR -> 0
                                  |
                                  +-> Update Event
                                      +-> TRGO 脉冲
```

CubeMX 里选择：

```text
Trigger Output (TRGO) = Update Event
```

含义：每次定时器溢出，都对外发出一个触发信号。

### ADC 外部触发

ADC 外部触发配置的含义：

```text
External Trigger Conversion Source = Timer 3 Trigger Out
External Trigger Conversion Edge   = Rising edge
```

意思是 ADC 不靠 CPU 一次次启动，而是等待 TIM3 的 TRGO 上升沿。TRGO 来一次，ADC 转换一次。

### 完整运行流程

```text
上电
-> HAL_Init()
-> SystemClock_Config()
-> MX_GPIO_Init()
-> MX_ADC1_Init()
-> MX_TIM3_Init()
-> MX_USART1_UART_Init()
-> ADC 校准
-> 启动 ADC，让 ADC 等 TRGO
-> 启动 TIM3，让 TIM3 周期性发 TRGO
-> 等内部温度传感器稳定
-> while(1) 中等待 ADC 转换完成
-> 读取 ADC
-> 换算温度
-> printf 输出
```

启动顺序很重要：

```text
先 ADC 校准
-> 再 HAL_ADC_Start()
-> 再 HAL_TIM_Base_Start()
```

如果先启动 TIM，TRGO 已经发出，但 ADC 还没准备好，会丢掉前几个触发。低频采样影响不大，高频采样时可能造成第一批数据异常。

### `PollForConversion` 在这里的角色

```c
HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY);
```

这里的 `Poll` 不是软件触发采样，而是 CPU 阻塞等待“这次由 TIM 触发的转换完成”。

```text
TIM 负责触发
ADC 负责转换
CPU 负责等待完成和读取结果
```

更工程化的高速采样方案通常会用 DMA 或中断，避免 CPU 一直阻塞等待。

### 内部温度传感器

STM32F103 的内部温度传感器连接到 `ADC1_IN16`。

特性：温度越高，输出电压越低。

数据手册典型参数：

| 参数 | 符号 | 典型值 | 含义 |
|------|------|--------|------|
| 25 ℃ 电压 | V25 | 1.43 V | 芯片 25 ℃ 时输出电压 |
| 平均斜率 | Avg_Slope | 4.3 mV/℃ | 温度变化对应电压变化 |

换算：

```text
V_sensor = ADC_Value / 4095 x 3.3

Temperature = (1.43 - V_sensor) / 0.0043 + 25
```

例：

```text
ADC_Value = 1676
V_sensor = 1676 / 4095 x 3.3 = 1.35 V
Temperature = (1.43 - 1.35) / 0.0043 + 25
            = 43.6 ℃
```

注意：

- 这是芯片内部硅片温度，不是空气温度。
- 芯片运行时会发热，通常比室温高 5~15 ℃。
- V25 和 Avg_Slope 是典型值，精确测温需要单独标定。

### ADC 时钟

```c
PeriphClkInit.AdcClockSelection = RCC_ADCPCLK2_DIV6;
```

常见配置：

| 项目 | 值 |
|------|----|
| PCLK2 | 72 MHz |
| ADC 分频 | /6 |
| ADC_CLK | 12 MHz |

F103 ADC 时钟不能超过 14 MHz，所以 12 MHz 是安全配置。

---

## CubeMX 配置

### TIM3

1. `Timers -> TIM3`。
2. `Clock Source` 选择 `Internal Clock`。
3. 配置 1 秒触发一次：

```text
PSC = 7199
ARR = 9999
TIM3_CLK = 72 MHz
触发周期 = (7199 + 1) x (9999 + 1) / 72 MHz = 1 s
```

4. `Trigger Output (TRGO)` 选择 `Update Event`。
5. 不需要打开 TIM3 NVIC，因为 TRGO 是硬件触发，不靠中断。

### ADC1

1. `ADC1` 勾选 `Temperature Sensor Channel`，即内部 `ADC1_IN16`。
2. `External Trigger Conversion Source` 选择 `Timer 3 Trigger Out`。
3. `External Trigger Conversion Edge` 选择 `Rising edge`。
4. `Continuous Conversion Mode` = `Disable`。
5. `Scan Conversion Mode` = `Disable`。
6. 采样时间建议设置长一些，例如 `239.5 Cycles`，内部温度传感器也需要较长采样时间。

### USART1

1. `USART1` 设置为 `Asynchronous`。
2. 波特率 `115200`。
3. 用于 `printf` 输出温度。

---

## HAL 函数 / API

### `HAL_ADC_Start`

```c
HAL_ADC_Start(&hadc1);
```

在外部触发模式下，这一步不是立刻采样，而是让 ADC 进入等待触发状态。

### `HAL_TIM_Base_Start`

```c
HAL_TIM_Base_Start(&htim3);
```

启动 TIM3 计数。每次溢出时，TIM3 通过 TRGO 触发 ADC。

注意：这里不需要 `_IT`，因为不需要进入 TIM 中断。

### `HAL_ADC_PollForConversion`

```c
HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY);
```

等待 ADC 转换完成。触发来自 TIM3，等待来自 CPU。

---

## 示例代码

### `main.c`

```c
#include "main.h"
#include "adc.h"
#include "tim.h"
#include "usart.h"
#include "gpio.h"
#include <stdio.h>

/* USER CODE BEGIN PV */
uint32_t adc_val = 0;
float voltage = 0.0f;
float tem = 0.0f;
/* USER CODE END PV */

int main(void)
{
    HAL_Init();
    SystemClock_Config();

    MX_GPIO_Init();
    MX_ADC1_Init();
    MX_TIM3_Init();
    MX_USART1_UART_Init();

    /* USER CODE BEGIN 2 */

    HAL_ADCEx_Calibration_Start(&hadc1);

    HAL_ADC_Start(&hadc1);

    HAL_TIM_Base_Start(&htim3);

    HAL_Delay(3000);

    /* USER CODE END 2 */

    while (1)
    {
        if (HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY) == HAL_OK)
        {
            adc_val = HAL_ADC_GetValue(&hadc1);
            voltage = (float)adc_val / 4095.0f * 3.3f;
            tem = (1.43f - voltage) / 0.0043f + 25.0f;

            printf("ADC=%lu, voltage=%.3f V, chip temperature=%.2f C\r\n",
                   adc_val,
                   voltage,
                   tem);
        }
    }
}
```

### `printf` 重定向

```c
/* USER CODE BEGIN 4 */
int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
/* USER CODE END 4 */
```

如果 `%f` 无法输出，需要在 CubeIDE 链接选项添加：

```text
-u _printf_float
```

### 启动顺序模板

```c
HAL_ADCEx_Calibration_Start(&hadc1);
HAL_ADC_Start(&hadc1);
HAL_TIM_Base_Start(&htim3);
```

记忆方式：

```text
先校准
-> 先让 ADC 准备好接收触发
-> 再让 TIM 开始发触发
```

---

## 常见错误

| 问题 | 常见原因 | 解决方法 |
|------|----------|----------|
| ADC 不触发 | TIM3 没配 TRGO | TIM3 `Trigger Output` 选 `Update Event` |
| ADC 不触发 | ADC 外部触发源没选 TIM3 | ADC `External Trigger` 选 `Timer 3 Trigger Out` |
| 只采一次或节奏异常 | Continuous 配置和外部触发理解混乱 | TIM 触发模式下关闭 Continuous |
| 前几次数据异常 | TIM 比 ADC 先启动 | 按校准 -> ADC Start -> TIM Start 顺序 |
| 温度比室温高 | 正常，测的是芯片硅片温度 | 只当作芯片内部温度参考 |
| 温度偏差明显 | V25/斜率使用典型值 | 精确测温需标定 |
| `printf` 不输出 | 串口没配、没重定向、波特率不对 | 检查 USART1 和 `__io_putchar` |
| `%f` 不显示 | 未启用浮点打印 | 添加 `-u _printf_float` |
| 程序一直卡在 Poll | TIM 未启动或触发源错误 | 先查 TIM3 TRGO 和 ADC trigger |

---

## 调试方法

1. 先确认 USART1 `printf` 能正常输出。
2. 单独用软件触发 ADC 读取内部温度传感器，确认 ADC 本身正常。
3. 再改成 TIM3 外部触发。
4. 在 `HAL_ADC_PollForConversion()` 后打断点，看是否每秒到达一次。
5. 若一直不到达，检查 TIM3 是否启动、TRGO 是否配置、ADC trigger 是否选对。
6. 打印原始 `adc_val` 和 `voltage`，不要只看温度结果。

排查链路：

```text
USART 输出正常
-> ADC 校准正常
-> ADC 软件触发可读
-> TIM3 周期正确
-> TRGO = Update Event
-> ADC Trigger = TIM3 TRGO
-> 启动顺序正确
```

---

## 工程意义

TIM 触发 ADC 的意义是：让采样时刻由硬件保证。

适合场景：

- 固定采样率的数据采集。
- 波形采样。
- 控制系统周期采样。
- 多传感器同步采集。
- 后续配合 DMA 做连续采样。

和软件循环采样相比：

| 方式 | 采样间隔 | CPU 负担 | 工程适用性 |
|------|----------|----------|------------|
| `while(1)` 软件触发 | 不稳定 | CPU 控制触发 | 简单实验 |
| TIM TRGO 触发 | 稳定 | CPU 不管触发时刻 | 工程采样 |
| TIM TRGO + DMA | 稳定 | CPU 只处理缓冲区 | 高频/连续采样 |

---

## 易混淆点

| 容易混淆 | 正确理解 |
|----------|----------|
| TRGO 和中断 | TRGO 是硬件内部触发信号，不需要进中断 |
| `HAL_TIM_Base_Start` 和 `_Start_IT` | 本节只要 TRGO，不需要 TIM 中断 |
| ADC Start 和软件触发 | 外部触发模式下 Start 是让 ADC 准备好等触发 |
| Poll 和触发 | Poll 只是等待转换完成，不负责触发 |
| Continuous 和外部触发 | 本节 TIM 来一次采一次，Continuous 关闭 |
| 芯片温度和室温 | 内部温度传感器测的是硅片温度 |
| V25/Avg_Slope | 数据手册典型值，不是每颗芯片精确值 |

---

## 我的理解

这一章的关键是把“谁触发谁”想清楚。

我现在的理解是：

- TIM3 是节拍器。
- TRGO 是 TIM3 发出的内部触发信号。
- ADC1 不再听 CPU 的每次命令，而是听 TIM3 的节拍。
- CPU 只是等结果、读结果、算结果。
- 内部温度传感器只是一个练习通道，真正工程里可以换成任意外部模拟信号。

以后做精确定时采样，我会按这个顺序检查：

```text
TIM 周期算对
-> TRGO 选对事件
-> ADC 外部触发源选对
-> ADC 先启动
-> TIM 后启动
-> 需要高效率时再加 DMA
```

---

*课程：[24]*
