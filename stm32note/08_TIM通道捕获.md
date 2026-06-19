# 08 TIM 通道捕获应用

> 覆盖课程：[25-1] [25-2]
>
> 目标：理解 TIM 输入捕获和编码器模式的本质，掌握 CubeMX 配置、HAL API、最小代码、常见错误和调试方法。

---

## 总览

### 知识地图

| 名词 | 工程含义 | 常见位置 |
|------|----------|----------|
| 输入捕获 | 外部边沿到来时，TIM 把当前 `CNT` 保存到 `CCR` | 超声波测距 |
| 编码器模式 | TIM 根据 A/B 相脉冲自动判断方向并加减计数 | 旋转编码器、电机测速 |
| A/B 相 | 增量式编码器的两路方波，相位相差约 90 度 | 编码器 |
| CNT | 定时器当前计数值 | 编码器、输入捕获 |
| CCR | 捕获/比较寄存器，输入捕获时保存边沿时间 | 输入捕获 |
| 上升沿 | 信号从低电平变高电平的瞬间 | 输入捕获 |
| 下降沿 | 信号从高电平变低电平的瞬间 | 输入捕获 |
| TRIG | 超声波模块触发脚，由 STM32 输出 10 us 高电平 | HC-SR04 |
| ECHO | 超声波模块回响脚，高电平宽度表示声波往返时间 | HC-SR04 |
| 脉宽测量 | 测量一个高电平持续了多久 | 输入捕获 |

### 本节关键 API

| API | 用途 |
|-----|------|
| `HAL_TIM_Encoder_Start()` | 启动定时器编码器模式 |
| `__HAL_TIM_GET_COUNTER()` | 读取 `CNT` 计数器 |
| `__HAL_TIM_SET_COUNTER()` | 设置或清零 `CNT` |
| `HAL_TIM_IC_Start()` | 启动输入捕获，不开中断 |
| `HAL_TIM_IC_Start_IT()` | 启动输入捕获，并开启捕获中断 |
| `HAL_TIM_ReadCapturedValue()` | 读取捕获寄存器 `CCR` |
| `__HAL_TIM_GET_FLAG()` | 查询 TIM 标志位 |
| `__HAL_TIM_CLEAR_FLAG()` | 清除 TIM 标志位 |
| `__HAL_TIM_SET_CAPTUREPOLARITY()` | 切换输入捕获边沿 |

---

## 一、编码器模式

### 本节目标

使用 STM32 定时器的 **Encoder Mode 编码器模式**读取增量式旋转编码器，实现：

- 顺时针旋转时计数增加。
- 逆时针旋转时计数减少。
- 通过 `CNT` 读取当前位置或一段时间内的增量。
- 后续可扩展为速度、方向、位移计算。

---

### 核心概念

| 概念 | 说明 |
|------|------|
| TIM 定时器 | STM32 内部硬件计数器，可计时、计数、PWM、输入捕获 |
| 通道捕获 | TIM 通道检测外部输入信号边沿 |
| 编码器模式 | TIM 根据 A/B 两相信号自动判断方向并加减计数 |
| A/B 相 | 编码器输出的两路方波，相位相差约 90 度 |
| CNT | TIM 当前计数值，编码器模式下表示位置或累计脉冲 |
| 上升沿 / 下降沿 | 信号从低到高 / 从高到低的瞬间 |

编码器模式通常使用两个定时器通道：

| 编码器信号 | TIM 通道 | 示例引脚 |
|------------|----------|----------|
| A 相 | TIM3_CH1 | PA6 |
| B 相 | TIM3_CH2 | PA7 |

---

### 本质理解

编码器模式的本质是：

**定时器根据 A/B 相脉冲的先后关系自动加减计数。**

流程：

```text
A/B 相输入
↓
TIM 硬件判断两相信号的先后关系
↓
判断旋转方向
↓
CNT 自动加减
↓
CPU 读取 CNT
```

普通 GPIO 读取编码器时，需要软件判断 A/B 相谁先变化，再决定 `+1` 还是 `-1`。编码器模式把这件事交给 TIM 硬件完成。

主程序只需要读取 `CNT`：

```c
encoder_cnt = __HAL_TIM_GET_COUNTER(&htim3);
```

