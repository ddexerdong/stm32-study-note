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

### API 速查

| HAL 函数 / 宏 | 作用 | 常见位置 |
|---|---|---|
| `HAL_TIM_Base_Start()` | 启动基本计数，不开启更新中断 | 初始化后、TRGO 触发源 |
| `HAL_TIM_Base_Start_IT()` | 启动计数并使能更新中断 | 周期任务启动 |
| `HAL_TIM_Base_Stop_IT()` | 停止计数并关闭更新中断 | 暂停周期任务、停机 |
| `HAL_TIM_PeriodElapsedCallback()` | 接收更新事件的用户弱函数回调 | 用户代码 |
| `__HAL_TIM_SET_COUNTER()` | 直接设置 CNT 当前值 | 重新开始测量、复位计数 |
| `__HAL_TIM_GET_COUNTER()` | 读取 CNT 当前值 | 调试、时间/位置采样 |
| `__HAL_TIM_SET_AUTORELOAD()` | 修改 ARR 周期上限 | 运行时变周期 |

### 使用注意

| HAL 函数 / 宏 | 前置条件 | 参数重点 | 返回值 / 阻塞与常见坑 |
|---|---|---|---|
| `HAL_TIM_Base_Start()` | `MX_TIMx_Init()` 已完成 | `TIM_HandleTypeDef *htim` | 返回 `HAL_StatusTypeDef`；非阻塞；误以为调用它会进入周期回调 |
| `HAL_TIM_Base_Start_IT()` | TIM NVIC 已配置 | 定时器句柄 | 返回状态；非阻塞；只开 NVIC 但没调用 `_Start_IT()`，回调仍不会进入 |
| `HAL_TIM_Base_Stop_IT()` | 定时器已以 IT 模式启动 | 定时器句柄 | 返回状态；同步完成；停止后忘记清理业务状态；与普通 `Stop` 混用 |
| `HAL_TIM_PeriodElapsedCallback()` | IRQHandler 调用 `HAL_TIM_IRQHandler()` | `htim`，需判断 `htim->Instance` | `void`；中断上下文；多个 TIM 共用回调却不判断来源；回调中阻塞 |
| `__HAL_TIM_SET_COUNTER()` | TIM 句柄有效 | 句柄、计数值 | 宏；非阻塞；运行中修改会改变当前周期，需理解副作用 |
| `__HAL_TIM_GET_COUNTER()` | TIM 已启动或允许读取静态值 | 定时器句柄 | 返回计数值；非阻塞；把 CNT 当成固定时间，忽略计数频率和溢出 |
| `__HAL_TIM_SET_AUTORELOAD()` | 了解预装载设置 | 句柄、新 ARR | 宏；非阻塞；新值可能在更新事件后才生效；公式仍要处理 `ARR + 1` |

### 典型调用顺序

```text
CubeMX 配置 TIM 时钟、PSC、ARR
-> 生成并调用 MX_TIMx_Init()
-> 需要回调时配置 NVIC
-> HAL_TIM_Base_Start_IT()
-> TIMx_IRQHandler() -> HAL_TIM_IRQHandler()
-> HAL_TIM_PeriodElapsedCallback() 判断 htim->Instance 并置标志
```

### 什么时候用哪个函数？

| 场景 | 推荐函数 |
|---|---|
| 只需要计数或输出 TRGO，不处理中断 | `HAL_TIM_Base_Start()` |
| 周期性进入回调 | `HAL_TIM_Base_Start_IT()` |
| 暂停周期中断 | `HAL_TIM_Base_Stop_IT()` |
| 读取当前计数 | `__HAL_TIM_GET_COUNTER()` |
| 重新从指定计数开始 | `__HAL_TIM_SET_COUNTER()` |

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
static volatile uint8_t tim_period_event;

static uint8_t Timer_Start(void)
{
    __HAL_TIM_SET_COUNTER(&htim2, 0U);
    return HAL_TIM_Base_Start_IT(&htim2) == HAL_OK;
}

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM2)
    {
        tim_period_event = 1U;
    }
    else if (htim->Instance == ANOTHER_TIM_INSTANCE)
    {
        App_OnAnotherTimerPeriod();
    }
}

/* main()：MX_TIM2_Init() 之后启动。 */
if (!Timer_Start())
{
    Error_Handler();
}

while (1)
{
    if (tim_period_event)
    {
        tim_period_event = 0U;
        uint32_t cnt_snapshot = __HAL_TIM_GET_COUNTER(&htim2);
        App_OnTimerPeriod(cnt_snapshot);
    }
}
```

`ANOTHER_TIM_INSTANCE` 仅表示多定时器分流思路，应替换为实际实例或删除该分支。PSC、ARR 和 IRQ 以 CubeMX 配置为准。

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
