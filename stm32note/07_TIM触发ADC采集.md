# 07 TIM 触发 ADC 采集

> 覆盖课程：[24]
>
> 例程：TIM3 硬件触发 ADC1 采集 STM32 内部温度传感器，printf 输出芯片温度。

---

## 这节课要解决什么问题？

之前 [05 ADC](05_ADC模数转换.md) 学的是在 while(1) 里手动调 `HAL_ADC_Start` 去采一次数据。但这种方式有两个问题：

1. **采集间隔不准**：while(1) 循环一圈要多少时间是不确定的，没法做到「每 1ms 精确采一次」
2. **CPU 要操心触发时机**：什么时候该采、采完了没，CPU 要一直管

这节学的就是：**让定时器的硬件信号自动去触发 ADC**，CPU 只管拿结果就行。

---

## 知识地图（先搞清楚这节课出现了哪些新东西）

| 新名词 | 一句话解释 | 在哪出现 |
|--------|----------|---------|
| **Update Event（更新事件）** | 定时器 CNT 计到 ARR 归零的那个瞬间 | TIM 内部 |
| **TRGO（Trigger Output，触发输出）** | TIM 把「Update Event」转换成的一个脉冲信号，可以送给其他外设 | TIM 的输出 |
| **External Trigger（外部触发）** | ADC 不从软件指令启动，而是等别的硬件发信号来启动 | ADC 的输入 |
| **内部温度传感器** | STM32 芯片里面自带的一个测温元件，连在 ADC1 通道 16 | ADC1_IN16 |
| **V25 / Avg_Slope** | 温度传感器的两个出厂参数：25℃ 时输出多少伏、每升高 1℃ 电压变多少 | 数据手册 |
| **HAL_MAX_DELAY** | 等于 `0xFFFFFFFF`，意思是「一直等，不设超时限制」 | HAL 库常量 |

---

## 关键原理

### 一、先理解三个角色

```
┌──────────────────────────────────────────────┐
│                  STM32 芯片内部                │
│                                              │
│   TIM3                    ADC1               │
│  ┌──────────┐    TRGO    ┌──────────────┐    │
│  │ CNT 计数  │───信号线──→│  收到触发信号  │    │
│  │ 到 ARR 后  │  (芯片内  │  开始一次转换  │    │
│  │ 产生 TRGO │  部连线)  │  转换完设 EOC │    │
│  └──────────┘           └──────────────┘    │
│                                              │
│  温度传感器（内部）──→ ADC1_IN16 通道         │
└──────────────────────────────────────────────┘
```

- **TIM3** = 一个精确定时器，只管按固定节奏发脉冲（TRGO）
- **ADC1** = 收到脉冲就去采集一次 IN16（温度传感器通道）
- **TRGO** = 芯片内部的一条「看不见的线」，把 TIM3 的脉冲送给 ADC1

### 二、TRGO 到底是什么？

之前 [03 中断与定时器](03_中断与定时器.md) 学过，TIM 的 CNT 加到 ARR 后会溢出。那时候我们关心的是「溢出 → 进中断回调」。

但这节课我们不需要进中断，我们关心的是溢出瞬间的**硬件信号**：

```
CNT: 0 → 1 → 2 → ... → ARR-1 → ARR → 0 (溢出！)
                                            └→ 这个瞬间 = Update Event
                                                   └→ 顺便发出一个 TRGO 脉冲
```

> **TRGO = Update Event 的「广播」版本。** TIM 把它从内部信号变成了一个可以送给其他外设的脉冲。

CubeMX 里选 `Trigger Output (TRGO) → Update Event` 的意思就是：「每次 CNT 计满溢出时，对外发一个 TRGO 脉冲」。

### 三、ADC 的「外部触发」是什么意思？

之前 ADC 的启动方式是在代码里主动调用 `HAL_ADC_Start`，这叫**软件触发**——CPU 说什么时候采就什么时候采。

这节我们给 ADC 换了一种启动方式：**不用 CPU 命令，而是等着 TIM3 的 TRGO 信号来敲门**。

```
软件触发（之前学的）：
  CPU 执行 HAL_ADC_Start → ADC 开始转换

外部触发（这节学的）：
  TIM3 TRGO 脉冲到达 → ADC 开始转换
  （CPU 不用管触发时机）
```

CubeMX 里配的 `External Trigger Conversion Source → Timer 3 Trigger Out` 就是告诉 ADC：「你别等 CPU 来指挥，你听 TIM3 的 TRGO 信号」。

### 四、程序运行的完整流程