---

### CubeMX 配置

以 `TIM3` 为例。

#### TIM3 配置

1. `Timers -> TIM3`。
2. `Combined Channels` 选择 `Encoder Mode`。
3. `Encoder Mode` 选择 `Encoder Mode TI1 and TI2`。
4. `Counter Settings`：
   - `Prescaler` = `0`
   - `Counter Period` = `65535`
5. `Input Capture`：
   - `IC1 Polarity` = `Rising Edge`
   - `IC2 Polarity` = `Rising Edge`
   - `IC1 Filter` 初始可设 `0`
   - `IC2 Filter` 初始可设 `0`

#### GPIO 配置

CubeMX 会自动把对应引脚配置为定时器复用输入。

| 信号 | 引脚 | 模式 |
|------|------|------|
| A 相 | PA6 / TIM3_CH1 | TIM 复用输入 |
| B 相 | PA7 / TIM3_CH2 | TIM 复用输入 |

#### NVIC 配置

只读取 `CNT` 时不需要开启中断。

```text
NVIC 不需要勾选 TIM3 global interrupt
```

如果需要处理溢出、测速周期或高级保护，再开启中断。

#### 时钟注意事项

编码器模式主要依赖外部脉冲计数，`Prescaler` 通常设为 `0`。如果后续要用定时器周期中断测速，再按 `TIMx_CLK` 计算定时周期。

---

### HAL 函数 / API

#### `HAL_TIM_Encoder_Start`

函数：

```c
HAL_TIM_Encoder_Start(&htim3, TIM_CHANNEL_ALL);
```

作用：

- 调用后，TIM3 的编码器接口开始工作。
- A/B 相脉冲进入 CH1/CH2 后，硬件会自动更新 `CNT`。
- 不调用时，TIM 不会开始编码器计数，`CNT` 通常不会随旋转变化。

使用场景：

- 读取旋转编码器位置。
- 读取电机编码器增量。
- 后续计算速度和方向。

参数说明：

| 参数 | 说明 |
|------|------|
| `&htim3` | TIM3 句柄 |
| `TIM_CHANNEL_ALL` | 同时启动 CH1 和 CH2 |

#### `__HAL_TIM_GET_COUNTER`

函数：

```c
uint16_t cnt = __HAL_TIM_GET_COUNTER(&htim3);
```

作用：

- 读取当前 `CNT` 值。
- 调用后不会改变计数器。
- 常用于读取当前位置或当前累计脉冲。

不调用会怎样：

- 程序无法知道当前编码器计数值。

参数说明：

| 参数 | 说明 |
|------|------|
| `&htim3` | 要读取的定时器句柄 |

#### `__HAL_TIM_SET_COUNTER`

函数：

```c
__HAL_TIM_SET_COUNTER(&htim3, 0);
```

作用：

- 设置 `CNT` 为指定值。
- 常用于清零位置、设置零点、读取增量后复位。

不调用会怎样：

- `CNT` 会一直累计，超过 `ARR` 后回绕。
- 做周期增量计算时不清零，后续差值不直观。

使用场景：

- 每 100 ms 读取一次编码器增量。
- 机械结构回到零点后重新标定。

---

### 示例代码

#### 读取当前位置

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
#include "main.h"
#include "tim.h"
#include "usart.h"
#include "gpio.h"
#include <stdio.h>

/* USER CODE BEGIN PV */
uint16_t encoder_cnt = 0;
/* USER CODE END PV */

int main(void)
{
    HAL_Init();
    SystemClock_Config();

    MX_GPIO_Init();
    MX_TIM3_Init();
    MX_USART1_UART_Init();

    /* USER CODE BEGIN 2 */
    HAL_TIM_Encoder_Start(&htim3, TIM_CHANNEL_ALL);
    __HAL_TIM_SET_COUNTER(&htim3, 0);
    /* USER CODE END 2 */

    while (1)
    {
        encoder_cnt = __HAL_TIM_GET_COUNTER(&htim3);

        printf("encoder cnt = %u\r\n", encoder_cnt);

        HAL_Delay(200);
    }
}

