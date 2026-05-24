# 06 定时器 PWM

> 覆盖课程：[22-1] [22-2] [23-1] [23-2]
>
> 目标：理解通用/高级定时器差异，掌握 PWM 产生原理，用 PWM 实现舵机控制和呼吸灯。

---

## 本节目标

完成本节后，应能做到：

- 理解 PWM 的频率、周期、占空比和脉宽。
- 使用 TIM 输出 PWM 信号。
- 通过修改 CCR 控制舵机角度和 LED 亮度。
- 区分通用定时器和高级定时器的典型用途。

---

## 核心概念

### 知识地图

| 名词 | 工程含义 | 常见位置 |
|------|----------|----------|
| PWM | Pulse Width Modulation，脉冲宽度调制，固定频率下改变高电平占比 | [23-1] |
| 占空比 | 高电平时间 / 周期 | [23-1] |
| PSC | Prescaler，预分频器，决定计数频率 | [23-1] |
| ARR | Auto Reload Register，自动重装载值，决定周期 | [23-1] |
| CNT | Counter，定时器当前计数值 | [23-1] |
| CCR | Capture/Compare Register，比较值，决定 PWM 高电平持续多久 | [23-1] |
| 输出比较 | CNT 与 CCR 比较后改变输出状态，PWM 是它的一种模式 | [23-1] |
| 互补输出 | 高级定时器同一通道输出一对反相信号 `CHx/CHxN` | [22-2] |
| 死区时间 | 互补输出切换时两路都关闭的保护时间 | [22-2] |
| 刹车输入 | 故障时硬件强制关闭 PWM 输出 | [22-2] |
| 舵机 | 通过 PWM 脉宽控制角度的位置伺服电机 | [23-1] |

### PWM HAL API

| 函数 / 宏 | 作用 |
|-----------|------|
| `HAL_TIM_PWM_Start(&htim, Channel)` | 启动 PWM 输出 |
| `HAL_TIM_PWM_Stop(&htim, Channel)` | 停止 PWM 输出 |
| `HAL_TIM_PWM_Start_IT(&htim, Channel)` | 启动 PWM 并开启相关中断 |
| `HAL_TIM_PWM_Stop_IT(&htim, Channel)` | 停止 PWM 中断 |
| `__HAL_TIM_SET_COMPARE(&htim, Channel, Value)` | 动态修改 CCR |
| `__HAL_TIM_GET_COMPARE(&htim, Channel)` | 读取 CCR |
| `HAL_TIM_PWM_PulseFinishedCallback(htim)` | PWM 脉冲完成回调 |

---

## 本质理解

PWM 的本质是：**频率固定，改变高电平持续时间**。

```text
周期固定：
|<--------- T --------->|

占空比 25%：
HIGH ____ LOW __________

占空比 50%：
HIGH ________ LOW ______

占空比 75%：
HIGH ____________ LOW __
```

对于 LED，人眼看到的是平均亮度；对于舵机，舵机识别的是高电平脉宽；对于电机，电机感受到的是平均电压/电流。

---

## 工作流程 / 原理

### 通用定时器能做什么

SysTick 主要给 HAL 提供 1 ms 时基；通用定时器功能更完整。

| 功能 | SysTick | TIM2/3/4 |
|------|---------|----------|
| 定时中断 | 支持，常用于 HAL 时基 | 支持，周期灵活 |
| PWM 输出 | 不支持 | 支持 |
| 输出比较 | 不支持 | 支持 |
| 输入捕获 | 不支持 | 支持 |
| 编码器模式 | 不支持 | 支持 |
| TRGO | 不支持 | 支持 |

### 定时器类型

STM32F103C8T6 常见定时器：

| 定时器 | 类型 | 总线 | 常见时钟 | 通道 |
|--------|------|------|----------|------|
| TIM1 | 高级定时器 | APB2 | 72 MHz | 4 通道 + 互补输出 |
| TIM2 | 通用定时器 | APB1 | 72 MHz | 4 通道 |
| TIM3 | 通用定时器 | APB1 | 72 MHz | 4 通道 |
| TIM4 | 通用定时器 | APB1 | 72 MHz | 4 通道 |

TIM2/3/4 在 APB1 上，但当 APB1 预分频为 `/2` 时，定时器时钟会变为 `PCLK1 x 2 = 72 MHz`。

### 高级定时器 TIM1

高级定时器比通用定时器多了面向电机/功率控制的功能：

```text
通用定时器：
CNT / PSC / ARR / CCR
-> 定时、PWM、输入捕获、输出比较

高级定时器：
通用定时器功能
+ 互补输出 CHxN
+ 死区时间
+ 刹车输入
+ 重复计数器 RCR
```

只做基础 PWM 或定时中断时，TIM1 和 TIM2/3/4 的使用方式很接近；互补、死区、刹车主要用于电机驱动和功率电路。

### PWM 产生逻辑

以 PWM Mode 1 为例：

