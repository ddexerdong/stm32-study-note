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

## HAL API

| API | 用途 |
|---|---|
| `HAL_GPIO_WritePin` | 方向和 STBY |
| `HAL_TIM_PWM_Start` | 启动 PWM |
| `__HAL_TIM_SET_COMPARE` | 更新速度 |
| `HAL_TIM_PWM_Stop` | 安全停止输出 |

## BSP 核心框架

> 代码性质：示例框架，用于理解流程，不能保证直接编译。

```c
void Motor_Init(void);
void Motor_SetSpeed(int16_t percent);
void Motor_Forward(void);
void Motor_Backward(void);
void Motor_Stop(void);

void Motor_SetSpeed(int16_t percent)
{
    percent = Clamp(percent, -100, 100);
    Motor_SetDirectionFromSign(percent);
    __HAL_TIM_SET_COMPARE(&htimx, TIM_CHANNEL_x,
                          PercentToCcr(Abs(percent)));
}
```

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