/* USER CODE BEGIN 4 */
int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
/* USER CODE END 4 */
```

#### 读取一段时间内的增量

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
#include "main.h"
#include "tim.h"
#include "usart.h"
#include "gpio.h"
#include <stdio.h>

/* USER CODE BEGIN PV */
int16_t encoder_delta = 0;
/* USER CODE END PV */

int main(void)
{
    HAL_Init();
    SystemClock_Config();

    MX_GPIO_Init();
    MX_TIM3_Init();
    MX_USART1_UART_Init();

    /* USER CODE BEGIN 2 */
    HAL_TIM_Encoder_Start(&htim3, TIM_CHANNEL_ALL);
    __HAL_TIM_SET_COUNTER(&htim3, 0);
    /* USER CODE END 2 */

    while (1)
    {
        encoder_delta = (int16_t)__HAL_TIM_GET_COUNTER(&htim3);

        __HAL_TIM_SET_COUNTER(&htim3, 0);

        printf("encoder delta = %d\r\n", encoder_delta);

        HAL_Delay(100);
    }
}

/* USER CODE BEGIN 4 */
int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
/* USER CODE END 4 */
```

说明：

- `uint16_t` 适合读取原始 `CNT`。
- `int16_t` 适合表示正反方向的短时间增量。
- 如果使用 `printf("%.2f")` 等浮点输出，需要在 CubeIDE 链接选项添加 `-u _printf_float`。

---

### 代码执行流程

```text
初始化 GPIO / TIM3 / USART
↓
HAL_TIM_Encoder_Start()
↓
编码器 A/B 相输入 TIM3_CH1 / TIM3_CH2
↓
TIM 自动判断方向
↓
CNT 自动加减
↓
主循环读取 CNT
↓
需要增量时读取后清零 CNT
```

---

### 常见错误

| 错误 | 原因 | 解决方法 |
|------|------|----------|
| 读数一直为 0 | 没有调用 `HAL_TIM_Encoder_Start()` | 初始化后启动编码器 |
| 方向反了 | A/B 相接反 | 交换 A/B 相，或调整极性 |
| 数值跳动 | 编码器信号抖动 | 增加 IC Filter，检查接线 |
| 数值变化太快 | 编码器模式计数倍率和预期不同 | 确认 Encoder Mode 和边沿设置 |
| 引脚无效 | 通道和 GPIO 不匹配 | 查 CubeMX Pinout 和数据手册 |
| 计数溢出 | `CNT` 超过 `ARR` 后回绕 | 使用增量读取，或定期清零 |
| 负方向增量异常 | 用 `uint16_t` 解释负方向变化 | 增量计算时转换为 `int16_t` |
| 串口无输出 | 没做 `printf` 重定向 | 实现 `__io_putchar()` |
| `while` 后误加分号 | 循环体变成空语句 | 检查 `while (...) ;` |

---

### 调试方法

#### 串口 `printf`

```c
printf("cnt = %u\r\n", __HAL_TIM_GET_COUNTER(&htim3));
```

观察旋转时计数是否变化，方向是否符合预期。

#### Debug

在读取 `CNT` 的位置打断点，观察：

- `htim3.Instance->CNT`
- `encoder_cnt`
- `encoder_delta`

#### 观察 CNT

CubeIDE Debug 中观察：

```text
TIM3->CNT
```

旋转编码器时，`CNT` 应随方向增加或减少。

#### 逻辑分析仪 / 示波器

观察 A/B 相：

- 是否有方波。
- 相位是否相差约 90 度。
- 是否有明显抖动。
- A/B 相是否接反。

---

### 易混淆点

| 易混淆点 | 正确理解 |
|----------|----------|
| 编码器模式和普通计数 | 普通计数只数脉冲，编码器模式还能根据 A/B 相判断方向 |
| CNT 和 CCR | 编码器模式主要看 `CNT`；输入捕获主要看 `CCR` |
| A 相和 B 相 | 两路信号相位不同，先后关系决定方向 |
| `uint16_t` 和 `int16_t` | 原始 `CNT` 可用 `uint16_t`，正负增量更适合 `int16_t` |
| 开启 CubeMX 配置和启动 TIM | CubeMX 只生成初始化，代码里还必须调用 `HAL_TIM_Encoder_Start()` |

---

### 工程意义

