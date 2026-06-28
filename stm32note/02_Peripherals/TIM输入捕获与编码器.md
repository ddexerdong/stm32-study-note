---
type: peripheral
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - st-reference-manual
  - st-hal-documentation
tags:
  - stm32
  - tim
  - input-capture
  - encoder
---
# TIM 输入捕获与编码器

> 能力定位：用 TIM 测量外部边沿、脉宽、频率和正交编码器位置。
>
> 原始来源：[[08_TIM通道捕获]]。

## 编码器模式

A/B 两相信号相差约 90 度。TIM 硬件根据相位关系自动让 CNT 加减，CPU 周期性读取 CNT 即可得到位置或增量。

关键点：输入滤波、计数范围、溢出、方向和采样周期。

## 输入捕获与超声波测距

输入捕获会在边沿到来时把 CNT 锁存到 CCR。先捕获上升沿，再捕获下降沿，两次 CCR 差值就是高电平持续计数。

```text
pulse_time = delta_count / counter_freq
distance = pulse_time * sound_speed / 2
```

声速、模块电平和计数频率必须按实际环境和硬件确认。

HC-SR04 一类模块可作为输入捕获练习：MCU 产生 TRIG，模块在 ECHO 输出与往返时间相关的脉宽。触发宽度、ECHO 电平和换算常数必须按具体模块手册与实测确认。

## HAL 主线

- `HAL_TIM_Encoder_Start()`
- `HAL_TIM_IC_Start()` / `HAL_TIM_IC_Start_IT()`
- `HAL_TIM_ReadCapturedValue()`
- `__HAL_TIM_SET_CAPTUREPOLARITY()`

## 调试方法

先看 A/B 相或 ECHO 原始波形，再看 CNT/CCR，最后验证换算公式。所有等待边沿流程都要有超时。

## 关联知识

- [[TIM定时器体系]]
- [[NVIC_EXTI_SysTick]]
- [[通用调试方法]]

---

## 本节目标

- 能解释编码器 A/B 相和 CNT 自动加减
- 能用 CCR 测量脉宽
- 能处理方向、溢出和超时

## 知识地图

| 名词 | 工程含义 |
|---|---|
| Encoder Mode | 用两路相位信号驱动 CNT 加减 |
| 输入捕获 | 边沿到来时把 CNT 锁存到 CCR |
| 极性切换 | 选择上升沿或下降沿捕获 |
| 计数溢出 | CNT 越过 ARR 后回绕 |
| 数字滤波 | 抑制输入毛刺 |

## 本质理解

编码器模式让硬件完成“看相位并计数”；输入捕获让硬件完成“在边沿瞬间拍下时间”。CPU 只读取结果，不需要高速轮询每个边沿。


## STM32F103 / HAL 中的实现方式

- F103 TIM 编码器接口使用 CH1/CH2 输入，CNT 方向反映相位关系。
- 输入捕获把 CNT 复制到 CCRx；两次捕获差值需考虑计数器回绕。
- 超声波 ECHO 可能超过一个计数周期，应选择足够计数范围或记录溢出次数。

## CubeMX 配置要点

1. 编码器：选择 Encoder Mode，配置两通道极性/滤波/ARR。
2. 输入捕获：选择 Input Capture direct mode，配置边沿和滤波。
3. 需要回调时启用 TIM NVIC。
4. 按 ECHO 电平范围设计分压或电平转换。

## 常用 HAL API

### API 速查