```
上电 → 初始化各外设
    ↓
① HAL_ADCEx_Calibration_Start  →  校准 ADC 内部电容
    ↓
② HAL_ADC_Start                →  启动 ADC，让它进入「等 TRGO 来敲门」的状态
    ↓
③ HAL_TIM_Base_Start           →  启动 TIM3，开始发 TRGO 脉冲
    ↓
④ HAL_Delay(3000)              →  等 3 秒，让温度传感器稳定下来
    ↓
┌─ while(1) ────────────────────────────────────┐
│                                                │
│   TIM3 每 1 秒溢出一次 → 发一个 TRGO           │
│         ↓                                      │
│   ADC1 收到 TRGO → 转换一次 IN16               │
│         ↓                                      │
│   PollForConversion 等到转换完成                │
│         ↓                                      │
│   算出温度 → printf                            │
│         ↓                                      │
│   回到循环顶部，等下一次 TIM3 溢出               │
│                                                │
└────────────────────────────────────────────────┘
```

### 五、PollForConversion 在这里的角色

```c
if (HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY) == HAL_OK)
```

- `HAL_ADC_PollForConversion`：阻塞等待，直到 ADC 完成一次转换（EOC 标志置位）
- `HAL_MAX_DELAY`：没有超时限制，一直等。因为 TIM3 每 1 秒一定会触发一次，不会等不到
- 拿到结果后，代码回到 while(1) 顶部，再次调用 `PollForConversion`，等**下一次** TIM3 触发

> 注意：这里虽然用了「Poll」（轮询）这个词，但触发源是 TIM3 硬件，不是 CPU 在轮询触发。CPU 只是轮询**等待转换完成**。

### 六、内部温度传感器详解

STM32F103C8T6 芯片内部自带一个温度传感器，不需要任何外接电路。

**在哪？** 连在 `ADC1_IN16` 通道上（CubeMX 勾选 Temperature Sensor Channel 就用它）。

**特性：负温度系数（NTC）**
```
温度越高 → 传感器输出电压越低
温度越低 → 传感器输出电压越高
```

**两个关键参数（来自 STM32 数据手册）：**

| 参数 | 符号 | 典型值 | 含义 |
|------|------|--------|------|
| 25℃ 基准电压 | V25 | 1.43V | 芯片 25℃ 时，传感器输出的电压 |
| 平均斜率 | Avg_Slope | 4.3mV/℃ | 温度每升高 1℃，电压下降 0.0043V |

#### 换算公式推导（一步一步来）

```
第 1 步：ADC 读数 → 电压值
  V_sensor = ADC值 / 4095 × 3.3

第 2 步：电压值 → 温度（利用 V25 和 Avg_Slope）
  电压比 25℃ 时低多少 = 1.43 - V_sensor
  对应温度升高了多少 = (1.43 - V_sensor) / 0.0043
  当前温度 = 25 + 上面算出来的值

  合并：Temp = (1.43 - V_sensor) / 0.0043 + 25
```

#### 具体算一个例子

```
假设 ADC 读到 1676：
  V_sensor = 1676 / 4095 × 3.3 = 1.35V

  Temp = (1.43 - 1.35) / 0.0043 + 25
       = 0.08 / 0.0043 + 25
       = 18.6 + 25
       = 43.6℃
```

> ⚠️ 这是**芯片内部硅片的温度**，不是空气温度。芯片跑程序时会发热，通常比室温高 5~15℃。
>
> ⚠️ V25 和 Avg_Slope 每个芯片都有细微差异（数据手册给的是典型值），这是正常制造误差。精确测温需要自己标定——不过这节课先不用管，用典型值就行。

### 七、为什么要 `HAL_Delay(3000)`？

```c
HAL_TIM_Base_Start(&htim3);
HAL_Delay(3000);  // 等 3 秒
```

等 3 秒的原因：内部温度传感器在上电后需要一段时间让读数稳定下来。如果启动后立刻采，前几秒的数据可能不准。这和 [05 ADC](05_ADC模数转换.md) 里 MQ 烟雾传感器需要预热是类似的道理。

### 八、为什么要校准 ADC？

```c
HAL_ADCEx_Calibration_Start(&hadc1);
```

F103 的 ADC 内部有采样保持电容，出厂时每个芯片的电容值有细微差异。校准就是测量这个差异然后补偿进去。**F103 的 ADC 每次上电后校准一次就行。**

> 如果忘了校准，ADC 读数会有几到几十个 LSB 的偏移（对温度换算影响不算巨大，但能校准就校准）。

### 九、ADC 时钟为什么是 PCLK2/6？

```c
PeriphClkInit.AdcClockSelection = RCC_ADCPCLK2_DIV6;
```

| 项目 | 值 |
|------|-----|
| PCLK2（APB2 总线时钟） | 72MHz |
| 分频 | ÷6 |
| ADC 时钟 | 12MHz |

F103 的 ADC 时钟不能超过 **14MHz**。72 ÷ 6 = 12，安全。CubeMX 通常会自动配好，但了解一下不会吃亏。