编码器模式常用于：

- 电机转速测量。
- 电机位置反馈。
- 旋钮菜单输入。
- 轮式机器人里程计。
- 闭环控制系统。

工程价值：**方向判断和计数由硬件完成，CPU 只需要读取结果。**

---

### 我的理解

我现在把编码器模式理解成“硬件版方向计数器”。

编码器给 TIM 输入 A/B 两路脉冲，TIM 不只是数脉冲，还会根据两路信号的先后顺序判断方向。最后结果体现在 `CNT` 上。
所以我写代码时不需要自己判断 A/B 相，只要启动编码器模式，然后读 `CNT`。

---

## 二、超声波测距

### 本节目标

使用 STM32 TIM 输入捕获测量超声波模块 `ECHO` 引脚的高电平持续时间，并换算距离。

常见模块：HC-SR04。

最终实现：

```text
TRIG 输出 10 us 高电平
↓
超声波模块发射声波
↓
ECHO 输出高电平
↓
TIM 测量 ECHO 高电平宽度
↓
根据时间计算距离
```

---

### 核心概念

| 概念 | 说明 |
|------|------|
| TIM 定时器 | 提供高精度计数 |
| 输入捕获 | 外部边沿到来时，把当前 `CNT` 保存到 `CCR` |
| 上升沿 | `ECHO` 从低电平变高电平，表示测距开始 |
| 下降沿 | `ECHO` 从高电平变低电平，表示测距结束 |
| CNT | TIM 当前计数值 |
| CCR | 捕获寄存器，保存边沿到来时的 `CNT` |
| TRIG | 超声波模块触发脚，由 STM32 输出 10 us 高电平 |
| ECHO | 超声波模块回响脚，高电平宽度表示声波往返时间 |
| 脉宽测量 | 测量一个高电平持续了多长时间 |

---

### 本质理解

超声波测距本质是：

**测量 `ECHO` 高电平持续时间，再根据声速换算距离。**

流程：

```text
TRIG 输出 10 us 高电平
↓
模块发射超声波
↓
ECHO 拉高
↓
声波遇到障碍物后返回
↓
ECHO 拉低
↓
TIM 捕获上升沿和下降沿
↓
计算 ECHO 高电平时间
↓
换算距离
```

距离公式：

```text
距离 = 声速 x 时间 / 2
```

空气中声速近似：

```text
340 m/s = 0.034 cm/us
```

所以：

```text
distance_cm = echo_time_us x 0.034 / 2
distance_cm = echo_time_us / 58.0
```

---

### CubeMX 配置

以 `TIM2_CH1` 测量 `ECHO` 为例。

#### GPIO 配置

| 信号 | 示例引脚 | 模式 |
|------|----------|------|
| TRIG | PB0 | GPIO_Output |
| ECHO | PA0 / TIM2_CH1 | Input Capture |

注意：HC-SR04 的 `ECHO` 高电平通常是 5 V。STM32F103 部分引脚可 5 V 容忍，但并非所有引脚和所有模式都安全。工程上建议使用电阻分压或电平转换，把 `ECHO` 降到 3.3 V。

#### TIM2 配置

1. `Timers -> TIM2`。
2. `Clock Source` 选择 `Internal Clock`。
3. `Channel1` 选择 `Input Capture direct mode`。
4. `Prescaler` = `71`。
5. `Counter Period` = `65535`。
6. `Input Capture Polarity` = `Rising Edge`。
7. `IC Filter` 初始可设 `0`。
8. `NVIC Settings` 勾选 `TIM2 global interrupt`。

若 `TIM2_CLK = 72 MHz`：

```text
Prescaler = 71
计数频率 = 72 MHz / (71 + 1) = 1 MHz
1 个 CNT = 1 us
```

这样捕获差值的单位就是微秒。

---

### HAL 函数 / API

#### `HAL_TIM_IC_Start`

函数：

```c
HAL_TIM_IC_Start(&htim2, TIM_CHANNEL_1);
```

作用：

- 启动输入捕获通道。
- 不开启中断，适合轮询捕获标志位。
- 不调用时，TIM 不会开始接收该通道捕获事件。

使用场景：

- 简单轮询捕获。
- 不希望使用中断的低频测量。

