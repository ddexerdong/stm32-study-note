---
type: concept
status: refactor
course: stm32-f103-hal
source:
  - local-notes
  - st-reference-manual
  - st-datasheet
  - st-hal-documentation
tags:
  - stm32
  - rcc
  - clock
---
# RCC 时钟树

> 能力定位：从时钟源推导 CPU、总线和外设的真实工作频率。
>
> 原始来源：[[01_准备与基础认知]]、[[06_定时器PWM]]。

## 时钟层级

| 名词 | 工程含义 |
|---|---|
| HSI | 芯片内部高速 RC 时钟 |
| HSE | 外部高速时钟源 |
| PLL | 对输入时钟倍频或分频 |
| SYSCLK | 系统主时钟 |
| AHB | 内核、存储器和 DMA 等使用的高速总线 |
| APB1 / APB2 | 不同外设所在的外设总线 |

## 工程链路

```text
HSI/HSE -> PLL -> SYSCLK -> AHB -> APB1/APB2 -> 外设时钟
```

外设寄存器在初始化前通常需要先使能对应 RCC 时钟。HAL/CubeMX 会生成时钟使能代码，但调试时仍要知道外设挂在哪条总线上。

## TIM 时钟提醒

当 APB 分频不为 1 时，某些 TIM 输入时钟可能是对应 APB 时钟的 2 倍。具体计算见 [[TIM定时器体系]]。

## 调试方法

- 从 CubeMX Clock Configuration 记录 SYSCLK、HCLK、PCLK1、PCLK2。
- 不要把系统主频直接当成所有外设时钟。
- 延时、串口、ADC、TIM 同时异常时，优先回查时钟树。

具体晶振参数以实际开发板原理图和 CubeMX 配置为准。

## 关联知识

- [[STM32体系结构与启动流程]]
- [[TIM定时器体系]]
- [[ADC与模拟采样]]

---

## 本节目标

- 能从 HSI/HSE/PLL 推导 SYSCLK、HCLK、PCLK1/PCLK2
- 能解释外设时钟使能和 TIM x2 规则
- 能从串口乱码或定时偏差反查时钟

## 知识地图

| 名词 | 工程含义 |
|---|---|
| RCC | 复位与时钟控制模块 |
| SYSCLK | 系统主时钟源选择结果 |
| HCLK | AHB/内核总线时钟 |
| PCLK1/PCLK2 | APB1/APB2 外设时钟 |
| PLL | 对输入时钟倍频形成目标频率 |

## 本质理解

时钟树决定所有数字外设的“时间尺”。串口波特率、TIM 周期、ADC 时钟和通信总线速率都不是孤立参数，而是时钟树分频后的结果。先确认真实时钟，再调外设。

为什么这样做：先建立硬件和数据流模型，再记 HAL API，才能在 API 变化或板卡变化时仍然定位问题。

## STM32F103 / HAL 中的实现方式

- F103 常见工程使用 HSE 经 PLL 得到 72 MHz SYSCLK，但外部晶振和分频必须以板卡为准。
- GPIO、AFIO、USART、ADC、TIM 等外设使用前需要 RCC 使能相应总线时钟，CubeMX 通常生成 `__HAL_RCC_xxx_CLK_ENABLE()`。
- APB 分频不为 1 时，定时器内核时钟可能是对应 PCLK 的 2 倍；详细规则以 RM0008 为准。

## CubeMX 配置要点

1. 在 RCC 选择 HSI/HSE 和晶振类型。
2. 进入 Clock Configuration，设置 PLL 和各总线分频。
3. 确认 PCLK1 不超过器件限制，ADC 时钟满足参考手册要求。
4. 生成代码后查看 `SystemClock_Config()` 的 Oscillator/Clock 配置结构体。

## 常用 HAL API

| API / 对象 | 作用 | 常见位置 |
|---|---|---|
| `HAL_RCC_OscConfig` | 配置振荡器和 PLL | SystemClock_Config |
| `HAL_RCC_ClockConfig` | 选择 SYSCLK 并设置总线分频 | SystemClock_Config |
| `HAL_RCC_GetSysClockFreq` | 获取 SYSCLK | 调试打印 |
| `HAL_RCC_GetPCLK1Freq` | 获取 PCLK1 | USART/TIM 计算 |
| `SystemCoreClockUpdate` | 更新内核时钟变量 | 时钟修改后 |

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
uint32_t sysclk = HAL_RCC_GetSysClockFreq();
uint32_t hclk   = HAL_RCC_GetHCLKFreq();
uint32_t pclk1  = HAL_RCC_GetPCLK1Freq();
uint32_t pclk2  = HAL_RCC_GetPCLK2Freq();
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| 串口乱码 | PCLK 或波特率基准错误 | 先确认时钟树再核对串口参数 |
| TIM 周期差一倍 | 忽略 APB 定时器 x2 | 按 RM0008 计算 TIMx_CLK |
| ADC 结果异常 | ADC 时钟超限或不稳定 | 调整 APB2/ADC 分频 |
| 外设寄存器不响应 | 忘记开 RCC 时钟 | 检查生成的时钟使能代码 |
| HSE 启动失败 | 晶振类型与硬件不符 | 按原理图选择 Crystal/Bypass |

## 与其它章节的关系

- [[TIM定时器体系]] 使用 TIMx_CLK。
- [[ADC与模拟采样]] 依赖 ADC 时钟和参考电压。
- [[USART与串口协议]] 的波特率由 PCLK 派生。

## 复习检查清单

- [ ] 能否不用看代码说清本章数据流和硬件链路？
- [ ] 能否在 CubeMX 中完成最小配置并说明每个关键选项？
- [ ] 能否写出初始化、启动和回调/轮询的最小框架？
- [ ] 能否用原始电平、波形、返回值或寄存器证明外设在工作？
- [ ] 能否按“硬件 -> 时钟/配置 -> Start -> 回调 -> 业务”排查故障？
- [ ] 能否指出本章哪些参数必须按原理图、手册或实测确认？
