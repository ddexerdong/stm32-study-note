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
  - timer
---
# TIM 定时器体系

> 能力定位：用统一的计数模型理解定时中断、PWM、输入捕获和触发输出。
>
> 原始来源：[[03_中断与定时器]]、[[06_定时器PWM]]。

## 计数模型

| 寄存器 / 事件 | 含义 |
|---|---|
| PSC | 预分频，决定计数器步进频率 |
| ARR | 自动重装值，决定计数周期 |
| CNT | 当前计数值 |
| Update Event | CNT 溢出或更新时产生的事件 |

```text
counter_clk = TIMx_CLK / (PSC + 1)
update_freq = counter_clk / (ARR + 1)
```

## TIM 时钟计算

TIMx_CLK 来自 APB 总线。当 APB 分频不为 1 时，定时器时钟可能按芯片规则变为 PCLK 的 2 倍。先看 [[RCC时钟树]]，再代公式。

## 类型差异

- 基本/通用定时器：基础定时、PWM、输入捕获等。
- 高级定时器：增加互补输出、死区、刹车和重复计数等能力。

## 基础中断流程

```text
CubeMX 配 PSC/ARR/NVIC
-> HAL_TIM_Base_Start_IT
-> Update Event
-> HAL_TIM_PeriodElapsedCallback
```

## 关联知识

- [[TIM_PWM输出]]
- [[TIM输入捕获与编码器]]
- [[TIM触发ADC]]

---

## 本节目标

- 能按 TIMx_CLK、PSC、ARR 计算周期
- 能区分基本/通用/高级定时器
- 能完成基础更新中断并正确分流回调

## 知识地图

| 名词 | 工程含义 |
|---|---|
| CNT | 当前计数器 |
| PSC | 预分频器 |
| ARR | 自动重装值 |
| CCR | 捕获/比较值 |
| Update Event | 溢出或更新产生的事件 |
| 影子寄存器 | 在更新事件统一装载新值 |

## 本质理解

定时器的统一模型是“时钟驱动 CNT 变化，并在比较或溢出时产生事件”。定时中断、PWM、输入捕获、编码器和 TRGO 都是对同一计数内核的不同使用方式。

为什么这样做：先建立硬件和数据流模型，再记 HAL API，才能在 API 变化或板卡变化时仍然定位问题。

## STM32F103 / HAL 中的实现方式

- F103 定时器挂在 APB1/APB2，总线分频可能使 TIMx_CLK 为 PCLK 的 2 倍。
- PSC/ARR 寄存器按“设置值 + 1”参与分频，公式中必须处理减 1。
- 预装载让 ARR/CCR 在更新事件同步生效，避免周期中途改变造成毛刺。
- TIM1 等高级定时器还支持 MOE、互补输出、死区和刹车。

## CubeMX 配置要点

1. 选择 TIM Base，设置 Prescaler、Counter Period 和计数模式。
2. 需要中断时启用 Update interrupt 和 NVIC。
3. 根据目标周期先算 TIMx_CLK，再反推 PSC/ARR。
4. 生成后调用 `HAL_TIM_Base_Start_IT`。

## 常用 HAL API

| API / 对象 | 作用 | 常见位置 |
|---|---|---|
| `HAL_TIM_Base_Start` | 启动基础计数 | 轮询/触发 |
| `HAL_TIM_Base_Start_IT` | 启动更新中断 | 周期任务 |
| `HAL_TIM_PeriodElapsedCallback` | 更新完成回调 | 设置任务标志 |
| `__HAL_TIM_GET_COUNTER` | 读取 CNT | 调试/测量 |
| `__HAL_TIM_SET_AUTORELOAD` | 更新 ARR | 动态周期 |

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
HAL_TIM_Base_Start_IT(&htim2);

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM2)
    {
        timer_flag = 1;
    }
}
```

## 调试方法

```text
确认 TIMx_CLK
-> 复算 PSC/ARR
-> 确认 Start_IT 返回 HAL_OK
-> 确认 NVIC 和 IRQHandler
-> 断点看 PeriodElapsedCallback
-> 检查 htim 来源和业务标志
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 周期差一倍 | 忽略 APB x2 | 按 RCC 重新算 TIMx_CLK |
| 频率略偏 | PSC/ARR 忘记减 1 | 按寄存器公式计算 |
| 回调不进 | 只初始化未 Start_IT | 显式启动并检查 NVIC |
| 错误定时器触发业务 | 回调未判断 htim | 按 Instance 分流 |
| TIM1 PWM 不输出 | MOE/通道配置缺失 | 检查高级定时器输出链 |

## 与其它章节的关系

- [[RCC时钟树]] 决定 TIMx_CLK。
- [[TIM_PWM输出]]、[[TIM输入捕获与编码器]]、[[TIM触发ADC]] 复用同一计数模型。

## 复习检查清单

- [ ] 能否不用看代码说清本章数据流和硬件链路？
- [ ] 能否在 CubeMX 中完成最小配置并说明每个关键选项？
- [ ] 能否写出初始化、启动和回调/轮询的最小框架？
- [ ] 能否用原始电平、波形、返回值或寄存器证明外设在工作？
- [ ] 能否按“硬件 -> 时钟/配置 -> Start -> 回调 -> 业务”排查故障？
- [ ] 能否指出本章哪些参数必须按原理图、手册或实测确认？