参数说明：

| 参数 | 说明 |
|------|------|
| `&htim2` | TIM2 句柄 |
| `TIM_CHANNEL_1` | 使用 CH1 输入捕获 |

#### `HAL_TIM_IC_Start_IT`

函数：

```c
HAL_TIM_IC_Start_IT(&htim2, TIM_CHANNEL_1);
```

作用：

- 启动输入捕获通道。
- 同时开启捕获中断。
- 边沿到来后会进入 `HAL_TIM_IC_CaptureCallback()`。

不调用会怎样：

- 捕获中断不会触发。
- 回调函数不会执行。

使用场景：

- 超声波测距。
- PWM 输入脉宽测量。
- 频率测量。

#### `HAL_TIM_ReadCapturedValue`

函数：

```c
uint32_t value = HAL_TIM_ReadCapturedValue(&htim2, TIM_CHANNEL_1);
```

作用：

- 读取捕获寄存器 `CCR` 中保存的值。
- 这个值是边沿到来瞬间的 `CNT`。

不调用会怎样：

- 程序无法知道边沿发生时的计数值。

参数说明：

| 参数 | 说明 |
|------|------|
| `&htim2` | TIM2 句柄 |
| `TIM_CHANNEL_1` | 读取 CH1 对应的捕获值 |

#### `__HAL_TIM_SET_CAPTUREPOLARITY`

函数：

```c
__HAL_TIM_SET_CAPTUREPOLARITY(&htim2,
                              TIM_CHANNEL_1,
                              TIM_INPUTCHANNELPOLARITY_FALLING);
```

作用：

- 切换输入捕获边沿。
- 测量 `ECHO` 高电平宽度时，先捕获上升沿，再切换到下降沿。

不调用会怎样：

- 如果一直只捕获上升沿，就无法得到高电平结束时间。

使用场景：

- 测量脉宽。
- 同一个通道先后捕获上升沿和下降沿。

#### `__HAL_TIM_GET_FLAG` / `__HAL_TIM_CLEAR_FLAG`

函数：

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
if (__HAL_TIM_GET_FLAG(&htim2, TIM_FLAG_CC1) != RESET)
{
    __HAL_TIM_CLEAR_FLAG(&htim2, TIM_FLAG_CC1);
}
```

作用：

- `GET_FLAG` 用于查询捕获事件是否发生。
- `CLEAR_FLAG` 用于清除捕获标志位。

使用场景：

- 轮询输入捕获。
- 手动处理 TIM 标志位。

说明：

- 使用 HAL 中断回调时，标志位通常由 HAL 框架处理。
- 轮询方式下如果不清标志位，下一次判断可能一直为真。

---

### 示例代码

#### 基于输入捕获中断测距

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
#include "main.h"
#include "tim.h"
#include "usart.h"
#include "gpio.h"
#include <stdio.h>

/* USER CODE BEGIN PV */
volatile uint32_t echo_rise = 0;
volatile uint32_t echo_fall = 0;
volatile uint32_t echo_time_us = 0;

volatile uint8_t capture_state = 0;
volatile uint8_t measure_done = 0;

float distance_cm = 0.0f;
/* USER CODE END PV */

/* USER CODE BEGIN 0 */
static void Delay_us(uint16_t us)
{
    __HAL_TIM_SET_COUNTER(&htim2, 0);
    while (__HAL_TIM_GET_COUNTER(&htim2) < us)
    {
    }
}

static void HCSR04_Trigger(void)
{
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);
    Delay_us(2);

    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);
    Delay_us(10);

    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);
}
/* USER CODE END 0 */

int main(void)
{
    HAL_Init();
    SystemClock_Config();

    MX_GPIO_Init();
    MX_TIM2_Init();
    MX_USART1_UART_Init();

    /* USER CODE BEGIN 2 */
    HAL_TIM_IC_Start_IT(&htim2, TIM_CHANNEL_1);
    /* USER CODE END 2 */

    while (1)
    {
        measure_done = 0;
        capture_state = 0;

        __HAL_TIM_SET_COUNTER(&htim2, 0);
        __HAL_TIM_SET_CAPTUREPOLARITY(&htim2,
                                      TIM_CHANNEL_1,
                                      TIM_INPUTCHANNELPOLARITY_RISING);

        HCSR04_Trigger();

        uint32_t start_tick = HAL_GetTick();

        while (measure_done == 0)
        {
            if (HAL_GetTick() - start_tick > 50)
            {
                printf("measure timeout\r\n");
                break;
            }
        }

        if (measure_done)
        {
            distance_cm = (float)echo_time_us / 58.0f;

            printf("echo = %lu us, distance = %.2f cm\r\n",
                   (unsigned long)echo_time_us,
                   distance_cm);
        }

        HAL_Delay(200);
    }
}

/* USER CODE BEGIN 4 */
void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM2 && htim->Channel == HAL_TIM_ACTIVE_CHANNEL_1)
    {
        if (capture_state == 0)
        {
            echo_rise = HAL_TIM_ReadCapturedValue(htim, TIM_CHANNEL_1);
            capture_state = 1;

            __HAL_TIM_SET_CAPTUREPOLARITY(htim,
                                          TIM_CHANNEL_1,
                                          TIM_INPUTCHANNELPOLARITY_FALLING);
        }
        else
        {
            echo_fall = HAL_TIM_ReadCapturedValue(htim, TIM_CHANNEL_1);

            if (echo_fall >= echo_rise)
            {
                echo_time_us = echo_fall - echo_rise;
            }
            else
            {
                echo_time_us = (65535u - echo_rise + 1u) + echo_fall;
            }

            capture_state = 0;
            measure_done = 1;

            __HAL_TIM_SET_CAPTUREPOLARITY(htim,
                                          TIM_CHANNEL_1,
                                          TIM_INPUTCHANNELPOLARITY_RISING);
        }
    }
}

int __io_putchar(int ch)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
/* USER CODE END 4 */
```

