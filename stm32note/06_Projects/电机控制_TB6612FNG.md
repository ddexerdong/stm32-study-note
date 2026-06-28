---
type: project
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - module-datasheet
tags:
  - stm32
  - motor
  - tb6612fng
  - pwm
---
# 电机控制：TB6612FNG

> 项目目标：组合 GPIO、PWM、电机驱动和工程封装控制有刷电机。
>
> 原始来源：[[09_电机与工程模板]]。

## 能力组合

- [[GPIO与AFIO]]：AIN1/AIN2/STBY 等逻辑控制。
- [[TIM_PWM输出]]：PWMA/PWMB 调速。
- [[BSP工程模板与代码组织]]：封装方向、速度、停止接口。

## 项目流程

```text
确认 VM/VCC 和共地
-> 单独验证方向 GPIO
-> 示波器验证 PWM
-> 使能 STBY
-> 低占空比试转
-> 验证正反转/停止
-> 加入限速和故障处理
```

> 视频待核对：课程接线、电机供电、PWM 通道、STBY、方向/停止/刹车真值表和演示现象。

具体引脚和参数以实际开发板原理图和 CubeMX 配置为准。

---

## 本节目标

- 能把 GPIO 方向、PWM 调速和驱动供电组合成安全控制链。
- 能用 BSP 封装初始化、方向、速度和停止接口。
- 能从电源、使能、方向和 PWM 波形逐层定位故障。

## 知识地图

| 信号/层 | 作用 |
|---|---|
| VM / VCC / GND | 电机功率、逻辑供电和参考地 |
| AIN1/AIN2 | 方向/停止/刹车模式选择 |
| PWMA | PWM 速度控制 |
| STBY | 待机/使能 |
| BSP | 把方向、速度和安全停机封装为接口 |

## 本质理解

STM32 只提供低功率逻辑和 PWM，TB6612FNG 承担电机电流。项目调试必须把“PWM 有波形”“驱动输出有电压”“电机机械转动”分成三层。

## 系统组成与控制流

```text
App 目标速度/方向
-> Motor BSP 限幅和状态机
-> GPIO AIN1/AIN2/STBY
-> TIM PWM PWMA
-> TB6612FNG H 桥
-> 直流电机
```

## CubeMX 配置

1. 方向和 STBY 配 GPIO_Output，默认安全停机。
2. PWMA 对应 TIM 通道配 PWM。
3. 根据目标频率计算 PSC/ARR，先用示波器确认。
4. 具体引脚、极性和真值表按原理图/手册/视频。

## 本项目使用的 HAL 层

### API 速查

| 项目动作 | HAL 函数 / 宏 |
|---|---|
| 设置 STBY 和方向输入 | `HAL_GPIO_WritePin()` |
| 启动 PWM 通道 | `HAL_TIM_PWM_Start()` |
| 更新速度 | `__HAL_TIM_SET_COMPARE()` |
| 停止 PWM 输出 | `HAL_TIM_PWM_Stop()` |
| 非阻塞换向保护计时 | `HAL_GetTick()` |

### 使用注意

| HAL 函数 / 宏 | 前置条件与调用位置 | 返回值 / 阻塞 | 常见坑 |
|---|---|---|---|
| `HAL_GPIO_WritePin()` | GPIO 初始化后，在 `Motor_Forward/Backward/Stop()` 内调用 | `void`；非阻塞 | 有效电平和真值表未按模块手册确认；换向时未先降速 |
| `HAL_TIM_PWM_Start()` | TIM PWM 初始化后，在 `Motor_Init()` 调用一次 | 返回 `HAL_StatusTypeDef`；非阻塞 | CubeMX 配好但未 Start；通道与 PWMA 接线不一致 |
| `__HAL_TIM_SET_COMPARE()` | PWM 已启动，在 `Motor_SetSpeed()` 中限制范围后调用 | 宏；非阻塞 | CCR 超过 ARR；把百分比直接当 CCR |
| `HAL_TIM_PWM_Stop()` | 故障停机或完全关闭通道 | 返回状态；同步停止 | 与“CCR=0 的滑行/制动状态”混为一谈 |
| `HAL_GetTick()` | App 状态机中实现停机等待 | 返回 tick；非阻塞 | 用长 `HAL_Delay()` 阻塞其它控制和通信 |

HAL 只负责 GPIO/PWM。TB6612FNG 的 STBY、方向、停止/制动真值表、电源限制和安全换向策略必须以模块手册、原理图和实测为准。

### 典型调用顺序

```text
MX_GPIO_Init()
-> MX_TIMx_Init()
-> HAL_TIM_PWM_Start()
-> Motor_Init() 进入安全停止态
-> GPIO 设置方向
-> __HAL_TIM_SET_COMPARE() 缓慢提高速度
-> 停机/换向时先降 CCR，再改方向
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| 普通方向/使能控制 | `HAL_GPIO_WritePin()` |
| 初始化连续 PWM | `HAL_TIM_PWM_Start()` |
| 运行时调速 | `__HAL_TIM_SET_COMPARE()` |
| 故障时关闭 PWM 通道 | `HAL_TIM_PWM_Stop()` + 安全 GPIO 状态 |

## BSP 核心框架

> 代码性质：示例框架，用于理解调用顺序，不能保证直接编译。

```c
typedef enum
{
    MOTOR_STOPPED = 0,
    MOTOR_FORWARD,
    MOTOR_BACKWARD
} MotorDirection_t;

