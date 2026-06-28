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
- GPIO、AFIO、USART、ADC、TIM 等外设使用前需要 RCC 使能相应总线时钟，CubeMX 通常生成与实例匹配的宏，例如 `__HAL_RCC_GPIOA_CLK_ENABLE()`。
- APB 分频不为 1 时，定时器内核时钟可能是对应 PCLK 的 2 倍；详细规则以 RM0008 为准。

## CubeMX 配置要点

1. 在 RCC 选择 HSI/HSE 和晶振类型。
2. 进入 Clock Configuration，设置 PLL 和各总线分频。
3. 确认 PCLK1 不超过器件限制，ADC 时钟满足参考手册要求。
4. 生成代码后查看 `SystemClock_Config()` 的 Oscillator/Clock 配置结构体。

## 常用 HAL API

### API 速查

| 函数 / 宏 | 类型 | 解决什么问题 | 常见调用位置 |
|---|---|---|---|
| `HAL_RCC_OscConfig()` | HAL API | 配置 HSI/HSE/PLL | `SystemClock_Config()` |
| `HAL_RCC_ClockConfig()` | HAL API | 选择 SYSCLK 并设置 AHB/APB 分频和 Flash 延时 | OscConfig 成功后 |
| `HAL_RCC_GetSysClockFreq()` | HAL API | 计算当前 SYSCLK | 调试/诊断 |
| `HAL_RCC_GetPCLK1Freq()` / `HAL_RCC_GetPCLK2Freq()` | HAL API | 获取 APB 外设总线时钟 | USART/CAN/ADC/TIM 诊断 |
| `__HAL_RCC_GPIOA_CLK_ENABLE()` 等实例宏 | HAL 宏 | 给具体 GPIO/外设打开总线时钟 | MSP/`MX_xxx_Init()` |
| `SystemCoreClockUpdate()` | CMSIS 系统函数，非 HAL API | 更新 `SystemCoreClock` 变量 | 手工改 RCC 后 |

### 使用注意

| 函数 / 宏 | 前置条件 | 返回值 / 阻塞 | 常见坑 |
|---|---|---|---|
| `HAL_RCC_OscConfig()` | 振荡源和 PLL 参数符合芯片/板卡 | 返回 `HAL_StatusTypeDef`；同步等待就绪 | HSE 类型、PLL 倍频照抄其它板卡 |
| `HAL_RCC_ClockConfig()` | 目标频率和 Flash Latency 匹配 | 返回状态；同步切换 | Flash 延时错误；APB 分频影响 TIM 时钟 |
| `HAL_RCC_GetSysClockFreq()` | RCC 已配置 | 返回 Hz；非阻塞 | 把 SYSCLK 直接当所有外设时钟 |
| `HAL_RCC_GetPCLK1Freq()` / `HAL_RCC_GetPCLK2Freq()` | RCC 已配置 | 返回 Hz；非阻塞 | 忽略 APB 分频时 TIM 时钟可能 x2 |
| `__HAL_RCC_GPIOA_CLK_ENABLE()` 等实例宏 | 宏与实际外设实例匹配 | 宏；非阻塞 | 未开时钟就访问外设；把宏写在业务循环反复调用 |
| `SystemCoreClockUpdate()` | 时钟切换完成 | `void`；同步计算 | 变量与真实 HCLK 不一致导致 DWT/延时换算错 |

### 典型调用顺序

```text
确认开发板 HSE/HSI 条件
-> HAL_RCC_OscConfig()
-> HAL_RCC_ClockConfig()
-> 更新/核对 SystemCoreClock
-> MX_xxx_Init() 中使能外设时钟
-> GetSysClock/PCLK 检查实际频率
```

### 什么时候用哪个函数？

| 场景 | 推荐接口 |
|---|---|
| 配置振荡器和 PLL | `HAL_RCC_OscConfig()` |
| 配置 SYSCLK/AHB/APB | `HAL_RCC_ClockConfig()` |
| 调试总线频率 | `HAL_RCC_GetSysClockFreq()` / `GetPCLKxFreq()` |
| 使能某外设时钟 | 使用与外设实例匹配的时钟使能宏，通常由 CubeMX 生成 |

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
typedef struct
{
    uint32_t sysclk_hz;
    uint32_t hclk_hz;
    uint32_t pclk1_hz;
    uint32_t pclk2_hz;
} ClockSnapshot_t;

static ClockSnapshot_t Clock_ReadSnapshot(void)
{
    ClockSnapshot_t clock = {
        .sysclk_hz = HAL_RCC_GetSysClockFreq(),
        .hclk_hz   = HAL_RCC_GetHCLKFreq(),
        .pclk1_hz  = HAL_RCC_GetPCLK1Freq(),
        .pclk2_hz  = HAL_RCC_GetPCLK2Freq(),
    };
    return clock;
}

/* main() 中 SystemClock_Config() 完成后调用。 */
ClockSnapshot_t clock_now = Clock_ReadSnapshot();
```

用调试器或串口观察 `clock_now`，再结合 APB 分频判断 TIM 实际时钟。外部晶振和目标主频以实际开发板原理图和 CubeMX Clock Configuration 为准。

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