说明：

- `TRIG` 示例使用 `PB0`。
- `ECHO` 示例使用 `TIM2_CH1`。
- `TIM2` 的计数频率配置为 1 MHz，所以 `CNT` 差值单位是 us。
- `echo_*` 和 `measure_done` 在中断和主循环之间共享，所以使用 `volatile`。
- 如果使用 `printf("%.2f")`，CubeIDE 需要添加链接参数 `-u _printf_float`。

---

### 代码执行流程

```text
初始化 GPIO / TIM2 / USART
↓
HAL_TIM_IC_Start_IT() 启动输入捕获
↓
设置捕获边沿为上升沿
↓
TRIG 输出 10 us 高电平
↓
ECHO 拉高
↓
TIM 捕获上升沿，CCR 保存 echo_rise
↓
切换为下降沿捕获
↓
ECHO 拉低
↓
TIM 捕获下降沿，CCR 保存 echo_fall
↓
echo_time_us = echo_fall - echo_rise
↓
distance_cm = echo_time_us / 58.0
↓
串口输出结果
```

---

### 常见错误

| 错误 | 原因 | 解决方法 |
|------|------|----------|
| 没有测量结果 | 没有调用 `HAL_TIM_IC_Start_IT()` | 初始化后启动输入捕获 |
| 只捕获一次 | 没有切换上升沿 / 下降沿 | 第一次捕获后切下降沿，第二次再切回上升沿 |
| 结果一直为 0 | `ECHO` 引脚通道选错 | 检查 `ECHO` 是否接到 `TIMx_CHx` |
| 距离明显不对 | 定时器计数频率不是 1 MHz | 检查 Prescaler 和 `TIMx_CLK` |
| 程序卡死 | 等待 `ECHO` 时没有超时 | 加 `HAL_GetTick()` 超时保护 |
| `while` 后误加分号 | `while (...);` 变成空循环 | 检查循环语句 |
| 定时器溢出 | `ECHO` 时间超过 `ARR` 范围 | 加溢出处理，或缩短超时时间 |
| `ECHO` 电平过高 | HC-SR04 输出 5 V | 使用分压或电平转换 |
| 轮询方式不稳定 | 标志位未清或边沿未切换 | 使用 `__HAL_TIM_CLEAR_FLAG()` 并检查边沿 |
| `printf` 不能输出 float | 未开启浮点打印 | 添加 `-u _printf_float` |
| 测距抖动 | 回波不稳定或目标太小 | 多次测量取平均 |