| HAL 函数 / 宏 | 作用 | 常见位置 |
|---|---|---|
| `HAL_TIM_IC_Start()` | 启动输入捕获，不开启捕获中断 | 轮询标志或硬件链路 |
| `HAL_TIM_IC_Start_IT()` | 启动输入捕获并开启通道中断 | 脉宽/频率测量 |
| `HAL_TIM_ReadCapturedValue()` | 读取边沿到来时锁存的 CCR | 捕获回调、测量处理 |
| `__HAL_TIM_SET_CAPTUREPOLARITY()` | 运行时切换上升/下降沿捕获 | 单通道双边沿测脉宽 |
| `HAL_TIM_IC_CaptureCallback()` | 接收输入捕获事件 | 用户回调 |
| `HAL_TIM_Encoder_Start()` | 启动编码器接口计数 | 初始化后 |
| `HAL_TIM_Encoder_Start_IT()` | 启动编码器接口并开启相关中断 | 需要更新/捕获事件的编码器方案 |
| `__HAL_TIM_GET_COUNTER()` | 读取编码器累计位置 | 固定周期测速/位置任务 |
| `__HAL_TIM_SET_COUNTER()` | 设定编码器/捕获计数起点 | 回零、开始新测量 |

### 使用注意

| HAL 函数 / 宏 | 前置条件 | 参数重点 | 返回值 / 阻塞与常见坑 |
|---|---|---|---|
| `HAL_TIM_IC_Start()` | IC 通道和引脚已配置 | 句柄、通道 | 返回 `HAL_StatusTypeDef`；非阻塞；误以为会进入捕获回调 |
| `HAL_TIM_IC_Start_IT()` | TIM NVIC 已使能 | 句柄、`TIM_CHANNEL_x` | 返回状态；非阻塞；通道选错或 NVIC 未开导致无回调 |
| `HAL_TIM_ReadCapturedValue()` | 对应通道已经捕获 | 句柄、通道 | 返回捕获值；非阻塞；把 CCR 当当前 CNT；忽略回绕和溢出次数 |
| `__HAL_TIM_SET_CAPTUREPOLARITY()` | 通道处于输入捕获模式 | 句柄、通道、极性宏 | 宏；非阻塞；切换边沿后未清状态或重置阶段变量 |
| `HAL_TIM_IC_CaptureCallback()` | IRQHandler 调用 `HAL_TIM_IRQHandler()` | `htim`，并检查 `HAL_TIM_ACTIVE_CHANNEL_1` 等实际活动通道值 | `void`；中断上下文；只判断定时器、不判断活动通道；回调做长计算 |
| `HAL_TIM_Encoder_Start()` | Encoder Mode、CH1/CH2 引脚已配置 | 句柄、通道选择 | 返回状态；非阻塞；A/B 相、极性或滤波不对；忘记启动两个通道 |
| `HAL_TIM_Encoder_Start_IT()` | NVIC 和中断策略已设计 | 句柄、通道选择 | 返回状态；非阻塞；测速通常不必每个边沿中断，盲目使用会增加 CPU 负担 |
| `__HAL_TIM_GET_COUNTER()` | 编码器 TIM 已启动 | 定时器句柄 | 返回 CNT；非阻塞；差分计算未处理回绕；有符号方向转换错误 |
| `__HAL_TIM_SET_COUNTER()` | 清楚运行中改 CNT 的影响 | 句柄、计数值 | 宏；非阻塞；与中断并发修改造成测量跳变 |

### 典型调用顺序