static MotorDirection_t motor_direction;
static uint32_t motor_speed_percent;

static uint32_t Motor_PercentToCcr(uint32_t percent)
{
    if (percent > MOTOR_MAX_PERCENT) percent = MOTOR_MAX_PERCENT;
    return ((htim3.Init.Period + 1U) * percent) / MOTOR_MAX_PERCENT;
}

void Motor_Stop(void)
{
    __HAL_TIM_SET_COMPARE(&htim3, MOTOR_TIM_CHANNEL, 0U);
    HAL_GPIO_WritePin(MOTOR_IN1_GPIO_Port, MOTOR_IN1_Pin,
                      MOTOR_STOP_IN1_LEVEL);
    HAL_GPIO_WritePin(MOTOR_IN2_GPIO_Port, MOTOR_IN2_Pin,
                      MOTOR_STOP_IN2_LEVEL);
    motor_direction = MOTOR_STOPPED;
    motor_speed_percent = 0U;
}

void Motor_Init(void)
{
    HAL_GPIO_WritePin(MOTOR_STBY_GPIO_Port, MOTOR_STBY_Pin,
                      MOTOR_STBY_DISABLE_LEVEL);
    Motor_Stop();
    if (HAL_TIM_PWM_Start(&htim3, MOTOR_TIM_CHANNEL) != HAL_OK)
        Error_Handler();
    HAL_GPIO_WritePin(MOTOR_STBY_GPIO_Port, MOTOR_STBY_Pin,
                      MOTOR_STBY_ENABLE_LEVEL);
}

void Motor_Forward(void)
{
    Motor_Stop();
    Motor_WaitSafeDirectionChange();
    HAL_GPIO_WritePin(MOTOR_IN1_GPIO_Port, MOTOR_IN1_Pin,
                      MOTOR_FORWARD_IN1_LEVEL);
    HAL_GPIO_WritePin(MOTOR_IN2_GPIO_Port, MOTOR_IN2_Pin,
                      MOTOR_FORWARD_IN2_LEVEL);
    motor_direction = MOTOR_FORWARD;
}

void Motor_Backward(void)
{
    Motor_Stop();
    Motor_WaitSafeDirectionChange();
    HAL_GPIO_WritePin(MOTOR_IN1_GPIO_Port, MOTOR_IN1_Pin,
                      MOTOR_BACKWARD_IN1_LEVEL);
    HAL_GPIO_WritePin(MOTOR_IN2_GPIO_Port, MOTOR_IN2_Pin,
                      MOTOR_BACKWARD_IN2_LEVEL);
    motor_direction = MOTOR_BACKWARD;
}

void Motor_SetSpeed(int16_t percent)
{
    percent = Clamp(percent, -MOTOR_MAX_PERCENT, MOTOR_MAX_PERCENT);
    if (percent > 0 && motor_direction != MOTOR_FORWARD) Motor_Forward();
    if (percent < 0 && motor_direction != MOTOR_BACKWARD) Motor_Backward();

    motor_speed_percent = (uint32_t)Abs(percent);
    __HAL_TIM_SET_COMPARE(&htim3, MOTOR_TIM_CHANNEL,
                          Motor_PercentToCcr(motor_speed_percent));
}
```

IN1/IN2/STBY 电平和停止/制动真值表必须按 TB6612FNG 手册与模块原理图核对。示例先停机再换向，实际安全等待应改为非阻塞状态机并结合机械负载实测。

## 调试顺序

```text
电机电源/共地
-> STBY
-> AIN1/AIN2 静态电平
-> PWMA 波形
-> 驱动输出
-> 低速空载
-> 正反转/停止
-> 带载和保护
```

## 常见坑

| 现象 | 原因 | 处理 |
|---|---|---|
| 电机不转 | VM/STBY/PWM 缺失 | 分层测量 |
| MCU 复位 | 电机电源冲击 | 独立供电、去耦并共地 |
| 方向反 | 真值表/电机线相反 | 按手册统一接口 |
| 低占空比不转 | 静摩擦和压降 | 设置启动补偿但限流 |
| 驱动发热 | 过流/短路/频率不当 | 断电检查负载与散热 |

## 待核对与扩展

- [ ] AIN1/AIN2/PWMA/STBY 接线和真值表。
- [ ] VM/VCC、电机额定电压电流和 PWM 频率。
- [ ] 过流、堵转和急停策略。

## 复习检查清单

- [ ] 能解释 STM32、驱动器和电机三层职责。
- [ ] 能用示波器验证 PWM。
- [ ] 能实现 Init/Forward/Backward/Stop/SetSpeed。
- [ ] 能设计安全默认状态。
- [ ] 能按电源、逻辑、PWM、功率输出顺序调试。
- [ ] 知道真值表和供电必须查官方手册。