---

### 调试方法

#### 串口 `printf`

优先打印原始脉宽：

```c
printf("echo_time_us = %lu\r\n", (unsigned long)echo_time_us);
```

如果 `echo_time_us` 正常但距离异常，重点查单位换算。

#### Debug

在回调函数中打断点：

```c
HAL_TIM_IC_CaptureCallback()
```

观察：

- 是否进入回调。
- `capture_state` 是否切换。
- `echo_rise` 是否记录。
- `echo_fall` 是否记录。
- `echo_time_us` 是否合理。

#### 观察 CNT / CCR

CubeIDE Debug 中观察：

```text
TIM2->CNT
TIM2->CCR1
```

上升沿或下降沿到来时，`CCR1` 应保存当时的 `CNT`。

#### 逻辑分析仪 / 示波器

观察：

- `TRIG` 是否有 10 us 高电平。
- `ECHO` 是否有高电平返回。
- `ECHO` 高电平宽度是否与代码计算接近。
- `ECHO` 是否为 5 V，确认 STM32 输入安全。

---

### 易混淆点

| 易混淆点 | 正确理解 |
|----------|----------|
| CNT 和 CCR | `CNT` 是一直运行的计数器；`CCR` 是边沿到来瞬间保存下来的 `CNT` |
| 输入捕获和输出比较 | 输入捕获是外部信号触发记录时间；输出比较是 TIM 到达比较值后改变输出 |
| 上升沿和下降沿 | 上升沿记录高电平开始，下降沿记录高电平结束 |
| `TRIG` 和 `ECHO` | `TRIG` 是 STM32 输出给模块；`ECHO` 是模块输出给 STM32 |
| 计数频率和距离公式 | 只有 1 个 CNT = 1 us 时，`echo_time_us / 58.0` 才能直接用 |
| 中断方式和轮询方式 | 中断靠回调处理；轮询要自己查标志位并清标志 |

---

### 工程意义

超声波测距是输入捕获的典型应用。它训练的是：

- 边沿捕获。
- 脉宽测量。
- 定时器计数单位换算。
- 超时保护。
- 传感器时序理解。

类似方法还可用于：

- PWM 输入测量。
- 频率测量。
- 红外接收脉宽解析。
- 流量计/转速传感器脉冲测量。

工程价值：**定时器硬件负责记录边沿时间，软件只需要计算两个捕获值的差。**

---

### 我的理解

我现在把超声波测距理解成“测一个高电平有多宽”。

`TRIG` 只是启动模块；真正有用的信息在 `ECHO` 的高电平宽度里。
TIM 输入捕获做的事情，就是在上升沿和下降沿分别记一次 `CNT`。两个捕获值相减，就是声波往返时间。再除以 58，就是厘米级距离。

---

## 总结

这一章两个应用都属于 TIM 通道输入能力：

```text
编码器模式：
A/B 相输入
↓
TIM 自动判断方向
↓
CNT 自动加减

超声波测距：
ECHO 输入
↓
TIM 捕获上升沿 / 下降沿
↓
CCR 保存时间点
↓
计算脉宽和距离
```

核心区别：

| 应用 | 主要看什么 | 本质 |
|------|------------|------|
| 编码器模式 | `CNT` | TIM 根据 A/B 相自动计数 |
| 超声波测距 | `CCR` 差值 | TIM 记录两个边沿的时间差 |

---

## 复习检查清单

- [ ] 能说明编码器模式为什么需要 A/B 两相信号。
- [ ] 能解释编码器模式下 `CNT` 自动加减计数的原因。
- [ ] 能区分输入捕获中的上升沿、下降沿和捕获极性切换。
- [ ] 能说明 `CCR` 保存的是边沿到来时的 `CNT` 时间点。
- [ ] 能解释超声波测距中 `TRIG` 和 `ECHO` 分别负责什么。
- [ ] 能按定时器计数频率把 `CCR` 差值换算成时间。
- [ ] 能说明超声波距离异常时为什么先查 TIM 计数频率和 ECHO 电平。
- [ ] 能按启动函数、NVIC、回调、CNT/CCR、边沿极性排查输入捕获问题。