```text
CNT: 0 -> 1 -> ... -> CCR -> ... -> ARR -> 0
     |<---- 高电平 ---->|<---- 低电平 ---->|

CNT < CCR  -> 输出有效电平
CNT >= CCR -> 输出无效电平
```

PWM Mode 2 与 Mode 1 电平相反，数学关系相同。

### 频率和占空比公式

```text
计数频率 = TIMx_CLK / (PSC + 1)

PWM 周期 = (PSC + 1) x (ARR + 1) / TIMx_CLK

PWM 频率 = TIMx_CLK / [(PSC + 1) x (ARR + 1)]

高电平时间 = CCR x (PSC + 1) / TIMx_CLK

占空比 = CCR / (ARR + 1) x 100%
```

运行时通常只改 `CCR`，这样占空比变化，PWM 频率保持不变。

### 舵机控制原理

常见舵机使用 50 Hz PWM，周期 20 ms。角度由高电平脉宽决定。

使用 `TIMx_CLK = 72 MHz`，`PSC = 71`，`ARR = 19999`：

```text
计数频率 = 72 MHz / 72 = 1 MHz
1 个计数 = 1 us
周期 = 20000 us = 20 ms
频率 = 50 Hz
```

脉宽与 CCR：

| 脉宽 | CCR | 角度参考 |
|------|-----|----------|
| 0.5 ms | 500 | -90° |
| 1.0 ms | 1000 | -45° |
| 1.5 ms | 1500 | 中点 |
| 2.0 ms | 2000 | +45° |
| 2.5 ms | 2500 | +90° |

不同舵机的安全脉宽范围不同，先用 `1000~2000` 测试，确认无卡死再扩大。

### 呼吸灯原理

LED 调光利用平均电流和人眼视觉暂留：

- PWM 频率足够高时，人眼不感到闪烁。
- 占空比越高，平均电流越大，视觉越亮。
- LED 调光通常建议 PWM 频率 1 kHz 以上。

---

## CubeMX 配置

### 舵机 PWM

1. `Timers -> TIM3`。
2. `Channel1` 选择 `PWM Generation CH1`。
3. 设置：
   - `Prescaler` = `71`
   - `Counter Period` = `19999`
   - `Pulse` = `1500`
4. 确认引脚自动变成复用推挽输出，例如 `PA6 = TIM3_CH1`。
5. 代码中启动：

```c
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);
```

### LED PWM

1. 选择支持 TIM PWM 的引脚。
2. 配置对应定时器通道为 `PWM Generation CHx`。
3. 例如 1 kHz PWM：

```text
TIMx_CLK = 72 MHz
PSC = 71       -> 计数频率 1 MHz
ARR = 999      -> 周期 1000 us
PWM = 1 kHz
```

4. 初始 `Pulse` 可设为 0。

---

## HAL 函数 / API

### `HAL_TIM_PWM_Start`

函数：

```c
HAL_StatusTypeDef HAL_TIM_PWM_Start(TIM_HandleTypeDef *htim,
                                    uint32_t Channel);
```

作用：

- 调用后，指定 TIM 通道开始输出 PWM。
- 不调用时，CubeMX 虽然完成初始化，但引脚不会真正输出 PWM 波形。
- 适合舵机控制、LED 调光、电机调速等场景。

```c
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);
```

注意：CubeMX 生成初始化不等于 PWM 已经输出，必须显式调用 `Start`。

### `__HAL_TIM_SET_COMPARE`

函数：

```c
__HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, 1500);
```

作用：

- 调用后，指定通道的 `CCR` 会被改写。
- PWM 输出的占空比或脉宽随之改变。
- 不调用时，PWM 会一直保持初始化时的占空比。
- 适合实时改变 LED 亮度、舵机角度或电机占空比。

参数说明：

- `&htim3`：定时器句柄。
- `TIM_CHANNEL_1`：通道。
- `1500`：新的 CCR 值。

这是宏，不是普通函数。CCR 必须在合理范围内，通常不应超过 `ARR + 1`。

---

## 示例代码

### 舵机控制

```c
/* USER CODE BEGIN 2 */
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);
/* USER CODE END 2 */

while (1)
{
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, 1000);
    HAL_Delay(1000);

    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, 1500);
    HAL_Delay(1000);

    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, 2000);
    HAL_Delay(1000);
}
```

### 舵机角度函数

```c
void Servo_SetPulse(uint16_t pulseUs)
{
    if (pulseUs < 500)
    {
        pulseUs = 500;
    }
    else if (pulseUs > 2500)
    {
        pulseUs = 2500;
    }

    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, pulseUs);
}
```

### 呼吸灯教学写法

```c
/* USER CODE BEGIN 2 */
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);
/* USER CODE END 2 */

while (1)
{
    for (uint16_t duty = 0; duty <= 1000; duty++)
    {
        __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, duty);
        HAL_Delay(1);
    }

    for (uint16_t duty = 1000; duty > 0; duty--)
    {
        __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, duty);
        HAL_Delay(1);
    }
}
```

