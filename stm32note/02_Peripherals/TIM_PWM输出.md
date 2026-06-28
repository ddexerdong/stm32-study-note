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
  - pwm
---
# TIM PWM 输出

> 能力定位：从目标频率和脉宽反推 PSC、ARR、CCR，并可靠输出 PWM。
>
> 原始来源：[[06_定时器PWM]]。

## PWM 模型

```text
PWM_freq = TIMx_CLK / ((PSC + 1) * (ARR + 1))
duty = CCR / (ARR + 1)
```

CCR 决定比较时刻。LED 调光关注平均占空比；舵机通常关注周期内的高电平脉宽，不能只套百分比。

呼吸灯是在保持 PWM 频率基本不变的前提下，按时间缓慢修改 CCR。渐变节拍应由主循环状态机或定时事件推进，避免用长阻塞循环拖慢其它任务。

## HAL 主线

- `HAL_TIM_PWM_Start()`：启动指定通道。
- `__HAL_TIM_SET_COMPARE()`：运行时更新 CCR。
- CubeMX 负责 GPIO 复用、通道模式、PSC 和 ARR。

## 高级定时器常见坑

- MOE/主输出未使能导致没有波形。
- 互补输出通道选错。
- 死区和刹车配置影响输出。
- 预装载使新 ARR/CCR 在更新事件后才生效。

## 调试方法

先用示波器或逻辑分析仪确认频率和脉宽，再连接舵机、电机或灯。通道、引脚和负载供电以实际开发板原理图和 CubeMX 配置为准。

## 关联知识

- [[TIM定时器体系]]
- [[电机控制_TB6612FNG]]
- [[WS2813E_RGB灯]]

---

## 本节目标

- 能计算 PWM 频率、占空比和脉宽
- 能配置并启动一个 PWM 通道
- 能用 CCR 驱动 LED、舵机或电机调速

## 知识地图

| 名词 | 工程含义 |
|---|---|
| PWM | 固定周期内改变有效电平持续时间 |
| 占空比 | 有效时间占周期比例 |
| CCR | 比较点，通常决定有效脉宽 |
| ARR | 周期上限 |
| MOE | 高级定时器主输出使能 |

## 本质理解

PWM 不是“输出模拟电压”，而是高速开关。负载通过平均能量、机械惯性或协议脉宽解释波形。LED 看平均亮度，舵机看脉宽，电机驱动看调制后的平均电压。

为什么这样做：先建立硬件和数据流模型，再记 HAL API，才能在 API 变化或板卡变化时仍然定位问题。

## STM32F103 / HAL 中的实现方式

- 边沿对齐 PWM 中，CNT 从 0 计到 ARR，CCR 决定通道翻转位置。
- F103 GPIO 必须配置为对应 TIM 通道的复用推挽。
- 高级定时器输出链可能受 MOE、互补输出、死区、刹车和预装载控制。

## CubeMX 配置要点

1. 选择 TIM PWM Generation 通道。
2. 设置 PSC/ARR 得到目标频率。
3. 设置 Pulse 初始 CCR。
4. 确认 Pinout 映射和 GPIO AF_PP。
5. 生成后调用 `HAL_TIM_PWM_Start`。

## 常用 HAL API

### API 速查

| HAL 函数 / 宏 | 作用 | 常见位置 |
|---|---|---|
| `HAL_TIM_PWM_Start()` | 启动指定 PWM 通道输出 | 外设初始化后 |
| `HAL_TIM_PWM_Stop()` | 停止指定 PWM 通道 | 安全停机、关闭输出 |
| `__HAL_TIM_SET_COMPARE()` | 修改通道 CCR，改变占空比/脉宽 | 调光、调速、舵机更新 |
| `__HAL_TIM_GET_COMPARE()` | 读取当前通道 CCR | 状态显示、调试 |
| `HAL_TIM_PWM_PulseFinishedCallback()` | 接收 PWM IT/DMA 脉冲完成事件 | 用户回调 |

### 使用注意

| HAL 函数 / 宏 | 前置条件 | 参数重点 | 返回值 / 阻塞与常见坑 |
|---|---|---|---|
| `HAL_TIM_PWM_Start()` | TIM PWM 模式与复用引脚已配置 | 句柄、`TIM_CHANNEL_x` | 返回 `HAL_StatusTypeDef`；非阻塞；CubeMX 配好并不等于已输出；通道参数与引脚不匹配 |
| `HAL_TIM_PWM_Stop()` | PWM 已启动 | 句柄、通道 | 返回状态；同步停止；只把 CCR 设 0 不等于完整停止通道 |
| `__HAL_TIM_SET_COMPARE()` | PWM 通道已初始化 | 句柄、通道、比较值 | 宏；非阻塞；CCR 超过 ARR；单位是计数值，不是直接百分比或微秒 |
| `__HAL_TIM_GET_COMPARE()` | 句柄与通道有效 | 句柄、通道 | 返回比较值；非阻塞；读取值不能证明引脚上真的有波形 |
| `HAL_TIM_PWM_PulseFinishedCallback()` | 使用 PWM IT/DMA 启动方式和相应中断 | `htim`，需判断实例/通道 | `void`；中断上下文；普通 `HAL_TIM_PWM_Start()` 不会因为每个周期自动调用它 |