---

## CubeMX 配置步骤

### TIM3 侧

1. Timers → **TIM3** → Clock Source: **Internal Clock**
2. PSC = 7199, ARR = 9999 → 定时 1 秒
3. **Trigger Output (TRGO) → Update Event** ← 这是这节课的**核心操作**
   - 含义：TIM3 每次溢出就发一个 TRGO 脉冲给其他外设
4. NVIC → **不用开**（TRGO 是纯硬件信号，不需要软件中断）

### ADC1 侧

1. ADC1 → **IN16** → 勾选 **Temperature Sensor Channel**
   - 这是内部温度传感器专用通道，不需要在芯片引脚上接线
2. **External Trigger Conversion Source → Timer 3 Trigger Out**
   - 含义：ADC 不靠 CPU 命令启动，而是等 TIM3 的 TRGO 信号来启动
3. **External Trigger Conversion Edge → Rising edge**
   - 含义：TRGO 脉冲的上升沿触发一次转换
4. Continuous Conversion Mode = **Disable**
   - TIM 来一次 TRGO 就采一次，不需要 ADC 自己连续采
5. Scan Conversion Mode = **Disable**（只有一个通道，不用扫描）

### USART1

1. Mode: **Asynchronous**，用于 printf 输出温度值

---

## 完整代码

### main.c

```c
#include "main.h"
#include "adc.h"
#include "tim.h"
#include "usart.h"
#include "gpio.h"
#include <stdio.h>          // printf 需要

/* USER CODE BEGIN PV */
uint32_t adc_val = 0;       // 存 ADC 原始读数
float voltage  = 0.0f;      // 存换算后的电压值
float tem      = 0.0f;      // 存换算后的温度值
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

    // ① ADC 校准 —— F103 上电必须做一次
    HAL_ADCEx_Calibration_Start(&hadc1);

    // ② 启动 ADC —— ADC 进入「等 TRGO 来敲门」状态
    HAL_ADC_Start(&hadc1);

    // ③ 启动 TIM3 —— 开始计数的同时开始发 TRGO 脉冲
    HAL_TIM_Base_Start(&htim3);

    // ④ 等 3 秒让温度传感器稳定
    HAL_Delay(3000);

    /* USER CODE END 2 */

    while (1)
    {
        // 阻塞等待本次 TIM 触发后的 ADC 转换完成
        if (HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY) == HAL_OK)
        {
            adc_val  = HAL_ADC_GetValue(&hadc1);              // 拿 ADC 原始值
            voltage  = (float)adc_val / 4095.0f * 3.3f;      // → 电压
            tem      = (1.43f - voltage) / 0.0043f + 25.0f;  // → 温度

            printf("The temperature of silicon chip : %f\r\n", tem);
        }
        // 循环回到顶部，等下一次 TIM3 触发
    }
}
```

### printf 重定向

```c
/* USER CODE BEGIN 4 */
int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
/* USER CODE END 4 */
```

> `__io_putchar`：CubeIDE 的 printf 底层输出函数。printf 内部每要输出一个字符，就会调用这个函数。我们把字符通过串口发出去，printf 的输出就能在串口助手里看到了。

---

## 启动顺序（为什么是这个顺序？）

```
① HAL_ADCEx_Calibration_Start   先校准
② HAL_ADC_Start                 ADC 进入等待触发状态
③ HAL_TIM_Base_Start            TIM 才开始发 TRGO
④ HAL_Delay(3000)               等传感器稳定
```

> 🔴 ② 和 ③ **不能反过来**。如果先启动 TIM3，TRGO 已经发出来了，但 ADC 还没准备好去收——前几个触发就浪费了。虽然这节定时 1 秒一次影响不大（最多浪费一个），但以后定时间隔很短的场合，顺序反了会出问题。养成习惯就好。

---

## 易错点

| 坑 | 原因 | 解决 |
|----|------|------|
| ADC 不触发 | TIM3 侧没配 TRGO | CubeMX → TIM3 → Trigger Output 选 Update Event |
| ADC 不触发 | ADC 侧 Ext Trigger 没选 TIM3 | ADC1 → External Trigger → Timer 3 Trigger Out |
| 温度值和室温差很多 | **正常现象**，芯片工作发热 | 内部温度 ≈ 室温 + 5~15℃ |
| 启动后前几条温度不准 | 内部温度传感器需要稳定时间 | 加 `HAL_Delay(3000)` |
| printf 不输出 | 串口没配好 / 波特率不对 | 检查 USART1 配置，串口助手 115200 |
| 打印出 `%f` 而不是数字 | 没加浮点数支持 | CubeIDE 链接选项加 `-u _printf_float` |
| 忘了 include | `#include <stdio.h>` 漏了 | printf 需要 stdio.h |

---

*课程：[24]*