这个写法直观，但 `HAL_Delay()` 会阻塞 CPU。工程里可改成定时器中断或主循环状态机。

### 非阻塞呼吸灯思路

```c
uint16_t duty = 0;
int8_t dir = 1;
uint32_t lastTick = 0;

void Breath_Task(void)
{
    if (HAL_GetTick() - lastTick >= 2)
    {
        lastTick = HAL_GetTick();

        __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, duty);

        if (dir > 0)
        {
            duty++;
            if (duty >= 1000)
            {
                dir = -1;
            }
        }
        else
        {
            duty--;
            if (duty == 0)
            {
                dir = 1;
            }
        }
    }
}

while (1)
{
    Breath_Task();
    // 其他任务可以继续运行
}
```

---

## 代码执行流程

```text
CubeMX 配置 TIM PWM 通道
↓
main() 初始化 GPIO 和 TIM
↓
HAL_TIM_PWM_Start() 启动 PWM 输出
↓
TIM 从 0 计数到 ARR
↓
CNT 与 CCR 比较后自动改变输出电平
↓
修改 CCR 改变占空比或脉宽
↓
外设表现为 LED 亮度变化或舵机角度变化
```

---

## 常见错误

| 问题 | 常见原因 | 解决方法 |
|------|----------|----------|
| PWM 引脚无输出 | 忘记 `HAL_TIM_PWM_Start()` | 初始化后显式启动 |
| PWM 引脚无输出 | 通道和引脚不匹配 | 查 CubeMX Pinout 和数据手册 |
| 占空比异常 | CCR 超过 ARR | 限制 CCR 范围 |
| 舵机不转 | 供电不足或未共地 | 外接舵机电源，并与 STM32 共地 |
| 舵机抖动 | 电源电流不够、脉宽超限 | 换独立电源，缩小脉宽范围 |
| 舵机角度不准 | 不同舵机脉宽范围不同 | 查手册或实测标定 |
| 呼吸灯闪烁 | PWM 频率太低 | 提高到 1 kHz 以上 |
| CPU 被占满 | 用大量 `HAL_Delay()` 做渐变 | 改成状态机或定时器节拍 |
| 用 TIM1 没输出 | 高级定时器输出使能配置不完整 | 优先用 TIM2/3/4 入门，或确认主输出使能 |

---

## 调试方法

### PWM 输出调试

1. 确认 `HAL_TIM_PWM_Start()` 已执行。
2. 确认通道号和引脚一致，例如 `TIM3_CH1` 对应的实际引脚。
3. 用示波器/逻辑分析仪看 PWM 波形。
4. 没工具时，可用 LED + 电阻观察亮度变化。
5. 检查 `PSC/ARR/CCR` 是否符合预期。

### 舵机调试

1. 舵机单独外接 5 V 或额定电源。
2. 舵机 GND 与 STM32 GND 共地。
3. 先输出 1500 us，让舵机回中位。
4. 再测试 1000 us 和 2000 us。
5. 不要一开始就打到 500/2500 us，避免机械卡死。

---

## 工程意义

PWM 是嵌入式里非常通用的控制手段：

- LED 调光。
- 蜂鸣器发声。
- 舵机位置控制。
- 直流电机调速。
- 开关电源控制。
- 半桥/全桥驱动。

定时器 PWM 的核心价值是：**波形由硬件持续输出，CPU 只需要偶尔改 CCR。**

---

## 易混淆点

| 容易混淆 | 正确理解 |
|----------|----------|
| PWM 频率和占空比 | 频率由 PSC/ARR 决定，占空比由 CCR 决定 |
| CCR 和 ARR | ARR 决定周期，CCR 决定高电平时间 |
| PWM Mode 1 和 Mode 2 | 只是有效电平相反 |
| 舵机控制占空比还是脉宽 | 舵机看的是脉宽，50 Hz 下才对应固定占空比 |
| LED 调光和舵机 PWM | LED 关心平均亮度，舵机关心特定脉宽 |
| TIM1 和 TIM3 | TIM1 是高级定时器，多了互补/死区/刹车 |
| `HAL_TIM_PWM_Start` 和 CubeMX 初始化 | CubeMX 只配置，Start 才真正输出 |

---

## 我的理解

PWM 这章的关键是把 `PSC / ARR / CCR` 三个数和真实波形对应起来。

我现在的理解是：

- `PSC` 决定定时器数得快不快。
- `ARR` 决定一个周期数到哪里结束。
- `CCR` 决定什么时候从高电平变低电平。
- 舵机不是看“亮度”那种占空比，而是看高电平持续了多少微秒。
- 呼吸灯只是反复改 CCR，让平均亮度慢慢变化。

以后遇到 PWM，我先按这个顺序算：

```text
确认 TIMx_CLK
-> 选目标 PWM 频率
-> 选 PSC 得到方便的计数单位
-> 算 ARR
-> 按脉宽/占空比算 CCR
-> 启动 PWM
-> 运行时只改 CCR
```

---

*课程：[22-1] [22-2] [23-1] [23-2]*