### 典型调用顺序

```text
CubeMX 配置 TIM PWM、PSC、ARR、Pulse 和复用引脚
-> MX_TIMx_Init()
-> HAL_TIM_PWM_Start(&htimx, TIM_CHANNEL_x)
-> __HAL_TIM_SET_COMPARE() 更新 CCR
-> 示波器确认频率和脉宽
-> 需要时 HAL_TIM_PWM_Stop()
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| 固定频率连续 PWM | `HAL_TIM_PWM_Start()` |
| 运行时改变亮度/速度/舵机脉宽 | `__HAL_TIM_SET_COMPARE()` |
| 查询当前设定的 CCR | `__HAL_TIM_GET_COMPARE()` |
| 彻底停止通道输出 | `HAL_TIM_PWM_Stop()` |
| 需要 DMA 发送一串比较值 | `HAL_TIM_PWM_Start_DMA()`，完成后再考虑 PulseFinished 回调 |

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
static uint32_t PercentToCcr(TIM_HandleTypeDef *htim, uint32_t percent)
{
    if (percent > PWM_DUTY_MAX_PERCENT)
    {
        percent = PWM_DUTY_MAX_PERCENT;
    }
    return ((htim->Init.Period + 1U) * percent) / PWM_DUTY_MAX_PERCENT;
}

static void BreathingLed_Task(void)
{
    static int32_t duty;
    static int32_t step = BREATH_STEP_PERCENT;

    duty += step;
    if ((duty >= (int32_t)PWM_DUTY_MAX_PERCENT) || (duty <= 0))
    {
        step = -step;
    }
    __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1,
                          PercentToCcr(&htim3, (uint32_t)duty));
}

static uint32_t ServoAngleToCcr(uint32_t angle)
{
    uint32_t pulse_span = SERVO_MAX_PULSE_TICKS - SERVO_MIN_PULSE_TICKS;
    return SERVO_MIN_PULSE_TICKS +
           (pulse_span * angle) / SERVO_MAX_ANGLE_DEG;
}

/* main()：MX_TIM3_Init() 后启动一次 PWM。 */
if (HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1) != HAL_OK)
{
    Error_Handler();
}

while (1)
{
    if (BreathingLed_TimeToUpdate())
    {
        BreathingLed_Task();
    }

    if (Servo_CommandReady())
    {
        uint32_t ccr = ServoAngleToCcr(Servo_GetTargetAngle());
        __HAL_TIM_SET_COMPARE(&htim3, SERVO_TIM_CHANNEL, ccr);
    }
}
```

`SERVO_MIN/MAX_PULSE_TICKS`、角度范围、PWM 通道和更新节拍必须按舵机手册与 TIM 计数频率标定。高级定时器主输出、死区和刹车设置以 CubeMX 实际生成结果为准。

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 无波形 | 忘记启动或通道错 | 调用 Start 并核对通道 |
| 引脚不动 | 没有 AF_PP/重映射错 | 检查 Pinout 和 AFIO |
| 频率错误 | TIM 时钟算错 | 回查 RCC/APB x2 |
| 舵机抖动 | 供电不足或脉宽不符 | 独立供电并按型号标定 |
| TIM1 无输出 | MOE/刹车状态 | 检查高级定时器主输出 |

## 与其它章节的关系

- [[TIM定时器体系]] 提供计数模型。
- [[电机控制_TB6612FNG]] 使用 PWM 调速。
- [[蜂鸣器与简单执行器]] 使用 PWM 产生音调。

## 复习检查清单

- [ ] 能否不用看代码说清本章数据流和硬件链路？
- [ ] 能否在 CubeMX 中完成最小配置并说明每个关键选项？
- [ ] 能否写出初始化、启动和回调/轮询的最小框架？
- [ ] 能否用原始电平、波形、返回值或寄存器证明外设在工作？
- [ ] 能否按“硬件 -> 时钟/配置 -> Start -> 回调 -> 业务”排查故障？
- [ ] 能否指出本章哪些参数必须按原理图、手册或实测确认？
