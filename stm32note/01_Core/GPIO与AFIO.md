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
  - gpio
  - afio
---
# GPIO 与 AFIO

> 能力定位：正确选择引脚模式，并理解引脚如何交给片上外设。
>
> 原始来源：[[01_准备与基础认知]]、[[02_GPIO与传感器]]。

## GPIO 模式

| 模式 | 用途 |
|---|---|
| 输入 | 按键、数字传感器、外部状态 |
| 推挽输出 | 主动输出高低电平，适合 LED 和普通控制信号 |
| 开漏输出 | 只能主动拉低，高电平依赖上拉 |
| 模拟模式 | ADC 输入，关闭不必要的数字输入路径 |
| 复用功能 | 把引脚交给 USART、TIM、SPI、I2C 等外设 |

## 上拉、下拉与浮空

输入引脚必须有稳定的默认电平。内部上拉/下拉是否适合，取决于外部电路；不要让业务输入长期悬空。

- 上拉输入：内部弱上拉提供默认高电平，外部器件通常负责拉低。
- 下拉输入：内部弱下拉提供默认低电平，外部器件通常负责拉高。
- 浮空输入：不提供默认电平，只适合外部电路已可靠驱动的信号。
- 模拟输入：关闭数字输入路径，供 ADC 等模拟外设使用。

## AFIO 与重映射

AFIO 负责部分复用功能和外部中断线路由。重映射不是“换一个 GPIO 名字”，而是把同一外设信号切换到另一组支持的引脚。

## EXTI 与 GPIO

GPIO 负责感知引脚电平；EXTI 负责把特定边沿转换成中断/事件。详细机制见 [[NVIC_EXTI_SysTick]]。

## 常用 HAL API

- `HAL_GPIO_ReadPin`
- `HAL_GPIO_WritePin`
- `HAL_GPIO_TogglePin`
- `HAL_GPIO_EXTI_Callback`

## 调试方法

先确认供电和共地，再检查实际引脚、模式、上下拉、有效电平和复用映射。LED、按键和模块接线以实际开发板原理图和 CubeMX 配置为准。

## 关联知识

- [[数字传感器与LM393]]
- [[NVIC_EXTI_SysTick]]
- [[USART与串口协议]]

---

## 本节目标

- 能按电气需求选择 F103 GPIO 模式
- 能完成 LED、按键、DO 传感器和蜂鸣器最小实验
- 能解释 AFIO 重映射与 EXTI 端口选择

## 知识地图

| 名词 | 工程含义 |
|---|---|
| ODR/BSRR | 输出数据与原子置位/复位寄存器 |
| IDR | 输入数据寄存器 |
| CRL/CRH | F103 引脚模式与速度配置寄存器 |
| AFIO | 复用功能、重映射和 EXTI 端口路由 |
| EXTI | 把引脚边沿变成事件/中断 |

## 本质理解

GPIO 是芯片和外部电路的边界。代码里的 SET/RESET 必须翻译成真实电压，再翻译成 LED、按键或模块的有效状态。模式选择错误时，业务逻辑写得再对也不会产生正确电气行为。

为什么这样做：先建立硬件和数据流模型，再记 HAL API，才能在 API 变化或板卡变化时仍然定位问题。

## STM32F103 / HAL 中的实现方式

- F103 每个 GPIO 通过 CRL/CRH 选择输入、输出、模拟或复用模式；输出可选推挽/开漏。
- `HAL_GPIO_WritePin` 通常利用 BSRR 原子改变输出，`ReadPin` 读取 IDR。
- USART/TIM/SPI/CAN 常用复用推挽，I2C 常用复用开漏，ADC 引脚应设 Analog。
- AFIO MAPR 管理部分外设重映射，EXTICR 选择某条 EXTI 来自哪个 GPIO 端口。

## CubeMX 配置要点

1. 在 Pinout 点选 GPIO_Output、GPIO_Input、Analog 或外设复用功能。
2. 在 GPIO 参数页设置上下拉、输出初值和速度。
3. 需要 EXTI 时选择边沿并在 NVIC 开启对应中断。
4. 需要 USART/SPI/TIM 重映射时确认新引脚和实际接线同步。

## 最小实验 / 最小框架

> 代码性质：可直接移植的最小实验框架，变量名需按 CubeMX 实际生成结果调整。

```c
GPIO_PinState key = HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin);
if (key == KEY_ACTIVE_LEVEL)
{
    HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
}
```

## 常见坑

| 现象 | 常见原因 | 处理方法 |
|---|---|---|
| LED 逻辑相反 | 低电平点亮 | 按原理图封装 LED_On/Off |
| PC13 驱动异常 | 该类引脚驱动能力/电路特殊 | 避免直接驱动大负载 |
| 输入随机跳 | 引脚悬空 | 配置上拉/下拉或外部电阻 |
| 按键多次触发 | 机械抖动 | 延时/时间戳/状态机消抖 |
| 重映射后串口不通 | 接线仍在旧引脚 | 同步修改接线与 AFIO |
| EXTI 不进 | 端口路由或 NVIC 错 | 检查 AFIO EXTICR 与中断线 |

## 与其它章节的关系

- [[NVIC_EXTI_SysTick]] 处理 GPIO 边沿中断。
- [[ADC与模拟采样]] 使用 GPIO Analog 模式。
- [[数字传感器与LM393]] 和 [[蜂鸣器与简单执行器]] 是 GPIO 模块案例。

## 复习检查清单

- [ ] 能否不用看代码说清本章数据流和硬件链路？
- [ ] 能否在 CubeMX 中完成最小配置并说明每个关键选项？
- [ ] 能否写出初始化、启动和回调/轮询的最小框架？
- [ ] 能否用原始电平、波形、返回值或寄存器证明外设在工作？
- [ ] 能否按“硬件 -> 时钟/配置 -> Start -> 回调 -> 业务”排查故障？
- [ ] 能否指出本章哪些参数必须按原理图、手册或实测确认？