```text
输入捕获：MX_TIMx_Init()
-> HAL_TIM_IC_Start_IT()
-> HAL_TIM_IC_CaptureCallback()
-> HAL_TIM_ReadCapturedValue()
-> 切换边沿/计算差值/处理溢出

编码器：MX_TIMx_Init()
-> HAL_TIM_Encoder_Start()
-> 固定周期读取 __HAL_TIM_GET_COUNTER()
-> 与上次值做带回绕的差分
-> 换算位置或速度
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| 轮询捕获结果 | `HAL_TIM_IC_Start()` + 状态/寄存器检查 |
| 边沿到来立即处理 | `HAL_TIM_IC_Start_IT()` + `HAL_TIM_IC_CaptureCallback()` |
| 编码器位置/低速周期测速 | `HAL_TIM_Encoder_Start()` + 周期读取 CNT |
| 单通道测高电平脉宽 | 捕获中断 + `__HAL_TIM_SET_CAPTUREPOLARITY()` |
| 编码器每边沿都触发软件 | 仅在确有需求时使用 `HAL_TIM_Encoder_Start_IT()` |

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
static uint16_t encoder_last_count;
static volatile uint32_t capture_rising;
static volatile uint32_t capture_width;
static volatile uint8_t capture_wait_falling;
static volatile uint8_t capture_ready;

static int16_t Encoder_ReadDelta(void)
{
    uint16_t now = (uint16_t)__HAL_TIM_GET_COUNTER(&htim3);
    int16_t delta = (int16_t)(now - encoder_last_count);
    encoder_last_count = now;
    return delta;
}

static uint32_t Capture_DeltaWithWrap(uint32_t start, uint32_t end,
                                      uint32_t period)
{
    return (end >= start) ? (end - start)
                          : ((period + 1U - start) + end);
}

void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim)
{
    if ((htim->Instance != CAPTURE_TIM_INSTANCE) ||
        (htim->Channel != HAL_TIM_ACTIVE_CHANNEL_1))
    {
        return;
    }

    uint32_t captured = HAL_TIM_ReadCapturedValue(htim, CAPTURE_TIM_CHANNEL);
    if (!capture_wait_falling)
    {
        capture_rising = captured;
        capture_wait_falling = 1U;
        __HAL_TIM_SET_CAPTUREPOLARITY(htim, CAPTURE_TIM_CHANNEL,
                                      TIM_INPUTCHANNELPOLARITY_FALLING);
    }
    else
    {
        capture_width = Capture_DeltaWithWrap(capture_rising, captured,
                                               htim->Init.Period);
        capture_wait_falling = 0U;
        capture_ready = 1U;
        __HAL_TIM_SET_CAPTUREPOLARITY(htim, CAPTURE_TIM_CHANNEL,
                                      TIM_INPUTCHANNELPOLARITY_RISING);
    }
}

if (HAL_TIM_Encoder_Start(&htim3, TIM_CHANNEL_ALL) != HAL_OK)
{
    Error_Handler();
}
if (HAL_TIM_IC_Start_IT(&htim2, CAPTURE_TIM_CHANNEL) != HAL_OK)
{
    Error_Handler();
}

while (1)
{
    if (Encoder_SpeedSampleDue())
    {
        int16_t delta = Encoder_ReadDelta();
        App_UpdateEncoderSpeed(delta, ENCODER_SAMPLE_PERIOD_MS);
    }

    if (capture_ready)
    {
        capture_ready = 0U;
        App_UpdatePulseWidth(capture_width, CAPTURE_COUNTER_HZ);
        Ultrasonic_UpdateDistance(capture_width, CAPTURE_COUNTER_HZ);
    }
}
```

超声波距离换算所用声速、触发脉宽、ECHO 电平和计数频率以模块手册、原理图和实测为准。若一次脉宽可能跨越多个计数周期，还需累计更新溢出次数。

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 编码器为 0 | 未 Start 或引脚错 | 启动并检查 A/B 波形 |
| 方向反 | A/B 相交换 | 交换接线或软件取反 |
| 计数跳动 | 毛刺/机械抖动 | 使用输入滤波并改善接线 |
| 捕获回调不进 | NVIC/Start_IT 缺失 | 检查中断链 |
| 距离偶尔巨大 | 未处理溢出或超时 | 扩展计数范围并拒绝无效帧 |

## 与其它章节的关系

- [[TIM定时器体系]] 提供 CNT/CCR。
- [[NVIC_EXTI_SysTick]] 解释捕获回调。
- [[通用调试方法]] 说明波形优先调试。

## 复习检查清单

- [ ] 能否不用看代码说清本章数据流和硬件链路？
- [ ] 能否在 CubeMX 中完成最小配置并说明每个关键选项？
- [ ] 能否写出初始化、启动和回调/轮询的最小框架？
- [ ] 能否用原始电平、波形、返回值或寄存器证明外设在工作？
- [ ] 能否按“硬件 -> 时钟/配置 -> Start -> 回调 -> 业务”排查故障？
- [ ] 能否指出本章哪些参数必须按原理图、手册或实测确认？
